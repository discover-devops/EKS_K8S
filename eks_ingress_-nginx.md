

#  **Ingress & Ingress Controller on AWS EKS — Full Hands-On Lab (Markdown Version)**

**Topic:** Kubernetes Ingress, Ingress Controller, Microservices Routing on AWS EKS
**Level:** Beginner → Intermediate
**Cluster Used:** EKS (v1.32+)
**Tools:** `kubectl`, `helm`, AWS Console


<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/4306c962-d3ef-4242-9bcd-b24b61cf8fd9" />



-----------------------------------------

# ##  1. Introduction — What is Ingress?

In Kubernetes:

| Component                  | Purpose                                                     |
| -------------------------- | ----------------------------------------------------------- |
| **Service (ClusterIP)**    | Exposes Pods inside the cluster                             |
| **Service (NodePort)**     | Exposes Pods on each node (not secure, not scalable)        |
| **Service (LoadBalancer)** | Creates a separate AWS ELB *per service*                    |
| **Ingress**                | **Single entry point** for all apps with HTTP routing rules |
| **Ingress Controller**     | The “brain” that reads Ingress rules and programs NGINX     |

### In short:

**Ingress Controller = NGINX load balancer inside the cluster**
**Ingress = Routing rules ("/path" → service A)**

---

# ##  2. Final Architecture (Important)

```
Client
   │
   ▼
AWS Application Load Balancer (created by ingress-nginx)
   │
   ▼
Ingress Controller (NGINX Pod)
   │
   ├── /service-a  →  service-a ClusterIP → Pod A
   └── /service-b  →  service-b ClusterIP → Pod B
```

Everything goes through **one** LoadBalancer → multiple microservices.

---

# ##  3. Prerequisites

✔ EKS cluster running
✔ `kubectl` configured
✔ `helm` installed
✔ Internet access for pulling images

Check nodes:

```sh
kubectl get nodes
```

---

# ##  4. Step 1 — Create Namespace for Ingress

```sh
kubectl create namespace ingress-nginx
```

---

# ##  5. Step 2 — Install NGINX Ingress Controller using Helm

```sh
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Install:

```sh
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.service.type=LoadBalancer
```

Check pods:

```sh
kubectl get pods -n ingress-nginx
```

Expected:

```
ingress-nginx-controller-xxxx   1/1   Running
```

Check LoadBalancer:

```sh
kubectl get svc -n ingress-nginx
```

Example result:

```
ingress-nginx-controller   LoadBalancer   <EXTERNAL-DNS>   80:31426/TCP,443:32337/TCP
```

Copy the external URL — you will use it later.

---

# ##  6. Step 3 — Create Namespace for Apps

```sh
kubectl create namespace demo-app
```

---

# ##  7. Step 4 — Deploy Service A

Create file `service-a.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-a
  namespace: demo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: service-a
  template:
    metadata:
      labels:
        app: service-a
    spec:
      containers:
      - name: service-a
        image: hashicorp/http-echo
        args:
        - "-text=Hello from Service A"
---
apiVersion: v1
kind: Service
metadata:
  name: service-a
  namespace: demo-app
spec:
  type: ClusterIP
  selector:
    app: service-a
  ports:
  - port: 80
    targetPort: 5678
```

Apply:

```sh
kubectl apply -f service-a.yaml
```

---

# ##  8. Step 5 — Deploy Service B

Create `service-b.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-b
  namespace: demo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: service-b
  template:
    metadata:
      labels:
        app: service-b
    spec:
      containers:
      - name: service-b
        image: hashicorp/http-echo
        args:
        - "-text=Hello from Service B"
---
apiVersion: v1
kind: Service
metadata:
  name: service-b
  namespace: demo-app
spec:
  type: ClusterIP
  selector:
    app: service-b
  ports:
  - port: 80
    targetPort: 5678
```

Apply:

```sh
kubectl apply -f service-b.yaml
```

Check:

```sh
kubectl get pods -n demo-app
kubectl get svc -n demo-app
```

---

# ##  9. Step 6 — Create Ingress Resource

Create `demo-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  namespace: demo-app
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /service-a
        pathType: Prefix
        backend:
          service:
            name: service-a
            port:
              number: 80
      - path: /service-b
        pathType: Prefix
        backend:
          service:
            name: service-b
            port:
              number: 80
```

Apply:

```sh
kubectl apply -f demo-ingress.yaml
```

Check:

```sh
kubectl get ingress -n demo-app
```

---

# ##  10. Step 7 — Fix Security Groups (Important on AWS)

NGINX LoadBalancer (AWS ELB) must reach nodes on NodePort (e.g., 31426).

### You must add this inbound rule:

| Security Group | Allow Port                     | Source                                 |
| -------------- | ------------------------------ | -------------------------------------- |
| **Node SG**    | NodePort range **30000–32767** | **Ingress Controller LoadBalancer SG** |

Your fix was correct:

On **Node SG → Inbound**: allow 30000–32767
Source: Select the **LoadBalancer SG** from the ELB

After adding rules → routing works.

---

# ##  11. Step 8 — Test Ingress

Use the ELB DNS:

```sh
LB=http://<ELB-DNS>
curl $LB/service-a
curl $LB/service-b
```

Expected output:

```
Hello from Service A
Hello from Service B
```

---

# ##  12. Troubleshooting Guide

### **502 Bad Gateway?**

Node SG is blocking NodePort
Wrong service port
Wrong targetPort
IngressClass missing
Pods not ready

### **Ingress shows no address?**

NGINX controller not deployed correctly.

### **404 Not Found?**

Wrong path or wrong service name.

---

# ##  13. Cleanup (Optional)

```sh
kubectl delete ns demo-app
helm uninstall ingress-nginx -n ingress-nginx
kubectl delete ns ingress-nginx
```

---

# ##  14. Interview Questions for Students

### **Basic**

1. What is the difference between Ingress and LoadBalancer service?
2. Why do we need an Ingress Controller instead of just Ingress?
3. What is the role of NGINX in this setup?

### **Intermediate**

4. Why do NodePort rules matter for AWS?
5. How does NGINX route traffic to different services?
6. What is IngressClassName and why is it important?

### **Advanced**

7. Explain the data flow from client → ELB → Ingress → Pod.
8. How would you implement TLS termination?
9. How do you implement rewrite rules?

---

