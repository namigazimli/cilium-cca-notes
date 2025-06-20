# Task 1
Enable the Cilium Ingress Controller and configure it such that only one loadbalancer gets created for all ingress.

```bash
helm upgrade cilium cilium/cilium \
    --namespace kube-system \
    --reuse-values \
    --set nodePort.enabled=true \
    --set ingressController.enabled=true \
    --set ingressController.loadbalancerMode=shared
```

# Task 2
In our cluster, we are hosting two applications: funchat.com and streameasy.com. Create an Ingress rule named multi-app-ingress in the default namespace with the following routing configurations:
- Route funchat.com/auth to the service chat-auth
- Route funchat.com/messages to the service chat-messages
- Route streameasy.com/video to the service streameasy-video
- Route streameasy.com/moderation to the service streameasy-moderation

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-app-ingress
  namespace: default
spec:
  ingressClassName: cilium
  rules:
  - host: "funchat.com"
    http:
      paths:
      - path: /auth
        pathType: Prefix
        backend:
          service:
            name: chat-auth
            port:
              number: 80
      - path: /messages
        pathType: Prefix
        backend:
          service:
            name: chat-messages
            port:
              number: 80
  - host: "streameasy.com"
    http:
      paths:
      - path: /video
        pathType: Prefix
        backend:
          service:
            name: streameasy-video
            port:
              number: 80
      - path: /moderation
        pathType: Prefix
        backend:
          service:
            name: streameasy-moderation
            port:
              number: 80
```

# Task 3
Enable Gateway API support in Cilium via Helm values.

```bash
helm upgrade cilium cilium/cilium  --namespace kube-system --reuse-values --set gatewayAPI.enabled=true
```

# Task 4
Create a Gateway named `my-gateway` in the default namespace using the Cilium GatewayClass. Listeners:
- HTTP on port `80`
- HTTPS on port `443`, TLS terminate using stored secret `my-cert` 

```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: cilium
  listeners:
  - protocol: HTTP
    port: 80
    name: http
    allowedRoutes:
      namespaces:
        from: Same
  - name: https
    protocol: HTTPS
    port: 443
    allowedRoutes:
      namespaces:
        from: Same
    tls:
      certificateRefs:
      - kind: Secret
        name: my-cert
```

# Task 5
Create an HTTPRoute named `multi-app-route` in the `default` namespace bound to `my-gateway` with the following routing:
- Host `blog.example.com`, paths `/home` → service `blog-home`, `/api` → service `blog-api`
- Host `shop.example.com`, paths `/cart` → service `shop-cart`, `/checkout` → service `shop-checkout`

```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: multi-app-route
spec:
  parentRefs:
  - name: my-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /home
      headers:
        - name: Host 
          value: blog.example.com
    backendRefs:
    - name: blog-home
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /api
      headers:
        - name: Host 
          value: blog.example.com
    backendRefs:
    - name: blog-api
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /cart
      headers:
        - name: Host 
          value: shop.example.com
    backendRefs:
    - name: shop-cart
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /checkout
      headers:
        - name: Host 
          value: shop.example.com
    backendRefs:
    - name: shop-checkout
      port: 80
```