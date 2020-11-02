# Install kubernetes Using kubeadm

After woring though installing [Kubernetes the Hard Way on Bare Metal](https://github.com/dleewo/kubernetes-the-hard-way-bare-metal), I wanted to install it using the kubeadm way.  Trying to so by followjng [Bootstrapping clusters with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/) wasn't as straightforward as I hoped as it involves jumping around t oother pages mid-step.

This is a guide on doing the install, step-by-step, and in order to make the process as simle as possiblem

## Software Versions

The software and versions installed will be:

* Kubernetes v1.19.3
* Calico v0.0.0
* Docker CE v 19.o.o

## Servers and Infrastructure

For this guide, I will be install a small cluster with a single control plane node and two worker nodes.  These will be VMs in VMWare ESXi, but should work for any VM or bare metal.

The VMs I will be using and their IP addresses are:

|Hostname|IP Address|vCPUs|RAM|Disk|
|:----|:----|:-:|---:|---:|
|`k8s-dev-master`|`192.168.20.37`|2|4GB|128GB|
|`k8s-dev-worker1`|`192.168.20.38`|2|4GB|128GB|
|`k8s-dev-worker2`|`192.168.20.39`|2|4GB|128GB|

You can use smaller VMs, but you will want at least 2 vCPUs and 2GB of RAM

Each server is running Ubuntu 20.04.1 64-bit

For the POD network, the CIDR `10.244.0.0/16` will be used.

On each server, disable swap if it not already disabled

```
sudo swapoff -a
```

This will disable swap immediatly, but it is only temporary and will be re-enabled if you reboot.  To disable swap permanently, edit the `/etc/fstab` file and comment out the swap line:
 
<img src="https://github.com/dleewo/kubernetes-the-hard-way-bare-metal/raw/main/images/fstab-swap.png" width="700" />

The next time you reboot, swap will be disabled.

## Step 1 - Install Docker

Docker will be used as the container for the cluster.  The original Docker installation guide is [here](https://docs.docker.com/engine/install/ubuntu/).  The following must be done on each server:

First, setup the docker repository

```
{
 sudo apt-get update
 sudo apt-get -y install apt-transport-https ca-certificates curl gnupg-agent software-properties-common
 curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
 sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
}
```

Then install the docker packages

```
{
    sudo apt-get update
    sudo apt-get -y install docker-ce docker-ce-cli containerd.io
}
 ```



## Step 2 - XXXXXXX


dddddd
