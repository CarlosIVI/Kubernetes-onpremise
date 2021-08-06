# Kubernetes HA on premise: Kubernetes cluster with Kubeadm, KeepAlived and HAProxy



This tutorial is focus in a particular way to bootstrap a Kubernetes cluster, this approach is very usefull to get know of all parts of a kubernetes cluster, its funtions and crucial information for a correct operation. For production enviroments is highly recommend use a Kubernetes service provide and manage by a Cloud provider due to all the complexity that carries manage, update and mantain in good conditions a k8s cluster.

> This tutorial was inspired by [Kubernetes the hard way](https://github.com/kelseyhightower/kubernetes-the-hard-way) 

## Target Audience

The target audience for this tutorial is someone that wants to learn how bootstrap a k8s cluster and understand its principal features and limitations in an on premise environment.

## Cluster Details

Kubernetes HA on premise guides you through bootstrapping a highly available Kubernetes cluster with end-to-end encryption.

* [kubernetes](https://github.com/kubernetes/kubernetes) v1.21.2
* [containerd](https://github.com/containerd/containerd) v1.5.2
* [calico](https://github.com/projectcalico/calico) v3.18.4
* [docker](https://github.com/docker/docker-ce) v19.03.14
* [etcd](https://github.com/etcd-io/etcd) v3.4.15
* [keepalived](https://github.com/acassen/keepalived) v2.2.2
* [haproxy](https://github.com/haproxy/haproxy)
* [kubeadm](https://github.com/kubernetes/kubeadm) 
* [ubuntu](https://github.com/ubuntu)


## Getting started

For this tutorial we are going to use 6 Virtual Machine on Ubuntu 18.04 LTS, you could use VMware or another hypervisior for creating all the VM, 2 of this VM are going to be used on the install HAProxy and KeepAlived those VM doesn't need too much RAM and CPU, with at least 500 MB of RAM and 1 CPU Core should be enough, on the official Kubernetes documentation recommends for cluster nodes 2 GB RAM and 2 CPU Cores.

### Summary

* [HAProxy and KeepAlived Machine] 500MB RAM | 1 CPU Core
* [HAProxy and KeepAlived Machine] 500MB RAM | 1 CPU Core
* [Kubernetes Master Node 1] 2GB RAM | 2 CPU Core
* [Kubernetes Master Node 2] 2GB RAM | 2 CPU Core
* [Kubernetes Worker Node 1] 2GB RAM | 2 CPU Core
* [Kubernetes Worker Node 2] 2GB RAM | 2 CPU Core

## Architecture explanation

The architecture to deploy on this tutorial is focus on ensure a High Avaiability Cluster, with redundant master. if you want a brief explanation of Kuberneres you could go to this link on this repo:

* [Introduction to Kubernetes](https://github.com/CarlosIVI/Kubernetes-onpremise/blob/main/introduction-to-kubernetes.md)

![imagen github](https://user-images.githubusercontent.com/31323133/125170141-02d60080-e173-11eb-820e-d6c82efac463.png)


## General pre-requisites

We are going to prepare all our cluster node virtual machines for the Kubernetes installation, first of all, we must disable the swap, we could do this by executing the following commands in the CLI:


```sh
sudo swapoff -a
```

After this you must go to the /etc/fstab file and comment the that refers to the swap.

![image](https://user-images.githubusercontent.com/31323133/127594065-382aefa8-7fb3-4e1c-aa95-04b2300b59d1.png)

It is important to update all packages and reboot systemctl.

```sh
sudo apt -y upgrade 
sudo systemctl reboot
```

## Installing Kubernetes

To install Kubernetes we must let iptables see bridged traffic, to make sure of that you must load module `br_netfilter` and you have to ensure `net.bridge.bridge-nf-call-iptables` is set in 1.

```sh
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-iptables  = 1
EOF

sudo sysctl --system
```
Is not necesary but recommended install utilities as VIM, GIT, CURL and WGET.

```sh
sudo apt -y install vim git curl wget
```

After that update the `apt` package index, download the Google Cloud public signing key and add the Kubernetes `apt` repository:

```sh
sudo apt update

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

```

Update again `apt` package index, install kubectl, kubeadm and kubelet

```sh
sudo apt-get update

sudo apt -y install kubelet kubeadm kubectl

sudo apt-mark hold kubelet kubeadm kubectl
```

## Containerd as a container runtime

> Note: On December 2020, Kubernetes anunced deprecation of Docker as Container Underlying Runtime

Is necesary to add config lines to `/etc/modules-load.d/container.conf`

```sh
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```
Apply changes without reboot 

```sh
sudo sysctl --system
```
Install containerd and create the containerd configuration file

```sh
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

In /etc/containerd/config.toml, around line 86, on section `[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]` add this lines, be careful with identation.

```sh
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```

Restar containerd

```sh
sudo systemctl restart containerd
```
Enable kubelet, pull config images and enable packet forwarding for IPV4 on  `/etc/sysctl.conf` 

```sh
systemctl enable kubelet
sudo kubeadm config images pull
echo 1 > /proc/sys/net/ipv4/ip_forward
nano /etc/sysctl.conf
```

![image](https://user-images.githubusercontent.com/31323133/127960742-b379d7e1-d616-46c7-886f-7bc309a52547.png)

## Load Balancer configuration

On the others ubuntu VM (500MB RAM) install HAProxy and KeepAlived with the Virtual Redundacy Routing Protocol (VRRP) for ensure high avaibility on the Kubernetes Cluster. Is necesary because kubeadm requires a principal IP to init a cluster, this IP is going to be used as an endpoint, if that endpoint dies, the whole cluster is going to fall (Workers may still alive but won't respond to changes)

Install keepalived and edit the configuration file on `/etc/keepalived/keepalived.conf`

```sh
global_defs {
   notification_email {
     email@email.com
   }
   notification_email_from email@email.com
   smtp_server localhost
   smtp_connect_timeout 30
}

vrrp_instance VI_1 {
    state MASTER         
    interface eth0       #Interface used to active VRRP
    virtual_router_id 101   
    priority 101         #If the node is MASTER priority must be the highest
    advert_int 1
    authentication {
        auth_type PASS  
        auth_pass 1111   #Must Change
    }
    virtual_ipaddress {
        5.5.5.5  #Virtual IP to use (Must be on the same network)
    }
}
```



