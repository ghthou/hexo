---
title: 生成带 LOGO 的二维码
tags:
  - Java
  - QRcode
categories:
  - Java
date: 2018-01-13 13:02:00
---
> 通过 [使用 zxing 生成二维码](/2018/01/13/使用-zxing-生成二维码) 我们可以实现简单二维码的生成, 但是二维码显示却过于单调, 本文变讲述如何利用 [thumbnailator](https://github.com/coobird/thumbnailator) 为我们的二维码添加 LOGO

[thumbnailator](https://github.com/coobird/thumbnailator) 是一个缩略图工具类库, 但它除了能缩略图片外, 还提供裁剪, 旋转, 水印等功能, 此次我们便借助它的水印 API 实现以上需求

### 项目环境
- jdk1.8
- [thumbnailator](https://github.com/coobird/thumbnailator)

```xml
<dependency>
    <groupId>net.coobird</groupId>
    <artifactId>thumbnailator</artifactId>
    <version>0.4.8</version>
</dependency>
```
### 操作步骤
#### 生成
- 准备一张二维码图片, 一张 logo 图片
```java
String qrCodeFilePath = "src/qrCode.jpg";
String logoFilePath = "src/logo.jpg";
```
![src/qrCode.png, 内容为 1](/images/生成带-LOGO-的二维码/qrCode.png)
![src/logo.png](/images/生成带-LOGO-的二维码/logo.png)

- 根据文件路径生成一个图片构造对象( `Thumbnails.of()` 方法还可以接收 `BufferedImage`,`InputStream`,`URL`,`String` 的可变参数类型)
```java
Thumbnails.Builder<File> builder = Thumbnails.of(new File(qrCodeFilePath));
```
- 创建一个水印对象, 水印对象需要三个参数 `Position position`, `BufferedImage watermarkImg`,`float opacity`, 其中 `Position` 是水印的坐标,`BufferedImage` 是水印图片,`opacity` 是不透明度, 值为 [0-1] 之间, 1 代表不透明.
```java
BufferedImage bufferedImage = ImageIO.read(new File(logoFilePath));
//Positions 实现了 Position 并提供九个坐标, 分别是 上左, 上中, 上右, 中左, 中中, 中右, 下左, 下中, 下右 我们使用正中的位置
Watermark watermark = new Watermark(Positions.CENTER, bufferedImage, 1F);
```
- 为二维码设置水印, 并设置缩略比例为 1(即不压缩), 输出到一个新文件中(`outputFormat()` 为指定输出格式, 如: jpg,png)
```java
builder.watermark(watermark).scale(1F).toFile(new File("src/logoQrCode.png"));
```
- 生成后的二维码
![src/logoQrCode.png](/images/生成带-LOGO-的二维码/logoQrCode.png)
#### 使用
通过以上方法我们能够简单的为二维码添加 LOGO, 但是实际使用时远没有这样简单, 有以下几个问题
- logo 图片的尺寸可能并不固定, 可能有大有小, 这样就会导致 logo 在二维码中太小或太大
这时我们可以在创建 `BufferedImage` 时将原图压缩 / 放大至指定尺寸, 也可以使用二维码的尺寸乘以一定比例
```java
//forceSize(int width, int height) 指将图片强制压缩为指定宽高, 如不强制, 可使用 size(int width, int height)
BufferedImage bufferedImage = Thumbnails.of(new File(logoFilePath)).forceSize(120, 120).asBufferedImage();
```
- 图片显示, logo 图片可能不是一个图片文件, 同时生成好之后的图片希望直接输出在页面上
`Thumbnails.of()` 可以接收 `BufferedImage`,`InputStream`,`URL`,`String`,`File` 的可变 (** 批量处理需要 **) 参数类型
`ImageIO.read()` 可以接收 `File`,`ImageInputStream`,`InputStream`,`URL` 参数类型
以下为 SpringMVC 动态生成带 LOGO 二维码示例,`QRCodeUtil.toBufferedImage()` 具体实现请看 [生成二维码之 Java (Google zxing) 篇](http://www.jianshu.com/p/05e9ee773898)
```java
/**
 * @param content 二维码内容
 * @param logoUrl logo 链接
 */
@RequestMapping(value = "/qrcode")
public void qrcode(String content, String logoUrl, @RequestParam(defaultValue = "300") int width, @RequestParam(defaultValue = "300") int height,HttpServletResponse response) {
    ServletOutputStream outputStream = null;
    try {
        outputStream = response.getOutputStream();
        // 根据 QRCodeUtil.toBufferedImage() 返回的 BufferedImage 创建图片构件对象
        Thumbnails.Builder<BufferedImage> builder = Thumbnails.of(QRCodeUtil.toBufferedImage(content, width, height));
        // 将 logo 的尺寸设置为二维码的 30% 大小, 可以自己根据需求调节
        BufferedImage logoImage = Thumbnails.of(new URL(logoUrl)).forceSize((int) (width * 0.3), (int) (height * 0.3)).asBufferedImage();
        // 设置水印位置(居中), 水印图片 BufferedImage, 不透明度(1F 代表不透明)
        builder.watermark(Positions.CENTER, logoImage, 1F).scale(1F);
        // 此处需指定图片格式, 否者报 Output format not specified 错误
        builder.outputFormat("png").toOutputStream(outputStream);
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        if (outputStream != null) {
            try {
                outputStream.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```
  ```java
  /**
  * 返回一个 BufferedImage 对象
  * @param content 二维码内容
  * @param width   宽
  * @param height  高
  */
  public static BufferedImage toBufferedImage(String content, int width, int height) throws WriterException, IOException {
      BitMatrix bitMatrix = new MultiFormatWriter().encode(content, BarcodeFormat.QR_CODE, width, height, hints);
  return MatrixToImageWriter.toBufferedImage(bitMatrix);
  }
  ```
#### 拓展思考
以上案例可知 [thumbnailator](https://github.com/coobird/thumbnailator) 水印功能的强大. 然而除了实现以上案例外, 我们也可以根据水印功能实现另外一种需求.
比如公司现在有一个推广需求, 让用户为我们的产品进行宣传, 达到多少次就进行奖励. 传统的实现方式是一段文字再配上一个专属链接. 但是此方式往往太过单调, 枯燥. 这个时候如果我们让设计提供一张内容丰富, 带有二维码的宣传图片, 然后根据专属链接生成二维码与宣传图片进行组合. 这时用户就有了一张自己的专属宣传图片. 此时通过直接宣传专属图片往往会比文字加链接有着更好的效果.
注: 此功能在图片合成时需要对二维码的位置进行定位, 此时可使用 `Position` 的实现类 `Coordinate` 完成
使用如下:
```java
//Coordinate 存在一个 Coordinate(int x, int y) 构造函数, x 为水印距离底图左边的像素, y 为上边
BufferedImage bufferedImage = ImageIO.read(new File(logoFilePath));
Watermark watermark = new Watermark(new Coordinate(100, 100), bufferedImage, 1F);
```
