# 01 — Container Filesystem Ephemerality

**Series:** Kubernetes Storage: From Ephemeral Containers to Persistent Data
**Format:** Concept + hands-on lab on your EKS cluster

---

## Recap from Module 00

Containers are ephemeral by design. This module proves exactly what that means, mechanically, with your own hands on a real cluster — not as something to take on faith.

---

## The data lifecycle problem, laid out step by step

**Scenario 1 — a container with no persistent storage:**

```
Pod Created (T+0s)
  Container starts with a fresh filesystem
  Application writes data to /var/lib/postgresql/data

Application Running (T+0 to T+100)
  Data accumulates — tables, indexes, WAL files
  All of it stored in the container's own writable layer

Pod Deleted / Crashes (T+100)
  Container filesystem destroyed

Pod Recreated (T+101)
  New container, fresh filesystem
  Application starts
  Looks for its data... WHERE IS IT?

Result: DATA LOST
```

**Scenario 2 — the same workload, but with a real volume mounted:**

```
Pod Created (T+0s)
  Container starts with a fresh filesystem
  A volume is mounted at /var/lib/postgresql/data
  That volume is SEPARATE from the container's own filesystem

Application Running (T+0 to T+100)
  Data written to the mounted volume
  The volume lives OUTSIDE the container's lifecycle entirely

Pod Deleted / Crashes (T+100)
  Container filesystem destroyed
  Volume remains completely intact

Pod Recreated (T+101)
  New container, fresh filesystem
  The SAME volume gets mounted at the same path again
  Application starts, looks for its data... it's all still there

Result: DATA PERSISTS
```

**The key difference between these two scenarios, in one sentence:** *the volume's lifecycle is completely independent of the container's lifecycle.* Everything else about the two scenarios is identical — same image, same app, same mount path. The only variable is whether something durable exists *outside* the container to begin with.

This module proves Scenario 1. The rest of the series is entirely about building Scenario 2, correctly, one layer at a time.

---

## 🔧 Hands-on lab: prove it yourself

### Step 1: Run a pod that writes data with no volume at all

```bash
kubectl run writer --image=busybox -- sh -c "echo 'hello world' > /data/file.txt && sleep 3600"
```

### Step 2: Confirm the file exists

```bash
kubectl exec writer -- cat /data/file.txt
```

Expected: `hello world`

**Before running the next step — predict what you'll see.** If we delete this pod and create a brand new one with the identical command, will the file still be there?

### Step 3: Delete and recreate

```bash
kubectl delete pod writer
kubectl run writer --image=busybox -- sh -c "cat /data/file.txt 2>/dev/null || echo 'file not found'"
```

```bash
kubectl logs writer
```

Expected: `file not found`

**Right — the file is gone.** Not corrupted, not moved — it never had anywhere durable to live in the first place.

### What actually happened, mechanically

When `kubectl delete pod writer` ran, the container's **writable layer** — the only place `/data/file.txt` ever existed — was destroyed along with the container itself. When the second `kubectl run writer` created a brand new pod, it got a **completely fresh filesystem**, built fresh from the `busybox` image's own read-only layers. There was never any mechanism connecting "the old container's writable layer" to "the new container's writable layer" — they're two entirely separate things that happen to share the same pod name and the same image, and nothing more.

---

## Why this is correct behavior, not a bug

It's worth being explicit about this with students: **Kubernetes is not failing here.** This is exactly the ephemerality the entire platform is built around — it's what makes rolling updates safe, what makes crash recovery automatic, what makes horizontal scaling trivial (recall the stateless/stateful distinction from the autoscaling series — a stateless pod can be killed and replaced with zero consequence, precisely because nothing it holds is assumed to be durable).

**The database disaster from Module 00 happened because someone ran a stateful workload as if it were stateless** — no volume, or a misconfigured one, sitting underneath a database that absolutely needed one. The platform behaved exactly as designed. The design assumption just didn't match the workload's actual requirement.

---

## What's next

We need something that breaks the assumption "this data lives inside the container's own filesystem" — something that exists as a genuinely separate object, mountable into a container, but not destroyed when that container is. Kubernetes' answer to this starts with the simplest possible option: `emptyDir`.

---

*End of Module 01. Module 02: emptyDir — the first Kubernetes volume type, and exactly where its limits are.*
