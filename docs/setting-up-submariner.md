# Setup submariner 

This section documents how to install and connect Kubernetes clusters using Submariner using Kind and OpenShift. 

## Installing subctl binary
The Submariner web page provides a bash script to download and install the latest version of submariner binary.
```bash
curl -Ls https://get.submariner.io | bash
export PATH=$PATH:~/.local/bin
echo export PATH=\$PATH:~/.local/bin >> ~/.profile
```
For more information on ways to install the subctl binary, consider visiting [submariner docs](https://submariner.io/operations/deployment/subctl/)

## CNI configuration
We have tested stretched kafka clusters using Calico CNI and OVN kubernetes CNI

| Environment    | CNI |
| -------- | ------- |
| Kind  | Calico    |
| Openshift | OVN Kubernetes     |

## Next
- [Setting Up Submariner on Kind](submariner-kind.md)
- [Setting Up Submariner on openshift](submariner-ocp.md)