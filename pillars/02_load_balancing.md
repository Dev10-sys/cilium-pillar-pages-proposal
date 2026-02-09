# High-Performance Datapath with Cilium (XDP, Maglev, DSR)

## Introduction

At standard scale, Kubernetes networking performance is rarely a bottleneck. Default implementations using `iptables` or standard kernel routing mechanisms are sufficient for typical web applications. However, as cluster throughput increases—reaching 100Gbps line rates, millions of packets per second (PPS), or extreme low-latency requirements—the overhead of the general-purpose Linux networking stack becomes visible.

Cilium addresses these high-performance requirements by programmable bypassing of standard kernel layers. By utilizing eBPF and XDP (eXpress Data Path), Cilium moves packet processing logic as close to the hardware as possible, minimizing CPU usage and latency while maintaining Kubernetes identity and security semantics.

## Where Traditional Kubernetes Networking Slows Down

In a standard Kubernetes datapath (e.g., `kube-proxy` with `iptables` or IPVS), a packet traversing a node undergoes several expensive operations:

- **Traversal Overhead**: `iptables` rules are processed sequentially. As the number of Services grows, the time to evaluate these rules increases linearly (O(n)), adding latency to every packet.
- **Context Switches**: Packets move between kernel space and user space or traverse deep into the kernel implementation, consuming CPU cycles for interrupt handling and memory allocations (`sk_buff`).
- **Conntrack Overhead**: The Linux connection tracking system (conntrack) maintains state for every flow. At high connection rates, lock contention on the conntrack table can cause jitter or packet drops.
- **Hairpinning**: Traffic often unnecessarily traverses a central load balancer or proxy, doubling the network path length for local or return traffic.

## Cilium’s High-Performance Datapath Philosophy

Cilium’s approach to performance is architectural, not just messy optimization. The philosophy is based on:

1.  **Early Execution**: Process packets at the earliest possible point in the driver or kernel.
2.  **Bypass Abstractions**: Skip general-purpose stack layers (like netfilter) if the intent of the packet is already known.
3.  **Deterministic Lookups**: Replace linear lists with O(1) hash maps (eBPF Maps) for routing and policy decisions.

## XDP (eXpress Data Path)

XDP allows eBPF programs to run directly in the network device driver, before the Linux kernel parses the packet headers or allocates major memory structures (`sk_buff`). This is the earliest capture point in the software stack.

When XDP is enabled in Cilium, the eBPF program parses the incoming packet immediately upon arrival at the NIC. If the packet is destined for a local Pod or meant to be load-balanced, XDP handles it instantly. If the packet needs to be dropped (e.g., DDoS protection), it is discarded before the kernel invests any CPU cycles in it.

![High-Performance Datapath Architecture](../assets/diagrams/)

**Figure 1: High-Performance Datapath: XDP, Maglev, and DSR Comparison**

The diagram illustrations the shortcut. XDP (Fast Path) processes the packet immediately at the driver level. Traditional networking (TC/Netfilter) requires the full kernel stack traversal.

## Maglev Load Balancing

Standard load balancing often uses round-robin or random selection. While simple, these algorithms break flow consistency if the backend set changes (e.g., during a deployment).

Cilium implements **Maglev consistent hashing** for Service load balancing. Maglev builds a large lookup table that maps flows to backends. When a backend is added or removed, only a tiny fraction of the table changes.

- **Consistency**: Minimizes connection resets during scaling events.
- **Statelessness**: The load balancer does not need to synchronize state across all nodes, enabling massive horizontal scalability.
- **Efficiency**: Lookup is a constant-time memory read, perfectly suited for the XDP layer.

![Maglev vs iptables Load Balancing](../assets/diagrams/)

**Figure 2: Performance Comparison: Linear iptables vs. O(1) Maglev Hashing**

## Direct Server Return (DSR)

In standard LoadBalancer services (SNAT), the request enters a node, gets forwarded to a backend, and the _response_ must traverse back through the initial node to be reverse-NATed before returning to the client. This makes the ingress node a bottleneck for bandwidth-heavy workloads (like video streaming).

Cilium implements **Direct Server Return (DSR)**.

1.  The ingress node encodes the client's true IP into the packet and forwards it to the backend.
2.  The backend processes the request.
3.  The backend replies _directly_ to the client using the client's IP, bypassing the ingress node entirely.

This removes the return hop limit, allowing the cluster's egress bandwidth to scale with the number of backend nodes rather than the capacity of the ingress gateway.

## Bypassing Conntrack Safely

For certain classes of traffic, Linux connection tracking is unnecessary overhead. If a protocol is stateless or if the load balancing decision is fully deterministic (via Maglev), Cilium can be configured to bypass conntrack.

This is particularly relevant for:

- **UDP-heavy workloads**: DNS, VOIP, or gaming servers where maintaining connection state provides little value but costs significant memory.
- **Asymmetric Routing**: Scenarios where ingress and egress paths differ (like DSR), which typically confuse standard stateful firewalls.

Cilium allows selective bypass policies, ensuring that security is maintained (via static rules) while performance hits from state management are removed.

## Observability at High Performance

A common trade-off in high-performance networking is losing visibility (e.g., bypassing `tcpdump` hooks).

Because Hubble reads from the eBPF datapath via a separate ring buffer, it can still extract metadata (Flow Logs) even when XDP is active. However, in extreme high-PPS environments, operators may choose to sample flows or disable L7 parsing to preserve every available CPU cycle for packet forwarding. Hubble allows dynamic configuration of inspection depth to balance visibility with throughput.

## Operational Safety

Enabling XDP and DSR represents a significant shift from standard networking behavior. Cilium ensures safety through:

- **Driver Compatibility Checks**: XDP requires driver support. Cilium detects supported drivers or falls back to "Generic XDP" (slower, but functional) if native support is missing.
- **Fail-Safe Loading**: The verifier ensures that high-performance programs do not crash the kernel.
- **Incremental Rollout**: DSR and XDP can be enabled on specific LoadBalancers or nodes, allowing validation before broad application.

## When High-Performance Mode Makes Sense

These advanced features are designed for specific use cases:

- **100Gbps+ Environments**: Where CPU interrupts from packet processing consume entire cores.
- **Telco / NFV**: Implementation of 5G User Plane Functions (UPF) or high-speed gateways.
- **AI/ML Data Pipelines**: Where massive datasets move between nodes, and DSR significantly speeds up transfer times.
- **DDoS Mitigation**: Using XDP to drop malicious traffic at the edge with minimal resource impact.

## Next Learning Steps

To deepen your understanding, investigate the following topics:

- **XDP Documentation**: Hardware compatibility lists and driver requirements.
- **Performance Tuning Guides**: Configuring interrupt affinity and ring buffer sizes.
- **Kernel Requirements**: Ensuring your OS supports the latest eBPF features needed for Maglev and DSR.
