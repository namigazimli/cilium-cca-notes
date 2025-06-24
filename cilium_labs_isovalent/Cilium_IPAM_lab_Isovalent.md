# Kind Cluster
We are going to be using Kind to set up our Kubernetes cluster, and on top of that Cilium. The cluster contains 3 nodes, including one control plane and 2 workers. Due to the fact that neither Cilium nor another CNI have yet been installed, they are all in the NotReady state. Check it with:
```shell
kubectl get nodes -o wide
```

# CNI and IPAM
Cilium implements the CNI specification to provide networking to Kubernetes.
![Alt text](https://play.instruqt.com/assets/tracks/xp88idchadn7/e3087cd51f8d72ff9012f2d14d2a8e09/assets/CNI.png)

When a new Pod is added in Kubernetes, it is first assigned to a Node via the Kubernetes Scheduler. The Kubelet running on that Node will then be notified of the new Pod assigned to it and needs to take action to create the containers implementing that Pod. For the networking part, the Kubelet uses CNI. The first step is for the Kubelet to check the CNI configuration on the Node, which is located in `/etc/cni/net.d/`. When Cilium is installed on the cluster, each Cilium agent creates a configuration file in `/etc/cni/net.d/05-cilium`.conflist which instructs Kubelet on how to configure Pod networking. The `/etc/cni/net.d/05-cilium.conflist` file uses a JSON format and looks like this by default:
```json
{
  "cniVersion": "0.3.1",
  "name": "cilium",
  "type": "cilium-cni",
  "enable-debug": false,
  "log-file": "/var/run/cilium/cilium-cni.log"
}
```
This file is currently absent from the nodes because Cilium is not yet installed, and the Kind cluster was created with the `networking.disableDefaultCNI: true` option in order to prevent managed pods from starting before the CNI is created. Check that no CNI configuration is present on the worker node:
```shell
docker exec -ti kind-worker ls /etc/cni/net.d/
```
For this reason, the Kubelet outputs error messages complaining that no CNI plugin is currently configured:
```shell
docker exec -ti kind-worker journalctl -u kubelet -n 5
```
When the CNI configuration file is present, the `name` parameter sets the name of the CNI plugin (`cilium`), while the `type` field instructs the Kubelet on which binary to call when adding a new Pod. The Kubelet expects to find a binary with the type name inside the /opt/cni/bin/ directory on the Node. List the existing types already present on the Node:
```shell
docker exec -ti kind-worker ls /opt/cni/bin/
```
You will see 4 options: `host-local`, `loopback`, `portmap`, and `ptp`. When Cilium is installed, a fifth binary, called `cilium-cni`, will be dropped in that directory. This binary will be used as a wrapper for each Kubelet to communicate with the Cilium Agent on its Node. The Kubelet will then call the `/opt/cni/bin/cilium-cni` binary:
- with an `ADD` command when a new Pod is created, to get the Pod's networking information
- with a `DEL` command when a Pod is deleted, which allows Cilium to clean up information about that Pod
In the next challenge, we will get started with the Kubernetes Host Scope IPAM mode.

# Verify the prerequisites
The Kubernetes Host Scope mode requires the Kube Controller Manager to be started with a CIDR and the `--allocate-node-cidr` flag set to `true`. In this lab, we are using a Kind cluster, which runs the controller manager with that option, so we can use that mode with this distribution of Kubernetes. Inspect the command used to start the Kube Controller Manager and verify that the required flags are passed:
```json
# kubectl -n kube-system get po kube-controller-manager-kind-control-plane -o json \
#  | jq '.spec.containers[].command'
[
  "kube-controller-manager",
  "--allocate-node-cidrs=true",
  "--authentication-kubeconfig=/etc/kubernetes/controller-manager.conf",
  "--authorization-kubeconfig=/etc/kubernetes/controller-manager.conf",
  "--bind-address=127.0.0.1",
  "--client-ca-file=/etc/kubernetes/pki/ca.crt",
  "--cluster-cidr=10.244.0.0/16",
  "--cluster-name=kind",
  "--cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt",
  "--cluster-signing-key-file=/etc/kubernetes/pki/ca.key",
  "--controllers=*,bootstrapsigner,tokencleaner",
  "--enable-hostpath-provisioner=true",
  "--kubeconfig=/etc/kubernetes/controller-manager.conf",
  "--leader-elect=true",
  "--requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt",
  "--root-ca-file=/etc/kubernetes/pki/ca.crt",
  "--service-account-private-key-file=/etc/kubernetes/pki/sa.key",
  "--service-cluster-ip-range=10.96.0.0/16",
  "--use-service-account-credentials=true"
]
```
Look in particular at two lines in the output:
- `--allocate-node-cidrs=true`: the requirement to use the Kubernetes Host Scope IPAM mode in Cilium
- `--cluster-cidr=10.244.0.0/16`: the pod CIDR set to be used for the Cluster
This Kind cluster is thus configured to use a Pod CIDR of `10.244.0.0/16`. The Kube Controller Manager will derive one `/24` subnet from this CIDR for each node in the cluster (which means that up to 256 nodes can be created). In this mode, the Kube Controller Manager is in charge of assigning CIDRs to the nodes. Therefore the pod CIDRs for each node are already in place before Cilium is installed. List each node's pod CIDR with:
```shell
kubectl get node -o jsonpath='{range .items[*]}{.metadata.name} {.spec.podCIDR}{"\n"}{end}' | column -t
```
Note that the first 3 subnets of the cluster's CIDR were allocated to the 3 nodes in our cluster:
- `10.244.0.0/24` for the `kind-control-plane` node
- `10.244.1.0/24` for the `kind-worker` node
- `10.244.2.0/24` for the `kind-worker2` node
The distribution happened sequentially to the Control Plane node and each worker node as this was done at cluster creation time.

# Install Cilium
Let's install Cilium with the Kubernetes Host Scope IPAM option:
```shell
cilium install --version v1.17.1 --set ipam.mode=kubernetes
```
Wait for Cilium to be installed:
```shell
cilium status --wait
```
As mentioned in the previous challenge, installing Cilium creates files on each Node:
- the `/etc/cni/net.d/05-cilium.conflist` CNI configuration file
- the `/opt/cni/bin/cilium-cni` binary to wrap calls to the Cilium Agent
Let's check the CNI configuration on the first worker node:
```shell
docker exec -ti kind-worker cat /etc/cni/net.d/05-cilium.conflist | jq
# {
#   "cniVersion": "0.3.1",
#   "name": "cilium",
#   "plugins": [
#     {
#       "type": "cilium-cni",
#       "enable-debug": false,
#       "log-file": "/var/run/cilium/cilium-cni.log"
#     }
#   ]
# }
```
And the executable cilium-cli plugin:
```shell
docker exec -ti kind-worker ls -l /opt/cni/bin/cilium-cni
```

# IPAM mode
Verify that the IPAM mode is set to kubernetes:
```shell
cilium config view | grep ipam
```
Cilium will use the pod CIDRs associated to each node (using the Node resources) and assign IPs from these subnets to the pods started on the nodes. Executing cilium-dbg status in a Cilium pod will let you get information on the IPAM status for that node. For example, let's retrieve the pod name for the first worker node:
```shell
WORKER_CILIUM_POD=$(kubectl -n kube-system get po -l k8s-app=cilium --field-selector spec.nodeName=kind-worker -o name)
echo $WORKER_CILIUM_POD     # pod/cilium-6855w
```
Then execute cilium-dbg status in it:
```shell
kubectl -n kube-system exec -ti $WORKER_CILIUM_POD -c cilium-agent -- cilium-dbg status | grep IPAM     # IPv4: 2/254 allocated from 10.244.1.0/24
```
You will see that the node is using at least 2 IPv4 IPs:
- one for the node's internal IP (also called router)
- one for the node's health IP
You can see these addresses in the CiliumNode resource. Get Cilium's Internal IP with:
```shell
kubectl get ciliumnode kind-worker -o jsonpath='{.spec.addresses[?(@.type=="CiliumInternalIP")].ip}{"\n"}'  # 10.244.1.164
```
And verify the Cilium Health IP with:
```shell
kubectl get ciliumnode kind-worker -o jsonpath='{.spec.health.ipv4}{"\n"}'      # 10.244.1.158
```

# Starting new Pods
Let's start a new pod called "netshoot" and see how Cilium will assign an IP to it:
```shell
kubectl run netshoot --image nicolaka/netshoot --command "sleep" "infinite"
```
Now find out on which node that pod was created:
```shell
NETSHOOT_NODE=$(kubectl get po netshoot -o jsonpath='{.spec.nodeName}')
echo $NETSHOOT_NODE     # kind-worker2
```
Next, find the Cilium pod for that node:
```shell
NETSHOOT_CILIUM_POD=$(kubectl -n kube-system get po -l k8s-app=cilium --field-selector spec.nodeName=$NETSHOOT_NODE -o name)
echo $NETSHOOT_CILIUM_POD     # pod/cilium-4jknh
```
And check the Cilium IPAM status on that node:
```shell
kubectl -n kube-system exec -ti $NETSHOOT_CILIUM_POD -c cilium-agent -- cilium-dbg status | grep IPAM # IPv4: 3/254 allocated from 10.244.2.0/24
```
The node is now using at least 3 IPs, which we can review using cilium-dbg status --verbose, under the "Allocated addresses" section:
```shell
kubectl -n kube-system exec -ti $NETSHOOT_CILIUM_POD -c cilium-agent -- cilium-dbg status --verbose | grep -A6 "Allocated"
```
This will show you:
- the IP of the new pod (`default/netshoot`)
- the internal IP of the node (`router`)
- the health IP of the node (`health`)
- some nodes might have other entries (for example for CoreDNS pods)
You can verify that each of these IPs is taken from the node's pod CIDR range as listed before.

# Google Kubernetes Engine
The Google Kubernetes Engine IPAM option uses Kubernetes Host Scope under the hood. When a GKE cluster is created, Google will assign a pod CIDR per node, which Cilium can then use to assign pod IPs.

# Conclusion
As you can tell, the Kubernetes Host Scope IPAM mode is quite simple to use, but it can only be used on Kubernetes clusters that are configured with pod CIDRs for each node. Additionally, it is not possible to configure the size of the CIDRs allocated to each node, so this option is not ideal if you need more than 252 pods on your nodes.
In the next challenge, we will see how to use multiple CIDRs to allocate to nodes.

---

# Cluster Scope IPAM

## Install Cilium 
The cluster for the previous challenge has been deleted, and we are now using a brand new Kind cluster. Let's install Cilium with the Cluster Scope IPAM option:
```shell
cilium install --version v1.17.1 --set ipam.mode=cluster-pool
```
Wait for Cilium to be installed:
```shell
cilium status --wait
```
By default, Cilium uses the 10.0.0.0/8 CIDR for Pods. Check that the IPAM option is set to cluster-pool, as well as the used CIDR:
```shell
cilium config view | grep cluster-pool
```
You'll also notice the `cluster-pool-ipv4-mask-size` parameter, which is set to `24`. This means that Cilium will split the cluster CIDR into `/24` subnets and assign one to each node in the cluster by adding it to the `spec.ipam.podCIDRs` field in their respective CiliumNode resources. In this respect, this installation will behave very similarly to the previous challenge using the Kubernetes Host Scope mode.
List the CIDRs associated to each node by looping on each CiliumNode resource:
```shell
kubectl get ciliumnode -o jsonpath='{range .items[*]}{.metadata.name} {.spec.ipam.podCIDRs[]}{"\n"}{end}' | column -t
```
Notice that they are once again the first 3 `/24` subnets derived from the `10.0.0.0/8` CIDR. However, the order of allocation is totally random, and it is very possible that the Control Plane node didn't get the first CIDR (`10.0.0.0/24`). Just as in the previous challenge, you can review the IPAM status of a node using `cilium-dbg status` inside the Cilium agent pod. Let's retrieve the pod name for the first worker node:
```shell
WORKER_CILIUM_POD=$(kubectl -n kube-system get po -l k8s-app=cilium --field-selector spec.nodeName=kind-worker -o name)
echo $WORKER_CILIUM_POD     # pod/cilium-wsrmh
```
Then execute cilium-dbg status in it:
```shell
kubectl -n kube-system exec -ti $WORKER_CILIUM_POD -c cilium-agent -- cilium-dbg status | grep IPAM
```
You will once again see that the node is using at least 2 IPv4 IPs for internal and health interfaces.

## Starting new pods
Let's start a pod called "netshoot":
```shell
kubectl run netshoot --image nicolaka/netshoot --command "sleep" "infinite"
```
Now find out on which node that pod was created:
```shell
NETSHOOT_NODE=$(kubectl get po netshoot -o jsonpath='{.spec.nodeName}')
echo $NETSHOOT_NODE     # kind-worker
```
Now find the Cilium pod for that node:
```shell
NETSHOOT_CILIUM_POD=$(kubectl -n kube-system get po -l k8s-app=cilium --field-selector spec.nodeName=$NETSHOOT_NODE -o name)
echo $NETSHOOT_CILIUM_POD     # pod/cilium-wsrmh
```
And check the Cilium IPAM status on that node:
```shell
kubectl -n kube-system exec -ti $NETSHOOT_CILIUM_POD -c cilium-agent -- cilium-dbg status | grep IPAM
```
The node is now using at least 3 IPs, which we can review using cilium-dbg status --verbose, under the "Allocated addresses" section:
```shell
kubectl -n kube-system exec -ti $NETSHOOT_CILIUM_POD -c cilium-agent -- cilium-dbg status --verbose | grep -A6 "Allocated"
```
This will show you:
- the IP of the new pod (`default/netshoot`) (10.0.0.129)
- the internal IP of the node (`router`)  (10.0.0.230)
- the health IP of the node (`health`)    (10.0.0.85)
- some nodes might have other entries (for example for CoreDNS pods)
You can verify that each of these IPs is taken from the node's pod CIDR range as listed before.
In the next challenge, we will see how you can specify the mask for each Kubernetes node, and allocate multiple CIDRs to the cluster.

---

# Cluster Scope (default) with Multiple CIDRs

## Install Cilium
For this challenge, we're going to tune the IPAM settings for Cilium, using the Cluster Scope mode.
```yml
cat << EOF > values.yaml
ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4MaskSize: 29
    clusterPoolIPv4PodCIDRList:
      - 10.0.42.0/28
      - 10.0.84.0/28
EOF
```
In this setup, we're provisioning two small CIDRs for the cluster, so we can observe what happens when they get exhausted. Since we've specified 29 as the mask size, each node will receive a /29 subnet, which corresponds to 6 usable IPs (8 IPs, minus 2 IPs reserved for Cilium Host and Cilium Health interfaces). Let's install Cilium with these settings:
```shell
cilium install --version v1.17.1 --values values.yaml
```
Verify the Cilium status:
```shell
cilium status --wait
```
And the IPAM options we passed:
```shell
cilium config view | grep cluster-pool
```
Let's list the CIDRs allocated to each node:
```shell
kubectl get ciliumnode -o jsonpath='{range .items[*]}{.metadata.name} {.spec.ipam.podCIDRs[*]}{"\n"}{end}' | column -t
# kind-control-plane  10.0.84.0/29
# kind-worker         10.0.42.0/29
# kind-worker2        10.0.42.8/29
```
Since the CIDRs we passed are /28 and we're allocating one /29 per node, only two nodes can consume IPs from the first CIDR block. For this reason, the third node gets a subnet from the second block, so you should see three subnets configured:
- `10.0.42.0/29`
- `10.0.42.8/29`
- `10.0.84.0/29`
As before, we can verify how many IPs are available based on these settings. Let's retrieve the pod name for the first worker node:
```shell
WORKER_CILIUM_POD=$(kubectl -n kube-system get po -l k8s-app=cilium --field-selector spec.nodeName=kind-worker -o name)
echo $WORKER_CILIUM_POD     # pod/cilium-vkzsr
```
Then execute cilium-dbg status in it:
```shell
kubectl -n kube-system exec -ti $WORKER_CILIUM_POD -c cilium-agent -- cilium-dbg status | grep IPAM
```
You will once again see that the node is using at least 2 IPv4 IPs for internal and health interfaces. Notice that only a few IPs are left available because of the small size of the /29 mask chosen for nodes.

## Starting new pods
Let's create a deployment to deploy pods over the cluster:
```shell
kubectl create deployment netshoot --image nginx --replicas 10
```
Check on which nodes the pods have been deployed:
```shell
kubectl get po -o wide
```
Execute the command multiple times until nothing changes anymore. Notice that some pods did not get an IP address, even though they have been allocated to a worker node. Check the pods that are not starting and on which nodes they are scheduled:
```shell
kubectl get po -o jsonpath='{range .items[?(@..waiting.reason=="ContainerCreating")]}{.metadata.name} {.spec.nodeName}{"\n"}{end}'
```
Let's take the first pod in this list:
```shell
read NO_START_POD NO_START_NODE < <(kubectl get po -o jsonpath='{range .items[?(@..waiting.reason=="ContainerCreating")]}{.metadata.name} {.spec.nodeName}{"\n"}{end}' | head -n1)
echo $NO_START_POD $NO_START_NODE   # netshoot-7b444c4bb9-6pmvv kind-worker
```
Describe the pod to see its logs:
```shell
kubectl describe po $NO_START_POD
```
You will see logs like:
```shell
Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox "16de49a59e5f352b5302ea115fdbe814094b77879e92d7a6d1441e3429d46484": plugin type="cilium-cni" failed (add): unable to allocate IP via local cilium agent: [POST /ipam][502] postIpamFailure "range is full"
```
The kubelet on the node is trying to retrieve IPs from the Cilium CNI plugin to assign to the new pod but failing to do so. The CNI plugin is replying that the range is full. Let's get the Cilium pod for the node:
```shell
NO_START_CILIUM_POD=$(kubectl -n kube-system get po -l k8s-app=cilium --field-selector spec.nodeName=$NO_START_NODE -o name)
echo $NO_START_CILIUM_POD
```
Check the IPAM usage on that node:
```shell
kubectl -n kube-system exec -ti $NO_START_CILIUM_POD -c cilium-agent -- cilium-dbg status | grep IPAM
```
All IPv4 are allocated (6/6), so it's not possible to create new pods on this node anymore. In the Cluster Scope mode, once IPs are exhausted on a node, it is not possible to add more on-the-fly. In the next challenge, we will have a look at the Multi-Pool mode, which allows to dynamically allocate new subnets to nodes when necessary.

---

# Multi pool

## Requirements
Multi-Pool mode is incompatible with tunnel mode, so we'll need to use direct routing instead. In order to keep things simple and avoid a BGP infrastructure, we will use the L2 auto routes functionality with:
```yml
routingMode: native
auto-direct-node-routes: true
ipv4-native-routing-cidr: 10.0.0.0/8
```

## Install Cilium
```yml
cat << EOF > values.yaml
routingMode: native
endpointRoutes:
  enabled: true

autoDirectNodeRoutes: true
ipv4NativeRoutingCIDR: 10.0.0.0/8

ipam:
  mode: multi-pool
  operator:
    autoCreateCiliumPodIPPools:
      default:
        ipv4:
          cidrs: ["10.10.0.0/16"]
          maskSize: 27
bpf:
  masquerade: true
# BPF-based masquerading requires kube-proxy replacement to be enabled
kubeProxyReplacement: "true"
k8sServiceHost: kind-control-plane
k8sServicePort: 6443
# Masquerading should be performed on all host egress interfaces
devices: ["eth+"]
EOF
```
Let's install Cilium with these settings:
```shell
cilium install --version 1.17.1 --values values.yaml
```
Wait for Cilium to be ready:
```shell
cilium status --wait
```
Verify the IPAM options we passed:
```shell
cilium config view | grep multi-pool
```

## Default IP Pool
Look again at the configuration we used to install Cilium, and notice the ipam section:
```yml
ipam:
  mode: multi-pool
  operator:
    autoCreateCiliumPodIPPools:
      default:
        ipv4:
          cidrs: ["10.10.0.0/16"]
          maskSize: 27
```
With these parameters, Cilium will automatically create /27-large pools at startup, based on the `10.10.0.0/16` CIDR. Verify the default CiliumPodIPPool resource that was created by Cilium:
```shell
kubectl get ciliumpodippool default -o yaml
```

## Add a second IP Pool
```yml
cat <<EOF | kubectl apply -f -
apiVersion: cilium.io/v2alpha1
kind: CiliumPodIPPool
metadata:
  name: mars
spec:
  ipv4:
    cidrs:
    - 10.20.0.0/16
    maskSize: 27
EOF
```
Verify that both pools exist:
```shell
kubectl get ciliumpodippools
```

## Starting new pods
One of the advantages of Multi-Pool is that it gives us more control on how we allocate IP addresses to Pods. Let’s create two deployments with two pods each. One will use IP addresses from the default pool while the other one will receive IPs from the `mars` pool:
```yml
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-default
spec:
  selector:
    matchLabels:
      app: nginx-default
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx-default
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.1
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-mars
spec:
  selector:
    matchLabels:
      app: nginx-mars
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx-mars
      annotations:
        ipam.cilium.io/ip-pool: mars
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.1
        ports:
        - containerPort: 80
EOF
```
Notice the `ipam.cilium.io/ip-pool` annotation on the `mars` pods, which will instruct Cilium to get IP addresses from the `mars` IP Pool. Wait for the pods to be running:
```shell
kubectl wait deployment nginx-default --for condition=Available=True
kubectl wait deployment nginx-mars --for condition=Available=True
```
Before we scale and observe what happens, let’s inspect how our initial pools are configured on the worker nodes. Check the kind-worker node:
```shell
kubectl get ciliumnodes kind-worker -o yaml | yq .spec.ipam.pools
```
Then check the kind-worker2 node:
```shell
kubectl get ciliumnodes kind-worker2 -o yaml | yq .spec.ipam.pools
```
Notice that both nodes have been configured with a /27 from each of the pools (default and mars).

## Scaling Up
Our two /27 are only enough for 64 pods so scaling up the mars deployment to 70 pods should cause Cilium to react:
```shell
kubectl scale deployment nginx-mars --replicas=70
```
Wait for the deployment to be ready (it should take up to a minute):
```shell
kubectl wait deployment nginx-mars --for condition=Available=True
```
Let's look at the pools on both nodes again:
```shell
kubectl get ciliumnodes kind-worker kind-worker2 -o yaml | \
  yq -o yaml '.items[] | {(.metadata.name): .spec.ipam.pools}'
```
Note the needed field above: as the number of required IPs significantly increased, another `/27` CIDR from the cluster-wide `mars` pool (`10.20.0.0/16`) was allocated to each Cilium Node. Check all the allocated CIDRs on all nodes:
```shell
kubectl get ciliumnodes kind-worker kind-worker2 -o yaml | \
  yq -o yaml '.items[] | {(.metadata.name): .spec.ipam.pools.allocated}'
```

## Scaling Down
Scale down the mars deployment:
```shell
kubectl scale deployment nginx-mars --replicas 1
```
Now check the pools again:
```shell
kubectl get ciliumnodes kind-worker kind-worker2 -o yaml | \
  yq -o yaml '.items[] | {(.metadata.name): .spec.ipam.pools}'
```
As you can see, Cilium has removed PodCIDRs that are no longer required, since the needed fields for the mars pool are back to 1.

## Conclusion
As we've seen, the Multi Pool mode is more flexible and allows to dynamically grow the number of available IPs on a node on-the-fly and let you assign IP addresses to workloads based on metadata such as annotations. Still, it requires to pass full CIDRs, and 2 IPs from each CIDR will be removed from the range by the allocator. What if we wanted to assign individual IPs to each node? We'll see how to achieve this in the next challenge, using the CRD-based mode.