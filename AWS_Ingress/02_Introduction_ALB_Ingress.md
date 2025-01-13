### Lecture 2: Ingress Introduction Part 2

In this lecture, we will explore the **AWS Application Load Balancer (ALB)**, its integration with Kubernetes through the **ALB Ingress Controller (ALB IC)**, and understand its features, modes, and workflows at a high level.

---

### **What is AWS Application Load Balancer (ALB)?**

The **Application Load Balancer (ALB)** is a modern, highly capable load balancer in AWS. It provides advanced features such as:
1. **Path-Based Routing**:
   - Routes traffic based on URL paths (e.g., `/app1`, `/app2`).
   - Multiple applications can share a single ALB by leveraging context paths.

2. **Host-Based Routing**:
   - Routes traffic based on hostnames (e.g., `app1.domain.com`, `app2.domain.com`).
   - Supports routing to different applications or services based on hostnames.

3. **Advanced HTTP Routing**:
   - Routes traffic based on HTTP headers, methods, query parameters, and source IP addresses.
   - Acts as a sophisticated HTTP routing solution.

4. **Redirection and Custom Responses**:
   - Redirects requests from one URL to another.
   - Returns custom HTTP responses.

5. **Integration with AWS Lambda**:
   - ALB can register Lambda functions as targets.

6. **Authentication**:
   - Out-of-the-box user authentication using corporate or social identities.

7. **Support for Containerized Applications**:
   - Native integration with AWS ECS and Kubernetes.

8. **Health Checks**:
   - Monitors the health of each target independently.

9. **IP-Based Target Registration**:
   - Registers targets using IP addresses, making it ideal for EKS and Fargate deployments.

---

### **ALB Ingress Controller (ALB IC)**

The **ALB Ingress Controller** integrates Kubernetes Ingress resources with ALB, automating the creation and management of AWS resources for Kubernetes workloads.

#### **Key Features**
1. **Automated Resource Creation**:
   - Provisions ALBs, target groups, listeners, and routing rules based on Kubernetes Ingress manifests.
   
2. **Traffic Modes**:
   - **Instance Mode**:
     - Registers EKS worker nodes as targets.
     - Routes traffic through NodePort services.
   - **IP Mode**:
     - Registers individual pod IPs as targets.
     - Directly routes traffic to pods, bypassing NodePort services.
     - Mandatory for **Fargate-only clusters**.

3. **Annotations**:
   - The controller uses annotations like `kubernetes.io/ingress.class: ALB` to identify and manage Ingress resources.

---

### **Instance Mode vs. IP Mode**

| **Feature**              | **Instance Mode**                | **IP Mode**                     |
|--------------------------|----------------------------------|---------------------------------|
| **Targets**              | EKS worker nodes                | Pods' IPs                      |
| **NodePort Service**     | Required                        | Not required                   |
| **Fargate Support**      | Not supported                   | Supported                      |
| **Performance**          | Slightly less efficient         | More efficient (direct routing) |

- Use **Instance Mode** for clusters with managed node groups.
- Use **IP Mode** for **Fargate-only** clusters.

---

### **How ALB Ingress Works**
Ref: https://aws.amazon.com/blogs/opensource/kubernetes-ingress-aws-alb-ingress-controller/

![image](https://github.com/user-attachments/assets/e56473f7-db7f-4747-a91b-a1fa3918209d)



The **ALB Ingress Controller** works as follows:
1. **Ingress Event Detection**:
   - The controller watches the Kubernetes API for Ingress resources.

2. **AWS Resource Provisioning**:
   - Creates an ALB (ELB v2) when a compatible Ingress resource is detected.

3. **Target Groups**:
   - Creates target groups for each Kubernetes service in the Ingress manifest.
   - Supports both NodePort and IP-based services.

4. **Listeners**:
   - Configures listeners for specified ports (e.g., HTTP 80, HTTPS 443).

5. **Routing Rules**:
   - Adds rules for context paths (e.g., `/app1`, `/products`) or hostnames (e.g., `app1.domain.com`).

6. **Cleanup**:
   - Deleting the Ingress resource removes all associated AWS resources.

---

### **Key Benefits**
- **Scalability**: Handles large-scale traffic distribution efficiently.
- **Flexibility**: Supports a wide range of routing and authentication features.
- **Integration**: Seamlessly integrates Kubernetes workloads with AWS services.

---

### **Next Steps**
In the next section, we will delve into **Demo 1: Installing the ALB Ingress Controller**, covering prerequisites, installation steps, and verification.

---

**Thank you, and see you in the next session!**
