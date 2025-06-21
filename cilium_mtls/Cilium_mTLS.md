Mutual Transport Layer Security (mTLS) is a mechanism that ensures the authenticity, integrity, and confidentiality of data exchanged between two entities over a network. Unlike traditional TLS, which involves a one-way authentication process where the client verifies the server’s identity, mutual TLS adds an additional layer of security by requiring both the client and the server to authenticate each other. Mutual TLS aims at providing authentication, confidentiality and integrity to service-to-service communications.

# Mutual Authentication in Cilium

Cilium’s mTLS-based Mutual Authentication support brings the mutual authentication handshake out-of-band for regular connections. For Cilium to meet most of the common requirements for service-to-service authentication and encryption, users must enable encryption. To address the challenge of identity verification in dynamic and heterogeneous environments, mutual authentication requires a framework secure identity verification for distributed systems. In Cilium’s current mutual authentication support, identity management is provided through the use of SPIFFE (Secure Production Identity Framework for Everyone).

![Alt text](https://cilium.io/static/e32fdb05a8b46d6010d2cf8837135bda/782ec/general-architecture.png)

# SPIFFE benefits

Here are some of the benefits provided by [SPIFFE](https://spiffe.io/):
- **Trustworthy identity issuance**: SPIFFE provides a standardized mechanism for issuing and managing identities. It ensures that each service in a distributed system receives a unique and verifiable identity, even in dynamic environments where services may scale up or down frequently.
- **Identity attestation**: SPIFFE allows services to prove their identities through attestation. It ensures that services can demonstrate their authenticity and integrity by providing verifiable evidence about their identity, like digital signatures or cryptographic proofs.
- **Dynamic and scalable environments**: SPIFFE addresses the challenges of identity management in dynamic environments. It supports automatic identity issuance, rotation, and revocation, which are critical in cloud-native architectures where services may be constantly deployed, updated, or retired.

# Cilium and SPIFFE

SPIFFE provides an API model that allows workloads to request an identity from a central server. In our case, a workload means the same thing that a Cilium Security Identity does - a set of pods described by a label set. A SPIFFE identity is a subclass of URI, and looks something like this: `spiffe://trust.domain/path/with/encoded/info`.

There are two main parts of a SPIFFE setup:
- A central SPIRE server, which forms the root of trust for the trust domain.
- A per-node SPIRE agent, which first gets its own identity from the SPIRE server, then validates the identity requests of workloads running on its node.

When a workload wants to get its identity, usually at startup, it connects to the local SPIRE agent using the SPIFFE workload API, and describes itself to the agent. The SPIRE agent then checks that the workload is really who it says it is, and then connects to the SPIRE server and attests that the workload is requesting an identity, and that the request is valid. The SPIRE agent checks a number of things about the workload, that the pod is actually running on the node it’s coming from, that the labels match, and so on. Once the SPIRE agent has requested an identity from the SPIRE server, it passes it back to the workload in the SVID (SPIFFE Verified Identity Document) format. This document includes a TLS keypair in the X.509 version. In the usual flow for SPIRE, the workload requests its own information from the SPIRE server. In Cilium’s support for SPIFFE, the Cilium agents get a common SPIFFE identity and can themselves ask for identities on behalf of other workloads. 

![Alt text](https://cilium.io/static/4d145735e3c36219e3845b51448f90fa/d2835/connection-based-mutual-auth.png)

# Prerequisites

- Mutual authentication is only currently supported with SPIFFE APIs for certificate management.
- The Cilium Helm chart includes an option to deploy a SPIRE server for mutual authentication. You may also deploy your own SPIRE server and configure Cilium to use it.

# Installation

You can enable mutual authentication and its associated SPIRE server with the following command. This command requires the Cilium CLI Helm mode version 0.15 or later.

```bash
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  --set authentication.mutual.spire.enabled=true \
  --set authentication.mutual.spire.install.enabled=true
```

# Benefits of Cilium's Mutual Authentication Layer

By separating the authentication handshake from data, several benefits are gained:
- The Implementation of mTLS-based authentication is simplified as it can be rolled out service by service easily.
- Any network protocol that is supported. No limitation to TCP only.
- The secrets used for authentication are safely kept away from any L3-L7 processing. This resolves a significant attack vector found in L7 proxy-based mTLS.
- Key rotation for authentication and encryption can be performed on live connections without disruptions.