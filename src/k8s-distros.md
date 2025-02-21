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
|K3S         |✅ Very easy to setup                         | 👎 Can make unwanted network changes while auto-configuration during setup   |
|            |✅ Uses fewer resources                       | 👎 Manual configuration changes are difficult                                |
|            |✅ Allows multi-node cluster setup            |                                                                              |
|------------|----------------------------------------------|------------------------------------------------------------------------------|
```


---
# K8S parts
* Container Runtime Interface (CRI)
* Container Network Interface (CNI)
* Contaienr Storage Interface (CSI)
* K8S (api, scheduler, controller-manager, kubelet, kube-proxy)
* DNS
* Gateway / Ingress

---
# K8S Architecture
## TODO: Diagram

---
# Simple Deployment

## TODO: Diagram
## we need a simple deployment, which can be done by means of docker(compose) / podman play

---
# kind

---

# minikube

---

# microk8s

---

# k0s

---

# k3s

---
# k3d

---

# openshift

---
