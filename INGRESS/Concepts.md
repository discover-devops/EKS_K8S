
## Start with the phone system analogy — everyone has lived this

"Imagine calling a company's customer service line. There are two completely different people who could handle your call, doing two completely different jobs:

**Person 1 — the phone network itself.** All it knows is: *this phone number* is trying to reach *that phone number*. It connects the call. It has absolutely no idea what you're going to say once the call connects — it can't hear a word of the conversation. Its entire job is just: 'connect number A to number B.'

**Person 2 — the actual receptionist who picks up.** This person *listens* to what you say. 'I want billing.' 'I want technical support.' 'I want sales.' Based on what you *say* — the actual content of your request — they transfer you to the right department.

Now here's the key question: **which one of these two can route you based on what you're asking for — the phone network, or the receptionist?**

Obviously — the receptionist. The phone network can only ever connect call A to call B. It cannot possibly say 'oh, they mentioned billing, let me route this specially' — it doesn't even hear the words. Only the receptionist, who actually listens to content, can make that decision."

## Now map it directly onto Kubernetes — same two roles, same two capabilities

"A `LoadBalancer` Service is the **phone network**. It only ever asks: 'traffic came in on this IP, this port — where do I send it?' It never looks *inside* the request. It has no idea if you typed `api.company.com` or `admin.company.com` into your browser, and it has no idea if you're requesting `/users` or `/products`. It just connects the call and forwards it.

**Ingress is the receptionist.** It actually *reads* your request — the hostname you typed, the path you're asking for — and *then* decides where to send you, based on that content.

That's the entire difference. One connects calls without listening. The other listens, and routes based on what it hears."

## Only now introduce the formal terms — as labels for what they already understand

"In networking, there's a name for 'just connecting based on address, without looking at content' — that's called **Layer 4**. And there's a name for 'actually reading the content of the request and deciding based on that' — that's called **Layer 7**.

You don't need to memorize what Layers 1 through 6 are. For this course, you only need to know:

- **Layer 4 = the phone network. Connects based on address only. Never reads content.**
- **Layer 7 = the receptionist. Reads the actual content, then decides.**

A `LoadBalancer` Service is a Layer 4 tool — it's the phone network. Every limitation we found — can't route by hostname, can't route by path, can't split traffic by percentage — all come from the exact same fact: **it's simply not capable of reading content, because that's not the job it does.** Ingress exists because we needed a receptionist, not just a phone network."

## The one line to close the explanation with, tying it all together

> "So when I say 'Layer 4 can't do host-based or path-based routing' — I don't mean it's *bad* at it. I mean it's structurally *incapable* of it, the same way a phone network can never route your call based on what you say, because it never listens in the first place. Ingress isn't a smarter version of the same tool — it's a fundamentally different layer of tool, one that actually reads the request instead of just connecting it."

That's the whole concept, in language that needs zero prior networking background, while still preserving "Layer 4" and "Layer 7" as the correct technical vocabulary once they're ready to hear it.



>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>


## The precise, correct statement

**A Kubernetes `Service` of `type: LoadBalancer` provisions an AWS Layer 4 load balancer — a Classic Load Balancer (or NLB, depending on annotations) — never an ALB.**

This isn't a limitation of AWS. AWS can absolutely create an ALB. It's a limitation of **what a Kubernetes Service object is capable of expressing**. A `Service` spec only contains: selector, port, targetPort. There is no field in a `Service` object for hostname or path. Since it has no such fields, the cloud provider integration that watches `Service` objects (`type: LoadBalancer`) has no information to hand to AWS beyond IP+port — so it can only ever request a Layer 4 load balancer, because that's all the information a `Service` object carries.

## Where ALB actually comes from in Kubernetes

**A Kubernetes `Ingress` object is what triggers ALB provisioning on EKS**, via the AWS Load Balancer Controller. `Ingress` objects *do* contain `host` and `path` fields — that's the extra information required to justify creating a Layer 7 device. The controller reads those fields and provisions a real ALB configured with matching listener rules.

## So the correct mapping is:

| Kubernetes object | AWS load balancer type provisioned | Layer |
|---|---|---|
| `Service, type: LoadBalancer` | Classic ELB or NLB | 4 |
| `Ingress` (with AWS Load Balancer Controller installed) | ALB | 7 |

## Corrected one-line statement for your slide/script

> "It's not that Layer 7 load balancers don't exist — ALB exists and is Layer 7 capable. The point is: a Kubernetes `Service` object has no fields for hostname or path, so it can only ever request a Layer 4 load balancer from AWS. To get an ALB, you need a Kubernetes object that actually carries hostname/path information — that object is `Ingress`, not `Service`."

