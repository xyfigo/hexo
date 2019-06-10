---
title: 使用Minio为NAS设置S3兼容的对象存储接口
date: 2019-01-28 09:05:11
tags: Minio
Categories: Technology
---

Minio 网关可以为NAS存储增加Amazon S3兼容的接口。可以同时运行多个minio实例指向同一个共享的NAS卷，作为分布式的对象网关。

# Run Minio Gateway for NAS Storage

## 使用Docker

将/shared/nasvol替换为挂载NAS的真实路径。

```shell
docker run -p 9000:9000 --name nas-s3 \
 -e "MINIO_ACCESS_KEY=minio" \
 -e "MINIO_SECRET_KEY=minio123" \
 -v /shared/nasvol:/container/vol \
 minio/minio gateway nas /container/vol
```

## 使用二进制安装

```shell
export MINIO_ACCESS_KEY=minio
export MINIO_SECRET_KEY=minio123
minio gateway nas /shared/nasvol
```

# Test using Minio Browser

Minio 网关有一个内嵌的对象浏览Web页面，地址是[http://127.0.0.1:9000](http://127.0.0.1:9000/) ，如果Minio服务正常启动将会看到如下页面：

{%asset_img minio-browser-gateway.png minio-browser-gateway%}

# Test using Minio Client `mc`

mc 提供了Unix命令行与Minio Server进行交互，例如：ls, cat, cp, mirror, diff等。它支持文件系统和S3兼容的云存储服务。

## 配置mc

```shell
mc config host add mynas http://gateway-ip:9000 access_key secret_key
```

## 列出NAS上的buckets

```shell
mc ls mynas
[2017-02-22 01:50:43 PST]     0B ferenginar/
[2017-02-26 21:43:51 PST]     0B my-bucket/
[2017-02-26 22:10:11 PST]     0B test-bucket1/
```



