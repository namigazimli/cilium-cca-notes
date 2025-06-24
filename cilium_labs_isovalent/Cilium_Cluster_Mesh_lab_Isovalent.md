# Two Kind Clusters
In this lab, we will create two Kind clusters and mesh them using Cilium. We'll have two requirements for these clusters:
- disable default CNI so we can easily install Cilium
- use disjoint pods and services subnets

# Koornacht Cluster
Let's have a look at the configuration for the first cluster, which we will be calling Koornacht:
```yml
# yq kind_koornacht.yml
---
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
networking:
  disableDefaultCNI: true
  podSubnet: 10.1.0.0/16
  serviceSubnet: 172.20.1.0/24
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
```
This cluster will feature one control-plane node and 2 worker nodes, and use `10.1.0.0/16` for the Pod network, and `172.20.1.0/24` for the Services. Create the Koornacht first cluster with:
```shell
# ⚠️ In the Koornacht tab
kind create cluster --name koornacht --config kind_koornacht.yaml
```
Verify that all 3 nodes are up:
```shell
# ⚠️ In the Koornacht tab
kubectl get nodes
```
The nodes are marked as `NotReady` because there is not CNI plugin set up yet. Then install Cilium on it:
```shell
# ⚠️ In the Koornacht tab
cilium install \
  --set cluster.name=koornacht \
  --set cluster.id=1 \
  --set ipam.mode=kubernetes
```
Let's also enable Hubble for observability, only on the Koornacht cluster:
```shell
# ⚠️ In the Koornacht tab
cilium hubble enable --ui
```
Verify that everything is fine with:
```shell
# ⚠️ In the Koornacht tab
cilium status --wait
```

# Tion Cluster
Let's now create a second Kind cluster, which we will call Tion.
```yml
# yq kind_tion.yml
---
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
networking:
  disableDefaultCNI: true
  podSubnet: 10.2.0.0/16
  serviceSubnet: 172.20.2.0/24
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```
This Tion cluster will also feature one control-plane node and 2 worker nodes, but it will use `10.2.0.0/16` for the Pod network, and `172.20.2.0/24` for the Services. Create the Tion cluster with:
```shell
# ⚠️ In the Tion tab
kind create cluster --name tion --config kind_tion.yaml
```
Verify that all 3 nodes are up:
```shell
# ⚠️ In the Tion tab
kubectl get nodes
```
Then install Cilium on it:
```shell
# ⚠️ In the Tion tab
cilium install \
  --set cluster.name=tion \
  --set cluster.id=2 \
  --set ipam.mode=kubernetes
```
Verify that everything is fine with:
```shell
# ⚠️ In the Tion tab
cilium status --wait
```
Now that we have two Kind clusters installed with Cilium, let's get them meshed!

# Cluster Mesh Architecture
When activating Cluster Mesh on Cilium clusters, a new Control Plane is deployed to manage the mesh for this cluster, along with its etcd key-value store. Agents of other clusters can then access this Cluster Mesh Control Plane in read-only mode, allowing them to access metadata about the cluster, such as Service names and corresponding IPs. 
![Alt text](https://play.instruqt.com/assets/tracks/gcdcyyilhrqt/0f3b2c9fcfd6b679ee94210a7c7589d2/assets/architecture.png)
Cilium Cluster Mesh allows to link multiple Kubernetes clusters, provided:
- all clusters run Cilium as CNI
- all worker nodes have a unique IP address and are able to connect to each other

# Enable Cluster Mesh
Enable Cluster Mesh on both clusters:
```shell
# ⚠️ In *both* the Koornacht and Tion tabs
cilium clustermesh enable --service-type NodePort
```
Several types of connectivity can be set up. We're using `NodePort` in our case as it's easier and we don't have dynamic load balancers available. For production clusters, it is strongly recommended to use `LoadBalancer` instead.
Wait for Cluster Mesh to be ready on both clusters:
```shell
# ⚠️ In *both* the Koornacht and Tion tabs
cilium clustermesh status --wait
```
You can also verify the Cluster Mesh status with cilium status:
```shell
# ⚠️ In *both* the Koornacht and Tion tabs
cilium status
```

# Mesh Clusters
Let's now connect the clusters by instructing one cluster to mesh with the second one. This needs to be done in a shell with access to both cluster contexts, so we'll use the >_ 🌐 Global tab for that:
```shell
# ⚠️ In the Global tab
cilium clustermesh connect \
  --context kind-koornacht \
  --destination-context kind-tion
```
Wait for both clusters to be ready:
```shell
# ⚠️ In *both* the Koornacht and Tion tabs
cilium clustermesh status --wait
```
Our two clusters are now meshed together. Let's deploy applications on them!

# Deploy application on Koornacht Cluster
We will deploy a simple HTTP application returning a JSON, including the name of the cluster:
```shell
# ⚠️ In the Koornacht tab
kubectl apply -f deployment.yaml
```
```yml
# deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rebel-base
spec:
  selector:
    matchLabels:
      name: rebel-base
  replicas: 2
  template:
    metadata:
      labels:
        name: rebel-base
    spec:
      containers:
      - name: rebel-base
        image: docker.io/nginx:1.15.8
        volumeMounts:
          - name: html
            mountPath: /usr/share/nginx/html/
        livenessProbe:
          httpGet:
            path: /
            port: 80
          periodSeconds: 1
        readinessProbe:
          httpGet:
            path: /
            port: 80
      volumes:
        - name: html
          configMap:
            name: rebel-base-response
            items:
              - key: message
                path: index.html
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: x-wing
spec:
  selector:
    matchLabels:
      name: x-wing
  replicas: 2
  template:
    metadata:
      labels:
        name: x-wing
    spec:
      containers:
      - name: x-wing-container
        image: docker.io/cilium/json-mock:1.2
        livenessProbe:
          exec:
            command:
            - curl
            - -sS
            - -o
            - /dev/null
            - localhost
        readinessProbe:
          exec:
            command:
            - curl
            - -sS
            - -o
            - /dev/null
            - localhost
```
The `ConfigMap` for this service contains the JSON reply, with the name of the Cluster hardcoded (`-o yaml` is added here to show you the content of the resource):
```shell
# ⚠️ In the Koornacht tab
kubectl apply -f configmap_koornacht.yaml -o yaml
apiVersion: v1
data:
  message: |
    {"Cluster": Koornacht", "Planet": "N'Zoth"}
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"message":"{\"Cluster\": \"Koornacht\", \"Planet\": \"N'Zoth\"}\n"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"rebel-base-response","namespace":"default"}}
  creationTimestamp: "2025-06-24T05:18:59Z
  name: rebel-base-response
  namespace: default
  resourceVersion: "3105"
  uid: d0b33442-b69d-4c12-b201-646f45d51705
```
Check that the pods are running properly (launch until all 4 pods are Running):
```shell
# ⚠️ In the Koornacht tab
kubectl get pod
```
Apply the Service for the application:
```shell
# ⚠️ In the Koornacht tab
kubectl apply -f service.yaml
```
```yml
# service.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: rebel-base
spec:
  type: ClusterIP
  ports:
  - port: 80
  selector:
    name: rebel-base
```
Let's test this service, using the x-wing pod deployed alongside the rebel-base deployment:
```shell
# ⚠️ In the Koornacht
kubectl exec -ti deployments/x-wing -- /bin/sh -c 'for i in $(seq 1 10); do curl rebel-base; done'
```
You should see 10 lines of log, all containing:
```json
{"Cluster":"Koornacht","Planet":"N'Zoth"}
```
Go the Hubble UI tab and select the `default` namespace. You will see the `x-wing` requests being sent to the `rebel-base` pods.

# Deploy application on Tion Cluster
We will deploy the same application and service on the Tion cluster, with a small difference: the JSON answer will reply with `Tion` since we're using a slightly different `ConfigMap`:
```shell
# ⚠️ In the Tion tab
kubectl apply -f deployment.yaml
kubectl apply -f configmap_tion.yaml -o yaml
kubectl apply -f service.yaml
```
Wait until the pods are ready (run kubectl get po until all pods are Ready) and check this second service:
```shell
# ⚠️ In the Tion tab
kubectl exec -ti deployments/x-wing -- /bin/sh -c 'for i in $(seq 1 10); do curl rebel-base; done'
```
After the pods start, you should see 10 lines of log, all containing:
```json
{"Cluster": "Tion", "Planet": "Foran Tutha"}
```
We now have similar applications running on both our clusters. Wouldn't it be great if we could load-balance traffic between them? This is precisely what we'll be doing in the next challenge!

# Making Services Global
When two or more clusters are meshed, Cilium allows you to set services as global in one or more clusters, by adding an annotation to them:
```yml
service.cilium.io/global: "true"
```
When this annotation is set, requests to this service will load-balance to all available services with the same name and namespace in all meshed clusters. One obvious usage of global services is **fault tolerance**. If a service becomes unavailable in one cluster, traffic can be redirected to the same service in other clusters, ensuring a continuity of service.
![Alt text](https://cilium.io/static/04d2d06e7e32665b74c968a9f7fc0a40/905a7/usecase_ha.webp)
Another use case of global services is **shared services**. This is particularly useful when sharing stateful services between multiple Kubernetes clusters. If all clusters are meshed, the stateless applications spread among multiple clusters can all access the stateful services located on a single shared cluster.
![Alt text](https://cilium.io/static/6e50204727fd8d86f24e8045e3065de2/905a7/usecase_shared_services.webp)

## Global Service
Let's make the service global on the mesh. Add the annotation to the service metadata:
```shell
# ⚠️ In *both* the Koornacht and Tion tabs
kubectl annotate service rebel-base service.cilium.io/global="true"
```
When accessing the service from either cluster, it should now be load-balanced between the two clusters, because it is marked as global:
```shell
# ⚠️ In the Koornacht tab
kubectl exec -ti deployments/x-wing -- /bin/sh -c 'for i in $(seq 1 10); do curl rebel-base; done'
```
You should see a mix of replies from the Koornacht and Tion clusters:
```json
{"Cluster": "Koornacht", "Planet": "N'Zoth"}
{"Cluster": "Koornacht", "Planet": "N'Zoth"}
{"Cluster": "Koornacht", "Planet": "N'Zoth"}
{"Cluster": "Tion", "Planet": "Foran Tutha"}
{"Cluster": "Tion", "Planet": "Foran Tutha"}
{"Cluster": "Koornacht", "Planet": "N'Zoth"}
{"Cluster": "Tion", "Planet": "Foran Tutha"}
```
Check the Hubble UI tab. After a little while, you will see the `x-wing` pod from the Koornacht cluster accessing `rebel-base` pods from both the Koornacht and Tion clusters.
Now test also from the Tion cluster:
```shell
# ⚠️ In the Tion tab
kubectl exec -ti deployments/x-wing -- /bin/sh -c 'for i in $(seq 1 10); do curl rebel-base; done'
```
You should see requests being load-balanced between the two clusters. Verify the Hubble UI tab that you see traffic coming from the Tion cluster to the Koornacht `rebel-base` pods. The service is now global on both clusters!

## Fault Resilience
With this setup in place, let's scale down the deployment on the Koornacht cluster:
```shell
# ⚠️ In the Koornacht tab
kubectl scale deployment rebel-base --replicas 0
```
Now check the replies when querying from the Koornacht cluster:
```shell
# ⚠️ In the Koornacht tab
kubectl exec -ti deployments/x-wing -- /bin/sh -c 'for i in $(seq 1 10); do curl rebel-base; done'
```
And from the Tion cluster:
```shell
# ⚠️ In the Tion tab
kubectl exec -ti deployments/x-wing -- /bin/sh -c 'for i in $(seq 1 10); do curl rebel-base; done'
```
You should see only entries like:
```json
{"Cluster": "Tion", "Planet": "Foran Tutha"}
```
You can see that requesting the service on both clusters now only yields answers from the Tion cluster, effectively making up for the missing pods on the Koornacht cluster. We've now seen how clusters can access all instances of an identical service across meshed cluster. What if we want to remove one specific instance of the service from the global service? We'll see how to do this in the next challenge!

# Global vs Shared
We've seen how the service.cilium.io/global annotation allows for a cluster to load-balance requests to a service to all meshed clusters with the same annotated service. What if you want to remove the service of a specific cluster from the global service? The `service.cilium.io/shared` annotation can be used for this. By default, a service marked as global is considered shared as well, so the value of `service.cilium.io/shared` is `true` for all clusters where the service is marked as global. Setting it to `false` in a cluster removes that specific service from the global service:
```yml
service.cilium.io/shared: "false"
```

## Scale back Service on Koornacht
First, let's scale deployment on the Koornacht cluster back to two:
```shell
# ⚠️ In the Koornacht tab
kubectl scale deployment rebel-base --replicas 2
```
Verify that the service is properly load-balanced from the both clusters:
```shell
# ⚠️ In *both* the Koornacht and Tion tabs
kubectl exec -ti deployments/x-wing -- /bin/sh -c 'for i in $(seq 1 10); do curl rebel-base; done'
```

## Disable Global Service on Koornacth Cluster
Now we want to prevent the Tion cluster from accessing the service running on the Koornacht cluster. Let's add the `service.cilium.io/shared=false` annotation to the service on Koornacht to opt out of the global service:
```shell
# ⚠️ In the Koornacht tab
kubectl annotate service rebel-base service.cilium.io/shared="false"
```
From the Koornacht cluster, requests are still load-balanced, as the service is global and the Tion cluster is allowing its service to be shared:
```shell
# ⚠️ In the Koornacht tab
kubectl exec -ti deployments/x-wing -- /bin/sh -c 'for i in $(seq 1 10); do curl rebel-base; done'
```
From the Tion cluster however, you should only see requests going to the Tion service, since Koornacht is preventing access to its service now:
```shell
# ⚠️ In the Tion tab
kubectl --context kind-tion exec -ti deployments/x-wing -- /bin/sh -c 'for i in $(seq 1 10); do curl rebel-base; done'
```
In the next challenge, we will see how to tune global services for finer affinity.

# Adding a local affinity
Let's consider the Koornacht service, which currently load balances to both clusters. We would like to make it so that it always sends traffic to the Koornacht pods if available, but sends to the Tion pods if no pods are found on the Koornacht cluster. In order to achieve this, let's add a new annotation to the Koornacht service:
```shell
# ⚠️ In the Koornacht tab
kubectl annotate service rebel-base service.cilium.io/affinity=local
```
Test the requests to the Koornacht service, which now only target the Koornacht pods:
```shell
# ⚠️ In the Koornacht tab
kubectl exec -ti deployments/x-wing -- /bin/sh -c 'for i in $(seq 1 10); do curl rebel-base; done'
```
Now scale down the Koornacht deployment:
```shell
# ⚠️ In the Koornacht tab
kubectl scale deployment rebel-base --replicas 0
```
And try again:
```shell
# ⚠️ In the Koornacht tab
kubectl exec -ti deployments/x-wing -- /bin/sh -c 'for i in $(seq 1 10); do curl rebel-base; done'
```
All traffic now goes to the Tion cluster. When the pods come back up on the Koornacht cluster, the service will send traffic to them again:
```shell
# ⚠️ In the Koornacht tab
kubectl scale deployment rebel-base --replicas 2
kubectl rollout status deployment/rebel-base
kubectl exec -ti deployments/x-wing -- /bin/sh -c 'for i in $(seq 1 10); do curl rebel-base; done'
```
The opposite effect can be obtained by using `remote` as the annotation value.