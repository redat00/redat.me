+++
title = 'Installing Your First Ever Kubernetes Cluster With Kubeadm'
date = 2025-04-27T23:08:57+02:00
draft = false
ShowToc = true
ShowCodeCopyButtons = true
ShowReadingTime = true
ShowShareButtons = true

[cover]
image = "images/cover.png"
+++

I'm still tinkering around with those mightful mini Dell PC that I got for free. This time however I wanted to play around with MetalLB and Rook. This meant that I had to install my own Kubernetes cluster.

While trying to set-up my cluster, I found that the official "Getting started" documentation for Kubernetes is kind of going everywhere and nowhere at the same time. The goal for this blog post is to demystify all of those weird pre-requisites and show how easy it can be to deploy a Kubernetes cluster. This blog post is a well a documentation for you, than for me.

# Pre-requisites

You first have to install your nodes in a certain way for them to be able to have kubernetes intalled on them.

Here's a quick list of the main pre-requisites that we'll go through : 

- [Load required modules on your system](#load-required-modules-on-your-system)
- [Make sure swap is disabled](#make-sure-swap-is-disabled)
- [Configure sysctl parameters](#configure-sysctl-parameters)
- [Install a container runtime (CRI-O)](#install-a-container-runtime-cri-o)

Please note that everything I'm presenting here is done on Ubuntu 24.04 nodes. While everything should also work under a Debian environment, I cannot be entirely sure of it.

## Load Required Modules On Your System

Under a Linux system, capabilities are added to your kernel through the use of "modules". Enabling/loading new modules will extend the capabilities of your host. 
In the case of a Kubernetes cluster, we have two modules that are really important : `br_netfilter` and `overlay`.

### overlay

Overlay is a module responsible for bringing the capabilities for your kernel to work with the `OverlayFS` filesystem.
While you might never have heard about this filesystem (as opposed to the more common ones such as ext4 or LVM), this filesystem is really important for us, as it basically allow container images to work in the most efficient way.

If you want to learn more about it and how everything works under the hood, you can read [this great article](https://www.terriblecode.com/blog/how-docker-images-work-union-file-systems-for-dummies/) that explain how a Docker image make use of the OverlayFS filesystem.

In our case however, the only thing we need to do, is check wether this module is already enabled and in the case it's not, enable it.

In order to check if the module is loaded, you can use the following command, and if you don't have a similar output, you can assume it's not.
```bash
~# lsmod | grep overlay
overlay               212992  19
```

In order to enable it you're going to create a file containing the the name of the module (ie. `overlay` in this case) in a specific folder so that your system know it should load this module at startup.

```bash
echo "overlay" >> /etc/modules-load.d/k8s.conf
```

Doing so will make sure that the module is loaded at boot, but what about right now ? The following command will load the kernel module right now, while the configuration file will make sure to do it at startup.

```bash
modprobe overlay
```

### br_netfilter

The second module we need to make sure is loaded is `br_netfilter`. This module is responsible for allowing to do packet filtering on bridges. Basically it brings the ability to use the same packet filtering that exist within the Linux kernel, the one you use with iptables/nftables and ebtables, to bridges.

In our case this module is required because some CNI (Container Network Interfaces) will work by making use of bridges, and secure those bridges by using this module.

Just like for the `overlay` module, we can check if it's enabled with the following command.

```bash
~# lsmod | grep br_netfilter
br_netfilter           32768  0
```

If no output is showing, then it's not enabled. Enable it at startup like you did with the `overlay` filter.

```bash
echo "br_netfilter" >> /etc/modules-load.d/k8s.conf
```

And enable right now without having to reboot with the following command.

```bash
modprobe br_netfilter
```

And voila! Required modules are now loaded, and will be loaded everytime your node start.

## Make Sure Swap Is Disabled

Kubernetes is not a huge fan of swap, for mutlilple reasons :

- It's hard to actually determine how much SWAP space is still free, and to not over-allocate the space throughout your pods.
- It's also a security concern, as the swap is a shared memory space accross your applications, and thus would result in potential security issues if one was able to access it in a wrongful way.

For all of those reason, Kubernetes recommend to disable the swap on any host of your cluster.

Only two steps are required in order to disable swap. First, run the following command, that will disable any existing swap temporarily without the need to reboot.

```bash
swapoff -a 
```

To make it permanent upon reboot, you will have to edit the `/etc/fstab` file which reference either the disk partition or the swapfile to be mounted for swap to work. Simply open it using your favorite editor.

```bash
vim /etc/fstab
```

And remove, or comment, any line that reference a swap partition, such as the following.

```bash
/dev/sda2              swap          swap      defaults              0      0
```

> In some cases swap might be setup through systemd. If you've edited the `fstab` file but the swap is still there upon reboot, you might be in this case. [This blog post](https://rageagainstshell.com/2019/12/disable-swap-in-systemd/) I found will actually guide you through disabling it definetely.

## Configure sysctl parameters

Some parameters are required to be set within the kernel so that everything works correctly. The four main rules are all related to network.

- `net.bridge.bridge-nf-call-iptables` : Make the packets coming from a bridge to also go through iptables, which is not the case by default. This behavior is required as some CNI within Kubernetes make use of `iptables` under the hood. 
- `net.bridge.bridge-nf-call-ip6tables` : Does exactly the same thing as the previous rule, but for IPv6.
- `net.ipv4.ip_forward` : Enable forwarding of packets, in the case of IPv4.
- `net.ipv6.conf.all.forwarding` : Enable forwarding of packages, in the case of IPv6.

Those parameters needs to be enabled through `sysctl` which is a software mechanism allowing to pass specific parameters to the kernel, either dynamically at any given moment, or at boot, through the use of configuration files.

In our case we want those parameters to persist after reboot. To do so, we can simply run the following commands.

```bash
echo "net.bridge.bridge-nf-call-iptables=1" >> /etc/sysctl.d/99-k8s.conf
echo "net.bridge.bridge-nf-call-ip6tables=1" >> /etc/sysctl.d/99-k8s.conf
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.d/99-k8s.conf
echo "net.ipv6.conf.all.forwarding=1" >> /etc/sysctl.d/99-k8s.conf
```

Once those commands are run, go ahead and reboot. 

To make sure that the parameters are correctly set, you can use the following command. The output should be `1`.

```bash
cat /proc/sys/net/bridge/bridge-nf-call-iptables
```

Adapt the command depending on the parameter you want to check. The filename and the directory tree are derived from the parameter name.

With this you should be all set! We can now go ahead and install our container runtime.

## Install A Container Runtime (CRI-O)

After all of the preparation we can finally go ahead and start installing stuff! First thing we'll install is simply the container runtime. It is required on all the nodes you'll add to your cluster!

If you're here I'll assume that you already know what Kubernetes is compared to solution such as Podman or Docker. 

In the case you're not aware of the difference, here's a quick reminder : Kubernetes is NOT a container engine. It is only and strictly a container orchestration solution. Meaning that all Kubernetes do is orchestrating how the containers are deployed on nodes, and, through the CNI, how they can communicate with each other, and the world.

All of this means that Kubernetes will require a container engine to be installed on each node. 

In our case, we'll install CRI-O. 

> Why CRI-O over `containerd` ? Well both solution are quite similar. However I prefer using CRI-O since it prevents me from having to configure `cgroups` on my nodes, as the default configuration of CRI-O is also the default configuration for `kubelet`.

### Add CRI-O Repository

CRI-O is not distributed through the official Ubuntu repositories. In order to have the ability to install it, you'll need to add the official CRI-O package repository on all your nodes.

The first step will be to define the version of CRI-O we're going to install. CRI-O follows the same version as Kubernetes, because of this, at the time of writing, the latest version will be `v1.32`

```bash
export CRIO_VERSION=v1.32
```

Proceed by updating your repos sources, and installing basic required packages.

```bash
apt-get update && apt-get install -y curl gpg software-properties-common apt-transport-https
```

And then download the gpg key of the official CRI-O Apt repository, and add the Apt configuration file for the new repository.

```bash
curl -fsSL https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/ /" | tee /etc/apt/sources.list.d/cri-o.list
```

### Install CRI-O 

Once the repository is set, go ahead and install the `cri-o` package.

```bash
apt-get update && apt-get install -y cri-o
```

Finish by starting the CRI-O service, and you'll be all down.

```bash
systemctl start crio.service
```

### Optional : Install `crictl`

If you wanted to, you could also install the `crictl` tool which would allow you to interact with the CRI-O engine directly! While it's not required, it could reveal itself handy in the event of a debugging session.

The installation of the `crictl` tool is not done through packages however, you'll have to dowload it from Github, where it's built and released.

Start by downloading the tarball of the latest version. In our case it's `v1.32.0`.

```bash
curl -L https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.32.0/crictl-v1.32.0-linux-amd64.tar.gz > crictl.tar.gz
```

Then, extract the tarball.

```bash
tar -xzf crictl.tar.gz -C /usr/bin
```

And finally, to make sure everything is working, simply run the following command, which should work without issues.

```bash
~# crictl ps
CONTAINER           IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID              POD                 NAMESPACE
```

# Installing The Cluster

## Install Kubernetes Tools

Now that everything non-related (at least not directly) to Kubernetes is installed, we can start installing and configuring actual Kubernetes stuff.

So what do you need in order to bootstrap a Kubernetes cluster ? Only three things, actually.

- Kubelet : The application responsible for communication with the local container runtime.
- Kubeadm : The tool used to administrate our Kubernetes cluster at a more deeper level. It will be used to bootstrap the cluster, or to add a node to a cluster.
- Kubectl : The tool used to interact with the Kubernetes cluster. It's used by "users" of the Kubernetes cluster, on top of administrator of the cluster.

### Add Kubernetes Packages Repository

First step will be to add the Kubernetes package repository on our node. This is similar to what we've done for CRI-O actually, but here are the actual commands.

Just like for CRI-O, we're setting the version of Kubernetes we plan to use. In my case the latest is `v1.32`.

```bash
export KUBERNETES_VERSION=v1.32
```

Then download the key used to sign the repository, and add the repository source list file.

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list
```

### Installing The Packages

Once the package repository is set-up, simply update the sources, and install the packages.

```bash
apt-get && apt-get install kubelet kubeadm kubectl
```

And voila, everything left to be done is the actual bootstrap of our cluster!

## Bootstrap The Cluster

This is only to be done on the master node! Not on every node.

Bootstrapping the cluster is really easy, as it's basically one simple command. 

```bash
kubeadm init
```

> During a production grade cluster setup, you might want to pass some more parameters to the `kubeadm init` command. In our case we're going simply, but check the possible parameters that can be passed using the `kubeadm init --help` command.

If everything was done perfectly using the beggining of this guide, you won't have any error, and your master will be healty !

You can assert that everything's fine by running the following command.

```bash
~# kubectl --kubeconfig /etc/kubernetes/admin.conf get nodes
NAME           STATUS     ROLES           AGE     VERSION
kube-master    NotReady   control-plane   7m55s   v1.32.3
```

## Installing A CNI

As I stated before, Kubernetes is not a runtime in any way. It is really only just an orchestrator. This mean that you will also be required to install what's called a "CNI".

CNI in the Kubernetes world stands for "Container Network Interface" and is a plugin that will be used to configure the network for pods. It is invoked by the kubelet on each node upon pod creation.

In our case we will install Calico as it is easy to install and get started with.

By running the following command you will apply configuration onto your Kubernetes cluster. It will mostly create what's called `CustomResourceDefinition`. In the Kubernetes world everything, mostly, is a `Resource`. A pod is a resource, a service account is a resource, a config map is a resource...

```bash
kubectl --kubeconfig /etc/kubernetes/admin.conf apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

> Out of curiosity you should read the content of the file we're applying, it will give you an idea toward what we're actually doing.

## Joining A Node To The Cluster

Now everything that's left to do is to add a new node to the cluster! For now you've only set-up stuff on the master node, but we will go ahead and add some worker node on which you can deploy actual stuff.

This is done in two steps : 
- We will first create a new token through kubeadm to be used by the worker node to join the cluster.
- Then we will use the provided command to join the worker node to the cluster.

### Creating The Token

The command to create a new token is really simple, but in our case we will add a little twist to it with the `--print-join-command` parameter which will provide a command we can then copy/paste.

```bash
~# kubeadm token create --print-join-command
kubeadm join 192.168.64.10:6443 --token cv3of2.tceacc5se0yu9hy8 --discovery-token-ca-cert-hash sha256:378dc5a64285e37e3d4618ab49149925d0b5a0b360ff0619b6c8ade511b365f4
```

### Joining The Cluster

Then, on the worker node you want to add to the cluster, simply run the provided command.

You can check the status of the nodes on the master using the `kubectl get nodes` command. They will be healhy when the status is `Ready`.

```bash
~# kubectl --kubeconfig  /etc/kubernetes/admin.conf get nodes
NAME          STATUS    ROLES           AGE   VERSION
kube-master   NotReady  control-plane   13d   v1.32.3
kube-worker01 Ready     <none>          13d   v1.32.3
kube-worker02 Ready     <none>          13d   v1.32.3
```

## Conclusion

And just like that, you've successfully installed a fully-featured Kubernetes cluster ready to be used to practice and to learn more! This cluster is close to a production grade cluster, and you can test the full extent of Kubernetes with it. 

Personally I will make use of this cluster to learn more about on-premise Kubernetes deployment, with solution such as Rook to provide storage through Ceph for my containers, and solutions such as MetalLB which bring load balancing to your Kubernetes cluster without having it to run in the cloud!

I hope that this little guide helped you figure out everything I struggled with in the beggining to get started with Kubernetes.

