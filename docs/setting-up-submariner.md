# Setting up Submariner

This guide explains how to install and configure Submariner to connect multiple Kubernetes clusters using either Kind or OpenShift.

## Installing the `subctl` binary

Submariner provides a script to download and install the latest subctl binary:

```bash
curl -Ls https://get.submariner.io | bash
export PATH=$PATH:~/.local/bin
echo export PATH=\$PATH:~/.local/bin >> ~/.profile
```
For more installation options, refer to the [Submariner documentation](https://submariner.io/operations/deployment/subctl/)

## CNI compatibility

We have tested stretch Kafka clusters with the following Container Network Interfaces (CNI):

| Environment    | CNI |
| -------- | ------- |
| Kind  | Calico    |
| OpenShift | OVN-Kubernetes   |
