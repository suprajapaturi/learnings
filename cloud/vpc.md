# Cloud VPC

> Cloud networking fundamentals (GCP + AWS/EKS), and how Kubernetes/OpenShift sits on top.
> Companion to k8s-internals-notes.md.

## What a VPC is
- **VPC (Virtual Private Cloud)** = your own private, isolated, software-defined network inside a cloud. Like a virtual data-center network.
- Your OpenShift nodes are **VMs in subnets of a VPC**. The `10.0.144.0/x` node network = the VPC subnet range.

```
┌──────────────── VPC (e.g. 10.0.0.0/16) ────────────────┐
│  Subnet A (zone -a): node VMs 10.0.145.x                │
│  Subnet B (zone -b): node VMs 10.0.144.x                │
│  Routes │ Firewalls │ NAT │ Load Balancers │ DNS        │
└──────────────────────────────────────────────────────────┘
              │ Internet / other VPCs / on-prem
```

## Core building blocks
| Component | What it is |
|-----------|-----------|
| **VPC** | isolated virtual network, defined by a CIDR (e.g. `10.0.0.0/16` ≈ 65k IPs) |
| **Subnet** | slice of the CIDR, pinned to **one AZ/zone**. Public vs private = a ROUTING decision |
| **Route table** | per-subnet; decides where traffic goes. Public = `0.0.0.0/0`→Internet Gateway; Private = `0.0.0.0/0`→NAT Gateway |
| **Internet Gateway (IGW)** | lets a public subnet talk to the internet (inbound+outbound) |
| **NAT Gateway / Cloud NAT** | lets PRIVATE subnet VMs reach internet OUTBOUND only (pull images); no inbound |
| **Security Group (AWS)** | **stateful**, attached to the resource (EC2/ENI/pod). Allow-only; return traffic auto-permitted. ≈ K8s NetworkPolicy mental model (resource-scoped whitelist) |
| **NACL (AWS)** | **stateless**, at the SUBNET boundary. Allow AND deny, rule-number order; must define both directions. Coarse backstop in front of SGs (usually left permissive) |
| **Load Balancer** | distributes external traffic to VMs/pods |

## Regions & zones = the HA foundation
- **Region** (e.g. `us-central1`) = geographic area. **Zone** (`-a/-b/-c`) = physically separate datacenter (independent power/network).
- K8s node label `topology.kubernetes.io/zone` = the cloud zone.
- Spread 3 etcd masters across zones → survives a **zone outage** (ties to etcd quorum: need 2/3, lose 1 zone = still quorum).
- **PodTopologySpread / anti-affinity** uses zone labels to spread replicas → survive zone failure.

## Public/private subnet shape (standard EKS/cloud VPC)
```
Internet → Internet Gateway → PUBLIC subnet (NAT Gateways + Load Balancers live here)
                                   │
                            PRIVATE subnet (worker nodes) — outbound via NAT, NEVER inbound from internet
```
- Public subnet = route to IGW. Private subnet = route to NAT Gateway. **It's the route table, not a label, that makes a subnet public/private.**

## ⭐ Pod IP allocation: OVN overlay vs EKS VPC CNI (KEY DIFFERENCE)
| | OpenShift (OVN-Kubernetes) | EKS (default VPC CNI) |
|---|----------------------------|------------------------|
| Pod IP source | separate **OVERLAY** range (172.24/14), NOT the node's subnet | the **SAME VPC subnet** as the node (e.g. 10.0.2.0/24) |
| Encapsulation | Geneve tunnels (VPC only sees node IPs) | NONE — pod IP is as real+routable as the node's |
| Pod↔pod/VPC traffic | over overlay | plain VPC routing, no NAT |
| IP mechanics | OVN IPAM, /24 per node (hostPrefix) | ENIs + secondary IPs per instance |

**EKS mechanics:** each EC2 instance has a primary **ENI** (network interface) + can attach more (instance-type-dependent). Each ENI holds a fixed number of secondary private IPs. VPC CNI keeps a **warm pool** of secondary IPs, assigns one to each pod directly.

### ⚠️ EKS gotcha: IP exhaustion caps pod density before CPU/mem
- Max pods/node on EKS = `(ENIs per instance type) × (IPs per ENI)`. A small instance can hit **"no available IP addresses"** and refuse pods at 20% CPU.
- OVN doesn't have this (overlay range is independent of node subnet).
- **Fix = PREFIX DELEGATION:** VPC CNI allocates a `/28` prefix (16 IPs) per ENI slot instead of single IPs → ~16x more pod IPs without more ENIs. Standard mitigation at the density wall.

## ⭐ kube-proxy: still separate on EKS
- **EKS**: VPC CNI does ONLY pod IP assignment → **kube-proxy still runs** (iptables/IPVS) for Service DNAT, same as vanilla K8s.
- **OpenShift (OVN)**: collapses BOTH jobs (pod networking + service routing) into one → no kube-proxy.
- (Cilium can also replace kube-proxy; VPC CNI + Calico do not.)

## How the cluster maps onto the VPC (two stacked networks)
- **VPC layer** (cloud): node IPs `10.0.x` — real routable addresses the cloud manages.
- **Overlay layer** (OVN, inside K8s): pod/service IPs `172.24/172.30` — VPC doesn't know them (tunneled inside VPC packets). *(EKS VPC CNI has NO overlay — pods are on the VPC directly.)*

## LoadBalancer Services — where K8s meets the VPC
```
kubectl create service LoadBalancer
  → cloud-controller-manager (CCM) calls the cloud API
  → cloud provisions a real Load Balancer + external IP
  → external traffic → NodePort → ClusterIP → pod
```
- **cloud-controller-manager (CCM)** = the bridge between K8s and the cloud (provisions LBs, node external IPs, routes). The integration point between the two worlds.

## Soundbites
> "A VPC is a private virtual network; nodes are VMs in its subnets. Public vs private subnet is a
> route-table decision — public routes 0.0.0.0/0 to an internet gateway, private to a NAT gateway
> (outbound only). Zones are separate datacenters, so spreading masters/replicas across zones gives
> HA, which is why etcd wants an odd count across 3 zones."

> "On EKS, VPC CNI gives every pod a real, routable IP from the same subnet as the node — no overlay —
> so pod density is capped by ENI/IP capacity per instance type, not just compute. Prefix delegation
> fixes that by allocating IP blocks instead of single addresses. Service routing is still handled
> separately by kube-proxy, since VPC CNI only owns pod networking, not load-balancing. OpenShift's
> OVN differs: pods live on a separate overlay range and OVN also replaces kube-proxy."

> "Security Groups are stateful, resource-attached, allow-only — the closest cloud analog to a K8s
> NetworkPolicy. NACLs are stateless subnet-boundary rules with allow+deny, used as a coarse backstop."
