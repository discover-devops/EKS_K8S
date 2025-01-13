### Setting Up a Private Node Group and Implementing a Classic Load Balancer in AWS EKS

#### Part 1: Deleting the Public Node Group

1. **List Existing Node Groups**  
   To list the current node groups in your EKS cluster, use the following command:  
   ```bash
   eksctl get nodegroup --cluster=my-lab-cluster --region=ap-south-1
   ```
   **Output:**
   ```
   CLUSTER         NODEGROUP       STATUS  CREATED                 MIN SIZE        MAX SIZE        DESIRED CAPACITY        INSTANCE TYPE IMAGE ID ASG NAME                                                TYPE
   my-lab-cluster  ng-5cd47d72     ACTIVE  2025-01-13T15:22:56Z    2               2               2                       t2.medium     AL2_x86_64       eks-ng-5cd47d72-8aca3021-5cd0-3917-fd62-792606e9b472    managed
   ```

2. **Delete the Public Node Group**  
   Replace `<NodeGroup-Name>` with the name of the node group to delete:  
   ```bash
   eksctl delete nodegroup --cluster=my-lab-cluster --region=ap-south-1 --name=ng-5cd47d72
   ```
   This removes the node group `ng-5cd47d72` from the EKS cluster.

---

#### Part 2: Creating a Private Node Group

1. **Create the Private Node Group**  
   Use the `--node-private-networking` option to create worker nodes in private subnets:  
   ```bash
   eksctl create nodegroup --cluster=my-lab-cluster \
                           --region=ap-south-1 \
                           --name=my-lab-private-ng \
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

2. **Verify Node Group Creation**  
   - Check the nodes using Kubernetes:  
     ```bash
     kubectl get nodes -o wide
     ```
     **Output Example:**
     ```
     NAME              STATUS   ROLES    AGE   VERSION   INTERNAL-IP    EXTERNAL-IP
     ip-10-0-1-25      Ready    <none>   5m    v1.22.9   10.0.1.25      <none>
     ip-10-0-2-15      Ready    <none>   5m    v1.22.9   10.0.2.15      <none>
     ```
     - The **EXTERNAL-IP** column should display `<none>`, confirming the nodes are in private subnets.

   - Verify in the AWS Management Console:  
     - Navigate to **EKS > my-lab-cluster > Compute > my-lab-private-ng**.
     - Check the **subnet** and confirm it is private.
     - Inspect the **Route Table** of the subnet to ensure outbound traffic routes via a **NAT Gateway**.

---

#### Part 3: Deploying the Application and Classic Load Balancer

1. **Deploy the Application**
   Apply the Kubernetes Deployment manifest:
   ```bash
   kubectl apply -f user-management-deployment.yaml
   ```

2. **Create the Classic Load Balancer**
   Apply the Service manifest to provision the Classic Load Balancer:
   ```bash
   kubectl apply -f user-management-service.yaml
   ```

3. **Verify the Load Balancer**
   - Use `kubectl get svc` to retrieve the external DNS of the Load Balancer:
     ```bash
     kubectl get svc user-management-service
     ```
   - Access the application using the DNS of the Load Balancer:
     ```bash
     http://<LoadBalancer-DNS>/user-management/health
     ```

---

### Summary

- Deleted the existing public node group (`ng-5cd47d72`) and replaced it with a private node group (`my-lab-private-ng`) for enhanced security.
- Deployed the **User Management Application** in the private subnets.
- Implemented a **Classic Load Balancer** to expose the application, ensuring high availability and secure communication.
