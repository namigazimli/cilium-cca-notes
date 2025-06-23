# Cilium Overview
Cilium provides Connectivity, Observability and Security capabilities in a Cloud Native World, and is based on eBPF.
![Alt text](https://play.instruqt.com/assets/tracks/ucdsyxm1sfzh/c602a2a595a6cc09ad58a6211dd93d9b/assets/cilium_overview.png)

# Identities, Protocol parsing & Observability
From inception, Cilium was designed for large-scale, highly-dynamic containerized environments. Cilium:
- natively understands container identities
- parses API protocols like HTTP, gRPC, and Kafka
- and provides visibility and security that is both simpler and more powerful than traditional approaches
![Alt text](https://play.instruqt.com/assets/tracks/ucdsyxm1sfzh/c8aefb5236f2b02b7c3d775ae88d8539/assets/identity_store.png)

# Hubble
Hubble is a fully distributed networking and security observability platform for Cloud Native workloads. Hubble is an open source software and is built on top of Cilium and eBPF to enable deep visibility into:
- the communication and behavior of services as well as
- the networking infrastructure in a completely transparent manner

Cilium Architecture:
![Alt text](https://play.instruqt.com/assets/tracks/ucdsyxm1sfzh/a782fe149f90c8a88716a69170e039f1/assets/cilium-arch.png)

# The Kind Cluster
```yaml
# yq /etc/kind/${KIND_CONFIG}.yaml
---
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      # localhost.run proxy
      - containerPort: 32042
        hostPort: 32042
      # Hubble relay
      - containerPort: 31234
        hostPort: 31234
      # Hubble UI
      - containerPort: 31235
        hostPort: 31235
  - role: worker
  - role: worker
networking:
  disableDefaultCNI: true
  kubeProxyMode: none
```

# Nodes
In the `nodes` section, you can see that the cluster consists of three nodes:
- 1 `control-plane` node running the Kubernetes control plane and etcd
- 2 `worker` nodes to deploy the applications

# Networking
In the networking section of the configuration file, the default CNI has been disabled so the cluster won't have any Pod network when it starts. Instead, Cilium is being deployed to the cluster to provide this functionality. To see if the Kind cluster is ready, verify that the cluster is properly running by listing its nodes:
```shell
kubectl get nodes
```
You should see the three nodes appear, all marked as NotReady. This is normal, since the CNI is disabled, and we will install Cilium in the next step. If you don't see all nodes, the workers nodes might still be joining the cluster. Relaunch the command until you can see all three nodes listed.

# The Cilium CLI
The cilium CLI tool is provided in this environment to install and check the status of Cilium in the cluster. Let's start by installing Cilium on the Kind cluster:
```shell
cilium install
```
Wait for the installation to finish —usually about a minute— and check the status with:
```shell
cilium status --wait
```
Now that Cilium is functional on our cluster, let's deploy a demo application on it!

# Deploy the demo application
Let's deploy a simple empire demo application. It is made of several microservices, each identified by Kubernetes labels:
- the Death Star: `org=empire`, `class=deathstar`
- the Imperial TIE fighter: `org=empire`, `class=tiefighter`
- the Rebel X-Wing: `org=alliance`, `class=xwing`
The deployment also includes a deathstar-service, which load-balances traffic to all pods with label `org=empire`, `class=deathstar`. Let's install everything via the manifest http-sw-app.yaml:
```shell
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/minikube/http-sw-app.yaml
```
In the output, note the newly created objects. To verify that everything is properly deployed, run:
```shell
kubectl get pods,svc
```
Each pod will also be represented in Cilium as an Endpoint. To retrieve a list of all endpoints managed by Cilium, the Cilium Endpoint (or `cep`) resource can be used:
```shell
kubectl get cep --all-namespaces
```
As you can see the demo application is properly installed now. Let's launch our first test!

# Check current access
To simulate our connectivity tests, we will be executing simple API calls using curl. Let's test if we can land our TIE fighter on the Death Star by running the following command:
```shell
kubectl exec tiefighter -- \
  curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```
The command above lets us get a shell on the tiefighter pod and run a HTTP POST request to the deathstar Service to request landing. The command should work —as the TIE fighter and the Death Star are on the same side of the galactic wars (i.e. the bad guys). Now test if you can land your X-wing (i.e. the good guys) with:
```shell
kubectl exec xwing -- \
  curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```
So far, it seems access is allowed! This is good for the rebel alliance —unfettered access to the DeathStar— but this should not be allowed, right? There is a security policy missing!

# Writing Network Policies
We’ll start with a basic policy restricting deathstar landing requests to only the ships that have the label `org=empire`. This blocks any ships without the `org=empire` label to even connect to the `deathstar` service. This is a simple policy that filters only on network layer 3 (IP protocol) and network layer 4 (TCP protocol), so it is often referred to as a L3/L4 network security policy.
![Alt text](https://play.instruqt.com/assets/tracks/ucdsyxm1sfzh/6ff094083acf6bf5d3e4e12a511628af/assets/star_wars_l3l4.png)

# Crafting Network Policies
We’ll start with the basic policy to restrict deathstar landing requests to the ships that have label `org=empire` only. We need to match on empire ships only, so we need to match on that label:
```yml
spec:
  description: "L3-L4 policy to restrict deathstar access to empire ships only"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
```
Furthermore, we have to make sure that ingress from endpoints with the label empire is allowed to port 80 for protocol tcp:
```yml
  ingress:
  - fromEndpoints:
    - matchLabels:
        org: empire
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
```

# Enforcing the Network Policy
There we can apply a preconfigured network policy with the values discussed above to our demo system:
```shell
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/minikube/sw_l3_l4_policy.yaml
```
Now let's try to land the empire tiefighter again (HTTP POST from tiefighter to deathstar on the /v1/request-landing path):
```shell
kubectl exec tiefighter -- \
  curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```
This still works, which is expected. In comparison, if you try to request landing from the xwing pod, you will see that the request will eventually time out:
```shell
kubectl exec xwing -- \
  curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```
Kill the request with `Ctrl+C` once you realize that it hangs. We have successfully blocked access to the deathstar from an X-Wing ship. Let's now see how we could make this policy a bit more fine-grained using L7 rules.

# Filtering on HTTP
We need to filter on a higher level: we need to filter the actual HTTP requests!
![Alt text](https://play.instruqt.com/assets/tracks/ucdsyxm1sfzh/5d08c712a544067c7d4904e6d59f9346/assets/star_wars_l7.png)

# Filtering paths
We need to extend the existing policy with an HTTP rule such as:
```yml
rules:
  http:
  - method: "POST"
    path: "/v1/request-landing"
```
This will restrict API access to only the /`v1/request-landing` path and will thus prevent users from accessing the `/v1/exhaust-port`, which caused a crash as we saw earlier.

# Enforcing the Network Policy
Apply this updated rule to your demo system:
```shell
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/minikube/sw_l3_l4_l7_policy.yaml
```
Run the same test as above, and see the different outcome:
```shell
kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
```
As you can see, with Cilium L7 security policies, we are able to restrict tiefighter's access to only the required API resources on deathstar, thereby implementing a “least privilege” security approach for communication between microservices.