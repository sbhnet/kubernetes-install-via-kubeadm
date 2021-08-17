# Install kubernetes Using kubeadm

After working though installing [Kubernetes the Hard Way on Bare Metal](https://github.com/dleewo/kubernetes-the-hard-way-bare-metal), I wanted to install it using the `kubeadm` way.  Trying to so by followjng [Bootstrapping clusters with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/) wasn't as straightforward as I hoped as it involves jumping around to other pages mid-step.

This is a guide on doing the install, step-by-step, and in order, to make the process as simple as possible for others to follow

## Software Versions

The software and versions installed will be:

* [Kubernetes](https://kubernetes.io/) v1.19.3
* [Calico](https://www.projectcalico.org/) v3.16.4
* [Docker](https://www.docker.com/) CE v 19.03.13

For kubernetes and Docker, the steps below will install whatever is the current release so your versions may differ.  The versions above are the latest versions as of November 2 2020.

## Servers and Infrastructure

For this guide, I will be installing a small cluster with a single control plane node and two worker nodes.  These will be VMs in VMWare ESXi, but should work for any VM or on bare metal machines.

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
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
EOF

echo '1' > /proc/sys/net/ipv4/ip_forward
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables



sudo swapoff -a

sed -i.bak -r 's/(.+ swap .+)/#\1/' /etc/fstab

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

Configure the docker daemon.  This swicthes the cgroup driver to `systemd`

```
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

Restart docker

```
sudo systemctl restart docker
```

You can verify Docker is working by running:

```
sudo docker version
```

> Output

```
Client: Docker Engine - Community
 Version:           19.03.13
 API version:       1.40
 Go version:        go1.13.15
 Git commit:        4484c46d9d
 Built:             Wed Sep 16 17:02:52 2020
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          19.03.13
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.13.15
  Git commit:       4484c46d9d
  Built:            Wed Sep 16 17:01:20 2020
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.3.7
  GitCommit:        8fba4e9a7d01810a393d5d25a3621dc101981175
 runc:
  Version:          1.0.0-rc10
  GitCommit:        dc9208a3303feef5b3839f4323d9beb36df0a9dd
 docker-init:
  Version:          0.18.0
  GitCommit:        fec3683
```


## Step 2 - Install Kubernetes Packages

This will install kubeadm, kubectl, and kubelt.  The folowing must be done on all threee servers

```
{
    sudo apt-get update && sudo apt-get install -y apt-transport-https curl
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
}
```

```
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```

We will now install the latest version of each package and then tell apt to not upgrade them

```
{
    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
}
```

## Step 3 - Configure The Control Plane

The commands in this state are only done in the `master` node

Run `kubeadm`

```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
> Output

```
W1102 18:50:59.642012    5183 configset.go:348] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.19.3
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-dev-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.20.37]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-dev-master localhost] and IPs [192.168.20.37 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-dev-master localhost] and IPs [192.168.20.37 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 22.513852 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.19" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node k8s-dev-master as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node k8s-dev-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: dp7eyd.jdpbnfw4z3ruu2ag
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.20.37:6443 --token --token dp7e ...[SNIP]...  ruu2ag \
    --discovery-token-ca-cert-hash sha256:5b4af75846 ...[SNIP]...  f46c553b37860a08b
```

You should copy and paste the last line with the token to a test file or temporary location so you will have it later when adding the node

As specified in the tail end of the output, run the following as a regular user to configure access to `kubectl`

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

To test, run the following as the regular user

```
kubectl get all -A
```

> Output

```
NAMESPACE     NAME                                         READY   STATUS    RESTARTS   AGE
kube-system   pod/coredns-f9fd979d6-pswmt                  0/1     Pending   0          2m12s
kube-system   pod/coredns-f9fd979d6-ztw6c                  0/1     Pending   0          2m12s
kube-system   pod/etcd-k8s-dev-master                      1/1     Running   0          2m21s
kube-system   pod/kube-apiserver-k8s-dev-master            1/1     Running   0          2m21s
kube-system   pod/kube-controller-manager-k8s-dev-master   1/1     Running   0          2m21s
kube-system   pod/kube-proxy-bnvb2                         1/1     Running   0          2m12s
kube-system   pod/kube-scheduler-k8s-dev-master            1/1     Running   0          2m21s

NAMESPACE     NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP                  2m30s
kube-system   service/kube-dns     ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   2m28s

NAMESPACE     NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-system   daemonset.apps/kube-proxy   1         1         1       1            1           kubernetes.io/os=linux   2m28s

NAMESPACE     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/coredns   0/2     2            0           2m28s

NAMESPACE     NAME                                DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/coredns-f9fd979d6   2         2         0       2m12s
```

## Step 4 - Add the POD Networking

We wil now install Calico for POD networking.  The YAML file is located here https://docs.projectcalico.org/v3.16/manifests/calico.yaml, but we will load a customized one that sets the POD CIDR to `10.244.0.0/16`.  No other changes were made.  If you wish to install a different, or newer version of Calico, you should download the yaml and look for the `CALICO_IPV4POOL_CIDR` setting, uncomment it and set the POD CIDR

```
     - name: CALICO_IPV4POOL_CIDR
       value: "10.244.0.0/16"
```

Run the following on the master node

```
kubectl apply -f https://raw.githubusercontent.com/dleewo/kubernetes-install-via-kubeadm/main/calico.yaml
```

> Output

```
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.apps/calico-node created
serviceaccount/calico-node created
deployment.apps/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
```

## Step 5 - Configure The Worker Nodes

We will now add the worker nodes to the cluster.  The following must be done on each worker node.  Be sure to use *your* tokens.  You should have copy-pasted and saved this from the output from the `kubeadm init` command

```
sudo kubeadm join 192.168.20.37:6443 --token dp7e ...[SNIP]...  ruu2ag \
    --discovery-token-ca-cert-hash sha256:5b4af75846 ...[SNIP]...  f46c553b37860a08b
```

> Output

```
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

To test, run the following on your master node:

```
kubectl get nodes
```

> Output

```
NAME              STATUS   ROLES    AGE     VERSION
k8s-dev-master    Ready    master   19m     v1.19.3
k8s-dev-worker1   Ready    <none>   2m24s   v1.19.3
k8s-dev-worker2   Ready    <none>   2m1s    v1.19.3
```

It could take a few mins for the status to swicth from `NotReady` to `Ready`.

Let's view everything that is running.  Run the followng on the master node

```
kubectl get all -A -o wide
```

> Output

```
NAMESPACE     NAME                                         READY   STATUS    RESTARTS   AGE     IP              NODE              NOMINATED NODE   READINESS GATES
kube-system   pod/calico-kube-controllers-7d569d95-sbbft   1/1     Running   0          7m45s   10.244.134.67   k8s-dev-master    <none>           <none>
kube-system   pod/calico-node-7xlpw                        1/1     Running   0          4m43s   192.168.20.39   k8s-dev-worker2   <none>           <none>
kube-system   pod/calico-node-gpkv9                        1/1     Running   0          7m45s   192.168.20.37   k8s-dev-master    <none>           <none>
kube-system   pod/calico-node-j8wrb                        1/1     Running   0          5m7s    192.168.20.38   k8s-dev-worker1   <none>           <none>
kube-system   pod/coredns-f9fd979d6-pswmt                  1/1     Running   0          22m     10.244.134.66   k8s-dev-master    <none>           <none>
kube-system   pod/coredns-f9fd979d6-ztw6c                  1/1     Running   0          22m     10.244.134.65   k8s-dev-master    <none>           <none>
kube-system   pod/etcd-k8s-dev-master                      1/1     Running   0          22m     192.168.20.37   k8s-dev-master    <none>           <none>
kube-system   pod/kube-apiserver-k8s-dev-master            1/1     Running   0          22m     192.168.20.37   k8s-dev-master    <none>           <none>
kube-system   pod/kube-controller-manager-k8s-dev-master   1/1     Running   0          22m     192.168.20.37   k8s-dev-master    <none>           <none>
kube-system   pod/kube-proxy-bnvb2                         1/1     Running   0          22m     192.168.20.37   k8s-dev-master    <none>           <none>
kube-system   pod/kube-proxy-lsfzq                         1/1     Running   0          5m7s    192.168.20.38   k8s-dev-worker1   <none>           <none>
kube-system   pod/kube-proxy-zhtlw                         1/1     Running   0          4m43s   192.168.20.39   k8s-dev-worker2   <none>           <none>
kube-system   pod/kube-scheduler-k8s-dev-master            1/1     Running   0          22m     192.168.20.37   k8s-dev-master    <none>           <none>

NAMESPACE     NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE   SELECTOR
default       service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP                  22m   <none>
kube-system   service/kube-dns     ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   22m   k8s-app=kube-dns

NAMESPACE     NAME                         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE     CONTAINERS    IMAGES                          SELECTOR
kube-system   daemonset.apps/calico-node   3         3         3       3            3           kubernetes.io/os=linux   7m46s   calico-node   calico/node:v3.16.4             k8s-app=calico-node
kube-system   daemonset.apps/kube-proxy    3         3         3       3            3           kubernetes.io/os=linux   22m     kube-proxy    k8s.gcr.io/kube-proxy:v1.19.3   k8s-app=kube-proxy

NAMESPACE     NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS                IMAGES                            SELECTOR
kube-system   deployment.apps/calico-kube-controllers   1/1     1            1           7m46s   calico-kube-controllers   calico/kube-controllers:v3.16.4   k8s-app=calico-kube-controllers
kube-system   deployment.apps/coredns                   2/2     2            2           22m     coredns                   k8s.gcr.io/coredns:1.7.0          k8s-app=kube-dns

NAMESPACE     NAME                                               DESIRED   CURRENT   READY   AGE     CONTAINERS                IMAGES                            SELECTOR
kube-system   replicaset.apps/calico-kube-controllers-7d569d95   1         1         1       7m46s   calico-kube-controllers   calico/kube-controllers:v3.16.4   k8s-app=calico-kube-controllers,pod-template-hash=7d569d95
kube-system   replicaset.apps/coredns-f9fd979d6                  2         2         2       22m     coredns                   k8s.gcr.io/coredns:1.7.0          k8s-app=kube-dns,pod-template-hash=f9fd979d6
```

## Step 6 - Verification and Smoke Test

You should now have a fully running kubernetes cluster.  In this step, we will perform a few tests to ensure everyting is working as expected, especially that the POD networking and CoreDNS are functioning correctly.

We will deploy 2 replicas of `nginx`

```
kubectl create deployment nginx --image=nginx --replicas=2
```
Check to see if each worker has a pod

```
 kubectl get pods -o wide
```

> Output

```
NAME                     READY   STATUS    RESTARTS   AGE    IP               NODE              NOMINATED NODE   READINESS GATES
nginx-6799fc88d8-2slzq   1/1     Running   0          57s    10.244.66.1      k8s-dev-worker2   <none>           <none>
nginx-6799fc88d8-mk4lj   1/1     Running   0          2m7s   10.244.198.129   k8s-dev-worker1   <none>           <none>
```

As we can see, each worker has a pod.  Let's now run a busybox container:

```
kubectl run -i -t busybox --image=busybox --restart=Never
```
From the busy box command line run `wget` using each of the POD IP addresses.

```
/ # wget http://10.244.66.1 -O -
Connecting to 10.244.66.1 (10.244.66.1:80)
writing to stdout
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
-                    100% |******************************************************************************************************|   612  0:00:00 ETA
written to stdout
/ # wget http://10.244.198.129 -O -
Connecting to 10.244.198.129 (10.244.198.129:80)
writing to stdout
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
-                    100% |******************************************************************************************************|   612  0:00:00 ETA
written to stdout
```

If successfully, you now that inter-POD networking is working, even when the pods are on different nodes

We wil now test the DNS.  Let's expose `nginx` as a service:

```
kubectl expose deployment nginx --port=80
```
Get the services:

```
kubectl get services
```

> Output

```
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   104m
nginx        ClusterIP   10.103.191.84   <none>        80/TCP    55s
```
 
Let's run `busybox` again

```
kubectl run -i -t busybox --image=busybox --restart=Never
```
and now run `wget` but use the service name `nginx` isntead of the IP address:

```
/ # wget http://nginx -O -
Connecting to nginx (10.103.191.84:80)
writing to stdout
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
-                    100% |************************************************************************|   612  0:00:00 ETA
written to stdout
```

Use the following to clean up the pods and deployments used for the test

```
kubectl delete pod busybox
kubectl delete deployment nginx
kubectl delete service nginx
```





