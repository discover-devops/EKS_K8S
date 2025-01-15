### Example **NGINX Deployment** and a **ClusterIP Service** to expose it within the Kubernetes cluster.

---

### **1. NGINX Deployment (deployment-nginx.yaml):**  
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3  # Deploy 3 replicas of the Pod
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
          image: nginx:latest
          ports:
            - containerPort: 80
```

---

### **Explanation (Deployment):**  
- **`replicas: 3`** – Creates 3 Pods running NGINX.  
- **`selector.matchLabels`** – Ensures the Deployment manages Pods with the label `app: nginx`.  
- **`template`** – Describes the Pod configuration:  
  - **`containers`** – Runs the `nginx:latest` image and exposes port `80`.  

---

### **2. ClusterIP Service (svc-nginx.yaml):**  
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80  # Service Port
      targetPort: 80  # Pod Port
```

---

### **Explanation (ClusterIP Service):**  
- **`port: 80`** – The port exposed on the service (clients access this).  
- **`targetPort: 80`** – The port on the Pod the service forwards traffic to.  
- **`selector`** – Selects Pods with the label `app: nginx`, ensuring traffic routes to NGINX Pods.  
- **`ClusterIP` (default)** – Exposes the service internally within the cluster (no external access).  

---

### **Deploying the YAML Files:**  
1. **Apply the Deployment:**  
   ```bash
   kubectl apply -f deployment-nginx.yaml
   ```  
2. **Apply the Service:**  
   ```bash
   kubectl apply -f svc-nginx.yaml
   ```  
3. **Verify Deployment and Service:**  
   ```bash
   kubectl get deployments
   kubectl get pods
   kubectl get svc
   ```

---

### **Testing the ClusterIP Service:**  
1. **Access from inside the cluster (e.g., another Pod):**  
   ```bash
   kubectl run test-pod --rm -it --image=busybox -- /bin/sh
   wget nginx-service
   ```
   - This should fetch the NGINX welcome page.  

2. **If External Access is Needed:**  
   Change the Service type to **NodePort** or **LoadBalancer**:  
   ```yaml
   spec:
     type: NodePort
   ```

---

