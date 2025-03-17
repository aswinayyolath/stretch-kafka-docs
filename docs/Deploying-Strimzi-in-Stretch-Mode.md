# Deploying Strimzi in Stretch Mode

This section details how to deploy the Strimzi Kafka Operator in Stretch Mode. The document is divided into two sections:

- Deploying stretch clusters in OpenShift Container Platform (OCP)
- Deploying stretch clusters in Kubernetes

<iframe width="560" height="315" src="https://www.youtube.com/embed/NEPgtXD6voA?si=xrKa_zrzDqhW-B3C" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

Let's first look at how to deploy a stretch cluster with OpenShift.

## Deploying stretch clusters in OpenShift

Deploying a stretch cluster in OpenShift is relatively straightforward. Follow these steps:

1. Navigate to the `OperatorHub` in the OpenShift console.
2. Search for Strimzi and install version **`0.44.0`** in your namespace. (You must install Strimzi in all Kubernetes clusters that are part of the stretch cluster deployment.)
3. Set `Update approval` to `Manual` to prevent OpenShift from automatically updating the operator to the latest stable version.
4. Manually approve the Install Plan and wait a few minutes for the operator to install. (Ensure that namespaces have the same name across all Kubernetes clusters.)

### Updating KafkaNodePool CRD

Once the operator is installed, update the `KafkaNodePool` CRD by adding the following under `.spec.versions[0].schema.openAPIV3Schema.properties.spec.properties`. (For prototype testing, update the CRD only in the central cluster.)

```yaml
    cluster:
        type: string
        description: Target Kubernetes Cluster where SPS will be created.
```

You can edit the CRD using the command:

```bash
oc edit crd kafkanodepools.kafka.strimzi.io
```

Alternatively, you can update it using the OpenShift console:

1. Navigate to `Administration` → `CustomResourceDefinitions`.
2. Select `KafkaNodePool` and edit the schema.

### Updating ClusterRole for Submariner

If `Submariner` is used for pod-to-pod communication, update the `strimzi-cluster-operator` `ClusterRole` to allow the creation of the `ServiceExport` CR (This update is required only in the central cluster). The `ServiceExport` CR specifies which services should be exported outside the cluster. More details can be found [here](https://multicluster.sigs.k8s.io/api-types/service-export/).

The updated `ClusterRole` should look like this:

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: strimzi-cluster-operator.-ty2SyQjgGwxj3ajI1Cg98uIuqXKKouGnWYwjG
  labels:
    <REDACTED>
rules:
  - verbs:
      - get
      - list
      - watch
      - create
      - delete
      - patch
      - update
    apiGroups:
      - rbac.authorization.k8s.io
      - multicluster.x-k8s.io # ----> Add this
    resources:
      - clusterrolebindings
      - serviceexports ## ----> Add this
  - verbs:
      - get
    apiGroups:
      - storage.k8s.io
    resources:
      - storageclasses
  - verbs:
      - get
      - list
    apiGroups:
      - ''
    resources:
      - nodes
```

### Creating Kubernetes secrets for remote cluster access

In the central cluster, create Kubernetes secrets to connect to the remote clusters. This is necessary because all CRs (`Kafka` and `KafkaNodePool`) are created in the central cluster, leading to Kafka pod and related resource creation in the remote clusters.

1. Create files containing Kubeconfig data for each remote cluster (for example):
     - cluster-a-config (for Kubernetes cluster cluster-a)
     - cluster-b-config (for Kubernetes cluster cluster-b)

2. Create secrets in the central cluster:

```bash
🔥🔥🔥 $ kubectl create secret generic secret-cluster-a \
  --from-file=kubeconfig=cluster-a-config
secret/secret-cluster-a created

🔥🔥🔥 $ kubectl create secret generic secret-cluster-b \
  --from-file=kubeconfig=cluster-b-config
secret/secret-cluster-b created
```

!!! note

    ✅ The secret name does not matter. Use the same secret name in the `STRIMZI_K8S_CLUSTERS` environment variable.


### Updating the operator image

1. Navigate to `Installed Operators` in the OpenShift console.
2. Select Strimzi and open the YAML view.
3. Locate the operator image:

```
quay.io/strimzi/operator@sha256:b07b81f7e282dea2e4e29a7c93cfdd911d4715a5c32fe4c4e3d7c71cba3091e8
```
4. Replace it with your built image. If using a pre-built image, replace the default `0.44.0` image with:

```
aswinayyolath/stretchcluster:latest
```

### Updating environment variables

**Central cluster**

```yaml
- name: STRIMZI_STRETCH_MODE
  value: 'true'
- name: STRIMZI_K8S_CLUSTERS
  value: |
      cluster-a.url=<cluster-a URL>
      cluster-a.secret=secret-cluster-a
      cluster-b.url=<cluster-b URL>
      cluster-b.secret=secret-cluster-b
- name: STRIMZI_NETWORK_POLICY_GENERATION
  value: 'false'
```

**Remote clusters**

```yaml
- name: STRIMZI_STRETCH_MODE
  value: 'true'
```

!!! note

    ✅ Optionally, we can set `STRIMZI_POD_SET_RECONCILIATION_ONLY` to true in the remote cluster operator deployment. This ensures that only the `SPS` controller runs, preventing other controllers from starting and handling other custom resources.

Logs reassuring that no other operators have started other than the SPS controller:
```bash
2025-03-17 08:54:49 INFO  PodSecurityProviderFactory:43 - Found PodSecurityProvider io.strimzi.plugin.security.profiles.impl.BaselinePodSecurityProvider
2025-03-17 08:54:49 INFO  PodSecurityProviderFactory:62 - Initializing PodSecurityProvider io.strimzi.plugin.security.profiles.impl.BaselinePodSecurityProvider
2025-03-17 08:54:49 INFO  ClusterOperator:86 - Creating ClusterOperator for namespace strimzi
2025-03-17 08:54:49 INFO  ClusterOperator:100 - Starting ClusterOperator for namespace strimzi
2025-03-17 08:54:49 INFO  ClusterOperator:154 - --Stretch Mode value in ClusterOperator-- true
2025-03-17 08:54:49 INFO  StrimziPodSetController:594 - Starting the StrimziPodSet controller
2025-03-17 08:54:49 INFO  ClusterOperator:137 - Setting up periodic reconciliation for namespace strimzi
2025-03-17 08:54:49 INFO  StrimziPodSetController:563 - Starting StrimziPodSet controller for namespace strimzi
2025-03-17 08:54:49 INFO  Main:193 - Cluster Operator verticle started in namespace strimzi without label selector
2025-03-17 08:54:49 WARN  VersionUsageUtils:60 - The client is using resource type 'strimzipodsets' with unstable version 'v1beta2'
2025-03-17 08:54:49 WARN  VersionUsageUtils:60 - The client is using resource type 'kafkas' with unstable version 'v1beta2'
2025-03-17 08:54:49 WARN  VersionUsageUtils:60 - The client is using resource type 'kafkaconnects' with unstable version 'v1beta2'
2025-03-17 08:54:49 WARN  VersionUsageUtils:60 - The client is using resource type 'kafkamirrormaker2s' with unstable version 'v1beta2'
2025-03-17 08:54:49 INFO  StrimziPodSetController:566 - Waiting for informers to sync
2025-03-17 08:54:52 INFO  StrimziPodSetController:571 - Informers are in-sync
2025-03-17 08:54:53 INFO  StrimziPodSetController:389 - Reconciliation #1(watch) StrimziPodSet(strimzi/my-cluster-stretch2-broker): StrimziPodSet will be reconciled
2025-03-17 08:54:53 INFO  StrimziPodSetController:425 - Reconciliation #1(watch) StrimziPodSet(strimzi/my-cluster-stretch2-broker): reconciled
2025-03-17 08:54:53 INFO  StrimziPodSetController:389 - Reconciliation #2(watch) StrimziPodSet(strimzi/my-cluster-stretch2-controller): StrimziPodSet will be reconciled
2025-03-17 08:54:53 INFO  StrimziPodSetController:425 - Reconciliation #2(watch) StrimziPodSet(strimzi/my-cluster-stretch2-controller): reconciled
2025-03-17 08:54:53 INFO  StrimziPodSetController:389 - Reconciliation #3(watch) StrimziPodSet(strimzi/my-cluster-stretch2-broker): StrimziPodSet will be reconciled
2025-03-17 08:54:53 INFO  StrimziPodSetController:425 - Reconciliation #3(watch) StrimziPodSet(strimzi/my-cluster-stretch2-broker): reconciled
2025-03-17 08:59:49 INFO  StrimziPodSetController:389 - Reconciliation #4(watch) StrimziPodSet(strimzi/my-cluster-stretch2-broker): StrimziPodSet will be reconciled
```



After making these changes, save the `ClusterServiceVersion` YAML, which will trigger a restart of the operator pod.


### Applying `Kafka` and `KafkaNodePool` CRs in the central cluster

Once the setup is complete, apply the `Kafka` and `KafkaNodePool` CRs in the central cluster. Below are example CRs:

```yaml

#Central Cluster CR (spec.cluster is missing in these CRs)
#----------------------------------------------------------
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: controller
  labels:
    strimzi.io/cluster: my-cluster
    strimzi.io/submariner-cluster-id: "cluster1" #-------> submariner-cluster-id will be used in controller.quorum.voters and advertised.listeners to enable cross cluster communication  
spec:                                            #-------> In addition to this every controller and broker Pods will have a SANS entry with the Submariner exported DNS name                       
  replicas: 3
  roles:
    - controller
  storage:
    type: jbod
    volumes:
      - id: 0
        type: ephemeral
        kraftMetadata: shared
---

apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: broker
  labels:
    strimzi.io/cluster: my-cluster
    strimzi.io/submariner-cluster-id: "cluster1"
spec:
  replicas: 3
  roles:
    - broker
  storage:
    type: jbod
    volumes:
      - id: 0
        type: ephemeral
        kraftMetadata: shared
---

#Remote Cluster CR (spec.cluster is present and the SPS will be deployed on K8s cluster which we referenced as cluster-a)
#-----------------------------------------------------------------------------------------------------------------------

apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: stretch1-controller
  labels:
    strimzi.io/cluster: my-cluster
    strimzi.io/submariner-cluster-id: "cluster2"
spec:
  cluster: cluster-a #------->  all resources part of this KNP will be created in K8s cluster-a
  replicas: 3
  roles:
    - controller
  storage:
    type: jbod
    volumes:
      - id: 0
        type: ephemeral
        kraftMetadata: shared
---

apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: stretch1-broker
  labels:
    strimzi.io/cluster: my-cluster
    strimzi.io/submariner-cluster-id: "cluster2"
spec:
  cluster: cluster-a #------->  all resources part of this KNP will be created in K8s cluster-a
  replicas: 3
  roles:
    - broker
  storage:
    type: jbod
    volumes:
      - id: 0
        type: ephemeral
        kraftMetadata: shared
---

#Remote Cluster CR (spec.cluster is present and the SPS will be deployed on K8s cluster which we referenced as cluster-b)
#-----------------------------------------------------------------------------------------------------------------------

apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: stretch2-controller
  labels:
    strimzi.io/cluster: my-cluster
    strimzi.io/submariner-cluster-id: "cluster3"
spec:
  cluster: cluster-b  #-------> all resources part of this KNP will be created in K8s cluster-b
  replicas: 3
  roles:
    - controller
  storage:
    type: jbod
    volumes:
      - id: 0
        type: ephemeral
        kraftMetadata: shared
---

apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: stretch2-broker
  labels:
    strimzi.io/cluster: my-cluster
    strimzi.io/submariner-cluster-id: "cluster3"
spec:
  cluster: cluster-b #-------> all resources part of this KNP will be created in K8s cluster-b
  replicas: 3
  roles:
    - broker
  storage:
    type: jbod
    volumes:
      - id: 0
        type: ephemeral
        kraftMetadata: shared
---


apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  annotations:
    strimzi.io/node-pools: enabled
    strimzi.io/kraft: enabled
    strimzi.io/cross-cluster-type: "submariner"  #------->  Cross Cluster Communication Technology in use 
spec:
  kafka:
    version: 3.8.0
    metadataVersion: 3.8-IV0
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
  entityOperator:
    topicOperator: {}
    userOperator: {}

```

### How it works

#### `spec.cluster` in `KafkaNodePool` (KNP)

The `spec.cluster` field in a `KafkaNodePool` (KNP) CR determines where the `KafkaNodePool` will be created:

- If `spec.cluster` is missing, the `KafkaNodePool` is assumed to be created in the central Kubernetes cluster (i.e., the cluster where the Kafka CR is applied).
- If `spec.cluster` is defined, the provided cluster name will be matched against the `STRIMZI_K8S_CLUSTERS` environment variable. The corresponding secrets for remote Kubernetes clusters will then be used to create the required resources in the specified cluster.

#### Submariner cluster ID

A Submariner cluster ID can be defined using the `submariner-cluster-id` label in all `KafkaNodePool` CRs.

- This label represents the cluster identifier used by Submariner for tunnel identification.
- Each cluster must have a unique Submariner cluster ID to ensure seamless communication.

#### Cross-cluster type

Instead of defining the cross-cluster type separately for each `KafkaNodePool`, it can be centrally managed using the `cross-cluster-type` annotation in the `Kafka` CR.

- The `Kafka` CR serves as a logical place to store this information since the cross-cluster type is a shared property across all clusters in a stretch Kafka setup.
- This ensures consistency and simplicity in configuration by avoiding redundant definitions in multiple `KafkaNodePool` CRs.

#### STRIMZI_NETWORK_POLICY_GENERATION

You need to set `STRIMZI_NETWORK_POLICY_GENERATION` to `false` because the default `NetworkPolicy` created by Strimzi restricts traffic to Kafka pods within a specific namespace. By default, Kafka pods can only receive traffic from:

- Kafka clients and Kafka-related components within the same cluster (on port 9090).
- Specific Strimzi components such as:
    - Cluster operator
    - Entity operator
    - Kafka exporter
    - Cruise control
    - (on ports 9091, 8443, 9092, and 9093).

This policy improves security by ensuring that only necessary services can communicate with Kafka pods. However, it also blocks traffic between Kubernetes clusters, which is required for stretch cluster deployments. To allow communication between Kubernetes clusters, you must set `STRIMZI_NETWORK_POLICY_GENERATION` to `false`.

#### STRIMZI_STRETCH_MODE

By default, Strimzi expects both the `Kafka` and `KafkaNodePool` (KNP) resources to be present in the same cluster where it creates the StrimziPodSet (SPS) and Kafka pods.

However, in a stretch cluster deployment, the member clusters (remote clusters) do not contain the `Kafka` and `KafkaNodePool` CRs. To address this, we introduced the `STRIMZI_STRETCH_MODE` environment variable.

- When `STRIMZI_STRETCH_MODE` is set to `true`, the Strimzi operator bypasses validation checks that require `Kafka` and `KafkaNodePool` CRs to exist in the cluster where Kafka pods are deployed.
- This allows Strimzi to create Kafka pods in remote clusters without requiring Kafka or KafkaNodePool CRs in those clusters.

#### Summary

- Use the **`submariner-cluster-id`** label in `KafkaNodePool` CRs to uniquely identify each cluster in Submariner’s tunnel setup.
- Use the **`cross-cluster-type`** annotation in the `Kafka` CR to centrally define the cross-cluster communication type.
- Leverage **`spec.cluster`** in KafkaNodePool to determine the target cluster for resource creation, defaulting to the central cluster if not specified.

### Central cluster considerations
The current setup functions even if no `KafkaNodePool` (KNP) resources are created for the central cluster. This means:

- KafkaNodePool CRs can be applied in the central cluster, triggering the creation of pods and Kubernetes resources only in remote clusters, with no additional resources required in the central cluster itself.
- In other words, it is entirely possible to create a stretch Kafka cluster where the central cluster hosts only the Strimzi cluster operator, with Kafka brokers running exclusively in remote clusters.
- This eliminates any dependency on having a `KafkaNodePool` resource in the central cluster.

## Deploying stretch clusters in Kubernetes
Deploying a stretch cluster in Kubernetes follows a similar process as in OpenShift, with a few key differences:

1. Installing Strimzi
2. Editing the cluster operator `Deployment` instead of the `ClusterServiceVersion`

### Installing Strimzi
Download Strimzi `0.44.0` using the following command:

```
wget https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.44.0/strimzi-0.44.0.tar.gz
```

Alternatively, you can download Strimzi 0.44.0 from the [release page](https://github.com/strimzi/strimzi-kafka-operator/releases/tag/0.44.0).

### Extract the downloaded file

```
tar -xvzf strimzi-0.44.0.tar.gz
```

By default, Strimzi resources are configured to work in the `myproject` namespace. If you want to use a different namespace, update the namespace references in the relevant files using:  

```bash
sed -i 's/namespace: .*/namespace: <yournamespace>/' install/cluster-operator/*RoleBinding*.yaml
```

Edit the `install/cluster-operator/060-Deployment-strimzi-cluster-operator.yaml` file:

- Set the STRIMZI_NAMESPACE environment variable to `yournamespace`.
- Add the following environment variables:

**Central cluster configuration**

Modify 060-Deployment-strimzi-cluster-operator.yaml and add:

```yaml
- name: STRIMZI_STRETCH_MODE
  value: 'true'
- name: STRIMZI_K8S_CLUSTERS
  value: |
      cluster-a.url=<cluster-a URL>
      cluster-a.secret=secret-cluster-a
      cluster-b.url=<cluster-b URL>
      cluster-b.secret=secret-cluster-b
- name: STRIMZI_NETWORK_POLICY_GENERATION
  value: 'false'
```

**Remote cluster configuration**

For remote clusters, add only:

```yaml
- name: STRIMZI_STRETCH_MODE
  value: 'true'
```

### Granting permissions to the cluster operator

```bash
kubectl create -f install/cluster-operator/020-RoleBinding-strimzi-cluster-operator.yaml -n <yournamespace>
kubectl create -f install/cluster-operator/031-RoleBinding-strimzi-cluster-operator-entity-operator-delegation.yaml -n <yournamespace>
```

These commands create role bindings that grant the cluster operator permission to access the Kafka cluster.

### Deploy CRDs and RBAC resources
Deploy the Custom Resource Definitions (CRDs) and role-based access control (RBAC) resources:

```bash
kubectl create -f install/cluster-operator/ -n <yournamespace>
```

### Remaining steps
The remaining steps are the same for both OpenShift and Kubernetes, such as:

- Updating the `KafkaNodePool` CRD
- Updating the `strimzi-cluster-operator` `ClusterRole`
- Creating a Kubeconfig secret in the central cluster
