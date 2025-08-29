
---

# Lesson 2: Build a Private Node Group on EKS (Foundation for Classic/NLB)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/013f17c3-961a-4960-80bd-570ad5bb7579" />


##  Learning objectives

* Understand **why** we move worker nodes to **private subnets** before using CLB/NLB.
* Create an **EKS managed node group** in private subnets with `eksctl`.
* Verify nodes have **no public IPs** and route **egress via NAT**.

---

##  Context (from Lesson 1)

* With `Service: LoadBalancer`, EKS creates a **cloud load balancer** (CLB by default).
* For production, apps should run on **private subnets**; only the LB is **internet‑facing**.
* Private nodes improve security and push all inbound traffic through **standard ports** (80/443) on the LB, not via random NodePort ranges.

---

##  Concept: What is a “private node group” and why?

* A **private node group** launches worker nodes **without public IPs**, in **private subnets**.
* **Inbound**: Users hit the **public Load Balancer**, which forwards to Services → Pods on private nodes.
* **Control‑plane & egress**: Nodes reach the EKS API, ECR, etc., via **NAT Gateway** (or VPC endpoints if configured).
* Benefits:

  * Smaller attack surface (no SSH/ingress to nodes from the internet).
  * Production‑aligned network design (LB public; workloads private).
  * Cleaner path to CLB/NLB now, ALB/Ingress later.

---

##  LAB: Create EKS Node Group in Private Subnets

### Step‑00: Prereqs (assumptions)

* Cluster exists: `eksdemo1` (region example: `us-east-1`)
* CLI tools installed & configured: **AWS CLI v2**, **kubectl**, **eksctl**
* You have an **EC2 key pair** (e.g., `kube-demo`) if you enable `--ssh-access`

---

### Step‑01: List current node groups

```bash
eksctl get nodegroup --cluster=eksdemo1
```

> You’ll likely see a public node group (e.g., `eksdemo1-ng-public1`).

---

### Step‑02 (Optional but safe): Drain workloads on the public nodes

If anything important is running, cordon & drain first:

```bash
kubectl get nodes
kubectl cordon <node-name>
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

---

### Step‑03: Delete the **public** node group

```bash
eksctl delete nodegroup eksdemo1-ng-public1 --cluster eksdemo1
```

> Wait for completion. Your control plane stays; only worker nodes are removed.

---

### Step‑04: Create a **private** managed node group

Key flag: `--node-private-networking`

```bash
eksctl create nodegroup --cluster=eksdemo1 \
  --region=us-east-1 \
  --name=eksdemo1-ng-private1 \
  --node-type=t3.medium \
  --nodes-min=2 \
  --nodes-max=4 \
  --node-volume-size=20 \
  --ssh-access \
  --ssh-public-key=kube-demo \
  --managed \
  --asg-access \
  --external-dns-access \
  --full-ecr-access \
  --appmesh-access \
  --alb-ingress-access \
  --node-private-networking
```

> This schedules the node group in your **cluster’s private subnets** and ensures nodes have **no public IPs**.

---

### Step‑05: Verify nodes are private (no public IP)

```bash
kubectl get nodes -o wide
```

* **EXTERNAL-IP** should show **\<none>**
* **INTERNAL-IP** should be a VPC private address (e.g., 10.x/172.31.x)

---

### Step‑06: (Console) Verify subnets/route tables

* AWS Console → **EKS** → Cluster `eksdemo1` → **Compute** → `eksdemo1-ng-private1`
* Open any **associated subnet**:

  * In **Route Table**, default route `0.0.0.0/0` should go to a **NAT Gateway** (private subnet behavior).
  * Public subnets would show an **Internet Gateway** route—private subnets should **not**.

---

##  Troubleshooting tips

* **Node group created but no Ready nodes?**

  * Check IAM permissions for the node group role.
  * Ensure subnets are **tagged** for EKS (eksctl usually handles this).
  * Confirm NAT Gateway exists and private subnets route to it (or use **VPC endpoints** to reduce NAT costs).
* **kubectl can’t talk to cluster?**

  * Update kubeconfig: `aws eks update-kubeconfig --name eksdemo1 --region us-east-1`
  * Ensure the **cluster endpoint** is accessible (public or private as per your design).
* **No images pulling from ECR?**

  * Confirm `--full-ecr-access` or attach ECR permissions; ensure NAT/VPC endpoint path to ECR.

---

##  What’s next (teaser for Lesson 3)

* Now that workloads are on **private nodes**, we’ll expose an app with:

  * **Classic Load Balancer (CLB)** via `Service: LoadBalancer`, and
  * **Network Load Balancer (NLB)** via annotation—compare behavior & performance.
* We’ll verify the LB created in **EC2 → Load Balancers**, hit the **DNS name**, and confirm end‑to‑end flow.

---

