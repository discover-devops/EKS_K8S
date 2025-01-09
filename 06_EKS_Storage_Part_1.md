### **Part-1: Creation of EBS CSI Driver for Kubernetes Storage on EKS**

---

### **Introduction**

In this part, we will:
1. Create an **IAM Policy** for the Amazon EBS CSI Driver.  
2. Attach the IAM Policy to the Worker Node IAM Role.  
3. Deploy the Amazon EBS CSI Driver in the EKS Cluster.  

---

---

### **Step 1: Create IAM Policy for EBS CSI Driver**

1. **Navigate to IAM in the AWS Management Console:**
   - Go to **Services → IAM → Policies → Create Policy**.  

2. **Switch to the JSON Tab** and paste the following JSON:  
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AttachVolume",
        "ec2:CreateSnapshot",
        "ec2:CreateTags",
        "ec2:CreateVolume",
        "ec2:DeleteSnapshot",
        "ec2:DeleteTags",
        "ec2:DeleteVolume",
        "ec2:DescribeInstances",
        "ec2:DescribeSnapshots",
        "ec2:DescribeTags",
        "ec2:DescribeVolumes",
        "ec2:DetachVolume"
      ],
      "Resource": "*"
    }
  ]
}
```

3. **Review the Policy in Visual Editor:**
   - Click **Next → Tags → Review**.  

4. **Provide a Name and Description:**
   - Name: `Amazon_EBS_CSI_Driver`  
   - Description: `Policy for EC2 Instances to access Elastic Block Store`.  

5. **Create the Policy:**
   - Click **Create Policy**.  

---

---

### **Step 2: Attach IAM Policy to the Worker Node IAM Role**

1. **Identify the IAM Role Associated with Worker Nodes:**
   - Run the following command to find the role ARN:  
     ```bash
     kubectl -n kube-system describe configmap aws-auth
     ```
   - Look for the `rolearn` field in the output:  
     ```text
     rolearn: arn:aws:iam::123456789012:role/eksctl-your-cluster-name-nodegroup-NodeInstanceRole-XXXXXXXXXXXX
     ```

2. **Attach the Policy to the IAM Role:**
   - Go to **Services → IAM → Roles**.  
   - Search for the role identified in the previous step.  
   - Click on the **Role Name** to open its details.  
   - Navigate to the **Permissions** tab and click **Attach Policies**.  
   - Search for `Amazon_EBS_CSI_Driver` and attach it.  

---

---

### **Step 3: Deploy the Amazon EBS CSI Driver**

1. **Ensure kubectl is Installed and Updated:**
   - Verify the kubectl version (should be 1.14 or later):  
     ```bash
     kubectl version --client --short
     ```

---

2. **Deploy the EBS CSI Driver:**
   - Use `kubectl apply` to deploy the EBS CSI driver:  
     ```bash
     kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"
     ```

---

3. **Verify the Deployment:**
   - Check that the EBS CSI pods are running in the `kube-system` namespace:  
     ```bash
     kubectl get pods -n kube-system | grep ebs
     ```

---

---

### **Summary of Commands**

| **Task**                           | **Command**                                                                 |
|------------------------------------|-----------------------------------------------------------------------------|
| **Find Worker Node IAM Role**      | `kubectl -n kube-system describe configmap aws-auth`                        |
| **Attach IAM Policy**              | Attach `Amazon_EBS_CSI_Driver` policy to the identified IAM Role via AWS Console |
| **Deploy EBS CSI Driver**          | `kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"` |
| **Verify Deployment**              | `kubectl get pods -n kube-system | grep ebs`                               |

---

