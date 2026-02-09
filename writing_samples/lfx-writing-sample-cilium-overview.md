# Kubernetes Networking at Scale: Architecture and Concepts

## Introduction

Container orchestration has fundamentally changed the requirements for data center networking. In traditional virtualization, network identity was tied to stable IP addresses, allowing firewalls and load balancers to control traffic using static rule sets.

Kubernetes inverts this model. Workloads (Pods) are ephemeral; they scale, restart, and migrate across nodes dynamically. An IP address that identifies a workload at one moment may be reassigned to a completely different workload minutes later. Traditional network security tools and standard CNI plugins, which rely on IP-to-identity mapping, often struggle to maintain consistency and performance in this volatile environment.

## The Challenge of Network Volatility

The core challenge in Kubernetes networking is **state churn**. As Pods are created and destroyed, the network control plane must propagate these changes to every node and security appliance.

In standard implementations (e.g., `iptables` or IPVS), this results in:

- **Control Plane Latency**: Propagating thousands of IP updates can lag behind the actual state of the cluster, leading to traffic blackholes or temporary security gaps.
- **Packet Processing Overhead**: Linear rule lists grow with the number of services, causing latency to increase as the cluster scales.
- **Loss of Identity**: Debugging connections based solely on IP addresses is error-prone when those IPs are recycled frequently.

## The Cilium Architecture: Identity-Based Networking

Cilium addresses the limitations of IP-centric networking by decoupling **Network Identity** from **Network Address**. It leverages **eBPF** (Extended Berkeley Packet Filter) to move network logic directly into the Linux kernel, bypassing the legacy userspace networking stack.

### Core Concepts

1.  **Identity, Not IPs**: Cilium assigns a numeric, cluster-wide **Security Identity** to workloads based on their Kubernetes labels (e.g., `app=frontend`). This identity persists across Pod restarts.
2.  **Kernel-Level Enforcement**: Policies are enforced efficiently in the kernel using eBPF maps (hash tables), enabling O(1) lookups regardless of policy size.
3.  **eBPF Datapath**: Instead of relying on rigid kernel modules, Cilium compiles and loads custom eBPF programs that handle routing, load balancing, and observability at the network interface level.

### High-Level Components

Cilium runs a user-space **Agent** on every node which listens to the Kubernetes API. The Agent compiles high-level intents (policies, services) into eBPF bytecode and loads them into the kernel.

![Cilium eBPF Architecture](./cilium-ebpf-lifecycle.svg)

**Figure 1: The Cilium eBPF Architecture**. The Agent translates Kubernetes state into efficient kernel-level logic. Packet filtering happens immediately at the network interface.

## Networking and Load Balancing

Cilium manages Pod-to-Pod and Pod-to-Service traffic using an optimized datapath that avoids the overhead of Network Address Translation (NAT) where possible.

- **Maglev Load Balancing**: For Service traffic, Cilium uses consistent hashing (Maglev) in eBPF. This eliminates the need to scan linear tables, ensuring that load balancing performance remains constant even as the number of Services grows.
- **Direct Routing**: Packets can be routed directly between nodes, or encapsulated in VXLAN/Geneve overlays for compatibility with existing network fabrics.
- **Multi-Cluster**: The **Cluster Mesh** feature allows the datapath to span multiple Kubernetes clusters, enabling global service load balancing and failover without complex gateway configurations.

## Security: The Identity-Based Model

Security enforcement in Cilium is defined by _who_ the workload is (Identity), not _where_ it is located (IP Address).

When a NetworkPolicy allows `frontend` to communicate with `backend`, Cilium translates this into a rule allowing `Identity(frontend)` to permit traffic to `Identity(backend)`.

![Identity-Based Enforcement Flow](./cilium-security-final.svg)

**Figure 2: Identity-Based Enforcement Flow**. The policy check is a simple hash map lookup. This design allows security rules to scale independently of the number of active Pods.

## Observability: Hubble

Operations teams require visibility into network behavior to debug connectivity and performance issues. Standard tools like `tcpdump` lack context regarding Kubernetes metadata.

**Hubble** is the observability plane integrated into Cilium. It captures flow data directly from the eBPF datapath, providing:

- **Service Dependency Maps**: Real-time visualization of service-to-service communication.
- **Flow Logs**: Detailed records of connections, including L7 info (HTTP methods, DNS queries) and drop verdicts (Policy Denied).
- **Metrics**: Prometheus-compatible metrics for network latency, throughput, and error rates.

![Hubble Observability Architecture](./cilium-hubble-deep-dive.svg)

**Figure 3: Hubble Observability: Kernel-Level Event Extraction**.

## Production Considerations

Cilium is designed for environments where reliability and performance are critical.

- **Resilience**: The eBPF datapath runs independently of the userspace agent. If the Cilium Agent updates or restarts, existing network connections remain uninterrupted.
- **Scale**: By using per-CPU maps and efficient hashing, Cilium supports clusters with significantly larger Service and Pod counts than traditional `kube-proxy` implementations.
- **Interoperability**: Cilium can integrate with existing BGP routers for on-premise deployments or use Egress Gateways to interface with legacy firewalls.

## Summary

Cilium provides a unified networking, security, and observability layer that creates a robust foundation for Kubernetes platforms. By utilizing eBPF, it resolves the operational friction caused by IP volatility and state churn, enabling platforms to scale securely.

For implementation details, refer to the [Installation Guide](#) or the [Network Policy Documentation](#).
