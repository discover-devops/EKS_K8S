# Step-by-Step Guide: Demonstrating Ingress on AWS EKS with a Sample Microservice

This tutorial will guide you through deploying a sample microservice application to an Amazon Elastic Kubernetes Service (EKS) cluster and configuring an Ingress to manage external access to its services.

## Introduction to Ingress and Ingress Controllers

### What is Ingress?

In Kubernetes, an **Ingress** is an API object that manages external access to the services within a cluster, typically for HTTP and HTTPS traffic. It acts as a smart router or an entry point, allowing you to define rules for routing traffic from outside the cluster to services inside the cluster. This can include:

-   **Host-based routing:** Directing traffic for `api.example.com` to one service and `frontend.example.com` to another.
-   **Path-based routing:** Directing traffic for `example.com/api` to the API service and `example.com/` to the frontend service.
-   **SSL/TLS Termination:** Handling HTTPS traffic by terminating the SSL connection at the Ingress point, offloading the work from your application pods.

Without an Ingress, you would typically expose services using a `LoadBalancer` type service, which creates a separate, dedicated load balancer for each service. This can be inefficient and costly.

### What is an Ingress Controller?

An Ingress resource on its own doesn't do anything. You need an **Ingress Controller** running in your cluster to make it work.

The Ingress Controller is a pod (or a set of pods) that constantly watches the Kubernetes API server for Ingress resources. When it sees an Ingress resource, it reads the routing rules defined in it and configures a load balancer (like an AWS Application Load Balancer, NGINX proxy, etc.) to match those rules.

For AWS EKS, the recommended Ingress Controller is the **AWS Load Balancer Controller**. This controller specifically manages AWS Application Load Balancers (ALBs) and Network Load Balancers (NLBs) to handle traffic defined by Ingress and Service resources.

## Prerequisites

Before you begin, ensure you have the following:

1.  **An AWS Account:** You will need an active AWS account with permissions to create and manage EKS clusters, IAM roles, and Application Load Balancers.
2.  **A running EKS Cluster:** This tutorial assumes you have a functional EKS cluster. If you don't, you can create one using `eksctl` or the AWS Management Console.
3.  **`kubectl` installed and configured:** Your `kubectl` command-line tool must be configured to communicate with your EKS cluster. You can verify this by running `kubectl get nodes`.
4.  **AWS CLI installed and configured:** The AWS Command Line Interface (CLI) should be installed and configured with credentials that have the necessary permissions. Verify by running `aws sts get-caller-identity`.
5.  **`helm` installed:** Helm is a package manager for Kubernetes that simplifies the deployment of applications and services. We will use it to install the AWS Load Balancer Controller.
6.  **`eksctl` (recommended):** `eksctl` is a simple CLI tool for creating and managing clusters on EKS. It is useful for setting up the necessary IAM OIDC provider for your cluster.

## Step 1: Deploy the Sample Microservice Application

### Application Overview

Our sample application consists of two simple web services:
-   **greeter-service**: A service that returns a simple greeting message.
-   **admin-service**: A service that returns an admin-specific message.

We will deploy these as separate Deployments and expose them internally with ClusterIP Services.

### Create a Namespace

First, let's create a dedicated namespace for our application to keep things organized.

```bash
kubectl create namespace ingress-demo
```

### Application Manifests

I'll create a file named `sample-app.yaml` and paste the following content into it.

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: greeter-deployment
  namespace: ingress-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: greeter
  template:
    metadata:
      labels:
        app: greeter
    spec:
      containers:
      - name: greeter
        image: hashicorp/http-echo
        args:
        - "-text=Hello, this is the Greeter Service!"
        ports:
        - containerPort: 5678

---
apiVersion: v1
kind: Service
metadata:
  name: greeter-service
  namespace: ingress-demo
spec:
  selector:
    app: greeter
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5678

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admin-deployment
  namespace: ingress-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: admin
  template:
    metadata:
      labels:
        app: admin
    spec:
      containers:
      - name: admin
        image: hashicorp/http-echo
        args:
        - "-text=Welcome, this is the Admin Service!"
        ports:
        - containerPort: 5678

---
apiVersion: v1
kind: Service
metadata:
  name: admin-service
  namespace: ingress-demo
spec:
  selector:
    app: admin
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5678
```

### Deploy the Application

Now, apply the manifest to your EKS cluster:

```bash
kubectl apply -f sample-app.yaml
```

### Verify the Deployment

Check that the pods and services are running correctly:

```bash
# Check the pods in the ingress-demo namespace
kubectl get pods -n ingress-demo

# Check the services
kubectl get services -n ingress-demo
```

You should see two pods running for each deployment (`greeter-deployment` and `admin-deployment`) and two `ClusterIP` services (`greeter-service` and `admin-service`). At this point, the services are only accessible from within the cluster.

## Step 2: Install the AWS Load Balancer Controller

The AWS Load Balancer Controller is what connects the Kubernetes Ingress resource to a physical AWS Application Load Balancer (ALB).

### Create an IAM OIDC Provider for Your Cluster

The controller needs to authenticate with AWS APIs to create and manage ALBs. The recommended way to do this is by using an IAM OIDC provider with your EKS cluster.

You can create one for your cluster using `eksctl`:

```bash
# Replace 'your-cluster-name' with the actual name of your EKS cluster
eksctl utils associate-iam-oidc-provider --region=us-west-2 --cluster=your-cluster-name --approve
```

### Create the IAM Policy and Service Account for the Controller

Next, we need to create an IAM policy with the required permissions and a Kubernetes service account for the controller, which will be annotated with the IAM role.

1.  **Download the IAM policy document:**

    ```bash
    curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.7/docs/install/iam_policy.json
    ```

2.  **Create the IAM policy:**

    ```bash
    aws iam create-policy \
        --policy-name AWSLoadBalancerControllerIAMPolicy \
        --policy-document file://iam_policy.json
    ```
    *Note the ARN of the policy that is output.*

3.  **Create the service account using `eksctl`:**

    Replace `your-cluster-name` and the policy ARN with your own values.

    ```bash
    eksctl create iamserviceaccount \
      --cluster=your-cluster-name \
      --namespace=kube-system \
      --name=aws-load-balancer-controller \
      --attach-policy-arn=arn:aws:iam::111122223333:policy/AWSLoadBalancerControllerIAMPolicy \
      --override-existing-serviceaccounts \
      --approve
    ```

### Install the Controller Using Helm

With the IAM roles and service account in place, we can now install the controller using Helm.

1.  **Add the EKS chart repository:**

    ```bash
    helm repo add eks https://aws.github.io/eks-charts
    ```

2.  **Update the repository:**

    ```bash
    helm repo update
    ```

3.  **Install the Helm chart:**

    Replace `your-cluster-name` with your cluster's name.

    ```bash
    helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
      -n kube-system \
      --set clusterName=your-cluster-name \
      --set serviceAccount.create=false \
      --set serviceAccount.name=aws-load-balancer-controller 
    ```

### Verify the Controller Installation

Check that the controller deployment is running:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

You should see the `aws-load-balancer-controller` deployment with `2/2` replicas ready.

## Step 3: Configure and Demonstrate Ingress

Now that the sample application is running and the Ingress controller is installed, we can create an Ingress resource to expose the application to external traffic.

### Create the Ingress Manifest

I'll create a file named `sample-ingress.yaml` and add the following content.

This manifest defines an Ingress object that will provision an AWS Application Load Balancer and configure it with rules for path-based routing.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sample-ingress
  namespace: ingress-demo
  annotations:
    # Specify the Ingress Class
    kubernetes.io/ingress.class: alb
    # Specify that this ALB is internet-facing
    alb.ingress.kubernetes.io/scheme: internet-facing
    # Specify the target type for the ALB
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: greeter-service
                port:
                  number: 80
          - path: /admin
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 80
```

### Key Annotations Explained:

-   `kubernetes.io/ingress.class: alb`: This tells the AWS Load Balancer Controller that it should handle this Ingress resource.
-   `alb.ingress.kubernetes.io/scheme: internet-facing`: This annotation creates a public, internet-facing ALB. For internal services, you would use `internal`.
-   `alb.ingress.kubernetes.io/target-type: ip`: This specifies that the ALB should route traffic directly to the pod IPs, which is more efficient than routing to NodePorts.

### Deploy the Ingress Resource

Apply the Ingress manifest to your cluster:

```bash
kubectl apply -f sample-ingress.yaml
```

## Step 4: Verify the Ingress Setup

After applying the Ingress manifest, the AWS Load Balancer Controller will start provisioning an Application Load Balancer in your AWS account. This can take a few minutes.

### Check the Ingress Status

To find the public DNS name of the ALB, inspect the Ingress resource:

```bash
kubectl get ingress sample-ingress -n ingress-demo -w
```

Wait for the `ADDRESS` field to be populated with a DNS name. It will look something like `k8s-ingres-xxxxxxxxxx-1234567890.us-west-2.elb.amazonaws.com`.

### Test the Routing Rules

Once the address is available, you can test the Ingress routing using `curl` or your web browser.

1.  **Test the root path (`/`):**

    This should route to the `greeter-service`.

    ```bash
    curl http://<your-alb-address>/
    ```

    **Expected Output:**
    ```
    Hello, this is the Greeter Service!
    ```

2.  **Test the `/admin` path:**

    This should route to the `admin-service`.

    ```bash
    curl http://<your-alb-address>/admin
    ```

    **Expected Output:**
    ```
    Welcome, this is the Admin Service!
    ```

If you receive these responses, your Ingress and Ingress Controller are working correctly! You have successfully exposed your microservices to external traffic using path-based routing.

## Step 5: Cleanup

To avoid incurring ongoing charges, it's important to clean up the resources you created.

### Delete the Application and Ingress Resources

First, delete the Ingress and the sample application by deleting the namespace:

```bash
kubectl delete namespace ingress-demo
```
This will remove the Ingress, Deployments, and Services you created. The AWS Load Balancer Controller will also automatically delete the corresponding ALB from your AWS account.

### Uninstall the AWS Load Balancer Controller

Next, uninstall the controller using Helm:

```bash
helm uninstall aws-load-balancer-controller -n kube-system
```

### Delete the IAM Policy

Finally, delete the IAM policy you created. You will need the policy ARN you noted earlier.

```bash
aws iam delete-policy --policy-arn <your-policy-arn>
```

By following these steps, you have successfully cleaned up all the resources created in this tutorial.
