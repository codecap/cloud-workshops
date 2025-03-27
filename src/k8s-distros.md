---
title:       Kubernetes Distributions
description: Workshop and Tasks
author:      Vladislav Nazarenko (vnazarenko@📯socket.de)
keywords:    Kubernetes,OpenShift,k3s.minikube,microk8s,kind
url:         
image:
transition: cover
theme: default
backgroundImage: url(https://codecap.github.io/cloud-workshops/assets/background.jpg)
paginate: true
---

# Kubernetes Distrubutions
![bg left:40% 80%](https://upload.wikimedia.org/wikipedia/commons/3/39/Kubernetes_logo_without_workmark.svg)

---
[Comparison](https://nubenetes.com/matrix-table/)

Test & Learn:
- kind
- minikube
- microk8s
- k3d

Vanila:
- [k0s](https://k0sproject.io/)
- k3s

openshift

## TODO: more distros wirh multi node mode ?



---

# Comparison



```
|------------|----------------------------------------------|------------------------------------------------------------------------------|
|            | Pro                                          | Contra                                                                       | 
|------------|----------------------------------------------|------------------------------------------------------------------------------|
|Minikube    |✅ Easy to do manual configurations           | 👎 Only single-node cluster                                                  |
|            |✅ Gives more access to the system            | 👎 Hard to set up                                                            |
|            |✅ The most widely used and the oldest distro | 👎 Uses more resources                                                       | 
|------------|----------------------------------------------|------------------------------------------------------------------------------|
|MicroK8S    |✅ Easy to setup                              | 👎 Can be set up as a multi-node cluster                                     |
|            |✅ Uses fewer resources                       | 👎 Can’t be installed on machines with ARM32 CPUs                            |
|            |✅ Up to date with Kubernetes releases        | 👎 Doesn’t give much access to the system                                    |
|------------|----------------------------------------------|------------------------------------------------------------------------------|
|Kind        |✅ Very easy to install                       | 👎 Hard to set up with other distros than Docker                             |
|            |✅ Uses fewer resources                       | 👎 Making manual configuration changes is difficult                          |
|            |✅ Easily accessible using docker commands    |                                                                              |
|            |✅ Containers are considered nodes            |                                                                              |
|------------|----------------------------------------------|------------------------------------------------------------------------------|
|k3s         |✅ Very easy to setup                         | 👎 Can make unwanted network changes while auto-configuration during setup   |
|            |✅ Uses fewer resources                       | 👎 Manual configuration changes are difficult                                |
|            |✅ Allows multi-node cluster setup            |                                                                              |
|------------|----------------------------------------------|------------------------------------------------------------------------------|
|k0s         |✅ Very easy to setup                         | 👎 lightwieght and lacks of full k8s distribuation                           |
|            |✅ Uses fewer resources                       | 👎 may effort require more efforts to implement specific configuration       |
|            |✅ Allows multi-node cluster setup            |                                                                              |
|            |✅ One binary                                 |                                                                              |
|------------|----------------------------------------------|------------------------------------------------------------------------------|

```

---
# K8S parts
* Container Runtime Interface (CRI)
* Container Network Interface (CNI)
* Contaienr Storage Interface (CSI)
* K8S (api, scheduler, controller-manager, kubelet, kube-proxy)

---
# K8S Architecture
## TODO: Diagram

---
# K8S Addons
* [Installing Addons on Kubernetes](https://kubernetes.io/docs/concepts/cluster-administration/addons/)

---
# Simple Deployment
(Emojivoto)[https://codecap.github.io/cloud-workshops/k8s-fundamentals.html#249]

---
# Arkade
```bash
# install arkade
curl -sLS https://get.arkade.dev | sh; mkdir -p  ~/.arkade/bin/; mv arkade ~/.arkade/bin/
echo 'export PATH="~/.arkade/bin/:$PATH"' > ~/.profile; source ~/.profile

arkade get kubectl
```

---
# Preparations
```bash

# on master01
ssh-keygen
MY_USER=rocky

echo "echo \"$(cat ~/.ssh/id_rsa.pub)\" >> ~$MY_USER/.ssh/authorized_keys"


echo "$MY_USER ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/$MY_USER
# then execute the outpit on each master and worker nodes
```


---
# Fix to many open files
```bash
# fix too many open files
echo "fs.inotify.max_user_watches = 524288" | sudo tee  -a /etc/sysctl.d/99-kind.conf
echo "fs.inotify.max_user_instances = 512"  | sudo tee  -a /etc/sysctl.d/99-kind.conf
sudo sysctl --system
```



---
# kubeadm
(Link)[https://phoenixnap.com/kb/install-kubernetes-on-rocky-linux]

```bash
# on each cluster node
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

echo "net.ipv4.ip_forward = 1"                 | tee -a /etc/sysctl.d/k8s.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" | tee -a /etc/sysctl.d/k8s.conf
echo "net.bridge.bridge-nf-call-iptables = 1"  | tee -a /etc/sysctl.d/k8s.conf
sysctl --system

echo "overlay"      | tee -a /etc/modules-load.d/k8s.conf
echo "br_netfilter" | tee -a /etc/modules-load.d/k8s.conf
modprobe overlay
modprobe br_netfilter

dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
dnf install -y containerd.io

mv /etc/containerd/config.toml /etc/containerd/config.toml.bak
containerd config default > /etc/containerd/config.toml
sed -e "s/SystemdCgroup = .*/SystemdCgroup = true/" -i /etc/containerd/config.toml
systemctl enable --now containerd.service


setenforce 0
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/sysconfig/selinux

# NOTE: check firewall

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes


kubeadm config images pull
kubeadm config images list


# on master01
MY_IP=$(ip a show eth0 | grep "inet " |  awk '{print $2}' | awk -F/ '{print $1}')
kubeadm init \
   --control-plane-endpoint "$MY_IP:6443" \
   --upload-certs \
   --pod-network-cidr "192.168.0.0/16" \
   --apiserver-advertise-address=$MY_IP

ctr namespace ls
ctr -n k8s.io image ls
ctr -n k8s.io container  ls

echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bashrc; source ~/.bashrc
kubectl get pods  -A
kubectl get nodes  -A

# CNI
curl -sS https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/calico.yaml \
  | sed -r -e "s/docker.io/quay.io/" \
  | kubectl apply -f -


# join master nodes
kubeadm join $MASTER01_IP:6443 --token    [TOKEN] \
        --discovery-token-ca-cert-hash    [HASH] \
        --control-plane --certificate-key [KEY]]

# join worker nodes
kubeadm join 10.10.20.39:6443 --token     [TOKEN] \
        --discovery-token-ca-cert-hash    [HASH]

# list nodes
kubectl get nodes  -owide
```


# kubeadm reset
```bash
kubeadm reset
rm -rf  /etc/kubernetes/manifests/*
rm -rf  /etc/cni/net.d
ctr -n k8s.io container  ls | grep runc | awk '{print $1}' | while read id; do ctr -n k8s.io container delete $id; done
dnf remove -y containerd.io
reboot
```



---
# kind
```bash
arkade get kind

kind create cluster --config - <<EOF
---
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: hello
nodes:
- role: control-plane
- role: control-plane
- role: control-plane
- role: worker
- role: worker
- role: worker
EOF
kind get kubeconfig --name hello > ~/.kube/config
```

---
# minikube
```bash
# install minikube
arkade get minikube

minikube start

kubectl get nodes
kubectl get pods -A

# install dashvoard
minikube dashboard
minikube addons enable metrics-server
# inctall ingres controller
minikube addons enable ingress
minikube addons enable ingress-dns
# install metallb
minikube addons enable metallb

# Pause Kubernetes without impacting deployed applications:
minikube pause
# Unpause a paused instance:
minikube unpause
# Halt the cluster:
minikube stop
# loging to the minikube node
minikube ssh
# Browse the catalog of easily installed Kubernetes services:
minikube addons list
# Create a second cluster running an older Kubernetes release:
minikube start -p aged --kubernetes-version=v1.16.1
# Delete all of the minikube clusters:
minikube delete --all

# NOTE: metallb deloyment from addons slide dindnot work
```
---
# microk8s
Ubuntu !
[addons](https://microk8s.io/docs/addons#heading--list)

```bash
sudo snap install microk8s --classic --channel=1.32

# chekc if ready
microk8s status --wait-ready

# join cluster
microk8s add-node

# start
microk8s stop

# stop
microk8s start

# enable addons
microk8s enable hostpath-storage
microk8s enable metrics-server
microk8s enable dashboard
microk8s enable metallb
microk8s enable ingress
microk8s enable registry

# build-in kubectl 
 microk8s kubectl ...
```


---
# k0s
[Docs](https://docs.k0sproject.io/stable/k0sctl-install/)

## single-node cluster
```bash
# install k0s
arkade get k0s

# install single-node cluster
k0s install controller --single
k0s start
k0s status
k0s kubectl get nodes
k0s kubeconfig  admin > ~/.kube/config
kubectl get nodes
kubectl get pods -A

# reset the cluster
k0s stop
k0s status
k0s reset
```

---
# k0sctl
```bash
# prepare
mkdir -p /run/k0s
cd /run/k0s
ln -s /run/containerd/containerd.sock
ln -s /run/containerd/containerd.sock
cd 

# on master01
arkade get k0sctl

MY_USER=rocky
IP_BASE=[IP_BASE]

cat > ~/k0sctl.yaml <<EOF
---
apiVersion: k0sctl.k0sproject.io/v1beta1
kind: Cluster
metadata:
  name: k0s-cluster
spec:
  hosts:
  - role: controller
    ssh:
      address: $IP_BASE.
      user: $MY_USER
      keyPath: ~/.ssh/id_rsa
  - role: controller
    ssh:
      address: $IP_BASE.
      user: $MY_USER
      keyPath: ~/.ssh/id_rsa
  - role: controller
    ssh:
      address: $IP_BASE.
      user: $MY_USER
      keyPath: ~/.ssh/id_rsa
  - role: worker
    ssh:
      address: $IP_BASE.
      user: $MY_USER
      keyPath: ~/.ssh/id_rsa
  - role: worker
    ssh:
      address: $IP_BASE.
      user: $MY_USER
      keyPath: ~/.ssh/id_rsa
  - role: worker
    ssh:
      address: $IP_BASE.
      user: $MY_USER
      keyPath: ~/.ssh/id_rsa
EOF
# Add NODE IPs into ~/k0sctl.yaml

k0sctl apply --config ~/k0sctl.yaml

mkdir ~/.kube
k0sctl kubeconfig > ~/.kube/config


# review runnning containers on the node
# master
systemctl status k0scontroller.service
cat /etc/systemd/system/k0scontroller.service

# worker
apt-get install containerd -y
ctr --address /run/k0s/containerd.sock -n k8s.io c ls

# reset
k0sctl reset --config ~/k0sctl.yaml
```



---
# k3s
```bash
IP_BASE="10.10.20"
IP_M01="$IP_BASE.106"
IP_M02="$IP_BASE.117"
IP_M03="$IP_BASE.17"
IP_W01="$IP_BASE.79"
IP_W02="$IP_BASE.37"
IP_W03="$IP_BASE.118"
MY_USER=rocky

# install k3sup
arkade get k3sup

# The first server starts the cluster
k3sup install \
  --cluster \
  --user $MY_USER \
  --ip $IP_M01

# add more servers(master nodes)
for i in {2..3}
k3sup join \
  --server \
  --ip $( eval "echo \$IP_M0$i") \
  --user $MY_USER \
  --server-user $MY_USER \
  --server-ip $IP_M01

# add agents (worker nodes)
for i in {1..3}
do
  echo "add W0$i"
  k3sup join \
    --ip $( eval "echo \$IP_W0$i") \
    --user $MY_USER \
    --server-user $MY_USER \
    --server-ip $IP_M01
done
```


---
# k3d (k3s in docker)
```bash
# create a new cluster
k3d cluster create                       \
  --servers 3                            \
  --agents 3                             \
  --k3s-arg "--disable=traefik@server:*" \
  hello-k3d


# get the kube-config for hte cluster
k3d kubeconfig merge hello-k3d --output ~/.kube/config

# TODO; check the IP in the config, if neccessary chenge to 127.0.0.1

# list nodes
kubectl get nodes -owide


# delete the cluste
k3d cluster delete hello-k3d
```
---
# Code Ready Containers ( openshift / okd )
## TODO: 4 vCPUs, 16 Gb RAM

```bash


# install
SRC_URL=https://github.com/minishift/minishift/releases/download/v1.32.0/minishift-1.32.0-linux-amd64.tgz
curl -sS -L -o- $SRC_URL | tar -xzvf - */minishift --strip-components 1
mv minishift /usr/local/bin/

# download
SRC_URL=https://developers.redhat.com/content-gateway/rest/mirror/pub/openshift-v4/clients/crc/latest/crc-linux-amd64.tar.xz
curl -sS -L -o- $SRC_URL | tar -xJvf - */crc --strip-components 1
mkdir -p ~/bin
mv crc ~/bin/


export PATH=$PATH:$HOME/bin
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bash_profile


# as root
MY_USER_NAME=testuser
echo "$MY_USER_NAME ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/$MY_USER_NAME


crc setup

crc start [--pull-secret-file FILE]

# pull secret 
https://console.redhat.com/openshift/create/local

# The server is accessible via web console at:
#   https://console-openshift-console.apps-crc.testing
# Log in as administrator:
#   Username: kubeadmin
#   Password: PerYD-xPIko-zbfmU-mtWVV
                                                                                                                                                                                                                                                                            
# Log in as user:
#   Username: developer
#   Password: developer


crc oc-env
oc login -u [USER] -p [PASSWORD] https://api.crc.testing:6443

crc stop

```
---
