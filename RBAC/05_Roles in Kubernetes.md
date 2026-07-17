# 05 — Roles in Kubernetes

**Series:** Kubernetes RBAC: Role-Based Access Control
**Format:** Concept + hands-on lab on your EKS cluster

---

## Recap from Sections 3-4

We now have both halves of the RBAC sentence — Resources (Section 3) and Verbs (Section 4). This section is about the object that actually combines them into an enforceable policy: the **Role**.

---

## The story: the job description

When a company hires people, they don't give each employee a verbal list of what they can do on their first day. They write a job description — responsibilities, boundaries, scope, all in writing. In Kubernetes, a **Role** is exactly that: a written, enforceable job description for cluster access.

**The Role doesn't know yet who will fill it.** That comes later, with RoleBinding (Section 6). First, we write the rules.

---

## What is a Kubernetes Role?

A Role defines a set of permissions **within one specific namespace**. It specifies which verbs are allowed on which resources.

```
Role = Permission Policy

A Role answers exactly two questions:
  1. Which resources can be accessed?
  2. Which actions can be performed on those resources?

A Role does NOT answer: Who has these permissions?
That is the job of the RoleBinding.
```

**A Role by itself grants nothing** — recall Section 2's proof of this directly: creating a Role has zero effect on `kubectl auth can-i` output until a RoleBinding connects it to an actual subject.

---

## Anatomy of a Role manifest

```yaml
# File: pod-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: test          # This Role applies ONLY to the test namespace
  name: pod-reader         # The name we will reference in RoleBindings
rules:
  - apiGroups: [""]        # Empty string means the core API group
    resources: ["pods"]    # Which resource type this rule applies to
    verbs: ["get", "list"] # Allowed actions: read a pod, list all pods
```

### Field by field

- **`apiVersion: rbac.authorization.k8s.io/v1`** — the API group for all RBAC objects
- **`kind: Role`** — declares this is namespace-scoped (contrast with `ClusterRole` in Section 8)
- **`metadata.namespace: test`** — this Role only has authority within the `test` namespace, full stop
- **`metadata.name: pod-reader`** — the identifier used when creating a RoleBinding
- **`rules`** — an array of permission rules, each with three parts:
  - **`apiGroups`** — the API group the resource belongs to. Core resources like `pods` use an empty string `""`. `deployments` use `apps`.
  - **`resources`** — the resource type(s) this rule targets
  - **`verbs`** — the allowed actions

---

## Hands-on lab, Part 1: apply and inspect a Role

```bash
kubectl create namespace test

cat <<'EOF' > pod-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: test
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
EOF

kubectl apply -f pod-reader.yaml
```

Confirm it was created:

```bash
kubectl get roles -n test
```

Inspect it in detail:

```bash
kubectl describe role pod-reader -n test
```

Expected output:
```
Name:         pod-reader
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  pods       []                 []              [get list]
```

**Confirm the Role genuinely grants nothing yet, on its own** — no subject has been bound to it:

```bash
kubectl auth can-i list pods -n test --as=system:serviceaccount:development:raj
```

Expected: `no`. Raj isn't bound to this Role at all yet — remember, Raj's *only* binding so far (from Section 4) is `pod-viewer` in the `development` namespace, which has no authority over the separate `test` namespace whatsoever.

---

## Hands-on lab, Part 2: build the full "developer" Role and prove the entire ALLOW/DENY matrix

Let's build a broader Role — one that permits a realistic developer's actual day-to-day actions — and prove every row of a decision table live, the same way your source material's table describes it.

```bash
kubectl create namespace devs

cat <<'EOF' > developer-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: devs
  name: developer-role
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "create", "delete"]
EOF

kubectl apply -f developer-role.yaml

cat <<'EOF' > raj-developer-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: raj-developer-binding
  namespace: devs
subjects:
- kind: ServiceAccount
  name: raj
  namespace: development
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f raj-developer-binding.yaml
```

> 📝 Note the subject's `namespace: development` (where Raj's ServiceAccount actually lives) versus the RoleBinding's own `namespace: devs` (where the Role and the granted permission apply). **A RoleBinding lives in the namespace whose resources it's granting access to — the subject itself can originate from anywhere.** This is a genuinely common point of confusion worth sitting on.

### Now prove every row of the ALLOW/DENY table, live

```bash
# Row 1: get pods in devs -- ALLOWED
kubectl auth can-i get pods -n devs --as=system:serviceaccount:development:raj

# Row 2: create pods in devs -- ALLOWED
kubectl auth can-i create pods -n devs --as=system:serviceaccount:development:raj

# Row 3: delete pods in devs -- ALLOWED
kubectl auth can-i delete pods -n devs --as=system:serviceaccount:development:raj

# Row 4: get SERVICES in devs -- DENIED (wrong resource type -- Role only covers pods)
kubectl auth can-i get services -n devs --as=system:serviceaccount:development:raj

# Row 5: get pods in test -- DENIED (wrong namespace -- this Role has zero authority outside devs)
kubectl auth can-i get pods -n test --as=system:serviceaccount:development:raj

# Row 6: get pods in a hypothetical production namespace -- DENIED (same reason as Row 5)
kubectl create namespace production
kubectl auth can-i get pods -n production --as=system:serviceaccount:development:raj
```

**Expected results, matching your source table exactly:** `yes, yes, yes, no, no, no`.

### Confirm Row 4 and Row 5 with real commands, not just `can-i`

```bash
kubectl get services -n devs --as=system:serviceaccount:development:raj
```
```
Error from server (Forbidden): services is forbidden: User "system:serviceaccount:development:raj" cannot list resource "services" in API group "" in the namespace "devs"
```

```bash
kubectl get pods -n test --as=system:serviceaccount:development:raj
```
```
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:development:raj" cannot list resource "pods" in API group "" in the namespace "test"
```

**Notice how precisely the error message identifies exactly which dimension failed** — Row 4's error names the resource (`services`), Row 5's error names the namespace (`test`) — this precision is genuinely useful for fast diagnosis, and it's why reading the *exact wording* of a Forbidden error, not just noting "it failed," is a real diagnostic skill.

---

## Cleanup

Keep everything from this section — Section 6 builds directly on the `developer-role` and its RoleBinding to go deeper on RoleBinding mechanics specifically. Nothing to clean up yet.

---

## Summary

| Concept | Key fact |
|---|---|
| Role | A namespace-scoped permission policy — answers "which resources, which verbs," nothing about "who" |
| A Role alone | Grants literally nothing — confirmed via `auth can-i` returning `no` even after creation |
| `rules[].apiGroups/resources/verbs` | The three-part structure of every individual permission rule |
| Namespace boundary | A Role's authority stops completely at its own namespace's edge — proven via Rows 5-6 above |
| Resource-type boundary | A Role's authority only covers the exact resource types listed — proven via Row 4 above |

**The one sentence to carry forward:** *"A Role is a precise, three-dimensional boundary — resource type, verb, and namespace, all three — and if a request falls outside even ONE of those three dimensions, it's denied, no matter how reasonable it might seem ('but Raj can already delete pods here, why can't he see services?'). RBAC has no concept of 'close enough.'"*

---

*End of Section 5. Section 6: RoleBinding — the object that finally connects a Subject to everything we just built.*
