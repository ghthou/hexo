---
title: Spring MVC 数据绑定学习笔记
tags:
  - Java
  - Spring MVC
categories:
  - Java
date: 2018-01-13 13:07:00
---
> 这是我在 [慕课网](http://www.imooc.com/) 观看 [SpringMVC 数据绑定入门](http://www.imooc.com/learn/558) 所做的学习笔记
> 其中包含对 **List，Set，Map，JSON，XML 的数据绑定以及 PropertyEditor、Formatter、Converter 三种自定义类型转换器 **

### List 类型绑定
  - 特点
     - List 对象绑定需要建立一个 List 集合包装类

      ```java
      public class User {
          private int age;
          private String name;
          // 省略 setter getter
      }
      public class ListUserWrap {
          private List<User> userList;
          // 省略 setter getter
      }
      ```
     - List 的长度为前台传入的 ** 集合最大下标加 1**

      ```java
      @ResponseBody
      @RequestMapping(value = "/list2")
      public String list2(ListUserWrap listUserWrap) {
          return "listUserWrapSize:" + listUserWrap.getUserList().size() + "\tlistUserWrap:" + listUserWrap;
      }
      ```
  - 测试数据
      > http://localhost/list2?userList[0].name=a&userList[1].name=b
      > listUserWrapSize:2 listUserWrap:ListUserWrap(userList=[User(age=0, name=a), User(age=0, name=b)])
      > http://localhost/list2?userList[0].name=a&userList[1].name=b&userList[3].name=c
      > listUserWrapSize:4 listUserWrap:ListUserWrap(userList=[User(age=0, name=a), User(age=0, name=b), User(age=0, name=null), User(age=0, name=c)])

### Set 类型绑定
  - 特点
      - Set 对象绑定需要建立一个 Set 集合包装类
      - 需要先定义 Set 集合长度，并且前台传过来的 Set 长度不能越界，否者报错
      - ** 设置 Set 长度时需要注意重写对象的 hashCode，equals 方法，否者后面的会掩盖前面的 **

      ```java
      public class SetUserWrap {
          private Set<User> userSet;
          // 省略 setter getter
          public SetUserWrap() {
              userSet = new LinkedHashSet<>();
              userSet.add(new User());
              User user = new User();
              user.setName("b");
              userSet.add(user);
          }
      }
      ```
  - 测试数据
      > http://localhost/set?userSet[0].name=a 因为预先定义的第二个对象的 name 为 b，所以此处返回 b
      > setUserWrapSize:2 setUserWrap:SetUserWrap(userSet=[User(age=0, name=a), User(age=0, name=b)])
      > http://localhost/set?userSet[0].name=a&userSet[1].name=bbb
      > setUserWrapSize:2 setUserWrap:SetUserWrap(userSet=[User(age=0, name=a), User(age=0, name=bbb)])

### Map 类型绑定        
  - 特点
      - 建立一个 Map 包装类

      ```java
      public class MapUserWrap {
          private Map<String, User> userMap;
          // 省略 setter getter
      }
      @ResponseBody
      @RequestMapping(value = "/map")
      public String map(MapUserWrap mapUserWrap) {
          return "mapUserWrapSize:" + mapUserWrap.getUserMap().size() + "\tmapUserWrap:" + mapUserWrap;
      }
      ```
  - 测试数据
      > http://localhost/map?userMap["a"].name=a&userMap["b"].name=b
      > mapUserWrapSize:2 mapUserWrap:MapUserWrap(userMap={a=User(age=0, name=a), b=User(age=0, name=b)})
      > http://localhost/map?userMap["a"].name=a&userMap["a"].name=b
      > mapUserWrapSize:1 mapUserWrap:MapUserWrap(userMap={a=User(age=0, name=a,b)})

### JSON 类型绑定
  - 前台在 body 区域传入以下类型格式字符串

  ```javascript
  {
      "name":"a",
      "age":1
  }
  ```
  ```java
  @ResponseBody
  @RequestMapping(value = "/json")
  public String json(@RequestBody User user) {
      return user.toString();
  }
  ```
### XML 类型绑定
  - 前台在 body 区域传入以下类型格式字符串

  ```xml
  <xmluser>
      <name>a</name>
      <age>1</age>
  </xmluser>
  ```
  - 建立一个 XML 包装类

  ```java
  @XmlRootElement(name = "xmluser")// 根节点标签名
  public class XmlUser {
      @XmlElement(name = "name")// 属性标签名
      private String name;
      @XmlElement(name = "age")// 属性标签名
      private int age;
  }
  ```
  ```java
  @ResponseBody
  @RequestMapping(value = "/xml")
  public String xml(@RequestBody XmlUser xmlUser) {
      return xmlUser.toString();
  }
    ```
### 自定义类型转换
#### 自定义类型转换 - PropertyEditor
  > 在 Controller 中编写一个带有 InitBinder 注解的方法，传入 WebDataBinder 对象，使用该对象注册指定类型的转换关系，对该方法所在 Controller 中使用该类型的方法参数有效

  ```java
  @ResponseBody
  @RequestMapping(value = "/date")
  public String date(Date date) {
      return date.toLocaleString();
  }
  @ResponseBody
  @RequestMapping(value = "/date2")
  public String date2(Date date2) {
      return date2.toLocaleString();
  }

  @InitBinder("date")// 指定对变量名为 date 进行转换, 不会对 date2 生效
  private void initDate(WebDataBinder binder) {
      binder.registerCustomEditor(Date.class, new CustomDateEditor(new SimpleDateFormat("yyyy-MM-dd"), true));
      //binder.registerCustomEditor(Class<?> requiredType, PropertyEditor propertyEditor)
      //new CustomDateEditor(DateFormat dateFormat, boolean allowEmpty)
      //allowEmpty 是否允许 Date 对象为空
  }
  ```
#### 自定义类型转换 - Formatter
  > 根据 `String` 类型自定义转换规则转换成需要的类型，需要实现 `org.springframework.format.Formatter<T>` 接口，`T`为想要转换的结果类型

  - 创建自定义 Formatter

  ```java
  public class DateFormatter implements Formatter<Date> {
      @Override
      public Date parse(String text, Locale locale) {
          SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
          Date date = null;
          try {
              date = dateFormat.parse(text);
          } catch (ParseException e) {
              e.printStackTrace();
          }
          return date;
      }

      @Override
      public String print(Date object, Locale locale) {
          return null;
      }
  }
  ```
  - 将自定义的 Formatter 注入到 SpringMVC 默认的 FormattingConversionServiceFactoryBean 中，同时将默认转换规则服务类配置为已经被注入的 bean 对象

  ```xml
  <mvc:annotation-driven conversion-service="myFormattingConversionService"/>
  <bean id="myFormattingConversionService" class="org.springframework.format.support.FormattingConversionServiceFactoryBean">
      <property name="formatters">,<!-- 此处为 formatters-->
          <set>
              <bean class="{包路径}.DateFormatter"></bean>
          </set>
      </property>
  </bean>
  ```

#### 自定义类型转换 - Converter
  > 自己指定数据来源类型及转换结果类型，相比 Formatter 更为灵活，需要实现 `org.springframework.core.convert.converter.Converter<S, T>` 接口，`S`为来源类型，`T`为结果类型

  - 创建自定义 Converter

  ```java
  public class DateConverter implements Converter<String, Date> {
      @Override
      public Date convert(String source) {
          SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
          Date date = null;
          try {
              date = dateFormat.parse(source);
          } catch (ParseException e) {
              e.printStackTrace();
          }
          return date;
      }
  }
  ```
  - 将自定义的 Converter 注入到 SpringMVC 默认的 FormattingConversionServiceFactoryBean 中，同时将默认转换规则服务类配置为已经被注入的 bean 对象

  ```xml
  <mvc:annotation-driven conversion-service="myFormattingConversionService"/>
  <bean id="myFormattingConversionService" class="org.springframework.format.support.FormattingConversionServiceFactoryBean">
      <property name="converters"><!-- 此处为 converters-->
          <set>
              <bean class="{包路径}.DateConverter"></bean>
          </set>
      </property>
  </bean>
  ```
