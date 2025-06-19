# Kubernetes Ingress Support
Cilium uses the standard [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) resource definition, with an `ingressClassName` of `cilium`. This can be used for path-based routing and for TLS termination. For backwards compatibility, the `kubernetes.io/ingress.class` annotation with value of `cilium` is also supported.

Cilium allows you to specify load balancer mode for the Ingress resource:
- `dedicated`: The Ingress controller will create a dedicated loadbalancer for the Ingress.
- `shared`: The Ingress controller will use a shared loadbalancer for all Ingress resources.

Each load balancer mode has its own benefits and drawbacks. The shared mode saves resources by sharing a single LoadBalancer config across all Ingress resources in the cluster, while the dedicated mode can help to avoid potential conflicts (e.g. path prefix) between resources.

# Prerequisites
- Cilium must be configured with NodePort enabled, using `nodePort.enabled=true` or by enabling the kube-proxy replacement with `kubeProxyReplacement=true`. For more information, see [kube-proxy replacement](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/#kubeproxy-free).
- Cilium must be configured with the L7 proxy enabled using `l7Proxy=true` (enabled by default).
- By default, the Ingress controller creates a Service of LoadBalancer type, so your environment will need to support this. Alternatively, you can change this to NodePort or, since Cilium 1.16+, directly expose the Cilium L7 proxy on the [host network](https://docs.cilium.io/en/stable/network/servicemesh/ingress/#gs-ingress-host-network-mode).

# Installation
Cilium Ingress Controller can be enabled with helm flag `ingressController.enabled` set as true. Please refer to [Installation using Helm](https://docs.cilium.io/en/stable/installation/k8s-install-helm/#k8s-install-helm) for a fresh installation.

```bash
helm upgrade cilium cilium/cilium --version 1.17.0 \
    --namespace kube-system \
    --reuse-values \
    --set ingressController.enabled=true \
    --set ingressController.loadbalancerMode=shared
kubectl -n kube-system rollout restart deployment/cilium-operator
kubectl -n kube-system rollout restart ds/cilium
```

Cilium can become the default ingress controller by setting the `--set ingressController.default=true` flag. This will create ingress entries even when the `ingressClass` is not set. If you only want to use envoy traffic management feature without Ingress support, you should only enable `--enable-envoy-config` flag. 

```bash
helm upgrade cilium cilium/cilium --version 1.17.0 \
    --namespace kube-system \
    --reuse-values \
    --set envoyConfig.enabled=true
kubectl -n kube-system rollout restart deployment/cilium-operator
kubectl -n kube-system rollout restart ds/cilium
```

Additionally, the proxy load-balancing feature can be configured with the `loadBalancer.l7.backend=envoy` flag.

```bash
helm upgrade cilium cilium/cilium --version 1.17.0 \
    --namespace kube-system \
    --reuse-values \
    --set loadBalancer.l7.backend=envoy
kubectl -n kube-system rollout restart deployment/cilium-operator
kubectl -n kube-system rollout restart ds/cilium
```