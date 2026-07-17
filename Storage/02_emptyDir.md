# 02 — emptyDir: The First Kubernetes Volume Type

**Series:** Kubernetes Storage: From Ephemeral Containers to Persistent Data
**Format:** Concept + hands-on lab on your EKS cluster

---

## Recap from Module 01

We proved that data written directly to a container's own filesystem vanishes the moment the pod is deleted — because that data never lived anywhere except the container's own disposable writable layer. We need something that exists as a genuinely separate object.

`emptyDir` is the first, simplest answer Kubernetes offers — and understanding exactly where its guarantees end is just as important as understanding what it solves.

---

## Before Kubernetes volumes: what about Docker volumes?

If you've used Docker before, you already know Docker has its own volume concept — `docker volume create`, then mount it into a container. Reasonable question: **if Kubernetes pods can restart on entirely different physical nodes, how would a Docker-style volume even help?**

**The limitation:** a Docker volume lives on one specific machine's disk. If a Kubernetes pod gets rescheduled to a *different* node — which happens routinely, for all sorts of reasons outside your control — a node-local volume simply isn't there anymore on the new node. **This is exactly why Kubernetes needed to invent its own volume abstractions, rather than just reusing Docker's directly** — the unit of scheduling in Kubernetes is far more fluid than "this container always runs on this one machine."

---

## Volume Type 1: emptyDir

### The analogy

Think of `emptyDir` as the safe inside your hotel room. If two people are sharing that room — two containers in one Pod — they can both use the same safe to pass documents back and forth. But the moment you check out of the room entirely (the Pod is deleted), hotel management empties the safe completely. It was never meant to survive your stay; it was only ever meant to survive you walking between the bed and the desk.

### The YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-pod
spec:
  containers:
  - name: writer
    image: busybox
    command: ['sh', '-c', 'echo "Hello from writer" > /data/message.txt && sleep 3600']
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: reader
    image: busybox
    command: ['sh', '-c', 'sleep 30 && cat /data/message.txt && sleep 3600']
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    emptyDir: {}
```

**Before running this — predict the outcome.** Two entirely separate containers, `writer` and `reader`, in the same Pod. Will `reader` actually be able to see the file `writer` created?

---

## 🔧 Hands-on lab

### Step 1: Deploy and verify

```bash
cat <<'EOF' > shared-volume-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-pod
spec:
  containers:
  - name: writer
    image: busybox
    command: ['sh', '-c', 'echo "Hello from writer" > /data/message.txt && sleep 3600']
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: reader
    image: busybox
    command: ['sh', '-c', 'sleep 30 && cat /data/message.txt && sleep 3600']
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    emptyDir: {}
EOF

kubectl apply -f shared-volume-pod.yaml
kubectl wait --for=condition=Ready pod/shared-volume-pod --timeout=60s
```

```bash
kubectl logs shared-volume-pod -c reader
```

Expected: `Hello from writer`

**It worked** — two entirely separate containers, running separate processes, with no direct communication between them, both seeing the exact same file. That's `emptyDir` doing its one job: giving multiple containers *within one pod* a shared filesystem.

### Step 2: The critical question — what happens when the Pod itself is deleted?

**Predict before running:** does `emptyDir` data survive pod deletion, the same way it survived being read by a second container?

```bash
kubectl delete pod shared-volume-pod
kubectl apply -f shared-volume-pod.yaml
kubectl wait --for=condition=Ready pod/shared-volume-pod --timeout=60s
kubectl logs shared-volume-pod -c reader
```

Expected: still shows `Hello from writer` — but **this is newly written data from this fresh pod's `writer` container, not data that survived from the deleted pod.** The `writer` container's command runs the exact same `echo` command every single time the pod starts, so it *looks* identical, but it's a brand new safe with brand new contents, not the same one carried over.

### Proving this distinction concretely

To make the "new data, not surviving data" point unmistakable, let's write something with a timestamp instead of a fixed string:

```bash
kubectl exec shared-volume-pod -c writer -- sh -c 'echo "Written at: $(date)" >> /data/timestamped.txt'
kubectl exec shared-volume-pod -c reader -- cat /data/timestamped.txt
```

Note the timestamp. Now delete and recreate:

```bash
kubectl delete pod shared-volume-pod
kubectl apply -f shared-volume-pod.yaml
kubectl wait --for=condition=Ready pod/shared-volume-pod --timeout=60s
kubectl exec shared-volume-pod -c reader -- cat /data/timestamped.txt 2>&1
```

Expected: `cat: can't open '/data/timestamped.txt': No such file or directory` — **this file is genuinely gone**, proving `emptyDir`'s contents die with the pod, exactly as the hotel-safe analogy describes.

---

## When is emptyDir actually useful, given it doesn't survive pod restarts?

Sit with this before reading on: if it can't persist anything, what's it actually for?

- **Scratch space** — sorting/processing large temporary files mid-request, where losing them on restart is completely fine
- **Caching** — a local cache that's fine to rebuild from scratch if the pod restarts
- **Sharing data between containers in the same pod** — exactly what we just proved, e.g., a main app container writing logs and a sidecar container shipping them elsewhere

---

## Pro-tip: emptyDir can be backed by RAM instead of disk

```yaml
volumes:
- name: shared-data
  emptyDir:
    medium: Memory
```

This is a genuine SRE technique — used for high-performance caches where disk I/O latency actually matters, or for sensitive data (like a decrypted secret used transiently) that you specifically want to guarantee **never touches physical disk at all**, not even temporarily.

### 🔧 Try it

```bash
cat <<'EOF' > ram-backed-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: ram-backed-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'echo "in-memory data" > /cache/data.txt && sleep 3600']
    volumeMounts:
    - name: cache
      mountPath: /cache
  volumes:
  - name: cache
    emptyDir:
      medium: Memory
EOF

kubectl apply -f ram-backed-pod.yaml
kubectl wait --for=condition=Ready pod/ram-backed-pod --timeout=30s
kubectl exec ram-backed-pod -- df -h /cache
```

Look at the filesystem type in the output — a RAM-backed `emptyDir` shows up as `tmpfs`, confirming it's genuinely living in memory, not on the node's disk.

---

## Cleanup

```bash
kubectl delete pod shared-volume-pod ram-backed-pod --ignore-not-found
```

---

## Summary

| Question | Answer |
|---|---|
| Survives container restart within the same pod? | Yes |
| Shared across multiple containers in one pod? | Yes |
| Survives pod deletion? | No |
| Survives pod rescheduling to a different node? | No |
| Good for | Scratch space, caching, inter-container sharing within a pod |
| Not good for | Anything you can't afford to lose |

**The one sentence to carry forward:** *"`emptyDir` solves the multi-container sharing problem, but it inherits the exact same lifetime as the pod itself — which means it solves nothing for our Module 00 database disaster. We need a volume whose lifecycle is independent of the pod, not just independent of the container."*

---

*End of Module 02. Module 03: hostPath — closer to real persistence, but with a trap of its own.*
