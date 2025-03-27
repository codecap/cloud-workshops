---
title:       Kubernetes Cluster Management
description: Workshop and Tasks
author:      Vladislav Nazarenko (vnazarenko@📯socket.de)
keywords:    Kubernetes,Cluster,Management
url:         
image:
transition: cover
theme: default
backgroundImage: url(https://codecap.github.io/cloud-workshops/assets/background.jpg)
paginate: true
---

# Kubernetes Cluster Management
![bg left:40% 80%](https://upload.wikimedia.org/wikipedia/commons/3/39/Kubernetes_logo_without_workmark.svg)

- https://karmada.io/



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
