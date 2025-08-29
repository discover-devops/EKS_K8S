
---

# **Lesson 5: Understanding ALB Ingress Controller on AWS EKS (Architecture & Concepts)**

---

##  **Lesson Overview**

In this lesson, we move one step forward from Classic and Network Load Balancers. We explore how **Application Load Balancer (ALB)** — the most feature-rich Layer 7 load balancer in AWS — integrates with **Kubernetes** using the **AWS ALB Ingress Controller** (also known as ALB IC or AWS Load Balancer Controller). We'll focus on **concepts and architecture only** in this lesson. Implementation begins in Lesson 6.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/1928cf84-84ec-4d39-afe1-dfdca029e905" />


---

##  **Why Application Load Balancer?**

ALB is not just a load balancer. It is a **smart Layer 7 traffic manager** that understands HTTP/HTTPS semantics and supports advanced features:

###  ALB Features:

* **Path-based routing**: `/api` → backend A, `/admin` → backend B
* **Host-based routing**: `app.example.com` vs `admin.example.com`
* **Header/method/query parameter-based routing**
* **HTTPS redirect and custom responses**
* **OIDC/Cognito authentication at the LB layer**
* **Lambda functions as targets**
* **Native integration with ECS, EKS, Fargate**
* **Monitoring via CloudWatch**
* **Dynamic health checks per Target Group**

> ALB empowers your Kubernetes Ingress to behave like an **enterprise-grade API Gateway**.

---

##  **How ALB Works in Kubernetes via Ingress Controller**

When you deploy an **Ingress resource** in Kubernetes (with `ingressClassName: alb`), the **ALB Ingress Controller** detects it and automatically provisions and manages AWS resources on your behalf.

###  Resources created by ALB Ingress Controller:

1. **ALB (ELBv2)** – An internet-facing or internal ALB is created.
2. **Target Groups** – One for each Kubernetes `Service`.
3. **Listeners** – Port 80/443 rules to route requests.
4. **Routing Rules** – Based on paths, hosts, etc. defined in Ingress manifest.
5. **Security Group Attachments**, **Subnets**, **Tags**, etc.

All these AWS resources are **tied to the lifecycle of the Ingress object**.

---

##  **Traffic Modes Supported**

### 1️ **Instance Mode**

* ALB forwards traffic to the **worker node IPs** (NodePort)
* Requires exposing K8s services as **type: NodePort**
* ALB ↦ Node ↦ NodePort ↦ Pod

 Default mode
 Compatible with EC2-based node groups
 Not compatible with Fargate (no NodePort allowed)

---

### 2️ **IP Mode**

* ALB directly targets **Pod IPs** via `target-type: ip`
* No need for NodePort
* ALB ↦ Pod

 Works with **EC2 + Fargate**
 Cleaner for microservices
 Set via annotation:

```yaml
alb.ingress.kubernetes.io/target-type: ip
```

---

##  **Step-by-Step Flow of ALB Ingress Controller**

1. **User applies an Ingress manifest** with `ingressClassName: alb`
2. ALB Controller (running in the cluster):

   * Watches Kubernetes API
   * Detects new Ingress resource
3. ALB Controller:

   * Creates an **ALB (ELBv2)** in AWS
   * Creates **Target Groups** for each backend service
   * Adds **Listeners** (port 80, 443) with routing rules
   * Registers **targets** (Pods or Nodes) based on mode
4. Application becomes accessible at:

   ```
   http://<ALB-DNS>/path1
   http://<ALB-DNS>/path2
   ```

---

##  **IAM & Permissions**

* ALB Controller requires an IAM Role with permissions to:

  * Create and manage ELBv2
  * Manage Target Groups, Listeners, Tags
* In EKS, this is configured using **IRSA (IAM Role for Service Account)**.

---

##  **Use Cases**

| Use Case                                        | Why ALB?                          |
| ----------------------------------------------- | --------------------------------- |
| Expose multiple apps via same LB                | Use path- or host-based routing   |
| Secure ingress with HTTPS + OIDC                | Terminate TLS + integrate Cognito |
| Want AWS-managed load balancer with WAF, Shield | ALB is fully integrated           |
| Fargate workloads                               | Must use ALB (with IP mode)       |
| Need Layer 7 features                           | Only ALB supports it              |

---

##  **Key Annotations**

| Annotation                                          | Purpose                             |
| --------------------------------------------------- | ----------------------------------- |
| `kubernetes.io/ingress.class: alb`                  | Required to use ALB IC              |
| `alb.ingress.kubernetes.io/scheme: internet-facing` | or `internal`                       |
| `alb.ingress.kubernetes.io/target-type: ip`         | or `instance`                       |
| `alb.ingress.kubernetes.io/group.name: shared-alb`  | Share 1 ALB across multiple Ingress |
| `alb.ingress.kubernetes.io/certificate-arn`         | For HTTPS termination               |
| `alb.ingress.kubernetes.io/actions.redirect`        | For URL redirects                   |

---

##  Summary Diagram (Conceptual Flow)

```
[ Internet ]
     │
     ▼
[ AWS ALB (ELBv2) ]
     │  ┌────────────────────┐
     │  │ Listeners & Rules │
     ▼  └────────────────────┘
[ Target Groups ] ←───── Ingress Resources
     │
     ▼
[ Services ] → [ Pods (EC2 or Fargate) ]
```

---

##  Summary: Why ALB Ingress?

* It brings the **power of AWS ALB** into Kubernetes-native routing.
* Enables **clean separation** of external traffic routing from application logic.
* Plays well with **EC2, Fargate, HTTPS, Cognito, WAF**, and other AWS-native services.
* Supports both **instance and IP modes**, making it flexible for all deployment models.

---

