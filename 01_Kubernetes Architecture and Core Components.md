### **Kubernetes Overview & Architecture (Detailed Explanation with Examples)**  

---

### **1. What is Kubernetes?**  
Kubernetes is an open-source platform that automates **container orchestration**—the deployment, scaling, and management of containerized applications.  
- It ensures applications run consistently across environments, making them resilient, scalable, and manageable.  

---

### **2. Kubernetes Architecture Overview**  

Kubernetes follows a **Master-Worker Node architecture**:  
- **Control Plane (Master Node):** Manages the cluster.  
- **Worker Nodes:** Run application workloads (Pods).  

---

### **3. Key Components**  

---

### **A. Control Plane Components (Master Node):**  
The control plane makes decisions about the cluster (e.g., scheduling) and detects/responds to cluster events.  

| Component                  | Description | Example Commands |
|----------------------------|-------------|------------------|
| **API Server**              | Acts as the frontend for Kubernetes. Receives and validates `kubectl` requests and updates the cluster state in etcd. | `kubectl apply`, `kubectl get` |
| **etcd**                    | A distributed key-value store that stores all cluster data (desired state). | `etcdctl get /registry` |
| **Controller Manager**       | Monitors the cluster and ensures the desired state (e.g., maintains Pod replicas). | Automatically creates new Pods if one fails |
| **Scheduler**               | Places (schedules) Pods on worker nodes based on resource availability and constraints. | `kubectl describe nodes` to check resources |

---

**🔹 Example:**  
When you run:  
```bash
kubectl run nginx --image=nginx
```
- The **API Server** receives the request.  
- The **Scheduler** assigns the Pod to a node.  
- The **Controller Manager** ensures the Pod is running by creating necessary replicas.  
- **etcd** stores the desired Pod state.  

---

### **B. Worker Node Components:**  
Worker nodes run the actual containerized workloads.  

| Component                | Description | Example Commands |
|--------------------------|-------------|------------------|
| **Kubelet**               | Agent on each node ensuring containers are running as expected. | `kubectl describe pod` (to see kubelet activity) |
| **Kube-Proxy**            | Maintains network rules to route traffic to the correct Pod across nodes. | `iptables -L` |
| **Container Runtime**     | Software that runs containers (Docker, containerd, CRI-O). | `docker ps`, `crictl ps` |

---

**🔹 Example:**  
- When the Scheduler assigns a Pod to a worker node, **Kubelet** starts the container.  
- **Kube-Proxy** ensures network traffic can reach the Pod via `ClusterIP` or `NodePort` Service.  

---

### **4. Understanding Core Concepts: Clusters, Nodes, Pods, and Namespaces**  

---

### **1. Cluster:**  
- A Kubernetes cluster consists of **at least one control plane** and multiple worker nodes.  
- The control plane manages the state of the cluster, while worker nodes run the containers.  

**🔹 Example:**  
```bash
kubectl get nodes
```
- Shows all nodes in the cluster.  

---

### **2. Nodes:**  
- A **Node** is a machine (virtual or physical) that runs Pods.  
- Nodes can be part of **cloud environments** (AWS, GCP, Azure) or on-premises.  

**🔹 Example:**  
```bash
kubectl describe node <node-name>
```
- Displays node health, resources, and allocated Pods.  

---

### **3. Pods (Smallest Deployable Unit):**  
- A **Pod** is a single instance of a running process in Kubernetes, often wrapping one or more containers.  
- Pods share network namespaces and storage volumes.  

**🔹 Example:**  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx:latest
```
```bash
kubectl apply -f nginx-pod.yaml
```
- Deploys a Pod running an NGINX container.  

---

### **4. Namespaces:**  
- Namespaces provide logical separation for resources within a cluster (multi-tenancy).  
- Default namespaces:  
  - **default** – Default for all resources.  
  - **kube-system** – Kubernetes internal services.  
  - **kube-public** – Publicly accessible resources.  

**🔹 Example:**  
```bash
kubectl get namespaces
kubectl create namespace dev
```
- Creates a namespace called `dev`.  

---

### **5. Essential kubectl Commands (CRUD Operations):**  

| Command                         | Description                          | Example                              |
|---------------------------------|--------------------------------------|--------------------------------------|
| **Create**                      | Creates Kubernetes resources.        | `kubectl run nginx --image=nginx`    |
| **Get**                         | Retrieves information about resources. | `kubectl get pods`                   |
| **Describe**                    | Shows detailed resource info.        | `kubectl describe pod nginx-pod`     |
| **Delete**                      | Deletes resources.                   | `kubectl delete pod nginx-pod`       |
| **Apply**                       | Applies YAML to create/update resources. | `kubectl apply -f pod.yaml`          |

---

### **Examples for Each Command:**  

---

### **1. Create a Pod (Using kubectl):**  
```bash
kubectl run nginx --image=nginx
```
- Directly creates a Pod running NGINX.  

---

### **2. Get Information About Pods and Nodes:**  
```bash
kubectl get pods
kubectl get nodes
kubectl get services
```
- Lists all running Pods, Nodes, and Services.  

---

### **3. Describe a Pod:**  
```bash
kubectl describe pod nginx-pod
```
- Shows container status, IP, and events for the Pod.  

---

### **4. Delete a Pod:**  
```bash
kubectl delete pod nginx-pod
```
- Deletes the specified Pod.  

---

### **5. Apply YAML (Declarative Way):**  
**nginx-pod.yaml:**  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx:latest
```
```bash
kubectl apply -f nginx-pod.yaml
```
- Creates the Pod defined in YAML.  

---

### **Key Takeaways:**  
- **API Server** is the gateway to the cluster.  
- **etcd** stores the cluster’s state.  
- **Kubelet** ensures Pods run as expected.  
- Use **kubectl** for daily operations and managing Kubernetes resources.  

