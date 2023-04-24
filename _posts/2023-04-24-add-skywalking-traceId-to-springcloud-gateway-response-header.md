---
title: SpringCloud Gateway 响应头添加 Skywalking TraceId
author: zhangzangqian
date: 2023-04-24 11:00:00 +0800
categories: [技术]
tags: [SpringCloud, Skywalking]
math: true
mermaid: true
---

在微服务架构中，一次请求可能会被多个服务处理，而每个服务又会产生相应的日志，且每个服务也会有多个实例。在这种情况下，如果系统发生异常，没有 Trace ID，那么在进行日志分析和追踪时就会非常困难，因为我们无法将所有相关的日志信息串联起来。

如果将 Trace ID 添加到响应头中，那么在进行日志分析和追踪时，配合日志收集分析平台，我们就可以通过这个 Trace ID 将所有相关的日志信息串联起来，便于分析和定位问题。

那么如何实现呢？微服务架构下 Api 网关是流量的统一出入口，在 Api 网关配置是最合适的，我们使用的 SpringCloud Gateway 作为微服务的应用网关，同时时 Skywalking 作为链路追踪工具。两者版本如下：

- SpringCloud Gateway 3.1.4
- Skywalking Agent 8.14.0

1. [下载](https://skywalking.apache.org/downloads/){: target="blank"}并解压 Skywalking Agent，并将 Agent 根目录下的 `optional-plugins/apm-spring-webflux-5.x-plugin-8.15.0.jar`、`optional-plugins/apm-spring-cloud-gateway-3.x-plugin-8.15.0.jar` 移动至 `plugins` 目录下。
2. SpringCloud Gateway 添加 pom 依赖如下：
    ```xml
    <dependency>
        <groupId>org.apache.skywalking</groupId>
        <artifactId>apm-toolkit-trace</artifactId>
        <version>8.14.0</version>
    </dependency>

    <dependency>
        <groupId>org.apache.skywalking</groupId>
        <artifactId>apm-toolkit-webflux</artifactId>
        <version>8.14.0</version>
    </dependency>
    ```
    {: file="pom.xml}
3. SpringCloud Gateway 创建 GloablFilter 拦截器，添加 SKywalking TraceId 至响应头。
    ```java
    @Component
    public class PutTraceIdIntoResponseHeaderFilter implements GlobalFilter {
        @Override
        public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
            String traceId = WebFluxSkyWalkingOperators.continueTracing(exchange, TraceContext::traceId);
            exchange.getResponse().getHeaders().set("x-trace-id", traceId);
            return chain.filter(exchange);
        }
    }
    ```

如此，大功告成，调用接口测试即可~~~

参考：https://github.com/apache/skywalking/discussions/10686