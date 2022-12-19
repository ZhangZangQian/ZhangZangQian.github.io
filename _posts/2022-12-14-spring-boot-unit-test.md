---
title: SpringBoot 中的单元测试
author: zhangzangqian
date: 2022-12-14 18:00:00 +0800
categories: [技术]
tags: [Java, SpringBoot]
math: true
mermaid: true
---

> 还是那个军工项目啊，因为对接了第三方平台的接口，但是那边还有没有开发完成，目前只给了接口文档，那咋联调呢？那就基于 mockito 整个单元测试吧，以前写过忘记了，又是各种查资料，现在重新整理出来。
{: .prompt-tip}

## 概念

> 单元测试（unit testing），是指对软件中的最小可测试单元进行检查和验证。至于“单元”的大小或范围，并没有一个明确的标准，“单元”可以是一个函数、方法、类、功能模块或者子系统。单元测试通常和白盒测试联系到一起，如果单从概念上来讲两者是有区别的，不过我们通常所说的“单元测试”和“白盒测试”都认为是和代码有关系的，所以在某些语境下也通常认为这两者是同一个东西。还有一种理解方式，单元测试和白盒测试就是对开发人员所编写的代码进行测试。

这是百度百科来的，通俗的讲，就是测试一个方法逻辑有没有漏洞，执行结果是否符合预期。

## 代码实操，快速开始

> 本案例为 SpringBoot 2.7.6 版本，基于 Junit5。Junit5 与 Junit4 注解上略有不同。
{: .prompt-tip}

1. 完成正常业务逻辑开发！demo 如下，简单的登录逻辑

    ```java
    @Data
    @Accessors(chain = true)
    public class User {

        private Integer id;
        private String name;
        private String password;

    }

    @Repository
    public class UserDAO {

        private List<User> db = Lists.newArrayList(new User().setId(1).setName("用户1").setPassword("pwd1"),
            new User().setId(2).setName("用户2").setPassword("pwd2"));

        public User selectById(int id) {
            return db.stream().filter(u -> u.getId().equals(id)).findAny().orElse(null);
        }

        public boolean save(User user) {
            return this.db.add(user);
        }

    }

    @Service
    public class UserService {

        @Autowired
        private UserDAO userDAO;

        public boolean login(User user) {
            User userInDB = userDAO.selectById(user.getId());

            if (user.getPassword().equals(userInDB.getPassword())) {
                return true;
            }

            return false;
        }

    }
    ```

2. 引入 SpringBoot 提供的单元测试 starter，通过 `Spring Initializr` 创建的项目一般都会自动引入，无需手动引入。

    ```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    ```
    {: file='pom.xml'}

3. 单元测试代码如下

    ```java
    @ExtendWith(SpringExtension.class)
    @SpringBootTest
    class UserServiceTest {

        @Autowired
        private UserService userService;

        @Test
        void login() {
            User param = new User().setId(1).setPassword("pwd");
            boolean login = userService.login(param);
            assert !login;
        }

    }
    ```

## 最佳实践，进阶使用

### Mockito

#### mock 普通方法

如此，一个简单的单元测试就完成了，但是前提是在所有代码全部完工的情况下，如果 `UserDAO.selectById` 方法是调用第三方服务的，并且此时如果第三方服务的接口还没有开发完成？那怎么测试？我们可以通过基于 `Mockito` 的方式进行，以 `UserDAO.selectById` 为例，使用 Mockito 创建一个 `UserDAO` 类型的对应，然后声明 `selectById` 传入何种参数时的返回结果，然后进行调用，模拟UserDAO 的正常运行，来进行代码逻辑的测试，实例如下。

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
class UserServiceTest {

    @Autowired
    private UserService userService;

    /**
    * 使用 MockBean 创建 UserDAO 对象
    */
    @MockBean
    private UserDAO userDAO;

    @Test
    void login() {
        // 定义当调用 userDAO.selectById 的入参为 1 时，
        // 返回 一个 User id 为 1 密码为 pwd 的 User 对象，
        // 与上文数据库中的id 为 1、密码为 pwd1 的对象做区分。
        Mockito.when(userDAO.selectById(1))
            .thenReturn(new User().setId(1).setName("用户1").setPassword("pwd"));

        User param = new User().setId(1).setPassword("pwd");
        // 此时 login 方法中调用 userDAO.selectById(1) 返回的用户对象密码为 pwd
        // 因此返回结果为 true
        boolean login = userService.login(param);
        assert login;
    }

}
```

#### mock 静态方法

有时候，我们要 mock 的方法可能是一个静态方法，那怎么做的？首先需要修改 pom 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
    <!-- 排除 mockito-core 依赖 -->
<!--     <exclusions>-->
<!--         <exclusion>-->
<!--             <groupId>org.mockito</groupId>-->
<!--             <artifactId>mockito-core</artifactId>-->
<!--         </exclusion>-->
<!--     </exclusions>-->
</dependency>

<!-- 引入 mockito-inline，mockito-inline 依赖了 mockito-core -->
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-inline</artifactId>
    <version>4.5.1</version>
</dependency>

```
{: file='pom.xml'}

测试代码如下

```java
@Service
public class UserService {

    @Autowired
    private UserDAO userDAO;

    public boolean loginV2(User user) {
        User userInDB = userDAO.selectById(user.getId());

        /**
        * 使用静态方法进行密码比对
        */
        return PwdChecker.check(user.getPassword(), userInDB.getPassword());
    }

}

public class PwdChecker {

    public static boolean check(String srcPwd, String distPwd) {
        return Objects.equals(srcPwd, distPwd);
    }

}

@ExtendWith(SpringExtension.class)
@SpringBootTest
class UserServiceTest {

    @Autowired
    private UserService userService;

    @MockBean
    private UserDAO userDAO;

    @Test
    void login() {
        // 必须在 try-with-resources 语句的作用域中使用，否则 mock 不生效。
        try (MockedStatic<PwdChecker> mockedStatic = Mockito.mockStatic(PwdChecker.class)) {
            // mock 设置调用 PwdChecker.check("pwd", "pwd1") 返回 true
            mockedStatic.when(() -> PwdChecker.check("pwd", "pwd1")).thenReturn(true);
            Mockito.when(userDAO.selectById(1))
                .thenReturn(new User().setId(1).setName("用户1").setPassword("pwd1"));
            User param = new User().setId(1).setPassword("pwd");

            // PwdChecker.check("pwd", "pw1d") 结果 true，所以 login 值为 true
            boolean login = userService.loginV2(param);
            assert login;
        }
    }

}

```

### 匹配任意参数

通过 `org.mockito.ArgumentMatchers` 的 `anyXx()` 方法，`xx` 表示类型，如下是匹配任意 `int` 值

```java
// 匹配任意 int 值都返回 new User().setId(1).setName("用户1").setPassword("pwd1") 这个对象
Mockito.when(userDAO.selectById(org.mockito.ArgumentMatchers.anyInt()))
                .thenReturn(new User().setId(1).setName("用户1").setPassword("pwd1"));
```

当某个方法参数是对象时

```java
@Service
public class UserService {

    @Autowired
    private UserDAO userDAO;

    public boolean register(User user) {
        return userDAO.save(user);
    }

}

@Test
void register() {

    Mockito.when(userDAO.save(ArgumentMatchers.any())).thenReturn(false);

    boolean register = userService.register(null);

    assert !register;

}

```

源码地址：<https://github.com/zhangzangqian13/spring-boot-unit-test-demo1/tree/main>

<!-- 
### 分层测试时，减少配置加载

通过启动可以看出，我们凄恻 -->