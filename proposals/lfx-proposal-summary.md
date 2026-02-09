# LFX Mentorship Proposal: Cilium Project Pillar Pages

## 1. Maintainer-Level Summary

**To:** Cilium Maintainers / CNCF Review Committee
**Subject:** Proposal for Creation of 8 Technical Pillar Pages for Cilium.io

The Cilium project has evolved from a sophisticated CNI plugin into a comprehensive platform for cloud-native networking, security, and observability. While the technical documentation is extensive, there is a significant gap in high-level architectural content that bridges the user journey from "initial discovery" to "technical evaluation."

Currently, users arriving at cilium.io via search engines often land on specific reference docs or blog posts that lack context. This creates friction adoption, as users must piece together the "why" and "how" from disparate sources.

This project solves this problem by establishing a structured series of **SEO Pillar Pages**. These pages serve as the canonical entry points for the core technical domains of Cilium: Networking, eBPF, Security, Observability, Service Mesh, Multi-Cluster, External Networking, and High-Performance Data Path.

Each pillar is designed to be:

1.  **Technically Rigorous**: Accurate descriptions of kernel-level mechanisms (eBPF, XDP, TC).
2.  **Production-Oriented**: Focused on real-world constraints (latency, scale, failure modes).
3.  **Vendor-Neutral**: Aligned with CNCF values, avoiding marketing hyperbole.

This initiative is high-impact because it directly improves the "top of funnel" for adoption. By answering the fundamental architectural questions clearly and authoritatively, we reduce the support burden on maintainers and accelerate the evaluation process for enterprise users.

---

## 2. LFX Proposal Structure

**Project Title:** Creation of Comprehensive Technical Pillar Pages for Cilium.io
**Project Description:** Develop a set of 8 deep-dive technical articles that serve as the authoritative reference for Cilium's core capabilities, filling the gap between marketing overviews and raw API documentation.

### Abstract

Cilium is complex. New users often struggle to understand how its components (eBPF, Hubble, Cluster Mesh) fit together. This project delivers a structured content library that explains these concepts from first principles, leveraging conceptual diagrams and production-grade examples.

### Problem Statement

- **Fragmentation**: Advanced concepts like "eBPF-based Service Mesh" or "BGP Control Plane" are scattered across blog posts and release notes.
- **Steep Learning Curve**: Existing docs assume prior knowledge, alienating intermediate users.
- **Search Visibility**: The project lacks high-ranking pages for generic terms like "Kubernetes Network Security" or "Multi-Cluster K8s".

### Deliverables

A complete series of 8 Markdown documents, peer-reviewed and ready for publication:

1.  Kubernetes Networking at Scale
2.  eBPF Explained Through Cilium
3.  Network Security with Cilium
4.  Observability with Cilium and Hubble
5.  Sidecar-Free Service Mesh
6.  Multi-Cluster Networking (Cluster Mesh)
7.  External Networking (BGP/Egress)
8.  High-Performance Datapath (XDP/Maglev)

### Technical Approach

- **Research**: Synthesize existing docs, talk announcements, and code comments.
- **Drafting**: Create content that prioritizes "Why" (architecture) over "How" (CLI commands).
- **Validation**: Ensure technical accuracy with maintainer reviews for each pillar.
- **Integration**: Format for direct integration into the existing Hugo/Gatsby site structure.

### Timeline (12 Weeks)

- **Weeks 1-2**: Research & Outline Finalization.
- **Weeks 3-4**: Draft Pillars 1 & 2 (Foundations).
- **Weeks 5-6**: Draft Pillars 3 & 4 (Security/Obs).
- **Weeks 7-8**: Draft Pillars 5 & 6 (Mesh/Cluster).
- **Weeks 9-10**: Draft Pillars 7 & 8 (Advanced Networking).
- **Weeks 11-12**: Final Review, SVG Asset Finalization, and Merge.

### Benefits to Community

- **Improved Education**: A clear learning path for new contributors and users.
- **Reduced Support Load**: Answering common architectural questions in static content.
- **SEO Authority**: Establishing cilium.io as the definitive source for cloud-native networking knowledge.

---

## 3. Deliverable Mapping

| Deliverable                 | Pillar Reference                | Outcome                                        | Validation Method                             |
| :-------------------------- | :------------------------------ | :--------------------------------------------- | :-------------------------------------------- |
| **Networking Fundamentals** | Pillar 1: Kubernetes Networking | Canonical guide to K8s model vs Cilium         | Maintainer Review                             |
| **eBPF Deep Dive**          | Pillar 2: eBPF Explained        | Technical explanation of kernel hooks          | Technical Accuracy Check                      |
| **Security Architecture**   | Pillar 3: Network Security      | Identity-based policy explanation              | Comparison with existing Policy docs          |
| **Observability Guide**     | Pillar 4: Observability         | Hubble architecture & flow explanation         | Feature set verification                      |
| **Service Mesh**            | Pillar 5: Sidecar-Free          | Explaining L7/Envoy integration                | Architecture diagram review                   |
| **Multi-Cluster**           | Pillar 6: Cluster Mesh          | Identity sync & shared state conceptualization | Review by Cluster Mesh team                   |
| **Physical Network**        | Pillar 7: External Networking   | BGP/Egress Gateway use cases                   | Verification against current BGP capabilities |
| **High Performance**        | Pillar 8: Datapath (XDP)        | XDP/Maglev/DSR technical breakdown             | Kernel-level correctness check                |

---

## 4. Risk & Mitigation

| Risk                   | Impact                                                 | Mitigation                                                                                                          |
| :--------------------- | :----------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------ |
| **Content Accuracy**   | Misleading users about technical details.              | Strict technical review process with subject matter experts (SMEs) for each pillar.                                 |
| **Kernel Drift**       | Docs becoming obsolete as kernel/eBPF features evolve. | Focus on architectural principles (which are stable) rather than specific kernel version flags.                     |
| **SEO Misalignment**   | Content failing to rank or attract traffic.            | Pre-validation of search intent; matching headings to common engineering queries.                                   |
| **Review Bottlenecks** | Delays in merging due to maintainer workload.          | Delivering pillars in batches of 2 to spread review load; ensuring content is "99% ready" before requesting review. |

---

## 5. Final Submission Checklist

**Already Complete:**

- [x] Full text for all 8 Pillar Pages.
- [x] Production-grade Architectural Diagrams (28+ Custom SVGs).
- [x] Consistent CNCF technical styling across all assets.
- [x] Content validated against current Cilium feature set.
- [x] LFX Writing Sample PDF Generated (with embedded SVG diagrams).

**Remaining Before Merge:**

- [ ] Maintainer technical sign-off.
- [ ] Integration into website build system (PR creation).
- [ ] Final SEO meta-tag optimization.

**Success Criteria:**

- All 8 pages merged into `cilium/cilium.io` repository.
- Positive feedback from the community regarding clarity.
- Measurable increase in organic search traffic to documentation sections.

---

_This proposal is ready for submission to the LFX Mentorship Platform._
