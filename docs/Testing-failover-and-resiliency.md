# Failover Testing
This document provides a detailed guide on testing failover scenarios, leader election, and topic availability in a multi-cluster Kafka deployment. 

### Cluster Setup

The Kafka clusters are deployed across multiple Kubernetes clusters, as shown below

```bash
$ kubectl get pods -n real --kubeconfig config-test-a
NAME                                                READY   STATUS    RESTARTS   AGE
my-cluster-broker-0                                 1/1     Running   0          3h56m
my-cluster-broker-1                                 1/1     Running   0          3h56m
my-cluster-broker-2                                 1/1     Running   0          3h56m
my-cluster-controller-3                             1/1     Running   0          3h55m
my-cluster-controller-4                             1/1     Running   0          3h55m
my-cluster-controller-5                             1/1     Running   0          3h55m
strimzi-cluster-operator-v0.44.0-6ff79bb868-vws4n   1/1     Running   0          4h20m
$ kubectl get pods -n real --kubeconfig config-test-b
NAME                                                READY   STATUS    RESTARTS   AGE
my-cluster-stretch1-broker-6                        1/1     Running   0          3h56m
my-cluster-stretch1-broker-7                        1/1     Running   0          3h56m
my-cluster-stretch1-broker-8                        1/1     Running   0          3h56m
my-cluster-stretch1-controller-10                   1/1     Running   0          3h56m
my-cluster-stretch1-controller-11                   1/1     Running   0          3h56m
my-cluster-stretch1-controller-9                    1/1     Running   0          3h56m
strimzi-cluster-operator-v0.44.0-78979ffb66-hkjf6   1/1     Running   0          2d17h
$ kubectl get pods -n real --kubeconfig config-test-c
NAME                                                READY   STATUS    RESTARTS   AGE
my-cluster-stretch2-broker-12                       1/1     Running   0          3h56m
my-cluster-stretch2-broker-13                       1/1     Running   0          3h56m
my-cluster-stretch2-broker-14                       1/1     Running   0          3h56m
my-cluster-stretch2-controller-15                   1/1     Running   0          3h56m
my-cluster-stretch2-controller-16                   1/1     Running   0          3h56m
my-cluster-stretch2-controller-17                   1/1     Running   0          3h56m
strimzi-cluster-operator-v0.44.0-6c4f95869b-2qgjg   1/1     Running   0          2d17h
```

## Topic availability and leader election testing

### Create a topic with all brokers participating
```bash
sh-5.1$ ./kafka-topics.sh --create --bootstrap-server my-cluster-kafka-bootstrap:9092 --replication-factor 9 --partitions 3 --topic long-replicated-topic
Created topic long-replicated-topic.
```

### Describe topic
```bash
sh-5.1$ ./kafka-topics.sh --describe --bootstrap-server my-cluster-kafka-bootstrap:9092  --topic long-replicated-topic
Topic: long-replicated-topic	TopicId: 0qzO6dCFRiy4n3aVgPZ-gA	PartitionCount: 3	ReplicationFactor: 9	Configs: min.insync.replicas=2
	Topic: long-replicated-topic	Partition: 0	Leader: 8	Replicas: 8,12,13,14,0,1,2,6,7	Isr: 8,12,13,14,0,1,2,6,7	Elr: 	LastKnownElr:
	Topic: long-replicated-topic	Partition: 1	Leader: 12	Replicas: 12,13,14,0,1,2,6,7,8	Isr: 12,13,14,0,1,2,6,7,8	Elr: 	LastKnownElr:
	Topic: long-replicated-topic	Partition: 2	Leader: 13	Replicas: 13,14,0,1,2,6,7,8,12	Isr: 13,14,0,1,2,6,7,8,12	Elr: 	LastKnownElr:
```
Partition 0, 1 and 2 have leaders as 8, 12 and 13 respectively. We're going to delete some of these leader brokers to see if new partition leaders are elected to another available broker.

### Deleting broker 8 to see leadership transition
```bash
$ kubectl delete pod my-cluster-stretch1-broker-8 -n real --kubeconfig config-test-b
pod "my-cluster-stretch1-broker-8" deleted
```
```bash
sh-5.1$ ./kafka-topics.sh --describe --bootstrap-server my-cluster-kafka-bootstrap:9092  --topic long-replicated-topic
Topic: long-replicated-topic	TopicId: 0qzO6dCFRiy4n3aVgPZ-gA	PartitionCount: 3	ReplicationFactor: 9	Configs: min.insync.replicas=2
	Topic: long-replicated-topic	Partition: 0	Leader: 12	Replicas: 8,12,13,14,0,1,2,6,7	Isr: 12,13,14,0,1,2,6,7	Elr: 	LastKnownElr:
	Topic: long-replicated-topic	Partition: 1	Leader: 12	Replicas: 12,13,14,0,1,2,6,7,8	Isr: 12,13,14,0,1,2,6,7	Elr: 	LastKnownElr:
	Topic: long-replicated-topic	Partition: 2	Leader: 13	Replicas: 13,14,0,1,2,6,7,8,12	Isr: 13,14,0,1,2,6,7,12	Elr: 	LastKnownElr:
```
We can see that leadership has been transitioned to 12

### Deleting broker 12 to see leadership transition
```bash
$ kubectl delete pod my-cluster-stretch2-broker-12 -n real --kubeconfig config-test-c
pod "my-cluster-stretch2-broker-12" deleted
```
```bash
sh-5.1$ ./kafka-topics.sh --describe --bootstrap-server my-cluster-kafka-bootstrap:9092  --topic long-replicated-topic
Topic: long-replicated-topic	TopicId: 0qzO6dCFRiy4n3aVgPZ-gA	PartitionCount: 3	ReplicationFactor: 9	Configs: min.insync.replicas=2
	Topic: long-replicated-topic	Partition: 0	Leader: 8	Replicas: 8,12,13,14,0,1,2,6,7	Isr: 0,14,1,6,13,2,7,8	Elr: 	LastKnownElr:
	Topic: long-replicated-topic	Partition: 1	Leader: 13	Replicas: 12,13,14,0,1,2,6,7,8	Isr: 0,14,1,6,13,2,7,8	Elr: 	LastKnownElr:
	Topic: long-replicated-topic	Partition: 2	Leader: 13	Replicas: 13,14,0,1,2,6,7,8,12	Isr: 0,14,1,6,13,2,7,8	Elr: 	LastKnownElr:
```

Conclusion: The topic remains stable and leader election happens normally even after the leader has been deleted

## Controller leader election test
The controllers across stretch cluster must have an active leader. If a leader controller goes down, any available controller from any cluster should be elected as leader.

### Checking Initial Leader
```bash
sh-5.1$ ./kafka-metadata-quorum.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 describe --status
ClusterId:              6LKJ0UFeRwKeyU_G1HdZxg
LeaderId:               5
LeaderEpoch:            116
HighWatermark:          30543
MaxFollowerLag:         0
MaxFollowerLagTimeMs:   295
CurrentVoters:          [16,17,3,4,5,9,10,11,15]
CurrentObservers:       [0,1,2,6,7,8,12,13,14]
```
The current leader is controller 5. If controller 5 goes down, a new controller must be elected as leader.

### Deleting Controller 5
```bash
$ kubectl delete pod my-cluster-controller-5 -n real --kubeconfig config-test-a
pod "my-cluster-controller-5" deleted
```

### New Leader Election Output
```bash
sh-5.1$ ./kafka-metadata-quorum.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 describe --status
ClusterId:              6LKJ0UFeRwKeyU_G1HdZxg
LeaderId:               4
LeaderEpoch:            118
HighWatermark:          30960
MaxFollowerLag:         30961
MaxFollowerLagTimeMs:   -1
CurrentVoters:          [16,17,3,4,5,9,10,11,15]
CurrentObservers:       [0,1,2,6,7,8,12,13,14]
```
The new leader is controller 4

### Deleting Controller 4
```bash
$ kubectl delete pod my-cluster-controller-4 -n real --kubeconfig config-test-a
pod "my-cluster-controller-4" deleted
```

### New Leader Election Output
```bash
sh-5.1$ ./kafka-metadata-quorum.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 describe --status
ClusterId:              6LKJ0UFeRwKeyU_G1HdZxg
LeaderId:               3
LeaderEpoch:            120
HighWatermark:          31046
MaxFollowerLag:         31047
MaxFollowerLagTimeMs:   -1
CurrentVoters:          [16,17,3,4,5,9,10,11,15]
CurrentObservers:       [0,1,2,6,7,8,12,13,14]
```

These results however shows leader election works inside a cluster only, hence we deleted even more leaders to see if leader election works across cluster.

### Checking cross cluster leader election
Current leader is 5
```bash
sh-5.1$ ./kafka-metadata-quorum.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 describe --status
ClusterId:              6LKJ0UFeRwKeyU_G1HdZxg
LeaderId:               5
LeaderEpoch:            122
HighWatermark:          31124
MaxFollowerLag:         31125
MaxFollowerLagTimeMs:   -1
CurrentVoters:          [16,17,3,4,5,9,10,11,15]
CurrentObservers:       [0,1,2,6,7,8,12,13,14]
```
Deleting leader 5
```bash
$ kubectl delete pod my-cluster-controller-5 -n real --kubeconfig config-test-a
pod "my-cluster-controller-5" deleted
```
New leader 9
```bash
sh-5.1$ ./kafka-metadata-quorum.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 describe --status
ClusterId:              6LKJ0UFeRwKeyU_G1HdZxg
LeaderId:               9
LeaderEpoch:            124
HighWatermark:          31186
MaxFollowerLag:         0
MaxFollowerLagTimeMs:   271
CurrentVoters:          [16,17,3,4,5,9,10,11,15]
CurrentObservers:       [0,1,2,6,7,8,12,13,14]
```
Conclusion: Cross cluster controller leader election works seamlessly in a stretch cluster setup