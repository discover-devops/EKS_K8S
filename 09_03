**step-by-step document on Advanced Scheduling and Resource Management in Kubernetes**, here we will cover key features like **Taints and Tolerations, Node Affinity/Anti-Affinity, Pod Affinity/Anti-Affinity, Resource Requests and Limits, and Priority Classes**. Below is a comprehensive guide:  

---

## **Step-by-Step Document: Advanced Scheduling and Resource Management in Kubernetes**

### **1. Introduction**
Kubernetes Advanced Scheduling and Resource Management allows fine-grained control over how pods are placed on nodes and how cluster resources are managed. These mechanisms improve reliability, performance, and efficient utilization of cluster resources.

---

### **2. Prerequisites**
- A Kubernetes cluster (e.g., on Minikube, AWS EKS, or Google GKE).
- `kubectl` CLI installed and configured to access your cluster.
- Basic understanding of Kubernetes Pods and Nodes.

---

### **3. Steps to Implement Advanced Scheduling**

#### **3.1 Taints and Tolerations**
Taints and tolerations ensure that pods are only scheduled on nodes with specific conditions. 

1. **Apply a Taint to a Node:**
   ```bash
   kubectl taint nodes <node-name> key=value:NoSchedule
   ```
   Example:
   ```bash
   kubectl taint nodes node1 environment=production:NoSchedule
   ```

2. **Add a Toleration to a Pod:**
   Create a YAML file (`toleration-pod.yaml`):
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: toleration-pod
   spec:
     tolerations:
       - key: "environment"
         operator: "Equal"
         value: "production"
         effect: "NoSchedule"
     containers:
       - name: nginx
         image: nginx
   ```
   Deploy the pod:
   ```bash
   kubectl apply -f toleration-pod.yaml
   ```

---

#### **3.2 Node Affinity and Anti-Affinity**
Node Affinity allows pods to specify rules about which nodes they should (or shouldn’t) be placed on.

1. **Node Affinity Example:**
   Add a label to the node:
   ```bash
   kubectl label nodes <node-name> environment=production
   ```

2. **Node Affinity Pod Spec:**
   Create a YAML file (`node-affinity-pod.yaml`):
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: node-affinity-pod
   spec:
     affinity:
       nodeAffinity:
         requiredDuringSchedulingIgnoredDuringExecution:
           nodeSelectorTerms:
             - matchExpressions:
                 - key: environment
                   operator: In
                   values:
                     - production
     containers:
       - name: nginx
         image: nginx
   ```
   Deploy the pod:
   ```bash
   kubectl apply -f node-affinity-pod.yaml
   ```

---

#### **3.3 Pod Affinity and Anti-Affinity**
Pod Affinity controls the scheduling of pods based on the presence (or absence) of other pods.

1. **Label a Pod:**
   ```bash
   kubectl label pod <pod-name> app=myapp
   ```

2. **Pod Affinity Spec:**
   Create a YAML file (`pod-affinity.yaml`):
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: pod-affinity-example
   spec:
     affinity:
       podAffinity:
         requiredDuringSchedulingIgnoredDuringExecution:
           labelSelector:
             matchExpressions:
               - key: app
                 operator: In
                 values:
                   - myapp
           topologyKey: "kubernetes.io/hostname"
     containers:
       - name: nginx
         image: nginx
   ```
   Deploy the pod:
   ```bash
   kubectl apply -f pod-affinity.yaml
   ```

---

### **4. Resource Management**

#### **4.1 Resource Requests and Limits**
Define how much CPU and memory a pod is guaranteed to use or can use at most.

1. **Example Pod with Resource Requests and Limits:**
   Create a YAML file (`resource-pod.yaml`):
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: resource-pod
   spec:
     containers:
       - name: nginx
         image: nginx
         resources:
           requests:
             memory: "64Mi"
             cpu: "250m"
           limits:
             memory: "128Mi"
             cpu: "500m"
   ```
   Deploy the pod:
   ```bash
   kubectl apply -f resource-pod.yaml
   ```

---

#### **4.2 Priority Classes**
Priority Classes determine the importance of a pod compared to others during resource contention.

1. **Create a Priority Class:**
   ```yaml
   apiVersion: scheduling.k8s.io/v1
   kind: PriorityClass
   metadata:
     name: high-priority
   value: 1000
   globalDefault: false
   description: "High-priority pods."
   ```
   Apply the YAML:
   ```bash
   kubectl apply -f priority-class.yaml
   ```

2. **Assign Priority to a Pod:**
   Create a YAML file (`priority-pod.yaml`):
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: priority-pod
   spec:
     priorityClassName: high-priority
     containers:
       - name: nginx
         image: nginx
   ```
   Deploy the pod:
   ```bash
   kubectl apply -f priority-pod.yaml
   ```

---

### **5. Verification**

#### **5.1 Check Node Labels:**
```bash
kubectl get nodes --show-labels
```

#### **5.2 Check Pod Scheduling:**
```bash
kubectl describe pod <pod-name>
```

#### **5.3 Check Priority Class:**
```bash
kubectl get priorityclass
```

#### **5.4 Check Resource Usage:**
```bash
kubectl top pod <pod-name>
```

---

### **6. Conclusion**
Using these advanced scheduling and resource management techniques, you can:
- Ensure workloads are placed where they are most effective.
- Optimize cluster resources.
- Prioritize critical applications over less important ones during resource contention.

