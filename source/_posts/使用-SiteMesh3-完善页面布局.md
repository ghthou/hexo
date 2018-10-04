---
title: 使用 SiteMesh3 完善页面布局
tags:
  - Java
  - SiteMesh
categories:
  - Java
date: 2018-01-13 12:54:00
---
> 在网页开发中, 大部分网页都具有相同的页头, 页尾, 菜单等模块. 一般情况下我们会将这些共用的代码单独抽取成一个页面, 然后进行包含. 虽然这样能够达到代码复用的效果, 但是如果引入的页面过多, 一来会带来修改不变的效果, 二来依然会形成多个页面使用相同的代码 (页面包含代码), 此时我们可以使用 SiteMesh3 来妥善解决这个问题

### 最初页面形式

![最初页面形式](/images/使用-SiteMesh3-完善页面布局/最初页面形式.png)

### 使用 JSP 进行页面包含

![页面包含](/images/使用-SiteMesh3-完善页面布局/页面包含.png)

进行页面包含后, 能达到一些共用页面的代码复用, 但是如果页面布局复杂的话, 会存在大量的页面包含代码, 依然给我们带来了不便
### SiteMesh3 方式
#### SiteMesh3 使用演示
使用 SiteMesh3 时需要先定义一个装饰器, 在这个装饰器中我们可以定义页面的布局, 然后配置动态内容输出位置即可
示例如下
```html
<!DOCTYPE>
<html lang="zh-cmn-Hans">
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8"/>
    
    <!--<sitemesh:write property="title"/> 会输出原始页面的 title 标签里面的内容 -->
    <title><sitemesh:write property="title"/></title>
    
    <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1"/>
    <meta name="renderer" content="webkit"/>
    <link rel="stylesheet" type="text/css" href="/static/css/web.css"/>
    
    <!--<sitemesh:write property="head"/> 会输出原始页面 head 标签里面的内容 (不包括 title 标签)-->
    <sitemesh:write property="head"/>
</head>
<body>
    <header>header</header>
    
    <!--<sitemesh:write property="body"/> 会输出原始页面 body 标签里面的内容 -->
    <sitemesh:write property="body"/>
    
    <footer>footer</footer>
    <script type="text/javascript" src="/static/js/web.js"></script>
</body>
</html>
```
假如此时存在一个这样的原始页面
```html
<!DOCTYPE>
<html>
<head>
    <title>title</title>
    <meta name="keywords" content="SiteMesh3">
</head>
<body>
        hello,world
</body>
</html>
```
经过 SiteMesh3 装饰后的页面如下
```html
<!DOCTYPE>
<html lang="zh-cmn-Hans">
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8"/>
    
    <title>title</title>
    
    <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1"/>
    <meta name="renderer" content="webkit"/>
    <link rel="stylesheet" type="text/css" href="/static/css/web.css"/>
    
    <meta name="keywords" content="SiteMesh3">
    
</head>
<body>

    <header>header</header>
        hello,world
    <footer>footer</footer>
    <script type="text/javascript" src="/static/js/web.js"></script>
</body>
</html>
```
可以发现使用 SiteMesh3 进行装饰能够让我们更加专注于一些与页面独有的代码逻辑, 能避免相同的代码在多个页面重复出现

#### SiteMesh3 使用说明
通过上面的一个小例子, 我们可以发现使用 SiteMesh3 需要一个装饰器页面. 由此可以牵扯出另外几个问题, 对哪些页面进行装饰? 使用哪个装饰页面装饰?
通过以下配置可以完成这些疑问
首先我们需要引入 SiteMesh3 相关的 jar 包
```xml
<dependency>
    <groupId>org.sitemesh</groupId>
    <artifactId>sitemesh</artifactId>
    <version>3.0.1</version>
</dependency>
```
其次, SiteMesh3 会对一些页面进行装饰, 所以我们需要添加一个过滤器来进行页面过滤
```xml
<filter>
    <filter-name>sitemesh</filter-name>
    <filter-class>org.sitemesh.config.ConfigurableSiteMeshFilter</filter-class>
</filter>
<filter-mapping>
    <filter-name>sitemesh</filter-name>
    <!-- 如果项目中的页面地址以  .do 或者 .action 结尾可以使用 *.do 或 *.action-->
    <url-pattern>/*</url-pattern>
</filter-mapping>
```
同时我们还需要配置装饰器, 要装饰的页面, 不需要装饰的页面等信息
在 `WEB-INF` 目录下创建一个 `sitemesh3.xml` 文件,请确保路径正确,SiteMesh3会根据 `/WEB-INF/sitemesh3.xml` 加载文件
文件内容如下
```xml
<?xml version="1.0" encoding="UTF-8"?>
<sitemesh>
    <!-- path 是指要进行装饰的页面 如 /index.jsp 对应 http://localhost:8080/index.jsp -->
    <!-- decorator 是指装饰器页面 /decorators/default.jsp 对应位置为 src/main/webapp/decorators/default.jsp--> 
    <!-- sitemesh3 会在 index.jsp 页面返回时提取 title head body 中的内容, 然后在 default.jsp 根据输出标签进行对应输出 -->
    <mapping path="/index.jsp" decorator="/decorators/default.jsp"/>
</sitemesh>
```
此时我们可以将上面演示中的文件内容建立 `index.jsp` 和 `default.jsp` 页面, 然后进行测试访问, 会发现能达到演示中的效果

#### SiteMesh3 高级配置
通过上面的简单配置我们可以实现一个最基本的页面装饰, 与此同时 SiteMesh3 还支持一些更为高级的配置

- 默认装饰器
```xml
<!-- 如果不填写 path 路径, 则 SiteMesh3 会在找不到匹配的装饰器时, 使用这个装饰器进行装饰 -->
<!-- 推荐在该页面中定义一些全网站共用的代码, 如: 网站统计相关脚本, 浏览器渲染方式等 meta 标签 -->
<mapping decorator="/decorators/default.jsp"/>
```
- 多个装饰器
```xml
<mapping>
    <path>/multi.jsp</path>
    <decorator>/decorators/multi_1.jsp</decorator>
    <decorator>/decorators/multi_2.jsp</decorator>
    <decorator>/decorators/multi_3.jsp</decorator>
</mapping>
```

装饰器会按照配置的先后顺序进行装饰
假如 
multi.jsp 中 body 内容为 0,
multi_1.jsp 中内容为 1 同时在下面输出原始页面中的body内容
```html
<body>
    <h3>1</h3>
    <sitemesh:write property="body"/>
</body>
```
multi_2.jsp 中内容为 2 同时在下面输出原始页面中的body内容

最终返回的页面内容为
2
1
0


- 不需要装饰
```
<!-- SiteMesh3 会先判断是否不需要装饰, 然后再判断是否存在匹配的装饰器 -->
<!-- /exclude/* 会对所有以 /exclude 开头的 url 都不会进行装饰 包括 /exclude/index.jsp,/exclude/item/index.jsp -->
<mapping path="/exclude/*" exclue="true"/>
```
- MIME 类型
```xml
<!-- 
默认情况下, SiteMesh3 只对响应头 Content-Type 中包含 text/html 的页面进行装配
如果会需要装配其他格式的页面, 添加多个 mime-type 标签即可
-->
<mime-type>text/html</mime-type>
<mime-type>application/vnd.wap.xhtml+xml</mime-type>
<mime-type>application/xhtml+xml</mime-type>
```
- 自定义输出标签
```xml
<!--
输出标签格式 <sitemesh:write property="body"/>
SiteMesh3 默认支持 title, head, body 三个标签, 如果需要支持其他标签, 可以通过实现 org.sitemesh.content.tagrules.TagRuleBundle 接口中的 install 方法来完成对标签的拓展
-->
<content-processor>
    <tag-rule-bundle class="com.github.ghthou.learning.sitemesh3.ExpandTagRuleBundle"/>
</content-processor>
```
ExpandTagRuleBundle 代码如下
```java
public class ExpandTagRuleBundle implements TagRuleBundle {

	@Override
	public void install(State defaultState, ContentProperty contentProperty, SiteMeshContext siteMeshContext) {
		defaultState.addRule("header", new ExportTagToContentRule(siteMeshContext, contentProperty.getChild("header"), false));
		defaultState.addRule("menu", new ExportTagToContentRule(siteMeshContext, contentProperty.getChild("menu"), false));
		defaultState.addRule("footer", new ExportTagToContentRule(siteMeshContext, contentProperty.getChild("footer"), false));
	}

	@Override
	public void cleanUp(State defaultState, ContentProperty contentProperty, SiteMeshContext siteMeshContext) {

	}
}
```
此时可以使用 `<sitemesh:write property="header"/>` 标签在装饰器中输出原始页面的 header 标签值

- 其他说明
    - `title`, `head`, `body` 标签的实现请参考             `org.sitemesh.content.tagrules.html.CoreHtmlTagRuleBundle`
    - `title`, `head`, `body` 等自定义标签只会提取原始页面中的第一个找到的标签
    - 默认配置文件路径为 `/WEB-INF/sitemesh3.xml` ,如果需要自定义配置路径,请在配置 `filter` 时配置 `configFile` 属性,如将 `sitemesh3.xml` 配置文件放在 `resources` 文件夹中
    
    ```xml
    <filter>
        <filter-name>sitemesh</filter-name>
        <filter-class>org.sitemesh.config.ConfigurableSiteMeshFilter</filter-class>
        <!--设置配置文件路径 默认为 /WEB-INF/sitemesh3.xml -->
        <init-param>
            <param-name>configFile</param-name>
            <param-value>/WEB-INF/classes/sitemesh3.xml</param-value>
        </init-param>
    </filter>
    ```
    - 如果装饰器是 `html` 文件会存在中文乱码的问题, 在 SpringMVC 中可在 `web.xml` 文件配置如下过滤器

    ```xml
    <filter>
        <filter-name>CharacterEncodingFilter</filter-name>
        <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
        <init-param>
            <param-name>encoding</param-name>
            <param-value>utf-8</param-value>
        </init-param>
        <init-param>
            <param-name>forceEncoding</param-name>
            <param-value>true</param-value>
        </init-param>
    </filter>
    <filter-mapping>
        <filter-name>CharacterEncodingFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
    ```
    - 在 SpringMVC 中因为 SiteMesh3 需要直接访问到装饰页面, 所以需要增加如下标签
    
    ```xml
    <mvc:default-servlet-handler/>
    <!-- 或者配置 url 与文件夹映射关系 -->
    <mvc:resources mapping="/decorators/**" location="/decorators/"/>
    ```
    - 如果装饰器为 JSP 页面, 则可以使用 EL 表达式获取原始页面中的属性
假如在 SpringMVC 中设置如下属性
    ```java
	@RequestMapping(value = "/index")
	public String index(Model model) {
		model.addAttribute("date", new Date().toLocaleString());
		return "index";
	}
    ```
        则可以在方法返回页面 `/index.jsp` 和该 url 的装饰页面 `/decorators/default.jsp` 中使用 EL 表达式获取 `date` 属性

### 参考资料
[SiteMesh3 官网文档](http://wiki.sitemesh.org/wiki/display/sitemesh3/Home)
[SiteMesh3 GitHub](https://github.com/sitemesh/sitemesh3)
