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
- 4 CPU
- 8 GB RAM
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

If there are no indexes created check the timzone on clusternodes:
```bash
timedatectl set-timezone UTC
reboot
```


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
MY_DO_TOKEN="[YOUR_TOKEN]"

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
![bg right:40% 90%](https://cert-manager.io/images/high-level-overview.svg)

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

curl https://grafana.$MY_CLUSTER_SHORTCUT.k8s.works.sckt.net -v
```

---
# External DNS
![bg right:45% 70%](https://github.com/kubernetes-sigs/external-dns/raw/master/docs/img/external-dns.png)

[external-dns](https://kubernetes-sigs.github.io/external-dns/latest/charts/external-dns/)
[Providers](https://github.com/kubernetes-sigs/external-dns?tab=readme-ov-file#new-providers)
```bash
# as root on client add conf file for dnsmasq
dnf install bind-utils -y # ensure we have dig command installed
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
# User Management

---
# Kubernetes Authentication
![bg right:35% 50%](https://riteshmblog.wordpress.com/wp-content/uploads/2022/05/kubernetes-secret.jpg)
- X509 client certificates
- Static token file
- Service account tokens
- OpenID Token
- Authenticating Proxy
- Anonymous request

---
# Kubernetes Authorization
![bg right:35% 50%](https://riteshmblog.wordpress.com/wp-content/uploads/2022/05/kubernetes-secret.jpg)
- Username
- Groupname
- Extrafields

---
# Authentication by Certificates
![bg right:35% 50%](https://riteshmblog.wordpress.com/wp-content/uploads/2022/05/kubernetes-secret.jpg)
```bash
# create a new key
openssl genrsa -out myuser-key.pem 2048

# creqte a CSR
openssl req -new -key myuser-key.pem -out myuser-csr.pem -subj "/CN=myuser/O=team-a/O=team-b"

# create CertificateSigningRequest in K8S
kubectl apply -f - <<EOF
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: myuser
spec:
  request: $(cat myuser-csr.pem  | base64 -w0)
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth
EOF
kubectl certificate approve myuser
kubectl get csr myuser -o yaml

# extract the certifcate
kubectl get csr myuser -o jsonpath='{.status.certificate}'  | base64 -d > myuser-crt.pem
# review certifcate
openssl x509 -text -noout -in myuser-crt.pem


kubectl config set-credentials myuser \
  --client-key=myuser-key.pem  \
  --client-certificate=myuser-crt.pem  \
  --embed-certs=true
```
---
# Authorization by Certificates
![bg right:35% 50%](https://riteshmblog.wordpress.com/wp-content/uploads/2022/05/kubernetes-secret.jpg)

```bash
# Review roles defined
kubectl get roles  -A 
kubectl get clusterroles
# create a binding
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: myuser-admin
subjects:
- kind: User
  name: myuser
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
EOF
# show the current user
kubectl krew install whoami
kubectl whoami
# set current user to myuser
kubectl config  set-context --current --user myuser
# test
kubectl whoami
kubectl get pods

```

---
# Predefined cluster roles
![bg right:35% 50%](https://github.com/kubernetes/community/raw/master/icons/svg/resources/unlabeled/c-role.svg)
[K8S Authorization - User Roles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles)

Standard roles:
- cluster-admin
- admin
- edit
- view

---
# Service Accounts / Tokens
![bg right:35% 50%](https://github.com/kubernetes/community/raw/master/icons/svg/resources/unlabeled/sa.svg)
```bash
kubectl -n default create serviceaccount jenkins
MY_TOKEN=$(kubectl create token jenkins)

kubectl apply -f - <<EOF
---
apiVersion: v1
kind: Secret
metadata:
  name: jenkins
  namespace: default
  annotations:
    kubernetes.io/service-account.name: "jenkins"
type: kubernetes.io/service-account-token
EOF

# put the token into ~/.kube/config
kubectl config set-credentials jenkins --token=$(kubectl get secrets jenkins -ojson  | jq .data.token -Mr  | base64 -d)
# review the token
kubectl get secrets jenkins -ojson  | jq .data.token -Mr  | base64 -d | awk -F . '{print $2}' | base64 -d | jq .

# add permissons
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins
  namespace: monitoring
subjects:
- kind: User
  name: system:serviceaccount:default:jenkins
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
EOF

# switch the user
kubectl config  set-context --current --user jenkins

# list some pods
kubectl get pods -n monitoring  # should work
kubeclt get pods -n kube-system # should not work
```

---
# Service Accounts
__usage within a pod__
![bg right:35% 50%](https://github.com/kubernetes/community/raw/master/icons/svg/resources/unlabeled/sa.svg)
```bash
# TODO: we are using elasticsearch
# Point to the internal API server hostname
APISERVER=https://kubernetes.default.svc
# Path to ServiceAccount token
SERVICEACCOUNT=/var/run/secrets/kubernetes.io/serviceaccount
# Read this Pod's namespace
NAMESPACE=$(cat ${SERVICEACCOUNT}/namespace)
# Read the ServiceAccount bearer token
TOKEN=$(cat ${SERVICEACCOUNT}/token)
# Reference the internal certificate authority (CA)
CACERT=${SERVICEACCOUNT}/ca.crt

# Explore the API with TOKEN
curl --cacert ${CACERT} --header "Authorization: Bearer ${TOKEN}" -X GET ${APISERVER}/api
# list namespaces
curl --cacert ${CACERT} --header "Authorization: Bearer ${TOKEN}" -X GET ${APISERVER}/api/v1/namespaces
# list pods
curl --cacert ${CACERT} --header "Authorization: Bearer ${TOKEN}" -X GET ${APISERVER}/api/v1/pods
# list pods wihtin a namespace
curl --cacert ${CACERT} --header "Authorization: Bearer ${TOKEN}" -X GET ${APISERVER}/api/v1/namespaces/kube-system/pods
```


--- 
# Roles vs ClusterRoles
![bg right:35% 50%](https://github.com/kubernetes/community/raw/master/icons/svg/resources/unlabeled/c-role.svg)
![bg right:35% 50%](https://github.com/kubernetes/community/raw/master/icons/svg/resources/unlabeled/role.svg)
```bash
kubectl get roles  -A
kubectl get clusterroles 

kubectl get rolebindings -A
kubectl get clusterrolebindings
```


---
# Aggregated Roles
![bg right:35% 50%](https://github.com/kubernetes/community/raw/master/icons/svg/resources/unlabeled/c-role.svg)
```bash
# review, take a look at permissions with crd
kubectl get clusterrole view  -oyaml
kubectl get clusterrole edit  -oyaml
kubectl get clusterrole admin -oyaml

# review, take a look at label rbac.authorization.k8s.io/aggregate-to-admin: "true"
# it means, specific role will be aggreagated to another one
kubectl get clusterroles  prometheuses.monitoring.coreos.com-v1-admin  -oyaml

```

---
# KeyCloak
![bg right:50% 50%](https://www.keycloak.org/resources/images/icon.svg)

---
# KeyCloak
![bg right:35% 50%](https://www.keycloak.org/resources/images/icon.svg)
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

---
# Certificate for KeyCloak
![bg right:35% 50%](https://www.keycloak.org/resources/images/icon.svg)
```bash
MY_DOMAIN=tst.k8s.mycompany.com
cat > keycloak-req.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = keycloak.$MY_DOMAIN
EOF

openssl genrsa -out  keycloak-root-ca-key.pem
openssl req -x509 -new -nodes -key keycloak-root-ca-key.pem \
  -days 3650 -sha256 -out keycloak-root-ca.pem -subj "/CN=kube-ca"

openssl genrsa -out keycloak-key.pem 2048

openssl req -new -key keycloak-key.pem -out keycloak-csr.pem \
  -subj "/CN=kube-ca" \
  -sha256 -config keycloak-req.cnf

openssl x509 -req -in keycloak-csr.pem \
  -CA keycloak-root-ca.pem -CAkey keycloak-root-ca-key.pem \
  -CAcreateserial -sha256 -out keycloak-crt.pem -days 3650 \
  -extensions v3_req -extfile keycloak-req.cnf

# put the certificate into keycloak-instance-tls secret
kubectl create secret tls  keycloak-instance-tls \
  --cert=keycloak-crt.pem \
  --key=keycloak-key.pem
```

---
# Create KeyCloak Instance
![bg right:35% 50%](https://www.keycloak.org/resources/images/icon.svg)
```bash
MY_DOMAIN="tst.k8s.mycompany.com"
kubectl apply -f - <<EOF
apiVersion: k8s.keycloak.org/v2alpha1
kind: Keycloak
metadata:
  name: keycloak-instance
spec:
  instances: 1
  hostname:
    hostname: keycloak.$MY_DOMAIN
  http:
    tlsSecret: keycloak-instance-tls
  ingress:
    enabled: false
---
apiVersion: v1
kind: Service
metadata:
  name: keycloak-instance-service-lb
spec:
  type: LoadBalancer
  ports:
  - name: https
    port: 9443
    protocol: TCP
    targetPort: 8443
  selector:
    app: keycloak
    app.kubernetes.io/instance: keycloak-instance
    app.kubernetes.io/managed-by: keycloak-operator
EOF

# check
curl --cacert keycloak-root-ca.pem https://keycloak.tst.k8s.mycompany.com:9443/realms/master/.well-known/openid-configuration
curl --cacert keycloak-root-ca.pem https://keycloak.tst.k8s.mycompany.com:9443/realms/master/
```


---
# Copy certificate
![bg right:35% 50%](https://www.keycloak.org/resources/images/icon.svg)
```bash
# from the client node
MY_MASTER_NODE=[IP]

# copy keycloak-root-ca.pem to each nodes
scp keycloak-root-ca.pem $MY_MASTER_NODE:/etc/rancher/k3s/keycloak-root-ca.pem"
```


---
# Configure kube-apiserver
![bg right:35% 50%](https://www.keycloak.org/resources/images/icon.svg)
```bash
# on every master node
MY_IP_PRFX=$(ip r | grep default | awk '{print $3}'| awk -F . '{print $1"."$2"."$3}') 
MY_IP=$(ip a | grep $MY_IP_PRFX | head -n1 | awk '{print $2}' | sed -e "s#/.*##")
MY_DOMAIN="tst.k8s.mycompany.com"

echo "$MY_IP keycloak.$MY_DOMAIN" >> /etc/hosts

# check
curl --cacert /etc/rancher/k3s/keycloak-root-ca.pem \
  -L  https://keycloak.tst.k8s.mycompany.com:9443

cat > /etc/rancher/k3s/config.yaml <<EOF
kube-apiserver-arg:
- "oidc-issuer-url=https://keycloak.$MY_DOMAIN:9443/realms/master"
- "oidc-ca-file=/etc/rancher/k3s/keycloak-root-ca.pem"
- "oidc-client-id=k8s"
- "oidc-username-claim=email"
- "oidc-groups-claim=groups"
- "oidc-username-prefix=oidc:"
- "oidc-groups-prefix=oidc:"
# logs to debug
# - 'audit-log-path=/var/lib/rancher/k3s/server/logs/audit.log'
# - 'audit-policy-file=/var/lib/rancher/k3s/server/audit.yaml'
# - 'audit-log-maxage=30'
# - 'audit-log-maxbackup=10'
# - 'audit-log-maxsize=100'
EOF

# policy for audit logs (debugging)
cat > /var/lib/rancher/k3s/server/audit.yaml <<EOF
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
EOF

# on master nodes
systemctl restart k3s.service
```

---
# Configure KeyCloak
![bg right:35% 50%](https://www.keycloak.org/resources/images/icon.svg)
```bash
# get credentials to login
kubectl get secrets keycloak-instance-initial-admin -ojson | jq .data.username -Mr | base64 -d
kubectl get secrets keycloak-instance-initial-admin -ojson | jq .data.password -Mr | base64 -d

# go to https://keycloak.$MY_DOMAIN in a browser and login
Add a new Client:                k8s
Client authentication            -> On
Direct access grants             -> On
Valid redirect URIs              -> * # ! not for productive use, securty issue
Valid post logout redirect URIs  -> * # ! not for productive use, securty issue

# go to Credentials to get CLIENT_SECRET

# add group membership mapper to client scope
# Client Scopes -> profile -> Mappers -> Add Mapper by Configuration -> Group Membership
name:                            k8s-groups
Token Claim Name:                groups
Full Group Path:                 off
# add a new group "operations"
# add a new user  "john"
Email verified                   -> On
Name                             john
Email                            john@$MY_DOMAIN
Join Groups:                     operations
# set password for the user
Temporary                        -> off
```


---
# Configure kubectl to use oidc (OpenID connect)
![bg right:40% 70%](https://upload.wikimedia.org/wikipedia/commons/thumb/a/a2/OpenID_logo_2.svg/2880px-OpenID_logo_2.svg.png)
```bash
# on the client
CLIENT_ID=k8s
# in keycloak: clients -> k8s -> Credentials
CLIENT_SECRET=[YOUR SECRET]

# setup client
kubectl krew install oidc-login
kubectl oidc-login setup \
  --oidc-issuer-url=https://keycloak.tst.k8s.mycompany.com:9443/realms/master \
  --oidc-client-id=$CLIENT_ID \
  --oidc-client-secret=$CLIENT_SECRET \
  --certificate-authority keycloak-root-ca.pem

# visit http://localhost:8000/ with browser

kubectl config set-credentials oidc \
  --exec-api-version=client.authentication.k8s.io/v1 \
  --exec-interactive-mode=Never \
  --exec-command=kubectl \
  --exec-arg=oidc-login \
  --exec-arg=get-token \
  --exec-arg="--oidc-issuer-url=https://keycloak.tst.k8s.mycompany.com:9443/realms/master" \
  --exec-arg="--oidc-client-id=$CLIENT_ID" \
  --exec-arg="--oidc-client-secret=$CLIENT_SECRET" \
  --exec-arg="--certificate-authority=keycloak-root-ca.pem"

kubectl oidc-login get-token \
  --oidc-issuer-url=https://keycloak.tst.k8s.mycompany.com:9443/realms/master \
  --oidc-client-id=$CLIENT_ID \
  --oidc-client-secret=$CLIENT_SECRET \
  --certificate-authority keycloak-root-ca.pem
```

---
# Review token
![bg right:35% 50%](https://jwt.io/_next/image?url=%2F_next%2Fstatic%2Fmedia%2Fjwt-flower.f20616b0.png&w=3840&q=75)
```bash
kubectl oidc-login get-token \
  --oidc-issuer-url=https://keycloak.tst.k8s.mycompany.com:9443/realms/master \
  --oidc-client-id=$CLIENT_ID \
  --oidc-client-secret=$CLIENT_SECRET \
  --certificate-authority keycloak-root-ca.pem \
  | jq .status.token -Mr  | awk -F. '{print $2}'  | base64 -d | jq .
```

---
# Set user permissions
![bg right:35% 50%](https://github.com/kubernetes/community/raw/master/icons/svg/resources/unlabeled/crb.svg)

```bash
MY_DOMAIN="tst.k8s.mycompany.com"
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: operations
subjects:
# You can specify more than one "subject"
- kind: User
  name: oidc:john@$MY_DOMAIN
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: oidc:operations
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole                   # this must be Role or ClusterRole
  name: edit                          # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
EOF
```

---
# Test user / group permissions
![bg right:35% 50%](https://github.com/kubernetes/community/raw/master/icons/svg/resources/unlabeled/user.svg)

```bash
kubectl krew install whoami

kubectl whoami
kubectl get pods

kubectl config  set-context --current --user oidc

kubectl whoami
kubectl get pods
```

---
# GitOps
![bg right:50% 70%](https://miro.medium.com/v2/resize:fit:720/format:webp/0*bBhXCGL9x6aoujJw.jpg)
[Intro](https://www.youtube.com/watch?v=f5EpcWp0THw)

__GitOps__ is a practice of managing infrastructure and apps with Git as the source of truth.



---
# ArgoCD
Argo CD is a Kubernetes-native continuous deployment (CD) tool.
![bg right:50% 70%](https://miro.medium.com/v2/resize:fit:720/format:webp/1*4AJqcQrbk9HUWkqizJ2TdA.png)


---
# Installation by OLM
![bg right:50% 70%](https://miro.medium.com/v2/resize:fit:720/format:webp/0*nHynQ_-YYFe3gGtN.png)

```bash
kubectl apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: my-argocd-operator
  namespace: operators
spec:
  channel: alpha
  name: argocd-operator
  source: operatorhubio-catalog
  sourceNamespace: olm
EOF


kubectl apply -f - <<EOF
---
apiVersion: v1
kind: Namespace
metadata:
  name: gitops-system
---
apiVersion: argoproj.io/v1beta1
kind: ArgoCD
metadata:
  name: argocd-instance
  namespace: gitops-system
spec:
  server:
    insecure: true
    host: argocd.tst.k8s.mycompany.com
    ingress:
      enabled: true
      ingressClassName: traefik
EOF

# get admin pass to login
kubectl get secret argocd-instance-cluster -ojson \
  | jq -Mr '.data["admin.password"]' | base64 -d
```


---
# Configure ArgoCD
![bg right:35% 95%](https://devtron.ai/blog/content/images/size/w1000/2025/03/Untitled--5-.jpg)
```bash
# namespaces which argocd is allowed to work on
kubectl patch secret argocd-instance-default-cluster-config \
  -p='{"data":{"namespaces": "'$(echo -n "default,emojivoto,gitops-system,ingress-nginx,guestbook"|base64 -w0)'"}}'
# argocd should be able to create cluster resources
kubectl patch secret argocd-instance-default-cluster-config \
  -p='{"data":{"clusterResources": "'$(echo -n "true"|base64 -w0)'"}}'
# more permissions for argocd
kubectl create clusterrolebinding argocd-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=gitops-system:argocd-instance-argocd-application-controller
```


---
# Guestbook
![bg right:35% 100%]()
```bash
kubectl apply -f - <<EOF
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: gitops-system
  finalizers:
    - resources-finalizer.argocd.argoproj.io/foreground
spec:
  destination:
    namespace: guestbook
    server: https://kubernetes.default.svc
  project: default
  source:
    path:    examples/gitops/guestbook
    repoURL: https://github.com/codecap/cloud-workshops.git
    targetRevision: HEAD
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - PruneLast=true
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
EOF
```
---
# Emojivoto
![bg right:40% 90%](https://constellation-docs.netlify.app/constellation/assets/images/example-emojivoto-55d31a6fd0c07e8e44b2b51d55dac60d.jpg)

```bash
kubectl apply -f - <<EOF
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: emojivoto
  namespace: gitops-system
  finalizers:
    - resources-finalizer.argocd.argoproj.io/foreground
spec:
  project: default
  destination:
    server:         https://kubernetes.default.svc
  source:
    repoURL:        https://github.com/BuoyantIO/emojivoto/
    path:           kustomize/deployment
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - PruneLast=true
    - CreateNamespace=false
    - PrunePropagationPolicy=foreground
EOF
```

---
# Nginx ingress controller
![bg right:45% 80%](https://www.svgrepo.com/show/373924/nginx.svg)
```bash
kubectl apply -f - <<EOF
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ingress-nginx
  namespace: gitops-system
  finalizers:
    - resources-finalizer.argocd.argoproj.io/foreground
spec:
  project: default
  destination:
    server:         https://kubernetes.default.svc
    namespace:      ingress-nginx
  source:
    repoURL:        https://kubernetes.github.io/ingress-nginx
    chart:          ingress-nginx
    targetRevision: 4.12.3
    helm:
      valuesObject:
        controller:
          admissionWebhooks:
            enabled: false
          service:
            ports:
              http:  18080
              https: 18443
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - PruneLast=true
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
EOF
```


---
# Deploy Applications from GIT
![bg right:50% 50%](https://www.svgrepo.com/show/303548/git-icon-logo.svg)

```bash
kubectl apply -f - <<EOF
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name:      app-loader
  namespace: gitops-system
  finalizers:
    - resources-finalizer.argocd.argoproj.io/foreground
spec:
  destination:
    namespace: gitops
    server: https://kubernetes.default.svc
  project: default
  source:
    path:    examples/gitops/applications
    repoURL: https://github.com/codecap/cloud-workshops.git
    targetRevision: HEAD
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - PruneLast=true
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
EOF
```

---
# Persistance
![bg right:50% 50%](https://www.cloudops.com/images/blog/post/RookCeph1.png)

---
# Persistance
![bg right:35% 90%](https://miro.medium.com/v2/resize:fit:720/format:webp/1*kbU4pDL2X1CDPWRofADmMg.png)
[K8s Docs](https://kubernetes.io/docs/concepts/storage/volumes/#image)

---
# Rook
![bg right:50% 50%](https://rook.io/images/rook-logo.svg)

```bash
# TODO: check time synced on every cluster node (ntp)
# deploy rook
kubectl apply -f https://github.com/rook/rook/raw/refs/tags/v1.17.1/deploy/examples/common.yaml
kubectl apply -f https://github.com/rook/rook/raw/refs/tags/v1.17.1/deploy/examples/operator.yaml
kubectl apply -f https://github.com/rook/rook/raw/refs/tags/v1.17.1/deploy/examples/crds.yaml
kubectl apply -f https://github.com/rook/rook/raw/refs/tags/v1.17.1/deploy/examples/cluster.yaml

# check
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

# visit in browser
https://ceph-dashboard.tst.k8s.mycompany.com

# get pasword for login
kubectl get secret rook-ceph-dashboard-password -ojson | jq  -Mr .data.password | base64 -d
```
---
# Storage Classes
![bg right:35% 50%](https://github.com/kubernetes/community/raw/master/icons/svg/resources/unlabeled/sc.svg)

```bash
# Create new staoge classes
kubectl apply -f https://github.com/rook/rook/raw/refs/tags/v1.17.1/deploy/examples/csi/rbd/storageclass.yaml
kubectl apply -f https://raw.githubusercontent.com/rook/rook/refs/tags/v1.17.1/deploy/examples/filesystem.yaml
kubectl apply -f https://raw.githubusercontent.com/rook/rook/refs/tags/v1.17.1/deploy/examples/csi/cephfs/storageclass.yaml
kubectl get storageclass

# Set a new storage class to be the default one
kubectl annotate storageclass local-path  storageclass.kubernetes.io/is-default-class=false --overwrite
kubectl annotate storageclass rook-cephfs storageclass.kubernetes.io/is-default-class=true  --overwrite
```

---
# Access Modes
![bg right:15% 90%](https://github.com/kubernetes/community/raw/master/icons/svg/resources/unlabeled/pvc.svg)

- __ReadWriteOnce (RWO)__ – The volume is mounted with read-write access for a single Node in your cluster. Any of the Pods running on that Node can read and write the volume’s contents.
- __ReadOnlyMany (ROX)__ – The volume can be concurrently mounted to any of the Nodes in your cluster, with read-only access for any Pod.
- __ReadWriteMany (RWX)__ – Similar to ReadOnlyMany, but with read-write access.
- __ReadWriteOncePod (RWOP)__ – Eenforces that read-write access is provided to a single Pod. No other Pods in the cluster will be able to use the volume simultaneously.


---
# Access Modes in Practice - RWO
![bg right:25% 70%](https://github.com/kubernetes/community/raw/master/icons/svg/resources/unlabeled/vol.svg)
```bash
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
```
---
# Access Modes in Practice - RWX
![bg right:25% 70%](https://github.com/kubernetes/community/raw/master/icons/svg/resources/unlabeled/vol.svg)
```bash
kubectl apply -f - <<EOF
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc
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
            claimName: cephfs-pvc
            readOnly: false
EOF
```
---
# PV and PVC
![bg right:45% 90%](https://miro.medium.com/v2/resize:fit:720/format:webp/1*eUpYUJz3bTBAcyMCI6ThOw.png)
__Persistent Volume__ — low level representation of a storage volume.

__Persistent Volume Claim__ — binding between a Pod and Persistent Volume.

__Storage Class__ — allows for dynamic provisioning of Persistent Volumes.


---
# Reclaim Policies
![bg right:35% 50%](https://github.com/kubernetes/community/raw/master/icons/svg/resources/unlabeled/vol.svg)
__Retain__ -  allows for manual reclamation of the resource. When the PersistentVolumeClaim is deleted, the PersistentVolume still exists and the volume is considered "released". An administrator can manually reclaim the volume.

__Delete__ - For volume plugins that support the Delete reclaim policy, deletion removes both the PersistentVolume object from Kubernetes, as well as the associated storage asset in the external infrastructure.

---
![bg right:35% 50%](https://github.com/kubernetes/community/raw/master/icons/svg/resources/unlabeled/vol.svg)
# Volume Binding Modes
- WaitForFirstConsumer
- Immediate
---
# Expanding Volumes
```bash
# edit  a pvc and change the size

```

---
# Container Storage Interface (CSI)
![bg right:25% 80%](https://kubernetes.io/images/blog-logging/2018-04-10-container-storage-interface-beta/csi-logo.png)
__Container Storage Interface (CSI)__  is an initiative to unify the storage interface of Container Orchestrator Systems (COs) like Kubernetes, Mesos, Docker swarm, cloud foundry, etc. combined with storage vendors like Ceph, Portworx, NetApp etc. This means, implementing a single CSI for a storage vendor is guaranteed to work with all COs.

---
# Container Storage Interface (CSI)
![bg right:50% 80%](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*vhFEJLBlQ24LLyHvG4ssaw.png)

---
# Volume Snapshots
![bg right:35% 50%](https://miro.medium.com/v2/resize:fit:4800/format:webp/1*LI6lYZu8NlllluGRsswACw.png)

```bash
# Install CRDS
for f in \
  groupsnapshot.storage.k8s.io_volumegroupsnapshotclasses.yaml \
  groupsnapshot.storage.k8s.io_volumegroupsnapshotcontents.yaml \
  groupsnapshot.storage.k8s.io_volumegroupsnapshots.yaml \
  snapshot.storage.k8s.io_volumesnapshotclasses.yaml \
  snapshot.storage.k8s.io_volumesnapshotcontents.yaml \
  snapshot.storage.k8s.io_volumesnapshots.yaml
do
  kubectl apply -f   https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/refs/tags/v8.2.1/client/config/crd/$f
done

# Install controler
for f in \
  rbac-snapshot-controller.yaml \
  setup-snapshot-controller.yaml
do
   kubectl apply -n kube-system -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/refs/tags/v8.2.1/deploy/kubernetes/snapshot-controller/$f
done

# Snaphostclass for RBD(ceph)
kubectl apply -f https://github.com/rook/rook/raw/refs/tags/v1.17.1/deploy/examples/csi/rbd/snapshotclass.yaml
# Snapshotclass for CephFS
kubectl apply -f  https://github.com/rook/rook/raw/refs/tags/v1.17.1/deploy/examples/csi/cephfs/snapshotclass.yaml
```

---
# Volume Snapshots
![bg right:35% 50%](https://miro.medium.com/v2/resize:fit:4800/format:webp/1*LI6lYZu8NlllluGRsswACw.png)

```bash
# Create a snapshot from existing PVC
kubectl apply -f - <<EOF
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: cephfs-pvc-snapshot
  namespace: default
spec:
  source:
    persistentVolumeClaimName: cephfs-pvc
  volumeSnapshotClassName: csi-cephfsplugin-snapclass
EOF
```

---
# PVC from Snapshots
![bg right:50% 35%](https://miro.medium.com/v2/resize:fit:4800/format:webp/1*LI6lYZu8NlllluGRsswACw.png)
```bash
# restore a snptshot to pvc
kubectl apply -f - <<EOF
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc-restore
  namespace: default
spec:
  accessModes:
  - ReadWriteMany
  dataSource:
    apiGroup: snapshot.storage.k8s.io
    kind: VolumeSnapshot
    name: cephfs-pvc-snapshot
  resources:
    requests:
      storage: 10Gi
  storageClassName: rook-cephfs
EOF
```
---
# PVC from Snapshots
![bg right:50% 35%](https://miro.medium.com/v2/resize:fit:4800/format:webp/1*LI6lYZu8NlllluGRsswACw.png)
```bash
# test
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: snapshot-test-pod
spec:
  containers:
  - name: my-container
    image: busybox
    command: ["sleep", "infinity"]
    volumeMounts:
    - mountPath: /data
      name: my-volume
  volumes:
  - name: my-volume
    persistentVolumeClaim:
      claimName: cephfs-pvc-restore
EOF
```

---
# Volume Group Snapshots
![bg right:30% 60%](https://miro.medium.com/v2/resize:fit:4800/format:webp/1*LI6lYZu8NlllluGRsswACw.png)
__group snapshot__ represents “copies” from multiple volumes that are taken at the same point in time.

---
# Backup and Recovery
* Velero
* Ark
* Kasten K10
* ...


---
![bg right:50% 80%](https://velero.io/docs/v1.8/img/velero.png)

---
# Velero
[Docs](https://velero.io/docs/main/)
![bg right:40% 90%](https://hewlettpackard.github.io/hpe-solutions-openshift/4.14-AMD-LTI/assets/img/f23.e9d41058.png)
- Take backups of your cluster and restore in case of loss.
- Migrate cluster resources to other clusters.
- Replicate your production cluster to development and testing clusters.


---
# S3 Stotage
[MinIO Docs](https://docs.min.io/docs/minio-quickstart-guide.html)
![bg right:35% 80%](https://min.io/resources/img/products/overview_main.svg)
```bash
# on the client
# as root
dnf -y install dnf-plugins-core
dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
dnf -y install docker-ce docker-ce-cli
systemctl enable --now docker
usermod -G docker deploy
# logout and login again as user deploy

# as deploy user
mkdir -p ${HOME}/minio/data

docker run  -d                      \
  --restart always                  \
  -p 9000:9000                      \
  -p 9001:9001                      \
  --user $(id -u):$(id -g)          \
  --name minio1                     \
  -e "MINIO_ROOT_USER=admin"        \
  -e "MINIO_ROOT_PASSWORD=password" \
  -v ${HOME}/minio/data:/data       \
  quay.io/minio/minio               \
  server /data --console-address ":9001"

```

---
# Install Velero
![bg right:35% 80%](https://velero.io/img/Velero.svg)
```bash
# on client
# Get velero CLI
curl -sS -L -o - \
  https://github.com/vmware-tanzu/velero/releases/download/v1.16.1/velero-v1.16.1-linux-amd64.tar.gz \
  | tar -xzvf - velero-v1.16.1-linux-amd64/velero --strip-components 1 
mv velero ~/bin/

# Place the config file
cat > credentials-velero <<EOF
[default]
aws_access_key_id = admin
aws_secret_access_key = password
EOF

# Install
velero install \
    --provider aws \
    --bucket velero \
    --features=EnableCSI \
    --secret-file ./credentials-velero \
    --plugins velero/velero-plugin-for-aws:v1.10.0 \
    --use-node-agent \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://10.10.20.200:9000

# Verify
velero backup-location get
velero snapshot-location get
velero plugin get
```



---
# Create Backup and Restore
![bg right:35% 80%](https://velero.io/img/Velero.svg)
```bash
# TODO: deploy emojivoto
# on client
# Backup
velero backup create emojivoto \
  --include-namespaces emojivoto

# Restore
velero restore create \
  --from-backup emojivoto \
  --namespace-mappings emojivoto:emojivoto-restored

# Details about restore
velero restore describe  emojivoto-20250714121311

# s3 client
MY_IP_PRFX=$(ip r | grep default | awk '{print $3}'| awk -F . '{print $1"."$2"."$3}') 
MY_IP=$(ip a | grep $MY_IP_PRFX | head -n1 | awk '{print $2}' | sed -e "s#/.*##")
arkade get mc
mc alias set s3 http://$MY_IP:9000 admin password
# review ~/.mc/config.json

mc --autocompletion
source ~/.bashrc

mv ls s3/
mv ls s3/velero
# ...
mc cat s3/velero/backups/emojivoto/emojivoto-resource-list.json.gz | zcat  | jq .
```

---
# Add some data into namespace
![bg right:25% 70%](https://github.com/kubernetes/community/raw/master/icons/svg/resources/unlabeled/vol.svg)
```bash
kubectl apply -f - <<EOF
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc
  namespace: emojivoto
spec:
  accessModes:
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
  namespace: emojivoto
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
            claimName: cephfs-pvc
            readOnly: false
EOF
```


---
# Create Backup and Restore
![bg right:35% 80%](https://velero.io/img/Velero.svg)
```bash
# on client
MY_IP_PRFX=$(ip r | grep default | awk '{print $3}'| awk -F . '{print $1"."$2"."$3}') 
MY_IP=$(ip a | grep $MY_IP_PRFX | head -n1 | awk '{print $2}' | sed -e "s#/.*##")
# After reviewing we should know how to fix warnings
# Create a new backup
velero backup create 250714-emojivoto-001    \
  --include-namespaces emojivoto         \
  --exclude-cluster-scoped-resources '*' \
  --exclude-namespace-scoped-resources clusterserviceversions.operators.coreos.com

# Restore
velero restore create --from-backup 250714-emojivoto-001 --namespace-mappings emojivoto:emojivoto-restored

# Install kopia
curl -sS -L -o - \
  https://github.com/kopia/kopia/releases/download/v0.20.1/kopia-0.20.1-linux-x64.tar.gz \
  | tar -xzvf - kopia-0.20.1-linux-x64/kopia --strip-components 1 
mv kopia ~/bin/

kopia repository connect s3    \
  --bucket velero              \
  --access-key admin           \
  --secret-access-key=password \
  --endpoint "$MY_IP:9000"     \
  --disable-tls                \
  --prefix kopia/emojivoto/    \
  --password static-passw0rd

kopia snapshot list --all
mkdir kopia-restore
kopia snapshot restore [SNAPSHOT_ID] kopia-restore/
```



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
- [K8S Auth](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [JWT Debugger](https://jwt.io/)
- [KeyCloak Basic Deployment](https://www.keycloak.org/operator/basic-deployment)
- [Rook storage orchestrator](https://rook.io/)
- [Volume Group Snapshots with Rook](https://rook.io/docs/rook/latest-release/Storage-Configuration/Ceph-CSI/ceph-csi-volume-group-snapshot/)
- [Kubernetes Volume Types](https://kubernetes.io/docs/concepts/storage/volumes/#image)
- [Velero](https://velero.io/docs/main/)
- [MinIO Docs](https://docs.min.io/docs/minio-quickstart-guide.html)
