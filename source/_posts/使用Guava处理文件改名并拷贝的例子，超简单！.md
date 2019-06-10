---
title: 使用Guava处理文件改名并拷贝的例子，超简单！
date: 2018-10-26 11:05:28
tags: Guava
categories: Technology
---

今天需要对原有档案文件进行一个处理：需要把原文件名中包括“DD”或“KJ”的pdf文档重新改名为“WS”，并复制到另一个文件夹。如果使用Java文件IO去处理比较繁琐，自己还需要处理Steam等很多接口。这时想到了Java库中的神器——谷歌的基础库Guava，能够提供很多便利的API给用户，可以轻而易举地进行文件操作和字符串处理等，这里把代码记录下来，有空闲可以发几篇Guava的使用。

这里有几点需要额外注意：

1. 文件夹路径的字符串表示，Windows中的路径在字符串双引号中要进行转义\\\\
2. 寻找路径下的文件，可以使用 Files.fileTreeTraverser().breadthFirstTraversal(rootDir)方法。
3. 使用public String replaceAll(String regex, String replacement)进行字符串的匹配和替换。

```java
package org.sbsm;

import com.google.common.base.CharMatcher;
import com.google.common.io.Files;

import java.io.File;
import java.io.IOException;

/**
 * Hello world!
 *
 */
public class App
{
    public static void main( String[] args )
    {
        String sourcePath = "F:\\changeFileName\\";
        String targetPath = "F:\\new\\";
        int count=0;

        File rootDir = new File(sourcePath);
            for(File f: Files.fileTreeTraverser().breadthFirstTraversal(rootDir)){
                if(f.isFile()){
                    String fName=f.getName();
                    String newName=fName.replaceAll("DD","WS").replaceAll("KJ","WS");
                    File newFile = new File(targetPath+newName);
                    try {
                        Files.copy(f,newFile);
                    }catch (IOException e) {
                        e.printStackTrace();
                    }
                    count++;
                    System.out.println(f.getName()+"处理成功"+"newFile:"+newFile.getName());
                }
            }
            System.out.println("复制文件总数："+count);

    }
}
```

使用Maven构建工程，需要在pom文件中引入Guava：

```xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>21.0</version>
</dependency>
```

总共执行了800多个文件的拷贝，比较稳定。