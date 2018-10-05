---
title: 使用 jQuery-qrcode 生成二维码
tags:
  - JavaScript
  - QRcode
categories:
  - JavaScript
date: 2018-01-13 12:51:00
---
> 本文讲述如何使用 [jquery-qrcode](https://github.com/jeromeetienne/jquery-qrcode) 生成二维码

### 示例
```html
<!-- 引入 jQuery 与 jquery.qrcode-->
<script type="text/javascript" src="jquery-2.2.4.min.js"></script>
<script type="text/javascript" src="jquery.qrcode.min.js"></script>
<!-- 创建二维码父级元素 -->
<div id="qrcode"></div>
<div id="qrcode2"></div>
<script type="text/javascript">
    // 选中要生成二维码的元素节点, 调用 qrcode 方法, 传入数据即可
    $("#qrcode").qrcode("https://google.com");
    //	        同时提供更多参数, 具体如下
    //render		绘制方式 canvas(绘制成一张图片) table(绘制成一个表格) 默认 canvas
    //width		    二维码宽度  默认 256
    //height        二维码高度  默认 256
    //correctLevel  容错等级    1(L),0(M),3(Q),2(H)    1 最低, 2 最高  默认为 2
    //background    背景颜色    默认白色    #ffffff
    //foreground    前景颜色    默认黑色    #000000
    // 使用示例
    $("#qrcode2").qrcode({
        text: "https://google.com",
        render: "table",
        width: 300,
        height: 300,
        correctLevel: 3,
        background: "#eceadb",
        foreground: "#444"
    })
</script>
```
### 中文乱码解决办法
由于 qrcode 的编码原因会导致中文乱码
如果能确保二维码中的内容为 ** 链接 **, 那么在使用前对内容进行 **encodeURI** 编码即可
```javascript
$("#qrcode").qrcode(encodeURI("https://google.com"));
```
如果不是链接, 需要对二维码内容进行编码, 方法来源于 [http://justcoding.iteye.com/blog/2213034](http://justcoding.iteye.com/blog/2213034)
```javascript
function toUtf8(str) {
    var out, i, len, c;
    out = "";
    len = str.length;
    for(i = 0; i < len; i++) {
        c = str.charCodeAt(i);
        if ((c>= 0x0001) && (c <= 0x007F)) {
            out += str.charAt(i);
        } else if (c> 0x07FF) {
            out += String.fromCharCode(0xE0 | ((c>> 12) & 0x0F));
            out += String.fromCharCode(0x80 | ((c>>  6) & 0x3F));
            out += String.fromCharCode(0x80 | ((c>>  0) & 0x3F));
        } else {
            out += String.fromCharCode(0xC0 | ((c>>  6) & 0x1F));
            out += String.fromCharCode(0x80 | ((c>>  0) & 0x3F));
        }
    }
    return out;
} 
$("#qrcode").qrcode(toUtf8("https://google.com"));
```

