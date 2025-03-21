# Securing Remote Cluster Credentials in Multi-Cluster Kafka Deployment

## Overview

In a multi-cluster Kafka deployment, the central Strimzi operator requires access to multiple remote Kubernetes clusters to manage Kafka nodes across them. To achieve this, users must provide the necessary credentials in Kubernetes Secrets, ensuring secure authentication while following best practices for least-privilege access.


## Defining Remote Cluster Credentials

The Strimzi operator in the central cluster references remote cluster credentials through `STRIMZI_K8S_CLUSTERS` environment variable in the central cluster's operator deployment.

```yaml
- name: STRIMZI_K8S_CLUSTERS
  value: |
      cluster-id-a.url=<cluster-a URL>
      cluster-id-a.secret=<secret-name-cluster-a>
      cluster-id-b.url=<cluster-b URL>
      cluster-id-b.secret=<secret-name-cluster-b>
```

Each referenced secret must contain a kubeconfig file under the kubeconfig key, which provides authentication details for the corresponding remote cluster.


## Recommended Security Practices

### 1. Use a Dedicated ServiceAccount with Scoped Permissions

Instead of embedding full administrator credentials in the kubeconfig, create a dedicated ServiceAccount in each remote cluster with only the necessary permissions.

#### Example

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: stretch-operator
  namespace: <namespace>
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: stretch-operator-role
  namespace: <namespace>
rules:
- apiGroups:
  - core.strimzi.io
  resources:
  - strimzipodsets
  verbs:
  - get
  - list
  - watch
  - create
  - update
  - patch
  - delete
- apiGroups:
  - ""
  resources:
  - services
  - configmaps
  - secrets
  - serviceaccounts
  verbs:
  - get
  - list
  - watch
  - create
  - update
  - patch
  - delete
- apiGroups:
  - multicluster.x-k8s.io
  resources:
  - serviceexports
  verbs:
  - get
  - list
  - watch
  - create
  - update
  - patch
  - delete
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: stretch-operator-rolebinding
  namespace: <namespace>
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: stretch-operator-role
subjects:
- kind: ServiceAccount
  name: stretch-operator
  namespace: <namespace>
```


### 2. Use Token-Based Authentication

To securely authenticate the Strimzi operator with remote clusters, use a long-lived token associated with the dedicated ServiceAccount.

#### Generating a Long-Lived Token

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-long-lived-secret
  annotations:
    kubernetes.io/service-account.name: stretch-operator
type: kubernetes.io/service-account-token
```

### 3. Store Minimal Credentials in Kubeconfig

Instead of using a full administrator kubeconfig, generate a scoped kubeconfig with only the necessary permissions.

#### Example Scoped Kubeconfig

```yaml
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://api.example.com:6443
  name: example-cluster
contexts:
- context:
    cluster: example-cluster
    namespace: <namespace>
    user: stretch-operator-user
  name: stretch-cluster-context
current-context: stretch-cluster-context
kind: Config
preferences: {}
users:
- name: stretch-operator-user
  user:
    token: <TOKEN>
```

## Alternative Authentication Mechanisms

While token-based authentication is one of the possible approach, users may choose alternative authentication mechanisms based on their security policies:

- mTLS authentication: Using client TLS certificates instead of tokens in the kubeconfig.
- Username/password authentication: With integration into external authentication providers.
- OIDC-based authentication: Leveraging identity providers for federated authentication.

The Strimzi operator remains agnostic to the authentication method, relying only on the kubeconfig provided by the user.


## Benefits of Secure Remote Authentication

- Principle of Least Privilege: Operators can only access the required resources.
- Long-Lived Credentials: Prevents frequent authentication failures.
- Flexible Authentication Options: Users can integrate existing security models without modifying the operator.

By following these best practices, users can securely manage remote cluster credentials while ensuring a robust and secure multi-cluster Kafka deployment with Strimzi.