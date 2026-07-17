# Chapter 1: The Growing API Company — Why Services Aren't Enough

**Series:** Kubernetes Traffic Management: From Services to Ingress

**Format:** Concept + context (no hands-on in this chapter — the lab starts once we introduce the Ingress Controller)

---

## Opening: The Growing API Company

Let's walk through a scenario that mirrors what almost every company experiences as it scales.

**Month 1:** Your startup launches with a simple API. One service, one `LoadBalancer` exposing it to the world. Life is simple.

**Month 6:** You now have:
- `api.company.com` — your main API
- `admin.company.com` — admin dashboard
- `blog.company.com` — marketing blog
- `docs.company.com` — documentation portal

Each of these is a separate Kubernetes Service. Each needs external access.

**Question to sit with before reading further:** if you expose each of these using a `LoadBalancer` Service, what problem are you creating?

Common answers people land on: cost, IP management, SSL certificate sprawl. All correct — let's quantify and then go one level deeper than the obvious answer.

---

## The cost problem, quantified

On AWS, each `LoadBalancer` Service provisions a real Elastic Load Balancer, costing roughly **$16–20/month**. 

With 4 services, that's **~$80/month** just for load balancers, before you've served a single byte of actual traffic. 

Now imagine a company with 20 externally-facing services — that's **$320–400/month** in load balancer costs alone, scaling linearly with every new service you ship.

But cost isn't the deepest issue here. There's a structural problem underneath it — let's find it.

---

## The deeper issue: no centralized control

Beyond the dollar cost, you're now managing:
- **SSL certificates across multiple load balancers**, independently, with no shared renewal or rotation strategy
- **Health checks configured separately** for each load balancer, with no consistent policy
- **Zero centralized visibility** into traffic patterns — every load balancer is its own island, with its own logs, its own metrics, no unified view of "what is my traffic actually doing right now"

This is the problem Ingress exists to solve. But before looking at the solution, it's worth sitting with the problem a bit longer — specifically, understanding *why* a `LoadBalancer` Service is structurally incapable of doing better, not just "expensive."

### The analogy

Imagine a large corporate office building. Using a separate `LoadBalancer` for every service is like forcing every single department — HR, IT, Sales, Legal — to hire its own dedicated street-level receptionist, install its own separate front door onto the street, and staff its own separate security desk. It's expensive, it's chaotic for visitors trying to figure out which door to use, and there's no single person who can tell you "here's everyone who's currently in the building."

---

## Part 1: Why Services Aren't Enough

### The LoadBalancer Service pattern

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: LoadBalancer
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 8080
```

What happens when you create this:

```bash
kubectl get service api-service
```
```
NAME          TYPE           EXTERNAL-IP      PORT(S)
api-service   LoadBalancer   34.123.45.67     80:31234/TCP
```

You get a real public IP. Your DNS points `api.company.com` at `34.123.45.67`. Traffic flows like this:

```
User → 34.123.45.67:80 → NodePort 31234 → Service → Pod
```

### What layer does this actually operate at?

**Layer 4 (Transport layer — TCP/UDP).** A `LoadBalancer` Service forwards packets based purely on IP address and port number. It has no concept of what's *inside* those packets.

### Can a Layer 4 load balancer make routing decisions based on URL path or hostname?

**No — and this is the crux of the entire chapter.** Because hostname and URL path are **Layer 7 (Application layer)** concepts — specifically, HTTP concepts. A Layer 4 load balancer only ever sees:

```
Source IP: 1.2.3.4
Destination IP: 34.123.45.67
Port: 80
Protocol: TCP
[Encrypted payload if HTTPS]
```

It fundamentally **cannot see**:
- **Hostname** — `api.company.com` vs. `admin.company.com` look identical to it; both are just "traffic arriving at this one IP on port 80"
- **Path** — `/users` vs. `/products` — invisible, since paths live inside the HTTP request, which Layer 4 never inspects
- **HTTP methods** — `GET` vs. `POST` — same problem
- **Headers** — cookies, auth tokens, anything carried in HTTP headers is opaque to it

### The analogy

A Layer 4 load balancer is like a mail-sorting machine that can only read the ZIP code printed on the outside of an envelope. It knows exactly which building to route the letter to — but it has no idea what's written inside. Because it can never open and read the letter (the Layer 7 HTTP data), it has no way to decide which specific department inside that building should actually receive it. Every letter with that ZIP code goes to the same loading dock, regardless of who it's actually addressed to internally.

---

## The limitations stack up — a concrete walkthrough

Say you provision two separate services:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: LoadBalancer   # Gets IP: 34.123.45.67
---
apiVersion: v1
kind: Service
metadata:
  name: admin-service
spec:
  type: LoadBalancer   # Gets IP: 35.234.56.78
```

### Problem 1 — multiple IPs, multiple load balancers

```
api.company.com    → A record → 34.123.45.67 → AWS ELB #1 ($18/mo)
admin.company.com  → A record → 35.234.56.78 → AWS ELB #2 ($18/mo)
```

Every new externally-facing service means another IP, another ELB, another monthly line item — and this scales linearly forever, with no ceiling.

### Problem 2 — SSL certificate management, fragmented

Each load balancer needs its own TLS certificate configuration. There's no single place to manage renewals, no shared wildcard cert strategy unless you manually replicate it across every ELB — and if you forget to renew one, that specific service silently starts failing HTTPS while everything else keeps working, making the failure hard to correlate.

### Problem 3 — no path-based routing, at all

Say you want:

```
api.company.com/v1  → api-v1-service
api.company.com/v2  → api-v2-service
```

**With plain `LoadBalancer` Services, this is not possible.** Same hostname, same IP — Layer 4 fundamentally cannot distinguish `/v1` from `/v2`, because the path is Layer 7 information that never reaches the load balancer's decision-making logic. You'd be forced into separate IPs (and separate DNS entries) even for what should logically be one API with two versions.

### One more concrete example worth adding — a real incident shape

Imagine `api.company.com` needs a **canary deployment** — you want 5% of traffic routed to a new version while 95% stays on the stable version, based on nothing more than a routing percentage. A Layer 4 `LoadBalancer` Service has **no mechanism for this at all** — it doesn't understand "requests," only "connections," and it has no weighting logic beyond raw connection distribution across whatever pods a single Service's selector matches. Canary deployments, A/B testing, and traffic-splitting by header or cookie are all **Layer 7 problems**, and a Layer 4 tool simply has no vocabulary to express them.

---

## Where this leaves us

Every limitation above traces back to the same root cause: **a `LoadBalancer` Service operates one layer too low to make the kind of intelligent, application-aware routing decisions a real multi-service company needs.** More IPs, fragmented SSL, no path routing, no traffic-splitting — these aren't separate bugs to patch individually. They're all downstream symptoms of operating at Layer 4 when the actual decisions that matter live at Layer 7.

**The question to carry into Chapter 2:** if you were designing a better system from scratch — one entry point, one IP, one place to manage SSL, with the intelligence to actually read HTTP requests and route based on hostname and path — what would that system need to be capable of?

That's exactly the gap Kubernetes' `Ingress` resource was built to close.

> 📝 **A forward note on the EKS-specific piece, so it's not a surprise later:** just like `NetworkPolicy` objects needed the VPC CNI's enforcement flag turned on before they did anything real, Kubernetes' `Ingress` object is also just an **API specification** — creating one, by itself, does nothing on its own. Something has to actually *implement* it. On EKS, that's a separate component called the **AWS Load Balancer Controller**, which we'll install in Chapter 3 before writing our first real `Ingress` object. Mentioning this now so the shape of "spec vs. enforcement" feels familiar when we get there — it's the same pattern you've already seen once.

---

*End of Chapter 1. Chapter 2: Introducing Ingress.*
