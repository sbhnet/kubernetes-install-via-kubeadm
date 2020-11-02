# Install kubernetes Using kubeadm

After woring though installing [Kubernetes the Hard Way on Bare Metal](https://github.com/dleewo/kubernetes-the-hard-way-bare-metal), I wanted to install it using the kubeadm way.  Trying to so by followjng [Bootstrapping clusters with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/) wasn't as straightforward as I hoped as it involves jumping around t oother pages mid-step.

This is a guide on doing the install, step-by-step, and in order to make the process as simle as possiblem

For this guide, I will be install a small cluster with a single control plane node and two worker nodes.  These will be VMs in VMWare ESXi, but should work for any VM or bare metal.

The VMs I will be using and their IP addresses are:

|Hostname|IP Address|vCPUs|RAM|Disk|
|:----|:----|--:--|---:|---:|
|`k8s-dev-master`|`192.168.20.37`|2|4GB|128GB|
|`k8s-dev-worker1`|`192.168.20.38`|2|4GB|128GB|
|`k8s-dev-worker2`|`192.168.20.39`|2|4GB|128GB|

You can use smaller VMs, but you will want at least 2 vCPUs and 2GB of RAM

For the POD network, the CIDR `10.244.0.0/16` will be used.

The software and versions installed will be:

* Kubernetes v1.19.3
* Calico v0.0.0

