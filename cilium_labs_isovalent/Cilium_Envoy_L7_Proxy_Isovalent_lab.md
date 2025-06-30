# Separate DaemonSet for Envoy
In Cilium 1.14, we are introducing support for Envoy as a DaemonSet. This provides a number of potential benefits, such as:
- Cilium Agent restarts (for example, for upgrades) do not impact the live traffic proxied via Envoy.
- Envoy patch release upgrades do not impact the Cilium Agent.
- Reduced blast radius in the (unlikely) event of a compromise
- Envoy application log isn’t mixed with the log of the Cilium Agent.
- Dedicated health probes for the Envoy proxy.
Cilium uses Envoy for multiple functionalities, and a good number has been added for Ingress and Gateway API support. However, Envoy is a very feature-rich software, and while Cilium provides several abstractions for Envoy configurations, it only covers some of the conceivable scenarios. For this reason, Cilium provides a Cilium Envoy Config CRDs, which lets users configure Envoy themselves to implement the features they want.

Let's have a look at our lab environment and see if Cilium has been installed correctly. The following command will wait for Cilium to be up and running and report its status:
```shell
cilium status --wait
```
During the lab deployment, Cilium was installed using helm and the following flags:
```shell
--set kubeProxyReplacement=true
```
KubeProxyReplacement (KPR) is a requirement for some of the features in this lab. Verify that Cilium was enabled and deployed with KPR:
```shell
cilium config view | grep -w "kube-proxy"
```
This will show that Cilium uses the Kube Proxy Replacement mode. In the next challenge, we will explore how Envoy allows to extend Cilium Network Policies all the way to layer 7 traffic.

# Securing traffic at L7
Standard Kubernetes Network Policies allow to filter traffic at layer 3 and layer 4:
- Layer 3 allows identities (typically IP addresses) to communicate regardless of the ports they use and the content they exchange.
- Layer 4 adds filtering on port (e.g. TCP/80) but still doesn't filter the content.
Cilium leverages the Envoy L7 proxy to allow filtering traffic at layer 7. This means you can filter the type of information applications are sharing, such as the HTTP path, headers, etc.

# Demo App
Deploy the demo app:
```shell
kubectl apply -f sw-pods.yaml
```
```yml
# sw-pods.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: deathstar
  labels:
    app.kubernetes.io/name: deathstar
spec:
  type: ClusterIP
  ports:
  - port: 80
  selector:
    org: empire
    class: deathstar
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deathstar
  labels:
    app.kubernetes.io/name: deathstar
spec:
  replicas: 2
  selector:
    matchLabels:
      org: empire
      class: deathstar
  template:
    metadata:
      labels:
        org: empire
        class: deathstar
        app.kubernetes.io/name: deathstar
    spec:
      containers:
      - name: deathstar
        # renovate: datasource=docker depName=quay.io/cilium/starwars
        image: quay.io/cilium/starwars@sha256:896dc536ec505778c03efedb73c3b7b83c8de11e74264c8c35291ff6d5fe8ada # v2.3
---
apiVersion: v1
kind: Pod
metadata:
  name: tiefighter
  labels:
    org: empire
    class: tiefighter
    app.kubernetes.io/name: tiefighter
spec:
  containers:
  - name: spaceship
    image: quay.io/cilium/json-mock:v1.3.8@sha256:5aad04835eda9025fe4561ad31be77fd55309af8158ca8663a72f6abb78c2603
---
apiVersion: v1
kind: Pod
metadata:
  name: xwing
  labels:
    app.kubernetes.io/name: xwing
    org: alliance
    class: xwing
spec:
  containers:
  - name: spaceship
    image: quay.io/cilium/json-mock:v1.3.8@sha256:5aad04835eda9025fe4561ad31be77fd55309af8158ca8663a72f6abb78c2603
```
This is a Star Wars themed demo which includes 4 pods:
- 2 `deathstar` pods
- 1 `tiefighter` pod
- 1 `xwing` pod
Wait for the Death Star to be deployed:
```shell
kubectl rollout status deployment/deathstar
```

# Test Death Star access
Test access to the Death Star from the X-Wing pod:
```shell
kubectl exec xwing -- \
  curl --max-time 1 -s -X POST deathstar.default.svc.cluster.local/v1/request-landing
```
The Death Star service will respond with `Ship landed`. Using the Hubble CLI, check the traffic that results from this request:
```shell
hubble observe --to-pod default/deathstar
```
Notice that it says that traffic was forwarded to the endpoint (i.e. a pod managed by Cilium, in Cilium terms):
```shell
Jun 30 20:06:51.306: default/xwing:41674 (ID:9770) -> default/deathstar-67c5c5c88-2qdws:80 (ID:57823) to-endpoint FORWARDED (TCP Flags: ACK, PSH)
Jun 30 20:06:51.307: default/xwing:41674 (ID:9770) -> default/deathstar-67c5c5c88-2qdws:80 (ID:57823) to-endpoint FORWARDED (TCP Flags: ACK, FIN)
Jun 30 20:06:51.307: default/xwing:41674 (ID:9770) -> default/deathstar-67c5c5c88-2qdws:80 (ID:57823) to-endpoint FORWARDED (TCP Flags: ACK)
```

# L3/L4 Network Policy
The X-Wing has access to the Death Star, which is problematic for the Death Star security team: You wouldn't want the Rebels to get too close to the Death Star, they might blow it up! In order to fix this, deploy a L3/L4 Network Policy to protect the Death Star by only allowing vessels labeled with org=empire to access it (which is not the case of the xwing pod).
```shell
kubectl apply -f l4-policy.yaml
```
```yml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
spec:
  description: "L3-L4 policy to restrict deathstar access to empire ships only"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
  - fromEndpoints:
    - matchLabels:
        org: empire
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
```
The `endpointSelector` field show that the network policy allows ingress traffic to pods labeled `org=empire` and `class=deathstar`, which matches the Death Star pods:
- from pods labeled `org=empire` (`fromEndpoints.matchLabels` field) for L3 filtering
- to port 80/TCP (`toPorts` field) for L4 filtering
This level of security is ensured by eBPF, directly in the kernel. This means even when the Cilium agent is not running, these rules are guaranteed to be applied (but not necessarily kept up-to-date). Try again to access the Death Star from the X-Wing pod:
```shell
kubectl exec xwing -- \
  curl --max-time 1 -s -X POST deathstar.default.svc.cluster.local/v1/request-landing
```
The requests are now blocked by the L4 Network Policy, and the packets are dropped directly by the kernel, causing a timeout. It is however still possible to access from the Tie Fighter pod, since it is labeled as `org=empire`:
```shell
kubectl exec tiefighter -- \
  curl --max-time 1 -s -X POST deathstar.default.svc.cluster.local/v1/request-landing
```

# The Death Star explodes!
While the L4 Network Policy prevents Rebel ships from accessing the Death Star, it leaves complete access to Imperial ships. What would happen if a Rebel managed to get access to an imperial ship. They could take advantage of their access rights to the Death Star to blow it up! Let's try this from a Tie Fighter pod. In the Terminal tab:
```shell
kubectl exec tiefighter -- \
  curl -s -X PUT deathstar.default.svc.cluster.local/v1/exhaust-port
```
The Death Star exploded! We need a more fine-grained Network Policy, only allowing a POST request to /v1/request-landing and nothing else. This can be achieved with L7 Network Policies, by leveraging Envoy integration in Cilium.

# L7 Network Policy
Let's modify the Network Policy to add a L7 rule:
```shell
kubectl apply -f l7-policy.yaml
```
Review the policy's spec:
```yml
# kubectl get cnp rule1 -o yaml | yq .spec
description: L7 policy to restrict access to specific HTTP call
endpointSelector:
  matchLabels:
    class: deathstar
    org: empire
ingress:
  - fromEndpoints:
      - matchLabels:
          org: empire
    toPorts:
      - ports:
          - port: "80"
            protocol: TCP
        rules:
          http:
            - method: POST
              path: /v1/request-landing
```
Compared with the previous version, a `rules` field was added to the L4 section of the policy:
```yml
        rules:
          http:
            - method: POST
              path: /v1/request-landing
```
It states that only HTTP traffic is allowed, and only if the request uses the `POST` method and targets the `/v1/request-landing` path. Layer 7 network policies are implemented using Envoy as a L7 proxy. Cilium dynamically programs Envoy to apply these security rules. If a network request matches the L3/L4 section of the policy, the eBPF programs in the kernel will allow it and send the traffic to the Envoy proxy, targeting a listener that is generated specifically for the L7 rule in the policy. Envoy is then responsible for replying to the request. Try again to explode the Death Star from the Tie Fighter pod:
```shell
kubectl exec tiefighter -- \
  curl -s -X PUT deathstar.default.svc.cluster.local/v1/exhaust-port
```
You will see Access denied, which corresponds to a 403 response from Envoy. Check the Hubble logs:
```shell
hubble observe \
  --from-pod default/tiefighter \
  --to-pod default/deathstar
```
You will see output like this:
```shell
Jun 30 20:15:59.586: default/tiefighter:57694 (ID:35210) -> default/deathstar-67c5c5c88-7wf4m:80 (ID:57823) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN)
Jun 30 20:15:59.587: default/tiefighter:57694 (ID:35210) -> default/deathstar-67c5c5c88-7wf4m:80 (ID:57823) to-proxy FORWARDED (TCP Flags: ACK, PSH)
Jun 30 20:15:59.588: default/tiefighter:57694 (ID:35210) -> default/deathstar-67c5c5c88-7wf4m:80 (ID:57823) to-proxy FORWARDED (TCP Flags: ACK, FIN)
Jun 30 20:15:59.588: default/tiefighter:57694 (ID:35210) -> default/deathstar-67c5c5c88-7wf4m:80 (ID:57823) http-request DROPPED (HTTP/1.1 PUT http://deathstar.default.svc.cluster.local/v1/exhaust-port)
```
Observe the various steps of the connection between the Tie Fighter and the Death Star:
- a `policy-verdict:L3-L4` trace marked as `INGRESS ALLOWED` when the in-kernel eBPF program allows the connection based on the L3/L4 Network Policy rule
- `to-proxy` traces marked as `FORWARDED`, which correspond to the traffic sent to the Envoy proxy (with SYN, ACK, and ACK, PSH traces)
- an `http-request` trace marked as `DROPPED` when Envoy responds to the request with an Access denied response, with details of the HTTP request that was blocked.
- finally, another `to-proxy` trace for the `ACK`, `FIN` step terminating the TCP connection
Now verify that the policy still allows the Tie Fighter to land on the Death Star:
```shell
kubectl exec tiefighter -- \
  curl -s -X POST deathstar.default.svc.cluster.local/v1/request-landing
```

# Separate DaemonSet
In Terminal 1, make the Tie Fighter request landing in a loop:
```shell
while [ 1 ]; do
  kubectl exec tiefighter -- \
    curl -s --max-time 1 -X POST deathstar.default.svc.cluster.local/v1/request-landing
  sleep 1
done
```
In Terminal 2, check a Cilium pod:
```shell
NODE=$(kubectl get po -l class=deathstar -o jsonpath='{.items[].spec.nodeName}')
echo $NODE
CILIUM=$(kubectl -n kube-system get po -l k8s-app=cilium --field-selector spec.nodeName=$NODE -o name)
kubectl -n kube-system exec $CILIUM -c cilium-agent -- \
  ps axu | grep envoy       # kind-worker2
```
There's no Envoy process running in the Cilium pod. Instead, there is a dedicated DaemonSet which deployed one Envoy pod per node:
```shell
kubectl -n kube-system get po -l k8s-app=cilium-envoy
```
This allows Envoy workers to be managed separately from the Cilium agents. In particular, it means operators can now tune resources for both Cilium and Envoy separately, as the pod privileges are also better tuned. Besides that, the Envoy logs are now accessible in the Envoy pods directly instead of the Cilium pods:
```shell
kubectl -n kube-system logs daemonsets/cilium-envoy
```
In the next challenge, we will explore the observability benefits of Envoy usage in Cilium.

# Proxied traffic
Using Hubble, check the traffic that is routed through the Envoy proxy:
```shell
hubble observe --type trace:to-proxy
```
Let's extract all the flow information based on the protocol (e.g. HTTP) and source pod (e.g. the Tie Fighter), then export the result with the JSON output option, and finally filter with `jq` to only see the `.flow.l7` field. This will show us the specific details parsed from the L7 traffic, such as the method and headers:
```shell
hubble observe --protocol http --from-pod default/tiefighter -o jsonpb | \
  head -n 1 | jq '.flow.l7'
```
Observe the details of the flow, in particular the envoy-specific headers added to the request:
- X-Envoy-Internal
- X-Request-Id
Then, look for replies:
```shell
hubble observe --protocol http --to-pod default/tiefighter -o jsonpb | \
  head -n 1 | jq '.flow.l7'
```
Observe the envoy headers:
- X-Envoy-Upstream-Service-Time
- X-Request-Id
All these flows are ingress flows, as you can see by filtering for HTTP flows in the egress direction, which should return nothing:
```shell
hubble observe --protocol http --traffic-direction egress
```
Let's use the X-Request-Id to match a request and its response. First, we'll need to make sure egress traffic from the Tie Fighter is captured by Envoy, so we'll need a L7 CNP for that. If we apply an egress CNP though, this will disrupt DNS requests, which are also egress traffic, so we need to add a DNS policy as well:
```shell
kubectl apply -f policies/dns.yaml -f policies/tiefighter.yaml
```
```yml
# policies/dns.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  namespace: default
  name: dns
spec:
  endpointSelector: {}
  egress:
    - toEndpoints:
        - matchLabels:
            io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*"
# policies/tiefighter.yaml
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: tiefighter
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      org: empire
      class: tiefighter
  egress:
    - toEndpoints:
        - matchLabels:
            class: deathstar
            org: empire
      toPorts:
        - ports:
            - port: "80"
              protocol: TCP
          rules:
            http:
            - method: POST
              path: /v1/request-landing
```
Now that this is applied, check for egress traffic again:
```shell
hubble observe --protocol http --traffic-direction egress
```
You can now see egress requests from the Tie Fighter being forwarded to the Death Star, as well as the responses from the Death Star:
```shell
Jun 30 20:27:43.349: default/tiefighter:39652 (ID:35210) <- default/deathstar-67c5c5c88-2qdws:80 (ID:57823) http-response FORWARDED (HTTP/1.1 200 0ms (POST http://deathstar.default.svc.cluster.local/v1/request-landing))
Jun 30 20:27:44.468: default/tiefighter:39662 (ID:35210) -> default/deathstar-67c5c5c88-2qdws:80 (ID:57823) http-request FORWARDED (HTTP/1.1 POST http://deathstar.default.svc.cluster.local/v1/request-landing)
Jun 30 20:27:44.469: default/tiefighter:39662 (ID:35210) <- default/deathstar-67c5c5c88-2qdws:80 (ID:57823) http-response FORWARDED (HTTP/1.1 200 0ms (POST http://deathstar.default.svc.cluster.local/v1/request-landing))
```
Now, let's match request IDs! Run the following command to record some Hubble HTTP flows and save them to a file: 
```shell
hubble observe --namespace default --protocol http -o jsonpb > flows.json
```
Find the first EGRESS flow in the file and get its ID:
```shell
REQUEST_ID=$(cat flows.json | jq -r '.flow | select(.source.labels[0]=="k8s:app.kubernetes.io/name=tiefighter" and .traffic_direction=="EGRESS") .l7.http.headers[] | select(.key=="X-Request-Id") .value' | head -n1)
echo $REQUEST_ID        # 796ea005-52b6-49d9-aaa9-f795e01e607f
```
Then find all flows with this request ID in the file and display their source identities:
```shell
cat flows.json | \
  jq 'select(.flow.l7.http.headers[] | .value == "'$REQUEST_ID'") .flow | {src_label: .source.labels[0], dst_label: .destination.labels[0], traffic_direction, type: .l7.type, time}'
```
You will see 4 sources:
- an `egress` flow from the `tiefighter` to the `deathstar`, corresponding to the original request
- the `ingress` flow for the same request, being forwarded from the proxy to the Death Star
- another `egress` flow for the response, from the `deathstar` to the `tiefighter`
- the corresponding `ingress` flow from the `deathstar` pod to the `tiefighter`
They might not appear in the same order depending on how flows were received from the Hubble relay servers on the different nodes.

# The problem of gRPC Load-Balancing
gRPC is a popular framework to build scalable and fast APIs. It is used extensively in cloud native development and micro-services architecture. gRPC-based applications are very commonly deployed in Kubernetes clusters, albeit with some restrictions. Kubernetes does not natively support gRPC Load Balancing out-of-the-box and therefore an additional tool —such as a proxy or a service mesh— is required to perform it. That is because the load-balancing decision has to be done at Layer 7, and not at Layer 3/4, and Kubernetes Services do not support L7 Load-Balancing. Users want a simpler solution to achieve L7 load-balancing: something as simple as applying an annotation to a Service. Since Cilium 1.13, you can use Cilium’s Envoy proxy to achieve load-balancing for L7 services, with a simple annotation on the Kubernetes Service. Let's explore how in this challenge!

# Deploy an application
For this demo we will use [GCP's microservices demo app](https://github.com/GoogleCloudPlatform/microservices-demo). Some of the micro-services used in this app leverage gRPC for service communications. Create a `grpc` namespace:
```shell
kubectl create namespace grpc
```
Install the app:
```shell
kubectl -n grpc apply -f /opt/gcp-microservices-demo.yml
```
The Pods should be up and running in less than a minute. Wait until the currently service is ready:
```shell
kubectl -n grpc rollout status deploy/currencyservice
```
Two of the micro-services (Product Catalog and Currency) are accessible over gRPC and fronted by a ClusterIP Service. We will use the Current service for out tests.

# Make gRPC requests to backend services
Let's try to access the currency service of the application, which lists the currencies the shopping app supports. Access to the service will be done over gRPC, using grpcurl (the equivalent of curl but for gRPC). We're going to access it, from another Pod. Deploy the Pod:
```shell
kubectl -n grpc apply -f pod.yaml
```
After 15-20 seconds, it should be ready. Check with:
```shell
kubectl -n grpc get -f pod.yaml
```
Let's now install grpcurl to access the service over gRPC. First, let's enter the shell of the Pod.
```shell
kubectl -n grpc exec -ti pod-worker -- zsh
```
Once in the shell of the client, run the following command to install `grpcurl`:
```shell
curl -sSL "https://github.com/fullstorydev/grpcurl/releases/download/v1.8.6/grpcurl_1.8.6_linux_x86_64.tar.gz" | tar -xz -C /usr/local/bin
```
Since gRPC is binary-encoded, you also need the proto definitions for the gRPC services in order to make gRPC requests. Download this for the demo app:
```shell
curl -o demo.proto https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/main/protos/demo.proto
```
Let's try accessing the currency service with:
```shell
grpcurl -v -plaintext -proto ./demo.proto \
  currencyservice:7000 \
  hipstershop.CurrencyService/GetSupportedCurrencies
```
This should be successful and you should see a list of currency codes (EUR, USD, JPY, etc...). Note the Response header:
```shell
content-type: application/grpc+proto
date: Mon, 30 Jun 2025 20:35:15 GMT
grpc-accept-encoding: identity,deflate,gzip
```
This is the default behaviour. Access is successful but as there's no L7 Load-Balancing capability yet, no L7 load-balancing is possible. Let's observe traffic with Hubble, the network and security visibility tool. In Terminal 2, run the following command to look at the traffic from the `pod-worker`:
```shell
hubble observe --from-pod grpc/pod-worker
```
If you filter based on traffic sent to the Envoy proxy, no flow should appear:
```shell
hubble observe -n grpc --type trace:to-proxy
```

# Enable Cilium Envoy Configs
In order to use L7 load balancing in the cluster, we first need to enable the CiliumEnvoyConfig CRD in the cluster, as this CRD will be used by Cilium to implement the L7 load-balancer using Envoy. Enabling this can be done by upgrading the Helm chart in the Terminal 2 tab:
```shell
cilium upgrade \
  --version 1.17.4 \
  --reuse-values \
  --set loadBalancer.l7.backend=envoy
```
Restart the Cilium Operator so it deploys the new CRD, and the Cilium agent so it is ready to honor the L7 annotations on Services:
```shell
kubectl -n kube-system rollout restart deployment cilium-operator
kubectl -n kube-system rollout restart daemonset cilium
kubectl -n kube-system rollout restart daemonset cilium-envoy
```
Wait for Cilium to be re-deployed and ready:
```shell
cilium status --wait
```
Verify that the Cilium Envoy Configs are activated:
```shell
cilium config view | grep envoy-config
```

# Enabling L7 Load Balancing
Let's now enable L7 Load-Balancing for these gRPC services. Simply annotating Services with these labels will enable this functionality. In Terminal 2:
```shell
kubectl -n grpc \
  annotate svc/currencyservice \
  "service.cilium.io/lb-l7=enabled"
```
Look at the CiliumEnvoyConfig CRD created (note you may need to run this command a couple of times):
```shell
kubectl -n grpc get cec
```
There should be a newly created CECs:
```shell
NAME                              AGE
cilium-envoy-lb-currencyservice   28s
```
Inspect the spec for the generated CEC:
```shell
kubectl -n grpc get cec cilium-envoy-lb-currencyservice -o yaml | yq .spec
```
The `services` section at the bottom shows that it applies to the `currencyservice` service in the `grpc` namespace. There are 3 Envoy resources listed:
- a `type.googleapis.com/envoy.config.listener.v3.Listener` resource to setup a gRPC listener in Envoy
- a `type.googleapis.com/envoy.config.route.v3.RouteConfiguration` resource which targets the `grpc/currentservice` service 100% of the time
- a `type.googleapis.com/envoy.config.cluster.v3.Cluster` resource which configures the behavior of the L7 load-balancer.

# Test L7 Load-Balancer
grpcurl -v -plaintext -proto ./demo.proto \
  currencyservice:7000 \
  hipstershop.CurrencyService/GetSupportedCurrencies
```
Notice the change in the Response headers replies:
```shell
content-type: application/grpc+proto
date: Mon, 30 Jun 2025 20:43:47 GMT
grpc-accept-encoding: identity,deflate,gzip
server: envoy
x-envoy-upstream-service-time: 2
```
Access is still successful and traffic has been forwarded to Envoy which is now handling L7 traffic to the destination pods and providing load-balancing.

# Fine-tuning L7 LB
Once Envoy is used as an internal load-balancer, you can fine-tune its behavior using the `service.cilium.io/lb-l7-algorithm` annotation. This allows to choose the type of Envoy load-balancer to use. In the Terminal 2 tab, add that annotation to the Current Service:
```shell
kubectl -n grpc \
  annotate svc/currencyservice \
  "service.cilium.io/lb-l7-algorithm=least_request"
```
The `type.googleapis.com/envoy.config.cluster.v3.Cluster` resource is now configured with `lbPolicy: LEAST_REQUEST`, so it will make use of the least request Envoy load-balancer. In the next challenge, we will explore an advanced use case for Cilium Envoy Configs.

# L7 Network Policy Enforcement with Envoy
As previously seen, when interpreting a Network Policy that contains both L3/L4 and L7 rules, Cilium creates eBPF programs to enforce at L3/L4, as well as Envoy listeners to implement the L7 rules. L3/L4 rules then forward accepted traffic to Envoy for L7 enforcement, and Envoy either forwards traffic to its destination or replies with a 403 (Access denied) response. For advanced situations, It is possible to specify an **Envoy listener** as a Cilium Network Policy parameter to fully control how Envoy behaves once the L3/L4 enforcement level redirects traffic to it. In this challenge, we will make use of the Network Policy listener parameter to set up an Envoy listener sending traffic to an external L7 proxy.

# Deploy an application
```shell
kubectl create ns proxy
kubectl -n proxy run --image nicolaka/netshoot \
  proxy-client \
  --command sleep infinite
kubectl -n proxy get po proxy-client
```
Then make a request to api.github.com:
```shell
kubectl -n proxy exec proxy-client -- \
  curl -s https://api.github.com/status | jq
```
Check the logs in Hubble:
```shell
hubble observe --from-pod proxy/proxy-client
```
You will see the traffic going from the client to a world identity, represented by its IP address:
```shell
Jun 30 20:50:42.448: proxy/proxy-client:55922 (ID:35118) -> 140.82.121.6:443 (world) to-stack FORWARDED (TCP Flags: SYN)
```
The trace uses the to-stack viewpoint, not to-proxy, as the traffic does not —yet— go through Envoy. 

# The Web Proxy
Let's set up a web proxy on the lab's VM. We will use 3proxy as a Docker container running in the kind network, so it's accessible from the Kubernetes cluster:
![Alt text](https://play.instruqt.com/assets/tracks/zqukyleyypoe/d2e3f524ae86710dda530691dd1779bd/assets/cnp-listener-diagram.png)
Deploy the proxy:
```shell
docker run --rm -d \
  --net kind \
  --name proxy-server \
  -e "3128/tcp" \
  ghcr.io/tarampampam/3proxy:latest
```
This proxy listens on ports 3128/TCP for HTTP traffic. Wait for the container to start:
```shell
docker inspect --format '{{.State.Status}}' proxy-server
```
nspect the Docker container to get its IP address:
```shell
docker inspect proxy-server -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

# Envoy Config
Next, deploy an Envoy Config to set up the Envoy listener we will be using. Using the Editor, open the file named `proxy-cec.yaml` and edit line 29 to replace `<<PROXY_IP>>` with the IP you just retrieved (which is usually `172.18.0.5`). Notice that this is a CEC (so it's namespaced) made of 2 resources:
- `type.googleapis.com/envoy.config.listener.v3.Listener` to set up a listener. This listener uses a TCP proxy, sending traffic to the downstream address via the Envoy dynamic variable `%DOWNSTREAM_LOCAL_ADDRESS%`
- `type.googleapis.com/envoy.config.cluster.v3.Cluster` which configures the behavior to send the traffic to the web proxy we just set up via its IP and the HTTP port `3128`.
Once the file is saved, apply it in the Terminal:
```shell
kubectl -n proxy apply -f proxy-cec.yaml
```
Now, we need to deploy a Network Policy to force traffic into the Envoy Proxy if it passes an L3/L4 filter:
```shell
kubectl -n proxy apply -f proxy-cnp.yaml
```
```yml
# proxy-cnp.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: proxy-policy
spec:
  endpointSelector: {}
  egress:
    - toEntities:
        - cluster
    - toEndpoints:
        - matchLabels:
            io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*"
    - toFQDNs:
        - matchPattern: "*.github.com"
      toPorts:
        - ports:
            - port: "80"
              protocol: TCP
            - port: "443"
              protocol: TCP
          listener:
            envoyConfig:
              kind: "CiliumEnvoyConfig"
              name: "proxy-envoy"
            name: "proxy-listener"
```
This is an egress policy that:
- allows all requests inside the cluster without filtering (`toEntities: [cluster]`)
- allow DNS requests to `kube-system/kube-dns` on port 53/UDP, and filters DNS request through the Cilium proxy
- allows HTTP (80/TCP) and HTTPS (443/TCP) traffic to DNS names matching `*.github.com`, and redirects this traffic through the `proxy-envoy` Envoy listener we previously deployed
Try the request again:
```shell
kubectl -n proxy exec proxy-client -- \
  curl -s https://api.github.com/status | jq
```
It should succeed just the same. The DNS name "api.github.com" is now known because we're using the DNS proxy in the Network Policy, and you can see the request being first vetted against the L3/L4 network policy (policy-verdict:L3-L4 EGRESS ALLOWED), then redirected to the Envoy proxy (to-proxy FORWARDED). Check the proxy logs:
```shell
docker logs proxy-server
```
You will see that the request went through the proxy, and was sent to the IP address you saw earlier in the Hubble logs:
```json
{"time_unix":1751317226, "proxy":{"type:":"PROXY", "port":3128}, "error":{"code":"00000"}, "auth":{"user":"-"}, "client":{"ip":"172.18.0.2", "port":59028}, "server":{"ip":"140.82.121.6", "port":443}, "bytes":{"sent":1863, "received":4776}, "request":{"hostname":"140.82.121.6"}, "message":"CONNECT 140.82.121.6:443 HTTP/1.1"}
```
The client IP is the IP address of the node on which the client pod is running (since it is used to source NAT the traffic when exiting the Kubernetes cluster in VXLAN mode).

# Envoy Admin UI
Starting with Cilium 1.16, it is now easy to access the Envoy Admin UI in Cilium. Upgrade the chart to enabled the Envoy Admin UI:
```shell
cilium upgrade \
  --version 1.17.4 \
  --reuse-values \
  --set envoy.debug.admin.enabled=true \
  --set envoy.debug.admin.port=9901
```
This will configure Envoy on each node to serve its admin UI on port 9901. We will then be able to access it either on the host port or via port forwarding. For practical reasons, we will use that second option in this lab.

# Exposing Admin UI
Let's inspect the Envoy proxy on the node where the proxy-client pod we previously deployed is running. First, retrieve the node name for that pod:
```shell
CLIENT_NODE=$(kubectl -n proxy get po proxy-client -o jsonpath='{.spec.nodeName}')
echo $CLIENT_NODE       # kind-worker
```
Wait for Cilium to be ready again:
```shell
cilium status --wait
```
Next, get the name of the Envoy pod running on that node:
```shell
ENVOY_POD=$(kubectl -n kube-system get po -l k8s-app=cilium-envoy --field-selector spec.nodeName=$CLIENT_NODE -o name)
echo $ENVOY_POD     # pod/cilium-envoy-7b86h
```
Finally, port-forward the 9901 port to localhost for that pod. Expose it to 0.0.0.0 so we can access it in the browser:
```shell
kubectl -n kube-system port-forward --address 0.0.0.0 $ENVOY_POD 9901
```
