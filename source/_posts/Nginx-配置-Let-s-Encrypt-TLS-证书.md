---
title: Nginx 配置 Let's Encrypt TLS 证书
tags:
  - Nginx
  - TLS
categories:
  - Nginx
date: 2018-01-13 10:48:00
---
> 本文是我参考互联网上一些 HTTPS 相关文章, 而后自己实践而来
主要内容由 ** 获取证书 **,**Nginx TSL 配置 ** 两部分组成

### 项目环境搭建
- Git       ([安装 Git](http://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/00137396287703354d8c6c01c904c7d9ff056ae23da865a000))
- Nginx     ([Nginx 安装配置](http://www.runoob.com/linux/nginx-install-setup.html))

### 获取证书
安装完 Git 后, 克隆 [certbot](https://github.com/certbot/certbot)(* 原名 letsencrypt*) 项目
```bash
$ git clone https://github.com/certbot/certbot
$ cd certbot
```
克隆完成后, 此时可以通过命令获取证书

- **Webroot**
该方式适用于本机中正在运行中的项目, 无须停止服务器即可申请
```bash
$ ./certbot-auto certonly --webroot -w /usr/local/nginx/html --email admin@example.com -d example.com -d www.example.com -d other.example.com
```
`--webroot` 为申请方式类型 
`-w /usr/local/nginx/html` 配置申请域名项目的 `Webroot` 跟路径, 以静态页面为例
```nginx
location / {
    root   /usr/local/nginx/html;
    index  index.html;
}
```
该方式会在 `/usr/local/nginx/html` 目录下创建一个 `.well-known/acme-challenge` 文件夹, 然后生成一个随机文件进行访问. 如
http://example.com/.well-known/acme-challenge/HGr8U1IeTW4kY_Z6UIyaakzOkyQgPr_7ArlLgtZE8SX
如果项目进行了反向代理, 请确保 `-w` 参数后面路径为 代理项目所在的根目录
`--email admin@example.com` 为申请者的 ** 邮箱 **, 在证书快过期时, 会发送通知邮件
`-d example.com` 为要 ** 申请的域名 **, 允许存在多个, 格式为 `-d 域名 -d 域名 `, ** 但要求域名访问地址为本机 **

- Standalone
该方式需要 ** 确保 80,443 端口没有被占用 **
```bash
$ ./certbot-auto certonly --standalone --email admin@example.com -d example.com -d www.example.com -d other.example.com
```
`--standalone` 为申请方式类型 
`--email admin@example.com -d example.com` 参数参考 `Webroot` 方式
如果验证没有问题, 稍候页面会弹出一个协议对话框, 点击 **Agree** 即可

- Manual
该方式适用于申请证书服务器不是域名所在服务器的情况
```bash
$ ./certbot-auto certonly --manual --email admin@example.com -d example.com -d www.example.com -d other.example.com
```
`--standalone` 为申请方式类型
`--email admin@example.com` 参数参考 `Webroot` 方式
`-d example.com` 为需要申请的域名,需要拥有对该域名所在机器的读写权限
运行后, 会要求用户在域名所在服务器的项目跟目录创建一个文件, 并指定文件内容
```bash
Make sure your web server displays the following content at
http://example.com/.well-known/acme-challenge/VE_oSzihad5ZNYPnE_OLT2aQ-BdT_M3z5ITj53wQ-Oc before continuing:
VE_oSzihad5ZNYPnE_OLT2aQ-BdT_M3z5ITj53wQ-Oc.JhmxNt13DUzzmC4_7VfnfWh1gmePbExxQygAMf9KTSo
# 省略
# /tmp/certbot/public_html 应为域名所在服务器的项目跟路径
mkdir -p /tmp/certbot/public_html/.well-known/acme-challenge
# 进入项目跟路径
cd /tmp/certbot/public_html
# 指定内容输入到指定文件中
printf "%s" VE_oSzihad5ZNYPnE_OLT2aQ-BdT_M3z5ITj53wQ-Oc.JhmxNt13DUzzmC4_7VfnfWh1gmePbExxQygAMf9KTSo > .well-known/acme-challenge/VE_oSzihad5ZNYPnE_OLT2aQ-BdT_M3z5ITj53wQ-Oc
# 省略
Press Enter to Continue
```
操作完成后按下回车键即可

申请操作成功后, 会在界面中输出证书的存放路径, 以及证书的到期时间 (**90 天 **)
```bash
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at
   /etc/letsencrypt/live/example.com/fullchain.pem. Your cert
   will expire on 2017-08-17. To obtain a new or tweaked version of
   this certificate in the future, simply run certbot-auto again. To
   non-interactively renew *all* of your certificates, run
   "certbot-auto renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```
生成证书中会创建 `/etc/letsencrypt` 文件夹, 证书文件默认存放在 `/etc/letsencrypt/live/example.com` 文件夹中, 其中 `example.com` 取自第一个域名
在 `example.com` 文件夹中包含 4 个文件 `cert.pem  chain.pem  fullchain.pem  privkey.pem`
- cert.pem 域名证书
- chain.pem 根证书及中间证书
- fullchain.pem 由 `cert.pem` 和 `chain.pem` 合并而成
- privkey.pem 证书私钥

** 创建一个 2048 位的 [Diffie-Hellman](https://zh.wikipedia.org/wiki/%E8%BF%AA%E8%8F%B2-%E8%B5%AB%E7%88%BE%E6%9B%BC%E5%AF%86%E9%91%B0%E4%BA%A4%E6%8F%9B) 文件 **(nginx 默认使用 1024 位的 Diffie–Hellman 进行密钥交换, 安全性太低)
```bash
openssl dhparam -out /etc/letsencrypt/live/dhparams.pem 2048
```

###nginx TSL 配置

首先对 http 协议进行 301 重定向到 https 协议
```nginx
server {
  listen 80;
  server_name example.com www.example.com;
  return 301 https://example.com$request_uri;
}
```
https 相关配置
```nginx
server {
    listen 443 ssl;
    server_name example.com www.example.com;
    
    # 配置站点证书文件地址
    ssl_certificate      /etc/letsencrypt/live/example.com/fullchain.pem;
    # 配置证书私钥
    ssl_certificate_key  /etc/letsencrypt/live/example.com/privkey.pem;
    
    # 配置 Diffie-Hellman 交换算法文件地址
    ssl_dhparam          /etc/letsencrypt/live/dhparams.pem;
    
    # 配置服务器可使用的加密算法
    ssl_ciphers 'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS';

    # 指定服务器密码算法在优先于客户端密码算法时，使用 SSLv3 和 TLS 协议
    ssl_prefer_server_ciphers  on;
    
    # ssl 版本 可用 SSLv2,SSLv3,TLSv1,TLSv1.1,TLSv1.2 
    # ie6 只支持 SSLv2,SSLv3 但是存在安全问题, 故不支持
    ssl_protocols        TLSv1 TLSv1.1 TLSv1.2;
    
    # 配置 TLS 握手后生成的 session 缓存空间大小 1m 大约能存储 4000 个 session
    ssl_session_cache          shared:SSL:50m;
    # session 超时时间
    ssl_session_timeout        1d;
    
    # 负载均衡时使用 此处暂时关闭 详情见 https://imququ.com/post/optimize-tls-handshake.html 
    # 1.5.9 及以上支持
    ssl_session_tickets off;
    
    # 浏览器可能会在建立 TLS 连接时在线验证证书有效性，从而阻塞 TLS 握手，拖慢整体速度。OCSP stapling 是一种优化措施，服务端通过它可以在证书链中封装证书颁发机构的 OCSP（Online Certificate Status Protocol）响应，从而让浏览器跳过在线查询。服务端获取 OCSP 一方面更快（因为服务端一般有更好的网络环境），另一方面可以更好地缓存 以上内容来自 https://imququ.com/post/my-nginx-conf-for-wpo.html
    # 1.3.7 及以上支持
    ssl_stapling               on;
    ssl_stapling_verify        on;
    # 根证书 + 中间证书
    ssl_trusted_certificate    /etc/letsencrypt/live/example.com/fullchain.pem;
    
    # HSTS 可以告诉浏览器，在指定的 max-age 内，始终通过 HTTPS 访问该域名。即使用户自己输入 HTTP 的地址，或者点击了 HTTP 链接，浏览器也会在本地替换为 HTTPS 再发送请求 相关配置见 https://imququ.com/post/sth-about-switch-to-https.html
    add_header Strict-Transport-Security max-age=60;
    
    # 在此填写原本 http 协议中的配置
}
```
以上配置完成后, 重启 nginx 即可完成对 https 的切换

使用以下命令即可进行 ** 续期 **, 续期成功后需要 ** Nginx 重新启动 **
```bash
$ ./certbot-auto renew
```
该命令只会对快到期的证书才会进行更新, 如果希望强制更新, 可以增加 `--force-renewal` 参数

通过 [Qualys SSL labs](https://www.ssllabs.com/index.html) 提供的 [SSL Server Test](https://www.ssllabs.com/ssltest/index.html) 服务可以查看网站的当前配置, 以及测试网站安全评分
另外该网站缓存了一些知名网站的测试结果, 点击以下链接可进行查看
- [Google](https://www.ssllabs.com/ssltest/analyze.html?d=google.com&s=216.58.195.78&hideResults=on)
- [GitHub](https://www.ssllabs.com/ssltest/analyze.html?d=github.com&s=192.30.253.113&hideResults=on)
- [Stack Overflow](https://www.ssllabs.com/ssltest/analyze.html?d=stackoverflow.com&s=151.101.129.69&hideResults=on)
- [Twitter](https://www.ssllabs.com/ssltest/analyze.html?d=twitter.com&s=104.244.42.129&hideResults=on)
- [Facebook](https://www.ssllabs.com/ssltest/analyze.html?d=www.facebook.com&s=31.13.70.36&hideResults=on)


### 其他说明
如果一个 IP 需要配置多张证书, 请查看 [关于启用 HTTPS 的一些经验分享（二）](https://imququ.com/post/sth-about-switch-to-https-2.html) **SNI 扩展 ** 部分
文中 `ssl_ciphers` 配置参考 [Mozilla SSL Configuration Generator](https://mozilla.github.io/server-side-tls/ssl-config-generator/) 中的常规配置而来
Mozilla 同时也提供 TSL 配置说明文档 [Security/Server Side TLS](https://wiki.mozilla.org/Security/Server_Side_TLS#Recommended_configurations)
另外也可参考 [cipherli](https://cipherli.st/) 提供的 TSL 配置模版
除了 `ssl_ciphers` 配置外, 其他皆是参考 [Jerry Qu](https://imququ.com/) 的 [本博客 Nginx 配置之完整篇](https://imququ.com/post/my-nginx-conf.html) 一文而来
另外一些配置的详情, 作用, 原理也可以在他的博客中进行搜索得知
本文同时也参考了以下博客
- [HTTPS 升级指南](http://www.ruanyifeng.com/blog/2016/08/migrate-from-http-to-https.html)
- [更换 SSL 证书：使用 Let's Encrypt](https://www.cooppor.com/post/using-lets-encrypt-free-ssl)
- [Let's Encrypt SSL 证书配置](http://www.jianshu.com/p/eaac0d082ba2)
另外在配置完成后可以 ** 观看以下博客 **, 以增强对 TSL,nginx 的了解
- [一些安全相关的 HTTP 响应头](https://imququ.com/post/web-security-and-response-header.html)
- [Content Security Policy 介绍](https://imququ.com/post/content-security-policy-reference.html)
- [关于启用 HTTPS 的一些经验分享（一）](https://imququ.com/post/sth-about-switch-to-https.html)
- [关于启用 HTTPS 的一些经验分享（二）](https://imququ.com/post/sth-about-switch-to-https-2.html)
- [关于启用 HTTPS 的一些经验分享（三）](https://imququ.com/post/sth-about-switch-to-https-3.html)
- [HTTP Public Key Pinning 介绍](https://imququ.com/post/http-public-key-pinning.html)
- [本博客 Nginx 配置之安全篇](https://imququ.com/post/my-nginx-conf-for-security.html)
- [本博客 Nginx 配置之性能篇](https://imququ.com/post/my-nginx-conf-for-wpo.html)
