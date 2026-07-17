# Chapter 6: SSL/TLS Termination

**Series:** Kubernetes Traffic Management: From Services to Ingress
**Format:** Concept + full hands-on lab on your EKS cluster

---

## Recap from Chapter 5

We can now route by host and by path. This chapter adds the third major responsibility Chapter 1 flagged as a pain point of the old multi-LoadBalancer world: **encryption.**

---

## Where should SSL termination happen?

**At the Ingress Controller.** The controller decrypts incoming HTTPS traffic, then forwards plain, unencrypted HTTP to your backend services inside the cluster.

### The analogy

Terminating SSL at the Ingress Controller is like having one heavily-armed security guard checking IDs at the main lobby of the building, rather than forcing every single department on every single floor to hire and train its own personal security guard. Once a visitor is cleared at the main desk, they walk freely to whichever department they need — the checking happens once, centrally, not redundantly at every door.

---

## Reading the reference diagram

The diagram for this chapter shows the flow as an onion, being peeled:

```
CLIENT → [encrypted packet, layers of TCP/IP/TLS/HTTP wrapped together]
       → CLOUD LOAD BALANCER (still encrypted — just forwards)
       → INGRESS POD — decryption happens HERE, the "unwrapping"
       → [unencrypted HTTP data, from here onward]
       → Service X → backend App Pods (1, 2, 3)
```

**The one line worth saying out loud in class:** *"Everything left of the Ingress Pod is encrypted. Everything right of it — inside the cluster — is plain, unencrypted HTTP."* The diagram calls this the boundary between the **Secure Zone** (external) and the **Unsecure Internal Zone** (inside the cluster) — and that labeling is deliberate, not decorative. It's flagging a real tradeoff, which we'll come back to at the end of this chapter.

---

## Creating a TLS Secret

Kubernetes stores your certificate and private key as a `Secret` object:

```bash
kubectl create secret tls api-tls-secret \
  --cert=api.crt \
  --key=api.key
```

## Wiring TLS into the Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
spec:
  tls:
  - hosts:
    - api.company.com
    - admin.company.com
    secretName: api-tls-secret
  rules:
  - host: api.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

**What happens to plain HTTP requests now — do they get rejected?**

**No, not by default.** Both HTTP (port 80) and HTTPS (port 443) keep working side by side unless you explicitly force a redirect:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
```

## Why terminate SSL centrally, rather than at each backend?

- **Centralized certificate management** — one place to install, renew, and rotate certs, instead of N separate copies scattered across every backend service
- **Reduced backend load** — TLS encryption/decryption is genuine CPU work; your application pods get to spend their CPU on actual business logic instead of cryptographic overhead
- **Easier rotation** — update one Secret, and every route covered by that Ingress picks up the new certificate automatically, with zero backend redeploys

---

## 🔧 Hands-on lab

### Step 1: Generate a self-signed certificate (since we don't own a real domain)

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout api.key -out api.crt \
  -subj "/CN=api.company.com/O=api.company.com"
```

`-x509` produces a self-signed cert directly (skipping the usual CSR step, fine for lab purposes), `-days 365` sets validity, and `-subj` bakes `api.company.com` into the certificate's Common Name — this matters for Step 4.

### Step 2: Create the Kubernetes Secret

```bash
kubectl create secret tls api-tls-secret --cert=api.crt --key=api.key
kubectl get secret api-tls-secret
```

### Step 3: Reuse Chapter 4's backend, add a TLS-enabled Ingress

```bash
cat <<'EOF' > tls-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.company.com
    secretName: api-tls-secret
  rules:
  - host: api.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
EOF

kubectl apply -f tls-ingress.yaml
```

> 📝 If `api-service` from Chapter 4 was already cleaned up, redeploy just that one Deployment/Service pair from Chapter 4's Step 1 before continuing.

### Step 4: Test HTTPS — the important nuance: SNI, not just the Host header

For plain HTTP, we routed by manually setting the `Host` header (Chapter 4). **HTTPS adds an earlier decision point: SNI (Server Name Indication)** — the hostname is sent *during the TLS handshake itself*, **before** any decryption happens, so the Ingress Controller knows which certificate to present before it can even read any HTTP headers. This means `-H "Host: ..."` alone isn't enough for HTTPS testing — we need `curl` to also present the correct SNI hostname, which means telling it to resolve `api.company.com` to our NLB directly:

```bash
export NLB_HOST=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

curl -k --resolve api.company.com:443:$(dig +short $NLB_HOST | head -1) https://api.company.com/
```

- **`--resolve api.company.com:443:<IP>`** — tells curl "when you go to connect to `api.company.com` on port 443, actually connect to this IP" — this is what makes both the SNI hostname *and* the HTTP Host header correctly say `api.company.com`, without needing real DNS
- **`-k`** — skip certificate validation, since our cert is self-signed and won't be trusted by curl's default CA bundle

Expected: `Response from API Service`, delivered over HTTPS.

### Step 5: Confirm the certificate itself is what we created

```bash
curl -kv --resolve api.company.com:443:$(dig +short $NLB_HOST | head -1) https://api.company.com/ 2>&1 | grep -A 2 "subject:"
```

Expected output should show `CN=api.company.com` — confirming NGINX presented *our* self-signed cert, matching Step 1.

### Step 6: Confirm plain HTTP still works (no forced redirect yet)

```bash
curl -H "Host: api.company.com" http://$NLB_HOST/
```

Expected: still works, same as Chapter 4 — HTTP and HTTPS coexisting, exactly as the concept section described.

### Step 7: Force HTTPS-only via the redirect annotation

```bash
kubectl annotate ingress secure-ingress nginx.ingress.kubernetes.io/ssl-redirect="true" --overwrite
```

```bash
curl -I -H "Host: api.company.com" http://$NLB_HOST/
```

Expected: `HTTP/1.1 308 Permanent Redirect` (or `301`, depending on NGINX version), with a `Location:` header pointing to the `https://` version of the same URL — HTTP requests are no longer served directly, only redirected.

---

## The tradeoff the diagram was flagging — worth discussing live

Recall the diagram's "Unsecure Internal Zone" label on everything past the Ingress Pod. **In a real production environment, "unencrypted inside the cluster" is a genuine, debated security tradeoff, not a solved problem.** For many teams, plain HTTP between the Ingress Controller and backend pods is an acceptable risk, since cluster-internal traffic is already isolated by NetworkPolicy (recall Module 2 of this course) and generally trusted. For stricter, zero-trust-style environments — particularly regulated industries — teams add a **service mesh** (like Istio or Linkerd) specifically to add mutual TLS (mTLS) *between* internal services too, so nothing inside the cluster is ever unencrypted either. That's a more advanced topic beyond this chapter's scope, but worth knowing the term for, since "isn't internal traffic unencrypted?" is a very reasonable question a security-conscious student will ask.

---

## Cleanup

```bash
kubectl delete -f tls-ingress.yaml
kubectl delete secret api-tls-secret
rm api.key api.crt
```

---

## Summary

| What we did | Why |
|---|---|
| Generated a self-signed cert with `openssl` | We don't own a real domain, but the mechanics are identical to a real CA-issued cert |
| Stored it as a `kubectl create secret tls` object | This is the standard, required format Ingress TLS configuration expects |
| Added a `tls:` block to the Ingress | Told the controller which hosts this cert applies to |
| Used `curl --resolve` instead of just `-H Host` | Because HTTPS routing starts at the SNI layer, during the TLS handshake, before any HTTP header is even readable |
| Verified the cert's `CN` in the handshake | Proved the Ingress Controller is presenting *our* cert, not some default |
| Enabled `ssl-redirect` | Forced all HTTP traffic to redirect to HTTPS, rather than silently allowing both |

**The one sentence to leave students with:** *"SSL termination centralizes decryption at one point — cheaper to operate, easier to rotate — but it also means everything past that point, inside your cluster, is running in plaintext by default. That's a deliberate tradeoff, not an oversight, and it's exactly what NetworkPolicy (from the previous module of this course) and, in stricter environments, a service mesh's mTLS are there to compensate for."*

---

*End of Chapter 6. Chapter 7: Real DevOps Scenarios.*
