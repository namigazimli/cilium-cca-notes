# LoadBalancr IP Address Management (LB IPAM)
LB IPAM is a feature that allows Cilium to assign IP addresses to Services of type `LoadBalancer`. This functionality is usually left up to a cloud provider, however, when deploying in a private cloud environment, these facilities are not always available. LB IPAM works in conjunction with features such as [Cilium BGP Control Plane](https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane/#bgp-control-plane) and [L2 Announcements / L2 Aware LB (Beta)](https://docs.cilium.io/en/stable/network/l2-announcements/#l2-announcements). Where LB IPAM is responsible for allocation and assigning of IPs to Service objects and other features are responsible for load balancing and/or advertisement of these IPs. Use Cilium BGP Control Plane to advertise the IP addresses assigned by LB IPAM over BGP and L2 Announcements / L2 Aware LB (Beta) to advertise them locally. LB IPAM is always enabled but dormant. The controller is awoken when the first IP Pool is added to the cluster.

# Pools
B IPAM has the notion of IP Pools which the administrator can create to tell Cilium which IP ranges can be used to allocate IPs from. A basic IP Pools with both an IPv4 and IPv6 range looks like this:
```yml
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: "blue-pool"
spec:
  blocks:
  - cidr: "10.0.10.0/24"
  - cidr: "2004::0/64"
  - start: "20.0.20.100"
    stop: "20.0.20.200"
  - start: "1.2.3.4"
```

# CIDRs, Ranges and reserved IPs
An IP pool can have multiple blocks of IPs. A block can be specified with CIDR notation (`<prefix>/<bits>`) or a range notation with a start and stop IP. When CIDRs are used to specify routable IP ranges, you might not want to allocate the first and the last IP of a CIDR. Typically the first IP is the “network address” and the last IP is the “broadcast address”. In some networks these IPs are not usable and they do not always play well with all network equipment. By default, LB-IPAM uses all IPs in a given CIDR. If you wish to reserve the first and last IPs of CIDRs, you can set the `.spec.allowFirstLastIPs` field to `No`. This option is ignored for /32 and /31 IPv4 CIDRs and /128 and /127 IPv6 CIDRs since these only have 1 or 2 IPs respectively. This setting only applies to blocks specified with `.spec.blocks[].cidr` and not to blocks specified with .spec.`blocks[].start` and `.spec.blocks[].stop`. 

**Warning**: In v1.15, `.spec.allowFirstLastIPs` defaults to `No`. This has changed to `Yes` in v1.16. Please set this field explicitly if you rely on the field being set to `No`.

# Service Selectors
IP Pools have an optional `.spec.serviceSelector` field which allows administrators to limit which services can get IPs from which pools using a label selector. The pool will allocate to any service if no service selector is specified.
```yml
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: "blue-pool"
spec:
  blocks:
  - cidr: "20.0.10.0/24"
  serviceSelector:
    matchExpressions:
      - {key: color, operator: In, values: [blue, cyan]}
---
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: "red-pool"
spec:
  blocks:
  - cidr: "20.0.10.0/24"
  serviceSelector:
    matchLabels:
      color: red
```

![Alt text](https://cdn.sanity.io/images/xinsvxfu/production/bc4c2a5da878e2f2f243712d66961af279b05fd5-1414x820.webp?auto=format&q=80&fit=clip&w=1280)

There are a few special purpose selector fields which don’t match on labels but instead on other metadata like `.meta.name` or `.meta.namespace`.
|   Selector    |   Field   |
|---------------|-----------|
|io.kubernetes.service.namespace|.meta.namespace|
|io.kubernetes.service.name|.meta.name|

For example:
```yml
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: "blue-pool"
spec:
  blocks:
  - cidr: "20.0.10.0/24"
  serviceSelector:
    matchLabels:
      "io.kubernetes.service.namespace": "tenant-a"
```

# Conflicts
IP Pools are not allowed to have overlapping CIDRs. When an administrator does create pools which overlap, a soft error is caused. The last added pool will be marked as `Conflicting` and no further allocation will happen from that pool. Therefore, administrators should always check the status of all pools after making modifications. For example, if we add 2 pools (`blue-pool` and `red-pool`) both with the same CIDR, we will see the following:
```shell
kubectl get ippools
NAME        DISABLED   CONFLICTING   IPS AVAILABLE   AGE
blue-pool   false      False         254             25m
red-pool    false      True          254             11s
```
The reason for the conflict is stated in the status and can be accessed like so
```shell
kubectl get ippools/red-pool -o jsonpath='{.status.conditions[?(@.type=="cilium.io/PoolConflict")].message}'
Pool conflicts since CIDR '20.0.10.0/24' overlaps CIDR '20.0.10.0/24' from IP Pool 'blue-pool'
```

# Disabling a Pool
IP Pools can be disabled. Disabling a pool will stop LB IPAM from allocating new IPs from the pool, but doesn’t remove existing allocations. This allows an administrator to slowly drain pool or reserve a pool for future use.
```yml
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: "blue-pool"
spec:
  blocks:
  - cidr: "20.0.10.0/24"
  disabled: true
```

# LoadBalancerClass
Kubernetes >= v1.24 supports multiple load balancers in the same cluster. Picking between load balancers is done with the `.spec.loadBalancerClass` field. When LB IPAM is enabled it allocates and assigns IPs for services with no load balancer class set. LB IPAM only does IP allocation and doesn’t provide load balancing services by itself. Therefore, users should pick one of the following Cilium load balancer classes, all of which use LB IPAM for allocation (if the feature is enabled):

|   loadBalancerClass       |   Feature                             |
|---------------------------|---------------------------------------|
|io.cilium/bgp-control-plane| Cilium BGP Control Plane              |
|io.cilium/l2-announcer     | L2 Announcements / L2Aware LB (Beta)  |

If the `.spec.loadBalancerClass` is set to a class which isn’t handled by Cilium’s LB IPAM, then Cilium’s LB IPAM will ignore the service entirely, not even setting a condition in the status. By default, if the `.spec.loadBalancerClass` field is not set, Cilium’s LB IPAM will assume it can allocate IPs for the service from its configured pools. If this isn’t the desired behavior, you can configure LB-IPAM to only allocate IPs for services from its configured pools when it has a recognized load balancer class by setting the following configuration in the Helm chart or ConfigMap:
1. Helm:
```shell
helm upgrade cilium cilium/cilium --version 1.17.5 \
   --namespace kube-system \
   --reuse-values \
   --set defaultLBServiceIPAM=none
```
2. ConfigMap:
```yml
default-lb-service-ipam: none
```

# Requesting IPs
Services can request specific IPs. The legacy way of doing so is via `.spec.loadBalancerIP` which takes a single IP address. This method has been deprecated in k8s v1.24 but is supported until its future removal. The new way of requesting specific IPs is to use annotations, `lbipam.cilium.io/ips` in the case of Cilium LB IPAM. This annotation takes a comma-separated list of IP addresses, allowing for multiple IPs to be requested at once. The service selector of the IP Pool still applies, requested IPs will not be allocated or assigned if the services don’t match the pool’s selector. Don’t configure the annotation to request the first or last IP of an IP pool. They are reserved for the network and broadcast addresses respectively.
```yml
apiVersion: v1
kind: Service
metadata:
  name: service-blue
  namespace: example
  labels:
    color: blue
  annotations:
    "lbipam.cilium.io/ips": "20.0.10.100,20.0.10.200"
spec:
  type: LoadBalancer
  ports:
  - port: 1234
```
```shell
kubectl -n example get svc
NAME           TYPE           CLUSTER-IP     EXTERNAL-IP               PORT(S)          AGE
service-blue   LoadBalancer   10.96.26.105   20.0.10.100,20.0.10.200   1234:30363/TCP   43s
```

You can also check out the following blogs: 
- [Migrating from MetalLB to Cilium](https://isovalent.com/blog/post/migrating-from-metallb-to-cilium/)
- [Overcoming Kubernetes IP Address Exhaustion with Cilium](https://isovalent.com/blog/post/overcoming-kubernetes-ip-address-exhaustion-with-cilium/)