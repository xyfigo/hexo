---
title: 裸机Kubernetes集群上运行Nginx Ingress的若干考虑
date: 2018-12-17 15:04:55
tags: Kubernetes Ingress
categories: Technology
---

在传统的云环境下，网络负载均衡是按需调用的，一个Kubernetes manifest可以提供一个单独的通信端点用于NGINX Ingress控制器与外部客户端通信。这样间接地可以与集群中的任何应用进行通信。裸机Kubernetes集群环境确实这一服务组件，所以需要进行一些不同的设置才能为外部用户提供类似的访问。

本文描述了在裸机Kubernetes环境上部署NGINX Ingress控制器的推荐的配置。

# MetalLB：一种纯软件的解决方案

# NodePort 服务方式

由于这种方式简单，所以这也是《安装指南》中用户默认部署的方式。

在这个配置下，NGINX容器保持与主机网络的隔离。因此它可以安全的绑定至任何的端口，包括标准的HTTP80端口和443端口。但是由于容器名称空间的隔离，集群网络外的客户端（例如公网用户）无法直接访问Ingress主机的80和443端口。所以外部客户端必须在HTTP请求后面加上分配给ingre-nginx服务的NodePort端口号。

{%asset_img  nodeport.jpg NodePort%}

### 示例

假设NodePort30100分配给ingress-nginx服务。

```
$ kubectl -n ingress-nginx get svc
NAME                   TYPE        CLUSTER-IP     PORT(S)
default-http-backend   ClusterIP   10.0.64.249    80/TCP
ingress-nginx          NodePort    10.0.220.217   80:30100/TCP,443:30101/TCP
```

Kubernetes节点的公共IP地址是203.0.113.2（下面的external IP只做个示例，在大多数裸机环境中这个值一般为<Node>）

```
$ kubectl describe node
NAME     STATUS   ROLES    EXTERNAL-IP
host-1   Ready    master   203.0.113.1
host-2   Ready    node     203.0.113.2
host-3   Ready    node     203.0.113.3
```

客户端可以通过myapp.example.com主机的30100端口来访问Ingress，在这里myapp.example.com域名会解析为203.0.113.2。

### 对主机系统的影响

虽然使用--service-node-port-range API服务标志重新配置NodePort范围，并且可以包括非特权端口（例如80和443）端口，这样听起来很有吸引力。但是这样做会导致非预料的情况和问题，包括并不限于使用了其他系统守护进程占用的端口，或授予kube-proxy某些它并不必须的特权。

因此并不推荐使用这一功能，请使用本文介绍的其他替代的做法。

NodePort方式有其他几个需要注意的限制：

- ## Source IP address

  NodePort类型的服务默认执行原地址转换。这意味着HTTP请求的源IP总是接受NGINX请求的kubernetes节点的IP地址。

  在NodePort设置中避免源IP的推荐方式是在ingress-nginx服务spec中设置externalTrafficPolicy的属性值为Local

  ### Warning 

  这个设置有效地识别和扔掉发送到没有运行NGINX Ingress控制器实例的Kubernetes节点。为了控制NGINX ingress控制器能够在哪些节点上进行调度，应当考虑分配NGINX Pods到特定节点。

  ### 示例

  在Kubernetes集群中包括3个节点（external IP作为示例显示，大部分裸机环境中这个字段指一般为<None>）

  ```
  $ kubectl describe node
  NAME     STATUS   ROLES    EXTERNAL-IP
  host-1   Ready    master   203.0.113.1
  host-2   Ready    node     203.0.113.2
  host-3   Ready    node     203.0.113.3
  ```

  ngingx-ingress-controller部署包括2个复制集

  ```
  $ kubectl -n ingress-nginx get pod -o wide
  NAME                                       READY   STATUS    IP           NODE
  default-http-backend-7c5bc89cc9-p86md      1/1     Running   172.17.1.1   host-2
  nginx-ingress-controller-cf9ff8c96-8vvf8   1/1     Running   172.17.0.3   host-3
  nginx-ingress-controller-cf9ff8c96-pxsds   1/1     Running   172.17.1.4   host-2
  ```

  发送到host-2和host-3的请求能够被转发到NGINX并且源客户端IP能够被保留下来，发送到host-1的请求将被丢弃，因为在这个节点上没有NGINX复制集。

- ## Ingress 状态

  因为NodePort服务不能获得定义中分配的LoadBalancerIP，NGINX Ingress控制器不能更新它管理的Ingress对象的状态。

  ```
  $ kubectl get ingress
  NAME           HOSTS               ADDRESS   PORTS
  test-ingress   myapp.example.com             80
  ```

  尽管事实上没有为NGINX ingress提供公共IP地址的负载均衡器，仍然能够强制更新所有被管理的Ingress对象，方法是设置ingress-nginx服务中的externalIPs字段。

  ### Warning

  设置externalIPs并不仅仅使NGINX ingress控制器更新ingress对象的状态。请在官方Kubernetes文档的Service页面阅读这个选项，以及关于ExternalIPs的章节获取更多信息。

  ### 示例

  在Kubernetes集群中包括3个节点（external IP作为示例显示，大部分裸机环境中这个字段指一般为< Node >）

  ```
  $ kubectl describe node
  NAME     STATUS   ROLES    EXTERNAL-IP
  host-1   Ready    master   203.0.113.1
  host-2   Ready    node     203.0.113.2
  host-3   Ready    node     203.0.113.3
  ```

  可以编辑ingress-nginx服务并添加一下的字段至spec对象：

  ```
  spec:
    externalIPs:
    - 203.0.113.1
    - 203.0.113.2
    - 203.0.113.3
  ```

  设置完毕后Ingress对象可以显示出以下属性：

  ```
  $ kubectl get ingress -o wide
  NAME           HOSTS               ADDRESS                               PORTS
  test-ingress   myapp.example.com   203.0.113.1,203.0.113.2,203.0.113.3   80
  ```

- ##  Redirects

  因为NGINX不知道NodePort Service执行的端口转发，后端应用程序应当负责生成重定向URLs，并将外部客户端使用的URL和端口考虑进去。

  ### 示例

  在不使用NodePort的情况下，通过NGINX生成重定向，例如HTTP到HTTPS的重定向或者domain到www.domain 。

  ```
  $ curl -D- http://myapp.example.com:30100`
  HTTP/1.1 308 Permanent Redirect
  Server: nginx/1.15.2
  Location: https://myapp.example.com/  #-> missing NodePort in HTTPS redirect
  ```



