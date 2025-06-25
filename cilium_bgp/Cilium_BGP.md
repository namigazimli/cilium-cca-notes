# Cilium BGP Control Plane
BGP Control Plane provides a way for Cilium to advertise routes to connected routers by using the [Border Gateway Protocol](https://datatracker.ietf.org/doc/html/rfc4271) (BGP). BGP Control Plane makes Pod networks and/or Services reachable from outside the cluster for environments that support BGP. Because BGP Control Plane does not program the [datapath](https://docs.cilium.io/en/stable/network/ebpf/#ebpf-datapath), do not use it to establish reachability within the cluster.

# Installation
1. Helm - Cilium BGP Control Plane can be enabled with Helm flag `bgpControlPlane.enabled` set as true.
```shell
helm upgrade cilium cilium/cilium --version 1.17.5 \
    --namespace kube-system \
    --reuse-values \
    --set bgpControlPlane.enabled=true
kubectl -n kube-system rollout restart ds/cilium
```
2. Cilium CLI - Cilium BGP Control Plane can be enabled with the following command
```shell
cilium install --version 1.17.0 --set bgpControlPlane.enabled=true
```
IPv4/IPv6 single-stack and dual-stack setup are supported. Note that the BGP Control Plane can only advertise the route of the address family that the Cilium is configured to use. You cannot advertise IPv4 routes when the Cilium Agent is configured to use only IPv6 address family. Conversely, you cannot advertise IPv6 routes when Cilium Agent is configured to use only IPv4 address family.

# Configuring BGP Control Plane
There are two ways to configure the BGP Control Plane. Using legacy `CiliumBGPPeeringPolicy` resource, or using newer BGP resources like `CiliumBGPClusterConfig`. Currently, both configuration options are supported, however `CiliumBGPPeeringPolicy` will be deprecated in the future.

# BGP Control Plane Resources
Cilium BGP control plane is managed by a set of custom resources which provide a flexible way to configure BGP peers, policies, and advertisements. The following resources are used to manage the BGP Control Plane:
- `CiliumBGPClusterConfig`: Defines BGP instances and peer configurations that are applied to multiple nodes.
- `CiliumBGPPeerConfig`: A common set of BGP peering setting. It can be used across multiple peers.
- `CiliumBGPAdvertisement`: Defines prefixes that are injected into the BGP routing table.
- `CiliumBGPNodeConfigOverride`: Defines node-specific BGP configuration to provide a finer control.
The relationship between various resources is shown in the below diagram:

![Alt text](https://docs.cilium.io/en/stable/_images/bgpv2.png)

# BGP Cluster Configuration
`CiliumBGPClusterConfig` resource is used to define BGP configuration for one or more nodes in the cluster based on its `nodeSelector` field. Each `CiliumBGPClusterConfig` defines one or more BGP instances, which are uniquely identified by their `name` field. A BGP instance can have one or more peers. Each peer is uniquely identified by its name field. The Peer autonomous number and peer address are defined by the `peerASN` and `peerAddress` fields, respectively. The configuration of the peers is defined by the `peerConfigRef` field, which is a reference to a peer configuration resource. `Group` and `kind` in `peerConfigRef` are optional and default to `cilium.io` and `CiliumBGPPeerConfig`, respectively.

**Warning**: The `CiliumBGPPeeringPolicy` and `CiliumBGPClusterConfig` should not be used together. If both resources are present and Cilium agent matches with both based on the node selector, `CiliumBGPPeeringPolicy` will take precedence.

Here is an example configuration of the `CiliumBGPClusterConfig` with a BGP instance named `instance-65000` and two peers configured under this BGP instance.
```yml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPClusterConfig
metadata:
  name: cilium-bgp
spec:
  nodeSelector:
    matchLabels:
      rack: rack0
  bgpInstances:
  - name: "instance-65000"
    localASN: 65000
    peers:
    - name: "peer-65000-tor1"
      peerASN: 65000
      peerAddress: fd00:10:0:0::1
      peerConfigRef:
        name: "cilium-peer"
    - name: "peer-65000-tor2"
      peerASN: 65000
      peerAddress: fd00:11:0:0::1
      peerConfigRef:
        name: "cilium-peer"
```

# BGP Peer Configuration
The CiliumBGPPeerConfig resource is used to define a BGP peer configuration. Multiple peers can share the same configuration and provide reference to the common CiliumBGPPeerConfig resource. The CiliumBGPPeerConfig resource contains configuration options for:
- [MD5 Password](https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-v2/#bgp-peer-configuration-password)
- [Timers](https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-v2/#bgp-peer-configuration-timers)
- [EBGP Multihop](https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-v2/#bgp-ebgp-multihop)
- [Graceful Restart](https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-v2/#bgp-peer-configuration-graceful-restart)
- [Transport](https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-v2/#bgp-peer-configuration-transport)
- [Address Families](https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-v2/#bgp-peer-configuration-afi)
Here is an example configuration of the `CiliumBGPPeerConfig` resource. In the next section, we will go over each configuration option.
```yml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeerConfig
metadata:
  name: cilium-peer
spec:
  timers:
    holdTimeSeconds: 9
    keepAliveTimeSeconds: 3
  authSecretRef: bgp-auth-secret
  ebgpMultihop: 4
  gracefulRestart:
    enabled: true
    restartTimeSeconds: 15
  families:
    - afi: ipv4
      safi: unicast
      advertisements:
        matchLabels:
          advertise: "bgp"
```

# BGP Advertisements
The `CiliumBGPAdvertisement` resource is used to define various advertisement types and attributes associated with them. The `advertisements` label selector defined in the `families` field of a peer configuration may match with one or more of the `CiliumBGPAdvertisement` resources.

# BGP Attributes
You can configure BGP path attributes for the prefixes advertised by Cilium BGP control plane using `attributes` field in `advertisements[*]`. There are two types of Path Attributes that can be advertised: `Communities` and `LocalPreference`. Here is an example configuration of the `CiliumBGPAdvertisement` resource that advertises pod prefixes with the community value of “65000:99” and local preference of 99.
```yml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPAdvertisement
metadata:
  name: bgp-advertisements
  labels:
    advertise: bgp
spec:
  advertisements:
    - advertisementType: "PodCIDR"
      attributes:
        communities:
          standard: [ "65000:99" ]
        localPreference: 99
```

# BGP Configuration Override
The `CiliumBGPNodeConfigOverride` resource can be used to override some of the auto-generated configuration on a per-node basis. Here is an example of the `CiliumBGPNodeConfigOverride` resource, that sets Router ID and local address used in each peer for the node with a name `bgpv2-cplane-dev-multi-homing-worker`.
```yml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPNodeConfigOverride
metadata:
  name: bgpv2-cplane-dev-multi-homing-worker
spec:
  bgpInstances:
    - name: "instance-65000"
      routerID: "192.168.10.1"
      localPort: 1790
      peers:
        - name: "peer-65000-tor1"
          localAddress: fd00:10:0:2::2
        - name: "peer-65000-tor2"
          localAddress: fd00:11:0:2::2
```
The name of `CiliumBGPNodeConfigOverride` resource must match the name of the node for which the configuration is intended. Similarly, the names of the BGP instance and peers must match with what is defined under `CiliumBGPClusterConfig`.

You can check out the following blogs:
- [Connecting your Kubernetes island to your network with Cilium BGP](https://isovalent.com/blog/post/connecting-your-kubernetes-island-to-your-network-with-cilium-bgp/)