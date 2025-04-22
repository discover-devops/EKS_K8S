
---

# Full Lab: Deploying a Multi-Language Microservices App on EKS (AMZN Linux)


![image](https://github.com/user-attachments/assets/86f9f9c1-b370-4627-a843-3dfd1aa381f1)

---

## Introduction

This hands-on lab guides you through deploying a web application composed of 3 microservices, each written in a different programming language, onto an Amazon EKS cluster.

###  Goals:
- Build Docker images for each service (Python, Java, Go)
- Push images to DockerHub
- Deploy them via Kubernetes `Deployment` and `Service`
- Expose login service externally via LoadBalancer

---

##  Application Architecture

| Microservice | Language | Port | Service Type  | Purpose             |
|--------------|----------|------|---------------|---------------------|
| login        | Python   | 5000 | LoadBalancer  | Web UI              |
| payment      | Java     | 8080 | ClusterIP     | Payment logic API   |
| delivery     | Go       | 9090 | ClusterIP     | Delivery logic API  |

---

##  Project Structure

```bash
microservices-app/
├── login/
│   ├── app.py
│   ├── Dockerfile
│   ├── deployment.yaml
│   └── service.yaml
├── payment/
│   ├── PaymentApplication.java
│   ├── Dockerfile
│   ├── deployment.yaml
│   └── service.yaml
├── delivery/
│   ├── main.go
│   ├── go.mod
│   ├── Dockerfile
│   ├── deployment.yaml
│   └── service.yaml

mkdir -p microservices-app/{login,payment,delivery} && \
touch microservices-app/login/{app.py,Dockerfile,deployment.yaml,service.yaml} && \
touch microservices-app/payment/{PaymentApplication.java,Dockerfile,deployment.yaml,service.yaml} && \
touch microservices-app/delivery/{main.go,Dockerfile,deployment.yaml,service.yaml}


```

---

#  Step 1: Write Application Code

---

### 🐍 login/app.py (Python + Flask)

```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def home():
    return "<h1>Login Page</h1><form><input type='text'><input type='submit'></form>"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

### ☕ payment/PaymentApplication.java (Java + Spark)

```java
import spark.Spark;

public class PaymentApplication {
    public static void main(String[] args) {
        Spark.port(8080);
        Spark.get("/pay", (req, res) -> "Payment Processed!");
    }
}
```

---

### 🐹 delivery/main.go (Go)

```go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/deliver", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "Delivery Scheduled!")
    })
    http.ListenAndServe(":9090", nil)
}
```

---

### delivery/go.mod

```go
module delivery-service
go 1.21
```

---

#  Step 2: Dockerfiles for All Services

---

### login/Dockerfile

```dockerfile
FROM python:3.8-slim
WORKDIR /app
COPY app.py .
RUN pip install flask
EXPOSE 5000
CMD ["python", "app.py"]
```

---

### payment/Dockerfile

```dockerfile
FROM openjdk:17
COPY PaymentApplication.java .
RUN curl -L -o spark-core-2.9.4.jar https://repo1.maven.org/maven2/com/sparkjava/spark-core/2.9.4/spark-core-2.9.4.jar
RUN javac -cp spark-core-2.9.4.jar PaymentApplication.java
CMD ["java", "-cp", ".:spark-core-2.9.4.jar", "PaymentApplication"]
```

---

### delivery/Dockerfile

```dockerfile
FROM golang:1.19
WORKDIR /app
COPY go.mod ./
COPY main.go ./
RUN go build -o delivery
CMD ["./delivery"]
```

---

#  Step 3: Build and Push Docker Images

> Replace with your DockerHub username: `discoverdevops`

```bash
# login service
cd login
docker build -t discoverdevops/micro_service:login .
docker push discoverdevops/micro_service:login

# payment service
cd ../payment
docker build -t discoverdevops/micro_service:payment .
docker push discoverdevops/micro_service:payment

# delivery service
cd ../delivery
docker build -t discoverdevops/micro_service:delivery .
docker push discoverdevops/micro_service:delivery
```

---

#  Step 4: Kubernetes YAMLs for Deployment & Services

---

### login/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: login-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: login
  template:
    metadata:
      labels:
        app: login
    spec:
      containers:
      - name: login
        image: discoverdevops/micro_service:login
        ports:
        - containerPort: 5000
```

---

### login/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: login-service
spec:
  type: LoadBalancer
  selector:
    app: login
  ports:
    - port: 80
      targetPort: 5000
```

---

### payment/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: payment
  template:
    metadata:
      labels:
        app: payment
    spec:
      containers:
      - name: payment
        image: discoverdevops/micro_service:payment
        ports:
        - containerPort: 8080
```

---

### payment/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment
  ports:
    - port: 8080
      targetPort: 8080
```

---

### delivery/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: delivery-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: delivery
  template:
    metadata:
      labels:
        app: delivery
    spec:
      containers:
      - name: delivery
        image: discoverdevops/micro_service:delivery
        ports:
        - containerPort: 9090
```

---

### delivery/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: delivery-service
spec:
  selector:
    app: delivery
  ports:
    - port: 9090
      targetPort: 9090
```

---

#  Step 5: Deploy to Amazon EKS

```bash
kubectl apply -f login/deployment.yaml
kubectl apply -f login/service.yaml

kubectl apply -f payment/deployment.yaml
kubectl apply -f payment/service.yaml

kubectl apply -f delivery/deployment.yaml
kubectl apply -f delivery/service.yaml
```

---

#  Step 6: Access the Login Service

```bash
kubectl get svc
```

Look for `login-service` with `EXTERNAL-IP`. Access in browser:

```
http://<external-ip>
```

You should see your login page.

---

#  Final Tips

- To test inter-service communication, use DNS names like:
  - `http://payment-service:8080/pay`
  - `http://delivery-service:9090/deliver`
- Add Ingress + DNS later for a production-ready setup
- Use Horizontal Pod Autoscaler for scaling
- Use Helm or Kustomize for packaging later

---

