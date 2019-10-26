---
title: maven打包时在META-INF/MANIFEST.MF中添加主清单属性
date: 2018-05-26 22:16:08
tags: [java,maven,manifest]
---

## 发现问题:
默认使用`mvn clean package`生成的jar文件中的 **MANIFEST.MF** 是这样的:
```
Manifest-Version: 1.0
Created-By: Apache Maven 3.3.9
Built-By: shengyayun
Build-Jdk: 10.0.1
```
这种jar文件执行的时候会返回 **xxx.jar中没有主清单属性**。



## 解决方法:
修改 **pom.xml**文件：
```xml
<plugin>
    <artifactId>maven-jar-plugin</artifactId>
    <version>3.0.2</version>
    <configuration>
        <archive>
            <manifestEntries>
                <Main-Class>com.shengyayun.App</Main-Class> #划重点
            </manifestEntries>
        </archive>
    </configuration>
</plugin>
```
现在重新打包生成的MANIFEST.MF中多了一行`Main-Class: com.shengyayun.App`