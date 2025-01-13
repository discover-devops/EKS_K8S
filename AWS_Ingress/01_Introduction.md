### Lecture 1: Introduction

Welcome to the AWS EKS Ingress Controller course! In this series, we will deep dive into **Kubernetes Ingress** concepts, leveraging **AWS Load Balancer Controller**, **Application Load Balancer (ALB)**, and **External DNS** for real-world applications.

We will conduct **14 practical demos** to explore and implement advanced AWS EKS features. These demos are designed to give you a comprehensive understanding of managing traffic routing, SSL configurations, DNS integrations, and more in a Kubernetes environment.

---

### **Overview of Demos**

#### **Section 1: AWS Load Balancer Controller Installation and Basics**
1. **Demo 1: Install AWS Load Balancer Controller**
   - Learn how to install and configure the AWS Load Balancer Controller on an EKS cluster.
   - Understand prerequisites, IAM roles, and Helm-based installation.

2. **Demo 2: Ingress Basics**
   - Introduction to Kubernetes Ingress resources.
   - Implement basic routing rules using ALB.

---

#### **Section 2: Advanced Routing with ALB**
3. **Demo 3: Context Path-Based Routing**
   - Route requests based on context paths (e.g., `/app1`, `/app2`).
   - Configure multiple backends for different paths.

4. **Demo 4: Ingress SSL**
   - Secure your application using SSL/TLS with certificates from AWS Certificate Manager (ACM).

5. **Demo 5: Ingress SSL Redirect**
   - Implement automatic HTTP to HTTPS redirection for your application.

---

#### **Section 3: External DNS Integration**
6. **Demo 6: Install External DNS**
   - Learn how to install and configure External DNS on Kubernetes to automate DNS record management in **AWS Route 53**.

7. **Demo 7: Ingress + External DNS**
   - Automate DNS record creation when deploying Kubernetes Ingress manifests.

8. **Demo 8: Kubernetes Service + External DNS**
   - Use External DNS to manage DNS records for Kubernetes `LoadBalancer` or `NodePort` services.

---

#### **Section 4: Advanced Ingress Features**
9. **Demo 9: Name-Based Virtual Host Routing**
   - Route traffic to backend services based on hostnames (e.g., `app1.domain.com`, `app2.domain.com`).

10. **Demo 10: SSL Discovery Host and DLS**
    - Automatically discover SSL certificates from ACM without hardcoding their ARNs.

11. **Demo 11: Ingress Groups**
    - Split large Ingress configurations into smaller manifests and group them with priority ordering.

12. **Demo 12: Ingress Target Types**
    - Explore **Instance Mode** (default) and **IP Mode** configurations for ALB target types.

13. **Demo 13: Ingress Internal ALB**
    - Deploy an internal ALB for applications requiring private access within the VPC.

---

#### **Section 5: Testing and Validation**
14. **Demo 14: Testing with Cold Ports**
    - Deploy and test applications with cold ports to validate Ingress rules and configurations.

---

### **Key Concepts Covered**
- **Kubernetes Ingress**: Manage external access to services in a cluster.
- **AWS Load Balancer Controller**: Enable integration with ALB/ELB.
- **External DNS**: Automate DNS records in Route 53.
- **Routing Strategies**:
  - Context Path-Based
  - Name-Based Virtual Hosting
  - SSL Redirect and Discovery
- **Ingress Groups**: Modular Ingress configurations.
- **Ingress Target Types**: Instance vs. IP mode.

---

### **Next Steps**
In the next Section, we will start with **Demo 1: Installing AWS Load Balancer Controller**. Ensure your AWS account, EKS cluster, and necessary permissions are ready for the hands-on session.

---

**Thank you, and see you in the next Section!**
