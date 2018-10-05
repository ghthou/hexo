---
title: Jsoup 防止 XSS 攻击
tags:
  - Java
  - 安全
categories:
  - Java
date: 2018-01-13 13:08:00
---
> 服务器处理富文本编辑器提交的内容时，因排版的需求不能对 HTML 标签进行转义，但为了防止 XSS 攻击，又必须过滤掉其中的 JS 代码，在 Java 中使用 Jsoup 正好可以满足此要求

### 实现原理
Jsoup 使用标签 ** 白名单 ** 的机制用来进行防止 XSS 攻击，假设白名单中只允许 p 标签存在，此时在一段 HTML 代码中，**只能存在 p 标签 **，其他标签将会被清除只保留被标签所包裹的内容，具体详情可查看参考资料
### 参考资料
[Jsoup 学习之 Whitelist 类](http://blog.csdn.net/xyw_blog/article/details/9145523)
### 项目依赖
 - jsoup-1.9.2
    ```xml
    <dependency>
        <groupId>org.jsoup</groupId>
        <artifactId>jsoup</artifactId>
        <version>1.9.2</version>
    </dependency>
    ```

### 代码示例
 - 创建进行测试的 HTML 代码
```java
String testHtml = "<div class='div'style='height: 100px;'>div 标签的内容 </div><p class='div'style='width: 50px;'>p 标签的内容 </p>";
```
 - 创建一个白名单对象
```java
Whitelist whitelist = new Whitelist();
```
 - 添加允许使用的标签，此时只允许 p 标签存在
```java
whitelist.addTags("p");
```
 - 对测试代码进行过滤，过滤规则就是创建的白名单
```java
String result1 = Jsoup.clean(testHtml, whitelist);
System.out.println(result1);// 输出:   div 标签的内容 < p>p 标签的内容 </p>
```
 - 此时我们发现 div 标签已经被过滤掉了，但是 p 标签中的属性也同时也被过滤掉了，因为白名单只允许了 p 标签，但是并未对属性加入白名单，此时将 p 标签中的 class 属性加入白名单中，再进行一次过滤
```java
whitelist.addAttributes("p","class");
String result2 = Jsoup.clean(testHtml, whitelist);
System.out.println(result2);// 输出:  div 标签的内容 < p class="div">p 标签的内容 </p>
```
 - 此时可见 class 属性已被允许存在，另外 `whitelist.addAttributes(String tag, String... keys)` 中的 keys 是一个可变数组，由此可知我们可以同时添加多个属性，如 `whitelist.addAttributes("p"，"class","style","title")`,`whitelist.addTags(String... tags)` 方法同理
 - 此时假如我在白名单中添加了多个标签，那么如何才能快速对所有标签添加共同属性
```java
whitelist.addAttributes(":all","style","title");
```
 - `:all` 表明给白名单中的所有标签添加 style，title 属性，此时我们将 div，h1 标签放入白名单，再进行测试
```java
whitelist.addTags("div","h1");
testHtml = "<h1 onclick='alert(1);'class='' style=''title=''>h1 内容 </h1><div class=''>div 内容 </div><p class='' style=''>p 内容 </p>";
String result3 = Jsoup.clean(testHtml, whitelist);
System.out.println(result3);// 输出:  <h1 style="title="">h1 内容 </h1><div>div 内容 </div><p class=''style="">p 内容 </p>
```
 - 结果分析
    1. 在 h1 标签中 onclick，class 属性被过滤掉了 因为他们不属于 h1 标签的白名单属性
    2. 在 div 标签中 class 属性被过滤掉了 理由同上
    3. 标签中的属性都被保留，因为 class 属性只添加 p 标签的白名单中 style 属性添加在所有标签的白名单中


- 但是在实际使用中，允许的标签往往很多，这时 jsoup 默认给我们提供了 5 个白名单对象

| 白名单对象           | 标签                        | 说明  |
|---                       |   ---                          |  ---  |
|none                     | 无                            | 只保留标签内文本内容 |
|simpleText           | b,em,i,strong,u         | 简单的文本标签 |
|basic         |a,b,blockquote,br,cite,code,dd,<br>dl,dt,em,i,li,ol,p,pre,q,small,span,<br>strike,strong,sub,sup,u,ul                   | 基本使用的标签 |
    |basicWithImages    | basic 的基础上添加了 img 标签 <br> 及 img 标签的 src,align,alt,height,width,title 属性                                               | 基本使用的加上 img 标签 |
|relaxed           |a,b,blockquote,br,caption,cite,<br>code,col,colgroup,dd,div,dl,dt,<br>em,h1,h2,h3,h4,h5,h6,i,img,li,<br>ol,p,pre,q,small,span,strike,strong,<br>sub,sup,table,tbody,td,tfoot,th,thead,tr,u,ul                  | 在 basicWithImages 的基础上又增加了一部分部分标签 |
 ** 如果没有图片上传的需求，使用 `basic`，否则使用 `basicWithImages`**

### 其他事项
在刚才测试的时候，会发现 Jsoup.clean()方法返回的代码已经被进行格式化，在标签及标签内容之间添加了 \n 回车符，如果不需要的话，可以使用 `Jsoup.clean(testHtml, "", whitelist, new Document.OutputSettings().prettyPrint(false));` 进行过滤

### 工具类
```java
import org.jsoup.Jsoup;
import org.jsoup.nodes.Document;
import org.jsoup.safety.Whitelist;
/**
 * 描述: 过滤 HTML 标签中 XSS 代码
 */
public class JsoupUtil {
    /** 
     * 使用自带的 basicWithImages 白名单
     * 允许的便签有 a,b,blockquote,br,cite,code,dd,dl,dt,em,i,li,ol,p,pre,q,small,span,strike,strong,sub,sup,u,ul,img  
     * 以及 a 标签的 href,img 标签的 src,align,alt,height,width,title 属性
     */
    private static final Whitelist whitelist = Whitelist.basicWithImages();
    /** 配置过滤化参数, 不对代码进行格式化 */
    private static final Document.OutputSettings outputSettings = new Document.OutputSettings().prettyPrint(false);
    static {
        // 富文本编辑时一些样式是使用 style 来进行实现的
        // 比如红色字体 style="color:red;"
        // 所以需要给所有标签添加 style 属性
        whitelist.addAttributes(":all", "style");
    }
    public static String clean(String content) {
        return Jsoup.clean(content, "", whitelist, outputSettings);
    }
}
```
