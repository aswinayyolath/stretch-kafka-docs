# Impact of Entity Operator Availability in a Stretched Kafka Cluster

This document outlines the role of the Strimzi Entity Operator in managing Kafka users and topics and explains how its availability affects operations within a multi-cluster Kafka deployment where the Entity Operator resides in a central Kubernetes cluster.

## Key Components of the Entity Operator

The Entity Operator in Strimzi comprises two primary sub-components:

### Topic Operator

- Monitors Kubernetes for `KafkaTopic` CRs.
- Automatically creates, updates, and deletes Kafka topics within the Kafka cluster based on the definitions in the `KafkaTopic` CRs.
- Ensures synchronization between the desired topic configurations in Kubernetes and the actual topic configurations in Kafka.

### User Operator

- Monitors Kubernetes for `KafkaUser` CRs.
- Manages security credentials (e.g., TLS certificates, SASL credentials) and configures user permissions and authentication within the Kafka cluster.
- Automates the provisioning and synchronization of Kafka user authentication and authorization settings.

## Why is the Entity Operator Essential?

The Entity Operator provides several key benefits:

- Eliminates the need for manual topic and user management through Kafka's administrative tools.
- Ensures Kafka users are configured with appropriate authentication and authorization settings as defined in Kubernetes.
- Enables the management of Kafka resources using Kubernetes-native Custom Resources, promoting a declarative approach.
- Maintains consistency between the desired state in Kubernetes and the actual state of topics and users in Kafka.

## How Client Applications Utilize KafkaTopic and KafkaUser CRs in Strimzi

Client applications interact with Kafka topics and users in Strimzi using Kubernetes Custom Resources:

- **`KafkaTopic` CRs:** Define and manage Kafka topics, specifying parameters like partitions, replication factor, and configuration.
- **`KafkaUser` CRs:** Define users and their security configurations for authentication and authorization.

## How Applications Utilize `KafkaTopic` CRs

### Creating a Topic

Developers define Kafka topics declaratively using `KafkaTopic` CRs. The Topic Operator ensures the creation of these topics within the Kafka cluster.

**Example `KafkaTopic` CR:**

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
    retention.ms: 86400000        # Data retention for 1 day
    segment.bytes: 1073741824     # 1GB segment size
```

**How Clients Use It**

Once the `my-topic` is created by the Topic Operator, client applications (producers and consumers) can publish and read messages from it as they would with any regular Kafka topic, provided they have the necessary permissions.

## How Applications Utilize `KafkaUser` CRs

### Creating a User for Authentication & Authorization

Client applications require a Kafka user to authenticate and communicate securely. A KafkaUser CR defines the user, the authentication method (e.g., TLS or SCRAM-SHA), and the permissions they should have.

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

*Note:* The Kubernetes Secret is primarily used to distribute the initial SCRAM username and password to clients. However, Kafka brokers internally store hashed verifiers of these credentials. When clients authenticate, they send their credentials to Kafka, which verifies them against the stored hashed verifiers rather than using the plaintext password from the Secret.

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


**Impact on Clients:**

The fact that the User Operator manages credentials and ACLs through Kafka's standard mechanisms means that the availability of the User Operator is crucial for:

- Creating and managing new user identities within Kafka.
- Ensuring that the correct authentication credentials are in place and accessible.
- Defining and enforcing authorization rules for user access to topics.

When the Central cluster (and thus the User Operator) is unavailable, the ability to perform these management tasks is lost, directly impacting the ability of clients to authenticate and operate with the expected level of access. While the underlying Kafka authentication and authorization capabilities exist within the brokers, the management and provisioning through the Kubernetes control plane are disrupted. This means that administrators will not be able to create, update, or delete Kafka users and topics, including performing credential rotations and ACL updates.


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


The failure of the central Kubernetes cluster will render the Entity Operator unavailable. This has the following implications for Kafka clients

#### Authentication

- Kafka brokers in the surviving member clusters rely on the configured authentication mechanisms and the presence of valid credentials for client authentication.
- If TLS certificates secrets are not replicated across all clusters, new client deployments and credential updates will fail. However, existing clients with valid TLS certificates will continue functioning until their certificates expire or require rotation. The duration for which they can operate depends entirely on the expiration date set when the certificates were issued.
- If SCRAM credentials have been successfully replicated across the Kafka brokers, existing clients should be able to continue authenticating, even if the Central cluster is down. The issue is with new client deployments or credential updates.
- Existing client connections that were authenticated before the central cluster failure might remain active for a period, but their continued operation depends on multiple factors. If a Kafka broker restarts, clients may need to re-authenticate, which could fail if they rely on new credentials from an unavailable Entity Operator. Additionally, the configured `session.timeout.ms` and Kafka’s reauthentication behavior may determine how long clients remain connected before being disconnected.
- Crucially, the management of credentials (e.g., rotation) through the User Operator will be unavailable.

#### Authorization

- The ACLs defined in KafkaUser CRs are configured on the Kafka brokers. These ACLs will generally remain in place.
- However, any new authorization rules or modifications to existing ones defined in KafkaUser CRs cannot be applied because the User Operator is down.
- TLS certificates used for authentication expire and rotate periodically. Without the User Operator, expired certificates cannot be renewed, leading to eventual authentication failures.

#### Topic Management

- Topics that were already created will continue to exist and function normally.
- Clients can continue to produce and consume messages on existing topics if they remain authenticated and authorized.
- Existing topics will continue to function, but administrators cannot modify topic configurations or delete topics through Kubernetes.
- No new topics can be created or updated through the Kubernetes-managed KafkaTopic CRs since the Topic Operator is unavailable.


#### Client Authentication & Secret Management in a Stretched Kafka Deployment

In a multi-cluster Kafka deployment, clients (producers/consumers) may run in different environments, each with its own authentication challenges. Below, we define what it means for clients running in different locations and how they manage Kafka authentication.

##### Clients Running in the Central Kubernetes Cluster

These are Kafka clients (applications) deployed in the same cluster where the Entity Operator runs.

**How They Authenticate & Connect**

- Credentials (e.g., username/password for SCRAM, TLS certificates) are automatically generated by the Entity Operator.
- Credentials are stored in Kubernetes Secrets within the same namespace.
- Kafka clients retrieve authentication details from these Kubernetes Secrets.
- They connect via internal Kubernetes service names (e.g., my-cluster-kafka-bootstrap:9092).

**Example Configuration (SCRAM Authentication)***

```yaml
env:
  - name: KAFKA_BOOTSTRAP_SERVERS
    value: "my-cluster-kafka-bootstrap:9092"
  - name: KAFKA_SASL_USERNAME
    valueFrom:
      secretKeyRef:
        name: my-user
        key: sasl.jaas.config
  - name: KAFKA_SSL_TRUSTSTORE_PASSWORD
    valueFrom:
      secretKeyRef:
        name: my-user
        key: truststore-password
```
**Note**: The values are not hardcoded, but are being pulled from the secret named 'my-user'.

✅ Key Takeaways

✔ Credentials are managed via Kubernetes Secrets, requiring no manual configuration.<br>
✔ Clients access Kafka via internal networking, making communication seamless.<br>
✔ Works out of the box without extra setup.<br>

##### Clients Running in a Remote Kubernetes Cluster

These are Kafka clients running in Kubernetes clusters that are **Not** the central cluster (where the Entity Operator runs).

**Challenges They Face**

🚨 The KafkaUser Secret does not exist in the remote cluster because the Entity Operator only runs in the central cluster.<br>
🚨 Kubernetes-based applications expect credentials to be stored as Secrets within their own cluster.<br>

**Potential Solutions**

✅ Manually Sync Secrets → Copy KafkaUser secrets from the central cluster to the remote cluster.<br>
✅ Use Kafka Access Operator (Future Feature) → Strimzi maintainers are working on cross-cluster user management.<br>
✅ Treat Clients as External Clients → Fetch credentials manually and connect via external listeners.


##### Clients Running Outside Kubernetes (External Clients)

These are Kafka clients running outside of any Kubernetes environment (e.g., VMs, bare-metal, cloud services).

**How They Authenticate & Connect**

- Credentials are manually retrieved from Kubernetes Secrets and stored externally—no ongoing dependency on Kubernetes Secrets (e.g., CI/CD pipelines, or environment variables, or manually using `kubectl get secret <secret-name> -o yaml`).
- Clients use external listeners (e.g., LoadBalancer, NodePort, Ingress, public DNS).
- Secrets are required for initial credential retrieval, but clients do not rely on them during runtime

**Example Configuration (SCRAM Authentication)**

```yaml
bootstrap.servers=my-kafka-bootstrap.central-cluster.com:9094
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
    username="my-user" \
    password="my-password";
ssl.truststore.location=/path/to/truststore.jks
ssl.truststore.password=mypassword
```

##### Comparison Table: How Clients Handle Authentication

| Feature                  | External Clients                              | Internal Clients (Central)                 | Internal Clients (Remote)                         |
|--------------------------|----------------------------------------------|--------------------------------------------|-------------------------------------------------|
| **How They Get Credentials**  | Manually fetched (Vault, CI/CD)            | Kubernetes Secret (Entity Operator)       | ❌ No automatic Secret (must be copied)        |
| **How They Connect to Kafka** | External listener (LoadBalancer, DNS)     | Internal service (`bootstrap:9092`)       | ✅ Works if Secret is manually synced          |
| **Requires Secret Sync?**  | ❌ No                                        | ❌ No                                      | ✅ Yes (or use external access)               |
| **Common Failure Points**  | Misconfigured authentication, TLS issues    | Works by default                          | ❌ Missing Secret → Clients fail to connect   |


### Why the Entity Operator's Absence Impacts Clients

As outlined in the 'Impact of Central Cluster Failure' section, the unavailability of the Entity Operator disrupts the declarative management of critical administrative functions like user authentication and topic lifecycle within your Kubernetes environment. This loss of control directly impacts the ability of administrators to create, update, or delete users and topics, and to manage their credentials and access.

### Best Practices for Stretched Kafka Deployments

To enhance the resilience of Kafka clients in a stretched deployment, especially in the event of a central cluster failure, the following best practices are recommended:

✅ Ensure that authentication credentials (TLS certificates and SCRAM hashed verifiers) are replicated across all Kafka brokers in the stretched cluster. This ensures that clients can continue to authenticate even if the central cluster is unavailable.

✅ Consider exploring alternative authentication and authorization solutions, such as the Kafka Access Operator, which might offer more distributed control and reduce the dependency on a single central cluster for these critical functions. This can improve the overall resilience of the deployment.