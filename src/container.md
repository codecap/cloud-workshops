---
title:       Container
description: Workshop and Tasks
author:      Vladislav Nazarenko (vnazarenko@📯socket.de)
keywords:    Container,runtimes,docker,podman
url:         
image:
transition: cover
theme: default
backgroundImage: url(https://codecap.github.io/cloud-workshops/assets/background.jpg)
paginate: true
---

# Container
![bg left:40% 80%](https://raw.githubusercontent.com/kubernetes/community/322066e7dba7c5043071392fec427a57f8660734/icons/svg/resources/unlabeled/pod.svg)

---

# What is a container?
* filesystem (image)
* resources (cpu/ram)
* capabilities / permissions
* network namespace
* running process

---

# How containers are different from VMs?
![bg right:40%  100%](https://download-hk.huawei.com/mdl/image/download?uuid=4a95b0f6a4de4ebb9c9caa1dfa0ab0a9)
* (normally) one running process
* can (co)exist on VM/Physical Host
* shares the same kernel with the host

---

# How containers are different from VMs?
![bg right:40% 100%](https://download-hk.huawei.com/mdl/image/download?uuid=43e73800cb8b482989764958bd7e7825)

---

# Which kind of containers do you know?
* docker
* podman
* lxc

---

# Why containers?
* build -> ship -> run
* simplify application delivery
* solve dependency problems

---

# Docker / containerd
![bg right:50% 70%](https://speedmedia.jfrog.com/08612fe1-9391-4cf3-ac1a-6dd49c36b276/media.jfrog.com/wp-content/uploads/2021/05/31004836/Containerd-Docker-Registry.png)

---

# Container Runtime
- [k8s doc](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#container-runtimes)
- Docker / containerd
- Podman / runc
- [crio-o](https://cri-o.io/)
- [kata](https://katacontainers.io/)

---

# Kata
![bg right:60% 80%](https://katacontainers.io/static/589e3d905652847b22c395fe6bbbace7/663f4/katacontainers_architecture_diagram.jpg)

---

# Docker vs Podman

* containerd vs runc
* root or rootless

---

# Docker vs Podman

* Dockerfile vs Containerfile
* Dockerfile is native to the Docker ecosystem
* According to CNFC naming - Containerfile
* The syntax is identical

---

# Install a Container Runtime
- [Docker on ubuntu](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)
- [Docker on RHEL](https://docs.docker.com/engine/install/rhel/#install-using-the-repository)
- Podman
```bash
# ubuntu
apt install -y podman

# rhel
dnf install -y podman
```

---

# Images

---

# Build
```bash
# create a Dockerfile/Containerfile
cat > Dockerfile <<EOF
FROM nginx
EOF

# try to build
docker build .
podman build . # wont work
```

---

# Image names
```bash
|  nginx ->  | docker.io/library/nginx:latest  | 
|  shortcut  | [REGISTRY/ [NS] / [IMAGE]:[TAG] |
```
```bash
cat > Containerfile <<EOF
FROM docker.io/library/nginx:1.27.3
ADD Containerfile /tmp
EOF
```
```bash
# build
podman build --tag myreigstry.local/test/mynginx:0.0.1 ./
# start
podman run --rm --name nginx --detach myreigstry.local/test/mynginx:0.0.1
```

---

# Build Instructions
```bash
ADD         - Add local or remote files and directories.
ARG         - Use build-time variables.
CMD         - Specify default commands.
COPY        - Copy files and directories.
ENTRYPOINT  - Specify default executable.
ENV         - Set environment variables.
FROM        - Create a new build stage from a base image.
HEALTHCHECK - Check a container's health on startup.
RUN         - Execute build commands.
SHELL       - Set the default shell of an image.
USER        - Set user and group ID.
WORKDIR     - Change working directory.
```
---
# Add more into image
```bash
FROM docker.io/library/nginx:1.27.3
    
WORKDIR /usr/share/nginx/html/

RUN  /usr/bin/date > /usr/share/nginx/html/index.html
COPY Containerfile /usr/share/nginx/html/
```

---

# Test changes
```bash
# build
podman build --tag myreigstry.local/test/mynginx:0.0.2 ./

# run
podman run --rm --name nginx --detach myreigstry.local/test/mynginx:0.0.2

# test
podman  exec nginx curl -sS localhost
podman  exec nginx curl -sS localhost/Containerfile
```

---

# docker/podman commands
```bash
podman image ls      ...
podman image history ...
podman image rm      ...
podman image save    ...
podman image load    ...
podman run           ...
podman stop          ...
podman rm            ...
podman start         ...
podman logs          ...
podman exec          ...
podman inspect       ...
```

---

# Image layers
![bg right:40% 70%](https://miro.medium.com/v2/resize:fit:720/format:webp/0*HoxAZTZ2b2C7AM--)

* Each instruction in Containerfile introduces a new layer in the image
* The more builds/instructions, the bigger image gets

---

# Best practices building images
* Use trusted base images
* Keep image small, minimize number of layers
* Clean-up after installation (dnf, apt)
* Use Multi-Stage Builds to produce smaller images
* Use .dockerignore / .containerignore
* Specify user and workdir
* Avoid secrets or configuration data
* Sign your images
* No need for ssh
* Build images for single process/service

---

# Access and Data
* (--)publish port or run in network host
* Use existing directory or (--)volume to access data within container

---

# Registries

* [docker hub](https://hub.docker.com)
* [quay](https://quay.io/)
* [amazon](https://gallery.ecr.aws)
* azure, gcr, ghcr ...
* private registries (Harbor, Quay, JFrog, Nexus ...)

---

# Run your own registry
```bash
podman run --detach --publish 5000:5000 --restart always \
  --name registry                                        \
  --volume registry:/var/lib/registry                    \
  registry:2
```

```bash
# push
podman image push myreigstry.local:5000/test/mynginx:0.0.2 --tls-verify=false
# search
podman image search localhost:5000/ --tls-verify=false
# pull
podman image pull myreigstry.local:5000/test/mynginx:0.0.2 --tls-verify=false
```
---

# Multi-Stage build

```bash
FROM golang:1.16-buster as builder

# compile var01
ADD https://raw.githubusercontent.com/minhaz1217/linux-configurations/refs/heads/master/docker/basic_go_api/src/main.go ./src/
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o /basic-go-api ./src/main.go

# compile var02
# RUN    git clone https://github.com/minhaz1217/linux-configurations.git \
#     && cp linux-configurations/docker/basic_go_api/src/main.go src/     \
#     && CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o /$APP_NAME ./src/main.go

# Then run stage
FROM rockylinux:9
# FROM debian:stable-slim
# FROM scratch
COPY --from=builder /basic-go-api .
CMD ["./basic-go-api"]
```

---

# Distroless Images
![](https://github.com/distroless/.github/raw/main/profile/distroless-logo.svg)
![bg right:40% 70%](https://www.upwind.io/wp-content/uploads/2024/01/Application-Container-image-Comparison-1024x737.png)

---

# docker compose
[Review examples](https://github.com/docker/awesome-compose)

Try
```bash
git clone https://github.com/docker/awesome-compose.git
cd awesome-compose/wordpress-mysql/
docker compose up
```
---

# podman play
```bash
SRC=https://raw.githubusercontent.com/codecap/cloud-workshops/refs/heads/main/src/container/podman/wordpress.yml
# review
curl $SRC
# start
podman play kube $SRC
# visit the page with your browser or use curl
curl -D- [YOUR_IP]:8080
# stop
podman play kube --down $SRC
```

---

# Tools
- [skopeo](https://github.com/containers/skopeo)
- [crane](https://github.com/google/go-containerregistry/blob/main/cmd/crane/doc/crane.md)
- [trivy](https://trivy.dev/latest/docs/)
- [dive](https://github.com/wagoodman/dive)
- [buildah](https://github.com/containers/buildah/blob/main/docs/tutorials/03-on-build.md)
- [hadolint](https://github.com/hadolint/hadolint)
- [curl](https://curl.se/)

---

# Tasks:
- Build an image
- Set labels on images, maintainer
- Set user, workdir
- Add hosts when building
- Try to run in --network=host
- Set an ARG in Containerfile
- Run a container from image, ports, volumes
- Build an Image as Multi-Stage build
- Create a distroless Image / Image with only one binary inside
- Push/pull images to / from a registry
- Pull a config file from a git repo, when starting container
- Run a container as system service
- Try to run multiple containers with docker-compose
- Try to run multiple containers with kube play (podman)

---

# Links
- [CNCF landscape](https://landscape.cncf.io/)
- [Dockerfile Reference](https://docs.docker.com/reference/dockerfile/)
- [Containerfile Reference](https://github.com/containers/common/blob/main/docs/Containerfile.5.md)
- [docker/awesome-compose](https://github.com/docker/awesome-compose)

---
# More Topics to review
- Multi Stage builds
- Distroless images
- docker-compose
- podman k8s manifest
