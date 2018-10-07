---
title: Spring Boot 项目启动异常时使用邮件通知
date: 2018-09-25 19:16:47
tags:
  - Java
  - Spring Boot
categories:
  - Java
---

在本机进行 Spring-Boot 项目开发时，如果启动时出现异常，我们可以在控制台中明确得知异常相关信息然后进行处理.但是在远程的测试，生产服务器中，如果是后台启动或者通过 CI 启动项目，我们无法直接查看控制台中输出的日志信息.此时如果启动过程中出现异常我们无法及时得知

现在我们可以使用 Spring-Boot 提供的 `FailureAnalysisReporter` 对启动过程中的异常进行处理，例如发送一份异常信息邮件到指定的邮箱中

### 实现思路

在 Spring-Boot 项目中如果启动过程中出现异常，Spring-Boot 会在控制台输出格式化的异常描述信息

比如常见的端口冲突，Spring-Boot 会输出如下信息

```log
2018-09-25 20:09:47.386 ERROR 12828 --- [           main]   o.s.b.d.LoggingFailureAnalysisReporter.report(LoggingFailureAnalysisReporter.java:42) : 

***************************
APPLICATION FAILED TO START
***************************

Description:

The Tomcat connector configured to listen on port 8080 failed to start. The port may already be in use or the connector may be misconfigured.

Action:

Verify the connector's configuration, identify and stop any process that's listening on port 8080, or configure this application to listen on another port.
```

可以发现 Spring-Boot 输出的异常信息相比异常栈更方便阅读，那么我们是否可以在 Spring-Boot 输出异常时同时将异常信息发送到指定邮箱中呢？

### 实现分析

通过上面的示例可知异常信息是由 `org.springframework.boot.diagnostics.LoggingFailureAnalysisReporter#report` 方法进行输出

```java
public void report(FailureAnalysis failureAnalysis) {
    if (logger.isDebugEnabled()) {
        logger.debug("Application failed to start due to an exception",
                failureAnalysis.getCause());
    }
    if (logger.isErrorEnabled()) {
        logger.error(buildMessage(failureAnalysis));
    }
}
```

通过 IDEA 的 `CTRL+B`（Declaration）快捷键可知该方法由`org.springframework.boot.diagnostics.FailureAnalyzers#report` 方法调用

```java
private boolean report(FailureAnalysis analysis, ClassLoader classLoader) {
    List<FailureAnalysisReporter> reporters = SpringFactoriesLoader
            .loadFactories(FailureAnalysisReporter.class, classLoader);
    if (analysis == null || reporters.isEmpty()) {
        return false;
    }
    for (FailureAnalysisReporter reporter : reporters) {
        reporter.report(analysis);
    }
    return true;
}
```

分析该方法的源代码，可以发现 Spring-Boot 会从 `META-INF/spring.factories` 中加载 `FailureAnalysisReporter` 的所有实例，然后循环调用 `report` 方法

而输出异常信息的 `LoggingFailureAnalysisReporter` 正是 `FailureAnalysisReporter` 的子类

所以我们可以在 `META-INF/spring.factories` 新增一个`FailureAnalysisReporter` 实例，在该实例的实现中发送异常信息到指定邮箱

### 具体实现

新建一个 `NotificationsFailureAnalysisReporter` 类并实现 `FailureAnalysisReporter` 接口

```java
public class NotificationsFailureAnalysisReporter implements FailureAnalysisReporter {

    /** 收件人key */
    private static final String START_EXCEPTION_NOTIFY_EMAIL_KEY = "x-start-exception.mail.to";

    private static final Logger log = LoggerFactory.getLogger(NotificationsFailureAnalysisReporter.class);

    @Override
    public void report(FailureAnalysis analysis) {
        // 获取 Environment 对象
        Environment environment = ApplicationTools.getEnvironment();
        // 如果当前环境不是开发环境
        if (!environment.acceptsProfiles(ProfileConstant.DEV)) {
            BeanFactory beanFactory = ApplicationTools.getBeanFactory();
            // 邮件发送 mailSender
            JavaMailSender mailSender = beanFactory.getBean(JavaMailSender.class);
            MailProperties mailProperties = beanFactory.getBean(MailProperties.class);

            // 应用名称
            String appName = environment.getProperty("spring.application.name");
            // 收件人
            String mailTo = environment.getProperty(START_EXCEPTION_NOTIFY_EMAIL_KEY);
            if (!StringUtils.hasText(mailTo)) {
                log.error("请配置接收启动异常信息的邮箱,配置参数为 {}", START_EXCEPTION_NOTIFY_EMAIL_KEY);
                return;
            }

            SimpleMailMessage mailMessage = new SimpleMailMessage();
            // 寄件人
            mailMessage.setFrom("Spring-Boot <" + mailProperties.getUsername() + ">");
            // 收件人
            mailMessage.setTo(mailTo);
            // 主题
            mailMessage.setSubject(appName + " 启动异常");
            // 内容
            mailMessage.setText(buildMessage(analysis));
            // 发送邮件
            mailSender.send(mailMessage);

        }

    }

    /**
     * 生成异常信息字符串
     *
     * @see LoggingFailureAnalysisReporter#buildMessage(org.springframework.boot.diagnostics.FailureAnalysis)
     */
    private String buildMessage(FailureAnalysis failureAnalysis) {
        StringBuilder builder = new StringBuilder();
        builder.append(String.format("***************************%n"));
        builder.append(String.format("%s%n", ApplicationTools.getEnvironment().getProperty("spring.application.name")));
        builder.append(String.format("***************************%n%n"));
        builder.append(String.format("Description:%n%n"));
        builder.append(String.format("%s%n", failureAnalysis.getDescription()));
        if (StringUtils.hasText(failureAnalysis.getAction())) {
            builder.append(String.format("%nAction:%n%n"));
            builder.append(String.format("%s%n", failureAnalysis.getAction()));
        }
        builder.append(String.format("%nException StackTrace:%n%n"));
        builder.append(String.format("%s%n", ExceptionUtils.getStackTrace(failureAnalysis.getCause())));
        return builder.toString();
    }
}

```

在当前项目的 `resources` 文件夹创建 `META-INF/spring.factories` 文件

在 `spring.factories` 文件中进行如下配置

```
org.springframework.boot.diagnostics.FailureAnalysisReporter=\
com.github.ghthou.startexceptionnotifications.diagnostics.NotificationsFailureAnalysisReporter
```

新建 `application.yml` 文件，配置如下

```yaml
spring:
  application:
    # 应用名称,用于与其他应用进行区分
    name: start-exception-notifications
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

# 接收异常信息的收件人帐号
x-start-exception:
  mail:
    to: example@example.com
```

### 测试

新建一个业务异常

```java
public class BusinessException extends RuntimeException {

    public BusinessException(String message) {
        super(message);
    }
}
```

新建一个启动异常监听器 `ThrowExceptionListener` ，用于模拟启动时异常

```java
public class ThrowExceptionListener implements ApplicationListener<ApplicationStartedEvent> {

    public static final String PROFILE = "throwException";

    @Override
    public void onApplicationEvent(ApplicationStartedEvent event) {
        if (event.getApplicationContext().getEnvironment().acceptsProfiles(PROFILE)) {
            throw new BusinessException("启动时出现异常");
        }
    }

}
```

在 `META-INF/spring.factories` 中追加以下配置

```
org.springframework.context.ApplicationListener=\
com.github.ghthou.startexceptionnotifications.listener.ThrowExceptionListener
```

在 `application.yml` 中追加以下配置，`test`表明为测试环境，`throwException`表明为启动时抛出异常

```yaml
spring:
  profiles:
    active: test,throwException
```

创建启动类

```java
@SpringBootApplication
public class StartExceptionHandleApplication {

    public static void main(String[] args) {
        SpringApplication.run(StartExceptionHandleApplication.class, args);
    }
}
```

此时启动 `main` 方法，发现控制台已经抛出自定义的 `BusinessException` 异常

但是并没有发送邮件，同时也没有输出 `org.springframework.boot.diagnostics.LoggingFailureAnalysisReporter#report` 中的格式化异常信息

通过 `debug` 进行分析，发现原因是 `org.springframework.boot.diagnostics.FailureAnalyzers#report`  中 `analysis` 参数为空导致后面的代码没有运行

```java
private boolean report(FailureAnalysis analysis, ClassLoader classLoader) {
    List<FailureAnalysisReporter> reporters = SpringFactoriesLoader
            .loadFactories(FailureAnalysisReporter.class, classLoader);
    if (analysis == null || reporters.isEmpty()) {
        return false;
    }
    for (FailureAnalysisReporter reporter : reporters) {
        reporter.report(analysis);
    }
    return true;
}
```

分析 `analysis` 参数的生成，原因是 Spring-Boot 默认的异常分析器中没有针对 `BusinessException` 的异常分析器.所以导致 `analysis` 为空

```java
private FailureAnalysis analyze(Throwable failure, List<FailureAnalyzer> analyzers) {
    for (FailureAnalyzer analyzer : analyzers) {
        try {
            FailureAnalysis analysis = analyzer.analyze(failure);
            if (analysis != null) {
                return analysis;
            }
        }
        catch (Throwable ex) {
            logger.debug("FailureAnalyzer " + analyzer + " failed", ex);
        }
    }
    return null;
}
```

### 修复

此时我们添加一个针对 `BusinessException` 异常的分析器

```java
public class BusinessExceptionFailureAnalyzer extends AbstractFailureAnalyzer<BusinessException> {

    @Override
    protected FailureAnalysis analyze(Throwable rootFailure, BusinessException cause) {
        return new FailureAnalysis(cause.getMessage(), cause.getMessage(), cause);
    }

}
```

在 `META-INF/spring.factories` 中追加以下配置

```
org.springframework.boot.diagnostics.FailureAnalyzer=\
com.github.ghthou.startexceptionnotifications.diagnostics.analyzer.BusinessExceptionFailureAnalyzer
```

由上可知，如果在启动过程出现未定义的异常 `FailureAnalyzer` ，则无法进行异常通知处理

理论上应该定义一个针对 `Exception` 异常的 `FailureAnalyzer` ,但是在 `Spring-Boot` 对异常的分析中，自定义的总是排在第一个(其他 `FailureAnalyzer` 的默认顺序为`Integer.MAX_VALUE`)，所以如果定义`Exception` 类型的  `FailureAnalyzer`  会导致 `Spring-Boot` 无法使用预定义的其他 `FailureAnalyzer` 

重新运行，发现还是无法发送邮件，通过 `debug` 发现是因为在 `com.github.ghthou.startexceptionnotifications.diagnostics.NotificationsFailureAnalysisReporter#report` 中 `Environment environment = ApplicationTools.getEnvironment();` 获取到的对象为 null 导致的

这是因为抛出异常的 `ThrowExceptionListener` 是在 `ApplicationPreparedEvent` 事件后触发的，根据 [Spring-Boot 应用事件和监听器](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-spring-application.html#boot-features-application-events-and-listeners) 可知此时上下文还没有刷新，所以 `ApplicationTools` 中的对象还没有注入导致的空指针异常

此时将 `ThrowExceptionListener` 所在的事件位置改为 `ApplicationStartedEvent` 即可解决该问题

由此可知如果异常是在 `ApplicationStartedEvent` 事件发生之前，则无法发送邮件

还有一种办法就是手动创建发送邮件的相关 `bean` ，比如解析配置文件中的邮件配置，然后创建 `JavaMailSender` 对象（见 `MailSenderPropertiesConfiguration`），再执行发送邮件操作，这样可以不依赖 `Environment`，`BeanFactory` 等对象

同时如果觉得邮件通知不够及时或不够多样，你也可以在 `NotificationsFailureAnalysisReporter.report` 方法中自定义通知处理，比如发送短信，微信，钉钉通知等

### 源码

[GitHub](https://github.com/ghthou/spring-boot-start-exception-notifications)



