---
title: Maven 打包源码
author: zhangzangqian
date: 2021-01-12 21:00:00 +0800
categories: [技术]
tags: [Java, maven]
math: true
mermaid: true
---

```xml
<build>
    <plugins>
        <plugin>
            <artifactId>maven-source-plugin</artifactId>
            <version>2.1</version>
            <configuration>
                <attach>true</attach>
            </configuration>
            <executions>
                <execution>
                    <phase>compile</phase>
                    <goals>
                        <goal>jar</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```
{: file='pom.xml'}

> 配置中指定了phase为compile，意思是在生命周期compile的时候就将源文件打包，即只要执行的mvn命令包括compile这一阶段，就会将源代码打包。同样，phase还可以指定为package、install等等。
{: .prompt-tip}