# Network observability with Hubble
Hubble, built on top of Cilium and eBPF, is a networking and security observability platform, enabling deep visibility into the communication and services of your cloud native workloads. Relying on eBPF, all visibility is programmable, allowing for a dynamic approach that minimizes overhead while providing deep and detailed visibility as required by users. Hubble has been created and specifically designed to best use these new eBPF powers. Observability is provided by Hubble which enables deep visibility into the communication and behavior of services as well as the networking infrastructure in a completely transparent manner. Hubble is able to provide visibility at the node level, cluster level or even across clusters in a Multi-Cluster (Cluster Mesh) scenario.  

By default, Hubble API operates within the scope of the individual node on which the Cilium agent runs. This confines the network insights to the traffic observed by the local Cilium agent. Hubble CLI (`hubble`) can be used to query the Hubble API provided via a local Unix Domain Socket. The Hubble CLI binary is installed by default on Cilium agent pods. Upon deploying Hubble Relay, network visibility is provided for the entire cluster or even multiple clusters in a ClusterMesh scenario. In this mode, Hubble data can be accessed by directing Hubble CLI (hubble) to the Hubble Relay service or via Hubble UI. Hubble UI is a web interface which enables automatic discovery of the services dependency graph at the L3/L4 and even L7 layer, allowing user-friendly visualization and filtering of data flows as a service map. 

Hubble can answer questions such as:
- What services are communicating with each other? How frequently? What does the service dependency graph look like?
- What HTTP calls are being made? What Kafka topics does a service consume from or produce to?
- Is any network communication failing? Why is communication failing? Is it DNS? Is it an application or network problem? Is the communication broken on layer 4 (TCP) or layer 7 (HTTP)?
- Which services have experienced a DNS resolution problem in the last 5 minutes? Which services have recently experienced an interrupted TCP connection or have seen connections timing out? What is the rate of unanswered TCP SYN requests?
- What is the rate of 5xx or 4xx HTTP response codes for a particular service or across all clusters?
- What is my cluster’s 95th and 99th percentile latency between HTTP requests and responses? Which services are performing the worst? What is the latency between the different services?
- Which services had connections blocked due to network policy? What services have been accessed from outside the cluster? Which services have resolved a particular DNS name?

# Components of Hubble
Below is a high-level overview of the components that make up Hubble Observability with Cilium in a Kubernetes Cluster.
![Alt text](https://cdn.sanity.io/images/xinsvxfu/production/a15c854a9769f69bb2e7ecdaed65fb78277893b3-800x418.png?auto=format&q=80&fit=clip&w=1280)
**Cilium Agent** – Runs the cilium-agent binary which acts as a CNI to manage connectivity, observability, and security for all CNI-managed Kubernetes pods.
**Hubble Server** - It runs on each node and retrieves the eBPF-based visibility from Cilium. It is embedded into the Cilium agent in order to achieve high performance and low-overhead. It offers a gRPC service to retrieve flows and expose Prometheus metrics.
**Hubble Relay** – Provides a cluster-wide API for querying Hubble flow data, which can be accessed directly or via the Hubble CLI and UI. When the Hubble Relay is deployed, Hubble provides full network visibility by providing a Hubble API which scopes the entire cluster or even multiple clusters in a ClusterMesh scenario.
**Hubble UI** – Provides a graphical UI for visualizing network flow data, network policy, and security-related events.

Consuming data from individual Hubble Servers and from the Hubble Relay can be done in two different ways: CLI-based and UI-based
- The Hubble CLI is a command-line binary able to connect to either the gRPC API of Hubble Relay or the local server to retrieve flow events.
- The graphical User Interface utilizes relay-based visibility to provide a graphical service dependency and connectivity map.

# Setting up Hubble Observability
Hubble is the observability layer of Cilium and can be used to obtain cluster-wide visibility into the network and security layer of your Kubernetes cluster.
1. Enable Hubble in Cilium - In order to enable Hubble, run the command `cilium hubble enable` as shown below:
```shell
cilium hubble enable
```
2. Enabling Hubble requires the TCP port 4244 to be open on all nodes running Cilium. This is required for Relay to operate correctly. Run `cilium status` to validate that Hubble is enabled and running:
```shell
cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:         OK
 \__/¯¯\__/    Operator:       OK
 /¯¯\__/¯¯\    Hubble:         OK
 \__/¯¯\__/    ClusterMesh:    disabled
    \__/
```
3. Install the Hubble Client - In order to access the observability data collected by Hubble, you must first install Hubble CLI. Download the latest hubble release:
```shell
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
HUBBLE_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then HUBBLE_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-${HUBBLE_ARCH}.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-${HUBBLE_ARCH}.tar.gz /usr/local/bin
rm hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
```
4. Validate Hubble API access - In order to access the Hubble API, create a port forward to the Hubble service from your local machine. This will allow you to connect the Hubble client to the local port `4245` and access the Hubble Relay service in your Kubernetes cluster.
```shell
cilium hubble port-forward&
```
5. Now you can validate that you can access the Hubble API via the installed CLI:
```shell
hubble status
```
6. You can also query the flow API and look for flows:
```shell
hubble observe
```

# Service Map and Hubble UI
Enable the Hubble UI by running one of the following commands:
1. Cilium CLI - If Hubble is already enabled with cilium hubble enable, you must first temporarily disable Hubble with cilium hubble disable. This is because the Hubble UI cannot be added at runtime.
```shell
cilium hubble enable --ui
```
2. Helm
```shell
helm upgrade cilium cilium/cilium --version 1.17.0 \
   --namespace $CILIUM_NAMESPACE \
   --reuse-values \
   --set hubble.relay.enabled=true \
   --set hubble.ui.enabled=true
```

# Open the Hubble UI
Open the Hubble UI in your browser by running cilium hubble ui. It will automatically set up a port forward to the hubble-ui service in your Kubernetes cluster and make it available on a local port on your machine.
```shell
cilium hubble ui
```
If your browser has not automatically opened the UI, open the page `http://localhost:12000` in your browser. You should see a screen with an invitation to select a namespace, use the namespace selector dropdown on the left top corner to select a namespace. 