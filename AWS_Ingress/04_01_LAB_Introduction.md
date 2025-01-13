### **LAB Introduction: Load Balancing Workloads on EKS using AWS Application Load Balancer**

This lab is designed to provide a **step-by-step** and **modular approach** to learning and implementing load balancing for Kubernetes workloads on Amazon EKS using the **AWS Application Load Balancer (ALB)**. 

The lab will cover essential topics for mastering ALB Ingress, ensuring you gain hands-on experience with each concept.

---

### **Topics Covered**

The lab is divided into the following topics:

| **S.No** | **Topic Name**                                       |
|----------|-----------------------------------------------------|
| 1        | **AWS Load Balancer Controller Installation**        |
| 2        | **ALB Ingress Basics**                               |
| 3        | **ALB Ingress Context Path-Based Routing**           |
| 4        | **ALB Ingress SSL**                                  |
| 5        | **ALB Ingress SSL Redirect (HTTP to HTTPS)**         |
| 6        | **ALB Ingress External DNS**                         |
| 7        | **ALB Ingress External DNS for Kubernetes Ingress**  |
| 8        | **ALB Ingress External DNS for Kubernetes Service**  |
| 9        | **ALB Ingress Name-Based Virtual Host Routing**      |
| 10       | **ALB Ingress SSL Discovery - Host**                 |
| 11       | **ALB Ingress SSL Discovery - TLS**                  |
| 12       | **ALB Ingress Groups**                               |
| 13       | **ALB Ingress Target Type - IP Mode**                |
| 14       | **ALB Ingress Internal Load Balancer**               |

---

### **References for Additional Understanding**
To deepen your knowledge and troubleshoot effectively, refer to the following resources:

1. **AWS Load Balancer Controller**:
   - Official documentation: [AWS Load Balancer Controller Documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller)

2. **ALB Ingress Annotations Reference**:
   - Comprehensive list of annotations: [ALB Ingress Annotations](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/ingress/annotations/)

3. **EKSCTL - Getting Started**:
   - Learn how to set up your EKS cluster: [EKSCTL Introduction](https://eksctl.io/introduction/#getting-started)

4. **External DNS**:
   - Repository and tutorials:
     - [External DNS GitHub Repository](https://github.com/kubernetes-sigs/external-dns)
     - [ALB Ingress with External DNS Tutorial](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/alb-ingress.md)
     - [AWS External DNS Tutorial](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md)

---

### **Learning Objectives**
By the end of this lab, you will:
- Understand the role of the **AWS Load Balancer Controller** in Kubernetes.
- Configure **context path-based routing**, **name-based routing**, and **SSL** for ALB Ingress.
- Integrate **External DNS** with Kubernetes Ingress and Services.
- Learn about **Ingress groups** and **target types** (Instance/IP mode).
- Deploy internal ALBs for private workloads.

---

### **Next Steps**
In the first module, we will begin with **AWS Load Balancer Controller Installation**, covering prerequisites, IAM roles and policies, and Helm-based deployment.

Let’s get started!
