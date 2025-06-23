Migrating to Cilium from another CNI is a very common task. But how do we minimize the impact during the migration? How do we ensure pods on the legacy CNI can still communicate to Cilium-managed pods during the migration? How do we execute the migration safely, while avoiding a overly complex approach or using a separate tool such as Multus?
With the use of the new Cilium CRD CiliumNodeConfig, running clusters can be migrated on a node-by-node basis, without disrupting existing traffic or requiring a complete cluster outage or rebuild. 

# The Kind cluster

The cluster has been deployed in the background, with Calico installed on it. 
```yaml
---
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      # nodepinger
      - containerPort: 32042
        hostPort: 32042
      # goldpinger
      - containerPort: 32043
        hostPort: 32043
  - role: worker
  - role: worker
  - role: worker
  - role: worker
networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16
```

In the `nodes` section, you can see that the cluster consists of five nodes:
- 1 `control-plane` node running the Kubernetes control plane and etcd
- 4 `worker` nodes to deploy the applications

In the `networking` section, of the configuration file, the default CNI had been disabled. Instead, Calico was deployed to the cluster to provide network connectivity. To check that the Kind cluster was successfully installed, verify that the nodes are up and joined:
```bash
kubectl get nodes
```
You should see 1 control-plane and 4 nodes appear, all marked as `Ready`. Check that Calico was successfully deployed by looking at the status of the Calico DaemonSet:
```bash
kubectl -n calico-system rollout status ds/calico-node
```
Let's check the PodCIDR on each node. (Note: the PodCIDR is the range the Pods will get an IP address from).
```bash
kubectl get ipamblocks.crd.projectcalico.org \
  -o jsonpath="{range .items[*]}{'podNetwork: '}{.spec.cidr}{'\t NodeIP: '}{.spec.affinity}{'\n'}{end}"
```

# Goldpinger

We have deployed a software called Goldpinger on the cluster. Goldpinger deploys one pod per node (with a DaemonSet) to monitor node connectivity. Check that the Nodepinger Daemonset is running properly:
```bash
kubectl rollout status daemonset nodepinger-goldpinger
kubectl get po -l app.kubernetes.io/instance=nodepinger -o wide
```

The first Goldpinger deployment (which we called "Nodepinger") is practical, but since it uses DaemonSets, these pods won't get deleted when we drain nodes for the migration. To show the difference, we'll also deploy Goldpinger a second time as a Deployment with 10 replicas:
```bash
kubectl apply -f /tmp/goldpinger_deploy.yaml
```
Check that the pods are running and spread on all 5 nodes:
```bash
kubectl rollout status deployment goldpinger
kubectl get po -l app=goldpinger -o wide
```
Then expose the pods as a service on NodePort 32043:
```bash
kubectl expose deployment goldpinger --type NodePort \
  --overrides '{"spec":{"ports": [{"port":80,"protocol":"TCP","targetPort":8080,"nodePort":32043}]}}'
```

```bash
curl -s http://localhost:32043/metrics | grep '^goldpinger_nodes_health_total'
```
You should see 10 healthy and 0 unhealthy pods (if it doesn't, try again).

# Migration Approaches
1. The ideal scenario is to build a brand new cluster and migrate workloads using a GitOps approach. This can however involve a lot of preparation work and potential disruptions.
2. Another method consists in reconfiguring /etc/cni/net.d/ to point to Cilium. However, existing Pods will still have been configured by the old network plugin while new Pods will be configured by the newer CNI plugin. To complete the migration, all Pods on the cluster that are configured by the old CNI must be recycled in order to be managed by the new CNI plugin.
3. A native approach to migrating a CNI would be to reconfigure all nodes with a new CNI and then gradually restart each node in the cluster, thus replacing the CNI when the node is brought back up and ensuring that all pods are part of the new CNI. This simple migration, while effective, comes at the cost of disrupting cluster connectivity during the rollout. Unmigrated and migrated nodes would be split in to two “islands” of connectivity, and pods would be randomly unable to reach one-another until the migration is complete.

# Migration via dual overlays
Cilium supports a hybrid mode, where two separate overlays are established across the cluster. While Pods on a given node can only be attached to one network, they have access to both Cilium and non-Cilium pods while the migration is taking place. As long as Cilium and the existing networking provider use a separate IP range, the Linux routing table takes care of separating traffic. 

We will use a model for live migrating between two deployed CNI implementations. This will have the benefit of reducing downtime of nodes and workloads and ensuring that workloads on both configured CNIs can communicate during migration. For live migration to work, Cilium will be installed with a separate CIDR range and encapsulation port than that of the currently installed CNI.

# Migration overview
The migration process utilizes the [per-node configuration feature](https://docs.cilium.io/en/latest/configuration/per-node-config/#per-node-configuration) to selectively enable Cilium CNI. This allows for a controlled rollout of Cilium without disrupting existing workloads. Cilium will be installed, first, in a mode where it establishes an overlay but does not provide CNI networking for any pods. Then, individual nodes will be migrated. In summary, the process looks like:
1. Prepare the cluster and install Cilium in “secondary” mode.
2. Cordon, drain, migrate, and reboot each node
3. Remove the existing network provider
4. (Optional) Reboot each node again

## Preparation: Pod CIDR
The first step is to select a new CIDR for pods. It must be distinct from all other CIDRs in use. Let's check the CIDR used by Calico:
```bash
kubectl get installations.operator.tigera.io default \
  -o jsonpath='{.spec.calicoNetwork.ipPools[*].cidr}{"\n"}'
```
Calico is using its default CIDR, which is `192.168.0.0/16`. In order to avoid conflicts, we will use `10.244.0.0/16` — which is the usual default on Kind — as the pod CIDR for Cilium.

## Preparation: Encapsulation Protocol
The second step is to select a different encapsulation protocol (Geneve instead of VXLAN for example) or a distinct encapsulation port. Check which encapsulation protocol Calico is using:
```bash
kubectl get installations.operator.tigera.io default \
  -o jsonpath='{.spec.calicoNetwork.ipPools[*].encapsulation}{"\n"}'
```
Calico is using `VXLANCrossSubnet`. In order to avoid clashing with Calico's VXLAN port, we need to use VXLAN with a non-default port. Since the standard port is `8472`, let's use `8473` instead.

## Cilium values for migration
We have pre-created a Cilium Helm configuration file values-migration.yaml based on the details above:
```yaml
---
operator:
  unmanagedPodWatcher:
    restart: false
tunnelPort: 8473
cni:
  customConf: true
  uninstall: false
ipam:
  mode: "cluster-pool"
  operator:
    clusterPoolIPv4PodCIDRList: ["10.244.0.0/16"]
policyEnforcementMode: "never"
bpf:
  hostLegacyRouting: true
```

Let's review some of the key parameters first:
- Operator Configuration:
```yaml
operator:
  unmanagedPodWatcher:
    restart: false
```
This is there to prevent the Cilium Operator from restarting Pods that are not being managed by Cilium (we don't want to disrupt the pods that are not managed by Cilium).
- VXLAN Configuration:
```yaml
tunnelPort: 8473
```
As highlighted earlier, this is to prevent Cilium's VXLAN from conflicting with Calico's.
- CNI Configuration:
```yaml
cni:
  customConf: true
  uninstall: false
```
The first setting (`customConf: true`) above temporarily skips writing the CNI configuration. This is to prevent Cilium from taking over immediately. Once the migration is complete, we will switch `customConf` back to the default `false` value. The second setting (`uninstall: false`) above will prevent Cilium from removing the CNI configuration file and plugin binaries for Calico, thus allowing the temporary migration state.
- IPAM Configuration:
```yaml
ipam:
  mode: "cluster-pool"
  operator:
    clusterPoolIPv4PodCIDRList: ["10.244.0.0/16"]
```
Live migration requires to use the Cluster Pool IPAM mode, and a Pod CIDR distinct from Calico's in order to perform the migration.
- Network Policies Configuration
```yaml
policyEnforcementMode: "never"
```
The above disables the enforcement of network policy until the migration is completed. We will enforce network policies post-migration.
- BPF Configuration
```yaml
bpf:
  hostLegacyRouting: true
```
This flag routes traffic via the host stack to provide connectivity during the migration. We will verify during the migration that Calico-managed pods and Cilium-managed pods have connectivity.

## Generate Cilium Helm values
In this lab, we will use cilium-cli to auto-detect settings specific to the underlying cluster platform (Kind in this case) and generate Helm values. We will then use the Helm chart to install Cilium. Let's generate the values-migration.yaml Helm values file:
```bash
cilium install \
  --helm-values values-migration.yaml \
  --dry-run-helm-values > values-initial.yaml
```
The above command:
1. uses Helm values from `values-migration.yaml`
2. automatically fills in the missing values through the use of the `helm-auto-gen-values` flag
3. creates a new Helm values file called `values-initial.yaml`

Review the created file:
```yaml
bpf:
  hostLegacyRouting: true
cluster:
  name: kind-kind
cni:
  customConf: true
  uninstall: false
ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList:
      - 10.244.0.0/16
operator:
  replicas: 1
  unmanagedPodWatcher:
    restart: false
policyEnforcementMode: never
routingMode: tunnel
tunnelPort: 8473
tunnelProtocol: vxlan
```
It is a combination of the values pulled from the `values-migration.yaml` file and the one auto-generated by the Cilium CLI, such as:
- `operator.replicas`
- `cluster.id` and `cluster.name`
- `kubeProxyReplacement`
- the `serviceAccounts` section
- `encryption.nodeEncryption`

## Prevent Calico from using Cilium interfaces
Calico has several modes of operation to decide which network interface to use by default on nodes that it manages. This is configured using the Tigera Operator's `spec.calicoNetwork.nodeAddressAutodetectionV4` (and respectively `nodeAddressAutodetectionV6` for IPv6) parameter. By default, it is set to `firstFound: true`, which will use the first detected network interface on the node. Check this value with:
```bash
kubectl get installations.operator.tigera.io default \
  -o jsonpath='{.spec.calicoNetwork.nodeAddressAutodetectionV4}{"\n"}'
```
When installing Cilium on the nodes, Cilium will create a new network interface called `cilium_host`. If Calico decides to use it as its default interface, Calico node routing will start failing. For this reason, we want to make sure that Calico ignores the `cilium_host` interface. Depending on your Tigera Operator settings (for example if you use the `interface` or `skipInterface` options), you might want to adjust the parameters to ensure `cilium_host` is not considered. In our lab, we will simply set `firstFound` to `false` and use `kubernetes: NodeInternalIP` instead, so Calico uses the node's internal IP as its main interface. Patch the Tigera Operator's configuration with:
```shell
kubectl patch installations.operator.tigera.io default --type=merge \
  --patch '{"spec": {"calicoNetwork": {"nodeAddressAutodetectionV4": {"firstFound": false, "kubernetes": "NodeInternalIP"}}}}'
```
And check the value again:
```shell
kubectl get installations.operator.tigera.io default \
  -o jsonpath='{.spec.calicoNetwork.nodeAddressAutodetectionV4}{"\n"}'
```

## Install Cilium
Let's now install Cilium using helm and the values we have just generated.
```shell
helm repo add cilium https://helm.cilium.io/
helm upgrade --install --namespace kube-system cilium cilium/cilium \
  --values values-initial.yaml
```
Verify that Cilium is properly installed (this might take a few minutes):
```bash
cilium status --wait
```
Note that the Cluster Pods entry indicates that no pods are managed by Cilium. While Cilium is installed on every node and an overlay is established between the nodes, it is not yet configured to manage pods on nodes. Check the CNI configuration on one of the nodes:
```shell
docker exec kind-worker ls /etc/cni/net.d/
10-calico.conflist
calico-kubeconfig
```
It still only contains Calico configuration, so Kubelets are unable to make use of Cilium for now.

## CiliumNodeConfig
In a standard installation, all Cilium agents have the same configuration, controlled by the cilium-config ConfigMap resource. However in our migration, we want to rollout Cilium on one node at a time. In order to achieve this, we will use the CiliumNodeConfig resource type (a CRD that was added in Cilium 1.13), which allows to configure Cilium agents on a per-node basis. A CiliumNodeConfig object consists of a set of fields and a label selector. The label selector defines to which nodes the configuration applies. Let's now create a per-node config that will instruct Cilium to “take over” CNI networking on the node.
```yaml
# ciliumnodeconfig.yaml
apiVersion: cilium.io/v2
kind: CiliumNodeConfig
metadata:
  namespace: kube-system
  name: cilium-default
spec:
  nodeSelector:
    matchLabels:
      io.cilium.migration/cilium-default: "true"
  defaults:
    write-cni-conf-when-ready: /host/etc/cni/net.d/05-cilium.conflist
    custom-cni-conf: "false"
    cni-chaining-mode: "none"
    cni-exclusive: "true"
```
This configuration will be used to switch the nodes' CNI configuration from Calico to Cilium. When the file is saved, apply it:
```bash
kubectl apply --server-side -f ciliumnodeconfig.yaml
```
Check the list of nodes on the cluster, with their labels:
```shell
kubectl get nodes --show-labels
```
There is currently no nodes matching the `io.cilium.migration/cilium-default: "true"` condition in our `CiliumNodeConfig` resource. For this reason, this configuration does not apply to any nodes for now. In the next challenge, we will start migrating nodes by applying this label to them, one by one.

## Cordon and drain the node
It is recommended to always cordon and drain at the beginning of the migration process, so that end-users are not impacted by any potential issues. **Cordoning** a node will prevent new pods from being scheduled on it. **Draining** a node will gracefully evict all the running pods from the node. This ensures that the pods are not abruptly terminated and that their workload is gracefully handled by other available nodes.
Let's get started with the kind-worker node:
```shell
NODE="kind-worker"
```
Let's cordon the node:
```shell
kubectl cordon $NODE
```
Expect to see an output such as n`ode/kind-worker cordoned`. Let's now drain the node. Note that we use the `ignore-daemonset` flag as several DaemonSets are still required to run. When we drain a node, the node is automatically cordoned. We cordoned first in this instance to provide clarity in the migration process, but you don't need to do both steps in the future.
```bash
kubectl drain $NODE --ignore-daemonsets
```
Let's verify no pods are running on the drained node (besides the goldpinger pod which is part of a DaemonSet):
```bash
kubectl get pods -o wide --field-selector spec.nodeName=$NODE
```
Verify that the Nodepinger still sees 5 pods:
```bash
curl -s http://localhost:32042/metrics | grep '^goldpinger_nodes_health_total'
```

## Label and restart
We can now label the node, which will cause the `CiliumNodeConfig` to apply to this node.
```shell
kubectl label node $NODE --overwrite "io.cilium.migration/cilium-default=true"
```
Restart Cilium on the node. That will trigger the creation of CNI configuration file.
```shell
kubectl -n kube-system delete pod --field-selector spec.nodeName=$NODE -l k8s-app=cilium
kubectl -n kube-system rollout status ds/cilium -w
```
Expect to see output such as:
```shell
pod "cilium-cshqf" deleted
Waiting for daemon set "cilium" rollout to finish: 4 of 5 updated pods are available...
daemon set "cilium" successfully rolled out
```
Check the CNI configurations on the node:
```shell
docker exec $NODE ls /etc/cni/net.d/
```
Cilium has renamed the Calico configuration (`10-calico.conflist`) and deployed its own configuration file `05-cilium.conflist`, so the Kubelet on that node is now configured to use Cilium as its CNI provider. Check the Nodepinger pods:
```shell
kubectl get po -l app.kubernetes.io/instance=nodepinger \
  --field-selector spec.nodeName=$NODE -o wide
```
It is still using the Calico pod CIDR, since the pod was not restarted yet. Delete it to force it to restart:
```shell
kubectl delete po -l app.kubernetes.io/instance=nodepinger \
  --field-selector spec.nodeName=$NODE
```
Check it again:
```shell
kubectl get po -l app.kubernetes.io/instance=nodepinger \
  --field-selector spec.nodeName=$NODE -o wide
```
It has now received an IP from the Cilium pod CIDR range (`10.244.0.0/16`). Verify that the connectivity is still fine:
```shell
curl -s http://localhost:32042/metrics | grep '^goldpinger_nodes_health_total'
```

## Verification
Let's check the Cilium status. It may take a minute or so.
```shell
cilium status --wait
```
The `Cluster Pods` section indicates that 1 Pod is managed by Cilium, the goldpinger Pod (since it is part of a DaemonSet). Check the Pod CIDR that was assigned to the node by Cilium:
```shell
kubectl get ciliumnode kind-worker \
  -o jsonpath='{.spec.ipam.podCIDRs[0]}{"\n"}'
```
We can finally uncordon the node with:
```shell
kubectl uncordon $NODE
```
Scale the goldpinger deployment so it spreads on the newly migrated node:
```shell
kubectl scale deployment goldpinger --replicas=15
```
Check the assigned IPs for the new pods:
```shell
kubectl get po -l app=goldpinger --field-selector spec.nodeName=$NODE -o wide
```
Then check that all pings work fine with the previously deployed pods in the deployment (it might take a little while, relaunch the command):
```shell
curl -s http://localhost:32043/metrics | grep '^goldpinger_nodes_health_total'
```

## Repeat for other worker nodes
Let's migrate all the other worker nodes, by looping on all of them:
```shell
for i in $(seq 2 4); do
  node="kind-worker${i}"
  echo "Migrating node ${node}"
  kubectl drain $node --ignore-daemonsets
  kubectl label node $node --overwrite "io.cilium.migration/cilium-default=true"
  kubectl -n kube-system delete pod --field-selector spec.nodeName=$node -l k8s-app=cilium
  kubectl -n kube-system rollout status ds/cilium -w
  kubectl uncordon $node
done
```
Check that the Cilium status is correct:
```shell
cilium status --wait
```
The Nodepinger DaemonSet pods will still be on the old Pod CIDR, so let's restart all of them:
```shell
kubectl rollout restart daemonset nodepinger-goldpinger
kubectl rollout status daemonset nodepinger-goldpinger
```
Now check that all pods on worker nodes have a Cilium IP address:
```shell
kubectl get po -o wide
```
Check that connectivity is still fine for both the DaemonSet and the Deployment Goldpinger apps:
```shell
curl -s http://localhost:32042/metrics | grep '^goldpinger_nodes_health_total'
curl -s http://localhost:32043/metrics | grep '^goldpinger_nodes_health_total'
```
The only node left to migrate is the control plane.

## Repeat for kind-control-plane
We can now proceed to the migration of the final node. Let's cordon the node:
```shell
NODE="kind-control-plane"
kubectl drain $NODE --ignore-daemonsets
kubectl label node $NODE --overwrite "io.cilium.migration/cilium-default=true"
kubectl -n kube-system delete pod --field-selector spec.nodeName=$NODE -l k8s-app=cilium
kubectl -n kube-system rollout status ds/cilium -w
kubectl uncordon $NODE
```
Ensure the Nodepinger pods are all restarted:
```shell
kubectl rollout restart daemonset nodepinger-goldpinger
kubectl rollout status daemonset nodepinger-goldpinger
```
In addition, restart all `csi-node-driver` DaemonSet pods inside the `calico-system` namespace to complete the migration of all existing pods to Cilium:
```shell
kubectl rollout restart daemonset -n calico-system csi-node-driver
kubectl rollout status daemonset -n calico-system csi-node-driver
```
The status of Cilium should be `OK` and all pods should be managed by Cilium:
```shell
cilium status --wait
```
Now the migration of the nodes is complete, let's clean things up!

## Update the Cilium Configuration
Now that Cilium is healthy, let's update the Cilium configuration. First, generate a new Helm values file by overriding some parameters:
```shell
cilium install \
  --helm-values values-initial.yaml \
  --helm-set operator.unmanagedPodWatcher.restart=true \
  --helm-set cni.customConf=false \
  --helm-set policyEnforcementMode=default \
  --dry-run-helm-values > values-final.yaml
```
Again, we are using the `cilium-cli` to generate an updated Helm config file. Check the differences with the previous values:
```shell
diff -u --color values-initial.yaml values-final.yaml
```
As you can see from checking the differences between the two files, we are changing three parameters:
- enabling Cilium to write the CNI configuration file by disabling a per node configuration (`customConf`)
- enabling Cilium to restart unmanaged pods
- enabling Network Policy Enforcement

Let's apply it and rollout the Cilium DaemonSet:
```shell
helm upgrade --install \
  --namespace kube-system cilium cilium/cilium \
  --values values-final.yaml
kubectl -n kube-system rollout restart daemonset cilium
cilium status --wait
```
Remove the CiliumNodeConfig resource:
```shell
kubectl delete -n kube-system ciliumnodeconfig cilium-default
```

## Delete the previous network plugin
Let's remove Calico as it is no longer needed:
```shell
kubectl delete --force -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/tigera-operator.yaml
kubectl delete --force -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/custom-resources.yaml
kubectl delete --force namespace calico-system
```

## Restart the Nodes
Finally, let's restart the nodes one by one:
```shell
for i in " " $(seq 2 4); do
  node="kind-worker${i}"
  echo "Restarting node ${node}"
  docker restart $node
  sleep 5 # wait for cilium to catch that the node is missing
  kubectl -n kube-system rollout status ds/cilium -w
done
```
Let's finish with the control plane node:
```shell
docker restart kind-control-plane
sleep 5
kubectl -n kube-system rollout status ds/cilium -w
```
Check Cilium status:
```shell
cilium status --wait
```
All pods should now be managed by Cilium (see the "Cluster Pods" section). Check that connectivity is still fine for both the DaemonSet and the Deployment Goldpinger apps:
```shell
curl -s http://localhost:32042/metrics | grep '^goldpinger_nodes_health_total'
curl -s http://localhost:32043/metrics | grep '^goldpinger_nodes_health_total'
```
Since we've restarted the only control plane in this cluster (you should have at least 3 in production clusters), the state might be a bit broken for a little while. Check the connectivity:
```shell
cilium connectivity test
```
We've now successfully migrated to Cilium!