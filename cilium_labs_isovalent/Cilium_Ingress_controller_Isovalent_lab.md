```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: basic-ingress
  namespace: default
spec:
  ingressClassName: cilium
  rules:
    - http:
        paths:
          - backend:
              service:
                name: details
                port:
                  number: 9080
            path: /details
            pathType: Prefix
          - backend:
              service:
                name: productpage
                port:
                  number: 9080
            path: /
            pathType: Prefix
```

```bash
INGRESS_IP=$(kubectl get ingress basic-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grpc-ingress
  namespace: default
spec:
  ingressClassName: cilium
  rules:
    - http:
        paths:
          - backend:
              service:
                name: productcatalogservice
                port:
                  number: 3550
            path: /hipstershop.ProductCatalogService
            pathType: Prefix
          - backend:
              service:
                name: currencyservice
                port:
                  number: 7000
            path: /hipstershop.CurrencyService
            pathType: Prefix
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: default
spec:
  ingressClassName: cilium
  rules:
    - host: hipstershop.cilium.rocks
      http:
        paths:
          - backend:
              service:
                name: productcatalogservice
                port:
                  number: 3550
            path: /hipstershop.ProductCatalogService
            pathType: Prefix
          - backend:
              service:
                name: currencyservice
                port:
                  number: 7000
            path: /hipstershop.CurrencyService
            pathType: Prefix
    - host: bookinfo.cilium.rocks
      http:
        paths:
          - backend:
              service:
                name: details
                port:
                  number: 9080
            path: /details
            pathType: Prefix
          - backend:
              service:
                name: productpage
                port:
                  number: 9080
            path: /
            pathType: Prefix
  tls:
    - hosts:
        - bookinfo.cilium.rocks
        - hipstershop.cilium.rocks
      secretName: demo-cert
``` 

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: "external-lockdown"
spec:
  description: "Block all the traffic originating from outside of the cluster"
  endpointSelector: {}
  ingress:
    - fromEntities:
        - cluster
---
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "hubble-world"
  namespace: "kube-system"
spec:
  description: "Allow access to Hubble from outside"
  endpointSelector:
    matchLabels:
      k8s-app: hubble-relay
  ingress:
    - fromEntities:
        - world
    - toPorts:
        - ports:
            - port: '4245'
              protocol: TCP
```

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: "allow-cidr"
spec:
  description: "Allow all the traffic originating from a specific CIDR"
  endpointSelector:
    matchExpressions:
      - key: reserved:ingress
        operator: Exists
  ingress:
    - fromCIDRSet:
        - cidr: 172.18.0.1/32 # The IP address of the bridge interface in the Kind network is 172.18.0.1
```

```yaml
---
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: "default-deny"
spec:
  description: "Block all the traffic (except DNS) by default"
  egress:
    - toEndpoints:
        - matchLabels:
            io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: '53'
              protocol: UDP
          rules:
            dns:
              - matchPattern: '*'
  endpointSelector:
    matchExpressions:
      - key: io.kubernetes.pod.namespace
        operator: NotIn
        values:
          - kube-system
```

This policy applies to the Ingress, and allows it to egress to any identity in the cluster. Apply it:

```yaml
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: allow-ingress-cluster
spec:
  description: "Allow all the egress traffic from reserved ingress identity to any endpoints in the cluster"
  endpointSelector:
    matchExpressions:
      - key: reserved:ingress
        operator: Exists
  egress:
    - toEntities:
        - cluster
```