### **How CRUD Operations Work in Kubernetes (kubectl apply):**  

`kubectl apply` is a **declarative** command used to manage Kubernetes resources. It compares the **desired state** (YAML file) with the **current state** of the cluster and performs necessary **CRUD (Create, Read, Update, Delete)** operations to reconcile the two.

---

### **1. CRUD Operations Behind `kubectl apply`:**  

| Operation | Description | Example |
|-----------|-------------|---------|
| **Create** | If the resource does not exist, Kubernetes creates it. | First-time applying a Deployment. |
| **Read**   | Kubernetes checks the current state of the resource (GET request). | Check existing Pod configurations. |
| **Update** | If the resource exists but the YAML changes, Kubernetes updates it. | Changing the container image version. |
| **Delete** | If a field is removed from the YAML, Kubernetes deletes it from the live resource. | Removing a label or port from the Deployment. |

---

### **Flow of `kubectl apply`:**  
1. **Reads the YAML file** to get the desired state.  
2. **Queries the Kubernetes API server** to check the current state (`kubectl get`).  
3. **Compares the two states** (desired vs. actual).  
4. **Performs CRUD operations** to reconcile differences:  
   - **Create** new resources if missing.  
   - **Update** existing resources if there are changes.  
   - **Delete** fields or resources that no longer exist in the YAML.  

---

### **Detailed Breakdown (Example with Deployment):**  

#### **Step 1: Create a Deployment (Initial Apply):**  
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
          image: nginx:1.20
```

**Command:**  
```bash
kubectl apply -f nginx-deployment.yaml
```
- **Kubernetes checks if** `nginx-deployment` **exists.**  
- If **not found (404),** it sends a **POST request** (Create).  

---

#### **Step 2: Update the Deployment (Apply Again):**  
Modify the image to a newer version:  
```yaml
containers:
  - name: nginx
    image: nginx:1.21
```

**Apply:**  
```bash
kubectl apply -f nginx-deployment.yaml
```
- Kubernetes **detects the difference** (`nginx:1.20` → `nginx:1.21`).  
- Sends a **PATCH request** (Update).  
- **Rolling update** is triggered without deleting the Deployment.  

---

#### **Step 3: Delete a Field (Apply Again):**  
Remove the `replicas` field:  
```yaml
spec:
  replicas: 3  # This line is removed
```

**Apply:**  
```bash
kubectl apply -f nginx-deployment.yaml
```
- Kubernetes **removes** the field by sending a **PATCH request** with the missing `replicas` value.  
- The deployment defaults to `1` replica (if not explicitly set).  

---

### **What Happens Internally?**  
- **kubectl apply** interacts with the **Kubernetes API Server**.  
- It uses **Server-Side Apply (SSA)** for field ownership, ensuring no accidental overwrites.  
- **Conflicts are handled gracefully** by merging YAML changes with existing configurations.  

---

### **Best Practices for kubectl apply:**  
- **Declarative Management:** Always use `apply` over `create` for long-term resource management.  
- **Version Control:** Store YAML files in Git for audit and rollbacks.  
- **Dry Run:** Use `--dry-run=client` to preview changes without applying.  
  ```bash
  kubectl apply -f pod.yaml --dry-run=client
  ```  
- **Force Apply:** Use `kubectl apply --force` if you encounter conflicts.  

---

### **Why Use Declarative Approach (`apply`) Over Imperative (`create`)?**  
| Feature                          | `kubectl apply` (Declarative) | `kubectl create` (Imperative)  |
|----------------------------------|------------------------------|-------------------------------|
| **Consistency**                  | Maintains cluster consistency| One-time resource creation    |
| **Tracking Changes**             | YAML files represent history | No versioning of changes      |
| **Reconciliation**               | Kubernetes corrects drift    | No state drift detection      |
| **Automation (GitOps)**          | Ideal for automation (CI/CD) | Manual management             |

---

