# Chapter 7: Real DevOps Scenarios

**Series:** Kubernetes Traffic Management: From Services to Ingress
**Format:** Three real incidents, reproduced live on your cluster — break it on purpose, diagnose it, fix it

---

## Why this chapter is different

Every chapter so far built something correctly. This one does the opposite on purpose: **we're going to deliberately misconfigure three real, commonly-seen production bugs, watch the exact symptom appear, diagnose it using the same commands a real on-call engineer would reach for, and then fix it** — because debugging skill is built by pattern-matching against symptoms you've actually seen, not by reading about them.

---

## Scenario 1: The 404 Mystery (pathType misconfiguration)

**What happened:** a developer deployed a new API at `api.company.com/v2` and wrote this rule:

```yaml
- path: /v2
  pathType: Exact   # ← the bug
  backend:
    service:
      name: api-v2-service
```

**Symptom reported:** `api.company.com/v2` works. `api.company.com/v2/users` returns 404.

**Diagnose before reading on:** why does the bare path work but nothing nested underneath it?

**Answer:** direct application of Chapter 5. `Exact` only ever matches the literal string `/v2`, character for character. `/v2/users` is a *different string* — it merely starts with `/v2`, which is precisely what `Prefix` (not `Exact`) is for.

### 🔧 Reproduce this live

```bash
cat <<'EOF' > scenario1-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: v2-app
spec:
  replicas: 1
  selector:
    matchLabels: {app: v2-app}
  template:
    metadata:
      labels: {app: v2-app}
    spec:
      containers:
      - name: http-echo
        image: hashicorp/http-echo
        args: ["-text=Response from API v2"]
        ports: [{containerPort: 5678}]
---
apiVersion: v1
kind: Service
metadata:
  name: api-v2-service
spec:
  selector: {app: v2-app}
  ports: [{port: 80, targetPort: 5678}]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: scenario1-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: api.company.com
    http:
      paths:
      - path: /v2
        pathType: Exact
        backend:
          service:
            name: api-v2-service
            port:
              number: 80
EOF

kubectl apply -f scenario1-app.yaml
kubectl wait --for=condition=Available deployment/v2-app --timeout=60s
```

```bash
export NLB_HOST=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

curl -o /dev/null -s -w "%{http_code}\n" -H "Host: api.company.com" http://$NLB_HOST/v2
curl -o /dev/null -s -w "%{http_code}\n" -H "Host: api.company.com" http://$NLB_HOST/v2/users
```

Expected: `200` then `404` — you've just reproduced the exact reported bug.

### The fix

```bash
kubectl patch ingress scenario1-ingress --type='json' \
  -p='[{"op": "replace", "path": "/spec/rules/0/http/paths/0/pathType", "value": "Prefix"}]'
```

```bash
curl -o /dev/null -s -w "%{http_code}\n" -H "Host: api.company.com" http://$NLB_HOST/v2/users
```

Expected: `200` now. Bug fixed, confirmed.

**Diagnostic pattern to remember:** *"Bare path works, anything nested underneath 404s"* is close to a signature for a `pathType: Exact` misconfiguration. Check `kubectl get ingress <name> -o yaml` for the `pathType` field before looking anywhere else.

---

## Scenario 2: SSL Certificate Mismatch (or is it?)

**What happened:** Ingress is configured for `api.company.com`. The certificate covers `*.company.com`. Users get an SSL warning anyway.

**First instinct, and why it's usually wrong:** *"the certificate must not actually cover that hostname."* Check it:

```bash
kubectl get secret api-tls-secret -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout
```
```
Subject: CN=*.company.com
```

**The certificate is fine.** A wildcard `*.company.com` cert legitimately covers `api.company.com` — that's exactly what a wildcard cert is for. So what's actually broken?

**Check the Ingress's own status instead of the certificate:**

```bash
kubectl describe ingress secure-ingress
```
```
Events:
  Warning  InvalidTLSSecret  5m  nginx-ingress-controller    TLS secret "api-tls-secret" does not contain a valid certificate
```

**The real bug: the Secret itself is malformed** — not the certificate's content, but how it's stored in Kubernetes. Common causes: `tls.crt` and `tls.key` swapped, one of the two keys missing entirely, or the PEM content corrupted during creation (e.g., someone manually base64-encoded an already-base64-encoded file).

### 🔧 Reproduce this live — deliberately create a broken Secret

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout scenario2.key -out scenario2.crt \
  -subj "/CN=*.company.com"

# Deliberately swap cert and key to reproduce the bug
kubectl create secret generic broken-tls-secret \
  --from-file=tls.crt=scenario2.key \
  --from-file=tls.key=scenario2.crt
```

```bash
cat <<'EOF' > scenario2-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: scenario2-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.company.com
    secretName: broken-tls-secret
  rules:
  - host: api.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-v2-service
            port:
              number: 80
EOF

kubectl apply -f scenario2-ingress.yaml
```

```bash
kubectl describe ingress scenario2-ingress | grep -A 3 Events
```

Expected: you should see the exact `InvalidTLSSecret` (or similar cert-parsing) warning event — reproduced live.

### How to verify Secret contents properly

```bash
kubectl get secret broken-tls-secret -o yaml
```

Look for both `tls.crt` and `tls.key` keys present. To actually confirm the *type* stored under each key:

```bash
kubectl get secret broken-tls-secret -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout 2>&1 | head -3
```

If this errors instead of printing certificate details, you've confirmed the object stored under `tls.crt` isn't actually a valid certificate — exactly our deliberately-swapped scenario.

### The fix — recreate the Secret correctly

```bash
kubectl delete secret broken-tls-secret
kubectl create secret tls fixed-tls-secret --cert=scenario2.crt --key=scenario2.key
kubectl patch ingress scenario2-ingress --type='json' \
  -p='[{"op": "replace", "path": "/spec/tls/0/secretName", "value": "fixed-tls-secret"}]'
```

```bash
kubectl describe ingress scenario2-ingress | grep -A 3 Events
```

Expected: no more `InvalidTLSSecret` warning.

**Diagnostic pattern to remember:** *"Certificate looks correct when inspected directly, but the app still shows an SSL warning"* → stop checking the certificate, and check `kubectl describe ingress` for Secret-related Events instead. The bug is almost always in how the Secret was **assembled**, not in the certificate's own content.

---

## Scenario 3: The Traffic Black Hole (Service selector mismatch)

**What happened:** Ingress rules look correct. SSL works. Requests return `503 Service Temporarily Unavailable`.

```bash
kubectl get ingress
```
```
NAME           HOSTS             ADDRESS          PORTS
app-ingress    api.company.com   34.123.45.67     80, 443
```

```bash
kubectl get svc api-service
```
```
NAME          TYPE        CLUSTER-IP      PORT(S)
api-service   ClusterIP   10.96.25.30     80/TCP
```

```bash
kubectl get pods -l app=api
```
```
NAME        READY   STATUS    RESTARTS
api-pod-1   1/1     Running   0
```

**Everything above looks healthy.** Ingress exists, Service exists, pod is `Running`. What's actually broken?

### The diagnostic step that finds it

```bash
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```
```
upstream connect error or disconnect/reset before headers. reset reason: connection failure
```

This tells us NGINX is trying to forward the request somewhere, and that destination is refusing the connection. **Check the Service's actual endpoints — not just that the Service object exists:**

```bash
kubectl get endpoints api-service
```
```
NAME          ENDPOINTS
api-service   <none>
```

**Found it.** The Service has **zero endpoints** — meaning it has never successfully matched a single pod, despite a pod being `Running`. **A `Service` object existing tells you nothing about whether it's actually wired to any real pod.**

### Root cause: selector/label mismatch

```yaml
# Service selector
spec:
  selector:
    app: api-backend    # ← what the Service is looking for

# Pod label
metadata:
  labels:
    app: api             # ← what the pod actually has
```

`api-backend` ≠ `api`. Kubernetes label matching is an exact string comparison — no fuzzy matching, no partial credit. The Service and the pod simply never found each other.

### 🔧 Reproduce this live

```bash
cat <<'EOF' > scenario3-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mismatched-app
spec:
  replicas: 1
  selector:
    matchLabels: {app: api}
  template:
    metadata:
      labels: {app: api}
    spec:
      containers:
      - name: http-echo
        image: hashicorp/http-echo
        args: ["-text=Response from mismatched app"]
        ports: [{containerPort: 5678}]
---
apiVersion: v1
kind: Service
metadata:
  name: mismatched-service
spec:
  selector:
    app: api-backend
  ports: [{port: 80, targetPort: 5678}]
EOF

kubectl apply -f scenario3-app.yaml
kubectl wait --for=condition=Available deployment/mismatched-app --timeout=60s
```

```bash
kubectl get endpoints mismatched-service
```

Expected: `<none>` — reproduced.

```bash
cat <<'EOF' > scenario3-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: scenario3-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: broken.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mismatched-service
            port:
              number: 80
EOF

kubectl apply -f scenario3-ingress.yaml
```

```bash
curl -o /dev/null -s -w "%{http_code}\n" -H "Host: broken.company.com" http://$NLB_HOST/
```

Expected: `503` — the exact reported symptom, reproduced end to end.

### The fix

```bash
kubectl patch service mismatched-service --type='json' \
  -p='[{"op": "replace", "path": "/spec/selector/app", "value": "api"}]'
```

```bash
kubectl get endpoints mismatched-service
```

Expected: now shows a real pod IP instead of `<none>`.

```bash
curl -o /dev/null -s -w "%{http_code}\n" -H "Host: broken.company.com" http://$NLB_HOST/
```

Expected: `200` now.

**Diagnostic pattern to remember:** *"Ingress correct, Service exists, pod is Running, but still 503"* → the missing check almost everyone skips is `kubectl get endpoints <service-name>`. A `Service` existing is not proof it's connected to anything. Endpoints being empty is the single fastest way to confirm a label mismatch, before wasting time re-reading YAML that looks fine at a glance.

---

## Cleanup

```bash
kubectl delete -f scenario1-app.yaml -f scenario2-ingress.yaml -f scenario3-app.yaml -f scenario3-ingress.yaml --ignore-not-found
kubectl delete secret broken-tls-secret fixed-tls-secret --ignore-not-found
rm -f scenario2.key scenario2.crt
```

---

## Summary — three diagnostic reflexes to build

| Symptom | Don't waste time here | Check this instead |
|---|---|---|
| Bare path works, nested paths 404 | Re-reading the Service/backend config | `pathType` field — `Exact` vs `Prefix` |
| Certificate content looks correct, SSL warning persists | Re-verifying the certificate's CN/SAN again | `kubectl describe ingress` → Events, for Secret-format errors |
| Ingress and Service both exist, pod is Running, still 503 | Re-reading the Ingress YAML | `kubectl get endpoints <service>` — empty means label mismatch |

**The one sentence to leave students with:** *"In every one of these three real incidents, every object involved looked completely fine when read in isolation — the bug was always in the relationship between two objects, not inside either object alone. `kubectl describe` and `kubectl get endpoints` exist specifically to surface that relationship, which a quiet `kubectl get` by itself will never show you."*

---

*End of Chapter 7. Chapter 8: Advanced Ingress Patterns.*
