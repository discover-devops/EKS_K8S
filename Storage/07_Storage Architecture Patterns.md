# 07 — Storage Architecture Patterns

**Series:** Kubernetes Storage: From Ephemeral Containers to Persistent Data
**Format:** Concept + hands-on lab on your EKS cluster

---

## Recap from Module 06

We've covered the full storage mechanism and how it breaks in production. This module steps back to architecture: **given a real production system, where does persistent storage actually belong?**

---

## The categorization exercise

You're building a production system with these components:

- Web frontend (stateless)
- API backend (stateless)
- Redis cache
- PostgreSQL database
- File upload service
- Logging aggregator

**Before reading on — which of these six genuinely need persistent storage, and which don't?**

Walking through each:
- **Web frontend, API backend** — stateless by design (recall the "kill it, does the next answer still make sense" test from earlier in this course). No persistent storage needed — any replica can serve any request.
- **Redis cache** — *usually* doesn't need persistence; the entire point of a cache is that losing it is recoverable (just rebuilds from the source of truth). Some setups do enable Redis persistence (RDB/AOF) for faster warm-starts after a restart, but it's an optimization, not a correctness requirement.
- **PostgreSQL database** — absolutely needs persistent storage. This is Module 00's entire opening incident.
- **File upload service** — needs persistent storage, and specifically needs to think carefully about **access mode** (more on this below).
- **Logging aggregator** — depends on architecture; if logs are shipped immediately to external storage (S3, Elasticsearch), the aggregator itself may not need much local persistence. If it buffers locally before shipping, it needs at least some durable buffer.

**The pattern worth naming:** *"Needs persistent storage" is not a yes/no property of a component type — it's a question about whether losing that specific data would be a correctness problem or just a performance hiccup.* A cache losing data is an inconvenience. A database losing data is Module 00.

---

## Pattern 1: StatefulSet with Persistent Storage — one PVC per replica

A single Postgres pod (Module 04) is fine for learning, but real production databases run as clustered, multi-replica systems. **Why can't we just use a Deployment with multiple replicas here, the same way we would for a stateless API?**

The problem: a Deployment's replicas are meant to be **interchangeable** — any replica can serve any request, and if you wrote a `volumeClaimTemplates`-style pattern onto a Deployment, every replica would try to mount the *same* PVC simultaneously, which either fails outright (RWO only allows one node) or silently corrupts data (if it somehow succeeded). Each database replica in a real cluster needs **its own separate, dedicated storage** — replica 1's data is not interchangeable with replica 2's data.

This is exactly what **StatefulSet** + **`volumeClaimTemplates`** solves.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-storage
      resources:
        requests:
          storage: 20Gi
```

**What's different about `volumeClaimTemplates` compared to a regular PVC reference?**

### The analogy

Instead of a group of tenants trying to share one locker, `volumeClaimTemplates` acts as a standing rule: *every time a new tenant moves in — a new replica is created — automatically write a fresh reservation ticket, so they get their own private locker, never shared with anyone else.*

**The mechanism:** `volumeClaimTemplates` isn't a single PVC — it's a *template* for generating one PVC **per replica**, automatically, named predictably (`data-postgres-0`, `data-postgres-1`, `data-postgres-2`). Each of those gets its own independently-provisioned PV. Replica 0 never sees replica 1's data, by construction.

### Hands-on lab

Reuse `fast-storage` from Module 05 (recreate it if you cleaned it up):

```bash
cat <<'EOF' > statefulset-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
EOF
kubectl apply -f statefulset-storageclass.yaml
```

```bash
cat <<'EOF' > postgres-statefulset.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        env:
        - name: POSTGRES_PASSWORD
          value: demo-password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-storage
      resources:
        requests:
          storage: 5Gi
EOF

kubectl apply -f postgres-statefulset.yaml
kubectl get pods -l app=postgres --watch
```

**Notice the pod creation order:** `postgres-0`, then `postgres-1`, then `postgres-2` — StatefulSets create replicas **sequentially, in order**, unlike a Deployment which creates all replicas in parallel. This ordering guarantee is part of what "Stateful" means here.

### Confirm each replica genuinely has its own separate PVC

```bash
kubectl get pvc -l app=postgres
```

Expected: three separate PVCs — `data-postgres-0`, `data-postgres-1`, `data-postgres-2` — each bound to its own independently-provisioned PV.

### Prove data isolation between replicas

```bash
kubectl exec postgres-0 -- psql -U postgres -c "CREATE TABLE replica_marker (id INT);"
kubectl exec postgres-0 -- psql -U postgres -c "INSERT INTO replica_marker VALUES (0);"
kubectl exec postgres-0 -- psql -U postgres -c "SELECT * FROM replica_marker;"
```

```bash
kubectl exec postgres-1 -- psql -U postgres -c "SELECT * FROM replica_marker;" 2>&1
```

**Expected:** an error — `relation "replica_marker" does not exist`. **This proves each replica's storage is genuinely separate**, not shared. (In a real production Postgres cluster, replicas would be wired together via streaming replication at the *application* layer — but the *storage* layer intentionally keeps them isolated, exactly as shown here; replication is Postgres's own job, not the StatefulSet's.)

### Prove identity is stable — delete `postgres-1` and watch it come back with the SAME name and SAME storage

```bash
kubectl delete pod postgres-1
kubectl get pods -l app=postgres --watch
```

**Expected:** a new pod is created, and it's named `postgres-1` again — not some random suffix like a Deployment would generate. It also reattaches to the exact same `data-postgres-1` PVC it had before. **Stable identity + stable storage per replica** is the entire point of StatefulSet.

---

## Pattern 2: Shared Storage for File Uploads — when you actually need RWX

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-uploads
spec:
  accessModes:
    - ReadWriteMany   # multiple pods can write simultaneously
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 100Gi
```

**When do you actually need `ReadWriteMany`?** Any time multiple pods, potentially on different nodes, need to write to the *same* files simultaneously — a classic example: a file upload service running several replicas behind a load balancer, where any replica might receive an upload, but all replicas need to be able to serve that same file back later, regardless of which replica originally received it.

**The trade-off:** recall from Module 04 — `ReadWriteOnce` is what most storage backends (including plain EBS) support, because distributed write-locking across nodes is genuinely hard. `ReadWriteMany` requires a storage backend specifically built for concurrent multi-node access — on AWS, that means **EFS**, not EBS. `storageClassName: nfs-storage` in the YAML above implies exactly this: a network-filesystem-backed StorageClass, not the block-storage-backed `fast-storage` we've used throughout this series.

>  **This series has used EBS (block storage, RWO-only) throughout.** Setting up real RWX storage on EKS requires installing the **EFS CSI driver** and provisioning an actual EFS filesystem — a meaningfully larger AWS setup (VPC mount targets, security groups) than anything in this module. That's deliberately being treated as its own dedicated future topic rather than folded in here — for now, understand the *concept and trade-off* (RWX needs a fundamentally different storage backend, not just a different access mode string), and know to reach for it specifically when multiple pods genuinely need simultaneous write access to the same files.

---

## Cleanup

```bash
kubectl delete statefulset postgres
kubectl delete pvc -l app=postgres
kubectl delete svc postgres
kubectl delete storageclass fast-storage
```

---

## Summary

| Pattern | Use when | Key mechanism |
|---|---|---|
| StatefulSet + `volumeClaimTemplates` | Each replica needs its own private, stable storage (databases, clustered stateful apps) | One auto-generated PVC per replica, stable naming, sequential creation |
| Shared RWX PVC | Multiple pods need simultaneous write access to the same files | Requires a network-filesystem backend (EFS on AWS), not plain block storage |

**The one sentence to carry forward:** *"Not every stateful workload needs the same storage pattern — a database wants isolated storage per replica precisely so replicas don't corrupt each other, while a file upload service wants the opposite: everyone sharing the same files. Picking the wrong pattern for the wrong workload is itself a production incident waiting to happen, just like the misconfigurations in Module 06."*

---

*End of Module 07. Module 08: Debugging Storage Issues — the systematic diagnostic checklist, reproduced live.*
