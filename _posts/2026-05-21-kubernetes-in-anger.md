---
layout: post
title: Kubernetes In Anger
category: kubernetes, eks, troubleshooting
mermaid: true
---

## 0. Quick start (emergency edition)

### Is this the right guide?

**YES, if:**
- You're debugging a live EKS production issue
- You need to upgrade/change EKS safely
- You want to prevent common EKS outages
- You're oncall for EKS workloads

**NO, if:**
- You're learning Kubernetes basics (try the official tutorials first)
- You need EKS setup instructions (use AWS documentation)
- You want comprehensive Kubernetes reference (use kubernetes.io)

### Emergency shortcuts

**Cluster is on fire right now?** → Jump to [Section 2.10 Tier-0 Incident Playbook](#210-what-to-do-when-a-tier-0-component-is-unhealthy-eks-incident-playbook)

**Need to upgrade safely?** → Jump to [Section 8 Upgrades and maintenance](#8-upgrades-and-maintenance)

**Investigating an incident?** → Start with [Section 1.2 Quick Cluster Health Snapshot](#12-quick-cluster-health-snapshot-30-seconds)

### Prerequisites

This guide assumes you know:
- Basic kubectl commands (`get`, `describe`, `logs`)
- AWS CLI basics
- What pods, services, and deployments are
- How to read YAML manifests

### What makes EKS different

EKS is not "just Kubernetes". Key differences that affect reliability:
- Pods get real VPC IPs (AWS VPC CNI)
- AWS services become dependencies (NAT, NLB, EBS)
- Node limits are AWS EC2 limits
- Networking failures look like application failures
- Upgrades affect multiple AWS components

---

## Introduction

### On running infrastructure

There's a common way of thinking about Kubernetes that goes something like this: you declare what you want, the system converges toward it, and your job is mostly done. Write the YAML, apply it, the scheduler places your pods, the controllers reconcile state, and everything just works.

This is roughly true until it isn't.

The thing about Kubernetes — and EKS specifically — is that it doesn't fail like a monolith fails. A monolith crashes and you know it. EKS degrades. DNS gets slow. A node hits a network limit you didn't know existed. Pods keep running but their connections reset every 6 minutes. The dashboard is green. Customers are complaining. You're staring at healthy pods wondering what's wrong with your application, when the real problem is three layers down in a conntrack table or a subnet that ran out of IPs.

Most other platforms fail at the boundary between your code and the infrastructure. EKS fails *inside* the infrastructure, in ways that look like your code is broken. This is the fundamental debugging challenge: the symptom is always "the app is slow" or "requests are failing", and the cause is somewhere in a stack of networking, scheduling, storage, and AWS service interactions that your application has no visibility into.

This matters because the instinct — "my app is returning 5xx, let me look at my app" — is wrong most of the time in EKS. The 5xx is real. But the fix is often in a probe configuration, a security group limit, a DNS resolver being overwhelmed, or a node that silently filled its conntrack table.

### The two jobs

If you run EKS in production, you have two jobs:

The first is building workloads that survive the platform misbehaving. Probes that don't cascade. Graceful shutdowns that actually drain. Pod distributions that tolerate losing a node or an AZ without paging anyone. This is the preventive work — the engineering equivalent of washing your hands.

The second is diagnosing live systems when things go wrong anyway. Connecting to a cluster that's on fire, figuring out what's actually broken vs what's just symptomatic, collecting evidence before it disappears, and fixing the right thing without making the incident worse. This is the equivalent of surgery — you're operating on a patient that's still awake and serving traffic.

Both matter. Most guides only cover the first one.

This guide is about both. It's a collection of patterns, failure modes, and diagnostic workflows that came from running EKS in production — the things that caused real incidents, the things that made debugging take hours instead of minutes, and the guardrails that prevented repeat occurrences.

### Who is this for?

This guide is not for beginners. There's a gap between knowing Kubernetes concepts (pods, deployments, services, kubectl) and actually being able to keep an EKS cluster healthy in production. There's a fumbling phase where you've read the docs, passed the certification maybe, deployed some workloads — and then something breaks at 2am and you realize you don't know where to look or what's safe to touch.

This assumes you know the basics. It does not assume you know how to debug a cluster that's misbehaving, how EKS-specific failure modes differ from generic Kubernetes ones, or what the safe sequence of actions is when you're staring at a production incident.

What you won't find here: how to set up EKS, what a pod is, or how to write a Deployment manifest. What you will find: what to do when pods are Pending and you don't know why, how to tell if DNS is the problem or just a symptom, why your NLB keeps resetting connections, and how to collect evidence before the cluster auto-heals and destroys your ability to do an RCA.

### How to read this guide

The guide is organized by domain — networking, storage, security, observability, scaling, upgrades, and so on. Each section mixes both jobs: how to build it right, and how to debug it when it breaks. You'll find design patterns and diagnostic runbooks side by side, because in practice you need both at the same time.

You can read it front-to-back if you're setting up a new cluster or onboarding to an existing one. Or you can jump to the relevant section when something breaks — each one is self-contained enough to be useful on its own.

If the cluster is on fire right now, start at Section 1. It gives you a triage sequence to identify the failure domain in under 2 minutes.

---
## 1. How to dive into an EKS cluster

When production is broken the first job is to mitigate — stop the bleeding, restore service, reduce blast radius. But you can't mitigate effectively if you don't know *what's broken*. Rollback the wrong thing and you've wasted 10 minutes. Upsize the wrong component and nothing changes.

So the actual first job is: figure out where the problem is, fast enough that you can pick the right mitigation within 2 minutes. Not root cause — just failure domain:

* One pod?
* One deployment / namespace?
* One node group / AZ?
* The entire cluster?
* AWS integration (CNI / LB / EBS)?
* An upstream dependency (RDS, Redis, external APIs)?

Once you know the failure domain, you mitigate (rollback, drain, upsize, block). Root cause comes after the incident is contained.

What follows is a reliable entry sequence to get that signal fast, without guessing.

---

### 1.1 Establish Context (don't debug the wrong cluster)

Before anything else, confirm you're looking at the right cluster and identity.

**Commands**

```bash
kubectl config current-context
kubectl cluster-info
aws sts get-caller-identity
```

**What you're checking**

* You're in the correct kubecontext (prod vs staging mistakes happen)
* You have valid AWS credentials
* The API server is reachable at all

If `kubectl cluster-info` is slow or timing out, that's already a strong signal:

* API server under load
* auth problems
* network path issues from your machine (VPN / corp DNS / proxy)

---

### 1.2 Quick Cluster Health Snapshot (30 seconds)

This is the fastest "is the cluster sick?" view.

**Commands**

```bash
kubectl get nodes -o wide
kubectl get pods -A --field-selector=status.phase!=Running
kubectl get events -A --sort-by=.lastTimestamp | tail -n 50
```

**What you're checking**

* Any nodes `NotReady`
* Any pods stuck in `Pending`, `CrashLoopBackOff`, `ImagePullBackOff`
* Events that scream the root cause:
  * `FailedScheduling`
  * `FailedMount`
  * `Unhealthy` (probe failures)
  * `Back-off restarting failed container`
  * `FailedCreatePodSandBox` (CNI problems)

This is where you decide:

* **workload issue** vs **node issue** vs **cluster-wide issue**

---

### 1.3 Confirm EKS System Components (the "platform basics")

If system components are down, application debugging is mostly pointless.

**Commands**

```bash
kubectl get pods -n kube-system -o wide
kubectl get ds -n kube-system
kubectl get deploy -n kube-system
```

**Focus on these in EKS**

* `coredns` (cluster DNS)
* `aws-node` (AWS VPC CNI)
* `kube-proxy` (unless you're on eBPF dataplane)
* `ebs-csi-node` / `ebs-csi-controller` (if you use EBS CSI)
* `metrics-server` (if HPA depends on it)

**Red flags**

* CoreDNS pods not ready → widespread service discovery failures
* `aws-node` not ready → pods can't get IPs / networking breaks
* EBS CSI issues → StatefulSets fail to mount volumes

---

### 1.4 Decide: Is this a "Scheduling" Problem?

If pods are Pending, do this immediately.

**Commands**

```bash
kubectl get pods -A | grep -E "Pending|ContainerCreating"
kubectl describe pod -n <ns> <pod>
```

**What you're looking for in `describe`**

* `FailedScheduling`
  * insufficient CPU/memory
  * taints not tolerated
  * affinity rules too strict
  * topology spread constraints blocking placement
* `Insufficient pods` / `Too many pods`
  * node has hit max pod density (ENI/IP limits)
* `node(s) had volume node affinity conflict`
  * common with EBS + AZ mismatch

If scheduling fails, do **not** restart the deployment blindly. It won't help.

---

### 1.5 Decide: Is this a "Node" Problem?

If nodes are NotReady or workloads are failing on specific nodes, zoom in.

**Commands**

```bash
kubectl describe node <node-name>
kubectl top nodes
kubectl get pods -A -o wide | grep <node-name>
```

**What to check on the node**

* `Conditions`: MemoryPressure / DiskPressure / PIDPressure
* `Allocatable` vs `Allocated resources`
* Events mentioning:
  * kubelet issues
  * container runtime issues
  * frequent reboots
  * network plugin failures

In EKS, node failures often correlate with:

* EBS issues
* CNI/IP exhaustion
* disk full (especially on small root volumes)
* aggressive DaemonSets consuming resources

---

### 1.6 Decide: Is this a "Network / CNI" Problem?

EKS networking failures are often AWS VPC CNI related.

**Symptoms**

* Pods stuck at `ContainerCreating`
* `FailedCreatePodSandBox`
* random cross-service timeouts
* sudden increase in Pending pods (no IPs)

**Commands**

```bash
kubectl -n kube-system logs -l k8s-app=aws-node --tail=100
kubectl -n kube-system describe ds aws-node
```

**Also check pod density**

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.pods}{"\n"}{end}'
```

If you're hitting pod density/IP limits:

* scaling nodes may help
* but the real fix is often **prefix delegation** / node type sizing / ENI planning

---

### 1.7 Decide: Is this a "DNS / CoreDNS" Problem?

DNS issues can look like "application is broken".

**Symptoms**

* timeouts to internal services
* failures resolving service names
* sudden spike in retries / connection errors

**Commands**

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=200
```

If CoreDNS is unhealthy:

* don't waste time debugging app-level service discovery logic
* fix CoreDNS capacity / upstream resolver / node networking first

---

### 1.8 Decide: Is this a "Storage / EBS CSI" Problem?

Stateful workloads fail differently.

**Symptoms**

* pods stuck in `ContainerCreating`
* `FailedMount`
* volumes not attaching

**Commands**

```bash
kubectl describe pod -n <ns> <pod>
kubectl get pvc -A
kubectl -n kube-system get pods | grep ebs
kubectl -n kube-system logs deploy/ebs-csi-controller --tail=200
```

**What you're looking for**

* volume attachment timeouts
* AZ mismatch
* stuck PV/PVC lifecycle
* CSI controller/node not healthy

---

### 1.9 Find the "Blast Radius" (what is actually impacted)

Before you attempt changes, quantify impact.

**Commands**

```bash
kubectl get pods -A | wc -l
kubectl get pods -A --field-selector=status.phase!=Running | head -n 50
kubectl get nodes | grep -v Ready
```

**Interpretation**

* If only one namespace is impacted → likely app or namespace-level dependency
* If one node group/AZ is impacted → capacity, subnet, EBS/AZ, or rollout targeting issue
* If kube-system is unhealthy → platform issue, stop chasing app symptoms

---

### 1.10 Evidence Collection (before you change anything)

Kubernetes evidence disappears fast (pods restart, nodes recycle, events roll over).

Capture minimal evidence first.

**Commands**

```bash
kubectl get events -A --sort-by=.lastTimestamp | tail -n 200 > events.txt
kubectl get nodes -o wide > nodes.txt
kubectl get pods -A -o wide > pods.txt
```

If it's a single workload incident:

```bash
kubectl describe pod -n <ns> <pod> > pod.describe.txt
kubectl logs -n <ns> <pod> --previous > pod.prev.log.txt
```

This makes the RCA possible later, without relying on memory and guesswork.

---

### 1.11 Node-Specific Failures (Conntrack, Sysctls, and Kubelet Settings)

Some of the nastiest EKS incidents are **node-local**. The cluster looks "fine", pods are "Running", but traffic becomes unreliable, latency spikes, or connections fail randomly.

These issues often come from:

* **conntrack exhaustion**
* **ephemeral port exhaustion**
* **kernel / sysctl defaults not sized for your traffic**
* **kubelet behaviour under pressure**
* **bad QoS due to incorrect requests/limits**

#### 1.11.1 Conntrack Exhaustion (classic "random networking failures")

**What it looks like**

* intermittent timeouts to upstream services
* random 5xx at ingress / Envoy / Nginx
* "connection reset", "i/o timeout", "no route to host" type errors
* affects specific nodes more than others
* spikes during traffic bursts or connection-heavy workloads

**Why it happens**
Linux conntrack tracks NAT and connection state. On busy nodes (especially with L7 proxies, service meshes, high churn, short-lived connections), conntrack tables fill up and the kernel starts dropping new connections.

**Quick checks (from Kubernetes side)**

1. Identify if failures correlate to a node:

```bash
kubectl get pods -A -o wide | grep <node-name>
kubectl describe node <node-name>
```

2. Check node-level kernel counters (best effort)
   If you have node access (SSM/SSH):

```bash
sudo sysctl net.netfilter.nf_conntrack_count
sudo sysctl net.netfilter.nf_conntrack_max
dmesg | egrep -i "conntrack|nf_conntrack"
```

**Strong indicators**

* `nf_conntrack_count` close to `nf_conntrack_max`
* kernel logs mentioning conntrack table full / dropped packets

**Fix patterns**

* Increase conntrack max (sysctl)
* Reduce connection churn (keep-alives, pooling)
* Spread load (more nodes / better pod distribution)
* Ensure nodes aren't overloaded with too many L7-heavy pods

---

#### 1.11.2 Ephemeral Port Exhaustion (the sneaky cousin)

**What it looks like**

* outbound calls failing from a node under burst load
* retries make it worse
* symptoms disappear when traffic drops

**Quick checks**
On node:

```bash
cat /proc/sys/net/ipv4/ip_local_port_range
ss -s
ss -ant state time-wait | wc -l
```

**Fix patterns**

* widen ephemeral port range
* reduce TIME_WAIT pressure (carefully)
* enable connection reuse/keepalive at clients and proxies

---

### 1.12 Evidence Collection Automation

When production is broken, evidence disappears fast. Capture it first, debug second.

**Quick evidence collection script**

```bash
#!/bin/bash
# Save as: collect-evidence.sh
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
EVIDENCE_DIR="evidence-${TIMESTAMP}"
mkdir -p "$EVIDENCE_DIR"

echo "Collecting evidence to $EVIDENCE_DIR..."

# Cluster overview
kubectl get nodes -o wide > "$EVIDENCE_DIR/nodes.txt"
kubectl get pods -A -o wide > "$EVIDENCE_DIR/pods-all.txt"
kubectl get events -A --sort-by=.lastTimestamp > "$EVIDENCE_DIR/events-all.txt"

# Tier-0 components
kubectl -n kube-system get pods -o wide > "$EVIDENCE_DIR/kube-system-pods.txt"
kubectl -n kube-system describe pods > "$EVIDENCE_DIR/kube-system-describe.txt"

# Recent events (last 200)
kubectl get events -A --sort-by=.lastTimestamp | tail -n 200 > "$EVIDENCE_DIR/events-recent.txt"

# Unhealthy pods
kubectl get pods -A --field-selector=status.phase!=Running > "$EVIDENCE_DIR/unhealthy-pods.txt"

# Node conditions
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\t"}{.status.conditions[?(@.type=="MemoryPressure")].status}{"\t"}{.status.conditions[?(@.type=="DiskPressure")].status}{"\n"}{end}' > "$EVIDENCE_DIR/node-conditions.txt"

echo "Evidence collected in $EVIDENCE_DIR"
echo "Attach this directory to your incident ticket"
```

**What to capture for specific failure types**

**DNS/CoreDNS issues**
```bash
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=500 > coredns-logs.txt
kubectl -n kube-system get pods -l k8s-app=kube-dns -o yaml > coredns-pods.yaml
```

**CNI/Networking issues**
```bash
kubectl -n kube-system logs -l k8s-app=aws-node --tail=500 > aws-node-logs.txt
kubectl get pods -A -o wide | grep -E "ContainerCreating|Pending" > stuck-pods.txt
```

**Node-specific issues**
```bash
# Replace NODE_NAME with actual node
kubectl describe node NODE_NAME > node-describe.txt
kubectl get pods -A -o wide | grep NODE_NAME > node-pods.txt
```

---

### 1.13 Common False Positives

These look like EKS issues but usually aren't:

#### "Kubernetes is slow" (but it's not)

**Symptoms:**
- kubectl commands are slow
- Deployments take forever
- "Everything is sluggish"

**Usually actually:**
- Your laptop's VPN/network to cluster
- AWS API throttling (too many concurrent kubectl users)
- Your kubeconfig pointing to wrong cluster/region

**Quick check:**
```bash
time kubectl get nodes
# Should complete in <2 seconds for healthy cluster
```

#### "Pods keep restarting" (but EKS is fine)

**Symptoms:**
- High restart count
- CrashLoopBackOff
- "Kubernetes keeps killing my app"

**Usually actually:**
- Application bugs (not Kubernetes bugs)
- Incorrect liveness probe configuration
- Resource limits too low (OOMKilled)
- Missing dependencies (DB, Redis, etc.)

**Quick check:**
```bash
kubectl describe pod POD_NAME
# Look at "Last State" and "Reason"
# OOMKilled = memory limit too low
# Error = application crash
```

#### "Service discovery is broken" (but DNS is fine)

**Symptoms:**
- Services can't reach each other
- "Connection refused" errors
- "Name resolution failures"

**Usually actually:**
- Wrong service name/namespace in application config
- Application listening on wrong port
- Readiness probe failing (pod not ready to receive traffic)
- Network policies blocking traffic

**Quick check:**
```bash
# Test DNS resolution from inside a pod
kubectl exec -it POD_NAME -- nslookup SERVICE_NAME.NAMESPACE.svc.cluster.local
# Test if service endpoints exist
kubectl get endpoints SERVICE_NAME
```

---

### What you should know by now

After running through the above, you should be able to answer these with evidence:

1. **What is broken, and what is not?**
   * One pod vs one workload vs one namespace vs one node group vs entire cluster

2. **Where is the failure domain?**
   * **Workload-level** (bad rollout, probe failures, config/secret issues)
   * **Cluster-level** (kube-system degradation, DNS, CNI, storage controller issues)
   * **AWS integration layer** (VPC CNI / ENI/IP limits, ALB/NLB behaviour, EBS attach/detach)
   * **Node-level** (resource pressure, kubelet instability, kernel/network issues)

3. **What category does this incident fall into?**
   * Scheduling/capacity (`FailedScheduling`, Pending pods)
   * Networking/CNI (pod sandbox failures, IP exhaustion, random timeouts)
   * DNS/CoreDNS (service discovery failures)
   * Storage/EBS CSI (FailedMount, volume attach issues)
   * Control plane/API issues (timeouts, throttling, admission webhook failures)

4. **Is this node-specific "kernel pain" or cluster-wide?**
   * Conntrack exhaustion / ephemeral port pressure
   * Mis-sized sysctls
   * Kubelet eviction behaviour under pressure
   * Incorrect QoS class due to bad requests/limits

5. **What is the blast radius and what's the next safe action?**
   * Can you isolate by draining/cordoning nodes?
   * Should you pause a rollout?
   * Should you scale out node groups / reduce pressure?
   * Or do you need to stop and fix platform components first?

6. **What evidence did you capture before making changes?**
   * Events, node state, pod distribution, and logs that will make the RCA real (and not guesswork)

If you can't answer these after Section 1, don't start "random fixes". You need more signal (metrics, kube-system logs, AWS-side telemetry) before touching production.

---
## 2. EKS Tier-0 components (what must stay healthy)

EKS gives you a managed control plane, but your workloads still depend on a small set of platform-critical components. If any of these degrade, the cluster looks partially alive while production is effectively down.

Below: what they do, how they fail, what you'll see, and what to check first.

---

### 2.1 EKS Control Plane (Managed, but not magic)

**What it includes**

* Kubernetes API server
* etcd (managed)
* controller-manager (managed)
* scheduler (managed)

**Common symptoms when control plane is unhealthy**

* `kubectl` commands are slow / timing out
* deployments take forever to apply
* controllers lag (HPA doesn't scale, pods don't reschedule)
* random "context deadline exceeded" errors

**First checks**

```bash
kubectl get --raw='/readyz?verbose'
kubectl get --raw='/livez?verbose'
kubectl get events -A --sort-by=.lastTimestamp | tail -n 50
```

**Pragmatic note**
Even if the control plane is "managed", you still need:

* sane API usage (avoid thundering herds from controllers/tools)
* sane webhook behaviour (one broken webhook can block deployments)

---

### 2.2 CoreDNS (Cluster DNS)

**Why it's Tier-0**
If DNS breaks, your apps don't "partially degrade". They fail in confusing ways:

* timeouts
* connection errors
* retries that amplify load

**Symptoms**

* services can't resolve (`*.svc.cluster.local`)
* random failures between pods
* sudden spike in upstream request errors

**First checks**

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=200
kubectl -n kube-system describe deploy coredns
```

**Common root causes**

* CoreDNS under-provisioned (CPU/mem)
* upstream resolver issues
* network problems to kube-dns service IP
* node-local DNS/cache behaviour (if enabled)

---

### 2.3 AWS VPC CNI (`aws-node`) — Networking Foundation

**Why it's Tier-0**
This is what gives pods IPs and makes pod networking real.
If this is unhealthy, pods won't start or won't communicate reliably.

**Symptoms**

* pods stuck in `ContainerCreating`
* `FailedCreatePodSandBox`
* sudden surge of Pending pods due to no IPs
* node-specific network failures

**First checks**

```bash
kubectl -n kube-system get pods -l k8s-app=aws-node -o wide
kubectl -n kube-system logs -l k8s-app=aws-node --tail=200
kubectl -n kube-system describe ds aws-node
```

**Typical EKS causes**

* subnet IP exhaustion
* ENI limits / pod density limits
* prefix delegation mis-sizing
* conntrack pressure (often shows up as "network flaky")

---

### 2.4 kube-proxy (Service Routing)

**Why it's Tier-0**
Even if pods are healthy, service routing can break:

* ClusterIP routing issues
* weird partial connectivity problems

**First checks**

```bash
kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide
kubectl -n kube-system logs -l k8s-app=kube-proxy --tail=200
```

**Pragmatic note**
If you're using an eBPF dataplane instead of kube-proxy, document it explicitly.
Debug steps change.

---

### 2.5 AWS Load Balancer Controller (ALB/NLB Integration)

**Why it's Tier-0**
This is the bridge between Kubernetes ingress/service objects and AWS load balancers.
If it breaks, external traffic fails even if the cluster is healthy.

**Symptoms**

* Ingress doesn't provision
* Target groups empty / unhealthy
* external traffic 4xx/5xx
* "it works inside the cluster but not from the internet"

**First checks**

```bash
kubectl -n kube-system get deploy | grep -i load-balancer
kubectl -n kube-system logs deploy/aws-load-balancer-controller --tail=200
kubectl get ingress -A
kubectl describe ingress -n <ns> <name>
```

**Common causes**

* IAM permissions issues
* security group rules wrong
* subnet tagging wrong
* controller version mismatch during upgrades

#### 2.5.1 NLB Idle Timeout + Keep-Alive (Silent Connection Kill)

When you use an **AWS Network Load Balancer (NLB)**, remember this:

* NLB is **L4**, not L7.
* It does **not** create a separate application-layer connection like an ALB does.
* It tracks TCP/UDP flows internally so it can route packets correctly.
* If a connection is **idle for ~350 seconds**, NLB will forget it.
* After that, if the client or server tries to send more data on that "old" connection, the NLB can respond with a **TCP RST**.

**What it looks like**

* random `connection reset by peer`
* intermittent failures for long-lived but mostly-idle connections
* higher failure rates on low-traffic tenants or long polling style traffic
* hard-to-reproduce "only happens sometimes" reports

**Why it happens**
Your application (or client) assumes the TCP connection is still valid because it was never explicitly closed.
But the NLB has expired the idle flow state, so the next packet hits a dead path and gets reset.

**Fix**
Enable **TCP keep-alives** so the connection never goes idle long enough to be forgotten.

You generally need:

* keep-alives enabled on the **server listener socket**
* keep-alives enabled on the **client side** too (if you control it)

**Pragmatic guidance**

* If you're running connection-heavy services behind NLB (gRPC, streaming, long-lived HTTP/1.1 keep-alive, custom TCP protocols), treat keep-alive tuning as a **production requirement**, not an optimization.
* If you can't control the client, you may need to:
  * reduce server-side idle timeouts
  * implement app-level heartbeats
  * or prefer ALB where L7 behaviour is needed

#### 2.5.2 NAT Idle Timeout + Keep-Alive (Egress Connection Resets)

When workloads in EKS talk to services on the public internet, traffic often goes through a **NAT device** (commonly AWS NAT Gateway). NAT devices track connection state, and they will **forget idle connections** after a timeout.

For example, **AWS NAT Gateway has an idle timeout of ~350 seconds**. After that, the NAT forgets the flow. If the client or server tries to send traffic on that old connection, it can result in a **TCP RST**.

**What it looks like**

* random outbound `connection reset by peer`
* flaky third-party API calls (only under low traffic / idle periods)
* long-lived connections that "randomly die"
* retries sometimes help, sometimes amplify load

**Why it happens**
Your application thinks it still has a valid TCP connection.
The NAT has expired the mapping due to idleness.
The next packet is treated as invalid state and gets reset.

**Fix: enable TCP keep-alives**

The simplest fix is to ensure connections don't remain idle long enough to be forgotten.

**Options**

1. **Enable TCP keep-alives in the application / proxy**
   * If your proxy supports keepalive tuning, configure it.
   * Example: Envoy has TCP keepalive configuration support.

2. **If the application/proxy doesn't support it**
   * Consider enabling it transparently using `LD_PRELOAD`
   * Tools like `libsetsockopt` can be used to apply `setsockopt()` defaults without changing application code

**Pragmatic guidance**

* Prefer fixing this at the proxy layer (Envoy / Nginx / HAProxy) where possible.
* If you use `LD_PRELOAD`, treat it as an engineering workaround:
  * document it
  * test it under load
  * make it part of your base image / runtime standard
  * expect debugging complexity later

**Monitor it (otherwise you'll rediscover it during incidents)**

If you use AWS NAT Gateway, monitor the metric:

* **IdleTimeoutCount**

A rising IdleTimeoutCount is a strong indicator of:

* too many idle-but-long-lived connections
* missing keepalive settings
* workload patterns that need pooling / reuse / heartbeat

---

### 2.6 EBS CSI Driver (Stateful Workloads Depend on It)

**Why it's Tier-0**
If you run StatefulSets with EBS, this is not optional.

**Symptoms**

* pods stuck in `ContainerCreating`
* volume attach/detach timeouts
* `FailedMount` events
* reschedules fail across AZs

**First checks**

```bash
kubectl -n kube-system get pods | grep ebs
kubectl -n kube-system logs deploy/ebs-csi-controller --tail=200
kubectl get pvc -A
kubectl describe pod -n <ns> <pod>
```

---

### 2.7 Metrics Server (Scaling + Visibility)

**Why it matters**
Not Tier-0 for *serving traffic*, but Tier-0 for *operating sanely*.
Without it:

* HPA breaks
* `kubectl top` is useless
* you lose quick visibility into node/pod pressure

**First checks**

```bash
kubectl -n kube-system get deploy metrics-server
kubectl -n kube-system logs deploy/metrics-server --tail=200
kubectl top nodes
kubectl top pods -A | head
```

---

### 2.8 EKS Addon Compatibility (Silent Failure Generator)

EKS upgrades are rarely just "upgrade Kubernetes". You're upgrading an ecosystem:

* CoreDNS
* kube-proxy
* VPC CNI
* CSI drivers
* LB controller

**Rule**
If you don't track addon versions and compatibility, you will eventually debug a failure caused by version skew.

---

### 2.9 AWS VPC / EC2 Network Limits (The Invisible Ceiling)

In AWS, network performance is not "infinite until it breaks". It is governed by a set of **hard limits** at the VPC/EC2 layer. Some of these limits are documented, many are not. When you hit them, AWS usually does not fail loudly — it fails as **latency, timeouts, dropped packets, and random connection resets**.

In EKS, this becomes easier to trigger because Kubernetes **binpacks many workloads onto one EC2 instance**, which means:

* one noisy pod can consume a node-level network limit
* every other pod on the node suffers
* symptoms look like "the app is slow" even though the app is fine

This section exists so we stop blaming applications for AWS network ceilings.

#### 2.9.1 Why this matters more in EKS (binpacking amplifies limits)

In a VM-per-service world, a single service hitting a network limit impacts itself.
In Kubernetes, a single node can run:

* dozens/hundreds of pods
* shared proxies (Envoy/Nginx)
* CoreDNS
* DaemonSets

Everything shares the same node-level network limits.

The result is predictable:

> A single workload can push the node over a limit and make unrelated services time out.

#### 2.9.2 Real-world failure mode: DNS lookups clustering onto CoreDNS nodes

By default, pods resolve DNS through CoreDNS:

**Traffic path**

```
application pod
  -> kube-dns service (CoreDNS pods)
    -> EC2 link-local resolver
```

Only a small number of CoreDNS pods typically run in a cluster, which means:

* DNS traffic concentrates onto a few nodes (where CoreDNS pods are scheduled)
* those EC2 instances can hit **link-local limits**
* the cluster sees "DNS is flaky" even though most nodes are fine

This is one of the easiest ways to create a cluster-wide incident with no obvious "broken component".

#### 2.9.3 Link-local traffic limits (169.254.x.x) — easy to hit, hard to debug

Each EC2 instance exposes local services via link-local addresses (example: `169.254.169.254`). These are used for:

* instance metadata
* temporary IAM credentials
* time sync and other local services
* DNS resolution via the VPC resolver path

These endpoints have limits. If you breach them, traffic gets rate-limited or dropped, and the failures are messy:

* timeouts
* slow DNS
* slow credential refresh
* sporadic errors that look unrelated

**Pragmatic rule**
Treat link-local as a **shared, rate-limited dependency**.

#### 2.9.4 DNS query limits (VPC resolver) — the 1024 packets/sec trap

Each EC2 instance has a hard cap on DNS traffic to the VPC resolver. As of this writing, it's effectively capped at:

* **1024 packets per second** *(packets, not queries)*

This distinction matters:

* A "DNS query" is not always one packet.
* With UDP, you usually pay at least:
  * 1 packet request + 1 packet response → **2 packets per query**
* That means you might only get **~512 queries/sec** in the simplest case.
* With larger responses, retries, TCP fallback, or DNSSEC, it gets worse.

In EKS, hundreds of pods on a node share this limit. It's trivial to breach it.

**What it looks like**

* intermittent DNS resolution timeouts
* cascading app failures (everything depends on DNS)
* retries amplify the packet rate and make it worse

**Mitigation**

* cache DNS aggressively (application-level where possible)
* consider node-local DNS caching (NodeLocal DNSCache / dnsmasq style)
* keep CoreDNS well distributed across nodes/AZs

#### 2.9.5 Security Group connection tracking limits (stateful firewall limits)

Security Groups are stateful. That means connection tracking happens, and there is a finite limit to how many concurrent connections an instance can sustain.

Important details:

* limits vary by instance type
* some limits are not clearly documented
* when you hit them, symptoms look like:
  * new TCP connections fail
  * connection establishment stalls
  * timeouts and latency spikes

**Where this hurts in EKS**

* reverse proxies handling lots of traffic
* long-lived connections (websocket, streaming, gRPC)
* high churn connection patterns (bad client behaviour, retries, load tests)

**Pragmatic guidance**

* long-lived connections are fine, but you must design for them:
  * keepalive tuning
  * connection pooling
  * horizontal scaling
  * avoid concentrating all traffic on a single node

#### 2.9.6 What to monitor (and alert on)

CloudWatch is not enough for this class of failures.

To monitor these limits properly, you need **node-level network driver metrics**.

AWS ENA exposes useful counters on each EC2 instance that are:

* not always available in CloudWatch by default
* best collected by scraping from the node and shipping to your monitoring system

**Action item**
Run a node-level metrics collector (DaemonSet or host agent) that scrapes ENA-related counters and publishes them to Prometheus / your metrics pipeline.

---

### 2.10 What to do when a Tier-0 component is unhealthy (EKS Incident Playbook)

Tier-0 failures are not limited to `kube-system`. In EKS, Tier-0 also includes AWS networking primitives that your cluster depends on:

* **NLB / ALB** (ingress and L4/L7 connectivity)
* **NAT Gateway** (egress to internet / third-party dependencies)
* **Security Group connection tracking** (stateful connection ceilings)

When any of these degrade, application symptoms become misleading. Your job is to stabilize the platform and the AWS networking layer first.

#### 2.10.1 Step 0 — Stop making the incident worse

**Do NOT**

* restart random deployments
* roll out unrelated changes
* run scripts that spam the Kubernetes API
* scale CoreDNS / proxies blindly without checking node headroom

**Do**

* pause ongoing rollouts if they're increasing churn
* capture evidence before it disappears

**Capture evidence (minimum)**

```bash
kubectl get nodes -o wide > nodes.txt
kubectl get pods -A -o wide > pods.txt
kubectl get events -A --sort-by=.lastTimestamp | tail -n 200 > events.txt
```

#### 2.10.2 Step 1 — Confirm blast radius (cluster vs node-group vs AZ vs edge)

Tier-0 failures often appear "global" but aren't. Quickly classify:

* **Cluster-wide**: kube-system pods unhealthy across nodes
* **Node-group/AZ-specific**: only one pool or AZ shows issues
* **Edge-only**: internal works, external traffic fails (NLB/ALB)
* **Egress-only**: internal works, outbound calls fail (NAT)

**Quick checks**

```bash
kubectl get nodes
kubectl get pods -n kube-system -o wide
kubectl get pods -A -o wide | head -n 50
```

#### 2.10.3 Step 2 — Identify the failing Tier-0 dependency (K8s + AWS)

**A) CoreDNS unhealthy → DNS failures everywhere**

**Checks**

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=200
```

**Safe actions**

* scale CoreDNS only if nodes have headroom
* spread CoreDNS pods (avoid hotspot nodes)
* reduce DNS pressure (cache / fix runaway clients)

**B) VPC CNI (`aws-node`) unhealthy → pod networking breaks**

**Checks**

```bash
kubectl -n kube-system get pods -l k8s-app=aws-node -o wide
kubectl -n kube-system logs -l k8s-app=aws-node --tail=200
```

**Safe actions**

* cordon impacted nodes
* scale out node groups if out of IPs / pod density
* verify subnet IP capacity / prefix delegation

**C) NAT idle timeout / keepalive mismatch → random outbound resets (egress path)**

Same 350s idle timeout as NLB, but on the egress path. See [Section 2.5.2](#252-nat-idle-timeout-keep-alive-egress-connection-resets) for full detail.

**Safe actions**

* enable TCP keep-alives in app/proxy
* monitor NAT `IdleTimeoutCount`

#### 2.10.4 Step 3 — Contain blast radius (prevent spread)

Once you know the failing Tier-0 dependency:

**Containment options**

* cordon nodes to stop new scheduling:

```bash
kubectl cordon <node>
```

* drain nodes only when safe:

```bash
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

* scale out node groups if the failure is capacity-related (IP/CPU/mem/conntrack)

**Key principle**
Containment > churn. Avoid actions that increase retries and reconnections.

#### 2.10.5 Step 4 — Restore Tier-0 health (stabilize, then recover)

Stabilize in this order:

1. Control plane responsiveness (kubectl/API sanity)
2. kube-system essentials (CoreDNS, aws-node, kube-proxy)
3. Storage controllers (EBS CSI if stateful workloads exist)
4. Ingress/egress path (NLB/NAT behaviours, keepalives, target health)
5. Workloads

Only after Tier-0 is stable should you restart/roll workloads.

#### 2.10.6 Step 5 — Validate recovery (don't declare victory early)

Recovery means:

* DNS stable
* new pods schedule and start
* ingress traffic healthy
* egress stable (no NAT idle reset spikes)
* error rate + tail latency back to baseline

**Quick checks**

```bash
kubectl get nodes
kubectl -n kube-system get pods -o wide
kubectl get pods -A --field-selector=status.phase!=Running | head
```

#### 2.10.7 Post-incident hardening (mandatory follow-up)

Every Tier-0 incident must produce:

* alerts (CoreDNS, aws-node, EBS CSI, LB controller)
* AWS networking alerts (NAT `IdleTimeoutCount`, node `conntrack_allowance_exceeded`)
* guardrails:
  * CoreDNS spread constraints
  * DNS caching strategy
  * keepalive defaults for ingress/egress
  * instance type sizing for connection-heavy services

---
## 3. Networking

This is where most "mysterious" production failures actually live. Unlike compute or storage failures that fail loudly, networking degrades gradually and inconsistently. A connection works 95% of the time. DNS resolves "most of the time". Latency spikes "only under load".

Below: the EKS networking stack and how to debug it systematically when things go sideways.

---

### 3.1 EKS Networking Stack (What Can Break and Where)

Understanding the layers helps you debug faster:

```
[Pod A] 
  ↓ (veth pair)
[Node's root netns] 
  ↓ (AWS VPC CNI / ENI)
[AWS VPC] 
  ↓ (routing, security groups, NACLs)
[Target: Pod B / Service / Internet]
```

**Each layer can fail differently:**

* **Pod network namespace**: wrong routes, missing interfaces
* **Node networking**: CNI plugin issues, IP exhaustion, conntrack
* **VPC layer**: security groups, routing tables, subnet capacity
* **AWS services**: NLB/ALB behavior, NAT timeouts, DNS resolver limits

---

### 3.2 AWS VPC CNI Deep Dive (The Foundation)

The AWS VPC CNI is what makes "pod gets a real VPC IP" work. When it breaks, symptoms are confusing because pods might start but not communicate, or communication works sometimes but not others.

#### 3.2.1 How AWS VPC CNI Works (Simplified)

1. **ENI allocation**: Each node gets multiple ENIs (network interfaces)
2. **IP allocation**: Each ENI gets either:
   - Multiple secondary IPs (legacy mode)
   - IP prefixes (/28 blocks) when prefix delegation is enabled
3. **Pod assignment**: Each pod gets one IP from the available pool
4. **Routing**: Node routes traffic between pod netns and ENI

Pod density isn't just CPU/memory limited — it's bounded by ENI limits and either IP-per-ENI limits (legacy) or prefix allocation limits (with prefix delegation).

**Prefix delegation benefits:**
- Dramatically increases pod density (from ~10-250 pods per node to ~110-750+ pods)
- Reduces ENI pressure on larger instance types
- More efficient IP utilization

#### 3.2.2 Common CNI Failure Modes

**A) IP Exhaustion (Pods stuck in Pending)**

**Symptoms:**
```bash
kubectl get pods -A | grep Pending
kubectl describe pod <pod> | grep -i "failed to allocate"
```

**Root causes:**
* Subnet out of IPs
* Node hit max pods per instance type
* Prefix delegation misconfigured or not enabled
* ENI limits reached without prefix delegation

**Quick diagnosis:**
```bash
# Check available IPs in subnet
aws ec2 describe-subnets --subnet-ids <subnet-id>

# Check pod density limits
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.pods}{"\n"}{end}'

# Check CNI logs
kubectl -n kube-system logs -l k8s-app=aws-node --tail=100
```

**B) ENI Attachment Failures**

**Symptoms:**
* New nodes can't schedule pods
* `FailedCreatePodSandBox` errors
* CNI timeouts during pod creation

**Quick diagnosis:**
```bash
kubectl -n kube-system logs -l k8s-app=aws-node | grep -i "eni\|attach\|interface"
```

**C) Cross-AZ Communication Issues**

**Symptoms:**
* Pods in different AZs can't reach each other
* Intermittent timeouts between services
* Works within AZ, fails across AZ

**Root causes:**
* Route table misconfigurations
* Security group rules
* NACLs blocking cross-AZ traffic

#### 3.2.3 CNI Configuration Tuning

**Key parameters to understand:**

```bash
# Check current CNI configuration
kubectl -n kube-system describe daemonset aws-node
```

**Important environment variables:**
* `ENABLE_PREFIX_DELEGATION`: Increases pod density
* `WARM_ENI_TARGET`: Pre-allocates ENIs for faster pod startup
* `WARM_IP_TARGET`: Pre-allocates IPs for faster pod startup
* `MAX_ENI`: Limits ENI usage per node

**Production tuning example:**
```yaml
env:
- name: ENABLE_PREFIX_DELEGATION
  value: "true"
- name: WARM_PREFIX_TARGET
  value: "1"        # Keep 1 prefix warm (16 IPs)
- name: WARM_IP_TARGET
  value: "3"        # Keep 3 individual IPs warm
- name: MAX_ENI
  value: "10"       # Limit ENI usage if needed
- name: AWS_VPC_K8S_CNI_EXTERNALSNAT
  value: "true"     # Preserve pod IPs for cross-VPC communication
```

**Understanding prefix vs IP mode:**
- **Legacy (IP mode)**: Each ENI gets ~15-50 secondary IPs depending on instance type
- **Prefix mode**: Each ENI gets /28 prefixes (16 IPs each), dramatically increasing density
- **Mixed mode**: Can use both prefixes and individual IPs on same ENI

**External SNAT configuration:**
- **Default (false)**: Pod traffic to external destinations gets SNATed to node IP
- **External SNAT (true)**: Pod retains its VPC IP when talking to external destinations
- **Critical for**: Cross-VPC communication, VPC peering, Transit Gateway scenarios
- **Why it matters**: Allows destination to see actual pod IP instead of node IP for logging, security groups, etc.

---

### 3.3 DNS and Service Discovery (CoreDNS Operational Reality)

DNS failures in Kubernetes don't just break service discovery—they cascade into timeouts, retries, and connection pool exhaustion that can take down entire applications.

#### 3.3.1 CoreDNS Under Load (When DNS Becomes the Bottleneck)

**Common failure pattern:**
1. Application makes many DNS queries (poor caching)
2. CoreDNS pods hit CPU/memory limits
3. DNS queries start timing out
4. Applications retry aggressively
5. DNS load increases, making timeouts worse
6. Cascade failure across services

**Symptoms:**
```bash
# DNS timeouts in application logs
kubectl logs <app-pod> | grep -i "dns\|resolve\|timeout"

# CoreDNS resource pressure
kubectl -n kube-system top pods -l k8s-app=kube-dns

# DNS query patterns
kubectl -n kube-system logs -l k8s-app=kube-dns | grep -E "NXDOMAIN|timeout|error"
```

#### 3.3.2 DNS Query Patterns That Kill Performance

**Bad patterns:**
* No DNS caching in applications
* Querying external domains from every pod
* Short TTL on frequently accessed services
* DNS queries in tight loops

**Example of problematic application behavior:**
```python
# BAD: DNS lookup on every request
def make_request():
    host = socket.gethostbyname("api.external.com")  # DNS lookup every time
    return requests.get(f"http://{host}/api")

# GOOD: Cache DNS resolution
dns_cache = {}
def make_request():
    if "api.external.com" not in dns_cache:
        dns_cache["api.external.com"] = socket.gethostbyname("api.external.com")
    host = dns_cache["api.external.com"]
    return requests.get(f"http://{host}/api")
```

#### 3.3.3 CoreDNS Scaling and Distribution

**Horizontal scaling:**
```bash
kubectl -n kube-system scale deployment coredns --replicas=5
```

**Anti-affinity to spread CoreDNS pods:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
spec:
  template:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: k8s-app
                  operator: In
                  values: ["kube-dns"]
              topologyKey: kubernetes.io/hostname
```

#### 3.3.4 NodeLocal DNSCache (Advanced DNS Optimization)

For clusters with heavy DNS load, NodeLocal DNSCache runs a DNS cache on each node:

**Benefits:**
* Reduces load on CoreDNS
* Improves DNS response times
* Reduces DNS-related network traffic

**Trade-offs:**
* Additional complexity
* More moving parts to debug
* Cache invalidation edge cases

**When to consider:**
* High DNS query volume (>1000 QPS cluster-wide)
* DNS-related performance issues
* Applications that can't implement proper DNS caching

#### 3.3.5 DNS Search Domain Optimization (ndots Configuration)

**The ndots problem:** By default, Kubernetes sets `ndots:5` in `/etc/resolv.conf`, causing excessive DNS queries for external domains.

**Default behavior analysis:**
```bash
# Inside a pod, resolving "google.com" triggers these queries:
# 1. google.com.default.svc.cluster.local
# 2. google.com.svc.cluster.local  
# 3. google.com.cluster.local
# 4. google.com.us-west-2.compute.internal
# 5. google.com.compute.internal
# 6. google.com (finally!)
```

**Impact on AWS linklocal limits:**
```bash
# Each failed query hits 169.254.169.254 (AWS DNS resolver)
# With ndots:5, external domains generate 6x DNS traffic
# AWS limit: 1024 PPS per instance - easily exceeded in dense clusters
```

**Optimized ndots configuration:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: optimized-dns-app
spec:
  dnsPolicy: ClusterFirst
  dnsConfig:
    options:
    - name: ndots
      value: "1"  # Reduce from default 5 to 1
    - name: edns0  # Enable DNS extensions
  containers:
  - name: app
    image: nginx
```

**Application-specific DNS optimization:**
```yaml
# For apps that primarily call external services
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-api-client
spec:
  template:
    spec:
      dnsConfig:
        options:
        - name: ndots
          value: "1"
      containers:
      - name: app
        env:
        - name: EXTERNAL_API_URL
          value: "https://api.example.com."  # Trailing dot = absolute FQDN
```

### 3.3.6 Listen backlog and connection handling

These are not DNS topics, but connection-level tuning is commonly needed alongside DNS optimization when debugging service latency.

**Listen Backlog Configuration for High-Traffic Services**

**Problem:** Default listen backlog (128) causes connection drops under bursty load.

**Root cause:** When services receive more concurrent connection attempts than the listen backlog can queue, connections are dropped at the kernel level.

**Solution - Configure via sysctls:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: high-traffic-service
spec:
  template:
    spec:
      securityContext:
        sysctls:
        - name: net.core.somaxconn
          value: "32000"  # Increase from default 128
        - name: net.ipv4.ip_local_port_range
          value: "1024 64000"  # Expand ephemeral port range
      containers:
      - name: app
        # Application configuration
```

**Monitor listen backlog with sidecar pattern:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monitored-service
spec:
  template:
    spec:
      containers:
      - name: app
        image: my-app:latest
        # Main application container
      - name: node-exporter
        image: prom/node-exporter:latest
        args:
        - --web.listen-address=0.0.0.0:9100
        - --collector.disable-defaults
        - --web.disable-exporter-metrics
        - --collector.conntrack
        - --collector.filefd
        - --collector.netstat
        - --collector.sockstat
        ports:
        - containerPort: 9100
          name: metrics
        securityContext:
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["all"]
```

**Prometheus alerts for listen backlog issues:**
```yaml
groups:
- name: listen-backlog
  rules:
  - alert: ListenDrops
    expr: sum by (k8s_cluster_name, pod) (rate(node_netstat_TcpExt_ListenDrops[5m]) > 0) > 5
    for: 2m
    annotations:
      summary: "Listen drops detected on {{ $labels.pod }}"
      description: "Pod {{ $labels.pod }} is dropping connections due to listen backlog overflow"

  - alert: ListenOverflows  
    expr: sum by (k8s_cluster_name, pod) (rate(node_netstat_TcpExt_ListenOverflows[5m]) > 0) > 5
    for: 2m
    annotations:
      summary: "Listen overflows detected on {{ $labels.pod }}"
      description: "Pod {{ $labels.pod }} has listen queue overflows - increase somaxconn"
```

**Protection Against Slow Clients**

**Problem:** Slow clients can exhaust thread/process pools in request-per-thread models.

**Attack vector simulation:**
```bash
# Simulate slow client sending 10KB slowly (1 byte per second)
(echo -e -n 'POST /api HTTP/1.1\r\nHost: example.com\r\nContent-Length: 10000\r\n\r\n'; 
 i=0; while [ $i -lt 10000 ]; do echo -n "a"; sleep 1; i=$((i+1)); done) \
 | socat -t 10 - TCP4:service.example.com:80
```

**Solution - Reverse proxy with buffering:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    events {
        worker_connections 1024;
    }
    http {
        # Buffer entire request before forwarding to backend
        proxy_request_buffering on;
        proxy_buffering on;
        
        # Timeout configurations
        client_header_timeout 10s;    # Max time to receive headers
        client_body_timeout 30s;      # Max time to receive body
        send_timeout 30s;             # Max time to send response
        keepalive_timeout 65s;        # Connection idle timeout
        
        upstream backend {
            server app-service:8080;
        }
        
        server {
            listen 80;
            location / {
                proxy_pass http://backend;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
            }
        }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-proxy
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        volumeMounts:
        - name: config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
      volumes:
      - name: config
        configMap:
          name: nginx-config
```

**Envoy configuration for slow client protection:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-config
data:
  envoy.yaml: |
    static_resources:
      listeners:
      - name: listener_0
        address:
          socket_address:
            address: 0.0.0.0
            port_value: 8080
        filter_chains:
        - filters:
          - name: envoy.filters.network.http_connection_manager
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
              request_timeout: 30s
              stream_idle_timeout: 300s
              request_headers_timeout: 10s
              http_filters:
              - name: envoy.filters.http.buffer
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.filters.http.buffer.v3.Buffer
                  max_request_bytes: 1048576  # 1MB buffer
              - name: envoy.filters.http.router
              route_config:
                name: local_route
                virtual_hosts:
                - name: backend
                  domains: ["*"]
                  routes:
                  - match:
                      prefix: "/"
                    route:
                      cluster: backend_cluster
      clusters:
      - name: backend_cluster
        connect_timeout: 5s
        type: STRICT_DNS
        lb_policy: ROUND_ROBIN
        load_assignment:
          cluster_name: backend_cluster
          endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: app-service
                    port_value: 8080
```


---

### 3.4 Service Mesh Networking (When L7 Proxy Becomes Critical Path)

Service meshes add another networking layer that can fail in EKS-specific ways.

#### 3.4.1 Envoy Sidecar Resource Limits

**Common failure:**
Envoy sidecar hits CPU/memory limits under load, causing:
* Request timeouts
* Connection pool exhaustion
* Circuit breaker activation

**Diagnosis:**
```bash
# Check sidecar resource usage
kubectl top pods --containers | grep envoy

# Check Envoy admin interface
kubectl exec <pod> -c istio-proxy -- curl localhost:15000/stats | grep -E "cx_|rq_|upstream"
```

**Tuning:**
```yaml
metadata:
  annotations:
    sidecar.istio.io/proxyCPU: "100m"
    sidecar.istio.io/proxyMemory: "128Mi"
    sidecar.istio.io/proxyCPULimit: "200m"
    sidecar.istio.io/proxyMemoryLimit: "256Mi"
```

#### 3.4.2 mTLS Certificate Rotation Issues

**Symptoms:**
* Intermittent 503 errors between services
* TLS handshake failures
* Services work sometimes, fail other times

**Diagnosis:**
```bash
# Check certificate expiration
kubectl exec <pod> -c istio-proxy -- openssl s_client -connect <service>:443 -servername <service> < /dev/null 2>/dev/null | openssl x509 -noout -dates

# Check Envoy TLS stats
kubectl exec <pod> -c istio-proxy -- curl localhost:15000/stats | grep ssl
```

---

### 3.5 Load Balancer Integration (ALB/NLB Operational Patterns)

#### 3.5.1 ALB Target Group Health Issues

**Common failure pattern:**
1. Pod starts and becomes "Ready"
2. ALB target group shows "unhealthy"
3. Traffic doesn't reach the pod
4. Application appears to be "not working"

**Root causes:**
* Health check path misconfigured
* Security group rules blocking ALB health checks
* Pod readiness probe vs ALB health check mismatch

**Diagnosis:**
```bash
# Check ALB target group health
aws elbv2 describe-target-health --target-group-arn <arn>

# Check ingress configuration
kubectl describe ingress <ingress-name>

# Check ALB controller logs
kubectl -n kube-system logs deployment/aws-load-balancer-controller
```

**Fix patterns:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '30'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '3'
```

#### 3.5.2 NLB Connection Tracking and Keep-Alive

The NLB idle timeout problem and TCP keepalive fix are covered in detail in [Section 2.5.1](#251-nlb-idle-timeout-keep-alive-silent-connection-kill). This section adds the language-specific code examples.

**For HTTP clients (Python):**
```python
import requests
from requests.adapters import HTTPAdapter

session = requests.Session()
adapter = HTTPAdapter(
    socket_options=[
        (socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1),
        (socket.IPPROTO_TCP, socket.TCP_KEEPIDLE, 300),
        (socket.IPPROTO_TCP, socket.TCP_KEEPINTVL, 5),
        (socket.IPPROTO_TCP, socket.TCP_KEEPCNT, 5),
    ]
)
session.mount("http://", adapter)
session.mount("https://", adapter)
```

**For gRPC (Python):**
```python
import grpc

options = [
    ('grpc.keepalive_time_ms', 300000),
    ('grpc.keepalive_timeout_ms', 5000),
    ('grpc.keepalive_permit_without_calls', True),
    ('grpc.http2.max_pings_without_data', 0),
]

channel = grpc.insecure_channel('service:50051', options=options)
```

---

### 3.6 Network Policies (Micro-segmentation That Actually Works)

Network policies in EKS require a CNI that supports them (like Calico). When they're misconfigured, they create "works sometimes" failures that are hard to debug.

#### 3.6.1 Common Network Policy Mistakes

**Mistake 1: Blocking DNS**
```yaml
# BAD: This blocks DNS resolution
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  # No egress rules = no DNS
```

**Fix: Always allow DNS**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to: []
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

**Mistake 2: Forgetting about health checks**
```yaml
# Need to allow kubelet health checks
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-health-checks
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from: []
    ports:
    - protocol: TCP
      port: 8080  # Your health check port
```

#### 3.6.2 Debugging Network Policy Issues

**Test connectivity between pods:**
```bash
# From source pod to target pod
kubectl exec -it <source-pod> -- nc -zv <target-pod-ip> <port>

# Test DNS resolution
kubectl exec -it <pod> -- nslookup kubernetes.default.svc.cluster.local

# Check if network policies are applied
kubectl get networkpolicy -A
kubectl describe networkpolicy <policy-name>
```

**Calico-specific debugging:**
```bash
# Check Calico policy status
kubectl exec -n kube-system <calico-node-pod> -- calicoctl get policy -o wide

# Check Calico logs
kubectl -n kube-system logs -l k8s-app=calico-node
```

---

### 3.7 Cross-AZ Networking (Latency and Cost Optimization)

#### 3.7.1 Understanding Cross-AZ Traffic Patterns

**Network latency between AZs:**
* Intra-AZ: ~0.1-0.5ms
* Inter-AZ: ~1-2ms
* Cross-region: 20-100ms+

**Cost implications:**
* Intra-AZ traffic: Free
* Inter-AZ traffic: $0.01/GB (as of 2024)
* Cross-region: $0.02/GB+

#### 3.7.2 Topology Spread Constraints for Network Optimization

**Spread pods across AZs for availability:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 6
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: web-app
```

**Keep related services in same AZ:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cache-service
spec:
  template:
    spec:
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: ["web-app"]
              topologyKey: topology.kubernetes.io/zone
```

---

### 3.8 Debugging Network Issues (Systematic Approach)

#### 3.8.1 Layer-by-Layer Debugging

**Step 1: Pod-to-Pod IP connectivity**
```bash
# Get pod IPs
kubectl get pods -o wide

# Test basic IP connectivity
kubectl exec -it <source-pod> -- ping <target-pod-ip>

# Test specific port
kubectl exec -it <source-pod> -- nc -zv <target-pod-ip> <port>
```

**Step 2: Service discovery**
```bash
# Test DNS resolution
kubectl exec -it <pod> -- nslookup <service-name>.<namespace>.svc.cluster.local

# Test service connectivity
kubectl exec -it <pod> -- curl <service-name>.<namespace>.svc.cluster.local:<port>
```

**Step 3: Ingress/Load balancer**
```bash
# Check ingress status
kubectl get ingress
kubectl describe ingress <ingress-name>

# Test from outside cluster
curl -v http://<load-balancer-dns>/health
```

#### 3.8.2 Network Debugging Tools

**Essential tools to have in debug pods:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: network-debug
spec:
  containers:
  - name: debug
    image: nicolaka/netshoot
    command: ["/bin/bash"]
    args: ["-c", "while true; do sleep 30; done;"]
    securityContext:
      capabilities:
        add: ["NET_ADMIN"]
```

**Useful commands in debug pod:**
```bash
# Network interface info
ip addr show
ip route show

# DNS debugging
dig @8.8.8.8 google.com
nslookup kubernetes.default.svc.cluster.local

# Port scanning
nmap -p 80,443,8080 <target-ip>

# Packet capture
tcpdump -i any -w /tmp/capture.pcap host <target-ip>

# Connection testing
nc -zv <host> <port>
telnet <host> <port>
```

#### 3.8.3 Performance Testing and Monitoring

**Network performance testing:**
```bash
# Bandwidth testing between pods
kubectl exec -it <pod1> -- iperf3 -s &
kubectl exec -it <pod2> -- iperf3 -c <pod1-ip>

# Latency testing
kubectl exec -it <pod1> -- ping -c 100 <pod2-ip>
```

**Key metrics to monitor:**
* DNS query latency and error rate
* Service-to-service latency (P50, P95, P99)
* Network throughput and packet loss
* Connection pool utilization
* Cross-AZ traffic volume and cost

---

### 3.9 Network Security (Defense in Depth)

#### 3.9.1 Security Groups vs Network Policies

**Security Groups (AWS level):**
* Applied at ENI level
* Stateful firewall rules
* Good for node-to-node and external access control

**Network Policies (Kubernetes level):**
* Applied at pod level
* More granular control
* Good for micro-segmentation within cluster

**Best practice: Use both layers**
```yaml
# Network policy for pod-to-pod
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-app-policy
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

#### 3.9.2 Pod Security Groups (EKS-specific)

For fine-grained security group control at pod level:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  annotations:
    eks.amazonaws.com/security-groups: sg-12345678,sg-87654321
spec:
  containers:
  - name: app
    image: myapp:latest
```

**When to use pod security groups:**
* Need different security rules per workload
* Compliance requirements for network isolation
* Integration with AWS security tools

**Trade-offs:**
* Additional complexity
* Potential performance impact
* Limited to specific instance types and CNI versions

---

### 3.10 Network Troubleshooting Runbook

#### 3.10.1 "Service is unreachable" Runbook

**Symptoms:** Application can't reach another service

**Step 1: Verify service exists and has endpoints**
```bash
kubectl get svc <service-name>
kubectl get endpoints <service-name>
```

**Step 2: Test DNS resolution**
```bash
kubectl exec -it <pod> -- nslookup <service-name>.<namespace>.svc.cluster.local
```

**Step 3: Test direct IP connectivity**
```bash
kubectl exec -it <pod> -- nc -zv <endpoint-ip> <port>
```

**Step 4: Check network policies**
```bash
kubectl get networkpolicy -n <namespace>
kubectl describe networkpolicy <policy-name>
```

**Step 5: Check security groups (if using pod security groups)**
```bash
aws ec2 describe-security-groups --group-ids <sg-id>
```

#### 3.10.2 "DNS is slow/failing" Runbook

**Symptoms:** DNS timeouts, slow service discovery

**Step 1: Check CoreDNS health**
```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=100
```

**Step 2: Test DNS from multiple pods**
```bash
kubectl exec -it <pod1> -- time nslookup kubernetes.default.svc.cluster.local
kubectl exec -it <pod2> -- time nslookup kubernetes.default.svc.cluster.local
```

**Step 3: Check DNS query patterns**
```bash
kubectl -n kube-system logs -l k8s-app=kube-dns | grep -E "NXDOMAIN|timeout" | tail -20
```

**Step 4: Monitor CoreDNS resource usage**
```bash
kubectl -n kube-system top pods -l k8s-app=kube-dns
```

**Step 5: Scale CoreDNS if needed**
```bash
kubectl -n kube-system scale deployment coredns --replicas=<new-count>
```

#### 3.10.3 "Load balancer not working" Runbook

**Symptoms:** External traffic can't reach services

**Step 1: Check ingress/service status**
```bash
kubectl get ingress
kubectl describe ingress <ingress-name>
kubectl get svc <service-name>
```

**Step 2: Check AWS Load Balancer Controller**
```bash
kubectl -n kube-system logs deployment/aws-load-balancer-controller
```

**Step 3: Verify target group health**
```bash
aws elbv2 describe-target-health --target-group-arn <arn>
```

**Step 4: Test internal connectivity**
```bash
kubectl exec -it <debug-pod> -- curl <service-name>:<port>/health
```

**Step 5: Check security group rules**
```bash
aws ec2 describe-security-groups --group-ids <alb-sg-id>
```

---

## 4. Workload identity and security

EKS security failures often look like "the application is broken" when the real issue is auth, authorization, or secrets. Misconfigured IRSA, missing encryption, bad RBAC — these create confusing incidents that send you chasing application bugs that don't exist.

---

### 4.1 IAM Roles for Service Accounts (IRSA) - The Foundation

IRSA is how pods get AWS permissions without embedding long-lived credentials. When it breaks, applications fail to access AWS services with cryptic permission errors.

#### 4.1.1 How IRSA Works (What Can Break)

```
[Pod with ServiceAccount] 
  ↓ (projected token volume)
[OIDC JWT Token] 
  ↓ (AWS STS AssumeRoleWithWebIdentity)
[Temporary AWS Credentials] 
  ↓ (AWS API calls)
[AWS Services: S3, RDS, etc.]
```

**Each step can fail:**
* ServiceAccount annotation missing/wrong
* OIDC provider not configured
* IAM role trust policy incorrect
* IAM role permissions insufficient
* Token projection/mounting issues

#### 4.1.2 Common IRSA Failure Patterns

**A) "Access Denied" but IAM role looks correct**

**Symptoms:**
```
AccessDenied: User: arn:aws:sts::123456789012:assumed-role/eksctl-my-cluster-nodegroup-NodeInstanceRole-XXXXX/i-1234567890abcdef0 is not authorized to perform: s3:GetObject
```

**Root cause:** Pod is using node IAM role instead of IRSA role

**Diagnosis:**
```bash
# Check if ServiceAccount has IRSA annotation
kubectl describe sa <service-account-name>

# Check if pod is using the ServiceAccount
kubectl describe pod <pod-name> | grep "Service Account"

# Check if OIDC provider exists
aws eks describe-cluster --name <cluster-name> --query "cluster.identity.oidc.issuer"
aws iam list-open-id-connect-providers
```

**B) Token projection failures**

**Symptoms:**
* Pod starts but AWS calls fail with authentication errors
* Missing `/var/run/secrets/eks.amazonaws.com/serviceaccount/token`

**Diagnosis:**
```bash
# Check if token is mounted
kubectl exec <pod-name> -- ls -la /var/run/secrets/eks.amazonaws.com/serviceaccount/

# Check token content (should be JWT)
kubectl exec <pod-name> -- cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token | cut -d. -f2 | base64 -d
```

#### 4.1.3 IRSA Setup and Troubleshooting

**Correct IRSA setup:**

1. **Create OIDC provider (one-time per cluster):**
```bash
eksctl utils associate-iam-oidc-provider --cluster <cluster-name> --approve
```

2. **Create IAM role with trust policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT-ID:oidc-provider/oidc.eks.REGION.amazonaws.com/id/OIDC-ID"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.REGION.amazonaws.com/id/OIDC-ID:sub": "system:serviceaccount:NAMESPACE:SERVICE-ACCOUNT-NAME",
          "oidc.eks.REGION.amazonaws.com/id/OIDC-ID:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

3. **Annotate ServiceAccount:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: my-namespace
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT-ID:role/my-irsa-role
```

4. **Use ServiceAccount in pod:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: my-service-account
  containers:
  - name: my-container
    image: my-app:latest
```

**Validation script:**
```bash
#!/bin/bash
# Test IRSA setup
NAMESPACE="my-namespace"
SA_NAME="my-service-account"
POD_NAME="test-pod"

echo "1. Checking ServiceAccount annotation..."
kubectl get sa $SA_NAME -n $NAMESPACE -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}'

echo -e "\n2. Checking pod ServiceAccount..."
kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.spec.serviceAccountName}'

echo -e "\n3. Checking token mount..."
kubectl exec $POD_NAME -n $NAMESPACE -- ls -la /var/run/secrets/eks.amazonaws.com/serviceaccount/

echo -e "\n4. Testing AWS credentials..."
kubectl exec $POD_NAME -n $NAMESPACE -- aws sts get-caller-identity
```

---

### 4.2 Pod Security Standards (Replacing PSPs)

Pod Security Policies (PSPs) are deprecated. Pod Security Standards are the replacement, but they work differently and can create new failure modes.

#### 4.2.1 Pod Security Standards Levels

**Privileged:** No restrictions (dangerous for production)
**Baseline:** Minimal restrictions, prevents known privilege escalations
**Restricted:** Heavily restricted, follows pod hardening best practices

#### 4.2.2 Common Pod Security Failures

**A) Pods rejected by admission controller**

**Symptoms:**
```
Error creating: pods "my-pod" is forbidden: violates PodSecurity "restricted:latest": 
allowPrivilegeEscalation != false, unrestricted capabilities, runAsNonRoot != true
```

**Fix patterns:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
```

**B) Applications fail due to security restrictions**

**Common issues:**
* App tries to write to read-only filesystem
* App needs specific capabilities
* App runs as root by default

**Debugging approach:**
```bash
# Check pod security context
kubectl describe pod <pod-name> | grep -A 20 "Security Context"

# Check container security context
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].securityContext}'

# Test file system permissions
kubectl exec <pod-name> -- touch /tmp/test-write
kubectl exec <pod-name> -- id
```

#### 4.2.3 Namespace-Level Pod Security Configuration

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Gradual rollout strategy:**
1. Start with `warn` mode to identify violations
2. Add `audit` mode to log violations
3. Finally enable `enforce` mode to block violations

---

### 4.3 Secrets Management (Beyond Kubernetes Secrets)

Kubernetes Secrets are base64 encoded, not encrypted at rest by default, and visible to anyone with cluster access. For production workloads, you need better secrets management.

#### 4.3.1 AWS Secrets Manager Integration

**Using AWS Load Balancer Controller with Secrets Manager:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  annotations:
    aws-load-balancer-controller.k8s.aws/secret-manager: "arn:aws:secretsmanager:region:account:secret:prod/db/credentials"
type: Opaque
```

**Using External Secrets Operator:**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-west-2
      auth:
        secretRef:
          accessKeyID:
            name: awssm-secret
            key: access-key
          secretAccessKey:
            name: awssm-secret
            key: secret-access-key
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secret
spec:
  refreshInterval: 15s
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: app-secret
    creationPolicy: Owner
  data:
  - secretKey: password
    remoteRef:
      key: prod/db/credentials
      property: password
```

#### 4.3.2 Secrets CSI Driver

**Mount secrets as volumes:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  serviceAccountName: app-service-account
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: secrets-store
      mountPath: "/mnt/secrets"
      readOnly: true
  volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: "app-secrets"
---
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: app-secrets
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "prod/db/credentials"
        objectType: "secretsmanager"
        jmesPath:
          - path: "password"
            objectAlias: "db-password"
```

#### 4.3.3 Secrets Rotation and Lifecycle

**Automatic rotation with External Secrets:**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: rotating-secret
spec:
  refreshInterval: 1h  # Check for updates every hour
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: app-secret
    creationPolicy: Owner
    template:
      metadata:
        annotations:
          reloader.stakater.com/match: "true"  # Trigger pod restart on change
```

**Monitoring secrets rotation:**
```bash
# Check External Secrets status
kubectl get externalsecrets
kubectl describe externalsecret <name>

# Check secret age
kubectl get secrets -o custom-columns=NAME:.metadata.name,AGE:.metadata.creationTimestamp
```

---

### 4.4 Network Security (Security Groups and Network Policies)

#### 4.4.1 Security Groups for Pods

EKS allows assigning security groups directly to pods for fine-grained network control:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  annotations:
    eks.amazonaws.com/security-groups: sg-12345678
spec:
  containers:
  - name: app
    image: myapp:latest
```

**When to use pod security groups:**
* Need different network rules per workload
* Compliance requirements for network isolation
* Integration with AWS security services

**Limitations:**
* Only works with supported instance types
* Requires specific CNI configuration
* Can impact performance

#### 4.4.2 Network Policies for Micro-segmentation

**Default deny all traffic:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

**Allow specific service communication:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-to-api
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web
    ports:
    - protocol: TCP
      port: 8080
```

**Always allow DNS and health checks:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-and-health
spec:
  podSelector: {}
  policyTypes:
  - Egress
  - Ingress
  egress:
  # Allow DNS
  - to: []
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  ingress:
  # Allow health checks from kubelet
  - from: []
    ports:
    - protocol: TCP
      port: 8080  # Your health check port
```

---

### 4.5 Image Security and Supply Chain

#### 4.5.1 Image Scanning and Vulnerability Management

**ECR image scanning:**
```bash
# Enable scan on push
aws ecr put-image-scanning-configuration --repository-name myapp --image-scanning-configuration scanOnPush=true

# Manual scan
aws ecr start-image-scan --repository-name myapp --image-id imageTag=latest

# Get scan results
aws ecr describe-image-scan-findings --repository-name myapp --image-id imageTag=latest
```

**Admission controller for image scanning:**
```yaml
apiVersion: v1
kind: ValidatingAdmissionWebhook
metadata:
  name: image-security-webhook
webhooks:
- name: image-scan-check
  clientConfig:
    service:
      name: image-security-service
      namespace: security-system
      path: "/validate"
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
```

#### 4.5.2 Image Signing and Verification

**Using Cosign for image signing:**
```bash
# Sign image
cosign sign --key cosign.key myregistry/myapp:v1.0.0

# Verify signature
cosign verify --key cosign.pub myregistry/myapp:v1.0.0
```

**Policy enforcement with Gatekeeper:**
```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: requiresignedimages
spec:
  crd:
    spec:
      names:
        kind: RequireSignedImages
      validation:
        properties:
          trustedKeys:
            type: array
            items:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package requiresignedimages
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not is_signed(container.image)
          msg := sprintf("Image %v is not signed", [container.image])
        }
        
        is_signed(image) {
          # Implementation depends on your signing verification logic
        }
```

---

### 4.6 Audit Logging and Compliance

#### 4.6.1 EKS Audit Logging Configuration

**Enable audit logging:**
```bash
aws eks update-cluster-config \
  --name my-cluster \
  --logging '{"enable":["api","audit","authenticator","controllerManager","scheduler"]}'
```

**Audit policy for security events:**
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log secret access
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets"]
# Log RBAC changes
- level: RequestResponse
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["*"]
# Log security context changes
- level: Request
  resources:
  - group: ""
    resources: ["pods"]
  namespaces: ["production"]
  omitStages:
  - RequestReceived
```

#### 4.6.2 Security Monitoring and Alerting

**Key security metrics to monitor:**
* Failed authentication attempts
* Privilege escalation attempts
* Unauthorized secret access
* Network policy violations
* Image pull failures from untrusted registries

**Example Prometheus alerts:**
```yaml
groups:
- name: kubernetes-security
  rules:
  - alert: UnauthorizedSecretAccess
    expr: increase(apiserver_audit_total{verb="get",objectRef_resource="secrets",user_username!~"system:.*"}[5m]) > 0
    labels:
      severity: warning
    annotations:
      summary: "Unauthorized access to secrets detected"
      
  - alert: PrivilegedPodCreated
    expr: increase(apiserver_audit_total{verb="create",objectRef_resource="pods",requestObject_spec_securityContext_privileged="true"}[5m]) > 0
    labels:
      severity: critical
    annotations:
      summary: "Privileged pod created"
```

---

### 4.7 Security Incident Response Runbook

#### 4.7.1 "Pod can't access AWS services" Runbook

**Symptoms:** AWS API calls failing with permission errors

**Step 1: Verify IRSA setup**
```bash
kubectl describe sa <service-account> | grep eks.amazonaws.com/role-arn
kubectl describe pod <pod> | grep "Service Account"
```

**Step 2: Check token projection**
```bash
kubectl exec <pod> -- ls -la /var/run/secrets/eks.amazonaws.com/serviceaccount/
kubectl exec <pod> -- aws sts get-caller-identity
```

**Step 3: Verify IAM role and policies**
```bash
aws iam get-role --role-name <irsa-role-name>
aws iam list-attached-role-policies --role-name <irsa-role-name>
```

**Step 4: Test permissions**
```bash
kubectl exec <pod> -- aws s3 ls  # Or whatever AWS service you're trying to access
```

#### 4.7.2 "Pods being rejected by security policies" Runbook

**Symptoms:** Pod creation fails with security policy violations

**Step 1: Check namespace security labels**
```bash
kubectl get namespace <namespace> -o yaml | grep pod-security
```

**Step 2: Identify specific violations**
```bash
kubectl describe pod <pod> | grep -A 10 "violates PodSecurity"
```

**Step 3: Fix security context**
```bash
# Check current security context
kubectl get pod <pod> -o jsonpath='{.spec.securityContext}'
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].securityContext}'
```

**Step 4: Apply fixes and redeploy**

#### 4.7.3 "Secrets not updating" Runbook

**Symptoms:** Application using old secret values

**Step 1: Check External Secrets status**
```bash
kubectl get externalsecrets
kubectl describe externalsecret <name>
```

**Step 2: Verify secret store connectivity**
```bash
kubectl get secretstore
kubectl describe secretstore <name>
```

**Step 3: Check AWS Secrets Manager**
```bash
aws secretsmanager describe-secret --secret-id <secret-name>
aws secretsmanager get-secret-value --secret-id <secret-name>
```

**Step 4: Force refresh**
```bash
kubectl annotate externalsecret <name> force-sync=$(date +%s)
```

---

### 4.8 Health probes (critical for reliable services)

> Note: health probes aren't a security topic, but they're here because probe misconfiguration is one of the most common causes of cascading failures during deployments. Misplaced in this section, but important enough to keep rather than move.

#### 4.8.1 Readiness Probe (Traffic Routing Control)

**Purpose:** "Is it a good idea to send traffic to this Pod right now?"

**Common misconception:** Since Kubernetes manages pods, graceful draining isn't needed.

**Reality:** Without proper readiness probes:
- Traffic sent to pods before they're ready
- Traffic continues to terminating pods
- Rolling updates cause 5xx errors

**Best practices:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-service
spec:
  template:
    spec:
      containers:
      - name: app
        image: my-app:latest
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10    # Wait for app to start
          periodSeconds: 5           # Check every 5s
          timeoutSeconds: 3          # 3s timeout per check
          successThreshold: 1        # 1 success = ready
          failureThreshold: 3        # 3 failures = not ready
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                # Fail readiness probe immediately
                touch /tmp/shutdown
                # Wait for load balancer to update
                sleep 15
```

**Readiness probe endpoint implementation:**
```go
// Go example
func readinessHandler(w http.ResponseWriter, r *http.Request) {
    // Check if shutdown initiated
    if _, err := os.Stat("/tmp/shutdown"); err == nil {
        http.Error(w, "Shutting down", http.StatusServiceUnavailable)
        return
    }
    
    // Check application readiness (NOT dependencies)
    if !app.IsReady() {
        http.Error(w, "Not ready", http.StatusServiceUnavailable)
        return
    }
    
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("OK"))
}
```

**Critical rule:** Never depend on downstream services in readiness probes. If a database restarts, removing healthy pods from load balancers makes the outage worse.

#### 4.8.2 Liveness Probe (Container Health Check)

**Purpose:** "Is the container healthy, or should we restart it?"

**When to use:** Only when your application can deadlock and needs restart to recover.

**When NOT to use:** If you don't know why you need it, don't configure it.

**Best practices:**
```yaml
containers:
- name: app
  image: my-app:latest
  livenessProbe:
    httpGet:
      path: /health/live    # Different from readiness!
      port: 8080
    initialDelaySeconds: 60  # Give app time to start
    periodSeconds: 30        # Check every 30s (less frequent than readiness)
    timeoutSeconds: 5
    failureThreshold: 3      # 3 failures before restart
```

**Liveness probe implementation:**
```go
func livenessHandler(w http.ResponseWriter, r *http.Request) {
    // Only check internal application health
    // Never check dependencies (databases, external APIs)
    
    if app.IsDeadlocked() {
        http.Error(w, "Deadlocked", http.StatusInternalServerError)
        return
    }
    
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("OK"))
}
```

**Critical rules:**
- Never use the same endpoint for liveness and readiness
- Never check external dependencies in liveness probes
- Use conservative timeouts to avoid false positives under load

#### 4.8.3 Startup Probe (Slow-Starting Applications)

**Purpose:** "Should we start running the liveness probe now?"

**Use case:** Applications that take longer to start than liveness probe allows.

```yaml
containers:
- name: slow-app
  image: java-app:latest
  startupProbe:
    httpGet:
      path: /health/startup
      port: 8080
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 30      # Allow 5 minutes for startup (30 * 10s)
  livenessProbe:
    httpGet:
      path: /health/live
      port: 8080
    periodSeconds: 30
    timeoutSeconds: 5
    failureThreshold: 3
```

#### 4.8.4 Probe Failure Troubleshooting

**Common probe failures:**

1. **Readiness probe failing during load:**
```bash
# Check probe configuration
kubectl describe pod <pod-name>

# Check application logs
kubectl logs <pod-name> --previous

# Test probe endpoint manually
kubectl exec <pod-name> -- curl -f http://localhost:8080/health/ready
```

2. **Liveness probe causing restart loops:**
```bash
# Check restart count
kubectl get pods -o wide

# Check events
kubectl describe pod <pod-name>

# Increase probe timeouts temporarily
kubectl patch deployment <deployment> -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "<container>",
          "livenessProbe": {
            "timeoutSeconds": 10,
            "failureThreshold": 5
          }
        }]
      }
    }
  }
}'
```

3. **Startup probe preventing application start:**
```bash
# Check startup probe status
kubectl get pods -o jsonpath='{.items[*].status.conditions[?(@.type=="Ready")].message}'

# Extend startup probe timeout
kubectl patch deployment <deployment> -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "<container>",
          "startupProbe": {
            "failureThreshold": 60
          }
        }]
      }
    }
  }
}'
```


## 5. Storage and persistent volumes

Storage failures are different from everything else in this guide: they can mean data loss, not just downtime. What follows is the operational reality of running stateful workloads on EKS — the failure modes that catch teams off-guard.

---

### 5.1 EBS CSI Driver (The Critical Path for Stateful Workloads)

The EBS CSI driver is what makes persistent volumes work in EKS. When it fails, StatefulSets can't start, volumes can't attach, and data becomes inaccessible.

#### 5.1.1 EBS CSI Architecture and Failure Points

```
[Pod with PVC] 
  ↓ (volume mount request)
[Kubelet] 
  ↓ (CSI calls)
[EBS CSI Node Plugin] 
  ↓ (AWS API calls)
[EBS Volume Attach/Mount]
```

**Each layer can fail:**
* **Pod level**: Wrong PVC references, security context issues
* **Kubelet level**: Mount failures, device path issues
* **CSI level**: Controller crashes, node plugin issues, IAM permissions
* **AWS level**: EBS limits, AZ constraints, volume states

#### 5.1.2 Common EBS CSI Failure Modes

**A) Pods stuck in ContainerCreating**

**Symptoms:**
```bash
kubectl get pods | grep ContainerCreating
kubectl describe pod <pod-name>
# Shows: FailedMount, timeout waiting for volume to be attached
```

**Root causes:**
* Volume already attached to another node
* AZ mismatch between pod and volume
* EBS CSI controller/node plugin unhealthy
* IAM permissions missing

**Diagnosis:**
```bash
# Check CSI components
kubectl -n kube-system get pods | grep ebs-csi
kubectl -n kube-system logs deployment/ebs-csi-controller
kubectl -n kube-system logs daemonset/ebs-csi-node

# Check volume attachment status
kubectl get volumeattachment
kubectl describe volumeattachment <va-name>

# Check AWS side
aws ec2 describe-volumes --volume-ids <volume-id>
```

**B) Volume attachment timeouts**

**Symptoms:**
* Pods fail to start after node replacement
* "Multi-Attach error for volume" messages
* Long delays in pod scheduling

**Common scenario:**
1. Node fails/terminates unexpectedly
2. EBS volume remains "attached" to dead node
3. New pod can't attach volume until detached
4. Detachment can take 6+ minutes

**Force detachment (emergency):**
```bash
# Find the volume
kubectl get pv <pv-name> -o jsonpath='{.spec.csi.volumeHandle}'

# Force detach from AWS side
aws ec2 detach-volume --volume-id <volume-id> --force

# Delete stale VolumeAttachment
kubectl delete volumeattachment <va-name>
```

#### 5.1.3 EBS CSI Configuration and Tuning

**Essential CSI controller configuration:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ebs-csi-controller
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: ebs-plugin
        args:
        - controller
        - --endpoint=$(CSI_ENDPOINT)
        - --logtostderr
        - --v=2
        - --timeout=60s  # Increase for slow EBS operations
        env:
        - name: AWS_REGION
          value: us-west-2
        resources:
          requests:
            cpu: 10m
            memory: 40Mi
          limits:
            cpu: 100m
            memory: 256Mi
```

**Node plugin tuning for high-density workloads:**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ebs-csi-node
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: ebs-plugin
        args:
        - node
        - --endpoint=$(CSI_ENDPOINT)
        - --logtostderr
        - --v=2
        resources:
          requests:
            cpu: 10m
            memory: 40Mi
          limits:
            cpu: 100m
            memory: 256Mi
        securityContext:
          privileged: true
        volumeMounts:
        - name: kubelet-dir
          mountPath: /var/lib/kubelet
          mountPropagation: "Bidirectional"
```

---

### 5.2 Storage Classes and Dynamic Provisioning

#### 5.2.1 Production Storage Class Configuration

**GP3 with proper defaults:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-retain
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"        # Baseline IOPS
  throughput: "125"   # MB/s
  encrypted: "true"
  kmsKeyId: "alias/ebs-encryption-key"
reclaimPolicy: Retain  # Prevent accidental data loss
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer  # Critical for AZ placement
```

**High-performance storage for databases:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: io2-high-perf
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iops: "10000"
  encrypted: "true"
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

#### 5.2.2 Volume Binding Mode Implications

**Immediate vs WaitForFirstConsumer:**

**Immediate (default):**
- Volume created immediately when PVC is created
- Can cause AZ mismatch if pod scheduled to different AZ
- Good for pre-provisioning scenarios

**WaitForFirstConsumer (recommended):**
- Volume created only when pod is scheduled
- Ensures volume and pod are in same AZ
- Required for multi-AZ clusters

**AZ mismatch failure example:**
```bash
# PVC created with Immediate binding in us-west-2a
kubectl get pv <pv-name> -o jsonpath='{.metadata.labels.topology\.ebs\.csi\.aws\.com/zone}'
# Output: us-west-2a

# Pod scheduled to us-west-2b
kubectl get pod <pod-name> -o wide
# Shows node in us-west-2b

# Result: FailedMount due to AZ mismatch
```

---

### 5.3 StatefulSets and Persistent Volume Lifecycle

#### 5.3.1 StatefulSet Volume Management

**Proper StatefulSet with volume claims:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: database
  replicas: 3
  template:
    spec:
      containers:
      - name: db
        image: postgres:13
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        env:
        - name: POSTGRES_DB
          value: myapp
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3-retain
      resources:
        requests:
          storage: 100Gi
```

#### 5.3.2 StatefulSet Scaling and Volume Orphaning

**The orphaned PVC problem:**
When you scale down a StatefulSet, PVCs are NOT automatically deleted:

```bash
# Scale down from 5 to 3 replicas
kubectl scale statefulset database --replicas=3

# PVCs for database-3 and database-4 remain
kubectl get pvc | grep database
# database-data-0   Bound
# database-data-1   Bound  
# database-data-2   Bound
# database-data-3   Bound  # Orphaned!
# database-data-4   Bound  # Orphaned!
```

**Manual cleanup required:**
```bash
# Delete orphaned PVCs (DANGEROUS - data loss!)
kubectl delete pvc database-data-3 database-data-4

# Or retain for potential scale-up
# PVCs will be reused if you scale back up
```

#### 5.3.3 StatefulSet Rolling Updates and Volume Safety

**Safe rolling update configuration:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # Never update more than 1 pod at a time
  podManagementPolicy: OrderedReady  # Wait for each pod to be ready
```

**Volume expansion during updates:**
```bash
# Expand PVC (requires allowVolumeExpansion: true)
kubectl patch pvc database-data-0 -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'

# Check expansion status
kubectl describe pvc database-data-0
# Look for: FileSystemResizePending or FileSystemResizeSuccessful

# May require pod restart to complete filesystem resize
kubectl delete pod database-0  # StatefulSet will recreate it
```

---

### 5.4 Backup and Disaster Recovery

#### 5.4.1 EBS Snapshot-Based Backups

**Volume Snapshot Class:**
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshot-class
driver: ebs.csi.aws.com
deletionPolicy: Retain  # Keep snapshots even if VolumeSnapshot is deleted
parameters:
  tagSpecification_1: "Name=CreatedBy,Value=EKS-CSI"
  tagSpecification_2: "Environment=Production"
```

**Creating snapshots:**
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: database-backup-20240126
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    persistentVolumeClaimName: database-data-0
```

**Restoring from snapshot:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-restored
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: gp3-retain
  resources:
    requests:
      storage: 100Gi
  dataSource:
    name: database-backup-20240126
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

#### 5.4.2 Application-Consistent Backups

**Pre/post hooks for database consistency:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres-with-backup-hooks
  annotations:
    pre.hook.backup.velero.io/command: '["/bin/bash", "-c", "PGPASSWORD=$POSTGRES_PASSWORD pg_dump -h localhost -U $POSTGRES_USER $POSTGRES_DB > /backup/dump.sql"]'
    post.hook.backup.velero.io/command: '["/bin/bash", "-c", "rm -f /backup/dump.sql"]'
spec:
  containers:
  - name: postgres
    image: postgres:13
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
    - name: backup
      mountPath: /backup
```

#### 5.4.3 Cross-Region Backup Strategy

**Automated cross-region snapshot copying:**
```bash
#!/bin/bash
# Copy EBS snapshots to DR region
SOURCE_REGION="us-west-2"
DR_REGION="us-east-1"

# Get recent snapshots
SNAPSHOTS=$(aws ec2 describe-snapshots \
  --region $SOURCE_REGION \
  --owner-ids self \
  --filters "Name=tag:Environment,Values=Production" \
  --query 'Snapshots[?StartTime>=`2024-01-25`].SnapshotId' \
  --output text)

for snapshot in $SNAPSHOTS; do
  echo "Copying $snapshot to $DR_REGION"
  aws ec2 copy-snapshot \
    --region $DR_REGION \
    --source-region $SOURCE_REGION \
    --source-snapshot-id $snapshot \
    --description "DR copy of $snapshot"
done
```

---

### 5.5 Performance and Monitoring

#### 5.5.1 EBS Performance Characteristics

**IOPS and throughput limits by volume type:**

| Volume Type | Max IOPS | Max Throughput | Use Case |
|-------------|----------|----------------|----------|
| gp3 | 16,000 | 1,000 MB/s | General purpose |
| io1 | 64,000 | 1,000 MB/s | High IOPS |
| io2 | 64,000 | 1,000 MB/s | Mission critical |
| io2 Block Express | 256,000 | 4,000 MB/s | Extreme performance |

**Instance-level limits also apply:**
```bash
# Check instance storage performance limits
aws ec2 describe-instance-types \
  --instance-types m5.large \
  --query 'InstanceTypes[0].EbsInfo'
```

#### 5.5.2 Storage Performance Monitoring

**Key metrics to monitor:**
```yaml
# Prometheus recording rules for storage
groups:
- name: storage-performance
  rules:
  - record: ebs:iops_utilization
    expr: rate(node_disk_reads_completed_total[5m]) + rate(node_disk_writes_completed_total[5m])
    
  - record: ebs:throughput_utilization  
    expr: rate(node_disk_read_bytes_total[5m]) + rate(node_disk_written_bytes_total[5m])
    
  - record: ebs:latency_p99
    expr: histogram_quantile(0.99, rate(node_disk_io_time_seconds_total[5m]))
```

**Storage alerts:**
```yaml
groups:
- name: storage-alerts
  rules:
  - alert: HighDiskLatency
    expr: ebs:latency_p99 > 0.1  # 100ms
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High disk latency detected"
      
  - alert: EBSVolumeStuck
    expr: increase(kubelet_volume_stats_available_bytes[10m]) == 0
    for: 10m
    labels:
      severity: critical
    annotations:
      summary: "EBS volume appears stuck"
```

#### 5.5.3 Storage Capacity Management

**Automatic PVC expansion:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pvc-autoresizer-config
data:
  config.yaml: |
    intervals:
      - name: "5min"
        interval: 5m
    rules:
      - name: "expand-when-80-percent-full"
        selector:
          matchLabels:
            app: database
        thresholds:
          - threshold: 80
            increase: "20%"
          - threshold: 90
            increase: "50%"
```

---

### 5.6 Multi-AZ and Cross-AZ Storage Patterns

#### 5.6.1 EBS Volume AZ Constraints

**The fundamental constraint:**
EBS volumes are AZ-specific and cannot be attached to instances in different AZs.

**Impact on StatefulSets:**
```yaml
# This will fail if pods get scheduled across AZs
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  replicas: 3
  template:
    spec:
      # No AZ constraints = pods can land anywhere
      # But PVCs are bound to specific AZs
      containers:
      - name: db
        image: postgres:13
```

**Solution - AZ-aware scheduling:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  replicas: 3
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: database
      containers:
      - name: db
        image: postgres:13
```

#### 5.6.2 Cross-AZ Data Replication Patterns

**For databases requiring cross-AZ replication:**
```yaml
# Primary in us-west-2a
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-primary
spec:
  replicas: 1
  template:
    spec:
      nodeSelector:
        topology.kubernetes.io/zone: us-west-2a
      containers:
      - name: postgres
        image: postgres:13
        env:
        - name: POSTGRES_REPLICATION_MODE
          value: master

---
# Replica in us-west-2b  
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-replica
spec:
  replicas: 1
  template:
    spec:
      nodeSelector:
        topology.kubernetes.io/zone: us-west-2b
      containers:
      - name: postgres
        image: postgres:13
        env:
        - name: POSTGRES_REPLICATION_MODE
          value: slave
        - name: POSTGRES_MASTER_SERVICE
          value: postgres-primary
```

---

### 5.7 Storage Troubleshooting Runbooks

#### 5.7.1 "Pod stuck in ContainerCreating" Runbook

**Symptoms:** Pod won't start, stuck in ContainerCreating state

**Step 1: Check pod events**
```bash
kubectl describe pod <pod-name>
# Look for: FailedMount, timeout, volume attachment errors
```

**Step 2: Check PVC status**
```bash
kubectl get pvc <pvc-name>
kubectl describe pvc <pvc-name>
# Status should be "Bound"
```

**Step 3: Check VolumeAttachment**
```bash
kubectl get volumeattachment
kubectl describe volumeattachment <va-name>
# Look for attachment errors
```

**Step 4: Check CSI components**
```bash
kubectl -n kube-system get pods | grep ebs-csi
kubectl -n kube-system logs deployment/ebs-csi-controller
kubectl -n kube-system logs daemonset/ebs-csi-node -c ebs-plugin
```

**Step 5: Check AWS EBS volume**
```bash
# Get volume ID from PV
kubectl get pv <pv-name> -o jsonpath='{.spec.csi.volumeHandle}'

# Check volume status
aws ec2 describe-volumes --volume-ids <volume-id>
# State should be "available" or "in-use"
```

#### 5.7.2 "Volume attachment timeout" Runbook

**Symptoms:** Long delays in pod startup, attachment timeout errors

**Step 1: Identify stuck attachment**
```bash
kubectl get volumeattachment -o wide
# Look for old attachments with "Attaching" status
```

**Step 2: Check if volume is stuck on dead node**
```bash
aws ec2 describe-volumes --volume-ids <volume-id> \
  --query 'Volumes[0].Attachments'
# Check if attached to non-existent instance
```

**Step 3: Force detachment (if safe)**
```bash
# Verify the instance is really dead
aws ec2 describe-instances --instance-ids <instance-id>

# Force detach
aws ec2 detach-volume --volume-id <volume-id> --force

# Clean up VolumeAttachment
kubectl delete volumeattachment <va-name>
```

**Step 4: Verify pod can start**
```bash
kubectl get pod <pod-name>
# Should transition to Running
```

#### 5.7.3 "PVC expansion stuck" Runbook

**Symptoms:** PVC shows larger size but pod still sees old size

**Step 1: Check PVC conditions**
```bash
kubectl describe pvc <pvc-name>
# Look for: FileSystemResizePending, VolumeResizeSuccessful
```

**Step 2: Check if pod restart is needed**
```bash
# Some filesystems require pod restart to complete resize
kubectl get pod <pod-name> -o jsonpath='{.metadata.creationTimestamp}'
kubectl get pvc <pvc-name> -o jsonpath='{.status.conditions[?(@.type=="FileSystemResizePending")].lastTransitionTime}'
```

**Step 3: Restart pod if needed**
```bash
kubectl delete pod <pod-name>
# StatefulSet/Deployment will recreate it
```

**Step 4: Verify expansion completed**
```bash
kubectl exec <pod-name> -- df -h /data
# Should show new size
```

---

## 6. Observability and monitoring

When monitoring is broken you can't tell the difference between "the app is slow" and "the cluster is degraded". This is about building observability that actually helps during incidents, not dashboards that look good in screenshots.

---

### 6.1 The EKS Observability Stack (What You Actually Need)

#### 6.1.1 Metrics Collection Architecture

```
[Application Metrics] → [Prometheus] → [Long-term Storage] → [Alerting/Dashboards]
[System Metrics] → [Node Exporter] ↗
[Kubernetes Metrics] → [kube-state-metrics] ↗
[AWS Metrics] → [CloudWatch] → [Prometheus via adapter] ↗
```

You need metrics at multiple layers because EKS failures can happen at any level:
* Application layer (your code)
* Kubernetes layer (pods, services, ingress)
* Node layer (CPU, memory, disk, network)
* AWS layer (EBS, ENI, load balancers)

#### 6.1.2 Essential Metrics Components

**Core Prometheus stack:**
```yaml
# Prometheus server configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    
    rule_files:
    - "/etc/prometheus/rules/*.yml"
    
    scrape_configs:
    # Kubernetes API server
    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https
    
    # Node metrics
    - job_name: 'kubernetes-nodes'
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
    
    # Pod metrics
    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
```

**Node exporter for system metrics:**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.6.1
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/host/root
        - --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)
        - --collector.netdev.device-exclude=^(veth.*|docker.*|br-.*|lo)$$
        - --collector.conntrack  # Critical for connection tracking issues
        - --collector.ethtool     # AWS ENA metrics
        - --collector.ethtool.metrics-include=^(ena_.*|.*_exceeded)$$
        ports:
        - containerPort: 9100
          hostPort: 9100
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
        - name: root
          mountPath: /host/root
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: root
        hostPath:
          path: /
```

---

### 6.2 EKS-Specific Monitoring (The Metrics That Matter)

#### 6.2.1 Control Plane Monitoring

**API Server health:**
```yaml
# Critical API server alerts
groups:
- name: kubernetes-apiserver
  rules:
  - alert: KubernetesApiServerDown
    expr: up{job="kubernetes-apiservers"} == 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Kubernetes API server is down"
      
  - alert: KubernetesApiServerLatency
    expr: histogram_quantile(0.99, sum(rate(apiserver_request_duration_seconds_bucket[5m])) by (le, verb)) > 1
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Kubernetes API server high latency"
      
  - alert: KubernetesApiServerErrors
    expr: sum(rate(apiserver_request_total{code=~"5.."}[5m])) / sum(rate(apiserver_request_total[5m])) > 0.05
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Kubernetes API server error rate > 5%"
```

#### 6.2.2 Node-Level Monitoring

**AWS-specific node metrics:**
```yaml
# AWS ENA network limits
groups:
- name: aws-node-limits
  rules:
  - alert: AWSNetworkLimitExceeded
    expr: rate(node_ethtool_conntrack_allowance_exceeded[5m]) > 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "AWS connection tracking limit exceeded on {{ $labels.instance }}"
      
  - alert: AWSLinkLocalLimitExceeded
    expr: rate(node_ethtool_linklocal_allowance_exceeded[5m]) > 5
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "AWS link-local rate limit exceeded on {{ $labels.instance }}"
      
  - alert: AWSBandwidthLimitExceeded
    expr: rate(node_ethtool_bw_in_allowance_exceeded[5m]) > 0 or rate(node_ethtool_bw_out_allowance_exceeded[5m]) > 0
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "AWS bandwidth limit exceeded on {{ $labels.instance }}"
```

**Connection tracking monitoring:**
```yaml
# Conntrack exhaustion alerts
- alert: ConntrackTableFull
  expr: (node_nf_conntrack_entries / node_nf_conntrack_entries_limit) > 0.8
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Conntrack table 80% full on {{ $labels.instance }}"
    
- alert: ConntrackTableNearlyFull
  expr: (node_nf_conntrack_entries / node_nf_conntrack_entries_limit) > 0.95
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Conntrack table 95% full on {{ $labels.instance }}"
```

#### 6.2.3 Pod and Container Monitoring

**Container resource monitoring:**
```yaml
# Container resource alerts
groups:
- name: container-resources
  rules:
  - alert: ContainerHighCPUUsage
    expr: (sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (namespace, pod, container) / sum(container_spec_cpu_quota{container!=""}/container_spec_cpu_period{container!=""}) by (namespace, pod, container)) > 0.9
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Container {{ $labels.namespace }}/{{ $labels.pod }}/{{ $labels.container }} high CPU usage"
      
  - alert: ContainerHighMemoryUsage
    expr: (container_memory_working_set_bytes{container!=""} / container_spec_memory_limit_bytes{container!=""}) > 0.9
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Container {{ $labels.namespace }}/{{ $labels.pod }}/{{ $labels.container }} high memory usage"
      
  - alert: ContainerOOMKilled
    expr: increase(kube_pod_container_status_restarts_total[5m]) > 0 and on(namespace, pod, container) kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
    for: 0m
    labels:
      severity: warning
    annotations:
      summary: "Container {{ $labels.namespace }}/{{ $labels.pod }}/{{ $labels.container }} was OOM killed"
```

---

### 6.3 DNS and Service Discovery Monitoring

#### 6.3.1 CoreDNS Performance Monitoring

**CoreDNS metrics collection:**
```yaml
# CoreDNS monitoring
- job_name: 'coredns'
  kubernetes_sd_configs:
  - role: endpoints
  relabel_configs:
  - source_labels: [__meta_kubernetes_service_name]
    action: keep
    regex: kube-dns
  - source_labels: [__meta_kubernetes_endpoint_port_name]
    action: keep
    regex: metrics
```

**CoreDNS alerts:**
```yaml
groups:
- name: coredns
  rules:
  - alert: CoreDNSDown
    expr: up{job="coredns"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "CoreDNS is down"
      
  - alert: CoreDNSHighLatency
    expr: histogram_quantile(0.99, sum(rate(coredns_dns_request_duration_seconds_bucket[5m])) by (le)) > 0.1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "CoreDNS high latency (99th percentile > 100ms)"
      
  - alert: CoreDNSHighErrorRate
    expr: sum(rate(coredns_dns_responses_total{rcode!="NOERROR"}[5m])) / sum(rate(coredns_dns_responses_total[5m])) > 0.05
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "CoreDNS error rate > 5%"
```

#### 6.3.2 Service Discovery Health Checks

**Synthetic DNS monitoring:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dns-monitor
  labels:
    app: dns-monitor
spec:
  containers:
  - name: monitor
    image: busybox
    command:
    - /bin/sh
    - -c
    - |
      while true; do
        # Test internal DNS
        if nslookup kubernetes.default.svc.cluster.local; then
          echo "internal_dns_success 1" | nc -u -w1 prometheus-pushgateway 9091
        else
          echo "internal_dns_success 0" | nc -u -w1 prometheus-pushgateway 9091
        fi
        
        # Test external DNS
        if nslookup google.com; then
          echo "external_dns_success 1" | nc -u -w1 prometheus-pushgateway 9091
        else
          echo "external_dns_success 0" | nc -u -w1 prometheus-pushgateway 9091
        fi
        
        sleep 30
      done
```

---

### 6.4 AWS Integration Monitoring

#### 6.4.1 Load Balancer Monitoring

**ALB/NLB CloudWatch metrics:**
```yaml
# CloudWatch exporter configuration for ALB metrics
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudwatch-exporter-config
data:
  config.yml: |
    region: us-west-2
    metrics:
    # ALB metrics
    - aws_namespace: AWS/ApplicationELB
      aws_metric_name: TargetResponseTime
      aws_dimensions: [LoadBalancer]
      aws_statistics: [Average]
      
    - aws_namespace: AWS/ApplicationELB
      aws_metric_name: HTTPCode_Target_5XX_Count
      aws_dimensions: [LoadBalancer]
      aws_statistics: [Sum]
      
    # NLB metrics  
    - aws_namespace: AWS/NetworkELB
      aws_metric_name: TCP_ELB_Reset_Count
      aws_dimensions: [LoadBalancer]
      aws_statistics: [Sum]
      
    # NAT Gateway metrics
    - aws_namespace: AWS/NatGateway
      aws_metric_name: IdleTimeoutCount
      aws_dimensions: [NatGatewayId]
      aws_statistics: [Sum]
```

**Load balancer alerts:**
```yaml
groups:
- name: aws-loadbalancer
  rules:
  - alert: ALBHighLatency
    expr: aws_applicationelb_target_response_time_average > 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "ALB {{ $labels.load_balancer }} high response time"
      
  - alert: ALBHighErrorRate
    expr: rate(aws_applicationelb_httpcode_target_5_xx_count_sum[5m]) > 10
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "ALB {{ $labels.load_balancer }} high 5xx error rate"
      
  - alert: NLBConnectionResets
    expr: rate(aws_networkelb_tcp_elb_reset_count_sum[5m]) > 5
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "NLB {{ $labels.load_balancer }} connection resets detected"
```

#### 6.4.2 EBS and Storage Monitoring

**EBS performance metrics:**
```yaml
# EBS CloudWatch metrics
- aws_namespace: AWS/EBS
  aws_metric_name: VolumeReadOps
  aws_dimensions: [VolumeId]
  aws_statistics: [Sum]
  
- aws_namespace: AWS/EBS
  aws_metric_name: VolumeWriteOps
  aws_dimensions: [VolumeId]
  aws_statistics: [Sum]
  
- aws_namespace: AWS/EBS
  aws_metric_name: VolumeTotalReadTime
  aws_dimensions: [VolumeId]
  aws_statistics: [Sum]
  
- aws_namespace: AWS/EBS
  aws_metric_name: BurstBalance
  aws_dimensions: [VolumeId]
  aws_statistics: [Average]
```

**Storage alerts:**
```yaml
groups:
- name: ebs-storage
  rules:
  - alert: EBSBurstBalanceLow
    expr: aws_ebs_burst_balance_average < 20
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "EBS volume {{ $labels.volume_id }} burst balance low"
      
  - alert: EBSHighLatency
    expr: (aws_ebs_volume_total_read_time_sum + aws_ebs_volume_total_write_time_sum) / (aws_ebs_volume_read_ops_sum + aws_ebs_volume_write_ops_sum) > 0.1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "EBS volume {{ $labels.volume_id }} high latency"
```

---

### 6.5 Application Performance Monitoring

#### 6.5.1 Golden Signals for Kubernetes Workloads

**The four golden signals adapted for Kubernetes:**

1. **Latency** - Request duration
2. **Traffic** - Request rate  
3. **Errors** - Error rate
4. **Saturation** - Resource utilization

**Application metrics instrumentation:**
```yaml
# Example application with Prometheus metrics
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8080
        env:
        - name: METRICS_ENABLED
          value: "true"
```

**Golden signals alerts:**
```yaml
groups:
- name: golden-signals
  rules:
  # Latency
  - alert: HighRequestLatency
    expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)) > 0.5
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High request latency for {{ $labels.service }}"
      
  # Traffic
  - alert: LowTrafficVolume
    expr: sum(rate(http_requests_total[5m])) by (service) < 1
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Low traffic volume for {{ $labels.service }}"
      
  # Errors
  - alert: HighErrorRate
    expr: sum(rate(http_requests_total{status=~"5.."}[5m])) by (service) / sum(rate(http_requests_total[5m])) by (service) > 0.05
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "High error rate for {{ $labels.service }}"
      
  # Saturation
  - alert: HighCPUSaturation
    expr: avg(rate(container_cpu_usage_seconds_total[5m])) by (namespace, pod) > 0.8
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High CPU saturation for {{ $labels.namespace }}/{{ $labels.pod }}"
```

#### 6.5.2 Distributed Tracing Integration

**Jaeger deployment for EKS:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
spec:
  template:
    spec:
      containers:
      - name: jaeger
        image: jaegertracing/all-in-one:latest
        env:
        - name: COLLECTOR_ZIPKIN_HTTP_PORT
          value: "9411"
        - name: SPAN_STORAGE_TYPE
          value: "elasticsearch"
        - name: ES_SERVER_URLS
          value: "http://elasticsearch:9200"
        ports:
        - containerPort: 16686  # UI
        - containerPort: 14268  # HTTP collector
        - containerPort: 6831   # UDP agent
```

---

### 6.6 Log Aggregation and Analysis

#### 6.6.1 Centralized Logging Architecture

**Fluent Bit for log collection:**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
spec:
  template:
    spec:
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:2.1.10
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc/
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch"
        - name: FLUENT_ELASTICSEARCH_PORT
          value: "9200"
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: fluent-bit-config
        configMap:
          name: fluent-bit-config
```

**Fluent Bit configuration:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020

    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            cri
        Tag               kube.*
        Refresh_Interval  5
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Merge_Log           On
        K8S-Logging.Parser  On
        K8S-Logging.Exclude Off

    [OUTPUT]
        Name  es
        Match *
        Host  ${FLUENT_ELASTICSEARCH_HOST}
        Port  ${FLUENT_ELASTICSEARCH_PORT}
        Index fluent-bit
        Type  _doc
```

#### 6.6.2 Log-Based Alerting

**Critical log patterns to monitor:**
```yaml
# Log-based alerts using Loki/Promtail
groups:
- name: log-alerts
  rules:
  - alert: PodCrashLooping
    expr: sum(rate({namespace="production"} |= "CrashLoopBackOff"[5m])) > 0
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "Pod crash looping detected in production"
      
  - alert: OutOfMemoryKills
    expr: sum(rate({namespace="production"} |= "OOMKilled"[5m])) > 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "OOM kills detected in production"
      
  - alert: ImagePullErrors
    expr: sum(rate({namespace="production"} |= "ImagePullBackOff"[5m])) > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Image pull errors in production"
```

---

### 6.7 Incident Response Dashboards

#### 6.7.1 EKS Incident Response Dashboard

**Critical metrics for incident response:**
```json
{
  "dashboard": {
    "title": "EKS Incident Response",
    "panels": [
      {
        "title": "Cluster Health Overview",
        "targets": [
          {
            "expr": "up{job=\"kubernetes-apiservers\"}",
            "legendFormat": "API Server"
          },
          {
            "expr": "up{job=\"coredns\"}",
            "legendFormat": "CoreDNS"
          },
          {
            "expr": "kube_node_status_condition{condition=\"Ready\",status=\"true\"}",
            "legendFormat": "Ready Nodes"
          }
        ]
      },
      {
        "title": "Pod Status Distribution",
        "targets": [
          {
            "expr": "sum by (phase) (kube_pod_status_phase)",
            "legendFormat": "{{ phase }}"
          }
        ]
      },
      {
        "title": "Resource Utilization",
        "targets": [
          {
            "expr": "100 - (avg(irate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
            "legendFormat": "CPU Usage %"
          },
          {
            "expr": "(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100",
            "legendFormat": "Memory Usage %"
          }
        ]
      },
      {
        "title": "Network Issues",
        "targets": [
          {
            "expr": "rate(node_ethtool_conntrack_allowance_exceeded[5m])",
            "legendFormat": "Conntrack Exceeded - {{ instance }}"
          },
          {
            "expr": "rate(node_ethtool_linklocal_allowance_exceeded[5m])",
            "legendFormat": "Link-local Exceeded - {{ instance }}"
          }
        ]
      }
    ]
  }
}
```

#### 6.7.2 Application Health Dashboard

**Service-level indicators:**
```json
{
  "dashboard": {
    "title": "Application Health",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (service)",
            "legendFormat": "{{ service }}"
          }
        ]
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) by (service) / sum(rate(http_requests_total[5m])) by (service)",
            "legendFormat": "{{ service }}"
          }
        ]
      },
      {
        "title": "Response Time",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))",
            "legendFormat": "95th percentile - {{ service }}"
          },
          {
            "expr": "histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))",
            "legendFormat": "50th percentile - {{ service }}"
          }
        ]
      }
    ]
  }
}
```

---

### 6.8 Monitoring Troubleshooting Runbooks

#### 6.8.1 "Metrics missing" Runbook

**Symptoms:** Dashboards show no data, alerts not firing

**Step 1: Check Prometheus targets**
```bash
# Access Prometheus UI
kubectl port-forward svc/prometheus 9090:9090

# Check targets status at http://localhost:9090/targets
# Look for targets in "DOWN" state
```

**Step 2: Verify service discovery**
```bash
# Check if services have correct annotations
kubectl get svc -o yaml | grep -A 5 -B 5 prometheus.io

# Check if pods are exposing metrics
kubectl exec <pod-name> -- curl localhost:8080/metrics
```

**Step 3: Check network connectivity**
```bash
# Test connectivity from Prometheus pod
kubectl exec prometheus-pod -- nc -zv <target-service> <port>
```

#### 6.8.2 "Alerts not firing" Runbook

**Symptoms:** Known issues not triggering alerts

**Step 1: Check alert rules**
```bash
# Access Prometheus rules page
# http://localhost:9090/rules

# Verify rule syntax
promtool check rules /path/to/rules.yml
```

**Step 2: Check Alertmanager**
```bash
kubectl logs deployment/alertmanager

# Check Alertmanager config
kubectl get configmap alertmanager-config -o yaml
```

**Step 3: Test alert conditions**
```bash
# Query the alert condition directly in Prometheus
# Example: up{job="kubernetes-apiservers"} == 0
```

---

This observability section provides the foundation for effective incident response in EKS. The focus is on metrics and alerts that actually help during outages, not just operational dashboards. The key is building observability that distinguishes between application issues and platform issues quickly.
## 7. Scaling and performance

Scaling failures show up as "the app is slow" when the real issue is resource contention, autoscaler misconfiguration, or hitting AWS service limits. What follows: scaling patterns that actually work under load, and the ways they break during traffic spikes.

---

### 7.1 Horizontal Pod Autoscaler (HPA) Operational Reality

#### 7.1.1 HPA Architecture and Dependencies

```
[Metrics Server] → [HPA Controller] → [Deployment/ReplicaSet] → [Pods]
     ↑                    ↑
[kubelet cAdvisor]   [Custom Metrics API]
```

**Critical dependencies:**
* **Metrics Server** - Must be healthy for CPU/memory-based scaling
* **Resource requests** - HPA cannot function without them
* **Custom metrics** - For advanced scaling (queue depth, response time)
* **Node capacity** - Scaling is useless if nodes can't accommodate new pods

#### 7.1.2 Common HPA Failure Modes

**A) HPA shows "unknown" metrics**

**Symptoms:**
```bash
kubectl get hpa
# NAME     REFERENCE          TARGETS         MINPODS   MAXPODS   REPLICAS
# web-app  Deployment/web-app <unknown>/80%   2         10        2
```

**Root causes:**
* Metrics server down or unhealthy
* Pods missing resource requests
* Metrics server can't reach kubelet

**Diagnosis:**
```bash
# Check metrics server
kubectl -n kube-system get pods -l k8s-app=metrics-server
kubectl -n kube-system logs -l k8s-app=metrics-server

# Check if metrics are available
kubectl top pods
kubectl top nodes

# Check pod resource requests
kubectl describe deployment web-app | grep -A 10 "Requests:"
```

**B) HPA scaling thrashing (rapid scale up/down)**

**Symptoms:**
* Replica count oscillates rapidly
* Pods constantly being created and destroyed
* Performance degrades due to churn

**Root causes:**
* Scaling thresholds too sensitive
* Resource requests don't match actual usage
* Missing stabilization windows

**Fix with stabilization:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # Wait 60s before scaling up again
      policies:
      - type: Percent
        value: 100                      # Max 100% increase per step
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300   # Wait 5min before scaling down
      policies:
      - type: Percent
        value: 10                       # Max 10% decrease per step
        periodSeconds: 60
```

#### 7.1.3 Custom Metrics Scaling

**Scaling based on queue depth:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: worker
  minReplicas: 1
  maxReplicas: 20
  metrics:
  - type: External
    external:
      metric:
        name: sqs_queue_depth
        selector:
          matchLabels:
            queue_name: "work-queue"
      target:
        type: AverageValue
        averageValue: "5"  # Scale up when queue depth > 5 per pod
```

**Prometheus adapter configuration:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: adapter-config
data:
  config.yaml: |
    rules:
    - seriesQuery: 'sqs_queue_depth{queue_name!=""}'
      resources:
        overrides:
          queue_name: {resource: "queue"}
      name:
        matches: "^sqs_queue_depth"
        as: "sqs_queue_depth"
      metricsQuery: 'avg(sqs_queue_depth{queue_name="<<.LabelMatchers>>"})'
```

---

### 7.2 Cluster Autoscaler (CA) Operational Patterns

#### 7.2.1 Cluster Autoscaler Configuration

**Production CA configuration:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.27.3
        name: cluster-autoscaler
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/eks-cluster-name
        - --balance-similar-node-groups
        - --scale-down-enabled=true
        - --scale-down-delay-after-add=10m
        - --scale-down-unneeded-time=10m
        - --scale-down-utilization-threshold=0.5
        - --skip-nodes-with-system-pods=false
        env:
        - name: AWS_REGION
          value: us-west-2
        resources:
          limits:
            cpu: 100m
            memory: 300Mi
          requests:
            cpu: 100m
            memory: 300Mi
```

#### 7.2.2 Common CA Failure Modes

**A) Nodes not scaling up despite pending pods**

**Symptoms:**
```bash
kubectl get pods -A | grep Pending
kubectl describe pod <pending-pod>
# Shows: 0/X nodes are available: insufficient cpu/memory
```

**Root causes:**
* Node group max size reached
* AWS service limits (EC2, EIP, etc.)
* Pod resource requests too large for any instance type
* Taints/tolerations preventing scheduling

**Diagnosis:**
```bash
# Check CA logs
kubectl -n kube-system logs deployment/cluster-autoscaler

# Check node group limits
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name>

# Check AWS service limits
aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A  # Running On-Demand instances
```

**B) Nodes scaling down too aggressively**

**Symptoms:**
* Pods getting evicted during low traffic
* Services become unavailable during scale-down
* Frequent node churn

**Tuning scale-down behavior:**
```yaml
# Cluster autoscaler configuration
- --scale-down-delay-after-add=10m      # Wait 10min after scale-up before considering scale-down
- --scale-down-unneeded-time=10m        # Node must be unneeded for 10min before removal
- --scale-down-utilization-threshold=0.5 # Only remove nodes <50% utilized
- --max-node-provision-time=15m         # Give up on node provisioning after 15min
```

#### 7.2.3 Node Group Strategy

**Multiple node groups for different workload types:**
```yaml
# General purpose workloads
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: production-cluster
nodeGroups:
- name: general-purpose
  instanceTypes: ["m5.large", "m5.xlarge", "m5.2xlarge"]
  minSize: 2
  maxSize: 20
  desiredCapacity: 5
  labels:
    workload-type: general
  tags:
    k8s.io/cluster-autoscaler/enabled: "true"
    k8s.io/cluster-autoscaler/production-cluster: "owned"

# Compute-intensive workloads
- name: compute-optimized
  instanceTypes: ["c5.2xlarge", "c5.4xlarge"]
  minSize: 0
  maxSize: 10
  desiredCapacity: 0
  labels:
    workload-type: compute
  taints:
    - key: workload-type
      value: compute
      effect: NoSchedule
  tags:
    k8s.io/cluster-autoscaler/enabled: "true"
    k8s.io/cluster-autoscaler/production-cluster: "owned"

# Memory-intensive workloads  
- name: memory-optimized
  instanceTypes: ["r5.xlarge", "r5.2xlarge"]
  minSize: 0
  maxSize: 5
  desiredCapacity: 0
  labels:
    workload-type: memory
  taints:
    - key: workload-type
      value: memory
      effect: NoSchedule
```

---

### 7.3 Vertical Pod Autoscaler (VPA) and Right-Sizing

#### 7.3.1 VPA for Resource Discovery

**VPA in recommendation mode:**
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  updatePolicy:
    updateMode: "Off"  # Only generate recommendations, don't auto-update
  resourcePolicy:
    containerPolicies:
    - containerName: web-app
      maxAllowed:
        cpu: 2
        memory: 4Gi
      minAllowed:
        cpu: 100m
        memory: 128Mi
```

**Getting VPA recommendations:**
```bash
# Get current recommendations
kubectl describe vpa web-app-vpa

# Example output:
# Recommendation:
#   Container Recommendations:
#     Container Name:  web-app
#     Lower Bound:
#       Cpu:     100m
#       Memory:  128Mi
#     Target:
#       Cpu:     250m
#       Memory:  512Mi
#     Uncapped Target:
#       Cpu:     250m
#       Memory:  512Mi
#     Upper Bound:
#       Cpu:     500m
#       Memory:  1Gi
```

#### 7.3.2 Resource Request Right-Sizing

**Common resource request mistakes:**
```yaml
# BAD: Overprovisioned
resources:
  requests:
    cpu: 2000m      # App only uses 200m
    memory: 4Gi     # App only uses 512Mi
  limits:
    cpu: 4000m
    memory: 8Gi

# GOOD: Right-sized based on actual usage
resources:
  requests:
    cpu: 250m       # Based on VPA recommendation + buffer
    memory: 512Mi   # Based on actual usage patterns
  limits:
    cpu: 500m       # 2x requests for burst capacity
    memory: 1Gi     # Hard limit to prevent OOM
```

**Resource monitoring for right-sizing:**
```bash
# Monitor actual resource usage
kubectl top pods --containers

# Get detailed resource usage over time
kubectl exec prometheus-pod -- promtool query instant \
  'avg_over_time(container_cpu_usage_seconds_total{container="web-app"}[24h])'
```

---

### 7.4 Performance Under Load

#### 7.4.1 Load Testing EKS Workloads

**Gradual load testing approach:**
```yaml
# Load test job
apiVersion: batch/v1
kind: Job
metadata:
  name: load-test
spec:
  parallelism: 10
  template:
    spec:
      containers:
      - name: load-test
        image: loadimpact/k6:latest
        command:
        - k6
        - run
        - --vus=50
        - --duration=10m
        - --rps=100
        - /scripts/load-test.js
        volumeMounts:
        - name: test-scripts
          mountPath: /scripts
      volumes:
      - name: test-scripts
        configMap:
          name: load-test-scripts
      restartPolicy: Never
```

**Load test script example:**
```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 100 },   // Ramp up
    { duration: '5m', target: 100 },   // Stay at 100 users
    { duration: '2m', target: 200 },   // Ramp up to 200
    { duration: '5m', target: 200 },   // Stay at 200
    { duration: '2m', target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% of requests under 500ms
    http_req_failed: ['rate<0.1'],     // Error rate under 10%
  },
};

export default function() {
  let response = http.get('http://web-app.default.svc.cluster.local/api/health');
  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  sleep(1);
}
```

#### 7.4.2 Performance Bottleneck Identification

**Common EKS performance bottlenecks:**

1. **CPU throttling due to limits**
2. **Memory pressure causing OOM kills**
3. **Network bandwidth limits (AWS instance-level)**
4. **Storage IOPS limits (EBS)**
5. **DNS resolution delays (CoreDNS overload)**
6. **Connection tracking limits (conntrack)**

**Performance monitoring queries:**
```yaml
# CPU throttling detection
sum(rate(container_cpu_cfs_throttled_seconds_total[5m])) by (namespace, pod, container) > 0

# Memory pressure detection  
(container_memory_working_set_bytes / container_spec_memory_limit_bytes) > 0.8

# Network bandwidth utilization
rate(node_network_receive_bytes_total[5m]) + rate(node_network_transmit_bytes_total[5m])

# Storage IOPS utilization
rate(node_disk_reads_completed_total[5m]) + rate(node_disk_writes_completed_total[5m])

# DNS latency
histogram_quantile(0.95, sum(rate(coredns_dns_request_duration_seconds_bucket[5m])) by (le))
```

---

### 7.5 AWS Service Limits and Quotas

#### 7.5.1 Common EKS-Related Service Limits

**EC2 limits that affect scaling:**
```bash
# Check current EC2 limits
aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A  # On-Demand instances
aws service-quotas get-service-quota --service-code ec2 --quota-code L-34B43A08  # All Standard Spot Instance Requests
aws service-quotas get-service-quota --service-code ec2 --quota-code L-0263D0A3  # Security Groups per VPC
aws service-quotas get-service-quota --service-code ec2 --quota-code L-FE5A380F  # Network Interfaces per VPC

# Check current usage
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running" --query 'Reservations[*].Instances[*].InstanceType' | jq -r '.[][] | select(. != null)' | sort | uniq -c
```

**EKS-specific limits:**
```bash
# EKS cluster limits
aws service-quotas get-service-quota --service-code eks --quota-code L-1194D53C  # Clusters per region
aws service-quotas get-service-quota --service-code eks --quota-code L-6D54EA21  # Managed node groups per cluster
aws service-quotas get-service-quota --service-code eks --quota-code L-CD136C55  # Nodes per managed node group
```

#### 7.5.2 Proactive Limit Monitoring

**Service limit monitoring:**
```yaml
# CloudWatch custom metrics for service limits
apiVersion: batch/v1
kind: CronJob
metadata:
  name: service-limit-monitor
spec:
  schedule: "0 */6 * * *"  # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: limit-monitor
            image: amazon/aws-cli:latest
            command:
            - /bin/bash
            - -c
            - |
              # Get current EC2 usage
              RUNNING_INSTANCES=$(aws ec2 describe-instances --filters "Name=instance-state-name,Values=running" --query 'Reservations[*].Instances[*].InstanceId' --output text | wc -w)
              
              # Get EC2 limit
              EC2_LIMIT=$(aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A --query 'Quota.Value' --output text)
              
              # Calculate utilization percentage
              UTILIZATION=$(echo "scale=2; $RUNNING_INSTANCES / $EC2_LIMIT * 100" | bc)
              
              # Send to CloudWatch
              aws cloudwatch put-metric-data \
                --namespace "AWS/ServiceLimits" \
                --metric-data MetricName=EC2InstanceUtilization,Value=$UTILIZATION,Unit=Percent
              
              echo "EC2 utilization: $UTILIZATION%"
          restartPolicy: OnFailure
```

---

### 7.6 Scaling Troubleshooting Runbooks

#### 7.6.1 "HPA not scaling" Runbook

**Symptoms:** Pods under load but HPA not creating more replicas

**Step 1: Check HPA status**
```bash
kubectl get hpa
kubectl describe hpa <hpa-name>
# Look for: current metrics, scaling events
```

**Step 2: Verify metrics availability**
```bash
# Check if metrics server is working
kubectl top pods
kubectl top nodes

# Check specific pod metrics
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/namespaces/<namespace>/pods/<pod-name>"
```

**Step 3: Check resource requests**
```bash
kubectl describe deployment <deployment-name> | grep -A 5 "Requests:"
# HPA requires CPU/memory requests to be set
```

**Step 4: Check node capacity**
```bash
kubectl describe nodes | grep -A 5 "Allocated resources"
# Verify nodes have capacity for new pods
```

#### 7.6.2 "Cluster Autoscaler not adding nodes" Runbook

**Symptoms:** Pods pending but no new nodes being created

**Step 1: Check CA logs**
```bash
kubectl -n kube-system logs deployment/cluster-autoscaler | tail -50
# Look for: scale-up events, errors, AWS API issues
```

**Step 2: Check pending pods**
```bash
kubectl get pods -A --field-selector=status.phase=Pending
kubectl describe pod <pending-pod>
# Look for: resource requirements, node selector constraints
```

**Step 3: Check node group limits**
```bash
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name>
# Check: min/max size, desired capacity, current instances
```

**Step 4: Check AWS service limits**
```bash
aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A
# Verify you haven't hit EC2 instance limits
```

#### 7.6.3 "Performance degraded under load" Runbook

**Symptoms:** Application slow during traffic spikes

**Step 1: Check resource utilization**
```bash
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory
kubectl top nodes
```

**Step 2: Check for CPU throttling**
```bash
# Look for throttling in Prometheus
# Query: sum(rate(container_cpu_cfs_throttled_seconds_total[5m])) by (namespace, pod)
```

**Step 3: Check network limits**
```bash
# Check AWS ENA metrics for network limits
kubectl exec node-exporter-pod -- cat /sys/class/net/eth0/statistics/rx_dropped
kubectl exec node-exporter-pod -- cat /sys/class/net/eth0/statistics/tx_dropped
```

**Step 4: Check DNS performance**
```bash
# Test DNS resolution speed
kubectl exec test-pod -- time nslookup kubernetes.default.svc.cluster.local

# Check CoreDNS metrics
kubectl -n kube-system logs -l k8s-app=kube-dns | grep -i error
```

---

## 8. Upgrades and maintenance

EKS upgrades are where "everything was working fine" becomes "production is down". Unlike app deploys, cluster upgrades touch every layer simultaneously and can fail in ways that are hard to predict and harder to roll back. Below: strategies that minimize risk and how to recover when upgrades go sideways.

---

### 8.1 EKS Upgrade Strategy (The Reality of Breaking Changes)

#### 8.1.1 EKS Upgrade Components

**What actually gets upgraded:**
```
[EKS Control Plane] → [Managed by AWS]
[EKS Add-ons] → [CoreDNS, kube-proxy, VPC CNI, EBS CSI]
[Node Groups] → [AMI, Kubernetes version, instance types]
[Third-party Components] → [Ingress controllers, service mesh, monitoring]
```

Each component can break independently, and version skew between components creates new failure modes.

#### 8.1.2 Pre-Upgrade Validation Checklist

**Compatibility matrix validation:**
```bash
#!/bin/bash
# EKS upgrade compatibility checker

CLUSTER_NAME="production-cluster"
CURRENT_VERSION=$(aws eks describe-cluster --name $CLUSTER_NAME --query 'cluster.version' --output text)
TARGET_VERSION="1.28"

echo "Current EKS version: $CURRENT_VERSION"
echo "Target EKS version: $TARGET_VERSION"

# Check addon compatibility
echo "Checking addon versions..."
aws eks describe-addon-versions --kubernetes-version $TARGET_VERSION --addon-name vpc-cni
aws eks describe-addon-versions --kubernetes-version $TARGET_VERSION --addon-name coredns
aws eks describe-addon-versions --kubernetes-version $TARGET_VERSION --addon-name kube-proxy

# Check deprecated APIs
echo "Checking for deprecated APIs..."
kubectl get --raw /api/v1 | jq '.resources[] | select(.name == "componentstatuses")' 
kubectl get --raw /apis/extensions/v1beta1 2>/dev/null || echo "extensions/v1beta1 not available (good)"

# Check node group versions
echo "Current node group versions:"
aws eks describe-nodegroup --cluster-name $CLUSTER_NAME --nodegroup-name primary --query 'nodegroup.version'
```

**Workload compatibility testing:**
```yaml
# Test job to validate workloads on new version
apiVersion: batch/v1
kind: Job
metadata:
  name: upgrade-compatibility-test
spec:
  template:
    spec:
      containers:
      - name: test
        image: kubectl:latest
        command:
        - /bin/bash
        - -c
        - |
          # Test basic functionality
          kubectl get nodes
          kubectl get pods -A
          
          # Test service discovery
          nslookup kubernetes.default.svc.cluster.local
          
          # Test storage
          kubectl get pvc -A
          kubectl get pv
          
          # Test networking
          kubectl get svc -A
          kubectl get ingress -A
          
          echo "Compatibility test completed"
      restartPolicy: Never
```

#### 8.1.3 Staged Upgrade Approach

**Phase 1: Control plane upgrade**
```bash
# Upgrade control plane first (managed by AWS)
aws eks update-cluster-version --name production-cluster --version 1.28

# Monitor upgrade progress
aws eks describe-update --name production-cluster --update-id <update-id>

# Validate control plane health
kubectl get nodes
kubectl get pods -n kube-system
```

**Phase 2: Add-on upgrades**
```bash
# Upgrade VPC CNI first (networking critical)
aws eks update-addon --cluster-name production-cluster --addon-name vpc-cni --addon-version v1.15.1-eksbuild.1

# Upgrade CoreDNS
aws eks update-addon --cluster-name production-cluster --addon-name coredns --addon-version v1.10.1-eksbuild.4

# Upgrade kube-proxy
aws eks update-addon --cluster-name production-cluster --addon-name kube-proxy --addon-version v1.28.2-eksbuild.2
```

**Phase 3: Node group upgrades (most risky)**
```bash
# Create new node group with new version
aws eks create-nodegroup \
  --cluster-name production-cluster \
  --nodegroup-name primary-v128 \
  --kubernetes-version 1.28 \
  --node-role arn:aws:iam::123456789012:role/NodeInstanceRole \
  --subnets subnet-12345 subnet-67890 \
  --instance-types m5.large \
  --ami-type AL2_x86_64 \
  --capacity-type ON_DEMAND \
  --scaling-config minSize=2,maxSize=10,desiredSize=3

# Gradually migrate workloads
kubectl cordon <old-node>
kubectl drain <old-node> --ignore-daemonsets --delete-emptydir-data --force --grace-period=300

# Delete old node group after validation
aws eks delete-nodegroup --cluster-name production-cluster --nodegroup-name primary-old
```

---

### 8.2 Node Group Replacement Strategies

#### 8.2.1 Blue-Green Node Group Strategy

**Advantages:**
- Zero downtime for stateless workloads
- Easy rollback if issues occur
- Full validation before switching traffic

**Implementation:**
```yaml
# Blue node group (current)
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: production-cluster
nodeGroups:
- name: blue-nodes
  instanceTypes: ["m5.large"]
  minSize: 3
  maxSize: 10
  desiredCapacity: 5
  labels:
    deployment-group: blue
  tags:
    Environment: production
    DeploymentGroup: blue

# Green node group (new version)
- name: green-nodes
  instanceTypes: ["m5.large"]
  minSize: 3
  maxSize: 10
  desiredCapacity: 5
  labels:
    deployment-group: green
  tags:
    Environment: production
    DeploymentGroup: green
```

**Migration process:**
```bash
# 1. Create green node group
eksctl create nodegroup --config-file=cluster-config.yaml --include="green-nodes"

# 2. Validate green nodes
kubectl get nodes -l deployment-group=green
kubectl describe nodes -l deployment-group=green

# 3. Migrate workloads gradually
for deployment in $(kubectl get deployments -o name); do
  echo "Migrating $deployment"
  kubectl patch $deployment -p '{"spec":{"template":{"spec":{"nodeSelector":{"deployment-group":"green"}}}}}'
  kubectl rollout status $deployment
  sleep 30
done

# 4. Validate applications on green nodes
./run-smoke-tests.sh

# 5. Remove blue nodes
kubectl cordon -l deployment-group=blue
kubectl drain -l deployment-group=blue --ignore-daemonsets --delete-emptydir-data
eksctl delete nodegroup --cluster=production-cluster --name=blue-nodes
```

#### 8.2.2 Rolling Node Group Updates

**For stateful workloads that can't move easily:**
```bash
# Update node group in place with rolling replacement
aws eks update-nodegroup-version \
  --cluster-name production-cluster \
  --nodegroup-name primary \
  --kubernetes-version 1.28 \
  --launch-template-version 2

# Monitor rolling update progress
aws eks describe-nodegroup \
  --cluster-name production-cluster \
  --nodegroup-name primary \
  --query 'nodegroup.updateConfig'
```

**Custom rolling update script:**
```bash
#!/bin/bash
# Custom node rolling update with validation

CLUSTER_NAME="production-cluster"
NODEGROUP_NAME="primary"

# Get list of nodes in node group
NODES=$(kubectl get nodes -l eks.amazonaws.com/nodegroup=$NODEGROUP_NAME -o jsonpath='{.items[*].metadata.name}')

for node in $NODES; do
  echo "Updating node: $node"
  
  # Cordon node
  kubectl cordon $node
  
  # Wait for new pods to be scheduled elsewhere
  sleep 60
  
  # Drain node
  kubectl drain $node --ignore-daemonsets --delete-emptydir-data --force --grace-period=300
  
  # Terminate instance (ASG will replace it)
  INSTANCE_ID=$(kubectl get node $node -o jsonpath='{.spec.providerID}' | cut -d'/' -f5)
  aws ec2 terminate-instances --instance-ids $INSTANCE_ID
  
  # Wait for replacement node to be ready
  echo "Waiting for replacement node..."
  while true; do
    READY_NODES=$(kubectl get nodes -l eks.amazonaws.com/nodegroup=$NODEGROUP_NAME --no-headers | grep " Ready " | wc -l)
    if [ $READY_NODES -ge $(echo $NODES | wc -w) ]; then
      break
    fi
    sleep 30
  done
  
  echo "Node $node replaced successfully"
done
```

---

### 8.3 Application Compatibility and API Deprecations

#### 8.3.1 Deprecated API Detection

**Automated API deprecation scanning:**
```bash
#!/bin/bash
# Scan for deprecated APIs in cluster

echo "Scanning for deprecated APIs..."

# Check for deprecated APIs in running resources
kubectl get --raw /api/v1 | jq -r '.resources[] | select(.name | contains("componentstatuses")) | .name'

# Check extensions/v1beta1 usage (deprecated in 1.22+)
kubectl get deployments.extensions -A 2>/dev/null && echo "WARNING: Found extensions/v1beta1 Deployments"
kubectl get ingresses.extensions -A 2>/dev/null && echo "WARNING: Found extensions/v1beta1 Ingresses"

# Check networking.k8s.io/v1beta1 usage (deprecated in 1.22+)
kubectl get ingresses.networking.k8s.io/v1beta1 -A 2>/dev/null && echo "WARNING: Found networking.k8s.io/v1beta1 Ingresses"

# Check policy/v1beta1 usage (deprecated in 1.25+)
kubectl get podsecuritypolicies 2>/dev/null && echo "WARNING: Found PodSecurityPolicies (deprecated)"

# Check autoscaling/v2beta1 usage (deprecated in 1.23+)
kubectl get hpa.autoscaling/v2beta1 -A 2>/dev/null && echo "WARNING: Found autoscaling/v2beta1 HPAs"

echo "Deprecated API scan completed"
```

**Pluto for comprehensive deprecation checking:**
```bash
# Install pluto
curl -L https://github.com/FairwindsOps/pluto/releases/download/v5.18.4/pluto_5.18.4_linux_amd64.tar.gz | tar xz
sudo mv pluto /usr/local/bin/

# Scan cluster for deprecated APIs
pluto detect-all-in-cluster --target-versions k8s=v1.28.0

# Scan Helm releases
pluto detect-helm --target-versions k8s=v1.28.0

# Scan files
pluto detect-files -d ./k8s-manifests --target-versions k8s=v1.28.0
```

#### 8.3.2 API Migration Strategies

**Ingress API migration (extensions/v1beta1 → networking.k8s.io/v1):**
```yaml
# OLD (deprecated in 1.22+)
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: web-app
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: web-app
          servicePort: 80

# NEW (required in 1.22+)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix  # Required field
        backend:
          service:        # Changed structure
            name: web-app
            port:
              number: 80
```

**HPA API migration (autoscaling/v2beta1 → autoscaling/v2):**
```yaml
# OLD (deprecated in 1.23+)
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 70

# NEW (required in 1.23+)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Changed structure
```

---

### 8.4 Rollback Strategies

#### 8.4.1 Control Plane Rollback Limitations

**Critical understanding:** EKS control plane upgrades **cannot be rolled back** (as of early 2026). Once upgraded, you can only move forward.

> **Note:** AWS is developing an EKS control plane rollback feature for inline upgrades, but it's not yet released. Until available, the limitations below apply.

**Current mitigation strategies:**
1. **Thorough testing** in staging environment
2. **Blue-green cluster** strategy for critical workloads
3. **Backup and restore** procedures for etcd data

#### 8.4.2 Node Group Rollback

**Quick node group rollback:**
```bash
# If new node group has issues, switch back to old one
kubectl patch deployment web-app -p '{"spec":{"template":{"spec":{"nodeSelector":{"deployment-group":"blue"}}}}}'

# Scale up old node group
aws eks update-nodegroup-config \
  --cluster-name production-cluster \
  --nodegroup-name blue-nodes \
  --scaling-config minSize=3,maxSize=10,desiredSize=5

# Delete problematic new node group
aws eks delete-nodegroup \
  --cluster-name production-cluster \
  --nodegroup-name green-nodes
```

#### 8.4.3 Add-on Rollback

**Rolling back EKS add-ons:**
```bash
# Check available versions
aws eks describe-addon-versions --addon-name vpc-cni --kubernetes-version 1.27

# Rollback to previous version
aws eks update-addon \
  --cluster-name production-cluster \
  --addon-name vpc-cni \
  --addon-version v1.14.1-eksbuild.1 \
  --resolve-conflicts OVERWRITE
```

---

### 8.5 Maintenance Windows and Disruption Management

#### 8.5.1 Planned Maintenance Strategy

**Maintenance window planning:**
```yaml
# Maintenance mode deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: maintenance-page
spec:
  replicas: 2
  selector:
    matchLabels:
      app: maintenance-page
  template:
    metadata:
      labels:
        app: maintenance-page
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: maintenance-content
          mountPath: /usr/share/nginx/html
      volumes:
      - name: maintenance-content
        configMap:
          name: maintenance-content
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: maintenance-content
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>Maintenance</title></head>
    <body>
      <h1>System Maintenance</h1>
      <p>We're performing scheduled maintenance. Please try again in 30 minutes.</p>
    </body>
    </html>
```

**Traffic switching for maintenance:**
```bash
# Switch ingress to maintenance page
kubectl patch ingress web-app -p '{"spec":{"rules":[{"host":"app.example.com","http":{"paths":[{"path":"/","pathType":"Prefix","backend":{"service":{"name":"maintenance-page","port":{"number":80}}}}]}}]}}'

# Perform maintenance operations
./upgrade-cluster.sh

# Switch back to application
kubectl patch ingress web-app -p '{"spec":{"rules":[{"host":"app.example.com","http":{"paths":[{"path":"/","pathType":"Prefix","backend":{"service":{"name":"web-app","port":{"number":80}}}}]}}]}}'
```

#### 8.5.2 Pod Disruption Budget Management

**Maintenance-aware PDB configuration:**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web-app
  # Allow more disruption during maintenance windows
  unhealthyPodEvictionPolicy: AlwaysAllow
```

**Temporary PDB adjustment for maintenance:**
```bash
# Relax PDB for maintenance
kubectl patch pdb web-app-pdb -p '{"spec":{"minAvailable":1}}'

# Perform node drains
kubectl drain node1 --ignore-daemonsets --delete-emptydir-data

# Restore strict PDB after maintenance
kubectl patch pdb web-app-pdb -p '{"spec":{"minAvailable":2}}'
```

---

### 8.6 Upgrade Troubleshooting Runbooks

#### 8.6.1 "Control plane upgrade stuck" Runbook

**Symptoms:** EKS upgrade shows "InProgress" for hours

**Step 1: Check upgrade status**
```bash
aws eks describe-update --name production-cluster --update-id <update-id>
# Look for: status, errors, created/modified timestamps
```

**Step 2: Check control plane health**
```bash
kubectl get nodes
kubectl get pods -n kube-system
kubectl get --raw='/readyz?verbose'
```

**Step 3: Check for blocking resources**
```bash
# Check for stuck finalizers
kubectl get all -A | grep Terminating

# Check for webhook issues
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
```

**Step 4: Contact AWS Support**
```bash
# If upgrade is truly stuck (>4 hours), open AWS support case
# Include: cluster name, update ID, timeline of events
```

#### 8.6.2 "Pods failing after node upgrade" Runbook

**Symptoms:** Applications not working after node group upgrade

**Step 1: Check pod status**
```bash
kubectl get pods -A | grep -v Running
kubectl describe pod <failing-pod>
# Look for: scheduling issues, image pull problems, volume mount failures
```

**Step 2: Check node conditions**
```bash
kubectl get nodes
kubectl describe node <new-node>
# Look for: Ready condition, resource availability, taints
```

**Step 3: Check networking**
```bash
# Test pod-to-pod connectivity
kubectl exec test-pod -- ping <other-pod-ip>

# Test DNS resolution
kubectl exec test-pod -- nslookup kubernetes.default.svc.cluster.local

# Check CNI health
kubectl -n kube-system logs -l k8s-app=aws-node
```

**Step 4: Check storage**
```bash
# Check PVC status
kubectl get pvc -A

# Check volume attachments
kubectl get volumeattachment

# Check CSI driver health
kubectl -n kube-system logs -l app=ebs-csi-controller
```

#### 8.6.3 "Add-on upgrade failed" Runbook

**Symptoms:** EKS add-on shows "DEGRADED" status

**Step 1: Check add-on status**
```bash
aws eks describe-addon --cluster-name production-cluster --addon-name vpc-cni
# Look for: status, health issues, configuration conflicts
```

**Step 2: Check add-on pods**
```bash
kubectl -n kube-system get pods -l k8s-app=aws-node
kubectl -n kube-system logs -l k8s-app=aws-node
```

**Step 3: Resolve conflicts**
```bash
# If configuration conflicts exist, resolve with OVERWRITE
aws eks update-addon \
  --cluster-name production-cluster \
  --addon-name vpc-cni \
  --resolve-conflicts OVERWRITE
```

**Step 4: Rollback if necessary**
```bash
# Check available versions
aws eks describe-addon-versions --addon-name vpc-cni --kubernetes-version 1.27

# Rollback to previous version
aws eks update-addon \
  --cluster-name production-cluster \
  --addon-name vpc-cni \
  --addon-version <previous-version>
```

---

### 8.7 Graceful Deployments and Pod Termination

**The deployment problem:** During rolling updates, pods can receive traffic while terminating or before they're ready, causing 5xx errors.

**Pod termination sequence:**
```
1. Pod marked for termination (status: Terminating)
2. Pod removed from Service endpoints (async)
3. SIGTERM sent to containers (async)
4. preStop hooks executed (if configured)
5. Grace period countdown starts (default: 30s)
6. SIGKILL sent if still running after grace period
```

**Race condition:** Steps 2 and 3 happen in parallel, so pods can receive traffic after SIGTERM.

#### 8.7.1 Graceful Shutdown Configuration

**Application-level graceful shutdown:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: graceful-app
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 60  # Allow time for cleanup
      containers:
      - name: app
        image: my-app:latest
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                # Signal app to stop accepting new requests
                kill -TERM 1
                # Wait for load balancer to update (AWS NLB ~10s, ALB ~15s)
                sleep 15
                # Allow existing requests to complete
                sleep 10
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
```

**Application code example (Go):**
```go
package main

import (
    "context"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    server := &http.Server{Addr: ":8080"}
    
    // Graceful shutdown handling
    go func() {
        sigChan := make(chan os.Signal, 1)
        signal.Notify(sigChan, syscall.SIGTERM, syscall.SIGINT)
        <-sigChan
        
        // Stop accepting new requests
        ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        defer cancel()
        
        server.Shutdown(ctx)
    }()
    
    server.ListenAndServe()
}
```

#### 8.7.2 Load Balancer Integration

**AWS NLB connection draining:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "350"
    # Enable connection draining
    service.beta.kubernetes.io/aws-load-balancer-connection-draining-enabled: "true"
    service.beta.kubernetes.io/aws-load-balancer-connection-draining-timeout: "60"
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: graceful-app
```

**Envoy proxy graceful shutdown:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-config
data:
  envoy.yaml: |
    static_resources:
      listeners:
      - name: listener_0
        address:
          socket_address:
            address: 0.0.0.0
            port_value: 8080
        filter_chains:
        - filters:
          - name: envoy.filters.network.http_connection_manager
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
              # Graceful shutdown settings
              drain_timeout: 30s
              delayed_close_timeout: 10s
              http_filters:
              - name: envoy.filters.http.router
              route_config:
                name: local_route
                virtual_hosts:
                - name: backend
                  domains: ["*"]
                  routes:
                  - match:
                      prefix: "/"
                    route:
                      cluster: backend_cluster
      clusters:
      - name: backend_cluster
        connect_timeout: 5s
        type: STRICT_DNS
        lb_policy: ROUND_ROBIN
        # Connection recycling for NLB compatibility
        max_requests_per_connection: 1000
        max_connection_duration: 300s
        load_assignment:
          cluster_name: backend_cluster
          endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: app-backend
                    port_value: 8080
```

#### 8.7.3 Rolling Update Strategy

**Deployment strategy for zero-downtime updates:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zero-downtime-app
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1        # Only terminate 1 pod at a time
      maxSurge: 2              # Allow 2 extra pods during update
  template:
    spec:
      terminationGracePeriodSeconds: 45
      containers:
      - name: app
        image: my-app:v2
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        lifecycle:
          preStop:
            httpGet:
              path: /shutdown
              port: 8080
```

**PodDisruptionBudget for controlled disruptions:**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 4  # Always keep 4 pods running
  selector:
    matchLabels:
      app: zero-downtime-app
```


## 9. Disaster recovery

When a cluster fails catastrophically, the pressure to restore service leads to rushed decisions that make recovery slower. Below: DR strategies that work under pressure — what can be recovered, what must be rebuilt, and how to not make it worse.

---

### 9.1 EKS Disaster Scenarios (What Actually Breaks)

#### 9.1.1 Control Plane Failures

**AWS-managed control plane issues:**
- Regional AWS service outages
- EKS API server degradation
- etcd corruption (rare but catastrophic)
- Certificate rotation failures

**Reality check:** You cannot directly access or repair the EKS control plane. Recovery depends entirely on AWS support and your backup strategies.

#### 9.1.2 Complete Cluster Loss

**Common causes:**
- Accidental cluster deletion
- VPC/networking misconfiguration making cluster unreachable
- All node groups terminated simultaneously
- Region-wide AWS outages

**Recovery time expectations:**
- New cluster provisioning: 10-15 minutes
- Add-on installation: 5-10 minutes
- Application restoration: Depends on backup strategy
- **Total RTO: 30 minutes to several hours**

#### 9.1.3 Data Layer Failures

**EBS volume failures:**
- Zone-wide EBS outages
- Volume corruption
- Snapshot restoration issues

**Application data loss:**
- StatefulSet data corruption
- Database failures
- Persistent volume claim issues

---

### 9.2 Backup Strategies (What to Backup and How)

#### 9.2.1 Cluster Configuration Backup

**Essential cluster state to backup:**
```bash
#!/bin/bash
# Cluster backup script

CLUSTER_NAME="production-cluster"
BACKUP_DIR="./cluster-backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p $BACKUP_DIR

# Backup cluster configuration
aws eks describe-cluster --name $CLUSTER_NAME > $BACKUP_DIR/cluster-config.json

# Backup node groups
aws eks list-nodegroups --cluster-name $CLUSTER_NAME --query 'nodegroups[]' --output text | \
while read nodegroup; do
  aws eks describe-nodegroup --cluster-name $CLUSTER_NAME --nodegroup-name $nodegroup > $BACKUP_DIR/nodegroup-$nodegroup.json
done

# Backup EKS add-ons
aws eks list-addons --cluster-name $CLUSTER_NAME --query 'addons[]' --output text | \
while read addon; do
  aws eks describe-addon --cluster-name $CLUSTER_NAME --addon-name $addon > $BACKUP_DIR/addon-$addon.json
done

# Backup VPC configuration
VPC_ID=$(aws eks describe-cluster --name $CLUSTER_NAME --query 'cluster.resourcesVpcConfig.vpcId' --output text)
aws ec2 describe-vpcs --vpc-ids $VPC_ID > $BACKUP_DIR/vpc-config.json
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" > $BACKUP_DIR/subnets.json
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" > $BACKUP_DIR/security-groups.json

echo "Cluster configuration backed up to $BACKUP_DIR"
```

#### 9.2.2 Application State Backup with Velero

**Velero installation for EKS:**
```bash
# Install Velero with AWS plugin
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket velero-backups-production \
  --backup-location-config region=us-west-2 \
  --snapshot-location-config region=us-west-2 \
  --secret-file ./credentials-velero
```

**Comprehensive backup schedule:**
```yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  template:
    includedNamespaces:
    - production
    - staging
    excludedResources:
    - events
    - events.events.k8s.io
    storageLocation: default
    volumeSnapshotLocations:
    - default
    ttl: 720h  # 30 days retention
```

**Critical workload backup:**
```yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: critical-workloads
spec:
  includedNamespaces:
  - production
  labelSelector:
    matchLabels:
      backup: critical
  snapshotVolumes: true
  includeClusterResources: true
  hooks:
    resources:
    - name: database-backup-hook
      includedNamespaces:
      - production
      labelSelector:
        matchLabels:
          app: database
      pre:
      - exec:
          container: database
          command:
          - /bin/bash
          - -c
          - "pg_dump -h localhost -U postgres mydb > /tmp/backup.sql"
          timeout: 300s
```

#### 9.2.3 etcd Backup Strategy

**Automated etcd backup (for self-managed clusters):**
```bash
#!/bin/bash
# etcd backup script (not applicable to EKS managed control plane)
# This is for reference if you have self-managed etcd

ETCDCTL_API=3 etcdctl snapshot save backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

# Upload to S3
aws s3 cp backup.db s3://etcd-backups/backup-$(date +%Y%m%d-%H%M%S).db
```

**Note:** EKS manages etcd backups automatically. You cannot directly backup EKS etcd.

---

### 9.3 Cross-Region Disaster Recovery

#### 9.3.1 Multi-Region EKS Architecture

**Active-passive setup:**
```yaml
# Primary region cluster
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: production-primary
  region: us-west-2
nodeGroups:
- name: primary-nodes
  instanceTypes: ["m5.large"]
  minSize: 3
  maxSize: 10
  desiredCapacity: 5
  availabilityZones: ["us-west-2a", "us-west-2b", "us-west-2c"]

---
# DR region cluster (smaller, can be scaled up)
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: production-dr
  region: us-east-1
nodeGroups:
- name: dr-nodes
  instanceTypes: ["m5.large"]
  minSize: 1
  maxSize: 10
  desiredCapacity: 2
  availabilityZones: ["us-east-1a", "us-east-1b", "us-east-1c"]
```

#### 9.3.2 Cross-Region Replication Strategy

**Database replication:**
```yaml
# RDS cross-region read replica
apiVersion: v1
kind: ConfigMap
metadata:
  name: database-config
data:
  primary-endpoint: "prod-db.us-west-2.rds.amazonaws.com"
  dr-endpoint: "prod-db-replica.us-east-1.rds.amazonaws.com"
  failover-script: |
    #!/bin/bash
    # Promote read replica to primary
    aws rds promote-read-replica \
      --db-instance-identifier prod-db-replica \
      --region us-east-1
```

**Application data replication:**
```bash
# Cross-region S3 replication for application assets
aws s3api put-bucket-replication \
  --bucket production-assets \
  --replication-configuration file://replication-config.json
```

#### 9.3.3 DNS Failover Configuration

**Route 53 health checks and failover:**
```json
{
  "Type": "A",
  "Name": "api.example.com",
  "SetIdentifier": "primary",
  "Failover": "PRIMARY",
  "AliasTarget": {
    "DNSName": "k8s-elb-primary.us-west-2.elb.amazonaws.com",
    "EvaluateTargetHealth": true
  },
  "HealthCheckId": "primary-health-check"
}
```

---

### 9.4 Recovery Procedures

#### 9.4.1 Complete Cluster Recreation

**Cluster recreation runbook:**
```bash
#!/bin/bash
# Complete cluster recovery procedure

set -e

CLUSTER_NAME="production-cluster"
REGION="us-west-2"
BACKUP_DIR="./latest-backup"

echo "Starting cluster recovery for $CLUSTER_NAME"

# Step 1: Recreate cluster
eksctl create cluster --config-file=$BACKUP_DIR/cluster-config.yaml

# Step 2: Wait for cluster to be ready
aws eks wait cluster-active --name $CLUSTER_NAME --region $REGION

# Step 3: Install essential add-ons
aws eks create-addon --cluster-name $CLUSTER_NAME --addon-name vpc-cni
aws eks create-addon --cluster-name $CLUSTER_NAME --addon-name coredns
aws eks create-addon --cluster-name $CLUSTER_NAME --addon-name kube-proxy

# Step 4: Install AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$CLUSTER_NAME

# Step 5: Install Velero
velero install --provider aws --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket velero-backups-production \
  --backup-location-config region=$REGION \
  --snapshot-location-config region=$REGION \
  --secret-file ./credentials-velero

# Step 6: Restore from backup
LATEST_BACKUP=$(velero backup get --output json | jq -r '.items[0].metadata.name')
velero restore create --from-backup $LATEST_BACKUP

echo "Cluster recovery initiated. Monitor with: kubectl get pods -A"
```

#### 9.4.2 Partial Recovery Scenarios

**Node group replacement:**
```bash
# If only node groups are affected
aws eks create-nodegroup \
  --cluster-name production-cluster \
  --nodegroup-name recovery-nodes \
  --kubernetes-version 1.28 \
  --node-role arn:aws:iam::123456789012:role/NodeInstanceRole \
  --subnets subnet-12345 subnet-67890 \
  --instance-types m5.large \
  --scaling-config minSize=3,maxSize=10,desiredSize=5

# Migrate workloads to new nodes
kubectl cordon -l eks.amazonaws.com/nodegroup=old-nodes
kubectl drain -l eks.amazonaws.com/nodegroup=old-nodes --ignore-daemonsets --delete-emptydir-data
```

**Application-only recovery:**
```bash
# If cluster is healthy but applications are corrupted
velero restore create app-recovery \
  --from-backup latest-backup \
  --include-namespaces production \
  --restore-volumes=true
```

#### 9.4.3 Data Recovery Procedures

**EBS volume recovery:**
```bash
# Restore from EBS snapshot
SNAPSHOT_ID="snap-1234567890abcdef0"
VOLUME_ID=$(aws ec2 create-volume \
  --snapshot-id $SNAPSHOT_ID \
  --availability-zone us-west-2a \
  --volume-type gp3 \
  --query 'VolumeId' --output text)

# Update PV to use new volume
kubectl patch pv pvc-12345 -p '{"spec":{"awsElasticBlockStore":{"volumeID":"'$VOLUME_ID'"}}}'
```

**Database recovery:**
```bash
# RDS point-in-time recovery
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-db \
  --target-db-instance-identifier prod-db-recovered \
  --restore-time 2024-01-15T10:00:00.000Z
```

---

### 9.5 Recovery Testing and Validation

#### 9.5.1 Disaster Recovery Testing Schedule

**Monthly DR drill:**
```bash
#!/bin/bash
# DR drill script - run in non-production environment

echo "Starting DR drill $(date)"

# Test 1: Backup restoration
velero restore create dr-test-$(date +%Y%m%d) \
  --from-backup latest-production-backup \
  --namespace-mappings production:dr-test

# Test 2: Application functionality
kubectl -n dr-test run test-pod --image=curlimages/curl --rm -it -- \
  curl http://web-app.dr-test.svc.cluster.local/health

# Test 3: Database connectivity
kubectl -n dr-test exec deployment/app -- \
  pg_isready -h database.dr-test.svc.cluster.local

# Test 4: External dependencies
kubectl -n dr-test exec deployment/app -- \
  curl -f https://api.external-service.com/health

echo "DR drill completed. Check results manually."
```

#### 9.5.2 Recovery Time Objective (RTO) Validation

**RTO measurement script:**
```bash
#!/bin/bash
# Measure actual recovery times

START_TIME=$(date +%s)

# Simulate cluster failure
kubectl delete deployment --all -n production

# Start recovery
velero restore create rto-test --from-backup latest-backup

# Wait for recovery completion
while true; do
  READY_PODS=$(kubectl get pods -n production --no-headers | grep Running | wc -l)
  TOTAL_PODS=$(kubectl get pods -n production --no-headers | wc -l)
  
  if [ $READY_PODS -eq $TOTAL_PODS ] && [ $TOTAL_PODS -gt 0 ]; then
    break
  fi
  
  sleep 10
done

END_TIME=$(date +%s)
RTO=$((END_TIME - START_TIME))

echo "Recovery completed in $RTO seconds"
echo "RTO target: 1800 seconds (30 minutes)"

if [ $RTO -lt 1800 ]; then
  echo "✅ RTO target met"
else
  echo "❌ RTO target exceeded"
fi
```

---

### 9.6 Disaster Recovery Runbooks

#### 9.6.1 "Complete cluster loss" Runbook

**Symptoms:** Cannot connect to cluster, AWS console shows cluster deleted/unavailable

**Step 1: Assess scope**
```bash
# Check if cluster exists
aws eks describe-cluster --name production-cluster

# Check if it's a regional AWS issue
curl -s https://status.aws.amazon.com/ | grep -i "service issues"
```

**Step 2: Activate DR procedures**
```bash
# Switch DNS to DR region (if available)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456789 \
  --change-batch file://failover-to-dr.json

# Scale up DR cluster
aws eks update-nodegroup-config \
  --cluster-name production-dr \
  --nodegroup-name dr-nodes \
  --scaling-config minSize=3,maxSize=20,desiredSize=10
```

**Step 3: Recreate primary cluster**
```bash
# Use backup configuration
eksctl create cluster --config-file=./backups/cluster-config.yaml

# Restore applications
velero restore create disaster-recovery \
  --from-backup $(velero backup get -o json | jq -r '.items[0].metadata.name')
```

#### 9.6.2 "Data corruption" Runbook

**Symptoms:** Applications running but data is corrupted/missing

**Step 1: Stop writes immediately**
```bash
# Scale down applications to prevent further corruption
kubectl scale deployment --replicas=0 -n production -l tier=application

# Cordon nodes to prevent new pods
kubectl cordon --all
```

**Step 2: Assess data integrity**
```bash
# Check database consistency
kubectl exec -it database-pod -- pg_dump --schema-only mydb > schema-backup.sql

# Check persistent volume data
kubectl exec -it app-pod -- find /data -name "*.log" -mtime -1 | head -10
```

**Step 3: Restore from backup**
```bash
# Restore database from point-in-time backup
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-db \
  --target-db-instance-identifier prod-db-restored \
  --restore-time $(date -d '1 hour ago' -Iseconds)

# Restore application data from Velero
velero restore create data-recovery \
  --from-backup latest-backup \
  --include-resources persistentvolumeclaims,persistentvolumes
```

#### 9.6.3 "Region-wide outage" Runbook

**Symptoms:** All AWS services in primary region unavailable

**Step 1: Immediate failover**
```bash
# Activate DR region immediately
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456789 \
  --change-batch file://emergency-failover.json

# Scale DR cluster to handle production load
kubectl scale deployment --replicas=5 -n production -l tier=web
kubectl scale deployment --replicas=3 -n production -l tier=api
```

**Step 2: Promote read replicas**
```bash
# Promote RDS read replica to primary
aws rds promote-read-replica \
  --db-instance-identifier prod-db-replica \
  --region us-east-1

# Update application configuration
kubectl patch configmap app-config -p '{"data":{"database_url":"prod-db-replica.us-east-1.rds.amazonaws.com"}}'
```

**Step 3: Monitor and adjust**
```bash
# Monitor application health in DR region
kubectl get pods -A | grep -v Running
kubectl top nodes
kubectl top pods -A --sort-by=cpu
```

---

## 10. Cost optimization

Every engineering decision here directly hits the budget. Below: the cost levers that actually matter in production EKS and how to pull them without breaking reliability.

---

### 10.1 EKS Cost Structure (Where Your Money Goes)

#### 10.1.1 EKS Cost Components

**Control plane costs:**
- EKS cluster: $0.10/hour per cluster ($73/month)
- Fargate: $0.04048/vCPU/hour + $0.004445/GB/hour
- Data transfer costs (often overlooked)

**Compute costs (largest component):**
- EC2 instances for node groups
- EBS volumes for node storage
- Data transfer between AZs
- NAT Gateway costs for private subnets

**Hidden costs:**
- Load balancer costs (ALB/NLB)
- EBS snapshots and backups
- CloudWatch logs and metrics
- Cross-AZ data transfer

#### 10.1.2 Cost Visibility and Tracking

**Essential cost tracking:**
```bash
# Get EKS cluster costs
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --filter file://eks-cost-filter.json

# EKS cost filter
cat > eks-cost-filter.json << EOF
{
  "Dimensions": {
    "Key": "SERVICE",
    "Values": ["Amazon Elastic Kubernetes Service", "Amazon Elastic Compute Cloud"]
  }
}
EOF
```

**Resource tagging for cost allocation:**
```yaml
apiVersion: v1
kind: Node
metadata:
  labels:
    cost-center: "engineering"
    environment: "production"
    team: "platform"
    project: "web-app"
```

---

### 10.2 Right-Sizing Workloads

#### 10.2.1 Resource Request Optimization

**The over-provisioning problem:**
```bash
# Find over-provisioned pods
kubectl top pods -A --sort-by=cpu | head -20
kubectl top pods -A --sort-by=memory | head -20

# Compare requests vs actual usage
kubectl get pods -A -o custom-columns=NAME:.metadata.name,NAMESPACE:.metadata.namespace,CPU_REQ:.spec.containers[*].resources.requests.cpu,MEM_REQ:.spec.containers[*].resources.requests.memory
```

**VPA for right-sizing recommendations:**
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  updatePolicy:
    updateMode: "Off"  # Recommendation only
  resourcePolicy:
    containerPolicies:
    - containerName: web-app
      maxAllowed:
        cpu: 2
        memory: 4Gi
      minAllowed:
        cpu: 100m
        memory: 128Mi
```

**Automated right-sizing script:**
```bash
#!/bin/bash
# Generate right-sizing recommendations

echo "Analyzing resource usage for right-sizing..."

for namespace in $(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'); do
  echo "Namespace: $namespace"
  
  kubectl top pods -n $namespace --no-headers | while read pod cpu memory; do
    # Get resource requests
    requests=$(kubectl get pod $pod -n $namespace -o jsonpath='{.spec.containers[*].resources.requests}')
    
    echo "Pod: $pod"
    echo "  Current usage: CPU=$cpu, Memory=$memory"
    echo "  Requests: $requests"
    echo "  Recommendation: Review if requests match usage"
    echo ""
  done
done
```

#### 10.2.2 Node Right-Sizing

**Instance type cost analysis:**
```bash
# Compare instance costs per vCPU and per GB RAM
aws ec2 describe-instance-types \
  --instance-types m5.large m5.xlarge m5.2xlarge c5.large c5.xlarge \
  --query 'InstanceTypes[*].[InstanceType,VCpuInfo.DefaultVCpus,MemoryInfo.SizeInMiB]' \
  --output table

# Get current pricing (requires AWS Pricing API)
aws pricing get-products \
  --service-code AmazonEC2 \
  --filters Type=TERM_MATCH,Field=instanceType,Value=m5.large \
  --filters Type=TERM_MATCH,Field=location,Value="US West (Oregon)"
```

**Node utilization analysis:**
```bash
# Check node resource utilization
kubectl top nodes

# Detailed node analysis
kubectl describe nodes | grep -A 5 "Allocated resources"

# Find underutilized nodes
kubectl get nodes -o custom-columns=NAME:.metadata.name,CPU_CAPACITY:.status.capacity.cpu,MEMORY_CAPACITY:.status.capacity.memory
```

---

### 10.3 Spot Instances and Mixed Instance Types

#### 10.3.1 Spot Instance Strategy

**Spot-optimized node group:**
```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: cost-optimized-cluster
nodeGroups:
- name: spot-nodes
  instanceTypes: 
  - m5.large
  - m5.xlarge
  - c5.large
  - c5.xlarge
  spot: true
  minSize: 2
  maxSize: 20
  desiredCapacity: 5
  labels:
    node-type: spot
  taints:
  - key: spot-instance
    value: "true"
    effect: NoSchedule
```

**Spot-tolerant workload configuration:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-processor
spec:
  replicas: 5
  template:
    spec:
      tolerations:
      - key: spot-instance
        operator: Equal
        value: "true"
        effect: NoSchedule
      nodeSelector:
        node-type: spot
      containers:
      - name: processor
        image: batch-processor:latest
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
```

#### 10.3.2 Mixed Instance Type Strategy

**Diversified node groups:**
```yaml
# On-demand for critical workloads
- name: on-demand-critical
  instanceTypes: ["m5.large"]
  minSize: 2
  maxSize: 5
  desiredCapacity: 2
  labels:
    node-type: on-demand
    workload-type: critical

# Spot for batch/stateless workloads  
- name: spot-batch
  instanceTypes: 
  - m5.large
  - m5.xlarge
  - c5.large
  - c5.xlarge
  spot: true
  minSize: 0
  maxSize: 50
  desiredCapacity: 5
  labels:
    node-type: spot
    workload-type: batch
```

**Workload placement strategy:**
```yaml
# Critical workloads on on-demand
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  template:
    spec:
      nodeSelector:
        node-type: on-demand
        workload-type: critical
      containers:
      - name: payment
        image: payment-service:latest

---
# Batch workloads on spot
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processing
spec:
  template:
    spec:
      tolerations:
      - key: spot-instance
        operator: Equal
        value: "true"
        effect: NoSchedule
      nodeSelector:
        node-type: spot
        workload-type: batch
```

---

### 10.4 Storage Cost Optimization

#### 10.4.1 EBS Volume Optimization

**Storage class cost comparison:**
```yaml
# gp3 (newer, more cost-effective)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-optimized
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"      # Baseline IOPS
  throughput: "125"  # Baseline throughput
allowVolumeExpansion: true
reclaimPolicy: Delete

# gp2 (legacy, more expensive for same performance)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2-legacy
provisioner: ebs.csi.aws.com
parameters:
  type: gp2
allowVolumeExpansion: true
reclaimPolicy: Delete
```

**Volume cleanup automation:**
```bash
#!/bin/bash
# Clean up unused EBS volumes

echo "Finding unused EBS volumes..."

# Get all EBS volumes
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[*].[VolumeId,CreateTime,Size,VolumeType]' \
  --output table

# Find volumes older than 30 days with no attachments
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query "Volumes[?CreateTime<='$(date -d '30 days ago' -Iseconds)'].[VolumeId,CreateTime,Size]" \
  --output table

echo "Review these volumes for deletion to reduce costs"
```

#### 10.4.2 Persistent Volume Reclaim Policies

**Cost-conscious reclaim policies:**
```yaml
# For development environments - Delete to avoid orphaned volumes
apiVersion: v1
kind: PersistentVolume
metadata:
  name: dev-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete  # Automatically delete when PVC is deleted
  storageClassName: gp3-optimized

# For production - Retain for safety, but monitor for cleanup
apiVersion: v1
kind: PersistentVolume
metadata:
  name: prod-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain  # Manual cleanup required
  storageClassName: gp3-optimized
```

---

### 10.5 Network Cost Optimization

#### 10.5.1 Cross-AZ Data Transfer Reduction

**Single-AZ node groups for specific workloads:**
```yaml
# For high-throughput, low-latency workloads
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
nodeGroups:
- name: single-az-compute
  instanceTypes: ["c5n.xlarge"]
  availabilityZones: ["us-west-2a"]  # Single AZ to avoid cross-AZ charges
  minSize: 2
  maxSize: 10
  labels:
    topology: single-az
```

**Pod anti-affinity for AZ awareness:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-processor
spec:
  replicas: 3
  template:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: ["data-processor"]
              topologyKey: topology.kubernetes.io/zone
```

#### 10.5.2 NAT Gateway Cost Optimization

**NAT Gateway alternatives:**
```bash
# Option 1: NAT instances (cheaper for high traffic)
# Create NAT instance instead of NAT Gateway for cost savings

# Option 2: VPC endpoints for AWS services
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.us-west-2.s3 \
  --route-table-ids rtb-12345678

# Option 3: Public subnets for non-sensitive workloads
# Move some workloads to public subnets to avoid NAT costs
```

---

### 10.6 Cluster Consolidation and Multi-Tenancy

#### 10.6.1 Cluster Consolidation Strategy

**When to consolidate clusters:**
- Multiple small clusters with low utilization
- Similar security requirements
- Shared operational overhead

**Namespace-based multi-tenancy:**
```yaml
# Resource quotas per team
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    persistentvolumeclaims: "10"
    services.loadbalancers: "2"

---
# Network policies for isolation
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: team-a-isolation
  namespace: team-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: team-a
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: team-a
  - to: []  # Allow egress to internet
    ports:
    - protocol: TCP
      port: 443
    - protocol: TCP
      port: 80
```

#### 10.6.2 Shared Services Strategy

**Centralized monitoring and logging:**
```yaml
# Shared monitoring namespace
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    shared-service: "true"

---
# Prometheus for all teams
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  template:
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        resources:
          requests:
            cpu: 2
            memory: 4Gi
          limits:
            cpu: 4
            memory: 8Gi
```

---

### 10.7 Cost Monitoring and Alerting

#### 10.7.1 Cost Anomaly Detection

**CloudWatch cost alerts:**
```bash
# Create cost budget with alerts
aws budgets create-budget \
  --account-id 123456789012 \
  --budget file://eks-cost-budget.json

# Budget configuration
cat > eks-cost-budget.json << EOF
{
  "BudgetName": "EKS-Monthly-Budget",
  "BudgetLimit": {
    "Amount": "5000",
    "Unit": "USD"
  },
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST",
  "CostFilters": {
    "Service": ["Amazon Elastic Kubernetes Service", "Amazon Elastic Compute Cloud"]
  }
}
EOF
```

#### 10.7.2 Resource Utilization Monitoring

**Cluster cost efficiency metrics:**
```bash
#!/bin/bash
# Calculate cluster cost efficiency

# Get total cluster capacity
TOTAL_CPU=$(kubectl get nodes -o jsonpath='{.items[*].status.capacity.cpu}' | tr ' ' '+' | bc)
TOTAL_MEMORY=$(kubectl get nodes -o jsonpath='{.items[*].status.capacity.memory}' | sed 's/Ki//g' | tr ' ' '+' | bc)

# Get allocated resources
ALLOCATED_CPU=$(kubectl describe nodes | grep -A 5 "Allocated resources" | grep "cpu" | awk '{print $2}' | sed 's/[^0-9]//g' | tr '\n' '+' | sed 's/+$//' | bc)
ALLOCATED_MEMORY=$(kubectl describe nodes | grep -A 5 "Allocated resources" | grep "memory" | awk '{print $2}' | sed 's/[^0-9]//g' | tr '\n' '+' | sed 's/+$//' | bc)

# Calculate utilization
CPU_UTILIZATION=$(echo "scale=2; $ALLOCATED_CPU / $TOTAL_CPU * 100" | bc)
MEMORY_UTILIZATION=$(echo "scale=2; $ALLOCATED_MEMORY / $TOTAL_MEMORY * 100" | bc)

echo "Cluster Resource Utilization:"
echo "CPU: ${CPU_UTILIZATION}%"
echo "Memory: ${MEMORY_UTILIZATION}%"

# Alert if utilization is too low (waste) or too high (risk)
if (( $(echo "$CPU_UTILIZATION < 30" | bc -l) )); then
  echo "⚠️  Low CPU utilization - consider downsizing"
elif (( $(echo "$CPU_UTILIZATION > 80" | bc -l) )); then
  echo "⚠️  High CPU utilization - consider scaling up"
fi
```

---

### 10.8 Cost Optimization Runbooks

#### 10.8.1 "Monthly cost spike" Investigation

**Step 1: Identify cost drivers**
```bash
# Get cost breakdown by service
aws ce get-cost-and-usage \
  --time-period Start=$(date -d 'last month' +%Y-%m-01),End=$(date +%Y-%m-01) \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE

# Get cost by resource tags
aws ce get-cost-and-usage \
  --time-period Start=$(date -d 'last month' +%Y-%m-01),End=$(date +%Y-%m-01) \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=TAG,Key=Environment
```

**Step 2: Analyze resource usage**
```bash
# Check for resource over-provisioning
kubectl top nodes
kubectl top pods -A --sort-by=cpu | head -20

# Look for unused resources
kubectl get pvc -A | grep -v Bound
aws ec2 describe-volumes --filters Name=status,Values=available
```

**Step 3: Implement immediate cost reductions**
```bash
# Scale down non-production environments
kubectl scale deployment --replicas=0 -n staging --all
kubectl scale deployment --replicas=1 -n development --all

# Clean up unused resources
kubectl delete pvc -A --field-selector=status.phase=Pending
```

#### 10.8.2 "Right-sizing recommendations" Runbook

**Step 1: Collect usage data**
```bash
# Install VPA recommender
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-0.13.0/vpa-release.yaml

# Create VPA for all deployments
for deployment in $(kubectl get deployments -A -o jsonpath='{.items[*].metadata.name}'); do
  kubectl create vpa ${deployment}-vpa --target-ref=Deployment/${deployment} --update-mode=Off
done
```

**Step 2: Analyze recommendations**
```bash
# Get VPA recommendations
kubectl get vpa -A -o custom-columns=NAME:.metadata.name,NAMESPACE:.metadata.namespace,CPU_TARGET:.status.recommendation.containerRecommendations[0].target.cpu,MEMORY_TARGET:.status.recommendation.containerRecommendations[0].target.memory
```

**Step 3: Apply optimizations**
```bash
# Update deployment with new resource requests
kubectl patch deployment web-app -p '{"spec":{"template":{"spec":{"containers":[{"name":"web-app","resources":{"requests":{"cpu":"200m","memory":"256Mi"}}}]}}}}'
```

---

## 11. Troubleshooting cookbook

Step-by-step solutions for the most common EKS production failures. Symptoms, diagnosis, root cause, and fixes you can run under pressure.

---

### 11.1 Pod Scheduling Failures

#### 11.1.1 "Pods stuck in Pending state"

**Symptoms:**
- New pods remain in Pending status
- `kubectl get pods` shows Pending for extended periods
- Applications fail to scale up

**Diagnosis:**
```bash
# Check pod events for scheduling failures
kubectl describe pod <pending-pod>

# Check node resource availability
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check for taints blocking scheduling
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

**Common root causes and fixes:**

**Insufficient resources:**
```bash
# Check cluster capacity
kubectl top nodes

# Scale cluster if needed
aws eks update-nodegroup-config \
  --cluster-name production-cluster \
  --nodegroup-name primary \
  --scaling-config minSize=3,maxSize=20,desiredSize=10
```

**Node selector mismatch:**
```bash
# Check pod node selector
kubectl get pod <pod> -o yaml | grep -A 5 nodeSelector

# Check available node labels
kubectl get nodes --show-labels

# Fix: Update pod spec or add labels to nodes
kubectl label node <node-name> environment=production
```

**Taints and tolerations:**
```bash
# Remove problematic taint
kubectl taint node <node-name> key:NoSchedule-

# Or add toleration to pod
kubectl patch deployment <deployment> -p '{
  "spec": {
    "template": {
      "spec": {
        "tolerations": [{
          "key": "key",
          "operator": "Equal",
          "value": "value",
          "effect": "NoSchedule"
        }]
      }
    }
  }
}'
```

#### 11.1.2 "Cluster Autoscaler thrashing (rapid scale up/down)"

**Symptoms:**
- Nodes constantly being added and removed
- Workload instability during scaling events
- High AWS costs from node churn

**Root cause:** Flaky readiness probes causing pods to appear unschedulable.

**Diagnosis:**
```bash
# Check Cluster Autoscaler logs
kubectl -n kube-system logs -l app=cluster-autoscaler --tail=100

# Look for rapid scale events
kubectl -n kube-system logs -l app=cluster-autoscaler | grep -E "(scale-up|scale-down)"

# Check pod readiness probe failures
kubectl get events --field-selector reason=Unhealthy --sort-by='.lastTimestamp'
```

**Fix:**
```bash
# Identify problematic deployment
kubectl describe pod <failing-pod> | grep -A 10 "Readiness probe failed"

# Fix readiness probe configuration
kubectl patch deployment <deployment> -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "<container>",
          "readinessProbe": {
            "initialDelaySeconds": 30,
            "periodSeconds": 10,
            "timeoutSeconds": 5,
            "failureThreshold": 3,
            "successThreshold": 1
          }
        }]
      }
    }
  }
}'

# Tune Cluster Autoscaler to reduce thrashing
kubectl -n kube-system patch deployment cluster-autoscaler -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "cluster-autoscaler",
          "command": [
            "./cluster-autoscaler",
            "--scale-down-delay-after-add=10m",
            "--scale-down-unneeded-time=10m",
            "--skip-nodes-with-local-storage=false"
          ]
        }]
      }
    }
  }
}'
```

#### 11.1.3 "DiskPressure causing pod evictions"

**Symptoms:**
- Pods being evicted with reason "DiskPressure"
- Node conditions show DiskPressure=True
- Container image pulls failing

**Root cause:** Large container images or excessive logging filling node disk.

**Diagnosis:**
```bash
# Check node disk usage
kubectl get nodes -o custom-columns=NAME:.metadata.name,DISK-PRESSURE:.status.conditions[?(@.type==\"DiskPressure\")].status

# Check disk usage on specific node
kubectl debug node/<node-name> -it --image=busybox -- df -h

# Check container image sizes
kubectl debug node/<node-name> -it --image=busybox -- crictl images | sort -k2 -h

# Check log sizes
kubectl debug node/<node-name> -it --image=busybox -- du -sh /var/log/containers/*
```

**Fix:**
```bash
# Clean up unused images
kubectl debug node/<node-name> -it --image=busybox -- crictl rmi --prune

# Restart containerd to clear cache
kubectl debug node/<node-name> -it --image=busybox -- systemctl restart containerd

# For EKS managed nodes, increase disk size
aws eks update-nodegroup-config \
  --cluster-name production-cluster \
  --nodegroup-name primary \
  --launch-template name=eks-node-template,version=2

# Configure log rotation
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: amazon-cloudwatch
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020
        storage.path  /var/fluent-bit/state/flb-storage/
        storage.sync  normal
        storage.checksum off
        storage.backlog.mem_limit 5M
        
    [INPUT]
        Name              tail
        Tag               application.*
        Exclude_Path      /var/log/containers/cloudwatch-agent*, /var/log/containers/fluent-bit*
        Path              /var/log/containers/*.log
        Docker_Mode       On
        Docker_Mode_Flush 5
        Docker_Mode_Parser container_firstline
        Parser            docker
        DB                /var/fluent-bit/state/flb_container.db
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On
        Refresh_Interval  10
        Rotate_Wait       30
        storage.type      filesystem
        Read_from_Head    Off
EOF
```

#### 11.1.4 "Zombie pods with stuck finalizers"

**Symptoms:**
- Pods stuck in Terminating state for extended periods
- `kubectl delete pod --force` doesn't work
- Namespace deletion hangs

**Root cause:** Custom finalizers not being processed due to controller failures.

**Diagnosis:**
```bash
# Find pods with finalizers
kubectl get pods --all-namespaces -o json | jq -r '.items[] | select(.metadata.finalizers != null) | "\(.metadata.namespace)/\(.metadata.name): \(.metadata.finalizers)"'

# Check specific pod finalizers
kubectl get pod <pod-name> -o json | jq '.metadata.finalizers'

# Check if controller managing finalizer is running
kubectl get pods -n <controller-namespace> | grep <controller-name>
```

**Fix:**
```bash
# Remove finalizers manually (DANGEROUS - only if controller is confirmed dead)
kubectl patch pod <pod-name> -p '{"metadata":{"finalizers":[]}}' --type=merge

# For namespace stuck in terminating
kubectl get namespace <namespace> -o json | jq '.spec.finalizers = []' | kubectl replace --raw "/api/v1/namespaces/<namespace>/finalize" -f -

# Restart the controller managing the finalizer
kubectl -n <controller-namespace> rollout restart deployment <controller-name>
```

### 11.2 API and Internal Networking Failures

#### 11.2.1 "API Server throttling causing cluster inaccessibility"

**Symptoms:**
- `kubectl` commands timing out or returning 429 errors
- Applications unable to communicate with API server
- High API server latency in metrics

**Root cause:** Too many concurrent API requests overwhelming the API server.

**Diagnosis:**
```bash
# Check API server metrics
kubectl top pods -n kube-system | grep kube-apiserver

# Check for throttling in API server logs (EKS managed - use CloudWatch)
aws logs filter-log-events \
  --log-group-name /aws/eks/production-cluster/cluster \
  --filter-pattern "throttling"

# Identify high-volume API clients
kubectl get events --sort-by='.lastTimestamp' | head -20
```

**Fix:**
```bash
# Identify and throttle problematic controllers
kubectl get deployments --all-namespaces -o wide | grep -v "1/1"

# Scale down misbehaving controllers temporarily
kubectl scale deployment <problematic-controller> --replicas=0 -n <namespace>

# For custom controllers, implement exponential backoff
# Add rate limiting to controller reconcile loops
```

#### 11.2.2 "Pod-to-pod communication failures"

**Symptoms:**
- Services unreachable from other pods
- Intermittent connection timeouts
- DNS resolution working but connections failing

**Root cause:** Network policies, security groups, or CNI issues.

**Diagnosis:**
```bash
# Test basic connectivity
kubectl run debug-pod --image=busybox -it --rm -- sh
# Inside pod: nslookup <service-name>.<namespace>.svc.cluster.local
# Inside pod: wget -qO- <service-name>.<namespace>.svc.cluster.local:8080

# Check network policies
kubectl get networkpolicies --all-namespaces

# Check AWS security groups (for pods using security groups)
aws ec2 describe-security-groups --group-ids <sg-id>

# Check CNI plugin status
kubectl -n kube-system get pods -l k8s-app=aws-node
kubectl -n kube-system logs -l k8s-app=aws-node
```

**Fix:**
```bash
# Allow traffic in network policy
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-debug-traffic
  namespace: <target-namespace>
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: <source-namespace>
    ports:
    - protocol: TCP
      port: 8080
EOF

# Restart CNI pods if needed
kubectl -n kube-system delete pods -l k8s-app=aws-node
```

### 11.3 Resource Management Issues

#### 11.3.1 "CronJobs exhausting cluster resources"

**Symptoms:**
- Cluster resource exhaustion during scheduled job runs
- Multiple CronJobs running simultaneously
- Node resource pressure during specific time windows

**Root cause:** CronJobs without resource limits running concurrently.

**Diagnosis:**
```bash
# Check running CronJobs
kubectl get cronjobs --all-namespaces

# Check job resource usage
kubectl top pods --all-namespaces | grep -E "(job|cron)"

# Check CronJob schedules for overlap
kubectl get cronjobs --all-namespaces -o custom-columns=NAME:.metadata.name,SCHEDULE:.spec.schedule,SUSPEND:.spec.suspend
```

**Fix:**
```bash
# Add resource limits to CronJob
kubectl patch cronjob <cronjob-name> -p '{
  "spec": {
    "jobTemplate": {
      "spec": {
        "template": {
          "spec": {
            "containers": [{
              "name": "<container-name>",
              "resources": {
                "requests": {
                  "cpu": "100m",
                  "memory": "256Mi"
                },
                "limits": {
                  "cpu": "500m",
                  "memory": "512Mi"
                }
              }
            }]
          }
        }
      }
    }
  }
}'

# Prevent concurrent executions
kubectl patch cronjob <cronjob-name> -p '{
  "spec": {
    "concurrencyPolicy": "Forbid"
  }
}'

# Stagger CronJob schedules
kubectl patch cronjob <cronjob-name> -p '{
  "spec": {
    "schedule": "5 2 * * *"
  }
}'
```

#### 11.3.2 "Excessive logging filling node disk"

**Symptoms:**
- Node DiskPressure conditions
- Pod evictions due to disk space
- `/var/log/containers/` consuming excessive space

**Root cause:** Applications logging at debug level or without log rotation.

**Diagnosis:**
```bash
# Check disk usage on nodes
kubectl get nodes -o custom-columns=NAME:.metadata.name,DISK-PRESSURE:.status.conditions[?(@.type==\"DiskPressure\")].status

# Find largest log files
kubectl debug node/<node-name> -it --image=busybox -- du -sh /var/log/containers/* | sort -h | tail -10

# Check specific pod log size
kubectl debug node/<node-name> -it --image=busybox -- ls -lah /var/log/containers/<pod-name>*
```

**Fix:**
```bash
# Reduce log level in application
kubectl set env deployment/<deployment-name> LOG_LEVEL=INFO

# Configure log rotation via containerd
kubectl debug node/<node-name> -it --image=busybox -- sh -c '
cat > /etc/containerd/config.toml << EOF
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true

[plugins."io.containerd.grpc.v1.cri"]
  max_container_log_line_size = 16384
  
[plugins."io.containerd.grpc.v1.cri".registry]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
      endpoint = ["https://registry-1.docker.io"]
EOF
systemctl restart containerd
'

# Clean up large log files immediately
kubectl debug node/<node-name> -it --image=busybox -- sh -c 'truncate -s 0 /var/log/containers/<large-log-file>'
```

### 11.4 Security and RBAC Issues

#### 11.4.1 "aws-auth ConfigMap corruption causing cluster lockout"

**Symptoms:**
- Unable to access cluster with existing IAM roles/users
- `kubectl` commands return "Unauthorized" errors
- Previously working IAM authentication suddenly fails
- New nodes unable to join cluster

**Root cause:** Malformed YAML in aws-auth ConfigMap due to indentation errors.

The aws-auth ConfigMap is the single point of failure for EKS cluster access. A single space or tab error can lock out all users.

**Diagnosis:**
```bash
# Check current aws-auth ConfigMap
kubectl get configmap aws-auth -n kube-system -o yaml

# Validate YAML syntax
kubectl get configmap aws-auth -n kube-system -o yaml | yq eval '.'

# Check for common issues
kubectl get configmap aws-auth -n kube-system -o yaml | grep -E "^\s*-\s*rolearn|^\s*-\s*userarn" | cat -A
```

**Emergency access recovery:**
```bash
# If locked out, use cluster creator credentials or root user
aws sts get-caller-identity

# Access via AWS Console EKS service or CloudShell
# Or use emergency break-glass role if configured
```

**Fix malformed aws-auth:**
```bash
# Backup current ConfigMap first
kubectl get configmap aws-auth -n kube-system -o yaml > aws-auth-backup.yaml

# Fix common indentation issues
kubectl patch configmap aws-auth -n kube-system -p '{
  "data": {
    "mapRoles": "- rolearn: arn:aws:iam::123456789012:role/eksctl-cluster-nodegroup-NodeInstanceRole\n  username: system:node:{{EC2PrivateDNSName}}\n  groups:\n    - system:bootstrappers\n    - system:nodes\n- rolearn: arn:aws:iam::123456789012:role/EKSAdminRole\n  username: admin\n  groups:\n    - system:masters",
    "mapUsers": "- userarn: arn:aws:iam::123456789012:user/developer\n  username: developer\n  groups:\n    - developers"
  }
}'

# Validate the fix
kubectl auth can-i '*' '*' --as=arn:aws:iam::123456789012:role/EKSAdminRole
```

**Correct aws-auth format:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/eksctl-cluster-nodegroup-NodeInstanceRole
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
    - rolearn: arn:aws:iam::123456789012:role/EKSAdminRole
      username: admin
      groups:
        - system:masters
  mapUsers: |
    - userarn: arn:aws:iam::123456789012:user/developer
      username: developer
      groups:
        - developers
```

**Common aws-auth mistakes:**
```yaml
# WRONG - Mixed tabs and spaces
mapRoles: |
  - rolearn: arn:aws:iam::123456789012:role/NodeRole
	username: system:node:{{EC2PrivateDNSName}}  # Tab here
    groups:  # Spaces here
      - system:nodes

# WRONG - Incorrect indentation
mapRoles: |
- rolearn: arn:aws:iam::123456789012:role/NodeRole  # Should be indented
  username: system:node:{{EC2PrivateDNSName}}

# WRONG - Missing pipe character
mapRoles:  # Missing |
  - rolearn: arn:aws:iam::123456789012:role/NodeRole

# WRONG - Extra characters
mapRoles: |
  - rolearn: arn:aws:iam::123456789012:role/NodeRole,  # Comma at end
    username: system:node:{{EC2PrivateDNSName}}
```

**Prevention and monitoring:**
```bash
# Validate before applying
yq eval '.data.mapRoles' aws-auth.yaml
yq eval '.data.mapUsers' aws-auth.yaml

# Set up monitoring for aws-auth changes
kubectl create -f - <<EOF
apiVersion: v1
kind: Event
metadata:
  name: aws-auth-monitor
  namespace: kube-system
EOF

# Use eksctl for safer aws-auth management
eksctl create iamidentitymapping \
  --cluster production-cluster \
  --region us-west-2 \
  --arn arn:aws:iam::123456789012:role/EKSAdminRole \
  --group system:masters \
  --username admin

# Always backup before changes
kubectl get configmap aws-auth -n kube-system -o yaml > "aws-auth-backup-$(date +%Y%m%d-%H%M%S).yaml"
```

#### 11.4.2 "Pod Security Policy not enforcing restrictions"

**Symptoms:**
- Privileged containers running despite PSP configuration
- Security policies being bypassed
- Containers running as root when they shouldn't

**Root cause:** Missing admission controller or misconfigured PSP.

**Diagnosis:**
```bash
# Check if PSP admission controller is enabled (EKS doesn't enable by default)
kubectl get pods -n kube-system kube-apiserver-* -o yaml | grep -A 5 admission-control

# Check existing PSPs
kubectl get psp

# Check pod security context
kubectl get pod <pod-name> -o yaml | grep -A 10 securityContext

# Check if pod is using PSP
kubectl describe pod <pod-name> | grep -i "psp\|security"
```

**Fix:**
```bash
# For EKS, use Pod Security Standards instead of PSP
kubectl label namespace <namespace> pod-security.kubernetes.io/enforce=restricted
kubectl label namespace <namespace> pod-security.kubernetes.io/audit=restricted
kubectl label namespace <namespace> pod-security.kubernetes.io/warn=restricted

# Create restrictive security context in deployment
kubectl patch deployment <deployment> -p '{
  "spec": {
    "template": {
      "spec": {
        "securityContext": {
          "runAsNonRoot": true,
          "runAsUser": 1000,
          "fsGroup": 2000
        },
        "containers": [{
          "name": "<container>",
          "securityContext": {
            "allowPrivilegeEscalation": false,
            "readOnlyRootFilesystem": true,
            "capabilities": {
              "drop": ["ALL"]
            }
          }
        }]
      }
    }
  }
}'
```

#### 11.4.3 "RBAC permissions too broad or too restrictive"

**Symptoms:**
- Users can access resources they shouldn't
- Service accounts failing with permission errors
- Applications unable to perform required operations

**Diagnosis:**
```bash
# Check current user permissions
kubectl auth can-i --list

# Check service account permissions
kubectl auth can-i --list --as=system:serviceaccount:<namespace>:<service-account>

# Check role bindings for user/service account
kubectl get rolebindings,clusterrolebindings --all-namespaces -o wide | grep <user-or-sa>

# Test specific permission
kubectl auth can-i create pods --as=system:serviceaccount:<namespace>:<service-account>
```

**Fix:**
```bash
# Create minimal role for service account
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: <namespace>
  name: <app>-role
rules:
- apiGroups: [""]
  resources: ["pods", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: <app>-binding
  namespace: <namespace>
subjects:
- kind: ServiceAccount
  name: <service-account>
  namespace: <namespace>
roleRef:
  kind: Role
  name: <app>-role
  apiGroup: rbac.authorization.k8s.io
EOF

# Remove overly broad cluster role binding
kubectl delete clusterrolebinding <overly-broad-binding>
```

#### 11.4.4 "Secrets exposed in environment variables or logs"

**Symptoms:**
- Sensitive data visible in pod environment
- Secrets appearing in application logs
- Configuration containing plaintext credentials

**Diagnosis:**
```bash
# Check environment variables in running pod
kubectl exec <pod-name> -- env | grep -i -E "(password|secret|key|token)"

# Check if secrets are mounted as files vs env vars
kubectl describe pod <pod-name> | grep -A 10 -B 5 -i secret

# Check recent logs for exposed secrets
kubectl logs <pod-name> | grep -i -E "(password|secret|key|token)" | head -5
```

**Fix:**
```bash
# Mount secrets as files instead of env vars
kubectl patch deployment <deployment> -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "<container>",
          "volumeMounts": [{
            "name": "secret-volume",
            "mountPath": "/etc/secrets",
            "readOnly": true
          }]
        }],
        "volumes": [{
          "name": "secret-volume",
          "secret": {
            "secretName": "<secret-name>",
            "defaultMode": 256
          }
        }]
      }
    }
  }
}'

# Remove secret from environment variables
kubectl patch deployment <deployment> -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "<container>",
          "env": null
        }]
      }
    }
  }
}'

# Use External Secrets Operator for better secret management
kubectl apply -f - <<EOF
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: <app>-secret
  namespace: <namespace>
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: <app>-secret
    creationPolicy: Owner
  data:
  - secretKey: password
    remoteRef:
      key: <secret-arn>
      property: password
EOF
```

#### 11.4.5 "PodDisruptionBudget blocking evictions"

**Symptoms:**
- Node drain operations hang
- Cluster autoscaler cannot scale down
- Rolling updates stuck

**Diagnosis:**
```bash
# Check PDB status
kubectl get pdb -A

# Check which pods are blocking eviction
kubectl describe pdb <pdb-name>
```

**Fix:**
```bash
# Temporarily relax PDB
kubectl patch pdb <pdb-name> -p '{"spec":{"minAvailable":1}}'

# Or scale up replicas to meet PDB requirements
kubectl scale deployment <deployment> --replicas=5

# Complete maintenance, then restore PDB
kubectl patch pdb <pdb-name> -p '{"spec":{"minAvailable":3}}'
```

---

### 11.5 Egress and Service Discovery Failures

#### 11.5.1 "Pods can't reach external services"

**Symptoms:**
- Timeouts connecting to external APIs
- DNS resolution works but connections fail
- Intermittent connectivity issues

**Diagnosis:**
```bash
# Test connectivity from pod
kubectl exec -it <pod> -- curl -v https://api.external.com

# Check NAT Gateway health
aws ec2 describe-nat-gateways --nat-gateway-ids <nat-gw-id>

# Check security group rules
aws ec2 describe-security-groups --group-ids <sg-id>
```

**Common fixes:**

**NAT Gateway issues:**
```bash
# Check NAT Gateway metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/NATGateway \
  --metric-name PacketsDropCount \
  --dimensions Name=NatGatewayId,Value=<nat-gw-id> \
  --start-time $(date -d '1 hour ago' -Iseconds) \
  --end-time $(date -Iseconds) \
  --period 300 \
  --statistics Sum
```

**Security group blocking traffic:**
```bash
# Add egress rule for HTTPS
aws ec2 authorize-security-group-egress \
  --group-id <sg-id> \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0
```

#### 11.5.2 "Service discovery not working"

**Symptoms:**
- Pods can't reach other services by name
- `nslookup` fails for service names
- Intermittent DNS failures

**Diagnosis:**
```bash
# Test DNS resolution
kubectl exec -it <pod> -- nslookup kubernetes.default.svc.cluster.local

# Check CoreDNS health
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=100

# Check service endpoints
kubectl get endpoints <service-name>
```

**Fixes:**

**CoreDNS not ready:**
```bash
# Scale up CoreDNS
kubectl -n kube-system scale deployment coredns --replicas=3

# Check CoreDNS configuration
kubectl -n kube-system get configmap coredns -o yaml
```

**Service has no endpoints:**
```bash
# Check if pods are ready
kubectl get pods -l app=<service-selector>

# Check service selector
kubectl describe service <service-name>

# Fix selector mismatch
kubectl patch service <service-name> -p '{"spec":{"selector":{"app":"correct-label"}}}'
```

---

### 11.6 Storage Issues

#### 11.6.1 "Pods stuck in ContainerCreating due to volume mount failures"

**Symptoms:**
- Pods stuck in ContainerCreating state
- Events show volume mount errors
- StatefulSet pods fail to start

**Diagnosis:**
```bash
# Check pod events
kubectl describe pod <pod>

# Check PVC status
kubectl get pvc

# Check volume attachment
kubectl get volumeattachment
```

**Common fixes:**

**EBS volume in wrong AZ:**
```bash
# Check pod and volume zones
kubectl get pod <pod> -o wide
kubectl describe pv <pv-name> | grep zone

# Delete pod to reschedule in correct AZ
kubectl delete pod <pod>
```

**CSI driver issues:**
```bash
# Check CSI driver health
kubectl -n kube-system get pods -l app=ebs-csi-controller
kubectl -n kube-system logs -l app=ebs-csi-controller

# Restart CSI driver if needed
kubectl -n kube-system rollout restart deployment ebs-csi-controller
```

#### 11.6.2 "PVC stuck in Pending state"

**Symptoms:**
- PVC remains in Pending status
- No PV created for dynamic provisioning
- Storage class issues

**Diagnosis:**
```bash
# Check PVC events
kubectl describe pvc <pvc-name>

# Check storage class
kubectl describe storageclass <storage-class>

# Check CSI provisioner logs
kubectl -n kube-system logs -l app=ebs-csi-controller
```

**Fixes:**

**Storage class misconfiguration:**
```bash
# Check available storage classes
kubectl get storageclass

# Create correct storage class
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
allowVolumeExpansion: true
reclaimPolicy: Delete
EOF
```

---

### 11.7 Application Failures

#### 11.7.1 "Pods crashing with OOMKilled"

**Symptoms:**
- Pods restart frequently
- Exit code 137 (OOMKilled)
- Application performance degradation

**Diagnosis:**
```bash
# Check pod resource usage
kubectl top pod <pod>

# Check pod events for OOM
kubectl describe pod <pod> | grep -i oom

# Check memory limits
kubectl get pod <pod> -o yaml | grep -A 5 resources
```

**Fixes:**

**Increase memory limits:**
```bash
# Update deployment with higher memory limits
kubectl patch deployment <deployment> -p '{"spec":{"template":{"spec":{"containers":[{"name":"<container>","resources":{"limits":{"memory":"2Gi"},"requests":{"memory":"1Gi"}}}]}}}}'
```

**Optimize application memory usage:**
```bash
# Check for memory leaks
kubectl exec -it <pod> -- ps aux --sort=-%mem | head

# Enable memory profiling (application-specific)
kubectl set env deployment/<deployment> GOMAXPROCS=2 GOMEMLIMIT=1GiB
```

#### 11.7.2 "Readiness probe failures causing traffic issues"

**Symptoms:**
- Pods not receiving traffic
- Service endpoints empty
- Load balancer health checks failing

**Diagnosis:**
```bash
# Check pod readiness
kubectl get pods -o wide

# Check readiness probe configuration
kubectl describe pod <pod> | grep -A 10 "Readiness"

# Test probe endpoint manually
kubectl exec -it <pod> -- curl localhost:8080/health
```

**Fixes:**

**Adjust probe timing:**
```bash
# Update probe configuration
kubectl patch deployment <deployment> -p '{"spec":{"template":{"spec":{"containers":[{"name":"<container>","readinessProbe":{"initialDelaySeconds":30,"periodSeconds":10,"timeoutSeconds":5,"failureThreshold":3}}]}}}}'
```

**Fix probe endpoint:**
```bash
# Check if health endpoint is correct
kubectl exec -it <pod> -- netstat -tlnp | grep :8080

# Update probe path if needed
kubectl patch deployment <deployment> -p '{"spec":{"template":{"spec":{"containers":[{"name":"<container>","readinessProbe":{"httpGet":{"path":"/healthz","port":8080}}}]}}}}'
```

---

### 11.8 Cluster-Level Issues

#### 11.8.1 "API server timeouts and high latency"

**Symptoms:**
- `kubectl` commands timeout
- High API server response times
- Cluster operations slow

**Diagnosis:**
```bash
# Check API server metrics
kubectl get --raw /metrics | grep apiserver_request_duration

# Check etcd health
kubectl get --raw /healthz/etcd

# Check for resource pressure
kubectl top nodes
```

**Fixes:**

**Reduce API server load:**
```bash
# Find clients making excessive requests
kubectl get events --sort-by='.lastTimestamp' | tail -20

# Scale down chatty controllers
kubectl scale deployment <noisy-controller> --replicas=0

# Increase API server resources (managed by AWS for EKS)
# Contact AWS support if persistent
```

#### 11.8.2 "Cluster autoscaler not scaling"

**Symptoms:**
- Pending pods but no new nodes
- Cluster autoscaler logs show errors
- Node groups not scaling up

**Diagnosis:**
```bash
# Check cluster autoscaler logs
kubectl -n kube-system logs -l app=cluster-autoscaler

# Check node group configuration
aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name <nodegroup>

# Check IAM permissions
aws sts get-caller-identity
```

**Fixes:**

**IAM permission issues:**
```bash
# Check autoscaler service account
kubectl -n kube-system describe sa cluster-autoscaler

# Verify IAM role has required permissions
aws iam get-role-policy --role-name <autoscaler-role> --policy-name <policy-name>
```

**Node group limits:**
```bash
# Increase node group max size
aws eks update-nodegroup-config \
  --cluster-name <cluster> \
  --nodegroup-name <nodegroup> \
  --scaling-config minSize=2,maxSize=20,desiredSize=5
```

---

### 11.9 Performance Issues

#### 11.9.1 "High CPU throttling affecting performance"

**Symptoms:**
- Application response times high
- CPU usage appears low but performance poor
- Intermittent slowdowns

**Diagnosis:**
```bash
# Check CPU throttling
kubectl exec -it <pod> -- cat /sys/fs/cgroup/cpu/cpu.stat | grep throttled

# Check CPU limits vs requests
kubectl describe pod <pod> | grep -A 10 Limits

# Monitor CPU usage patterns
kubectl top pod <pod> --containers
```

**Fixes:**

**Adjust CPU limits:**
```bash
# Remove CPU limits for CPU-intensive workloads
kubectl patch deployment <deployment> -p '{"spec":{"template":{"spec":{"containers":[{"name":"<container>","resources":{"limits":{"cpu":null}}}]}}}}'

# Or increase CPU limits
kubectl patch deployment <deployment> -p '{"spec":{"template":{"spec":{"containers":[{"name":"<container>","resources":{"limits":{"cpu":"2000m"}}}]}}}}'
```

#### 11.9.2 "Disk I/O bottlenecks"

**Symptoms:**
- High disk wait times
- Application timeouts during disk operations
- EBS volume performance issues

**Diagnosis:**
```bash
# Check disk I/O from pod
kubectl exec -it <pod> -- iostat -x 1 5

# Check EBS volume metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EBS \
  --metric-name VolumeReadOps \
  --dimensions Name=VolumeId,Value=<volume-id> \
  --start-time $(date -d '1 hour ago' -Iseconds) \
  --end-time $(date -Iseconds) \
  --period 300 \
  --statistics Average
```

**Fixes:**

**Upgrade to higher IOPS volume:**
```bash
# Modify EBS volume type
aws ec2 modify-volume \
  --volume-id <volume-id> \
  --volume-type gp3 \
  --iops 10000
```

---

### 11.10 Emergency Procedures

#### 11.10.1 "Cluster completely unresponsive"

**Immediate actions:**
```bash
# 1. Check if it's a regional AWS issue
curl -s https://status.aws.amazon.com/

# 2. Try different kubectl context/region
kubectl config use-context <backup-context>

# 3. Check EKS cluster status
aws eks describe-cluster --name <cluster> --region <region>

# 4. If control plane is down, activate DR procedures
# Switch DNS to DR region
aws route53 change-resource-record-sets \
  --hosted-zone-id <zone-id> \
  --change-batch file://failover.json
```

#### 11.10.2 "Mass pod failures across cluster"

**Immediate actions:**
```bash
# 1. Stop any ongoing deployments
kubectl rollout pause deployment/<deployment>

# 2. Check for cluster-wide issues
kubectl get nodes
kubectl -n kube-system get pods

# 3. Check recent changes
kubectl get events --sort-by='.lastTimestamp' | tail -50

# 4. Rollback recent changes if identified
kubectl rollout undo deployment/<deployment>
```

---

### 11.11 Quick Reference Commands

#### 11.11.1 Essential Debugging Commands

```bash
# Pod debugging
kubectl describe pod <pod>
kubectl logs <pod> --previous
kubectl exec -it <pod> -- /bin/bash

# Service debugging
kubectl describe service <service>
kubectl get endpoints <service>
kubectl port-forward service/<service> 8080:80

# Node debugging
kubectl describe node <node>
kubectl top node <node>
kubectl get pods -A -o wide --field-selector spec.nodeName=<node>

# Cluster debugging
kubectl cluster-info
kubectl get events --sort-by='.lastTimestamp'
kubectl get all -A | grep -v Running
```

#### 11.11.2 Emergency Recovery Commands

```bash
# Force delete stuck resources
kubectl delete pod <pod> --force --grace-period=0
kubectl patch pvc <pvc> -p '{"metadata":{"finalizers":null}}'

# Emergency scaling
kubectl scale deployment <deployment> --replicas=0
kubectl scale deployment <deployment> --replicas=3

# Quick rollback
kubectl rollout undo deployment/<deployment>
kubectl rollout status deployment/<deployment>
```

---

This troubleshooting cookbook provides step-by-step solutions for the most common EKS production failures. Each scenario is designed to be used under pressure, with clear symptoms, diagnosis steps, and proven fixes. The key is systematic diagnosis before attempting fixes, and having emergency procedures ready for critical situations.
## 12. EKS at scale

Scale introduces failure modes that simply don't exist in smaller clusters. Hundreds of nodes, thousands of pods, multiple clusters — different operational patterns, different failure scenarios.

---

### 12.1 Multi-Cluster Patterns

#### 12.1.1 When to Use Multiple Clusters

**Cluster boundaries that make sense:**
- **Environment isolation** (prod/staging/dev)
- **Team isolation** (different blast radius requirements)
- **Compliance boundaries** (PCI/SOX/HIPAA workloads)
- **Geographic distribution** (latency/data residency)
- **Scale limits** (approaching EKS/EC2 quotas)

**Anti-patterns to avoid:**
- One cluster per microservice (operational overhead)
- Clusters for cost allocation (use namespaces + tagging)
- Clusters for different Kubernetes versions (use node groups)

#### 12.1.2 Multi-Cluster Networking

**Cross-cluster service communication:**
```yaml
# External DNS for cross-cluster service discovery
apiVersion: v1
kind: Service
metadata:
  name: user-service
  annotations:
    external-dns.alpha.kubernetes.io/hostname: user-service.prod.internal
spec:
  type: LoadBalancer
  selector:
    app: user-service
  ports:
  - port: 80
    targetPort: 8080
```

**VPC peering for cluster connectivity:**
```bash
# Create VPC peering between clusters
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-cluster1 \
  --peer-vpc-id vpc-cluster2 \
  --peer-region us-west-2

# Update route tables for cross-cluster communication
aws ec2 create-route \
  --route-table-id rtb-cluster1 \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id pcx-12345
```

#### 12.1.3 Multi-Cluster Management

**Centralized cluster management with ArgoCD:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production-workloads
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/k8s-manifests
    targetRevision: HEAD
    path: production
  destination:
    server: https://prod-cluster.us-west-2.eks.amazonaws.com
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Cluster inventory management:**
```bash
#!/bin/bash
# Multi-cluster inventory script

CLUSTERS=(
  "prod-us-west-2"
  "prod-us-east-1"
  "staging-us-west-2"
  "dev-us-west-2"
)

for cluster in "${CLUSTERS[@]}"; do
  echo "=== Cluster: $cluster ==="
  
  # Update kubeconfig
  aws eks update-kubeconfig --name $cluster --region ${cluster##*-}
  
  # Get cluster info
  echo "Nodes: $(kubectl get nodes --no-headers | wc -l)"
  echo "Pods: $(kubectl get pods -A --no-headers | wc -l)"
  echo "Version: $(kubectl version --short --client=false | grep Server)"
  
  # Check critical components
  kubectl get pods -n kube-system | grep -E "(coredns|aws-node|kube-proxy)" | grep -v Running && echo "⚠️  System pods not ready"
  
  echo ""
done
```

---

### 12.2 Large Node Pool Management

#### 12.2.1 Node Pool Strategies at Scale

**Diversified instance types for resilience:**
```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: large-scale-cluster
nodeGroups:
- name: general-purpose
  instanceTypes: 
  - m5.large
  - m5.xlarge
  - m5a.large
  - m5a.xlarge
  - m4.large
  - m4.xlarge
  minSize: 10
  maxSize: 500
  desiredCapacity: 50
  spot: true
  labels:
    node-class: general-purpose
```

**Dedicated node pools for specific workloads:**
```yaml
# High-memory workloads
- name: memory-optimized
  instanceTypes: ["r5.xlarge", "r5.2xlarge"]
  minSize: 2
  maxSize: 50
  labels:
    node-class: memory-optimized
  taints:
  - key: workload-type
    value: memory-intensive
    effect: NoSchedule

# GPU workloads
- name: gpu-nodes
  instanceTypes: ["p3.2xlarge", "p3.8xlarge"]
  minSize: 0
  maxSize: 20
  labels:
    node-class: gpu
  taints:
  - key: nvidia.com/gpu
    value: "true"
    effect: NoSchedule
```

#### 12.2.2 Node Lifecycle Management at Scale

**Automated node replacement:**
```bash
#!/bin/bash
# Automated node replacement for large clusters

# Find nodes older than 30 days
OLD_NODES=$(kubectl get nodes -o json | jq -r '.items[] | select(.metadata.creationTimestamp | fromdateiso8601 < (now - 30*24*3600)) | .metadata.name')

for node in $OLD_NODES; do
  echo "Replacing old node: $node"
  
  # Cordon node
  kubectl cordon $node
  
  # Drain node with timeout
  timeout 600 kubectl drain $node --ignore-daemonsets --delete-emptydir-data --force
  
  # Get instance ID and terminate
  INSTANCE_ID=$(kubectl get node $node -o jsonpath='{.spec.providerID}' | cut -d'/' -f5)
  aws ec2 terminate-instances --instance-ids $INSTANCE_ID
  
  # Wait for replacement
  sleep 300
done
```

**Node health monitoring:**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-health-monitor
spec:
  selector:
    matchLabels:
      app: node-health-monitor
  template:
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: monitor
        image: node-health-monitor:latest
        securityContext:
          privileged: true
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        command:
        - /bin/bash
        - -c
        - |
          while true; do
            # Check disk space
            DISK_USAGE=$(df /host/proc/1/root | tail -1 | awk '{print $5}' | sed 's/%//')
            if [ $DISK_USAGE -gt 85 ]; then
              kubectl annotate node $NODE_NAME node.kubernetes.io/disk-pressure=true --overwrite
            fi
            
            # Check memory pressure
            MEM_AVAILABLE=$(cat /host/proc/meminfo | grep MemAvailable | awk '{print $2}')
            MEM_TOTAL=$(cat /host/proc/meminfo | grep MemTotal | awk '{print $2}')
            MEM_USAGE=$(echo "scale=2; (1 - $MEM_AVAILABLE/$MEM_TOTAL) * 100" | bc)
            
            if (( $(echo "$MEM_USAGE > 90" | bc -l) )); then
              kubectl annotate node $NODE_NAME node.kubernetes.io/memory-pressure=true --overwrite
            fi
            
            sleep 60
          done
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
```

---

### 12.3 High-Density Pod Scheduling

#### 12.3.1 Pod Density Optimization

**Understanding EKS pod limits:**
```bash
# Check maximum pods per node type
curl -s https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/refs/heads/master/misc/eni-max-pods.txt | grep -E "(m5|c5|r5)"

# Current pod density
kubectl get nodes -o custom-columns=NAME:.metadata.name,PODS:.status.capacity.pods,RUNNING:.status.allocatable.pods
```

**High-density scheduling configuration:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: amazon-vpc-cni
  namespace: kube-system
data:
  enable-prefix-delegation: "true"  # Increases pod density
  warm-prefix-target: "1"
  warm-ip-target: "5"
  minimum-ip-target: "10"
```

#### 12.3.2 Resource Fragmentation Prevention

**Pod resource standardization:**
```yaml
# Standard resource classes
apiVersion: v1
kind: LimitRange
metadata:
  name: standard-resources
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    min:
      cpu: 50m
      memory: 64Mi
    max:
      cpu: 4
      memory: 8Gi
```

**Anti-fragmentation scheduling:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: high-density-app
spec:
  replicas: 100
  template:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: ["high-density-app"]
              topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: app:latest
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

---

### 12.4 API Server Performance at Scale

#### 12.4.1 API Server Load Management

**Client-side rate limiting:**
```bash
# Configure kubectl rate limiting
export KUBECTL_QPS=50
export KUBECTL_BURST=100

# For applications using client-go
kubectl patch deployment controller-manager -p '{"spec":{"template":{"spec":{"containers":[{"name":"manager","env":[{"name":"QPS","value":"20"},{"name":"BURST","value":"30"}]}]}}}}'
```

**Watch optimization:**
```yaml
# Efficient controller pattern
apiVersion: apps/v1
kind: Deployment
metadata:
  name: efficient-controller
spec:
  template:
    spec:
      containers:
      - name: controller
        image: controller:latest
        env:
        - name: WATCH_NAMESPACE
          value: "production"  # Limit watch scope
        - name: RESYNC_PERIOD
          value: "10m"  # Reduce resync frequency
        - name: WORKER_COUNT
          value: "5"  # Limit concurrent workers
```

#### 12.4.2 etcd Performance Optimization

**etcd monitoring at scale:**
```bash
# Monitor etcd performance metrics
kubectl get --raw /metrics | grep etcd_request_duration_seconds

# Check etcd database size
kubectl get --raw /metrics | grep etcd_mvcc_db_total_size_in_bytes

# Monitor watch streams
kubectl get --raw /metrics | grep etcd_network_client_grpc_received_bytes_total
```

---

### 12.5 Cross-Region and Multi-Region Patterns

#### 12.5.1 Active-Active Multi-Region Setup

**Regional cluster configuration:**
```yaml
# US-West-2 cluster
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: prod-us-west-2
  region: us-west-2
  tags:
    Environment: production
    Region: us-west-2
    Role: primary

# US-East-1 cluster  
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: prod-us-east-1
  region: us-east-1
  tags:
    Environment: production
    Region: us-east-1
    Role: secondary
```

**Cross-region service mesh:**
```yaml
# Istio multi-cluster configuration
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: cross-cluster-gateway
spec:
  selector:
    istio: eastwestgateway
  servers:
  - port:
      number: 15443
      name: tls
      protocol: TLS
    tls:
      mode: ISTIO_MUTUAL
    hosts:
    - "*.local"
```

#### 12.5.2 Global Load Balancing

**Route 53 health checks for multi-region:**
```bash
# Create health check for each region
aws route53 create-health-check \
  --caller-reference "eks-us-west-2-$(date +%s)" \
  --health-check-config Type=HTTPS,ResourcePath=/health,FullyQualifiedDomainName=api-us-west-2.example.com,Port=443

# Create weighted routing policy
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456789 \
  --change-batch file://multi-region-routing.json
```

---

### 12.6 Scaling Challenges and Solutions

#### 12.6.1 Cluster Autoscaler at Scale

**Multi-AZ autoscaling configuration:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: cluster-autoscaler
        image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.21.0
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/production-cluster
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false
        - --scale-down-delay-after-add=10m
        - --scale-down-unneeded-time=10m
        - --max-node-provision-time=15m
```

#### 12.6.2 Karpenter for Large-Scale Provisioning

**Karpenter configuration for scale:**
```yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: large-scale-provisioner
spec:
  limits:
    resources:
      cpu: 10000
      memory: 10000Gi
  requirements:
  - key: karpenter.sh/capacity-type
    operator: In
    values: ["spot", "on-demand"]
  - key: kubernetes.io/arch
    operator: In
    values: ["amd64"]
  - key: node.kubernetes.io/instance-type
    operator: In
    values: ["m5.large", "m5.xlarge", "m5.2xlarge", "c5.large", "c5.xlarge", "c5.2xlarge"]
  providerRef:
    name: large-scale-nodepool
  ttlSecondsAfterEmpty: 30
  ttlSecondsUntilExpired: 2592000  # 30 days
```

---

### 12.7 Operational Patterns at Scale

#### 12.7.1 Centralized Logging and Monitoring

**Fluent Bit configuration for high-throughput:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         5
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020
        storage.path  /var/fluent-bit/state/flb-storage/
        storage.sync  normal
        storage.checksum off
        storage.backlog.mem_limit 50M
    
    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        multiline.parser  docker, cri
        Tag               kube.*
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On
        Skip_Empty_Lines  On
        storage.type      filesystem
        Refresh_Interval  10
```

#### 12.7.2 GitOps at Scale

**ArgoCD application-of-applications pattern:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/k8s-apps
    targetRevision: HEAD
    path: production/applications
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

---

### 12.8 Scale-Related Failure Patterns

#### 12.8.1 "Too many pods per node" Failure

**Symptoms:**
- Pods stuck in Pending with "Too many pods" error
- Node capacity reached but resources available

**Diagnosis:**
```bash
# Check pod limits per node
kubectl describe node <node> | grep -A 10 "Allocatable"

# Check current pod count
kubectl get pods -A -o wide | grep <node> | wc -l
```

**Solutions:**
```bash
# Enable prefix delegation for higher pod density
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true

# Or use larger instance types
aws eks update-nodegroup-config \
  --cluster-name production-cluster \
  --nodegroup-name primary \
  --instance-types m5.xlarge
```

#### 12.8.2 "API server overwhelmed" Failure

**Symptoms:**
- kubectl commands timeout
- High API server CPU/memory
- etcd performance degradation

**Diagnosis:**
```bash
# Check API server metrics
kubectl get --raw /metrics | grep apiserver_request_total

# Check for excessive watch connections
kubectl get --raw /metrics | grep apiserver_registered_watchers
```

**Solutions:**
```bash
# Implement client-side rate limiting
kubectl patch deployment <controller> -p '{"spec":{"template":{"spec":{"containers":[{"name":"controller","env":[{"name":"QPS","value":"10"},{"name":"BURST","value":"15"}]}]}}}}'

# Scale down chatty controllers
kubectl scale deployment <noisy-controller> --replicas=1
```

---

## Appendices

### Appendix A: Reference Materials and Cheat Sheets

#### A.1 Essential kubectl Commands for EKS Troubleshooting

**Pod debugging:**
```bash
# Get pod details with events
kubectl describe pod <pod-name>

# Get logs from previous container instance
kubectl logs <pod-name> --previous

# Get logs from specific container in multi-container pod
kubectl logs <pod-name> -c <container-name>

# Execute commands in pod
kubectl exec -it <pod-name> -- /bin/bash

# Port forward for local debugging
kubectl port-forward pod/<pod-name> 8080:80
```

**Service and networking:**
```bash
# Check service endpoints
kubectl get endpoints <service-name>

# Debug service connectivity
kubectl run debug-pod --image=nicolaka/netshoot --rm -it -- bash

# Check DNS resolution
kubectl exec -it <pod> -- nslookup kubernetes.default.svc.cluster.local

# Test service connectivity
kubectl exec -it <pod> -- curl <service-name>.<namespace>.svc.cluster.local
```

**Node and cluster debugging:**
```bash
# Get node resource usage
kubectl top nodes

# Describe node conditions and capacity
kubectl describe node <node-name>

# Get all pods on a specific node
kubectl get pods -A -o wide --field-selector spec.nodeName=<node-name>

# Check cluster component health
kubectl get componentstatuses
kubectl cluster-info
```

#### A.2 AWS CLI Commands for EKS Operations

**Cluster management:**
```bash
# Update kubeconfig for EKS cluster
aws eks update-kubeconfig --region <region> --name <cluster-name>

# Get cluster information
aws eks describe-cluster --name <cluster-name>

# List all EKS clusters
aws eks list-clusters

# Get cluster endpoint and certificate
aws eks describe-cluster --name <cluster-name> --query 'cluster.{endpoint:endpoint,ca:certificateAuthority.data}'
```

**Node group operations:**
```bash
# List node groups
aws eks list-nodegroups --cluster-name <cluster-name>

# Describe node group
aws eks describe-nodegroup --cluster-name <cluster-name> --nodegroup-name <nodegroup-name>

# Update node group scaling
aws eks update-nodegroup-config --cluster-name <cluster-name> --nodegroup-name <nodegroup-name> --scaling-config minSize=2,maxSize=10,desiredSize=5
```

**Add-on management:**
```bash
# List available add-ons
aws eks describe-addon-versions --kubernetes-version 1.28

# Install add-on
aws eks create-addon --cluster-name <cluster-name> --addon-name vpc-cni --addon-version <version>

# Update add-on
aws eks update-addon --cluster-name <cluster-name> --addon-name vpc-cni --addon-version <new-version>
```

#### A.3 Useful Tools and Utilities

**Network debugging tools:**
```bash
# Install netshoot for comprehensive network debugging
kubectl run netshoot --image=nicolaka/netshoot --rm -it -- bash

# Inside netshoot pod:
nslookup kubernetes.default.svc.cluster.local
dig @8.8.8.8 google.com
curl -v http://service-name.namespace.svc.cluster.local
traceroute 8.8.8.8
ss -tulpn
```

**Resource analysis tools:**
```bash
# Install kube-capacity for resource analysis
kubectl krew install resource-capacity
kubectl resource-capacity

# Install kubectl-top for enhanced resource monitoring
kubectl krew install top
kubectl top pod --sort-by=cpu
```

---

### Appendix B: EKS Networking Deep Dive - Pod Egress Traffic Flow

This appendix provides a detailed technical analysis of how pods in EKS use link-local addresses for egress traffic, using the `ip` command family to trace the complete network path.

#### B.1 EKS Pod Networking Architecture

**Understanding the network stack:**
```
[Pod Container] 
    ↓ (veth pair)
[Pod Network Namespace] 
    ↓ (veth peer)
[Node Root Network Namespace]
    ↓ (AWS VPC CNI routing)
[ENI on EC2 Instance]
    ↓ (VPC routing)
[Internet Gateway / NAT Gateway]
    ↓
[Internet]
```

#### B.2 Tracing Pod Egress Traffic Step-by-Step

**Step 1: Examine pod network namespace**
```bash
# Get pod details and node
kubectl get pod <pod-name> -o wide

# SSH to the node or use kubectl exec
kubectl exec -it <pod-name> -- bash

# Inside the pod, examine network configuration
ip addr show
ip route show
ip route show table all
```

**Example output from inside pod:**
```bash
# ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    inet 127.0.0.1/8 scope host lo
    
3: eth0@if123: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP
    inet 10.0.1.45/32 scope global eth0  # Pod IP from VPC CIDR
    
# ip route show
default via 169.254.1.1 dev eth0  # Link-local gateway
169.254.1.1 dev eth0 scope link   # Link-local route
```

**Step 2: Examine the veth pair connection**
```bash
# From the node (not inside pod), find the pod's network namespace
docker ps | grep <pod-name>
docker inspect <container-id> | grep NetworkMode

# Find the veth pair
ip link show | grep -A1 -B1 "veth"

# Example output:
# 123: veth12345@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue master eni-abc123
#     link/ether 12:34:56:78:9a:bc brd ff:ff:ff:ff:ff:ff link-netnsid 1
```

**Step 3: Trace the link-local gateway (169.254.1.1)**
```bash
# From the node, examine routing for link-local traffic
ip route show table all | grep 169.254.1.1

# Check ARP table for link-local gateway
ip neigh show | grep 169.254.1.1

# Example output:
# 169.254.1.1 dev eni-abc123 lladdr 12:34:56:78:9a:bc REACHABLE
```

**The key insight:** The link-local address 169.254.1.1 is not a real gateway. It's a virtual gateway created by the AWS VPC CNI plugin that maps to the node's primary ENI.

#### B.3 AWS VPC CNI Link-Local Implementation

**Step 4: Understanding the CNI's link-local magic**
```bash
# Examine the ENI that serves as the "gateway"
ip addr show eni-abc123

# Check routing rules for the ENI
ip rule show
ip route show table main
ip route show table local

# Example output:
# ip addr show eni-abc123
# 4: eni-abc123: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP
#     inet 10.0.1.10/24 brd 10.0.1.255 scope global eni-abc123  # Node's primary IP
```

**Step 5: Tracing the actual egress path**
```bash
# From inside the pod, trace route to external destination
kubectl exec -it <pod-name> -- traceroute 8.8.8.8

# From the node, capture traffic to see the actual path
tcpdump -i eni-abc123 host 8.8.8.8 -n

# Check iptables rules that handle the traffic
iptables -t nat -L POSTROUTING -n -v
iptables -t filter -L FORWARD -n -v
```

#### B.4 The Complete Traffic Flow Analysis

**Detailed packet flow for pod egress:**

1. **Pod generates traffic:**
```bash
# Inside pod: curl https://api.github.com
# Packet: src=10.0.1.45 (pod IP), dst=140.82.112.3 (github.com)
# Route lookup: default via 169.254.1.1 dev eth0
```

2. **Traffic hits veth pair:**
```bash
# Packet moves from pod's eth0 to node's veth123
# Node receives packet on veth123 interface
# Source IP still: 10.0.1.45, Destination: 140.82.112.3
```

3. **Node routing decision:**
```bash
# Node routing table lookup
ip route get 140.82.112.3 from 10.0.1.45

# Example output:
# 140.82.112.3 from 10.0.1.45 via 10.0.1.1 dev eni-abc123 src 10.0.1.10
#     cache
```

4. **SNAT (Source NAT) transformation:**
```bash
# iptables POSTROUTING chain applies SNAT
iptables -t nat -L POSTROUTING -n -v | grep -A5 -B5 "10.0.1.45"

# Packet transformation:
# Before SNAT: src=10.0.1.45 (pod IP), dst=140.82.112.3
# After SNAT:  src=10.0.1.10 (node IP), dst=140.82.112.3
```

5. **Egress via ENI:**
```bash
# Packet exits via node's primary ENI
# AWS VPC routing takes over
# If private subnet: packet goes to NAT Gateway
# If public subnet: packet goes to Internet Gateway
```

#### B.5 Advanced Debugging Techniques

**Monitoring link-local traffic:**
```bash
# Monitor ARP traffic for link-local gateway
tcpdump -i any arp and host 169.254.1.1

# Monitor all traffic to/from link-local subnet
tcpdump -i any net 169.254.0.0/16

# Check conntrack entries for pod traffic
conntrack -L | grep 10.0.1.45
```

**Understanding AWS VPC CNI's iptables rules:**
```bash
# AWS VPC CNI creates specific iptables rules
iptables -t nat -L AWS-VPC-CNI-POSTROUTING -n -v
iptables -t filter -L AWS-VPC-CNI-FORWARD -n -v

# Example rules:
# Chain AWS-VPC-CNI-POSTROUTING (1 references)
# target     prot opt source               destination         
# MASQUERADE  all  --  10.0.1.0/24         !10.0.0.0/16        /* AWS VPC CNI */
```

**Debugging external SNAT mode:**
```bash
# Check if external SNAT is enabled
kubectl -n kube-system get daemonset aws-node -o yaml | grep AWS_VPC_K8S_CNI_EXTERNALSNAT

# If external SNAT is disabled (default), node does SNAT
# If external SNAT is enabled, NAT Gateway/Instance does SNAT
```

#### B.6 Common Link-Local Issues and Debugging

**Issue 1: Link-local gateway unreachable**
```bash
# Symptoms: Pod can't reach external services
# Debug from inside pod:
ping 169.254.1.1  # Should succeed
ip route get 8.8.8.8  # Should show via 169.254.1.1

# If ping fails, check veth pair:
# From node:
ip link show | grep veth
ethtool veth123  # Check if link is up
```

**Issue 2: ARP resolution failures**
```bash
# Check ARP table from pod's perspective
kubectl exec -it <pod> -- ip neigh show

# Should show:
# 169.254.1.1 dev eth0 lladdr xx:xx:xx:xx:xx:xx REACHABLE

# If FAILED or missing, check CNI plugin health:
kubectl -n kube-system logs -l k8s-app=aws-node
```

**Issue 3: SNAT not working**
```bash
# Check if pod traffic is being SNATed correctly
# From node, monitor outgoing traffic:
tcpdump -i eni-abc123 src host 10.0.1.45  # Should see no traffic (SNATed)
tcpdump -i eni-abc123 src host 10.0.1.10  # Should see SNATed traffic

# Check iptables SNAT rules:
iptables -t nat -L POSTROUTING -n -v | grep 10.0.1.45
```

#### B.7 Performance Implications of Link-Local Routing

**Understanding the overhead:**
```bash
# Measure latency through the link-local path
kubectl exec -it <pod> -- ping -c 10 169.254.1.1

# Compare with direct node communication
ping -c 10 10.0.1.10  # From another node

# Monitor CPU usage of network processing
top -p $(pgrep -f aws-node)
```

**Optimizing for high-throughput workloads:**
```bash
# Check network buffer sizes
kubectl exec -it <pod> -- cat /proc/sys/net/core/rmem_max
kubectl exec -it <pod> -- cat /proc/sys/net/core/wmem_max

# Monitor network interface statistics
kubectl exec -it <pod> -- cat /proc/net/dev
```

This deep dive shows that the "link-local gateway" at 169.254.1.1 is actually a clever abstraction by the AWS VPC CNI. It's not a real gateway but a virtual endpoint that allows pods to send traffic to the node's ENI through the veth pair, where iptables rules then handle SNAT and routing to the actual destination.

---

### Appendix C: Prometheus queries and EKS limits

> Tools and utilities are in [Appendix A.3](#a3-useful-tools-and-utilities) to avoid duplication.

#### C.2 Prometheus Queries for EKS Monitoring

**Node health:**
```promql
# Node CPU usage
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Node memory usage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Node disk usage
(1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100
```

**Pod resource usage:**
```promql
# Pod CPU usage
rate(container_cpu_usage_seconds_total[5m])

# Pod memory usage
container_memory_working_set_bytes

# Pod network I/O
rate(container_network_receive_bytes_total[5m])
rate(container_network_transmit_bytes_total[5m])
```

#### C.3 Common EKS Limits and Quotas

**EKS Service Limits:**
- Clusters per region: 100
- Node groups per cluster: 30
- Nodes per node group: 450
- Fargate profiles per cluster: 10

**EC2 Instance Limits (affects node groups):**
- Default vCPU limit varies by instance family
- Spot instance limits separate from On-Demand
- Elastic IP addresses: 5 per region (affects NAT Gateways)

**VPC Limits (affects networking):**
- VPCs per region: 5
- Subnets per VPC: 200
- Route tables per VPC: 200
- Security groups per VPC: 2,500

---

This comprehensive appendix provides essential reference materials, deep technical analysis of EKS networking, and practical tools for day-to-day EKS operations. The link-local address deep dive reveals the sophisticated networking abstraction that makes EKS pod networking appear simple while handling complex VPC integration behind the scenes.
