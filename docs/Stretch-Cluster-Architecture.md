# Stretch cluster architecture

In this section, we will discuss the architecture of a stretch Kafka cluster.

![stretch-architecture](image/image.png){ loading=lazy }

The stretch cluster design extends the Strimzi Kafka operator to support the distribution of brokers and controllers across multiple Kubernetes clusters. The primary goal is to enhance the high availability of the Kafka data plane.

The above diagram outlines the high-level topology and design concepts for such a deployment. Stretch Kafka clusters require multiple Kubernetes clusters: users can configure any number of clusters, but for simplicity, the diagram illustrates a three-cluster setup.

Kafka clusters should be deployed in environments that allow low-latency communication between brokers and controllers. Stretch Kafka clusters are best suited for data centers or availability zones within a single region. They should not be deployed across geographically distant regions where high latency could degrade performance.

## Central vs. Member (Remote) Clusters

In a stretch cluster setup:

- `Kafka` and `KafkaNodePool` Custom Resources (CRs) are applied in a single Kubernetes cluster, known as the "central" cluster.
- The Cluster Operator (CO) runs in the central cluster and is responsible for creating Kubernetes resources required to bring up Kafka broker and controller pods in member (remote) clusters.
- Cluster operators must be deployed in all Kubernetes clusters.
    - The cluster operator in the central cluster manages all required resources across clusters.
    - The cluster operator in remote clusters ensures that Kafka pods are restarted if they crash.

## Handling failures

The stretch cluster architecture ensures fault tolerance even if one or more Kubernetes clusters experience an outage. If a remote cluster fails, the remaining clusters continue serving Kafka clients.

Additionally, if the central cluster goes down, Kafka operations remain unaffected, as the Kafka pods in the remaining clusters continue to function and serve clients without disruption.
