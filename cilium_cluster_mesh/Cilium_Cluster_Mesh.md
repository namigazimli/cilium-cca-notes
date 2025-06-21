# Multi-Cluster (Cluster Mesh)
Cluster mesh extends the networking datapath across multiple clusters. It allows endpoints in all connected clusters to communicate while providing full policy enforcement. Load-balancing is available via Kubernetes annotations. 

# KVStoreMesh
KVStoreMesh is an extension of Cluster Mesh. It caches the information obtained from the remote clusters in a local kvstore (such as etcd), to which all local Cilium agents connect. This is different from vanilla Cluster Mesh, where each agent directly pulls the information from the remote clusters. KVStoreMesh enables improved scalability and isolation.

# Setting up Cluster Mesh
This is a step-by-step guide on how to build a mesh of Kubernetes clusters by connecting them together, enable pod-to-pod connectivity across all clusters, define global services to load-balance between clusters and enforce security policies to restrict access.

## Prerequisites

**Cluster Addressing Requirements**
- All clusters must be configured with the same datapath mode. Cilium install may default to [Encapsulation](https://docs.cilium.io/en/stable/network/concepts/routing/#encapsulation) or [Native-Routing](https://docs.cilium.io/en/stable/network/concepts/routing/#native-routing) mode depending on the specific cloud environment.
- PodCIDR ranges in all clusters and all nodes must be non-conflicting and unique IP addresses.
- Nodes in all clusters must have IP connectivity between each other using the configured InternalIP for each node. This requirement is typically met by establishing peering or VPN tunnels between the networks of the nodes of each cluster.
- The network between clusters must allow the inter-cluster communication. The exact ports are documented in the [Firewall Rules](https://docs.cilium.io/en/stable/operations/system_requirements/#firewall-requirements) section.

## Additional requirements for Native-routed Datapath modes
- Cilium in each cluster must be configured with a native routing CIDR that covers all the PodCIDR ranges across all connected clusters. Cluster CIDRs are typically allocated from the `10.0.0.0/8` private address space. When this is the case a native routing CIDR such as `10.0.0.0/8` should cover all clusters:
    - ConfigMap option `ipv4-native-routing-cidr=10.0.0.0/8`
    - Helm option `--set ipv4NativeRoutingCIDR=10.0.0.0/8`
    - `cilium install` option `--set ipv4NativeRoutingCIDR=10.0.0.0/8`
- In addition to nodes, pods in all clusters must have IP connectivity between each other. This requirement is typically met by establishing peering or VPN tunnels between the networks of the nodes of each cluster
- The network between clusters must allow pod-to-pod inter-cluster communication across any ports that the pods may use. This is typically accomplished with firewall rules allowing pods in different clusters to reach each other on all ports.

## Scaling Limitations
- By default, the maximum number of clusters that can be connected together using Cluster Mesh is 255. By using the option `maxConnectedClusters` this limit can be set to 511, at the expense of lowering the maximum number of cluster-local identities. Reference the following table for valid configurations and their corresponding cluster-local identity limits:

|   MaxConnectedClusters    |   Maximum cluster-local identities   |
|---------------------------|--------------------------------------|
|   255 (default)           |   65535                              |
|   511                     |   32767                              |

- All clusters across a Cluster Mesh must be configured with the same `maxConnectedClusters` value.
    - ConfigMap option `maxConnectedClusters=511`
    - Helm option `--set maxConnectedClusters=511`
    - `cilium install` option `--set maxConnectedClusters=511`

`MaxConnectedClusters` can only be set once during Cilium installation and should not be changed for existing clusters. Changing this option on a live cluster may result in connection disruption and possible incorrect enforcement of network policies.

## Prepare the Clusters
For the rest of this tutorial, we will assume that you intend to connect two clusters together with the kubectl configuration context stored in the environment variables `$CLUSTER1` and `$CLUSTER2`. This context name is the same as you typically pass to `kubectl --context`.

## Specify the Cluster Name and ID
Cilium needs to be installed onto each cluster. Each cluster must be assigned a unique human-readable name as well as a numeric cluster ID (1-255). The cluster name must respect the following constraints:
- It must contain at most 32 characters;
- It must begin and end with a lower case alphanumeric character;
- It may contain lower case alphanumeric characters and dashes between.

It is best to assign both the cluster name and the cluster ID at installation time:
- ConfigMap options `cluster-name` and `cluster-id`
- Helm options `cluster.name` and `cluster.id`
- Cilium CLI install options `--set cluster.name` and `--set cluster.id`

Example install using the Cilium CLI:
```shell
cilium install --set cluster.name=$CLUSTER1 --set cluster.id=1 --context $CLUSTER1
cilium install --set cluster.name=$CLUSTER2 --set cluster.id=2 --context $CLUSTER2
```

If you change the cluster ID and/or cluster name in a cluster with running workloads, you will need to restart all workloads. The cluster ID is used to generate the security identity and it will need to be re-created in order to establish access across clusters.

## Shared Certificate Authority
If you are planning to run Hubble Relay across clusters, it is best to share a certificate authority (CA) between the clusters as it will enable mTLS across clusters to just work. You can propagate the CA copying the Kubernetes secret containing the CA from one cluster to another:
```shell
kubectl --context=$CLUSTER1 get secret -n kube-system cilium-ca -o yaml | \
  kubectl --context $CLUSTER2 create -f -
```

## Enable Cluster Mesh
Enable all required components by running `cilium clustermesh enable` in the context of both clusters. This will deploy the `clustermesh-apiserver` into the cluster and generate all required certificates and import them as Kubernetes secrets. It will also attempt to auto-detect the best service type for the LoadBalancer to expose the Cluster Mesh control plane to other clusters:
```shell
cilium clustermesh enable --context $CLUSTER1
cilium clustermesh enable --context $CLUSTER2
```

Starting from v1.16 KVStoreMesh is enabled by default. You can opt out of KVStoreMesh when enabling the Cluster Mesh.
```shell
cilium clustermesh enable --context $CLUSTER1 --enable-kvstoremesh=false
cilium clustermesh enable --context $CLUSTER2 --enable-kvstoremesh=false
```

In some cases, the service type cannot be automatically detected and you need to specify it manually. This can be done with the option `--service-type`. The possible values are:
- **LoadBalancer**: A Kubernetes service of type LoadBalancer is used to expose the control plane. This uses a stable LoadBalancer IP and is typically the best option.
- **NodePort**: A Kubernetes service of type NodePort is used to expose the control plane. This requires stable Node IPs. If a node disappears, the Cluster Mesh may have to reconnect to a different node. If all nodes have become unavailable, you may have to re-connect the clusters to extract new node IPs.
- **ClusterIP**: A Kubernetes service of type ClusterIP is used to expose the control plane. This requires the ClusterIPs are routable between clusters.

Wait for the Cluster Mesh components to come up by invoking `cilium clustermesh status --wait`. If you are using a service of type LoadBalancer then this will also wait for the LoadBalancer to be assigned an IP.
```shell
cilium clustermesh status --context $CLUSTER1 --wait
cilium clustermesh status --context $CLUSTER2 --wait
```

## Connect Clusters
Finally, connect the clusters. This step only needs to be done in one direction. The connection will automatically be established in both directions:
```shell
cilium clustermesh connect --context $CLUSTER1 --destination-context $CLUSTER2
```
It may take a bit for the clusters to be connected. You can run `cilium clustermesh status --wait` to wait for the connection to be successful:
```shell
cilium clustermesh status --context $CLUSTER1 --wait
```
If this step does not complete successfully, proceed to the troubleshooting section.

## Test Pod connectivity between clusters
Congratulations, you have successfully connected your clusters together. You can validate the connectivity by running the connectivity test in multi cluster mode:
```shell
cilium connectivity test --context $CLUSTER1 --multi-cluster $CLUSTER2
```

## Troubleshooting
Use the following list of steps to troubleshoot issues with ClusterMesh:
1. Validate that Cilium pods are healthy and ready:
```shell
cilium status --context $CLUSTER1
cilium status --context $CLUSTER2
```
2. Validate that Cluster Mesh is enabled and operational:
```shell
cilium clustermesh status --context $CLUSTER1
cilium clustermesh status --context $CLUSTER2
```