This repository contains a comprehensive collection of notes, configurations, and lab guides focused on Cilium, eBPF, and various aspects of Kubernetes networking, security, and observability. It serves as a personal learning resource, detailing concepts, practical implementations, and troubleshooting tips.

# Key Topics & Features
This repository covers a wide array of topics, organized to help understand the core principles and advanced functionalities:

# Cilium Architecture & Core Concepts
- **What is Cilium?** Explores Cilium as an open-source solution for networking, security, and observability in cloud-native environments, powered by eBPF.
- **Core Components**: Details the roles of the Cilium Agent (managing eBPF programs, network policies, load balancing, and identity management), Cilium Operator (handling cluster-wide resources and IPAM), CNI Plugin, and Cilium CLI for interaction and debugging.
- **Identity-based Security**: Emphasizes Cilium's approach to decoupling security from IP addresses by using workload identity derived from labels and metadata, enabling dynamic policy enforcement.

# Networking Features
- **IP Address Management (IPAM)**: Covers different IPAM modes including Kubernetes Host Scope, Cluster Scope, and Multi-Pool mode, highlighting their allocation strategies and flexibility.
- **Routing Modes**: Discusses Encapsulation (VXLAN, GENEVE) for network overlays and Native/Direct Routing for optimal performance when the underlying network supports Pod CIDRs.
- **Service Load Balancing**: Explores how Cilium handles internal (ClusterIP) and external (NodePort, LoadBalancer) load balancing, including advanced techniques like Direct Server Return (DSR) and Layer 7 (L7) load balancing for HTTP, gRPC, and Kafka.
- **L2 Announcements / LB IPAM**: Details how Cilium can announce Service IP addresses on the local area network using ARP, particularly useful for bare-metal deployments without BGP, and how LB IPAM automatically assigns IPs to LoadBalancer Services.
- **Egress Gateway**: Provides methods to route all outbound traffic from specific pods through a designated node with a predictable IP address, essential for integrating with external firewalls and legacy systems.
- **Cluster Mesh**: Explains how to connect multiple Kubernetes clusters for cross-cluster pod-to-pod communication, global services for load balancing between clusters, and fault tolerance.
- **BGP Control Plane**: Covers configuring Cilium to advertise Pod networks and/or Services to external routers using Border Gateway Protocol (BGP).

# Security Features
- **Cilium Network Policies**: In-depth coverage of defining granular network security rules at Layers 3 (IP/CIDR, Entities, DNS Names), 4 (TCP, UDP), and 7 (HTTP, gRPC, Kafka) based on workload identity (labels).
- **Transparent Encryption**: How Cilium encrypts traffic between endpoints using IPsec or WireGuard, ensuring data confidentiality without application changes.
- **Mutual TLS (mTLS)**: Details Cilium's support for mTLS-based mutual authentication, leveraging SPIFFE for secure identity verification for service-to-service communications.
- **Host Firewall**: Extends network policy enforcement to the nodes themselves, allowing consistent security across both pods and the underlying hosts.
- **Runtime Security & Observability (Tetragon)**: Introduction to Cilium Tetragon for deep visibility into process execution, network sockets, and file access at the kernel level, enabling detection and prevention of malicious activities.

# Observability
- **Hubble**: Comprehensive platform for network and security observability, built on Cilium and eBPF. Provides real-time flow monitoring, network security auditing, and event visualization through a CLI and UI, offering deep insights into pod-to-pod communication and policy enforcement.
- **eBPF Tracing**: How eBPF enables high-fidelity, low-overhead tracing and metrics directly from the kernel, providing detailed visibility into system calls, network events, and more, without altering applications.

# Integration & Migration
- **Kubernetes Ingress & Gateway API**: Explores Cilium's native support for Kubernetes Ingress and the newer Gateway API. Covers HTTP routing, TLS termination and passthrough, traffic splitting, header modification, and cross-namespace routing capabilities.
- **CNI Migration**: Guides on migrating from other CNIs (e.g., Calico) to Cilium with minimal disruption, using features like dual overlays and per-node configuration.
- **MetalLB Migration**: Explains how Cilium's LB IPAM can replace external load balancer solutions like MetalLB for bare-metal clusters.

# Repository Structure
The content is organized into directories, generally corresponding to major Cilium features or related concepts. Each directory contains Markdown files (.md) for notes and lab steps, and sometimes PDF documents (.pdf) for deeper dives or cheatsheets.
- `cilium_architecture/`: Core Cilium concepts, components, and use cases.
- `cilium_bgp/`: Notes and labs on BGP integration.
- `cilium_cluster_mesh/`: Guides for multi-cluster setups and global services.
- `cilium_ebpf_books/`: PDFs on eBPF fundamentals and its application in networking/security.
- `cilium_egress_gateway/`: Details on configuring egress policies.
- `cilium_ingress_gateway_api/`: Information on Kubernetes Ingress and Gateway API with Cilium.
- `cilium_labs_isovalent/`: Lab guides from Isovalent, covering various Cilium features.
- `cilium_labs_kodekloud/`: Lab guides from KodeKloud, focusing on network policies.
- `cilium_loadbalancer_ipam&l2_service_announcement/`: Notes on IPAM for LoadBalancers and L2 announcements.
- `cilium_mtls/`: Deep dive into mTLS and transparent encryption.
- `cilium_network_policies/`: Detailed examples and tutorials for Cilium Network Policies.
- `cilium_observability_with_hubble/`: Explores Hubble for network observability.

# How to Use/Explore
- **Navigate the Directories**: Browse through the folders to find topics of interest.
- **Read Markdown Files**: Most conceptual explanations and lab steps are in .md files, which are easy to read directly in GitHub.
- **Consult PDFs**: The cilium_ebpf_books/ directory contains valuable PDF resources for deeper understanding of eBPF and related areas.
- **Follow Lab Guides**: The lab guides provide step-by-step instructions to set up and experiment with Cilium features. These are invaluable for hands-on learning.

# Audience
This repository is beneficial for:
- **Kubernetes Administrators & Operators**: To understand and implement advanced networking and security within their clusters.
- **Network Engineers**: Transitioning into cloud-native networking, utilizing familiar concepts like BGP and ARP in a Kubernetes context.
- **Security Professionals**: Focusing on network segmentation, traffic enforcement, and runtime security in containerized environments.
- **Anyone preparing for Cilium certifications(e.g., CCA)**: The content directly correlates with practical knowledge required for these certifications.

**Resources**
- [Cilium Official Documentation](https://docs.cilium.io/en/stable/)
- [Kodekloud Cilium Course](https://kodekloud.com/courses/cilium-certified-associate-cca)
- [Isovalent labs](https://isovalent.com/labs/)
- [Isovalent blogs](https://isovalent.com/blog/)
- [Isovalent books](https://isovalent.com/books/)
- [Linux Foundation Cilium course](https://training.linuxfoundation.org/training/introduction-to-cilium-lfs146/)