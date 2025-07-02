# Installation using Helm
This guide will show you how to install Cilium using Helm. This involves a couple of additional steps compared to the [Cilium Quick Installation](./Install_Cilium.md) and requires you to manually select the best datapath and IPAM mode for your particular environment. 

# Install Cilium
Setup Helm repository:
```shell
helm repo add cilium https://helm.cilium.io/
```
These are the generic instructions on how to install Cilium into any Kubernetes cluster using the default configuration options below. Please see the other tabs for distribution/platform specific instructions which also list the ideal default configuration for particular platforms.

**Default Configuration**:

|   Datapath    |   IPAM        |   Datastore       |
| ------------- | ------------- | ----------------- |
| Encapsulation | Cluster Pool  | Kubernetes CRD    |

**Requirements**:
- Kubernetes must be configured to use CNI
- Linux kernel >= 5.4

Deploy Cilium release via Helm:
```shell
helm install cilium cilium/cilium --version 1.17.5 \
  --namespace kube-system
```

# Validate the installation
To validate that Cilium has been properly installed, you can run
```shell
cilium status --wait
```
![Alt text](./cilium_status.png)

Run the following command to validate that your cluster has proper network connectivity:
```shell
cilium connectivity test
ℹ️  Monitor aggregation detected, will skip some flow validation steps
✨ [k8s-cluster] Creating namespace for connectivity check...
(...)
---------------------------------------------------------------------------------------------------------------------
📋 Test Report
---------------------------------------------------------------------------------------------------------------------
✅ 69/69 tests successful (0 warnings)

```