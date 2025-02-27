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
* ⚙️ API
* 🔗 Any kind of integrations via API

---
# Kubernetes Architecture
![](https://kubernetes.io/images/docs/components-of-kubernetes.svg)

---
# Hello, k8s 👋
  ```bash
# arkade - Marketplace For Developer Tools
curl -sLS https://get.arkade.dev | sh; mkdir -p  ~/.arkade/bin/; mv arkade ~/.arkade/bin/
echo 'export PATH="~/.arkade/bin/:$PATH"' > ~/.bash_profile; source ~/.bash_profile

# Kind - kubernetes in docker
arkade install kind kubectl

# Kubernetes Cluster
kind create cluster --config - <<EOF
---
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: hello
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
EOF
kind get kubeconfig --name hello > ~/.kube/config
```


---
# Review the cluster
```bash
# get nodes
kubectl get nodes
# get more info about nodes
kubectl get nodes -owide
# get pods (in current namespace)
kubectl get pods
# get pods in all namespaces
kubectl get pods -A
# get more information about pods
kubectl get pods -A -owide
# get services
kubectl get services  -A -owide
# get deployments
kubectl get deployment  -A -owide
```

---
# Hello, yaml 👋

```bash
kubectl  -n kube-system get deploy coredns -oyaml
```

---
# Kubernetes Pod
![bg right:45% 80%](https://kubernetes.io/docs/tutorials/kubernetes-basics/public/images/module_03_pods.svg)
* may consist of multiple containers
* has own network stack
* C(ontainers) share the same IP
* C can reach each other via the IP
* C may share volumes

---
# Kubernetes Service
![bg right:45% 100%](https://media.licdn.com/dms/image/v2/D4D12AQGmXfnZAybeVw/article-cover_image-shrink_720_1280/article-cover_image-shrink_720_1280/0/1674655778535?e=1746057600&v=beta&t=0Lw3N1Hm289KScInrv77t8lH4GoK5oa7fT0p6_iiDfQ)
* Network traffic distribution
* Across pods by selector (labels)
* Type ClusterIP - within cluster
* Type LoadBalancer - from outside

---
# Kubernetes Deployment
![bg right:45% 100%](https://www.researchgate.net/publication/371543145/figure/fig1/AS:11431281167757737@1686745974914/Hierarchical-structure-of-Deployment-ReplicaSet-and-Pod-adapted-from-official.ppm)
* Version Management
* Deployment(how to) Management
* Rollback
* Horizontal Scale
```bash
kubectl -n kube-system get deploy
kubectl -n kube-system get replicaset
kubectl -n kube-system get pods --selector k8s-app=kube-dns -owide
kubectl -n kube-system scale deploy coredns --replicas 4
```
 
---
# Deploy and Expose
![bg right:35% 100%](https://kubernetes.io/docs/tutorials/kubernetes-basics/public/images/module_04_labels.svg)
* Pods - ephemeral, names and ip change
* ReplicaSet - version management
* ReplicaSet creates pods with labels
* ReplicaSet can be scaled up/down
* Deployment can switch active RS
* Service uses selector(labels) to find pods
* Service excludes from pool unhealthy pods
* Service has a static ip
* Service hadles tcp and udp, any port
* Service can have no ip, Cluster IP, External IP
* ``` kubectl -n kube-system get service```

---
# Kubernetes Ingress Controller / Gateway API
![bg right:45% 100%](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*AgWCYOe3yMevVfzT_1EHog.png)
* Ingress works with http/https
* Forwards traffic to service
* Rules by  hostname, path and more

---
# Kubernetes Architecture
![bg right:50% 100%](https://raw.githubusercontent.com/kubernetes/website/e32c776926325ea00b47beb4dff41fc9b4490c6f/static/images/docs/architecture.svg)
\* Proxy - Ingress Controller (reverse proxy for Kubernetes)

---
# Kubernetes Addons
[Install Addons](k8s-addons.html)

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
