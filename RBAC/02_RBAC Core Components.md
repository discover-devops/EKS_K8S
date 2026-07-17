# 02 — RBAC Core Components

**Series:** Kubernetes RBAC: Role-Based Access Control
**Format:** Concept + a short proof-of-concept lab (the full Role/RoleBinding build comes in Sections 5-6)

---

## Recap from Section 1

We established *why* access control matters. This section introduces the actual vocabulary and objects — the four building blocks every single RBAC decision in Kubernetes is made from.

---

## The story: a new hire joins the team

Raj has just joined as a backend developer. On his first day, the platform team needs to give him access to the cluster — but they don't want him deleting services, editing secrets, or touching the production namespace.

They need a system that says: *Raj can view pods and create pods in the `development` namespace, nothing more, nothing less.* That system is Role-Based Access Control.

---

## The four building blocks

Think of them as the grammar of a sentence: **Subject, Verb, Object**, and the rule that connects them.

### Subject — who is acting?

The entity requesting access. Three types exist in Kubernetes:

| Type | Example | Notes |
|---|---|---|
| **User** | Raj, `dicktracy` | A human being with credentials. Kubernetes itself has no built-in concept of user accounts or a user database — users are authenticated *externally* (certificates, an OIDC provider, or on EKS specifically, IAM) and RBAC only ever deals with the *identity string* that authentication produces. |
| **Group** | `devs`, `tech-leads` | A collection of users, useful for granting the same access to many people at once without managing each individually |
| **ServiceAccount** | `cicd-bot` | A non-human identity for an application or automated process running *inside* the cluster. Unlike Users, ServiceAccounts genuinely are native Kubernetes objects — you can `kubectl get serviceaccounts` and see them directly. |

> 📝 **Why this matters for our labs:** since Kubernetes has no built-in "create a user" command, and setting up a real external identity provider is out of scope for a hands-on lab, **we'll simulate "Raj" using a ServiceAccount** throughout this series — the RBAC mechanics (Role, RoleBinding, verb evaluation) are identical regardless of which subject type is involved. This is a completely standard, realistic testing pattern — many real teams test RBAC policy changes exactly this way before rolling them out to real human users.

### Role — what is permitted?

A named collection of permissions. It answers: *which actions are allowed on which resources?* A Role named `pod-viewer` might permit `list` and `get` on `pods`. (Full deep-dive with hands-on YAML in Section 5.)

### RoleBinding — who gets the Role?

Connects a Subject to a Role. **Without a RoleBinding, a Role has no effect whatsoever** — it just sits there, inert. Think of the Role as a job description, and the RoleBinding as the actual employment contract that assigns that job description to one specific person. (Full deep-dive in Section 6.)

### Resource — what is being accessed?

The Kubernetes objects subjects want to interact with: pods, deployments, services, configmaps, secrets, nodes, and many more. (Full catalog in Section 3.)

---

## The RBAC sentence structure

Every access decision in Kubernetes reads like an English sentence:

```
Subject   +   Verb    +   Resource

Raj           list        pods
devs          create      deployments
cicd-bot      delete      pods
```

## The RBAC evaluation flow

When Raj runs a `kubectl` command, Kubernetes evaluates it in this order:

```
Raj (User)
   |
   v
RoleBinding  -->  maps Raj to a Role
   |
   v
Role  -->  defines allowed verbs on resources
   |
   v
Resource + Verb  -->  ALLOW or DENY
```

**If there is no RoleBinding connecting Raj to a Role that permits the requested action, Kubernetes denies the request.** This is the single most important sentence in this entire section, worth repeating explicitly:

> **Kubernetes follows a deny-by-default model. Nothing is permitted unless explicitly granted.**

---

## 🔧 Prove deny-by-default, right now, before building anything else

This is worth seeing with your own eyes before we build a single Role or RoleBinding — because everything in Sections 5-8 is really just "how do we carve out specific exceptions to this default."

### Step 1: Create a namespace and a brand-new ServiceAccount with zero permissions granted

```bash
kubectl create namespace development
kubectl create serviceaccount raj -n development
```

### Step 2: Try a series of actions as this identity, using `kubectl auth can-i`

```bash
kubectl auth can-i list pods -n development --as=system:serviceaccount:development:raj
kubectl auth can-i get pods -n development --as=system:serviceaccount:development:raj
kubectl auth can-i create deployments -n development --as=system:serviceaccount:development:raj
kubectl auth can-i delete secrets -n development --as=system:serviceaccount:development:raj
```

**Expected: `no`, for every single one of these.** We haven't created a single Role or RoleBinding yet — and that absence is precisely why every action is denied. This ServiceAccount exists, it's a completely valid identity, and it *still* can't do anything at all, because nothing has explicitly granted it permission.

### Step 3: Confirm this is universal — try an absurd, clearly-should-never-be-allowed action too

```bash
kubectl auth can-i delete nodes --as=system:serviceaccount:development:raj
```

**Expected: also `no`.** Deny-by-default doesn't grade actions by how dangerous they are — it denies *everything* uniformly, until something explicitly says otherwise. There's no built-in notion of "obviously this ServiceAccount shouldn't delete nodes, but reading a pod is probably fine" — the default is total silence, and that silence means no.

---

## The Principle of Least Privilege

Everything you'll build for the rest of this series follows one foundational security principle: **grant only what is needed, nothing more, nothing less.**

- A developer who only needs to read logs should never have delete permissions.
- A CI/CD pipeline that only deploys to staging should never have access to production.

**You will hear this exact term — "principle of least privilege" — in security reviews, compliance audits, and job interviews.** Every RBAC decision you make, for the rest of your career, should start with the same question: *"What is the minimum access this subject actually needs to do their job?"* Not "what's convenient to grant," not "what might they need someday" — the minimum, right now, for the actual task.

---

## Cleanup

```bash
kubectl delete serviceaccount raj -n development
```

> 📝 Keep the `development` namespace — we'll reuse it starting in Section 5, when Raj gets his first real Role.

---

## Summary

| Building block | Question it answers | Section with full hands-on detail |
|---|---|---|
| Subject | Who is acting? | This section |
| Resource | What's being accessed? | Section 3 |
| Verb | What action is being attempted? | Section 4 |
| Role | What's permitted? | Section 5 |
| RoleBinding | Who actually gets the Role? | Section 6 |

**The one sentence to carry forward:** *"You just watched a completely valid, real identity get denied every single action, because Kubernetes assumes 'no' until something explicitly says 'yes.' Every section from here forward is about carefully, deliberately saying 'yes' to exactly the right things — and nothing else."*

---

*End of Section 2. Section 3: Kubernetes Resources — the full catalog of what subjects can be granted access to.*
