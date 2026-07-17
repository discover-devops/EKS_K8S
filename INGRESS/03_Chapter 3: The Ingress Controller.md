# Chapter 3: The Ingress Controller

**Series:** Kubernetes Traffic Management: From Services to Ingress
**Format:** Concept + first hands-on lab of this series

---

## Recap from Chapter 2

An Ingress Resource, by itself, does nothing. It's a rulebook with no receptionist. We closed Chapter 2 on a specific, unanswered question: **where does the `EXTERNAL-IP` actually come from, if there's no such field in the Ingress YAML?**

The answer is the **Ingress Controller** — the actual running software that reads Ingress Resources and turns them into real traffic routing.

---

## Blueprint and Builder

- **Ingress Resource** = the blueprint — routing rules, on paper, changing nothing by itself
- **Ingress Controller** = the builder — the thing that actually reads the blueprint and makes it real

Nothing happens in your cluster until the builder shows up. This is the exact same "spec vs. enforcement" gap you've already seen twice now — `NetworkPolicy` objects needing the VPC CNI's enforcement flag, and now Ingress Resources needing an actual controller. **This is a recurring pattern in Kubernetes worth naming explicitly to students: a huge number of Kubernetes objects are pure declarations of intent, and mean nothing until some separate controller is watching for them and acting.**

---

## ⚠️ Important: which controller to teach, given where we are in 2026

The most widely-referenced Ingress Controller in tutorials — **`ingress-nginx`** — is now **end-of-life**. Its maintainers announced best-effort maintenance only through **March 2026**; after that: no further releases, no bugfixes, no security patches. Existing deployments keep running, but new production use is explicitly discouraged by the project itself, in favor of either a **Gateway API** implementation or, on EKS specifically, the **AWS Load Balancer Controller**.

**For this course, we'll do both, deliberately:**
- Install `ingress-nginx` first, because it's the clearest way to *see* the controller-pod-does-the-routing architecture explicitly (and it's still what the vast majority of existing real-world clusters run today, EOL or not)
- Flag the AWS Load Balancer Controller as the actual production-recommended path on EKS, with its meaningfully different architecture (no controller pod in the traffic path at all — the ALB itself does Layer 7 routing natively)

---

## Installing the Ingress Controller (ingress-nginx, for teaching the concept)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/aws/deploy.yaml
```

**Before running this — predict what gets created.** What components would a controller need to actually function?

```bash
kubectl get all -n ingress-nginx
```

Expected components: a Deployment (the controller pods), a ConfigMap (NGINX configuration), RBAC resources (the controller needs permission to watch Ingress objects cluster-wide), and — the interesting one:

```
NAME                                   TYPE           EXTERNAL-IP      PORT(S)
ingress-nginx-controller               LoadBalancer   34.123.45.67     80:31080/TCP,443:31443/TCP
```

**The Ingress Controller itself is exposed via a `LoadBalancer` Service.** This is the moment to directly resolve the ALB/NLB confusion from earlier: on AWS, this provisions a real **NLB — a Layer 4 device.** That NLB's only job is "get traffic from the internet into the cluster, onto the right NodePort." **The actual Layer 7 decision — reading the Host header, matching the path, picking a backend Service — happens entirely in software, inside the NGINX controller pod itself, after the NLB has already handed the connection off.** The load balancer and the routing logic are two separate things sitting at two different layers, even though they're bundled together conceptually under "Ingress."

**The key operational point:** you only need **one** `LoadBalancer` — and therefore one NLB, one monthly bill — for **every** Ingress Resource in your entire cluster. This is Chapter 1's Problem 1, fully solved: one entry point, arbitrarily many routing rules behind it.

---

## Tracing one request, start to finish

This is the most important diagram in the whole chapter — walk through it slowly, because it's the concrete answer to "where does Layer 7 routing actually happen."

```
1. USER'S BROWSER
   GET https://api.company.com/users

2. DNS RESOLUTION
   api.company.com → 34.123.45.67   (the Ingress Controller's NLB IP)

3. CLOUD LOAD BALANCER (34.123.45.67) — Layer 4
   Forwards to: Kubernetes NodePort (31080/31443)
   Decision made here: NONE. Just IP + port forwarding.

4. INGRESS CONTROLLER POD (NGINX) — Layer 7, this is where it happens
   Step 1: Receive HTTPS request
   Step 2: TLS termination (decrypt)
   Step 3: Read HTTP headers — Host: api.company.com, Path: /users
   Step 4: Match against Ingress rules — host ✓, path ✓
   Step 5: Forward to backend — service: api-service, port: 80

5. SERVICE: api-service (ClusterIP)
   kube-proxy selects one backend pod (api-pod-2, say)

6. POD: api-pod-2
   Application receives GET /users, processes, returns JSON

   Response flows back the exact same path, in reverse.
```

**Direct answer to "where does Layer 7 routing happen":** step 4, inside the Ingress Controller pod — nowhere else. Steps 3 and 5 are both doing dumb, content-blind forwarding (Layer 4 logic, or in step 5's case, simple label-based pod selection). Only step 4 actually opens the HTTP request and makes a decision based on what's inside it.

---

## 🔧 Lab: install and verify on your own cluster

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/aws/deploy.yaml
kubectl get pods -n ingress-nginx --watch
```

Wait for the controller pod to show `Running`, `1/1`. Then:

```bash
kubectl get svc -n ingress-nginx
```

Note the `EXTERNAL-IP` on `ingress-nginx-controller` — this is your NLB's DNS hostname (AWS NLBs use a hostname, not a bare IP, unlike the simplified `34.123.45.67` shown in the diagram above for teaching clarity).

```bash
kubectl get pods -n ingress-nginx -o wide
```

Confirm the controller pod's IP — this is the actual Layer 7 engine from step 4 above, running as a real pod, on a real node, in your real cluster.

> 📝 We're not creating an Ingress Resource yet — that's Chapter 4 (Host-Based Routing) and Chapter 5 (Path-Based Routing). Right now, the controller is running with nothing to route, which is a perfectly valid, expected state: the builder has shown up on site, but no blueprint has been handed over yet.

---

## Forward note: the AWS-native alternative

The **AWS Load Balancer Controller** takes a structurally different approach: instead of routing traffic *through* a controller pod running NGINX, it provisions a **real ALB** and configures the ALB's own native listener rules directly — meaning steps 3 and 4 above **collapse into one step**, done natively by AWS infrastructure, with no NGINX pod anywhere in the traffic path. We'll install and contrast this directly in a later chapter, once host-based and path-based routing concepts are solid using the simpler `ingress-nginx` model first.

---

*End of Chapter 3. Chapter 4: Host-Based Routing.*
