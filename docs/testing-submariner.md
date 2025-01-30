# Validating Submariner Setup

## Testing connections between cluseters
```bash
$ subctl show connections --kubeconfig config-str2-a
Cluster "<REDACTED>:6443"

 ✓ Showing Connections

GATEWAY              CLUSTER    REMOTE IP      NAT   CABLE DRIVER   SUBNETS        STATUS      RTT avg.
worker0.<REDACTED>   cluster2   10.13.26.218   no    libreswan      242.1.0.0/16   connected   1.2786ms
worker0.<REDACTED>   cluster3   10.15.133.94   no    libreswan      242.2.0.0/16   connected   614.899µs
```

## Diagnosing issues with the submariner setup
```bash
$ subctl diagnose all --kubeconfig config-str2-a
Cluster "<REDACTED>:6443"
 ✓ Checking Submariner support for the Kubernetes version
 ✓ Kubernetes version "v1.28.15+ff493be" is supported

 ✓ Globalnet deployment detected - checking that globalnet CIDRs do not overlap
 ✓ Checking DaemonSet "submariner-gateway"
 ✓ Checking DaemonSet "submariner-routeagent"
 ✓ Checking DaemonSet "submariner-globalnet"
 ✓ Checking DaemonSet "submariner-metrics-proxy"
 ✓ Checking Deployment "submariner-lighthouse-agent"
 ✓ Checking Deployment "submariner-lighthouse-coredns"
 ✓ Checking the status of all Submariner pods
 ✓ Checking that gateway metrics are accessible from non-gateway nodes
 ✓ Checking that globalnet metrics are accessible from non-gateway nodes

 ✓ Checking Submariner support for the CNI network plugin
 ✓ The detected CNI network plugin ("OVNKubernetes") is supported
 ✓ Checking OVN version
 ✓ The ovn-nb database version 7.1.0 is supported
 ✓ Checking gateway connections
 ✓ Checking Submariner support for the kube-proxy mode
 ✓ Cluster is running with "OVNKubernetes" CNI which internally implements kube-proxy functionality
 ✓ Checking that firewall configuration allows intra-cluster VXLAN traffic
 ✗ Checking that Globalnet is correctly configured and functioning
 ✗ No matching GlobalIngressIP resource found for exported service "real/my-cluster-kafka-brokers"

 ✓ Checking that services have been exported properly

Skipping inter-cluster firewall check as it requires two kubeconfigs. Please run "subctl diagnose firewall inter-cluster" command manually.

subctl version: v0.18.0
```

## Diagnosing intra cluster network connectivity
```bash
$ subctl diagnose firewall intra-cluster --kubeconfig config-str2-a
Cluster "<REDACTED>:6443"
 ✓ Checking that firewall configuration allows intra-cluster VXLAN traffic
```