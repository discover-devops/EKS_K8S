# 03 — Kubernetes Resources (Items)

**Series:** Kubernetes RBAC: Role-Based Access Control
**Format:** Concept + hands-on lab on your EKS cluster

---

## Recap from Section 2

We introduced the four building blocks — Subject, Role, RoleBinding, Resource — and proved deny-by-default with Raj's ServiceAccount. This section goes deep on just one of those four: **Resource** — the actual catalog of *what* a Role can grant access to.

---

## The story: what is inside the cluster?

Imagine your Kubernetes cluster is a large warehouse. Inside are different types of storage units: shelves for packages, refrigerators for perishables, locked vaults for valuables. Each storage type holds a different kind of item.

In Kubernetes, these storage types are called **resources**, and the items inside them are the **objects** you create and manage. When you write a Role, you're deciding **which storage units a person is allowed to open, and what they can do with what's inside.**

---

## Common Kubernetes resources

| Resource | Description |
|---|---|
| `pods` | The smallest deployable unit — a container or group of containers running together |
| `deployments` | Manages how many replicas of a pod run, handles rolling updates |
| `services` | Exposes pods to network traffic, internally or externally |
| `configmaps` | Non-sensitive configuration data as key-value pairs |
| `secrets` | Sensitive data — passwords, tokens, certificates |
| `nodes` | The physical or virtual machines forming the cluster infrastructure |
| `namespaces` | Virtual clusters within a cluster, used to isolate environments |
| `persistentvolumeclaims` | Requests for storage space by applications (recall the entire Storage series) |

---

## 🔧 Practical: viewing every resource type your cluster actually knows about

```bash
kubectl api-resources
```

This lists **every resource type available in the cluster** — not just the common ones above, but everything, including custom resources any installed operator or CRD has added (recall Karpenter's `NodePool` and `EC2NodeClass` from the autoscaling series — those show up here too, since they're genuine API resources, just not built-in ones).

Filtered to just a few resources we care about right now:

```bash
kubectl api-resources | grep -E "^pods |^services |^deployments |^nodes |^namespaces |^secrets "
```

---

## Key point: Namespaced vs. cluster-level resources

Look at the `NAMESPACED` column in your `kubectl api-resources` output:

```
NAMESPACED = true   -->  pods, services, deployments, secrets, configmaps
                          These live INSIDE a namespace

NAMESPACED = false  -->  nodes, namespaces, persistentvolumes
                          These are CLUSTER-WIDE
                          You need a ClusterRole for cluster-level resources
                          (covered in Section 8)
```

**This distinction is the single most important thing to internalize in this section**, because it directly determines whether a `Role` (namespace-scoped) is even *capable* of granting access to something, or whether you're forced to reach for a `ClusterRole` instead — regardless of how narrowly you try to scope the permission.

### 🔧 Prove this distinction hands-on

```bash
kubectl api-resources --namespaced=true | head -10
kubectl api-resources --namespaced=false | head -10
```

Notice: `pods`, `deployments`, `services`, `secrets` all appear in the first list. `nodes`, `namespaces`, `persistentvolumes` (note: **not** `persistentvolumeclaims` — PVCs are namespaced, but the underlying PVs they bind to are not, exactly matching what you learned in the Storage series about admin-vs-developer scope) appear in the second.

**Why this split exists, conceptually:** a `pod` inherently belongs to one specific team's workload, in one specific namespace — it makes sense to scope permission to it narrowly. A `node` is shared physical infrastructure underneath *every* namespace simultaneously — there's no meaningful way to say "Raj can only see this node when acting within the `development` namespace," because the node itself doesn't belong to any one namespace. This is precisely why cluster-scoped resources need a cluster-scoped permission object.

---

## 🔧 Practical: listing resources in a namespace

Let's put something real into our `development` namespace from Section 2, so there's something to actually list.

```bash
kubectl run pinger-dk --image=busybox -n development -- sleep 3600
kubectl create deployment sample-app --image=nginx -n development
```

```bash
kubectl get pods -n development
```

```bash
kubectl get all -n development
```

**Notice `get all` doesn't actually mean "everything"** — it's a curated shorthand covering the most common resource types (pods, deployments, services, replicasets, and a few others), not the full `api-resources` catalog. ConfigMaps, Secrets, and PVCs, for instance, don't show up under `get all` even though they're namespaced resources living in that same namespace — worth knowing so you don't assume `get all` gives a complete picture during a real investigation.

```bash
kubectl describe pod pinger-dk -n development
```

`describe` (recall this from every prior series in this course) is where the real diagnostic detail lives — events, resource requests, node placement, all of it — versus `get`, which just shows a status summary.

---

## Cleanup

```bash
kubectl delete pod pinger-dk -n development
kubectl delete deployment sample-app -n development
```

---

## Summary

| Concept | Key fact |
|---|---|
| Resources | The "storage unit types" a Role can grant access to — pods, deployments, secrets, nodes, etc. |
| `kubectl api-resources` | The authoritative, complete list of every resource type your cluster actually has, including custom ones |
| `NAMESPACED: true` | Lives inside one namespace — a `Role` can grant access to it |
| `NAMESPACED: false` | Cluster-wide, no owning namespace — **requires a `ClusterRole`**, no exceptions |
| `kubectl get all` | A convenience shorthand, NOT a complete resource listing — Secrets/ConfigMaps/PVCs are excluded |

**The one sentence to carry forward:** *"Before you ever write a Role, you need to know two things about the resource you're granting access to: what it's called in the API, and whether `kubectl api-resources` says it's namespaced — because that single `true`/`false` value decides whether a Role can even do the job, or whether Section 8's ClusterRole is mandatory instead."*

---

*End of Section 3. Section 4: Kubernetes Actions (Verbs) — the other half of every RBAC sentence.*
