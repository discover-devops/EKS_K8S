 **Part-2 Lab** written exactly in the same clear, instructor-friendly style as your **Helm + IRSA EBS CSI Driver** lab.
This lab demonstrates **dynamic provisioning of Amazon EBS volumes in an EKS cluster** using the driver you just installed.

---

# **Lab 2: Dynamically Provisioning Amazon EBS Storage on AWS EKS**

---

##  Lab Overview

In this lab, you will:

1. Create a **StorageClass** that uses the AWS EBS CSI Driver
2. Create a **PersistentVolumeClaim (PVC)** requesting storage
3. Deploy a **test Pod** that mounts the PVC
4. Verify that data is stored on the EBS volume and persists across Pod restarts
5. (Optional) Clean up the created resources

---

##  Prerequisites

Before you begin, ensure that:

* Your **EKS cluster** is up and running
* The **EBS CSI Driver** is already installed (via Helm + IRSA from Lab 1)
* You can run:

  ```bash
  kubectl get nodes
  kubectl get csidrivers
  kubectl get pods -n kube-system | grep ebs
  ```

  All driver pods should be **Running**

---

##  Step 1 — Create a StorageClass

A **StorageClass** defines how Kubernetes dynamically provisions storage volumes.

### 1.1 Create `storageclass.yaml`

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: gp3
  fsType: ext4
```

> **Notes**
>
> * `gp3` – default general-purpose SSD type
> * `WaitForFirstConsumer` – ensures the EBS volume is created in the same AZ as the pod
> * `allowVolumeExpansion` – lets you resize later

### 1.2 Apply and Verify

```bash
kubectl apply -f storageclass.yaml
kubectl get storageclass
```

You should see:

```
NAME       PROVISIONER           AGE
ebs-gp3    ebs.csi.aws.com       5s
```

---

##  Step 2 — Create a PersistentVolumeClaim (PVC)

A **PVC** lets a pod request a storage volume from the StorageClass.

### 2.1 Create `pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: ebs-gp3
```

### 2.2 Apply and Verify

```bash
kubectl apply -f pvc.yaml
kubectl get pvc
```

Output:

```
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
app-data   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   5Gi        RWO            ebs-gp3        4s
```

Kubernetes automatically created a **PersistentVolume (PV)** backed by an **EBS volume** in AWS.

---

##  Step 3 — Deploy a Test Pod

### 3.1 Create `test-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ebs-test-pod
spec:
  containers:
    - name: nginx
      image: nginx:stable
      volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: app-data
```

### 3.2 Apply and Verify

```bash
kubectl apply -f test-pod.yaml
kubectl get pods
```

Expected:

```
NAME           READY   STATUS    RESTARTS   AGE
ebs-test-pod   1/1     Running   0          20s
```

---

##  Step 4 — Test EBS Persistence

### 4.1 Write Data

```bash
kubectl exec -it ebs-test-pod -- sh
echo "Hello from EBS dynamic volume!" > /usr/share/nginx/html/index.html
cat /usr/share/nginx/html/index.html
exit
```

### 4.2 Delete and Recreate the Pod

```bash
kubectl delete pod ebs-test-pod
kubectl apply -f test-pod.yaml
```

### 4.3 Verify Data Still Exists

```bash
kubectl exec -it ebs-test-pod -- cat /usr/share/nginx/html/index.html
```

 Output:

```
Hello from EBS dynamic volume!
```

This confirms that data persisted on the EBS volume even though the pod was deleted.

---

## 🔍 Step 5 — Check the EBS Volume in AWS

List volumes created by the CSI driver:

```bash
aws ec2 describe-volumes \
  --filters "Name=tag:kubernetes.io/created-for/pvc/name,Values=app-data" \
  --query "Volumes[*].{ID:VolumeId,State:State,AZ:AvailabilityZone,Size:Size}"
```

You should see one volume tagged with your PVC name.

---

##  Step 6 — Cleanup (Optional)

If you want to remove all resources:

```bash
kubectl delete pod ebs-test-pod
kubectl delete pvc app-data
kubectl delete storageclass ebs-gp3
```

(Deleting the PVC also deletes the associated EBS volume.)

---

##  Lab Summary

| Component             | Purpose                                             |
| --------------------- | --------------------------------------------------- |
| **StorageClass**      | Defines how EBS volumes are provisioned dynamically |
| **PVC**               | Requests persistent storage from the StorageClass   |
| **PV (auto-created)** | Represents the actual EBS volume                    |
| **Pod**               | Mounts the PVC and writes persistent data           |

---

##  Key Learning Points

1. **Dynamic provisioning** eliminates the need to manually create PV/EBS volumes.
2. The **EBS CSI Driver** communicates with AWS APIs to provision, attach, and detach volumes automatically.
3. The **StorageClass → PVC → PV** chain is how Kubernetes abstracts cloud storage.
4. Data in EBS persists across Pod restarts and deletions as long as the PVC remains.
5. Deleting a PVC automatically deletes its backing EBS volume (unless the reclaim policy is set to Retain).

---

##  Extension Exercise (for students)

* Modify the StorageClass to use `gp2` or `io1` and note the difference in provisioning time.
* Set `allowVolumeExpansion: true` and try resizing the PVC from 5 Gi → 10 Gi.
* Use a Deployment or StatefulSet instead of a single Pod.
* Check volume tags in AWS to see how Kubernetes labels EBS resources.

---

###  End of Lab 2 — Dynamic EBS Provisioning on EKS
