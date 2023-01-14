---
title: Maven 打包可执行 Jar 包
author: zhangzangqian
date: 2019-02-27 21:00:00 +0800
categories: [技术]
tags: [Java, maven]
math: true
mermaid: true
---

在 `pom.xml` 中添加如下内容

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-assembly-plugin</artifactId>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>single</goal>
            </goals>
            <configuration>
                <archive>
                <manifest>
                    <mainClass>
                        <!-- 主类全限定名（程序执行入口） -->
                        com.xx.xx.xx.ClassName
                    </mainClass>
                </manifest>
                </archive>
                <descriptorRefs>
                    <descriptorRef>jar-with-dependencies</descriptorRef>
                </descriptorRefs>
            </configuration>
        </execution>
    </executions>
</plugin>
```
{: file='pom.xml'}

执行 `mvn package` 后，会在 `target` 文件夹下生成两个 jar 包，一个是不带依赖的 jar 包，一个是后缀有 -dependencies 带有依赖的 jar 包，如下所示：

```console
-rw-r--r--  1 zhangzangqian  staff   3.9M Dec 28 18:53 mysql-ms-demo-1.0-SNAPSHOT-jar-with-dependencies.jar
-rw-r--r--  1 zhangzangqian  staff   3.6K Dec 28 18:53 mysql-ms-demo-1.0-SNAPSHOT.jar
```

运行 jar 包选择带有依赖的即可，命令如下：

```bash
java -jar mysql-ms-demo-1.0-SNAPSHOT-jar-with-dependencies.jar
```