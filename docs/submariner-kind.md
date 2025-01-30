## Setup submariner on kind
This section documents installation of submariner on kind using calico CNI.

### Installing Kind with Cliaco CNI
The submariner's shipyard project is used to install kind for our use case.
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

### Increase  inotify resource limits.
Since kind clusters have a default limit on the resources that can be allocated on it, we might not be able to deploy an entire streched kafka clusters on it. Hence, we increase the limits on the clusters.
```bash
$ sudo sysctl fs.inotify.max_user_watches=524288
$ sudo sysctl fs.inotify.max_user_instances=512
```

### List clusters
Verify the installation of the kind clusters using the get clusters command
```bash
$ kind get clusters
```

### Check the current Context
Use the kubectl to vew the cluster details using contexts. 
<br>
Export the `KUBECONFIG` env VAR for switching between clusters using contexts.
```bash
$ export KUBECONFIG=$(find $(git rev-parse --show-toplevel)/output/kubeconfigs/ -type f -printf %p:)
$ kubectl config get-contexts
```

### Confirm that we have two nodes in each cluster
```bash
$ kubectl --context cluster1 get nodes
$ kubectl --context cluster2 get nodes
```

## Deploy Calico 
Calico is deployed by creating the tigera-operator from project calico, modifying IP addresses in the CRs and applying them on each cluster.
<br>
Create a folder for calico manifests
```bash
$ mkdir calico_manifests
```
### Deploy Calico on cluster1
```bash
$ kubectl --context cluster1 create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/tigera-operator.yaml

$ wget -O calico_manifests/custom-resources.yaml  https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/custom-resources.yaml

$ sed -i 's,cidr: 192.168.0.0/16,cidr: 10.130.0.0/16,g' calico_manifests/custom-resources.yaml

$ sed -i 's,VXLANCrossSubnet,VXLAN,g' calico_manifests/custom-resources.yaml

$ kubectl --context cluster1 apply -f calico_manifests/custom-resources.yaml
```
The CIDRs are changed from the default value to some non overlapping one

### Install Calico on Cluster 2
```bash
$ kubectl --context cluster2 create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/tigera-operator.yaml

$ wget -O calico_manifests/custom-resources.yaml https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/custom-resources.yaml

$ sed -i 's,cidr: 192.168.0.0/16,cidr: 10.131.0.0/16,g' calico_manifests/custom-resources.yaml

$ sed -i 's,VXLANCrossSubnet,VXLAN,g' calico_manifests/custom-resources.yaml

$ kubectl --context cluster2 apply -f calico_manifests/custom-resources.yaml
```
## Deploy Submariner
### Deploying the broker on one of the clusters
```bash
$ subctl deploy-broker --context cluster1
```

### Connecting the clusters to the broker
```bash
$ subctl join --context cluster1  broker-info.subm --clusterid cluster1 --natt=false
$ subctl join --context cluster2  broker-info.subm --clusterid cluster2 --natt=false
```

###  Check Submariner connections
```bash
$ subctl show connections  --context cluster2   
$ subctl show connections  --context cluster1
```
