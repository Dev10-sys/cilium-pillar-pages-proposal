# Network Security with Cilium

## Introduction

By default, Kubernetes implements a flat network model where all Pods can communicate with each other. While this simplifies application deployment, it creates significant security risks in multi-tenant or production environments. Traditional network security models, which rely on static IP addresses and firewalls, struggle to adapt to the dynamic nature of container orchestration.

Cilium addresses this by implementing identity-based security. Instead of relying on volatile network identifiers, Cilium uses metadata (labels) to define security identities. This allows security policies to remain stable even as the underlying network topology changes.

## Why IP-Based Security Breaks in Kubernetes

In traditional infrastructure, a server's IP address acts as its identity. Firewall rules allow traffic from `10.1.1.5` to `10.1.2.6`. In Kubernetes, Pods are ephemeral. When a Deployment scales up or rolls out an update, old Pods terminate and new Pods launch with different IP addresses.

Maintaining IP-based allow-lists in this environment is operationally unsafe. It requires constant synchronization between the orchestration layer and the firewall. If this synchronization lags, legitimate traffic is blocked, or worse, an IP reused by a different workload inadvertently inherits old permissions.

## Cilium’s Identity Model

Cilium decouples security from network addressing. When a Pod starts, Cilium observes its Kubernetes labels (e.g., `app=frontend`, `env=prod`). Based on these labels, Cilium assigns a numeric **Security Identity** to the Pod.

This Identity is cluster-wide. All Pods sharing the same set of security-relevant labels share the same Identity. When a Pod restarts and receives a new IP, its Identity remains constant. Security policies define allowed communication between these Identities (e.g., "Identity A can talk to Identity B that has `role=backend`"), completely ignoring the specific IP addresses involved.

![Identity Derivations](../assets/diagrams/)

**Figure 1: Identity Derivations from Labels**

![Identity Resolution Flow](../assets/diagrams/)

**Figure 2: The Control Plane Process of Identity Resolution**

The diagram describes how labels translate to Identities. Despite having different IP addresses, multiple Pod instances share the same Identity. This consolidates rule generation; a policy needs only one rule for "Identity 1532", regardless of how many Pods that Identity represents.

## Policy Enforcement in the Kernel

Cilium enforces policies using eBPF programs attached to the network hooks (discussed in the previous documentation module). The enforcement occurs immediately when a packet enters the datapath.

When a packet arrives, the eBPF program:

1.  Identifies the source Identity (carried in packet encapsulation or looked up via IP-to-Identity maps).
2.  Identifies the destination Identity.
3.  Performs a hash map lookup in the policy table.

This lookup determines if traffic is `ALLOWED` or `DENIED`. Because this logic runs in the kernel, invalid traffic is dropped instantly, protecting the application from processing unauthorized packets.

![Identity-Based Policy Enforcement](../assets/diagrams/)

**Figure 3: Cilium Network Security: Identity-Based Policy Enforcement**

The diagram illustrates the decision process. The policy check occurs sequentially: first the Identity relation, then the Layer 4 parameters (port/protocol). Only packets passing all checks reach the destination Pod.

## NetworkPolicy and CiliumNetworkPolicy

Cilium supports standard Kubernetes `NetworkPolicy` resources. It converts them into Identity-based eBPF rules automatically.

However, Cilium extends this capability with `CiliumNetworkPolicy` (CNP). CNP creates a superset of features, supporting:

- **Layer 7 Rules**: Filtering HTTP verbs or paths.
- **DNS Hostname Rules**: Allowing egress to specific domains (e.g., `*.github.com`) rather than IPs.
- **Node-Level Policies**: Restricting host-to-pod or pod-to-host traffic explicitly.

## Layer 3 and Layer 4 Enforcement

Most security policies operate at Layer 3 (IP/Identity) and Layer 4 (Port/Protocol).

- **Ingress Policies**: Control which identities can connect to a selected Pod.
- **Egress Policies**: Control which identities a selected Pod can connect to.
- **Default Deny**: If a Pod is selected by _any_ policy, Cilium switches that Pod to a "default deny" mode. Only traffic explicitly allowed by a policy is permitted.

For L3/L4 traffic, the entire enforcement process happens in eBPF at line rate.

## Layer 7 Awareness (High-Level)

Standard firewalls see traffic as opaque TCP streams. `CiliumNetworkPolicy` can inspect the application layer. Usage includes allowing an HTTP `GET` to `/public` while denying a `POST` to `/admin`.

To implement this, Cilium must parse the application protocol. Since eBPF is primarily designed for packet headers, Cilium redirects L7-policy-restricted traffic to a userspace proxy (Envoy) managed by the Cilium Agent.

- **Fast Path**: L3/L4 traffic stays in the kernel.
- **Slow Path**: Traffic requiring L7 inspection is proxied, inspected, and then forwarded or dropped.

This redirection only occurs for traffic matching specific L7 rules, minimizing latency penalties.

## Observability and Policy Debugging

A major challenge with network policies is distinguishing between network failures and security blocks. Cilium integrates with Hubble to provide explicit "Policy Verdicts".

When a packet is dropped, Hubble records:

- The Source and Destination Identities.
- The specific policy that caused the drop (or "default-deny" if no policy allowed it).
- The direction (Ingress/Egress).

This visibility allows engineers to audit security posture and debug connectivity issues without guessing.

## Common Security Misconfigurations

**Implicit Deny**: Users often apply a policy to restrict one port but forget to allow essential infrastructure traffic, such as DNS (UDP port 53). Without an explicit allow rule for DNS, the Pod loses all name resolution capabilities.

**Label Misuse**: Since identities derive from labels, incorrect labeling on Pods results in incorrect security posture. Platform teams must enforce strict labeling conventions.

**Policy Ordering**: Unlike firewalls with ordered lists, Kubernetes policies are additive. If _any_ policy allows a connection, it is allowed. Users cannot create a "deny" rule that overrides an "allow" rule (though Cilium Clusterwide Network Policies can provide distinct tiers).

## How This Scales in Production

Identity-based security scales efficiently.

- **O(1) Evaluation**: The complexity of checking a packet depends on the map lookup, not the number of policies.
- **Small Map Updates**: When a Deployment scales from 10 to 1,000 Pods, the policy map does not change because the _Identity_ (and the rules governing it) remains constant. Only the endpoint map (mapping IPs to that Identity) receives updates.
- **Caching**: The Cilium Agent caches identities to minimize API server load.

## Next Learning Steps

To deepen your understanding, investigate the following topics:

- **Cilium Policy Documentation**: Syntax references for CNP.
- **Hubble Policy Visibility**: Using the UI to audit drops.
- **Advanced L7 Policy**: Configuring proxy redirects for gRPC and HTTP.
