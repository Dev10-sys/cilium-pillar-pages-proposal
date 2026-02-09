# External Networking with Cilium

## Introduction

In the standard Kubernetes model, the cluster network is often treated as an isolated island. Pods communicate freely with each other, but traffic leaving the cluster is typically obscured by Network Address Translation (NAT) or tunneled through overlays. While this simplifies cloud deployments, it creates significant friction in on-premise, edge, and hybrid environments where Kubernetes must integrate with existing physical networks.

Connecting Kubernetes workloads directly to enterprise routers, firewalls, and legacy databases requires bridging the gap between dynamic, ephemeral container networking and static, policy-driven physical infrastructure.

## Why External Networking Is Hard in Kubernetes

Integrating Kubernetes with external networks presents several challenges:

- **Dynamic Pod IPs**: Pods are ephemeral and receive random IP addresses. Traditional firewalls and routers expect stable, known IP ranges for access control lists (ACLs).
- **Firewall Compliance**: Security teams often require specific workloads to exit the cluster via deterministic IP addresses for audit and allow-listing purposes. Default Kubernetes masquerading (SNAT) hides the source identity of the Pod, making granular policy enforcement impossible outside the cluster.
- **LoadBalancer Limitations**: On bare metal, the `LoadBalancer` service type does not work out-of-the-box without a cloud controller. Operators often resort to inefficient `NodePort` services or complex add-ons.
- **Operational Risks**: Heavy reliance on NAT introduces connection tracking overhead and obscures visibility, complicating troubleshooting when connectivity to external systems fails.

## Cilium’s External Networking Model

Cilium addresses these challenges by making Kubernetes a first-class citizen of the data center network. Instead of hiding the cluster behind NAT gateways or overlays, Cilium enables native routing integration using eBPF and standard networking protocols.

This approach allows Pods and Services to be reachable via routable IPs, controlled by precise BGP advertisements and eBPF-based egress logic. This eliminates the dependency on proprietary cloud load balancers or third-party hardware appliances.

## BGP Control Plane with Cilium

Border Gateway Protocol (BGP) is the standard for routing in data centers. Cilium includes a built-in BGP control plane that allows Kubernetes nodes to peer directly with Top-of-Rack (ToR) switches or core routers.

Through BGP, Cilium creates a direct routing path:

- **Pod CIDR Advertisement**: Cilium advertises the IP ranges of Pods to the upstream network, making Pods directly reachable without NAT (if desired).
- **Service IP Advertisement**: Cilium advertises LoadBalancer IP addresses (VIPs) to the network, enabling external traffic to reach Kubernetes Services via the shortest path (Equal-Cost Multi-Path routing).

![External Networking Deep-Dive](../assets/diagrams/)

**Figure 1: Cilium BGP Integration and Egress Gateway Architecture**

The diagram shows Cilium Agents peering with the physical network. The router learns exactly where Pods and Services are located, allowing it to route traffic directly to the correct node.

## Egress Gateways

For scenarios where direct Pod routing is not possible or desirable (e.g., communicating with a strict legacy firewall), Cilium provides **Egress Gateways**.

An Egress Gateway allows operators to assign a stable, static IP address to the outbound traffic of a specific namespace or Pod. Instead of traffic masquerading as the diverse node IPs, it is tunneled to a designated gateway node and exits with a predictable source IP.

This satisfies security requirements for:

- **Identity Consistency**: Traffic from the "payments" namespace always originates from `10.20.1.5`.
- **Audit Trails**: Firewalls can log flows from specific applications based on their egress IP.
- **Policy Enforcement**: External systems can safely allow-list the gateway IP without needing updates every time the cluster scales.

![Cilium Egress Gateway](../assets/diagrams/)

**Figure 2: Deterministic Outbound Traffic with Cilium Egress Gateway**

## Bare-Metal Load Balancing

Cilium provides a native implementation for `LoadBalancer` services on bare metal, replacing the need for tools like MetalLB.

- **Service IPAM**: Cilium manages pools of external IP addresses and assigns them to Services.
- **L2 vs L3 Announcements**:
  - **L2 (ARP/NDP)**: Suitable for small clusters where nodes share a broadcast domain. Cilium responds to ARP requests for the Service IP.
  - **L3 (BGP)**: Suitable for larger, routed networks. Nodes advertise the Service IP to upstream routers via BGP.
- **DSR (Direct Server Return)**: Cilium supports performance optimizations where the return traffic from a Pod bypasses the load balancing node and goes directly to the client, preventing the gateway from becoming a bottleneck.

## Security and Policy Integration

External networking in Cilium enables identity-based security policies for traffic entering and leaving the cluster.

- **Ingress filtering**: Policies can restrict which external CIDRs are allowed to access specific Services.
- **Egress filtering**: Policies can restrict Pods to specific FQDNs or external IPs.
- **Identity Preservation**: By avoiding excessive NAT, observability tools can see the true source and destination of flows, making security audits more accurate.

## Operational Characteristics

Integrating with physical networks requires stability and resilience:

- **Graceful Failover**: With BGP and ECMP, if a node fails, the router automatically stops sending traffic to it, and Cilium redistributes the load to remaining healthy nodes.
- **Router Independence**: Cilium works with any BGP-compliant router (Cisco, Arista, Juniper, FRR), avoiding vendor lock-in.
- **Incremental Adoption**: BGP peering can be enabled on a subset of nodes or specific IP pools, allowing teams to verify the integration before rolling it out cluster-wide.

## When External Networking with Cilium Makes Sense

This architecture is critical for:

- **On-Premise Private Clouds**: Replacing expensive hardware load balancers with software-defined routing.
- **Edge Computing**: Connecting widely distributed, small clusters to local networks.
- **Telco / NFV**: Deploying network functions that require direct, high-performance datapath integration.
- **Mainframe / Legacy Integration**: Applications that must connect to systems shielded by rigid, IP-based firewalls.

## Next Learning Steps

To deepen your understanding, investigate the following topics:

- **BGP Control Plane Docs**: Configuring peering sessions and policies.
- **Egress Gateway Patterns**: Assigning static IPs to specific namespaces.
- **Bare Metal Guides**: Deploying Type=LoadBalancer services without cloud providers.
