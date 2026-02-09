# eBPF Explained Through Cilium

## Introduction

Extending the Linux kernel networking stack has traditionally been difficult. Standard tools like `iptables` rely on static rule lists that grow linearly with the number of rules. In dynamic environments like Kubernetes, where Pods and Services churn frequently, updating these lists becomes computationally expensive and slow.

Cilium overcomes these limitations by using eBPF to program the kernel directly. Instead of traversing long chains of rules, Cilium generates efficient, event-driven programs that execute logic specific to the current cluster state. This allows networking to scale independently of the number of active rules.

## Where Packet Processing Happens

The Linux networking stack processes packets in layers. When a packet arrives at the Network Interface Card (NIC), it passes to the device driver, then to the kernel's protocol stack (IP, TCP/UDP), and finally to a socket where the application reads the data.

Standard firewalls often intervene late in this process, after the kernel has already allocated memory (sk_buffs) and parsed headers.

Cilium intercepts packets much earlier. By attaching eBPF programs to the **Traffic Control (TC)** ingress hook or the **XDP (eXpress Data Path)** hook, Cilium can parse, redirect, or drop packets before the kernel performs heavy processing.

![Packet Processing with XDP/TC](../assets/diagrams/)

**Figure 1: Packet Processing Flow with eBPF (XDP and TC Hooks)**

The diagram illustrations the processing path. The eBPF program at the TC hook inspects the packet immediately after the driver. If the packet is denied or needs redirection (e.g., for Load Balancing), the logic executes there, bypassing the traditional stack overhead.

## What eBPF Is (Practical View)

eBPF is a kernel technology that enables safe, dynamic programmability. It was designed to allow users to run custom logic inside the kernel without changing the kernel source code or loading risky kernel modules.

Operationally, "running in the kernel" means the code executes with high privileges and access to internal kernel structures. However, unlike kernel modules, eBPF programs are sandboxed. They run in a restricted virtual machine within the kernel, ensuring they cannot access arbitrary memory or destabilize the system.

## Safety and Verification

The **Verifier** is the core safety mechanism of eBPF. Before any program is loaded into the kernel, the Verifier analyzes its bytecode.

It enforces strict constraints:

1.  **Termination**: The program must finish execution quickly (no infinite loops/blocking).
2.  **Memory Access**: The program can only access approved memory regions and data structures.
3.  **Type Safety**: The program must use kernel helper functions correctly.

If a program fails verification, the kernel rejects it. This guarantees that a bug in a Cilium eBPF program cannot crash the underlying node, a critical requirement for production clusters.

## How Cilium Uses eBPF

Cilium uses eBPF to implement the entire datapath logic.

1.  **Hooks**: Cilium attaches programs to network interfaces on the node. These programs trigger on every packet event.
2.  **Identity Enforcement**: The eBPF program reads the packet headers to determine the source Identity. It checks this Identity against a policy map (hash table) to permit or deny the traffic.
3.  **Load Balancing**: For Service traffic, the eBPF program modifies the destination IP address (DNAT) to match a backend Pod. It handles this translation entirely in the kernel.
4.  **No Context Switching**: Because the logic runs in the kernel, packets do not need to be copied to userspace for processing.

## eBPF Datapath Lifecycle

The **Cilium Agent** (running in userspace) manages the eBPF programs.

1.  **Compilation**: The Agent generates eBPF bytecode based on the node's configuration.
2.  **Loading**: The Agent loads the bytecode into the kernel using system calls.
3.  **Map Updates**: Dynamic data—such as NetworkPolicies, Service endpoints, and connection tracking tables—is stored in **eBPF Maps**.
4.  **Atomic Changes**: When a Kubernetes Service changes, the Agent updates the specific entry in the eBPF Map. The loaded eBPF program sees this change immediately. There is no need to reload the program or disrupt traffic.

![Cilium eBPF Lifecycle](../assets/diagrams/)

**Figure 2: The eBPF Program Lifecycle and Datapath Architecture**

The diagram shows the separation between control and execution. The Agent continually updates the Maps based on Kubernetes events. The eBPF Program running in the kernel uses these Maps to make decisions for every packet in real-time.

## Performance Characteristics

eBPF changes the complexity class of network processing.

**Constant-Time Lookups**: eBPF Maps are implemented as hash tables. Looking up a policy requires O(1) time. Whether there are 10 rules or 10,000 rules, the cost to check a packet remains consistent.

**Reduced Latency**: By avoiding the context switch to userspace (common in sidecar proxies or userspace implementations), Cilium reduces the CPU instructions required per packet.

**Scalability**: In large clusters, this efficiency prevents the compiled ruleset from becoming a bottleneck. The datapath throughput remains stable even as the cluster churns.

## Common Misunderstandings About eBPF

**"eBPF is unsafe"**: This is incorrect. The Verifier ensures eBPF is significantly safer than kernel modules. It effectively eliminates the risk of memory corruption crashes.

**"eBPF replaces the kernel networking stack"**: eBPF extends the stack. Cilium can pass packets to the standard stack when needed (e.g., for local process delivery) or bypass it for forwarding. It utilizes existing kernel capabilities like routing tables and drivers.

**"eBPF is only for networking"**: While this document focuses on networking, eBPF is a general-purpose engine used for security (profiling syscalls) and observability (tracing function execution).

## How eBPF Enables Cilium Features

**Network Policy**: eBPF allows Cilium to enforce policies based on Identity rather than IP. It can also filter based on L4 ports/protocols efficiently.

**Service Load Balancing**: Cilium implements a distributed load balancer using eBPF, replacing `kube-proxy`. This supports direct server return and advanced hashing algorithms (Maglev).

**Observability (Hubble)**: Cilium emits visibility events directly from the eBPF datapath. This provides deep insights (e.g., "DNS latency," "TCP retransmits") with minimal overhead.

**Layer 7 Filtering**: For L7 policies (e.g., HTTP paths), eBPF redirects traffic to a userspace proxy (Envoy) only when necessary, keeping the fast path optimized for L3/L4.

## Relationship to Other Kernel Mechanisms

Cilium and eBPF is designed to complement the Linux kernel. It offloads high-frequency, complex packet logic to optimized eBPF programs while relying on the kernel's mature subsystems for standard tasks. This hybrid approach allows Cilium to deliver high performance and advanced features without completely reinventing the operating system's networking foundation.

## Next Learning Steps

To deepen your understanding, investigate the following topics:

- **Cilium Datapath Documentation**: Technical deep dive into specific hooks.
- **Hubble Internals**: How eBPF ring buffers transfer data to userspace.
- **Advanced Policy Topics**: API-aware enforcement and DNS-based policies.
