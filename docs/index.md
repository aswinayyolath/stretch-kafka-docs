# Stretch Cluster using Strimzi

Apache Kafka is widely used for high-throughput, real-time data streaming, but deploying it across multiple Kubernetes clusters presents unique challenges. A Stretch Kafka Cluster is a deployment model where Kafka brokers and controllers are distributed across multiple Kubernetes clusters while maintaining a single logical Kafka cluster. This enables improved fault tolerance, scalability, and disaster recovery while keeping latency low.

This documentation provides step-by-step guidance on setting up a Stretch Kafka Cluster using Submariner or Cilium for cross-cluster networking. The goal is to enable seamless communication between Kafka brokers and controllers running in different Kubernetes clusters, ensuring high availability and resilience.



!!! note

    🚨 This not an offically supported Strimzi feature or documentation.🚨
    
    

## Overview of stretch Kafka clusters

### What is a stretch Kafka cluster?
A Stretch Kafka Cluster extends a single Kafka deployment across multiple Kubernetes clusters. Unlike a standard Kafka deployment, which runs within a single Kubernetes cluster, a stretch cluster allows brokers and controllers to be distributed across multiple clusters while functioning as a unified Kafka instance.

### Key benefits of stretch Kafka clusters

- **High Availability and Disaster Recovery**: If one Kubernetes cluster fails, Kafka continues to operate using nodes in the remaining K8s clusters.
- **Scalability**: Kafka brokers can be added across multiple K8s clusters to handle increasing workloads.
- **Optimized Workload Distribution**: Workloads can be balanced across different K8s clusters for improved resource utilization.

### Deployment considerations

- **Low-Latency Networking**: Kafka requires low-latency communication. Stretch clusters should be deployed in data centers or availability zones within a single region, avoiding intercontinental deployments.
- **KRaft-based Deployment** This stretch cluster implementation requires KRaft as both Strimzi and Kafka are transitioning away from Zookeeper.

## Cloud native cross-cluster communication technologies

### Challenges in multi-cluster Kafka deployment

Kafka brokers and controllers need stable and low-latency communication across clusters. However, Kubernetes clusters are isolated environments, making cross-cluster networking complex. Some of the key challenges include:

- **Service discovery**: Brokers in one cluster need to discover and communicate with brokers in another cluster.
- **Network connectivity**: Kubernetes clusters usually have separate network overlays, preventing direct pod-to-pod communication.
- **Security**: Ensuring secure cross-cluster communication is critical to prevent unauthorized access.

### Why Submariner?
Submariner is a CNCF project that provides secure cross-cluster networking for Kubernetes. It enables:

- **Direct pod-to-pod connectivity**: Allowing Kafka brokers and controllers to communicate seamlessly.
- **Service discovery across Kubernetes cluster boundaries**: Kafka services become reachable from any cluster.
- **Secure communication**: Using IPsec, WireGuard, or VXLAN for encrypted cross-cluster traffic.

### Why Cilium?
Cilium is an eBPF-based networking solution that enhances Kubernetes security and observability. It provides:

- **Transparent multi-cluster networking**: Using Cilium ClusterMesh for service discovery and connectivity.
- **Enhanced security**: Enforcing fine-grained security policies across clusters.
- **Performance optimization**: Using eBPF for high-performance packet processing.

### Which one to choose?
Both Submariner and Cilium offer cross-cluster networking, but they cater to different use cases:

| Feature    | Submariner | Cilium |
| -------- | ------- | ----------- |
| Best for  | Hybrid and multi-cloud deployments | Kubernetes-native networking |
| Connectivity | Pod-to-pod networking via tunnels   | Transparent multi-cluster service mesh |
| Security | IPsec/WireGuard encryption | eBPF-based security policies |
