
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




Let's break this YAML down field by field — this is the one students need to understand deeply, since the other two policies (API, database) are just the same pattern repeated.

## The `cat <<'EOF' > frontend-policy.yaml ... EOF` part

This is just a **heredoc** — a bash trick to write multiple lines of text into a file without opening an editor. Everything between `<<'EOF'` and the closing `EOF` gets written, verbatim, into `frontend-policy.yaml`. Nothing Kubernetes-specific here — it's purely "create this file with this content." The actual Kubernetes object only gets created afterward, when you run `kubectl apply -f frontend-policy.yaml`.

## Now the actual policy content, section by section

```yaml
metadata:
  name: frontend-policy
  namespace: production
```
Standard Kubernetes object metadata — this policy is named `frontend-policy`, and it lives in the `production` namespace (meaning it can only reference/select things within `production`, unless you explicitly use a `namespaceSelector` to reach outside it).

```yaml
spec:
  podSelector:
    matchLabels:
      tier: frontend
```
**This is the most important line — it answers "WHO does this policy apply to?"** It says: *"Apply this policy to every pod in the `production` namespace that has the label `tier: frontend`."* Recall from your Deployment YAML — the frontend pods carry exactly that label. So this policy is now specifically attached to your 2 frontend pods, and nothing else.

```yaml
  policyTypes:
  - Ingress
  - Egress
```
This declares that this policy is going to control **both** directions of traffic for the frontend pods — traffic **coming into** them (Ingress) and traffic **going out from** them (Egress). If you only listed `Ingress` here, the `egress` rules below would be ignored entirely, even if you wrote them.

## Now the two rule blocks — this is where "who's allowed" actually gets defined

### The Ingress block — who's allowed to send traffic INTO the frontend pods

```yaml
  ingress:
  - from:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 80
```
`from: podSelector: {}` — an **empty** selector means "match every pod in this namespace" (same empty-selector trick as default-deny's `podSelector: {}`, just used here to mean "allow from," not "apply to"). So this says: *"any pod in `production` may send traffic to the frontend pods, but only on TCP port 80."* In a tighter real-world setup you might restrict this further (e.g., only allow from an ingress controller), but for this lab we're keeping frontend open to receive traffic from anywhere inside the namespace on port 80 (the nginx port).

### The Egress block — who the frontend pods are allowed to send traffic OUT to

```yaml
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - protocol: TCP
      port: 3000
```
This is the **core security rule** of this whole policy: *"frontend pods may only send outbound traffic to pods labeled `tier: api`, and only on port 3000."* This is what makes Frontend→API work and is also **why Frontend→Database is blocked** — the database isn't mentioned anywhere in this egress list, so it's implicitly denied (recall: once a pod is selected by any policy, only explicitly listed traffic is allowed — everything else defaults to deny).

```yaml
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```
This is the **second, separate** egress rule — a completely independent entry in the same `egress` list. It says: *"frontend pods may also send traffic to any pod in a namespace labeled `name: kube-system`, but only on UDP port 53."* This is exclusively the DNS exception — port 53 is the standard DNS port, and `kube-system` is where CoreDNS lives (recall Step 2, where we manually labeled that namespace `name: kube-system` — this exact selector is *why* that labeling step was necessary).

## The one-sentence summary to give students

> "This policy says: I am attaching myself to every pod labeled `tier: frontend`. Anyone in this namespace may talk **to** me on port 80. I, in turn, am only allowed to talk **out** to two places: pods labeled `tier: api` on port 3000, and DNS in `kube-system` on port 53. Anything else — including talking directly to the database — I simply cannot do, because it was never listed."

## One good visual way to reinforce this live

Draw it as a simple box: **frontend** pod in the middle, one arrow pointing **in** (labeled "port 80, from anyone"), and two arrows pointing **out** (one labeled "port 3000, to `tier: api`," one labeled "port 53 UDP, to `kube-system`"). That's the entire policy, visually — everything not drawn as an arrow is blocked.
