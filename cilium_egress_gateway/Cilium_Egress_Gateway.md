# Egress Gateway
Pods typically have ever-changing IP addresses in Kubernetes environments. Even if masquerading is used to mitigate this, the IP addresses of nodes can also change frequently. Egress gateways provide a way to route all outbound traffic from certain pods through a specific node with a predictable IP address. This predictable IP can be useful for scenarios where the traffic destination requires a known source IP, for instance, when working with legacy systems or firewall rules. Egress Gateway with Cilium fundamentally transforms Kubernetes networking by addressing dynamic IP challenges, ensuring seamless integration with legacy systems and enhancing network security. It provides precise control over traffic routing, enabling selective direction of pod traffic through stable, predictable IP addresses. This feature enables granular traffic management, effective monitoring and filtering, and workload-specific routing, all while facilitating interoperability with systems requiring known source IPs. The egress gateway allows fine-grained control over which pods' traffic should be routed through the gateway node. This is done by applying egress gateway policies that use label selectors to target specific pods. This selective routing can help in implementing security policies, achieving network isolation, and managing network costs. In multi-tenant Kubernetes clusters, different workloads might need to interact with different external systems that have specific network requirements. Egress gateways can help meet these requirements by allowing the configuration of workload-specific routing rules.

![Alt text](https://cilium.io/static/egress-3-8da78d8583f4045c9bddbab7e87dd03c.png)

When the egress gateway feature is enabled and egress gateway policies are in place, matching packets that leave the cluster are masqueraded with selected, predictable IPs associated with the gateway nodes. As an example, this feature can be used in combination with legacy firewalls to allow traffic to legacy infrastructure only from specific pods within a given namespace. The pods typically have ever-changing IP addresses, and even if masquerading was to be used as a way to mitigate this, the IP addresses of nodes can also change frequently over time. This document explains how to enable the egress gateway feature and how to configure egress gateway policies to route and SNAT the egress traffic for a specific workload.

# Preliminary Considerations
Cilium must make use of network-facing interfaces and IP addresses present on the designated gateway nodes. These interfaces and IP addresses must be provisioned and configured by the operator based on their networking environment. The process is highly-dependent on said networking environment. For example, in AWS/EKS, and depending on the requirements, this may mean creating one or more Elastic Network Interfaces with one or more IP addresses and attaching them to instances that serve as gateway nodes so that AWS can adequately route traffic flowing from and to the instances. Other cloud providers have similar networking requirements and constructs. Additionally, the enablement of the egress gateway feature requires that both BPF masquerading and the kube-proxy replacement are enabled, which may not be possible in all environments (due to, e.g., incompatible kernel versions).

# Delay for enforcement of egress policies on new pods
When new pods are started, there is a delay before egress gateway policies are applied for those pods. That means traffic from those pods may leave the cluster with a source IP address (pod IP or node IP) that doesn’t match the egress gateway IP. That egressing traffic will also not be redirected through the gateway node.

# Incompatible with other features
Because egress gateway isn’t compatible with identity allocation mode `kvstore`, you must use Kubernetes as Cilium’s identity store (`identityAllocationMode` set to `crd`). This is the default setting for new installations. Egress gateway is not compatible with the Cluster Mesh feature. The gateway selected by an egress gateway policy must be in the same cluster as the selected pods. Egress gateway is not compatible with the CiliumEndpointSlice feature. Egress gateway is not supported for IPv6 traffic.

# Enable egress gateway
The egress gateway feature and all the requirements can be enabled as follow:
1. Helm
```shell
helm upgrade cilium cilium/cilium \
   --namespace kube-system \
   --reuse-values \
   --set egressGateway.enabled=true \
   --set bpf.masquerade=true \
   --set kubeProxyReplacement=true
```
2. ConfigMap
```yml
enable-bpf-masquerade: true
enable-ipv4-egress-gateway: true
kube-proxy-replacement: true
```

Rollout both the agent pods and the operator pods to make the changes effective:
```shell
kubectl rollout restart ds cilium -n kube-system
kubectl rollout restart deploy cilium-operator -n kube-system
```

# Writing egress gateway policies
The API provided by Cilium to drive the egress gateway feature is the `CiliumEgressGatewayPolicy` resource. `CiliumEgressGatewayPolicy` is a cluster-scoped custom resource definition, so a `.metadata.namespace` field should not be specified. 
```yml
apiVersion: cilium.io/v2
kind: CiliumEgressGatewayPolicy
metadata:
  name: example-policy
```
To target pods belonging to a given namespace only labels/expressions should be used instead. The `selectors` field of a `CiliumEgressGatewayPolicy` resource is used to select source pods via a label selector. This can be done using `matchLabels`:
```yml
selectors:
- podSelector:
    matchLabels:
      labelKey: labelVal
```
It can also be done using matchExpressions:
```yml
selectors:
- podSelector:
    matchExpressions:
    - {key: testKey, operator: In, values: [testVal]}
    - {key: testKey2, operator: NotIn, values: [testVal2]}
```
Moreover, multiple podSelector can be specified:
```yml
selectors:
- podSelector:
  [..]
- podSelector:
  [..]
```
To select pods belonging to a given namespace, the special io.kubernetes.pod.namespace label should be used. To only select pods on certain nodes, you can use the nodeSelector:
```yml
selectors:
- podSelector:
    matchLabels:
      labelKey: labelVal
  nodeSelector:
    matchLabels:
      nodeLabelKey: nodeLabelVal
```
One or more IPv4 destination CIDRs can be specified with `destinationCIDRs`:
```yml
destinationCIDRs:
- "a.b.c.d/32"
- "e.f.g.0/24"
```
It’s possible to specify exceptions to the `destinationCIDRs` list with `excludedCIDRs`:
```yml
destinationCIDRs:
- "a.b.0.0/16"
excludedCIDRs:
- "a.b.c.0/24"
```

**Note**: Any IP belonging to these ranges which is also an internal cluster IP (e.g. pods, nodes, Kubernetes API server) will be excluded from the egress gateway SNAT logic.

The node that should act as gateway node for a given policy can be configured with the egressGateway field. The node is matched based on its labels, with the nodeSelector field:
```yml
egressGateway:
  nodeSelector:
    matchLabels:
      testLabel: testVal
```

**Note**: In case multiple nodes are a match for the given set of labels, the first node in lexical ordering based on their name will be selected.

**Note**: If there is no match for the given set of labels, Cilium drops the traffic that matches the destination CIDR(s).

The IP address that should be used to SNAT traffic must also be configured. There are 3 different ways this can be achieved:
1. By specifying the interface:
```yml
egressGateway:
  nodeSelector:
    matchLabels:
      testLabel: testVal
  interface: ethX
```
In this case the first IPv4 address assigned to the `ethX` interface will be used.
2. By explicitly specifying the egress IP:
```yml
egressGateway:
  nodeSelector:
    matchLabels:
      testLabel: testVal
  egressIP: a.b.c.d
```
**Warning**: The egress IP must be assigned to a network device on the node.
3. By omitting both egressIP and interface properties, which will make the agent use the first IPv4 assigned to the interface for the default route.
```yml
egressGateway:
  nodeSelector:
    matchLabels:
      testLabel: testVal
```

Regardless of which way the egress IP is configured, the user must ensure that Cilium is running on the device that has the egress IP assigned to it, by setting the --devices agent option accordingly.

**Warning**: The `egressIP` and `interface` properties cannot both be specified in the egressGateway spec. Egress Gateway Policies that contain both of these properties will be ignored by Cilium.
**Note**: When Cilium is unable to select the Egress IP for an Egress Gateway policy (for example because the specified `egressIP` is not configured for any network interface on the gateway node), then the gateway node will drop traffic that matches the policy with the reason `No Egress IP configured`.
**Note**: After Cilium has selected the Egress IP for an Egress Gateway policy (or failed to do so), it does not automatically respond to a change in the gateway node’s network configuration (for example if an IP address is added or deleted). You can force a fresh selection by re-applying the Egress Gateway policy.

# Example policy
Below is an example of a CiliumEgressGatewayPolicy resource that conforms to the specification above:
```yml
apiVersion: cilium.io/v2
kind: CiliumEgressGatewayPolicy
metadata:
  name: egress-sample
spec:
  # Specify which pods should be subject to the current policy.
  # Multiple pod selectors can be specified.
  selectors:
  - podSelector:
      matchLabels:
        org: empire
        class: mediabot
        # The following label selects default namespace
        io.kubernetes.pod.namespace: default
    nodeSelector: # optional, if not specified the policy applies to all nodes
      matchLabels:
        node.kubernetes.io/name: node1 # only traffic from this node will be SNATed
  # Specify which destination CIDR(s) this policy applies to.
  # Multiple CIDRs can be specified.
  destinationCIDRs:
  - "0.0.0.0/0"

  # Configure the gateway node.
  egressGateway:
    # Specify which node should act as gateway for this policy.
    nodeSelector:
      matchLabels:
        node.kubernetes.io/name: node2

    # Specify the IP address used to SNAT traffic matched by the policy.
    # It must exist as an IP associated with a network interface on the instance.
    egressIP: 10.168.60.100

    # Alternatively it's possible to specify the interface to be used for egress traffic.
    # In this case the first IPv4 assigned to that interface will be used as egress IP.
    # interface: enp0s8
```
Creating the `CiliumEgressGatewayPolicy` resource above would cause all traffic originating from pods with the `org: empire` and `class: mediabot` labels in the `default` namespace on node `node1` and destined to `0.0.0.0/0` (i.e. all traffic leaving the cluster) to be routed through the gateway node with the `node.kubernetes.io/name: node2` label, which will then SNAT said traffic with the `10.168.60.100` egress IP.