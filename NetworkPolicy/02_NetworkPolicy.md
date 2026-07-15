
## What Step 1 actually did

**The core problem:** EKS's default networking component (the VPC CNI, running as the `aws-node` pods you just saw) ships with NetworkPolicy enforcement **turned off** by default. Without this step, every `NetworkPolicy` you create later in the lab would still get accepted by Kubernetes (`kubectl apply` would say "created," no errors) — but it would have **zero actual effect on traffic**. That's the exact trap discussed in the module: policies that exist on paper but enforce nothing.

## What each command specifically did

**1. The `aws eks update-addon` call** — you didn't install anything new. `vpc-cni` was already running (it's mandatory — every EKS cluster has it, it's what gives pods their networking in the first place). What you did was **reconfigure the existing add-on**, flipping one setting: `enableNetworkPolicy: true`. Think of it like a light switch that already existed on the wall — you didn't install new wiring, you just flipped the switch that was sitting there off.

**2. What happens behind that flag, mechanically:** once enabled, AWS automatically deploys an extra piece — the **network policy agent** — onto every node, as part of the same `aws-node` DaemonSet you're already seeing. This is what actually reads your `NetworkPolicy` objects and translates them into **eBPF programs** — small, kernel-level packet filters that intercept traffic *before* it even reaches your container, and decide allow/deny at that layer. This is why it's fast (kernel-level, not an extra network hop) and why it needed to be explicitly turned on (it's genuinely new machinery running on every node, not just a config flag with no real footprint).

**3. `describe-addon ... status: ACTIVE`** — this confirmed the reconfiguration finished successfully and rolled out to the cluster. If it had failed, you'd see `DEGRADED` or `FAILED` here instead.

**4. `kubectl get pods -n kube-system | grep aws-node`** — this just confirmed the DaemonSet pods are healthy post-update (`2/2 Running` — note **2 containers per pod now**, not 1; that second container is exactly the new network policy agent we just talked about. If a student asks "how do I know the policy feature is actually active, not just the addon," point them at that `2/2` — a single VPC CNI pod without policy enforcement would typically show `1/1`).

## The one-sentence version for students who just want the takeaway

> "We didn't build anything new — we turned on a dormant feature inside EKS's existing networking layer, which deploys a background agent on every node whose whole job is to actually enforce the NetworkPolicy YAML files we're about to write. Without this step, every policy in this lab would silently do nothing."

That's the full scope of Step 1 — everything from Step 2 onward (namespace, apps, policies) is standard Kubernetes and would look identical on any cluster; Step 1 is the one EKS-specific piece of plumbing required to make it actually work.
