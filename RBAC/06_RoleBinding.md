# 06 — RoleBinding

**Series:** Kubernetes RBAC: Role-Based Access Control
**Format:** Concept + hands-on lab on your EKS cluster

---

## Recap from Section 5

We built Roles — job descriptions with no one assigned to them yet. This section is about the object that actually assigns someone: **RoleBinding**.

---

## The story: the employment contract

A company has written a detailed job description: Software Developer. It lists every task a developer is permitted to do. But until an actual person **signs the employment contract referencing that job description, no one has those permissions.**

In Kubernetes, the **Role is the job description. The RoleBinding is the employment contract** that links a person to that role.

---

## What is a RoleBinding?

A RoleBinding grants the permissions defined in a Role to one or more subjects, within a specific namespace. It references a Role by name, and lists the subjects who receive those permissions.

## Anatomy of a RoleBinding manifest

```yaml
# File: user-pod-reader-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods            # Name of this RoleBinding
  namespace: test            # The namespace this binding applies to
subjects:                    # WHO receives the permissions
  - kind: User
    name: dicktracy@kubernetes
    apiGroup: rbac.authorization.k8s.io
roleRef:                     # WHICH Role is being assigned
  kind: Role
  name: pod-reader           # Must match an existing Role name
  apiGroup: rbac.authorization.k8s.io
```

- **`subjects`** — an array of subjects who will receive the Role. Each has a `kind` (`User`, `Group`, or `ServiceAccount`) and a `name`.
- **`roleRef`** — a reference to the Role being assigned. **Once created, `roleRef` cannot be changed.** You must delete and recreate the binding to point it at a different Role.

---

## Hands-on lab, Part 1: user-level RoleBinding

We'll reuse the `pod-reader` Role from Section 5 (`test` namespace). Since creating a real certificate-based `User` is a heavier setup (its own appendix, coming later in this series), **we'll use `--as` impersonation to act as `dicktracy@kubernetes` without needing a real cert** — impersonation makes the identity real enough for RBAC evaluation purposes, exactly as Section 4 explained.

```bash
cat <<'EOF' > user-pod-reader-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: test
subjects:
  - kind: User
    name: dicktracy@kubernetes
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f user-pod-reader-rolebinding.yaml
kubectl get rolebindings -n test
```

Verify the grant:

```bash
kubectl auth can-i list pods -n test --as=dicktracy@kubernetes
```
Expected: `yes`.

```bash
kubectl auth can-i create pods -n test --as=dicktracy@kubernetes
```
Expected: `no` — `pod-reader`'s rules only ever granted `get` and `list`, nothing more, and a RoleBinding cannot expand what a Role already defines; it can only *assign* it.

---

## Why does `apiGroup` appear on both `subjects` AND `roleRef`?

RBAC itself is a specific API group within Kubernetes: `rbac.authorization.k8s.io`. Writing `apiGroup: rbac.authorization.k8s.io` on a **subject** tells Kubernetes which API group is responsible for interpreting that subject's identity type (`User`/`Group` are core RBAC concepts, defined by this API group). Writing it on **`roleRef`** tells Kubernetes which API group owns the Role object being referenced. **Both fields point at the same RBAC system — just answering two different questions ("who is this subject, according to which system" vs. "what is this Role, according to which system") that happen to have the same answer in a pure-RBAC setup.**

---

## Binding to a Group instead of an individual

One of RoleBinding's most powerful features: it can target a **Group**, not just one individual User. Every user in that group automatically receives the Role's permissions — this is how permissions get managed at real organizational scale, rather than one binding per person.

```yaml
# File: simple-dev-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devs-binding
  namespace: devs
subjects:
  - kind: Group               # Binds to an entire group
    name: devs                # Group name embedded in the user's certificate
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: simple-dev-role
  apiGroup: rbac.authorization.k8s.io
```

### How groups actually get established — a preview of the future appendix

Recall the diagram from earlier in this series — creating a real user involves generating an RSA key and an OpenSSL certificate signing request. **Group membership is embedded directly into that certificate**, via the `-subj` field:

```bash
openssl req -new -key dicktracy.key -out dicktracy.csr \
  -subj "/CN=dicktracy/O=devs/O=tech-leads"
```

```
/CN=dicktracy    -->  Sets the username
/O=devs          -->  Adds dicktracy to the devs group
/O=tech-leads    -->  Adds dicktracy to the tech-leads group
```

**A RoleBinding targeting the group `devs` automatically applies to `dicktracy`**, because the certificate itself declares that group membership — Kubernetes never stores a separate "list of who's in which group" anywhere; group membership is a property of the *credential itself*, established once at certificate-creation time. We'll build this full flow end-to-end in the dedicated user-creation appendix later in this series.

---

## Hands-on lab, Part 2: group-level RoleBinding, proven via `--as-group`

Kubernetes' impersonation feature has a companion flag specifically for this — `--as-group` — letting us simulate "this identity carries this group membership" without needing a real certificate yet.

```bash
cat <<'EOF' > simple-dev-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: simple-dev-role
  namespace: devs
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
EOF

cat <<'EOF' > simple-dev-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devs-binding
  namespace: devs
subjects:
  - kind: Group
    name: devs
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: simple-dev-role
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f simple-dev-role.yaml
kubectl apply -f simple-dev-rolebinding.yaml
```

### Prove group membership alone is sufficient — try TWO different usernames, both in the `devs` group

```bash
kubectl auth can-i list pods -n devs --as=dicktracy@kubernetes --as-group=devs
kubectl auth can-i list pods -n devs --as=someone-else@kubernetes --as-group=devs
```

**Expected: `yes` for both**, even though `someone-else@kubernetes` was never mentioned anywhere in any Role or RoleBinding. **This is the entire point of group binding** — the RoleBinding never names an individual person; it names the group, and Kubernetes trusts whatever the credential itself asserts about group membership.

### Prove the group name itself is what matters — not just "having some group"

```bash
kubectl auth can-i list pods -n devs --as=dicktracy@kubernetes --as-group=marketing
```

Expected: `no` — `dicktracy` is impersonated as a member of `marketing` this time, not `devs`, and no Role or RoleBinding in the `devs` namespace has anything to do with a `marketing` group.

```bash
kubectl auth can-i list pods -n devs --as=dicktracy@kubernetes
```

Expected: `no` — impersonating the user with **no group at all** also fails, confirming the grant genuinely flows through group membership, not just through recognizing the username `dicktracy` from earlier in this section.

---

## Multiple subjects in one binding

A single RoleBinding can grant the same Role to several subjects simultaneously — mixing individual Users, ServiceAccounts, and Groups freely in the same `subjects` array:

```yaml
subjects:
  - kind: User
    name: raj@kubernetes
    apiGroup: rbac.authorization.k8s.io
  - kind: User
    name: priya@kubernetes
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: devs
    apiGroup: rbac.authorization.k8s.io
```

---

## Prove `roleRef` is genuinely immutable

```bash
kubectl patch rolebinding devs-binding -n devs --type='json' \
  -p='[{"op": "replace", "path": "/roleRef/name", "value": "pod-reader"}]'
```

Expected: an error rejecting the patch — `roleRef` is immutable once the RoleBinding is created, exactly as your source material states. The only way to actually change which Role a binding points to is:

```bash
kubectl delete rolebinding devs-binding -n devs
# then recreate it with the new roleRef
```

---

## Cleanup

Keep everything — Section 7 combines all of this into one complete end-to-end flow example.

---

## Summary

| Concept | Key fact |
|---|---|
| RoleBinding | Connects a Subject (User/Group/ServiceAccount) to a Role, within one namespace |
| `roleRef` | Immutable once set — delete and recreate to change it |
| Binding to a `User` | Grants the named individual, and only that individual |
| Binding to a `Group` | Grants everyone whose credential asserts that group membership — proven live via `--as-group` |
| `apiGroup` on both `subjects` and `roleRef` | Both point at the same `rbac.authorization.k8s.io` system, answering "who/what is this" from two different angles |

**The one sentence to carry forward:** *"A Role with no RoleBinding is a job description nobody signed. A RoleBinding to a Group means anyone whose credential carries that group membership is automatically covered — which is powerful at scale, but also means the actual security boundary lives in how carefully group membership is assigned at certificate-creation time, not in the RoleBinding YAML itself."*

---

*End of Section 6. Section 7: Complete RBAC Flow Example — Role and RoleBinding, from an empty namespace to a fully working permission grant, start to finish.*
