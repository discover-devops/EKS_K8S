# 05 — StorageClasses and Dynamic Provisioning

**Series:** Kubernetes Storage: From Ephemeral Containers to Persistent Data
**Format:** Concept + hands-on lab on your EKS cluster, using real AWS EBS volumes

---

## Recap from Module 04

We proved PV/PVC binding works — but every PV in that module was hand-written by us, acting as the admin. **Imagine 100 developers, each needing their own storage. Manually writing a PV for every single request doesn't scale.** How might this be automated?

That automation is what **StorageClasses** and **dynamic provisioning** solve.

### The analogy

A StorageClass is like an automated valet service. Instead of the property manager building lockers by hand, in advance, and hoping there's one free when a tenant shows up — the tenant simply hands a ticket (the PVC) to the valet, and the valet immediately builds a brand-new locker on the spot, specifically sized for that one tenant, no pre-planning required.

---

## Prerequisite: the EBS CSI driver

Real dynamic provisioning on EKS needs the **EBS CSI driver** installed — this is the actual software that talks to the AWS EBS API on your behalf whenever a StorageClass needs to conjure up a real volume.

### Check if it's already installed

```bash
export CLUSTER_NAME=scaling-demo-cluster
export AWS_REGION=ap-south-1
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws eks describe-addon --cluster-name $CLUSTER_NAME --region $AWS_REGION --addon-name aws-ebs-csi-driver --query "addon.status" --output text 2>&1
```

If this returns `ACTIVE`, skip to the StorageClass section below. If it errors (`ResourceNotFoundException`), install it now:

```bash
aws eks describe-cluster --name $CLUSTER_NAME --region $AWS_REGION --query "cluster.identity.oidc.issuer" --output text
```

If this doesn't return a URL, associate OIDC first:
```bash
eksctl utils associate-iam-oidc-provider --cluster $CLUSTER_NAME --region $AWS_REGION --approve
```

Then create the IAM role and install the driver:

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster $CLUSTER_NAME \
  --region $AWS_REGION \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicyV2 \
  --approve

aws eks create-addon \
  --cluster-name $CLUSTER_NAME \
  --region $AWS_REGION \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/AmazonEKS_EBS_CSI_DriverRole
```

Wait for it to become `ACTIVE`, then confirm:

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
```

---

## The StorageClass — the provisioner

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iopsPerGB: "10"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
```

**What do you think `provisioner` actually does?** It tells Kubernetes exactly which storage backend to call when this StorageClass is used — in this case, `ebs.csi.aws.com`, the AWS EBS CSI driver we just confirmed is running.

### The CSI note, worth stating explicitly

Notice we're using `provisioner: ebs.csi.aws.com`, not an older name like `kubernetes.io/aws-ebs`. That older form is an **in-tree provisioner** — legacy code that used to ship built directly into core Kubernetes. Virtually everything has since migrated to the **Container Storage Interface (CSI)** — `ebs.csi.aws.com` is the modern standard. CSI's real benefit: **storage vendors can update their drivers independently, without waiting for a Kubernetes core release** — exactly the same "pluggable implementation behind a stable API" pattern you've already seen with Ingress Controllers and CNI plugins throughout this course.

Common modern provisioners, for reference:
- `ebs.csi.aws.com` — AWS EBS volumes
- `pd.csi.storage.gke.io` — Google Persistent Disks
- `disk.csi.azure.com` — Azure Disks

### `volumeBindingMode: WaitForFirstConsumer` — why wait?

**Why would Kubernetes deliberately delay creating the volume, instead of provisioning it the instant the PVC is created?** Because AWS EBS volumes are **zone-specific** — created in one specific Availability Zone, and only attachable to nodes in that same zone. If Kubernetes provisioned the volume immediately, before knowing which node the pod will actually be scheduled to, it risks creating the volume in the *wrong* AZ entirely — a mistake that isn't fixable after the fact; the pod would simply never be able to attach to it.

`WaitForFirstConsumer` means: **don't create the actual EBS volume until a real pod that needs it has been scheduled** — at that point, Kubernetes knows exactly which AZ the volume needs to live in, and creates it there, guaranteed to match.

---

## 🔧 Hands-on lab

### Step 1: Create the StorageClass

```bash
cat <<'EOF' > storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
EOF

kubectl apply -f storageclass.yaml
kubectl get storageclass fast-storage
```

### Step 2: Create a PVC referencing it — and watch what happens

```bash
cat <<'EOF' > dynamic-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-storage
  resources:
    requests:
      storage: 5Gi
EOF

kubectl apply -f dynamic-pvc.yaml
kubectl get pvc dynamic-pvc
```

**Expected `STATUS`: `Pending`.** Before continuing — why? Look back at the StorageClass we just created. **`WaitForFirstConsumer`** — no pod has claimed this PVC yet, so Kubernetes deliberately hasn't provisioned anything real yet.

### Step 3: Create a pod that actually uses this PVC

```bash
cat <<'EOF' > app-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
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
      claimName: dynamic-pvc
EOF

kubectl apply -f app-pod.yaml
```

### Step 4: Watch the PVC bind, live

```bash
kubectl get pvc dynamic-pvc --watch
```

Within roughly 30-60 seconds (real AWS API call + real EBS volume creation happening), `STATUS` should flip from `Pending` to `Bound`, with a `VOLUME` name that looks like `pvc-<random-uuid>` — not something we ever named ourselves.

```bash
kubectl get pv
```

**A PV was automatically created — we never wrote one.** This is dynamic provisioning, proven live.

### Step 5: What actually happened behind the scenes

```bash
kubectl describe pvc dynamic-pvc | grep -A 5 Events
```

Trace the sequence: pod scheduled → PVC's provisioning finally triggered (because `WaitForFirstConsumer` was satisfied) → the EBS CSI driver called the real AWS API → a real EBS volume was created in the same AZ as the node the pod landed on → the PV object was created to represent it → PVC bound to that new PV.

### Step 6: Confirm this is a real AWS resource

```bash
kubectl get pv $(kubectl get pvc dynamic-pvc -o jsonpath='{.spec.volumeName}') -o jsonpath='{.spec.csi.volumeHandle}'
```

This prints a real EBS volume ID (`vol-xxxxxxxxxxxx`). Confirm it independently via AWS:

```bash
aws ec2 describe-volumes --region $AWS_REGION --volume-ids $(kubectl get pv $(kubectl get pvc dynamic-pvc -o jsonpath='{.spec.volumeName}') -o jsonpath='{.spec.csi.volumeHandle}') --query "Volumes[0].{State:State,AZ:AvailabilityZone,Size:Size}" --output table
```

**This is a genuine EBS volume, sitting in your AWS account, that you never manually created.** One PVC, one pod — and Kubernetes handled every single infrastructure step in between.

---

## Cleanup

```bash
kubectl delete pod app-pod
kubectl delete pvc dynamic-pvc
kubectl delete storageclass fast-storage
```

> 📝 Since this StorageClass had `reclaimPolicy: Delete`, deleting the PVC above will also delete the real EBS volume automatically — confirm with `aws ec2 describe-volumes` if you want to see it disappear.

---

## Summary

| Module 04 (manual) | Module 05 (dynamic) |
|---|---|
| Admin hand-writes a PV | StorageClass auto-generates a PV on demand |
| Fixed capacity, fixed AZ, decided upfront | Sized and zoned exactly to match the actual PVC and pod |
| Doesn't scale past a handful of volumes | Scales to however many developers need storage, unattended |

**The one sentence to carry forward:** *"StorageClasses turn storage provisioning from a manual admin chore into a self-service request — but automation doesn't remove operational risk, it just relocates it. The next module walks through three real incidents that happen specifically because of how this automation gets misconfigured."*

---

*End of Module 05. Module 06: SRE Incident Scenarios — accidental deletion, zone mismatches, and out-of-space databases, reproduced live.*
