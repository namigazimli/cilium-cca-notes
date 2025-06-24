# Prepare Egress Nodes
In this lab, we will dedicate two nodes in our cluster to be used as egress nodes: kind-worker3 and kind-worker4. They will be used as egress nodes to source NAT traffic.
![Alt text](https://play.instruqt.com/assets/tracks/wm9tp6yqnexf/714c4ce528cc2905b3257d04099f3152/assets/example-walkthrough-architecture.png)

While not technically necessary, we will prevent workload from being scheduled on these nodes, so we can see the traffic going out through egress nodes. In order to ensure that we don't deploy any of our test pods to the egress nodes, let's taint them (any taint key will do, we're choosing `egress-gw` here):
```shell
kubectl taint node kind-worker3 egress-gw:NoSchedule
kubectl taint node kind-worker4 egress-gw:NoSchedule
```
Let's also label the nodes. These labels will be used later on in our Gateway policy:
```shell
kubectl label nodes kind-worker3 egress-gw=true
kubectl label nodes kind-worker4 egress-gw=true
```

# Setup Secondary Network
All the Kind nodes are attached to a Docker network called `kind`, which uses the `172.18.0.0/16` IPv4 CIDR. Verify this:
```shell
docker network inspect -f '{{range.IPAM.Config}}{{.Subnet}}, {{end}}' kind
```
Let's add a new dummy interface called `net0` to both `kind-worker3` and `kind-worker4`, with a new address in the `172.18.0.0/16` network. First, add `172.18.0.42/16` to `kind-worker3`:
```shell
docker exec kind-worker3 ip link add net0 type dummy
docker exec kind-worker3 ip a add 172.18.0.42/16 dev net0
docker exec kind-worker3 ip link set net0 up
```
Next, do the same with `172.18.0.43/16` for `kind-worker4`:
```shell
docker exec kind-worker4 ip link add net0 type dummy
docker exec kind-worker4 ip a add 172.18.0.43/16 dev net0
docker exec kind-worker4 ip link set net0 up
```
These IP addresses will be used as egress IPs by Cilium.

# Install Cilium
Install Cilium to the cluster:
```shell
cilium install \
  --version 1.17.1 \
  --set kubeProxyReplacement=true \
  --set egressGateway.enabled=true \
  --set bpf.masquerade=true \
  --set l7Proxy=false \
  --set devices="{eth+,net+}"
```
Let's explain these flags:
- BPF masquerading and kube-proxy replacement are requirements for the Egress Gateway feature.
- The L7 proxy is incompatible with Egress Gateway so we're disabling it.
- We are attaching two network interfaces to the egress nodes, called `eth0` and `net0`.
Verify that Cilium is running fine:
```shell
cilium status --wait
```
Verify also that Cilium was started with the Egress Gateway feature:
```shell
cilium config view | grep egress-gateway
```
On `worker3` and `worker4`, we expect that Cilium has detected both `eth0` and `net0` interfaces and set them up for masquerading. Verify this on `worker3`:
```shell
CILIUM3_POD=$(kubectl -n kube-system get po -l k8s-app=cilium --field-selector spec.nodeName=kind-worker3 -o name)
kubectl -n kube-system exec -ti $CILIUM3_POD -- cilium status
```
Check the KubeProxyReplacement and Masquerading fields. You can perform the same check on worker4:
```shell
CILIUM4_POD=$(kubectl -n kube-system get po -l k8s-app=cilium --field-selector spec.nodeName=kind-worker4 -o name)
kubectl -n kube-system exec -ti $CILIUM4_POD -- cilium status
```
In the next challenge, you will deploy an application outside of the Kubernetes cluster and try to access it from within the cluster.

# Add the Outpost Server
Let's deploy the outpost application. It needs to be attached to the kind network, and we will pass the allowed source IP addresses as environment variables:
```shell
docker run -d \
  --name remote-outpost \
  --network kind \
  -e ALLOWED_IP=172.18.0.42,172.18.0.43 \
   quay.io/isovalent-dev/egressgw-whatismyip:latest
```
Retrieve the container's IP in a variable:
```shell
OUTPOST=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' remote-outpost)
echo $OUTPOST
```
And test it:
```shell
curl http://$OUTPOST:8000
```
You will get:
```shell
Access denied. Your source IP (172.18.0.1) doesn't match the allowed IPs (172.18.0.42,172.18.0.43)
```
This shows that the outpost service was accessed from the host bridge IP for the `kind` Docker network, which is `172.18.0.1`. The application is refusing to answer because it only accepts requests coming from `172.18.0.42` or `172.18.0.43` as we previously configured.

# Deploy Poller Pods
Let's deploy two starships in the default namespace: an imperial Tie Fighter and a rebel X-Wing. We'll adjust the labels to reflect their loyalty:
```shell
kubectl run tiefighter \
  --labels "org=empire,class=tiefighter" \
  --image docker.io/tgraf/netperf
kubectl run xwing \
  --labels "org=alliance,class=xwing" \
  --image docker.io/tgraf/netperf
```
Now try to reach the outpost container from the Tie Fighter:
```shell
kubectl exec -ti tiefighter -- curl --max-time 2 http://$OUTPOST:8000
```
You will see something like:
```shell
Access denied. Your source IP (172.18.0.4) doesn't match the allowed IPs (172.18.0.42,172.18.0.43)
```
The source IP is the internal IP of the node where the Tie Fighter pod is running. Since we use tunneling (VXLAN), traffic is source NAT'ed with the node's IP address. You will get a similar result with the X-Wing (the source IP might be different if the pod runs on a different node):
```shell
kubectl exec -ti xwing -- curl --max-time 2 http://$OUTPOST:8000
```
In the next challenge, we will add a Gateway Policy to route traffic to the outpost server.

# Add an Egress Gateway Policy
Let's create an Egress Gateway Policy to route traffic from Alliance starships to the kind Docker network (172.18.0.0/16) through an egress nodes. With this policy, traffic coming from pods labeled as org=alliance will be source NAT'ed through one of the two egress nodes (kind-worker3 and kind-worker4), using their extra IP (172.18.0.42 and 172.18.0.43 respectively).

![Alt text](https://play.instruqt.com/assets/tracks/wm9tp6yqnexf/45a82d9bd572d97d0789d55937406e13/assets/complete_diagram.png)

We will select the egress nodes by using their egress-gw label. Cilium will then pick one of the nodes for the policy. Open the Editor tab, copy the following manifest, and paste it in the egress-gw-policy.yaml:
```yml
# egress-gw-policy.yaml
apiVersion: cilium.io/v2
kind: CiliumEgressGatewayPolicy
metadata:
  name: outpost
spec:
  destinationCIDRs:
  - "172.18.0.0/16"
  selectors:
  - podSelector:
      matchLabels:
        org: alliance
  egressGateway:
    nodeSelector:
      matchLabels:
        egress-gw: 'true'
    interface: net0
```
Apply it:
```shell
kubectl apply -f egress-gw-policy.yaml
```
Try to access the outpost server again from the X-Wing pod:
```shell
kubectl exec -ti xwing -- \
  curl --max-time 2 http://172.18.0.7:8000
```
The connection is now accepted, as the traffic exits the cluster through one of the two allowed IP addresses. Now check again with the Tie Fighter:
```shell
kubectl exec -ti tiefighter -- \
  curl --max-time 2 http://172.18.0.7:8000
```
Since the Tie Fighter pod doesn't match the policy's selector, it still accesses the outpost through its node's IP address, which is not valid. To be sure, let's deploy another alliance starship, a Y-Wing:
```shell
kubectl run ywing \
  --labels "org=alliance,class=ywing" \
  --image docker.io/tgraf/netperf
```
Then test access to the outpost:
```shell
kubectl exec -ti ywing -- \
  curl --max-time 2 http://172.18.0.7:8000
```
It works, because the policy uses org=alliance, which matches the Y-Wing pod!

# Egress Gateway HA
The Egress Gateway HA feature uses a different Kubernetes CRD called `IsovalentEgressGatewayPolicy`. Let's remove the previous gateway policy:
```shell
kubectl delete -f egress-gw-policy.yaml
```
Then edit the egress-gw-policy-ha.yaml file in the Editor tab with the following content:
```yml
# egress-gw-policy-ha.yaml
apiVersion: isovalent.com/v1
kind: IsovalentEgressGatewayPolicy
metadata:
  name: outpost-ha
spec:
  destinationCIDRs:
  - "172.18.0.0/16"
  selectors:
  - podSelector:
      matchLabels:
        org: alliance
  egressGroups:
    - nodeSelector:
        matchLabels:
          egress-gw: 'true'
      interface: net0
```
Comparing with the previous policy's spec, you can note that the gateway policy has been moved to an `egressGroups` section, which can be used to assign multiple egress interfaces to a single policy. In the case of this lab, we have kept the same rule, as it matches both `kind-worker3` and `kind-worker4` nodes. If you wanted to be more specific, you could set the node and exit IP address with a section such as:
```yml
egressGroups:
  - nodeSelector:
      matchLabels:
        kubernetes.io/hostname: kind-worker3
    egressIP: 172.18.0.42
  - nodeSelector:
      matchLabels:
        kubernetes.io/hostname: kind-worker4
    egressIP: 172.18.0.43
```
Apply the new policy:
```shell
kubectl apply -f egress-gw-policy-ha.yaml
```
Now test the access again:
```shell
for i in $(seq 1 10); do
  kubectl exec -ti xwing -- \
    curl --max-time 2 http://172.18.0.7:8000
done
```
Traffic is now distributed between both egress nodes, exiting from either `172.18.0.42` or `172.18.0.43`.

# Resilience
Let's remove one of the egress nodes from the pool:
```shell
kubectl label node kind-worker3 egress-gw-
```
Test the access again:
```shell
for i in $(seq 1 10); do
  kubectl exec -ti xwing -- \
    curl --max-time 2 http://172.18.0.7:8000
done
```
Traffic continues to flow through `kind-worker4` with IP `172.18.0.43`. You can set up more egress nodes to increase resilience. Add the label again:
```shell
kubectl label node kind-worker4 egress-gw=true
```

# A New Outpost
If you remember from the beginning of this lab, the current outpost is configured to accept connections from two IP addresses: 172.18.0.42 and 172.18.0.43.

Let's start a new outpost (called remote-outpost-2) that will accept 172.18.0.84 and 172.18.0.85 instead:
```shell
docker run -d \
  --name remote-outpost-2 \
  --network kind \
  -e ALLOWED_IP=172.18.0.84,172.18.0.85 \
   quay.io/isovalent-dev/egressgw-whatismyip:latest
```
Get the IP for this new outpost:
```shell
OUTPOST2=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' remote-outpost-2)
echo $OUTPOST2
```
And let's make requests to it using the current config:
```shell
for i in $(seq 1 10); do
  kubectl exec -ti xwing -- \
    curl --max-time 2 http://$OUTPOST2:8000
done
```
Unsurprisingly, the requests get rejected by the outpost as they come from the currently configured egress IP addresses: `172.18.0.42` and `172.18.0.43`.

# Configure Egress IPAM
In the Editor tab, edit the egress-gw-policy-ha.yaml policy and make two changes:
- remove the `spec.egressGroups.interface` setting (last line).
- add a new entry for `spec.egressCIDRs` as an array of one entry: `172.18.0.84/31`
The file should now look like this:
```yml
# egress-gw-policy-ha.yaml
apiVersion: isovalent.com/v1
kind: IsovalentEgressGatewayPolicy
metadata:
  name: outpost-ha
spec:
  destinationCIDRs:
  - "172.18.0.0/16"
  selectors:
  - podSelector:
      matchLabels:
        org: alliance
  egressGroups:
    - nodeSelector:
        matchLabels:
          egress-gw: 'true'
  egressCIDRs:
    - "172.18.0.84/31"
```
Notice that the egressGroups do not provide a way to select an IP or interface anymore, since the egressCIDRs IPAM setting will be used instead. Apply the new manifest in the Terminal tab:
```shell
kubectl apply -f egress-gw-policy-ha.yaml
```

Let's verify that the IP addresses were properly assigned to the interfaces by running multiple requests to the new outpost:
```shell
for i in $(seq 1 10); do
  kubectl exec -ti xwing -- \
    curl --max-time 2 http://$OUTPOST2:8000
done
```
All the requests now exit through either `172.18.0.84` or `172.18.0.85`!

# Label the Nodes
At this point, we have two gateway nodes in an egressGroups and egress traffic is load-balanced across both of them. In some scenarios, it may be desired to prefer forwarding traffic to some of the gateway nodes in a group, depending on their physical location.

![Alt text](https://play.instruqt.com/assets/tracks/wm9tp6yqnexf/ff7617bfd43cf984c1f9badda86b2238/assets/egress-gw-az.png)

To simulate this scenario, let's split all 4 Kubernetes nodes into two groups, using the well-known Kubernetes topology label. Apply the topology:
```shell
kubectl label node kind-worker topology.kubernetes.io/zone=east
kubectl label node kind-worker3 topology.kubernetes.io/zone=east

kubectl label node kind-worker2 topology.kubernetes.io/zone=west
kubectl label node kind-worker4 topology.kubernetes.io/zone=west
```
Verify the labels:
```shell
kubectl get no --show-labels | \
  grep --color topology.kubernetes.io/zone=
```
Inspect the possible values for azAffinity value in the IsovalentEgressGatewayPolicy CRD:
```shell
kubectl explain isovalentegressgatewaypolicies.spec.azAffinity
```

Edit the `egress-gw-policy-ha.yaml` file in the Editor tab. Add an `azAffinity` parameter to the spec to select local gateways first and fall back to gateways in other zones only once all local gateways become unavailable:
```yaml
azAffinity: localOnlyFirst
```
The result should look like this:
```yml
apiVersion: isovalent.com/v1
kind: IsovalentEgressGatewayPolicy
metadata:
  name: outpost-ha
spec:
  azAffinity: localOnlyFirst
  destinationCIDRs:
  - "172.18.0.0/16"
  selectors:
  - podSelector:
      matchLabels:
        org: alliance
  egressCIDRs:
    - "172.18.0.84/31"
  egressGroups:
    - nodeSelector:
        matchLabels:
          egress-gw: 'true'
```
Deploy the updated policy:
```shell
kubectl apply -f egress-gw-policy-ha.yaml
```

Inspect the resulting Egress Gateway Policy:
```shell
kubectl get isovalentegressgatewaypolicies outpost-ha -o yaml | yq
```
Notice the `status.groupStatuses.activeGatewayIPsByAZ` section, which maps the internal gateway IPs (node internal IPs) to each AZ:
```yml
activeGatewayIPsByAZ:
    east:
      - 172.18.0.3
    west:
      - 172.18.0.2
```
Note that you can also see the dynamic IPs previously assigned by the egress IPAM in the egressIPByGatewayIP section.

# Locate the Pod
Let's find in which zone the X-Wing pod is running. First, identify the node it is running on:
```shell
kubectl get pod xwing -o wide
```
Next, check the zone for that node:
```shell
kubectl get no kind-worker2 --show-labels | \
  grep --color topology.kubernetes.io/zone=
```
Find the egress IP associated with the egress node in that zone:
```shell
docker exec kind-worker4 ip -br add show dev net0
```
Verify that egress traffic from the X-Wing leaves via that local gateway. The result of the following commands should return the IP you just retrieved:
```shell
for i in $(seq 1 10); do
  kubectl exec -ti xwing -- \
    curl --max-time 2 http://172.18.0.8:8000
done
```

# Resilience
Local gateway (or gateways) will continue to be used until it becomes unavailable, in which case all traffic fails over to gateways in other availability zones. This can be simulated by temporarily suspending of the egress gateway node:
```shell
docker pause kind-worker4
```
Test the access again and observe how the egress IP changes:
```shell
for i in $(seq 1 10); do
  kubectl exec -ti xwing -- \
    curl --max-time 2 http://172.18.0.8:8000
done
```
Traffic continues to flow through another availability zone until at least one of the local gateways have recovered:
```shell
docker unpause kind-worker4
```
Test again: traffic should flow again through the local gateway:
```shell
for i in $(seq 1 10); do
  kubectl exec -ti xwing -- \
    curl --max-time 2 http://172.18.0.8:8000
done
```