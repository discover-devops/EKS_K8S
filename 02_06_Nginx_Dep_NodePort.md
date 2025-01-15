### Example to create an **NGINX Deployment** with a **NodePort Service** to expose it externally.

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

### **2. NodePort Service (svc-nginx-nodeport.yaml):**  
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80         # Service Port
      targetPort: 80   # Pod Port
      nodePort: 30080  # Exposes the service on Node's port 30080
```

---

### **Explanation:**  
#### **Deployment:**  
- **3 NGINX Pods** will be created (`replicas: 3`).  
- Each Pod listens on port `80`.  
- The Pods are labeled `app: nginx` for easy Service selection.  

#### **NodePort Service:**  
- **`type: NodePort`** – Exposes the service externally on the node's IP address.  
- **`nodePort: 30080`** – Access NGINX by visiting `<NodeIP>:30080`.  
- **`port: 80`** – Service listens internally on port 80.  
- **`targetPort: 80`** – Routes traffic to Pod's port 80.  

---

### **Deploying the YAML Files:**  
1. **Apply the Deployment:**  
   ```bash
   kubectl apply -f deployment-nginx.yaml
   ```  
2. **Apply the NodePort Service:**  
   ```bash
   kubectl apply -f svc-nginx-nodeport.yaml
   ```  
3. **Verify Deployment and Service:**  
   ```bash
   kubectl get deployments
   kubectl get pods
   kubectl get svc
   ```

---

### **Accessing the NGINX Service (Externally):**  
```bash
curl http://<NodeIP>:30080
```
- **NodeIP** – Can be retrieved by running:  
  ```bash
  kubectl get nodes -o wide
  ```

---

### **Testing and Troubleshooting:**  
- If the port is already in use, change `nodePort` to another value between **30000-32767**.  
- **Pod Logs (if needed):**  
  ```bash
  kubectl logs <pod-name>
  ```

