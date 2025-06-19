# Task 1
In the `video-app` namespace, an upload pod should only egress to `video-encoding` on TCP port **5000**. The existing policy allows port **4000**, so upload cannot reach video-encoding on the correct port. Update the `my-policy` CiliumNetworkPolicy so that port **5000** is permitted.

```yaml
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: my-policy
  namespace: video-app
spec:
  endpointSelector:
    matchLabels:
      role: video-encoding
  egress:
  - toEndpoints:
    - matchLabels:
        role: video-encoding
    toPorts:
    - ports:
      - port: "5000" # Updated port number to 5000
        protocol: TCP
```

# Task 2
In the `video-app` namespace, the `subscription` pod should only receive TCP port **80** traffic from pods labeled `role=content-access`. For some reason, all pods are still able to communicate with the subscription service. Find out the cause and update `my-policy` accordingly.

```yaml
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: my-policy
  namespace: video-app
spec:
  endpointSelector:
    matchLabels:
      role: subscription
  ingress:
  - fromEndpoints:
    - matchLabels:
        role: content-access
    # Added following section for receving traffic on port 80
    toPorts:    
    - ports:
      - port: "80"
        protocol: TCP
```

# Task 3
The `admin` pod in the `admin` namespace must connect to the `content-management` pod in `video-app` on TCP port **443**. Two policies exist: `content-management-policy` (in video-app) and `admin-policy` (also in video-app). Figure out why the admin service can’t talk to the content-management service.

```yaml
# content-management-policy
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: content-management-policy
  namespace: video-app
spec:
  endpointSelector:
    matchLabels:
      role: content-management
  ingress:
  - fromEndpoints:
    - matchLabels:
        role: admin
        k8s:io.kubernetes.pod.namespace: admin  # Added namespace filter
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP

# admin-policy
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: admin-policy
  namespace: video-app  # Updated namespace to video-app
spec:
  endpointSelector:
    matchLabels:
      role: admin
  egress:
  - toEndpoints:
    - matchLabels:
        role: content-management
        k8s:io.kubernetes.pod.namespace: video-app
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
```

# Task 4
The `subscription` service in the `video-app` namespace communicates with the `notification` service on port **3000** in the `video-app` namespace. Recently, an engineer implemented an `egress` policy for the subscription service to permit egress traffic to the notification service. However, after applying the policy, the application encountered issues. The engineer confirmed that the subscription service could access the notification service Pod’s IP on port **3000**, yet the application remained non-functional. Review and fix the policy `subscription-policy`.

```yaml
---
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: subscription-policy
  namespace: video-app
spec:
  endpointSelector:
    matchLabels:
      role: subscription
  egress:
  - toEndpoints:
    - matchLabels:
        role: notification
    toPorts:
    - ports:
      - port: "3000" # Updated port number to 3000
        protocol: TCP
  # Added for DNS queries
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
        - matchPattern: '*'
```

# Task 5
A **cluster-wide** policy named `external-lockdown` is currently blocking all external ingress (`fromEntities: world`), but it’s also preventing pods from talking to each other internally. Update `external-lockdown` so it continues to block external traffic yet allows **intra-cluster pod-to-pod communication**.

```yaml
---
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: external-lockdown
  namespace: video-app
spec:
  endpointSelector: {}
  ingressDeny:
  - fromEntities:
    - world
  ingress:
  - fromEntities:
    - all
``` 