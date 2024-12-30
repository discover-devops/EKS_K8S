# EKS_K8S

### **Course Title: Mastering Kubernetes on AWS EKS – Theory and Hands-On Labs**  
**Target Audience:** Cloud Architects, DevOps Engineers, System Administrators, Developers  
**Level:** Intermediate to Advanced  

---

## **Course Objectives:**  
- Understand Kubernetes fundamentals and its architecture.  
- Learn how to deploy and manage containerized applications on AWS EKS.  
- Implement scaling, networking, and security best practices.  
- Gain hands-on experience with EKS cluster provisioning, deployment, and troubleshooting.  
- Master real-world use cases for enterprise applications on AWS EKS.  

---

## **Course Outline:**  

---

### **Session 1: Introduction to Kubernetes and AWS EKS**  
**Theory:**  
- Overview of Kubernetes: Why Kubernetes?  
- Kubernetes Architecture: Nodes, Pods, Deployments, Services  
- Introduction to AWS EKS:  
  - What is EKS?  
  - EKS Architecture  
  - EKS vs Self-Managed Kubernetes  
- Key AWS Services for Kubernetes (IAM, VPC, EC2, S3, ELB)  

**Lab:**  
- **Provisioning an EKS Cluster using AWS Console**  
  - Creating IAM roles for EKS  
  - Creating an EKS cluster  
  - Verifying EKS cluster setup using kubectl  

---

### **Session 2: Deploying Applications on EKS**  
**Theory:**  
- Kubernetes Objects and YAML Basics  
- Deployments and ReplicaSets  
- Services: ClusterIP, NodePort, LoadBalancer  
- ConfigMaps and Secrets  

**Lab:**  
- **Deploying an NGINX Application on EKS**  
  - Writing Deployment YAML files  
  - Exposing application with LoadBalancer service  
  - Verifying deployment using kubectl  

---

### **Session 3: Scaling and Auto-Healing**  
**Theory:**  
- Horizontal Pod Autoscaler (HPA)  
- Vertical Pod Autoscaler  
- Cluster Autoscaler on AWS EKS  
- Pod Disruption Budgets  

**Lab:**  
- **Implementing Horizontal Pod Autoscaler**  
  - Deploying a sample application  
  - Configuring HPA  
  - Simulating traffic and monitoring pod scaling  

---

### **Session 4: Networking and Security**  
**Theory:**  
- Kubernetes Networking Model  
- AWS VPC CNI Plugin for EKS  
- Network Policies  
- IAM Roles for Service Accounts (IRSA)  
- Pod Security Policies and RBAC  

**Lab:**  
- **Configuring IAM Roles for Pods**  
  - Enabling IRSA on EKS  
  - Creating service account and attaching IAM policies  
  - Verifying pod access to AWS services (e.g., S3)  

---

### **Session 5: Monitoring, Logging, and Troubleshooting**  
**Theory:**  
- Monitoring EKS with CloudWatch  
- Kubernetes Logging with FluentD and CloudWatch  
- Troubleshooting Pods and Nodes  
- Debugging Kubernetes Applications  

**Lab:**  
- **Monitoring and Troubleshooting EKS**  
  - Integrating CloudWatch with EKS  
  - Viewing application logs  
  - Debugging failed pods and node issues  

---

## **Capstone Project (Optional)**  
- **Deploy a full-stack application with multiple services on AWS EKS**  
- Configure Load Balancer, HPA, and IAM integration  
- Implement best practices for security and scaling  

---

## **Prerequisites:**  
- Basic understanding of Kubernetes  
- Familiarity with AWS services (EC2, VPC, IAM)  
- Experience with Docker and containerization  

---

## **Outcomes:**  
- Deploy and manage containerized applications on EKS confidently  
- Implement auto-scaling, networking, and security best practices  
- Troubleshoot and monitor Kubernetes clusters effectively  
- Build production-grade Kubernetes applications on AWS  

---

