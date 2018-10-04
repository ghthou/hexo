---
title: Java 接入 Google Authenticator
tags:
  - Java
  - Google Authenticator
  - 三方接入
categories:
  - Java
date: 2018-01-13 13:11:00
---
> 在网络攻击日益泛滥的今天, 用户的密码可能会因为各种原因泄漏. 而一些涉及用户重要数据的服务, 如 QQ, 邮箱, 银行, 购物等等. 一但被有心人利用, 那么除了自己隐私泄漏的风险外, 还存在自己身份被冒充的危害, 更有可能而导致极其严重的结果. 为此谷歌推出了 `Google Authenticator` 服务, 其原理是在登录时除了输入密码外, 还需根据 `Google Authenticator APP` 输入一个实时计算的验证码. 凭借此验证码, 即使在密码泄漏的情况下, 他人也无法登录你的账户

### 相关原理
`Google Authenticator` 使用了一种基于 ** 时间 ** 的 `TOTP` 算法, 其中时间的选取为自 `1970-01-01 00:00:00` 以来的毫秒数除以 `30` 与 客户端及服务端约定的 ** 密钥 ** 进行计算, 计算结果为一个 **6 位数的字符串 **(* 首位数可能为 0, 所以为字符串 *), 所以在 `Google Authenticator` 中我们可以看见验证码每个 30 秒就会刷新一次. 更多详情可查看 [Google 账户两步验证的工作原理](https://blog.seetee.me/archives/73.html) 一文

### 实现思路
由上可知, 生成验证码有俩个重要的参数, 其一为 ** 客户端与服务端约定的密钥 **, 其二便为 **30 秒的个数 **
```java
/**
 * 随机生成一个密钥
 */
public static String createSecretKey() {
	SecureRandom random = new SecureRandom();
	byte[] bytes = new byte[20];
	random.nextBytes(bytes);
	Base32 base32 = new Base32();
	String secretKey = base32.encodeToString(bytes);
	return secretKey.toLowerCase();
}
```
```java
//1970-01-01 00:00:00 以来的毫秒数除以 30 
long time = System.currentTimeMillis() / 1000 / 30;
```
根据这两个参数就可以生成一个验证码
```java
/**
 * 根据密钥获取验证码
 * 返回字符串是因为验证码有可能以 0 开头
 * @param secretKey 密钥
 * @param time      第几个 30 秒 System.currentTimeMillis() / 1000 / 30
 */
public static String getTOTP(String secretKey, long time) {
    Base32 base32 = new Base32();
    byte[] bytes = base32.decode(secretKey.toUpperCase());
    String hexKey = Hex.encodeHexString(bytes);
    String hexTime = Long.toHexString(time);
    return TOTP.generateTOTP(hexKey, hexTime, "6");
}
```    
因为 `Google Authenticator`(* 以下简称 APP*) 计算验证码也需要 ** 密钥 ** 的参与, 而时间 APP 则会在本地获取, 所以我们需要将 ** 密钥保存在 APP 中 **, 同时为了与其他账户进行区分, 除了密钥外, 我们还需要录入 ** 服务名称 **,** 用户账户 ** 信息. 而为了方便用户信息的录入, 我们一般将所有信息生成一张二维码图片, 让用户通过扫码自动填写相关信息
```java
/**
 * 生成 Google Authenticator 二维码所需信息
 * Google Authenticator 约定的二维码信息格式 : otpauth://totp/{issuer}:{account}?secret={secret}&issuer={issuer}
 * 参数需要 url 编码 + 号需要替换成 %20
 * @param secret  密钥 使用 createSecretKey 方法生成
 * @param account 用户账户 如: example@domain.com 138XXXXXXXX
 * @param issuer  服务名称 如: Google Github 印象笔记
 */
public static String createGoogleAuthQRCodeData(String secret, String account, String issuer) {
    String qrCodeData = "otpauth://totp/%s?secret=%s&issuer=%s";
	try {
		return String.format(qrCodeData, URLEncoder.encode(issuer + ":" + account, "UTF-8").replace("+", "%20"), URLEncoder.encode(secret, "UTF-8")
				.replace("+", "%20"), URLEncoder.encode(issuer, "UTF-8").replace("+", "%20"));
	} catch (UnsupportedEncodingException e) {
		e.printStackTrace();
	}
	return "";
}
```
此时再根据上述信息生成二维码, 二维码生成方式可参考以下两种方案

- [使用 zxing 生成二维码.md](/2018/01/13/使用-zxing-生成二维码/)
- [使用 jQuery-qrcode 生成二维码.md](/2018/01/13/使用-jQuery-qrcode-生成二维码/)

此时选择使用 `Java` 的方式返回一个二维码图片流
```java
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
```
扫描二维码
![扫描二维码](/images/Java-接入-Google-Authenticator/扫描二维码.png)
扫描成功后会新增一栏验证码信息
![扫描后新增一栏信息](/images/Java-接入-Google-Authenticator/扫描后新增一栏信息.png)
再让用户输入验证码, 与服务端进行校验, 如果校验通过, 则表明用户可以完好使用该功能
因为验证码是使用基于时间的 `TOTP` 算法, 依赖于客户端与服务端时间的一致性. 如果客户端时间与服务端时间相差过大, 那在用户没有同步时间的情况下, 永远与服务端进行匹配. 同时服务端也有可能出现时间偏差的情况, 这样反而导致时间正确的用户校验无法通过
为了解决这种情况, 我们可以使用 ** 时间偏移量 ** 来解决该问题,`Google Authenticator` 验证码的时间参数为 `1970-01-01 00:00:00 以来的毫秒数除以 30`, 所以每 30 秒就会更新一次. 但是我们在后台进行校验时, 除了与当前生成的二维码进行校验外, 还会对当前时间参数 ** 前后偏移量 ** 生成的验证码进行校验, 只要其中任意一个能够校验通过, 就代表该验证码是有效的
```java
/** 时间前后偏移量 */
private static final int timeExcursion = 3;
/**
 * 校验方法
 * @param secretKey 密钥
 * @param code      用户输入的 TOTP 验证码
 */
public static boolean verify(String secretKey, String code) {
    long time = System.currentTimeMillis() / 1000 / 30;
    for (int i = -timeExcursion; i <= timeExcursion; i++) {
        String totp = getTOTP(secretKey, time + i);
        if (code.equals(totp)) {
            return true;
        }
    }
    return false;
}
```
### 其他说明
根据以上代码我们可以简单的创建一个 `Google Authenticator` 的应用. 但是与此同时, 我们也发现 `Google Authenticator` 严重依赖手机, 又因为 `Google Authenticator` ** 没有同步功能 **, 所以如果用户一不小心删除了记录信息, 或者 APP 被卸载, 手机系统重装等情况. 就会导致 `Google Authenticator` 成为使用者的障碍. 此时我们可以使用 **[Authy](https://www.authy.com/app/)** 这款支持 ** 同步功能 ** 的 APP 以解决删除, 卸载, 重装等问题. 同时 **[Authy](https://www.authy.com/app/)** 也存在 **[Chrome 插件](https://chrome.google.com/webstore/detail/authy/gaedmjdfmmahhbjefcbgaolhhanlaolb?hl=cn)** 版本, 用于解决在手机丢失的情况下获取验证码.
除了 **[Authy](https://www.authy.com/)** 这个选择外, 我们还可以使用 ** 备用验证码 ** 的机制用户用于解决上述问题. 即在用户绑定 `Google Authenticator` 成功后自动为用户生成多个 ** 备用验证码 **, 然后在前台显示. 并让用户进行保存, 再让用户使用备用验证码进行校验, 以确保用户保存成功, 可以参考 ** 印象笔记 ** 的用法 [如何开启印象笔记登录两步验证？](https://help.evernote.com/hc/sr-me/articles/213420077--%E5%A6%82%E4%BD%95%E5%BC%80%E5%90%AF%E5%8D%B0%E8%B1%A1%E7%AC%94%E8%AE%B0%E7%99%BB%E5%BD%95%E4%B8%A4%E6%AD%A5%E9%AA%8C%E8%AF%81-)

以上所使用的代码可在 [Google-Authenticator](https://github.com/ghthou/Google-Authenticator) 中查看

另外本文同时参考了以下资料
- [Google Authenticator compatible 2-Factor Auth in Java](http://www.asaph.org/2016/04/google-authenticator-2fa-java.html)
- [twofactorauth](https://github.com/asaph/twofactorauth)
- [Google Authenticator JAVA 实例](http://awtqty-zhang.iteye.com/blog/1986275)
- [Google 账户两步验证的工作原理](https://blog.seetee.me/archives/73.html)
- [GoogleAuth](https://github.com/wstrange/GoogleAuth)
