# How To Install Vanilla Kuberntes on Centos 7

## Introduction

This guide will describe the process of setting up a vanilla Kubernetes (K8s) cluster on a single Dell server using Kernel-based Virtual Machines (KVM). The goal is to create a fully functional, minimalistic Kubernetes environment that can serve as a robust test or development cluster.

**Why Use KVM on CentOS 7?**

- **Virtualization**: KVM provides a type-1 (bare-metal) hypervisor, which means it operates directly on the hardware of your Dell server, offering better performance and resource utilization compared to type-2 hypervisors. This is ideal for creating isolated environments where each virtual machine acts as a node in your Kubernetes cluster.

- **CentOS 7**: Known for its stability and compatibility with enterprise-level applications, CentOS 7 remains a preferred choice among system administrators for setting up robust server environments. Its long-term support and wide community backing ensure that you have access to necessary documentation and community help.

- **Test/Dev Environment**: Installing Kubernetes in a virtualized setup allows for flexibility in testing different configurations, experimenting with new features, or simulating production environments without the risk of affecting real production systems.

**Key Benefits of This Setup:**

- **Isolation**: Each virtual machine can be configured with different resources or network settings, providing an excellent environment for testing various scenarios or application behaviors.

- **Resource Management**: KVM on a Dell server allows for fine-tuned control over resource allocation, which is perfect for simulating different workloads or testing scalability.

- **Learning and Experimentation**: For those learning Kubernetes, this setup offers a hands-on approach to understanding cluster management, networking, and container orchestration without the cost of cloud services.

**What This Guide Covers:**

- **Docker Installation**: Installing Docker as the container runtime for Kubernetes.
- **Kubernetes Components**: Installing `kubeadm`, `kubelet`, and `kubectl` on your nodes.
- **Cluster Initialization**: Bootstrapping your Kubernetes master node and joining worker nodes.
- **Networking**: Configuring networking for your VMs to ensure proper communication within the cluster.
- **Post-Installation**: Verifying your cluster's health and basic operations.

## Prerequisites

- A Dell server with sufficient CPU, memory, and disk resources (at least 16 GB RAM and 4 CPUs recommended) with Centos 7 installed. Note: for the host machine the OS system does not have to be Centos it can be any Linux distribution that supports KVM.

- KVM and associated tools (libvirt, virt-manager, and virsh) installed on the server.
  
- Centos 7 OS installed on two KVM VMs one for the master and one for the worker nodes.

- Basic familiarity with Linux command-line operations.

Here the official sites for installing Centos7 and KVM. However, there are plenty of good tutorials available on the Web.  
[Centos Install Guide](https://docs.centos.org/en-US/centos/install-guide/ "Centos Web Site")  
[KVM How to](https://linux-kvm.org/page/RunningKVM)

## Step-by-Step Instructions

### ***Steps 1 thorugh 10 are to be performed on master node and all worker nodes!***

#### 1. Disable selinux and firewalld

Note, if you need a firewall running must open the ports required for all applications. For example, the API server uses port 6443. Click on the below link for the ports required.
<https://kubernetes.io/docs/reference/networking/ports-and-protocols/>

```bash
setenforce 0
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
systemctl stop firewalld
systemctl disable firewalld
modprobe br_netfilter
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
```

#### 2. Disable swap

```bash
sudo swapoff -a
```

#### 3. Install Kubernetes repository

```bash
cat <<EOF> /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=<https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64>
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=<https://packages.cloud.google.com/yum/doc/yum-key.gpg> <https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg>
EOF
```

#### 4. Install Kubernetes

Configure kubernetes repo

```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```

#### 5. Install kubectl, kubeadm and kubectl

```bash
yum update
yum install -y kubelet kubeadm kubectl
systemctl enable kubelet
systemctl start kubelet
```

#### 6. Install Docker

If you need to unistall an old verison of docker.

```bash
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine

```

#### 7. Configure docker repository

```bash
yum install -y yum-utils

yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

#### 8. Install docker docker compose and containerd

```bash
yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Start and enable docker

```bash
systemctl start docker
systemctl enable docker
```

#### 9.Ensure docker is running

```bash
docker run hello-world
```

#### 10. Remove config.toml file

You must remove this file otherwise when the cluster is initialized it will fail.

```bash
cd /etc/ containerd/
rm config.toml
systemctl start containerd
systemctl enable containerd
```

### ***These next steps are to be performed on the Master Node only!***

````bash
kubeadm init
````

This will take a few minutes. At the end of the intialization you should see the message.

```bash

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  /docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each worker node as root:
  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```
For example on your cluster replace your local info such as:

```bash
kubeadm join --token 7c0767.e8467d0b00930f2e 172.18.10.84:6443 --discovery-token-ca-cert-hash sha256:839784bd665967b84c61eed98a797beb684d6cda716a07200d524b6d49cf127
```

## Installing a Pod Network

The final step is to install a pod network application for intra cluster communication, in this case Calico.
Download the Calico networking manifest for the Kubernetes API datastore.

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml -O
```

Apply the calico yaml to the cluster.

```bash
kubectl apply -f calico.yaml
kubectl get pods -n kube-system
```

### Verifying Installation

Form the master node verify your nodes are in the ready state using the kubectl get nodes command.

```bash
demo@training-kube-master$ kubectl get nodes -o wide
```

If you are interested in a video demonstration on how to perform basic interactions with pods and nodes click on the link below.
Note this is a partial video for demonstration purposes only.

[Sample Work K8s lab](https://app.screencast.com/qMiDaXOTnY9rt)


It will cover:

- **Viewing the cluster's current state** – Understanding what's running in the cluster.
- **Creating and managing Pods** – The basic units of deployment in Kubernetes.
- **Scaling applications** – Adjusting the number of running instances of an application.
- **Exploring Kubernetes objects** – Learning about services, deployments, and namespaces.
