## Setting up submariner on ocp

This section documents the use of submariner on three ocp clusters using the globalnet controller. 
<br>
We have 3 ocp clusters which can be connected through 3 config files `config-str2-a` `config-str2-b` and `config-str2-c`

### Deploy the broker
One of the ocp clusters are selected to deploy the broker. 
```bash
$ subctl deploy-broker --globalnet --kubeconfig config-str2-a
```
Once the broker is deployed on this cluster, a `broker-info.subm` file will be saved in the current working directory which contains the details and authentication details of the broker which can be used by the clusters to join into this submariner mesh.

!!! Globalnet configuration
    Most openshift cluster that we have access to, have the same Pod and Service CIDRs. For this to work with submariner, we use a globalnet controller to overlay a network with non overlapping IP address space in the mesh

### Join clusters to broker
Join the ocp clusters to the broker. Give unique cluster ids for each of these brokers. This property must be specified in the KNP CR while we install strimzi in stretch mode.

!!! Cluster-ID
    Take a note of the clusterID given for each ocp cluster as this will be required for the KafkaNodePool CR while creating the node pools in stretch mode. Each cluster ID should properly point to the correct ocp cluster. More on this in the <TODO> section 

```bash
$ subctl join --kubeconfig config-str2-a --clusterid cluster1  broker-info.subm --check-broker-certificate=false
```

```bash
$ subctl join --kubeconfig config-str2-b --clusterid cluster2  broker-info.subm --check-broker-certificate=false
```

```bash
$ subctl join --kubeconfig config-str2-c --clusterid cluster3  broker-info.subm --check-broker-certificate=false
```

### Test connections between clusters
Once the clusters have been joined to the broker, we can test the connections with the `show connections` command
```bash
$ subctl show connections --kubeconfig config-str2-a
Cluster "<REDACTED>:6443"

 ✓ Showing Connections

GATEWAY              CLUSTER    REMOTE IP      NAT   CABLE DRIVER   SUBNETS        STATUS      RTT avg.
worker0.<REDACTED>   cluster2   10.13.26.218   no    libreswan      242.1.0.0/16   connected   1.2786ms
worker0.<REDACTED>   cluster3   10.15.133.94   no    libreswan      242.2.0.0/16   connected   614.899µs
```
The output for this command should contain all the clusters that are connnected to this cluster.