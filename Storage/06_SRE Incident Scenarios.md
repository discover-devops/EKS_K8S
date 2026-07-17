# 06 — SRE Incident Scenarios (Storage Edition)

**Series:** Kubernetes Storage: From Ephemeral Containers to Persistent Data
**Format:** Three real incidents, reproduced live — break it on purpose, diagnose it, fix it

---

## Recap from Module 05

We now have the full mechanism: StorageClass → dynamic PV → bound PVC → real EBS volume. This module walks through three real incidents that happen specifically *because* of how that automation gets misconfigured — the same "reproduce, diagnose, fix" format from the Ingress series' real DevOps scenarios chapter.

---

## Scenario 1: The Accidentally Deleted PVC

**What happened:** a developer wanted to restart their pod.

```bash
kubectl delete pod database-pod
```

But they also, accidentally, ran:

```bash
kubectl delete pvc database-pvc
```

With `reclaimPolicy: Delete` in effect, this doesn't just remove a Kubernetes object — it triggers the EBS CSI driver to **delete the actual underlying volume**. All data, gone, irreversibly, the moment that command executed.

### The analogy

The tenant accidentally ripped up their own reservation ticket. Because the policy was set to `Delete`, the property manager didn't wait around to check if that was intentional — they immediately incinerated the physical locker and everything inside it.

###  Reproduce this live

```bash
cat <<'EOF' > scenario1-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: risky-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
EOF

cat <<'EOF' > scenario1-pvc-pod.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: risky-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: risky-storage
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: risky-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'echo "Critical customer data" > /data/important.txt && sleep 3600']
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: risky-pvc
EOF

kubectl apply -f scenario1-storageclass.yaml
kubectl apply -f scenario1-pvc-pod.yaml
kubectl wait --for=condition=Ready pod/risky-pod --timeout=90s
```

Confirm the volume is real:
```bash
export VOLUME_ID=$(kubectl get pv $(kubectl get pvc risky-pvc -o jsonpath='{.spec.volumeName}') -o jsonpath='{.spec.csi.volumeHandle}')
echo $VOLUME_ID
aws ec2 describe-volumes --region $AWS_REGION --volume-ids $VOLUME_ID --query "Volumes[0].State"
```

**Now reproduce the incident:**

```bash
kubectl delete pod risky-pod
kubectl delete pvc risky-pvc
```

```bash
kubectl get pv
aws ec2 describe-volumes --region $AWS_REGION --volume-ids $VOLUME_ID 2>&1
```

**Expected:** the PV is gone, and the AWS CLI call returns `InvalidVolume.NotFound` — the real EBS volume, and everything on it, has been permanently deleted. Reproduced live, exactly as described.

### Stop and think: what should have been configured differently?

### The fix — `Retain`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: critical-database-pv
spec:
  persistentVolumeReclaimPolicy: Retain   # keep storage even if the PVC is deleted
  # ... rest of spec
```

**But there's a deeper problem worth raising:** even with `Retain`, someone can still delete the PVC — `Retain` only protects the underlying *volume*, not the developer's ability to accidentally trigger the deletion in the first place. What else would actually protect against this?

- **RBAC restrictions** — remove `delete` permission on `persistentvolumeclaims` from roles that don't explicitly need it, especially in production namespaces
- **Regular backups and snapshots** — the actual safety net, independent of any Kubernetes-level policy
- **Namespace-level protections** — some clusters use admission webhooks (like Kyverno or OPA Gatekeeper) to outright block PVC deletion on labeled "production-critical" objects

---

## Scenario 2: The Zone Mismatch

**What happened:**
- PVC created, volume provisioned in `us-west-2a`
- Pod later scheduled onto a node in `us-west-2b`
- Pod stuck in `ContainerCreating`: `"Volume is in different availability zone"`

### The analogy

The valet built the storage locker in the East Wing of the building, but the only available apartment room that day was in the West Wing. You simply cannot reach the locker from there — it's not a permissions problem, it's a physical distance problem.

###  Reproduce this live

First, check your own cluster's node AZs — this determines whether we can reproduce the mismatch naturally or need to force it:

```bash
kubectl get nodes -o custom-columns='NAME:.metadata.name,ZONE:.metadata.labels.topology\.kubernetes\.io/zone'
```

If your two nodes are in **different** AZs (common on EKS, since managed node groups spread across AZs by default), we can reproduce this directly:

```bash
export NODE_A=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
export NODE_A_ZONE=$(kubectl get node $NODE_A -o jsonpath='{.metadata.labels.topology\.kubernetes\.io/zone}')
export NODE_B=$(kubectl get nodes -o jsonpath='{.items[1].metadata.name}')
export NODE_B_ZONE=$(kubectl get node $NODE_B -o jsonpath='{.metadata.labels.topology\.kubernetes\.io/zone}')
echo "Node A ($NODE_A) is in $NODE_A_ZONE"
echo "Node B ($NODE_B) is in $NODE_B_ZONE"
```

Create a StorageClass **without** `WaitForFirstConsumer` — using `Immediate` binding instead, which is exactly the misconfiguration that causes this incident in the first place:

```bash
cat <<EOF > scenario2-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: immediate-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: Immediate
reclaimPolicy: Delete
allowedTopologies:
- matchLabelExpressions:
  - key: topology.kubernetes.io/zone
    values:
    - ${NODE_A_ZONE}
EOF

kubectl apply -f scenario2-storageclass.yaml
```

```bash
cat <<'EOF' > scenario2-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: zone-mismatch-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: immediate-storage
  resources:
    requests:
      storage: 1Gi
EOF

kubectl apply -f scenario2-pvc.yaml
kubectl get pvc zone-mismatch-pvc
```

Because `volumeBindingMode: Immediate` provisions the volume **right away**, without waiting to know which node a pod will land on, the volume gets created in `$NODE_A_ZONE` immediately. Now force a pod using this PVC onto `$NODE_B` — the other zone:

```bash
cat <<EOF > scenario2-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: zone-mismatch-pod
spec:
  nodeName: ${NODE_B}
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: zone-mismatch-pvc
EOF

kubectl apply -f scenario2-pod.yaml
kubectl get pod zone-mismatch-pod
kubectl describe pod zone-mismatch-pod | grep -A 5 Events
```

**Expected:** `ContainerCreating`, with an event mentioning volume node affinity conflict / zone mismatch — reproduced. If your two nodes happen to be in the same AZ, this step will succeed instead — in that case, simply note this is the exact scenario that *would* occur on a cluster spanning multiple AZs, which most production EKS clusters do.

### The fix

Use `WaitForFirstConsumer` (Module 05's default) so the volume is never created until the scheduler has already committed to a node — eliminating the possibility of this mismatch entirely, rather than working around it after the fact:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: regional-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer   # the actual fix
```

**The real lesson:** this incident is entirely preventable at the StorageClass level, before a single PVC is ever created — it's a config choice, not something to patch reactively per-incident.

---

## Scenario 3: The Out-of-Space Database

**What happened:**
- PVC requested: `10Gi`
- Database grew to `9.8Gi`
- Kubelet killed the pod: `"disk pressure"`
- Application down

**How would you detect this before it becomes critical? How would you fix it?**

###  Reproduce this live

```bash
cat <<'EOF' > scenario3-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
EOF

cat <<'EOF' > scenario3-pvc-pod.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: tight-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: expandable-storage
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: tight-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'sleep 3600']
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: tight-pvc
EOF

kubectl apply -f scenario3-storageclass.yaml
kubectl apply -f scenario3-pvc-pod.yaml
kubectl wait --for=condition=Ready pod/tight-pod --timeout=90s
```

Fill the volume close to capacity:

```bash
kubectl exec tight-pod -- dd if=/dev/zero of=/data/fill.bin bs=1M count=950
kubectl exec tight-pod -- df -h /data
```

Try writing more than what's left:

```bash
kubectl exec tight-pod -- dd if=/dev/zero of=/data/overflow.bin bs=1M count=200
```

**Expected:** a `No space left on device` error — reproduced, the exact application-level symptom of the reported incident.

### Checking disk usage — the detection side

```bash
kubectl exec tight-pod -- df -h /data
```

Or via the Metrics API, for programmatic monitoring/alerting:

```bash
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods/tight-pod | jq '.containers[].usage'
```

**The detection lesson:** this should never be discovered via an application crash — it should be caught by a monitoring alert on volume utilization crossing some threshold (e.g., 80%), well before the pod-killing disk-pressure event ever happens.

### The fix — live volume expansion

```bash
kubectl get storageclass expandable-storage -o jsonpath='{.allowVolumeExpansion}'
```

Confirm `true`. Then:

```bash
kubectl patch pvc tight-pvc -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'
kubectl get pvc tight-pvc --watch
```

Watch `CAPACITY` update from `1Gi` to `2Gi`, live, with **zero pod restart required** — this is a genuine online resize.

```bash
kubectl exec tight-pod -- df -h /data
kubectl exec tight-pod -- dd if=/dev/zero of=/data/overflow.bin bs=1M count=200
```

Expected: this now succeeds, where it failed before.

---

## Cleanup

```bash
kubectl delete pod risky-pod zone-mismatch-pod tight-pod --ignore-not-found
kubectl delete pvc risky-pvc zone-mismatch-pvc tight-pvc --ignore-not-found
kubectl delete storageclass risky-storage immediate-storage expandable-storage --ignore-not-found
```

---

## Summary

| Scenario | Root cause | Fix |
|---|---|---|
| Accidental PVC deletion | `reclaimPolicy: Delete` + no protection against accidental deletion | `Retain` + RBAC restrictions + backups |
| Zone mismatch | `volumeBindingMode: Immediate` provisioning before the scheduler commits to a node | `WaitForFirstConsumer` |
| Out-of-space database | No proactive monitoring, no expansion configured | `allowVolumeExpansion: true` + utilization alerting |

**The one sentence to carry forward:** *"All three of these incidents share a pattern: the failure mode was fully preventable through a single StorageClass field, chosen correctly upfront — `reclaimPolicy`, `volumeBindingMode`, `allowVolumeExpansion`. None of these needed to be discovered the hard way, in production, at 2am."*

---

*End of Module 06. Module 07: Storage Architecture Patterns — StatefulSets, shared storage, and deciding what actually needs persistence.*
