# Cilium network policies

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: ingress-rule-from-all-endpoints-in-dev-namespace
  namespace: dev
spec:
  endpointSelector:
    matchLabels:
      role: backend
  ingress:
  - fromEndpoints:
    - {}
```

In this policy pods with label `role: backend` will accept all traffic from all endpoints in the `dev` namespace.

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: egress-rule
  namespace: dev
spec:
  endpointSelector:
    matchLabels:
      role: frontend
  egress:
  - toEndpoints:
    - matchLabels:
        role: backend
```

In this policy pods with label `role: frontend` will access all traffic to pods with label `role: backend` in the `dev` namespace.

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: deny-all-traffic
  namespace: dev
spec:
  endpointSelector:
    matchLabels:
      role: frontend
  ingress:
  - {}
  egress:
  - {}
```

This policy will deny all traffic from all endpoints in the `dev` namespace to pods with label `role: frontend` and vice versa.

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: service-rule
  namespace: dev
spec:
  endpointSelector:
    matchLabels:
      role: frontend
  egress:
  - toServices:
    # Services may be referenced by namespace + name
    - k8sService:
        serviceName: backend
        namespace: dev
    # Services may be referenced by namespace + label selector
    - k8sServiceSelector:
        selector:
          matchLabels:
            app: app1
        namespace: dev
```

In this policy we can allow egress traffic from endpoints which have the label `role: frontend` to services with the label `app: app1` in the `dev` namespace and `backend` service in the `dev` namespace.

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: l4-rule
  namespace: dev
spec:
  endpointSelector:
    matchLabels:
      role: frontend
  egress:
  - toPorts:
    - ports:
      - port: "80"
        protocol: TCP
```

In this case allowing traffic from endpoints which have the label `role: frontend` to ports TCP/80.

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: l3-and-l4-rule
  namespace: dev
spec:
  endpointSelector:
    matchLabels:
      role: backend
  ingress:
  - fromEndpoints:
    - matchLabels:
        role: frontend
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
```

In this policy our endpoints with the label `role: backend` will accept traffic from endpoints with the `role: frontend` through port TCP/80 in the `dev` namespace.

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: l7-rule
  namespace: dev
spec:
  endpointSelector:
    matchLabels:
      role: backend
  ingress:
  - fromEndpoints:
    - matchLabels:
        role: frontend
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/products"
```

In this policy our endpoints with the label `role: backend` will accept traffic from endpoints with the `role: frontend` through port TCP/80 and only `HTTP GET /products` requests in the `dev` namespace.

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: l7-dns-rule
  namespace: dev
spec:
  endpointSelector:
    matchLabels:
      role: backend
  egress:
  - toEndpoints:
    - matchLabels:
        k8s:io.kubernetes.pod.namespace: kube-system
        k8s:k8s-app: kube-dns
    toPorts:
      - ports:
          - port: "53"
            protocol: ANY
        rules:
          dns:
            - matchName: "google.com"
            - matchPattern: "*.google.com"
  - toFQDNs:
      - matchName: "google.com"
      - matchPattern: "*.google.com"
    toPorts:
      - ports:
          - port: "80"
            protocol: TCP
```

The `l7-dns-rule` CiliumNetworkPolicy ensures that pods with the `role: backend` in the `dev` namespace can:
- Perform DNS lookups for `google.com` and its subdomains by allowing UDP/TCP traffic on port 53 to the `kube-dns` pods in the `kube-system` namespace, with L7 DNS filtering applied.
- Make HTTP requests to `google.com` and its subdomains by allowing TCP traffic on port 80 to those specific FQDNs.

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: kube-apiserver-entity-rule
  namespace: dev
spec:
  endpointSelector:
    matchLabels:
      role: backend
  egress:
  - toEntities:
    - kube-apiserver
```

The `kube-apiserver-entity-rule` CiliumNetworkPolicy ensures that pods with the label `role: backend` in the `dev` namespace are explicitly allowed to initiate connections to the Kubernetes API Server.

```yaml
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: allow-dns
spec:
  endpointSelector: {}  # This matches all source pods 
  egress:
  - toEndpoints:
    - matchLabels:
        k8s-app: kube-dns
        io.kubernetes.pod.namespace: kube-system
    toPorts:
    - ports:
      - port: "53"
        protocol: UDP
      - port: "53"
        protocol: TCP
```

It ensures that every single pod in every namespace of the cluster is permitted to send DNS queries (both UDP and TCP) to the `kube-dns` service running in the `kube-system` namespace.