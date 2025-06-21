# Migration methods

There are two primary methods to migrate Ingress API resources to Gateway API:
- manual: manually creating Gateway API resources based on existing Ingress API resources.
- automated: creating rules using the [ingress2gateway](https://github.com/kubernetes-sigs/ingress2gateway) tool. The ingress2gateway project reads Ingress resources from a Kubernetes cluster based on your current Kube Config. It outputs YAML for equivalent Gateway API resources to stdout.

```yaml
# ingress2gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  annotations:
    gateway.networking.k8s.io/generator: ingress2gateway-0.3.0
  creationTimestamp: null
  name: cilium
  namespace: default
spec:
  gatewayClassName: cilium
  listeners:
  - name: http
    port: 80
    protocol: HTTP
  - hostname: bookinfo.cilium.rocks
    name: bookinfo-cilium-rocks-http
    port: 80
    protocol: HTTP
  - hostname: bookinfo.cilium.rocks
    name: bookinfo-cilium-rocks-https
    port: 443
    protocol: HTTPS
    tls:
      certificateRefs:
      - group: null
        kind: null
        name: demo-cert
  - hostname: hipstershop.cilium.rocks
    name: hipstershop-cilium-rocks-http
    port: 80
    protocol: HTTP
  - hostname: hipstershop.cilium.rocks
    name: hipstershop-cilium-rocks-https
    port: 443
    protocol: HTTPS
    tls:
      certificateRefs:
      - group: null
        kind: null
        name: demo-cert
status: {}
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  annotations:
    gateway.networking.k8s.io/generator: ingress2gateway-0.3.0
  creationTimestamp: null
  name: basic-ingress-all-hosts
  namespace: default
spec:
  parentRefs:
  - name: cilium
  rules:
  - backendRefs:
    - name: details
      port: 9080
    matches:
    - path:
        type: PathPrefix
        value: /details
  - backendRefs:
    - name: productpage
      port: 9080
    matches:
    - path:
        type: PathPrefix
        value: /
  - backendRefs:
    - name: productcatalogservice
      port: 3550
    matches:
    - path:
        type: PathPrefix
        value: /hipstershop.ProductCatalogService
  - backendRefs:
    - name: currencyservice
      port: 7000
    matches:
    - path:
        type: PathPrefix
        value: /hipstershop.CurrencyService
status:
  parents: []
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  annotations:
    gateway.networking.k8s.io/generator: ingress2gateway-0.3.0
  creationTimestamp: null
  name: tls-ingress-bookinfo-cilium-rocks
  namespace: default
spec:
  hostnames:
  - bookinfo.cilium.rocks
  parentRefs:
  - name: cilium
  rules:
  - backendRefs:
    - name: details
      port: 9080
    matches:
    - path:
        type: PathPrefix
        value: /details
  - backendRefs:
    - name: productpage
      port: 9080
    matches:
    - path:
        type: PathPrefix
        value: /
status:
  parents: []
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  annotations:
    gateway.networking.k8s.io/generator: ingress2gateway-0.3.0
  creationTimestamp: null
  name: tls-ingress-hipstershop-cilium-rocks
  namespace: default
spec:
  hostnames:
  - hipstershop.cilium.rocks
  parentRefs:
  - name: cilium
  rules:
  - backendRefs:
    - name: productcatalogservice
      port: 3550
    matches:
    - path:
        type: PathPrefix
        value: /hipstershop.ProductCatalogService
  - backendRefs:
    - name: currencyservice
      port: 7000
    matches:
    - path:
        type: PathPrefix
        value: /hipstershop.CurrencyService
status:
  parents: []
```

```bash
kubectl apply -f ingress2gateway.yaml
```