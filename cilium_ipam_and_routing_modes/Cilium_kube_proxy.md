# Kubernetes without kube-proxy
This guide explains how to provision a Kubernetes cluster without kube-proxy, and to use Cilium to fully replace it. For existing installations with kube-proxy running as a DaemonSet, remove it by using the following commands below.
```shell
kubectl -n kube-system delete ds kube-proxy
kubectl -n kube-system delete cm kube-proxy
iptables-save | grep -v KUBE | iptables-restore
```
Setup Helm repository:
```shell
helm repo add cilium https://helm.cilium.io
```
Next, generate the required YAML files and deploy them. Make sure you correctly set your `API_SERVER_IP` and `API_SERVER_PORT` below with the control-plane node IP address and the kube-apiserver port number reported by `kubeadm init` (Kubeadm will use port `6443` by default). Specifying this is necessary as `kubeadm init` is run explicitly without setting up kube-proxy and as a consequence, although it exports `KUBERNETES_SERVICE_HOST` and `KUBERNETES_SERVICE_PORT` with a ClusterIP of the kube-apiserver service to the environment, there is no kube-proxy in our setup provisioning that service. Therefore, the Cilium agent needs to be made aware of this information with the following configuration:
```shell
API_SERVER_IP=<your_api_server_ip>
# Kubeadm default is 6443
API_SERVER_PORT=<your_api_server_port>
helm install cilium cilium/cilium --version 1.17.5 \
    --namespace kube-system \
    --set kubeProxyReplacement=true \
    --set k8sServiceHost=${API_SERVER_IP} \
    --set k8sServicePort=${API_SERVER_PORT}
```
**Note** - Cilium will automatically mount cgroup v2 filesystem required to attach BPF cgroup programs by default at the path `/run/cilium/cgroupv2`. To do that, it needs to mount the host `/proc` inside an init container launched by the DaemonSet temporarily. If you need to disable the auto-mount, specify `--set cgroup.autoMount.enabled=false`, and set the host mount point where cgroup v2 filesystem is already mounted by using `--set cgroup.hostRoot`. For example, if not already mounted, you can mount cgroup v2 filesystem by running the below command on the host, and specify `--set cgroup.hostRoot=/sys/fs/cgroup`.
```shell
mount -t cgroup2 none /sys/fs/cgroup
```
This will install Cilium as a CNI plugin with the eBPF kube-proxy replacement to implement handling of Kubernetes services of type ClusterIP, NodePort, LoadBalancer and services with externalIPs. As well, the eBPF kube-proxy replacement also supports hostPort for containers such that using portmap is not necessary anymore. Note, in above Helm configuration, the `kubeProxyReplacement` has been set to `true` mode. This means that the Cilium agent will bail out in case the underlying Linux kernel support is missing. By default, Helm sets `kubeProxyReplacement=false`, which only enables per-packet in-cluster load-balancing of ClusterIP services. Cilium’s eBPF kube-proxy replacement is supported in direct routing as well as in tunneling mode.

# Client Source IP Preservation
Cilium’s eBPF kube-proxy replacement implements various options to avoid performing SNAT on NodePort requests where the client source IP address would otherwise be lost on its path to the service endpoint. 
- `externalTrafficPolicy=Local`: The `Local` policy is generally supported through the eBPF implementation. In-cluster connectivity for services with `externalTrafficPolicy=Local` is possible and can also be reached from nodes which have no local backends, meaning, given SNAT does not need to be performed, all service endpoints are available for load balancing from in-cluster side.
- `externalTrafficPolicy=Cluster`: For the `Cluster` policy which is the default upon service creation, multiple options exist for achieving client source IP preservation for external traffic, that is, operating the kube-proxy replacement in [DSR](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/#dsr-mode) or [Hybrid](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/#hybrid-mode) mode if only TCP-based services are exposed to the outside world for the latter.

# Internal Traffic Policy
Similar to externalTrafficPolicy described above, Cilium’s eBPF kube-proxy replacement supports internalTrafficPolicy, which translates the above semantics to in-cluster traffic. 
- For services with `internalTrafficPolicy=Local`, traffic originated from pods in the current cluster is routed only to endpoints within the same node the traffic originated from.
- `internalTrafficPolicy=Cluster` is the default, and it doesn’t restrict the endpoints that can handle internal (in-cluster) traffic.
The following table gives an idea of what backends are used to serve connections to a service, depending on the external and internal traffic policies:

| Traffic policy | Service backends used | |
|---|---|---|
| Internal | External | for North-South traffic | for East-West traffic |
| Cluster | Cluster | All (default) | All (default) |
| Cluster | Local | Node-local only | All (default) |
| Local | Cluster | All (default) | Node-local only |
| Local | Local | Node-local only | Node-local only |

