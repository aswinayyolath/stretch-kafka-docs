## Installing Submariner on Kind

This section details installing Submariner on Kind using Calico CNI.

### Installing Kind with Calico CNI

We use Submariner's Shipyard project to install Submariner on Kind:

```bash
$ git clone https://github.com/submariner-io/shipyard.git
$ cd shipyard/


$ cat > deploy.two.clusters.nocni.yaml << EOF
nodes: control-plane worker
clusters:
 cluster1:
   cni: none
 cluster2:
   cni: none
EOF

$ make SETTINGS=deploy.two.clusters.nocni.yaml clusters
```

### Increasing inotify resource limits

Kind clusters have default resource limits that may be insufficient for stretch Kafka clusters. Increase the limits using:

```bash
$ sudo sysctl fs.inotify.max_user_watches=524288
$ sudo sysctl fs.inotify.max_user_instances=512
```
### Verifying cluster installation

####  List clusters

```bash
$ kind get clusters
```

### Check current context

Set the correct `KUBECONFIG` environment variable and list available contexts:

```bash
$ export KUBECONFIG=$(find $(git rev-parse --show-toplevel)/output/kubeconfigs/ -type f -printf %p:)
$ kubectl config get-contexts
```

### Verify node counts in each cluster
```bash
$ kubectl --context cluster1 get nodes
$ kubectl --context cluster2 get nodes
```

## Deploying Calico

Create a folder for Calico manifests:


```bash
$ mkdir calico_manifests
```
### Install Calico on cluster 1

```bash
$ kubectl --context cluster1 create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/tigera-operator.yaml

$ wget -O calico_manifests/custom-resources.yaml  https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/custom-resources.yaml

$ sed -i 's,cidr: 192.168.0.0/16,cidr: 10.130.0.0/16,g' calico_manifests/custom-resources.yaml

$ sed -i 's,VXLANCrossSubnet,VXLAN,g' calico_manifests/custom-resources.yaml

$ kubectl --context cluster1 apply -f calico_manifests/custom-resources.yaml
```

### Install Calico on cluster 2
```bash
$ kubectl --context cluster2 create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/tigera-operator.yaml

$ wget -O calico_manifests/custom-resources.yaml https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/custom-resources.yaml

$ sed -i 's,cidr: 192.168.0.0/16,cidr: 10.131.0.0/16,g' calico_manifests/custom-resources.yaml

$ sed -i 's,VXLANCrossSubnet,VXLAN,g' calico_manifests/custom-resources.yaml

$ kubectl --context cluster2 apply -f calico_manifests/custom-resources.yaml
```
## Deploying Submariner

### Deploying the broker

```bash
$ subctl deploy-broker --context cluster1
```

### Connecting clusters to the broker

```bash
$ subctl join --context cluster1  broker-info.subm --clusterid cluster1 --natt=false
$ subctl join --context cluster2  broker-info.subm --clusterid cluster2 --natt=false
```

###  Checking Submariner connections

```bash
$ subctl show connections  --context cluster2   
$ subctl show connections  --context cluster1
```
