### **Scenarios When a Pod Gets Restarted in Kubernetes:**  

Kubernetes automatically restarts Pods under various circumstances to ensure application availability and resiliency. Below are the key scenarios:  

---

### **1. Pod Container Crashes (CrashLoopBackOff):**  
- **Scenario:** If the application inside the container crashes due to an error (e.g., segfault, out-of-memory, etc.).  
- **Behavior:**  
  - Kubernetes restarts the container based on the **restartPolicy**.  
  - If the issue persists, the Pod enters a `CrashLoopBackOff` state.  

**Example:**  
```yaml
restartPolicy: Always
```
- **`Always` (default):** Restart container if it fails.  
- **`OnFailure`:** Restart only if the container exits with a non-zero status.  
- **`Never`:** Never restart the container (used for batch jobs).  

---

### **2. Node Failure / Reboot:**  
- **Scenario:** If the Kubernetes node hosting the Pod fails or reboots.  
- **Behavior:**  
  - Kubernetes reschedules the Pod to a different healthy node.  
  - Pod restarts automatically once the node becomes available or the Pod is placed on another node.  

---

### **3. OOMKill (Out of Memory Kill):**  
- **Scenario:** If the container exceeds its allocated memory limit.  
- **Behavior:**  
  - The node's kernel (via **OOM Killer**) terminates the container.  
  - Kubernetes restarts the Pod to recover the service.  

**Example – Memory Limit Exceeded:**  
```yaml
resources:
  limits:
    memory: "128Mi"
```

---

### **4. Liveness Probe Failure:**  
- **Scenario:** If a liveness probe fails, Kubernetes assumes the application is unhealthy and restarts the container.  
- **Behavior:**  
  - Kubernetes kills and restarts the Pod to recover the application.  

**Example – Liveness Probe:**  
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
```

---

### **5. Manual Deletion of Pods (Controlled by ReplicaSet/Deployment):**  
- **Scenario:** If a Pod is manually deleted (`kubectl delete pod`).  
- **Behavior:**  
  - **ReplicaSet or Deployment** recreates the Pod automatically to maintain the desired replica count.  

---

### **6. Deployment Rolling Updates:**  
- **Scenario:** When a **new image version** is deployed (rolling update).  
- **Behavior:**  
  - Kubernetes gradually terminates old Pods and starts new ones to avoid downtime.  

**Example – Update NGINX Image:**  
```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.21
```

---

### **7. Eviction Due to Resource Pressure:**  
- **Scenario:** If the node experiences **CPU/memory pressure**, Kubernetes evicts low-priority Pods to free resources.  
- **Behavior:**  
  - Evicted Pods are rescheduled and restarted on another node.  

---

### **8. PodSpec Changes (Configuration Updates):**  
- **Scenario:** If a Pod's YAML configuration (labels, environment variables, etc.) is updated.  
- **Behavior:**  
  - Kubernetes terminates and restarts the Pod to apply changes.  
  - **Example:** Changing environment variables or configMap values triggers a restart.  

---

### **9. Image Pull Failure:**  
- **Scenario:** If Kubernetes fails to pull the container image from the registry.  
- **Behavior:**  
  - Kubernetes retries the image pull and restarts the Pod after pulling the image successfully.  

---

### **10. Node Drain (During Maintenance):**  
- **Scenario:** If a node is drained for maintenance (`kubectl drain <node>`).  
- **Behavior:**  
  - Pods running on the drained node are terminated and rescheduled to other available nodes.  

---

### **11. Kubernetes Component Restarts:**  
- **Scenario:** If kubelet, kube-proxy, or other critical components restart.  
- **Behavior:**  
  - This can cause Pods to restart or reschedule based on health checks.  

---

### **12. Network Partition (Node Isolation):**  
- **Scenario:** If a node is isolated from the cluster due to network issues.  
- **Behavior:**  
  - Kubernetes marks Pods on the isolated node as `Not Ready` and reschedules them on other nodes.  

---

### **How to Monitor Pod Restarts:**  
```bash
kubectl get pods
kubectl describe pod <pod-name>
```
- Look for `Restart Count` under Pod description.  

---

### **Common Restart Prevention Techniques:**  
1. **Resource Requests and Limits:**  
   ```yaml
   resources:
     requests:
       memory: "64Mi"
       cpu: "250m"
     limits:
       memory: "128Mi"
       cpu: "500m"
   ```

2. **Liveness and Readiness Probes:**  
   Ensure liveness probes accurately reflect application health.  

3. **Node Affinity and Taints:**  
   Place critical Pods on nodes with sufficient resources.  

4. **Horizontal Pod Autoscaler (HPA):**  
   Scale Pods dynamically to handle increased loads.  

