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
* Handful of Apps ➡ Docker
* Starting from Tens of Apps ➡ K8S

---
# Microservices Architecture

Microservices can be deveploed, deployed and scaled independetly.

How many microservices do you have ?

| Company          | Number|
|------------------|-------|
| [Uber](https://www.uber.com/en-DE/blog/up-portable-microservices-ready-for-the-cloud/)     | 4500+ |
| [LinkedIn](https://www.linkedin.com/pulse/microservice-architecture-lakshmi-barathi/) | 4000+ |
| [Ebay](https://dzone.com/articles/microservices-at-ebay-part-2-sharing-modules-acros)     | 1000+ |
| [Netflix](https://www.geeksforgeeks.org/the-story-of-netflix-and-microservices/)  | 1000+ |

---
# Why Kubernetes ?
* Orchestration ➡ Cloud Operating System
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
# Why is it possible?

* Why so many advantages over Docker(Compose/Swarm)?
* The price is the complexity
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
![bg right:45% 50%](https://kind.sigs.k8s.io/logo/logo.png)

  ```bash
# arkade - Marketplace For Developer Tools
curl -sLS https://get.arkade.dev | sh; mkdir -p  ~/.arkade/bin/; mv arkade ~/.arkade/bin/
echo 'export PATH="~/.arkade/bin/:$PATH"' > ~/.bash_profile; source ~/.bash_profile

# Kind - kubernetes in docker
arkade get kind kubectl

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


# fix too many open files
echo "fs.inotify.max_user_watches = 524288" | sudo tee  -a /etc/sysctl.d/99-kind.conf
echo "fs.inotify.max_user_instances = 512"  | sudo tee  -a /etc/sysctl.d/99-kind.conf
sudo sysctl --system


# dnsmasq
sudo dnf install -y dnsmasq
echo "address=/.tst.k8s.mycompany.com/172.18.0.240" | sudo tee  -a  /etc/dnsmasq.d/tst.k8s.mycompany.com.conf 
echo "server=10.40.10.10" | sudo tee  -a  /etc/dnsmasq.d/tst.k8s.mycompany.com.conf 

# TODO: DNS Config
# TODO: diagram dns -> LoadbalacerIP -> Infress-Nginx -> Service
sudo systemctl enable --now dnsmasq
```
[Kind Known Bugs](https://kind.sigs.k8s.io/docs/user/known-issues/#pod-errors-due-to-too-many-open-files)

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
# Nice to meet you, yaml 👋

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

```bash
kunectl --namespace kube-system get pods 
kubectl --namespace kube-system get pods  -oyaml | less
kubectl --namespace kube-system get pods  -owide
kubectl --namespace kube-system get pods  -o custom-columns=ip:.status.podIP,name:.metadata.name
kubectl --namespace kube-system exec -ti [SOMEPOD] -- bash
kubectl --namespace kube-system logs deploy/coredns
```

---
# Kubernetes Service
![bg right:45% 100%](https://media.licdn.com/dms/image/v2/D4D12AQGmXfnZAybeVw/article-cover_image-shrink_720_1280/article-cover_image-shrink_720_1280/0/1674655778535?e=1746057600&v=beta&t=0Lw3N1Hm289KScInrv77t8lH4GoK5oa7fT0p6_iiDfQ)
* Network traffic distribution
* Across pods by selector (labels)
* Type ClusterIP - within cluster
* Type LoadBalancer - from outside

```bash
kubectl get services -A
kubectl -n default get services --show-labels
kubectl get services  -A
kubectl get endpointslices -A --show-labels
```
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
# Networking
![bg right:50% 100%](https://raw.githubusercontent.com/kubernetes/website/e32c776926325ea00b47beb4dff41fc9b4490c6f/content/en/docs/images/kubernetes-cluster-network.svg)
* Nodes
* Pods
* Services

---
# Routing
![bg right:40% 90%](https://miro.medium.com/v2/resize:fit:720/format:webp/1*Ow5A6_zjjwdKkf2Yi1KgYw.png)
* Every kubernetes node is a router

---
# Kubernetes Addons
[Install Addons](k8s-addons.html)

---
# Service Type LoadBalancer
![bg right:45% 100%](https://www.unixarena.com/wp-content/uploads/2022/09/K8s-LoadBalancer-MetalLB-1024x603.jpg)


```bash
kubectl get services -A


NAMESPACE              NAME                                   TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                      AGE
default                kubernetes                             ClusterIP      10.96.0.1       <none>         443/TCP                      5d20h
ingress-nginx          ingress-nginx-controller               LoadBalancer   10.96.239.70    172.18.0.240   80:31635/TCP,443:32731/TCP   4d20h
ingress-nginx          ingress-nginx-controller-admission     ClusterIP      10.96.59.9      <none>         443/TCP                      4d20h
kube-system            kube-dns                               ClusterIP      10.96.0.10      <none>         53/UDP,53/TCP,9153/TCP       5d20h
kube-system            metrics-server                         ClusterIP      10.96.82.4      <none>         443/TCP                      4d16h
kubernetes-dashboard   kubernetes-dashboard-api               ClusterIP      10.96.205.221   <none>         8000/TCP                     4d20h
kubernetes-dashboard   kubernetes-dashboard-auth              ClusterIP      10.96.147.201   <none>         8000/TCP                     4d20h
kubernetes-dashboard   kubernetes-dashboard-kong-proxy        ClusterIP      10.96.104.54    <none>         443/TCP                      4d20h
kubernetes-dashboard   kubernetes-dashboard-metrics-scraper   ClusterIP      10.96.247.38    <none>         8000/TCP                     4d20h
kubernetes-dashboard   kubernetes-dashboard-web               ClusterIP      10.96.139.51    <none>         8000/TCP                     4d20h
metallb-system         metallb-webhook-service                ClusterIP      10.96.125.55    <none>         443/TCP                      5d17h
pvc-test               nginx                                  ClusterIP      10.96.155.88    <none>         80/TCP                       15m
```

---
# Data Persistence
![bg right:50% 50%](https://miro.medium.com/v2/resize:fit:720/format:webp/1*rz0glgDyD1irAp0Cqdd0gQ.png)
```bash
kubectl create namespace pvc-test
kubectl apply -f - <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: nginx
  name: nginx
  namespace: pvc-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: quay.io/jitesoft/nginx:1.27.3
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        resources:
          requests:
            cpu: 100m
      #   volumeMounts:
      #   - mountPath: "/usr/local/nginx/html"
      #     name: pvc-storage
      # volumes:
      # - name: pvc-storage
      #   persistentVolumeClaim:
      #     claimName: pvc-example
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example
  namespace: pvc-test
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: pvc-test
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
  namespace: pvc-test
spec:
  ingressClassName: nginx
  rules:
  - host: nginx.tst.k8s.mycompany.com
    http:
      paths:
      - backend:
          service:
            name: nginx
            port:
              number: 80
        path: /
        pathType: ImplementationSpecific
EOF
```


---

# Configuration Management
- ConfigMaps and Secrets
```bash
# ConfigMaps
kubectl get configmap -n metallb-system metallb-excludel2 -oyaml

# Secret
kubectl get secret -n metallb-system metallb-memberlist -oyaml
kubectl get secret -n metallb-system metallb-memberlist -ojsonpath='{.data.secretkey}'

# How do they used
kubectl get daemonset -n metallb-system  metallb-speaker -oyaml
```
---
# Tutorial: Emojivoto Application
[Deployment Source](https://github.com/BuoyantIO/emojivoto)
![bg right:31% 100%](https://raw.githubusercontent.com/BuoyantIO/emojivoto/refs/heads/main/assets/emojivoto-topology.png)
```bash
# deploy the applicattion
kubectl apply -k github.com/BuoyantIO/emojivoto/kustomize/deployment

# add ingress to access
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name:      emojivoto
  namespace: emojivoto
spec:
  ingressClassName: nginx
  rules:
  - host: emojivoto.tst.k8s.mycompany.com
    http:
      paths:
      - backend:
          service:
            name: web-svc
            port:
              number: 80
        path: /
        pathType: ImplementationSpecific
EOF
```


---
# Tutorial: Guestboook Application
[Docs](https://kubernetes.io/docs/tutorials/stateless-application/guestbook/)

```bash
kubectl -n guestbook apply   -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/application/guestbook/redis-leader-deployment.yaml
kubectl -n guestbook apply   -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/application/guestbook/redis-leader-service.yaml
kubectl -n guestbook apply   -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/application/guestbook/redis-follower-deployment.yaml
kubectl -n guestbook apply   -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/application/guestbook/redis-follower-service.yaml
kubectl -n guestbook apply   -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/application/guestbook/frontend-deployment.yaml
kubectl -n guestbook apply   -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/application/guestbook/frontend-service.yaml

kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name:      guestbook
  namespace: guestbook
spec:
  ingressClassName: nginx
  rules:
  - host: guestbook.tst.k8s.mycompany.com
    http:
      paths:
      - backend:
          service:
            name: frontend
            port:
              number: 80
        path: /
        pathType: ImplementationSpecific
EOF
```

---
# DNS and Service Discovery

```
domain:       .cluster.local

globally:     [SERVICE_NAME].[NAMESPACE].svc.cluster.local
in namespace: [SERVICE_NAME]
```

---
# Authentication
![bg right:45% 30%](https://riteshmblog.wordpress.com/wp-content/uploads/2022/05/kubernetes-secret.jpg)
```bash
# create a new key
openssl genrsa -out user.key 2048

# creqte a CSR
openssl req -new -key user.key -out user.csr -subj "/CN=user/O=team-a/O=team-b"

# create CertificateSigningRequest in K8S
kubectl apply -f - <<EOF
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: user
spec:
  request: $(cat user.csr  | base64 -w0)
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth
EOF
kubectl certificate approve user
kubectl get csr user -o yaml

# extract the certifcate
kubectl get csr user -o jsonpath='{.status.certificate}'  | base64 -d > user.crt
# review certifcate
openssl x509 -text -noout -in user.crt
```
---
# Authorization - RBAC
![bg right:45% 90%](https://media.licdn.com/dms/image/v2/D5612AQGf017zCKs4jg/article-cover_image-shrink_423_752/article-cover_image-shrink_423_752/0/1711521370903?e=1746662400&v=beta&t=mkNjSzHogOwqAm89UVKUgLPoXvIRoQmHJpgc1-_hMrM)

```bash
# review available clusteroles
kubectl get clusterroles

# review rolebinding
kubectl create rolebinding user-binding --clusterrole=edit --user=user    --namespace=[NAMESPACE] --dry-run=client -oyaml
kubectl create rolebinding user-binding --clusterrole=edit --group=team-a --namespace=[NAMESPACE] --dry-run=client -oyaml

# create rolebinding
kubectl create rolebinding user-binding --clusterrole=edit --group=team-a --namespace=[NAMESPACE]

# set user creds
kubectl config set-credentials user --client-key=user.key --client-certificate=user.crt --embed-certs=true

# create a new context
kubectl config set-context user.[NAMESPACE] --cluster=kind-hello --user=user --namespace=[NAMESPACE]
kubectl config get-contexts
kubectl config use-context  user.[NAMESPACE]

# test new context
kubesystem get pods --namespace [NAMESPACE]
```  
Predefined cluster roles:
* cluster-admin
* admin
* edit
* view

---
# Authorization - RBAC

Roles/ClusterRoles can be assigned to
* user
* groups
* serviceaccounts

---
# ReplicatonController, DaemonSet, StatefulSet

```bash
# ReplicaSet
kubectl get rs -n emojivoto -owide                # review rs with labels
kubectl get pods -A -l  app=web-svc -owide -owide # get pods by labels

# DaemonSet
kubectl get ds -owide -A                          # review ds with labels
kubectl get pods -A -l  app=kindnet -owide -owide # get pods by labels

# StatefulSet
kubectl get statefulset -owide -A                 # review StatefulSet wit labelss
# ...
```


---
# Tasks
- install kubectl, helm, kind
- create a kind k8s cluster
- install addons
- [deploy/change/scale/delete a application](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/)
- [Run a Single-Instance Stateful Application](https://kubernetes.io/docs/tasks/run-application/run-single-instance-stateful-application/)
- create a Service of Type LoadBalancer
- list pods behind a service
- create an ingress to access an web-based application

---
# Links
- [Kubernetes Docs](https://kubernetes.io/docs/home/)
- [RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Certificate Signing Requests](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/)
- [arkade](https://github.com/alexellis/arkade)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
- [helm](https://helm.sh/)
- [kind](https://kind.sigs.k8s.io/)

---
# Configure Firefox to access Nginx Ingress
[Tunnel with putty](https://www.forbesconrad.com/blog/putty-port-forward-settings-for-socks-5-proxy/)

Then socks5 proxy with firefox
```
Firefox -> settings -> Proxy -> Manual proxy configuration

SOCKSv5: ✔
host:    localhost
port:    1080
```
