Chapter 3 Lab - Installing the Ingress Controller - Plain Explanation

This document explains, step by step, what happened when you ran the install command on your EKS cluster. No special formatting, just plain reading.

STEP 1 - What command did you run

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/aws/deploy.yaml

This single command downloaded one big YAML file from GitHub and applied every object inside it to your cluster in one shot. That file contains many different Kubernetes objects bundled together, not just one.

STEP 2 - What actually got created, one line at a time

namespace/ingress-nginx created
This created a new namespace called ingress-nginx. Think of a namespace as a separate folder inside your cluster. Everything related to the ingress controller will live inside this one folder, kept separate from your other apps.

serviceaccount/ingress-nginx created
This created an identity that the controller pod will use when it talks to the Kubernetes API. Think of it as a login account for the controller itself, not for a human.

role and clusterrole lines (several of them)
These are permission rules. The controller needs permission to read Ingress objects across the entire cluster, read Services, read Secrets for TLS certificates, and so on. These lines are Kubernetes granting the controller those permissions.

rolebinding and clusterrolebinding lines
These connect the permissions above to the actual service account. A role by itself does nothing until it is bound to someone. This is that binding step.

configmap/ingress-nginx-controller created
This is a configuration file stored inside Kubernetes. It holds settings for how NGINX itself should behave, things like timeouts and buffer sizes.

service/ingress-nginx-controller created
This is the important one. This created a Service of type LoadBalancer. This is what triggered AWS to actually provision a real Network Load Balancer outside your cluster. This is the entry point for all external traffic.

service/ingress-nginx-controller-admission created
This is a small internal Service, ClusterIP only, not reachable from outside. It is used only for validation checks when someone applies an Ingress object later. Not part of the main traffic path.

deployment.apps/ingress-nginx-controller created
This is the actual controller software itself. A Deployment manages the pod that runs NGINX and does the real Layer 7 routing work.

job.batch lines (admission-create and admission-patch)
These are one-time setup jobs. They run once, generate a certificate needed for the admission webhook to work securely, then finish and disappear. You saw them go through ContainerCreating, then Completed, then Terminating in your output. That full lifecycle is expected and correct.

ingressclass.networking.k8s.io/nginx created
This registers nginx as an available IngressClass. Later, when you write an actual Ingress object, you will reference this class name so Kubernetes knows which controller should handle that particular Ingress.

validatingwebhookconfiguration created
This tells Kubernetes to call the admission service before accepting any new Ingress object, to check it is valid before saving it.

STEP 3 - Checking what you got, command by command

Command you ran
kubectl get all -n ingress-nginx

What it showed the first time
pod/ingress-nginx-controller-c74ccf678-jd7gg   0/1   Running   0   10s

At this exact moment the pod was still starting up. Running means the container process has started, but 0/1 means it had not yet passed its readiness check. This is completely normal for the first few seconds. It is not an error.

Command you ran again a bit later
kubectl get pods -n ingress-nginx

What it showed
ingress-nginx-controller-c74ccf678-jd7gg   1/1   Running   0   64s

Now it shows 1/1. This means the pod is fully ready and serving traffic. The difference between 0/1 and 1/1 is simply time passing while NGINX finished starting up inside the container.

STEP 4 - The most important line to understand

service/ingress-nginx-controller   LoadBalancer   10.100.48.157   aef276c7c98614d3a810a049096c5f01-44684a75608246d7.elb.ap-south-1.amazonaws.com   80:31240/TCP,443:30418/TCP

Read this line piece by piece.

TYPE is LoadBalancer. This confirms AWS created a real load balancer for you automatically, just by you creating this one Service object.

CLUSTER-IP is 10.100.48.157. This is an internal-only address, only reachable from inside the cluster. Not useful to you directly.

EXTERNAL-IP is that long amazonaws.com address. This is your real, public-facing Network Load Balancer hostname. This is the address the whole internet would use to reach your cluster, if you pointed a DNS name at it. Notice it is a hostname, not a plain number like 34.123.45.67. That is normal for AWS specifically. AWS Network Load Balancers are addressed by hostname, not by a fixed IP, unlike the simplified example used in the concept explanation earlier.

PORTS shows 80:31240/TCP,443:30418/TCP. The left side, 80 and 443, are the ports the load balancer itself listens on, the normal web ports. The right side, 31240 and 30418, are NodePorts, meaning ports opened directly on your worker nodes. The load balancer receives traffic on 80 or 443, then forwards it to one of your nodes on port 31240 or 30418, and from there it reaches the controller pod.

STEP 5 - Checking where the controller pod actually lives

Command you ran
kubectl get pods -n ingress-nginx -o wide

What it showed
ingress-nginx-controller-c74ccf678-jd7gg   1/1   Running   0   79s   192.168.60.178   ip-192-168-55-39.ap-south-1.compute.internal

This confirms the controller pod has its own internal pod IP, 192.168.60.178, and is physically running on one specific node, ip-192-168-55-39. This is the real pod that will read Ingress rules later and perform the actual Layer 7 routing decisions we discussed in the concept part of this chapter.

STEP 6 - Where things stand right now

At this point you have successfully installed the builder, but you have not yet handed it a blueprint. In plain terms, the controller is running and ready, it has a real public load balancer in front of it, but there is no Ingress object yet telling it what to route. If you sent traffic to that load balancer address right now, NGINX would respond, but only with a generic default backend response, since it has no rules configured yet.

This is expected. Writing the first real Ingress object, with actual routing rules, is the next chapter.

SUMMARY IN ONE PARAGRAPH

You ran one command that created many objects at once. The two that matter most are the Deployment, which is the actual controller software doing the work, and the Service of type LoadBalancer, which caused AWS to create a real Network Load Balancer and gave you a public hostname. Everything else, the roles, bindings, config map, admission jobs, and webhook, exist to support those two main pieces working correctly and securely. Your lab output shows every one of these pieces came up successfully, with no errors anywhere in the output you shared.
