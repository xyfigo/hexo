---
title: 开发常用中间件、数据库的docker快速启动
date: 2019-04-10 21:18:55
tags: Docker
categories: Technology
---

[TOC]

# MySQL

```shell
docker run --name=orders-mysqldb -d -p 3306:3306 -e MYSQL_ROOT_HOST=% -e MYSQL_ROOT_PASSWORD=root -e MYSQL_USER=orders -e MYSQL_PASSWORD=orders -e MYSQL_DATABASE=orders mysql/mysql-server:5.7.24```
```

# Redis

无密码：

```shell
docker run --name some-redis -p 6379:6379 -d redis redis-server --appendonly yes
```

带密码：
```shell
docker run --name some-redis -p 6379:6379 -d redis redis-server --appendonly yes --requirepass "mypassword"
```