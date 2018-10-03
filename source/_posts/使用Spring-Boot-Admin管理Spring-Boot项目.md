---
title: 使用 Spring-Boot-Admin 管理 Spring-Boot 项目
date: 2018-09-26 21:08:26
tags:
  - Java
  - Spring-Boot
categories:
  - Java
---

[`spring-boot-actuator`](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready.html) 模块虽然为 Spring-Boot 项目提供了监控及管理的 `API` ,但是并没有提供对应的 UI 管理系统,此时我们可以使用开源的 [Spring-Boot-Amin](https://github.com/codecentric/spring-boot-admin) 来为我们的 Spring-Boot 项目提供一个可视化的管理页面

### 创建项目

首先在创建一个简单的 Spring-Boot 项目,可以选择在  [start.spring.io](https://start.spring.io/)  中创建,然后导入

在项目顶级 `pom.xml` 定义 `spring-boot-admin` 的依赖版本

```xml
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <java.version>1.8</java.version>
  
    <spring-boot-admin.version>2.0.3</spring-boot-admin.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>de.codecentric</groupId>
            <artifactId>spring-boot-admin-dependencies</artifactId>
            <version>${spring-boot-admin.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

#### 创建 client 模块

在项目中创建一个用于与 `spring-boot-admin` 集成的演示模块 `client`

引入以下依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>de.codecentric</groupId>
        <artifactId>spring-boot-admin-starter-client</artifactId>
    </dependency>
</dependencies>
```

创建启动类

```java
@RestController
@SpringBootApplication
public class ClientApplication {

    @RequestMapping("/")
    public String index() {
        return "hello world";
    }

    public static void main(String[] args) {
        SpringApplication.run(ClientApplication.class, args);
    }
}
```

此时可以通过启动 `main` 方法查看启动是否正常

#### 创建 server 模块

在项目中创建 `spring-boot-admin` 服务模块 `server`

引入如下依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>de.codecentric</groupId>
        <artifactId>spring-boot-admin-starter-server</artifactId>
    </dependency>
</dependencies>
```

创建启动类

```java
@EnableAdminServer
@SpringBootApplication
public class ServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ServerApplication.class, args);
    }
}
```

创建配置文件 `application.yml`

```yaml
server:
  # 配置服务端口
  port: 8090
```

访问 [localhost:8090](http://localhost:8090) 即可进入 `spring-boot-admin` 管理页面,但是此时还没有项目与它进行集成,所以应用数为0

### 集成 Spring-Boot-Admin 服务

在演示模块 `client` 中添加配置文件 `application.yml`

```yaml
spring:
  application:
    # 项目名称
    name: client
  boot:
    admin:
      client:
        # 配置文档
        # http://codecentric.github.io/spring-boot-admin/current/#spring-boot-admin-client
        enabled: true
        # spring-boot-admin 服务端地址
        url: http://localhost:8090

management:
  endpoints:
    web:
      exposure:
        # 开放所有端点
        include: "*"
```

再重启 `client`项目,此时 `client` 项目会向 `http://localhost:8090` 注册自己的服务信息

刷新 [localhost:8090](http://localhost:8090) 页面,可发现 `client` 已经注册成功,此时可以查看 `client` 相关信息

### 高级配置

#### 修改注册的服务地址

在默认配置中使用 `InetAddress.getLocalHost().getCanonicalHostName()` 获取服务地址,本地环境一般没有问题,但是在生产环境中可能获取的是本机内网`IP`,导致`spring-boot-admin` 无法访问注册的服务,所以需要更改注册时的服务地址,配置方式如下

```yaml
spring:
  boot:
    admin:
      client:
        instance:
          service-base-url: "可供 spring-boot-admin 访问的基础地址,格式为 http://{主机或IP地址}:{端口}/"
```

#### 安全配置

#####  server 端安全配置

使用`spring-security`实现登录验证

首先引入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

创建`spring-security` 配置类

```java
@Configuration
public class SpringSecurityConfig extends WebSecurityConfigurerAdapter {

    /**
     * 配置文件中的 spring.boot.admin.context-path 属性值
     * 下列中涉及到该参数的都是为了处理对 context-path 的支持
     */
    private final String adminContextPath;

    public SecuritySecureConfig(AdminServerProperties adminServerProperties) {
        this.adminContextPath = adminServerProperties.getContextPath();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // @formatter:off
        SavedRequestAwareAuthenticationSuccessHandler successHandler = new SavedRequestAwareAuthenticationSuccessHandler();
        successHandler.setTargetUrlParameter("redirectTo");
        successHandler.setDefaultTargetUrl(adminContextPath + "/");

        http.authorizeRequests()
                // 对静态资源与登录url放行
                .antMatchers(adminContextPath + "/assets/**").permitAll()
                .antMatchers(adminContextPath + "/login").permitAll()
                // 其他请求需要登录
                .anyRequest().authenticated()
                .and()
                // 配置登录url,主要是为了支持 spring.boot.admin.context-path 属性
                .formLogin().loginPage(adminContextPath + "/login").successHandler(successHandler).and()
                // 配置注销url
                .logout().logoutUrl(adminContextPath + "/logout").and()
                .httpBasic().and()
                .csrf()
                // 配置 csrf,在前端使用 cookie 保存 csrf 参数
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                // 对 instances,actuator url忽略 csrf 验证
                .ignoringAntMatchers(
                        adminContextPath + "/instances",
                        adminContextPath + "/actuator/**"
                );
        // @formatter:on
    }
}
```

在`application.yml`配置默认帐号,密码

```yaml
spring:
  security:
    user:
      # 配置默认用户
      name: root
      password: root
```

如果帐号密码是保存在数据库中,配置方式请查看该博客 [使用数据库进行身份认证](https://www.baeldung.com/spring-security-authentication-with-a-database)

重启 `server` 模块,此次刷新[localhost:8090](http://localhost:8090) ,会重定向到登录页面,输入配置的默认帐号,密码即可登录

但是因为 `server` 端配置了登录验证,所以导致客户端无法注册,此时需要在 `client` 配置登录 `server` 所需的帐号与密码

 配置方式如下

```yaml
spring:
  boot:
    admin:
      client:
        # spring-boot-admin 服务端帐号密码
        username: root
        password: root
```

##### client 端安全配置

`client` 端主要是保护 `management` 中开放的 `web`  端点

首先引入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

创建`spring-security` 配置类

```java
@Configuration
public class SpringSecurityConfig extends WebSecurityConfigurerAdapter {

    /**
     * 配置文件中的 management.endpoints.web.base-path 属性值
     * 这里只对 web 端点做登录验证
     */
    private final String endpointsBasePath;

    public SpringSecurityConfig(WebEndpointProperties webEndpointProperties) {
        this.endpointsBasePath = webEndpointProperties.getBasePath();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                // web端点需要登录
                .antMatchers(endpointsBasePath + "/**").authenticated()
                // 其他请求进行忽略
                .anyRequest().permitAll()
                // 使用 httpBasic 进行登录验证,这是 spring-boot-admin server 的验证方式
                .and().httpBasic()
        // 如果不配置 formLogin,不会重定向到登录页,因为不需要登录页面
        // .and().formLogin()
        ;
        // 忽略对端点的 csrf 验证,或者直接关闭 csrf,http.csrf().disable();
        http.csrf().ignoringAntMatchers(endpointsBasePath + "/**");
    }
}
```

在`application.yml`配置默认帐号,密码

```yaml
spring:
  security:
    user:
      # 配置默认用户
      name: admin
      password: admin
```

同时为了允许 `server`  访问 `client` 的`web`端点,我们在注册时需要提供 `client` 的帐号与密码

```yaml
spring:
  boot:
    admin:
      client:
        instance:
          metadata:
            # 配置登录本服务需要的帐号与密码,用于让 spring-boot-admin 登录本服务
            user.name: ${spring.security.user.name}
            user.password: ${spring.security.user.password}
```

###### 端点配置

在演示项目中,我们选择了开放所有端口,为了安全起见,我们应该只开放需要的端口

其中各个端点的作用请看官网文档 [Spring-Boot 端点](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-endpoints.html#production-ready-endpoints)

```yaml
management:
  endpoints:
    jmx:
      exposure:
        exclude: "*"
    web:
      exposure:
        include: "*"
        # 排除关闭服务端点
        exclude:
          - shutdown
  endpoint:
    health:
      # 只有用户登录才显示 health 详情
      show-details: when_authorized
    # 禁用关闭服务端点
    shutdown:
      enabled: false
```

#### 配置日志文件实时查看

如果想在 `server` 查看 `client` 当前写入的日志文件内容,只需在 `client` 的配置文件中配置 `logging.path`或`logging.file` 属性即可,此时 `server` 会存在一个 `Logfile` 的标签,点击该标签即可查看

#### 配置邮件通知

首先在`server`引入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

邮箱发送相关配置

```yaml
spring:
  mail:
    # 邮箱服务器地址,如QQ邮箱的为 smtp.qq.com,阿里云企业邮箱的为 smtp.qiye.aliyun.com
    host: smtp.example.com
    # 发件人帐号
    username: example@example.com
    # 发件人密码
    password: example
    # 邮箱服务器端口
    port: 465
    # 邮件协议
    protocol: smtp
    # 默认字符编码
    default-encoding: UTF-8
    # 测试配置可用性
    test-connection: true
    # 如果使用加密端口需要添加下面的配置,否则不需要
    properties:
      mail.smtp.ssl.enable: true
```

`server` 发送邮件配置

```yaml
spring:
  boot:
    admin:
      notify:
        mail:
          # http://codecentric.github.io/spring-boot-admin/current/#mail-notifications
          enabled: true
          # 发件人
          from: Spring-Boot-Admin <${spring.mail.username}>
          # 收件人,如果存在多个,使用逗号分隔
          to: example@example.com
```

此时重启 `server` 端,然后停止`client`端,检查收件箱,可以发现已经接收到下线通知(接受邮件可能存在延迟)

### 重构

在上面的示例中,已经演示了一些基本的配置,而在实际开发中,一般情况下是一个 `server` 对应多个 `client`,多个 `client` 之间注册 `server` 的配置应该是一样的,此时我们可以选择将一些通用的配置抽取出来,以达到复用的效果

创建 `commons` 模块

在 `resources` 下创建 `config` 文件夹,用于存放通用的配置文件

创建通用邮箱配置文件 `application-commons-mail.yml`

```yaml
spring:
  mail:
    # 邮箱服务器地址,如QQ邮箱的为 smtp.qq.com,阿里云企业邮箱的为 smtp.qiye.aliyun.com
    host: smtp.example.com
    # 发件人帐号
    username: example@example.com
    # 发件人密码
    password: example
    # 邮箱服务器端口
    port: 465
    # 邮件协议
    protocol: smtp
    # 默认字符编码
    default-encoding: UTF-8
    # 测试配置可用性
    test-connection: true
    # 如果使用加密端口需要添加下面的配置,否则不需要
    properties:
      mail.smtp.ssl.enable: true
```

创建 `client` 端默认用户配置文件 `application-commons-spring-boot-admin-client-user.yml`

```yaml
spring:
  security:
    user:
      # 配置默认用户
      name: admin
      password: admin
```

创建 `server` 端用户及`url`配置文件 `application-commons-spring-boot-admin-server.yml`

``` yaml
spring-boot-admin:
  server:
    # spring-boot-admin server 端口
    port: 8090
    # spring-boot-admin server 注册基本地址
    url: http://localhost:${spring-boot-admin.server.port}
    # spring-boot-admin server 用户名
    username: root
    # spring-boot-admin server 密码
    password: root
```

创建 `client` 集成 `spring-boot-admin` 的通用配置文件 `application-commons-spring-boot-admin-client.yml`

```yaml
spring:
  profiles:
    include:
      # 引入前面的俩个配置文件
      - commons-spring-boot-admin-client-user
      - commons-spring-boot-admin-server
  boot:
    admin:
      client:
        # 配置文档
        # http://codecentric.github.io/spring-boot-admin/current/#spring-boot-admin-client
        enabled: true
        # 服务端地址
        url: ${spring-boot-admin.server.url}
        # 服务端帐号
        username: ${spring-boot-admin.server.username}
        # 服务端密码
        password: ${spring-boot-admin.server.password}
        instance:
          metadata:
            # 配置登录本服务需要的帐号与密码,用于让 spring-boot-admin 登录本服务
            user.name: ${spring.security.user.name}
            user.password: ${spring.security.user.password}

management:
  endpoints:
    jmx:
      exposure:
        exclude: "*"
    web:
      exposure:
        include: "*"
        # 排除关闭服务端点
        exclude:
          - shutdown
  endpoint:
    health:
      # 只有用户登录才显示 health 详情
      show-details: when_authorized
    # 禁用关闭服务端点
    shutdown:
      enabled: false
```

将 `client` 端的 `spring-security` 配置类 `SpringSecurityConfig` 迁移到 `commons` 模块

在 `client` 端的启动类 `ClientApplication` 上添加一个注解配置 `@ImportAutoConfiguration(SpringSecurityConfig.class)` 表示应用这个配置类

修改 `client` 的配置文件 `application.yml` 为

```yaml
spring:
  application:
    # 项目名称
    name: client
  profiles:
    include:
      - commons-spring-boot-admin-client
```

修改 `server` 的配置文件 `application.yml` 为

```yaml
server:
  # 配置服务端口
  port: ${spring-boot-admin.server.port}
spring:
  profiles:
    include:
      - commons-mail
      - commons-spring-boot-admin-server
  application:
    # 应用名称,用于与其他应用进行区分
    name: spring-boot-admin
  security:
    user:
      # 配置默认用户
      name: ${spring-boot-admin.server.username}
      password: ${spring-boot-admin.server.password}
  boot:
    admin:
      notify:
        mail:
          # http://codecentric.github.io/spring-boot-admin/current/#mail-notifications
          enabled: true
          # 发件人
          from: Spring-Boot-Admin <${spring.mail.username}>
          # 收件人,如果存在多个,使用逗号分隔
          to: example@example.com
```

### 源码

[GitHub](https://github.com/ghthou/spring-boot-admin-samples)









