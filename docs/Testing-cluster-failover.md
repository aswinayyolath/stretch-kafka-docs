# Cluster Failover Testing
## Objective

This test simulates a scenario in which the central cluster goes down. It evaluates the behavior of Kafka topics, leader election, in-sync replicas (ISRs), and how a stretched Kafka cluster handles failover. After testing the failover, the central cluster is recovered to determine if the old state is restored.

### Cluster setup
We have three Kubernetes clusters with a stretched Kafka deployment across them.
## Cluster Configurations

### Central Cluster (calico-1)
```bash
$ .kube % kubectl get pods --kubeconfig calico-1 -n strimzi
NAME                                        READY   STATUS    RESTARTS   AGE
my-cluster-broker-0                         1/1     Running   0          23h
my-cluster-broker-1                         1/1     Running   0          23h
my-cluster-broker-2                         1/1     Running   0          23h
my-cluster-controller-3                     1/1     Running   0          23h
my-cluster-controller-4                     1/1     Running   0          23h
my-cluster-controller-5                     1/1     Running   0          23h
strimzi-cluster-operator-5b7b9d9bf6-w4hxl   1/1     Running   0          23h
```

### Stretch Cluster 1 (calico-2)

```bash
$ .kube % kubectl get pods --kubeconfig calico-2 -n strimzi
NAME                                        READY   STATUS    RESTARTS   AGE
my-cluster-stretch1-broker-6                1/1     Running   0          23h
my-cluster-stretch1-broker-7                1/1     Running   0          23h
my-cluster-stretch1-broker-8                1/1     Running   0          23h
my-cluster-stretch1-controller-10           1/1     Running   0          23h
my-cluster-stretch1-controller-11           1/1     Running   0          23h
my-cluster-stretch1-controller-9            1/1     Running   0          23h
strimzi-cluster-operator-6d7db9dd95-k5m24   1/1     Running   0          23h
```

### Stretch Cluster 2 (calico-3)

```bash
$ .kube % kubectl get pods --kubeconfig calico-3 -n strimzi
NAME                                        READY   STATUS    RESTARTS   AGE
my-cluster-stretch2-broker-12               1/1     Running   0          23h
my-cluster-stretch2-broker-13               1/1     Running   0          23h
my-cluster-stretch2-broker-14               1/1     Running   0          23h
my-cluster-stretch2-controller-15           1/1     Running   0          23h
my-cluster-stretch2-controller-16           1/1     Running   0          23h
my-cluster-stretch2-controller-17           1/1     Running   0          23h
strimzi-cluster-operator-7966fb9659-zqfmv   1/1     Running   0          23h
```

The central cluster contains the Kafka and KafkaNodePool CRs:
```bash
$ .kube % kubectl get kafka -n strimzi --kubeconfig calico-1
NAME         DESIRED KAFKA REPLICAS   DESIRED ZK REPLICAS   READY   METADATA STATE   WARNINGS
my-cluster
$ .kube % kubectl get kafkanodepool -n strimzi --kubeconfig calico-1
NAME                  DESIRED REPLICAS   ROLES            NODEIDS
broker                3                  ["broker"]       [0,1,2]
controller            3                  ["controller"]   [3,4,5]
stretch1-broker       3                  ["broker"]       [6,7,8]
stretch1-controller   3                  ["controller"]   [9,10,11]
stretch2-broker       3                  ["broker"]       [12,13,14]
stretch2-controller   3                  ["controller"]   [15,16,17]
```

### Listing the metadata quorum
Checking if the metadata shows all the brokers and controllers from all the cluster
```bash
[kafka@my-cluster-broker-0 kafka]$ bin/kafka-metadata-quorum.sh --bootstrap-server my-cluster-kafka-bootstrap.strimzi.svc:9092 describe --status
ClusterId:              1RYWwDxMT8mT0lpiqtc69w
LeaderId:               11
LeaderEpoch:            195814
HighWatermark:          168406
MaxFollowerLag:         0
MaxFollowerLagTimeMs:   54
CurrentVoters:          [16,17,3,4,5,9,10,11,15]
CurrentObservers:       [0,1,2,6,7,8,12,13,14]
```

### Creating a topic for testing
```bash
[kafka@my-cluster-broker-0 kafka]$ bin/kafka-topics.sh --create --bootstrap-server my-cluster-kafka-bootstrap.strimzi.svc:9092 --replication-factor 6 --partitions 6 --topic failover-test
Created topic failover-test.
```

Describing the topics shows that partition 4 and 5 have leaders from the central cluster and partition's all ISRs contains all brokers from the central cluster.
```bash
[kafka@my-cluster-broker-0 kafka]$ bin/kafka-topics.sh --describe --bootstrap-server my-cluster-kafka-bootstrap.strimzi.svc:9092 --topic failover-test
Topic: failover-test	TopicId: 7U-yMkfgT1GfJRY-DoyEhQ	PartitionCount: 6	ReplicationFactor: 6	Configs: min.insync.replicas=2
	Topic: failover-test	Partition: 0	Leader: 8	Replicas: 8,12,13,14,0,1	Isr: 8,12,13,14,0,1	Elr: 	LastKnownElr:
	Topic: failover-test	Partition: 1	Leader: 12	Replicas: 12,13,14,0,1,2	Isr: 12,13,14,0,1,2	Elr: 	LastKnownElr:
	Topic: failover-test	Partition: 2	Leader: 13	Replicas: 13,14,0,1,2,6	Isr: 13,14,0,1,2,6	Elr: 	LastKnownElr:
	Topic: failover-test	Partition: 3	Leader: 14	Replicas: 14,0,1,2,6,7	Isr: 14,0,1,2,6,7	Elr: 	LastKnownElr:
	Topic: failover-test	Partition: 4	Leader: 0	Replicas: 0,1,2,6,7,8	Isr: 0,1,2,6,7,8	Elr: 	LastKnownElr:
	Topic: failover-test	Partition: 5	Leader: 1	Replicas: 1,2,6,7,8,12	Isr: 1,2,6,7,8,12	Elr: 	LastKnownElr:
```

### Producing and consuming messages from the topic 
```bash
[kafka@my-cluster-broker-0 kafka]$ bin/kafka-console-producer.sh --bootstrap-server  my-cluster-kafka-bootstrap.strimzi.svc:9092 --topic failover-test
>asdfasdf
>this is stretch
>sending data from one cluster to the other
>Hello Kafka
>pushing enough messages to kafka cluster
>Hello world
>Testing
>Testing asdf
```

```bash
[kafka@my-cluster-stretch1-broker-8 kafka]$ bin/kafka-console-consumer.sh --bootstrap-server  my-cluster-kafka-bootstrap.strimzi.svc:9092 --topic failover-test
asdfasdf
this is stretch
sending data from one cluster to the other
Hello Kafka
pushing enough messages to kafka cluster
Hello world
Testing
Testing asdf
^CProcessed a total of 8 messages
```

### Shutting down the central kubernetes cluster 
Manually shutting down the central cluster to simulate a cluster failure
```bash
$ .kube % kubectl get pods --kubeconfig calico-1 -n strimzi -v=8
I0313 13:57:19.830013   14803 loader.go:395] Config loaded from file:  calico-1
I0313 13:57:19.834745   14803 round_trippers.go:463] GET https://9.46.88.97:6443/api/v1/namespaces/strimzi/pods?limit=500
I0313 13:57:19.834762   14803 round_trippers.go:469] Request Headers:
I0313 13:57:19.834768   14803 round_trippers.go:473]     Accept: application/json;as=Table;v=v1;g=meta.k8s.io,application/json;as=Table;v=v1beta1;g=meta.k8s.io,application/json
I0313 13:57:19.834771   14803 round_trippers.go:473]     User-Agent: kubectl1.31.1/v1.31.1 (darwin/arm64) kubernetes/948afe5
I0313 13:57:49.837331   14803 round_trippers.go:574] Response Status:  in 30002 milliseconds
I0313 13:57:49.837405   14803 round_trippers.go:577] Response Headers:
I0313 13:57:49.838573   14803 helpers.go:264] Connection error: Get https://9.46.88.97:6443/api/v1/namespaces/strimzi/pods?limit=500: dial tcp 9.46.88.97:6443: i/o timeout
Unable to connect to the server: dial tcp 9.46.88.97:6443: i/o timeout
```

### Testing if we can produce and consume with the other two clusters
We're using cluster 2 and 3 to produce and consume
```bash
[kafka@my-cluster-stretch2-broker-12 kafka]$ bin/kafka-console-producer.sh --bootstrap-server  my-cluster-kafka-bootstrap.strimzi.svc:9092 --topic failover-test
>hello world
>Seems like the data is not lost
```

```bash
[kafka@my-cluster-stretch1-broker-8 kafka]$ bin/kafka-console-consumer.sh --bootstrap-server  my-cluster-kafka-bootstrap.strimzi.svc:9092 --topic failover-test --from-beginning
asdfasdf
this is stretch
sending data from one cluster to the other
Hello Kafka
pushing enough messages to kafka cluster
Hello world
Testing
Testing asdf
hello world
Seems like the data is not lost
^CProcessed a total of 10 messages
```
Produce and consume works fine and it seems like the previously sent messages are being retrieved properly.


### Checking if leader election elected new partition leaders
Partition 4 and 5 which had leaders from the failed central cluster are assigned new leaders from available cluster (cluster 2).
```bash
[kafka@my-cluster-stretch2-broker-12 kafka]$ bin/kafka-topics.sh --describe --bootstrap-server my-cluster-kafka-bootstrap.strimzi.svc:9092 --topic failover-test
Topic: failover-test	TopicId: 7U-yMkfgT1GfJRY-DoyEhQ	PartitionCount: 6	ReplicationFactor: 6	Configs: min.insync.replicas=2
	Topic: failover-test	Partition: 0	Leader: 8	Replicas: 8,12,13,14,0,1	Isr: 8,12,14,13	Elr: 	LastKnownElr:
	Topic: failover-test	Partition: 1	Leader: 12	Replicas: 12,13,14,0,1,2	Isr: 12,14,13	Elr: 	LastKnownElr:
	Topic: failover-test	Partition: 2	Leader: 13	Replicas: 13,14,0,1,2,6	Isr: 14,6,13	Elr: 	LastKnownElr:
	Topic: failover-test	Partition: 3	Leader: 14	Replicas: 14,0,1,2,6,7	Isr: 14,6,7	Elr: 	LastKnownElr:
	Topic: failover-test	Partition: 4	Leader: 6	Replicas: 0,1,2,6,7,8	Isr: 6,7,8	Elr: 	LastKnownElr:
	Topic: failover-test	Partition: 5	Leader: 6	Replicas: 1,2,6,7,8,12	Isr: 6,7,8,12	Elr: 	LastKnownElr:
```

### Recovering the central cluster
Manually recovering the central cluster from failure
```bash
$ .kube % kubectl get pods --kubeconfig calico-1 -n strimzi -w
NAME                                        READY   STATUS    RESTARTS       AGE
my-cluster-broker-0                         0/1     Running   1 (2m6s ago)   24h
my-cluster-broker-1                         0/1     Running   1 (2m9s ago)   23h
my-cluster-broker-2                         0/1     Running   1 (2m9s ago)   24h
my-cluster-controller-3                     1/1     Running   1 (2m9s ago)   24h
my-cluster-controller-4                     1/1     Running   1 (2m6s ago)   24h
my-cluster-controller-5                     1/1     Running   1 (2m9s ago)   24h
strimzi-cluster-operator-5b7b9d9bf6-w4hxl   0/1     Running   1 (2m6s ago)   24h
strimzi-cluster-operator-5b7b9d9bf6-w4hxl   1/1     Running   1 (2m11s ago)   24h
my-cluster-broker-0                         1/1     Running   1 (2m12s ago)   24h
my-cluster-broker-2                         1/1     Running   1 (2m17s ago)   24h
my-cluster-broker-1                         1/1     Running   1 (2m56s ago)   23h
```

### Testing if the ISR's come back from the central clusters
The brokers from the recoered cluster are getting in sync with the partitions and is being reflected in the ISRs
```bash
[kafka@my-cluster-stretch2-broker-12 kafka]$ bin/kafka-topics.sh --describe --bootstrap-server my-cluster-kafka-bootstrap.strimzi.svc:9092 --topic failover-test
Topic: failover-test	TopicId: 7U-yMkfgT1GfJRY-DoyEhQ	PartitionCount: 6	ReplicationFactor: 6	Configs: min.insync.replicas=2
	Topic: failover-test	Partition: 0	Leader: 8	Replicas: 8,12,13,14,0,1	Isr: 0,14,13,12,8	Elr: 	LastKnownElr:
	Topic: failover-test	Partition: 1	Leader: 12	Replicas: 12,13,14,0,1,2	Isr: 0,14,13,2,12	Elr: 	LastKnownElr:
	Topic: failover-test	Partition: 2	Leader: 14	Replicas: 13,14,0,1,2,6	Isr: 0,14,6,13,2	Elr: 	LastKnownElr:
	Topic: failover-test	Partition: 3	Leader: 14	Replicas: 14,0,1,2,6,7	Isr: 0,14,6,2,7	Elr: 	LastKnownElr:
	Topic: failover-test	Partition: 4	Leader: 6	Replicas: 0,1,2,6,7,8	Isr: 0,6,2,7,8	Elr: 	LastKnownElr:
	Topic: failover-test	Partition: 5	Leader: 6	Replicas: 1,2,6,7,8,12	Isr: 6,2,12,7,8	Elr: 	LastKnownElr:
```