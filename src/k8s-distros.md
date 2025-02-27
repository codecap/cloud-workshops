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

- kind
- minikube
- microk8s
- [k0s](https://k0sproject.io/)
- k3s
- k3d
- openshift



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
* DNS
* Gateway / Ingress
* [Installing Addons on Kubernetes](https://kubernetes.io/docs/concepts/cluster-administration/addons/)

---
# Simple Deployment

## TODO: Diagram
## we need a simple deployment, which can be done by means of docker(compose) / podman play

---
# Arkade
```bash
# install arkade
curl -sLS https://get.arkade.dev | sh
mkdir -p  ~/.arkade/bin/
mv arkade ~/.arkade/bin/

echo 'export PATH="~/.arkade/bin/:$PATH"' > ~/.profile
source ~/.profile
arkade get kubectl
```

---


# Prepare VMs
```bash

IP_BASE="10.10.10"
IP_M01="$IP_BASE.103"
IP_M02="$IP_BASE.243"
IP_M03="$IP_BASE.217"
IP_W01="$IP_BASE.150"
IP_W02="$IP_BASE.119"
IP_W03="$IP_BASE.114"

cat > ~/.ssh/id_rsa <<EOF
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----

EOF
chmod 0600 ~/.ssh/id_rsa


```
---
# kind

---

# minikube

---

# microk8s

---

# k0s
```bash
arkade get k0sctl

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
      address: $IP_M01
      user: ubuntu
      keyPath: ~/.ssh/id_rsa
  - role: controller
    ssh:
      address: $IP_M02
      user: ubuntu
      keyPath: ~/.ssh/id_rsa
  - role: controller
    ssh:
      address: $IP_M03
      user: ubuntu
      keyPath: ~/.ssh/id_rsa
  - role: worker
    ssh:
      address: $IP_W01
      user: ubuntu
      keyPath: ~/.ssh/id_rsa
  - role: worker
    ssh:
      address: $IP_W02
      user: ubuntu
      keyPath: ~/.ssh/id_rsa
  - role: worker
    ssh:
      address: $IP_W03
      user: ubuntu
      keyPath: ~/.ssh/id_rsa
EOF



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
```



---

# k3s

```bash

IP_BASE="10.10.10"
IP_M01="$IP_BASE.103"
IP_M02="$IP_BASE.243"
IP_M03="$IP_BASE.217"
IP_W01="$IP_BASE.150"
IP_W02="$IP_BASE.119"
IP_W03="$IP_BASE.144"

USER=ubuntu


# The first server starts the cluster
k3sup install \
  --cluster \
  --user $USER \
  --ip $IP_M01

# add more servers(master nodes)
for i in {2..3}
k3sup join \
  --server \
  --ip $( eval "echo \$IP_M0$i") \
  --user $USER \
  --server-user $USER \
  --server-ip $IP_M01

# add agents (worker nodes)
for i in {1..3}
do
  echo "add W0$i"
  k3sup join \
    --ip $( eval "echo \$IP_W0$i") \
    --user $USER \
    --server-user $USER \
    --server-ip $IP_M01
done
```


---
# k3d

---

# openshift

---
