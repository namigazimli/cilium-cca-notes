# Kind Cluster
We are going to be using Kind to set up our Kubernetes cluster, and on top of that Cilium. Let's have a look at its configuration:
```yml
# yq cluster.yaml
kind: Cluster
name: kind
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  podSubnet: "10.1.0.0/16"
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-ip: "10.0.1.2"
            node-labels: "rack=rack0"
  - role: worker
    kubeadmConfigPatches:
      - |
        kind: JoinConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-ip: "10.0.2.2"
            node-labels: "rack=rack0"
  - role: worker
    kubeadmConfigPatches:
      - |
        kind: JoinConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-ip: "10.0.3.2"
            node-labels: "rack=rack1"
  - role: worker
    kubeadmConfigPatches:
      - |
        kind: JoinConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-ip: "10.0.4.2"
            node-labels: "rack=rack1"
containerdConfigPatches:
  - |-
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."localhost:5000"]
      endpoint = ["http://kind-registry:5000"]
```

# Nodes
In the `nodes` section, you can see that the cluster consists of four nodes:
- 1 `control-plane` node running the Kubernetes control plane and etcd
- 3 `worker` nodes to deploy the applications

# Networking
In the networking section of the configuration file, the default CNI has been disabled so the cluster won't have any Pod network when it starts. Instead, Cilium will be deployed to the cluster to provide this functionality. To see if the Kind cluster is installed, verify that the nodes are up and joined:
```shell
kubectl get nodes
```
You should see the four nodes appear, all marked as NotReady. This is normal, since the CNI is disabled, and we will install Cilium later on in this lab. Before we install Cilium on it, we will be using a platform called [containerlab](https://containerlab.dev/) to simulate the networking backbone Cilium will peer with. In this lab, containerlab is also responsible for assigning internal IP to the Kubernetes nodes. Notice that, if you run the following command, no IP addresses has been allocated to the nodes yet:
```shell
kubectl get nodes -o wide
```
That's OK, we will be deploying containerlab in the next task.

# Containerlab
Containerlab is a platform that enables users to deploy virtual networking topologies, based on containers and virtual machines. One of the virtual routing appliances that can be deployed via Containerlab is FRR - a feature-rich open-source networking platform. By the end of the lab, you will have established BGP peering with the FRR virtual devices.

![Alt text](https://play.instruqt.com/assets/tracks/iw3fh5u5td7t/95514a88599eaf8bcdc594061a0a3c39/assets/bgp_overview.png)

# Inspect the network topology
If you're curious, you can check out in details the containerlab topology we are deploying as part of the lab.
```yml
name: bgp-topo
topology:
  kinds:
    linux:
      cmd: bash
  nodes:
    router0:
      kind: linux
      image: frrouting/frr:v8.2.2
      labels:
        app: frr
      exec:
        # NAT everything in here to go outside of the lab
        - iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
        # Loopback IP (IP address of the router itself)
        - ip addr add 10.0.0.0/32 dev lo
        # Terminate rest of the 10.0.0.0/8 in here
        - ip route add blackhole 10.0.0.0/8
        # Boiler plate to make FRR work
        - touch /etc/frr/vtysh.conf
        - sed -i -e 's/bgpd=no/bgpd=yes/g' /etc/frr/daemons
        - /usr/lib/frr/frrinit.sh start
        # FRR configuration
        - >-
          vtysh -c 'conf t' -c 'frr defaults datacenter' -c 'router bgp 65000' -c '  bgp router-id 10.0.0.0' -c '  no bgp ebgp-requires-policy' -c '  neighbor ROUTERS peer-group' -c '  neighbor ROUTERS remote-as external' -c '  neighbor ROUTERS default-originate' -c '  neighbor net0 interface peer-group ROUTERS' -c '  neighbor net1 interface peer-group ROUTERS' -c '  address-family ipv4 unicast' -c '    redistribute connected' -c '  exit-address-family' -c '!'
    tor0:
      kind: linux
      image: frrouting/frr:v8.2.2
      labels:
        app: frr
      exec:
        - ip link del eth0
        - ip addr add 10.0.0.1/32 dev lo
        - ip addr add 10.0.1.1/24 dev net1
        - ip addr add 10.0.2.1/24 dev net2
        - touch /etc/frr/vtysh.conf
        - sed -i -e 's/bgpd=no/bgpd=yes/g' /etc/frr/daemons
        - /usr/lib/frr/frrinit.sh start
        - >-
          vtysh -c 'conf t' -c 'frr defaults datacenter' -c 'router bgp 65010' -c '  bgp router-id 10.0.0.1' -c '  no bgp ebgp-requires-policy' -c '  neighbor ROUTERS peer-group' -c '  neighbor ROUTERS remote-as external' -c '  neighbor SERVERS peer-group' -c '  neighbor SERVERS remote-as internal' -c '  neighbor net0 interface peer-group ROUTERS' -c '  neighbor 10.0.1.2 peer-group SERVERS' -c '  neighbor 10.0.2.2 peer-group SERVERS' -c '  address-family ipv4 unicast' -c '    redistribute connected' -c '  exit-address-family' -c '!'
    tor1:
      kind: linux
      image: frrouting/frr:v8.2.2
      labels:
        app: frr
      exec:
        - ip link del eth0
        - ip addr add 10.0.0.2/32 dev lo
        - ip addr add 10.0.3.1/24 dev net1
        - ip addr add 10.0.4.1/24 dev net2
        - touch /etc/frr/vtysh.conf
        - sed -i -e 's/bgpd=no/bgpd=yes/g' /etc/frr/daemons
        - /usr/lib/frr/frrinit.sh start
        - >-
          vtysh -c 'conf t' -c 'frr defaults datacenter' -c 'router bgp 65011' -c '  bgp router-id 10.0.0.2' -c '  bgp bestpath as-path multipath-relax' -c '  no bgp ebgp-requires-policy' -c '  neighbor ROUTERS peer-group' -c '  neighbor ROUTERS remote-as external' -c '  neighbor SERVERS peer-group' -c '  neighbor SERVERS remote-as internal' -c '  neighbor net0 interface peer-group ROUTERS' -c '  neighbor 10.0.3.2 peer-group SERVERS' -c '  neighbor 10.0.4.2 peer-group SERVERS' -c '  address-family ipv4 unicast' -c '    redistribute connected' -c '  exit-address-family' -c '!'
    srv-control-plane:
      kind: linux
      image: nicolaka/netshoot:latest
      network-mode: container:kind-control-plane
      exec:
        # Cilium currently doesn't support BGP Unnumbered
        - ip addr add 10.0.1.2/24 dev net0
        # Cilium currently doesn't support importing routes
        - ip route replace default via 10.0.1.1
    srv-worker:
      kind: linux
      image: nicolaka/netshoot:latest
      network-mode: container:kind-worker
      exec:
        - ip addr add 10.0.2.2/24 dev net0
        - ip route replace default via 10.0.2.1
    srv-worker2:
      kind: linux
      image: nicolaka/netshoot:latest
      network-mode: container:kind-worker2
      exec:
        - ip addr add 10.0.3.2/24 dev net0
        - ip route replace default via 10.0.3.1
    srv-worker3:
      kind: linux
      image: nicolaka/netshoot:latest
      network-mode: container:kind-worker3
      exec:
        - ip addr add 10.0.4.2/24 dev net0
        - ip route replace default via 10.0.4.1
  links:
    - endpoints: ["router0:net0", "tor0:net0"]
    - endpoints: ["router0:net1", "tor1:net0"]
    - endpoints: ["tor0:net1", "srv-control-plane:net0"]
    - endpoints: ["tor0:net2", "srv-worker:net0"]
    - endpoints: ["tor1:net1", "srv-worker2:net0"]
    - endpoints: ["tor1:net2", "srv-worker3:net0"]
```
The main thing to notice is that we are deploying 3 main routing nodes: a backbone router (router0) and two Top of Rack (ToR) routers (tor0 and tor1). We are pre-configuring them at boot time with their IP and BGP configuration. At the end of the YAML file, you will also note we are establishing virtual links between the backbone and the ToR routers. In the following tasks, we will configure Cilium to run BGP on the kind nodes and to establish BGP peering with the ToR devices. Here is what the overall final topology looks like (note you can resize this window if the diagram is too small):

![Alt text](https://play.instruqt.com/assets/tracks/iw3fh5u5td7t/529cae28190e8dd429add5c2283823a1/assets/BGP-final.jpg)

# Deploy the networking topology
In the terminal, deploy the topology previously described:
```shell
containerlab -t topo.yaml deploy
```
This typically only takes a few seconds to complete.

# Verify our connectivity
At this stage, BGP should be up between our Top of Rack switches and the backbone router router0. Let's verify this with this command.
```shell
docker exec -it clab-bgp-topo-router0 vtysh -c 'show bgp ipv4 summary wide'
```
Let's explain briefly this command.
- `docker exec -it` lets us enter the `router0` shell. As mentioned earlier, `router0` is based on the open-source Free Range Routing platform (FRR).
- `vtysh` is the integrated shell on FRR devices.
- `show bgp ipv4 summary wide` lets us check the BGP status.
Once you run this command, you will an output such as:
```shell
IPv4 Unicast Summary (VRF default):
BGP router identifier 10.0.0.0, local AS number 65000 vrf-id 0
BGP table version 8
RIB entries 15, using 2760 bytes of memory
Peers 2, using 1433 KiB of memory
Peer groups 1, using 64 bytes of memory

Neighbor        V         AS    LocalAS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
tor0(net0)      4      65010      65000        27        26        0    0    0 00:00:54            3        9 N/A
tor1(net1)      4      65011      65000        27        27        0    0    0 00:00:55            3        9 N/A

Total number of neighbors 2
```
If you're familiar with using BGP on traditional CLIs such as Cisco IOS, this will look familiar. If not, let's go through some of the key outputs of the command above. This commands provides information about the BGP status on router0. It shows router0's local AS number (65000), the remote AS number of the routers it is peering with (65010 for tor0 and 65011 for tor1). It also shows, in the Up/Down column where the session is established (if that's the case, it will show for how long the session has been up - in our case, it's been up for 00:01:41). Finally, it shows how many prefixes have been received and sent (see State/PfxRcd and PfxSnt). Let's run this command on the Top of Rack switches. Two of the sessions remain "Active" - it means the peering sessions are configured and actively trying to peer but they are not established yet. It's to be expected: BGP is not established with the Kind nodes as we haven't deployed Cilium yet.
On tor0:
```shell
docker exec -it clab-bgp-topo-tor0 vtysh -c 'show bgp ipv4 summary wide'
```
On tor1:
```shell
docker exec -it clab-bgp-topo-tor1 vtysh -c 'show bgp ipv4 summary wide'
```
In the next step, we will be deploying Cilium on the nodes.

# Cilium CLI
The cilium CLI tool is provided in this environment to install and check the status of Cilium in the cluster. Let's start by installing Cilium on the Kind cluster, with BGP enabled.
```shell
cilium install \
    --version v1.17.1 \
    --set ipam.mode=kubernetes \
    --set routingMode=native \
    --set ipv4NativeRoutingCIDR="10.0.0.0/8" \
    --set bgpControlPlane.enabled=true \
    --set k8s.requireIPv4PodCIDR=true
```
The installation usually takes a couple of minutes. While we wait for the installation to complete, let's review some Cilium BGP aspects:
- As you can see in the Cilium Helm values above, bgpControlPlane is the main requirement to enable BGP on Cilium.
- The configuration for BGP peers and Autonomous System Numbers (ASN) will be configured through a Kubernetes CRD (that's the next task).
For more details on the BGP configuration options, you can read up more on the [official Cilium BGP documentation](https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane/). The installation should now have finished. Let's verify the status of Cilium:
```shell
cilium status --wait
```
Cilium is now functional on our cluster. Let's verify that BGP has been successfully enabled by checking the Cilium configuration:
```shell
cilium config view | grep enable-bgp
```
Next, we are going to deploy our BGP Peering Policies and verify that the BGP sessions are established.

# BGP Configuration
Let's first walk through the BGP Peering configuration. With Cilium's BGP v2 control plane introduced in Cilium 1.16.0, peering policies can be provisioned using a combination of three Kubernetes CRDs:
- `CiliumBGPClusterConfig` sets up the peering endpoints, and can refer to one or multiple `CiliumBGPPeerConfig` resources (using resource names)
- `CiliumBGPPeerConfig` configures how a peering behaves, and can refer to one or multiple
- `CiliumBGPAdvertisement` resources (using selectors)
- `CiliumBGPAdvertisement` specifies which CIDRs should be advertised via BGP.
```yml
# yq cilium-bgp-peering-policies.yaml
---
apiVersion: "cilium.io/v2alpha1"
kind: CiliumBGPClusterConfig
metadata:
  name: rack0
spec:
  nodeSelector:
    matchLabels:
      rack: rack0
  bgpInstances:
    - name: "instance-65010"
      localASN: 65010
      peers:
        - name: "peer-65010-rack0"
          peerASN: 65010
          peerAddress: "10.0.0.1"
          peerConfigRef:
            name: "peer-config-generic"
---
apiVersion: "cilium.io/v2alpha1"
kind: CiliumBGPClusterConfig
metadata:
  name: rack1
spec:
  nodeSelector:
    matchLabels:
      rack: rack1
  bgpInstances:
    - name: "instance-65011"
      localASN: 65011
      peers:
        - name: "peer-65011-rack1"
          peerASN: 65011
          peerAddress: "10.0.0.2"
          peerConfigRef:
            name: "peer-config-generic"
---
apiVersion: "cilium.io/v2alpha1"
kind: CiliumBGPPeerConfig
metadata:
  name: peer-config-generic
spec:
  families:
    - afi: ipv4
      safi: unicast
      advertisements:
        matchLabels:
          advertise: "pod-cidr"
---
apiVersion: "cilium.io/v2alpha1"
kind: CiliumBGPAdvertisement
metadata:
  name: pod-cidr
  labels:
    advertise: pod-cidr
spec:
  advertisements:
    - advertisementType: "PodCIDR"
```
The key aspects of the policy are:
- the remote peer IP address (`peerAddress`) and AS Number (`peerASN`)
- your own local AS Number (`localASN`) And that's it!
In this lab, we specify the loopback IP addresses of our BGP peers: the Top of Rack devices `tor0` (10.0.0.1/32) and `tor1` (10.0.0.2/32). Note that BGP configuration on Cilium is label-based - the Cilium-managed nodes with a matching label will deploy a virtual router for BGP peering purposes. Verify the label configuration with the following commands:
```shell
kubectl get nodes -l 'rack in (rack0,rack1)'
```

# Deploy the BGP Peering Policies CRD configuration
It's time to now deploy the BGP peering policy.
```shell
cilium apply -f cilium-bgp-peering-policies.yaml
```

# Verify successful BGP peering
Now that we have set up our BGP peering, the peering sessions between the Cilium nodes and the Top of Rack switches should be established successfully. Let's verify that the sessions have been established and that routes are learned successfully (it might take a few seconds for the sessions to come up).
On `tor0`:
```shell
docker exec -it clab-bgp-topo-tor0 vtysh -c 'show bgp ipv4 summary wide'
```
On `tor1`:
```shell
docker exec -it clab-bgp-topo-tor1 vtysh -c 'show bgp ipv4 summary wide'
```
This time, you should see that the session between the ToR devices and the Cilium nodes are no longer "Active" (that is to say, unsuccessfully trying to establish peering) but up (you will see how long the session has been up on the `Up/Down` column).

# Deploy our networking utility pods
We will also be deploying a networking utility called netshoot by using a DaemonSet. We will be using it to verify end-to-end connectivity at the end of the lab.
```yml
# yq netshoot-ds.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: netshoot
spec:
  selector:
    matchLabels:
      app: netshoot
  template:
    metadata:
      labels:
        app: netshoot
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: netshoot
        image: nicolaka/netshoot:latest
        command: ["sleep", "infinite"]
```
```shell
kubectl apply -f netshoot-ds.yaml
```
To verify the netshoot pods have been successfully deployed, simply run:
```shell
kubectl rollout status ds/netshoot -w
```
It should take 30 seconds for the Pods to be ready.

# Verify end-to-end connectivity
We will now be running a series of connectivity tests, from a source Pod on a node in rack0 to a destination Pod in rack1. These packets will traverse the our virtual networking backbone and validate that the whole data path is working as expected. Run the following commands. First, let's find the name of a source Pod in rack0.
```shell
SRC_POD=$(kubectl get pods -o wide | grep "kind-worker " | awk '{ print($1); }')
```
Next, let's get the IP address of a destination Pod in rack1.
```shell
DST_IP=$(kubectl get pods -o wide | grep worker3 | awk '{ print($6); }')
```
Finally, let's execute a ping from the source Pod to the destination IP.
```shell
kubectl exec -it $SRC_POD -- ping $DST_IP
```
You should see packets flowing across your virtual data center. Well done: your Kubernetes Pods located in different rack servers in your (virtual) datacenter can communicate together across the network backbone! Great job - you have successfully completed this lab and now understand how you can use BGP on Cilium to easily connect your Kubernetes clusters to your DC network.