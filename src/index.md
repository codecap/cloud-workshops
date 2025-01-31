# Seeburger


## Workshop


[//]: # (TODO: about me)
[//]: # (TODO: about me)



## Relevance Table
```bash
| Network/SysAdmins      |   | Security |   | Devs |   | Operations(24/7) | Management |
| ---------------------- | - | -------- | - | ---- | - | ---------------- |----------- |
| container              | x | x        | x |      | x |                  |            |
| k8s-distros            |   |          | x |      | x |                  |            |
| k8s-fundamentals       | x | x        | x | x    | x |                  |            |
| k8s-cluster_management |   | x        |   | x    |   |                  |            |
| k8s-app_management     |   | x        | x |      |   |                  |            |
| k8s-networking         | x |          |   | x    |   |                  |            |
| k8s-storage            |   |          | x |      |   |                  |            |
| k8s-security           |   | x        |   |      |   |                  |            |
| k8s-observability      | ? |          |   | x    |   |                  |            |
| k8s-troubleshouting    | x |          | x | x    |   |                  |            |
```

---

## Requirements
- Linux VM with root access
- Access to Internet


---
## Content
Links:
- [Intro](intro.html)
- [Container](container.html)
- [K8S-Distros](k8s-distros.html)
- [K8S-Fundamentals](k8s-fundamentals.html)
- K8S-ClusterManagement
- KS8-AppManagement
- K8S-Networking
- K8S-Storage
- K8S-Security
- K8S-Observability
- K8S-Troubleshooting
----



```

## Plan
```yaml
k8s-distros:
    intro:
        - k8s vanilla
        - openshift / okd
        - k3s
        - minikube:
            source: https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Fx86-64%2Fstable%2Fbinary+download
        - microk8s
        - kind
    tasks:
        - create a kind cluster
        - review kube-system namespace
        - deploy a hello-world app
k8s-fundamentals:
    intro:
        - Kubernetes Resources
            - Namespaces
            - Pods
            - Services
            - Deployment / DaemonSet / Statefulset
            - Secrets
            - PersistantVolumeClaim
            - Ingress
            - Jobs / Cronjobs
            - NetworkPolicy
        - Kubernetes Architecture
        - Kubernetes API
        - Containers
        - Scheduling
        - access to cluster (kubectl, config, )
    tasks:
        - create a kind cluster
        - review ns, pods, services, deployments, daemonsets, pvc, ingress, jobs, network policy
        - deploy an app by using deployment, attach en EmptyDir volume, make the app available via service
        - deploy a DaemonSet
        - create job / cronjob
k8s-cluster_management:
    intro:
        - control plane
        - dataplane
        - high availability
        - networking (pods, services)
        - cni
        - updates/upgrades:
            podDisruptionBudget: https://livewyer.io/blog/2024/05/08/comparison-of-service-meshes/
        - kubeadm
        - Deploy Dashboard
        - growing the cluster, autoscaling
        - etcd backup and restore
        - crds
        - Operators
        - ServiceAccounts, SecurityContexts, Capabilities
    tasks:
        - deploy a cluster
        - review cni
        - update a cluster
        - add a worker node to the cluster
        - backup etcd
ks8-app_management:
    intro:
        - deployment
        - rolling update / rollback
        - liveness / readiness
        - requests, limits, quotas
        - scaling applications
        - access from inside and outside of the cluster
        - monitoring / logging / telemetry
        - deployment strategies (e.g. blue/green or canary)
        - templating(helm, kustomize)
    task:
        - Deploy an app with helm/customize.
        - use custom: image_tag, vhost
        - update / scale
        - deploy a db

k8s-networking:
    intro:
        - Host Level
        - CNI
        - Pods
        - Services
        - NodeIP
        - ClusterIP
        - LoadBalancer
        - NetworkPolicy
        - DNS
        - Ingress
        - MetalLB
    tasks:
        - deploy a hello-world app accessible wie Cluster IP (own namespace)
        - deploy a second hello-world app acciesble via Ingress (own namespace)
        - Access IPs via Ingress, Service, Pod, DNS
        - Implement a NetworkPolicy for one of the apps
        - Create a Headless Service for the second app
        - deploy an ExternalService

k8s-storage:
        intro:
            - CSI
            - StorageClass
            - PV / PVC
            - VollumeSnapshots
            - persistent and ephemeral volumes
        tasks:
            - deploy an App using persiant storage (nginx with an index.html)
            - change index.html, restart app, access index.html via service/ingress
            - deploy an App with ephemeral volume (EmptyDir)
            - deploy an App with ephemeral volume (Image)
            - deploy an App with local volume type (hostPath), adjust the deployment to make the app run on one single node
k8s-security:
    intro:
        - API Access
        - Authentication / Authorization
        - ceritifcates
        - RBAC:
            oidc: https://www.armosec.io/blog/kubernetes-user-management/
        - CIS Benchmark
        - Image Scanner
        - Pod Security standards
        - Pod Security Admission
        - Secrets:
            vault: https://blog.devops.dev/vault-integration-mechanisms-in-kubernetes-comparative-analysis-61e3f582e2f4
        - Service Mesh / Traffic Encryption
        - Ingress with TLS
        - Minimize base image footprint
        - sign and validate artifacts
        - Analyse user workloads and container images (Kubesec, KubeLinter)
        - Audit logs
        - Policy Agents
        - falco:
            desc: detection of abnormal behavior, potential security threats, and compliance violations
            source: https://medium.com/@omar.kamal.abouraya/how-i-used-falco-to-secure-my-kubernetes-cluster-without-touching-critical-pods-159ad4546890
    task:
        - run a security benchmark
        - create a certifacate
        - create an ingress with TLS
        - create a service account, assign rbac policy, verify
        - scan in image, review an image scan result in a registry
        - assign restricted policy to a namespace


k8s-observability:
    intro:
        - Monitoring
        - Logging
        - ServiceMesh
        - Telemetry
    tasks:
        - review logs ind k8s-system namespace
        - deploy prometheus, configure monitroing fron app
        - deploy a service mesh with an test app, review how traffic flows
        - deploy telemetry, review how 


k8s-troubleShooting:
    intro:
        - Networking
        - Pods can not start
        - Service / Labels
        - debug container
    tasks:
        - deploy and debug an app
```
