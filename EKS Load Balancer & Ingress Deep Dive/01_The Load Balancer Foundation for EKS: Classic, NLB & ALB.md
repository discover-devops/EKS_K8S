
---

#  **Lesson 1: AWS Load Balancer Fundamentals for Kubernetes – From NodePort to Production-Ready Architecture**

---

##  **Introduction**

In this foundational lesson, we explore how Kubernetes Services interact with AWS Load Balancers, and why understanding this relationship is essential for building scalable, secure, and production-grade architectures on EKS (Elastic Kubernetes Service). We'll begin by understanding AWS's native load balancing capabilities and see how Kubernetes uses them internally.

---

##  **Elastic Load Balancing in AWS**

AWS offers **three types of load balancers** to provide high availability and distribute incoming traffic:

1. **Classic Load Balancer (CLB)** – Legacy LB that supports **HTTP, HTTPS, TCP, SSL** protocols.
2. **Network Load Balancer (NLB)** – Best for **TCP/UDP/TLS**, high-performance, scalable applications.
3. **Application Load Balancer (ALB)** – L7 load balancing for **HTTP/HTTPS**, ideal for path- and host-based routing.

🔗 AWS provides [official comparison tables](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/elb-types.html) that highlight the features of each LB type.

>  Note: Classic Load Balancer is considered **legacy** and may be deprecated in the future, but it is still the **default** load balancer created by Kubernetes `Service` objects of type `LoadBalancer` in AWS.

---

##  **How Kubernetes Uses Load Balancers**

When you create a **Kubernetes Service** with `type: LoadBalancer`, EKS (via AWS cloud provider integration) automatically provisions a **Classic Load Balancer** in AWS. This Load Balancer is used to expose your app to the public internet.

This allows you to:

* Expose apps on standard HTTP (port 80) and HTTPS (port 443).
* Avoid hardcoding Node IP + random port (as done with NodePort).
* Use DNS URLs to access services externally.

---

##  **Architecture Shift: From Public to Private Subnets**

In early-stage EKS setups, **worker nodes** (EC2 instances) are often placed in **public subnets**. However, in production:

* **Worker nodes should be in private subnets** to increase security.
* **Cluster control plane can remain public** or private depending on your access strategy.
* Nodes in private subnets communicate with the control plane using **NAT Gateways** or **VPC private link endpoints**.

In this updated design:

* Load balancer (CLB or NLB) sits in a **public subnet**.
* Requests from users hit the load balancer.
* Traffic is routed to **Pods in private subnets** via the Kubernetes service.
* Apps can connect to **databases like RDS** via `ExternalName` services.

---

##  **End-to-End Traffic Flow**

1. User accesses `http://CLB-DNS-URL/user-mgmt/health-status`.
2. **Classic Load Balancer** forwards the request to the Kubernetes Service.
3. The Service routes the request to the appropriate **Pod**.
4. The Pod connects to an **RDS database** using an **ExternalName service**.
5. Response is sent back via the same path.

This is a complete and production-aligned setup where:

* Worker nodes and database are in private subnets.
* Load balancer is in a public subnet.
* Secure, scalable communication is enabled using NAT and Kubernetes constructs.

---

##  **Summary**

* AWS provides multiple load balancer types (CLB, NLB, ALB) with different capabilities.
* Kubernetes `Service type: LoadBalancer` integrates with AWS and creates a **Classic ELB by default**.
* Using **NodePort** services is not recommended for production.
* To make workloads production-ready:

  * Use **LoadBalancers** (CLB/NLB) to expose your apps.
  * Shift worker nodes to **private subnets**.
  * Use **DNS + standard ports** (80/443) for real-world accessibility.
* In future lessons, you’ll learn to use **ALB with Ingress Controllers** for even better routing, cost control, and SSL termination.

---

