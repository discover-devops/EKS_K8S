### **Part-2: PersistentVolume (PV) and PersistentVolumeClaim (PVC) with EBS on EKS**

---

### **Objective:**
In this part, we will:
1. Create an **EBS-backed PersistentVolume (PV)**.  
2. Create a **PersistentVolumeClaim (PVC)** to bind the PV.  
3. Deploy a test Pod to verify the EBS storage.  

---

---

### **Step 1: Create a StorageClass for EBS**

A **StorageClass** defines how storage is dynamically provisioned in Kubernetes.

1. **Create a StorageClass YAML file (`storageclass.yaml`):**
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

2. **Apply the StorageClass:**
```bash
kubectl apply -f storageclass.yaml
```

3. **Verify the StorageClass:**
```bash
kubectl get storageclass
```

---

---

### **Step 2: Create a PersistentVolumeClaim (PVC)**

A PVC allows a Pod to request storage dynamically.

1. **Create a PVC YAML file (`pvc.yaml`):**
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

2. **Apply the PVC:**
```bash
kubectl apply -f pvc.yaml
```

3. **Verify the PVC:**
```bash
kubectl get pvc
```
- The status should show as **`Bound`**.

---

---

### **Step 3: Deploy a Test Pod with PVC**

1. **Create a Pod YAML file (`test-pod.yaml`):**
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

2. **Deploy the Test Pod:**
```bash
kubectl apply -f test-pod.yaml
```

3. **Verify the Pod:**
```bash
kubectl get pods
```
- Ensure the Pod is running.

---

---

### **Step 4: Test EBS Storage**

1. **Write Data to the EBS Volume:**
```bash
kubectl exec -it test-pod -- /bin/sh
echo "Hello from EBS!" > /usr/share/nginx/html/index.html
exit
```

2. **Verify Data Persistence:**
   - Delete the Pod:
   ```bash
   kubectl delete pod test-pod
   ```

   - Recreate the Pod:
   ```bash
   kubectl apply -f test-pod.yaml
   ```

   - Check the Data:
   ```bash
   kubectl exec -it test-pod -- cat /usr/share/nginx/html/index.html
   ```
   - You should see: **`Hello from EBS!`**

---

---

### **Step 5: Cleanup (Optional)**

1. **Delete the Test Pod:**
```bash
kubectl delete pod test-pod
```

2. **Delete the PVC:**
```bash
kubectl delete pvc ebs-pvc
```

3. **Delete the StorageClass:**
```bash
kubectl delete storageclass ebs-sc
```

---

---

### **Key Commands Summary**

| **Task**                              | **Command**                                    |
|---------------------------------------|------------------------------------------------|
| **Create StorageClass**               | `kubectl apply -f storageclass.yaml`           |
| **Create PVC**                        | `kubectl apply -f pvc.yaml`                    |
| **Deploy Test Pod**                   | `kubectl apply -f test-pod.yaml`               |
| **Verify PVC Binding**                | `kubectl get pvc`                              |
| **Write Data to EBS**                 | `kubectl exec -it test-pod -- /bin/sh`         |
| **Check Data Persistence**            | `kubectl exec -it test-pod -- cat <file-path>` |

---

### **Conclusion**

- This setup demonstrates how to configure and use **EBS-backed persistent storage** in Kubernetes.  
- EBS storage persists data across Pod restarts, making it ideal for stateful applications like databases or file storage.  



## In case the GITREPO didn't work

This error indicates an issue with the specified path or Kustomize overlay structure in the AWS EBS CSI Driver repository. The `kustomize` tool is unable to find the expected resources in the specified location.

---

### **Solution: Use Helm to Install AWS EBS CSI Driver (Recommended)**

Helm is a more reliable and straightforward way to deploy the AWS EBS CSI Driver.

---

#### **1. Add the Helm Repository**
```bash
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update
```

---

#### **2. Install the AWS EBS CSI Driver**
```bash
helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set enableVolumeScheduling=true \
  --set enableVolumeResizing=true \
  --set enableVolumeSnapshot=true
```

---

#### **3. Verify Installation**
```bash
kubectl get pods -n kube-system | grep ebs
```
Ensure the EBS CSI Controller pods are running successfully.

---

### **Solution: Manually Clone and Apply the Kustomize Files**

If you prefer using `kubectl apply -k`, the files must be manually cloned to resolve path issues.

---

#### **1. Clone the Repository Locally**
```bash
git clone https://github.com/kubernetes-sigs/aws-ebs-csi-driver.git
cd aws-ebs-csi-driver/deploy/kubernetes/overlays/stable
```

---

#### **2. Apply the Overlays Locally**
```bash
kubectl apply -k .
```

---

#### **3. Verify the Deployment**
```bash
kubectl get pods -n kube-system | grep ebs
```

---

### **Troubleshooting Tips**

1. **Check Kustomize Version:** Ensure you have a recent version of `kubectl` or `kustomize`.
   ```bash
   kubectl version --client
   kustomize version
   ```

2. **Inspect Directory Structure:** If errors persist, navigate into the local repository and confirm the directory structure.

3. **Switch to Stable Versions:** Use specific tags or branches to avoid using an unstable version of the EBS CSI Driver.


### In case HELM not  installed

The error `-bash: helm: command not found` indicates that Helm (a package manager for Kubernetes) is not installed on your system. Follow these steps to install Helm and proceed with the deployment of the AWS EBS CSI Driver.

---

### **Step 1: Install Helm**

#### **1.1. Download the Helm Binary**
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
```

#### **1.2. Make the Script Executable**
```bash
chmod +x get_helm.sh
```

#### **1.3. Run the Script**
```bash
./get_helm.sh
```

This will download and install the latest version of Helm.

---

#### **1.4. Verify the Helm Installation**
```bash
helm version
```
You should see output like this:
```text
version.BuildInfo{Version:"v3.x.x", GitCommit:"xxxx", GitTreeState:"clean", GoVersion:"go1.x.x"}
```

---

### **Step 2: Add the AWS EBS CSI Driver Helm Repository**

After Helm is installed, proceed with the following:

```bash
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update
```

---

### **Step 3: Install the AWS EBS CSI Driver**

Use Helm to install the EBS CSI Driver:

```bash
helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set enableVolumeScheduling=true \
  --set enableVolumeResizing=true \
  --set enableVolumeSnapshot=true
```

---

### **Step 4: Verify Installation**

Check that the EBS CSI Driver pods are running:

```bash
kubectl get pods -n kube-system | grep ebs
```

---

### **Alternative: Install Helm Using a Package Manager**

If you prefer, you can use a package manager to install Helm:

- **On macOS:**
  ```bash
  brew install helm
  ```

- **On Ubuntu/Debian:**
  ```bash
  sudo apt-get update
  sudo snap install helm --classic
  ```

- **On CentOS/RHEL:**
  ```bash
  sudo yum install helm
  ```

---

