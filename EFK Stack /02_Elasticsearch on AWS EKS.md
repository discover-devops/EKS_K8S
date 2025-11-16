
---

#  Elasticsearch on AWS EKS — Concept + Full Hands-On Tutorial

> **Goal:**
> Understand what Elasticsearch is, why it’s used for monitoring microservices, and then deploy a **production-style single-node Elasticsearch 7.17** on **AWS EKS**, using:
>
> * AWS EBS CSI driver
> * A dedicated `gp3-storage` StorageClass
> * **ECK Operator** (official Elastic operator for Kubernetes)

Later, you can plug in **Kibana + Fluent Bit** to complete the full EFK stack.

---

## 1. What is Elasticsearch?

Elasticsearch is:

* A **distributed search and analytics engine**
* Built on top of **Lucene**
* Designed to store and search **JSON documents** at scale

You can think of it as:

* A **NoSQL document database** that is optimized for:

  * Full-text search
  * Filtering & aggregations
  * Time-series data (logs, metrics, traces)

### 1.1 Key Concepts 

* **Cluster** – group of one or more Elasticsearch nodes
* **Node** – a single Elasticsearch instance (a pod in Kubernetes)
* **Index** – like a “database” or “table”; stores related documents, usually by use case or time-period (e.g., `logs-2025.11.16`)
* **Document** – JSON object you index (e.g., one log line or one order)
* **Shards** – how Elasticsearch splits an index into pieces for scaling
* **Replicas** – copies of shards for high availability

---

## 2. Why Elasticsearch for Microservice Monitoring?

In a microservices world:

* Each microservice runs in its **own pod** (or multiple pods)
* Logs are written to **stdout/stderr** (container logs)
* Pods scale up/down, move across nodes, and die

So the challenge is:

> “How do I **centralize logs** from *all* microservices across *all* pods, and then **search & analyze** them easily?”

This is where the **EFK stack** comes in:

* **Fluent Bit** – log forwarder that runs as a **DaemonSet** on each node, reads container logs, and ships them to…
* **Elasticsearch** – stores logs in indices; supports powerful search and aggregations
* **Kibana** – UI on top of Elasticsearch to visualize logs, build dashboards, and search

### Log Flow (Explain Simply)

```text
Microservice Pod → Container Logs (/var/log/containers) 
→ Fluent Bit DaemonSet 
→ Elasticsearch Index (e.g., eks-logs-2025.11.16)
→ Kibana Discover / Dashboard
```

So Elasticsearch is the **heart** of this logging system.

---

## 3. Architecture on AWS EKS

We are going to build this:

* **EKS Cluster** (managed K8s control plane)
* **Worker Nodes** in private/public subnets
* **EBS volumes** for Elasticsearch data
* **EBS CSI Driver** to dynamically create PVs from EBS
* **StorageClass `gp3-storage`** for Elasticsearch
* **ECK Operator** to manage Elasticsearch
* **Elasticsearch single-node cluster** (for lab/demo)

Namespaces:

* `demo-app` – where your sample microservice runs
* `logging` – where Elasticsearch (and later Kibana, Fluent Bit) live

---

## 4. Prerequisites

### 4.1 Tools Installed on Bastion/Jump Host

On your EC2 bastion (Amazon Linux):

```bash
aws --version
kubectl version --client
eksctl version
helm version
```

You already had:

* `aws-cli`
* `kubectl`
* `eksctl`
* `helm`

### 4.2 AWS Auth & Region

Set your region (e.g., `ap-south-1`) and verify:

```bash
export AWS_REGION=ap-south-1
export AWS_DEFAULT_REGION=ap-south-1
aws configure get region   # (may be empty, but env vars take precedence)
```

---

## 5. Step 1 – Create EKS Cluster for Elasticsearch

We’ll use `eksctl` with a simple config.

Create `cluster.yaml`:

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: efk-demo
  region: ap-south-1
  version: "1.30"

nodeGroups:
  - name: ng-1
    instanceType: t3.medium
    desiredCapacity: 3
    minSize: 2
    maxSize: 4
    volumeSize: 20
    ssh:
      allow: true

iam:
  withOIDC: true
```

### Why each field?

* **version 1.30** – recent k8s version compatible with drivers
* **t3.medium x3** – enough CPU/RAM for ES + demo app
* **withOIDC: true** – required for IAM Roles for Service Accounts (IRSA), used by CSI, ALB etc.

Create the cluster:

```bash
eksctl create cluster -f cluster.yaml
```

This takes some time. Once finished:

```bash
aws eks update-kubeconfig --name efk-demo --region ap-south-1
kubectl get nodes -o wide
kubectl get ns
```

You should see:

* 3 nodes in `Ready` state
* Namespaces: `default`, `kube-system`, `kube-public`, `kube-node-lease`

Create namespaces:

```bash
kubectl create namespace demo-app
kubectl create namespace logging
kubectl get ns
```

---

## 6. Step 2 – Deploy a Sample Microservice (for Logs)

We create a simple Nginx-based app that prints log lines every few seconds.

### 6.1 Deployment

`demo-app.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: demo-app
  labels:
    app: demo-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
      - name: demo-app
        image: nginx
        ports:
        - containerPort: 80
        args:
        - /bin/sh
        - -c
        - |
          while true; do
            echo "$(date) - INFO - Demo App Log: Request received!";
            sleep 5;
          done
```

### Why this design?

* 3 replicas → simulates a microservice with scaling
* Nginx + a simple shell loop that keeps writing to stdout → logs go to `/var/log/containers`, which Fluent Bit will later collect.

Apply:

```bash
kubectl apply -f demo-app.yaml
kubectl get pods -n demo-app
```

### 6.2 Service

`demo-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-service
  namespace: demo-app
spec:
  type: NodePort
  selector:
    app: demo-app
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

Apply:

```bash
kubectl apply -f demo-service.yaml
kubectl get svc -n demo-app
```

Now you have a microservice generating logs; we just haven’t wired them to Elasticsearch yet.

---

## 7. Step 3 – Storage Foundation for Elasticsearch

Elasticsearch is **stateful**. It must store data on **disks (EBS)**, not on ephemeral container filesystems.

So we need:

1. **EBS CSI Driver** – so Kubernetes can dynamically create EBS volumes
2. **IAM permissions** for the CSI driver (so it can call EC2 APIs)
3. A **StorageClass** (`gp3-storage`) using that driver

### 7.1 Install AWS EBS CSI Driver (Helm)

```bash
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update

helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  -n kube-system
```

Check:

```bash
kubectl get pods -n kube-system | grep ebs
```

You should see:

* `ebs-csi-controller-...`
* `ebs-csi-node-...`

Initially, your **controller** was crashing because it was missing IAM permissions. The logs clearly showed:

> `UnauthorizedOperation: ec2:DescribeAvailabilityZones`

So we fixed it by adding the correct IAM policy.

### 7.2 Attach IAM Policy to NodeInstanceRole

First, find your node IAM role:

```bash
aws iam list-roles | grep NodeInstanceRole
```

Example:

```text
eksctl-efk-demo-nodegroup-ng-1-NodeInstanceRole-Iau4CZ1UOT3e
```

Attach the AWS-managed EBS CSI policy:

```bash
aws iam attach-role-policy \
  --role-name eksctl-efk-demo-nodegroup-ng-1-NodeInstanceRole-Iau4CZ1UOT3e \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
```

Then restart the controller pods:

```bash
kubectl delete pod -n kube-system -l app=ebs-csi-controller
kubectl get pods -n kube-system | grep ebs
```

Now you should see:

```text
ebs-csi-controller-...   5/5   Running
ebs-csi-node-...         3/3   Running
```

> **Teaching Point:**
> The CSI driver is just another app in the cluster that needs AWS IAM permissions to talk to EC2 and create EBS volumes.

### 7.3 Create gp3 StorageClass

Now we define the storage “profile” ES will use.

`gp3-storage.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

Apply:

```bash
kubectl apply -f gp3-storage.yaml
kubectl get storageclass
```

You should see:

```text
gp3-storage   ebs.csi.aws.com   ...
gp2           kubernetes.io/aws-ebs   ...
```

> **Why `WaitForFirstConsumer`?**
> Kubernetes waits until a pod is scheduled on a node before deciding in which AZ to create the EBS volume. This prevents “volume in wrong AZ” issues.

---

## 8. Step 4 – Deploy Elasticsearch using ECK Operator

Instead of fighting with changing Helm charts and 8.x TLS/auth complexity, we use **ECK (Elastic Cloud on Kubernetes)**:

* Official from Elastic
* Manages:

  * Pods
  * Upgrades
  * Certificates
  * Credentials
* Very clean for EKS labs and real setups

### 8.1 Install ECK CRDs + Operator

```bash
kubectl apply -f https://download.elastic.co/downloads/eck/2.10.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.10.0/operator.yaml
```

Check:

```bash
kubectl get pods -n elastic-system
```

Expected:

```text
elastic-operator-0   1/1   Running
```

### 8.2 Define a Single-Node Elasticsearch Cluster

Create `elasticsearch.yaml`:

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: efk-es
  namespace: logging
spec:
  version: 7.17.0
  nodeSets:
  - name: default
    count: 1
    config:
      node.store.allow_mmap: false
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
        storageClassName: gp3-storage
```

Key points to explain:

* **version: 7.17.0**
  Stable, widely used, and simpler security model than 8.x for labs.

* **count: 1**
  Single node is fine for learning + small setups.

* **volumeClaimTemplates**
  ECK will automatically create a **PVC** named:
  `elasticsearch-data-efk-es-es-default-0`
  using the `gp3-storage` StorageClass, which in turn uses the EBS CSI driver.

Apply:

```bash
kubectl apply -f elasticsearch.yaml
```

Watch:

```bash
kubectl get pods -n logging
kubectl get pvc -n logging
```

What you should eventually see:

```text
NAME                                  STATUS   VOLUME        STORAGECLASS   AGE
elasticsearch-data-efk-es-es-default-0 Bound   pvc-xxxx...   gp3-storage    ...

NAME                  READY   STATUS    RESTARTS   AGE
efk-es-es-default-0   1/1     Running   0          Xm
```

> **Teaching Insight:**
> Initially, the pod was `Pending` with event
> *“pod has unbound immediate PersistentVolumeClaims”*
> This clearly means: PVC not yet bound → no volume → pod cannot start.
> Once we created the `gp3-storage` StorageClass and EBS CSI was fixed, the PVC became `Bound` and the pod started running.

---

## 9. Step 5 – Test Elasticsearch

### 9.1 Get Elastic User Password

ECK creates a secure cluster with a default `elastic` user.

Get the password:

```bash
kubectl get secret efk-es-es-elastic-user -n logging -o go-template='{{.data.elastic | base64decode}}'
echo
```

Copy it.

### 9.2 Port-Forward and Test

Forward the ES HTTP service:

```bash
kubectl port-forward service/efk-es-es-http -n logging 9200:9200
```

In another terminal:

```bash
curl -u "elastic:<PASSWORD_FROM_SECRET>" https://localhost:9200 -k
```

You should see basic cluster info JSON.

> **Explain `-k`:**
> We skip TLS verification because ECK uses a self-signed cert by default.

---

## 10. How This Connects to Microservice Monitoring

Right now:

* Your **demo-app** is generating logs in `demo-app` namespace
* Your **Elasticsearch** is running in `logging` namespace, ready to receive logs

When we add **Fluent Bit**:

* It will run as a **DaemonSet**, tail `/var/log/containers/*` on every node
* It will parse logs and send them to Elasticsearch over HTTP(S)
* We’ll store them in indices like `eks-logs-YYYY.MM.DD`
* Kibana will then query Elasticsearch and help you:

  * Search logs by service, pod, namespace, level (`INFO/ERROR`)
  * Build dashboards (errors per minute, latency, etc.)
  * Filter requests corresponding to incidents

This is exactly how you monitor microservice systems in production.

---

## 11. Helm vs ECK – What Should You Teach?

Your original requirement: **“Deploy Elasticsearch using Helm”**.

Given where the ecosystem is today:

* **Elastic’s Helm charts for 7.x** are no longer published in the official repo.
* 8.x charts + security + TLS + Kibana integration = **more complexity for students**.
* **Bitnami’s ES Helm chart** pulls images from Docker Hub and now hits:

  * Rate limits
  * Access/policy changes post-Aug 2025 (as you saw with `ImagePullBackOff`).

So for a **clean, modern, and “best practices” tutorial in 2025**, I would present it like this:

### Recommended in Class

* Use **ECK Operator** (what we just did)
  → Official, stable, works great with EKS + EBS

### Mention in Theory

* Helm-based installations exist (Elastic official chart, Bitnami chart), but:

  * Version mismatch issues (7.x vs 8.x)
  * Image hosting / rate-limiting challenges
  * More moving parts to debug

You *are* still using Helm in this end-to-end story:

* For **AWS EBS CSI Driver**
* For **(later) Fluent Bit**
* For **(optionally) Ingress controllers, etc.**

So students will see Helm in action, but Elasticsearch itself is better taught via ECK now.

---

Next you will see **Part 2 of the tutorial**:

* Deploy **Kibana** with ECK (LoadBalancer)
* Deploy **Fluent Bit** with Helm
* Wire everything and show **demo-app logs inside Kibana Discover**

For now, this document is the **“ElasticSearch on EKS – Part 1: Concepts + Deployment”**.
