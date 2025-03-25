# Entity Operator

The Entity Operator in Strimzi is responsible for managing Kafka users and topics. It automates the creation, configuration, and security settings of these entities, ensuring smooth integration with Kafka clusters deployed via Strimzi.

## Key Components of Entity Operator

The Entity Operator consists of two main sub-components:

### Topic Operator

- Watches for KafkaTopic CRs in Kubernetes.
- Automatically creates, updates, and deletes topics in Kafka based on KafkaTopic CR definitions.
- Keeps Kubernetes and Kafka topic configurations in sync.
- Ensures desired state consistency between Kubernetes and Kafka.

### User Operator

- Watches for KafkaUser CRs in Kubernetes.
- Manages security credentials (TLS certificates, SASL credentials).
- Ensures user permissions and authentication are correctly configured.

## Why is the Entity Operator Useful?

- Eliminates the need for manual topic and user management.
- Ensures Kafka users have appropriate authentication and authorization settings.
- Enables declarative management using Kubernetes CRs.
- Keeps configurations between Kubernetes and Kafka in sync.

## How Client Applications Use KafkaTopic and KafkaUser CRs in Strimzi

The client applications interact with Kafka topics and users in Strimzi using Kubernetes native resources

- KafkaTopic CRs define and manage Kafka topics.
- KafkaUser CRs define users and security credentials for authentication & authorization.

## How Applications Use KafkaTopic CRs

### Creating a Topic

Developers define a topic declaratively using a KafkaTopic CR. The Topic Operator ensures this topic is created in Kafka.

**Example KafkaTopic CR**

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: my-topic
  labels:
    strimzi.io/cluster: my-cluster  # Must match the Kafka cluster name
spec:
  partitions: 3
  replicas: 2
  config:
    retention.ms: 86400000  # Data retention for 1 day
    segment.bytes: 1073741824  # 1GB segment size
```

**How clients use it**

Once the topic is created, client applications (producers & consumers) can publish and read messages from `my-topic` like any regular Kafka topic.

## How Applications Use KafkaUser CRs

### Creating a User for Authentication & Authorization

Client applications need a Kafka user to authenticate and communicate securely. A KafkaUser CR defines the user, authentication method (TLS/SCRAM-SHA), and permissions.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: my-app-user
  labels:
    strimzi.io/cluster: my-cluster  # Must match the Kafka cluster name
spec:
  authentication:
    type: tls  # TLS-based auth
  authorization:
    type: simple
    acls:
      - resource:
          type: topic
          name: my-topic
          patternType: literal
        operations: 
          - Read
          - Write
```

**How clients use it**

### Authentication

- If `TLS` authentication is enabled, Strimzi will generate a secret containing the user's TLS certificates.
- If `SCRAM-SHA` authentication is enabled, Strimzi will generate a username and password in a Kubernetes secret.

### Authorization (ACLs)

- In the above example, the user my-app-user has Read & Write access to my-topic.

Clients will only be able to perform allowed operations.

## How Clients Retrieve and Use Credentials

After creating a KafkaUser, Strimzi automatically generates a Kubernetes Secret with the credentials.

**Example**

```bash
kubectl get secret my-app-user -o yaml
```

It will contain

#### For TLS authentication

- ca.crt (CA certificate)
- user.crt (Client certificate)
- user.key (Client private key)

#### For SCRAM-SHA authentication

- password (Base64-encoded password)

### Using These Credentials in a Kafka Client

**Example**

##### Java Producer Example (TLS Authentication)


```java
Properties props = new Properties();
props.put("bootstrap.servers", "my-cluster-kafka-bootstrap:9093");
props.put("security.protocol", "SSL");
props.put("ssl.truststore.location", "/etc/secrets/ca.p12");
props.put("ssl.truststore.password", "password");
props.put("ssl.keystore.location", "/etc/secrets/user.p12");
props.put("ssl.keystore.password", "password");

KafkaProducer<String, String> producer = new KafkaProducer<>(props);
```

#### Java Consumer Example (SCRAM-SHA Authentication)

```java
Properties props = new Properties();
props.put("bootstrap.servers", "my-cluster-kafka-bootstrap:9093");
props.put("security.protocol", "SASL_SSL");
props.put("sasl.mechanism", "SCRAM-SHA-512");
props.put("sasl.jaas.config", "org.apache.kafka.common.security.scram.ScramLoginModule required username='my-app-user' password='my-secret-password';");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);

```

### Summary of How Applications Use KafkaTopic & KafkaUser CRs

| Action    | Operator Responsible |
| -------- | ------- |
| Developer creates a `KafkaTopic` CR  | Topic Operator creates & syncs the topic in Kafka    |
| Developer creates a KafkaUser CR | User Operator creates the user & credentials  |
| Application retrieves credentials from Kubernetes Secrets	| Application mounts the secrets for authentication |
| Application connects to Kafka using these credentials | Producer/Consumer communicates with Kafka  |


## Impact of Central Cluster Failure on Kafka Clients in a Stretch Cluster

In stretch Kafka deployment, where

- ✅ Kafka brokers and controllers are spread across multiple Kubernetes clusters.
- ✅ The central cluster hosts all Kafka CRs, including Kafka, KafkaNodePool, KafkaUser, and KafkaTopic.
- ✅ The Entity Operator (managing users & topics) runs in the central cluster.

## Does the Kafka Cluster Still Function?

Yes, but with conditions.

- Brokers in other Kubernetes clusters are still running.
- Kafka controllers may still be running (if a quorum is maintained).

Replication between brokers (spread across clusters) continues as long as there’s a majority quorum of controllers.

- ✅ If a majority of controllers are still available in the surviving clusters, Kafka continues running.
- 🚨 If the majority of controllers were in the central cluster and lost, Kafka will experience a complete outage.

💡 Mitigation -> user need to ensure the `controller.quorum.voters` are spread across clusters to avoid losing the majority.

## Can Kafka Clients Still Communicate with Brokers?

Yes, if at least one broker remains reachable.

Clients connect to brokers, not the Entity Operator or the central cluster.

- If clients' bootstrap servers list includes brokers running in the surviving clusters, they can still connect.
- If a leader partition for a topic is hosted on a broker in a surviving cluster, the topic remains available.
- If all leader partitions for a topic were on brokers in the central cluster, that topic becomes unavailable.

💡 Mitigation -> Use Kafka’s rack awareness (`broker.rack`) and ISR (In-Sync Replica) balancing to distribute leader partitions across clusters.

## Can Clients Still Produce & Consume Messages?

Yes, if leader partitions are available.
No, if all leader partitions were in the lost central cluster.

**Producing Messages**

- A producer can send messages only if the leader partition for a topic is still available in a surviving cluster.
- If the leader was in the lost central cluster, Kafka automatically elects a new leader (if ISR exists).
- If no ISR exists, production fails.

**Consuming Messages**

- Consumers can still fetch messages from available partitions.
- If consumer group metadata was stored in a lost broker, the group might experience issues.

💡 Mitigation -> Enable `min.insync.replicas` and leader election across clusters to ensure partition availability.


## What About Entity Operator Functions?
The Entity Operator becomes unavailable when the central cluster goes down. However, this does not impact existing Kafka clients directly because

- Kafka clients do not interact with the Entity Operator at runtime.
- User authentication still works as long as secrets (TLS/SCRAM) were distributed to all clusters.
- Topics and ACLs remain intact but cannot be updated or created until the central cluster recovers.

Kafka clients (producers and consumers) can still authenticate and connect to Kafka brokers in the surviving clusters as long as the necessary authentication credentials (secrets) are available in those clusters.

If the KafkaUser secrets only exist in the central cluster, then when it goes down, clients in other clusters cannot authenticate to Kafka brokers. However, if these secrets were already copied to all clusters where brokers are running, authentication will still work even if the central cluster is down.

To avoid authentication failures when the central cluster goes down, you must:

- Replicate KafkaUser secrets across all clusters where Kafka brokers exist.
