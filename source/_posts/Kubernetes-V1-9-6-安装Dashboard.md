---
title: Kubernetes V1.9.6 安装Dashboard
date: 2018-12-14 23:19:59
tags: Kubernetes
categories: Technology
---

[TOC]

今天想为测试用的Kubernetes V1.9.6版本集群安装Dashboard，以便更直观的了解各种Kubernetes概念。然后参照网上的教程进行了操作，遇到了HTTPS证书的问题，始终打不开网页。经过Google了一番，终于找到了解决方案，在此记录一下。

# 安装Dashboard

## 1. 下载文件
  wget https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml

## 2. 修改配置文件
添加type: NodePort，暴露Dashboard服务。注意这里只添加行type: NodePort和nodePort: 30001即可，其他配置不用改，大概位置在末尾的Dashboard Service的spec中，参考如下。

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30001
  selector:
    k8s-app: kubernetes-dashboard
```

## 3. 下载镜像 (每个节点)

由于网络原因，配置文件中的k8s.gcr.io/kubernetes-dashboard-amd64:v1.8.3镜像无法下载，所以需要进行以下操作提前下载好。由于我使用了docker 代理，并且在代理机器上拨了翻墙VPN，所以可以访问到谷歌的镜像库下周k8s有关插件的镜像。

## 4. 安装dashboard
kubectl create -f kubernetes-dashboard.yaml

## 5. 授予Dashboard账户集群管理权限

编写配置文件

vim kubernetes-dashboard-admin.rbac.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-admin
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard-admin
  namespace: kube-system
```

创建账户管理

kubectl create -f kubernetes-dashboard-admin.rbac.yaml

## 6. 查看dashboard运行的node的IP
```
$ kubectl -n kube-system get pods -o wide|grep dashboard|awk '{print $7}'
```



#  排除NET::ERR_CERT_INVALID错误

可以通过之前设置的MasterIP：端口访问到dashboard，但是如果没有设置签名证书，而是用了原始的证书，会出现认证问题，即打开浏览器访问，会报告NET::ERR_CERT_INVALID错误，如下图：

```
{% asset_img  ERR_CERT_INVALID.png NET::ERR_CERT_INVALID%}
```



如访问提示了证书错误`NET::ERR_CERT_INVALID`，原因是由于物理机的浏览器证书不可用。我们可以生成一个私有证书或者使用公有证书，下面开始配置证书。

## 1.查看kubernetes-dashboard 容器跑在哪台node节点上，这里跑在docker-slave2上

```
root@docker-master1 pki]# kubectl get pod -n kube-system -o wide
NAME                                     READY   STATUS    RESTARTS   AGE     IP               NODE             NOMINATED NODE
coredns-576cbf47c7-l5wlh                 1/1     Running   1          9d      10.244.0.5       docker-master1   <none>
coredns-576cbf47c7-zrl66                 1/1     Running   1          9d      10.244.0.4       docker-master1   <none>
etcd-docker-master1                      1/1     Running   1          9d      192.168.20.210   docker-master1   <none>
kube-apiserver-docker-master1            1/1     Running   2          9d      192.168.20.210   docker-master1   <none>
kube-controller-manager-docker-master1   1/1     Running   2          9d      192.168.20.210   docker-master1   <none>
kube-flannel-ds-amd64-c7wz6              1/1     Running   0          9d      192.168.20.213   docker-slave1    <none>
kube-flannel-ds-amd64-hqvz9              1/1     Running   0          9d      192.168.20.214   docker-slave2    <none>
kube-flannel-ds-amd64-w7n4s              1/1     Running   2          9d      192.168.20.210   docker-master1   <none>
kube-proxy-8gj2w                         1/1     Running   1          9d      192.168.20.210   docker-master1   <none>
kube-proxy-mt6dk                         1/1     Running   0          9d      192.168.20.213   docker-slave1    <none>
kube-proxy-qtxz7                         1/1     Running   0          9d      192.168.20.214   docker-slave2    <none>
kube-scheduler-docker-master1            1/1     Running   2          9d      192.168.20.210   docker-master1   <none>
kubernetes-dashboard-5f864b6c5f-5s2rw    1/1     Running   0          5d21h   10.244.3.9       docker-slave2    <none>

```



## 2. 在docker-slave2节点上查看kubernetes-dashboard容器ID

root@docker-slave2 ~]# docker ps | grep dashboard
384d9dc0170b        registry.cn-hangzhou.aliyuncs.com/kube_containers/kubernetes-dashboard-amd64   "/dashboard --insecu…"   5 days ago          Up 44 hours                             k8s_kubernetes-dashboard_kubernetes-dashboard-5f864b6c5f-5s2rw_kube-system_94c8c50b-f484-11e8-80e8-000c29c3dca5_0

## 3.查看kubernetes-dashboard容器certs所挂载的宿主主机目录
```
[root@docker-slave2 ~]# docker inspect -f {{.Mounts}} 384d9dc0170b
"Mounts": [
            {
                "Type": "bind",
                "Source": "/var/lib/kubelet/pods/94c8c50b-f484-11e8-80e8-000c29c3dca5/volumes/kubernetes.io~empty-dir/tmp-volume",
                "Destination": "/tmp",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            },
            {
                "Type": "bind",
                "Source": "/var/lib/kubelet/pods/94c8c50b-f484-11e8-80e8-000c29c3dca5/volumes/kubernetes.io~secret/kubernetes-dashboard-token-tbctd",
                "Destination": "/var/run/secrets/kubernetes.io/serviceaccount",
                "Mode": "ro",
                "RW": false,
                "Propagation": "rprivate"
            },
            {
                "Type": "bind",
                "Source": "/var/lib/kubelet/pods/94c8c50b-f484-11e8-80e8-000c29c3dca5/etc-hosts",
                "Destination": "/etc/hosts",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            },
            {
                "Type": "bind",
                "Source": "/var/lib/kubelet/pods/94c8c50b-f484-11e8-80e8-000c29c3dca5/containers/kubernetes-dashboard/0e84c511",
                "Destination": "/dev/termination-log",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            },
            {
                "Type": "bind",
                "Source": "/var/lib/kubelet/pods/94c8c50b-f484-11e8-80e8-000c29c3dca5/volumes/kubernetes.io~secret/kubernetes-dashboard-certs",
                "Destination": "/certs",
                "Mode": "ro",
                "RW": false,
                "Propagation": "rprivate"
            }
        ],
```



## 4. 这里以私有证书配置，生成dashboard证书

```
openssl genrsa -des3 -passout pass:x -out dashboard.pass.key 2048
openssl rsa -passin pass:x -in dashboard.pass.key -out dashboard.key
openssl req -new -key dashboard.key -out dashboard.csr
openssl x509 -req -sha256 -days 365 -in dashboard.csr -signkey dashboard.key -out dashboard.crt
```

## 5.将生成的dashboard.crt  dashboard.key放到certs对应的宿主主机souce目录

```
scp dashboard.crt dashboard.key 192.168.20.214:/var/lib/kubelet/pods/94c8c50b-f484-11e8-80e8-000c29c3dca5/volumes/kubernetes.io~secret/kubernetes-dashboard-certs
```

## 6.重启kubernetes-dashboard容器

```
docker restart 384d9dc0170b
```

完成以上步骤即可访问kubernetes-dashboard web了，由于使用的是私有证书，所以还是会弹出不安全的连接，需要添加例外。

## 7. 登录

页面上有两种登录方式，这时我们使用token的方式登录。token的获取方式如下。

在master节点执行

```
$kubectl -n kube-system get secret | grep kubernetes-dashboard-admin|awk '{print "secret/"$1}'|xargs kubectl describe -n kube-system|grep token:|awk -F : '{print $2}'|xargs echo

eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbi10b2tlbi1qYm0ycCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6Ijg5ZmFiOGFmLTNjYzEtMTFlOC1iODQ4LTAwMGMyOTQwYWRiYSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTprdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiJ9.YS-ZklZ8fbkDp3tuOxFHyhiflXtCGDY0C5C3PYU1ot7YFCGA67_vDKY55OiE36sZNGNhWEmK52Yak7SrFZ75KwyMbM7TK69SGLftFiMedsUCfuUpBPB-Fc4beaxMuWWqVcHOs892VfE6I85xhhYLv_xD6t8x2DcJ1Cl6c5UVg_GBw13cSVaSA7asMpVuSj8MdOQcBNIUaRaxY04PDvZDWIN8Cqud9yDNkueFeuqP3DN_rN0FzLGg0Lqv3Q-fm4hKcIiiVi6E9J-i_T8QCsoKE36wEWg3hJdUTmzBufew2YrbPH4f0Aezq-OeKT8-x89vQwkbj1vttiVVtluTTX53T
```

上面获取到的就是token了，复制到登录页就可以登录了。