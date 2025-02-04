## Setting up Submariner with OpenShift

This section explains setting up Submariner with OpenShift clusters using the GlobalNet controller.

### Before you begin

The following steps assume that you are creating a stretch cluster across three OpenShift clusters and have created three separate kubeconfig files, one for each OpenShift cluster. Each file should only contain a reference to a single cluster. Only one of these clusters will host the Submariner broker.

### Deploying the broker

Deploy the broker to one OpenShift cluster. You specify the OpenShift cluster that will host the broker by passing the appropriate kubeconfig file to the command:

```bash
$ subctl deploy-broker --globalnet --kubeconfig config-str2-a
```

Once deployed, a `broker-info.subm` file will be generated containing authentication and connection details.


!!! Globalnet configuration
    Most OpenShift clusters created using automation jobs share the same Pod and Service CIDRs. To prevent conflicts, the GlobalNet controller assigns unique, non-overlapping IPs to each cluster.

### Joining clusters to the broker

Assign a unique clusterid to each OpenShift cluster:

!!! Cluster-ID
    These clusterid values must match the values used in the KafkaNodePool (KNP) CR when deploying Strimzi in stretch mode.

```bash
$ subctl join --kubeconfig config-str2-a --clusterid cluster1  broker-info.subm --check-broker-certificate=false
```

```bash
$ subctl join --kubeconfig config-str2-b --clusterid cluster2  broker-info.subm --check-broker-certificate=false
```

```bash
$ subctl join --kubeconfig config-str2-c --clusterid cluster3  broker-info.subm --check-broker-certificate=false
```

### Testing cluster connectivity

Check if the clusters are correctly connected:

```bash
$ subctl show connections --kubeconfig config-str2-a
Cluster "<REDACTED>:6443"

 ✓ Showing Connections

GATEWAY              CLUSTER    REMOTE IP      NAT   CABLE DRIVER   SUBNETS        STATUS      RTT avg.
worker0.<REDACTED>   cluster2   10.13.26.218   no    libreswan      242.1.0.0/16   connected   1.2786ms
worker0.<REDACTED>   cluster3   10.15.133.94   no    libreswan      242.2.0.0/16   connected   614.899µs
```

This confirms successful cross-cluster communication.
