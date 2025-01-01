### **Kubernetes Homework Assignments**  
**Objective:** Reinforce understanding of **Pods, ReplicaSets, and Deployments** through hands-on exercises.  

---

### **Homework 1: Managing Pods (Basic Pod Operations)**  
**Goal:** Understand how to create, modify, and delete Pods directly.  

---

#### **Tasks:**  
1. **Create a Pod** running **NGINX** using a YAML manifest.  
2. **Expose port 80** inside the container (use the `containerPort` field).  
3. Deploy the Pod and verify it is running using `kubectl get pods`.  
4. **Delete the Pod** manually and observe if it restarts.  
5. **Bonus:** Try deleting the Pod while keeping the YAML manifest. Recreate it using `kubectl apply`.  

---

#### **Hints:**  
- Use the following YAML to create the Pod:  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```  
- Deploy the Pod:  
```bash
kubectl apply -f nginx-pod.yaml
```  

---

---

### **Homework 2: Implementing ReplicaSets (Scaling & Healing Pods)**  
**Goal:** Learn how to maintain a desired number of Pod replicas and handle scaling.  

---

#### **Tasks:**  
1. Create a **ReplicaSet** that manages **3 replicas** of NGINX Pods.  
2. Deploy the ReplicaSet using a YAML manifest.  
3. **Scale the ReplicaSet** to 5 replicas using `kubectl scale`.  
4. **Delete one Pod** manually and observe if a new Pod is created automatically.  
5. **Bonus:** Modify the container image from `nginx:latest` to `nginx:1.21` and apply the change.  

---

#### **Hints:**  
- Use this YAML to create the ReplicaSet:  
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
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
          image: nginx:latest
          ports:
            - containerPort: 80
```  
- Scale the ReplicaSet:  
```bash
kubectl scale rs nginx-replicaset --replicas=5
```  

---

---

### **Homework 3: Deployments and Rolling Updates**  
**Goal:** Learn how to manage application updates and rollbacks with Deployments.  

---

#### **Tasks:**  
1. Create a **Deployment** to run **3 NGINX Pods**.  
2. Apply the Deployment and verify it is running.  
3. **Perform a rolling update** by changing the NGINX image version (e.g., `nginx:1.21`).  
4. **Roll back** the Deployment to the previous version if the update fails.  
5. **Scale the Deployment** to 6 replicas and observe how new Pods are added.  

---

#### **Hints:**  
- Use this YAML to create the Deployment:  
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
          image: nginx:latest
          ports:
            - containerPort: 80
```  
- Update the Deployment:  
```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.21
```  
- Rollback the Deployment:  
```bash
kubectl rollout undo deployment/nginx-deployment
```  

---

---

### **Homework Submission Requirements:**  
1. **Screenshots** of running Pods, ReplicaSets, and Deployments (`kubectl get pods`).  
2. YAML manifests used for each task.  
3. Short explanation of what happens when Pods are manually deleted from a ReplicaSet or Deployment.  

---

