---
layout: post
title: Life of a Packet in Amazon EKS
category: aws, eks, network
mermaid: true
---

# Life of a Packet in Amazon EKS

If you already know Kubernetes architecture, skip to section 3.

---

## 1. Kubernetes architecture

Kubernetes has two planes. The **control plane** runs the API server, etcd, scheduler, controller manager, and cloud controller manager. The **data plane** is worker nodes where pods run. Each node has kubelet (starts pods, checks health) and kube-proxy (configures network rules for service traffic).

```
+----------------------- Kubernetes Cluster ---------------------+
|                                                                |
|  +--------------- Control Plane ----------------+              |
|  |                                              |              |
|  |  +-----------+  +------+  +-----------+      |              |
|  |  |API Server |  | etcd |  | Scheduler |      |              |
|  |  +-----------+  +------+  +-----------+      |              |
|  |  +------------------+  +----------------+    |              |
|  |  |Controller Manager|  |Cloud Controller|    |              |
|  |  +------------------+  +----------------+    |              |
|  +----------------------------------------------+              |
|                          |                                     |
|                     kubectl / API                              |
|                          |                                     |
|  +--------------- Data Plane -------------------------------+  |
|  |                                                          |  |
|  |  +--- Node 1 ---+  +--- Node 2 ---+  +--- Node N ---+    |  |
|  |  | kubelet      |  | kubelet      |  | kubelet      |    |  |
|  |  | kube-proxy   |  | kube-proxy   |  | kube-proxy   |    |  |
|  |  | +----++----+ |  | +----++----+ |  | +----++----+ |    |  |
|  |  | |Pod ||Pod | |  | |Pod ||Pod | |  | |Pod ||Pod | |    |  |
|  |  | +----++----+ |  | +----++----+ |  | +----++----+ |    |  |
|  |  +--------------+  +--------------+  +--------------+    |  |
|  +----------------------------------------------------------+  |
+----------------------------------------------------------------+
```

In EKS, the control plane lives in an AWS-managed VPC (not yours). It runs at least 2 API server instances and 3 etcd instances across 3 AZs, all in an EC2 Auto Scaling Group. The Kubernetes API sits behind an NLB.

Your worker nodes live in your VPC. They reach the control plane through cross-account ENIs (X-ENIs) in at least two AZs.

```
+---------- AWS-Managed VPC (EKS Service) ----------+
|                                                    |
|  +--- AZ-a ----+  +--- AZ-b -----+ +--- AZ-c ----+ |
|  | API Server   | | API Server   | |             | |
|  | + Scheduler  | | + Scheduler  | |             | |
|  | + Ctrl Mgr   | | + Ctrl Mgr   | |             | |
|  |              | |              | |             | |
|  |  etcd        | |  etcd        | |  etcd       | |
|  +------+-------+ +------+-------+ +------+------+ |
|         |                |                |        |
+---------+----------------+----------------+--------+
          |                |                |
     +----+----------------+----------------+----+
     |           NLB (Kubernetes API)            |
     +----+----------------+----------------+----+
          |                |                |
  ========+================+================+======== Cross-Account ENIs (X-ENIs)
          |                |                |
+---------+----------------+----------------+--------+
|  +------+------+ +------+------+ +------+-------+  |
|  |  Worker     | |  Worker     | |  Worker      |  |
|  |  Node(s)    | |  Node(s)    | |  Node(s)     |  |
|  |  AZ-a       | |  AZ-b       | |  AZ-c        |  |
|  +-------------+ +-------------+ +--------------+  |
|                                                    |
|                Your VPC                            |
+----------------------------------------------------+
```

---

## 2. The Kubernetes network model

Kubernetes requires that every pod gets its own IP, pods can talk to each other without NAT, and agents on a node can reach all pods on that node.

A pod can have multiple containers. They share a network namespace: they talk over localhost and share a single `eth0` for everything external.

```
+------------- Pod ----------------+
|                                  |
|  +-----------+  +-----------+    |
|  |Container A|  |Container B|    |
|  +-----+-----+  +-----+-----+    |
|        |   localhost   |         |
|        +-------+-------+         |
|                |                 |
|          +-----+------+          |
|          |    lo      |          |
|          | 127.0.0.1  |          |
|          +------------+          |
|          +------------+          |
|          |   eth0     |          |
|          | 10.0.3.42  |          |
|          +-----+------+          |
|                |                 |
+----------------+-----------------+
                 |
           to the network
```

---

## 3. How pods connect to the network

Each pod gets its own Linux network namespace, connected to the host's root namespace through a **veth pair**, a virtual Ethernet cable with one end in the pod and one end on the host.

How those veth interfaces plug into the host's IP stack varies by implementation. Kubernetes doesn't own this part. A spec called **CNI** (Container Network Interface) defines plugins that handle the wiring: creating the pod's interface, assigning an IP, setting up the veth pair.

Built-in plugins include loopback, bridge, and ipvlan. Third-party ones include Calico, Cilium, and Amazon VPC CNI.

```
+---- Pod Netns (Pod A) -------+      +---- Root Network Namespace (Node) ----------+
|                              |      |                                             |
|  +----------+                |      |    +----------+                             |
|  |   eth0   |<-- veth pair --+------+--->|  veth1   |                             |
|  |10.0.3.42 |                |      |    +----------+                             |
|  +----------+                |      |                                             |
|                              |      |    +----------+     +----------+            |
+------------------------------+      |    |  veth2   |     |  ENI-0   |--> VPC     |
                                      |    +----------+     | (primary)|            |
+---- Pod Netns (Pod B) -------+      |         ^           +----------+            |
|                              |      |         |           +----------+            |
|  +----------+                |      |         |           |  ENI-1   |--> VPC     |
|  |   eth0   |<-- veth pair --+------+---------+           |(secondary|            |
|  |10.0.3.55 |                |      |                     +----------+            |
|  +----------+                |      |                                             |
|                              |      |     Linux IP Stack / Routing Tables         |
+------------------------------+      +---------------------------------------------+
```

### What VPC CNI does

VPC CNI assigns pod IPs from the VPC CIDR using secondary IPs or prefix delegation on the node's EC2 ENIs. Pod IPs are real, routable VPC IPs, not overlay addresses.

As pods come and go, VPC CNI adds or removes ENIs on the node to keep enough IPs available. It also configures routing entries on the host and routing + ARP entries inside each pod's namespace.

---

## 4. Inside the pod and node

### Inside the pod

```
$ ip addr show                          $ ip route show
lo:   127.0.0.1/8                       default via 169.254.1.1 dev eth0
eth0: 10.0.3.42/32   <-- /32 mask!     169.254.1.1 dev eth0 scope link

$ arp -a
? (169.254.1.1) at ee:35:a3:c4:21:b7 [ether] PERM   <-- 'M' flag = manual entry
                     ^
                     |
                     +-- This MAC belongs to the veth on the HOST side
```

Three things to notice: the pod's `eth0` has a **/32 subnet mask**, the default gateway is **169.254.1.1** (a link-local address), and the ARP entry for that gateway is a **permanent manual entry** pointing to the host-side veth's MAC.

These three pieces form a system. Each one exists for a specific reason.

### Why `/32`?

An interface with a `/24` (say `10.0.3.42/24`) tells the kernel "there are 254 other hosts on this subnet, ARP for their MAC and send directly." The pod would broadcast ARP on the veth, bypassing the host's routing tables.

A `/32` means the subnet contains exactly one IP, the pod itself. No other IP is "on-link." The kernel routes every packet through the default gateway, which delivers it to the host-side veth. The node's policy-based routing tables take it from there.

Without the `/32`, pods could ARP for each other directly over the veth, bypassing host routing. That would break VPC CNI's control over traffic paths.

### Why a link-local address?

A link-local address (`169.254.0.0/16` for IPv4, `fe80::/10` for IPv6) is only valid on a single network link. Routers will never forward it. You've seen this range before: Windows/Mac APIPA fallback when DHCP fails, and the AWS metadata service at `169.254.169.254`.

VPC CNI uses `169.254.1.1` because it can't collide with any real VPC IP (VPCs use `10.x`, `172.16-31.x`, `192.168.x`), it can't leak beyond the veth pair, and it doesn't need coordination. Every pod on every node uses the same address.

Nobody actually "owns" `169.254.1.1`. The host-side veth has no IP at all. The trick works entirely through the `PERM` ARP entry: the kernel looks up `169.254.1.1` in its cache, finds the veth MAC, and sends the frame there. No ARP exchange happens on the wire. Running `arping -I eth0 169.254.1.1` from inside the pod returns zero responses. Normal traffic works fine because the kernel uses the cache, not the wire.

### On the node

The node has multiple routing tables. Policy-based routing uses the source IP of the traffic to pick which table to consult.

```
+--- Node Routing Architecture ----------------------------------------+
|                                                                      |
|  Policy-Based Routing Table (ip rule)                                |
|  +---------------------------------------------------------------+   |
|  | from 10.0.3.42  lookup main    <-- Pod 51 (sec IP on ENI-1)   |   |
|  | from 10.0.3.55  lookup 2       <-- Pod 61 (sec IP on ENI-2)   |   |
|  | from all        lookup main                                   |   |
|  +----------+----------------------------+-----------------------+   |
|             |                            |                           |
|             v                            v                           |
|  +--- Main Routing Table ---+  +--- Routing Table 2 -------+         |
|  |                          |  |                            |        |
|  | 10.0.3.42 dev veth1      |  | default via 10.0.0.1       |        |
|  | 10.0.3.55 dev veth2      |  |         dev eni2           |        |
|  | 10.0.0.0/24 dev eni0     |  | (single entry!)            |        |
|  | default via 10.0.0.1     |  |                            |        |
|  |         dev eni0         |  +----------------------------+        |
|  +---------------------------+                                       |
|                                                                      |
|  The main table has routes to local pods AND the subnet.             |
|  Table 2 only has a default gateway. Traffic from pods on            |
|  ENI-2 always hits the VPC router, even for same-subnet              |
|  destinations.                                                       |
+----------------------------------------------------------------------+
```

---

## 5. Packet walks

### 5a. Ingress to a pod

Traffic arrives at the node for Pod 61 (a secondary IP on ENI 2):

```
                   +--------------- Node -------------------+
                   |                                        |
 Incoming traffic  |  Policy Routing    Main Routing Table  |
 dst: Pod 61 IP ->-|  ------------->   ------------------>  |
                   |  "lookup main"     "Pod 61 -> veth2"   |
                   |                          |             |
                   |                          v             |
                   |                   +--------------+     |
                   |                   |   Pod 61     |     |
                   |                   |  (via veth2) |     |
                   |                   +--------------+     |
                   +----------------------------------------+
```

Policy routing says "look up main table." Main table has a `/32` route for Pod 61 pointing at its veth.

### 5b. Pod egress to the VPC

Pod 61 sends traffic somewhere in the VPC:

```
+---------------- Node ------------------------------------------+
|                                                                |
|  +--------+   Policy Routing       Routing Table 2             |
|  | Pod 61 |-->--------------->    ------------------>          |
|  |        |   "from Pod61 IP       "default gw via ENI-2"      |
|  +--------+    lookup table 2"            |                    |
|                                           v                    |
|                                     +----------+               |
|                                     |  ENI-2   |----> VPC Router
|                                     +----------+               |
|                                                                |
+----------------------------------------------------------------+
```

Source IP is Pod 61's, so policy routing sends it to table 2. Table 2 only has a default gateway through ENI-2. Traffic always goes to the VPC router, even if the destination is on the same subnet. There's no other route in that table.

### 5c. Pod-to-pod, same node

Pod 51 talks to Pod 61, both on the same node:

```
+------------------------- Node --------------------------------+
|                                                               |
|  +--------+                                     +--------+    |
|  | Pod 51 |                                     | Pod 61 |    |
|  +---+----+                                     +---^----+    |
|      | src MAC: Pod51 MAC                           |         |
|      | dst MAC: veth1 MAC                           |         |
|      v                                              |         |
|  +--------+   Policy    Main Table    +--------+    |         |
|  | veth1  |-->Routing-->Pod61->veth2->| veth2  |----+         |
|  +--------+   "lookup                 +--------+              |
|                main"     src MAC: veth2 MAC                   |
|                           dst MAC: Pod61 MAC                  |
|                                                               |
|           No ENIs involved -- traffic stays within the node   |
+---------------------------------------------------------------+
```

Main table, `/32` route for Pod 61, forwarded through its veth. Never touches an ENI.

### 5d. Pod-to-pod, across nodes

Pod 51 on Node A to Pod 81 on Node B:

```
+----------- Node A ----------------+          +----------- Node B -----------------+
|                                    |          |                                   |
| +--------+                         |          |                       +--------+  |
| | Pod 51 |                         |          |                       | Pod 81 |  |
| +---+----+                         |          |                       +---^----+  |
|     |                              |          |                           |       |
|     v                              |          |                           |       |
|  Policy -> Main Table              |          |  Policy -> Main Table     |       |
|  "Pod81 IP on same subnet          |          |  "Pod81 -> veth"          |       |
|   as ENI-0 -> forward via ENI-0"   |          |                           |       |
|           |                        |          |      +--------+           |       |
|           v                        |          |      | veth   |-----------+       |
|     +----------+                   |          |      +----^---+                   |
|     |  ENI-0   |-------------------+---(VPC)--+----------+                        |
|     +----------+                   |          |      +--------+                   |
|                                    |          |      | ENI-0  |                   |
| src MAC: NodeA ENI-0 MAC           |          |      +--------+                   |
| dst MAC: NodeB ENI-0 MAC           |          |                                   |
+------------------------------------+          | src MAC: veth MAC                 |
                                                | dst MAC: Pod81 MAC                |
                                                +-----------------------------------+
```

Node A's main table sees Pod 81's IP on the same subnet as ENI-0, forwards via ENI-0 across the VPC. Node B receives it, main table finds the `/32` route for Pod 81, delivers through the veth.

### 5e. The return path (secondary ENI asymmetry)

Pod 81 responds.

```
+----------- Node B ----------------+          +----------- Node A -----------------+
|                                    |          |                                   |
| +--------+                         |          |                       +--------+  |
| | Pod 81 |                         |          |                       | Pod 51 |  |
| +---+----+                         |          |                       +---^----+  |
|     |                              |          |                           |       |
|     v                              |          |  Policy -> Main Table     |       |
|  Policy -> Table 2                 |          |  "Pod51 -> veth"          |       |
|  "default gw via ENI-2" <--!       |          |                           |       |
|           |                        |          |                           |       |
|           v                        |          |                           |       |
|     +----------+                   |          |     +----------+          |       |
|     |  ENI-2   |-----+             |          |     |  ENI-0   |----------+       |
|     +----------+     |             |          |     +-----^----+                  |
|                      |             |          |           |                       |
+----------------------+-------------+          +-----------+-----------------------+
                       |                                    |
                       v                                    |
               +--------------+                             |
               |  VPC Router  |-----------------------------+
               | (default gw) |
               +--------------+

    Even though Pod 81 and Pod 51 are on the SAME SUBNET,
    traffic goes through the VPC router because routing
    table 2 only has a default gateway entry!
```

Pod 81 is on a secondary ENI, so its traffic uses table 2. Table 2 only knows the default gateway. Even though Pod 51 is on the same subnet, the response goes through the VPC router. Extra hop, but transparent.

---

## 6. Kubernetes Services

Pods are ephemeral. They die, get recreated, come back with new IPs. This can happen thousands of times a day. Other services can't track individual pod IPs.

A **Kubernetes Service** groups pods by label selectors and gives them a stable virtual IP (the "ClusterIP"). An **endpoints controller** keeps the backing pod list current.

```
                          +----------------------+
                          |  Kubernetes Service  |
                          |  name: app1-service  |
                          |  VIP: 172.20.0.100   |
                          |  selector: name=app1 |
                          +----------+-----------+
                                     |
                         +-----------+-----------+
                         |           |           |
                    +----v---+  +----v---+  +----v---+
                    | Pod    |  | Pod    |  | Pod    |
                    | app1   |  | app1   |  | app1   |
                    |10.0.1.5|  |10.0.2.8|  |10.0.3.2|
                    |(ep 1)  |  |(ep 2)  |  |(ep 3)  |
                    +--------+  +--------+  +--------+
                     Node A      Node B      Node C
```

Three service types, each building on the previous:

| Type | What it does |
|------|-------------|
| **ClusterIP** | Virtual IP reachable only inside the cluster |
| **NodePort** | Opens a port on every node, forwards to the service. Built on ClusterIP. |
| **LoadBalancer** | Provisions a cloud LB in front of NodePort. Built on NodePort. |

---

## 7. ClusterIP

When you create a ClusterIP service, kube-proxy watches the API server and programs iptables rules for the service's VIP on every node. Kubernetes DNS assigns a name like `app1-service.default.svc.cluster.local`.

```
       Node 51                                     Node 71
+-----------------------------------------+   +----------------------+
|                                         |   |                      |
| +-------+ +-------------------------+   |   | +-------+            |
| |Pod 51 |>|      iptables           |   |   | |Pod 71 | (part of   |
| |(app2) | |                         |   |   | |(app1) |  app1 svc) |
| +-------+ | 1. Load balance:        |   |   | +---^---+            |
|  dst:     |    pick Pod 71 IP       |   |   |     |                |
|  SVC VIP  |    (round-robin, may    |   |   |     |                |
|           |     pick ANY pod, even  |   |   | +---+---------------+|
|           |     local ones are not  |   |   | | Node 71 forwards  ||
|           |     preferred!)         |   |   | | to local Pod 71   ||
|           |                         |   |   | +-------------------+|
|           | 2. DNAT:                |   |   |                      |
|           |    dst: VIP -> Pod71 IP |   |   +---+------------------+
|           |                         |   |       ^
|           | 3. Mark flow:           |   |       |
|           |    (stateful tracking   |   |   +---+
|           |     for return traffic) |   |   |
|           +----------+--------------+   |   |
|                      |                  |   |
|                      v                  |   |
|               Forward to Node 71 ------+---+
|               dst IP: Pod 71 IP        |
|                                        |
+----------------------------------------+
```

iptables picks a backend pod (round-robin, no preference for local pods), DNATs the destination from the VIP to the pod IP, and marks the flow for stateful tracking.

Pod 51 doesn't know the VIP by default. It resolves the service DNS name first, and that DNS query itself goes through a ClusterIP service (kube-dns):

```
Pod 51 --DNS query--> kube-dns Service VIP --iptables DNAT--> CoreDNS Pod
                                                                   |
Pod 51 <--DNS response (Service VIP: 172.20.0.100)----------------+
```

On the return path, iptables on Node 51 matches the response as return traffic (stateful match) and SNATs the source IP from Pod 71 back to the service VIP. Pod 51 never sees Pod 71's IP.

```
 Node 71                                    Node 51
+------------------+    +---------------------------------------------+
|                  |    |                                             |
|  +-------+       |    |    +-----------------------------+ +-------+|
|  |Pod 71 |-------+----+--->|         iptables            |>|Pod 51 ||
|  |       |       |    |    |                             | |       ||
|  +-------+       |    |    | 1. Identify return traffic  | +-------+|
|  src: Pod71 IP   |    |    |    (stateful match)         |          |
|  dst: Pod51 IP   |    |    |                             |          |
|                  |    |    | 2. SNAT:                    |          |
+------------------+    |    |    src IP: Pod71 -> SVC VIP |          |
                        |    |    (Pod 51 thinks it's      |          |
                        |    |     talking to the VIP,     |          |
                        |    |     not Pod 71 directly)    |          |
                        |    +-----------------------------+          |
                        +---------------------------------------------+
```

kube-proxy can also use IPVS or eBPF instead of iptables for better load balancing characteristics.

---

## 8. NodePort

NodePort exposes applications externally, mostly for testing. It builds on ClusterIP but also configures iptables to listen on a specific port (range 30000-32767) on every node.

```
       Node 51 (receives traffic)                Node 71
+--------------------------------------------+  +------------------+
|                                            |  |                  |
| External +--------------------------------+|  | +-------+        |
| Client ->|       iptables (4 tasks)       ||  | |Pod 71 |        |
|          |                                ||  | |(app1) |        |
| dst:     | 1. Load balance -> Pod 71 IP   ||  | +---^---+        |
| Node51   |    (may pick remote pod even   ||  |     |            |
| :31234   |     if local pod exists!)      ||  |     |            |
|          |                                ||  +-----+------------+
|          | 2. DNAT: dst -> Pod 71 IP      ||        |
|          |                                ||        |
|          | 3. SNAT: src -> Node 51 IP     |+--------+
|          |    (flow symmetry! without     ||
|          |     this, Pod71 would respond  ||
|          |     directly to client from    ||
|          |     a different IP, breaking   ||
|          |     the connection)            ||
|          |                                ||
|          | 4. Mark flow (stateful)        ||
|          +--------------------------------+|
+--------------------------------------------+
```

The difference from ClusterIP: iptables now does four things instead of three. The extra one is SNAT, rewriting the source IP to the node's IP. Without it, Pod 71 would respond directly to the client from a different IP, and the client would drop the response:

```
WITHOUT SNAT (broken):                   WITH SNAT (works):

Client --> Node51:31234                   Client --> Node51:31234
             |  dst NAT to Pod71               |  dst NAT to Pod71
             v                                 |  src NAT to Node51 IP
           Pod71                               v
             |                               Pod71
             |  responds to Client IP          |
             v  src: Pod71 IP  X BROKEN!       |  responds to Node51 IP
           Client sees response from           v
           unknown IP -- drops it!           Node51
                                               |  reverse NAT
                                               v
                                             Client sees response from
                                             Node51:31234  OK works!
```

The client IP is always SNATed away, even when the destination pod is on the same node. The application never sees the real client IP.

The operational problem: you need to track node IPs (which change as nodes fail and get replaced) and distribute traffic across them. You need a load balancer.

---

## 9. LoadBalancer

LoadBalancer builds on NodePort. A service controller provisions a cloud load balancer that forwards to the NodePort on each node.

AWS has two controllers:

| Controller | Source | Provisions |
|---|---|---|
| Service controller | Built into K8s | CLB (legacy) or NLB |
| AWS Load Balancer Controller | K8s SIG project on GitHub | NLB + ALB, Target Type IP, Ingress |

### Default behavior (target type = instance)

```
                                              +------------ PROBLEM ---------------+
                                              | Node 51 has NO pods for this       |
                                              | service, but NLB thinks it's       |
                                              | healthy because the NodePort       |
                                              | health check passes on ALL nodes   |
                                              +------------------------------------+

  Client              NLB                  Node 51                  Node 71
    |                  |                     |                        |
    |--- request ----->|                     |                        |
    |   dst: NLB IP    |                     |                        |
    |   port: 80       |                     |                        |
    |                  |-- forward --------->|                        |
    |                  |   dst: Node51 IP    |                        |
    |                  |   port: 31234       |                        |
    |                  |   (NodePort)        |                        |
    |                  |                     | iptables:              |
    |                  |                     |  1. LB -> Pod71 IP     |
    |                  |                     |  2. DNAT dst -> Pod71  |
    |                  |                     |  3. SNAT src -> Node51 |
    |                  |                     |  4. Mark flow          |
    |                  |                     |                        |
    |                  |                     |---- forward ---------> |
    |                  |                     |                        | -> Pod 71
    |                  |                     |                        |
    |                  |                     |<--- response --------- |
    |                  |<-- response --------|                        |
    |                  |    (after iptables  |                        |
    |<-- response -----|     reverse NAT)    |                        |
    |                  |                     |                        |
```

NLB health-checks the NodePort, and all nodes pass, even ones with no pods for that service. Traffic can land on Node 51 (no local pod) and bounce to Node 71 (has the pod). This is **traffic tromboning**: extra hops, latency, cross-AZ data transfer charges, and the client IP gets SNATed.

### Fix 1: externalTrafficPolicy: Local

```yaml
spec:
  externalTrafficPolicy: Local
```

```
  Client              NLB                   Node 51            Node 71
    |                  |                      |                   |
    |--- request ----->|                      |                   |
    |                  |                      |                   |
    |                  |  health check        |                   |
    |                  |  (NodePort) -------->| PASS              |
    |                  |                      |                   |
    |                  |  health check        |                   |
    |                  |  (EXTRA port) ------>| FAIL              |
    |                  |  "any local pods?"   |  (no local pods)  |
    |                  |                      |                   |
    |                  |  health check        |                   |
    |                  |  (EXTRA port) -------+------------------>| PASS
    |                  |                      |                   |  (has Pod71)
    |                  |                      |                   |
    |                  |-- forward directly --+------------------>|
    |                  |   (skips Node 51!)   |                   |
    |                  |                      |                   | iptables:
    |                  |                      |                   |  DNAT only
    |                  |                      |                   |  (no SNAT!)
    |                  |                      |                   |  -> Pod 71
    |                  |                      |                   |
    |<-----------------+----------------------+--- response ----- |
    |                  |                      |  client IP visible|
```

NLB runs an additional health check on a different port. Nodes without local pods fail it. iptables only forwards to local pods and skips SNAT, so the application sees the real client IP.

Trade-off: uneven pod distribution means uneven traffic distribution.

### Fix 2: Target Type IP (requires AWS Load Balancer Controller)

NLB targets pods directly instead of nodes.

```
  Client              NLB                            Node 71
    |                  |                                |
    |--- request ----->|                                |
    |   dst: NLB IP    |                                |
    |                  |                                |
    |                  |  Target Group:                 |
    |                  |  +----------------------+      |
    |                  |  | Pod71 IP    healthy  |      |
    |                  |  | Pod81 IP    healthy  |      |
    |                  |  | Pod91 IP    healthy  |      |
    |                  |  | (pods, NOT nodes!)   |      |
    |                  |  +----------------------+      |
    |                  |                                |
    |                  |------ forward ---------------->|
    |                  |  dst IP: Pod71 IP directly     |
    |                  |                                |
    |                  |       iptables does NOTHING    |
    |                  |       (dst IP already is       |
    |                  |        Pod71 IP)               |
    |                  |                                | -> Pod 71
    |                  |                                |
    |<-----------------+-------- response ------------- |
    |                  |                                |
```

No NodePort, no iptables. The destination IP is already the pod IP, so the node just forwards through the veth. Source IP is the NLB's by default; enable `preserve client IP` via annotation if needed.

Watch ELB service quotas. Thousands of pods means thousands of targets in a single target group.

---

## 10. Ingress (Layer 7)

Everything above is Layer 4. Kubernetes Services don't understand HTTP paths or hostnames.

**Ingress** handles Layer 7 routing. You define rules like `/order` goes to order-service, `/rating` goes to rating-service:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
spec:
  rules:
  - http:
      paths:
      - path: /order
        backend:
          service:
            name: order-service
      - path: /rating
        backend:
          service:
            name: rating-service
```

Kubernetes expects an Ingress controller to provision a load balancer that implements these rules. The AWS Load Balancer Controller provisions an ALB.

With Target Type IP, the ALB sends traffic directly to pods:

```
  Client              ALB                          Node 71
    |                  |                              |
    |-- GET /order --->|                              |
    |                  |                              |
    |                  |  L7 routing decision:        |
    |                  |  /order -> order-service     |
    |                  |  pick Pod 71 from targets    |
    |                  |                              |
    |                  |---- forward ---------------->|
    |                  |  dst: Pod71 IP               |
    |                  |  src: ALB IP  (always SNAT!) |
    |                  |                              |
    |                  |     iptables: does NOTHING   |
    |                  |                              | -> Pod 71
    |                  |                              |
    |<-- response -----|<---- response -------------- |
    |  src: ALB IP     |                              |
```

ALB always SNATs. You can't preserve the client IP at L4, but `X-Forwarded-For` headers carry it at L7.

### Full end-to-end: DNS to pod

```
  Client            Route 53       Internet GW        ALB                Pod
    |                  |               |                |                  |
    | DNS: portal.     |               |                |                  |
    | example.com ---->|               |                |                  |
    |                  |               |                |                  |
    |<- alias record --|               |                |                  |
    |   returns ALB    |               |                |                  |
    |   public IPs:    |               |                |                  |
    |   [IP1,IP2,IP3]  |               |                |                  |
    |                  |               |                |                  |
    | picks IP3                        |                |                  |
    |-------- request (dst: IP3) ----->|                |                  |
    |                                  |                |                  |
    |                                  | DNAT:          |                  |
    |                                  | dst: Public IP3|                  |
    |                                  |  -> ALB Private|                  |
    |                                  |     IP (AZ-c)  |                  |
    |                                  |                |                  |
    |                                  |--- forward --->|                  |
    |                                  |                |                  |
    |                                  |                | L7 routing       |
    |                                  |                | decision         |
    |                                  |                |                  |
    |                                  |                | Cross-zone LB ON |
    |                                  |                | may pick pod in  |
    |                                  |                | ANY AZ           |
    |                                  |                |                  |
    |                                  |                |--- forward ----->|
    |                                  |                |  dst: Pod IP     |
    |                                  |                |  src: ALB IP     |
    |                                  |                |                  |
    |<------------------------- response ----------------------------------|
```

Route 53 returns ALB public IPs. The Internet Gateway DNATs the public IP to the ALB's private IP. Cross-zone load balancing is on by default, so the ALB may forward to any AZ. Disable it for AZ-local routing.

---

## 11. Pod egress

By default (`AWS_VPC_K8S_CNI_EXTERNALSNAT=false`), VPC CNI applies SNAT on the node itself for all pod traffic leaving the VPC CIDR. The pod IP gets rewritten to the node's primary ENI IP before the packet even leaves the node:

```
+------------ Node ----------------------+
|                                        |
|  +---------------+                     |
|  |    Pod        |                     |
|  | 192.168.1.51  |                     |
|  +-------+-------+                     |
|          | dst: 8.8.8.8 (internet)     |        Internet Gateway
|          | src: 192.168.1.51           |             |
|          v                             |             |
|  +---------------+                     |             |
|  |   VPC CNI     |                     |             |
|  |   SNAT #1     |                     |             |
|  |   src: .51 -> |                     |             |
|  |   .50 (node   |                     |             |
|  |   primary ENI |                     |             |
|  |   primary IP) |                     |             |
|  +-------+-------+                     |             |
|          | src: 192.168.1.50           |             |
|          v                             |             |
|     +----------+                       |             |
|     | ENI-0    |-----------------------+------------>|
|     | (primary)|                       |             | SNAT #2
|     +----------+                       |             | src: .50 -> Public IP
|                                        |             | (associated with .50)
+----------------------------------------+             |
                                                       +-------> Internet
                                                       |  src: Public IP
                                                       |  dst: 8.8.8.8
```

Two-stage SNAT: VPC CNI rewrites the pod IP to the node's primary ENI IP, then the IGW/NAT Gateway rewrites it to the public IP.

### AWS_VPC_K8S_CNI_EXTERNALSNAT

The default (`false`) means VPC CNI does SNAT on the node. All pods on a node share that node's IP for outbound traffic. External services see one IP per node, not per pod.

Setting `AWS_VPC_K8S_CNI_EXTERNALSNAT=true` disables the node-level SNAT. The pod's real IP survives all the way to the NAT Gateway, which does the only SNAT (pod IP directly to public IP). The node doesn't rewrite anything.

```
EXTERNALSNAT=false (default):           EXTERNALSNAT=true:

Pod (.51) --> Node SNAT (.50) -->       Pod (.51) --> Node (no SNAT) -->
  NAT GW SNAT (public IP) -->            NAT GW SNAT (public IP) -->
  Internet                                Internet

External sees: node IP                  External sees: pod IP
(all pods on this node                  (until NAT GW, where it
 look the same)                          becomes the public IP)
```

When `EXTERNALSNAT=true`:
- Pod IPs are visible in VPC flow logs and security group tracking, making it easier to trace which pod is talking to what
- External APIs that rate-limit by source IP won't treat all pods on a node as one client
- Service meshes (Istio, Linkerd) work correctly since they expect pod IPs in the traffic
- You need your VPC routing and NAT Gateway set up to handle pod CIDR traffic (the pod IPs need a route to the NAT Gateway)

```bash
# check current setting
kubectl get daemonset aws-node -n kube-system -o json | \
  jq '.spec.template.spec.containers[0].env[] | select(.name=="AWS_VPC_K8S_CNI_EXTERNALSNAT")'

# enable it
kubectl set env daemonset -n kube-system aws-node AWS_VPC_K8S_CNI_EXTERNALSNAT=true
```

The experiments in the appendix were run with `EXTERNALSNAT=true`. That's why the `checkip.amazonaws.com` result shows the NAT Gateway's public IP directly, not the node IP — VPC CNI isn't doing node-level SNAT.

---

## Summary

Inbound external traffic, from least to most efficient:

```
  LEAST EFFICIENT                                            MOST EFFICIENT
  <--------------------------------------------------------------------------->

  LoadBalancer         + externalTraffic    + Target Type IP   Ingress + ALB
  (default)              Policy: Local       (NLB)             + Target IP
  +----------+         +----------+         +----------+      +----------+
  | NLB      |         | NLB      |         | NLB      |      | ALB      |
  |   |      |         |   |      |         |   |      |      |   |      |
  | Node     |         | Node     |         | Pod      |      | Pod      |
  | (any!)   |         | (w/ pod) |         | directly |      | directly |
  |   |      |         |   |      |         |          |      |          |
  | iptables |         | iptables |         | no       |      | no       |
  | DNAT+SNAT|         | DNAT only|         | iptables |      | iptables |
  |   |      |         |   |      |         |          |      |          |
  | Node     |         | Pod      |         +----------+      +----------+
  | (maybe)  |         | (local)  |
  |   |      |         |          |         Client IP:         Client IP:
  | Pod      |         +----------+         via annotation    via X-Forwarded-For
  +----------+
                       Client IP:
  Client IP:           visible
  SNATed away
```

| Approach | Hops | SNAT? | Client IP visible? | iptables? |
|----------|------|-------|-------------------|-----------|
| LoadBalancer (default) | NLB -> Node -> maybe another Node -> Pod | DNAT+SNAT | No | Yes |
| + `externalTrafficPolicy: Local` | NLB -> Node (with pod) -> Pod | DNAT only | Yes | Yes |
| + Target Type IP (NLB) | NLB -> Pod | NLB IP | Via annotation | No |
| Ingress + ALB + Target IP | ALB -> Pod | ALB IP | Via X-Forwarded-For | No |

---

## Appendix: Experiments on a live cluster

Everything above was verified on a live EKS cluster using two [netshoot](https://github.com/nicolaka/netshoot) toolbox pods and an echoserver deployment. If you want to follow along, apply these manifests:

**tools.yaml** — namespace + netshoot toolbox pod for debugging:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: packetlab
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: toolbox
  namespace: packetlab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: toolbox
  template:
    metadata:
      labels:
        app: toolbox
    spec:
      containers:
        - name: netshoot
          image: ghcr.io/nicolaka/netshoot:latest
          imagePullPolicy: IfNotPresent
          command: ["sleep", "infinity"]
          securityContext:
            allowPrivilegeEscalation: true
            capabilities:
              add: ["NET_ADMIN", "NET_RAW"]
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

**echoserver.yaml** — echo server with ClusterIP and NodePort services:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echoserver
  namespace: packetlab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echoserver
  template:
    metadata:
      labels:
        app: echoserver
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: echoserver
      containers:
        - name: echoserver
          image: registry.k8s.io/echoserver:1.10
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: echo-clusterip
  namespace: packetlab
spec:
  type: ClusterIP
  selector:
    app: echoserver
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: echo-nodeport
  namespace: packetlab
spec:
  type: NodePort
  selector:
    app: echoserver
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
```

```bash
$ kubectl apply -f tools.yaml
$ kubectl apply -f echoserver.yaml
```

```bash
$ kubectl -n packetlab get pods -o wide
NAME                         READY   STATUS    IP              NODE
toolbox-a-6fb5b9b786-lszr4   1/1     Running   10.0.1.51   ip-10-0-0-75.ec2.internal
toolbox-b-754f9d4d47-qbnzl   1/1     Running   10.0.2.29   ip-10-0-0-49.ec2.internal
echoserver-545755bc68-v5kdv  1/1     Running   10.0.3.193  ip-10-0-1-228.ec2.internal
```

- Toolbox A: `10.0.1.51` on node `10.0.0.75`
- Toolbox B: `10.0.2.29` on node `10.0.0.49`
- Echoserver: `10.0.3.193` on node `10.0.1.228`

### Experiment 1: Pod interfaces, routes, and ARP

```bash
$ kubectl -n packetlab exec toolbox-a-6fb5b9b786-lszr4 -- ip addr show
3: eth0@if336: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP group default
    link/ether 0a:12:c5:d0:e4:e1 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.1.51/32 scope global eth0         <-- /32!

$ kubectl -n packetlab exec toolbox-a-6fb5b9b786-lszr4 -- ip route show
default via 169.254.1.1 dev eth0
169.254.1.1 dev eth0 scope link

$ kubectl -n packetlab exec toolbox-a-6fb5b9b786-lszr4 -- arp -a
? (169.254.1.1) at 5a:f3:56:0b:86:d4 [ether] PERM on eth0

$ kubectl -n packetlab exec toolbox-a-6fb5b9b786-lszr4 -- cat /etc/resolv.conf
search packetlab.svc.cluster.local svc.cluster.local cluster.local ec2.internal
nameserver 172.20.0.10
options ndots:5
```

`/32` on eth0, `169.254.1.1` fake gateway, `PERM` ARP entry with MAC `5a:f3:56:0b:86:d4`, kube-dns at `172.20.0.10`.

### Experiment 2: Node-side veth, ENIs, and routing tables

SSH to the node:

```bash
$ ip link show eniaee705c1f69
336: eniaee705c1f69@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP
    link/ether 5a:f3:56:0b:86:d4 brd ff:ff:ff:ff:ff:ff link-netns cni-01fc068f-c24b-470b-7551-ac4d8bc24a67
```

MAC `5a:f3:56:0b:86:d4` matches the pod's ARP entry. The pod sends frames here thinking it's the gateway.

```bash
$ ip addr show ens5   # Primary ENI: 10.0.0.75 (main table)
$ ip addr show ens6   # Secondary ENI: 10.0.0.85 (table 2)
$ ip addr show ens7   # Third ENI: 10.0.0.84 (table 3)
```

| ENI | Interface | IP | Routing Table |
|-----|-----------|-----|--------------|
| Primary | ens5 | 10.0.0.75 | main |
| Secondary | ens6 | 10.0.0.85 | 2 |
| Third | ens7 | 10.0.0.84 | 3 |

```bash
$ ip rule show
512:    from all to 10.0.1.51 lookup main     <-- ingress TO our pod
1536:   from 10.0.1.51 lookup 2               <-- egress FROM our pod

$ ip route show table main
default via 10.0.0.1 dev ens5 proto dhcp src 10.0.0.75 metric 512
10.0.0.0/24 dev ens5 proto kernel scope link src 10.0.0.75 metric 512
10.0.1.51 dev eniaee705c1f69 scope link        <-- our pod's veth

$ ip route show table 2
default via 10.0.0.1 dev ens6
10.0.0.1 dev ens6 scope link
```

Table 2: two entries, just the default gateway via ens6. Pods on secondary ENIs always go through the VPC router.

### Experiment 3: Pod-to-pod connectivity

```bash
$ kubectl -n packetlab exec toolbox-a-6fb5b9b786-lszr4 -- ping -c 3 10.0.0.75
64 bytes from 10.0.0.75: icmp_seq=1 ttl=127 time=0.062 ms     <-- same node, ~0.06ms

$ kubectl -n packetlab exec toolbox-a-6fb5b9b786-lszr4 -- ping -c 3 10.0.2.29
64 bytes from 10.0.2.29: icmp_seq=1 ttl=125 time=1.67 ms      <-- cross-node, ~0.4ms
64 bytes from 10.0.2.29: icmp_seq=2 ttl=125 time=0.424 ms
```

TTL 125 = three hops (pod to host, host to VPC router, VPC router to remote host).

### Experiment 4: DNS through ClusterIP

```bash
$ kubectl get svc -n kube-system kube-dns -o wide
NAME       TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                  SELECTOR
kube-dns   ClusterIP   172.20.0.10   <none>        53/UDP,53/TCP,9153/TCP   k8s-app=kube-dns

$ kubectl -n packetlab exec toolbox-a-6fb5b9b786-lszr4 -- dig +short kubernetes.default.svc.cluster.local
172.20.0.1
```

iptables on the node:

```bash
$ sudo iptables -t nat -L KUBE-SVC-TCOU7JCQXEZGVUNU -n
Chain KUBE-SVC-TCOU7JCQXEZGVUNU (1 references)
target                      prot opt source    destination
KUBE-SEP-43APF7RZ6NYHDOWA  all  --  0.0.0.0/0  0.0.0.0/0  /* -> 10.0.1.181:53 */ statistic mode random probability 0.50000000000
KUBE-SEP-5PFK5H2IGDZ73DVD  all  --  0.0.0.0/0  0.0.0.0/0  /* -> 10.0.3.231:53 */
```

50/50 random split across two CoreDNS pods.

### Experiment 5: ClusterIP service

```bash
$ kubectl -n packetlab get svc echo-clusterip -o wide
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   SELECTOR
echo-clusterip   ClusterIP   172.20.122.189   <none>        80/TCP    app=echoserver

$ kubectl -n packetlab exec toolbox-a-6fb5b9b786-lszr4 -- curl -s http://echo-clusterip.packetlab.svc.cluster.local
Request Information:
    client_address=10.0.1.51          <-- real pod IP, no SNAT
```

iptables DNAT chain:

```bash
$ sudo iptables -t nat -L KUBE-SVC-B23I5IPCVQPNADR7 -n
KUBE-SEP-MNWX5K5OIII64PTW  all  --  0.0.0.0/0  0.0.0.0/0  /* -> 10.0.2.127:8080 */ statistic mode random probability 0.50000000000
KUBE-SEP-XAUWYAMA2TYGF5RO  all  --  0.0.0.0/0  0.0.0.0/0  /* -> 10.0.3.193:8080 */
```

Conntrack after a request:

```bash
$ sudo conntrack -L | grep 172.20.122.189
tcp  6  60  TIME_WAIT
  src=10.0.1.51 dst=172.20.122.189 sport=59432 dport=80
  src=10.0.3.193 dst=10.0.1.51 sport=8080 dport=59432
  [ASSURED] mark=0
```

Original: toolbox -> service VIP. Reply: echoserver -> toolbox. iptables reverse-NATs so the toolbox thinks it talked to the VIP.

### Experiment 6: NodePort service

```bash
$ kubectl -n packetlab get svc echo-nodeport -o wide
NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        SELECTOR
echo-nodeport   NodePort   172.20.152.84   <none>        80:32385/TCP   app=echoserver
```

iptables: `KUBE-MARK-MASQ` fires on all NodePort traffic (not just hairpin like ClusterIP), marking packets for SNAT in `KUBE-POSTROUTING`:

```bash
$ sudo iptables -t nat -L KUBE-POSTROUTING -n
RETURN      all  --  0.0.0.0/0  0.0.0.0/0  mark match ! 0x4000/0x4000
MARK        all  --  0.0.0.0/0  0.0.0.0/0  MARK xor 0x4000
MASQUERADE  all  --  0.0.0.0/0  0.0.0.0/0  /* kubernetes service traffic requiring SNAT */ random-fully
```

From the same node:

```bash
$ curl -s http://localhost:32385
    client_address=10.0.0.75          <-- node IP, not us
```

From a different node:

```bash
$ ssh ip-10-0-0-49.ec2.internal -- curl -s http://10.0.0.75:32385
    client_address=10.0.0.75          <-- still the receiving node
```

Both times echoserver sees the receiving node's IP. The actual client gets SNATed for flow symmetry.

### Experiment 7: Pod egress

```bash
$ kubectl -n packetlab exec toolbox-a-6fb5b9b786-lszr4 -- curl -s https://checkip.amazonaws.com
44.218.31.211
```

Neither pod IP nor node IP. Two-stage SNAT: VPC CNI rewrites pod IP to node primary ENI IP, then IGW/NAT GW rewrites to public IP.

```bash
$ sudo iptables -t mangle -L PREROUTING -n
CONNMARK  all  --  0.0.0.0/0  0.0.0.0/0  /* AWS, primary ENI */ ADDRTYPE match dst-type LOCAL limit-in CONNMARK or 0x80
CONNMARK  all  --  0.0.0.0/0  0.0.0.0/0  /* AWS, primary ENI */ CONNMARK restore mask 0x80
```

VPC CNI marks with `0x80`. Policy rule `fwmark 0x80/0x80 lookup main` routes return traffic through the main table.

### Experiment 8: Full node interface view

```bash
$ ip link show
2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001          <-- Primary ENI
165: ens6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001        <-- Secondary ENI
264: ens7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001        <-- Third ENI
4: eni36f7c958f87@if3: ...                                    <-- veth for pod
336: eniaee705c1f69@if3: ...                                  <-- OUR TOOLBOX POD
```

Three ENIs (`ensN`), a veth pair per pod (`eniXXX@if3`). VPC CNI added secondary ENIs as pod count grew.

### What we verified

| What | Experiment | Result |
|------|-----------|--------|
| /32 mask on pod eth0 | 1 | `10.0.1.51/32` |
| Fake gateway 169.254.1.1 | 1 | default route via 169.254.1.1 |
| Static ARP entry | 1 | `5a:f3:56:0b:86:d4 PERM` |
| ARP MAC = host-side veth | 2 | both show `5a:f3:56:0b:86:d4` |
| Policy routing (ip rule) | 2 | `to pod -> main`, `from pod -> table 2` |
| Secondary table only has default gw | 2 | `table 2`: just `default via ... dev ens6` |
| Same-node: sub-ms | 3 | 0.05ms |
| Cross-node: ~0.4ms | 3 | 0.4-1.7ms via VPC fabric |
| DNS through ClusterIP | 4 | 172.20.0.10 -> DNAT to CoreDNS pods |
| ClusterIP: round-robin | 4 | `statistic mode random probability 0.5` |
| ClusterIP: no SNAT | 5 | echoserver sees real pod IP |
| Conntrack tracks NAT | 5 | bidirectional mapping visible |
| NodePort: SNAT masks client | 6 | echoserver sees node IP |
| NodePort: SNAT even on localhost | 6 | localhost curl still shows node IP |
| NodePort: cross-node SNAT | 6 | remote curl shows receiving node IP |
| Pod egress works | 7 | checkip returns public IP |
| VPC CNI mangle marks | 7 | CONNMARK 0x80 for primary ENI |
| Multiple ENIs per node | 8 | ens5, ens6, ens7 |

### Cleanup

```bash
kubectl delete namespace packetlab
```
