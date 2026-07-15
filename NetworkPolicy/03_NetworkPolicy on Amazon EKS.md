# Module 2 Lab: NetworkPolicy on Amazon EKS — Step by Step

---

## What is a NetworkPolicy?

A `NetworkPolicy` is a Kubernetes object that controls **which pods are allowed to talk to which other pods** — like a firewall, but instead of matching on IP addresses, it matches on **pod labels** (e.g., `tier: frontend`, `tier: api`).

By default, Kubernetes has **no restrictions at all** — every pod can talk to every other pod, across every namespace. A `NetworkPolicy` is how you turn that off and explicitly say "only these specific pods may talk to these other specific pods, on these specific ports."

## What's the use case?

**Blast radius control.** If an attacker compromises one low-privilege pod (say, your public-facing frontend), a `NetworkPolicy` is what stops them from using that foothold to walk straight into your database and steal data. Without one, a single compromised pod can reach *everything*. With one, a compromised frontend pod can only reach exactly what you explicitly allowed it to reach — nothing else, even if the attacker knows the internal IP or DNS name of the database.

The concrete goal in this lab: enforce that **Frontend → API → Database** is the only path traffic can take, and prove that **Frontend → Database directly** is physically blocked, not just "not officially allowed."

---

## Step 1: Enable NetworkPolicy enforcement on EKS

Kubernetes `NetworkPolicy` objects only work if the underlying network plugin (CNI) actually enforces them. On EKS, the default VPC CNI supports this, but it's **off by default** — this is the one prerequisite step, or nothing below will actually block anything.

```bash
export CLUSTER_NAME=scaling-demo-cluster
export AWS_REGION=ap-south-1

aws eks update-addon \
  --cluster-name $CLUSTER_NAME \
  --region $AWS_REGION \
  --addon-name vpc-cni \
  --configuration-values '{"enableNetworkPolicy": "true"}' \
  --resolve-conflicts OVERWRITE
```

Wait ~1-2 minutes, then verify:

```bash
aws eks describe-addon --cluster-name $CLUSTER_NAME --region $AWS_REGION \
  --addon-name vpc-cni --query "addon.status" --output text
```

Should show `ACTIVE`. Confirm the network policy agent is running on your nodes:

```bash
kubectl get pods -n kube-system | grep aws-node
```

---

## Step 2: Create and label the namespace

```bash
kubectl create namespace production
kubectl label namespace production name=production
kubectl label namespace kube-system name=kube-system
```

> 📝 The `name=production` / `name=kube-system` labels are required — later policies use `namespaceSelector: matchLabels: name: <ns>` to identify namespaces, and Kubernetes doesn't add this label automatically.

---

## Step 3: Deploy the three-tier app (Frontend → API → Database)

```bash
cat <<'EOF' > three-tier-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: banking-app
      tier: frontend
  template:
    metadata:
      labels:
        app: banking-app
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: production
spec:
  selector:
    app: banking-app
    tier: frontend
  ports:
  - port: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: banking-app
      tier: api
  template:
    metadata:
      labels:
        app: banking-app
        tier: api
    spec:
      containers:
      - name: nodejs
        image: node:18-alpine
        command: ["node", "-e", "require('http').createServer((req,res)=>res.end('API OK')).listen(3000)"]
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: production
spec:
  selector:
    app: banking-app
    tier: api
  ports:
  - port: 3000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: banking-app
      tier: database
  template:
    metadata:
      labels:
        app: banking-app
        tier: database
    spec:
      containers:
      - name: postgres
        image: postgres:14
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          value: "demo-password-not-for-production"
---
apiVersion: v1
kind: Service
metadata:
  name: database-service
  namespace: production
spec:
  selector:
    app: banking-app
    tier: database
  ports:
  - port: 5432
EOF

kubectl apply -f three-tier-app.yaml
kubectl wait --for=condition=Available deployment/frontend deployment/api deployment/database -n production --timeout=120s
```

---

## Step 4: Prove the problem — before any policy exists

```bash
kubectl exec -it deployment/frontend -n production -- bash -c "timeout 5 bash -c 'echo > /dev/tcp/database-service/5432' && echo CONNECTED || echo FAILED"
```

**This will succeed right now** — frontend can talk directly to the database. This is the exact problem NetworkPolicy exists to fix. Keep this in mind for comparison after Step 6.

---

## Step 5: Apply default-deny (lock everything down first)

```bash
cat <<'EOF' > default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

kubectl apply -f default-deny.yaml
```

`podSelector: {}` = applies to every pod in `production`. No rules defined = allow nothing. This blocks **all** traffic, in every direction, for every pod.

### Confirm total lockdown:

```bash
kubectl exec -it deployment/frontend -n production -- wget -qO- --timeout=5 http://api-service:3000
```

Should now **fail** — even the legitimate frontend→API path is blocked, because we haven't explicitly allowed anything yet.

---

## Step 6: Apply the three policies — allow only the intended paths

**Frontend → API only, plus DNS:**

```bash
cat <<'EOF' > frontend-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - protocol: TCP
      port: 3000
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
EOF
```

**API → Database only, plus DNS:**

```bash
cat <<'EOF' > api-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 3000
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
EOF
```

**Database accepts only from API, plus DNS:**

```bash
cat <<'EOF' > database-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - protocol: TCP
      port: 5432
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
EOF

kubectl apply -f frontend-policy.yaml -f api-policy.yaml -f database-policy.yaml
```

---

## Step 7: Test every path — this is the payoff

**✅ Frontend → API (should WORK):**
```bash
kubectl exec -it deployment/frontend -n production -- wget -qO- --timeout=5 http://api-service:3000
```
Expected: `API OK`

**❌ Frontend → Database directly (should FAIL — this is the whole point):**
```bash
kubectl exec -it deployment/frontend -n production -- nc -zv -w 5 database-service 5432
```
Expected: timeout/refused. Compare this to Step 4, where the exact same command succeeded.

**✅ API → Database (should WORK):**
```bash
kubectl exec -it deployment/api -n production -- nc -zv -w 5 database-service 5432
```
Expected: success.

**✅ DNS still works:**
```bash
kubectl exec -it deployment/frontend -n production -- nslookup api-service
```
Expected: resolves successfully.

---

## Step 8: Cleanup

```bash
kubectl delete -f frontend-policy.yaml -f api-policy.yaml -f database-policy.yaml -f default-deny.yaml --ignore-not-found
kubectl delete -f three-tier-app.yaml --ignore-not-found
kubectl delete namespace production --ignore-not-found
```

### Verify:

```bash
kubectl get namespace production 2>&1
```

Should return `NotFound`.

---

## Summary

| Test | Before policies | After policies |
|---|---|---|
| Frontend → API | ✅ Works (default allow) | ✅ Works (explicitly allowed) |
| Frontend → Database | ✅ Works (the security hole) | ❌ Blocked (the fix) |
| API → Database | ✅ Works | ✅ Works (explicitly allowed) |

That's the entire lesson: **default Kubernetes allows everything; NetworkPolicy makes you explicitly define the only paths that should exist, and blocks everything else — including the exact lateral-movement path an attacker would use.**
