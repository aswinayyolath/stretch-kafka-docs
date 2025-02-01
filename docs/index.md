# Stretch Kafka Cluster Documentation

Apache Kafka is widely used for high-throughput, real-time data streaming, but deploying it across multiple Kubernetes clusters presents unique challenges. A Stretch Kafka Cluster is a deployment model where Kafka brokers and controllers are distributed across multiple Kubernetes clusters while maintaining a single logical Kafka cluster. This enables improved fault tolerance, scalability, and disaster recovery while keeping latency low.

This documentation provides step-by-step guidance on setting up a Stretch Kafka Cluster using Submariner and Cilium for cross-cluster networking. The goal is to enable seamless communication between Kafka brokers and controllers running in different Kubernetes clusters, ensuring high availability and resilience.



!!! note

    🚨 This not an offically supported Strimzi feature or documentation.🚨
    
    

## Overview of Stretch Kafka Clusters

### What is a Stretch Kafka Cluster?
A Stretch Kafka Cluster extends a single Kafka deployment across multiple Kubernetes clusters. Unlike a standard Kafka deployment, which runs within a single Kubernetes cluster, a stretch cluster allows brokers and controllers to be distributed across multiple clusters while functioning as a unified Kafka instance.

### Key Benefits of Stretch Kafka Clusters

- **High Availability**: If one Kubernetes cluster fails, Kafka continues to operate using nodes in the remaining K8s clusters.
- **Scalability**: Kafka brokers can be added across multiple K8s clusters to handle increasing workloads.
- **Disaster Recovery**: With brokers in multiple K8s clusters, the system remains operational even if a Kubernetes cluster experiences an outage.
- **Optimized Workload Distribution**: Workloads can be balanced across different K8s clusters for improved resource utilization.

### Deployment Considerations

- **Low-Latency Networking**: Kafka requires low-latency communication. Stretch clusters should be deployed in data centers or availability zones within a single region, avoiding intercontinental deployments.
- **KRaft-based Deployment** This stretch cluster implementation focuses on KRaft mode, as Strimzi and Kafka are transitioning away from Zookeeper.

## Why Submariner and Cilium for Cross-Cluster Communication?

### Challenges in Multi-Cluster Kafka Deployment

Kafka brokers and controllers need stable and low-latency communication across clusters. However, Kubernetes clusters are isolated environments, making cross-cluster networking complex. Some of the key challenges include:

- **Service Discovery**: Brokers in one cluster need to discover and communicate with brokers in another cluster.
- **Network Connectivity**: Kubernetes clusters usually have separate network overlays, preventing direct pod-to-pod communication.
- **Security**: Ensuring secure cross-cluster communication is critical to prevent unauthorized access.

### Why Submariner?
Submariner is a CNCF project that provides secure cross-cluster networking for Kubernetes. It enables:

- **Direct Pod-to-Pod Connectivity**: Allowing Kafka brokers and controllers to communicate seamlessly.
- **Service Discovery Across Clusters**: Making Kafka services reachable from any cluster.
- **Secure Communication**: Using IPsec, WireGuard, or VXLAN for encrypted cross-cluster traffic.

### Why Cilium?
Cilium is an eBPF-based networking solution that enhances Kubernetes security and observability. It provides:

- **Transparent Multi-Cluster Networking**: Using Cilium ClusterMesh for service discovery and connectivity.
- **Enhanced Security**: Enforcing fine-grained security policies across clusters.
- **Performance Optimization**: Using eBPF for high-performance packet processing.

### Which One to Choose?
Both Submariner and Cilium offer cross-cluster networking, but they cater to different use cases:

| Feature    | Submariner | Cilium |
| -------- | ------- | ----------- |
| Best for  | Hybrid and multi-cloud deployments | Kubernetes-native networking |
| Connectivity | Pod-to-pod networking via tunnels   | Transparent multi-cluster service mesh |
| Security | IPsec/WireGuard encryption | eBPF-based security policies |
