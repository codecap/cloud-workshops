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


---
# Prerequisites
- Client Node
- 3 K8S Master Nodes
- 3 K8S Worker Nodes

*Sizing (at least)*:
- 2 CPU
- 4 GB RAM
- 40 GB Disk

---
# Preparations
![bg right:50% 50%](https://image.pngaaa.com/935/5527935-middle.png)
```bash
# on every node as root
MY_USER=deploy
useradd -m $MY_USER
echo "$MY_USER ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/$MY_USER
timedatectl set-timezone UTC

# on the client node, login as regular user
su - $MY_USER

# on client node
ssh-keygen

#  on clien node
# the output of the following command should be executed on every node
MY_USER=deploy
cat <<EOF
#
# please execute on every node as regular user
#
mkdir -p ~$MY_USER/.ssh; chmod 700 ~$MY_USER/.ssh;
echo "$(cat ~/.ssh/id_rsa.pub)" >> ~$MY_USER/.ssh/authorized_keys; chmod 600 ~$MY_USER/.ssh/authorized_keys
chown -R $MY_USER. ~$MY_USER/.ssh
EOF

# install arkade on client node as regular user
curl -sLS https://get.arkade.dev | sh; mkdir -p  ~/.arkade/bin/; mv arkade ~/.arkade/bin/
echo 'export PATH="~/.arkade/bin/:$PATH"' >> ~/.bash_profile; source ~/.bash_profile
```

---
# Preparations
![bg right:50% 50%](https://image.pngaaa.com/935/5527935-middle.png)
- Install and configure dnsmasq on the client node as desribed [here](https://codecap.github.io/cloud-workshops/k8s-addons.html#3)
- Set Variables
```bash
# on the client node
IP_BASE="10.11.30"
IP_START_FROM="100"
IP_M01="$IP_BASE.$((IP_START_FROM + 0))"
IP_M02="$IP_BASE.$((IP_START_FROM + 1))"
IP_M03="$IP_BASE.$((IP_START_FROM + 2))"
IP_W01="$IP_BASE.$((IP_START_FROM + 3))"
IP_W02="$IP_BASE.$((IP_START_FROM + 4))"
IP_W03="$IP_BASE.$((IP_START_FROM + 5))"
MY_USER=deploy
```
---
# Preparations
![bg right:50% 50%](https://image.pngaaa.com/935/5527935-middle.png)
__Some tools for your comfort__
```bash
# on client node as regular user
sudo dnf install 'dnf-command(config-manager)'
sudo dnf config-manager --add-repo https://kubecolor.github.io/packages/rpm/kubecolor.repo
sudo dnf install -y kubecolor bash-completion

arkade get kubectl
sudo ln -s ~/.arkade/bin/kubectl /usr/local/bin/
kubectl completion bash | sudo tee -a /etc/profile.d/kubectl-completion.sh
source /etc/profile
mkidr ~/bin
ln -s ~/.arkade/bin/kubectl ~/bin/kubectl
```

```bash
# on client node
echo 'alias k="kubecolor"'                      >> ~/.bash_profile
echo 'complete -o default -F __start_kubectl k' >> ~/.bash_profile
source ~/.bash_profile
```

---
# Preparations
![bg right:50% 50%](https://image.pngaaa.com/935/5527935-middle.png)
__[krew](https://krew.sigs.k8s.io/docs/user-guide/setup/install/) - get plugins for kubectl__
[Plugins available](https://krew.sigs.k8s.io/plugins/)

```bash
# on client node
sudo dnf install -y git
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
echo 'export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"' >> ~/.bash_profile
source ~/.bash_profile
```

```bash
# on client node
#
# Install krew plugins
#
# on client node
for p in  ctx ns node-shell
do
  kubectl krew install $p
done

# Install fzf for ctx and ns
curl  https://raw.githubusercontent.com/junegunn/fzf/master/install | bash
source ~/.bashrc

# Install kube-ps1
mkdir -p ~/bin
curl -o ~/bin/kube-ps1.sh https://raw.githubusercontent.com/jonmosco/kube-ps1/master/kube-ps1.sh
echo ""                                   >> ~/.bash_profile
echo "source ~/bin/kube-ps1.sh"           >> ~/.bash_profile
echo "PS1='[\u@\h \W \$(kube_ps1)]\\\$ '" >> ~/.bash_profile
```



---
# kubeadm
![bg right:50% 50%](https://kubernetes.io/images/kubeadm-stacked-color.png)

[Link](https://phoenixnap.com/kb/install-kubernetes-on-rocky-linux)

__Prepare Installation__
```bash
# on each cluster node as root
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

setenforce 0
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/sysconfig/selinux

systemctl disable --now  firewalld 
```
__Install Containerd__
```bash
# on each cluster node as root
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
dnf install -y containerd.io
mv /etc/containerd/config.toml /etc/containerd/config.toml.bak
containerd config default > /etc/containerd/config.toml
sed -e "s/SystemdCgroup = .*/SystemdCgroup = true/" -i /etc/containerd/config.toml
systemctl enable --now containerd.service
```

---
# kubeadm
![bg right:50% 50%](https://kubernetes.io/images/kubeadm-stacked-color.png)
__Install and prepare kubeadm__
```bash
# on each cluster node as root
K8S_VERSION="1.32"
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v$K8S_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v$K8S_VERSION/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

kubeadm config images pull --kubernetes-version stable-$K8S_VERSION
kubeadm config images list
```
---
# kubeadm
![bg right:50% 50%](https://kubernetes.io/images/kubeadm-stacked-color.png)
__Install K8S__
```bash
# on master01
MY_NIC=ens192
MY_IP=$(ip a show $MY_NIC | grep "inet " |  awk '{print $2}' | awk -F/ '{print $1}')
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
kubectl get nodes -A

# CNI
curl -sS https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/calico.yaml \
  | sed -r -e "s/docker.io/quay.io/" \
  | kubectl apply -f -


# join master nodes
kubeadm join $MASTER01_IP:6443 --token    [TOKEN] \
        --discovery-token-ca-cert-hash    [HASH]  \
        --control-plane --certificate-key [KEY]]

# join worker nodes
kubeadm $MASTER01_IP:6443 --token     [TOKEN] \
        --discovery-token-ca-cert-hash    [HASH]

# list nodes
kubectl get nodes  -owide
```
---
# kubeadm
![bg right:50% 50%](https://kubernetes.io/images/kubeadm-stacked-color.png)

__Copy $KUBECONFIG__
```bash
# on cliet node as regular user
mkdir -p ~/.kube
scp $IP_M01:/etc/kubernetes/admin.conf  ~/.kube/config
```

__Reset__
```bash
# on each cluster node as root
kubeadm reset --force
rm -rf  /etc/kubernetes/manifests/*
rm -rf  /etc/cni/net.d
systemctl disable --now kubelet

dnf remove -y containerd.io kubelet kubeadm kubectl
reboot
```
---
# k0s
![bg right:50% 50%](https://docs.k0sproject.io/stable/img/k0s-logo-2025-horizontal.svg#only-light)
```bash
# on the client node
arkade get k0sctl

# variables were set in the preparations step
cat > ~/k0sctl.yaml <<EOF
---
apiVersion: k0sctl.k0sproject.io/v1beta1
kind: Cluster
metadata:
  name: k0s-cluster
spec:
  hosts:
  - role: controller
    ssh: { address: $IP_M01,  user: $MY_USER,  keyPath: ~/.ssh/id_rsa }
  - role: controller
    ssh: { address: $IP_M02,  user: $MY_USER,  keyPath: ~/.ssh/id_rsa }
  - role: controller
    ssh: { address: $IP_M03,  user: $MY_USER,  keyPath: ~/.ssh/id_rsa }
  - role: worker
    ssh: { address: $IP_W01,  user: $MY_USER,  keyPath: ~/.ssh/id_rsa }
  - role: worker
    ssh: { address: $IP_W02,  user: $MY_USER,  keyPath: ~/.ssh/id_rsa }
  - role: worker
    ssh: { address: $IP_W03,  user: $MY_USER,  keyPath: ~/.ssh/id_rsa }
EOF
```

---
# k0s
![bg right:50% 50%](https://docs.k0sproject.io/stable/img/k0s-logo-2025-horizontal.svg#only-light)
```bash
# on the client node
k0sctl apply --config ~/k0sctl.yaml

mkdir -p ~/.kube
k0sctl kubeconfig > ~/.kube/config

# review on  master nodes
systemctl status k0scontroller.service
cat /etc/systemd/system/k0scontroller.service

# review on worker nodes
systemctl status k0sworker.service
cat  /etc/systemd/system/k0sworker.service

# on worker if ctr installed
ctr --address /run/k0s/containerd.sock -n k8s.io c ls

# reset
k0sctl reset --config ~/k0sctl.yaml
```

---
# K3S
![bg right:60% 50%](https://k3s.io/img/k3s-logo-light.svg)
```bash
# on client node
arkade get k3sup

# The first server starts the cluster
k3sup install \
  --cluster \
  --user $MY_USER \
  --ip $IP_M01

# add more servers(master nodes)
for i in {2..3}
do
  k3sup join \
    --server \
    --ip $( eval "echo \$IP_M0$i") \
    --user $MY_USER \
    --server-user $MY_USER \
    --server-ip $IP_M01
done

# add agents (worker nodes)
for i in {1..3}
do
  k3sup join \
    --ip $( eval "echo \$IP_W0$i") \
    --user $MY_USER \
    --server-user $MY_USER \
    --server-ip $IP_M01
done

mkdir  -p ~/.kube
mv ~/kubeconfig ~/.kube/config

# Reset
# on master nodes
/usr/local/bin/k3s-uninstall.sh
# on worker nodes
/usr/local/bin/k3s-agent-uninstall.sh

```

---
# Operators
![bg right:50% 60%](https://miro.medium.com/v2/format:webp/1*IsSys2dTeU7cgWPXlpSMGA.png)
Software extensions to Kubernetes to manage applications and their components.

---
# Operators
![bg right:50% 75%](https://pushbuildtestdeploy.com/images/k8s-operators.png)

* An extension of the Kubernetes API -  a way to interact with the cluster, with YAML files(CRDs).
* A controller is needed to implement API calls(CRDs) and perform actions - create a Deployment, StatefulSet, run a job... __Its task is to maintain a state__.

---
# Operators

* Check an example operator on [operatorhub.io](https://operatorhub.io)
* Check Capability Level
* Check GitHub Repo ond Maintainers
* Check Custom Resource Definitions

---
# Custom Resource Definitions (CRDs)
![bg right:50% 75%](https://raw.githubusercontent.com/kubernetes/community/refs/heads/master/icons/svg/resources/unlabeled/crd.svg)


---
# CRDs

![bg right:50% 75%](https://www.cncf.io/wp-content/uploads/2022/07/k8s-operator.webp)

[Extend the Kubernetes API with CustomResourceDefinitions](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)

```bash
apiVersion: k8s.mariadb.com/v1alpha1
kind: MariaDB
metadata:
  name: mariadb
spec:
  rootPasswordSecretKeyRef:
    name: mariadb
    key: password
  database: mariadb
  port: 3306
  storage:
    size: 1Gi
  metrics:
    enabled: true
```

---
# OLM
Operator Lifecycle Manager
[Docs](https://olm.operatorframework.io/docs/getting-started/)
![bg right:50% 75%](https://olm.operatorframework.io/images/logo.svg)
```bash
# on client node
arkade get operator-sdk
operator-sdk olm install
# check

kubectl -n olm get all
kubectl -n olm get ClusterServiceVersion
kubectl -n olm get catalogsource
kubectl -n operators get subscriptions
kubectl -n operators get installplans
kubectl -n operators get clusterserviceversions

# list of available operators
kubectl -n olm get packagemanifest 

```
---
# Monitoring
[Prometheus Crash Course](https://www.youtube.com/watch?v=BEBsuA5tgUU)
![bg right:50% 80%](https://miro.medium.com/v2/resize:fit:4800/format:webp/1*7thrW4Wa5y6b03PxtPlQzA.jpeg)

---
# Monitoring
[Prometheus Operator](https://prometheus-operator.dev/docs/getting-started/introduction/)
![bg right:50% 50%](https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/refs/heads/main/Documentation/logos/prometheus-operator-logo.svg)
```bash
kubectl create ns monitoring

# Prom
kubectl -n olm get packagemanifest  | grep prometheus
kubectl -n olm get packagemanifest  prometheus -oyaml

kubectl apply -f - <<EOF
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: prometheus
  namespace: operators
spec:
  channel: beta
  installPlanApproval: Automatic
  name: prometheus
  source: operatorhubio-catalog
  sourceNamespace: olm
EOF
```
---
# Monitoring
![bg right:50% 90%](https://picluster.ricsanfre.com/assets/img/prometheus-stack-architecture.png )
__Try to create a prometheus instance__
```bash
kubectl apply -f - <<EOF
---
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name:      prom-a
  namespace: monitoring
spec: {}
EOF

# check
kubectl -n operators  get subscription
kubectl -n operators  get installplans
kubectl -n operators  get clusterserviceversions
kubectl -n monitoring get prometheus prom-a
kubectl -n monitoring get pods
# remove
kubectl -n monitoring delete prometheus prom-a
```

---
# Monitoring
__Install and configure__
![bg right:50% 50%](https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/refs/heads/main/Documentation/logos/prometheus-operator-logo.svg)
```bash
#
# This repository collects Kubernetes manifests, Grafana dashboards, 
# and Prometheus rules combined with documentation and scripts to provide
# easy to operate end-to-end Kubernetes cluster monitoring with Prometheus
# using the Prometheus Operator.
#
git clone https://github.com/prometheus-operator/kube-prometheus.git
cd kube-prometheus/
git checkout remotes/origin/release-0.13

#
# Apply all manifests in the directory
#
kubectl apply -f manifests/

# operator is already installed by OLM, so remove this instance
for i in manifests/prometheusOperator-*; do kubectl delete -f $i; done

#
# There are default network policies, which deny access from outside
# For ease of this presentation we just remove them
#
for f in  manifests/{alertmanager,grafana,prometheus}-networkPolicy.yaml
do
  kubectl delete -f $f
done

```
---
# Monitoring
![bg right:50% 50%](https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/refs/heads/main/Documentation/logos/prometheus-operator-logo.svg)
```bash
kubectl apply -f - <<EOF
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name:      grafana
  namespace: monitoring
spec:
  ingressClassName: traefik
  rules:
  - host: grafana.tst.k8s.mycompany.com
    http:
      paths:
      - backend:
          service:
            name: grafana
            port:
              name: http
        path: /
        pathType: ImplementationSpecific
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name:      prometheus
  namespace: monitoring
spec:
  ingressClassName: traefik
  rules:
  - host: prometheus.tst.k8s.mycompany.com
    http:
      paths:
      - backend:
          service:
            name: prometheus-k8s
            port:
              number: 9090
        path: /
        pathType: ImplementationSpecific
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name:      alertmanager
  namespace: monitoring
spec:
  ingressClassName: traefik
  rules:
  - host: alertmanager.tst.k8s.mycompany.com
    http:
      paths:
      - backend:
          service:
            name: alertmanager-main
            port:
              number: 9093
        path: 
        pathType: ImplementationSpecific
EOF
```
---
# Export Monitoring Data
![bg right:40% 90%](https://miro.medium.com/v2/resize:fit:720/format:webp/1*H3nzCLJvta-Qj1CXimwOkA.png)
[Remote Write](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#remote_write)
With remote write, data is pushed out from Prometheus to other systems.

---
# Export Monitoring Data
[Federation](https://prometheus.io/docs/prometheus/latest/federation/)
![bg right:40% 80%](https://last9.ghost.io/content/images/size/w1000/2024/01/Prometheus_Federation_Federated_Architecture.jpg)
With federation, a Prometheus server pulls data from other Prometheus servers.

---
# Export Monitoring Data
![bg right:40% 30%](https://upload.wikimedia.org/wikipedia/commons/3/38/Prometheus_software_logo.svg)

__Pull data from in-cluster into your global prometheus instance__
```bash
# on client node
cat <<"EOF" | sudo tee /etc/yum.repos.d/prometheus.repo 
[prometheus]
name=prometheus
baseurl=https://packagecloud.io/prometheus-rpm/release/el/$releasever/$basearch
repo_gpgcheck=1
enabled=1
gpgkey=https://packagecloud.io/prometheus-rpm/release/gpgkey
       https://raw.githubusercontent.com/lest/prometheus-rpm/master/RPM-GPG-KEY-prometheus-rpm
gpgcheck=0
metadata_expire=300
EOF
sudo dnf -y  install prometheus2

# list of jobs
curl -sS prometheus.tst.k8s.mycompany.com/api/v1/status/config \
  | jq -Mr .data.yaml \
  | python3 -c 'import json, sys, yaml ; y = yaml.safe_load(sys.stdin.read()) ; print(json.dumps(y))' \
  | jq -Mr .scrape_configs[].job_name

# test your selector
curl -D- -G --data-urlencode 'match[]={job=~".+"}' prometheus.tst.k8s.mycompany.com/federate
curl -D- -G --data-urlencode 'match[]={job=~".+", __name__="kube_deployment_created" }' prometheus.tst.k8s.mycompany.com/federate
curl -D- -G --data-urlencode 'match[]={job=~".+", __name__="kube_deployment_created", namespace="monitoring" }' prometheus.tst.k8s.mycompany.com/federate
```

---
# Export Monitoring Data
![bg right:40% 30%](https://upload.wikimedia.org/wikipedia/commons/3/38/Prometheus_software_logo.svg) 
__Pull data from in-cluster to your global prometheus instance__
```bash
# on client node
cat <<EOF | sudo tee /etc/prometheus/prometheus.yml 
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: 'global-view'
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]':
          - '{job=~".+"}' # Collect all
    static_configs:
      - targets:
        - 'prometheus.tst.k8s.mycompany.com'
EOF
sudo systemctl enable --now  prometheus
# test by visiting $MY_IP:9090
```

---
# Logging
ElasticSearch + Logstash + Kibana = ELK Stack
![bg right:50% 70%](https://elastic-stack.readthedocs.io/en/latest/_images/elk_overview.png)

[What is ELK Stack](https://www.youtube.com/watch?v=jT-y6oS10jk)

---
# Container Logs
![bg right:50% 70%](https://media2.dev.to/dynamic/image/width=800%2Cheight=%2Cfit=scale-down%2Cgravity=auto%2Cformat=auto/https%3A%2F%2Fdev-to-uploads.s3.amazonaws.com%2Fuploads%2Farticles%2Fks09mcjba5dt4tik7ie7.png)
```bash
# log in to a cluster node
kubectl get nodes
kubectl node-shell [NODE_NAME]
# on the node review log dir
ls -lh /var/log/containers
```

---
# Logging Operator
![bg right:50% 70%](https://github.com/OT-CONTAINER-KIT/logging-operator/raw/master/static/logging-operator-arc.png)
```bash
kubectl apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: my-logging-operator
  namespace: operators
spec:
  channel: beta
  name: logging-operator
  source: operatorhubio-catalog
  sourceNamespace: olm
EOF

# check
kubectl -n operators get subscription
kubectl -n operators get installplans
kubectl -n operators get clusterserviceversions
```

---
# Logging - ElasticSearch
![bg right:45% 90%](https://github.com/OT-CONTAINER-KIT/logging-operator/blob/master/static/es-architecture.png?raw=true)
```bash
kubectl create ns logging

kubectl apply -f - <<EOF
---
apiVersion: logging.logging.opstreelabs.in/v1beta1
kind: Elasticsearch
metadata:
  name:      elasticsearch
  namespace: logging
spec:
  esClusterName: prod
  esVersion:     7.17.27
EOF
```
---
# Logging - Kibana
![bg right:50% 70%](https://github.com/OT-CONTAINER-KIT/logging-operator/blob/master/static/kibana-architecture.png?raw=true)
```bash
kubectl apply -f - <<EOF
---
apiVersion: logging.logging.opstreelabs.in/v1beta1
kind: Kibana
metadata:
  name:      kibana
  namespace: logging
spec:
  replicas: 1
  esCluster:
    host:        http://elasticsearch-master:9200
    esVersion:   7.17.27
    clusterName: elasticsearch
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name:      kibana
  namespace: logging
spec:
  ingressClassName: traefik
  rules:
  - host: kibana.tst.k8s.mycompany.com
    http:
      paths:
      - backend:
          service:
            name: kibana
            port:
              name: http
        path: /
        pathType: ImplementationSpecific
EOF
```
---
# Logging - Fluentd
![bg right:50% 70%](https://github.com/OT-CONTAINER-KIT/logging-operator/blob/master/static/kibana-architecture.png?raw=true) 

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: logging
  labels:
    app: fluentd
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluentd
  labels:
    app: fluentd
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - namespaces
  verbs:
  - get
  - list
  - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd
roleRef:
  kind: ClusterRole
  name: fluentd
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: fluentd
  namespace: logging
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccount: fluentd
      serviceAccountName: fluentd
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.16-debian-elasticsearch7-1
        env:
          - name:  FLUENT_ELASTICSEARCH_HOST
            value: "elasticsearch-master"
          - name:  FLUENT_ELASTICSEARCH_PORT
            value: "9200"
          - name: FLUENT_ELASTICSEARCH_SCHEME
            value: "http"
          - name: FLUENTD_SYSTEMD_CONF
            value: disable
          - name: FLUENT_CONTAINER_TAIL_EXCLUDE_PATH
            value: /var/log/containers/fluent*
          - name: FLUENT_CONTAINER_TAIL_PARSER_TYPE
            value: /^(?<time>.+) (?<stream>stdout|stderr)( (?<logtag>.))? (?<log>.*)$/
        resources:
          limits:
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
EOF
```
---
# Review Logs in Kibana
![bg right:35% 90%](https://assets.digitalocean.com/articles/kubernetes_efk/kibana_logs.png) 
http://kibana.tst.k8s.mycompany.com

Discover ➡ Create Index Pattern ➡ logstash-*

---
# Logging - Export Data
__Cross Cluster Replication__
![bg right:45% 70%](https://www.elastic.co/docs/deploy-manage/images/elasticsearch-reference-ccr-arch-central-reporting.png) 

---
# Logging - Export Data
__Reindexing with Logstash__
![bg right:45% 70%](https://us1.discourse-cdn.com/flex019/uploads/mauve_hedgehog/original/2X/d/da3a38f00a0b837a2c576b2faa213b223b09e0f9.png)

[Logstash Input Plugins](https://www.elastic.co/docs/reference/logstash/plugins/input-plugins)
[Logstash Output Plugins](https://www.elastic.co/docs/reference/logstash/plugins/output-plugins)
[Logstash with OpenSearch(in/out)](https://opensearch.org/blog/introducing-logstash-input-opensearch-plugin-for-opensearch/)
[Logstash OpenSearch Input Plugin](https://github.com/opensearch-project/logstash-input-opensearch)
[Logstash OpenSeatch Output Plugin](https://github.com/opensearch-project/logstash-output-opensearch)

---
# Logging - how to access source
```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name:      elasticsearch-master
  namespace: logging
spec:
  ingressClassName: traefik
  rules:
  - host: elasticsearch-master.tst.k8s.mycompany.com
    http:
      paths:
      - backend:
          service:
            name: elasticsearch-master
            port:
              name: http
        path: /
        pathType: ImplementationSpecific
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: elasticsearch-master
    role: master
  name: elasticsearch-master-transport
  namespace: logging
spec:
  ports:
    - name: http
      port: 9200
      protocol: TCP
      targetPort: 9200
    - name: transport
      port: 9300
      protocol: TCP
      targetPort: 9300
  selector:
    app: elasticsearch-master
    role: master
  type: LoadBalancer
```


---
# Cert manager
![bg right:45% 40%](https://raw.githubusercontent.com/cert-manager/cert-manager/d53c0b9270f8cd90d908460d69502694e1838f5f/logo/logo-small.png)

[Cert Manager - Intro (first 4 min)](https://www.youtube.com/watch?v=Xv1bdeVnGGY)
[Cert Manager Issuer](https://cert-manager.io/docs/configuration/acme/#dns-zones)
[Let's Encrypt - Challenge Types](https://letsencrypt.org/docs/challenge-types/)

```bash
kubectl apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: my-cert-manager
  namespace: operators
spec:
  channel: stable
  name: cert-manager
  source: operatorhubio-catalog
  sourceNamespace: olm
EOF
```

---
# Cert manager - Conf
![bg right:45% 80%](https://miro.medium.com/v2/resize:fit:720/format:webp/1*x_eHjJCn8euTNF6hFTuVZA.png)

```bash
# We have to patch ClusterServiceVersion to activate DNS01 challange type
CERT_MANAGERR_CURRENT_CSV=$(kubectl get subscriptions.operators.coreos.com  my-cert-manager -ojsonpath --template  "{.status.currentCSV}")

kubectl get csv $CERT_MANAGERR_CURRENT_CSV -ojson | jq .spec.install.spec.deployments[0]

kubectl patch csv $CERT_MANAGERR_CURRENT_CSV -n operators --type "json" -p '[
{"op":"add","path":"/spec/install/spec/deployments/0/spec/template/spec/containers/0/args/-","value":"--dns01-recursive-nameservers-only"},
{"op":"add","path":"/spec/install/spec/deployments/0/spec/template/spec/containers/0/args/-","value":"--dns01-recursive-nameservers=ns1.digitalocean.com:53"}
]'

kubectl get csv $CERT_MANAGERR_CURRENT_CSV -ojson | jq .spec.install.spec.deployments[0]

# cert-manager operator should be now redeployed
# check
kubectl -n operators get subscription
kubectl -n operators get installplans
kubectl -n operators get clusterserviceversions

MY_CLUSTER_SHORTCUT='[PUT_YOURS!]'
MY_DOMAIN="$MY_CLUSTER_SHORTCUT.k8s.works.sckt.net"

kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
  namespace: operators
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@$MY_DOMAIN
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
    - dns01:
        digitalocean:
          tokenSecretRef:
            name: digitalocean-dns
            key: access-token
      selector:
        dnsZones:
        - '$MY_DOMAIN'
---
apiVersion: v1
kind: Secret
metadata:
  name: digitalocean-dns
  namespace: operators
data:
  access-token: "$(echo -n $MY_DO_TOKEN | base64 -w0)"
EOF
```

---
# Cert manager - in Action
![bg right:45% 80%](https://cert-manager.io/images/high-level-overview.svg)

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-tls
  namespace: monitoring
  annotations:
    ingress.kubernetes.io/tls-acme: "true"
    cert-manager.io/cluster-issuer: letsencrypt-production
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - grafana.$MY_DOMAIN
      secretName: grafana-tls
  rules:
    - host: grafana.$MY_DOMAIN
      http:
        paths:
        - backend:
            service:
              name: grafana
              port:
                name: http
          path: /
          pathType: ImplementationSpecific
EOF

# check
kubectl get secret grafana-tls  -oyaml
kubectl get secret grafana-tls -ojson | jq '.data["tls.crt"]' -Mr | base64 -d
kubectl get secret grafana-tls -ojson | jq '.data["tls.crt"]' -Mr | base64 -d | openssl x509 -text -noout

curl https://grafana.vnt.k8s.works.sckt.net -v
```

---
# External DNS
![bg right:45% 70%](https://github.com/kubernetes-sigs/external-dns/raw/master/docs/img/external-dns.png)

[external-dns](https://kubernetes-sigs.github.io/external-dns/latest/charts/external-dns/)
[Providers](https://github.com/kubernetes-sigs/external-dns?tab=readme-ov-file#new-providers)
```bash
# as root on client add conf file for dnsmasq
DO_NAMESERVER=$(dig ns1.digitalocean.com +noall +answer  | awk '{print $NF}')


echo "server=/k8s.works.sckt.net/$DO_NAMESERVER" > /etc/dnsmasq.d/k8s.works.sckt.net.conf
systemctl restart dnsmasq.service

# as regular user on client
MY_CLUSTER_SHORTCUT='[PUT_YOURS!]'
MY_DOMAIN="$MY_CLUSTER_SHORTCUT.k8s.works.sckt.net"
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm upgrade external-dns external-dns/external-dns  -n operators --install -f - <<EOF
provider: 
  name: digitalocean
sources: [ingress]
rbac:
  create: true
policy: upsert-only
domainFilters: [$(echo $MY_DOMAIN | awk  -F . '{print $(NF-1)"."$NF}')]
env:
  - name: DO_TOKEN
    valueFrom:
      secretKeyRef:
        name: digitalocean-dns  
        key: access-token
#txtOwnerId: "externaldns"
managedRecordTypes: ['A']
logLevel: debug
EOF

# check dns entries
MY_DO_TOKEN="[YOUR_TOKEN]"
arkade get doctl
doctl auth init --access-token $MY_DO_TOKEN
doctl compute domain records list sckt.net
```



---
# Certificate and DNS Management
![bg right:45% 90%](https://jirak.net/wp/wp-content/uploads/2022/11/self-service-DNS-certs-K8s_challenge.png)


---
# Persistance
```bash
kubectl apply -f https://github.com/rook/rook/raw/refs/tags/v1.17.1/deploy/examples/common.yaml
kubectl apply -f https://github.com/rook/rook/raw/refs/tags/v1.17.1/deploy/examples/operator.yaml

kubectl apply -f https://github.com/rook/rook/raw/refs/tags/v1.17.1/deploy/examples/crds.yaml

kubectl apply -f https://github.com/rook/rook/raw/refs/tags/v1.17.1/deploy/examples/cluster.yaml

kubectl krew install rook-ceph
kubectl rook-ceph ceph -s

# set dashboard via http
# kubectl edit CephCluster rook-ceph
#   dashboard:
#     enabled: true
#     port: 8080
#     ssl: false



# TODO: dashboard access no verified
kubectl apply -f - <<EOF
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name:      ceph-dashboard
  namespace: rook-ceph
spec:
  ingressClassName: traefik
  rules:
  - host: ceph-dashboard.tst.k8s.mycompany.com
    http:
      paths:
      - backend:
          service:
            name: rook-ceph-mgr-dashboard
            port:
              name: http-dashboard
        path: /
        pathType: ImplementationSpecific
EOF

# visit
ceph-dashboard.tst.k8s.mycompany.com

# get pasword for login
kubectl get secret rook-ceph-dashboard-password -ojson | jq  -Mr .data.password | base64 -d


# Create new staoge classes
kubectl apply -f https://github.com/rook/rook/raw/refs/tags/v1.17.1/deploy/examples/csi/rbd/storageclass.yaml
kubectl apply -f https://raw.githubusercontent.com/rook/rook/refs/tags/v1.17.1/deploy/examples/filesystem.yaml
kubectl apply -f https://raw.githubusercontent.com/rook/rook/refs/tags/v1.17.1/deploy/examples/csi/cephfs/storageclass.yaml
kubectl get storageclass

# set a new staoge class to be the default one
kubectl annotate storageclass local-path       storageclass.kubernetes.io/is-default-class=false --overwrite
kubectl annotate storageclass rook-ceph-block  storageclass.kubernetes.io/is-default-class=true  --overwrite


# TODO: deffiernce between ceph-block and cephfs

# deploy a test application to use persitancy via PVCs

# ReadWriteOnce
kubectl apply -f - <<EOF
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: rook-ceph-block
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: nginx:1.14.2
        volumeMounts:
         - name: mypvc
           mountPath: /var/lib/www/html
      volumes:
        - name: mypvc
          persistentVolumeClaim:
            claimName: rbd-pvc
            readOnly: false
EOF

# ReadWriteMany
kubectl apply -f - <<EOF
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd-pvc
spec:
  accessModes:
    # - ReadWriteOnce
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: rook-cephfs
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: nginx:1.14.2
        volumeMounts:
         - name: mypvc
           mountPath: /var/lib/www/html
      volumes:
        - name: mypvc
          persistentVolumeClaim:
            claimName: rbd-pvc
            readOnly: false
EOF

```


# PV and PVC

# Storage Classes

# Access Modes
* ReadWriteOnce (RWO) – The volume is mounted with read-write access for a single Node in your cluster. Any of the Pods running on that Node can read and write the volume’s contents.
* ReadOnlyMany (ROX) – The volume can be concurrently mounted to any of the Nodes in your cluster, with read-only access for any Pod.
* ReadWriteMany (RWX) – Similar to ReadOnlyMany, but with read-write access.
* ReadWriteOncePod (RWOP) – This new variant, introduced as a beta feature in Kubernetes v1.27, enforces that read-write access is provided to a single Pod. No other Pods in the cluster will be able to use the volume simultaneously.

# Reclaim Policies
* Retain
* Delete


# Volume Binding Modes

* WaitForFirstConsumer
* Immediate

# Expanding Volumes

# Container Storage Interface (CSI)

# Snaptshots
https://blog.palark.com/kubernetes-snaphots-usage/

# Provisioning Volumes from Snapshots









[Volume Types](https://kubernetes.io/docs/concepts/storage/volumes/#image)



![bg right:35% 90%](https://miro.medium.com/v2/resize:fit:720/format:webp/1*kbU4pDL2X1CDPWRofADmMg.png)



---
# Backup and Recovery
* Velero
* Ark
* Kasten K10
---
# User Management


[k8s Auth](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
[k8s authentication with KeyCloak OIDC](https://medium.com/@amirhosseineidy/kubernetes-authentication-with-keycloak-oidc-63571eaeed61)
[Prepare KeyCloak](https://geek-cookbook.funkypenguin.co.nz/recipes/kubernetes/keycloak/)
[k3s with KeyCloak](https://geek-cookbook.funkypenguin.co.nz/kubernetes/oidc-authentication/k3s-keycloak/#)
[JWT Debugger](https://jwt.io/)
[cleint id & client secret](https://stackoverflow.com/questions/44752273/do-keycloak-clients-have-a-client-secret/69726692#69726692)

```bash
CP_URL=$(kubectl cluster-info  | grep "control" | awk '{print $NF}')

helm repo add permission-manager https://sighupio.github.io/permission-manager
helm upgrade --install --namespace permission-manager --create-namespace  permission-manager permission-manager/permission-manager -f - <<EOF
ingress:
  enabled: true
  annotations:
    # kubernetes.io/ingress.class: nginx
    # cert-manager.io/cluster-issuer: my-cluster-issuer
  hosts:
    - host: permission-manager.tst.k8s.mycompany.com
      paths:
        - path: /
          pathType: ImplementationSpecific
image:
  tag: "v1.9.0"
config:
  clusterName: "tst"
  controlPlaneAddress: "$CP_URL"
  basicAuthPassword: "password"
EOF
```


# KeyCloak
[KeyCLoak Basic Deployment](https://www.keycloak.org/operator/basic-deployment)


```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: my-keycloak-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: operatorgroup
  namespace: my-keycloak-operator
spec:
  targetNamespaces:
  - my-keycloak-operator
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: my-keycloak-operator
  namespace: my-keycloak-operator
spec:
  channel: fast
  name: keycloak-operator
  source: operatorhubio-catalog
  sourceNamespace: olm
EOF

```




```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: keycloak-db-secret
  namespace: my-keycloak-operator
data:
  password: $(echo -n admin    | base64 -w0)
  username: $(echo -n password | base64 -w0)
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql-db
spec:
  serviceName: postgresql-db-service
  selector:
    matchLabels:
      app: postgresql-db
  replicas: 1
  template:
    metadata:
      labels:
        app: postgresql-db
    spec:
      containers:
        - name: postgresql-db
          image: postgres:15
          volumeMounts:
            - mountPath: /data
              name: cache-volume
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: keycloak-db-secret
                  key:  username
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: keycloak-db-secret
                  key:  password
            - name: PGDATA
              value: /data/pgdata
            - name: POSTGRES_DB
              value: keycloak
      volumes:
        - name: cache-volume
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-db
spec:
  selector:
    app: postgresql-db
  ports:
  - port: 5432
    targetPort: 5432
EOF

```




```bash
kubectl apply -f - <<EOF
apiVersion: k8s.keycloak.org/v2alpha1
kind: Keycloak
metadata:
  name: example-kc
spec:
  instances: 1
  db:
    vendor: postgres
    host: postgres-db
    usernameSecret:
      name: keycloak-db-secret
      key: username
    passwordSecret:
      name: keycloak-db-secret
      key: password
  http:
    tlsSecret: keycloak-tls
  hostname:
    hostname: keycloak.$MY_DOMAIN
  ingress:
    enabled: true
    className: traefik
    annotations:
      ingress.kubernetes.io/tls-acme: "true"
      cert-manager.io/cluster-issuer: letsencrypt-production
  proxy:
    headers: xforwarded # double check your reverse proxy sets and overwrites the X-Forwarded-* headers


# TODO: pathc ingress with tls
EOF


kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  labels:
    app: keycloak
    app.kubernetes.io/instance: example-kc
    app.kubernetes.io/managed-by: keycloak-operator
  name: example-kc-service-lb
  namespace: my-keycloak-operator
spec:
  type: LoadBalancer
  ports:
  - name: https
    port: 8443
    protocol: TCP
    targetPort: 8443
  - name: management
    port: 9000
    protocol: TCP
    targetPort: 9000
  selector:
    app: keycloak
    app.kubernetes.io/instance: example-kc
    app.kubernetes.io/managed-by: keycloak-operator
EOF




# on servers
cat > /etc/rancher/k3s/config.yaml <<EOF
kube-apiserver-arg:
- "oidc-issuer-url=https://keycloak.vna.k8s.works.sckt.net:8443/auth/realms/master/"
- "oidc-client-id=kube-apiserver"
- "oidc-username-claim=email"
- "oidc-groups-claim=groups"
EOF
systemctl restart k3s


# client

kubectl krew install oidc-login
kubectl oidc-login setup --oidc-issuer-url=https://keycloak.vna.k8s.works.sckt.net:8443/auth/realms/maste --oidc-client-id=kube-apiserver --oidc-client-secret=test


```




---
# GitOps
https://tecadmin.net/building-a-gitops-workflow-with-kubernetes-and-argocd/

---
# Tasks
- Create a multi-node cluster with kubeadm
- Create a multi-node cluster with k0s
- Create a multi-node cluster with k3s
- Install [Addons](https://codecap.github.io/cloud-workshops/k8s-addons.html) if needed
- Deploy Prometheus, Alertmanager and Grafana to monitor you k8s cluster
- Deploy ELK Stack to collect logs
- Deploy/Change/Scale/Celete a [staless application](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/)
- Review Monitoring Data for the deployed application
- Review Logs for the deployed application
- Access the application by ingress and service of type LoadBalancer

---
# Links
- [These slides](https://codecap.github.io/cloud-workshops/k8s-cluster-management.html)
- [kubectl cheatsheet](https://github.com/eon01/KubectlCheatSheet)
- [kubectl Quick Reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- [Kubernetes with kubeadm on Rocky Linux](https://phoenixnap.com/kb/install-kubernetes-on-rocky-linux)
- [K0S](https://docs.k0sproject.io/stable/k0sctl-install/)
- [K3S](https://docs.k3s.io/)
- [operatorhub.io](https://operatorhub.io)
- [Open Lifecycle Manager](https://olm.operatorframework.io/docs/getting-started/)
- [Prometheus](https://www.google.com/url?sa=t&source=web&rct=j&opi=89978449&url=https://prometheus.io/&ved=2ahUKEwjM3-Ts35GNAxVm1QIHHSksC8IQFnoECBgQAQ&usg=AOvVaw2SXvTedblZYzyyTOzZ8Y5x)
- [Prometheus Crash Course](https://www.youtube.com/watch?v=BEBsuA5tgUU)
- [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus)
- [ELK Stack](https://www.elastic.co/elastic-stack)
- [What is ELK Stack](https://www.youtube.com/watch?v=jT-y6oS10jk)
- [OpenSearch](https://opensearch.org/)
- [external-dns](https://kubernetes-sigs.github.io/external-dns/latest/charts/external-dns/)
- [Providers](https://github.com/kubernetes-sigs/external-dns?tab=readme-ov-file#new-providers)
- [Cert Manager - Intro](https://www.youtube.com/watch?v=Xv1bdeVnGGY)
- [Let's Encrypt - Challenge Types](https://letsencrypt.org/docs/challenge-types/)
