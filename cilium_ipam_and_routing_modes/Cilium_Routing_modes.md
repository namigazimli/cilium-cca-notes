Cilium offers a flexible networking approach for Kubernetes clusters. When it comes to routing traffic between nodes, Cilium provides two primary methods: encapsulation (usually via VXLAN or Geneve tunneling) and direct routing (often called “native routing”). Both methods have their advantages and trade-offs:

# Encapsulation
By default, when deploying Cilium, the encapsulate mode is enabled, which makes use of the GENEVE or VXLAN (SDN Overlays) encapsulation and tunneling mechanism. This method enables all nodes in a cluster to tunnel to each other creating n x n tunnels, where n is the number of nodes. As long as nodes have IP reachability to each other, traffic will be routed by default. This mode offers simpler onboarding, larger address space allocation for pods, automatic configuration of nodes, and maintenance of identity of packet metadata. However, if operating in this model, it’s highly recommended to enable jumbo frames on the physical network as the VXLAN/GENEVE tunneling adds additional overhead to the original packet. This may not always be possible and will lead to packet fragmentation, sending more packets than needed, and potentially sporadically having tunnels go offline. 

## Requirements on the network
- Encapsulation relies on normal node to node connectivity. This means that if Cilium nodes can already reach each other, all routing requirements are already met.
- The underlying network must support IPv4. See GitHub issue 17240 for the status of IPv6-based tunneling.
- The underlying network and firewalls must allow encapsulated packets:
|   Encapsulation mode  |   Port Range/Protocol   |
|-----------------------|-------------------------|
| VXLAN (default)       |   8472/UDP              |
| GENEVE                |   6081/UDP              |

## Advantages
- **Overlay simplicity**: Tunneling encapsulates cluster traffic in an overlay network, which means you don’t need to worry about the underlying network’s intricacies. This encapsulation abstracts away potential network conflicts and policies.
- **Uniform addressing**: In an encapsulated overlay, you have full control over IP addressing, making it easier to avoid IP conflicts with the underlying network. Due to not depending on any underlying networking limitations, the available addressing space is potentially much larger and allows to run any number of pods per node if the PodCIDR size is configured accordingly.
- **Potential for better multi-cluster support**: For setups that involve stretching clusters over different data centers or cloud providers, overlay networks can provide a consistent network space.
- **Auto-configuration**: When running together with an orchestration system such as Kubernetes, the list of all nodes in the cluster including their associated allocation prefix node is made available to each agent automatically. New nodes joining the cluster will automatically be incorporated into the mesh.
- **Identity context**: Encapsulation protocols allow for the carrying of metadata along with the network packet. Cilium makes use of this ability to transfer metadata such as the source security identity. The identity transfer is an optimization designed to avoid one identity lookup on the remote node.

## Disadvantages
- **Performance overhead**: Encapsulation and decapsulation of packets introduce some overhead, which might lead to slightly reduced network performance compared to native routing.
- **MTU considerations**: Due to encapsulation, the Maximum Transmission Unit (MTU) size of packets can be an issue. Encapsulated packets are larger, and if not managed correctly, this can lead to fragmentation or dropped packets. This results in a lower maximum throughput rate for a particular network connection. This can be largely mitigated by enabling jumbo frames (50 bytes of overhead for each 1500 bytes vs 50 bytes of overhead for each 9000 bytes).
- **Complexity**: While the overlay abstracts underlying network complexities, it also introduces its own set of complexities, like managing the overlay, potential tunneling issues, etc.

# Direct (Native) routing
The native routing datapath is enabled with `routing-mode: native` and enables the native packet forwarding mode. The native packet forwarding mode leverages the routing capabilities of the network Cilium runs on instead of performing encapsulation.
![Alt text](https://docs.cilium.io/en/stable/_images/native_routing.png)
In native routing mode, Cilium will delegate all packets which are not addressed to another local endpoint to the routing subsystem of the Linux kernel. This means that the packet will be routed as if a local process would have emitted the packet. As a result, the network connecting the cluster nodes must be capable of routing PodCIDRs. Cilium automatically enables IP forwarding in the Linux kernel when native routing is configured.

## Requirements on the network
- In order to run the native routing mode, the network connecting the hosts on which Cilium is running on must be capable of forwarding IP traffic using addresses given to pods or other workloads.
- The Linux kernel on the node must be aware on how to forward packets of pods or other workloads of all nodes running Cilium. This can be achieved in two ways:
    1. The node itself does not know how to route all pod IPs but a router exists on the network that knows how to reach all other pods. In this scenario, the Linux node is configured to contain a default route to point to such a router. This model is used for cloud provider network integration. See Google Cloud, AWS ENI, and Azure IPAM for more details.
    2. Each individual node is made aware of all pod IPs of all other nodes and routes are inserted into the Linux kernel routing table to represent this. If all nodes share a single L2 network, then this can be taken care of by enabling the option `auto-direct-node-routes: true`. Otherwise, an additional system component such as a BGP daemon must be run to distribute the routes. See the guide Using Kube-Router to Run BGP (deprecated) on how to achieve this using the kube-router project.

## Configuration
The following configuration options must be set to run the datapath in native routing mode:
- `routing-mode: native`: Enable native routing mode.
- `ipv4-native-routing-cidr: x.x.x.x/y`: Set the CIDR in which native routing can be performed.
The following configuration options are optional when running the datapath in native routing mode:
- `direct-routing-skip-unreachable`: If a BGP daemon is running and there is multiple native subnets to the cluster network, `direct-routing-skip-unreachable: true` can be added alongside `auto-direct-node-routes` to give each node L2 connectivity in each zone without traffic always needing to be routed by the BGP routers.

## Advantages
- **Performance**: Native routing generally offers better performance since it avoids the overhead of encapsulation. Traffic flows directly between nodes without the added layer of tunneling.
- **Simplicity in networking**: Without the tunneling layer, the networking stack has fewer layers to traverse, potentially reducing troubleshooting complexity in certain scenarios.
- **No MTU issues from encapsulation**: Since there’s no encapsulation, you avoid the MTU size issues associated with overlay networks.

## Disadvantages
- **Requires a cooperative underlying network**: The underlying network must be aware of the Pod CIDRs, and routes must be propagated appropriately. This might not always be possible, especially in certain cloud environments.
- **IP addressing conflicts**: Without an overlay to provide a separate address space, you must ensure that the Pod CIDRs do not conflict with the underlying network’s other IP ranges.
- **Potential security concerns**: Depending on your environment, using direct routing might expose more of the cluster’s internal traffic patterns to the underlying network, which could be a concern if the environment isn’t fully trusted.