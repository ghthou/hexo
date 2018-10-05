---
title: Spring MVC 配置接收 String 参数时自动去除前后空格
date: 2018-10-04 20:49:17
tags:
  - Java
  - Spring MVC
  - Spring Boot
categories:
  - Java
---

在接收 `String` 类型参数时,前后可能存在一些空格,如果未曾去除就直接保存的话,可能会对一些特殊的业务场景造成致命影响.为了杜绝这种情况,需要在接收参数时进行前后空格清除处理

而接收`String`参数主要存在俩种情况

### 配置

#### 接收`url`或`form`表单中的参数

对于这种情况,`Spring MVC` 提供了一个 `org.springframework.beans.propertyeditors.StringTrimmerEditor` 类,我们只需要在参数绑定中进行注册就行,方式如下

```java
@ControllerAdvice
public class ControllerStringParamTrimConfig {

    @InitBinder
    public void initBinder(WebDataBinder binder) {
        // 创建 String trim 编辑器
      	// 构造方法中 boolean 参数含义为如果是空白字符串,是否转换为null
      	// 即如果为true,那么 " " 会被转换为 null,否者为 ""
        StringTrimmerEditor propertyEditor = new StringTrimmerEditor(false);
        // 为 String 类对象注册编辑器
        binder.registerCustomEditor(String.class, propertyEditor);
    }
}
```

#### 接收`Request Body`中`JSON`或`XML`对象参数

在这里,`Spring MVC` 是使用 `Jackson` 对参数进行反序列化,所以对于 `String` 的处理是在 `Jackson` 中配置

[如何自定义 Jackson ObjectMapper](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/html/howto-spring-mvc.html#howto-customize-the-jackson-objectmapper)

```java
@Bean
public Jackson2ObjectMapperBuilderCustomizer jackson2ObjectMapperBuilderCustomizer() {
    return new Jackson2ObjectMapperBuilderCustomizer() {
        @Override
        public void customize(Jackson2ObjectMapperBuilder jacksonObjectMapperBuilder) {
          	// 为 String 类型自定义反序列化操作
            jacksonObjectMapperBuilder
                    .deserializerByType(String.class, new StdScalarDeserializer<String>(String.class) {
                        @Override
                        public String deserialize(JsonParser jsonParser, DeserializationContext ctx)
                                throws IOException {
                          	// 去除前后空格
                            return StringUtils.trimWhitespace(jsonParser.getValueAsString());
                        }
                    });
        }
    };
}
```

### 测试

#### Controller

```java
@RestController
public class SpringTrimController {

    @GetMapping("/url")
    public String urlParam(String name) {
        return name;
    }

    @PostMapping("/form")
    public User formParam(User u) {
        return u;
    }

    @PostMapping(value = "/body")
    public User bodyParam(@RequestBody User u) {
        return u;
    }
}
```

#### 对象

```java
public class User {

    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

#### 测试类

```java
@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureMockMvc
public class SpringMvcStringTrimSamplesApplicationTests {

    @Autowired
    private MockMvc mockMvc;

    private static final String PARAM = " test ";
    private static final String RESULT = PARAM.trim();

    @Test
    public void url() throws Exception {
        mockMvc.perform(MockMvcRequestBuilders.get("/url?name=" + PARAM))
                .andDo(MockMvcResultHandlers.print())
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.content().string(RESULT));
    }

    @Test
    public void form() throws Exception {
        mockMvc.perform(MockMvcRequestBuilders.post("/form")
                .contentType(MediaType.APPLICATION_FORM_URLENCODED)
                .content("name=" + PARAM)
        )
                .andDo(MockMvcResultHandlers.print())
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.content().contentType(MediaType.APPLICATION_JSON_UTF8_VALUE))
                .andExpect(MockMvcResultMatchers.jsonPath("$.name").value(RESULT));
    }

    @Test
    public void body() throws Exception {
        String json = "{\"name\":\"" + PARAM + "\"}";
        mockMvc.perform(MockMvcRequestBuilders.post("/body")
                .contentType(MediaType.APPLICATION_JSON_UTF8_VALUE)
                .content(json)
        )
                .andDo(MockMvcResultHandlers.print())
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.content().contentType(MediaType.APPLICATION_JSON_UTF8_VALUE))
                .andExpect(MockMvcResultMatchers.jsonPath("$.name").value(RESULT));
    }

    @Test
    public void xmlBody() throws Exception {
        String json = "<root><name>" + PARAM + "</name></root>";
        mockMvc.perform(MockMvcRequestBuilders.post("/body")
                .contentType(MediaType.APPLICATION_XML_VALUE)
                .content(json)
        )
                .andDo(MockMvcResultHandlers.print())
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.content().contentType(MediaType.APPLICATION_JSON_UTF8_VALUE))
                .andExpect(MockMvcResultMatchers.jsonPath("$.name").value(RESULT));
    }
}

```

### 重构

对与这种所有项目都需要的通用配置,我们应该抽取一个公共模块,然后通过引入依赖来实现自动配置

创建 `commons` 模块

创建自动配置类 `WebMvcStringTrimAutoConfiguration`

```java
@Configuration
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class})
@AutoConfigureAfter(WebMvcAutoConfiguration.class)
public class WebMvcStringTrimAutoConfiguration {

    @ControllerAdvice
    public static class ControllerStringParamTrimConfig {

        @InitBinder
        public void initBinder(WebDataBinder binder) {
            // 创建 String trim 编辑器
            // 构造方法中 boolean 参数含义为如果是空白字符串,是否转换为null
            // 即如果为true,那么 " " 会被转换为 null,否者为 ""
            StringTrimmerEditor propertyEditor = new StringTrimmerEditor(false);
            // 为 String 类对象注册编辑器
            binder.registerCustomEditor(String.class, propertyEditor);
        }
    }

    @Bean
    public Jackson2ObjectMapperBuilderCustomizer jackson2ObjectMapperBuilderCustomizer() {
        return new Jackson2ObjectMapperBuilderCustomizer() {
            @Override
            public void customize(Jackson2ObjectMapperBuilder jacksonObjectMapperBuilder) {
                jacksonObjectMapperBuilder
                        .deserializerByType(String.class, new StdScalarDeserializer<String>(String.class) {
                            @Override
                            public String deserialize(JsonParser jsonParser, DeserializationContext ctx)
                                    throws IOException {
                                return StringUtils.trimWhitespace(jsonParser.getValueAsString());
                            }
                        });
            }
        };
    }
}

```

配置引入依赖后存在 `SpringBootApplication` ,`EnableAutoConfiguration` 注解时自动配置

在 `resurces` 创建 `META-INF/spring.factories` 文件

```
# 自动配置
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.github.ghthou.springmvcstringtrimsamples.commons.autoconfigure.WebMvcStringTrimAutoConfiguration
```

### 源码

[GitHub](https://github.com/ghthou/spring-mvc-string-trim-samples)









