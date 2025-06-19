# Task 1
Configure a CiliumNetworkPolicy for the db pod in the `prod` namespace so that pods labeled `role=db` only allow `ingress` from pods labeled `role=backend` in the same namespace.

```yaml
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: db-rule
  namespace: prod
spec:
  endpointSelector:
    matchLabels:
      role: db
  ingress:
  - fromEndpoints:
    - matchLabels:
        role: backend
```
# Task 2
Create a new CiliumNetworkPolicy for the backend pods within the `staging` namespace, permitting `ingress` traffic exclusively from the frontend pods located in either the `staging` or `production` namespace.

```yaml
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: backend-rule
  namespace: staging
spec:
  endpointSelector:
    matchLabels:
      role: backend
  ingress:
  - fromEndpoints:
    - matchLabels:
        role: frontend
      matchExpressions:
      - key: k8s:io.kubernetes.pod.namespace
        operator: In 
        values:
        - staging
        - prod
```

# Task 3
In the prod namespace, restrict `role=frontend` pods so they can only `egress` to pods labeled `role=backend` in the same namespace.

```yaml
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: egress-rule
  namespace: prod 
spec:
  endpointSelector:
    matchLabels:
      role: frontend
  egress:
  - toEndpoints:
    - matchLabels:
        role: backend
```

# Task 4
In the `dev` namespace, allow all `ingress` and `egress` between `any` pods (no restrictions).

```yaml
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: ingress-rule-from-all-endpoints-in-dev-namespace
  namespace: dev
spec:
  endpointSelector: {}
  ingress:
  - fromEndpoints:
    - {}
  egress:
  - toEndpoints:
    - {}
```

# Task 5
Your task is to configure a network policy `orders-egress-to-inventory-products` with the necessary permissions so that pods with the label `role=orders` can `egress` on **port 3000** to the `inventory` pod and `product` pod.

```yaml
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: orders-egress-to-inventory-products
  namespace: prod
spec:
  endpointSelector:
    matchLabels:
      role: frontend
  egress:
  - toEndpoints:
    - matchLabels:
        role: inventory
        role: product
    toPorts:
      - ports:
        - port: "3000"
          protocol: TCP
```

# Task 6
In `admin` namespace, allow `role=admin` to `egress` on **port 4000** to any `role=user`, and **port 5000** to any `role=products` pods, across `all` namespaces.

```yaml
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: admin-egress-policy
  namespace: admin
spec:
  endpointSelector:
    matchLabels:
      role: admin
  egress:
  - toEndpoints:
    - matchLabels:
        role: user
    toPorts:
      - ports:
        - port: "3000"
          protocol: TCP
  - toEndpoints:
    - matchLabels:
        role: products
    toPorts:
      - ports:
        - port: "5000"
          protocol: TCP
```

# Task 6
The `payment` service (located in the `prod` namespace) requires communication with an external card validation service, which is accessible at the IP address **200.100.17.1**. Create an `egress` policy `cidr-rule` that enables the payment service to send traffic to the external card `validation` service specifically on port **443**.

```yaml
---
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "cidr-rule"
  namespace: prod
spec:
  endpointSelector:
    matchLabels:
      role: payment
  egress:
  - toCIDR:
    - 200.100.17.1/32
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
```

# Task 7
The `payment` service must also be configured to communicate with an external fraud detection service located at the **IP range 100.10.0.0/24**, excluding the address **100.10.0.50**. Add an additional rule to the previously configured policy `cidr-rule` for the `payment` service and update it to enable communication with the external fraud detection service on **port 3000**.

```yaml
---
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "cidr-rule"
  namespace: prod
spec:
  endpointSelector:
    matchLabels:
      role: payment
  egress:
  - toCIDR:
    - 200.100.17.1/32
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
  - toCIDRSet:
    - cidr: 100.10.0.0/24
      except:
      - 100.10.0.50/32
    toPorts:
    - ports:
      - port: "3000"
        protocol: TCP
```

# Task 8
The end users will interact with the application by accessing the `webapp` hosted on the `product` service. Configure an `ingress` policy `my-policy` to allow all traffic from outside the cluster to the products service `role=products`.

```yaml
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: my-policy
  namespace: prod
spec:
  endpointSelector:
    matchLabels:
      role: products
  ingress:
    - fromEntities:
      - world
```

# Task 9
In the `admin` namespace, a `monitoring` service pod has been set up with `role=monitoring`. This service will need to talk to all the nodes in the cluster. Configure an `egress` policy `my-policy` to explicitly allow it to talk to all nodes by configuring `role=monitoring` pods to `egress` to `host` and `remote-node` entities (so they can reach all cluster nodes).

```yaml
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: my-policy
  namespace: admin
spec:
  endpointSelector:
    matchLabels:
      role: monitoring
  egress:
    - toEntities:
      - host
      - remote-node
```

# Task 10
In the `prod` namespace, configure a network policy `my-policy` to allow ingress on HTTP **port 80** to pods with label `role=user` from any pod in the same namespace and **only** for these HTTP methods/paths:
- GET /users
- POST /users
- PATCH /users
- GET /auth/token

```yaml
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: my-policy
  namespace: prod
spec:
  endpointSelector:
    matchLabels:
      role: user 
  ingress:
  - fromEndpoints:
    - matchLabels: {}
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/users"
        - method: "GET"
          path: "/auth/token"
        - method: "POST"
          path: "/users"
        - method: "PATCH"
          path: "/users"
```

# Task 11
A `warehouse` service has been established in the `prod` namespace. Configure a policy named `my-policy` for the `warehouse` service to enable it to send `DNS` requests to the `kube-dns` server located in the `kube-system` namespace for the following Fully Qualified Domain Names (FQDNs):
- kodekloud.com
- api.kodekloud.com
- engineer.kodekloud.com

```yaml
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: "my-policy"
  namespace: prod
spec:
  endpointSelector:
    matchLabels:
      app: warehouse
  egress:
  - toEndpoints:
    - matchLabels:
       "k8s:io.kubernetes.pod.namespace": kube-system
       "k8s:k8s-app": kube-dns
    toPorts:
      - ports:
         - port: "53"
           protocol: ANY
        rules:
          dns:
            - matchName: "kodekloud.com"
            - matchName: "api.kodekloud.com"
            - matchName: "engineer.kodekloud.com"
```
# Task 12
Let’s make sure that all pods in our cluster can talk to the `kube-dns` server. Create a `CiliumClusterwideNetworkPolicy` with name: `allow-dns-clusterwide` to allow `all` pods to `egress` `DNS` queries (port 53 ANY, any FQDN) to the kube-dns server.

```yaml
---
apiVersion: "cilium.io/v2"
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: allow-dns-clusterwide
spec:
  endpointSelector: {}
  egress:
    - toEndpoints:
      - matchLabels:
          "k8s:io.kubernetes.pod.namespace": kube-system
          "k8s:k8s-app": kube-dns
      toPorts:
        - ports:
           - port: "53"
             protocol: ANY
          rules:
            dns:
              - matchPattern: "*"
```