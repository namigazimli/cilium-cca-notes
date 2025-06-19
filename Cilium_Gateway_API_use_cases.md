# Cilium Gateway API use cases

Cilium already uses Envoy for L7 policy and observability for some protocols, and this same component is used as the sidecar proxy in many popular Service Mesh implementations. So it's a natural step to extend Cilium to offer more of the features commonly associated with Service Mesh — though contrary to other solutions, without the need for any pod sidecars. Instead, this Envoy proxy is embedded with Cilium, which means that only one Envoy container is required per node.

```yaml
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: echo-1
  name: echo-1
spec:
  ports:
  - port: 8080
    name: high
    protocol: TCP
    targetPort: 8080
  selector:
    app: echo-1
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: echo-1
  name: echo-1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo-1
  template:
    metadata:
      labels:
        app: echo-1
    spec:
      containers:
      - image: gcr.io/kubernetes-e2e-test-images/echoserver:2.2
        name: echo-1
        ports:
        - containerPort: 8080
        env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: cilium-gw
spec:
  gatewayClassName: cilium
  listeners:
  - protocol: HTTP
    port: 80
    name: web-gw-echo
    allowedRoutes:
      namespaces:
        from: Same
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: example-route-1
spec:
  parentRefs:
  - name: cilium-gw
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /echo
    backendRefs:
    - kind: Service
      name: echo-1
      port: 8080

---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: header-http-echo
spec:
  parentRefs:
  - name: cilium-gw
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /cilium-add-a-request-header
    filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        add:
        - name: my-cilium-header-name
          value: my-cilium-header-value
    backendRefs:
      - name: echo-1
        port: 8080
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: response-header-modifier
spec:
  parentRefs:
  - name: cilium-gw
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /multiple
    filters:
    - type: ResponseHeaderModifier
      responseHeaderModifier:
        add:
        - name: X-Header-Add-1
          value: header-add-1
        - name: X-Header-Add-2
          value: header-add-2
        - name: X-Header-Add-3
          value: header-add-3
    backendRefs:
    - name: echo-1
      port: 8080

---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: request-mirror
spec:
  parentRefs:
  - name: cilium-gw
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /mirror
    filters:
    - type: RequestMirror
      requestMirror:
        backendRef:
          name: infra-backend-v2
          port: 8080
    backendRefs:
    - name: infra-backend-v1
      port: 8080

---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: rewrite-path
spec:
  parentRefs:
  - name: cilium-gw
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /prefix/one
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /one
    backendRefs:
    - name: infra-backend-v1
      port: 8080


apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: redirect-path
spec:
  parentRefs:
  - name: cilium-gw
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /original-prefix
    filters:
    - type: RequestRedirect
      requestRedirect:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /replacement-prefix
  - matches:
    - path:
        type: PathPrefix
        value: /path-and-host
    filters:
    - type: RequestRedirect
      requestRedirect:
        hostname: example.org
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /replacement-prefix
  - matches:
    - path:
        type: PathPrefix
        value: /path-and-status
    filters:
    - type: RequestRedirect
      requestRedirect:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /replacement-prefix
        statusCode: 301
  - matches:
    - path:
        type: PathPrefix
        value: /scheme-and-host
    filters:
    - type: RequestRedirect
      requestRedirect:
        hostname: example.org
        scheme: "https"
```

The Gateway API has core support for cross Namespace routing. This is useful when more than one user or team is sharing the underlying networking infrastructure, yet control and configuration must be segmented to minimize access and fault domains. Gateways and Routes can be deployed into different Namespaces and Routes can attach to Gateways across Namespace boundaries. This allows user access control to be applied differently across Namespaces for Routes and Gateways, effectively segmenting access and control to different parts of the cluster-wide routing configuration. The ability for Routes to attach to Gateways across Namespace boundaries are governed by Route attachment. Route attachment is explored in this lab and demonstrates how independent teams can safely share the same Gateway.

Route attachment is an important concept that dictates how Routes attach to Gateways and program their routing rules. It is especially relevant when there are Routes across Namespaces that share one or more Gateways. Gateway and Route attachment is bidirectional - attachment can only succeed if the Gateway owner and Route owner both agree to the relationship. Gateways support attachment constraints which are fields on Gateway listeners that restrict which Routes can be attached. Gateways support Namespaces and Route types as attachment constraints. Any Routes that do not meet the attachment constraints are not able to attach to that Gateway. Similarly, Routes explicitly reference Gateways that they want to attach to through the Route's parentRef field. Together these create a handshake between the infra owners and application owners that enables them to independently define how applications are exposed through Gateways. This is effectively a policy that reduces administrative overhead. App owners can specify which Gateways their apps should use and infra owners can constrain the Namespaces and types of Routes that a Gateway accepts.

```yaml
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: shared-gateway
  namespace: infra-ns
spec:
  gatewayClassName: cilium
  listeners:
    - name: shared-http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              shared-gateway-access: "true"
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: cross-namespace
  namespace: hr
spec:
  parentRefs:
    - name: shared-gateway
      namespace: infra-ns
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /hr
      backendRefs:
        - kind: Service
          name: echo-hr
          port: 9080
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: cross-namespace
  namespace: product
spec:
  parentRefs:
    - name: shared-gateway
      namespace: infra-ns
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /product
      backendRefs:
        - kind: Service
          name: echo-product
          port: 9080
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: cross-namespace
  namespace: careers
spec:
  parentRefs:
    - name: shared-gateway
      namespace: infra-ns
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /careers
      backendRefs:
        - kind: Service
          name: echo-careers
          port: 9080
```

While HTTP is still the king of protocols in the web, gRPC is increasingly used, in particular for its low latency and high throughput capabilities.