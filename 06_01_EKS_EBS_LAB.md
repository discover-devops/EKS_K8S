# Here's your **complete step-by-step guide** for setting up **Amazon EBS-backed storage on AWS EKS** using the **EBS CSI Driver**, along with **PersistentVolumeClaim (PVC), a test Pod**, and **Helm-based installation options**.

---

# 🛠 Amazon EBS Storage with Kubernetes on EKS — Step-by-Step Guide

---

##  **Part 1: Install Amazon EBS CSI Driver**

### **Step 1.1: Create IAM Policy for EBS CSI Driver**

1. Go to **IAM → Policies → Create Policy**
2. Choose **JSON** tab and paste:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AttachVolume", "ec2:CreateSnapshot", "ec2:CreateTags", "ec2:CreateVolume",
        "ec2:DeleteSnapshot", "ec2:DeleteTags", "ec2:DeleteVolume",
        "ec2:DescribeInstances", "ec2:DescribeSnapshots", "ec2:DescribeTags", "ec2:DescribeVolumes",
        "ec2:DetachVolume"
      ],
      "Resource": "*"
    }
  ]
}
```
3. Click **Next** → **Review**
4. Name: `Amazon_EBS_CSI_Driver`
5. Click **Create Policy**

---

### **Step 1.2: Attach Policy to Node IAM Role**

1. Find Node Role using:
```bash
kubectl -n kube-system describe configmap aws-auth
```
Look for:
```
rolearn: arn:aws:iam::ACCOUNT_ID:role/eksctl-<cluster-name>-nodegroup-NodeInstanceRole-XXXXXXXX
```

2. Go to IAM → Roles → Attach the policy `Amazon_EBS_CSI_Driver` to that role.

---

### **Step 1.3: Deploy EBS CSI Driver**

####  Option 1: Preferred (via **Helm**)  
If `helm` is **not installed**, run:
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod +x get_helm.sh
./get_helm.sh
```

Then:
```bash
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update

helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set enableVolumeScheduling=true \
  --set enableVolumeResizing=true \
  --set enableVolumeSnapshot=true
```

####  Option 2: (via `kubectl apply -k`) — if Git path fails

```bash
git clone https://github.com/kubernetes-sigs/aws-ebs-csi-driver.git
cd aws-ebs-csi-driver/deploy/kubernetes/overlays/stable
kubectl apply -k .
```

 Verify:
```bash
kubectl get pods -n kube-system | grep ebs
```

---

##  **Part 2: Create PVC + Pod Using EBS**

---

### **Step 2.1: Create StorageClass**

 `storageclass.yaml`
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
```

 Apply:
```bash
kubectl apply -f storageclass.yaml
kubectl get storageclass
```

---

### **Step 2.2: Create PVC**

 `pvc.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: ebs-sc
```

 Apply:
```bash
kubectl apply -f pvc.yaml
kubectl get pvc
```
Status should be: `Bound`

---

### **Step 2.3: Deploy Test Pod**

 `test-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: ebs-volume
  volumes:
    - name: ebs-volume
      persistentVolumeClaim:
        claimName: ebs-pvc
```

 Deploy:
```bash
kubectl apply -f test-pod.yaml
kubectl get pods
```

---

### **Step 2.4: Verify Persistence**

Write to the volume:
```bash
kubectl exec -it test-pod -- /bin/sh
echo "Hello from EBS!" > /usr/share/nginx/html/index.html
exit
```

 Delete the Pod:
```bash
kubectl delete pod test-pod
```

 Recreate:
```bash
kubectl apply -f test-pod.yaml
```

 Verify Data:
```bash
kubectl exec -it test-pod -- cat /usr/share/nginx/html/index.html
```
Output:
```
Hello from EBS!
```

---

### **Step 2.5: Cleanup (Optional)**

```bash
kubectl delete pod test-pod
kubectl delete pvc ebs-pvc
kubectl delete storageclass ebs-sc
```

---

##  Summary: Why Use EBS on EKS?

| Feature                         | Value                                                                 |
|---------------------------------|-----------------------------------------------------------------------|
| Persistent storage              | Data survives pod restarts and crashes                               |
| AWS-native integration          | Easy snapshot, backup, and monitoring                                |
| Dynamic provisioning            | Kubernetes will create and bind volumes automatically                |
| Stateful applications           | Best for databases, CMS, file servers, logs                          |
| Volume management with StorageClass | Customize volume types (e.g., gp3, io2) with automation          |

---

