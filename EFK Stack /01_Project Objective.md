 **“EFK Stack on AWS EKS”** lab.

---

## 1. Project Objective (One Line)

**Build an end-to-end logging pipeline on AWS EKS where a sample microservice runs on the cluster, its logs are collected by Fluent Bit/Fluentd, stored in Elasticsearch, and visualized in Kibana — all deployed using Helm.**

---

## 2. Why This Project Matters

In any real Kubernetes environment, you need:

* **Centralized logs** (not `kubectl logs` on random pods)
* **Searchable & filterable data** (by app, namespace, severity)
* **Dashboards & visual insights** (errors, latencies, trends)
* **Scalable & production-like setup**

This project gives you a **mini production-style logging setup** on EKS:

* EKS = real managed K8s on AWS
* EFK = classic logging stack used in enterprises
* Helm = industry-standard way to package & deploy K8s apps

Perfect to showcase in **interviews, demos, or internal trainings**.

---

## 3. High-Level Architecture

**Components:**

1. **AWS EKS Cluster**

   * Managed control plane
   * Worker nodes where pods run

2. **Sample Microservice (Demo App)**

   * Simple HTTP service (e.g., Node.js / Python / Go / Java)
   * Exposes `/api/hello`, `/health` etc.
   * Prints structured logs (`INFO`, `ERROR`, request IDs) to stdout

3. **EFK Stack (via Helm)**

   * **Elasticsearch**: Stores logs (data layer)
   * **Fluent Bit / Fluentd**: Log collector & forwarder (agent on each node)
   * **Kibana**: UI for searching & visualizing logs

4. **AWS Infrastructure Around It (Optional in v1)**

   * IAM roles for EKS nodes / pods
   * EBS volumes for Elasticsearch storage
   * NLB/ALB / Ingress for Kibana access

---

## 4. Logical Flow of Logs

1. **App generates logs**

   * Your microservice writes logs to `stdout` (and/or a file in `/var/log/...`).

2. **Kubelet writes pod logs to node filesystem**

   * By default: `/var/log/containers/*.log`

3. **Fluent Bit / Fluentd DaemonSet**

   * Runs on every node
   * Tails container log files
   * Parses app / Kubernetes metadata (namespace, pod name, labels)
   * Forwards logs to Elasticsearch using HTTP/ES protocol.

4. **Elasticsearch**

   * Receives logs as **documents**
   * Indexes them into indices (e.g., `logs-<namespace>-yyyy.MM.dd`)

5. **Kibana**

   * Connects to Elasticsearch
   * You configure an **index pattern** (e.g., `logs-*`)
   * Use Discover, Filters, and Dashboards to see:

     * Logs by app, namespace, log level
     * Error spikes over time
     * Top endpoints / services throwing errors

---

## 5. What You Will Demonstrate End-to-End

In this project you will be able to **showcase**:

1. **Kubernetes + EKS skills**

   * Creating namespaces
   * Deployments, Services
   * NodeGroups, context switching

2. **Helm skills**

   * Adding Helm repos
   * Installing and upgrading charts
   * Using `values.yaml` to customize:

     * Storage for Elasticsearch
     * Resource requests/limits
     * Kibana service type (NodePort / LoadBalancer / Ingress)
     * Fluent Bit / Fluentd filters & parsers

3. **Observability & Logging**

   * End-to-end log pipeline
   * Index patterns and queries in Kibana
   * Filtering logs by:

     * `kubernetes.namespace_name`
     * `kubernetes.labels.app`
     * `log.level`

4. **AWS Awareness (Optional Enhancements)**

   * Using EBS for ES data nodes
   * Using an ALB Ingress Controller for Kibana & sample app
   * Securing Kibana behind basic auth / security group / IP whitelist

---

## 6. Project Structure (Conceptual)

You can think of the project like this:

1. **Cluster & Base Setup**

   * Create / reuse EKS cluster
   * Configure `kubectl` & `aws-auth`
   * Create `logging` and `demo-app` namespaces

2. **Deploy Sample Microservice**

   * Simple Deployment + Service (NodePort or ClusterIP + Ingress)
   * The app must generate logs frequently (e.g., on every HTTP request)
   * Add labels like `app: demo-api` to identify in Kibana

3. **Deploy EFK via Helm**

   * **Phase 1: Elasticsearch**

     * Deploy with Helm chart
     * Configure persistent storage (EBS)
   * **Phase 2: Kibana**

     * Deploy Kibana and expose it (NodePort/LoadBalancer/Ingress)
   * **Phase 3: Fluent Bit / Fluentd**

     * Deploy as DaemonSet via Helm
     * Configure input (container logs) and output (Elasticsearch URL)
     * Add parsers/filters as needed

4. **Verification & Observability**

   * Hit the microservice endpoint using `curl` or browser
   * Confirm logs appear in Kibana:

     * Set index pattern
     * Run search queries,
     * Add simple visualizations (bar chart: logs per app, per level)

5. **Cleanup (Optional but Good Practice)**

   * Delete Helm releases
   * Delete namespaces
   * (Optionally) delete EKS cluster to avoid cost

---

## 7. Learning Outcomes

By completing this project, you’ll be able to:

* Explain **how logs travel** from Pods → Node → Fluent Bit → Elasticsearch → Kibana.
* Compare EFK with **CloudWatch-only** logging.
* Show a **real EKS + Helm + EFK** stack running end-to-end.
* Use this as a **portfolio project** or interview discussion:

  * “I built a complete EFK logging solution on EKS using Helm.”

---


