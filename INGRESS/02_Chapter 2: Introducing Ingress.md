# Chapter 2: Introducing Ingress

**Series:** Kubernetes Traffic Management: From Services to Ingress
**Format:** Concept + context (hands-on lab begins in Chapter 3, once we install a real Ingress Controller)

---

## Recap from Chapter 1

Every limitation we found — multiple IPs, fragmented SSL, no path-based routing, no traffic-splitting — traced back to one root cause: `LoadBalancer` Services operate at **Layer 4**, and the decisions a real multi-service company needs (route by hostname, route by path, split traffic by percentage) all live at **Layer 7**. We closed with a question: what would a system need to be capable of, to actually solve this?

That system is **Ingress**.

---

## Ingress is Kubernetes' Layer 7 (HTTP/HTTPS) routing solution

But here's the part that trips people up when they first meet Ingress: **"Ingress" is not one thing. It's two components working together, and understanding the split between them is the single most important concept in this chapter.**

1. **Ingress Resource** — a Kubernetes object that *declares* routing rules (host X goes to service Y, path A goes to service B)
2. **Ingress Controller** — the actual running software that *reads* those rules and *implements* them as real traffic routing

Miss this distinction, and everything downstream gets confusing — including why an `Ingress` YAML can apply successfully with zero errors and still do absolutely nothing (a trap you've already seen once with `NetworkPolicy` objects on a CNI that doesn't enforce them).

### The analogy

- **Ingress Controller** = the master receptionist sitting at the main desk of the corporate building. This is the person actually doing the work — greeting visitors, deciding where to send them.
- **Ingress Resource** = the corporate directory rulebook sitting on the receptionist's desk, telling them exactly which department to send a visitor to, based on what the visitor is asking for.

The rulebook, by itself, does nothing. It's just paper. It only becomes useful once there's an actual receptionist reading it and acting on it. **Write an Ingress Resource with no Ingress Controller installed, and you have a rulebook sitting on an empty desk — visitors show up, and nobody is there to read the rulebook and direct them anywhere.**

### Why did Kubernetes separate these into two pieces?

Sit with this before reading on: what's the benefit of splitting "the rules" from "the thing that enforces the rules," rather than just building one combined object?

**The answer: pluggability.** The Ingress Resource is pure declarative configuration — it says *what* you want, not *how* to achieve it. The actual implementation is swappable: you can run **NGINX**, **Traefik**, **HAProxy**, or a cloud-native controller (like the **AWS Load Balancer Controller** we'll install in Chapter 3) — and your Ingress Resource YAML stays *identical* no matter which one you choose. This is the same design philosophy Kubernetes uses everywhere: the API describes intent, a pluggable controller fulfills it. (You've actually already seen this exact pattern once — Cluster Autoscaler and Karpenter are both "controllers" that read Kubernetes' desired state and act on AWS infrastructure differently underneath, while the pods needing capacity don't care which one is running.)

---

## The diagram — reading it correctly

The reference diagram for this chapter shows the full shape of what we're building toward:

```
                    client
                       |
     ┌─────────────────────────────────────┐
     │              Ingress                 │
     │  www.app1.com   www.app2.com   www.app3.com
     └──────┬──────────────┬──────────────┬─┘
            │              │              │
        service        service        service
            │              │              │
       pod pod pod    pod pod pod    pod pod pod
       (Deployment /   (Deployment /  (Deployment /
        ReplicaSet)     ReplicaSet)    ReplicaSet)

             all inside: kubernetes cluster
```

One client, one entry point, one Ingress — and from there, **host-based routing** fans traffic out to three completely independent Services, each backing its own set of pods. This is the single-entry-point model that directly solves Chapter 1's Problem 1 (multiple IPs, multiple load balancers) — now it's **one IP, one Ingress, three internal routing rules.**

> 📝 **Terminology note:** the diagram labels each pod group as managed by a "replication controller" — this is legacy terminology from very early Kubernetes. In any cluster you'll actually work with today, that role is filled by a **ReplicaSet**, which itself is managed by a **Deployment** (exactly like every Deployment you've written throughout this entire course). `ReplicationController` still technically exists in the API for backward compatibility, but you should never create one directly — always use a Deployment.

---

## The Ingress Resource — a first look

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
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

**Before reading the explanation below — predict what this does.** If you're running this live in class, give students 30 seconds to discuss with a neighbor before revealing the answer.

### What this Ingress actually says

- Requests arriving for host **`api.company.com`** — this is **host-based routing**, the exact capability Chapter 1 proved a Layer 4 `LoadBalancer` could never provide
- Matching path **`/`** (with `pathType: Prefix`, meaning "starts with `/`," which in practice matches everything) — this is **path-based routing**, Chapter 1's Problem 3, now solved
- Forward matching traffic to **`api-service`**, on port **80** — same `Service` object type you already know, nothing new here; Ingress routes *to* Services, it doesn't replace them

### The crucial thing missing from this YAML

Look again. **Is there an `EXTERNAL-IP` anywhere in this manifest?** Compare this to a `LoadBalancer` Service, where `kubectl get service` immediately shows you a real public IP the moment you create it.

There's nothing like that here. So — **where does traffic actually enter the cluster?** What is the client's browser actually connecting to, if not something declared in this YAML?

Sit with that question. The answer is the entire subject of **Chapter 3: The Ingress Controller** — because the honest truth is: **this YAML, by itself, creates no entry point at all.** It's a rulebook with no receptionist yet. The IP address, the actual listener accepting traffic from the internet, only comes into existence once a real Ingress Controller is installed and takes ownership of this resource.

---

*End of Chapter 2. Chapter 3: The Ingress Controller — where we finally install something that makes this YAML actually do anything.*
