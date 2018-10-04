---
title: IntelliJ IDEA 使用 lombok 减少冗长代码
tags:
  - Java
  - IntelliJ IDEA
  - Lombok
categories:
  - Java
date: 2018-01-13 13:14:00
---
> 对于 POJO, 我们需要为其中的每个字段生成 getter,setter 方法, 虽然能够使用 IDE 快速为我们生成. 但如果需要修改字段名称及字段类型, 那么就需要删除在重新进行生成, 终究还是有一些不方便. 如果使用 lombok, 可以通过一些简单的注解直接生成我们所需要的代码, 能极大的提高开发体验.

1. 安装插件
![IDEA 安装 lombok.png](/images/IntelliJ-IDEA-使用-lombok-减少冗长代码/IDEA安装lombok.png)
2. 启用插件
![启用插件. png](/images/IntelliJ-IDEA-使用-lombok-减少冗长代码/启用插件.png)
3. 在项目中使用
    ```xml
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.16.14</version>
        <scope>provided</scope>
    </dependency>
    ```
4. lombok 常用注解介绍
    1. [@NonNull](https://projectlombok.org/features/NonNull.html)
        > 使用 @NonNull 注解修饰的字段 通过 set 方法设置时如果为 null, 将抛出 NullPointerException

    2. [@Cleanup](https://projectlombok.org/features/Cleanup.html)
        > 主要用来修饰 IO 流相关类, 会在 finally 代码块中对该资源进行 close();
        
    3. [@Getter,@Setter](https://projectlombok.org/features/GetterSetter.html)
        > 为字段生成 getter,setter 方法, 标记到类上表明为所有字段生成
        
    4. [@ToString](https://projectlombok.org/features/ToString.html)
        > 生成 toString 方法, 默认打印所有非静态字段
        
    5. [@EqualsAndHashCode](https://projectlombok.org/features/EqualsAndHashCode.html)
        > 生成 equals 和 hashCode 方法
        
    6. [@NoArgsConstructor,@RequiredArgsConstructor,@AllArgsConstructor](https://projectlombok.org/features/Constructor.html)
        > NoArgsConstructor 无参构造函数
        > RequiredArgsConstructor 为未初始化的 final 字段和使用 @NonNull 标注的字段生成构造函数
        > AllArgsConstructor 为所有字段生成构造函数
    7. [@Data](https://projectlombok.org/features/Data.html)
        > 相当于同时使用 @Getter,@Setter,@ToString,@EqualsAndHashCode,@RequiredArgsConstructor
    8. [@Value](https://projectlombok.org/features/Value.html)
        > 使用后, 类将使用 final 进行修饰, 同时使用 @ToString,@EqualsAndHashCode,@AllArgsConstructor,@Getter
        
    9. [@Builder](https://projectlombok.org/features/Builder.html)
        > 创建一个静态内部类, 使用该类可以使用链式调用创建对象
        > 如 User 对象中存在 name,age 字段, User user=User.builder().name("姓名").age(20).build()
    10. [@SneakyThrows](https://projectlombok.org/features/SneakyThrows.html)
        > 对标注的方法进行 try catch 后抛出异常, 可在 value 输入需要 catch 的异常数组, 默认 catch Throwable
        
    11. [@Synchronized](https://projectlombok.org/features/Synchronized.html)
        > 在标注的方法内 使用 synchronized(\$lock) {} 对代码进行包裹 ,$lock 为 new Object[0]
    12. [@Log,@CommonsLog,@JBossLog,@Log,@Log4j,@Log4j2,@Slf4j,@XSlf4j](https://projectlombok.org/features/Log.html)
        > 生成一个当前类的日志对象, 可以使用 topic 指定要获取的日志名称

5. 自定义配置        
    > 虽然 lombok 能为我们快速生成代码, 但是有一些生成的代码依然无法满足我们的需求. 此时可配置 lombok.config 来解决问题

    以下列出一些常用的配置
    ```config
    lombok.getter.noIsPrefix=true(默认: false)  #lombok 默认对 boolean 类型字段生成的 get 方法使用 is 前缀, 通过此配置则使用 get 前缀
    lombok.accessors.chain=true(默认: false) #默认的 set 方法返回 void 设置为 true 返回调用对象本身
    lombok.accessors.fluent=true(默认: false) #如果设置为 true. get,set 方法将不带 get,set 前缀, 直接以字段名为方法名
    lombok.log.fieldName=logger(默认: log) #设置 log 类注解返回的字段名称
    ```
    ** 注 **: 在 IDEA 中,`lombok.config` 文件 请放置于 `src\main\java` 目录下, 在 `src\main\resources` 中将不生效
6. 参考资料
    1. [lombok 官网](https://projectlombok.org/)
    2. [IDEA lombok 插件](https://github.com/mplushnikov/lombok-intellij-plugin)
