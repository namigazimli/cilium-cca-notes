# Cilium Network Policies
Ever since its inception, Cilium has supported Kubernetes Network Policies to enforce traffic control to and from pods at L3/L4. But Cilium Network Policies even go even further: by leveraging eBPF, it can provide greater visibility into packets and enforce traffic policies at L7 and can filter traffic based on criteria such as FQDN, protocol (such as kafka, grpc), etc... Creating and manipulating these Network Policies is done declaratively using YAML manifests. What if we could apply the Kubernetes Network Policy operating model to our hosts? Wouldn't it be nice to have a consistent security model across not just our pods, but also the hosts running the pods? Let's look at how the Cilium Host Firewall can achieve this.
![Alt text](https://play.instruqt.com/assets/tracks/nv1njxicaqaq/9f1c26e7c8cd868a13b5f7be33dfc8d3/assets/host_bastion_anim.gif)

# Kind Cluster
```yaml
# yq /etc/kind/${KIND_CONFIG}.yaml
---
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      # localhost.run proxy
      - containerPort: 32042
        hostPort: 32042
      # Hubble relay
      - containerPort: 31234
        hostPort: 31234
      # Hubble UI
      - containerPort: 31235
        hostPort: 31235
  - role: worker
  - role: worker
networking:
  disableDefaultCNI: true
  kubeProxyMode: none
```

# Nodes
In the `nodes` section, you can see that the cluster consists of three nodes:
- 1 `control-plane` node running the Kubernetes control plane and etcd
- 2 `worker` nodes to deploy the applications

# Networking
In the networking section of the configuration file, the default CNI has been disabled so the cluster won't have any Pod network when it starts. Instead, Cilium is being deployed to the cluster to provide this functionality. To see if the Kind cluster is ready, verify that the cluster is properly running by listing its nodes:
```shell
kubectl get nodes
```
You should see the three nodes appear, all marked as NotReady. This is normal, since the CNI is disabled, and we will install Cilium in the next step. If you don't see all nodes, the workers nodes might still be joining the cluster. Relaunch the command until you can see all three nodes listed.
Now that we have a Kind cluster, let's install Cilium on it!

# The Cilium CLI
The `cilium` CLI tool can install and update Cilium on a cluster, as well as activate features —such as Hubble and Cluster Mesh. In order to use the Cilium Host Firewall, we need to explicitly enable it. We also need to use Kube Proxy Replacement (KPR) mode, as it is a requirement for the Host Firewall feature:
```shell
cilium install \
  --version 1.17.1 \
  --set hostFirewall.enabled=true \
  --set kubeProxyReplacement=true \
  --set bpf.monitorAggregation=none
```

# Activate Hubble
Activate Hubble so we can get observability of the flows:
```shell
cilium hubble enable
```
After a few minutes, Cilium should be installed and Hubble enabled, and you can check that all is running well:
```shell
cilium status --wait
```
Verify that the Host Firewall feature is activated:
```shell
cilium config view | grep host-firewall
```
Now that Cilium is functional, let's install SSH on our nodes!

# SSH access to nodes
In this task, we have installed SSH servers on the nodes (simulated by Kind as Kubernetes nodes and running as Docker containers) and will test accessing the nodes via SSH.
Kind runs nodes as Docker containers, not virtual machines. In order to mimic a real cluster, we have installed and started SSH servers on each node. Let's verify that we can access port 22 on each of the nodes.
```shell
for node in $(docker ps --format '{{.Names}}'); do
  echo "==== Testing connection to node $node ===="
  IP=$(docker inspect $node -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}')
  nc -vz -w2 $IP 22
done
```
Every connection should succeed, showing that we're able to use SSH from the host to any of the nodes in the cluster.

# Viewing flows in Hubble
Look at the logs in Hubble. You should see TCP/22 requests to all nodes being forwarded:
```shell
hubble observe --to-identity 1 --port 22 -f
Jun 22 19:35:24.429: 172.18.0.1:34582 (world) <> 172.18.0.2:22 (host) from-network FORWARDED (TCP Flags: SYN)
Jun 22 19:35:24.429: 172.18.0.1:34582 (world) <> 172.18.0.2:22 (host) from-network FORWARDED (TCP Flags: ACK)
Jun 22 19:35:24.429: 172.18.0.1:34582 (world) <> 172.18.0.2:22 (host) from-network FORWARDED (TCP Flags: ACK, FIN)
Jun 22 19:35:24.441: 172.18.0.1:34582 (world) <> 172.18.0.2:22 (host) from-network FORWARDED (TCP Flags: RST)
Jun 22 19:35:24.451: 172.18.0.1:53928 (world) <> 172.18.0.4:22 (host) from-network FORWARDED (TCP Flags: SYN)
Jun 22 19:35:24.451: 172.18.0.1:53928 (world) <> 172.18.0.4:22 (host) from-network FORWARDED (TCP Flags: ACK)
Jun 22 19:35:24.451: 172.18.0.1:53928 (world) <> 172.18.0.4:22 (host) from-network FORWARDED (TCP Flags: ACK, FIN)
```
Now that we have SSH set up, let's start filtering traffic!

# Host Network Policies
Now that we've checked that we could access the nodes via SSH, let's deploy Network Policies to enforce traffic requirements on our nodes. We will look at how the policies are enforced and the role of identities in network policies. In this lab, we want to ensure that only the Control Plane node can be accessed from outside the cluster using SSH.
![Alt text](https://play.instruqt.com/assets/tracks/nv1njxicaqaq/a7a7f247efb902dbda8fc1ecf1c044f7/assets/host_bastion_1.png)
Within the cluster, all SSH connectivity will be allowed, making it possible to use the Control Plane node as a bastion host.
![Alt text](https://play.instruqt.com/assets/tracks/nv1njxicaqaq/ca22f5dccac309b5ce6636c9d1315df5/assets/host_bastion_2.png)

# The Host identity
Let's check the current status of policy enforcement for the nodes. In order to achieve this, we need to identify the Cilium pod running on a given node. For example, for the control plane node:
```shell
kubectl get pods -n kube-system -l k8s-app=cilium
```
This should list three Cilium nodes. Let's select the first one of them, and exec into it to list the endpoints known on that node:
```shell
kubectl exec -it -n kube-system cilium-g6h4z -- cilium endpoint list
```
Notice the line with label `reserved:host` and an identity of `1`. It's a special reserved identity for the local host. This information can be used for example to filter Hubble output using the `--identity 1` parameter.

# Crafting a Network Policy
In order to secure the access to our nodes, we want to restrict SSH access to them. We'll use the following rules to secure the nodes:
- for SSH (`tcp/22`), we'll use the Control Plane node as a bastion to access other nodes
- we still need to access the Kubernetes API server (`tcp/6443`) on the control plane
- the nodes need to be able to talk to each other on VXLAN (`udp/8472`)
Everything else should be denied by default. In order to implement these rules, we will use `CiliumClusterwideNetworkPolicy` (or `ccnp`) resources. This type of Network Policies applies globally to the whole cluster instead of being restricted to a single namespace like standard standard Network Policy resources.

# API server access
Let's start with API server access. Let's craft a `CiliumClusterwideNetworkPolicy` targeting the Control Plane node, and allowing ingress on ports 6443 for protocol TCP. Inspect the Control Plane node's labels with:
```shell
kubectl get no kind-control-plane -o yaml | yq .metadata.labels
beta.kubernetes.io/arch: amd64
beta.kubernetes.io/os: linux
kubernetes.io/arch: amd64
kubernetes.io/hostname: kind-control-plane
kubernetes.io/os: linux
node-role.kubernetes.io/control-plane: ""
node.kubernetes.io/exclude-from-external-load-balancers: ""
```
The node has a distinct label `node-role.kubernetes.io/control-plane` with an empty value. We will use this to target the node in the policy:
```yaml
---
apiVersion: "cilium.io/v2"
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: "control-plane-apiserver"
spec:
  description: "Allow Kubernetes API Server to Control Plane"
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/control-plane: ""
  ingress:
  - toPorts:
    - ports:
      - port: "6443"
        protocol: TCP
```
Copy and paste this YAML manifest into ccnp-control-plane-apiserver.yaml and apply it:
```shell
kubectl apply -f ccnp-control-plane-apiserver.yaml
```

# Default-deny rule
Now we can craft a default-deny rule, knowing we won't block our access to the API Server. Save the following content to ccnp-default-deny.yaml:
```yaml
---
apiVersion: "cilium.io/v2"
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: "default-deny"
spec:
  description: "Block all unknown traffic to nodes"
  nodeSelector: {}
  ingress:
  - fromEntities:
    - cluster
```
Using the `fromEntities: ['cluster']` filter only allows global traffic if it comes from within the cluster. As a result, this rule effectively blocks all traffic to nodes, unless, unless it comes from within the cluster. Apply it:
```shell
kubectl apply -f ccnp-default-deny.yaml
```
Verify that you can still access the API Server by listing the `CiliumClusterwideNetworkPolicy` resources:
```shell
kubectl get ccnp
```

In the next terminal, start observing Hubble logs:
```shell
hubble observe --identity 1 --port 22 -f
```

# SSH Access
Let's see if we try to access the nodes via SSH now. Switch to the first terminal tab and attempt SSH connections to all nodes:
```shell
for node in $(docker ps --format '{{.Names}}'); do
  echo "==== Testing connection to node $node ===="
  IP=$(docker inspect $node -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}')
  nc -vz -w2 $IP 22
done
```
We get timeouts for all connections, and you can see the dropped packets in the second terminal tab:
```shell
Jun 22 19:49:41.421: 172.18.0.1:46366 (world) <> 172.18.0.2:22 (host) from-network FORWARDED (TCP Flags: SYN)
Jun 22 19:49:41.421: 172.18.0.1:46366 (world) <> 172.18.0.2:22 (host) policy-verdict:none INGRESS DENIED (TCP Flags: SYN)
Jun 22 19:49:41.421: 172.18.0.1:46366 (world) <> 172.18.0.2:22 (host) Policy denied DROPPED (TCP Flags: SYN)
Jun 22 19:49:42.448: 172.18.0.1:46366 (world) <> 172.18.0.2:22 (host) from-network FORWARDED (TCP Flags: SYN)
Jun 22 19:49:42.448: 172.18.0.1:46366 (world) <> 172.18.0.2:22 (host) policy-verdict:none INGRESS DENIED (TCP Flags: SYN)
```
We see packets being dropped to TCP/22 on all 3 nodes. Let us now implement the Network Policy to use the Control Plane node as a bastion host. Create a new CiliumClusterwideNetworkPolicy to allow SSH connections to the Control Plane node. Copy the following content into ccnp-control-plane-ssh.yaml and apply it:
```yaml
---
apiVersion: "cilium.io/v2"
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: "ssh"
spec:
  description: "SSH access on Control Plane"
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/control-plane: ""
  ingress:
  - toPorts:
    - ports:
      - port: "22"
        protocol: TCP
```
```shell
kubectl apply -f ccnp-control-plane-ssh.yaml
```
Now test the SSH connections again:
```shell
for node in $(docker ps --format '{{.Names}}'); do
  echo "==== Testing connection to node $node ===="
  IP=$(docker inspect $node -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}')
  nc -vz -w2 $IP 22
done
```
It should only work on one node: the control plane. Check the logs again in the second terminal tab to verify that packets are forwarded to the control plane, but blocked to other nodes.
```shell
Jun 22 19:52:22.320: 172.18.0.1:46764 (world) <> 172.18.0.4:22 (host) Policy denied DROPPED (TCP Flags: SYN)
Jun 22 19:52:23.331: 172.18.0.1:41094 (world) <> 172.18.0.3:22 (host) from-network FORWARDED (TCP Flags: SYN)
Jun 22 19:52:23.331: 172.18.0.1:41094 (world) -> 172.18.0.3:22 (host) policy-verdict:L4-Only INGRESS ALLOWED (TCP Flags: SYN)
```

# Access from the bastion
The last part to check is that nodes can still be access via SSH through the bastion host. In the first terminal, start a shell on the control plane host:
```shell
docker exec -ti kind-control-plane bash
```
Then test the SSH port access to all nodes:
```shell
for node in $(kubectl get node -o name); do
  echo "==== Testing connection to node $node ===="
  IP=$(kubectl get $node -o jsonpath='{.status.addresses[0].address}');
  nc -vz -w2 $IP 22;
done
```
All connections should succeed, since all traffic is allowed within the cluster. Congratulations, you have now successfully secured SSH access to the nodes, using the Control Plane node as a bastion host!