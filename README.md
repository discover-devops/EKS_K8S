### **Kubernetes 10- Course Curriculum (1 Hour Per  with Labs)**  

---

### ** 1 – Core Concepts & Architecture**  
- **Kubernetes Overview & Architecture**  
  - Control Plane Components: API Server, etcd, Controller Manager, Scheduler  
  - Worker Node Components: Kubelet, Kube-Proxy, Container Runtime  
- **Understanding Clusters, Nodes, Pods, and Namespaces**  
- **kubectl Commands (Basics):**  
  - Create, Get, Delete, Describe, Apply  

**🔹 Lab:**  
- Set up a Kubernetes cluster (Minikube/kind/k3s)  
- Deploy a basic NGINX Pod using `kubectl run`  
- Explore the cluster using `kubectl get nodes` and `kubectl get ns`  

---

### ** 2 – Workloads: ReplicaSets, Deployments & StatefulSets**  
- **ReplicaSets vs Deployments vs StatefulSets**  
  - Pod Scaling with ReplicaSets  
  - Rolling Updates with Deployments  
  - StatefulSet Use Cases (Stateful Applications)  
- **Pod Lifecycle**  

**🔹 Lab:**  
- Deploy an NGINX Deployment (3 replicas)  
- Scale the Deployment and perform a rolling update  

---

### ** 3 – Services & Networking Basics**  
- **Kubernetes Services:**  
  - ClusterIP, NodePort, LoadBalancer  
- **Pod-to-Pod Communication**  
- **Service Discovery & DNS Basics (CoreDNS)**  

**🔹 Lab:**  
- Expose the NGINX Deployment using a NodePort Service  
- Access the NGINX service using `curl`  

---

### ** 4 – Role-Based Access Control (RBAC)**  
- **RBAC Concepts:**  
  - Roles, ClusterRoles, RoleBindings, ClusterRoleBindings  
- **Service Accounts and Permissions**  

**🔹 Lab:**  
- Create a Role to limit Pod access  
- Bind the Role to a specific ServiceAccount  

---

### ** 5 – Cluster Networking (Advanced)**  
- **Cluster Networking Deep Dive**  
- **CNI (Container Network Interface) Plugins:** Calico, Flannel, Cilium  
- **Ingress Controllers:**  
  - NGINX Ingress, Traefik  
- **Network Policies for Pod Isolation**  

**🔹 Lab:**  
- Install and configure an NGINX Ingress Controller  
- Apply Network Policies to restrict Pod access  

---

### ** 6 – Persistent Storage**  
- **Persistent Volumes (PVs) & Persistent Volume Claims (PVCs)**  
- **Dynamic Provisioning with Storage Classes**  
- **StatefulSet with Persistent Storage**  

**🔹 Lab:**  
- Create a Persistent Volume and Claim  
- Deploy a MySQL StatefulSet with dynamic provisioning  

---

### ** 7 – ConfigMaps and Secrets**  
- **ConfigMaps:** Managing Configurations  
- **Secrets:** Storing Sensitive Data  
- **Using Environment Variables and Volumes**  

**🔹 Lab:**  
- Create ConfigMaps and inject them into Pods  
- Deploy a Pod that uses a Secret to access credentials  

---

### ** 8 – Scheduling, Taints & Tolerations**  
- **Understanding Taints and Tolerations**  
- **Node Affinity and Anti-Affinity**  
- **Pod Priority & Preemption**  

**🔹 Lab:**  
- Use Taints to prevent Pod scheduling on specific nodes  
- Apply Node Affinity rules to control Pod placement  

---

### ** 9 – Resource Management**  
- **Resource Requests and Limits**  
- **Horizontal Pod Autoscaler (HPA)**  
- **Vertical Pod Autoscaler (VPA)**  

**🔹 Lab:**  
- Set resource requests/limits on Pods  
- Deploy HPA to automatically scale Pods based on CPU usage  

---

### ** 10 – Real-World Deployment & Final Lab**  
- **Deploying a Multi-Tier Application:**  
  - Frontend: NGINX  
  - Backend: Redis/MySQL  
  - Ingress: NGINX Ingress Controller  
  - PVCs for data persistence  
- **Rolling Updates and Rollbacks**  
- **Network Policies for Security**  

**🔹 Lab:**  
- Deploy and test a full-stack application with an Ingress Controller  
- Perform rolling updates and verify the rollback process  

---

### **Course Outcome:**  
- Master Kubernetes core concepts, advanced networking, storage, and scheduling.  
- Gain practical hands-on experience through structured labs.  
- Be ready to manage, troubleshoot, and deploy Kubernetes workloads confidently.  

