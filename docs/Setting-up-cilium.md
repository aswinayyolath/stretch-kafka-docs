# Setting Up Cilium
This page outlines the steps required to install and configure Cilium for stretch clusters using Calico as the CNI across three Kubernetes clusters.

### Prerequisites
- A LoadBalancer address must be available for the Cilium API mesh server.
- Calico CNI should be installed on all clusters.
- Non-conflicting IP space across clusters.

## Step 1: Install Cilium on the First Cluster
```bash
cilium install --set cluster.name=cluster1 --set cluster.id=1 --context kubernetes-admin@stretch-calico-1
```

## Step 2: Copy Cilium CA to other Clusters for Mutual TLS
```bash
kubectl --context=kubernetes-admin@stretch-calico-1 get secret -n kube-system cilium-ca -o yaml | \
  kubectl --context=kubernetes-admin@stretch-calico-2 create -f -
```
```bash
kubectl --context=kubernetes-admin@stretch-calico-1 get secret -n kube-system cilium-ca -o yaml | \
  kubectl --context=kubernetes-admin@stretch-calico-3 create -f -
```

## Step 3: Install Cilium on Remaining Clusters
```bash
cilium install --set cluster.name=cluster2 --set cluster.id=2 --context kubernetes-admin@stretch-calico-2
cilium install --set cluster.name=cluster3 --set cluster.id=3 --context kubernetes-admin@stretch-calico-3
```

## Step 4: Verify Cilium Installation
```bash
cilium status --context kubernetes-admin@stretch-calico-1 --wait
cilium status --context kubernetes-admin@stretch-calico-2 --wait
cilium status --context kubernetes-admin@stretch-calico-3 --wait
```
All 3 clusters should show all resources "OK"

## Step 5: Enabling Cluster Mesh
To enable communication between clusters, use the following command to expose the ClusterMesh API as a LoadBalancer service
```bash
cilium clustermesh enable --context kubernetes-admin@stretch-calico-1 --service-type LoadBalancer
cilium clustermesh enable --context kubernetes-admin@stretch-calico-2 --service-type LoadBalancer
cilium clustermesh enable --context kubernetes-admin@stretch-calico-3 --service-type LoadBalancer
```


## Step 6: Verify ClusterMesh Status
```bash
cilium clustermesh status --context kubernetes-admin@stretch-calico-1 --wait
cilium clustermesh status --context kubernetes-admin@stretch-calico-2 --wait
cilium clustermesh status --context kubernetes-admin@stretch-calico-3 --wait
```
All three clusters should show "OK" for all resources.

## Step 7: Connecting the clusters
```bash
cilium clustermesh connect --context kubernetes-admin@stretch-calico-1 --destination-context kubernetes-admin@stretch-calico-2
cilium clustermesh connect --context kubernetes-admin@stretch-calico-1 --destination-context kubernetes-admin@stretch-calico-3
cilium clustermesh connect --context kubernetes-admin@stretch-calico-2 --destination-context kubernetes-admin@stretch-calico-3
```

## Step 8: Verify Cluster Connectivity
```bash
cilium clustermesh status --context kubernetes-admin@stretch-calico-2 --wait
```
This command should show connections to the other two clusters.

## Step 9: Testing Pod Connectivity
Try to ping the pods from one cluster to the other using IP address. Ideally, use and nginx server and webclient to test this connectivity.
<br><br>

## Setting Up CoreDNS for Cross-Cluster DNS Resolution

### Exposing CoreDNS as NodePort Service
Cilium ClusterMesh does not provide cross-cluster DNS resolution for headless service by default. We can modify CoreDNS to enable this functionality.

### Exposing CoreDNS all three clusters
Create a NodePort service to expose a CoreDNS across all three clusters.
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: core-dns-nodeport
    kubernetes.io/name: CoreDNS
  name: core-dns-nodeport
  namespace: kube-system
spec:
  ports:
  - name: dns
    port: 53
    protocol: UDP
    targetPort: 53
    nodePort: 30053
  - name: dns-tcp
    port: 53
    protocol: TCP
    targetPort: 53
    nodePort: 30053
  selector:
    k8s-app: kube-dns
  sessionAffinity: None
  type: NodePort
```
Apply this service to all three clusters

```bash
kubectl --context=kubernetes-admin@stretch-calico-1 apply -f core-dns-nodeport.yaml
kubectl --context=kubernetes-admin@stretch-calico-2 apply -f core-dns-nodeport.yaml
kubectl --context=kubernetes-admin@stretch-calico-3 apply -f core-dns-nodeport.yaml
```
Here, the DNS service is exposed on port 30053 of infraIP (xx.xx.xx.xx:30053).

### Setting up CoreDNS configmap to forward requests
We'll use a service address convention to determine the cluster details of a pod based on its service address, such as:
```
my-cluster-broker-100.cluster1.my-cluster-kafka-brokers.strimzi.svc.cluster.local
```
Here, cluster1 is injected into the address to identify which DNS service should resolve it.
#### Editing the CoreDNS ConfigMap
```bash
kubectl edit cm coredns -n kube-system --kubeconfig calico-1
```
And set up these rules
```yaml
apiVersion: v1
data:
  Corefile: |
    cluster2.svc.cluster.local.:53 {
      rewrite stop {
          name substring cluster2.svc.cluster.local svc.cluster.local answer auto
      }
      forward . x.xx.xx.34:30053 {
          expire 10s
          policy round_robin
      }
      cache 10
    }
	cluster3.svc.cluster.local.:53 {
      rewrite stop {
          name substring cluster3.svc.cluster.local svc.cluster.local answer auto
      }
      forward . x.xx.xx.149:30053 {
          expire 10s
          policy round_robin
      }
      cache 10
    }
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        rewrite stop {
          name substring cluster1.svc.cluster.local svc.cluster.local answer auto
        }
		template IN ANY clusterset.local {
           match "(?P<broker>[a-zA-Z0-9-]+)\.(?P<cluster>[a-zA-Z0-9-]+)\.(?P<service>[a-zA-Z0-9-]+)\.(?P<ns>[a-zA-Z0-9-]+)\.svc\.clusterset\.local.$"
           answer "{{ .Name }} 60 IN CNAME {{ .Group.broker }}.{{ .Group.service }}.{{ .Group.ns }}.{{ .Group.cluster }}.svc.cluster.local"
        }
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
```

Essentially we have added

```yaml
cluster2.svc.cluster.local.:53 {
      rewrite stop {
          name substring cluster2.svc.cluster.local svc.cluster.local answer auto
      }
      forward . x.xx.xx.34:30053 {
          expire 10s
          policy round_robin
      }
      cache 10
    }
cluster3.svc.cluster.local.:53 {
      rewrite stop {
          name substring cluster3.svc.cluster.local svc.cluster.local answer auto
      }
      forward . x.xx.xx.149:30053 {
          expire 10s
          policy round_robin
      }
      cache 10
    }
```
When this DNS service is asked to resolve an address ending with `cluster2.svc.cluster.local`, it rewrites the string with a valid one (`.svc.cluster.local`, removing cluster2) and forwards it to the DNS service in the 2nd cluster. The forward rule is followed by ip address of the 2nd cluster to forward all request in that category to the 2nd cluster's coreDNS service. Similarly it forwards the 3rd clusters address to the 3rd cluster for resolution.

<br>  Another rewrite
```yaml
rewrite stop {
          name substring cluster1.svc.cluster.local svc.cluster.local answer auto
        }
template IN ANY clusterset.local {
           match "(?P<broker>[a-zA-Z0-9-]+)\.(?P<cluster>[a-zA-Z0-9-]+)\.(?P<service>[a-zA-Z0-9-]+)\.(?P<ns>[a-zA-Z0-9-]+)\.svc\.clusterset\.local.$"
           answer "{{ .Name }} 60 IN CNAME {{ .Group.broker }}.{{ .Group.service }}.{{ .Group.ns }}.{{ .Group.cluster }}.svc.cluster.local"
        }
```
is added for the DNS to resolve it's own addresses by removing the `cluster1` part from the request and resolve it normally.

The Configmap of cluster 2 is modified to:
```yaml
apiVersion: v1
data:
  Corefile: |
    cluster1.svc.cluster.local.:53 {
      rewrite stop {
          name substring cluster1.svc.cluster.local svc.cluster.local answer auto
      }
      forward . x.xx.xx.97:30053 {
          expire 10s
          policy round_robin
      }
      cache 10
    }
	cluster3.svc.cluster.local.:53 {
      rewrite stop {
          name substring cluster3.svc.cluster.local svc.cluster.local answer auto
      }
      forward . x.xx.xx.149:30053 {
          expire 10s
          policy round_robin
      }
      cache 10
    }
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        rewrite stop {
          name substring cluster2.svc.cluster.local svc.cluster.local answer auto
        }
		template IN ANY clusterset.local {
           match "(?P<broker>[a-zA-Z0-9-]+)\.(?P<cluster>[a-zA-Z0-9-]+)\.(?P<service>[a-zA-Z0-9-]+)\.(?P<ns>[a-zA-Z0-9-]+)\.svc\.clusterset\.local.$"
           answer "{{ .Name }} 60 IN CNAME {{ .Group.broker }}.{{ .Group.service }}.{{ .Group.ns }}.{{ .Group.cluster }}.svc.cluster.local"
        }
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
```
This sets the resolving rules in reverse. All addresses ending with `cluster1.svc.cluster.local` will modified and sent to the DNS service in the other cluster which can resolve it normally. The addresses ending with `cluster2.svc.cluster.local` will be resolved by the local DNS cluster by removing the `cluster2` part from the address. And cluster 3 address will be sent to resolution by cluster 3 DNS.

Cluster 3's coreDNS is modified to:
```yaml
apiVersion: v1
data:
  Corefile: |
    cluster1.svc.cluster.local.:53 {
      rewrite stop {
          name substring cluster1.svc.cluster.local svc.cluster.local answer auto
      }
      forward . x.xx.xx.97:30053 {
          expire 10s
          policy round_robin
      }
      cache 10
    }
    cluster2.svc.cluster.local.:53 {
      rewrite stop {
          name substring cluster2.svc.cluster.local svc.cluster.local answer auto
      }
      forward . x.xx.xx.34:30053 {
          expire 10s
          policy round_robin
      }
      cache 10
    }
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        rewrite stop {
          name substring cluster3.svc.cluster.local svc.cluster.local answer auto
        }
		template IN ANY clusterset.local {
           match "(?P<broker>[a-zA-Z0-9-]+)\.(?P<cluster>[a-zA-Z0-9-]+)\.(?P<service>[a-zA-Z0-9-]+)\.(?P<ns>[a-zA-Z0-9-]+)\.svc\.clusterset\.local.$"
           answer "{{ .Name }} 60 IN CNAME {{ .Group.broker }}.{{ .Group.service }}.{{ .Group.ns }}.{{ .Group.cluster }}.svc.cluster.local"
        }
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
```



#### Try to DNS resolve using dig
```bash
🔥🔥🔥 $ dig my-cluster-controller-4.cluster1.my-cluster-kafka-brokers.strimzi.svc.clusterset.local  @xx.xx.xx.xx -p 30053

; <<>> DiG 9.10.6 <<>> my-cluster-controller-4.cluster1.my-cluster-kafka-brokers.strimzi.svc.clusterset.local @xx.xx.xx.xx -p 30053
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 17247
;; flags: qr aa rd; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;my-cluster-controller-4.cluster1.my-cluster-kafka-brokers.strimzi.svc.clusterset.local. IN A

;; ANSWER SECTION:
my-cluster-controller-4.cluster1.my-cluster-kafka-brokers.strimzi.svc.clusterset.local. 10 IN CNAME my-cluster-controller-4.my-cluster-kafka-brokers.strimzi.cluster1.svc.cluster.local.
my-cluster-controller-4.my-cluster-kafka-brokers.strimzi.cluster1.svc.cluster.local. 10 IN A 10.0.1.73

;; Query time: 270 msec
;; SERVER: xx.x.xx.xx#30053(xx.xx.xx.xx)
;; WHEN: Fri Feb 07 14:22:13 IST 2025
;; MSG SIZE  rcvd: 401
```

Try with all combination
```
cluster-a address -> cluster-a DNS
cluster-a address -> cluster-b DNS
cluster-a address -> cluster-c DNS
cluster-b address -> cluster-a DNS
cluster-b address -> cluster-b DNS
cluster-b address -> cluster-c DNS
cluster-c address -> cluster-a DNS
cluster-c address -> cluster-b DNS
cluster-c address -> cluster-c DNS
```

Now proceed with installing the operator on stretch mode where the kafka CR's cross-cluster-type label is referenced as cilium
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  annotations:
    strimzi.io/node-pools: enabled
    strimzi.io/kraft: enabled
    strimzi.io/cross-cluster-type: "cilium"  #-- change this to cilium instead of submainer 
```
No edits to clusterRole is needed as no Service Exports are needed for cilium.
