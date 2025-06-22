While this lab focuses on two key features to improve traffic optimization, Service Traffic Distribution and Local Redirect Policies, Cilium offers several other features that can be leveraged for Traffic Engineering, such as:
- Egress Gateway (optionally with High Availability and Topology Aware affinity)
- BGP peering (with support for communities, timers, etc.)
- multiple routing modes (e.g. direct routing)

Given that, by default, Kubernetes has little awareness of the underlying platform it runs on, network traffic is sent without consideration, causing potential inefficiencies such as increased delay as traffic is sent to a different availability zone or to a remote DNS server.

In this lab, you will learn a couple of features that will let you optimize traffic within the cluster:
- Service Traffic Distribution
- Local Redirect Policy

# Kind Cluster
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
featureGates:
  ServiceTrafficDistribution: true
nodes:
  - role: control-plane
  - role: worker
    labels:
      topology.kubernetes.io/zone: port
  - role: worker
    labels:
      topology.kubernetes.io/zone: starboard
networking:
  disableDefaultCNI: true
  kubeProxyMode: none
```
Notice that:
- The `featureGates.ServiceTrafficDistribution` is set to `true` to enable this feature that will be explore in the next challenge.
- The `networking.disableDefaultCNI` flag removes the default CNI from the Kind cluster, and we installed Cilium instead.
- The `networking.kubeProxyMode` flag disabled KubeProxy on the Kind cluster. Cilium will play that role.

# Nodes
In the `nodes` section, you can see that the cluster consists of three nodes:
- 1 `control-plane` node running the Kubernetes control plane and etcd
- 2 `worker` nodes to deploy the applications
These two worker nodes have been labeled with a `topology.kubernetes.io/zone` label, the way a cloud provider would typically label nodes by availability zone. We will use these labels later.

# Cilium
Cilium was pre-installed during the lab boot-up with the following command:
```shell
cilium install \
  --version 1.17.1 \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set localRedirectPolicy=true \
  --set loadBalancer.serviceTopology=true
```

The following command will wait for Cilium to be up and running and report its status:
```shell
cilium status --wait
```

# Two Droids and Two Jedis
For this lab, we will deploy multiple pods in the tatooine namespace:
- c3-po and r2-d2, representing two droids, respectively scheduled on the kind-worker and kind-worker2 nodes
- obi-wan and luke, representing two Jedis
- a stormtroopers DaemonSet deployment one stormtrooper pod on each of the nodes.
Apply the manifest:
```yaml
# kubectl apply -f tatooine.yml
apiVersion: v1
kind: Pod
metadata:
  name: c3-po
  namespace: landspeeder
  labels:
    name: c3-po
    role: droid
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: [starboard]
  containers:
    - name: nginx
      image: nginx:latest
      volumeMounts:
        - name: c3-po-config-volume
          mountPath: /etc/nginx/conf.d
  volumes:
    - name: c3-po-config-volume
      configMap:
        name: c3-po-config

---
apiVersion: v1
kind: Pod
metadata:
  name: r2-d2
  namespace: landspeeder
  labels:
    name: r2-d2
    role: droid
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: [port]
  containers:
    - name: nginx
      image: nginx:latest
      volumeMounts:
        - name: r2-d2-config-volume
          mountPath: /etc/nginx/conf.d
  volumes:
    - name: r2-d2-config-volume
      configMap:
        name: r2-d2-config

---
apiVersion: v1
kind: Pod
metadata:
  name: obi-wan
  namespace: landspeeder
  labels:
    name: obi-wan
    role: jedi
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: [port]
  containers:
    - name: nginx
      image: nginx:latest
      volumeMounts:
        - name: obi-wan-config-volume
          mountPath: /etc/nginx/conf.d
  volumes:
    - name: obi-wan-config-volume
      configMap:
        name: obi-wan-config
---
apiVersion: v1
kind: Pod
metadata:
  name: luke
  namespace: landspeeder
  labels:
    name: luke
    role: jedi
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: [starboard]
  containers:
    - name: nginx
      image: nginx:latest
      volumeMounts:
        - name: luke-config-volume
          mountPath: /etc/nginx/conf.d
  volumes:
    - name: luke-config-volume
      configMap:
        name: luke-config
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: stormtroopers
  namespace: spaceport
spec:
  selector:
    matchLabels:
      name: stormtrooper
  template:
    metadata:
      labels:
        name: stormtrooper
        role: soldier
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: c3-po-config
  namespace: landspeeder
data:
  default.conf: |
    server {
      listen 80;
      location / {
        add_header X-Identification "C3-PO";
        return 200 'Greetings, Human! I am C-3PO, human-cyborg relations. Fluent in over six million forms of communication, and at your service!\n';
      }
    }

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: r2-d2-config
  namespace: landspeeder
data:
  default.conf: |
    server {
      listen 80;
      location / {
        add_header X-Identification "R2-D2";
        return 200 'Beep beep boop boop! *whistles*\n';
      }
    }

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: obi-wan-config
  namespace: landspeeder
data:
  default.conf: |
    server {
      listen 80;
      add_header X-Identification "Obi-Wan" always;
      add_header X-Saber-Color "blue 🔵" always; 

      location / {
        return 404 'These are not the droids you are looking for.\n';
      }
    }
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: luke-config
  namespace: landspeeder
data:
  default.conf: |
    server {
      listen 80;
      add_header X-Identification "Luke" always;
      add_header X-Saber-Color "green 🟢" always; 

      location / {
        return 404 'Like Obi-Wan said 🤷\n';
      }
    }
```
Check the resources and wait for the pods to be ready:
```shell
kubectl get -f tatooine.yaml -o wide
```

# Service Traffic Distribution
Cilium 1.16 introduced support for Kubernetes’ new “Traffic Distribution for Services” model which aims at providing topology-aware traffic engineering. It can be considered the successor to features such as Topology-Aware Routing and Topology-Aware Hints. Service Traffic Distribution enables users to express preferences on traffic policy for a Kubernetes Service.
Let's learn more, with an example.

# Searching for Droids
Let's deploy a droids service in the landspeeder namespace:
```yaml
# kubectl -n landspeeder apply -f droids.yaml -o yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"droids","namespace":"landspeeder"},"spec":{"ports":[{"port":80,"protocol":"TCP","targetPort":80}],"selector":{"role":"jedi"},"type":"ClusterIP"}}
  creationTimestamp: "2025-06-22T09:19:21Z"
  name: droids
  namespace: landspeeder
  resourceVersion: "2348"
  uid: 450a4356-de93-47dc-8d87-0721d6541943
spec:
  clusterIP: 10.96.213.114
  clusterIPs:
  - 10.96.213.114
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    role: jedi
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```
This service works as an L4 load-balancer (implemented by Cilium's kube-proxy replacement), sending traffic to pods labeled `role=jedi`.
![Alt text](https://play.instruqt.com/assets/tracks/yd8ni0pb6cxk/2b199bd13bf9f0f4b09687f8fdb84446/assets/traffic-distribution-base.png)

Find the `stormtrooper` pods running on `kind-worker` and `kind-worker2` respectively:
```shell
TROOPER1=$(kubectl -n spaceport get po -l role=soldier --field-selector spec.nodeName=kind-worker -o name)
echo $TROOPER1
TROOPER2=$(kubectl -n spaceport get po -l role=soldier --field-selector spec.nodeName=kind-worker2 -o name)
echo $TROOPER2
```
Test the service by running this a few times:
```shell
kubectl -n spaceport exec $TROOPER1 -- \
  curl -si droids.landspeeder.svc.cluster.local
```
You will see HTTP 404 responses coming from both the obi-wan and luke pods (check the values of the response headers to identify the backends). You can also try it with the second worker node:
```shell
kubectl -n spaceport exec $TROOPER2 -- \
  curl -si droids.landspeeder.svc.cluster.local
```

# Service Traffic Distribution
[Service Traffic Distribution](https://kubernetes.io/docs/concepts/services-networking/service/#traffic-distribution) enables users to express preferences on traffic policy for a Kubernetes Service. Currently, the only supported value is `PreferClose`, indicating a preference for routing traffic to endpoints that are topologically proximate to the client. By keeping the traffic within a local zone, users can optimize performance, cost and reliability. Introduced in Kubernetes 1.30, Service Traffic Distribution can be enabled directly in the Service specification (rather than using annotations as it was the case with Topology-Aware Hints & Routing). The worker nodes in our cluster have been tagged with the well-known `topology.kubernetes.io/zone` label in order to specify their topology. Inspect the node labels:
```shell
kubectl get no --show-labels | \
  GREP_COLORS='mt=1;32' grep --color=always -E '[^=]+$' | \
  grep --color=always topology.kubernetes.io/zone=
```
The `kind-worker` node is tagged as `port` (i.e. left side of the aircraft) while `kind-worker2` is set to `starboard` (the right side of the aircraft). The pods in the `landspeeder` (the aircraft) namespace have been specified with a node affinity to either the `port` or `starboard` value of the `topology.kubernetes.io/zone` label, emulating their deployment in multiple availability zones. For example, check the `r2-d2` pod's affinity:
```yaml
# kubectl -n landspeeder get pod r2-d2 -o yaml | yq .spec.affinity
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
      - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
              - port
```

# Local Zone Preference
With the </> Editor, uncomment the `spec.trafficDistribution` setting in `droids.yaml` and re-apply the manifest in the >_ Terminal.
```yaml
# kubectl -n landspeeder apply -f droids.yaml -o yaml
---
apiVersion: v1
kind: Service
metadata:
  name: droids
  namespace: landspeeder
spec:
  trafficDistribution: PreferClose
  selector:
    role: jedi
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```
Now make queries to the droids service from the pod on the first worker:
```shell
kubectl -n spaceport exec $TROOPER1 -- \
  curl -si droids.landspeeder.svc.cluster.local
```
Repeat the command multiple times. All responses should come from Obi-Wan (on the left side of the aircraft). Try with the second worker:
```shell
kubectl -n spaceport exec $TROOPER2 -- \
  curl -si droids.landspeeder.svc.cluster.local
```
All responses should now come from Luke (on the right side of the aircraft), as Cilium is applying the topology affinity to choose the service backend based on the caller.
![Alt text](https://play.instruqt.com/assets/tracks/yd8ni0pb6cxk/ff53894e9253c2892089074566d28bcf/assets/traffic-distribution-affinity.png)

# Resilience
Delete the luke pod:
```shell
kubectl -n landspeeder delete pod luke
```
Then try accessing the droids service from the second worker again:
```shell
kubectl -n spaceport exec $TROOPER2 -- \
  curl -si droids.landspeeder.svc.cluster.local
```
The traffic now goes to Obi-Wan, as there is no available backend implementing the `droids` service in the topological vicinity of the second trooper pod.

# What did we learn?
With Service Traffic Distribution, you can now avoid these pesky inter-AZ data charges and reduce the latency of your applications. But Service Traffic Distribution is not the only traffic engineering feature we want to highlight in this lab. Just as Service Traffic Distribution helps keep the traffic within the zone, Local Redirect Policy (LRP) prevents traffic from even leaving the node by directing the traffic bounds for services to the local backend.

**Local Redirect Policy** - Local Redirect Policy (LRP) was initially introduced in Cilium 1.9 and is commonly adopted by users looking at optimizing networking performance and reducing latency. Local Redirect Policy enables pod traffic destined to an IP address and port/protocol tuple or Kubernetes service to be redirected locally to backend pod(s) within a node, using eBPF.

Without LRP:
![Alt text](https://play.instruqt.com/assets/tracks/yd8ni0pb6cxk/b32ca437fc0d4e552afe043fdaebff45/assets/traffic-distribution-lrp-no-lrp.png)

With LRP:
![Alt text](https://play.instruqt.com/assets/tracks/yd8ni0pb6cxk/1ebbbe4f461fa0f03c808fe3ffec7ec5/assets/traffic-distribution-lrp-with-lrp.png)

# Where are the Droids Hiding?
The Storm Troopers are not satisfied. They're pretty sure Obi-Wan is hiding something and intend to check the droids on the aircraft. Let's check the state of the current droids service. First, get the service cluster IP:
```shell
kubectl -n landspeeder get svc droids
```
Let's get the Cilium agent on the first worker node:
```shell
CILIUM1=$(kubectl -n kube-system get po -l k8s-app=cilium --field-selector spec.nodeName=kind-worker -o name)
echo $CILIUM1
```
Next, verify on that node that Cilium’s eBPF kube-proxy replacement created a ClusterIP service entry for this IP:
```shell
kubectl exec -it -n kube-system $CILIUM1 -c cilium-agent -- \
  cilium-dbg service list | \
  grep -E '^ID|10.96.213.114:80' | \
  GREP_COLORS='mt=01;31' grep --color=always -B1 '10.96.213.114:80' | \
  GREP_COLORS='mt=01;32:sl=01;33' grep --color=always -B1 -E 'LocalRedirect|ClusterIP'
```
You can see that it currently behaves as a standard ClusterIP service pointing to a single IP (in yellow, which is the `obi-wan` pod, since the `luke` pod has been deleted). Verify that the backend IP corresponds to that of the `obi-wan` pod:
```shell
kubectl -n landspeeder get pod obi-wan -o wide
```

# Create Cilium Local Redirect Policy
There are two types of Local Redirect Policies:
- ServiceMatcher (which we will see now and in the next task)
- AddressMatcher (which you will see in the final challenge).
When using ServiceMatcher, traffic bound for a particular service will be redirected to only node-local backend pods. When a policy of this type is applied, the existing service entry created by Cilium’s eBPF kube-proxy replacement will be replaced with a new service entry of type LocalRedirect. This entry may only have node-local backend pods. Apply the CiliumLocalRedirectPolicy that will override the droids service to point it to the actual droid pods:
```yaml
# kubectl apply -f droids-lrp.yaml -o yaml | yq .spec
redirectBackend:
  localEndpointSelector:
    matchLabels:
      role: droid
  toPorts:
    - port: "80"
      protocol: TCP
redirectFrontend:
  serviceMatcher:
    namespace: landspeeder
    serviceName: droids
skipRedirectFromBackend: false
```
You can see that this will cause traffic bound for the service `droids` to be redirected to local backends with the `role=droid` label.
![Alt text](https://play.instruqt.com/assets/tracks/yd8ni0pb6cxk/9120654ec5774e5e8a43f1ca246483fc/assets/traffic-distribution-lrp.png)
Verify Cilium's eBPF service table again:
```shell
kubectl exec -it -n kube-system $CILIUM1 -c cilium-agent -- \
  cilium-dbg service list | \
  grep -E '^ID|10.96.213.114:80' | \
  GREP_COLORS='mt=01;31' grep --color=always -B1 '10.96.213.114:80' | \
  GREP_COLORS='mt=01;32:sl=01;33' grep --color=always -B1 -E 'LocalRedirect|ClusterIP'
```
The service is now marked as a local redirect to a single IP, which is the IP of the local pod labeled role=droid. Verify this:
```shell
kubectl -n landspeeder get po -l role=droid --field-selector spec.nodeName=kind-worker -o wide
```
Let's get the troopers per node again:
```shell
TROOPER1=$(kubectl -n spaceport get po -l role=soldier --field-selector spec.nodeName=kind-worker -o name)
echo $TROOPER1
TROOPER2=$(kubectl -n spaceport get po -l role=soldier --field-selector spec.nodeName=kind-worker2 -o name)
echo $TROOPER2
```
Then, access the droids service from the first worker:
```shell
kubectl -n spaceport exec $TROOPER1 -- \
  curl -si droids.landspeeder.svc.cluster.local
```
All responses now come from R2-D2, as it is the pod labeled role=droid on the first worker. Try on the second worker:
```shell
kubectl -n spaceport exec $TROOPER2 -- \
  curl -si droids.landspeeder.svc.cluster.local
```
All responses come from C3-PO, the droid pod local to the second node.

# NodeLocal DNS Cache
After exploring a fun use case for LRP, let's now consider the most common use case for it: NodeLocal DNS Cache. First, let's have a quick refresher on the default DNS configuration in Kubernetes. Consider a typical Kubernetes cluster: 'ClusterFirst' is the default DNS mode, unless `dnsPolicy` is explicitly specified. In this mode, any DNS query that does not match the configured cluster domain suffix, such as "www.kubernetes.io", is forwarded to an upstream nameserver by the DNS server.

# Implementing NodeLocal DNS Cache with a Local Redirect Policy
NodeLocal DNSCache improves Cluster DNS performance by running a DNS caching agent on cluster nodes as a DaemonSet. By leveraging Local Redirect Policy and NodeLocal DNS Cache, Pods will reach out to the DNS caching agent running on the same node. The local caching agent will query kube-dns service for cache misses of cluster hostnames ("cluster.local" suffix by default).

# NodeLocal DNS cache
Before using LRP to redirect DNS traffic and optimize latency, we first need to create a local DNS node-cache. Let's review the DNS node cache configuration:
```yaml
# yq node-local-dns.yaml
---
apiVersion: "cilium.io/v2"
apiVersion: v1
kind: ServiceAccount
metadata:
  name: node-local-dns
  namespace: kube-system
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns-upstream
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/name: "KubeDNSUpstream"
spec:
  ports:
    - name: dns
      port: 53
      protocol: UDP
      targetPort: 53
    - name: dns-tcp
      port: 53
      protocol: TCP
      targetPort: 53
  selector:
    k8s-app: kube-dns
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: node-local-dns
  namespace: kube-system
data:
  Corefile: |
    cluster.local:53 {
        errors
        cache {
                success 9984 30
                denial 9984 5
        }
        reload
        loop
        bind 0.0.0.0
        forward . __PILLAR__CLUSTER__DNS__ {
                force_tcp
        }
        prometheus :9253
        health
        }
    in-addr.arpa:53 {
        errors
        cache 30
        reload
        loop
        bind 0.0.0.0
        forward . __PILLAR__CLUSTER__DNS__ {
                force_tcp
        }
        prometheus :9253
        }
    ip6.arpa:53 {
        errors
        cache 30
        reload
        loop
        bind 0.0.0.0
        forward . __PILLAR__CLUSTER__DNS__ {
                force_tcp
        }
        prometheus :9253
        }
    .:53 {
        errors
        cache 30
        reload
        loop
        bind 0.0.0.0
        forward . __PILLAR__UPSTREAM__SERVERS__
        prometheus :9253
        }

---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-local-dns
  namespace: kube-system
  labels:
    k8s-app: node-local-dns
spec:
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 10%
  selector:
    matchLabels:
      k8s-app: node-local-dns
  template:
    metadata:
      labels:
        k8s-app: node-local-dns
      annotations:
        policy.cilium.io/no-track-port: "53"
        prometheus.io/port: "9253"
        prometheus.io/scrape: "true"
    spec:
      priorityClassName: system-node-critical
      serviceAccountName: node-local-dns
      dnsPolicy: Default # Don't use cluster DNS.
      tolerations:
        - key: "CriticalAddonsOnly"
          operator: "Exists"
        - effect: "NoExecute"
          operator: "Exists"
        - effect: "NoSchedule"
          operator: "Exists"
      containers:
        - name: node-cache
          image: registry.k8s.io/dns/k8s-dns-node-cache:1.15.16
          resources:
            requests:
              cpu: 25m
              memory: 5Mi
          args: ["-localip", "169.254.20.10,10.96.0.10", "-conf", "/etc/Corefile", "-upstreamsvc", "kube-dns-upstream", "-skipteardown=true", "-setupinterface=false", "-setupiptables=false"]
          ports:
            - containerPort: 53
              name: dns
              protocol: UDP
            - containerPort: 53
              name: dns-tcp
              protocol: TCP
            - containerPort: 9253
              name: metrics
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 60
            timeoutSeconds: 5
          volumeMounts:
            - name: config-volume
              mountPath: /etc/coredns
            - name: kube-dns-config
              mountPath: /etc/kube-dns
      volumes:
        - name: kube-dns-config
          configMap:
            name: kube-dns
            optional: true
        - name: config-volume
          configMap:
            name: node-local-dns
            items:
              - key: Corefile
                path: Corefile.base
```
Let's summarize some of the core components:
- The ServiceAccount named `node-local-dns` will be used by the DaemonSet to run with specific permissions in the cluster.
- The Service named `kube-dns-upstream` listens on port 53 for DNS queries over both UDP and TCP and forwards them to a target port on the DNS pods selected by the `k8s-app: kube-dns` label (the `coreDNS` pods). This service is used to handle upstream DNS queries when the node-local DNS cache is not able to resolve a request.
- The ConfigMap defines the CoreDNS configuration (Corefile) for DNS caching on the node level.
    - It defines behavior for DNS requests in the `cluster.local` domain (Kubernetes service discovery) and external DNS queries (outside `cluster.local`).
    - Caching is configured with success and denial limits for both cluster.local and reverse lookup zones (in-addr.arpa and ip6.arpa)
    - Metrics are exposed on port 9253 for Prometheus, and health checks are enabled.
- The DaemonSet named `node-local-dns` runs a node-cache container on every node in the cluster.
    - It ensures that each node has a local DNS caching service, which significantly reduces DNS lookup times for workloads on the node.
    - The `dnsPolicy: Default` ensures that the DaemonSet doesn't use the cluster's default DNS settings but instead relies on its own DNS configuration.
    - The `args` define the local IP addresses for the cache (`169.254.20.10` and `10.96.0.10`) which are the node-local IPs used by this service.
    - It exposes ports 53 for DNS (both TCP and UDP) and port 9253 for metrics.
    - Health checks are configured via the `/health` endpoint.
Deploy the NodeLocal Daemonset and associated components:
```shell
kubectl apply -f node-local-dns.yaml
```

# LRP for NodeLocal DNS cache
Review the Local Redirect Policy:
```yaml
# yq node-local-dns-lrp.yaml
---
apiVersion: "cilium.io/v2"
kind: CiliumLocalRedirectPolicy
metadata:
  name: "nodelocaldns"
  namespace: kube-system
spec:
  redirectFrontend:
    serviceMatcher:
      serviceName: kube-dns
      namespace: kube-system
  redirectBackend:
    localEndpointSelector:
      matchLabels:
        k8s-app: node-local-dns
    toPorts:
      - port: "53"
        name: dns
        protocol: UDP
      - port: "53"
        name: dns-tcp
        protocol: TCP
```
It will redirect traffic bound to the CoreDNS servers and will instead forward them to the node-local DNS cache agents. Let's now deploy the policy:
```shell
kubectl apply -f node-local-dns-lrp.yaml
```
DNS traffic will now go to the local node-cache first. You can verify by checking the DNS cache’s metrics. Let's make DNS requests from the Obi Wan pod (on the first worker node), and check the DNS request counts on both the local DNS cache and the one running on the other worker node. First, let's get the IP of the DNS cache pod on the first worker node (local to Obi Wan):
```shell
DNS_LOCAL=$(kubectl -n kube-system get po -l k8s-app=node-local-dns --field-selector spec.nodeName=kind-worker -o jsonpath='{.items[].status.podIP}')
echo $DNS_LOCAL # 10.244.1.186
```
Let's also get the IP of the 'non-local' DNS entry:
```shell
DNS_NON_LOCAL=$(kubectl -n kube-system get po -l k8s-app=node-local-dns --field-selector spec.nodeName=kind-worker2 -o jsonpath='{.items[].status.podIP}')
echo $DNS_NON_LOCAL # 10.244.2.138
```
Let's check the metrics on both DNS entries. Let's run a few DNS commands from our client pod:
```shell
echo "## Stats on Local DNS Cache ##"
kubectl -n landspeeder exec obi-wan -- curl $DNS_LOCAL:9253/metrics | grep coredns_dns_request_count_total
echo "## Stats on Non-Local DNS Cache ##"
kubectl -n landspeeder exec obi-wan -- curl $DNS_NON_LOCAL:9253/metrics | grep coredns_dns_request_count_total
```
Let's run a few DNS requests:
```shell
kubectl -n landspeeder exec obi-wan -- getent hosts swapi.dev
kubectl -n landspeeder exec obi-wan -- getent hosts isovalent.com
kubectl -n landspeeder exec obi-wan -- getent hosts cilium.io
```
Now verify that they only increase on the local one:
```shell
echo "## Stats on Local DNS Cache ##"
kubectl -n landspeeder exec obi-wan -- curl $DNS_LOCAL:9253/metrics | grep coredns_dns_request_count_total
echo "## Stats on Non-Local DNS Cache ##"
kubectl -n landspeeder exec obi-wan -- curl $DNS_NON_LOCAL:9253/metrics | grep coredns_dns_request_count_total
```
Expect to see the number of DNS requests stats increasing on the node local DNS (13 in the output below) and stats:
```shell
## Stats on Local DNS Cache ##
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 38399    0 38399    0     0  17.6M      0 --:--# HELP coredns_dns_request_count_total Counter of DNS requests made per zone, protocol and family.
# TYPE coredns_dns_request_count_total counter
coredns_dns_request_count_total{family="1",proto="udp",server="dns://0.0.0.0:53",zone="."} 37
coredns_dns_request_count_total{family="1",proto="udp",server="dns://0.0.0.0:53",zone="cluster.local."} 49
coredns_dns_request_count_total{family="1",proto="udp",server="dns://0.0.0.0:53",zone="in-addr.arpa."} 10
:-- --:--:-- --:--:-- 18.3M
coredns_dns_request_count_total{family="1",proto="udp",server="dns://0.0.0.0:53",zone="ip6.arpa."} 10
## Stats on Non-Local DNS Cache ##
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0# HELP coredns_dns_request_count_total Counter of DNS requests made per zone, protocol and family.
# TYPE coredns_dns_request_count_total counter
coredns_dns_request_count_total{family="1",proto="udp",server="dns://0.0.0.0:53",zone="."} 1
coredns_dns_request_count_total{family="1",proto="udp",server="dns://0.0.0.0:53",zone="cluster.local."} 1
coredns_dns_request_count_total{family="1",proto="udp",server="dns://0.0.0.0:53",zone="in-addr.arpa."} 10
coredns_dns_request_count_total{family="1",proto="udp",server="dns://0.0.0.0:53",zone="ip6.arpa."} 10
```