---
title: Spring Boot 整合 Swagger3
author: zhangzangqian
date: 2022-11-17 19:00:00 +0800
categories: [技术]
tags: [Java, SpringBoot]
math: true
mermaid: true
---

## 1. 引入依赖

```xml
<!-- https://mvnrepository.com/artifact/io.springfox/springfox-boot-starter -->
<dependency>
  <groupId>io.springfox</groupId>
  <artifactId>springfox-boot-starter</artifactId>
  <version>3.0.0</version>
</dependency>
```

## 2. 添加配置

```yaml
api-doc:
    # 是否开启swagger，生产环境一般关闭，所以这里定义一个变量
    enable: true
    application-name: 【立帝货】
    application-version: 1.0
    application-description: 上知五百年，中知五百年，下知五百年，智能问答系统
    try-host: http://localhost:${server.port}
```

```java
@Component
@ConfigurationProperties("api-doc")
@Data
public class ApiDocProperties {

    /**
    * 是否开启swagger，生产环境一般关闭，所以这里定义一个变量
    */
    private Boolean enable = false;

    /**
    * 项目应用名
    */
    private String applicationName;

    /**
    * 项目版本信息
    */
    private String applicationVersion;

    /**
    * 项目描述信息
    */
    private String applicationDescription;

    /**
    * 接口调试地址
    */
    private String tryHost;

}

@Configuration
@EnableOpenApi
public class ApiDocConfig implements WebMvcConfigurer {

    private final ApiDocProperties apiDocProperties;

    public ApiDocConfig(ApiDocProperties apiDocProperties) {
        this.apiDocProperties = apiDocProperties;
    }

    @Bean
    public Docket docket() {
        return new Docket(DocumentationType.OAS_30).pathMapping("/")

            // 定义是否开启swagger，false为关闭，可以通过变量控制
            .enable(apiDocProperties.getEnable())

            // 将api的元信息设置为包含在json ResourceListing响应中。
            .apiInfo(apiInfo())

            // 接口调试地址
            .host(apiDocProperties.getTryHost())

            // 选择哪些接口作为swagger的doc发布
            .select().apis(RequestHandlerSelectors.any()).paths(PathSelectors.any()).build()

            // 支持的通讯协议集合
            .protocols(Sets.newHashSet("http"));
    }

    /**
    * API 页面上半部分展示信息
    */
    private ApiInfo apiInfo() {
        return new ApiInfoBuilder().title("【" + apiDocProperties.getApplicationName() + "】接口文档")
            .description(apiDocProperties.getApplicationDescription())
            .contact(new Contact("张臧乾", null, "zhangy@mgdaas.com")).version(
                "Application Version: " + apiDocProperties.getApplicationVersion() + ", Spring Boot Version: " + SpringBootVersion.getVersion())
            .build();
    }

    /**
    * 通用拦截器排除swagger设置，所有拦截器都会自动加swagger相关的资源排除信息
    */
    @SuppressWarnings("unchecked")
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        try {
            Field registrationsField = FieldUtils.getField(InterceptorRegistry.class, "registrations", true);
            List<InterceptorRegistration> registrations =
                (List<InterceptorRegistration>)ReflectionUtils.getField(registrationsField, registry);
            if (registrations != null) {
                for (InterceptorRegistration interceptorRegistration : registrations) {
                    interceptorRegistration.excludePathPatterns("/swagger**/**").excludePathPatterns("/webjars/**")
                        .excludePathPatterns("/v3/**").excludePathPatterns("/doc.html");
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```

## 3. 使用方式

```java
@RestController
@RequestMapping("/knowledge/v1")
@Api(tags = "知识控制器")
public class KnowledgeController {

    @Autowired
    private KnowledgeService knowledgeService;

    @PostMapping("/add")
    @Operation(summary = "新增")
    public Result<Void> addKnowledge(
        @Parameter(description = "知识参数", required = true) @RequestBody KnowledgeVO knowledgeVO) {
        knowledgeService.save(knowledgeVO);
        return Result.ok();
    }
}

@Data
@Accessors(chain = true)
@ApiModel("知识")
public class KnowledgeVO {

    @ApiModelProperty("id")
    private Long id;

    @ApiModelProperty("名称")
    private String name;

    @ApiModelProperty("属性列表")
    private List<KnowledgeAttributeVO> attributes;

}
```
    
## 4. 运行界面
![swagger preview](/assets/images/swagger-1.png){: width="30%" height="30%" .w-75 .normal}
![swagger preview](/assets/images/swagger-2.png){: width="30%" height="30%" .w-75 .normal}