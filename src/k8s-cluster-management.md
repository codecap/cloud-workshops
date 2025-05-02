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
Install and configure dnsmasq as desribed [here](https://codecap.github.io/cloud-workshops/k8s-addons.html#3)

```bash
# on the client node
IP_BASE="10.10.20"
IP_C01="$IP_BASE.200"
IP_M01="$IP_BASE.101"
IP_M02="$IP_BASE.102"
IP_M03="$IP_BASE.103"
IP_W01="$IP_BASE.141"
IP_W02="$IP_BASE.142"
IP_W03="$IP_BASE.143"
MY_USER=rocky
```
---
# Preparations
![bg right:50% 50%](https://image.pngaaa.com/935/5527935-middle.png)
```bash
# on the client node
ssh-keygen

# the output of the following command should be executed on every node
echo "echo \"$(cat ~/.ssh/id_rsa.pub)\" >> ~$MY_USER/.ssh/authorized_keys"

# install arkade
curl -sLS https://get.arkade.dev | sh; mkdir -p  ~/.arkade/bin/; mv arkade ~/.arkade/bin/
echo 'export PATH="~/.arkade/bin/:$PATH"' >> ~/.bash_profile; source ~/.bash_profile

# on every node 
echo "$MY_USER ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/$MY_USER
```


---
# Preparations
![bg right:50% 50%](https://image.pngaaa.com/935/5527935-middle.png)
__Some tools for your comfort__
```bash
dnf install 'dnf-command(config-manager)'
dnf config-manager --add-repo https://kubecolor.github.io/packages/rpm/kubecolor.repo
dnf install -y kubecolor

dnf install -y kubecolor

kubectl completion bash > /etc/profile.d/kubectl-completion.sh
source /etc/profile
```

```bash
echo 'alias k="kubecolor"'                      >> ~/.bash_profile
echo 'complete -o default -F __start_kubectl k' >> ~/.bash_profile
source ~/.bash_profile
```

---
# Preparations
![bg right:50% 50%](https://image.pngaaa.com/935/5527935-middle.png)
__krew - get plagins for kubectl__

```bash
dnf install -y git
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
#
# Install krew plugins
#
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

setenforce 0
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/sysconfig/selinux

systemctl disable --now  firewalld 
```
__Install Containerd__
```bash
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
__Install kubeadm__
```bash
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

kubeadm config images pull
kubeadm config images list
```
---
# kubeadm
![bg right:50% 50%](https://kubernetes.io/images/kubeadm-stacked-color.png)
__Install K8S__
```bash
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
---
# kubeadm
![bg right:50% 50%](https://kubernetes.io/images/kubeadm-stacked-color.png)
__Reset__
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
![bg right:50% 50%](https://docs.k0sproject.io/stable/img/k0s-logo-2025-horizontal.svg#only-light)
```bash
# NODE: do we need this?
# prepare
# mkdir -p /run/k0s
# cd /run/k0s
# ln -s /run/containerd/containerd.sock
# ln -s /run/containerd/containerd.sock
# cd 
```

---
# k0sctl
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
# k0sctl
![bg right:50% 50%](https://docs.k0sproject.io/stable/img/k0s-logo-2025-horizontal.svg#only-light)
```bash
k0sctl apply --config ~/k0sctl.yaml

mkdir -p ~/.kube
k0sctl kubeconfig > ~/.kube/config

# review runnning containers on the node
# master
systemctl status k0scontroller.service
cat /etc/systemd/system/k0scontroller.service

# worker
ctr --address /run/k0s/containerd.sock -n k8s.io c ls

# reset
k0sctl reset --config ~/k0sctl.yaml
```

---
# k3s
![bg right:60% 50%](https://k3s.io/img/k3s-logo-light.svg)
```bash
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
mv ~/.kube/kubeconfig ~/.kube/config
```

---
# OLM
Operator Lifecycle Manager
[operatorhub.io](https://operatorhub.io)
[Docs](https://olm.operatorframework.io/docs/getting-started/0)
![bg right:40% 50%](https://olm.operatorframework.io/images/logo.svg)
```bash
arkade get operator-sdk
operator-sdk olm install
# check

kubectl -n olm get all
kubectl -n olm get ClusterServiceVersion
kubectl -n olm get catalogsource
kubectl -n operators get subscriptions
kubectl -n operators get installplans

# list of available operators
kubectl -n olm get packagemanifest 

```

---
# Monitoring
[Prometheus Operator](https://prometheus-operator.dev/docs/getting-started/introduction/)
![bg right:40% 50%](https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/refs/heads/main/Documentation/logos/prometheus-operator-logo.svg)
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
![bg right:40% 50%](https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/refs/heads/main/Documentation/logos/prometheus-operator-logo.svg)
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
kubectl -n operators get subscription
kubectl -n operators get installplans
kubectl -n monitoring get prometheus prom-a
kubectl -n monitoring get pods
# remove
kubectl -n monitoring delete prometheus prom-a
```

---
# Monitoring
__Install and configure__
![bg right:40% 50%](https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/refs/heads/main/Documentation/logos/prometheus-operator-logo.svg)
```bash
git clone https://github.com/prometheus-operator/kube-prometheus.git
cd kube-prometheus/
git checkout remotes/origin/release-0.13

kubectl create ns monitoring
kubectl apply -f manifests/

kubectl delete -f   manifests/prometheusOperator-deployment.yaml
kubectl delete -f   manifests/{alertmanager,grafana,prometheus}-networkPolicy.yaml

```
---
# Monitoring
![bg right:40% 50%](https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/refs/heads/main/Documentation/logos/prometheus-operator-logo.svg)
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
![bg right:50% 80%](https://miro.medium.com/v2/resize:fit:720/format:webp/1*H3nzCLJvta-Qj1CXimwOkA.png)
[Remote Write](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#remote_write)
With remote write, data is pushed out from Prometheus to other systems.

---
# Export Monitoring Data
[Federation](https://prometheus.io/docs/prometheus/latest/federation/)
![bg right:50% 60%](https://last9.ghost.io/content/images/size/w1000/2024/01/Prometheus_Federation_Federated_Architecture.jpg)
With federation, a Prometheus server pulls data from other Prometheus servers.

---
# Export Monitoring Data
![bg right:40% 30%](https://upload.wikimedia.org/wikipedia/commons/3/38/Prometheus_software_logo.svg)

__Pull data from in-cluster into your global prometheus instance__
```bash
cat > /etc/yum.repos.d/prometheus.repo <<"EOF"
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
dnf -y  install prometheus2

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
__Pull data from in-cluster your global prometheus instance__
```bash
cat > /etc/prometheus/prometheus.yml <<EOF
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
systemctl enable --now  prometheus
```

---
# Logging

---
# Monitoring

---
# Persistancy

---
# Cert manager + ingress

---
# Backup and Recovery

---
# User Management

---
