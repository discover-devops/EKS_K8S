# 10 — The Capstone Lab

**Series:** Kubernetes Storage: From Ephemeral Containers to Persistent Data
**Format:** One combined hands-on lab, exercising everything from Modules 00-09

---

## The scenario

You're deploying a small production system for an internal tool: a stateless web frontend, and a PostgreSQL database that must never lose data. This capstone deliberately applies the **Decision Framework from Module 09** to real infrastructure, end to end — every choice below is a direct, justified answer to one of those four questions.

---

## Step 0: Run the Decision Framework first, out loud, before writing any YAML

**Frontend:**
- Survive pod restart? No real state to lose — it's stateless.
- -> **No volume needed at all.**

**Database:**
- Survive pod restart? Yes — this is exactly Module 00's incident.
- Node-specific? No — must follow the pod anywhere.
- Multiple simultaneous writers? No — single-writer Postgres.
- Criticality? Extremely high.
- -> **StatefulSet + `volumeClaimTemplates`, `ReadWriteOnce`, `reclaimPolicy: Retain`.**

This is the entire design decision, made explicit, before a single command runs.

---

## Step 1: Prerequisites

```bash
export CLUSTER_NAME=scaling-demo-cluster
export AWS_REGION=ap-south-1

aws eks describe-addon --cluster-name $CLUSTER_NAME --region $AWS_REGION --addon-name aws-ebs-csi-driver --query "addon.status" --output text
```

Confirm `ACTIVE` (installed back in Module 05 — if not, revisit that module's install steps first).

---

## Step 2: Deploy the StorageClass — deliberately Retain this time

```bash
cat <<'EOF' > capstone-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: production-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
allowVolumeExpansion: true
EOF

kubectl apply -f capstone-storageclass.yaml
```

**Justify every field before moving on:** `WaitForFirstConsumer` (Module 06, avoid zone mismatch), `Retain` (Module 06, avoid accidental data loss), `allowVolumeExpansion: true` (Module 06, avoid the out-of-space incident).

---

## Step 3: Deploy the database as a StatefulSet

```bash
cat <<'EOF' > capstone-database.yaml
apiVersion: v1
kind: Service
metadata:
  name: capstone-db
spec:
  clusterIP: None
  selector:
    app: capstone-db
  ports:
  - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: capstone-db
spec:
  serviceName: capstone-db
  replicas: 1
  selector:
    matchLabels:
      app: capstone-db
  template:
    metadata:
      labels:
        app: capstone-db
    spec:
      containers:
      - name: postgres
        image: postgres:14
        env:
        - name: POSTGRES_PASSWORD
          value: capstone-demo-password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: production-storage
      resources:
        requests:
          storage: 5Gi
EOF

kubectl apply -f capstone-database.yaml
kubectl wait --for=condition=Ready pod/capstone-db-0 --timeout=90s
```

---

## Step 4: Deploy the stateless frontend — no volume at all

```bash
cat <<'EOF' > capstone-frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: capstone-frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: capstone-frontend
  template:
    metadata:
      labels:
        app: capstone-frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
EOF

kubectl apply -f capstone-frontend.yaml
kubectl wait --for=condition=Available deployment/capstone-frontend --timeout=60s
```

---

## Exercise 1: Prove Data Persistence (the capstone's own version)

**Your task, before looking below:** write real data into `capstone-db-0`, delete the pod, recreate the StatefulSet's pod, and verify the data survived.

### Solution — try it yourself first

```bash
kubectl exec capstone-db-0 -- psql -U postgres -c "CREATE TABLE audit_log (event TEXT, ts TIMESTAMP DEFAULT now());"
kubectl exec capstone-db-0 -- psql -U postgres -c "INSERT INTO audit_log (event) VALUES ('capstone lab started');"
kubectl exec capstone-db-0 -- psql -U postgres -c "SELECT * FROM audit_log;"

kubectl delete pod capstone-db-0
kubectl wait --for=condition=Ready pod/capstone-db-0 --timeout=90s

kubectl exec capstone-db-0 -- psql -U postgres -c "SELECT * FROM audit_log;"
```

Expected: the row survives. This is the direct, final proof that every design decision from Step 0 actually worked.

---

## Exercise 2: Diagnose This Failure

```
kubectl get pods
NAME             READY   STATUS              RESTARTS   AGE
app-pod          0/1     ContainerCreating   0          5m

kubectl describe pod app-pod | grep -A 5 Events
# You see: FailedMount: Volume not found
```

**What commands would you run to diagnose this? What are the possible causes? How would you fix each?**

### Solution — try it yourself first

Diagnostic sequence (straight from Module 08's checklist):
```bash
kubectl get pvc                          # does the referenced PVC even exist?
kubectl describe pvc <name>              # is it Bound, or stuck Pending?
kubectl get pv                           # does a real PV exist behind it?
```

Possible causes, and their fixes:
- **PVC name typo in the pod spec** -> fix the `claimName` field
- **PVC was never created** -> create it
- **PVC exists but is stuck `Pending`** -> this becomes a Module 08 Case A investigation (StorageClass typo, no matching PV, or waiting for a consumer)
- **PV was manually deleted while still referenced** -> recreate it, or point the PVC elsewhere

---

## Discussion Questions — final wrap-up

**Question 1:** You have a logging agent that needs to read log files from `/var/log` on each node. What volume type would you use? Why?

### Model answer

`hostPath` (Module 03). This is precisely the scenario where node-locality is a *feature*, not a bug — the agent's entire job is reading that specific node's own logs, and it runs as a DaemonSet (one pod per node), so there's no "wrong node" problem to worry about.

**Question 2:** Your company wants to cut costs. Someone suggests setting `reclaimPolicy: Delete` on all PVs, to automatically clean up storage. What are the risks? When is this appropriate?

### Model answer

This is Module 06, Scenario 1's exact root cause, generalized as policy. Appropriate for: genuinely temporary/dev/staging workloads, CI test databases, anything truly disposable. Inappropriate for: production databases, anything where a PVC deletion (accidental or intentional) should not equal permanent, irreversible data loss. The real fix isn't "never use `Delete`" — it's matching the policy to the data's actual criticality, per Module 09's Question 4.

**Question 3:** You need to run a database with 3 replicas, each needing its own storage. Would you use a Deployment or StatefulSet? Why?

### Model answer

StatefulSet (Module 07). A Deployment's replicas are interchangeable by design; a database's replicas are explicitly *not* interchangeable — each needs its own private, stable storage, with a stable identity that survives pod recreation. `volumeClaimTemplates` is the mechanism that makes "one PVC per replica, automatically" possible — a Deployment has no equivalent concept.

---

## Cleanup — tear down the entire capstone

```bash
kubectl delete deployment capstone-frontend
kubectl delete statefulset capstone-db
kubectl delete pvc -l app=capstone-db
kubectl delete svc capstone-db
kubectl delete storageclass production-storage
```

Since `production-storage` used `reclaimPolicy: Retain`, the underlying EBS volume will **not** be deleted automatically — find and remove it manually if you want a fully clean AWS account:

```bash
aws ec2 describe-volumes --region $AWS_REGION --filters "Name=tag:kubernetes.io/created-for/pvc/name,Values=data-capstone-db-0" --query "Volumes[].VolumeId" --output text
```

---

## Series wrap-up

You've now built, broken, and fixed every layer of Kubernetes storage, hands-on, on a real EKS cluster:

- **Module 00-01:** why containers can't hold state on their own
- **Module 02-03:** the two "almost, but not quite" answers — `emptyDir` and `hostPath`
- **Module 04-05:** the real answer — PV/PVC, manual then automated
- **Module 06:** three real incidents this exact mechanism can still produce if misconfigured
- **Module 07:** the architecture patterns real systems actually use
- **Module 08:** the systematic way to diagnose any of it when it breaks
- **Module 09:** the mental models that compress all of it into four questions
- **Module 10:** all of it, combined, in one system you built with your own hands

**The one sentence to close the entire series on:** *"Containers are ephemeral by design. Storage must be persistent by intention. Every module in this series was, in one way or another, about turning that intention into an explicit, correct configuration choice — and now you've made that choice correctly, end to end, on real infrastructure."*
