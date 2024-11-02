---
layout: post
title: "安装Nextcloud - 你的私有网盘"
subtitle: "Nextcloud - your private cloud storage"
author: "Yang"
header-style: text
tags:
  - Linux
  - Nextcloud
  - Storage
  - CentOS
---

> 这篇文章转载自[我的公众号](https://mp.weixin.qq.com/s/-fh4i2Pku0VwzUOXJ-5eiA)

引言
--

相信大家在日常生活中经常要用到网盘，无论是分享文件还是备份数据。但是一个靠谱的网盘提供商却不太好找，无论是某家限速的，还是某家彻底关闭的。其实也有不同的方案，放弃网盘提供商，自己解决问题,比如在自己家里搭建NAS。也可以自己搭建nextcloud，在家里的电脑/服务器上（用内网穿透等技术），或者在公有云的实例上。

Nextcloud是一款开源免费的私有云存储网盘项目，可以让你快速便捷地搭建一套属于自己或团队的云同步网盘，从而实现跨平台跨设备文件同步、共享、版本控制、团队协作等功能。它的客户端覆盖了Windows、Mac、Android、iOS、Linux 等各种平台。

Nextcloud项目开源地址：

[https://github.com/nextcloud/server](https://github.com/nextcloud/server)

Nextcloud项目官方站点：

[https://nextcloud.com/](https://nextcloud.com/)

![](https://yangyanghoho.github.io/img/in-post/post-nextcloud/11111.jpg)

手头上有一台CentOS的公有云虚拟机，那这次就用这台VM来尝试安装Nextcloud吧。

安装Nextcloud大概需要以下组件：

```ts
PHP
PHP-FPM
NGINX
MARIADB
NEXTCLOUD
```

NGINX安装相对简单，且我的服务器已经安装了nginx，并且一直在80端口运行中。所以先从PHP开始。


安装PHP相关组件
--

在服务器上用命令检查当前是否安装了PHP以及版本，发现我已经安装了php72w和php-fpm。

```ts
rpm -qa | grep php
```

于是直接安装对应版本的php-intl：


```ts
sudo yum install php72w-intl
```


调试NGINX和PHP
--

更改以下两个配置文件的user为同一个，我的nginx是用root用户运行的，那么php也用root。

```ts
/etc/nginx/nginx.conf
/etc/php-fpm.d/www.conf
```

取消www.conf的以下行的注释：

```ts
env[HOSTNAME] = $HOSTNAME
env[PATH] = /usr/local/bin:/usr/bin:/bin
env[TMP] = /tmp
env[TMPDIR] = /tmp
env[TEMP] = /tmp
```

并更改nginx default.conf的php部分：

```ts
sudo vi /etc/nginx/conf.d/default.conf
    location ~ \.php$ {
        root           /usr/share/nginx/html;
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }
```

创建session目录：

```ts
sudo mkdir -p /var/lib/php/session
```

启动php-fpm和nginx：

```ts
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
nginx -t
sudo systemctl restart nginx
```

注意php-fpm默认不能用root运行，可以通过以下命令解决：

```ts
php-fpm -R
```

在nginx html目录下，新建一个文件，名字为phpinfo.php：

```ts
<?php
phpinfo();
?>
```

如果一切顺利，现在用浏览器访问以下网址，应该可以看到php的版本信息：

```ts
http://your_domain/phpinfo.php
```

配置MARIADB
--

CentOS一般自带mariadb，直接进入数据库：

```ts
sudo mysql -u root -p;
```

创建nextcloud数据库：

```ts
create database nextcloud_db;
create user nextcloud_user@localhost identified by 'nextcloud_pass';
grant all privileges on nextcloud_db.* to nextcloud_user@localhost identified by 'nextcloud_pass';
flush privileges;
```

安装NEXTCLOUD
--

下载nextcloud：

```ts
wget https://download.nextcloud.com/server/releases/nextcloud-20.0.0.zip  --no-check-certificate
```

注意不同版本的nextcloud对php的版本有不同要求，这里我用nextcloud-20.0.0。

解压并放到nginx的html文件夹：

```ts
unzip nextcloud-20.0.0.zip
sudo mv nextcloud /usr/share/nginx/html/
sudo mkdir -p /usr/share/nginx/html/nextcloud/data/
```


配置NGINX
--

Nextcloud的官方网站给出了两种配置nginx的方法。

```ts
https://docs.nextcloud.com/server/20/admin_manual/installation/nginx.html
```

我选用其中一种子目录的方式，也就是说最后访问nextcloud应该是酱紫的：

```ts
http://your_domain/nextcloud
```

最后检查配置并重启nginx：
```ts
nginx -t
sudo systemctl restart nginx
```

初始配置
--

至此，如果一切顺利，访问以下url，就可以开始配置nextcloud了：

```ts
http://your_domain/nextcloud
```

配置页面大概长酱紫：

![](https://yangyanghoho.github.io/img/in-post/post-nextcloud/22222.jpg)

填好参数提交之后就坐等成功啦！用admin密码登陆之后大概酱紫：

![](https://yangyanghoho.github.io/img/in-post/post-nextcloud/33333.jpg)


结语
--

周末弄好之后，还下载了IOS客户端，感觉整个APP运行也不错，启动很快，感觉比较轻量化。

如果担心HTTP不够安全，可以做一套自签名的证书，具体可以参考本站RocketChat那篇文章。

将生成的crt和key填到nginx的配置里面，再把80端口的监听改成301重定向到https就行啦！

最后，祝大家玩的愉快！倒腾这类东西一定要注意心态呀。
