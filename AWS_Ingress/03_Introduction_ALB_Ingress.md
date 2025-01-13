### Lecture 3: Installing AWS Load Balancer Controller on AWS EKS

Welcome to the practical session where we will install the **AWS Load Balancer Controller** on an EKS cluster. In this lecture, we will explore how the AWS Load Balancer Controller integrates with EKS to automate the creation of Application Load Balancers (ALBs) and Network Load Balancers (NLBs). 

---

### **Why AWS Load Balancer Controller?**
The **AWS Load Balancer Controller** is a significant upgrade from the older ALB Ingress Controller, supporting:
1. **ALB and NLB Creation**:
   - ALB: Managed via Kubernetes Ingress resources.
   - NLB: Managed via Kubernetes Service annotations.
2. **Latest Kubernetes API Support**:
   - Supports `networking.k8s.io/v1` API introduced in Kubernetes 1.22.
3. **Deprecation of ALB Ingress Controller**:
   - No future development for the old controller; the new controller is more robust and feature-rich.

---

### **Key Components Created by AWS Load Balancer Controller**
1. **IAM Role and Policy**:
   - Grants permissions to create, update, and delete ALBs/NLBs.
2. **Service Account**:
   - Annotated with the IAM role for controller pods to access AWS resources.
3. **Controller Deployment**:
   - Creates and manages ALB/NLB resources in AWS.
4. **Webhook Service**:
   - Validates and mutates Ingress resources.
5. **TLS Secret**:
   - Manages SSL/TLS configurations for the load balancers.

---

### **Architecture Overview**

1. **Kubernetes Components**:
   - `Service Account`: Links the Kubernetes controller with AWS IAM.
   - `Deployment`: Manages controller pods for creating AWS resources.
   - `Ingress Resource`: Defines routing rules for ALB.
   
2. **AWS Components**:
   - **ALB/NLB**: Provisioned and managed based on Kubernetes manifests.
   - **Target Groups**: Defined for NodePort or pod IP targets.
   - **IAM Role**: Authorizes the controller to interact with AWS resources.

---

### **Installation Steps**

#### **Step 1: Prerequisites**
- **AWS CLI** and **kubectl** installed.
- EKS cluster up and running.
- IAM permissions for creating policies and roles.

#### **Step 2: Create IAM Policy**
Create a policy for the controller to interact with AWS resources.

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

> Replace `iam_policy.json` with the official policy file from the [AWS documentation](https://github.com/kubernetes-sigs/aws-load-balancer-controller).

#### **Step 3: Create IAM Role and Service Account**
Associate the IAM policy with an IAM role and link it to the Kubernetes service account.

```bash
eksctl create iamserviceaccount \
  --cluster <cluster-name> \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

#### **Step 4: Install the Load Balancer Controller**
Use **Helm** to install the controller.

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<aws-region>
```

#### **Step 5: Verify the Installation**
Check that the controller pods are running.

```bash
kubectl get pods -n kube-system
```

Expected output:
```plaintext
NAME                                        READY   STATUS    RESTARTS   AGE
aws-load-balancer-controller-xxxxx          1/1     Running   0          xxm
```

---

### **How It Works**
1. **Ingress Events**:
   - The controller listens for Ingress resource events.
2. **AWS Resource Creation**:
   - Creates ALBs or NLBs based on annotations in the manifests.
3. **Role of Service Account**:
   - Service account pods use IAM roles to interact with AWS.

---

### **Key Annotations**
- **ALB Creation**: `kubernetes.io/ingress.class: alb`
- **NLB Creation**: `service.beta.kubernetes.io/aws-load-balancer-type: nlb`
- **Target Types**:
  - Instance mode: `alb.ingress.kubernetes.io/target-type: instance`
  - IP mode: `alb.ingress.kubernetes.io/target-type: ip`

---

### **Next Steps**
In the next lecture, we will deploy a sample application and configure basic ingress to understand the practical use of the AWS Load Balancer Controller.

---

**Thank you, and see you in the next session!**
