# Day 1

These are my notes from the 1st day of studying from the CNCF Official Certified Kubernetes Administrator guide.

## Getting Your Lab Environment Setup

One of the most difficult things about preparing for the CKA exams is getting your lab environment setup.

Kubernetes is designed to function as a distributed system, and the detached nature of the software can make it hard to experiment with if you only have access to a single machine.

My setup is probably unorthodox, but it is definitely working for me.

### My At-Home Lab Setup:

#### 1 Physical Machine (kmaster) - HP Compaq 8200 Elite [CentOS 7/4-core CPU/8G RAM/120G SSD]

I have 2 of these machines. They are cheap $90 dollar units that I obtained from Fry's Electronics. Using the Mac program <a href="https://www.balena.io/etcher/">Etcher</a>, I mounted an ISO of CentOS 7 onto a local USB and installed them easily onto these units. They sit next to my home WiFi router and both have DHCP addresses which are automatically assigned when I plug them into my router via cat5 cables.

Again, I have 2 of these units. One acts as a Kubernetes master, the other acts as a <a href="https://puppet.com/open-source/#osp">Puppet</a> server for this machine and my other k8s VMs in my home network. I also have this physical machine versioned using <a href="https://github.com/teejee2008/timeshift">Timeshift</a>, a snapshot program for physical Linux hosts. If I ever need to revert to a blank, fresh instance of a Kubernetes master (or if I somehow brick my k8s install) Timeshift allows me to do this.  Puppet is how I keep my machines patched, configured, and ready for Kubernetes testing.

![HP Compaq 8200 Elite](https://images.anandtech.com/doci/4867/s-glamour.jpg)

#### 2 VirtualBox VMs (kube0x)-  MacBook Virtual Machines [CentOS 7/3-core CPU/4G RAM/30G VDI]

The worker nodes for my home VM nodes are running off a 2018 MacBook Pro [macOS Catalina/i7 4-core CPU/16G RAM/500 SSD]. The machines are run using VirtualBox 6. I use the snapshot feature of VirtualBox as a tool to revert my machines to a previous state in case I need to.

![MacBook Laptop](http://ascetism.com/mac_book.png)

#### Full Setup

Simple, simple, simple. 3-node setup including a kubernetes master. Again, the nodes communicate over DHCP and the manual editing of the `/etc/hosts` file with all the local DNS names.

![Kubernetes Testing Setup](http://ascetism.com/Kubernetes_setup.png)

The CNCF guides (plus a ton of books out there) suggest that you use an AWS or Google Cloud Platform account to run the labs in the cloud.

I would definitely try and and get a home setup configured first before trying the cloud. You'll save money and get a more intimate understanding of the software, instead of the abstraction of the cloud.

# Kubernetes Installation Steps

Installation is obviously important. These are the steps that I have "translated" from the CNCF guide on Ubuntu installation to CentOS 7. I believe RedHat/CentOS align a bit closer to the production environments me and my professional network work with, so the rest of this guide will be written to accomodate CentOS 7.

### Getting Kubernetes Installed

#### Update your system

Start by getting your system updated. This guide shows how to work with CentOS 7. Search for the equivalent commands if you're using something other than that.

```bash
[user@host:~]$ sudo yum upgrade -y
```

#### Install the latest version of Docker

These are the installation steps for Docker from the official documentation.

Start by adding some YUM repo tools and adding the Docker repository.

```bash
[user@host:~]$ sudo yum install -y yum-utils

[user@host:~]$ sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

Once you have added the repository, you can then run the following command to install Docker. Once the installation completes, be sure to start the service on your system.

```bash
[user@host:~]$ sudo yum install docker-ce docker-ce-cli containerd.io

[user@host:~]$ sudo systemctl start docker
```

Additionally, you can verify your installation works by running the following command and looking for this output.
```bash
[user@host:~]$ sudo docker run hello-world

Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
0e03bdcc26d7: Pull complete 
Digest: sha256:4cf9c47f86df71d48364001ede3a4fcd85ae80ce02ebad74156906caff5378bc
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
```

### Linux Modules You'll Need

We'll explain the modules first and install them later.

There are 2 Linux modules that you'll need to allow Kubernetes to run happily.

----

-<b>overlay</b>: Enables use of "OverlayFS."

Brought into the Linux kernel mainline with version 3.18, OverlayFS allows one directory tree to be overlaid onto another directory tree. Docker will use this for storage.

It is recommended that you use the lastest version of this module: `overlay2`.

If you are using a modern Linux Kernel and current version of Docker, these should already be taken care of. When we say "modern," we mean anything version 4.0 or higher of the Linux kernel, or RHEL/CentOS using version 3.10.0-514 and above.

You can verify what storage driver is being used by running the following command (don't worry about any warnings. We'll fix em.):

```bash
[user@host:~]$ sudo docker info | grep -i storage
WARNING: bridge-nf-call-iptables is disabled
WARNING: bridge-nf-call-ip6tables is disabled
 Storage Driver: overlay2
```

If you're seeing `overlay2`, you're in good shape.

If not, check the Docker's documentation for instruction on how to change this.

-<b>br-netfilter</b>: Allows Linux to create "Network Bridges" and filter network traffic.

A bridge is a piece of software used to unite two or more network segments. 

For our intents and purposes, this allows Docker to bridge the network between the containerized environments and the network that the host machine is using. With this module enabled, we can also configure the "bridge-nf-call" parameters that Docker warned about.

These "bridge-nf-call" params control whether or not packets traversing the bridge are sent to iptables for processing. For Kubernetes case, we want packets to be seen by iptables because that's how Kubernetes applies some routing rules to its pods.


All in all, this module is nesscesary for Docker & Kuberenetes networking.

----

### Enable Linux Modules for Docker

Now that you know what these pieces of software do, you can enable them by running the following commands on your Linux host.

```bash
[user@host:~]$ sudo modprobe overlay; sudo modprobe br_netfilter;
```

We also want to ensure that the "bridge-nf-call" params persist beyond a reboot or system crash. To do this, we can add a few lines to `/etc/sysctl.conf`.

Due to permissions, you'll have to open the file and write to it directly. Here, I use the VIM command to open and write the `net.bridge` values. Add the following lines to this file and run the preceeding command to apply the rules.

```bash
[user@host:~]$ sudo vim /etc/sysctl.conf

...

net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1

[user@host:~]$ sudo /usr/sbin/sysctl -p
```

Once you reload the sysctl values with that last command, all the br_netfilter values should be properly set.

### Installing Kubernetes On CentOS 7

Direct install Kubernetes can be very picky, so we'll start by ensuring that we have the absolute basics covered.

#### Ensure Essential System Settings Are Configured

We'll start by ensuring that our hostnames are statically configured.

Run the following command and enter the values below the command into the very last line of the `/etc/hosts` file.

```bash
[user@host:~]$ sudo vim /etc/hosts
...
#Enter the IP Address of your system
192.168.0.44	kmaster kmaster.node.io
```

Next, let's ensure that we disable SWAP on our system. We can do this by ensuring the following:

A.) There is no "swap" filesystem listed in `/etc/fstab`
B.) If there is a swap filesystem, remove it
C.) Turn off swap explicitly to ensure it won't effect Kubernetes.

```bash
[user@host ~]$ sudo vim /etc/fstab
...
#
# /etc/fstab
# Created by anaconda on Thu Mar 12 17:19:31 2020
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=15b19a19-fc77-439a-8030-c4e21112f81a /boot                   xfs     defaults        0 0
--/dev/mapper/centos-swap swap                    swap    defaults        0 0
```

In the above example, swap is listed as the last filesystem. Remove that line if you find something similar in yours.

The final act in ensuring swap is off is to enter the following command.

```bash
[user@host ~]$ sudo swapoff -a
```

We can verify that our swap is off by ensuring that the "`Swap:`" row in the below command shows "0B" - See below for more details.

```bash
[user@host ~]$ free -h
              total        used        free      shared  buff/cache   available
Mem:           2.9G        282M        747M        8.5M        1.9G        2.5G
Swap:            0B          0B          0B
```

We're using Docker in this example, and we need to ensure that "systemctl" is the cgroup driver for Docker.

First, create the Docker json file where we will specify "systemctl" as the cgroup driver for Docker

```bash
[user@host ~]$ sudo cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF
```

Next, make a folder for the docker systemd file to be created. Reload and restart Docker to ensure the changes take effect.

```bash
[user@host ~]$ sudo mkdir -p /etc/systemd/system/docker.service.d

[user@host ~]$ sudo systemctl daemon-reload
[user@host ~]$ sudo systemctl restart docker
[user@host ~]$ sudo systemctl status docker -l
[user@host ~]$ sudo systemctl enable docker
[user@host ~]$ sudo docker info | grep group
 Cgroup Driver: systemd
```

The following commands will disable SELinux and persist that change beyond a reboot or crash.

```bash
[user@host:~]$ sudo setenforce 0
[user@host:~]$ sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

Lastly, let's ensure that firewalld is disable

```bash
[user@host:~]$ sudo systemctl stop firewalld
[user@host:~]$ sudo systemctl disable firewalld
```


#### Install Latest Kubernetes Repository

Run the following command below to register a new repo on CentOS 7.

```bash
[user@host:~]$ sudo cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```

#### Install Kubernetes (Kubelet, Kubeadm, kubectl)

The most current way to install Kubernetes is to use an administrative tool called `kubeadm`.

The following commands will install `kubeadm` as well as the Kubernetes daemon and Kubernetes command-line tool.

```bash
[user@host:~]$ sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

[user@host:~]$ sudo systemctl enable --now kubelet
```

#### Stage Local System Config For Kubernetes

We will create a `kubeadm` configuration file. This simply provides some explicit settings explaining to k8s how we're running things.

Run the following command to create the kubelet directory. There are also steps to generate a config with the appropriate shortname for the `controlPlaneEndpoint` field below:

```bash


[user@host:~]$ sudo cat <<EOF | sudo tee /var/lib/kubelet/config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: 1.19.1             #<-- Use the word stable for newest version
controlPlaneEndpoint: "kmaster:6443"  #<-- Use the k8s shortname alias not the IP
networking:
  podSubnet: 192.168.0.0/16           #<-- Match the IP range from Calico (Calico is a plugin we will install later)
EOF
```

#### Starting The Kubernetes Cluster

To get Kubernetes running, we can use the `kubeadm` tool with the following options:

```bash
[user@host:~]$ sudo kubeadm init --config=/var/lib/kubelet/config.yaml --upload-certs \
| sudo tee kubeadm-init.out

W1015 14:29:58.192060    7596 configset.go:348] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.19.1
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kmaster kmaster.node.io kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.0.44]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [kmaster.node.io localhost] and IPs [192.168.0.44 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [kmaster.node.io localhost] and IPs [192.168.0.44 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
...
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join kmaster:6443 --token s3v9gi.mx55td15ydcyr81o \
    --discovery-token-ca-cert-hash sha256:9b96698c4fcb2733a2af67f5aecf377b0b08a350393eedd175e2e9f928dbcb7d \
    --control-plane --certificate-key 48932f7622c31b1b8d9523284af1d67c45e8b28d5f1c357fd8b3e9f1fb63d48a

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join kmaster:6443 --token s3v9gi.mx55td15ydcyr81o \
    --discovery-token-ca-cert-hash sha256:9b96698c4fcb2733a2af67f5aecf377b0b08a350393eedd175e2e9f928dbcb7d
```

If you're being invited to join nodes using the `kubeadm join` command, Kubernetes was installed successfully.

#### Allow Sys Users k8s Access

Allowing general system users to control Kuberenetes is easy. Just run the commands below to create the config required.

```bash
[user@host:~]$ mkdir -p $HOME/.kube
[user@host:~]$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[user@host:~]$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
[user@host:~]$ cat $HOME/.kube/config
```

#### Installing A Network Plugin (Calico)

For this example, we will use the networking plugin that is suggested by the CNCF (Calico).

Calico is an open source networking and network security solution for containers, virtual machines, and native host-based workloads. Calico’s network policy engine has been in use since the early iterations of k8s network API.

We can employ Calico's default network config, which is more than sufficient for our purposes. We will download and apply this default config using the `kubectl apply` command.

```bash
[user@host:~]$ kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

### Finishing Touches

To wrap up, we'll install some helper tools that will allow tab completion for Kubernetes commands. Run the followings commands to get these tools setup.

```bash
[user@host:~]$ sudo yum install bash-completion -y
[user@host:~]$ source <(kubectl completion bash)
[user@host:~]$ echo "source <(kubectl completion bash)" >> $HOME/.bashrc
```
