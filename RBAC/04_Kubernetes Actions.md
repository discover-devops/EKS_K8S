# 04 — Kubernetes Actions (Verbs / Processes)

**Series:** Kubernetes RBAC: Role-Based Access Control
**Format:** Concept + hands-on lab on your EKS cluster

---

## Recap from Section 3

We covered Resources — the *what*. This section covers Verbs — the *what are you allowed to do to it*. Put them together, plus a Subject, and you have the complete RBAC sentence from Section 2: `Subject + Verb + Resource`.

---

## The story: librarian rules

Think about a public library. A student can browse shelves and read books, but they cannot rewrite books, permanently remove books from the shelves, or access the restricted archive. A librarian can do all of those things. A janitor can enter the building and clean, but cannot check out books on behalf of others.

**Each role in the library has a different set of permitted actions.** Kubernetes verbs work exactly the same way — they define what you're allowed to *do* with a resource, completely independent of whether you're even allowed to see it exists in the first place.

---

## The standard Kubernetes verbs

| Verb | `kubectl` equivalent | What it does |
|---|---|---|
| `get` | `kubectl get pod mypod` | Retrieve a single specific resource by name |
| `list` | `kubectl get pods` | List all resources of a type in a namespace |
| `watch` | `kubectl get pods -w` | Stream real-time updates to a resource list |
| `create` | `kubectl apply -f pod.yaml` | Create a new resource from a manifest |
| `update` | `kubectl set image ...` | Modify an existing resource in place |
| `patch` | `kubectl patch ...` | Apply a partial update to a resource |
| `delete` | `kubectl delete pod mypod` | Remove a resource permanently |
| `exec` | `kubectl exec -it mypod ...` | Execute a command inside a running container |
| `logs` | `kubectl logs mypod` | Stream the logs of a container |

**Worth noticing immediately:** `get` and `list` are genuinely separate verbs, not the same permission phrased two ways. Someone with `list` but not `get` can see that pods exist and their names, but can't retrieve full detail on any specific one. Someone with `get` but not `list` can look up a pod *by exact name* if they already know it, but can't browse to discover what's running. This distinction trips people up constantly — it's worth stating explicitly here before it causes confusion later.

---

## 🔧 Hands-on lab: verb restrictions, proven live

We'll set up a minimal permission grant now (Sections 5-6 cover the full Role/RoleBinding mechanics in depth — for this section, treat the YAML below purely as necessary setup to demonstrate verb behavior).

### Step 1: Grant Raj `list` and `get` on pods only, in `development`

```bash
cat <<'EOF' > raj-pod-viewer-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-viewer
  namespace: development
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
EOF

kubectl apply -f raj-pod-viewer-role.yaml

cat <<'EOF' > raj-pod-viewer-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: raj-pod-viewer-binding
  namespace: development
subjects:
- kind: ServiceAccount
  name: raj
  namespace: development
roleRef:
  kind: Role
  name: pod-viewer
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl create serviceaccount raj -n development --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f raj-pod-viewer-binding.yaml
```

Put a real pod there to test against:

```bash
kubectl run mypod --image=nginx -n development
```

### Step 2: Confirm the actions that SHOULD succeed

```bash
kubectl auth can-i list pods -n development --as=system:serviceaccount:development:raj
kubectl auth can-i get pods -n development --as=system:serviceaccount:development:raj
```

Expected: `yes`, `yes`.

### Step 3: Confirm the actions that SHOULD fail

```bash
kubectl auth can-i delete pods -n development --as=system:serviceaccount:development:raj
kubectl auth can-i create pods -n development --as=system:serviceaccount:development:raj
kubectl auth can-i "exec" pods -n development --as=system:serviceaccount:development:raj
```

Expected: `no`, `no`, `no` — Raj can see and inspect pods, and nothing else. `list`/`get` were explicitly granted; everything else falls back to Section 2's deny-by-default.

### Step 4: See the actual Forbidden error, not just `can-i`'s yes/no

`auth can-i` is a dry-run check — let's see what a real denied request actually looks like, using impersonation directly against a real command:

```bash
kubectl delete pod mypod -n development --as=system:serviceaccount:development:raj
```

Expected output, something like:
```
Error from server (Forbidden): pods "mypod" is forbidden: User "system:serviceaccount:development:raj" cannot delete resource "pods" in API group "" in the namespace "development"
```

**This is the exact message you'll see in real incident logs and CI/CD failures** — worth recognizing on sight, since `auth can-i` is a convenience check, but this Forbidden message is what actually shows up when something breaks in production.

---

## Understanding `--as` — how impersonation actually works

The `--as` flag uses a genuine Kubernetes feature called **impersonation**. When you run:

```bash
kubectl auth can-i list pods --as=dicktracy@kubernetes
```

**your own current credentials** are asking the API server to evaluate the request *as if* it came from `dicktracy` instead. This is not a trick or a client-side simulation — it's a real server-side feature.

**This only works if your current identity has been granted the `impersonate` verb** — which `cluster-admin` (what your own `kubectl` session is almost certainly running as right now, since you created this EKS cluster) has by default.

**If you're ever operating with a more restricted admin account and `--as` suddenly returns `Forbidden`, this is why — it's not a bug.** It means the account you're currently authenticated as lacks impersonation rights, which is itself a legitimate, deliberate RBAC restriction (you generally don't want every admin able to impersonate *any* other identity in the cluster at will — that's its own significant privilege, worth granting narrowly).

### 🔧 Confirm your own session actually has impersonate rights

```bash
kubectl auth can-i impersonate serviceaccounts
```

Expected: `yes` (since you're operating as cluster-admin via your EKS access). This is *why* every `--as=` command in this entire series has been working — it's not automatic for every identity, it's a privilege your current session happens to hold.

---

## Practical: `kubectl auth can-i` as your primary verification tool

```bash
# Check what the CURRENT identity (you) can do
kubectl auth can-i list pods -n development

# Check what a specific OTHER identity can do
kubectl auth can-i list pods -n development --as=system:serviceaccount:development:raj
kubectl auth can-i delete pods -n development --as=system:serviceaccount:development:raj
```

Output is always exactly `yes` or `no` — this makes it trivially scriptable for CI/CD pipelines that want to verify RBAC policy *before* deploying it, not discover a Forbidden error after something's already broken in production.

---

## Cleanup

Keep `raj-pod-viewer-role.yaml` and its binding — we'll build directly on this exact setup in Sections 5 and 6. Just remove the test pod:

```bash
kubectl delete pod mypod -n development
```

---

## Summary

| Concept | Key fact |
|---|---|
| Verbs | The *action* half of the RBAC sentence — `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`, `exec`, `logs` |
| `get` vs `list` | Genuinely separate permissions — knowing a pod's name vs. being able to browse/discover pods are different grants |
| `kubectl auth can-i` | The fast, scriptable, dry-run way to check permissions before they matter |
| `--as=` | Real server-side impersonation, not a simulation — requires the `impersonate` verb on your own current identity |
| Real Forbidden errors | What actually shows up in production when a verb isn't granted — worth recognizing on sight |

**The one sentence to carry forward:** *"A Resource without a Verb is meaningless — 'Raj can access pods' isn't a real RBAC statement until you specify exactly which actions, because 'access' could mean anything from harmlessly browsing to permanently deleting production workloads."*

---

*End of Section 4. Section 5: Roles in Kubernetes — the full, deliberate construction of the object that ties Resources and Verbs together.*
