---
title: Kubectl 常用命令
date: 2018-12-17 13:26:24
tags: Kubernetes
categories: Technology
---

# 1.进入某个pod

如何进入kubernetes的一个pod呢，其实和进入docker的一个容器相似：

进入docker容器 ：

```
docker exec -ti  <your-container-name>   /bin/sh
```

进入pod：

```
kubectl exec -ti <your-pod-name>  -n <your-namespace>  -- /bin/sh
```

