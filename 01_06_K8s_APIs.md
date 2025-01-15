To **list API objects** in Kubernetes, you can use the following commands and tools:

---

### **1. List All API Resources (Objects):**  
```bash
kubectl api-resources
```
- **Output:** Lists all available API objects (like Pods, Deployments, Services), their short names, API versions, and if they are namespaced.  

---

### **2. List All API Versions:**  
```bash
kubectl api-versions
```
- **Output:** Displays all available API versions.  
- Example: `apps/v1`, `batch/v1`, `v1` (core API group).  

---

### **3. Get Details of a Specific API Object:**  
```bash
kubectl explain <resource>
```
- Example:  
  ```bash
  kubectl explain pod
  kubectl explain deployment
  ```  
- **Output:** Shows the fields and descriptions of the specified object.  

---

### **4. List Resources from Specific API Groups:**  
```bash
kubectl api-resources --api-group=apps
```
- **Example:** Lists all resources in the `apps` API group (like Deployments, StatefulSets).  

---

### **5. Get Detailed Schema for an Object (YAML):**  
```bash
kubectl explain pod --recursive
```
- **Output:** Shows a full breakdown of the object fields and structure in a nested format.  

---

### **6. View Live API Objects (JSON):**  
```bash
kubectl get --raw /api/v1 | jq
```
- **Output:** Lists live API objects directly from the Kubernetes API server.  
- Requires **jq** to format the output.  

---

### **7. List Objects by Namespace:**  
```bash
kubectl get all -n <namespace>
```
- **Example:**  
  ```bash
  kubectl get all -n kube-system
  ```
- Lists all running objects (Pods, Services, Deployments, etc.) in a specific namespace.  

---

### **Common API Resources and Their Short Names:**  
| Resource          | Short Name | API Group | Namespaced |  
|-------------------|------------|-----------|------------|  
| Pod               | po         | v1        | Yes        |  
| Deployment        | deploy     | apps/v1   | Yes        |  
| Service           | svc        | v1        | Yes        |  
| ConfigMap         | cm         | v1        | Yes        |  
| PersistentVolume  | pv         | v1        | No         |  
| Ingress           | ing        | networking.k8s.io/v1 | Yes |  
| Job               | job        | batch/v1  | Yes        |  

---

### **Why List API Objects?**  
- **Discoverability:** Helps explore the available Kubernetes resources.  
- **Version Control:** Check if the correct API versions are in use.  
- **Troubleshooting:** Verify the existence of objects or API groups during debugging.  

