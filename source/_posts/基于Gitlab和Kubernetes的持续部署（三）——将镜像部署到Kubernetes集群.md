---
title: 基于Gitlab和Kubernetes的持续部署（三）——将镜像部署到Kubernetes集群
date: 2018-11-09 09:37:54
tags: Gitlab, kubenetes
categories: Technology
---

本文将使用Gitlab CI的Kubenetes集群特性将前面构建的容器镜像部署到Kubernetes。

#  基本环境

- kubernetes集群
- Gitlab实例
  - 开启Gitlab Container Registry
  - 配置并启用Gitlab runner：CI runner必须可与Kubernetes APIserver通信。
- Kubectl配置Kubernetes集群访问
- Kubernetes ServiceAccount
  - 有特殊的权限。

# Setp1. 从Kubernetes获取ServiceAccount Token

Kubernetes1.6及以上版本增加了基于角色的访问控制（RBAC），ServiceAccount需要正确的权限来部署你想要的名称空间。使用kubeadm初始化的1.6及以上版本的Kubernetes集群，已经默认为API Server开启了RBAC，可以查看Master Node上API Server的静态Pod定义文件：

```shell
[root@Kubernetes ~]# cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep RBAC
    - --authorization-mode=Node,RBAC
```

Kubernetes1.5及以下版本只需要创建一个ServiceAccount或者使用一个选定名称空间中已存在的ServiceAccount。

1. 首先考虑为部署到kubernetes里的Gitlab项目创建一个专门的名称空间：

   ```shell
   [root@Kubernetes ~]# kubectl get namespace
   NAME          STATUS    AGE
   default       Active    164d
   kube-public   Active    164d
   kube-system   Active    164d
   [root@Kubernetes ~]# kubectl create namespace devops
   namespace "devops" created
   [root@Kubernetes pki]# kubectl get namespaces
   NAME          STATUS    AGE
   default       Active    161d
   devops        Active    14s
   kube-public   Active    161d
   kube-system   Active    161d
   ```

2. 获取DevOps名称空间下默认的ServiceAccount Token

   ```shell
   [root@Kubernetes pki]# kubectl get -n devops secret
   NAME                  TYPE                                  DATA      AGE
   default-token-ph7sj   kubernetes.io/service-account-token   3         2m
   [root@Kubernetes pki]# kubectl get -n devops secret default-token-ph7sj -o yaml
   apiVersion: v1
   data:
     ca.crt: [REDACATED]
     namespace: [REDACATED]
     token: [THIS IS YOUR TOKEN BASE64 ENCODED]
   kind: Secret
   metadata:
     annotations:
       kubernetes.io/service-account.name: default
     [...]
     name: default-token-nmx1q
     namespace: presentation-gitlab-k8s
     [...]
   ```

3. 因为Token是进行了base64编码的，索引需要对Token进行base64解码

   ```shell
   $ echo YOUR_TOKEN_HERE | base64 -d
   YOUR_DECODED_TOKEN
   ```

# Setp2. 获取Kubernetes的CA授权

Kubernetes的授权在文件夹/etc/kubernetes/pki目录（我的是V1.9版本）

```shell
$ cat /etc/kubernetes/pki/ca.crt
-----BEGIN CERTIFICATE-----
[REDACATED]
-----END CERTIFICATE-----
```

# Step3. 在Gitlab CI里创建Kubernetes集群

需要ServiceAccount Token（step1.）、CA 授权（step2.）、Kubernetes API地址和需要运行应用的名称空间。在Gitlab中选择CI/CD->Kubernetes，进入如下页面进行设置：

{% asset_img gitlab-ci-kubernetes-cluster-add-existing-cluster.png %}

其中Kubernetes Cluster name可以任意设置，Project namespace是可选项，此处设置为你需要创建应用的Kubernetes名称空间，我使用的名称空间是DevOps。

# Step4. 为项目添加.gitlab-ci.yml

我的项目是一个简单的Spring-boot Java项目，使用Maven进行构建，比较简单。此处贴出.gitlab-ci.yml

```yaml
image: docker:latest
services:
  - docker:dind

variables:
  DOCKER_DRIVER: overlay
  SPRING_PROFILES_ACTIVE: gitlab-ci

stages:
  - build
  - package
  - deploy

maven-build:
  image: maven:3-jdk-8
  stage: build
  script: "mvn clean package -B -Dmaven.test.skip=true"
  artifacts:
    paths:
      - target/*.jar

docker-build:
  stage: package
  script:
  - docker info
  - docker build -t registry.gitlab.nasg.gov.cn/${CI_PROJECT_PATH}:latest .
  - docker tag registry.gitlab.nasg.gov.cn/${CI_PROJECT_PATH}:latest registry.gitlab.nasg.gov.cn/${CI_PROJECT_PATH}:${CI_COMMIT_REF_NAME}
  - docker login -u gitlab-ci-token -p $CI_BUILD_TOKEN registry.gitlab.nasg.gov.cn
  - test ! -z "${CI_COMMIT_TAG}" && docker push registry.zerbytes.net/${CI_PROJECT_PATH}:latest
  - docker push registry.gitlab.nasg.gov.cn/${CI_PROJECT_PATH}:${CI_COMMIT_REF_NAME}

k8s-deploy:
  image: lachlanevenson/k8s-kubectl:latest
  stage: deploy
  environment:
      name: live
      url: https://live-presentation-gitlab-k8s.edenmal.net
  script:
  - kubectl version
  - echo "${KUBE_CA_PEM}" > kube_ca.pem
  - kubectl config set-cluster kubernetes --server=${KUBE_URL} --certificate-authority=kube_ca.pem
  - kubectl config set-credentials kube-admin --token=${KUBE_TOKEN}
  - kubectl config set-context kubernetes-system --cluster=kubernetes --user=kube-admin
  - kubectl config use-context kubernetes-system
  - kubectl cluster-info
  - kubectl create -f deployment.yml -n devops
#  - gcloud auth activate-service-account --key-file key.json
#  - gcloud config set compute/zone europe-west1-c
#  - gcloud config set project actuator-sample
#  - gcloud config set container/use_client_certificate True
#  - gcloud container clusters get-credentials actuator-sample
#  - kubectl delete secret registry.gitlab.com
#  - kubectl create secret docker-registry registry.gitlab.com --docker-server=https://registry.gitlab.com --docker-username=marcolenzo --docker-password=$REGISTRY_PASSWD --docker-email=lenzo.marco@gmail.com
#  - kubectl apply -f deployment.yml
```

# Step 5. 在Kubernetes中增加Docker login 信息

为了能够从GitLab registry（GitLab自带的私有镜像仓库功能）下载部署所需的镜像，必须在Kubernetes中增加一个Secret元素，用它访问GitLab Registry的Docker login信息。下面的命令必须在安装有Kubectl的节点中运行。

```shell
# YOUR_SECRET_NAME for example "registry-example-gitlab-key"
$ kubectl create \
    -n devops \
    secret docker-registry regsecret \
    --docker-server=registry.gitlab.nasg.gov.cn \
    --docker-username=xiaof \
    --docker-password=******* \
    --docker-email=xiaof@sbsm.gov.cn
```

