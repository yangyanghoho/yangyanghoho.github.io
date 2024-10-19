---
layout: post
title: "安装RocketChat-你的私人即时聊天平台"
subtitle: "RocketChat - your private messenger"
author: "Yang"
header-style: text
tags:
  - Linux
  - RocketChat
  - Proxy
---

> 这篇文章转载自[我的公众号](https://mp.weixin.qq.com/s?__biz=Mzg5NjYxNzI3NA==&mid=2247483728&idx=1&sn=7b38de60ea5721a393ccdc311c875205&chksm=c07f1f05f70896139360497145e5cf6a113d7286faf64e9c02dc394ce5eee758c606f3422c0d&token=1624824886)

引言
--

RocketChat是一款优秀的开源聊天软件。它支持各种平台，IOS、Android、Web、Windows以及Linux，安装部署简单，功能简单易用，特别适合小公司或小团体自建聊天平台。

项目开源地址：

[https://github.com/RocketChat](https://github.com/RocketChat)

项目官方站点：

[https://rocket.chat](https://rocket.chat)

其中，IOS app支持我们常用的一些功能，比如发送文字，语音和图片等等。试用一段时间之后，个人觉得还不错，比较简洁。

RocketChat支持两种部署方式，一种是使用其提供的服务器，一种是自己搭建服务运行环境。后者所有聊天都通过https在自己搭建的环境处理，安全可控，自然选择这种方法来折腾试试。


端到端方案
--

本次环境搭建采用公有云的一台CentOS服务器，两台iphone，一台PC来验证。服务器后端可以对接RocketChat提供的推送服务（每月免费推送10000条），服务监听3000端口，并使用nginx做反向代理，实现https登录。如果没有公网地址，也可以在内网搭建，仅限内网使用，或者使用其他方案实现外部访问亦可。


服务端安装
--

服务端安装支持多种平台和多种方式，这里我们采用CentOS的手动安装。安装简单，注意文件夹权限等细节即可，具体见官网文档：

[https://docs.rocket.chat/quick-start/installing-and-updating/manual-installation/centos](https://docs.rocket.chat/quick-start/installing-and-updating/manual-installation/centos)

这里有一个坑，服务端新版本4.2.0在CentOS上有bug，需要用旧版本，见下面：

[https://releases.rocket.chat/3.9.4/download](https://releases.rocket.chat/3.9.4/download)

安装完成后，访问服务器的3000端口，按需配置：

[http://yourdomain.com:3000](http://yourdomain.com:3000)

![](https://yangyanghoho.github.io/img/in-post/post-chat/11.jpg)

之后就可以把网页分享给其他朋友来一起聊天了。


移动端调试
--

于是开始满心欢喜的在app store下载了IOS的rocketchat app，在登陆页面提交链接之后，直接弹出来报错，要求使用https。估计是由于在app store上架的app有严格的https协议要求，下面链接是apple对服务端tls证书的要求：

[https://support.apple.com/en-us/HT210176](https://support.apple.com/en-us/HT210176)

懒得去authorized CA去弄证书，反正只是自己用，还是打算用自签名https证书。踩了很多坑，研究半天，终于知道了正确的被apple接受的证书制作方法：

```ts
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt -subj "/C=CN/ST=Liaoning/L=Dalian/O=TestCorp/OU=TestCorpWeb/CN=TestRootCA"
openssl x509 -in ca.crt -out ca.cer -outform der
openssl genrsa -out nginx.key 2048
openssl req -new -key nginx.key -out nginx.csr  -config san.cfg  -subj  "/C=CN/ST=Liaoning/L=Dalian/O=TestCorp/OU=TestCorpWeb/CN=yourdomain.com"
openssl x509 -req -in nginx.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out nginx.crt -extensions req_ext -extfile san.cfg -days 825 -sha256
```

将自己的ca.crt文件导入到浏览器受信任证书列表里，将nginx.crt，nginx.key配置到nginx服务器上，将ca.cer作为描述文件安装到IOS客户端，并在可信根证书中启用。

我依然把rocketchat作为网站的子目录来使用，那么最后访问rocketchat的域名应该是：

[https://yourdomain.com/rocketchat](https://yourdomain.com/rocketchat)

参考的ssl配置：

[https://developer.rocket.chat/mobile-app/mobile-app-environment-setup/supporting-ssl-for-development-on-rocket.chat](https://developer.rocket.chat/mobile-app/mobile-app-environment-setup/supporting-ssl-for-development-on-rocket.chat)

最后，使用iphone安装根证书后访问域名，登陆app：

![](https://yangyanghoho.github.io/img/in-post/post-chat/22.jpg)

经测试，可以正常发送消息，但是app不在主页的时候没有推送，只能使用rocketchat提供的推送了，自己弄一个IOS推送服务超出我的能力了。rocketchat提供的推送服务见下面网页：

[https://docs.rocket.chat/guides/administration/admin-panel/connectivity-services](https://docs.rocket.chat/guides/administration/admin-panel/connectivity-services)

10000条/月也凑合够用了。


结语
--

激动！感觉弄好了还是挺有成就感的，希望大家不吝点赞转发，谢谢！


nginx配置参考
--

```ts

upstream backend {
    server 127.0.0.1:3000;
}
    location ~/rocketchat(.*)$ {

        # set max upload size
        client_max_body_size 512M;

        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $http_host;

        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Nginx-Proxy true;

        proxy_redirect off;
    }
```
