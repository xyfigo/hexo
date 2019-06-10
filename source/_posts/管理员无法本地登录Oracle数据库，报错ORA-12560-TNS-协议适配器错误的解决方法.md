---
title: '管理员无法本地登录Oracle数据库，报错ORA-12560: TNS: 协议适配器错误的解决方法'
date: 2018-12-13 15:55:31
tags: Oracle 运维
categories: Technology
---

今日财务报销平台出现了数据源连接不上，查找问题后发现数据库故障。

当使用管理员本地登录数据库进行管理的时候，出现了报错：

```plsql
C:\Users\Administrator>sqlplus / as sysdba
ORA-12560: TNS: 协议适配器错误的解决方法
```

貌似是由于监听连接数过高，无法进行连接响应了。故重启了服务器后故障恢复。