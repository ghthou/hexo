---
title: 使用 gzip 压缩请求正文
tags:
  - Java
  - Gzip
categories:
  - Java
date: 2018-01-13 12:58:00
---
> 在一些论坛, 博客等项目中. 用户发送的帖子, 文章内容可能会存在太长的情况. 这时如果用户的网速不佳, 或者网络不稳定. 那么将会面临 ** 响应过慢、发送失败 ** 的情况. 如果网站还有自动保存的功能的话, 这种情况会明显增多. 这时如果将传输的内容在本地进行压缩上传, 然后在服务器进行解压. 对长文本的处理能够得到完好解决, 同时极大减少了移动端用户的网络开销.

本文创作思路来源于 [Jerry Qu](https://imququ.com) 的博客 [如何压缩 HTTP 请求正文](https://imququ.com/post/how-to-compress-http-request-body.html)

### 实现思路
在前台对请求正文使用 `pako_deflate.js` 进行本地 `gzip` 格式压缩
在后台使用 `Java` 对请求正文进行解压

### 操作环境
- jdk 1.8.0_77
- idea 2016.2.1
- maven 3.3.9

### 项目依赖
- commons-io-2.5 (简化 IO 操作)
- json-lib-2.4 (处理请求正文中的参数)
- spring-webmvc-4.3.4.RELEASE
- pako_deflate-1.0.3.js (JS 文本压缩工具类)

###JS 压缩请求正文
因为只在前台进行压缩, 所以只需引用 [pako](https://github.com/nodeca/pako) 的压缩专用文件 [pako_deflate.min.js](https://github.com/nodeca/pako/blob/master/dist/pako_deflate.min.js)
又因为我在项目中主要使用 jQuery 发送 Ajax 请求, 所以引入 jQuery
```js
<script src="jquery-2.2.4.min.js"></script>
<script src="pako_deflate.min.js"></script>
```
将发送的参数转换为 JSON 字符串
```js
var params = encodeURIComponent(JSON.stringify({
    title: "标题",
    content: "内容"
}));
```
gzip 虽然能极大的压缩请求正文. 但是如果内容过小, 压缩后内容反而会增大, 经测试, 对于 `params.length` 大于 1000 的文本压缩效果能够达到 **60%** 以上, 所以在压缩前, 需要对内容进行判断
```js
var params = encodeURIComponent(JSON.stringify({
    title: title,
    content: content
}));
var compressBeginLen = params.length;
if (compressBeginLen > 1000) {
    // 对 JSON 字符串进行压缩
    // pako.gzip(params) 默认返回一个 Uint8Array 对象, 如果此时使用 Ajax 进行请求, 参数会以数组的形式进行发送
    // 为了解决该问题, 添加 {to: "string"} 参数, 返回一个二进制的字符串
    params = pako.gzip(params, {to: "string"});
}
$.ajax({
    url: "/gzip",
    data: params,
    dataType: "text",
    type: "post",
    headers: {
        // 如果 compressBeginLen 大于 1000, 标记此次请求的参数使用了 gzip 压缩
        "Content-Encoding": params.length>1000?"gzip":""
    },
    success: function (data) {
        //dosomething
    }
})
```

###Java 解压请求正文
首先获取 `Content-Encoding` 请求头, 根据该请求头中的内容进行逻辑处理
```java
@ResponseBody
@RequestMapping(value = "/gzip")
public String gzip(HttpServletRequest request) {
    String params = "";
    try {
        // 获取 Content-Encoding 请求头
        String contentEncoding = request.getHeader("Content-Encoding");
        if (contentEncoding != null && contentEncoding.equals("gzip")) {
            // 获取输入流
            BufferedReader reader = request.getReader();
            // 将输入流中的请求实体转换为 byte 数组, 进行 gzip 解压
            byte[] bytes = IOUtils.toByteArray(reader, "iso-8859-1");
            // 对 bytes 数组进行解压
            params = GZIPUtil.uncompress(bytes);
        } else {
            BufferedReader reader = request.getReader();
            params = IOUtils.toString(reader);
        }
        if (params != null && params.trim().length() > 0) {
            // 因为前台对参数进行了 url 编码, 在此进行解码
            params = URLDecoder.decode(params, "utf-8");
            // 将解码后的参数转换为 json 对象
            JSONObject json = JSONObject.fromObject(params);
            // 从 json 对象中获取参数进行后续操作
            System.out.println("title:\t" + json.getString("title"));
            System.out.println("content:\t" + json.getString("content"));
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
    return params;
}
```
Java gzip 解压方法 `GZIPUtil.uncompress` 参考 [Java 使用 GZIP 进行压缩和解压缩（GZIPOutputStream，GZIPInputStream）](http://blog.csdn.net/wenqisun/article/details/51121460) 一文而成
```java
/**
 * 解压 gzip 格式 byte 数组
 * @param bytes gzip 格式 byte 数组
 * @param charset 字符集
 */
public static String uncompress(byte[] bytes, String charset) {
	if (bytes == null || bytes.length == 0) {
		return null;
	}
	ByteArrayOutputStream byteArrayOutputStream = null;
	ByteArrayInputStream byteArrayInputStream = null;
	GZIPInputStream gzipInputStream = null;
	try {
		byteArrayOutputStream = new ByteArrayOutputStream();
		byteArrayInputStream = new ByteArrayInputStream(bytes);
		gzipInputStream = new GZIPInputStream(byteArrayInputStream);
		// 使用 org.apache.commons.io.IOUtils 简化流的操作
		IOUtils.copy(gzipInputStream, byteArrayOutputStream);
		return byteArrayOutputStream.toString(charset);
	} catch (IOException e) {
		e.printStackTrace();
	} finally {
		// 释放流资源
		IOUtils.closeQuietly(gzipInputStream);
		IOUtils.closeQuietly(byteArrayInputStream);
		IOUtils.closeQuietly(byteArrayOutputStream);
	}
	return null;
}
```
另外 [Jerry Qu](https://imququ.com) 实现了一个服务器使用 Node.js 解压的 [DEMO](https://qgy18.com/request-compress/) 并提供 deflate,zlib,gzip 三种压缩, 解压方式
以上完整代码可在 [gzip](https://github.com/ghthou/gzip) 项目中查看
