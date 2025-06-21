# Deploy the demo

In this example, we will look at adding mutual authentication to the Star Wars demo deployed in the [Getting Started with the Star Wars Demo](https://docs.cilium.io/en/v1.13/gettingstarted/demo/) docs and available in the [Getting Started with Cilium](https://isovalent.com/labs/cilium-getting-started) lab. Deploy the Star Wars environment and the L3/L4 network policy used in the demo (there's no mutual authentication in the network policy yet).

```bash
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/minikube/http-sw-app.yaml
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/minikube/sw_l3_l4_policy.yaml
```

Review the Network Policy:
```bash
kubectl get cnp rule1 -o yaml | yq .spec
```
```yaml
description: L3-L4 policy to restrict deathstar access to empire ships only
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
```

This network policy will only allow traffic from endpoints labeled with `org=empire` to endpoints with both the `class=deathstar` and `org=empire` labels, over TCP port `80`.

# Check Network Policy

Let's verify that the connectivity model is the expected one. First, verify that the Death Star Deployment is ready:
```bash
kubectl rollout status deployment deathstar -w
```

Next, verify that Tie Fighters (Empire space ships) are allowed to land on the Death Star:
```bash
kubectl exec tiefighter -- \
  curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```

Then verify that the X-Wing ship (belong to the Alliance) is denied access to the Death Star:
```bash
kubectl exec xwing -- \
  curl -s --connect-timeout 1 -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```

# Enforcing Mutual Authentication

Rolling out mutual authentication with Cilium is as simple as adding the following to an existing or new CiliumNetworkPolicy:
```yaml
spec:
  egress|ingress:
    authentication:
        mode: "required"
```

Let's do that now. We will be using this policy:
```shell
yq sw_l3_l4_l7_mutual_authentication_policy.yaml
```
```yaml
---
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
spec:
  description: "Mutual authentication enabled L7 policy"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
    - fromEndpoints:
        - matchLabels:
            org: empire
      authentication:
        mode: "required"
      toPorts:
        - ports:
            - port: "80"
              protocol: TCP
          rules:
            http:
              - method: "POST"
                path: "/v1/request-landing"
```

Review the changes we will be making with the existing network policy:
```shell
KUBECTL_EXTERNAL_DIFF='colordiff -u' \
  kubectl diff -f sw_l3_l4_l7_mutual_authentication_policy.yaml | \
  grep -A30 ' spec:'
```

The notable differences are:
- we are changing the description of the policy
- we are adding L7 filtering (only allowing HTTP POST to the /v1/request-landing)
- we are adding authentication.mode: required to our ingress rules. This will ensure that, in addition to the existing policy requirements, ingress access is only for mutually authenticated workloads.

Let's now apply this policy.
```bash
kubectl apply -f sw_l3_l4_l7_mutual_authentication_policy.yaml
```

# Verifying Connectivity after Enabling Mutual Authentication

Re-try the connectivity tests. Let's start with the `tiefighter` calling the `/request-landing` path:
```shell
kubectl exec tiefighter -- \
  curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```

This should still succeed. It may take some time for the response to print in the terminal. Let's then try access from the `tiefigher` to the `/exhaust-port` path:
```shell
kubectl exec tiefighter -- \
  curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
```

This second request should be denied, thanks to the new L7 Network Policy, preventing any `tiefighter` - compromised or not - from accessing the `/exhaust-port`.

```bash
kubectl exec xwing -- \
  curl -s --connect-timeout 1 -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```

This third one should time out, thanks to the L3/L4 Network Policy.

# Observing Mutual Authentication with Hubble

Let's now observe Mutual Authentication with Hubble. Run the connectivity checks again:
```shell
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
kubectl exec xwing -- curl -s --connect-timeout 1 -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```

First, let's look at flows from the `xwing` to the `deathstar`. The network policy should have dropped flows from the `xwing` as the `xwing` has not got the right labels.
```bash
hubble observe --type drop --from-pod default/xwing
```

The policy verdict for this traffic should be DROPPED by the L3/L4 section of the Network Policy:
```bash
Jun 21 07:11:32.910: default/xwing:50450 (ID:46398) <> default/deathstar-67c5c5c88-r2jsj:80 (ID:10411) Policy denied DROPPED (TCP Flags: SYN)
```

Let's now look at traffic from the `tiefighter` to the `deathstar`. The network policy should have dropped the first flow from the `tiefighter` to the `deathstar` Service over `/request-landing`. Why? Because the first packet to match the mutual authentication-based network policy will kickstart the mutual authentication handshake.
```shell
hubble observe --type drop --from-pod default/tiefighter
```

Expect an output such as:
```bash
Jun 21 07:16:24.433: default/tiefighter:56890 (ID:6788) <> default/deathstar-67c5c5c88-r2jsj:80 (ID:10411) Authentication required DROPPED (TCP Flags: SYN)
```

Again, this is expected: the first packet from tiefighter to deathstar is dropped as this is how Cilium is notified to start the mutual authentication process. You should see a similar behaviour when looking for flows with the policy-verdict filter:
```bash
hubble observe --type policy-verdict --from-pod default/tiefighter
```

Expect logs such as (yours might be different, depending on how many times you have tried access between the Pods):
```shell
Jun 21 07:10:54.392: default/tiefighter:57670 (ID:6788) -> default/deathstar-67c5c5c88-r2jsj:80 (ID:10411) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN)
Jun 21 07:16:24.433: default/tiefighter:56890 (ID:6788) <> default/deathstar-67c5c5c88-r2jsj:80 (ID:10411) policy-verdict:L3-L4 INGRESS DENIED (TCP Flags: SYN; Auth: SPIRE)
Jun 21 07:16:25.488: default/tiefighter:56890 (ID:6788) -> default/deathstar-67c5c5c88-r2jsj:80 (ID:10411) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN; Auth: SPIRE)
```

# Logs Explanation

Let's explain these 3 lines of logs.
1. ALLOWED log (no mutual auth)
```bash
default/deathstar-67c5c5c88-r2jsj:80 (ID:10411) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN)
```
The first request was allowed as it happened before we applied Mutual Authentication (note that `Auth: SPIRE` is not in the error message).

2. DENIED (mutual auth)
```bash
default/deathstar-67c5c5c88-r2jsj:80 (ID:10411) policy-verdict:L3-L4 INGRESS DENIED (TCP Flags: SYN; Auth: SPIRE)
```
The second request was denied because the mutual authentication handshake had not completed yet.

3. ALLOWED (mutual auth)
```bash
default/deathstar-67c5c5c88-r2jsj:80 (ID:10411) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN; Auth: SPIRE)
```
The last request was successful, as the handshake was successful.