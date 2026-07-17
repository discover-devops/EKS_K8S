# 07 — Complete RBAC Flow Example

**Series:** Kubernetes RBAC: Role-Based Access Control
**Format:** Full end-to-end hands-on lab, EKS-adapted

---

## Critical EKS difference from the standard tutorial version of this lab

The classic version of this exercise (signing a client certificate with the cluster's own CA key) **requires filesystem access to `/etc/kubernetes/pki/ca.key`** — the cluster's Certificate Authority private key. This works on **self-managed clusters** (`kubeadm`), where you control the control plane machine directly.

**On EKS, this is architecturally impossible.** EKS is a managed control plane — AWS runs the API server and holds the CA private key entirely within AWS's own infrastructure. There is no `/etc/kubernetes/pki/` directory you can reach, by design, regardless of your AWS permissions.

**The real EKS-native equivalent: AWS IAM + EKS Access Entries.** Instead of a certificate's `/CN=` and `/O=` fields declaring a username and group membership, an **Access Entry** does the same job — mapping an AWS IAM principal directly to a Kubernetes username and group list. Same end result (a real identity, with real group membership, that RBAC can bind against), completely different, EKS-appropriate mechanism.

---

## Step 1 (EKS version): Create the IAM identity and its Access Entry

```bash
export CLUSTER_NAME=scaling-demo-cluster
export AWS_REGION=ap-south-1
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

Create a dedicated IAM user to represent `dicktracy` (in a real org this would already exist — we're creating one purely for this lab):

```bash
aws iam create-user --user-name dicktracy
```

Now create the **Access Entry** — this is the direct equivalent of the certificate's `-subj "/CN=dicktracy/O=devs/O=tech-leads"`:

```bash
aws eks create-access-entry \
  --cluster-name $CLUSTER_NAME \
  --region $AWS_REGION \
  --principal-arn arn:aws:iam::${AWS_ACCOUNT_ID}:user/dicktracy \
  --kubernetes-groups devs tech-leads \
  --type STANDARD
```

**Compare this directly to the certificate approach:** `--kubernetes-groups devs tech-leads` is doing exactly what `/O=devs /O=tech-leads` did in the certificate — declaring, at the identity-creation layer, which RBAC groups this principal belongs to. Kubernetes RBAC never needs to know or care that the underlying mechanism changed from "certificate field" to "IAM Access Entry field" — from RBAC's perspective, both produce the same thing: an authenticated identity carrying group memberships.

### Verify the Access Entry

```bash
aws eks describe-access-entry \
  --cluster-name $CLUSTER_NAME \
  --region $AWS_REGION \
  --principal-arn arn:aws:iam::${AWS_ACCOUNT_ID}:user/dicktracy \
  --query "accessEntry.kubernetesGroups"
```

Expected: `["devs", "tech-leads"]`.

---

## Step 2: Create namespaces

```bash
kubectl create namespace test --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace devs --dry-run=client -o yaml | kubectl apply -f -
kubectl get ns
```

Expected: both `test` and `devs` present (should already exist from Sections 5-6).

---

## Step 3: Apply the Roles and RoleBindings from Sections 5-6

```bash
kubectl apply -f pod-reader.yaml
kubectl apply -f user-pod-reader-rolebinding.yaml
kubectl apply -f simple-dev-role.yaml
kubectl apply -f simple-dev-rolebinding.yaml
```

Recall from Section 6: `user-pod-reader-rolebinding.yaml` binds `User: dicktracy@kubernetes` to `pod-reader` in `test`. `simple-dev-rolebinding.yaml` binds `Group: devs` to `simple-dev-role` in `devs`.

**One important naming note:** on EKS, the Kubernetes-side username produced by an Access Entry defaults to the IAM ARN itself, unless you customize it. For this lab to line up exactly with the `dicktracy@kubernetes` username already baked into our Section 6 RoleBinding YAML, set a custom username on the Access Entry:

```bash
aws eks update-access-entry \
  --cluster-name $CLUSTER_NAME \
  --region $AWS_REGION \
  --principal-arn arn:aws:iam::${AWS_ACCOUNT_ID}:user/dicktracy \
  --username dicktracy@kubernetes
```

---

## Step 4: Verify permissions before switching context -- via impersonation

```bash
kubectl auth can-i list pods -n test --as=dicktracy@kubernetes
# Expected: yes

kubectl auth can-i create pods -n test --as=dicktracy@kubernetes
# Expected: no

kubectl auth can-i list pods -n devs --as=dicktracy@kubernetes --as-group=devs
# Expected: yes (because dicktracy belongs to the devs group)

kubectl auth can-i create pods -n devs --as=dicktracy@kubernetes --as-group=devs
# Expected: yes

kubectl auth can-i list pods -n default --as=dicktracy@kubernetes
# Expected: no
```

**Every result should match your source material's expected table exactly** — the underlying identity-creation mechanism changed (IAM Access Entry instead of a signed certificate), but RBAC evaluation itself is completely identical, because by the time a request reaches RBAC, all it sees is a username string and a list of group strings — it has no idea, and doesn't care, whether those came from a certificate or from IAM.

---

## Step 5 (optional, real IAM credentials required): actually switch context and test live

To genuinely switch `kubectl` context and run commands *as* `dicktracy`, rather than testing via impersonation, you'd need a real, separate set of AWS credentials for the `dicktracy` IAM user (an access key, or an assumable role) — configured as a distinct AWS CLI profile:

```bash
aws iam create-access-key --user-name dicktracy
# Store the returned AccessKeyId/SecretAccessKey as a new profile, e.g.:
aws configure set aws_access_key_id <KEY_ID> --profile dicktracy
aws configure set aws_secret_access_key <SECRET> --profile dicktracy

aws eks update-kubeconfig --name $CLUSTER_NAME --region $AWS_REGION \
  --profile dicktracy --alias dicktracy@kubernetes
```

```bash
kubectl config get-contexts
kubectl config use-context dicktracy@kubernetes

kubectl get pods -n test
# Expected: No resources found (empty, but permitted)

kubectl run testpod --image=nginx -n test
# Expected: Error: pods is forbidden: User "dicktracy@kubernetes" cannot create resource "pods"

kubectl run devpod --image=nginx -n devs
kubectl get pods -n devs
# Expected: pod created and listed successfully

kubectl config use-context <your-original-admin-context>
```

> Note: this step is genuinely optional for a live teaching session -- Step 4's impersonation-based verification already proves the exact same RBAC behavior with zero extra AWS credential setup. Step 5 is worth doing once, live, specifically to show students that impersonation isn't "faking" anything -- a real, separately-authenticated identity gets byte-for-byte identical results.

---

## Summary: the RBAC permission matrix for dicktracy

| Action | Namespace | Reason | Result |
|---|---|---|---|
| `list pods` | `test` | `pod-reader` role via user binding | ALLOWED |
| `create pods` | `test` | `pod-reader` only has `get`, `list` | DENIED |
| `list pods` | `devs` | `simple-dev-role` via group binding | ALLOWED |
| `create pods` | `devs` | `simple-dev-role` via group binding | ALLOWED |
| `list pods` | `default` | No role or binding exists | DENIED |
| `create pods` | `default` | No role or binding exists | DENIED |

---

## Cleanup

```bash
kubectl config use-context <your-original-admin-context>
kubectl delete -f simple-dev-rolebinding.yaml -f simple-dev-role.yaml -f user-pod-reader-rolebinding.yaml -f pod-reader.yaml
aws eks delete-access-entry --cluster-name $CLUSTER_NAME --region $AWS_REGION --principal-arn arn:aws:iam::${AWS_ACCOUNT_ID}:user/dicktracy
aws iam delete-access-key --user-name dicktracy --access-key-id <KEY_ID_IF_CREATED>
aws iam delete-user --user-name dicktracy
```

---

## The one sentence to carry forward

*"Everything in Sections 1 through 6 -- Subject, Resource, Verb, Role, RoleBinding -- is pure Kubernetes RBAC, and works completely identically no matter which cloud, or no cloud at all, you're running on. The only thing that ever changes between platforms is Step 1: how a real, trustworthy identity gets created in the first place. On a self-managed cluster, that's a certificate signed by your own CA. On EKS, it's an IAM principal mapped through an Access Entry. RBAC itself never notices the difference."*

---

*End of Section 7. Section 8: Role vs ClusterRole -- the final piece, extending everything you've built to cluster-scoped resources.*
