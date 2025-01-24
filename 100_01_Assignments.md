Here are **5 real-world Kubernetes assignments** designed to help you understand and apply Kubernetes concepts effectively. 
These assignments cover various Kubernetes features and mimic real-world use cases:

---

### **Assignment 1: Deploy a Highly Available Web Application with Load Balancing**
#### **Scenario:**
You are tasked to deploy a highly available web application in Kubernetes for a client. The application should handle internet traffic and remain resilient during node failures.

#### **Requirements:**
1. Create a **Deployment** with 3 replicas of an NGINX application.
2. Configure a **Service** of type `LoadBalancer` to expose the application to the internet.
3. Ensure the deployment uses the following specifications:
   - Resource Requests: `cpu: 100m`, `memory: 128Mi`
   - Resource Limits: `cpu: 200m`, `memory: 256Mi`
4. Use a **ConfigMap** to store environment-specific variables for the application (e.g., site name or environment type).
5. Simulate traffic using a tool like Apache Benchmark (`ab`) or `curl` and ensure the application scales properly.

---

### **Assignment 2: Implement Horizontal Pod Autoscaler**
#### **Scenario:**
Your organization needs an application that automatically adjusts its resources during traffic surges. You need to implement an **HPA** for this application.

#### **Requirements:**
1. Deploy a Python Flask app using Kubernetes.
2. Configure resource requests and limits for the app:
   - CPU request: `100m`
   - Memory request: `128Mi`
3. Set up an **HPA** to scale the pods between 2 and 10 replicas, targeting 50% CPU utilization.
4. Simulate high CPU usage by sending a large number of requests to the application.
5. Observe and document how the HPA scales up and down based on traffic.

---

### **Assignment 3: Deploy a Stateful Application Using StatefulSets**
#### **Scenario:**
You need to deploy a MySQL database that persists data even if pods are restarted. Use **StatefulSets** to achieve this.

#### **Requirements:**
1. Deploy MySQL using a **StatefulSet** with 2 replicas.
2. Use a **PersistentVolumeClaim (PVC)** to create persistent storage for each MySQL pod.
3. Configure a **Service** of type `ClusterIP` to allow communication between the MySQL pods.
4. Test data persistence by:
   - Writing data to MySQL.
   - Deleting and recreating one of the pods.
   - Verifying that the data still exists after the pod restarts.

---

### **Assignment 4: Configure Kubernetes Ingress for Microservices**
#### **Scenario:**
Your team has a microservices-based architecture consisting of multiple services. You need to route external traffic to specific services based on the URL path.

#### **Requirements:**
1. Deploy three microservices (`service1`, `service2`, and `service3`) using Deployments:
   - `service1`: Returns "Hello from Service 1"
   - `service2`: Returns "Hello from Service 2"
   - `service3`: Returns "Hello from Service 3"
2. Configure **Ingress** to route traffic as follows:
   - `/service1` → `service1`
   - `/service2` → `service2`
   - `/service3` → `service3`
3. Secure the Ingress with an SSL certificate using **Cert-Manager**.
4. Test the configuration by sending requests to the respective endpoints and verifying the responses.

---

### **Assignment 5: Implement Cluster Autoscaler in EKS**
#### **Scenario:**
Your cluster has a fixed number of nodes but occasionally experiences resource shortages during peak traffic. You need to implement the **Cluster Autoscaler** to dynamically add or remove nodes.

#### **Requirements:**
1. Deploy an application that consumes a significant amount of resources:
   - Example: A resource-intensive Python or Java application with custom resource requests.
2. Configure an **AWS Auto Scaling Group (ASG)** for worker nodes with the following settings:
   - Min nodes: 2
   - Max nodes: 5
3. Deploy the **Cluster Autoscaler** in your EKS cluster.
4. Simulate high resource usage by deploying additional pods and verify that new nodes are added.
5. Scale down the workload and observe the Cluster Autoscaler removing underutilized nodes.

---

### **Submission Expectations**
For each assignment, you are expected to:
1. Provide YAML manifests for Deployments, Services, ConfigMaps, StatefulSets, and Ingress as applicable.
2. Share screenshots or terminal outputs showing the working configuration and outcomes.
3. Submit a brief write-up explaining the steps you followed and any challenges faced.

