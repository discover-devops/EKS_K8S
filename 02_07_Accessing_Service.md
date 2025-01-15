

To access the service externally via `NodePort`, you need to use the **Node's external IP** along with the `nodePort` (30080).  

---

### **Steps to Access the NGINX Service Externally:**  

1. **Find the Node IP (External IP):**  
```bash
kubectl get nodes -o wide
```
- Look under the **EXTERNAL-IP** column.  
- If no external IP is assigned, use the **INTERNAL-IP** (if accessing from inside the node).  

---

2. **Access NGINX Using curl:**  
```bash
curl http://<NodeIP>:30080
```
- Replace `<NodeIP>` with the IP from step 1.  
- Example:  
```bash
curl http://192.168.1.10:30080
```

---

### **If Node Has No External IP (Minikube or Bare Metal):**  
- **Option 1:** Use the node's internal IP (from `kubectl get nodes -o wide`).  
- **Option 2:** Use `minikube service` (if using Minikube):  
   ```bash
   minikube service nginx-service
   ```
   This opens the service in the default web browser.  

- **Option 3:** Port forward the service:  
   ```bash
   kubectl port-forward svc/nginx-service 8080:80
   curl http://localhost:8080
   ```

---

### **Verify the Service Status (Optional):**  
```bash
kubectl get svc nginx-service
```
- Ensure `nodePort` (30080) is listed correctly.  

---

### **Common Issues:**  
- **Timeout or No Response?**  
  - Ensure Pods are running:  
    ```bash
    kubectl get pods
    ```
  - Check for pod errors or crashes:  
    ```bash
    kubectl describe pod <pod-name>
    ```
  - Validate service is mapped correctly:  
    ```bash
    kubectl describe svc nginx-service
    ```


