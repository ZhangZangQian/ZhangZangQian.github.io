---
title: JDK11 新特性
author: zhangzangqian
date: 2022-12-20 21:00:00 +0800
categories: [技术]
tags: [Java, JDK]
math: true
mermaid: true
---

## String API

新增一些方法：`isBlank`、 `lines`、 `strip`、 `stripLeading`、 `stripTrailing` 和 `repeat`。让我们看看如何利用新方法来从多行字符串中提取非空白行：

```java
String multilineString = "Baeldung helps \n \n developers \n explore Java.";
List<String> lines = multilineString.lines()
  .filter(line -> !line.isBlank())
  .map(String::strip)
  .collect(Collectors.toList());
assertThat(lines).containsExactly("Baeldung helps", "developers", "explore Java.");
```

## File API

对 `Files` 类增加了 `writeString` 和 `readString` 两个静态方法，可以直接把 `String` 写入文件，或者把整个文件读出为一个 `String`:
```java
Files.writeString(
    Path.of("./", "tmp.txt"), // 路径
    "hello, jdk11 files api", // 内容
    StandardCharsets.UTF_8); // 编码
String s = Files.readString(
    Paths.get("./tmp.txt"), // 路径
    StandardCharsets.UTF_8); // 编码
```
这两个方法可以大大简化读取配置文件之类的问题。

## Collection 转 Array

`java.util.Collection` 接口包含一个新的默认转 `Array` 方法，该方法采用 `IntFunction` 参数。这使得从集合中创建正确类型的数组变得更加容易：

```java
List sampleList = Arrays.asList("Java", "Kotlin");
String[] sampleArray = sampleList.toArray(String[]::new);
assertThat(sampleArray).containsExactly("Java", "Kotlin");
```

## Not 断言

```java
List<String> sampleList = Arrays.asList("Java", "\n \n", "Kotlin", " ");
List withoutBlanks = sampleList.stream()
  .filter(Predicate.not(String::isBlank))
  .collect(Collectors.toList());
assertThat(withoutBlanks).containsExactly("Java", "Kotlin");
```
## Lambda 中的 LocalVar 语法

Java 11中添加了对在 lambda 参数中使用本地变量语法（var关键字）的支持。我们可以利用此功能将修饰符应用于我们的局部变量，例如定义类型注释：

```java
List<String> sampleList = Arrays.asList("Java", "Kotlin");
String resultString = sampleList.stream()
  .map((@Nonnull var x) -> x.toUpperCase())
  .collect(Collectors.joining(", "));
assertThat(resultString).isEqualTo("JAVA, KOTLIN");
```

## Http Client

`java.net.http` 包的新 HTTP 客户端是在 Java 9 中引入的，它现已成为 Java 11 的标准功能。新的 HTTP API 提高了整体性能，并为 HTTP/1.1 和 HTTP/2 提供了支持：

```java
HttpClient httpClient = HttpClient.newBuilder()
  .version(HttpClient.Version.HTTP_2)
  .connectTimeout(Duration.ofSeconds(20))
  .build();
HttpRequest httpRequest = HttpRequest.newBuilder()
  .GET()
  .uri(URI.create("http://localhost:" + port))
  .build();
HttpResponse httpResponse = httpClient.send(httpRequest, HttpResponse.BodyHandlers.ofString());
assertThat(httpResponse.body()).isEqualTo("Hello from the server!");
```

## 直接执行 Java 文件

```bash
$ java HelloWorld.java
Hello Java 11!
```

除了新增的API外，JDK11还带来了 EpsilonGC，就是什么也不做的 GC，以及 ZGC，一个几乎可以做到毫秒级暂停的GC。ZGC 还处于实验阶段，所以启动它需要命令行参数 `-XX:+UnlockExperimentalVMOptions -XX:+UseZGC。`

JDK11是一个LTS版本（Long-Term-Support），所以放心升级吧！！！