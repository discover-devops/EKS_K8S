 **detailed YAML files and step-by-step instructions** for each of the five Kubernetes assignments:

---

### **Assignment 1: Deploy a Highly Available Web Application with Load Balancing**
#### **Step-by-Step Guide**
1. Create a Deployment for NGINX with 3 replicas.
2. Configure a LoadBalancer Service to expose it publicly.
3. Use a ConfigMap for environment variables.

#### **YAML Files**
**Deployment:**
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
        envFrom:
        - configMapRef:
            name: nginx-config
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
```

**ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  SITE_NAME: "My Awesome Site"
```

**Service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

Apply the files:
```bash
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-config.yaml
kubectl apply -f nginx-service.yaml
```

---

### **Assignment 2: Implement Horizontal Pod Autoscaler**
#### **Step-by-Step Guide**
1. Create a Python Flask app.
2. Deploy it in Kubernetes.
3. Configure an HPA targeting 50% CPU utilization.

#### **YAML Files**
**Python Flask Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flask
  template:
    metadata:
      labels:
        app: flask
    spec:
      containers:
      - name: flask
        image: <your-dockerhub-repo>/flask-app:latest
        ports:
        - containerPort: 5000
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
```

**HPA:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: flask-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: flask-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Apply the files:
```bash
kubectl apply -f flask-app.yaml
kubectl apply -f flask-hpa.yaml
```

Simulate traffic:
```bash
ab -n 10000 -c 100 http://<LoadBalancer-IP>:5000/
```

---

### **Assignment 3: Deploy a Stateful Application Using StatefulSets**
#### **Step-by-Step Guide**
1. Deploy MySQL with StatefulSet and PersistentVolumeClaims.
2. Verify data persistence after pod restarts.

#### **YAML Files**
**StatefulSet:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"
  replicas: 2
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "rootpassword"
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

**Service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
    targetPort: 3306
  clusterIP: None
  selector:
    app: mysql
```

Apply the files:
```bash
kubectl apply -f mysql-statefulset.yaml
kubectl apply -f mysql-service.yaml
```

Test persistence:
- Connect to MySQL: `kubectl exec -it <mysql-pod-name> -- mysql -u root -p`.
- Insert data, delete a pod, and verify data is retained.

---

### **Assignment 4: Configure Kubernetes Ingress for Microservices**
#### **Step-by-Step Guide**
1. Deploy three microservices.
2. Set up Ingress with path-based routing and SSL.

#### **YAML Files**
**Ingress:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - mydomain.com
    secretName: my-tls-secret
  rules:
  - host: mydomain.com
    http:
      paths:
      - path: /service1
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 80
      - path: /service2
        pathType: Prefix
        backend:
          service:
            name: service2
            port:
              number: 80
      - path: /service3
        pathType: Prefix
        backend:
          service:
            name: service3
            port:
              number: 80
```

---

### **Assignment 5: Implement Cluster Autoscaler in EKS**
#### **Step-by-Step Guide**
1. Configure an ASG for worker nodes.
2. Deploy the Cluster Autoscaler.
3. Simulate scaling up/down scenarios.

#### **YAML Files**
Install Cluster Autoscaler:
```bash
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/latest/download/cluster-autoscaler-autodiscover.yaml
```

Simulate load:
- Deploy a resource-heavy app.
- Monitor scale-up with:
```bash
kubectl get nodes
kubectl -n kube-system logs -f deployment/cluster-autoscaler
```

---

