# 08 — Debugging Storage Issues

**Series:** Kubernetes Storage: From Ephemeral Containers to Persistent Data
**Format:** Concept + hands-on lab on your EKS cluster

---

## Recap from Module 06 and 07

Module 06 walked through three specific incidents end-to-end. This module builds the **general-purpose diagnostic checklist** — the systematic sequence of commands to run for *any* storage problem, before you even know what's wrong yet.

---

## The question to start with

**A pod is stuck. What commands would you actually run, in what order, before you know anything else about the problem?**

## The debugging checklist

```bash
# 1. Check PVC status
kubectl get pvc

# 2. Describe the PVC for events — this is where the real diagnostic information usually is
kubectl describe pvc my-pvc

# 3. Check the PV
kubectl get pv

# 4. Check pod events
kubectl describe pod my-pod

# 5. Check node events
kubectl describe node <node-name>

# 6. Check controller logs (rarely needed, but the last resort)
kubectl logs -n kube-system -l component=kube-controller-manager
```

**Why this specific order?** It moves from the object *closest to the developer's mental model* (the PVC) outward toward infrastructure (PV → pod → node → controller). Most storage problems resolve at steps 1-4; you rarely need to go further than that.

---

## The triage tree

### Issue: PVC stuck in `Pending`

```
Check: kubectl describe pvc
├─ No PV matches requirements → Create a matching PV, or adjust the PVC
├─ StorageClass not found → Create the StorageClass, or fix a typo in storageClassName
└─ Waiting for consumer → Create a pod that actually uses this PVC (recall Module 05's WaitForFirstConsumer)
```

### Issue: Pod stuck in `ContainerCreating`

```
Check: kubectl describe pod
├─ FailedMount → The volume doesn't exist, or the mount path is wrong
├─ FailedAttachVolume → Zone mismatch (Module 06, Scenario 2), or node capacity issue
└─ Timeout → Check the storage backend's own connectivity (CSI driver health)
```

### Issue: Application reports disk full

```
Check: kubectl exec pod -- df -h
├─ PVC itself is full → Resize the PVC (Module 06, Scenario 3)
└─ A DIFFERENT mount is full → Check emptyDir or hostPath usage instead — not every "disk full" is the PVC's fault
```

---

## 🔧 Hands-on lab: reproduce two fresh triage branches we haven't hit yet

### Case A: `StorageClass not found`

```bash
cat <<'EOF' > debug-case-a.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: typo-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: fast-storag3
  resources:
    requests:
      storage: 1Gi
EOF

kubectl apply -f debug-case-a.yaml
kubectl get pvc typo-pvc
```

Expected `STATUS`: `Pending`, indefinitely.

```bash
kubectl describe pvc typo-pvc | grep -A 5 Events
```

**Expected:** either no events at all, or a `ProvisioningFailed` event mentioning the StorageClass couldn't be found. **This is a genuinely tricky one to diagnose** because there's often no loud error — just silence and `Pending` forever. The fix is purely inspecting the spelling:

```bash
kubectl get pvc typo-pvc -o jsonpath='{.spec.storageClassName}'
kubectl get storageclass
```

Comparing these two outputs side by side is usually how this bug actually gets caught in practice — a one-character typo (`fast-storag3` vs `fast-storage`) that's easy to miss just reading the YAML.

```bash
kubectl delete -f debug-case-a.yaml
```

### Case B: `FailedMount` — the volume genuinely doesn't exist

```bash
cat <<'EOF' > debug-case-b.yaml
apiVersion: v1
kind: Pod
metadata:
  name: failedmount-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: nonexistent-pvc
EOF

kubectl apply -f debug-case-b.yaml
kubectl get pod failedmount-pod
```

Expected: stuck `ContainerCreating`.

```bash
kubectl describe pod failedmount-pod | grep -A 5 Events
```

**Expected:** an event referencing the PVC `nonexistent-pvc` not being found — this pod references a PVC that was never created. Note how this is a genuinely *different* root cause from Case A (missing StorageClass) even though both eventually show up as "something's stuck" — which is exactly why `kubectl describe`, not just `kubectl get`, is the step that actually distinguishes them.

```bash
kubectl delete -f debug-case-b.yaml
```

---

## The meta-lesson: `kubectl get` tells you THAT something's wrong; `kubectl describe` tells you WHY

Across this entire series — Module 06's incidents, and these two fresh cases — notice the pattern: **`kubectl get` almost never contains enough information to actually diagnose anything.** It shows `Pending`, `ContainerCreating`, `Bound` — states, not reasons. **`kubectl describe` (specifically its `Events` section) is where the actual diagnostic signal lives**, every single time, across every scenario in this series.

If there's one habit worth instilling: **whenever `kubectl get` shows anything other than the expected healthy state, the very next command should be `kubectl describe` on that exact object — not a guess, not re-reading the YAML, not restarting anything.**

---

## Cleanup

```bash
kubectl delete pvc typo-pvc --ignore-not-found
kubectl delete pod failedmount-pod --ignore-not-found
```

---

## Summary — the full checklist, one more time, as a reference card

| Symptom | First command | What you're looking for |
|---|---|---|
| PVC `Pending` | `kubectl describe pvc <name>` | No matching PV / StorageClass typo or missing / waiting for a consumer pod |
| Pod `ContainerCreating` | `kubectl describe pod <name>` | `FailedMount` (PVC/volume issue) vs `FailedAttachVolume` (zone/capacity issue) |
| "Disk full" from the app | `kubectl exec pod -- df -h` | Confirm it's actually the PVC mount, not `emptyDir` or the container's own layer |
| Still unclear | `kubectl describe node <node>` | Node-level capacity or attachment limits |
| Still unclear | Controller/CSI driver logs | Last resort — something's wrong in the provisioning pipeline itself |

**The one sentence to carry forward:** *"Every storage bug in this entire series — from Module 01's ephemeral filesystem through Module 06's incidents to these two fresh cases — was fully diagnosable from `kubectl describe` output alone. That's not a coincidence; it's the reason `describe` exists, and it's the single highest-leverage habit this whole series can leave you with."*

---

*End of Module 08. Module 09: Key Takeaways & Decision Framework — the mental models that tie the whole series together.*
