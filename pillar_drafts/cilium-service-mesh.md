# Sidecar-Free Service Mesh with Cilium

## Why the Sidecar Model Becomes a Problem

Traditional service mesh implementations rely on a "sidecar" architecture. In this model, every application Pod runs an accompanying proxy container (typically Envoy) to intercept and manage traffic. While this provides granular control, it introduces significant operational and resource challenges as cluster size increases.

- **Resource Overhead**: Running two containers per Pod effectively doubles the control plane load and increases the memory footprint. A cluster with 1,000 Pods requires 1,000 additional proxy instances, consuming CPU and memory resources per instance.
- **Operational Complexity**: Injecting sidecars typically requires mutating webhooks, which can complicate Pod startup and debugging. Race conditions between the application and the proxy can occasionally lead to startup failures or dropped traffic during initialization.
- **Latency**: Traffic traverses the network stack of the sidecar for every request, adding context switches and serialization steps to the data path.

![Sidecar vs Sidecar-Free Comparison](../assets/diagrams/)

**Figure 1: Architectural Comparison: Sidecar vs. Sidecar-Free (Cilium)**

## Cilium’s Sidecar-Free Architecture

Cilium implements a sidecar-free service mesh by leveraging eBPF to handle traffic management directly within the Linux kernel. This approach decouples service mesh capabilities from the application Pods.

Instead of injecting a proxy into every Pod, Cilium utilizes a per-node architecture. eBPF programs attached to the network interface handle standard L3/L4 traffic (routing, load balancing, and firewalling) efficiently in kernel space. For Layer 7 operations (such as HTTP parsing or gRPC transcoding), eBPF redirects traffic to a single, per-node Envoy instance managed by the Cilium Agent.

![Sidecar-less Fast Path vs Slow Path](../assets/diagrams/)

**Figure 2: Sidecar-less Service Mesh: Fast Path vs. Slow Path**

The diagram illustrates how eBPF acts as a router within the node. Standard network traffic bypasses the proxy entirely (Fast Path). Only traffic requiring complex L7 processing is redirected to the per-node Envoy instance (Slow Path). This reduces the need for redundant sidecars while retaining advanced traffic management capabilities.

## How Traffic Is Managed Without Sidecars

Cilium processes traffic differently based on the required level of inspection:

- **L3/L4 (Fast Path)**: Features like identity-based security, standard Kubernetes NetworkPolicies, and simple load balancing are handled entirely in eBPF. This processing occurs at line rate with minimal overhead.
- **L7 (Slow Path)**: When a policy requires inspecting HTTP headers, retrying requests, or splitting traffic percentage-wise (e.g., for Canary releases), eBPF transparently redirects the packet to the node-local Envoy proxy.
- **Traffic Termination**: The proxy terminates the connection, executes the required logic (e.g., "rewrite path /v1 to /v2"), and initiates a connection to the destination. To the application, this redirection is transparent.

## Gateway API and Ingress Integration

Cilium supports the Kubernetes Gateway API, the standard for unifying Ingress and Service Mesh configuration.

The Gateway API defines standard resources (`Gateway`, `HTTPRoute`, `TLSRoute`) that express routing intent. Cilium implements this by translating these resources into eBPF routing rules and Envoy configurations. This enables a clear separation of duties: infrastructure teams manage `Gateways` (listeners/IPs), while application teams manage `HTTPRoutes` (application logic).

![Gateway API Implementation](../assets/diagrams/)

**Figure 3: Gateway API Implementation for Ingress & Service Mesh**

Cilium acts as both the north-south ingress controller and the east-west service mesh router. A single implementation handles traffic entering the cluster and traffic moving between services, reducing the number of distinct networking components required.

## mTLS and Authentication Model

Mutual TLS (mTLS) in a service mesh typically provides both encryption and authentication. Cilium decouples these concerns to offer flexibility.

- **Authentication**: Cilium uses its Identity-based security model to authenticate workloads based on Kubernetes labels. This creates a secure "who can talk to whom" graph enforced in the kernel.
- **Encryption**: Cilium can establish a cluster-wide mesh of encrypted tunnels (using WireGuard or IPsec) between nodes, securing data in transit transparently.
- **mTLS Support**: For environments requiring strict X.509 mTLS at the application level, Cilium can enforce mTLS handshakes either via the per-node proxy or by integrating with SPIFFE/SPIRE for workload attestation.

## Performance Characteristics

The sidecar-free model alters the performance profile of the mesh:

- **Reduced Latency**: By removing the sidecar from the path of standard L3/L4 traffic, latency typically decreases for HTTP workloads and TCP streams compared to a sidecar-per-pod model.
- **Lower Memory Usage**: Avoiding sidecar injection reduces the total memory reservation required per cluster. The per-node proxy scales with the node's traffic volume rather than the number of Pods.
- **CPU Efficiency**: eBPF processing is optimized and JIT-compiled, reducing CPU cycles compared to userspace packet switching.

## Operational Impact

Migrating to a sidecar-free architecture simplifies Day 2 operations:

- **Upgrade Safety**: Upgrading the service mesh (Cilium Agent) does not require restarting application Pods. The eBPF programs continue running in the kernel during the agent update, minimizing traffic disruption.
- **No Injection Webhooks**: Eliminating the sidecar injection step removes a source of potential deployment failures and startup latency.
- **Reduced Blast Radius**: While the scope of the proxy is the node, Cilium's isolation mechanisms and support for canary rollouts of the agent minimize the risk of node-wide impact during configuration changes.

## When a Sidecar-Free Mesh Makes Sense

Cilium Service Mesh is well-suited for:

- **High-Scale Environments**: Clusters with high Pod density where sidecar resource overhead is significant.
- **Performance-Critical Apps**: Workloads sensitive to the latency introduced by multiple proxy hops.
- **Simplified Operations**: Teams seeking service mesh features (Canary, Tracing, Encryption) without managing the complexity of sidecar lifecycles.

## Next Learning Steps

To deepen your understanding, investigate the following topics:

- **Cilium Service Mesh Documentation**: Detailed configuration guides for L7 traffic management.
- **Gateway API Examples**: Implementation patterns for advanced routing strategies using the Gateway API.
- **L7 Policy**: Writing Cilium Network Policies that filter HTTP traffic.
