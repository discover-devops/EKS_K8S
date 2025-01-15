### **Step-by-Step Guide: NGINX Rolling Update in Kubernetes Deployment**  

---

### **Objective:**  
Perform a rolling update to upgrade the NGINX version in a Kubernetes Deployment without downtime.  

---

### **Prerequisites:**  
- Kubernetes cluster running (Minikube, K3s, or cloud-based).  
- kubectl CLI installed and configured.  

---

---

### **Step 1: Create the Initial NGINX Deployment**  

#### **1. Create the Deployment YAML (nginx-deployment.yaml):**  
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.20  # Initial Version
          ports:
            - containerPort: 80
```

---

#### **2. Deploy the NGINX Deployment:**  
```bash
kubectl apply -f nginx-deployment.yaml
```

---

#### **3. Verify the Deployment and Running Pods:**  
```bash
kubectl get deployments
kubectl get pods
```
- Ensure 3 Pods are running with `nginx:1.20` version.  
- Check detailed status:  
```bash
kubectl describe deployment nginx-deployment
```

---

---

### **Step 2: Perform Rolling Update (Upgrade NGINX Version)**  

---

#### **1. Update the NGINX Image to 1.21:**  
```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.21
```
- This updates the container image **from nginx:1.20 to nginx:1.21**.  

---

#### **2. Verify the Update Process (Rolling Update):**  
```bash
kubectl rollout status deployment/nginx-deployment
```
- Shows the status of the rolling update.  
- Kubernetes gradually terminates old Pods and creates new ones.  

---

#### **3. Check New Pod Versions:**  
```bash
kubectl get pods
kubectl describe pods | grep "Image"
```
- Verify that new Pods use `nginx:1.21`.  

---

---

### **Step 3: Rollback the Deployment (If Update Fails)**  

---

#### **1. Simulate a Failed Update (Optional):**  
```bash
kubectl set image deployment/nginx-deployment nginx=nginx:broken
```
- This sets an invalid image (`nginx:broken`), causing Pods to fail.  

---

#### **2. Check Pod Status:**  
```bash
kubectl get pods
kubectl describe pod <pod-name>
```
- Pods may show `CrashLoopBackOff` or `ImagePullBackOff`.  

---

#### **3. Rollback to the Previous Stable Version:**  
```bash
kubectl rollout undo deployment/nginx-deployment
```
- This reverts the Deployment to the previous image (`nginx:1.21`).  

---

#### **4. Confirm Rollback:**  
```bash
kubectl rollout history deployment/nginx-deployment
kubectl get pods
```
- Verify the Pods are recreated with the stable NGINX version.  

---

---

### **Step 4: Scale the Deployment (Optional)**  
```bash
kubectl scale deployment nginx-deployment --replicas=5
```
- Scale to 5 Pods.  
- Verify:  
```bash
kubectl get pods
```

---

---

### **Step 5: Cleanup (Optional)**  
```bash
kubectl delete deployment nginx-deployment
```
- Removes the Deployment and all related Pods.  

---

---

### **Verification Commands (Optional):**  
```bash
kubectl get replicasets
kubectl get deployments
kubectl describe deployment nginx-deployment
```

---

---

### **Summary of Key Commands:**  
| Command | Description |  
|---------|-------------|  
| `kubectl apply -f <file>.yaml` | Deploy from YAML |  
| `kubectl set image deployment <name> <container>=<image>` | Update container image |  
| `kubectl rollout status deployment <name>` | Monitor rolling update |  
| `kubectl rollout undo deployment <name>` | Rollback deployment |  
| `kubectl scale deployment <name> --replicas=X` | Scale replicas |  

---

