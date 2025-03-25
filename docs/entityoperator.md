# Impact of Entity Operator Availability in a Stretch Kafka Cluster

The Entity Operator in Strimzi is responsible for managing Kafka users and topics. It automates the creation, configuration, and security settings of these entities, ensuring smooth integration with Kafka clusters deployed via Strimzi. This document explains how its availability affects topic and user management when deployed in a multi-cluster Kafka setup.

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

✅ Kafka brokers and controllers are spread across multiple Kubernetes clusters.<br>
✅ The central cluster hosts all Kafka CRs, including Kafka, KafkaNodePool, KafkaUser, and KafkaTopic.<br>
✅ The Entity Operator (managing users & topics) runs in the central cluster.


## What About Entity Operator Functions?
The Entity Operator becomes unavailable when the central cluster goes down. However, this does not impact existing Kafka clients directly because

- Kafka clients do not interact with the Entity Operator at runtime.
- User authentication still works as long as secrets (TLS/SCRAM) were distributed to all clusters.
- Topics and ACLs remain intact but cannot be updated or created until the central cluster recovers.

## What Happens If No Cluster Has KafkaUser and KafkaTopic CRs?

If the central cluster is the only one hosting KafkaUser and KafkaTopic CRs, then when it goes down:

1. User Authentication Risks

   - Kafka brokers in surviving clusters rely on existing secrets for authentication.
   - If KafkaUser secrets were only stored in the central cluster and not replicated, brokers in other clusters will be unable to authenticate client requests.
   - New client connections will fail since brokers cannot verify credentials.
   - Existing client connections may remain active if they were authenticated before the central cluster failure, but they will eventually be disconnected when session timeouts occur.

2. Topic Management Limitations

   - Topics that were already created will continue to exist and function normally.
   - Clients can still produce and consume messages only if they are already authenticated before the central cluster failure.
   - No new topics can be created or updated since the KafkaTopic CRs and Entity Operator are unavailable.

### Mitigation Strategies

To ensure Kafka clients remain functional even when the central cluster goes down, we should implement the following best practices

✅ Replicate KafkaUser secrets across all clusters where Kafka brokers exist.

- This ensures authentication remains functional even if the central cluster is unavailable.

✅ Ensure Kafka brokers cache authentication data where possible(This needs verification).

- Some authentication mechanisms (like SCRAM) allow brokers to cache credentials temporarily.
- This can help avoid immediate authentication failures if the central cluster is temporarily down.

✅ Alternatively we can Explore options like KafkaAccess Operator. This reduces dependency on a single cluster for authentication.
