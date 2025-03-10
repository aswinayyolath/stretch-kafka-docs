# Rack Awareness in Stretch Clusters

## Overview
Kubernetes uses zone-aware scheduling to distribute pod replicas across different zones for improved fault tolerance. Some Kubernetes clusters, such as AWS-based clusters, already have built-in zone awareness. However, for clusters that are not zone-aware, each worker node must be manually labeled with a zone.

## Configuring Rack Awareness
To enable rack awareness in a Stretch Kafka cluster, follow these steps:

### Step 1: Label Worker Nodes
Label each Kubernetes worker node with its respective zone using the following command:

```bash
🔥🔥🔥 $ kubectl label node <workernodename> topology.kubernetes.io/zone=<value>
```

Repeat this command for all worker nodes in each cluster involved in the Stretch setup.

**Example: Central Cluster**

```bash
🔥🔥🔥 $ kubectl get node --selector='!node-role.kubernetes.io/master' --show-labels | awk '{print $6}'
```

**Output:**

```bash
beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker0.example.com,kubernetes.io/os=linux,node-role.kubernetes.io/worker=,topology.kubernetes.io/zone=cluster1
beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker1.example.com,kubernetes.io/os=linux,node-role.kubernetes.io/worker=,topology.kubernetes.io/zone=cluster1
```

**Example: Remote Clusters**

```bash
🔥🔥🔥 $ kubectl get node --kubeconfig cluster-a-config --selector='!node-role.kubernetes.io/master' --show-labels | awk '{print $6}'
```

**Output:**

```bash
beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker0.remote1.example.com,kubernetes.io/os=linux,node-role.kubernetes.io/worker=,topology.kubernetes.io/zone=cluster2
```

Repeat the above steps for all remote clusters involved.

### Step 2: Update Kafka Custom Resource
Once all worker nodes are labeled, update the Kafka CR in the Central Cluster to include rack awareness settings:

```yaml
spec:
  kafka:
    rack:
      topologyKey: topology.kubernetes.io/zone
```

### Step 3: Configure ClusterRoleBinding
Strimzi creates a ClusterRoleBinding in the Central Cluster to allow necessary permissions for fetching node details. This must be manually created in the remote clusters for now (**Note**: Once we start real implementation we can make code changes such that this `ClusterRoleBinding` will be automatically created in the remote cluster by the Operator running in the central cluster )

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: strimzi-my-cluster-kafka-init
  labels:
    app.kubernetes.io/instance: my-cluster
    app.kubernetes.io/managed-by: strimzi-cluster-operator
    app.kubernetes.io/name: kafka
    app.kubernetes.io/part-of: strimzi-my-cluster
    strimzi.io/cluster: my-cluster
    strimzi.io/component-type: kafka
    strimzi.io/kind: Kafka
    strimzi.io/name: my-cluster-kafka
subjects:
  - kind: ServiceAccount
    name: my-cluster-kafka
    namespace: kafka-namespace
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: strimzi-kafka-broker
```

## Verifying Rack Awareness

### Create a Topic

```bash
🔥🔥🔥 $ bin/kafka-topics.sh --create --bootstrap-server my-cluster-kafka-bootstrap.kafka-namespace.svc.cluster.local:9092 --replication-factor 3 --partitions 6 --topic test-topic-1
```

### Describe the Topic

```bash
🔥🔥🔥 $ bin/kafka-topics.sh --describe --bootstrap-server my-cluster-kafka-bootstrap.kafka-namespace.svc.cluster.local:9092 --topic test-topic-1
Example Output:

Topic: test-topic-1     PartitionCount: 6       ReplicationFactor: 3
Partition: 0    Leader: 2       Replicas: 2,14,6        Isr: 2,14,6
Partition: 1    Leader: 12      Replicas: 12,7,0        Isr: 12,7,0
Partition: 2    Leader: 8       Replicas: 8,1,13        Isr: 8,1,13
```

### Check Broker Configuration

```bash
🔥🔥🔥 $ cat /opt/kafka/init/rack.id
Output for brokers in different clusters:

cluster1
cluster2
cluster3
```

### Verify Broker ConfigMaps Contain Rack ID Settings

```yaml
    ##########
    # Rack ID
    ##########
    broker.rack=${strimzidir:/opt/kafka/init:rack.id}
```

Strimzi typically sets this environment variable in the broker pod configMap based on rackId property in the Kafka CR.

### Verify rack details using kafka-broker-api-versions.sh

```bash
sh-5.1$ bin/kafka-broker-api-versions.sh --bootstrap-server my-cluster-kafka-bootstrap.real.svc:9092 
my-cluster-stretch2-broker-12.cluster2.my-cluster-kafka-brokers.real.svc.clusterset.local:9092 (id: 12 rack: cluster3) -> (
        Produce(0): 0 to 11 [usable: 11],
        ......
)
my-cluster-stretch2-broker-13.cluster2.my-cluster-kafka-brokers.real.svc.clusterset.local:9092 (id: 13 rack: cluster3) -> (
        Produce(0): 0 to 11 [usable: 11],
        ......
)
my-cluster-stretch1-broker-7.cluster3.my-cluster-kafka-brokers.real.svc.clusterset.local:9092 (id: 7 rack: cluster1) -> (
        Produce(0): 0 to 11 [usable: 11],
        ......
)
my-cluster-broker-1.cluster1.my-cluster-kafka-brokers.real.svc.clusterset.local:9092 (id: 1 rack: cluster2) -> (
        Produce(0): 0 to 11 [usable: 11],
        ......
)
my-cluster-stretch1-broker-6.cluster3.my-cluster-kafka-brokers.real.svc.clusterset.local:9092 (id: 6 rack: cluster1) -> (
        ......
)
my-cluster-stretch1-broker-8.cluster3.my-cluster-kafka-brokers.real.svc.clusterset.local:9092 (id: 8 rack: cluster1) -> (
        Produce(0): 0 to 11 [usable: 11],
        ......
)
my-cluster-broker-0.cluster1.my-cluster-kafka-brokers.real.svc.clusterset.local:9092 (id: 0 rack: cluster2) -> (
        Produce(0): 0 to 11 [usable: 11],
        ......
)
my-cluster-broker-2.cluster1.my-cluster-kafka-brokers.real.svc.clusterset.local:9092 (id: 2 rack: cluster2) -> (
        Produce(0): 0 to 11 [usable: 11],
        ......
)
my-cluster-stretch2-broker-14.cluster2.my-cluster-kafka-brokers.real.svc.clusterset.local:9092 (id: 14 rack: cluster3) -> (
        Produce(0): 0 to 11 [usable: 11],
        ......
)
sh-5.1$ 
```

- This command lists all Kafka brokers along with their rack information.
- The rack ID is displayed next to each broker (rack: clusterX), confirming that the rack-aware configuration is correctly applied.

With this setup, Kafka can distribute replicas across different zones, ensuring resilience in a multi-cluster deployment.