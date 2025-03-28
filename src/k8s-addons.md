---
title:       Workshops
description: Core Addons for Kubernetes
author:      Vladislav Nazarenko (vnazarenko@📯socket.de)
keywords:    Kubernetes,Addons
url:         
image:
transition: cover
theme: default
backgroundImage: url(https://codecap.github.io/cloud-workshops/assets/background.jpg)
paginate: true

---

# Kubernetes Addons

---

# Linux: STDIN, STDOUT, STDERR
![bg right:30% 60%](https://cdn.hashnode.com/res/hashnode/image/upload/v1630813456614/NTviFs-yT.png?auto=compress,format&format=webp)

```bash
# cat - Concatenate FILE(s) to standard output.

#
# read from a file and put the output into stdout
#
cat file

#
# read from a file and redirect the output into a new-file
#
cat file > new-file

#
# read from stdin put to stdout
#
cat <<EOF
Text to print
EOF

#
# read from stdin and redirect output to a file
#
cat > a-file <<EOF
Text to print
EOF
```

---
# DNS
```bash
MY_COMPANY_NAMESERVER=$(cat /etc/resolv.conf  | grep nameserver | head -n1 | awk '{print $NF}')
MY_NIC=ens192
MY_IP=$(ip a show $MY_NIC | grep "inet " | awk '{print $2}' | awk -F / '{print $1}')

sudo dnf install -y dnsmasq
NETWORK_NAME=kind
IP_BASE=$(docker network inspect $NETWORK_NAME | jq .[0].IPAM -Mr | grep Gateway | awk -F '"' '{print $4}' | sed -r -e 's/[.][0-9]+$//')
  echo "address=/.tst.k8s.mycompany.com/${IP_BASE}.240" | sudo tee  -a  /etc/dnsmasq.d/tst.k8s.mycompany.com.conf 
  echo "server=$MY_COMPANY_NAMESERVER" | sudo tee  -a  /etc/dnsmasq.d/tst.k8s.mycompany.com.conf 
  echo "interface=$MY_NIC" | sudo tee  -a  /etc/dnsmasq.d/tst.k8s.mycompany.com.conf 

sed -e "s/dns=.*/dns=$MY_IP;/" -i  /etc/NetworkManager/system-connections/ens192.nmconnection
nmcli device reapply MY_NIC

systemctl enable --now dnsmasq
```

---
# Helm
![bg right:45% 90%](https://lazzaretti.me/images/blog/2024/introduction-to-helm-3/helm3-intro.png)
## Package manager for Kubernetes

Helm is the best way to find, share, and use software built for Kubernetes.

```bash
curl -sLS https://get.arkade.dev | sh; mkdir -p  ~/.arkade/bin/; mv arkade ~/.arkade/bin/
echo 'export PATH="~/.arkade/bin/:$PATH"' > ~/.bash_profile; source ~/.bash_profile

arkade get helm
```
---
# MetalLB
![bg right:45% 100%](https://www.redhat.com/rhdc/managed-files/ohc/MetalLB%20advanced%20configuration-1.png)
[](https://codecap.github.io/cloud-workshops/assets/metallb-layer2.png)

[MetalLB docs](https://metallb.io/configuration/)

```bash
APP_NAME=metallb
helm repo add $APP_NAME https://metallb.github.io/metallb
helm install $APP_NAME $APP_NAME/$APP_NAME -n $APP_NAME-system --create-namespace
```
---
# MetalLB - Configuration
```bash
NETWORK_NAME=kind
# NETWORK_NAME=k3d-hello-k3d
IP_BASE=$(docker network inspect $NETWORK_NAME | jq .[0].IPAM -Mr | grep Gateway | awk -F '"' '{print $4}' | sed -r -e 's/[.][0-9]+$//')
START_IP="${IP_BASE}.240"
END_IP="${IP_BASE}.254"

kubectl apply -f- <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: example
  namespace: metallb-system
spec:
  addresses:
  - $START_IP-$END_IP
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: empty
  namespace: metallb-system
EOF
```
---

# Ingress Nginx
![bg right:45% 100%](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*AgWCYOe3yMevVfzT_1EHog.png)
[Kubernetes Ingress Docs](https://kubernetes.io/docs/concepts/services-networking/ingress/)
[Ingress Nginx Docs](https://kubernetes.github.io/ingress-nginx/user-guide/basic-usage/)


```bash
APP_NAME=ingress-nginx
helm repo add $APP_NAME https://kubernetes.github.io/ingress-nginx
helm install $APP_NAME $APP_NAME/$APP_NAME --namespace $APP_NAME --create-namespace
```

---
# Metrics Server
![bg right:45% 90%](https://github.com/kubernetes/design-proposals-archive/raw/main/instrumentation/monitoring_architecture.png?raw=true)

```bash
APP_NAME=metrics-server
helm repo add $APP_NAME https://kubernetes-sigs.github.io/metrics-server/
helm install $APP_NAME $APP_NAME/$APP_NAME --namespace kube-system
```

---
# Kubernetes Dashboard

```bash
APP_NAME=kubernetes-dashboard
DOMAIN=tst.k8s.mycompany.com
helm repo add $APP_NAME https://kubernetes.github.io/dashboard/
helm install $APP_NAME $APP_NAME/$APP_NAME --namespace $APP_NAME --create-namespace --values - <<EOF
---
app:
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - $APP_NAME.$DOMAIN
EOF

```
--- 
## Access to the Dashboard
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: $APP_NAME
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: $APP_NAME
---
apiVersion: v1
kind: Secret
metadata:
  name: admin-user
  namespace: $APP_NAME
  annotations:
    kubernetes.io/service-account.name: "admin-user"
type: kubernetes.io/service-account-token
EOF
# get the token to access dashboard
kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath="{.data.token}" | base64 -d
```
