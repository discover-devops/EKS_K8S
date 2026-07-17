# 03 — hostPath: Node-Local Persistence and Its Trap

**Series:** Kubernetes Storage: From Ephemeral Containers to Persistent Data
**Format:** Concept + hands-on lab on your EKS cluster

---

## Recap from Module 02

`emptyDir` solves sharing data between containers *within* one pod, but its lifetime is tied exactly to that pod's lifetime — delete the pod, lose the data. We need something whose lifetime is genuinely independent of any single pod. `hostPath` is the next step toward that — but it comes with a very specific, easy-to-miss limitation.

---

## The analogy

Think of `hostPath` as a storage unit located in the basement of the specific apartment building you're renting in. It survives if you move out of your room and into a *different* room — **but only if your next room is in the exact same building.** Move to a different building entirely, and that basement storage unit might as well not exist for you anymore.

## The YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'echo "Data on host" > /host-data/file.txt && sleep 3600']
    volumeMounts:
    - name: host-storage
      mountPath: /host-data
  volumes:
  - name: host-storage
    hostPath:
      path: /tmp/kubernetes-data
      type: DirectoryOrCreate
```

**This mounts a real directory from the underlying EC2 node's own filesystem directly into the container.** No abstraction layer, no separate storage system — just a folder that already exists (or gets created) on that one specific machine.

**Think about this before continuing:** what happens if this pod gets deleted and recreated — once landing on the *same* node, and once landing on a *different* node?

---

## 🔧 Hands-on lab: prove the same-node vs. different-node behavior directly

Since Kubernetes' scheduler decides node placement for you, and might coincidentally reuse the same node on a simple delete/recreate, we'll **force** placement explicitly using `nodeName` — this makes the same-node and different-node cases fully deterministic and provable, rather than hoping the scheduler happens to demonstrate the point for us.

### Step 1: Get your two node names

```bash
kubectl get nodes
```

Note the two node names — we'll refer to them as `<NODE_A>` and `<NODE_B>` below. Export them:

```bash
export NODE_A=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
export NODE_B=$(kubectl get nodes -o jsonpath='{.items[1].metadata.name}')
echo "Node A: $NODE_A"
echo "Node B: $NODE_B"
```

### Step 2: Deploy the hostPath pod, pinned explicitly to Node A

```bash
cat <<EOF > hostpath-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  nodeName: ${NODE_A}
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'echo "Data written at: \$(date)" > /host-data/file.txt && sleep 3600']
    volumeMounts:
    - name: host-storage
      mountPath: /host-data
  volumes:
  - name: host-storage
    hostPath:
      path: /tmp/kubernetes-data
      type: DirectoryOrCreate
EOF

kubectl apply -f hostpath-pod.yaml
kubectl wait --for=condition=Ready pod/hostpath-pod --timeout=30s
```

### Step 3: Confirm the data, and confirm which node it's actually on

```bash
kubectl exec hostpath-pod -- cat /host-data/file.txt
kubectl get pod hostpath-pod -o wide
```

Note the timestamp and confirm `NODE` matches `$NODE_A`.

### Step 4: Delete the pod, then read back the SAME node's disk with a fresh pod

We deliberately don't just "recreate the same pod," since its `command` would overwrite the file with a new timestamp, hiding whether the *old* data actually survived. Instead, we delete it, then use a **read-only** pod pinned to the same node to check what's actually still sitting on disk:

```bash
kubectl delete pod hostpath-pod

cat <<EOF > hostpath-reader-samenode.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-reader-samenode
spec:
  nodeName: ${NODE_A}
  containers:
  - name: reader
    image: busybox
    command: ['sh', '-c', 'cat /host-data/file.txt 2>&1; sleep 3600']
    volumeMounts:
    - name: host-storage
      mountPath: /host-data
  volumes:
  - name: host-storage
    hostPath:
      path: /tmp/kubernetes-data
      type: DirectoryOrCreate
EOF

kubectl apply -f hostpath-reader-samenode.yaml
kubectl wait --for=condition=Ready pod/hostpath-reader-samenode --timeout=30s
kubectl logs hostpath-reader-samenode
```

**Expected: the original timestamp from Step 3 is still there.** Same node, same underlying directory on disk — the data survived pod deletion, because it was never inside the *pod* to begin with; it was always sitting on the *node's own disk*, at `/tmp/kubernetes-data`, completely independent of any pod's lifecycle.

### Step 5: Now the critical test — read the SAME path, pinned to the DIFFERENT node

```bash
cat <<EOF > hostpath-reader-diffnode.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-reader-diffnode
spec:
  nodeName: ${NODE_B}
  containers:
  - name: reader
    image: busybox
    command: ['sh', '-c', 'cat /host-data/file.txt 2>&1; sleep 3600']
    volumeMounts:
    - name: host-storage
      mountPath: /host-data
  volumes:
  - name: host-storage
    hostPath:
      path: /tmp/kubernetes-data
      type: DirectoryOrCreate
EOF

kubectl apply -f hostpath-reader-diffnode.yaml
kubectl wait --for=condition=Ready pod/hostpath-reader-diffnode --timeout=30s
kubectl logs hostpath-reader-diffnode
```

**Expected: `cat: can't open '/host-data/file.txt': No such file or directory`.** Same exact path (`/tmp/kubernetes-data/file.txt`), same exact YAML shape — but `Node B` has never heard of this file, because it's a genuinely different physical machine with its own, entirely separate disk. `DirectoryOrCreate` just silently created a fresh, empty directory at that path on Node B, since nothing existed there yet.

**This is the entire lesson of `hostPath`, proven with your own hands:** *the data followed the node, not the pod.*

---

## When does hostPath actually make sense?

- **Node-specific system logs** — reading logs that genuinely only exist on that one node (e.g., a log-shipping DaemonSet reading `/var/log` on each node it runs on)
- **Accessing host device files** — hardware-level access that's inherently tied to one physical machine
- **Testing and local development** — where node-locality genuinely doesn't matter because you're running a single-node setup anyway

## The security warning worth stating explicitly

**Why is `hostPath` considered a real security risk?** Because it gives a container direct access to the underlying node's filesystem — depending on which path is mounted, a compromised container could potentially read sensitive host files, or in worse misconfigurations, escalate privileges by writing to paths that affect the node itself (like binaries the kubelet trusts, or Docker's own socket if mounted). This is exactly the kind of access `NetworkPolicy` (from the earlier module of this course) has no power over — `hostPath` is a *filesystem*-level concern, not a *network*-level one, and it needs to be restricted through Pod Security Standards / admission controls instead.

---

## Cleanup

```bash
kubectl delete pod hostpath-pod hostpath-reader-samenode hostpath-reader-diffnode --ignore-not-found
```

---

## Summary

| Scenario | Data survives? |
|---|---|
| Container restarts, same pod | Yes |
| Pod deleted and recreated, **same node** | Yes |
| Pod deleted and recreated, **different node** | **No** |

**The one sentence to carry forward:** *"`hostPath` genuinely breaks the pod-lifetime limitation `emptyDir` had — but it silently replaces it with a node-lifetime limitation instead, and in a dynamic cluster where pods reschedule onto whichever node has room, that's not a safe bet for anything you actually care about. We need storage that follows the workload regardless of which node it lands on — which is exactly what PersistentVolumes and PersistentVolumeClaims solve."*

---

*End of Module 03. Module 04: The PersistentVolume Abstraction — the real production answer.*
