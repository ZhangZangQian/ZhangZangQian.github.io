---
title: Spring Boot 注解方式进行表单验证
author: zhangzangqian
date: 2022-11-17 21:00:00 +0800
categories: [技术]
tags: [Java, SpringBoot]
math: true
mermaid: true
---
在实际业务开发中肯定会遇到参数校验的场景，参数少还好说，在遇到大量参数校验的场景时，单写 `if-else` 就很痛苦了，因此 Spring 为我们提供了通过注解方式进行参数校验的功能。

## 1. 引入依赖

如果是 Spring Boot `2.3` 以后版本，需要引入 `spring-boot-starter-validation`，`2.3`之前已经集成在 `spring-boot-starter-web` 中。
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

## 2. 使用方式


### 基础使用

```java
/**
* 使用 @Validated 注解标识需要对参数 knowledgeVO 进行校验
*/
@PostMapping("/add")
public Result<Void> addKnowledge(@RequestBody @Validated KnowledgeVO knowledgeVO) {
    return Result.ok();
}

@PostMapping("/update")
@Operation(summary = "修改")
public Result<Void> updateKnowledge(@Parameter(description = "知识参数", required = true) @RequestBody @Validated(
    {OnUpdate.class}) KnowledgeVO knowledgeVO) {
    knowledgeService.save(knowledgeVO);
    return Result.ok();
}

/**
* 在属性上添加注解标识需要对该字段进行校验
*/
@Data
@Accessors(chain = true)
public class KnowledgeVO {

    @NotBlank(message = "名称不能为空")
    @Size(min = 1, max = 100)
    private String name;

    @NotEmpty(message = "属性列表不能为空")
    @Valid
    private List<KnowledgeAttributeVO> attributes;

}

@Data
@Accessors(chain = true)
public class KnowledgeAttributeVO {

    @NotNull(message = "id 不能为空", groups = OnUpdate.class)
    private Long id;

    @NotBlank(message = "属性名不能为空")
    private String name;

    @NotBlank(message = "属性值不能为空")
    private String value;

}

```

### 常用注解

|注解|说明|
|:---|:------|
|@Null|	限制只能为null|
|@NotNull|	限制必须不为null,一般用来校验Integer类型不能为空|
|@AssertFalse|	限制必须为false|
|@AssertTrue|	限制必须为true|
|@DecimalMax(value)|	限制必须为一个不大于指定值的数字|
|@DecimalMin(value)|	限制必须为一个不小于指定值的数字|
|@Digits(integer,fraction)|	限制必须为一个小数，且整数部分的位数不能超过integer，<br/>小数部分的位数不能超过fraction|
|@Future|	限制必须是一个将来的日期|
|@Max(value)|	限制必须为一个不大于指定值的数字|
|@Min(value)|	限制必须为一个不小于指定值的数字|
|@Past|	限制必须是一个过去的日期|
|@Pattern(value)|	限制必须符合指定的正则表达式|
|@Size(max,min)|	限制字符长度必须在min到max之间|
|@Past|	验证注解的元素值（日期类型）比当前时间早|
|@NotEmpty|	验证注解的元素值不为null且不为空（字符串长度不为0、集合大小不为0）,一般用<br/>来校验List类型不能为空|
|@NotBlank|	验证注解的元素值不为空（不为null、去除首位空格后长度为0），一般用<br/>来校验String类型不能为空,不同于@NotEmpty，@NotBlank只应用于<br/>字符串且在比较时会去除字符串的空格|
|@Email|	验证注解的元素值是Email，也可以通过正则表达式和flag指定自定义的email格式|
|@Valid|添加到属性上时表示需要对属性进行循环校验|


### 进阶使用（校验分组）

当出现这种场景，当新增时不需要校验 id，修改时需要校验 id 不为空，这怎么搞呢？通过分组的方式，代码见下图
```java
/**
* 新建类作为分组标识
*/
public interface OnUpdate extends Default {

}

@Data
@Accessors(chain = true)
@ApiModel("知识属性")
public class KnowledgeAttributeVO {

    /**
    * 通过 groups 声明 id 处于 OnUpdate 分组，同时 OnUpdate 集成自 Default 接口
    * （所有没有声明 groups 的注解均属 Default 分组），因此使用 OnUpdate 分组的注解进
    * 行校验时也会对 Default 分组的字段进行校验
    */
    @NotNull(message = "id 不能为空", groups = OnUpdate.class)
    private Long id;

    @NotBlank(message = "属性名不能为空")
    private String name;

    @NotBlank(message = "属性值不能为空")
    private String value;

}

/**
* @Validated({OnUpdate.class}) 表示使用 OnUpdate 分组的注解进行表单校验
*/
@PostMapping("/update")
public Result<Void> updateKnowledge(@RequestBody @Validated({OnUpdate.class}) KnowledgeVO knowledgeVO) {
    knowledgeService.save(knowledgeVO);
    return Result.ok();
}
```

整合代码后运行代码执行结果如下：
![/spring-boot-vlidation-run-result.png](/assets/images/spring-boot-vlidation-run-result.png){: width="30%" height="30%" .w-75 .normal}

再看后台运行日志
```Console
2022-11-17 20:48:45.828  WARN 93016 --- [nio-8613-exec-1] .w.s.m.s.DefaultHandlerExceptionResolver : Resolved [org.springframework.web.bind.MethodArgumentNotValidException: Validation failed for argument [0] in public com.mgdaas.lidihuo.bean.response.Result<java.lang.Void> com.mgdaas.lidihuo.controller.KnowledgeController.updateKnowledge(com.mgdaas.lidihuo.bean.request.KnowledgeVO) with 3 errors: [Field error in object 'knowledgeVO' on field 'attributes': rejected value [null]; codes [NotEmpty.knowledgeVO.attributes,NotEmpty.attributes,NotEmpty.java.util.List,NotEmpty]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [knowledgeVO.attributes,attributes]; arguments []; default message [attributes]]; default message [属性列表不能为空]] [Field error in object 'knowledgeVO' on field 'id': rejected value [null]; codes [NotNull.knowledgeVO.id,NotNull.id,NotNull.java.lang.Long,NotNull]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [knowledgeVO.id,id]; arguments []; default message [id]]; default message [id 不能为空]] [Field error in object 'knowledgeVO' on field 'name': rejected value [null]; codes [NotBlank.knowledgeVO.name,NotBlank.name,NotBlank.java.lang.String,NotBlank]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [knowledgeVO.name,name]; arguments []; default message [name]]; default message [名称不能为空]] ]
```

在实际开发中参数异常信息需要返回出去，因此我们可以结合 ControllerAdivce 来进行使用

```java
@RestController
@ControllerAdvice
@Slf4j
public class GlobalExceptionHandlerController {

    @ExceptionHandler({MethodArgumentNotValidException.class})
    public Result<Void> handleMethodArgumentNotValidException(HttpServletRequest request,
        MethodArgumentNotValidException ex) {
        List<FieldError> fieldErrors = ex.getBindingResult().getFieldErrors();
        String message = fieldErrors.stream().map(DefaultMessageSourceResolvable::getDefaultMessage)
            .collect(Collectors.joining(","));
        log.debug("调用接口：{} 发生异常，错误信息：{}", request.getRequestURI(), Throwables.getStackTraceAsString(ex));
        return Result.fail(message);
    }

}

```

如此我们再看返回结果：
```json
{
	"code": 400,
	"message": "id 不能为空,名称不能为空,属性列表不能为空",
	"data": null
}
```

### 自定义校验

实际业务场景中 Spring 为我提供的注解肯定还是不能完全满足校验需求的，比如说校验某个字段数据是否存在，示例如下

```java
/**
* 自定义校验注解
*
*/
@Documented
@Target({FIELD})
@Retention(RUNTIME)
@Constraint(validatedBy = {CategoryIdExistsValidator.class})
public @interface CategoryIdExists {

    /**
    * 默认错误信息
    */
    String message() default "类别 id 不存在";

    Class[] groups() default {};

    Class[] payload() default {};

}

/**
* 校验逻辑实现类
* 
*/
public class CategoryIdExistsValidator implements ConstraintValidator<CategoryIdExists, Long> {

    @Autowired
    private CategoryRepository categoryRepository;

    @Override
    public boolean isValid(Long value, ConstraintValidatorContext context) {
        // 为空时不校验
        if (value == null) {
            return true;
        }
        // 禁用默认消息
        context.disableDefaultConstraintViolation();

        // 查询数据库
        Optional<Category> optional = categoryRepository.findById(value);
        if (optional.isPresent()) {
            return true;
        }

        // 校验不通过设置错误信息
        context.buildConstraintViolationWithTemplate("类别 id:" + value + "  不存在").addConstraintViolation();
        return false;
    }
}

@Data
@Accessors(chain = true)
@ApiModel("知识类别")
public class CategoryVO {

    @ApiModelProperty("类别 id")
    @NotNull(message = "类别 id 不能为空", groups = {OnUpdate.class})
    @CategoryIdExists(groups = {OnUpdate.class})
    private Long id;

    @NotBlank(message = "类别名称不能为空")
    private String name;

    @CategoryIdExists
    private Long parentId;

}
```
