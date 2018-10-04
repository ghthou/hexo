---
title: 使用 zxing 生成二维码
tags:
  - Java
  - QRcode
categories:
  - Java
date: 2018-01-13 13:01:00
---
- 参考资料
    - [JAVA 生成二维码](http://www.imooc.com/learn/531)
    - [笔记, 谷歌 Zxing 二维码，用数据流输出到页面显示](http://blog.csdn.net/morning99/article/details/48825035)
- 项目环境
    - jdk1.8(**zxing 生成二维码图片文件需要 jdk1.7 及以上版本 **)
    - zxing-javase

    ```xml
    <dependency>
        <groupId>com.google.zxing</groupId>
        <artifactId>javase</artifactId>
        <version>3.3.0</version>
    </dependency>
    ```
    - 工具类代码
    
    ```java
    import com.google.zxing.BarcodeFormat;
    import com.google.zxing.EncodeHintType;
    import com.google.zxing.MultiFormatWriter;
    import com.google.zxing.WriterException;
    import com.google.zxing.client.j2se.MatrixToImageWriter;
    import com.google.zxing.common.BitMatrix;
    import com.google.zxing.qrcode.decoder.ErrorCorrectionLevel;
    
    import java.io.File;
    import java.io.IOException;
    import java.io.OutputStream;
    import java.util.HashMap;
    import java.util.Map;
    
    /**
     * 二维码工具类
     */
    public class QRCodeUtil {
        private static final int width = 300;// 默认二维码宽度
        private static final int height = 300;// 默认二维码高度
        private static final String format = "png";// 默认二维码文件格式
        private static final Map<EncodeHintType, Object> hints = new HashMap();// 二维码参数
    
        static {
            hints.put(EncodeHintType.CHARACTER_SET, "utf-8");// 字符编码
            hints.put(EncodeHintType.ERROR_CORRECTION, ErrorCorrectionLevel.H);// 容错等级 L、M、Q、H 其中 L 为最低, H 为最高
            hints.put(EncodeHintType.MARGIN, 2);// 二维码与图片边距
        }
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
        /**
         * 将二维码图片输出到一个流中
         * @param content 二维码内容
         * @param stream  输出流
         * @param width   宽
         * @param height  高
         */
        public static void writeToStream(String content, OutputStream stream, int width, int height) throws WriterException, IOException {
            BitMatrix bitMatrix = new MultiFormatWriter().encode(content, BarcodeFormat.QR_CODE, width, height, hints);
            MatrixToImageWriter.writeToStream(bitMatrix, format, stream);
        }
        /**
         * 生成二维码图片文件
         * @param content 二维码内容
         * @param path    文件保存路径
         * @param width   宽
         * @param height  高
         */
        public static void createQRCode(String content, String path, int width, int height) throws WriterException, IOException {
            BitMatrix bitMatrix = new MultiFormatWriter().encode(content, BarcodeFormat.QR_CODE, width, height, hints);
            //toPath() 方法由 jdk1.7 及以上提供
            MatrixToImageWriter.writeToPath(bitMatrix, format, new File(path).toPath());
        }
    }
    ```
    - 使用 SpringMVC 动态生成二维码
    ```java
	@RequestMapping(value = "/qrcode")
    public void qrcode(String content, @RequestParam(defaultValue = "300", required = false) int width,@RequestParam(defaultValue = "300", required = false) int height, HttpServletResponse response) {
        ServletOutputStream outputStream = null;
        try {
            outputStream = response.getOutputStream();
            QRCodeUtil.writeToStream(content, outputStream, width, height);
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
    - [QRCodeUtils.java 下载](https://raw.githubusercontent.com/ghthou/Google-Authenticator/master/src/main/java/z/study/googleAuthenticator/util/QRCodeUtils.java)(右键另存为)
    - JavaScript 生成二维码可参考 [使用 jQuery-qrcode 生成二维码.md](/2018/01/13/使用-jQuery-qrcode-生成二维码/)
    - 如果想生成带 LOGO 的二维码可参考 [生成带 LOGO 的二维码](/2018/01/13/生成带-LOGO-的二维码/)
