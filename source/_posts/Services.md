---
title: Services
date: 2018-06-12 14:42:38
tags: Kubenetes
categories: Technology
---

![forest](https://blog.zhangruipeng.me/hexo-theme-minos/gallery/forest.jpg)

Kubenetes Pods 不是永生的。当他们生命结束后无法复活。实际上[ReplicationController](https://v1-9.docs.kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)动态的创建和销毁Pods（例如，应用扩展、停用或者回滚更新时）。由于每个Pod拥有各自的Ip地址，所以这些IP地址无法被固定下来。这导致一个问题：如果某些Pods（所谓后端）在kubenetes集群中向其他Pods提供了某些功能，这些前端是怎么样找到并记录与哪一个后端关联的呢？**通过Services**

Kubenetes Service是一组有内在逻辑Pods的抽象，并且定义了如何访问他们的策略，有时称为微服务。一组成为Service的Pods通常使用[Label Selector](https://v1-9.docs.kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors)进行定义。

举个例子，假如一个图像处理后端程序运行3个副本。这些副本是可替代的，前端不关心使用哪一个副本。而实际上组成后端的Pods会改变，前端业务无法知道和跟踪后端的列表。Services抽象使这种解耦变得可能。

对于Kubenetes原生的应用，kubenetes提供简单的Endpoints API，每当服务中的Pods改变的时候都会进行更新。对于非原生的应用，kubenetes提供虚拟Ip的网桥，转至后端的Pods。

# 定义服务

Kubenetes服务是一个REST对象，与Pod类似。像所有的REST对象一样，Service 定义能够被提交至apiserver用来创建新的实例。例如，有一组具有“app=MyApp”标签的Pods，对外暴露9376端口。
{% codeblock lang:yaml %}
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec: 
  selector:
​    app: MyApp
  ports:
  - protocol:TCP
      port: 80
      targetPort:9376
{% endcodeblock %}

这个配置将创建一个新的服务对象，命名为“my-service"，在任何标签是“app=MyApp”的Pod上映射目标端口是9376。这个服务会被分配一个IP地址（有时被称为“cluster IP”），改地址被服务代理使用。这个服务的选择器会持续不断地进行检索，结果会被提交到一样被命名为“my-service"的Endpoints对象。

注意，一个服务能够映射一个入栈端口到一个出栈端口。默认地，目标端口会被设置与上面“port”字段一样的值。更有意思的是，目标端口能够被设置成为一个字符串，引用在后端Pods中一样的端口名。在每个后端pod中，被分配给那个名称的实际端口可以是不同的。这给部署和演化服务提供了很大的灵活性。例如，你可以在下一个后端软件版本中改变pods暴露的端口，而对前端没有影响。

kubenetes服务支持TCP和UDP协议。默认是TCP。
## 没有选择器的服务##

服务通常是对kubenetes Pods的访问抽象，但是他们也可以抽象其他类型的后端。例如：

- 在生产环境中使用外部数据库，而在测试环境中使用自己的数据库。

- 在另一个命名空间或者另一个集群中将一个服务咨询另一个服务。

- 将一部分负载迁移到kubenetes中，而其中一些负载在kubenetes外的后端。

在上面的情景下，你可以定义一个没有选择器的服务：

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
```

因为这个服务没有选择器，相关联的Endpoints对象是不会被创建。你可以手动的映射服务到自己的特定endpoints：

```yaml
kind: Endpoints
apiVersion: v1
metadata:
  name: my-service
subsets:
  - addresses:
      - ip: 1.2.3.4
    ports:
      - port: 9376
```

注意：Endpoint IP不能设置为回环地址 (127.0.0.0/8) ,本地连接地址(169.254.0.0/16) ，或者本地广播地址(224.0.0.0/24)。

访问一个没有选择器的服务同有选择器的服务一样。流量会被路由至用户定义的endpoints（在上面的例子中是`1.2.3.4:9376`）

一个ExternalName服务是另一类没有选择器的特殊服务。它不定义任何端口和endpoints。它提供了一种向集群外的外部服务返回别名的方式。

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
```

当查找my-service.prod.svc.CLUSTER主机时，集群DNS服务会返回一个值为my.database.example.com的CNAME记录。访问这个服务的方式跟访问其他服务一样，仅有的不同只是在DNS级别进行重定向，没有进行代理和转发。如果以后决定将数据库迁移到集群中，可以启动pods，添加适当的选择器和端点并改变服务的类型。

# 虚拟IP和服务代理

kubenetes中的每一个节点都会运行kube-proxy。kube-proxy负责实现一种ExternalName以外的虚拟IP形式。在kubenetes1.0版本中，服务是4层的（IP上的TCP/UDP）结构，代理纯粹在用户空间。在kubenetes1.1中，添加了Ingress API表示7层（HTTP）服务，也添加了iptables 代理，并从1.2版本后成为默认的操作模式。在kubenetes1.8-beta.0中，增加了ipvs proxy。

## Proxy-mode：userspace

在这个模式下，kube-proxy观察kubenetes master添加和删除Service和endpoint对象。对于每一个服务它都会在本地节点打开一个端口（任意选择）。任意与代理端口的连接都会被代理到其中一个服务后端Pods（在endpoints中报告的）。使用哪一个后端pod是基于服务的SessionAffinity决定。最终它会安装iptables规则，这个规则捕获到服务cluster IP（虚拟IP）和端口的流量，并转发流量到代理后端pod的代理端口。默认地，后端选择使用轮询方式。

![services-userspace-overview ](https://v1-9.docs.kubernetes.io/images/docs/services-userspace-overview.svg)

注意，在上图中clusterIP被写成ServiceIP

