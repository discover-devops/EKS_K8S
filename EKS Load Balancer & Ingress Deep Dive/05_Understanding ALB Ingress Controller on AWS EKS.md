
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



### How Traffic Flows


step by step — from the moment a user types `https://mymovieapp.com` in the browser, all the way to your **Pod** running inside **EKS**.

This explanation is tailored for a **production-grade Kubernetes setup on AWS EKS using ALB Ingress Controller**, where your application (e.g., `mymovieapp`) is exposed securely using **ALB** + **Route 53**.

---

##  Scenario

**User types**:
`https://mymovieapp.com`

---

##  Step-by-Step Traffic Flow from Laptop to Pod

---

###  **1. User’s Browser → DNS Resolution Begins**

* The user opens a browser and types `mymovieapp.com`
* The browser first checks the **local DNS cache**
* If not found, it sends a DNS query to the **ISP’s resolver**
* The resolver recursively reaches out to:

  * **Root DNS → `.com` TLD DNS → Route 53 (Authoritative)**

---

###  **2. Route 53 Resolves Domain to ALB**

* In **Route 53**, you’ve configured an **A or CNAME record**:

  * `mymovieapp.com` → **Alias or CNAME → ALB DNS**

    * e.g., `a1234abcd.elb.us-east-1.amazonaws.com`
* Route 53 replies with the **ALB’s DNS name / IP address**.

---

###  **3. Browser Sends HTTPS Request to ALB (ELBv2)**

* The browser initiates a **TLS handshake** (if HTTPS) with the ALB
* ALB serves the **SSL certificate** you’ve attached via Ingress annotation
* Browser trusts the certificate → connection is established
* Browser sends HTTP request to `https://mymovieapp.com`

---

###  **4. ALB Listeners & Rules Take Over**

* ALB has **Listeners** (port 443 / 80) and **Rules**:

  * Host-based: `mymovieapp.com` → Target Group A
  * Path-based: `/admin` → Target Group B, etc.
* Based on these rules, ALB forwards the request to a **Target Group**

---

###  **5. Target Group Routes to Kubernetes (via Ingress Controller)**

Depending on **target type**:

#### If `target-type: instance`:

* ALB sends traffic to **Node’s IP\:NodePort**
* K8s service routes request to appropriate **Pod**

#### If `target-type: ip` (recommended):

* ALB directly sends traffic to **Pod IPs** (via Service)

---

###  **6. Pod Receives the Request & Responds**

* Pod is running inside EKS (in private subnets)
* It processes the request (e.g., return movie listings)
* Sends response back via same path:

  * **Pod → ALB → User’s Browser**

---

##  Full Traffic Diagram (in words)

<img width="1024" height="1024" alt="image" src="https://github.com/user-attachments/assets/94d19c3a-dbe7-4344-bf45-668f7f2579ae" />


```text
User Laptop (Browser)
    |
    └──> DNS Query: mymovieapp.com
            |
            └──> Route 53 (Authoritative DNS)
                     |
                     └──> ALIAS → ALB DNS
                              |
                              ▼
                      [AWS ALB (ELBv2)]
                              |
                       ┌─────────────┐
                       │ HTTPS Listener│
                       └─────────────┘
                              |
                     Match Host: mymovieapp.com
                     Match Path: /
                              ▼
                    Target Group: EKS Service
                              |
                     ┌────────────┐
                     │  Ingress   │
                     │ Controller │
                     └────────────┘
                              |
                      Routes to Service
                              ▼
                           Pod (in private subnet)
                              |
                          Responds
                              ▼
                         ALB → Browser
```

---

##  Key AWS Components Involved

| Component              | Role                                                    |
| ---------------------- | ------------------------------------------------------- |
| Route 53               | DNS resolution (mymovieapp.com → ALB)                   |
| ALB (ELBv2)            | Terminates TLS, applies host/path rules, routes traffic |
| ALB Ingress Controller | Watches Ingress objects, manages ALB+Target Groups      |
| EKS                    | Hosts the application Pods in private subnets           |
| Kubernetes Service     | Routes to Pod(s)                                        |

---

##  Summary (What Makes This Architecture Powerful)

* **Zero manual provisioning**: ALB + Target Groups are created by **Ingress Controller**
* **One ALB can serve multiple apps** (via host/path rules)
* **Pod IP mode** avoids NodePort complexity
* **Route 53 + HTTPS + TLS termination** all native and secure
* **Scalable + observable + production-grade** using ALB logs, CloudWatch, WAF, etc.

---

