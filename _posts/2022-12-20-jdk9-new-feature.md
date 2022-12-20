---
title: JDK9 新特性
author: zhangzangqian
date: 2022-12-20 16:00:00 +0800
categories: [技术]
tags: [Java, JDK]
math: true
mermaid: true
---

> 自从用了 JDK 1.8 之后这么多年，一直没有关心过 JDK 新版本的新特性，后续认真整理一个系列。

## 模块化

- [Java 9的模块化--壮士断"腕"的涅槃](https://zhuanlan.zhihu.com/p/24800180){: target="blank"}
- [模块---廖雪峰的官方网站](https://www.liaoxuefeng.com/wiki/1252599548343744/1281795926523938#0){: target="blank"}
- [Project Jigsaw: Module System Quick-Start Guide](https://openjdk.org/projects/jigsaw/quick-start){: target="blank"}

## 新的 Http 客户端

旧的 HtpURLConnection 的替代品，支持 HTTP 2 和 WebSocket，性能与 Apache HttpClient、Netty 和 Jetty 相当。新的 API 位于 `java.net.http` 包下。如下代码是一个快速创建 GET 请求的 demo：

```java
HttpRequest request = HttpRequest.newBuilder()
    .uri(new URI("https://postman-echo.com/get"))
    .GET()
    .build();

HttpResponse<String> response = HttpClient.newHttpClient()
    .send(request, HttpResponse.BodyHandler.asString());
```

## JShell 命令行工具

可以执行 Java 语言的交互式命令行工具，例子如下：

```bash
➜  ~ jshell                  
|  欢迎使用 JShell -- 版本 11.0.13
|  要大致了解该版本, 请键入: /help intro

jshell> int a = 10
a ==> 10

jshell> var b = 10
b ==> 10

jshell> a + b
$3 ==> 20

jshell> 
```
如果想了解有关 JShell 工具的更多信息，请查看[Java 9 REPL Basics (Part-1) ](https://www.digitalocean.com/community/tutorials/java-repl-jshell){: target="blank"}和[Java 9 REPL Features (Part-2)](https://www.digitalocean.com/community/tutorials/jshell-java-shell){: target="blank"}。

## VarHandle

位于 `java.lang.invoke` 包下，由 `VarHandle` 和 `MethodHandles` 组成。提供了等价于 `java.util.concurrent.atomic` 和 `sun.misc.Unsafe` 操作对象字段和数组元素的功能（性能相似）。

```java
// AbstractQueuedSynchronizer 类中的使用案例
private static final VarHandle STATE;

static {
    try {
        MethodHandles.Lookup l = MethodHandles.lookup();
        STATE = l.findVarHandle(AbstractQueuedSynchronizer.class, "state", int.class);
    } catch (ReflectiveOperationException e) {
        throw new ExceptionInInitializerError(e);
    }
}

protected final boolean compareAndSetState(int expect, int update) {
    return STATE.compareAndSet(this, expect, update);
}
```

**使用 Java9 模块化系统，将无法从应用程序代码中使用`sun.misc.Unsafe`。**


## 进程 API 改进

1. `java.lang.ProcessHandle` 新增了很多功能

    ```java
    ProcessHandle self = ProcessHandle.current();
    long PID = self.getPid();
    ProcessHandle.Info procInfo = self.info();
    
    Optional<String[]> args = procInfo.arguments();
    Optional<String> cmd =  procInfo.commandLine();
    Optional<Instant> startTime = procInfo.startInstant();
    Optional<Duration> cpuUsage = procInfo.totalCpuDuration();
    ```
    当前方法返回了一个代表当前 JVM 进程的类，子类提供了当前进程的详细信息。

2. 停止进程

    停止当前进程的所有子进程：
    ```java
    childProc = ProcessHandle.current().children();
    childProc.forEach(procHandle -> {
        assertTrue("Could not kill process " + procHandle.getPid(), procHandle.destroy());
    });
    ```

## 语法改进

1. try-with-resources，在 Java 7 中，可以实现资源的自动关闭，但必须在 try 子句中声明一个新的变量，在 Java 9 后，**try 子句中可以引用一个 final 或者事实上是不可变的变量**，例子如下:

    ```java
    MyAutoCloseable mac = new MyAutoCloseable();
    try (mac) {
        // do some stuff with mac
    }

    try (new MyAutoCloseable() { }.finalWrapper.finalCloseable) {
    // do some stuff with finalCloseable
    } catch (Exception ex) { }
    ```

2. 钻石操作符，可以在匿名内部类中使用，例子如下

    ```java
    // 编译报错信息：Cannot use “<>” with anonymous inner classes
    Comparator<Object> comparator = new Comparator<>() {
        @Override
        public int compare(Object o1, Object o2) {
            return 0;
        }
    };

    //java9特性五：钻石操作符的升级
    @Test
    public void test2() {
        //钻石操作符与匿名内部类在java 8中不能共存。在java9可以。
        Comparator<Object> com = new Comparator<>() {
            @Override
            public int compare(Object o1, Object o2) {
                return 0;
            }
        };

        //jdk7中的新特性：类型推断
        ArrayList<String> list = new ArrayList<>();

    }
    ```
3. 接口中的 private 方法

    ```java
    interface InterfaceWithPrivateMethods {
        private static String staticPrivate() {
            return "static private";
        }
        
        private String instancePrivate() {
            return "instance private";
        }
        
        default void check() {
            String result = staticPrivate();
            InterfaceWithPrivateMethods pvt = new InterfaceWithPrivateMethods() {
                // anonymous class
            };
            result = pvt.instancePrivate();
        }
    }}
    ```

## jcmd 子命令

我们将获得JVM中加载的所有类及其继承结构的列表。在下面的示例中，我们可以看到 JVM 中加载的 `java.lang.Socket` 的层次结构：

```bash
jdk-9\bin>jcmd 14056 VM.class_hierarchy -i -s java.net.Socket
14056:
java.lang.Object/null
|--java.net.Socket/null
|  implements java.io.Closeable/null (declared intf)
|  implements java.lang.AutoCloseable/null (inherited intf)
|  |--org.eclipse.ecf.internal.provider.filetransfer.httpclient4.CloseMonitoringSocket
|  |  implements java.lang.AutoCloseable/null (inherited intf)
|  |  implements java.io.Closeable/null (inherited intf)
|  |--javax.net.ssl.SSLSocket/null
|  |  implements java.lang.AutoCloseable/null (inherited intf)
|  |  implements java.io.Closeable/null (inherited intf)

```
jcmd 命令的第一个参数是 JVM 的进程ID（PID），您可以使用命令 `jcmd 14056 VM.flags -all` 找到所有可用的参数。


## 发布订阅框架

Java SE 9 Reactive Streams API是一个发布/订阅框架，使用Java语言非常轻松地实现异步、可扩展和并行应用程序。Java SE 9引入了以下API，用于在基于Java的应用程序中开发反应流。

- java.util.concurrent.Flow
- java.util.concurrent.Flow.Publisher
- java.util.concurrent.Flow.Subscriber
- java.util.concurrent.Flow.Processor

## 统一的 JVM 日志

> This feature introduces a common logging system for all components of the JVM. It provides the infrastructure to do the logging, but it does not add the actual logging calls from all JVM components. It also does not add logging to Java code in the JDK.

此功能为JVM的所有组件引入了一个通用的日志记录系统。它提供了进行日志记录的基础设施，但不会添加来自所有JVM组件的实际日志记录调用。它也不会在JDK中添加对Java代码的日志记录。

> The logging framework defines a set of tags – for example, gc, compiler, threads, etc. We can use the command line parameter -Xlog to turn on logging during startup.

日志框架定义了一组标签——例如gc、编译器、线程等。我们可以在启动期间使用命令行参数-Xlog打开日志记录。

> Let's log messages tagged with ‘gc' tag using ‘debug' level to a file called ‘gc.txt' with no decorations:

让我们使用“调试”级别将标记为“gc”标签的消息记录到一个名为“gc.txt”的文件，没有装饰：


```bash
java -Xlog:gc=debug:file=gc.txt:none ...
```

> -Xlog:help will output possible options and examples. Logging configuration can be modified runtime using jcmd command. We are going to set GC logs to info and redirect them to a file – gc_logs:

-Xlog:help 将输出可能的选项和示例。可以使用jcmd命令修改日志配置的运行时。我们将把GC日志设置为info，并将其重定向到一个文件-gc_logs：

```bash
jcmd 9615 VM.log output=gc_logs what=gc
```

> 本段解释机翻过来的，没详细了解。

## 新 API

### 不可变集合

```java
List immutableList = List.of();
List immutableList = List.of("one","two","three");
```

### Stream 增强

```bash
jshell> Stream.of(1,2,3,4,5,6,7,8,9,10).takeWhile(i -> i < 5 )
                 .forEach(System.out::println);
```