# Chapter 5: Path-Based Routing

**Series:** Kubernetes Traffic Management: From Services to Ingress
**Format:** Concept + full hands-on lab on your EKS cluster

---

## Recap from Chapter 4

Host-based routing solved "one entry point, many domains." This chapter solves a related but different problem: **one domain, many services, split by URL path.**

---

## The scenario

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
spec:
  rules:
  - host: api.company.com
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
      - path: /products
        pathType: Prefix
        backend:
          service:
            name: product-service
            port:
              number: 80
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
```

**Question: what happens to a request for `api.company.com/users/123`?**

Walking through the Ingress Controller's decision process, step by step:
1. Sees `Host: api.company.com` ✓ — matches the one host rule
2. Looks at the path: `/users/123`
3. Matches against the `/users` prefix ✓ — `/users/123` **starts with** `/users`
4. Routes to `user-service`

## The trickier question

**What happens to `api.company.com/`?** It matches the host, but doesn't start with `/users`, `/products`, or `/orders` — it matches **none** of the three path rules.

**Answer: a `404 Not Found`, returned directly by the Ingress Controller itself** — the request never even reaches any of your backend pods, because NGINX rejected it before forwarding anywhere.

## Fixing the "nothing matched" case: `defaultBackend`

```yaml
spec:
  defaultBackend:
    service:
      name: default-service
      port:
        number: 80
```

This is a **catch-all** — any request that fails to match every single rule in the entire Ingress object gets sent here instead of returning a bare 404. Useful for a friendly "page not found" service, a redirect to your homepage, or simply a controlled error response instead of NGINX's generic default page.

---

## PathType — three options, and they behave very differently

```yaml
# Prefix — matches the beginning
- path: /api
  pathType: Prefix
  # Matches: /api, /api/, /api/users, /api/anything

# Exact — must match exactly, character for character
- path: /api
  pathType: Exact
  # Matches ONLY: /api
  # Does NOT match: /api/, /api/users

# ImplementationSpecific — behavior depends entirely on which controller you're running
- path: /api/*
  pathType: ImplementationSpecific
  # Behavior varies by controller — avoid unless you have a specific reason
```

**Real scenario:** you want `/api/v1/*` routed to a v1 service, and `/api/v2/*` routed to a v2 service.

**Which pathType?** — `Prefix`. `Exact` would only ever match the literal string `/api/v1`, rejecting every real request like `/api/v1/orders` or `/api/v1/users/42`, since those aren't character-for-character identical to `/api/v1`. `Prefix` is the correct choice any time you want to match "this path and everything nested under it," which is the overwhelmingly common real-world case for API versioning, service splitting, and most path-based routing in general.

---

## 🔧 Hands-on lab

### Step 1: Deploy three backend services, and a default backend

```bash
cat <<'EOF' > path-routing-apps.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-app
spec:
  replicas: 1
  selector:
    matchLabels: {app: user-app}
  template:
    metadata:
      labels: {app: user-app}
    spec:
      containers:
      - name: http-echo
        image: hashicorp/http-echo
        args: ["-text=Response from User Service"]
        ports: [{containerPort: 5678}]
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector: {app: user-app}
  ports: [{port: 80, targetPort: 5678}]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-app
spec:
  replicas: 1
  selector:
    matchLabels: {app: product-app}
  template:
    metadata:
      labels: {app: product-app}
    spec:
      containers:
      - name: http-echo
        image: hashicorp/http-echo
        args: ["-text=Response from Product Service"]
        ports: [{containerPort: 5678}]
---
apiVersion: v1
kind: Service
metadata:
  name: product-service
spec:
  selector: {app: product-app}
  ports: [{port: 80, targetPort: 5678}]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-app
spec:
  replicas: 1
  selector:
    matchLabels: {app: order-app}
  template:
    metadata:
      labels: {app: order-app}
    spec:
      containers:
      - name: http-echo
        image: hashicorp/http-echo
        args: ["-text=Response from Order Service"]
        ports: [{containerPort: 5678}]
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector: {app: order-app}
  ports: [{port: 80, targetPort: 5678}]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: default-app
spec:
  replicas: 1
  selector:
    matchLabels: {app: default-app}
  template:
    metadata:
      labels: {app: default-app}
    spec:
      containers:
      - name: http-echo
        image: hashicorp/http-echo
        args: ["-text=404 - Nothing matched this path"]
        ports: [{containerPort: 5678}]
---
apiVersion: v1
kind: Service
metadata:
  name: default-service
spec:
  selector: {app: default-app}
  ports: [{port: 80, targetPort: 5678}]
EOF

kubectl apply -f path-routing-apps.yaml
kubectl wait --for=condition=Available deployment --all --timeout=60s
```

### Step 2: Create the path-based Ingress, with a default backend

```bash
cat <<'EOF' > path-based-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: default-service
      port:
        number: 80
  rules:
  - host: api.company.com
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
      - path: /products
        pathType: Prefix
        backend:
          service:
            name: product-service
            port:
              number: 80
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
EOF

kubectl apply -f path-based-ingress.yaml
kubectl get ingress path-based-ingress
```

### Step 3: Get your NLB address (reuse from Chapter 4, or regrab it)

```bash
export NLB_HOST=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo $NLB_HOST
```

### Step 4: Test each path — should route correctly

```bash
curl -H "Host: api.company.com" http://$NLB_HOST/users
```
Expected: `Response from User Service`

```bash
curl -H "Host: api.company.com" http://$NLB_HOST/users/123
```
Expected: **also** `Response from User Service` — this directly proves `Prefix` matching. `/users/123` isn't identical to `/users`, but it *starts with* `/users`, which is exactly what `Prefix` means.

```bash
curl -H "Host: api.company.com" http://$NLB_HOST/products
```
Expected: `Response from Product Service`

```bash
curl -H "Host: api.company.com" http://$NLB_HOST/orders
```
Expected: `Response from Order Service`

### Step 5: Test the unmatched path — proves `defaultBackend`

```bash
curl -H "Host: api.company.com" http://$NLB_HOST/
```
Expected: `404 - Nothing matched this path` — served by our `default-service`, **not** a raw NGINX 404 page. This proves `defaultBackend` is actually intercepting the "nothing matched" case, rather than letting it fall through to NGINX's generic error page.

### Step 6: Prove `Prefix` vs `Exact` directly — a focused side-by-side test

Let's add one more rule using `Exact`, so we can see the difference live instead of just reading about it.

```bash
cat <<'EOF' > pathtype-demo-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pathtype-demo-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: pathtype-demo.company.com
    http:
      paths:
      - path: /exact-test
        pathType: Exact
        backend:
          service:
            name: user-service
            port:
              number: 80
EOF

kubectl apply -f pathtype-demo-ingress.yaml
```

```bash
curl -o /dev/null -s -w "%{http_code}\n" -H "Host: pathtype-demo.company.com" http://$NLB_HOST/exact-test
```
Expected: `200` — exact match.

```bash
curl -o /dev/null -s -w "%{http_code}\n" -H "Host: pathtype-demo.company.com" http://$NLB_HOST/exact-test/
```
Expected: `404` — the trailing slash makes this a **different string**, and `Exact` requires character-for-character identity. (`Prefix` would have matched this fine.)

```bash
curl -o /dev/null -s -w "%{http_code}\n" -H "Host: pathtype-demo.company.com" http://$NLB_HOST/exact-test/anything
```
Expected: `404` — same reason, `Exact` never matches anything beyond the literal string.

**This is the concrete, hands-on proof of the "API v1/v2" scenario question from the concept section** — if you'd used `Exact` there instead of `Prefix`, every real request like `/api/v1/orders` would 404, because `Exact` only ever matches the bare `/api/v1` string with nothing after it.

---

## Cleanup

```bash
kubectl delete -f pathtype-demo-ingress.yaml
kubectl delete -f path-based-ingress.yaml
kubectl delete -f path-routing-apps.yaml
```

---

## Summary

| What we tested | Result | What it proves |
|---|---|---|
| `/users` and `/users/123` | Both → User Service | `Prefix` matches the path and everything nested under it |
| `/products`, `/orders` | Correct services | Multiple path rules coexist cleanly under one host |
| `/` (unmatched) | Custom 404 from `default-service` | `defaultBackend` intercepts anything no rule catches |
| `/exact-test` with `Exact` | 200 | Exact match works for the literal string |
| `/exact-test/` and `/exact-test/anything` with `Exact` | 404 | `Exact` rejects anything not character-for-character identical |

**The one sentence to leave students with:** *"`Prefix` answers 'does this path start with what I specified' — `Exact` answers 'is this path, character for character, identical to what I specified.' Almost every real API versioning or service-splitting scenario needs `Prefix`; reach for `Exact` only when you deliberately want to reject anything nested underneath."*

---

*End of Chapter 5. Chapter 6: SSL/TLS Termination.*
