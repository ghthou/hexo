---
title: Spring Boot 项目启动异常时使用邮件通知
date: 2018-09-25 19:16:47
tags:
  - Java
  - Spring Boot
categories:
  - Java
---

在本机进行 Spring Boot 项目开发时，如果启动时出现异常，可以在控制台中明确得知异常相关信息然后进行处理.但在远程的测试、生产服务器中，一般是后台启动或者通过 CI 启动项目，无法直接查看控制台中输出的日志信息. 此时如果启动过程中出现异常我们无法及时得知

但是通过 [Spring Boot  文档](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-spring-application.html#boot-features-application-events-and-listeners)可知, 可以通过监听 `ApplicationFailedEvent` 事件进行启动异常处理

> 6 . An `ApplicationFailedEvent` is sent if there is an exception on startup.

现在我们希望在出现这些异常时, 发送一封邮件到我预定义的邮箱中

#### 创建邮件工具类

发送邮件我们使用 `spring-boot-starter-mail` 提供的 `JavaMailSender` 接口, 通过配置 `spring.mail` 系列参数 Spring Boot 会自动创建 `JavaMailSender` 

但是可能存在发生异常时 `JavaMailSender` 还没有创建,所以最好是由我们手动创建 `JavaMailSender` 对象

以下是发送邮件工具类 `EmailUtils`

```java
package com.github.ghthou.startexceptionnotifications.samples.util;

import java.text.MessageFormat;
import java.util.Properties;

import org.apache.commons.lang3.StringUtils;
import org.apache.commons.lang3.exception.ExceptionUtils;
import org.springframework.boot.context.event.ApplicationFailedEvent;
import org.springframework.core.io.ClassPathResource;
import org.springframework.core.io.support.PropertiesLoaderUtils;
import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.JavaMailSenderImpl;

import com.github.ghthou.startexceptionnotifications.samples.properties.NotificationsProperties;

import lombok.SneakyThrows;

public class EmailUtils {

    public static final NotificationsProperties NOTIFICATIONS_PROPERTIES;
    public static final JavaMailSender JAVA_MAIL_SENDER;

    static {
        NOTIFICATIONS_PROPERTIES = initNotificationsProperties();
        JAVA_MAIL_SENDER = initJavaMailSender();
    }

    private static JavaMailSender initJavaMailSender() {
        JavaMailSenderImpl sender = new JavaMailSenderImpl();
        applyProperties(sender);
        return sender;
    }
    private static void applyProperties(JavaMailSenderImpl sender) {
        NotificationsProperties properties = NOTIFICATIONS_PROPERTIES;

        sender.setHost(properties.getHost());
        if (properties.getPort() != null) {
            sender.setPort(properties.getPort());
        }
        sender.setUsername(properties.getUsername());
        sender.setPassword(properties.getPassword());
        sender.setProtocol(properties.getProtocol());
        if (properties.getDefaultEncoding() != null) {
            sender.setDefaultEncoding(properties.getDefaultEncoding());
        }
        if (!properties.getProperties().isEmpty()) {
            sender.setJavaMailProperties(asProperties(properties.getProperties()));
        }
    }

    private static Properties asProperties(String source) {
        Properties properties = new Properties();
        for (String pro : StringUtils.split(source, ",")) {
            String[] split = StringUtils.split(pro, "=");
            if (split.length == 2) {
                properties.put(split[0], split[1]);
            }
        }
        return properties;
    }

    @SneakyThrows
    private static NotificationsProperties initNotificationsProperties() {
        Properties p = PropertiesLoaderUtils.loadProperties(new ClassPathResource("notifications.properties"));
        NotificationsProperties notificationsProperties = new NotificationsProperties();
        notificationsProperties.setAppName(p.getProperty("notifications.appName"));
        notificationsProperties.setTo(p.getProperty("notifications.to"));
        notificationsProperties.setHost(p.getProperty("notifications.host"));
        notificationsProperties.setPort(Integer.valueOf(p.getProperty("notifications.port")));
        notificationsProperties.setProtocol(p.getProperty("notifications.protocol"));
        notificationsProperties.setUsername(p.getProperty("notifications.username"));
        notificationsProperties.setPassword(p.getProperty("notifications.password"));
        notificationsProperties.setDefaultEncoding(p.getProperty("notifications.defaultEncoding"));
        notificationsProperties.setProperties(p.getProperty("notifications.properties"));
        return notificationsProperties;
    }

    private static JavaMailSender getJavaMailSender() {
        return JAVA_MAIL_SENDER;
    }

    public static void send(SimpleMailMessage simpleMessage) {
        getJavaMailSender().send(simpleMessage);
    }

    public static SimpleMailMessage createSimpleMailMessage(ApplicationFailedEvent event) {
        SimpleMailMessage simpleMessage = new SimpleMailMessage();
        // 发件人
        simpleMessage.setFrom(MessageFormat.format("Spring Boot 启动异常 <{0}>", NOTIFICATIONS_PROPERTIES.getUsername()));
        // 收件人
        simpleMessage.setTo(NOTIFICATIONS_PROPERTIES.getTo());
        // 主题
        simpleMessage.setSubject(MessageFormat.format("{0} 启动异常", NOTIFICATIONS_PROPERTIES.getAppName()));
        // 内容
        simpleMessage.setText(ExceptionUtils.getStackTrace(event.getException()));
        return simpleMessage;
    }

}

```

#### 创建启动异常监听器

然后创建一个 Spring 监听器 `StartExceptionNotificationsListener`,监听启动异常, 然后判断当前环境,如果不是 `dev` 环境,进行异常通知

```java
package com.github.ghthou.startexceptionnotifications.samples.listener;

import org.springframework.boot.context.event.ApplicationFailedEvent;
import org.springframework.context.ApplicationListener;
import org.springframework.context.ConfigurableApplicationContext;

import com.github.ghthou.startexceptionnotifications.samples.util.EmailUtils;

public class StartExceptionNotificationsListener implements ApplicationListener<ApplicationFailedEvent> {

    @Override
    public void onApplicationEvent(ApplicationFailedEvent event) {
        ConfigurableApplicationContext applicationContext = event.getApplicationContext();
        // 如果不是 dev 环境,因为 dev 环境会查看控制台
        if (applicationContext == null || applicationContext.getEnvironment().acceptsProfiles("!dev")) {
            // 进行异常通知
            EmailUtils.send(EmailUtils.createSimpleMailMessage(event));
        }
    }
}

```

#### 测试

配置自动注册 `StartExceptionNotificationsListener`

在 `resources/META-INF/spring.factories` 中进行以下配置

```
org.springframework.context.ApplicationListener=\
com.github.ghthou.startexceptionnotifications.samples.listener.StartExceptionNotificationsListener
```

创建用于模拟启动过程中出现异常的启动监听器 `ApplicationStartingEventListener`,`ApplicationEnvironmentPreparedEventListener` 等

然后在 `SpringApplication` 手动添加这些事件

```java
package com.github.ghthou.startexceptionnotifications.samples;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class StartExceptionNotificationsApplication {

    public static void main(String[] args) {
        SpringApplication springApplication = new SpringApplication(StartExceptionNotificationsApplication.class);
        // 不触发
        // springApplication.addListeners(new ApplicationStartingEventListener());
        // 触发
        // springApplication.addListeners(new ApplicationEnvironmentPreparedEventListener());
        // 触发
        // springApplication.addListeners(new ApplicationPreparedEventListener());
        // 触发
        // springApplication.addListeners(new ApplicationStartedEventListener());
        // 不触发
        // springApplication.addListeners(new ApplicationReadyEventListener());
        springApplication.run(args);
    }
}

```

#### 测试结果

最后通过测试结果可知

- [ ] ApplicationStartingEvent
- [x] ApplicationEnvironmentPreparedEvent
- [x] ApplicationPreparedEvent
- [x] ApplicationStartedEvent
- [ ] ApplicationReadyEvent

三个事件会进行 `ApplicationFailedEvent` 事件处理, 不过 `ApplicationStartingEvent` 事件一般不会产生异常, 而 `ApplicationReadyEvent` 是启动完成后触发的事件

#### 拓展

目前是使用 email 通知, 如果希望使用 短信,微信,钉钉 通知的话, 在 `StartExceptionNotificationsListener` 中自定义通知处理即可

#### 源码

[GitHub](https://github.com/ghthou/spring-boot-start-exception-notifications-samples)



