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

```bash
# cat - Concatenate FILE(s) to standard output.

#
# read from a file and put the output into stdout
#
cat file

#
# read from a file and redirect the output into a file
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


![bg right:30% 60%](https://cdn.hashnode.com/res/hashnode/image/upload/v1630813456614/NTviFs-yT.png?auto=compress,format&format=webp)


---

# MetalLB
![bg right:%40 30%](https://codecap.github.io/cloud-workshops/assets/metallb-layer2.png)

---
```bash
APP_NAME=metallb
helm repo add $APP_NAME https://metallb.github.io/metallb
helm install $APP_NAME $APP_NAME/$APP_NAME -n $APP_NAME-system --create-namespace
```
---

## MetalLB - Configuration
```bash
START_IP="X.Y.Z.240"
END_IP="X.Y.Z.254"

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

## Ingress Nginx

### TODO: diagram


```bash
APP_NAME=ingress-nginx
helm repo add $APP_NAME https://kubernetes.github.io/ingress-nginx
helm install $APP_NAME $APP_NAME/$APP_NAME --namespace $APP_NAME --create-namespace
```
---

## Kubernetes Dashboard

### TODO: diagram


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
