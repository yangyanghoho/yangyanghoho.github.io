---
layout: post
title: "基于Squid实现HTTPS代理"
subtitle: "Use Squid as HTTPS proxy"
author: "Yang"
header-style: text
tags:
  - Linux
  - Squid
  - Proxy
  - CentOS
---

> 这篇文章转载自[我的公众号](https://mp.weixin.qq.com/s/Q9sCmEZAWp6eAJtF9QAu8Q)

引言
--

本文的主旨是介绍一个自己部署代理HTTPS服务的方案，让上网更安全，更隐私。先决条件如下：

* 客户端设备（PC和安卓手机）
* 可以部署代理服务的Linux服务器

安装服务
--

首先基于CentOS7直接安装squid：

```ts
yum install squid
```

按需要更改监听端口http_port等配置：

```ts
vi /etc/squid/squid.conf
```

校验配置文件：

```ts
squid -k parse
```

启动服务并开启开机自启动：

```ts
systemctl start squid
systemctl status squid
systemctl enable squid
```

至此，squid可以支持HTTP代理了，但是怎么让它支持HTTPS呢？

配置TLS
--

既然要端到端的支持HTTPS，那么代理服务上必然要启用TLS并搭配证书咯。用下面的命令制作证书：

```ts
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt -subj "/C=CN/ST=Liaoning/L=Dalian/O=TestCorp/OU=TestCorpWeb/CN=TestRootCA"
openssl x509 -in ca.crt -out ca.cer -outform der
openssl genrsa -out nginx.key 2048
openssl req -new -key nginx.key -out nginx.csr  -config san.cfg  -subj  "/C=CN/ST=Liaoning/L=Dalian/O=TestCorp/OU=TestCorpWeb/CN=yourdomain.com"
openssl x509 -req -in nginx.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out nginx.crt -extensions req_ext -extfile san.cfg -days 825 -sha256
```

做好后，跟其他比如nginx的证书一样，将.crt，.key上传到代理服务器上，将ca.cer安装到PC端的浏览器的信任证书里。（有ca的可以只做4~6步，这样跟旧的证书用的就是一个ca。）

然后在服务器上执行：

```ts
mkdir -p /var/lib/squid
rm -rf /var/lib/squid/ssl_db
/usr/lib64/squid/ssl_crtd -c -s /var/lib/squid/ssl_db
chown -R squid:squid /var/lib/squid
```

在服务器上配置squid启用HTTPS，修改squid的配置文件：

```ts
https_port 3128 tcpkeepalive=60,30,3 generate-host-certificates=on dynamic_cert_mem_cache_size=20MB cert=/etc/squid/squid.crt key=/etc/squid/squid.key
sslcrtd_program /usr/lib64/squid/ssl_crtd -s /var/lib/squid/ssl_db -M 20MB
sslproxy_cert_error allow all
```

检查配置无误的话，重启进程：

```ts
squid -k parse 
systemctl restart squid
```

测试PC端Firefox
--

进入firefox settings->network settings->Manual proxy configuration，将代理服务器的ip或者域名+端口填写到HTTPS代理里面。

测试发现https代理可用，只有一个小瑕疵，https的代理，是需要客户端用http connect提供需要代理的域名，而抓包可见http connect消息是未经TLS加密的明文http消息。类似这样：

![](https://yangzai.tech/img/in-post/post-squid/11.jpg)

为了保护隐私，后发现有pac的方法可以实现真正的https代理，具体pac文件例子如下，实现的是去往微软官网时用HTTPS代理服务器，其他网站直连：

```ts
function FindProxyForURL(url, host) {
   if (shExpMatch(url,"*.microsoft.com/*")) {
     return "HTTPS your_ip:your_port";
   }
   return "DIRECT";
}
```

使用pac用HTTPS类型代理的时候，可以真正实现没有明文HTTP connect的报文传输。


测试移动端
--

IOS和Android都可以在wifi里面设置手动代理（https://your_ip:port），但是跟pc端一样，可以用来做代理，但是有明文的http connect。

当然，也可以用pac的方法，设置自动代理，但是无论IOS还是Android好像都无视pac文件中的HTTPS字眼，不好用。（且IOS要求pac的网址是https的）。

另外需要注意的是，Android好像早就不信任自签名证书了，安装之后访问自己的网站还是会有提示。另外注意IOS手机端firefox和Android firefox正式版都不支持about:config。

最后，决定从Android Firefox Beta版本入手，看看用Firefox的about:config能否在移动端实现。

下载Beta版本并在手机上安装：

[https://github.com/mozilla-mobile/firefox-android/releases](https://github.com/mozilla-mobile/firefox-android/releases)

浏览器地址栏输入about:config进入配置，将下面参数（可以让Firefox信任自签名证书）改为true：

```ts
security.enterprise_roots.enabled
```

将下面参数改为pac文件的地址：

```ts
network.proxy.autoconfig_url
```

将下面参数改为2：

```ts
network.proxy.type
```

经验证访问microsoft.com成功，且没有明文的http connect。（注意要将ca.cer安装到安卓手机上。）

另外需要注意的是，每次重新打开浏览器之后第一个参数都会重置为false，参考下面的步骤永久打开这个参数：

```ts
Run the browser.
Go to Settings → About Firefox.
Tap the Firefox logo five times.
Navigate to Settings → Secret Settings.
Toggle Use third party CA certificates.
```

参考文档
--

* [https://support.kaspersky.com/kwts/6.1/zh-Hans/193662.htm](https://support.kaspersky.com/kwts/6.1/zh-Hans/193662.htm)

* [http://wiki.squid-cache.org/Features/HTTPS](http://wiki.squid-cache.org/Features/HTTPS)

* [https://www.chromium.org/developers/design-documents/secure-web-proxy/](https://www.chromium.org/developers/design-documents/secure-web-proxy/)

* [https://developer.apple.com/forums/thread/655287](https://developer.apple.com/forums/thread/655287)

* [https://www.topbug.net/blog/2015/03/02/configure-proxy-using-pac-files-on-firefox-for-android/](https://www.topbug.net/blog/2015/03/02/configure-proxy-using-pac-files-on-firefox-for-android/)

* [https://adguard.com/kb/adguard-for-android/solving-problems/firefox-certificates/](https://adguard.com/kb/adguard-for-android/solving-problems/firefox-certificates/)
