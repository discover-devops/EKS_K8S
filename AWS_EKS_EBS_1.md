
Below is a **Step-by-Step Guide: Installing AWS EBS CSI Driver on EKS (Helm + IRSA)** 

---

# Lab: Installing AWS EBS CSI Driver on an EKS Cluster

## 0. Lab Overview

In this lab we will:

1. Verify the EKS cluster and region
2. Enable IAM OIDC provider for the cluster
3. Create an IAM Policy for the EBS CSI driver
4. Create an IAM Role + Kubernetes ServiceAccount (IRSA)
5. Add the AWS EBS CSI Driver Helm repository
6. Install the EBS CSI Driver using Helm
7. Verify that the driver is running correctly
8. (Optional) Force controller pods to restart after policy changes

---

## 1. Prerequisites

* An existing **EKS cluster** (here: `my-lab-cluster` in `ap-south-1`)
* **kubectl** configured to talk to the cluster
* **AWS CLI** installed and configured with your account
* **eksctl** installed
* **Helm** installed

You should be able to run:

```bash
aws sts get-caller-identity
kubectl get nodes
eksctl version
helm version
```

---

## 2. Verify Cluster and Region

First confirm the cluster’s OIDC issuer and region:

```bash
aws eks describe-cluster \
  --name my-lab-cluster \
  --region ap-south-1 \
  --query "cluster.identity.oidc.issuer"
```

You should see something like:

```text
"https://oidc.eks.ap-south-1.amazonaws.com/id/7E0C381D5A4863463F863FC4311004A4"
```

---

## 3. Associate IAM OIDC Provider with the Cluster

This lets Kubernetes service accounts assume IAM roles (IRSA).

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster my-lab-cluster \
  --region ap-south-1 \
  --approve
```

Expected output (similar):

```text
[ℹ]  will create IAM Open ID Connect provider for cluster "my-lab-cluster" in "ap-south-1"
[✔]  created IAM Open ID Connect provider for cluster "my-lab-cluster" in "ap-south-1"
```

---

## 4. Create IAM Policy for EBS CSI Driver

### 4.1. IAM Policy JSON

Create a file `AmazonEKS_EBS_CSI_DriverRole.json` with the following content:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateSnapshot",
        "ec2:DeleteSnapshot",
        "ec2:CreateTags",
        "ec2:DeleteTags",
        "ec2:CreateVolume",
        "ec2:DeleteVolume",
        "ec2:AttachVolume",
        "ec2:DetachVolume",
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeInstances",
        "ec2:DescribeSnapshots",
        "ec2:DescribeTags",
        "ec2:DescribeVolumes",
        "ec2:DescribeVolumesModifications",
        "ec2:ModifyVolume"
      ],
      "Resource": "*"
    }
  ]
}
```

### 4.2. Create the Policy

```bash
aws iam create-policy \
  --policy-name AmazonEKS_EBS_CSI_DriverRole \
  --policy-document file://AmazonEKS_EBS_CSI_DriverRole.json
```

Note the returned **Policy ARN**, it will look like:

```text
arn:aws:iam::<ACCOUNT_ID>:policy/AmazonEKS_EBS_CSI_DriverRole
```

(Use your real Account ID.)

---

## 5. Create IAM Role + Kubernetes ServiceAccount (IRSA)

Now we create a **Kubernetes ServiceAccount** `ebs-csi-controller-sa` in `kube-system` namespace, and attach the IAM policy using `eksctl`.

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster my-lab-cluster \
  --region ap-south-1 \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AmazonEKS_EBS_CSI_DriverRole \
  --approve \
  --override-existing-serviceaccounts \
  --role-name AmazonEKS_EBS_CSI_DriverRole
```

Key points:

* `--override-existing-serviceaccounts` makes sure metadata (IRSA annotation) is updated even if the SA already exists.
* This command creates:

  * IAM Role: **AmazonEKS_EBS_CSI_DriverRole**
  * Kubernetes SA: **kube-system/ebs-csi-controller-sa**
  * IRSA trust relationship between them.

### 5.1. Verify ServiceAccount

```bash
kubectl get sa ebs-csi-controller-sa -n kube-system -o yaml
```

You should see an annotation like:

```yaml
annotations:
  eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/AmazonEKS_EBS_CSI_DriverRole
```

---

## 6. Add the AWS EBS CSI Driver Helm Repository

```bash
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update
```

Output:

```text
"aws-ebs-csi-driver" has been added to your repositories
...Successfully got an update from the "aws-ebs-csi-driver" chart repository
Update Complete. ⎈Happy Helming!⎈
```

---

## 7. Install the EBS CSI Driver using Helm

We tell Helm **not** to create its own service account and instead use the one we created (`ebs-csi-controller-sa`).

```bash
helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.create=false \
  --set controller.serviceAccount.name=ebs-csi-controller-sa
```

You should see:

```text
NAME: aws-ebs-csi-driver
NAMESPACE: kube-system
STATUS: deployed
DESCRIPTION: Install complete
```

---

## 8. Verify the EBS CSI Pods

Check that controller and node pods are running:

```bash
kubectl get pods -n kube-system | grep ebs
```

Final healthy state should look like:

```text
ebs-csi-controller-7b8b68576c-mtxxv   5/5     Running   0   60s
ebs-csi-controller-7b8b68576c-zd6vr   5/5     Running   0   59s
ebs-csi-node-f2dj4                    3/3     Running   0   19m
ebs-csi-node-w5dwn                    3/3     Running   0   19m
```

* **Controller pods**: 5/5 containers running
* **Node DaemonSet pods**: 3/3 containers running

This means the EBS CSI Driver is installed and ready.

---

## 9. (Optional) Restart Controller Pods After Policy Changes

If you ever modify the IAM policy attached to the role, you can force the controller pods to restart so they re-pick the updated permissions:

```bash
kubectl delete pod -n kube-system -l app=ebs-csi-controller
```

Kubernetes will automatically recreate them. Re-check:

```bash
kubectl get pods -n kube-system | grep ebs
```

Wait until they are again **5/5 Running**.

---

## 10. You Should Understand

By the end of this lab, you should be able to explain:

1. **Why** we need a CSI driver for EBS (to dynamically provision EBS volumes in EKS).
2. **What IRSA is** and why we used an IAM OIDC provider + IAM role + ServiceAccount.
3. **How Helm is used** to deploy the EBS CSI driver.
4. How to verify that **controller** and **node** components of the driver are healthy.

---


3: NOw I want to show how to dynamically provission AWS EBS for AWS EKS cluster

