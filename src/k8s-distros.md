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
# Overview
Test & Learn:
- [kind](https://kind.sigs.k8s.io/)
- [minikube](https://minikube.sigs.k8s.io/docs/)
- [microk8s](https://microk8s.io/)
- [k3d](https://k3d.io/)

Vanila:
- [k0s](https://k0sproject.io/)
- [k3s](https://k3s.io/)
- [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

OpenShift

---
# Comparison

```
|------------|----------------------------------------------|------------------------------------------------------------------------------|
|            | Pro                                          | Contra                                                                       | 
|------------|----------------------------------------------|------------------------------------------------------------------------------|
|Kind        |✅ Very easy to install                       | 👎 Hard to set up with other distros than Docker                             |
|            |✅ Uses fewer resources                       | 👎 Making manual configuration changes is difficult                          |
|            |✅ Easily accessible using docker commands    |                                                                              |
|            |✅ Containers are considered nodes            |                                                                              |
|------------|----------------------------------------------|------------------------------------------------------------------------------|
|Minikube    |✅ Easy to do manual configurations           | 👎 Only single-node cluster                                                  |
|            |✅ Gives more access to the system            | 👎 Hard to set up                                                            |
|            |✅ The most widely used and the oldest distro | 👎 Uses more resources                                                       | 
|------------|----------------------------------------------|------------------------------------------------------------------------------|
|MicroK8S    |✅ Easy to setup                              | 👎 Can be set up as a multi-node cluster                                     |
|            |✅ Uses fewer resources                       | 👎 Can’t be installed on machines with ARM32 CPUs                            |
|            |✅ Up to date with Kubernetes releases        | 👎 Doesn’t give much access to the system                                    |
|------------|----------------------------------------------|------------------------------------------------------------------------------|
|k3d         | k3s in docker                                |                                                                              |
|            |                                              |                                                                              |
|            | ➡ k3s                                       |  ➡ k3s                                                                      |
|------------|----------------------------------------------|------------------------------------------------------------------------------|
|k3s         |✅ Very easy to setup                         | 👎 Can make unwanted network changes while auto-configuration during setup   |
|            |✅ Uses fewer resources                       | 👎 Manual configuration changes are difficult                                |
|            |✅ Allows multi-node cluster setup            |                                                                              |
|------------|----------------------------------------------|------------------------------------------------------------------------------|
|k0s         |✅ Very easy to setup                         | 👎 lightwieght and lacks of full k8s distribuation                           |
|            |✅ Uses fewer resources                       | 👎 may effort require more efforts to implement specific configuration       |
|            |✅ Allows multi-node cluster setup            | 👎 does not use OS to install dependencies                                   |
|            |✅ One binary                                 |                                                                              |
|            |✅ No host OS dependencies                    |                                                                              |
|------------|----------------------------------------------|------------------------------------------------------------------------------|
|OpenShift   |✅ Standartized                               | 👎 Less flexible due to built-in enterprise features                         |
|            |✅ on-premise, in cloud - same installer      | 👎 Big Footprint                                                             |
|            |✅ Full Featured K8S Environment              | 👎 Licence Costs / OKD Stream                                                |
|            |✅ Enhanced Security and Compliance           | 👎                                                                           |
|            |✅ Automation and Scalability                 | 👎                                                                           |
|------------|----------------------------------------------|------------------------------------------------------------------------------|

```

---
# K8S parts
- 📦 OCI (Container Image and Runtime)
- 🚀 CRI (Container Runtime Interface)
- 🛜 CNI (Container Network Interface)
- 💾 CSI (Container Storage Interface)
- ⚙️ API
- 🔗 Any kind of integrations via API

---
# K8S Architecture
![](https://kubernetes.io/images/docs/components-of-kubernetes.svg)

---
# Test your K8S Cluster
![bg right:40% 50%](https://upload.wikimedia.org/wikipedia/commons/3/39/Kubernetes_logo_without_workmark.svg)
-  [K8S Addons](https://codecap.github.io/cloud-workshops/k8s-addons.html)
-  [Emojivoto](https://codecap.github.io/cloud-workshops/k8s-fundamentals.html#24)

---
# Preparations
```bash
# docker 
dnf -y install dnf-plugins-core
dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
dnf -y install docker-ce docker-ce-cli
systemctl enable --now docker

useradd -m -G docker testuser
echo "testuser ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/testuser
su - testuser

# selinux
sed -r -e "s/^(SELINUX=).*/\1permissive/" -i /etc/selinux/config
reboot

# install arkade
curl -sLS https://get.arkade.dev | sh; mkdir -p  ~/.arkade/bin/; mv arkade ~/.arkade/bin/
echo 'export PATH="~/.arkade/bin/:$PATH"' > ~/.bash_profile; source ~/.bash_profile

arkade get kubectl
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
# kind
![bg right:50% 50%](https://kind.sigs.k8s.io/logo/logo.png)

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

# to delete hello cluster
kind delete cluster --name hello
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
# TODO: to test ingress with metallb do ssh first
minikube ssh
# Browse the catalog of easily installed Kubernetes services:
minikube addons list
# Create a second cluster running an older Kubernetes release:
minikube start -p aged --kubernetes-version=v1.16.1
# Delete all of the minikube clusters:
minikube delete --all
```
![bg right:50% 50%](https://miro.medium.com/v2/resize:fit:400/0*KzqL3xqmXzV5PPjX.png)

---
# microk8s
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
![bg right:50% 50%](https://res.cloudinary.com/canonical/image/fetch/f_auto,q_auto,fl_sanitize,c_fill,w_460,h_231/https://ubuntu.com/wp-content/uploads/305a/microk8s-sticker.png)


---
# k0s
[Zero Friction Kubernetes](https://docs.k0sproject.io/stable/k0sctl-install/)
__Single-node cluster__
```bash
# NOTE: as root
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

# TODO: ip a show kube-bridge
# TODO: set IP_BASE=10.244.0 when installing metallb

```
![bg right:50% 50%](https://docs.k0sproject.io/v0.9.0/img/k0s-logo-full-color.svg)

---
# k3d
[k3s in docker](https://k3d.io/stable/)
```bash
# as root
arkade get k3d

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
![bg right:50% 50%](https://k3d.io/stable/static/img/k3d_logo_black_blue.svg)

---
# OpenShift / OKD
![bg right:20% 50%](https://upload.wikimedia.org/wikipedia/commons/thumb/3/3a/OpenShift-LogoType.svg/1920px-OpenShift-LogoType.svg.png)

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
# HomeTasks
- create a kind cluster, install addons, deploy test application
- create a minikube cluster, install addons, deploy test application
- create a k0s cluster, install addons, deploy test application
- create a k3d cluster, install addons, deploy test application
- create a crc cluster, install addons, deploy test application

---
# Links
