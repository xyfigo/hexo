---
title: Mvnw 简介
date: 2018-11-07 21:24:29
tags: Mvnw
categories: Technology
---

# 背景

maven是一款非常流行的java项目构建软件，它集项目的依赖管理、测试用例运行、打包、构件管理于一身，是我们工作的好帮手，maven飞速发展，它的发行版本也越来越多，如果我们的项目是基于maven构件的，那么如何保证拿到我们项目源码的同事的maven版本和我们开发时的版本一致呢，可能你认为很简单，一个公司嘛，规定所有的同事都用一maven版本不就万事大吉了吗？一个组织内部这是可行的，要是你开源了一个项目呢？如何保证你使用的maven的版本和下载你源码的人的maven的版本一致呢，这时候mvnw就大显身手了。
mvnw 全名是maven wrapper,它的原理是在maven-wrapper.properties文件中记录你要使用的maven版本，当用户执行mvnw clean 命令时，发现当前用户的maven版本和期望的版本不一致，那么就下载期望的版本，然后用期望的版本来执行mvn命令，比如刚才的mvn clean。
为项目添加mvnw支持很简单，有两种方式：

# 方法一，在Pom.Xml中添加Plugin声明：

```
<plugin>
    <groupId>com.rimerosolutions.maven.plugins</groupId>
    <artifactId>wrapper-maven-plugin</artifactId>
    <version>0.0.4</version>
</plugin>
```

这样当我们执行mvn wrapper:wrapper 时，会帮我们生成mvnw.bat, mvnw, maven/maven-wrapper.jar, maven/maven-wrapper.properties这些文件。
然后我们就可以使用mvnw代替mvn命令执行所有的maven命令，比如mvnw clean package

# 方法二，直接执行Goal

```
#表示我们期望使用的maven的版本为3.3.3
mvn -N io.takari:maven:wrapper -Dmaven=3.3.3
```

产生的内容和第一种方式是一样的，只是目录结构不一样，maven-wrapper.jar和maven-wrapper.properties在".mvn/wrapper"目录下

# 使用的注意事项

1、由于我们使用了新的maven ,如果你的settings.xml没有放在当前用户下的.m2目录下，那么执行mvnw时不会去读取你原来的settings.xml文件
2、在mvnw.bat中有如下的一段脚本
if exist "%M2_HOME%\bin\mvn.cmd" goto init
意思是如果找到mvn.cmd就执行初始化操作，但是maven早期版本不叫mvn.cmd,而是叫mvn.bat,所以会报"Error: M2_HOME is set to an invalid directory"错误，改成你本地的maven的匹配后缀就好了。