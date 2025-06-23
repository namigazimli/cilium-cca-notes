# Welcome to the Gateway API lab
Before we can install Cilium with the Gateway API feature, there are a couple of important prerequisites to know:
- Cilium must be configured with `kubeProxyReplacement` set to true.
- CRD (Custom Resource Definition) from Gateway API **must** be installed beforehand.
As part of the lab deployment script, several CRDs were installed. Verify that they are available.
```shell
kubectl get crd \
  gatewayclasses.gateway.networking.k8s.io \
  gateways.gateway.networking.k8s.io \
  httproutes.gateway.networking.k8s.io \
  referencegrants.gateway.networking.k8s.io \
  tlsroutes.gateway.networking.k8s.io
```
During the lab deployment, Cilium was installed using the following command:
```shell
cilium install --version v1.17.1 \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set gatewayAPI.enabled=true \
```
Let's have a look at our lab environment and see if Cilium has been installed correctly. The following command will wait for Cilium to be up and running and report its status:
```shell
cilium status --wait
```
Verify that Cilium was enabled and deployed with the Gateway API feature:
```shell
cilium config view | grep -w "enable-gateway-api"
```

# What are a GatewayClass and a Gateway?
If the CRDs have been deployed beforehand, a GatewayClass will be deployed by Cilium during its installation (assuming the Gateway API option has been selected). Let's verify that a GatewayClass has been deployed and accepted:
```shell
kubectl get gatewayclasses
```
The GatewayClass is a type of Gateway that can be deployed: in other words, it is a template. This is done in a way to let infrastructure providers offer different types of Gateways. Users can then choose the Gateway they like. For instance, an infrastructure provider may create two `GatewayClasses` named `internet` and `private` to reflect Gateways that define Internet-facing vs private, internal applications. In our case, the Cilium Gateway API (`io.cilium/gateway-controller`) will be instantiated. This schema below represents the various components used by Gateway APIs. When using Ingress, all the functionalities were defined in one API. By deconstructing the ingress routing requirements into multiple APIs, users benefit from a more generic, flexible and role-oriented model.
![Alt text](https://play.instruqt.com/assets/tracks/ndhj358im0mf/6475cefe7240dd9335c2aed1dc6b8f10/assets/api-model.png)
The actual L7 traffic rules are defined in the `HTTPRoute` API. In the next challenge, you will deploy an application and set up GatewayAPI HTTPRoutes to route HTTP traffic into the cluster.

# We need a Load Balancer
The Cilium Service Mesh Gateway API Controller requires the ability to create `LoadBalancer` Kubernetes services. Since we are using Kind on a Virtual Machine, we do not benefit from an underlying Cloud Provider's load balancer integration. For this lab, we will use Cilium's own LoadBalancer [capabilities](https://docs.cilium.io/en/latest/network/l2-announcements/) to provide IP Address Management (IPAM) and Layer 2 announcement of IP addresses assigned to LoadBalancer services.

# Deploy an application
Let's deploy the sample application in the cluster.
```shell
kubectl apply -f /opt/bookinfo.yml
```
```yml
# Copyright Istio Authors
#
#   Licensed under the Apache License, Version 2.0 (the "License");
#   you may not use this file except in compliance with the License.
#   You may obtain a copy of the License at
#
#       http://www.apache.org/licenses/LICENSE-2.0
#
#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.

##################################################################################################
# This file defines the services, service accounts, and deployments for the Bookinfo sample.
#
# To apply all 4 Bookinfo services, their corresponding service accounts, and deployments:
#
#   kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
#
# Alternatively, you can deploy any resource separately:
#
#   kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml -l service=reviews # reviews Service
#   kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml -l account=reviews # reviews ServiceAccount
#   kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml -l app=reviews,version=v3 # reviews-v3 Deployment
##################################################################################################

##################################################################################################
# Details service
##################################################################################################
apiVersion: v1
kind: Service
metadata:
  name: details
  labels:
    app: details
    service: details
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: details
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookinfo-details
  labels:
    account: details
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: details-v1
  labels:
    app: details
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: details
      version: v1
  template:
    metadata:
      labels:
        app: details
        version: v1
    spec:
      serviceAccountName: bookinfo-details
      containers:
      - name: details
        image: docker.io/istio/examples-bookinfo-details-v1:1.16.2
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080
        securityContext:
          runAsUser: 1000
---
##################################################################################################
# Ratings service
##################################################################################################
apiVersion: v1
kind: Service
metadata:
  name: ratings
  labels:
    app: ratings
    service: ratings
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: ratings
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookinfo-ratings
  labels:
    account: ratings
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ratings-v1
  labels:
    app: ratings
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ratings
      version: v1
  template:
    metadata:
      labels:
        app: ratings
        version: v1
    spec:
      serviceAccountName: bookinfo-ratings
      containers:
      - name: ratings
        image: docker.io/istio/examples-bookinfo-ratings-v1:1.16.2
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080
        securityContext:
          runAsUser: 1000
---
##################################################################################################
# Reviews service
##################################################################################################
apiVersion: v1
kind: Service
metadata:
  name: reviews
  labels:
    app: reviews
    service: reviews
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: reviews
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookinfo-reviews
  labels:
    account: reviews
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reviews-v1
  labels:
    app: reviews
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: reviews
      version: v1
  template:
    metadata:
      labels:
        app: reviews
        version: v1
    spec:
      serviceAccountName: bookinfo-reviews
      containers:
      - name: reviews
        image: docker.io/istio/examples-bookinfo-reviews-v1:1.16.2
        imagePullPolicy: IfNotPresent
        env:
        - name: LOG_DIR
          value: "/tmp/logs"
        ports:
        - containerPort: 9080
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: wlp-output
          mountPath: /opt/ibm/wlp/output
        securityContext:
          runAsUser: 1000
      volumes:
      - name: wlp-output
        emptyDir: {}
      - name: tmp
        emptyDir: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reviews-v2
  labels:
    app: reviews
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: reviews
      version: v2
  template:
    metadata:
      labels:
        app: reviews
        version: v2
    spec:
      serviceAccountName: bookinfo-reviews
      containers:
      - name: reviews
        image: docker.io/istio/examples-bookinfo-reviews-v2:1.16.2
        imagePullPolicy: IfNotPresent
        env:
        - name: LOG_DIR
          value: "/tmp/logs"
        ports:
        - containerPort: 9080
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: wlp-output
          mountPath: /opt/ibm/wlp/output
        securityContext:
          runAsUser: 1000
      volumes:
      - name: wlp-output
        emptyDir: {}
      - name: tmp
        emptyDir: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reviews-v3
  labels:
    app: reviews
    version: v3
spec:
  replicas: 1
  selector:
    matchLabels:
      app: reviews
      version: v3
  template:
    metadata:
      labels:
        app: reviews
        version: v3
    spec:
      serviceAccountName: bookinfo-reviews
      containers:
      - name: reviews
        image: docker.io/istio/examples-bookinfo-reviews-v3:1.16.2
        imagePullPolicy: IfNotPresent
        env:
        - name: LOG_DIR
          value: "/tmp/logs"
        ports:
        - containerPort: 9080
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: wlp-output
          mountPath: /opt/ibm/wlp/output
        securityContext:
          runAsUser: 1000
      volumes:
      - name: wlp-output
        emptyDir: {}
      - name: tmp
        emptyDir: {}
---
##################################################################################################
# Productpage services
##################################################################################################
apiVersion: v1
kind: Service
metadata:
  name: productpage
  labels:
    app: productpage
    service: productpage
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: productpage
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookinfo-productpage
  labels:
    account: productpage
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: productpage-v1
  labels:
    app: productpage
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: productpage
      version: v1
  template:
    metadata:
      labels:
        app: productpage
        version: v1
    spec:
      serviceAccountName: bookinfo-productpage
      containers:
      - name: productpage
        image: docker.io/istio/examples-bookinfo-productpage-v1:1.16.2
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        securityContext:
          runAsUser: 1000
      volumes:
      - name: tmp
        emptyDir: {}
```
You can find more details about the Bookinfo application on the Istio [website](https://istio.io/latest/docs/examples/bookinfo/).
Check that the application is properly deployed:
```shell
kubectl get pods
```
You should see multiple pods being deployed in the default namespace. Wait until they are Running (should take 30 to 45 seconds). Notice that with Cilium Service Mesh there is no Envoy sidecar created alongside each of the demo app microservices. With a sidecar implementation the output would show 2/2 READY: one for the microservice and one for the Envoy sidecar. Have a quick look at the Services deployed:
```shell
kubectl get svc
```
Note these Services are only internal-facing (ClusterIP) and therefore there is no access from outside the cluster to these Services. 

# Deploy the Gateway
Let's deploy the Gateway with the following manifest:
```shell
kubectl apply -f basic-http.yaml
```
```yaml
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: cilium
  listeners:
  - protocol: HTTP
    port: 80
    name: web-gw
    allowedRoutes:
      namespaces:
        from: Same
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: http-app-1
spec:
  parentRefs:
  - name: my-gateway
    namespace: default
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /details
    backendRefs:
    - name: details
      port: 9080
  - matches:
    - headers:
      - type: Exact
        name: magic
        value: foo
      queryParams:
      - type: Exact
        name: great
        value: example
      path:
        type: PathPrefix
        value: /
      method: GET
    backendRefs:
    - name: productpage
      port: 9080
```
Review the configuration used by the Gateway:
```yml
spec:
  gatewayClassName: cilium
  listeners:
  - protocol: HTTP
    port: 80
    name: web-gw
    allowedRoutes:
      namespaces:
        from: Same
```
First, note in the `Gateway` section that the `gatewayClassName` field uses the value `cilium`. This refers to the Cilium `GatewayClass` previously configured. The Gateway will listen on port 80 for HTTP traffic coming southbound into the cluster. The `allowedRoutes` is here to specify the namespaces from which Routes may be attached to this Gateway. `Same` means only Routes in the same namespace may be used by this Gateway.
Note that, if we were to use `All` instead of `Same`, we would enable this gateway to be associated with routes in any namespace and it would enable us to use a single gateway across multiple namespaces that may be managed by different teams.

We could specify different namespaces in the HTTPRoutes – therefore, for example, you could send the traffic to `https://acme.com/payments` in a namespace where a payment app is deployed and `https://acme.com/ads` in a namespace used by the ads team for their application. Let's now review the HTTPRoute manifest. HTTPRoute is a GatewayAPI type for specifying routing behaviour of HTTP requests from a Gateway listener to a Kubernetes Service.It is made of Rules to direct the traffic based on your requirements. This first Rule is essentially a simple L7 proxy route: for HTTP traffic with a path starting with `/details`, forward the traffic over to the `details` Service over port 9080.
```yml
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /details
    backendRefs:
    - name: details
      port: 9080
```
The second rule is similar but it's leveraging different matching criteria. If the HTTP request has:
- a HTTP header with a name set to `magic` with a value of `foo`, **AND**
- the HTTP method is "`GET`", **AND**
- the HTTP query param is named `great` with a value of `example`, Then the traffic will be sent to the `productpage` service over port 9080.
```yml
 rules:
  - matches:
   - headers:
      - type: Exact
        name: magic
        value: foo
      queryParams:
      - type: Exact
        name: great
        value: example
      path:
        type: PathPrefix
        value: /
      method: GET
    backendRefs:
    - name: productpage
      port: 9080
```
As you can see, you can deploy sophisticated L7 traffic rules that are consistent (with Ingress API, annotations were often required to achieve such routing goals and that created inconsistencies from one Ingress controller to another). One of the benefits of these new APIs is that the Gateway API is essentially split into separate functions – one to describe the Gateway and one for the Routes to the back-end services. By splitting these two functions, it gives operators the ability to change and swap gateways but keep the same routing configuration. In other words: if you decide you want to use a different Gateway API controller instead, you will be able to reuse the same manifest. Let's have another look at the Services now that the Gateway has been deployed:
```shell
kubectl get svc
```
You will see a `LoadBalancer` service named `cilium-gateway-my-gateway` which was created for the Gateway API. The same external IP address is also associated to the Gateway:
```shell
kubectl get gateway
```
Let's retrieve this IP address:
```shell
GATEWAY=$(kubectl get gateway my-gateway -o jsonpath='{.status.addresses[0].value}')
echo $GATEWAY # 172.18.255.200
```

# HTTP path matching
Let's now check that traffic based on the URL path is proxied by the Gateway API. Check that you can make HTTP requests to that external address:
```shell
curl --fail -s http://$GATEWAY/details/1 | jq
```
Because the path starts with `/details`, this traffic will match the first rule and will be proxied to the `details` Service over port 9080. The `curl` request should be successful and return the following output:
```json
{
  "id": 1,
  "author": "William Shakespeare",
  "year": 1595,
  "type": "paperback",
  "pages": 200,
  "publisher": "PublisherA",
  "language": "English",
  "ISBN-10": "1234567890",
  "ISBN-13": "123-1234567890"
}
```

# HTTP Header matching
This time, we will route traffic based on HTTP parameters like header values, method and query parameters. Run the following command: 
```shell
curl -v -H 'magic: foo' "http://$GATEWAY?great=example"
```
The `curl` query should be successful and return a successful `200` code and a verbose HTML reply (look out for `Hello! This is a simple bookstore application consisting of three services as shown below`) Next, we'll review an Gateway API example with HTTPS traffic.

# Create TLS certificate and private key
In this task, we will be using Gateway API for HTTPS traffic routing; therefore we will need a TLS certificate for data encryption. For demonstration purposes we will use a TLS certificate signed by a made-up, self-signed certificate authority (CA). One easy way to do this is with `mkcert`. Create a certificate that will validate `bookinfo.cilium.rocks` and `hipstershop.cilium.rocks`, as these are the host names used in this Gateway example:
```shell
mkcert '*.cilium.rocks'
```
Mkcert created a key (`_wildcard.cilium.rocks-key.pem`) and a certificate (`_wildcard.cilium.rocks.pem`) that we will use for the Gateway service. Create a Kubernetes TLS secret with this key and certificate:
```shell
kubectl create secret tls demo-cert \
  --key=_wildcard.cilium.rocks-key.pem \
  --cert=_wildcard.cilium.rocks.pem
```

# Deploy the Gateway
Review the HTTPS Gateway API example provided in the current directory:
```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: tls-gateway
spec:
  gatewayClassName: cilium
  listeners:
    - name: https-1
      protocol: HTTPS
      port: 443
      hostname: "bookinfo.cilium.rocks"
      tls:
        certificateRefs:
          - kind: Secret
            name: demo-cert
    - name: https-2
      protocol: HTTPS
      port: 443
      hostname: "hipstershop.cilium.rocks"
      tls:
        certificateRefs:
          - kind: Secret
            name: demo-cert
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: https-app-route-1
spec:
  parentRefs:
    - name: tls-gateway
  hostnames:
    - "bookinfo.cilium.rocks"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /details
      backendRefs:
        - name: details
          port: 9080
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: https-app-route-2
spec:
  parentRefs:
    - name: tls-gateway
  hostnames:
    - "hipstershop.cilium.rocks"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: productpage
          port: 9080
```
It is almost identical to the one we reviewed previously. Just notice the following in the Gateway manifest:
```yml
spec:
  gatewayClassName: cilium
  listeners:
  - name: https-1
    protocol: HTTPS
    port: 443
    hostname: "bookinfo.cilium.rocks"
    tls:
      certificateRefs:
      - kind: Secret
        name: demo-cert
```
And the following in the HTTPRoute manifest:
```yml
spec:
  parentRefs:
  - name: tls-gateway
  hostnames:
  - "bookinfo.cilium.rocks"
```
The HTTPS Gateway API examples build up on what was done in the HTTP example and adds TLS termination for two HTTP routes:
- the `/details` prefix will be routed to the `details` HTTP service deployed in the HTTP challenge
- the `/` prefix will be routed to the `productpage` HTTP service deployed in the HTTP challenge
These services will be secured via TLS and accessible on two domain names:
- `bookinfo.cilium.rocks`
- `hipstershop.cilium.rocks`
In our example, the Gateway serves the TLS certificate defined in the `demo-cert` Secret resource for all requests to `bookinfo.cilium.rocks` and to `hipstershop.cilium.rocks`. Let's now deploy the Gateway to the cluster:
```shell
kubectl apply -f basic-https.yaml
```
This creates a LoadBalancer service, which after around 30 seconds or so should be populated with an external IP address. Verify that the Gateway has an load balancer IP address assigned:
```shell
kubectl get gateway tls-gateway
```
Then assign this IP to the GATEWAY variable so we can make use of it:
```shell
GATEWAY=$(kubectl get gateway tls-gateway -o jsonpath='{.status.addresses[0].value}')
echo $GATEWAY # 172.18.255.201
```

# Make requests
Install the Mkcert CA into your system so cURL can trust it:
```shell
mkcert -install
```
Now let's make a request to the Gateway:
```shell
curl -s \
  --resolve bookinfo.cilium.rocks:443:${GATEWAY} \
  https://bookinfo.cilium.rocks/details/1 | jq
```
The data should be properly retrieved, using HTTPS (and thus, the TLS handshake was properly achieved). In the next challenge, we will see how to use Gateway API for general TLS traffic.

# TLSRoute
In the previous task, we looked at the `TLS Termination` and how the Gateway can terminate HTTPS traffic from a client and route the unencrypted HTTP traffic based on HTTP properties, like path, method or headers. In this task, we will look at `TLSRoute`. This resource lets you passthrough TLS traffic from the client all the way to the Pods - meaning the traffic is encrypted end-to-end.

# Deploy the Demo App
We will be using a NGINX web server. Review the NGINX configuration.
```conf
# nginx.conf
events {
}

http {
  log_format main '$remote_addr - $remote_user [$time_local]  $status '
  '"$request" $body_bytes_sent "$http_referer" '
  '"$http_user_agent" "$http_x_forwarded_for"';
  access_log /var/log/nginx/access.log main;
  error_log  /var/log/nginx/error.log;

  server {
    listen 443 ssl;

    root /usr/share/nginx/html;
    index index.html;

    server_name nginx.cilium.rocks;
    ssl_certificate /etc/nginx-server-certs/tls.crt;
    ssl_certificate_key /etc/nginx-server-certs/tls.key;
  }
}
```
As you can see, it listens on port 443 for SSL traffic. Notice it specifies the certificate and key previously created. We will need to mount the files to the right path (`/etc/nginx-server-certs`) when we deploy the server. The NGINX server configuration is held in a Kubernetes ConfigMap. Let's create it.
```shell
kubectl create configmap nginx-configmap --from-file=nginx.conf=./nginx.conf
```
Review the NGINX server Deployment and the Service fronting it:
```yml
# tls-service.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
    - port: 443
      protocol: TCP
  selector:
    run: my-nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 1
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
        - name: my-nginx
          image: nginx
          ports:
            - containerPort: 443
          volumeMounts:
            - name: nginx-index-file
              mountPath: /usr/share/nginx/html/
            - name: nginx-config
              mountPath: /etc/nginx
              readOnly: true
            - name: nginx-server-certs
              mountPath: /etc/nginx-server-certs
              readOnly: true
      volumes:
        - name: nginx-index-file
          configMap:
            name: index-html-configmap
        - name: nginx-config
          configMap:
            name: nginx-configmap
        - name: nginx-server-certs
          secret:
            secretName: demo-cert
```
As you can see, we are deploying a container with the `nginx` image, mounting several files such as the HTML index, the NGINX configuration and the certs. Note that we are reusing the `demo-cert` TLS secret we created earlier.
```shell
kubectl apply -f tls-service.yaml
```
Verify the Service and Deployment have been deployed successfully:
```shell
kubectl get svc,deployment my-nginx
```

# Deploy the Gateway
Review the Gateway API configuration files provided in the current directory:
```yml
# tls-gateway.yaml
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: cilium-tls-gateway
spec:
  gatewayClassName: cilium
  listeners:
    - name: https
      hostname: "nginx.cilium.rocks"
      port: 443
      protocol: TLS
      tls:
        mode: Passthrough
      allowedRoutes:
        namespaces:
          from: All
# tls-route.yaml
---
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TLSRoute
metadata:
  name: nginx
spec:
  parentRefs:
    - name: cilium-tls-gateway
  hostnames:
    - "nginx.cilium.rocks"
  rules:
    - backendRefs:
        - name: my-nginx
          port: 443
```
They are almost identical to the one we reviewed in the previous tasks. Just notice the Passthrough mode set in the Gateway manifest:
```yml
spec:
  gatewayClassName: cilium
  listeners:
  - name: https
    hostname: "nginx.cilium.rocks"
    port: 443
    protocol: TLS
    tls:
      mode: Passthrough
    allowedRoutes:
      namespaces:
        from: All
```
Previously, we used the `HTTPRoute` resource. This time, we are using `TLSRoute`. Earlier you saw how you can terminate the TLS connection at the Gateway. That was using the Gateway API in `Terminate` mode. In this instance, the Gateway is in `Passthrough` mode: the difference is that the traffic remains encrypted all the way through between the client and the pod. In other words:
1) In `Terminate`:
- Client -> Gateway: HTTPS
- Gateway -> Pod: HTTP
2) In `Passthrough`:
- Client -> Gateway: HTTPS
- Gateway -> Pod: HTTPS
The Gateway does not actually inspect the traffic aside from using the SNI header for routing. Indeed the `hostnames` field defines a set of SNI names that should match against the SNI attribute of TLS ClientHello message in TLS handshake. Let's now deploy the Gateway and the TLSRoute to the cluster:
```shell
kubectl apply -f tls-gateway.yaml -f tls-route.yaml
```
This creates a LoadBalancer service, which after around 30 seconds or so should be populated with an external IP address. Verify that the Gateway has a LoadBalancer IP address assigned:
```shell
kubectl get gateway cilium-tls-gateway
```
Then assign this IP to the GATEWAY variable so we can make use of it:
```shell
GATEWAY=$(kubectl get gateway cilium-tls-gateway -o jsonpath='{.status.addresses[0].value}')
echo $GATEWAY # 172.18.255.202
```
Let's also double check the TLSRoute has been provisioned successfully and has been attached to the Gateway.
```shell
kubectl get tlsroutes.gateway.networking.k8s.io -o json | jq '.items[0].status.parents[0]'
```
Expect an outcome such as:
```json
{
  "conditions": [
    {
      "lastTransitionTime": "2025-06-23T07:28:50Z",
      "message": "Accepted TLSRoute",
      "observedGeneration": 1,
      "reason": "Accepted",
      "status": "True",
      "type": "Accepted"
    },
    {
      "lastTransitionTime": "2025-06-23T07:28:50Z",
      "message": "Service reference is valid",
      "observedGeneration": 1,
      "reason": "ResolvedRefs",
      "status": "True",
      "type": "ResolvedRefs"
    }
  ],
  "controllerName": "io.cilium/gateway-controller",
  "parentRef": {
    "group": "gateway.networking.k8s.io",
    "kind": "Gateway",
    "name": "cilium-tls-gateway"
  }
}
```

# Make TLS requests
Now let's make a request over HTTPS to the Gateway:
```shell
curl -v \
  --resolve "nginx.cilium.rocks:443:$GATEWAY" \
  "https://nginx.cilium.rocks:443"
```
The data should be properly retrieved, using HTTPS (and thus, the TLS handshake was properly achieved). There are several things to note in the output.
- It should be successful (you should see at the end, a HTML output with `Cilium rocks`.).
- The connection was established over port 443 - you should see `Connected to nginx.cilium.rocks (172.18.255.202) port 443`.
- You should see TLS handshake and TLS version negotiation. Expect the negotiations to have resulted in TLSv1.3 being used.
- Expect to see a successful certificate verification (look out for `SSL certificate verify ok`).

# Traffic splitting
Cilium Gateway API comes fully integrated with a HTTP traffic splitting engine. In order to introduce a new version of an app, operators would often start pushing some traffic to a new backend and see how users react and how the app fares under load. It’s also known as A/B testing, blue-green deployments or canary releases. You can now do it natively, with Cilium Gateway API weights. No need to install another tool or Service Mesh.

# Deploy an application
First, let's deploy a sample echo application in the cluster. The application will reply to the client and, in the body of the reply, will include information about the pod and node receiving the original request. We will use this information to illustrate that the traffic is split between multiple Kubernetes Services.
```shell
kubectl apply -f echo-servers.yaml
```
Look at the YAML file with the command below. You'll see we are deploying multiple pods and services. The services are called `echo-1` and `echo-2` and traffic will be split between these services.
```yml
# echo-servers.yaml
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
apiVersion: v1
kind: Service
metadata:
  labels:
    app: echo-2
  name: echo-2
spec:
  ports:
    - port: 8090
      name: high
      protocol: TCP
      targetPort: 8080
  selector:
    app: echo-2
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: echo-2
  name: echo-2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo-2
  template:
    metadata:
      labels:
        app: echo-2
    spec:
      containers:
        - image: gcr.io/kubernetes-e2e-test-images/echoserver:2.2
          name: echo-2
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
```
Check that the application is properly deployed:
```shell
kubectl get pods
```
You should see multiple pods being deployed in the `default` namespace. Wait until they are `Running` (should take 10 to 15 seconds). Have a quick look at the Services deployed:
```shell
kubectl get svc
```
Note these Services are only internal-facing (ClusterIP) and therefore there is no access from outside the cluster to these Services. 

# Load-Balance Traffic
Let's deploy the HTTPRoute with the following manifest:
```shell
kubectl apply -f load-balancing-http-route.yaml
```
Let's review the HTTPRoute manifest.
```yml
# load-balancing-http-route.yaml
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: load-balancing-route
spec:
  parentRefs:
    - name: my-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /echo
      backendRefs:
        - kind: Service
          name: echo-1
          port: 8080
          weight: 50
        - kind: Service
          name: echo-2
          port: 8090
          weight: 50
```
This Rule is essentially a simple L7 proxy route: for HTTP traffic with a path starting with /echo, forward the traffic over to the echo-1 and echo-2 Services over port 8080 and 8090 respectively. Notice the even 50/50 weighing.

# Even Traffic Splitting
Let's retrieve the IP address associated with the Gateway again:
```shell
GATEWAY=$(kubectl get gateway my-gateway -o jsonpath='{.status.addresses[0].value}')
echo $GATEWAY # 172.18.255.200
```
Let's now check that traffic based on the URL path is proxied by the Gateway API. Check that you can make HTTP requests to that external address:
```shell
curl --fail -s http://$GATEWAY/echo
```
Notice that, in the reply, you get the name of the pod that received the query. For example:
```shell
Hostname: echo-2-7ccd4fd567-gwd2c
```
Note that you can also see the headers in the original request. This will be useful in an upcoming task. Repeat the command several times. You should see the reply being balanced evenly across both pods/nodes. Let's double check that traffic is evenly split across multiple Pods by running a loop and counting the requests:
```shell
for _ in {1..500}; do
  curl -s -k "http://$GATEWAY/echo" >> curlresponses.txt;
done
```
Verify that the responses have been (more or less) evenly spread.
```shell
grep -o "Hostname: echo-." curlresponses.txt | sort | uniq -c
    # 251 Hostname: echo-1
    # 249 Hostname: echo-2
```

## 99/1 Traffic Split
This time, we will be applying a different weight. We could change the previous manifest and re-apply it or, if you don't mind using vi, we can edit the HTTPRoute specification directly on the API Server. Let's use this second option. Run:
```shell
kubectl edit httproute load-balancing-route
```
The `vi` editor will be automatically launch. Replace the weights from 50 for both `echo-1` and `echo-2` to **99** for `echo-1` and **1** for `echo-2`. Exit the editor using ESC followed by :wq (to save the file as you exit). Let's run another loop and count the replies again, with the following command:
```shell
for _ in {1..500}; do
  curl -s -k "http://$GATEWAY/echo" >> curlresponses991.txt;
done
```
Verify that the responses are spread with about 99% of them to `echo-1` and about 1% of them to `echo-2`.
```shell
grep -o "Hostname: echo-." curlresponses991.txt | sort | uniq -c
    # 497 Hostname: echo-1
    #   3 Hostname: echo-2
```