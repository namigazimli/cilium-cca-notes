# L2 Announcements / L2 Aware LB (Beta)
L2 Announcements is a feature which makes services visible and reachable on the local area network. This feature is primarily intended for on-premises deployments within networks without BGP based routing such as office or campus networks. When used, this feature will respond to ARP queries for ExternalIPs and/or LoadBalancer IPs. These IPs are Virtual IPs (not installed on network devices) on multiple nodes, so for each service one node at a time will respond to the ARP queries and respond with its MAC address. This node will perform load balancing with the service load balancing feature, thus acting as a north/south load balancer. The advantage of this feature over NodePort services is that each service can use a unique IP so multiple services can use the same port numbers. When using NodePorts, it is up to the client to decide to which host to send traffic, and if a node goes down, the IP+Port combo becomes unusable. With L2 announcements the service VIP simply migrates to another node and will continue to work.

# Configuration
The L2 Announcements feature and all the requirements can be enabled as follows:
1. Helm
```shell
helm upgrade cilium ./cilium \
   --namespace kube-system \
   --reuse-values \
   --set l2announcements.enabled=true \
   --set k8sClientRateLimit.qps={QPS} \
   --set k8sClientRateLimit.burst={BURST} \
   --set kubeProxyReplacement=true \
   --set k8sServiceHost=${API_SERVER_IP} \
   --set k8sServicePort=${API_SERVER_PORT}
```
2. ConfigMap
```yml
enable-l2-announcements: true
kube-proxy-replacement: true
k8s-client-qps: {QPS}
k8s-client-burst: {BURST}
```

**Warning**: Sizing the client rate limit (`k8sClientRateLimit.qps` and `k8sClientRateLimit.burst`) is important when using this feature due to increased API usage. See Sizing client rate limit for sizing guidelines.

# Prerequisites
- Kube Proxy replacement mode must be enabled. For more information, see Kubernetes Without kube-proxy.
- All devices on which L2 Aware LB will be announced should be enabled and included in the `--devices` flag or `devices` Helm option if explicitly set, see NodePort Devices, Port and Bind settings.
- The `externalIPs.enabled=true` Helm option must be set, if usage of externalIPs is desired. Otherwise service load balancing for external IPs is disabled.

# Limitations
- The feature currently does not support IPv6/NDP.
- Due to the way L3->L2 translation protocols work, one node receives all ARP requests for a specific IP, so no load balancing can happen before traffic hits the cluster.
- The feature currently has no traffic balancing mechanism so nodes within the same policy might be asymmetrically loaded. For details see Leader Election.
- The feature is incompatible with the `externalTrafficPolicy: Local` on services as it may cause service IPs to be announced on nodes without pods causing traffic drops.

# Policies
Policies provide fine-grained control over which services should be announced, where, and how. This is an example policy using all optional fields:
```yml
apiVersion: "cilium.io/v2alpha1"
kind: CiliumL2AnnouncementPolicy
metadata:
  name: policy1
spec:
  serviceSelector:
    matchLabels:
      color: blue
  nodeSelector:
    matchExpressions:
      - key: node-role.kubernetes.io/control-plane
        operator: DoesNotExist
  interfaces:
  - ^eth[0-9]+
  externalIPs: true
  loadBalancerIPs: true
```

# Service Selector
The service selector is a label selector that determines which services are selected by this policy. If no service selector is provided, all services are selected by the policy. A service must have loadBalancerClass unspecified or set to `io.cilium/l2-announcer` to be selected by a policy for announcement. There are a few special purpose selector fields which don’t match on labels but instead on other metadata like `.meta.name` or `.meta.namespace`. 

|   Selector    |   Field   |
|---------------|-----------|
|io.kubernetes.service.namespace|.meta.namespace|
|io.kubernetes.service.name|.meta.name|

# Node Selector
The node selector field is a label selector which determines which nodes are candidates to announce the services from. It might be desirable to pick a subset of nodes in you cluster, since the chosen node (see Leader Election) will act as the north/south load balancer for all of the traffic for a particular service.

# Interfaces
The interfaces field is a list of regular expressions (golang syntax) that determine over which network interfaces the selected services will be announced. This field is optional, if not specified all interfaces will be used. The expressions are OR-ed together, so any network device matching any of the expressions will be matched. L2 announcements only work if the selected devices are also part of the set of devices specified in the `devices` Helm option, see NodePort Devices, Port and Bind settings.

# Leases
The leases are created in the same namespace where Cilium is deployed, typically kube-system. You can inspect the leases with the following command:
```shell
kubectl -n kube-system get lease
NAME                                  HOLDER                                                    AGE
cilium-l2announce-default-deathstar   worker-node                                               2d20h
cilium-operator-resource-lock         worker-node2-tPDVulKoRK                                   2d20h
kube-controller-manager               control-plane-node_9bd97f6c-cd0c-4565-8486-e718deb310e4   2d21h
kube-scheduler                        control-plane-node_2c490643-dd95-4f73-8862-139afe771ffd   2d21h
```
The leases starting with `cilium-l2announce-` are leases used by this feature. The last part of the name is the namespace and service name. The holder indicates the name of the node that currently holds the lease and thus announced the IPs of that given service.

# L2 Pod Announcements
L2 Pod Announcements announce Pod IP addresses on the L2 network using Gratuitous ARP replies. When enabled, the node transmits Gratuitous ARP replies for every locally created pod, on the configured network interface(s). This feature is enabled separately from the above L2 announcements feature. To enable L2 Pod Announcements, set the following:
1. Helm
```shell
helm upgrade cilium ./cilium \
   --namespace kube-system \
   --reuse-values \
   --set l2podAnnouncements.enabled=true \
   --set l2podAnnouncements.interface=eth0
```
2. ConfigMap
```yml
enable-l2-pod-announcements: true
l2-pod-announcements-interface: eth0
```
The `l2podAnnouncements.interface`/`l2-pod-announcements-interfac`e options allows you to specify one interface use to send announcements. If you would like to send announcements on multiple interfaces, you should use the `l2podAnnouncements.interfacePattern`/l`2-pod-announcements-interface-pattern` option instead. This option takes a regex, matching on multiple interfaces.
1. Helm
```shell
helm upgrade cilium ./cilium \
   --namespace kube-system \
   --reuse-values \
   --set l2podAnnouncements.enabled=true \
   --set l2podAnnouncements.interfacePattern='^(eth0|ens1)$'
```
2. ConfigMap
```yml
enable-l2-pod-announcements: true
l2-pod-announcements-interface-pattern: "^(eth0|ens1)$"
```
**Note**: Since this feature has no IPv6 support yet, only ARP messages are sent, no Unsolicited Neighbor Advertisements are sent.

You can check out the following blogs:
- [Complete Guide: Cilium L2 Announcements for LoadBalancer Services in Bare-Metal Kubernetes](https://dev.to/azalio/complete-guide-cilium-l2-announcements-for-loadbalancer-services-in-bare-metal-kubernetes-3jl2)