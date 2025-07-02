# Policy Enforcement modes
The configuration of the Cilium agent and the Cilium Network Policy determines whether an endpoint accepts traffic from a source or not. The agent can be put into the following three policy enforcement modes:
1. **default** - This is the default behavior for policy enforcement. In this mode, endpoints have unrestricted network access until selected by policy. Upon being selected by a policy, the endpoint permits only allowed traffic. This state is per-direction and can be adjusted on a per-policy basis. For more details, see the [dedicated section on default mode](https://docs.cilium.io/en/latest/security/policy/intro/#policy-mode-default).
2. **always** - With always mode, policy enforcement is enabled on all endpoints even if no rules select specific endpoints. If you want to configure health entity to check cluster-wide connectivity when you start cilium-agent with `enable-policy: always`, you will likely want to enable communications to and from the health endpoint. See Example: Add Health Endpoint.
3. **never** - With never mode, policy enforcement is disabled on all endpoints, even if rules do select specific endpoints. In other words, all traffic is allowed from any source (on ingress) or destination (on egress).
To configure the policy enforcement mode, adjust the Helm value `policyEnforcementMode` or the corresponding configuration flag `enable-policy`.

# Endpoint default policy
By default, all egress and ingress traffic is allowed for all endpoints. When an endpoint is selected by a network policy, it transitions to a default-deny state, where only **explicitly allowed** traffic is permitted. This state is per-direction:
- If any rule selects an [Endpoint](https://docs.cilium.io/en/latest/gettingstarted/terminology/#endpoint) and the rule has an **ingress** section, the endpoint goes into default deny-mode for ingress.
- If any rule selects an [Endpoint](https://docs.cilium.io/en/latest/gettingstarted/terminology/#endpoint) and the rule has an **egress** section, the endpoint goes into default-deny mode for egress.
This means that endpoints start without any restrictions, and the first policy will switch the endpoint’s default enforcement mode (per direction). It is possible to create policies that do not enable the default-deny mode for selected endpoints. The field `EnableDefaultDeny` configures this. Rules with `EnableDefaultDeny` disabled are ignored when determining the default mode. For example, this policy causes all DNS traffic to be intercepted, but does not block any traffic, even if it is the first policy to apply to an endpoint. An administrator can safely apply this policy cluster-wide, without the risk that it transitions an endpoint in to default-deny and causes legitimate traffic to be dropped.
```yml
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: intercept-all-dns
spec:
  endpointSelector:
    matchExpressions:
      - key: "io.kubernetes.pod.namespace"
        operator: "NotIn"
        values:
        - "kube-system"
      - key: "k8s-app"
        operator: "NotIn"
        values:
        - kube-dns
  enableDefaultDeny:
    egress: false
    ingress: false
  egress:
    - toEndpoints:
        - matchLabels:
            io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
          - port: "53"
            protocol: TCP
          - port: "53"
            protocol: UDP
          rules:
            dns:
              - matchPattern: "*"
```

**Warning** - `EnableDefaultDeny` does not apply to layer-7 policy. Adding a layer-7 rule that does not include a layer-7 allow-all will cause drops, even when default-deny is explicitly disabled.

# Rule Basics
All policy rules are based upon a whitelist model, that is, each rule in the policy allows traffic that matches the rule. If two rules exist, and one would match a broader set of traffic, then all traffic matching the broader rule will be allowed. If there is an intersection between two or more rules, then traffic matching the union of those rules will be allowed. Finally, if traffic does not match any of the rules, it will be dropped pursuant to the Policy Enforcement Modes. Policy rules share a common base type which specifies which endpoints the rule applies to and common metadata to identify the rule. Each rule is split into an ingress section and an egress section. The ingress section contains the rules which must be applied to traffic entering the endpoint, and the egress section contains rules applied to traffic coming from the endpoint matching the endpoint selector. Either ingress, egress, or both can be provided. If both ingress and egress are omitted, the rule has no effect


# Layer 3 Cilium Network Policies examples
The layer 3 policy establishes the base connectivity rules regarding which endpoints can talk to each other. Layer 3 policies can be specified using the following methods:
- **Endpoints based**: This is used to describe the relationship if both endpoints are managed by Cilium and are thus assigned labels. The advantage of this method is that IP addresses are not encoded into the policies and the policy is completely decoupled from the addressing.
- **Services based**: This is an intermediate form between Labels and CIDR and makes use of the services concept in the orchestration system. A good example of this is the Kubernetes concept of Service endpoints which are automatically maintained to contain all backend IP addresses of a service. This allows to avoid hardcoding IP addresses into the policy even if the destination endpoint is not controlled by Cilium.
- **Entities based**: Entities are used to describe remote peers which can be categorized without knowing their IP addresses. This includes connectivity to the local host serving the endpoints or all connectivity to outside of the cluster. 
- **Node based**: This is an extension of `remote-node` entity. Optionally nodes can have unique identity that can be used to allow/block access only from specific ones.
- **IP/CIDR based**: This is used to describe the relationship to or from external services if the remote peer is not an endpoint. This requires to hardcode either IP addresses or subnets into the policies. This construct should be used as a last resort as it requires stable IP or subnet assignments.
- **DNS based**: Selects remote, non-cluster, peers using DNS names converted to IPs via DNS lookups. It shares all limitations of the IP/CIDR based rules above. DNS information is acquired by routing DNS traffic via a proxy. DNS TTLs are respected.

## Endpoints based
Endpoints-based L3 policy is used to establish rules between endpoints inside the cluster managed by Cilium. Endpoints-based L3 policies are defined by using an Endpoint Selector inside a rule to select what kind of traffic can be received (on ingress), or sent (on egress). An empty Endpoint Selector allows all traffic.

### Ingress
An endpoint is allowed to receive traffic from another endpoint if at least one ingress rule exists which selects the destination endpoint with the Endpoint Selector in the `endpointSelector` field. To restrict traffic upon ingress to the selected endpoint, the rule selects the source endpoint with the Endpoint Selector in the `fromEndpoints` field.

#### Simple ingress allow
The following example illustrates how to use a simple ingress rule to allow communication from endpoints with the label `role=frontend` to endpoints with the label `role=backend`.
```yml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "l3-rule"
spec:
  endpointSelector:
    matchLabels:
      role: backend
  ingress:
  - fromEndpoints:
    - matchLabels:
        role: frontend
```

#### Ingress allow all endpoints
In this policy pods with label `role: backend` will accept all traffic from all endpoints in the `dev` namespace.
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

### Egress
An endpoint is allowed to send traffic to another endpoint if at least one egress rule exists which selects the destination endpoint with the Endpoint Selector in the `endpointSelector` field. To restrict traffic upon egress to the selected endpoint, the rule selects the destination endpoint with the Endpoint Selector in the `toEndpoints` field.

#### Simple egress allow
In this policy pods with label `role: frontend` will access all traffic to pods with label `role: backend` in the `dev` namespace.
```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: l3-egress-rule
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

#### Egress allow all endpoints
An empty Endpoint Selector will select all egress endpoints from an endpoint based on the CiliumNetworkPolicy namespace (`default` by default). The following rule allows all egress traffic from endpoints with the label `role=frontend` to all other endpoints in the same namespace:
```yml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "allow-all-from-frontend"
spec:
  endpointSelector:
    matchLabels:
      role: frontend
  egress:
  - toEndpoints:
    - {}
```

### Ingress/Egress default deny
An endpoint can be put into the default deny mode at ingress or egress if a rule selects the endpoint and contains the respective rule section ingress or egress. This policy will deny all traffic from all endpoints in the `dev` namespace to pods with label `role: frontend` and vice versa.
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

## Service based
Traffic from endpoints to services running in your cluster can be allowed via `toServices` statements in Egress rules. Policies can reference Kubernetes Services by name or label selector. This example shows how to allow all endpoints with the label `role=frontend` to talk to all endpoints of Kubernetes service `backend` in the `dev` namespace as well as all services with label `app=app1` in the `staging` namespace.

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
        namespace: staging
```

## Entities based
`fromEntities` is used to describe the entities that can access the selected endpoints. `toEntities` is used to describe the entities that can be accessed by the selected endpoints. The following entities are defined:
- **host**: The host entity includes the local host. This also includes all containers running in host networking mode on the local host.
- **remote-node**: Any node in any of the connected clusters other than the local host. This also includes all containers running in host-networking mode on remote nodes.
- **kube-apiserver**: The kube-apiserver entity represents the kube-apiserver in a Kubernetes cluster. This entity represents both deployments of the kube-apiserver: within the cluster and outside of the cluster.
- **ingress**: The ingress entity represents the Cilium Envoy instance that handles ingress L7 traffic. Be aware that this also applies for pod-to-pod traffic within the same cluster when using ingress endpoints (also known as hairpinning).
- **cluster**: Cluster is the logical group of all network endpoints inside of the local cluster. This includes all Cilium-managed endpoints of the local cluster, unmanaged endpoints in the local cluster, as well as the host, remote-node, and init identities. This also includes all remote nodes in a clustermesh scenario.
- **init**: The init entity contains all endpoints in bootstrap phase for which the security identity has not been resolved yet. This is typically only observed in non-Kubernetes environments. See section Endpoint Lifecycle for details.
- **health**: The health entity represents the health endpoints, used to check cluster connectivity health. Each node managed by Cilium hosts a health endpoint. See Checking cluster connectivity health for details on health checks.
- **unmanaged**: The unmanaged entity represents endpoints not managed by Cilium. Unmanaged endpoints are considered part of the cluster and are included in the cluster entity.
- **world**: The world entity corresponds to all endpoints outside of the cluster. Allowing to world is identical to allowing to CIDR 0.0.0.0/0. An alternative to allowing from and to world is to define fine grained DNS or CIDR based policies.
- all: The all entity represents the combination of all known clusters as well world and whitelists all communication.

### Access to/from kube-apiserver
The `kube-apiserver-entity-rule` CiliumNetworkPolicy ensures that pods with the label `role: backend` in the `dev` namespace are explicitly allowed to initiate connections to the Kubernetes API Server.
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

### Access to/from local host
Allow all endpoints with the label `env=dev` to access the host that is serving the particular endpoint.
```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "dev-to-host"
spec:
  endpointSelector:
    matchLabels:
      env: dev
  egress:
    - toEntities:
      - host
```

### Access to/from outside cluster
This example shows how to enable access from outside of the cluster to all endpoints that have the label `role=public`.
```yml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "from-world-to-role-public"
spec:
  endpointSelector:
    matchLabels:
      role: public
  ingress:
    - fromEntities:
      - world
```

## IP/CIDR based
CIDR policies are used to define policies to and from endpoints which are not managed by Cilium and thus do not have labels associated with them. These are typically external services, VMs or metal machines running in particular subnets. CIDR policy can also be used to limit access to external services, for example to limit external access to a particular IP range. CIDR policies can be applied at ingress or egress. CIDR rules apply if Cilium cannot map the source or destination to an identity derived from endpoint labels, ie the Special Identities. For example, CIDR rules will apply to traffic where one side of the connection is:
- A network endpoint outside the cluster
- The host network namespace where the pod is running.
- Within the cluster prefix but the IP’s networking is not provided by Cilium.
- (optional) Node IPs within the cluster
Conversely, CIDR rules do not apply to traffic where both sides of the connection are either managed by Cilium or use an IP belonging to a node in the cluster (including host networking pods). This traffic may be allowed using labels, services or entities -based policies as described above.

### Allow to external CIDR block
This example shows how to allow all endpoints with the label `app=myService` to talk to the external IP `20.1.1.1`, as well as the CIDR prefix `10.0.0.0/8`, but not CIDR prefix `10.96.0.0/12`
```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "cidr-rule"
spec:
  endpointSelector:
    matchLabels:
      app: myService
  egress:
  - toCIDR:
    - 20.1.1.1/32
  - toCIDRSet:
    - cidr: 10.0.0.0/8
      except:
      - 10.96.0.0/12
```

## DNS based
DNS policies are used to define Layer 3 policies to endpoints that are not managed by Cilium, but have DNS queryable domain names. The IP addresses provided in DNS responses are allowed by Cilium in a similar manner to IPs in CIDR based policies. They are an alternative when the remote IPs may change or are not know prior, or when DNS is more convenient. An L3 CIDR based rule is generated for every toFQDNs rule and applies to the same endpoints. The IP information is selected for insertion by matchName or matchPattern rules, and is collected from all DNS responses seen by Cilium on the node. Multiple selectors may be included in a single egress rule. `toFQDNs` egress rules cannot contain any other L3 rules, such as `toEndpoints` (under Endpoints Based) and `toCIDRs` (under CIDR Based). They may contain L4/L7 rules, such as `toPorts` (see Layer 4 Examples) with, optionally, HTTP and Kafka sections

### Example
The example below allows all DNS traffic on port 53 to the DNS service and intercepts it via the DNS Proxy. If using a non-standard DNS port for a DNS application behind a Kubernetes Service, the port must match the backend port. When the application makes a request for my-remote-service.com, Cilium learns the IP address and will allow traffic due to the match on the name under the toFQDNs.matchName rule.
```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "to-fqdn"
spec:
  endpointSelector:
    matchLabels:
      app: test-app
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
    - toFQDNs:
        - matchName: "my-remote-service.com"
```

# Layer 4 Cilium Network Policies examples
The following rule limits all endpoints with the label `role: frontend` to only be able to emit packets using TCP on port 80, to any layer 3 destination.
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

## Mix rule with L3 and L4 rules
In this policy our endpoints with the label `role: backend` will accept traffic from endpoints with the `role: frontend` through port TCP/80 in the `dev` namespace.
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

# Layer 7 examples
Layer 7 policy rules are embedded into Layer 4 Examples rules and can be specified for ingress and egress. L7Rules structure is a base type containing an enumeration of protocol specific fields. The structure is implemented as a union, i.e. only one member field can be used per port. If multiple `toPorts` rules with identical `PortProtocol` select an overlapping list of endpoints, then the layer 7 rules are combined together if they are of the same type. If the type differs, the policy is rejected.

## Allow GET /products requests
The following example allows `GET` requests to the URL `/products` from the endpoints with the labels `role=frontend` to endpoints with the labels `role=backend`, but requests to any other URL, or using another method, will be rejected. Requests on ports other than port `80` will be dropped.

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

## DNS Policy
The `l7-dns-rule` CiliumNetworkPolicy ensures that pods with the `role: backend` in the `dev` namespace can:
- Perform DNS lookups for `google.com` and its subdomains by allowing UDP/TCP traffic on port 53 to the `kube-dns` pods in the `kube-system` namespace, with L7 DNS filtering applied.
- Make HTTP requests to `google.com` and its subdomains by allowing TCP traffic on port 80 to those specific FQDNs.
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

# Clusterwide Network Policies
CiliumNetworkPolicy only allows to bind a policy restricted to a particular namespace. There can be situations where one wants to have a cluster-scoped effect of the policy, which can be done using Cilium’s CiliumClusterwideNetworkPolicy Kubernetes custom resource. The specification of the policy is same as that of CiliumNetworkPolicy except that it is not namespaced. 
In the cluster, this policy will allow ingress traffic from pods matching the label `name=luke` from any namespace to pods matching the labels `name=leia` in any namespace.
```yml
apiVersion: "cilium.io/v2"
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: "clusterwide-policy-example"
spec:
  description: "Policy for selective ingress allow to a pod from only a pod with given label"
  endpointSelector:
    matchLabels:
      name: leia
  ingress:
  - fromEndpoints:
    - matchLabels:
        name: luke
```

The following example ensures that every single pod in every namespace of the cluster is permitted to send DNS queries (both UDP and TCP) to the `kube-dns` service running in the `kube-system` namespace.
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