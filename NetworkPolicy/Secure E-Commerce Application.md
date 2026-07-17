# Module 3: Comprehensive Hands-On Lab — Secure E-Commerce Application

**Combines everything from Modules 1 & 2: HPA + NetworkPolicy, on one realistic 5-tier app.**

**Architecture:**
```
Web (nginx) → API Gateway (Node.js) → Product Service (Python)  →┐
                                     → Order Service (Go)        →┴→ Database (PostgreSQL)
```

---

## Prerequisites

```bash
export CLUSTER_NAME=scaling-demo-cluster
export AWS_REGION=ap-south-1
```

Confirm Metrics Server (needed for HPA) and NetworkPolicy enforcement (needed for the policies) are both active on your cluster:

```bash
kubectl get deployment metrics-server -n kube-system
aws eks describe-addon --cluster-name $CLUSTER_NAME --region $AWS_REGION --addon-name vpc-cni --query "addon.configurationValues" --output text
```

If NetworkPolicy enforcement isn't on (from Module 2):
```bash
aws eks update-addon --cluster-name $CLUSTER_NAME --region $AWS_REGION --addon-name vpc-cni \
  --configuration-values '{"enableNetworkPolicy": "true"}' --resolve-conflicts OVERWRITE
```

---

## Step 1: Deploy the full 5-tier application

```bash
cat <<'EOF' > ecommerce-app.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce
  labels:
    name: ecommerce
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: ecommerce
spec:
  replicas: 2
  selector:
    matchLabels: {app: ecommerce, component: web}
  template:
    metadata:
      labels: {app: ecommerce, component: web}
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports: [{containerPort: 80}]
        resources:
          requests: {cpu: 50m, memory: 64Mi}
          limits: {cpu: 100m, memory: 128Mi}
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: ecommerce
spec:
  selector: {app: ecommerce, component: web}
  ports: [{port: 80}]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: ecommerce
spec:
  replicas: 2
  selector:
    matchLabels: {app: ecommerce, component: api-gateway}
  template:
    metadata:
      labels: {app: ecommerce, component: api-gateway}
    spec:
      containers:
      - name: nodejs
        image: node:18-alpine
        command: ["node", "-e", "require('http').createServer((req,res)=>res.end('gateway OK')).listen(4000)"]
        ports: [{containerPort: 4000}]
        resources:
          requests: {cpu: 100m, memory: 128Mi}
          limits: {cpu: 200m, memory: 256Mi}
---
apiVersion: v1
kind: Service
metadata:
  name: api-gateway-service
  namespace: ecommerce
spec:
  selector: {app: ecommerce, component: api-gateway}
  ports: [{port: 4000}]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  namespace: ecommerce
spec:
  replicas: 2
  selector:
    matchLabels: {app: ecommerce, component: product-service}
  template:
    metadata:
      labels: {app: ecommerce, component: product-service}
    spec:
      containers:
      - name: python
        image: python:3.11-alpine
        command: ["python3", "-m", "http.server", "5000"]
        ports: [{containerPort: 5000}]
        resources:
          requests: {cpu: 100m, memory: 128Mi}
          limits: {cpu: 200m, memory: 256Mi}
---
apiVersion: v1
kind: Service
metadata:
  name: product-service
  namespace: ecommerce
spec:
  selector: {app: ecommerce, component: product-service}
  ports: [{port: 5000}]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: ecommerce
spec:
  replicas: 2
  selector:
    matchLabels: {app: ecommerce, component: order-service}
  template:
    metadata:
      labels: {app: ecommerce, component: order-service}
    spec:
      containers:
      - name: golang
        image: golang:1.22-alpine
        command: ["sh", "-c", "while true; do echo -e 'HTTP/1.1 200 OK\\n\\norder OK' | nc -l -p 6000; done"]
        ports: [{containerPort: 6000}]
        resources:
          requests: {cpu: 100m, memory: 128Mi}
          limits: {cpu: 200m, memory: 256Mi}
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: ecommerce
spec:
  selector: {app: ecommerce, component: order-service}
  ports: [{port: 6000}]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: ecommerce
spec:
  replicas: 1
  selector:
    matchLabels: {app: ecommerce, component: database}
  template:
    metadata:
      labels: {app: ecommerce, component: database}
    spec:
      containers:
      - name: postgres
        image: postgres:14
        ports: [{containerPort: 5432}]
        env: [{name: POSTGRES_PASSWORD, value: "demo-password"}]
        resources:
          requests: {cpu: 100m, memory: 256Mi}
          limits: {cpu: 300m, memory: 512Mi}
---
apiVersion: v1
kind: Service
metadata:
  name: database-service
  namespace: ecommerce
spec:
  selector: {app: ecommerce, component: database}
  ports: [{port: 5432}]
EOF

kubectl apply -f ecommerce-app.yaml
kubectl wait --for=condition=Available deployment --all -n ecommerce --timeout=180s
```

> 📝 Order Service's Go container uses `nc` (BusyBox Alpine images include it) as a minimal stand-in HTTP responder — no app code needed for this lab.

### Verify:
```bash
kubectl get pods -n ecommerce
kubectl get svc -n ecommerce
```

---

## Step 2: Configure Autoscaling (HPA on Web + API Gateway)

```bash
cat <<'EOF' > ecommerce-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
  namespace: ecommerce
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target: {type: Utilization, averageUtilization: 60}
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway-hpa
  namespace: ecommerce
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  minReplicas: 2
  maxReplicas: 8
  metrics:
  - type: Resource
    resource:
      name: cpu
      target: {type: Utilization, averageUtilization: 60}
EOF

kubectl apply -f ecommerce-hpa.yaml
kubectl get hpa -n ecommerce
```

**Expected:** `TARGETS` shows `<something>%/60%` within ~30s (not `<unknown>` — every Deployment above has `resources.requests` set, which is why this works).

---

## Step 3: Apply Security Policies (Enforce the Architecture)

**Intended traffic map:**
```
Web → API Gateway
API Gateway → Product Service, Order Service
Product Service → Database
Order Service → Database
Everything → DNS (kube-system)
```

```bash
cat <<'EOF' > ecommerce-netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: ecommerce
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-policy
  namespace: ecommerce
spec:
  podSelector:
    matchLabels: {component: web}
  policyTypes: [Ingress, Egress]
  ingress:
  - from: [{podSelector: {}}]
    ports: [{protocol: TCP, port: 80}]
  egress:
  - to: [{podSelector: {matchLabels: {component: api-gateway}}}]
    ports: [{protocol: TCP, port: 4000}]
  - to: [{namespaceSelector: {matchLabels: {name: kube-system}}}]
    ports: [{protocol: UDP, port: 53}]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-gateway-policy
  namespace: ecommerce
spec:
  podSelector:
    matchLabels: {component: api-gateway}
  policyTypes: [Ingress, Egress]
  ingress:
  - from: [{podSelector: {matchLabels: {component: web}}}]
    ports: [{protocol: TCP, port: 4000}]
  egress:
  - to: [{podSelector: {matchLabels: {component: product-service}}}]
    ports: [{protocol: TCP, port: 5000}]
  - to: [{podSelector: {matchLabels: {component: order-service}}}]
    ports: [{protocol: TCP, port: 6000}]
  - to: [{namespaceSelector: {matchLabels: {name: kube-system}}}]
    ports: [{protocol: UDP, port: 53}]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: product-service-policy
  namespace: ecommerce
spec:
  podSelector:
    matchLabels: {component: product-service}
  policyTypes: [Ingress, Egress]
  ingress:
  - from: [{podSelector: {matchLabels: {component: api-gateway}}}]
    ports: [{protocol: TCP, port: 5000}]
  egress:
  - to: [{podSelector: {matchLabels: {component: database}}}]
    ports: [{protocol: TCP, port: 5432}]
  - to: [{namespaceSelector: {matchLabels: {name: kube-system}}}]
    ports: [{protocol: UDP, port: 53}]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-service-policy
  namespace: ecommerce
spec:
  podSelector:
    matchLabels: {component: order-service}
  policyTypes: [Ingress, Egress]
  ingress:
  - from: [{podSelector: {matchLabels: {component: api-gateway}}}]
    ports: [{protocol: TCP, port: 6000}]
  egress:
  - to: [{podSelector: {matchLabels: {component: database}}}]
    ports: [{protocol: TCP, port: 5432}]
  - to: [{namespaceSelector: {matchLabels: {name: kube-system}}}]
    ports: [{protocol: UDP, port: 53}]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: ecommerce
spec:
  podSelector:
    matchLabels: {component: database}
  policyTypes: [Ingress, Egress]
  ingress:
  - from: [{podSelector: {matchLabels: {component: product-service}}}]
    ports: [{protocol: TCP, port: 5432}]
  - from: [{podSelector: {matchLabels: {component: order-service}}}]
    ports: [{protocol: TCP, port: 5432}]
  egress:
  - to: [{namespaceSelector: {matchLabels: {name: kube-system}}}]
    ports: [{protocol: UDP, port: 53}]
EOF

kubectl label namespace kube-system name=kube-system --overwrite
kubectl apply -f ecommerce-netpol.yaml
kubectl get networkpolicy -n ecommerce
```

**Why five policies, not one:** each component only gets rules for *its own* traffic — this is the same additive-policy-per-tier pattern from Module 2's three-tier lab, just extended to five tiers. `default-deny` locks everything first; each subsequent policy reopens only its own intended paths.

---

## Step 4: Test Everything

### 4a. Verify HPA is reading metrics correctly:
```bash
kubectl get hpa -n ecommerce
```

### 4b. Test allowed path — Web → API Gateway (should WORK):
```bash
kubectl exec -it deployment/web -n ecommerce -- bash -c "timeout 5 bash -c 'echo > /dev/tcp/api-gateway-service/4000' && echo CONNECTED || echo FAILED"
```
Expected: `CONNECTED`

### 4c. Test blocked path — Web → Database directly (should FAIL, proves the architecture is enforced):
```bash
kubectl exec -it deployment/web -n ecommerce -- bash -c "timeout 5 bash -c 'echo > /dev/tcp/database-service/5432' && echo CONNECTED || echo FAILED"
```
Expected: `FAILED`

### 4d. Test blocked path — Web → Product Service directly, bypassing API Gateway (should FAIL):
```bash
kubectl exec -it deployment/web -n ecommerce -- bash -c "timeout 5 bash -c 'echo > /dev/tcp/product-service/5000' && echo CONNECTED || echo FAILED"
```
Expected: `FAILED`

### 4e. Test allowed path — API Gateway → Product Service and Order Service (should WORK):
```bash
kubectl exec -it deployment/api-gateway -n ecommerce -- sh -c "timeout 5 sh -c 'echo > /dev/tcp/product-service/5000' && echo CONNECTED || echo FAILED"
kubectl exec -it deployment/api-gateway -n ecommerce -- sh -c "timeout 5 sh -c 'echo > /dev/tcp/order-service/6000' && echo CONNECTED || echo FAILED"
```
Expected: `CONNECTED` for both

### 4f. Generate load and watch Web scale:

```bash
for i in 1 2 3; do
  kubectl run load-gen-$i -n ecommerce --image=busybox --restart=Never -- \
    /bin/sh -c "while true; do wget -q -O- http://web-service; done"
done
```

```bash
kubectl get hpa -n ecommerce --watch
```

In a second terminal:
```bash
kubectl get pods -n ecommerce -l component=web --watch
```

Expected: `TARGETS` climbs past 60%, `web` replicas scale up from 2 toward `maxReplicas: 10` within 1-2 minutes.

### 4g. Stop load, confirm scale-down eventually happens (5 min stabilization window):
```bash
for i in 1 2 3; do kubectl delete pod load-gen-$i -n ecommerce --ignore-not-found; done
```

---

## Step 5: Cleanup

```bash
kubectl delete namespace ecommerce
```

```bash
kubectl get namespace ecommerce 2>&1
```
Should return `NotFound`.

---

## Summary Table

| Test | Path | Expected |
|---|---|---|
| Web → API Gateway | Intended |  CONNECTED |
| API Gateway → Product Service | Intended |  CONNECTED |
| API Gateway → Order Service | Intended |  CONNECTED |
| Web → Database directly | Bypass attempt |  FAILED |
| Web → Product Service directly | Bypass attempt |  FAILED |
| Web replicas under load | HPA (2 → up to 10) | Scales up within ~1-2 min |
