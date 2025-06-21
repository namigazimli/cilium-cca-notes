# Transparent Encryption

Cilium supports the transparent encryption of Cilium-managed host traffic and traffic between Cilium-managed endpoints either using IPsec or WireGuard®:
- [IPsec Transparent Encryption](https://docs.cilium.io/en/stable/security/network/encryption-ipsec/)
- [WireGuard Transparent Encryption](https://docs.cilium.io/en/stable/security/network/encryption-wireguard/)

# IPsec Transparent Encryption

This guide explains how to configure Cilium to use IPsec based transparent encryption using Kubernetes secrets to distribute the IPsec keys. After this configuration is complete all traffic between Cilium-managed endpoints, as well as Cilium-managed host traffic, will be encrypted using IPsec. This guide uses Kubernetes secrets to distribute keys. Alternatively, keys may be manually distributed. Packets are not encrypted when they are destined to the same node from which they were sent. This behavior is intended. Encryption would provide no benefits in that case, given that the raw traffic can be observed on the node anyway.

## Generate & Import the PSK

First, create a Kubernetes secret for the IPsec configuration to be stored. The example below demonstrates generation of the necessary IPsec configuration which will be distributed as a Kubernetes secret called `cilium-ipsec-keys`. A Kubernetes secret should consist of one key-value pair where the key is the name of the file to be mounted as a volume in cilium-agent pods, and the value is an IPsec configuration in the following format:

```bash
key-id encryption-algorithms PSK-in-hex-format key-size
```

`Secret` resources need to be deployed in the same namespace as Cilium! In the example below, GCM-128-AES is used. However, any of the algorithms supported by Linux may be used. To generate the secret, you may use the following command:

```bash
kubectl create -n kube-system secret generic cilium-ipsec-keys \
    --from-literal=keys="3+ rfc4106(gcm(aes)) $(echo $(dd if=/dev/urandom count=20 bs=1 2> /dev/null | xxd -p -c 64)) 128"
```

The + sign in the secret is strongly recommended. It will force the use of per-tunnel IPsec keys. The former global IPsec keys are considered insecure (cf. [GHSA-pwqm-x5x6-5586](https://github.com/cilium/cilium/security/advisories/GHSA-pwqm-x5x6-5586)) and were deprecated in v1.16. When using +, the per-tunnel keys will be derived from the secret you generated.

The secret can be seen with `kubectl -n kube-system get secrets` and will be listed as `cilium-ipsec-keys`.

```bash
kubectl -n kube-system get secrets cilium-ipsec-keys
NAME                TYPE     DATA   AGE
cilium-ipsec-keys   Opaque   1      176m
```

## Enable encryption in Cilium

If you are deploying Cilium with the Cilium CLI, pass the following options:

```bash
cilium install --version 1.17.0 --set encryption.enabled=true --set encryption.type=ipsec
```

When using Cilium in any direct routing configuration, ensure that the native routing CIDR is set properly. This is done using `--ipv4-native-routing-cidr=CIDR` with the CLI or `--set ipv4NativeRoutingCIDR=CIDR` with Helm. At this point the Cilium managed nodes will be using IPsec for all traffic.

## Troubleshooting

- If the `cilium` Pods fail to start after enabling encryption, double-check if the IPsec `Secret` and Cilium are deployed in the same namespace together.
- Check for `level=warning` and `level=error` messages in the Cilium log files:
    - If there is a warning message similar to `Device eth0 does not exist`, use `--set encryption.ipsec.interface=ethX` to set the encryption interface.
- Run `cilium-dbg encrypt status` in the Cilium Pod:
```bash
cilium-dbg encrypt status
Encryption: IPsec
Decryption interface(s): eth0, eth1, eth2
Keys in use: 4
Max Seq. Number: 0x1e3/0xffffffff
Errors: 0
```
If the error counter is non-zero, additional information will be displayed with the specific errors the kernel encountered. If the sequence number reaches its maximum value, it will also result in errors.

## Disabling Encryption

To disable the encryption, regenerate the YAML with the option `encryption.enabled=false`. 

![Alt text](https://cilium.io/static/6cb06fa55abdf88bfe94504fcde73e10/95a1d/mutualAuthDiagram.png)