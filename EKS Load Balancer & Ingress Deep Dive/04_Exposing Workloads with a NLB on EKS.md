
---

# Lesson 4: Exposing Workloads with a **Network Load Balancer (NLB)** on EKS

<img width="1024" height="1024" alt="image" src="https://github.com/user-attachments/assets/d0989377-c541-4957-92a1-ffb676fcc75b" />


##  Objectives

* Understand when to choose **NLB** vs Classic LB (CLB).
* Expose a private‑subnet workload through a **Kubernetes Service (type=LoadBalancer)** that provisions an **AWS NLB**.
* Verify end‑to‑end traffic from Internet → NLB → Pods on private nodes.

---

##  Context (where we are)

* From Lesson 2, your **EKS worker nodes run in private subnets** (no public IPs).
* You already grasp CLB behavior from Lesson 3.
* Now we’ll provision an **NLB** using a **single annotation** on a Service.

**Why NLB?**

* L4 (TCP/UDP/TLS) performance at scale, ultra‑low latency.
* Ideal for TCP/UDP services and high‑throughput HTTP when you don’t need L7 features (rewrites, host/path routing).
* Can target **instance** (nodes) or **ip** (Pod IPs). We’ll use **IP targets** to hit Pods directly (great with EKS VPC CNI).

---

##  High-Level Architecture

```
[ Client / Browser / curl ]
              |
              v
     [ AWS NLB  (internet-facing) ]
              |
              v
   K8s Service (type=LoadBalancer, NLB, IP targets)
              |
              v
       Pods (private subnets)
       └─ Deployment: nebula-api (HTTP echo)
```

* **One Service** with `type=LoadBalancer` + NLB annotation → AWS provisions **Network Load Balancer**.
* Traffic enters via NLB :80 and lands on **Pod IPs** (target-type=ip), even though Pods are on **private nodes**.

---

##  LAB: Deploy “nebula-api” behind an NLB

> **Namespace & app names are original**: `edge-demo` namespace, `nebula-api` service/app.

### 0) Pre-reqs

* EKS cluster reachable via `kubectl`
* Helm not required for this lab
* Nodes are in **private subnets** (from Lesson 2)

---

### 1) Apply manifests

Create a single file `nlb-nebula.yaml` with the following content:

```yaml
# --------------------------------------------
# Namespace
# --------------------------------------------
apiVersion: v1
kind: Namespace
metadata:
  name: edge-demo
  labels:
    app.kubernetes.io/part-of: edge-stack

---
# --------------------------------------------
# Deployment: nebula-api (simple HTTP echo)
# --------------------------------------------
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nebula-api
  namespace: edge-demo
  labels:
    app.kubernetes.io/name: nebula-api
    app.kubernetes.io/part-of: edge-stack
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: nebula-api
  template:
    metadata:
      labels:
        app.kubernetes.io/name: nebula-api
    spec:
      containers:
        - name: http-echo
          image: hashicorp/http-echo:0.2.3
          args: ["-text=Hello from NEBULA-API via NLB"]
          ports:
            - name: http
              containerPort: 5678
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 2
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
      terminationGracePeriodSeconds: 5

---
# --------------------------------------------
# Service: type=LoadBalancer, AWS NLB + IP targets
# --------------------------------------------
apiVersion: v1
kind: Service
metadata:
  name: nebula-api
  namespace: edge-demo
  labels:
    app.kubernetes.io/name: nebula-api
    app.kubernetes.io/part-of: edge-stack
  annotations:
    # Tell EKS to provision an AWS Network Load Balancer
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"

    # Use IP targets so NLB sends traffic directly to Pod IPs
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"

    # Cross-zone load balancing for better distribution (optional but recommended)
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"

    # Make it internal by uncommenting the next line (for private entry points)
    # service.beta.kubernetes.io/aws-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: nebula-api
  ports:
    - name: http
      port: 80           # NLB listener
      targetPort: http   # containerPort 5678
      protocol: TCP
  externalTrafficPolicy: Cluster
```

Apply:

```bash
kubectl apply -f nlb-nebula.yaml
```

---

### 2) Verify resources

```bash
kubectl -n edge-demo get deploy,svc,pods -o wide
# Watch for EXTERNAL-IP on the Service (this is your NLB DNS)
kubectl -n edge-demo get svc nebula-api -w
```

Open AWS Console:

* **EC2 → Load Balancers** → Type should be **Network**
* **Target Groups**: check targets show **healthy**

  * Because we’re using **IP targets**, you’ll see **Pod IPs** registered (not NodePorts).

---

### 3) Test end-to-end

When `EXTERNAL-IP` / DNS appears, test:

```bash
LB=$(kubectl -n edge-demo get svc nebula-api -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "NLB: http://$LB"
curl -s http://$LB
# -> Hello from NEBULA-API via NLB
```

Scale and observe:

```bash
kubectl -n edge-demo scale deploy/nebula-api --replicas=5
kubectl -n edge-demo get pods -o wide
# Repeated curls should still respond; NLB distributes across new Pods (IP targets)
```

---

### 4) Clean up

```bash
kubectl delete -f nlb-nebula.yaml
# Wait ~1–3 minutes for AWS to remove the NLB
```

---

##  Design Notes (what to explain to stakeholders)

* **Why NLB here?** We want a **high-performance L4 entry point**; application doesn’t need L7 features (rewrites/auth/rules). For L7, we’ll move to **ALB + Ingress** in the next lessons.
* **IP targets vs Instance targets**

  * **IP targets** (used above) route **directly to Pods**, skipping node‑port indirection, often simpler & efficient.
  * **Instance targets** route to **nodes** on a NodePort; fine too, but one more hop and port to manage.
* **Security**: Pods remain in **private subnets**; only the NLB is **internet-facing**. Pair with security groups, WAF (if ALB later), and private API endpoints as needed.
* **Observability**: Add logs/metrics via CloudWatch Container Insights or your APM (Datadog, Prometheus/Grafana).

---

##  Key Takeaways

* A single annotation flips a Service to **provision an NLB**.
* With **IP targets**, the NLB sends traffic straight to Pods, ideal on EKS with the VPC CNI.
* This is the **clean L4 pattern** before introducing **ALB/Ingress** for sophisticated HTTP routing & TLS.

---

