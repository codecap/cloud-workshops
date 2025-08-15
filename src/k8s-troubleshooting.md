---
title:       Kubernetes Troubleshooting
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

# Kubernetes Troubleshooting
![bg left:40% 80%](https://upload.wikimedia.org/wikipedia/commons/3/39/Kubernetes_logo_without_workmark.svg)

---
## Simple Scenarios ⭐
![bg right:35% 80%](https://assets.zyrosite.com/cdn-cgi/image/format=auto,fit=crop/YD0y4WNK2NF309oZ/10595761-YNqNDGrBpeU5ywZO.png)

---
### Crashing Pod (CrashLoopBackoff)
![bg right:35% 50%](https://raw.githubusercontent.com/kubernetes/community/690273e13778a52736ed5f2d83597319186f637a/icons/svg/infrastructure_components/unlabeled/control-plane.svg)
Apply the manifest. Ensure the pod can start.
```bash
kubectl apply -f https://raw.githubusercontent.com/robusta-dev/kubernetes-demos/main/crashpod/broken.yaml
```

---
### OOMKilled Pod (Out of Memory Kill)
![bg right:35% 50%](https://raw.githubusercontent.com/kubernetes/community/690273e13778a52736ed5f2d83597319186f637a/icons/svg/infrastructure_components/unlabeled/control-plane.svg)
Apply the manifest. Monitor memory consumption. After some time the pod will be killed by OS. Fix the problem.
```bash
kubectl apply -f https://raw.githubusercontent.com/robusta-dev/kubernetes-demos/main/oomkill/oomkill_job.yaml
```

---
### Pending Pod (Unschedulable due to Node Selectors)
![bg right:35% 50%](https://raw.githubusercontent.com/kubernetes/community/690273e13778a52736ed5f2d83597319186f637a/icons/svg/infrastructure_components/unlabeled/control-plane.svg)
Apply the manifest, ensure the pod can start
```bash
kubectl apply -f https://raw.githubusercontent.com/robusta-dev/kubernetes-demos/refs/heads/main/pending_pods/pending_pod_node_selector.yaml
```

---
### ImagePullBackOff
![bg right:35% 50%](https://raw.githubusercontent.com/kubernetes/community/690273e13778a52736ed5f2d83597319186f637a/icons/svg/infrastructure_components/unlabeled/control-plane.svg)
Ensure deployment pods can start
```bash
kubectl apply -f https://raw.githubusercontent.com/robusta-dev/kubernetes-demos/main/image_pull_backoff/no_such_image.yaml
```

---
### Liveness Probe Failure
![bg right:35% 50%](https://raw.githubusercontent.com/kubernetes/community/690273e13778a52736ed5f2d83597319186f637a/icons/svg/infrastructure_components/unlabeled/control-plane.svg)
If liveness probe fails, the pod will be restarted after some period of time. Make sure the probe is successful.
```bash
kubectl apply -f https://raw.githubusercontent.com/robusta-dev/kubernetes-demos/main/liveness_probe_fail/failing_liveness_probe.yaml
```

---
### Readiness Probe Failure
![bg right:35% 50%](https://raw.githubusercontent.com/kubernetes/community/690273e13778a52736ed5f2d83597319186f637a/icons/svg/infrastructure_components/unlabeled/control-plane.svg)
Deploy. The pods should be shown as ready
```bash
kubectl apply -f https://raw.githubusercontent.com/robusta-dev/kubernetes-demos/main/readiness_probe_fail/failing_readiness_probe.yaml
```

---
### Wrong Mount Path
![bg right:35% 50%](https://raw.githubusercontent.com/kubernetes/community/690273e13778a52736ed5f2d83597319186f637a/icons/svg/infrastructure_components/unlabeled/control-plane.svg)
Apply the manifest. Check the logs of running pods. The reader container in the deployment should not log error messages.
```bash
kubectl apply -f https://raw.githubusercontent.com/mhausenblas/troubleshooting-k8s-apps/refs/heads/master/04_storage-failedmount.yaml
```

---
## Advanced Scenarios ⭐⭐
![bg right:35% 80%](https://assets.zyrosite.com/cdn-cgi/image/format=auto,fit=crop/YD0y4WNK2NF309oZ/10595761-YNqNDGrBpeU5ywZO.png)

---
### Crashing v2
![bg right:35% 50%](https://raw.githubusercontent.com/kubernetes/community/690273e13778a52736ed5f2d83597319186f637a/icons/svg/infrastructure_components/unlabeled/control-plane.svg)
Ensure deployment pods keep running.
```bash
kubectl apply -f https://raw.githubusercontent.com/robusta-dev/kubernetes-demos/refs/heads/main/crashpod.v2/crashloop-cert-app.yaml
# refresh cert
# kubectl apply -f https://raw.githubusercontent.com/robusta-dev/kubernetes-demos/refs/heads/main/crashpod.v2/new-cert.yaml
```

---
### Application Failure
Apply manifests. Ensure meme-service is working correctly and curl-deployment can fetch data from it.
![bg right:35% 50%](https://raw.githubusercontent.com/kubernetes/community/690273e13778a52736ed5f2d83597319186f637a/icons/svg/infrastructure_components/unlabeled/control-plane.svg)
```bash
for i in https://raw.githubusercontent.com/robusta-dev/kubernetes-demos/refs/heads/main/holmes-meme-generator/failure/{config,curl,deployment}.yaml
do
  kubectl apply -f $i
done
```

---
### Init Containers
Apply manifest. Ensure the pod can start and keeps runnning.
![bg right:35% 50%](https://raw.githubusercontent.com/kubernetes/community/690273e13778a52736ed5f2d83597319186f637a/icons/svg/infrastructure_components/unlabeled/control-plane.svg)
```bash
kubectl apply -f https://raw.githubusercontent.com/robusta-dev/kubernetes-demos/refs/heads/main/init_crashloop_backoff/create_init_crashloop_backoff.yaml
```

---
### Pod Resources
![bg right:35% 50%](https://raw.githubusercontent.com/kubernetes/community/690273e13778a52736ed5f2d83597319186f637a/icons/svg/infrastructure_components/unlabeled/control-plane.svg)
Apply manifest. Ensure deployment pods can start and keep running
```bash
kubectl apply -f https://raw.githubusercontent.com/robusta-dev/kubernetes-demos/refs/heads/main/pending_pods/pending_pod_resources.yaml
```

---
## More Scenarios ⭐⭐⭐
![bg right:35% 80%](https://assets.zyrosite.com/cdn-cgi/image/format=auto,fit=crop/YD0y4WNK2NF309oZ/10595761-YNqNDGrBpeU5ywZO.png)

---
### Preps
![bg right:35% 50%](https://raw.githubusercontent.com/kubernetes/community/690273e13778a52736ed5f2d83597319186f637a/icons/svg/infrastructure_components/unlabeled/control-plane.svg)
Please apply the preparation manifest before we can continue
```bash
kubectl apply -f https://raw.githubusercontent.com/codecap/cloud-workshops/refs/heads/main/examples/troubleshooting/000_preps.yaml
```

---
### Use Case: no running pods
![bg right:35% 50%](https://raw.githubusercontent.com/kubernetes/community/690273e13778a52736ed5f2d83597319186f637a/icons/svg/infrastructure_components/unlabeled/control-plane.svg)
Ensure that deployment pods are up and running
```bash
kubectl apply -f https://raw.githubusercontent.com/codecap/cloud-workshops/refs/heads/main/examples/troubleshooting/010_no-running-pods.yaml
```

---
### Use Case: wrong resource used
![bg right:35% 50%](https://raw.githubusercontent.com/kubernetes/community/690273e13778a52736ed5f2d83597319186f637a/icons/svg/infrastructure_components/unlabeled/control-plane.svg)
Ensure that deployment pods are up and running
```bash
kubectl apply -f https://raw.githubusercontent.com/codecap/cloud-workshops/refs/heads/main/examples/troubleshooting/020_wrong-resource-used.yaml
```

---
### Use Case: ConfigMap
![bg right:35% 50%](https://raw.githubusercontent.com/kubernetes/community/690273e13778a52736ed5f2d83597319186f637a/icons/svg/infrastructure_components/unlabeled/control-plane.svg)
Ensure that deployment pods are up and running
```bash
kubectl apply -f https://raw.githubusercontent.com/codecap/cloud-workshops/refs/heads/main/examples/troubleshooting/030_configmap.yaml
```

---
### PVC use case
![bg right:35% 50%](https://raw.githubusercontent.com/kubernetes/community/690273e13778a52736ed5f2d83597319186f637a/icons/svg/infrastructure_components/unlabeled/control-plane.svg)
Ensure that pvc is created, bound and deployment pods are up and running
```bash
kubectl apply -f https://raw.githubusercontent.com/codecap/cloud-workshops/refs/heads/main/examples/troubleshooting/040_pvc.yml
```

---
### Service configuration
![bg right:35% 50%](https://raw.githubusercontent.com/kubernetes/community/690273e13778a52736ed5f2d83597319186f637a/icons/svg/infrastructure_components/unlabeled/control-plane.svg)
Ensure connection can be established to the service via external und cluster IPs
```bash
kubectl apply -f https://raw.githubusercontent.com/codecap/cloud-workshops/refs/heads/main/examples/troubleshooting/050_service.yml
```

---
### Ingress configuration
![bg right:35% 50%](https://raw.githubusercontent.com/kubernetes/community/690273e13778a52736ed5f2d83597319186f637a/icons/svg/infrastructure_components/unlabeled/control-plane.svg)
Ensure connection can be established via ingress
```bash
kubectl apply -f https://raw.githubusercontent.com/codecap/cloud-workshops/refs/heads/main/examples/troubleshooting/060_ingress.yml
```


---
## More Scenarios ⭐⭐⭐
![bg right:35% 80%](https://assets.zyrosite.com/cdn-cgi/image/format=auto,fit=crop/YD0y4WNK2NF309oZ/10595761-YNqNDGrBpeU5ywZO.png)

---
### Troubleshoot Exercizes
![bg right:35% 50%](https://raw.githubusercontent.com/kubernetes/community/690273e13778a52736ed5f2d83597319186f637a/icons/svg/infrastructure_components/unlabeled/control-plane.svg)

07: Probes
09: Networking
10: Access via Ingress
11: Multiple Errors in App Deployment
12: Multi-service application fails to start
13: Application fails to start
14: Not routable traffic (☝) 2 Namespaces
15: Java web application fails to operate

```bash
git clone https://github.com/syseleven/kubernetes-debugging-workshop.git
cd kubernetes-debugging-workshop

EXERCIZE_NAME=[EXERCIZE]
cd $EXERCIZE_NAME
kubectl create ns $EXERCIZE_NAME
kubectl        ns $EXERCIZE_NAME
kubectl apply -f manifests
```
---
## Helm
![bg right:35% 50%](https://helm.sh/img/helm.svg)
Try to configure and install by means of helm one(or multiple) of the following applications
  - jenkins
  - quay
  - harbor
  - mariadb
  - postgresql
  - foreman
  - kgateway



---
## Debugging Hints
![bg right:35% 80%](https://assets.zyrosite.com/cdn-cgi/image/format=auto,fit=crop/YD0y4WNK2NF309oZ/10595761-YNqNDGrBpeU5ywZO.png)
```bash
# get logs
kubectl logs

# get current state
kubectl describe

# get current kubernetes events
kubectl events

# get the definition of youe resource, inspect status
kubectl get -oyaml

# start a debug pod in the same network namespace as the target one
kubectl debug

# when a pod fails to start, introduce an inifnit loop as "command" and inspect the application inside of the pod
...
  command: 
    - /bin/sh
  args:
    - -c
    - while sleep 10; do date; done
...

```
