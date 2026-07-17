# Chapter 3 Lab Walkthrough — Understanding Your Ingress Controller Installation

This document explains, step by step, exactly what happened when you ran the install command on your EKS cluster, using your own actual output as the reference.

---

## Step 1: The command you ran

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/aws/deploy.yaml
```

This single command downloaded one large YAML file from GitHub and applied **every object inside it** to your cluster in one shot. That file bundles together many different Kubernetes objects — not just one.

---

## Step 2: What actually got created

### `namespace/ingress-nginx created`
Created a new namespace called `ingress-nginx`. Think of a namespace as a separate folder inside your cluster — everything related to the Ingress Controller lives inside this one folder, kept separate from your other apps.

### `serviceaccount/ingress-nginx created`
Created an identity the controller pod uses when talking to the Kubernetes API — a login account for the controller itself, not for a human.

### `role` / `clusterrole` lines (several)
Permission rules. The controller needs permission to read Ingress objects cluster-wide, read Services, read Secrets for TLS certificates, and more. These lines grant those permissions.

### `rolebinding` / `clusterrolebinding` lines
Connect the permissions above to the actual service account. A role by itself does nothing until it's bound to someone — this is that binding step.

### `configmap/ingress-nginx-controller created`
A configuration file stored inside Kubernetes, holding NGINX settings — timeouts, buffer sizes, and similar.

### `service/ingress-nginx-controller created` — the important one
This created a Service of `type: LoadBalancer`. **This is what triggered AWS to provision a real Network Load Balancer outside your cluster.** This is the entry point for all external traffic.

### `service/ingress-nginx-controller-admission created`
A small internal Service, `ClusterIP` only, not reachable from outside. Used only for validation checks when someone applies an Ingress object later — not part of the main traffic path.

### `deployment.apps/ingress-nginx-controller created`
The actual controller software. This Deployment manages the pod that runs NGINX and performs the real Layer 7 routing work.

### `job.batch` lines (`admission-create`, `admission-patch`)
One-time setup jobs. They run once, generate a certificate needed for the admission webhook to work securely, then finish and disappear. You saw them cycle through `ContainerCreating` → `Completed` → `Terminating` in your output — that full lifecycle is expected and correct, not an error.

### `ingressclass.networking.k8s.io/nginx created`
Registers `nginx` as an available IngressClass. When you write a real Ingress object later, you'll reference this class name so Kubernetes knows which controller should handle it.

### `validatingwebhookconfiguration created`
Tells Kubernetes to call the admission service before accepting any new Ingress object, to validate it before saving it.

---

## Step 3: Verifying what you got

### First check:
```bash
kubectl get all -n ingress-nginx
```
```
pod/ingress-nginx-controller-c74ccf678-jd7gg   0/1   Running   0   10s
```
At this exact moment, the pod was still starting up. `Running` means the container process started, but `0/1` means it hadn't yet passed its readiness check. **Completely normal in the first few seconds — not an error.**

### Second check, moments later:
```bash
kubectl get pods -n ingress-nginx
```
```
ingress-nginx-controller-c74ccf678-jd7gg   1/1   Running   0   64s
```
Now `1/1` — the pod is fully ready and serving traffic. The difference between `0/1` and `1/1` is simply time passing while NGINX finished starting up inside the container.

---

## Step 4: The most important line to understand

```
service/ingress-nginx-controller   LoadBalancer   10.100.48.157   aef276c7c98614d3a810a049096c5f01-44684a75608246d7.elb.ap-south-1.amazonaws.com   80:31240/TCP,443:30418/TCP
```

Breaking this down field by field:

| Field | Value | Meaning |
|---|---|---|
| `TYPE` | `LoadBalancer` | Confirms AWS created a real load balancer automatically, triggered just by creating this Service object |
| `CLUSTER-IP` | `10.100.48.157` | Internal-only address, reachable only from inside the cluster — not useful to you directly |
| `EXTERNAL-IP` | `aef276c7...amazonaws.com` | Your real, public-facing Network Load Balancer hostname — this is what the internet would use to reach your cluster |
| `PORT(S)` | `80:31240/TCP,443:30418/TCP` | Left side = ports the load balancer listens on (standard web ports). Right side = NodePorts opened on your worker nodes |

**Why a hostname instead of a plain IP:** notice it's a long `amazonaws.com` address, not something like `34.123.45.67`. This is normal and AWS-specific — AWS Network Load Balancers are addressed by hostname, not a fixed IP, unlike the simplified example used in the concept explanation earlier.

**The traffic path this describes:** the load balancer receives traffic on port 80 or 443, forwards it to one of your nodes on NodePort 31240 or 30418, and from there it reaches the controller pod.

---

## Step 5: Locating the controller pod itself

```bash
kubectl get pods -n ingress-nginx -o wide
```
```
ingress-nginx-controller-c74ccf678-jd7gg   1/1   Running   0   79s   192.168.60.178   ip-192-168-55-39.ap-south-1.compute.internal
```

This confirms the controller pod has its own internal pod IP (`192.168.60.178`) and is physically running on one specific node (`ip-192-168-55-39`). **This is the real pod that will read Ingress rules and perform the actual Layer 7 routing decisions** covered in the concept part of this chapter.

---

## Step 6: Where things stand right now

You've successfully installed **the builder**, but you haven't yet handed it **a blueprint**. In plain terms:

- The controller is running and ready ✅
- It has a real public load balancer in front of it ✅
- There is no Ingress object yet telling it what to route ⏳

If you sent traffic to that load balancer address right now, NGINX would respond — but only with a generic default backend response, since it has no rules configured yet.

**This is the expected, correct state to be in.** Writing the first real Ingress object, with actual routing rules, is the next chapter.

---

## Summary

You ran one command that created many objects at once. The two that matter most:

1. **The Deployment** — the actual controller software doing the work
2. **The Service of `type: LoadBalancer`** — which caused AWS to create a real Network Load Balancer and gave you a public hostname

Everything else — roles, bindings, the config map, admission jobs, the webhook — exists to support those two main pieces working correctly and securely. Your lab output shows every one of these pieces came up successfully, with no errors.
