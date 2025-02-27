---
title:       Kubernetes Fundamentals
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

# Kubernetes Fundamentals
![bg left:40% 80%](https://upload.wikimedia.org/wikipedia/commons/3/39/Kubernetes_logo_without_workmark.svg)

---
# Why Kubernetes ?
* Handful of Apps -> Docker
* Starting Tens   -> K8S

---
# Why Kubernetes ?
* Orchestration -> Cloud Operating System
* Deployment
* Access 
* Networking
* Operations
* Data Persistancy
* Scaling
* Observability
* Extendible
* Integrations (Cloud, DNS, Certificates, Auth, Secrets)
---
# Why it's possibie?

* Why so many advantages over Docker(Swarm)?
* The price is complexity
* K8S - is extandible API

---
# What is API?
![bg right:50% 100%](https://images.tutorialedge.net/uploads/rest-api.png)
Application Programming Interface is a set of tools and protocols that allows different software applications to communicate with each other.

[API Examples](https://katalon.com/resources-center/blog/api-examples)

---
# What is Kubernetes?
* 📦 OCI (Container Image and Runtime)
* 🚀 CRI (Container Runtime Interface)
* 🛜 CNI (Container Network Interface)
* 💾 CSI (Container Storage Interface)
* ⚙️ API ❗❗❗
* 🔗 Any kind of integrations via API

---
# Nice to meet you, k8s 👋
[How to install docker]()
[How to install arkade]()


```bash
# Kind - kubernetes in docker
arkade install kind kubectl
```

---
# Kubernetes Architecture
![](https://kubernetes.io/images/docs/components-of-kubernetes.svg)

---
# Kubernetes Architecture
![bg right:50% 100%](https://raw.githubusercontent.com/kubernetes/website/e32c776926325ea00b47beb4dff41fc9b4490c6f/static/images/docs/architecture.svg)

---
# Review
* apiserver
* scheduler
...

in kind

show pods

---
# Kubernetes Pod
![bg right:45% 80%](https://kubernetes.io/docs/tutorials/kubernetes-basics/public/images/module_03_pods.svg)
* may consist of multiple containers
* has own network stack
* C(ontainers) share the same IP
* C can reach each other via the IP
* C may share volumes

---
# Networking
![bg right:50% 100%](https://raw.githubusercontent.com/kubernetes/website/e32c776926325ea00b47beb4dff41fc9b4490c6f/content/en/docs/images/kubernetes-cluster-network.svg)
* Nodes
* Pods
* Services

---
# Routing
![bg right:50% 100%](https://miro.medium.com/v2/resize:fit:720/format:webp/1*Ow5A6_zjjwdKkf2Yi1KgYw.png)

---
# servie types: ClusterIP, LoadBalancer

---
# Access: HTTP
## Ingress Controller
![](https://kubernetes.io/docs/images/ingress.svg)

---

# Access: HTTP
## GatewayAPI
![](https://kubernetes.io/docs/images/gateway-request-flow.svg)

---
# Access: ServiceType LoadBalancer
![](https://kubernetes.io/docs/images/ingress.svg)

---


# data persistancy: PV, PVC, StorageClasses

---

# Configuration Management
- ConfigMaps and Secrets

---

# ReplicatonController, DaemonSet, StatefulSet

---
# DNS and Service Discovery

---
# Athentication

---
# Authorization - RBAC

---
# -> k8s-addons

---
# -> typical deployment(articlerating): namespace, deploy, service, labels, selectors, ingress

---


# Tasks
* create a kind k8s cluster
* install addons
* deploy a hello-world app
* deploy a staful app
* list pods behind a service
---
