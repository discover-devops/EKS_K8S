# 04 — The PersistentVolume Abstraction

**Series:** Kubernetes Storage: From Ephemeral Containers to Persistent Data
**Format:** Concept + hands-on lab on your EKS cluster

---

## Recap from Module 03

We've now established two things that don't work for anything we actually care about:
- `emptyDir` dies with the pod
- `hostPath` dies the moment a pod lands on a different node

We need storage that survives pod restarts **and** can follow a pod across nodes. **How might you design a system to solve this? What components would you actually need?**

Sit with that for a moment before reading on — Kubernetes' real answer involves two separate resources working together, and the *reason* they're separate is itself an important design lesson.

---

## The separation of concerns

Kubernetes solves this with two resources: **PersistentVolume (PV)** and **PersistentVolumeClaim (PVC)**. Think of the roles like this:

- **Cluster administrators provision storage** — physical volumes, cloud disks, whatever the underlying infrastructure actually is
- **Developers request storage** — "I need 10GB for my database," with no need to know or care where it physically comes from
- **Kubernetes binds requests to available storage** — matching one to the other automatically

### The analogy

The cluster administrator is the property manager who buys and builds secure, external storage lockers (**PersistentVolumes**). The developer is the tenant who fills out a request form (**PersistentVolumeClaim**) saying "I need a 10GB locker." Kubernetes itself is the concierge who matches the tenant's request against an available locker and hands over the key.

**Why do you think Kubernetes deliberately separates these two concerns, rather than having developers directly provision their own storage?**

The benefit: **developers don't need to know infrastructure details** — they never need to know what an EBS volume ID looks like, which availability zone anything is in, or how IAM permissions for storage work. **Admins retain control over the actual resources** — what storage backends are allowed, how big volumes can get, what the reclaim behavior is. This is the same separation-of-concerns principle you've already seen in this course (recall: developers write Ingress objects without knowing which controller implements them; developers write NetworkPolicy objects without knowing which CNI enforces them).

---

## PersistentVolume (PV) — the storage resource

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-database
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
```

>  We're deliberately using a `hostPath`-backed PV for this module — it needs zero AWS setup and lets us focus purely on understanding the **PV/PVC binding mechanism itself**. Module 05 introduces the real, cloud-backed version (`ebs.csi.aws.com`), once this binding logic is second nature.

### Field by field

**`capacity.storage`** — how much space this PV represents.

**`accessModes`** — how this volume is allowed to be mounted:

| Mode | Meaning |
|---|---|
| `ReadWriteOnce` (RWO) |  **Common misunderstanding: this does NOT mean "one Pod."** It means **one node**. In many environments, multiple Pods scheduled onto the *same* node can all read/write an RWO volume simultaneously — but a Pod on a *different* node cannot touch it at all. This distinction genuinely matters when troubleshooting scheduling issues, since "why can't my second replica mount this volume" often traces back to this exact misunderstanding. |
| `ReadOnlyMany` (ROX) | Multiple nodes, read-only |
| `ReadWriteMany` (RWX) | Multiple nodes, read-write |

**Why do you think most storage systems only support `ReadWriteOnce`?** Because distributed write locking is genuinely hard — coordinating multiple machines writing to the same block storage simultaneously without corrupting data requires a level of distributed consensus that most storage backends (like a plain EBS volume) simply don't implement. `ReadWriteMany` requires a storage backend specifically designed for concurrent access (like NFS or EFS) — not something an ordinary block device can do safely.

**`persistentVolumeReclaimPolicy`** — what happens when the claim using this PV is deleted:

| Policy | Behavior |
|---|---|
| `Retain` | Data is kept, requiring manual cleanup |
| `Delete` | The underlying storage itself is removed |
| `Recycle` | A basic scrub — **deprecated, don't use** |

---

## PersistentVolumeClaim (PVC) — the storage request

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual
```

This is the developer's view — plainly saying "I need 5GB of `ReadWriteOnce` storage." **How does Kubernetes decide which PV to hand this PVC?** It matches on: sufficient capacity, compatible access mode, and matching `storageClassName` — all three have to line up for a bind to happen.

---

## 🔧 Hands-on lab

### Step 1: Create the PV (the admin's task)

```bash
cat <<'EOF' > pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-database
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
    type: DirectoryOrCreate
EOF

kubectl apply -f pv.yaml
kubectl get pv
```

Expected `STATUS`: `Available` — the locker exists, nobody's claimed it yet.

### Step 2: Create the PVC (the developer's task)

```bash
cat <<'EOF' > pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual
EOF

kubectl apply -f pvc.yaml
kubectl get pvc
```

Expected: `database-pvc` shows `STATUS: Bound`, `VOLUME: pv-database` — even though the PVC only asked for `5Gi` and the PV offers `10Gi`. **A PVC binds to a PV that meets or exceeds its request — it doesn't need to match exactly.**

### Step 3: Confirm the PV's own status changed too

```bash
kubectl get pv
```

Notice `STATUS` is now `Bound`, and `CLAIM` shows `default/database-pvc`. **What do you predict happens if you now create a second PVC, also requesting 5Gi with `storageClassName: manual`?**

```bash
cat <<'EOF' > pvc-second.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc-second
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual
EOF

kubectl apply -f pvc-second.yaml
kubectl get pvc database-pvc-second
```

Expected: `STATUS: Pending`, and it stays that way. **There's only one PV available, and it's already bound** — Kubernetes doesn't split, share, or oversubscribe a single PV across multiple PVCs. Clean up this second PVC before continuing:

```bash
kubectl delete pvc database-pvc-second
```

### Step 4: Use the PVC in a real Postgres pod

```bash
cat <<'EOF' > database-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: database-pod
spec:
  containers:
  - name: postgres
    image: postgres:14
    env:
    - name: POSTGRES_PASSWORD
      value: demo-password
    volumeMounts:
    - name: postgres-storage
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: postgres-storage
    persistentVolumeClaim:
      claimName: database-pvc
EOF

kubectl apply -f database-pod.yaml
kubectl wait --for=condition=Ready pod/database-pod --timeout=60s
```

### Step 5: Write real data, then prove it survives pod deletion

**What experiment would prove data survives a pod restart?** Write data, delete the pod, recreate it, check the data is still there — exactly the pattern from every module so far, now with a real database.

```bash
kubectl exec database-pod -- psql -U postgres -c "CREATE TABLE users (id INT, name TEXT);"
kubectl exec database-pod -- psql -U postgres -c "INSERT INTO users VALUES (1, 'Alice');"
kubectl exec database-pod -- psql -U postgres -c "SELECT * FROM users;"
```

Confirm you see Alice's row. Now the real test:

```bash
kubectl delete pod database-pod
kubectl apply -f database-pod.yaml
kubectl wait --for=condition=Ready pod/database-pod --timeout=60s
kubectl exec database-pod -- psql -U postgres -c "SELECT * FROM users;"
```

**Expected: Alice's row is still there.** This is the direct, hands-on resolution to Module 00's opening incident — the exact same "delete pod, recreate pod" sequence that destroyed 14 hours of production data, except this time the data survives, because it was never inside the pod's own filesystem to begin with.

---

## Cleanup

```bash
kubectl delete pod database-pod --ignore-not-found
kubectl delete pvc database-pvc --ignore-not-found
kubectl delete pv pv-database --ignore-not-found
```

---

## Summary

| Object | Role | Analogy |
|---|---|---|
| `PersistentVolume` | The actual storage resource, created by an admin | The locker |
| `PersistentVolumeClaim` | A request for storage, created by a developer | The reservation form |
| Binding | Kubernetes automatically matching claim to volume | The concierge handing over the key |

**The one sentence to carry forward:** *"PV/PVC finally gives us storage whose lifecycle is independent of any pod AND any single node — but notice everything in this module was still created manually by hand, by us, acting as the admin. Real production teams don't hand-create a PV for every single request — that's exactly the gap StorageClasses and dynamic provisioning close."*

---

*End of Module 04. Module 05: StorageClasses and Dynamic Provisioning — automating this at scale, with real AWS EBS volumes.*
