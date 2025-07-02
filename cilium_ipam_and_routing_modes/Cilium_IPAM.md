IP Address Management (IPAM) is responsible for the allocation and management of IP addresses used by network endpoints (container and others) managed by Cilium. Don’t change the IPAM mode of an existing cluster. Changing the IPAM mode in a live environment may cause persistent disruption of connectivity for existing workloads. The safest path to change IPAM mode is to install a fresh Kubernetes cluster with the new IPAM configuration.
- [Cluster Scope (default)](https://docs.cilium.io/en/stable/network/concepts/ipam/cluster-pool/)
- [Kubernetes Host Scope](https://docs.cilium.io/en/stable/network/concepts/ipam/kubernetes/)
- [Multi-Pool (beta)](https://docs.cilium.io/en/stable/network/concepts/ipam/multi-pool/)
- [Azure IPAM](https://docs.cilium.io/en/stable/network/concepts/ipam/azure/)
- [AWS ENI](https://docs.cilium.io/en/stable/network/concepts/ipam/eni/)
- [Google Kubernetes Engine](https://docs.cilium.io/en/stable/network/concepts/ipam/gke/)
- [CRD-backed](https://docs.cilium.io/en/stable/network/concepts/ipam/crd/)

# Kubernetes Host Scope
The Kubernetes host-scope IPAM mode is enabled with ipam: kubernetes and delegates the address allocation to each individual node in the cluster. IPs are allocated out of the PodCIDR range associated to each node by Kubernetes. 
![Alt text](https://docs.cilium.io/en/stable/_images/k8s_hostscope.png)
In this mode, the Cilium agent will wait on startup until the PodCIDR range is made available via the Kubernetes v1.Node object for all enabled address families via one of the following methods:
1. via v1.Node resource field
|         Field         |            Description            |
|-----------------------|-----------------------------------|
|   spec.podCIDRs       | IPv4 and/or IPv6 PodCIDR range    |
|   spec.podCIDR        | IPv4 or IPv6 PodCIDR range        |

**Note** - It is important to run the `kube-controller-manager` with the flag `--allocate-node-cidrs` flag to indicate to Kubernetes that PodCIDR ranges should be allocated.
2. via v1.Node annotation

|       Annotation       |            Description            |
|------------------------|-----------------------------------|
| network.cilium.io/ipv4-pod-cidr | IPv4 PodCIDR range                |
| network.cilium.io/ipv6-pod-cidr | IPv6 PodCIDR range                |
| network.cilium.io/ipv4-cilium-host | IPv4 address of the cilium host interface |
| network.cilium.io/ipv6-cilium-host | IPv6 address of the cilium host interface |
| network.cilium.io/ipv4-health-ip | IPv4 address of the cilium-health endpoint |
| network.cilium.io/ipv6-health-ip | IPv6 address of the cilium-health endpoint |
| network.cilium.io/ipv4-Ingress-ip | IPv4 address of the cilium-ingress endpoint |
| network.cilium.io/ipv6-Ingress-ip | IPv6 address of the cilium-ingress endpoint |

**Note** - The annotation-based mechanism is primarily useful in combination with older Kubernetes versions which do not support spec.podCIDRs yet but support for both IPv4 and IPv6 is enabled.

## Configuration
The following **ConfigMap** options exist to configure Kubernetes host-scope:
- `ipam: kubernetes`: Enables Kubernetes IPAM mode. Enabling this option will automatically enable `k8s-require-ipv4-pod-cidr` if `enable-ipv4` is true and `k8s-require-ipv6-pod-cidr` if `enable-ipv6` is true.
- `k8s-require-ipv4-pod-cidr: true`: instructs the Cilium agent to wait until an IPv4 PodCIDR is made available via the Kubernetes node resource.
- `k8s-require-ipv6-pod-cidr: true`: instructs the Cilium agent to wait until an IPv6 PodCIDR is made available via the Kubernetes node resource.
With **Helm** the previous options can be defined as:
- `ipam: kubernetes`: `--set ipam.mode=kubernetes`
- `k8s-require-ipv4-pod-cidr: true`: `--set k8s.requireIPv4PodCIDR=true`, which only works with `--set ipam.mode=kubernetes`
- `k8s-require-ipv6-pod-cidr: true`: `--set k8s.requireIPv6PodCIDR=true`, which only works with `--set ipam.mode=kubernetes`

When using the Kubernetes Host Scope, Cilium relies on CIDRs already allocated to the Node Kubernetes resources by the Kubernetes Controller Manager. In this mode, a single large cluster-wide prefix is assigned to a cluster, and Kubernetes carves out a subnet of that prefix for every node. Cilium would then assign IP addresses from each subnet. Let's take a look in a cluster with Cilium deployed in this mode:
```shell
cilium config view | grep ipam
ipam                                       kubernetes
```
Cilium will use the PodCIDRs associated with each node (using the `Node` resources) and assign IPs from these subnets to the pods started on the nodes. In our cluster, the `kind-worker` node is assigned the `10.244.1.0/24` PodCIDR.
```shell
kubectl get ciliumnode kind-worker -o jsonpath='{.spec.ipam}'
{"podCIDRs":["10.244.1.0/24"]}
```
Executing cilium-dbg status in a Cilium agent will let you check how many IP addresses were assigned out of that prefix (note that [the Cilium CLI running on the agent](https://docs.cilium.io/en/stable/cheatsheet/) is different from the [Cilium CNI binary typically used for installation](https://isovalent.com/blog/post/cilium-cheat-sheet/)):
```shell
WORKER_CILIUM_POD=$(kubectl -n kube-system get po -l k8s-app=cilium --field-selector spec.nodeName=kind-worker -o name)
echo $WORKER_CILIUM_POD
pod/cilium-7krnk
kubectl -n kube-system exec -ti $WORKER_CILIUM_POD -c cilium-agent -- cilium-dbg status | grep IPAM
IPAM:                    IPv4: 5/254 allocated from 10.244.1.0/24
```
While this mode is simple, it is inflexible. With it, it is not possible to configure the size of the CIDRs allocated to each node, nor is it possible to add additional CIDRs to the cluster or to individual nodes, making precise IP address planning crucial prior to cluster deployment.

# Cluster Scope (default)
The cluster-scope IPAM mode assigns per-node PodCIDRs to each node and allocates IPs using a host-scope allocator on each node. It is thus similar to the Kubernetes Host Scope mode. The difference is that instead of Kubernetes assigning the per-node PodCIDRs via the Kubernetes `v1.Node` resource, the Cilium operator will manage the per-node PodCIDRs via the `v2.CiliumNode` resource. The advantage of this mode is that it does not depend on Kubernetes being configured to hand out per-node PodCIDRs.

## Architecture
![Alt text](https://docs.cilium.io/en/stable/_images/cluster_pool.png)
This is useful if Kubernetes cannot be configured to hand out PodCIDRs or if more control is needed. In this mode, the Cilium agent will wait on startup until the podCIDRs range are made available via the Cilium Node `v2.CiliumNode` object for all enabled address families via the resource field set in the `v2.CiliumNode`:

|         Field         |            Description            |
|-----------------------|-----------------------------------|
|   spec.ipam.podCIDRs  | IPv4 and/or IPv6 PodCIDR range    |

## Expanding the cluster pool
Don’t change any existing elements of the `clusterPoolIPv4PodCIDRList` list, as changes cause unexpected behavior. If the pool is exhausted, add a new element to the list instead. The minimum mask length is `/30`, with a recommended minimum mask length of at least `/29`. The reason to add new elements rather than change existing elements is that the allocator reserves 2 IPs per CIDR block for the network and broadcast addresses. Changing `clusterPoolIPv4MaskSize` is also not possible.

## Check for conflicting node CIDRs
`10.0.0.0/8` is the default pod CIDR. If your node network is in the same range you will lose connectivity to other nodes. All egress traffic will be assumed to target pods on a given node rather than other nodes. You can solve it in two ways:
- Explicitly set `clusterPoolIPv4PodCIDRList` to a non-conflicting CIDR
- Use a different CIDR for your nodes
The Cluster Scope mode works in a similar way to the Kubernetes Host Scope mode, but Cilium allocates pod CIDRs to the nodes itself (instead of Kube Controller Manager). As it doesn't require a specific configuration of the Kubernetes cluster, this mode is the default option in Cilium. In this mode, the Cilium Operator is in charge of allocating pod CIDRs to each node. It uses the `CiliumNode` resources instead of the `Node` resources to achieve this (this avoids potential clashes with CIDRs assigned by the Kubernetes Controller Manager). This mode is easy to understand and lets you assign multiple IP ranges per cluster but comes with some limitations:
1) it doesn't let you add PodCIDRs dynamically to a cluster and 
2) it doesn't provide as much control as users might like on how Pods receive their IP addresses from.