# Kubernetes Networking: From Basics to Production (Using Cilium)

## Introduction

Kubernetes networking addresses the challenge of connecting containers across multiple hosts. In traditional server models, applications bind to specific IP addresses and ports. In Kubernetes, containers are ephemeral; they start, stop, and move between nodes dynamically.

This dynamism introduces complexity. Containers must communicate with each other, external users, and backend services. The network must track these changing endpoints automatically.

## How Kubernetes Networking Works

Kubernetes mandates a flat network model.

1. Every Pod possesses a unique IP address.
2. Pods act as if they are on the same logical network; they communicate without Network Address Translation (NAT).
3. Agents on a node (like the kubelet) can communicate with all Pods on that node.

### Pods and Services

A **Pod** serves as the atomic unit of scheduling. It hosts one or more containers that share a network namespace and a single IP.

A **Service** provides a stable abstraction over a set of Pods. It assigns a virtual IP (ClusterIP) and a DNS entry. The Service mechanism load-balances traffic sent to this virtual IP across the healthy backend Pods.

### Node-to-Node Communication

When a Pod on Node A transmits data to a Pod on Node B, the packet traverses the physical or virtual network fabric between nodes. Depending on the configuration (Overlay vs. Direct Routing), the packet may be encapsulated.

> **Note**: The following diagrams provide a high-level conceptual overview.

![Service Load Balancing: iptables vs Maglev](../assets/diagrams/)

**Figure 1: Service Load Balancing Scalability: Linear iptables vs. O(1) Maglev Hashing**

The diagram illustrates cross-node traffic. The packet exits the source Pod’s namespace, enters the Node A host network, traverses to Node B, and is forwarded to the destination Pod.

## Limitations of Default Kubernetes Networking

Standard implementations, such as kube-proxy in `iptables` mode, rely on Linux kernel firewall rules for routing and load balancing. While stable and reliable for small-to-medium clusters, this approach faces challenges at scale.

### Scalability Challenges

Iptables rules are processed sequentially (O(N) complexity). As the number of Services and Pods grows, the number of rules expands. In clusters with thousands of services, evaluating these rules consumes significant CPU. Furthermore, updating the rule set becomes slow, delaying network convergence during scaling events.

### Security Limitations

By default, Kubernetes allows unrestricted Pod-to-Pod communication. Network Policies can limit this, but implementing large-scale policies with IP-based rules is cumbersome. Since Pod IPs change frequently, security systems relying on IPs must constantly update, creating a fragile synchronization loop.

### Visibility Gaps

Standard network tools (like `tcpdump` or flow logs) see traffic between IP addresses. They lack Kubernetes context. Debugging requires manually correlating IP addresses from logs with Pod history to understand which application was responsible for network traffic.

## What is a CNI Plugin

Kubernetes delegates network implementation to **Container Network Interface (CNI)** plugins.

A CNI plugin handles:

- **IP Address Management (IPAM)**: Allocating IPs to Pods.
- **Connectivity**: Creating virtual interfaces (veth pairs) to connect Pods to the host.
- **Routing**: Configuring the kernel to route packets between Pods.
- **Policy**: Enforcing network policies.

The container runtime calls the CNI plugin when a Pod initializes or terminates.

## How Cilium Approaches Kubernetes Networking

Cilium utilizes **eBPF (Extended Berkeley Packet Filter)** to process network traffic, bypassing legacy iptables chains for forwarding and load balancing.

### Identity-Based Networking

Cilium abstracts network identity from IP addresses. It assigns a numerical identity to endpoints based on metadata (labels). If a frontend Pod restarts and receives a new IP, its identity (derived from `app=frontend`) persists. Security policies target these stable identities, decoupling security from network addressing.

### Policy Enforcement at Kernel Level

By attaching programs to network hooks, Cilium filters packets early in the kernel processing path. This avoids the overhead of traversing the full TCP/IP stack for packets that should be dropped.

### High-Level Explanation of eBPF

eBPF allows the execution of sandboxed programs within the operating system kernel. It extends kernel capabilities safely without changing source code or loading modules.

**Safety is a core design principle.** The kernel **Verifier** analyzes every eBPF program before loading it. The Verifier guarantees that the program runs to completion, does not access invalid memory, and cannot crash the system.

## Architecture Overview

Cilium separates the control plane (userspace) from the datapath (kernel).

![eBPF Program Lifecycle](../assets/diagrams/)

**Figure 2: eBPF Program Lifecycle and Architecture in Cilium**

- **Cilium Agent**: Runs as a DaemonSet in userspace. It observes Kubernetes resources (Services, NetworkPolicies) and compiles appropriate eBPF programs.
- **eBPF Datapath**: Consists of compiled programs loaded into the kernel. These programs process packets at line rate, independent of the Agent.
- **Cilium Operator**: Manages global tasks like identity synchronization and garbage collection.
- **Hubble**: Extracts visibility data directly from the eBPF datapath with minimal overhead.

## Real Request Flow Example

Consider a user request targeting a web application in the cluster.

1. **Ingress**: The packet arrives at the node's network interface. An eBPF program executes immediately.
2. **Load Balancing**: The eBPF logic performs a hash map lookup to select a backend Pod for the Service.
3. **Policy Enforcement**: The datapath validates the source identity against the destination identity. This check occurs per-packet in the kernel.
4. **Forwarding**: If allowed, the packet is redirected to the virtual device of the destination Pod.
5. **Observability**: Hubble asynchronously aggregates flow metadata (latency, protocols, drops) for analysis.

## Common Mistakes and Misunderstandings

**Relying on IP Allow-lists**: Attempting to manage security via IP ranges is an anti-pattern in cloud-native environments due to IP churn. Use identity-based policies.

**Enforcing Deny-All Too Early**: Applying a generic "deny-all" policy without understanding traffic flows often causes outages. Use Hubble to audit traffic before enforcement.

**Ignoring Observability**: Treating the network as a "black box" increases Mean Time To Repair (MTTR). Leverage tools that provide Kubernetes-aware network context.

## How This Fits Into Production Environments

**Scalability**: The eBPF datapath scales efficiently. Looking up endpoints in an eBPF map is an O(1) operation, compared to the linear O(N) search in iptables chains. Performance remains consistent regardless of cluster size.

**Debugging**: Hubble allows engineers to trace policy decisions. It provides evidence of exactly why a packet was dropped (e.g., "Denied by Policy X"), eliminating ambiguity.

**Security Posture**: Identity-aware policies enable granular control, such as restricting specific API paths or HTTP methods, beyond simple port blocking.

## Next Learning Steps

To deepen your understanding, investigate the following topics:

- **Hubble UI**: Visualizing real-time service dependencies.
- **Cluster Mesh**: Connecting disparate Kubernetes clusters.
- **Layer 7 Policy**: Filtering traffic based on application data (HTTP/DNS).
- **BGP Integration**: Leveraging standard routing protocols for service announcement.
