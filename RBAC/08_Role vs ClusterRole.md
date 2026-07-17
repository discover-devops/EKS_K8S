# 08 — Role vs ClusterRole

**Series:** Kubernetes RBAC: Role-Based Access Control
**Format:** Concept + hands-on lab on your EKS cluster — final section of this series

---

## Recap from Section 7

We built a complete Role/RoleBinding flow, end to end, on real EKS infrastructure. Every namespace-scoped resource type from Section 3 is now something we know how to control access to. But recall Section 3's own warning: **some resources — nodes, namespaces, PersistentVolumes — are cluster-wide, with no owning namespace.** A `Role` is structurally incapable of granting access to them, no matter how the YAML is written. This section closes that gap.

---

## The story: floor manager vs. building manager

A floor manager at an office can manage everything on their assigned floor — who sits where, which meeting rooms are booked, which supply cabinets are accessible. But they cannot make decisions about the building's fire safety systems, the elevator controls, or building entry passes. Those require someone with building-wide authority.

**In Kubernetes, a Role is the floor manager. A ClusterRole is the building manager.**

---

## The fundamental difference

| Property | Role | ClusterRole |
|---|---|---|
| Scope | Single namespace only | All namespaces + cluster-wide |
| Can manage nodes? | No | Yes |
| Can manage namespaces? | No | Yes |
| Can manage pods? | Yes, but only in its namespace | Yes, across all namespaces |
| Bound using | `RoleBinding` | `ClusterRoleBinding` (or `RoleBinding`) |
| Use case | Developer access to one namespace | Admin, monitoring, CI/CD pipelines |

**One entry in this table is easy to skim past and shouldn't be:** *"Bound using: `ClusterRoleBinding` (or `RoleBinding`)."* A ClusterRole isn't automatically cluster-wide in its *effect* — only in what it's *capable of* granting. How you bind it determines the actual blast radius. We'll prove this distinction directly in the lab below, since it's one of the most commonly misunderstood parts of RBAC in real production clusters.

---

## ClusterRole manifest — nearly identical to a Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole            # Note: ClusterRole, not Role
metadata:
  name: node-reader          # No namespace field - applies cluster-wide
rules:
  - apiGroups: [""]
    resources: ["nodes"]     # Nodes are cluster-level resources
    verbs: ["get", "list", "watch"]
```

The only structural differences from a Role: `kind: ClusterRole`, and no `metadata.namespace` field at all — there's no namespace to put it in, because the object itself isn't scoped to one.

## ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes-global
subjects:
  - kind: User
    name: monitoring-agent@kubernetes
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## Hands-on lab, Part 1: apply and verify a cluster-wide grant

```bash
cat <<'EOF' > node-reader-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
EOF

cat <<'EOF' > read-nodes-global.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes-global
subjects:
  - kind: User
    name: monitoring-agent@kubernetes
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f node-reader-clusterrole.yaml
kubectl apply -f read-nodes-global.yaml
```

```bash
kubectl auth can-i list nodes --as=monitoring-agent@kubernetes
```

Expected: `yes` — and critically, **no `-n` namespace flag was even relevant here**, because `nodes` isn't a namespaced resource in the first place (recall Section 3). Confirm a `Role` genuinely could never have granted this, even if you tried:

```bash
kubectl auth can-i list pods -n test --as=monitoring-agent@kubernetes
```

Expected: `no` — `node-reader` only ever granted `nodes`, nothing about `pods`.

---

## Hands-on lab, Part 2: the crucial nuance — binding a ClusterRole via a plain RoleBinding

This is the "or `RoleBinding`" from the table above, and it's genuinely useful in real clusters — **you can take a ClusterRole's *rules* (which must be about namespaced resources, for this pattern to make sense) and apply them to just ONE namespace**, by binding it with a regular `RoleBinding` instead of a `ClusterRoleBinding`. This is exactly how Kubernetes' own built-in `view` and `edit` ClusterRoles (explored below) are meant to be used in practice — defined once, cluster-wide, but bound per-namespace as needed.

```bash
cat <<'EOF' > pod-manager-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-manager
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "create", "delete"]
EOF

kubectl apply -f pod-manager-clusterrole.yaml
```

Now bind it with a **namespace-scoped `RoleBinding`**, restricted to just `test`:

```bash
cat <<'EOF' > pod-manager-scoped-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-manager-in-test-only
  namespace: test
subjects:
  - kind: User
    name: priya@kubernetes
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: pod-manager
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f pod-manager-scoped-binding.yaml
```

### Prove the effect is genuinely namespace-limited, despite the underlying object being a ClusterRole

```bash
kubectl auth can-i create pods -n test --as=priya@kubernetes
# Expected: yes

kubectl auth can-i create pods -n devs --as=priya@kubernetes
# Expected: no
```

**This is the entire lesson of this section, proven directly:** `pod-manager` is a `ClusterRole` — an object fully *capable* of being applied cluster-wide — but because we bound it with a `RoleBinding` scoped to `test`, its actual effect on `priya` stops completely at that namespace's edge. **The object type (`ClusterRole` vs `Role`) determines capability. The binding type (`ClusterRoleBinding` vs `RoleBinding`) determines actual reach.** These are two independent decisions, not one.

---

## Hands-on lab, Part 3: exploring Kubernetes' built-in ClusterRoles

Kubernetes ships with several pre-built ClusterRoles — worth knowing them by name, since they appear constantly in real production RBAC setups instead of custom-written Roles.

```bash
kubectl get clusterroles
```

```bash
kubectl describe clusterrole view
```

`view` — read-only across almost everything, deliberately excluding Secrets' actual contents (you can see a Secret *exists*, not its decoded data).

```bash
kubectl describe clusterrole edit
```

`edit` — read/write on most resources, but **not** RBAC objects themselves (a subject with `edit` cannot grant itself more permissions — an important, deliberate self-escalation prevention).

```bash
kubectl describe clusterrole cluster-admin
```

Look closely at the `PolicyRule` output — you should see:
```
Resources  ...  Verbs
*          ...  [*]
```

**`*` on both resources and verbs means literally everything, on every resource type, with every action, cluster-wide.** This is the maximum possible privilege in Kubernetes RBAC.

### Prove exactly how dangerous `cluster-admin` is, directly

```bash
cat <<'EOF' > danger-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: danger-test-binding
subjects:
  - kind: User
    name: overprivileged-test-user@kubernetes
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f danger-binding.yaml
```

```bash
kubectl auth can-i delete nodes --as=overprivileged-test-user@kubernetes
kubectl auth can-i delete namespaces --as=overprivileged-test-user@kubernetes
kubectl auth can-i "*" "*" --as=overprivileged-test-user@kubernetes
```

**Expected: `yes` to all three** — including deleting the very infrastructure your entire cluster runs on. This is precisely the disaster class Section 1's opening story warned about, except now amplified to cluster-wide, infrastructure-destroying scope, granted by a single `ClusterRoleBinding` to `cluster-admin`. **This is exactly why the Principle of Least Privilege (Section 2) exists — `cluster-admin` should be reserved for a tiny handful of genuinely trusted break-glass identities, never handed out as a default convenience.**

---

## Decision guide: when to use Role vs ClusterRole

**Use `Role` when:**
- The user needs access to resources in one specific namespace only
- You want namespace-level isolation between teams
- You're giving developer or tester access

**Use `ClusterRole` when:**
- The user needs to manage nodes, namespaces, or PersistentVolumes (recall: `Role` is structurally incapable of this, per Section 3)
- A monitoring tool needs read access across all namespaces
- A CI/CD system needs to deploy to any namespace
- You're granting cluster administrator rights (with extreme caution)

---

## Cleanup

```bash
kubectl delete -f danger-binding.yaml
kubectl delete -f pod-manager-scoped-binding.yaml
kubectl delete -f pod-manager-clusterrole.yaml
kubectl delete -f read-nodes-global.yaml
kubectl delete -f node-reader-clusterrole.yaml
```

If you completed Section 7's optional Step 5:
```bash
aws eks delete-access-entry --cluster-name $CLUSTER_NAME --region $AWS_REGION --principal-arn arn:aws:iam::${AWS_ACCOUNT_ID}:user/dicktracy 2>/dev/null; true
aws iam delete-user --user-name dicktracy 2>/dev/null; true
```

---

## Series summary — every section, in one table

| Section | What it established |
|---|---|
| 01 | Why access control matters — deny-by-default prevents unbounded blast radius |
| 02 | The four building blocks — Subject, Resource, Verb, and how they combine |
| 03 | Resources — the full catalog, and the namespaced vs. cluster-scoped split |
| 04 | Verbs — the actions half of every RBAC sentence, plus `auth can-i` and impersonation |
| 05 | Role — the namespace-scoped permission policy itself |
| 06 | RoleBinding — connecting a Subject to a Role, individually or via Group |
| 07 | The complete flow, end to end, on real EKS infrastructure via IAM Access Entries |
| 08 | ClusterRole — extending everything to cluster-wide resources, and the object-vs-binding scope distinction |

**The one sentence to close the entire series on:** *"Every single incident this series opened with — the Friday-evening production outage, the compromised CI/CD token, the missing audit trail — comes down to the same root cause: someone had more access than their actual job required, because nobody had deliberately drawn the boundary. RBAC is never about distrust of any individual. It's about making sure a single mistake, by anyone, has a bounded, survivable blast radius — exactly the same principle as NetworkPolicy's blast-radius containment, just enforced at the API layer instead of the network layer."*
