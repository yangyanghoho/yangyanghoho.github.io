---
layout: post
title: "从这里开始，保护云主机呀！"
subtitle: "To protect your cloud vm"
author: "Yang"
header-style: text
tags:
  - Linux
  - Cloud
  - CentOS
---

> 这篇文章转载自[我的公众号](https://mp.weixin.qq.com/s/3-UIV6sNV8On5BLzddLUwg)

引言
--

云服务因为其强大的功能，相对低的价格和灵活的部署已经走入了很多人的工作和生活中，很多朋友也都有了自己的云服务器。像我自己就用一台CentOS的VM加一个公网地址，可以做很多事情：

* 做个论坛
* 搭个博客
* 个人主页
* 内网穿透
* 个人网盘
* 公众号或者小程序的服务端
* 你懂的...

但是如果大家有一个可以从公网IP默认SSH端口登陆的服务器，大家就会知道互联网的凶残了，一旦被盯上，暴力破解啊各种乱猜啊就来了，像酱紫：

![](https://mmbiz.qpic.cn/mmbiz_png/8f64ibHobbquAOeS5U8J5MiaS3ymLFp6SMDsxacrJgKuLc5n3ibODib2HwqZR3rTL2V7shm1dlkRcDRQNjlHjibh2eQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

里面有大量的各种IP用各种用户名尝试登陆并失败的记录，如果我们的服务器用户名很好猜（比如test），密码又简单（比如1234qwer），那被破解就是迟早的事儿嘛。

为了不轻易交出我们服务器的控制权，我们可以做很多事情来增加潜在攻击者的破解难度。


更改默认SSH端口
--

SSH默认监听TCP的22端口，我们可以通过更改服务器上的SSH配置来更改SSH监听端口，这样对方就需要扫描端口才能知道我们的SSH端口。以CentOS7为“栗”，更改/etc/ssh/sshd_config文件里面的Port：

![](https://mmbiz.qpic.cn/mmbiz_png/8f64ibHobbquAOeS5U8J5MiaS3ymLFp6SMmVibzANZ72lBtuGhjmZic1XxicaulCUiah9Ytia9uSJyevIC70TOIxo0dHg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

再重启SSH进程即可。


有限放通入站端口
--

云服务提供商必然提供入方向的端口过滤功能，所以我们可以通过配置有限制的放通可选端口，并拒绝其他端口的访问请求，像酱紫：

![](https://mmbiz.qpic.cn/mmbiz_png/8f64ibHobbquAOeS5U8J5MiaS3ymLFp6SMFGu8s3TJCxn5GiaicRgsw5H1XEEsPLAJmMMX6TJYKr74nhPy7PXqmcoQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

用HTTP默认放通80，用HTTPS默认放通443，不要留无用的端口。


增加密码强度
--

设置密码需要增加复杂性，大小写字母/数字/特殊符号都要有。增加密码复杂性的同时也要保证自己好记。


将登陆失败的IP拉黑
--

即便更改了SSH默认端口，我们会发现，依然会有各种用户名来自各种IP尝试SSH到我们的公网IP，所以最后动用杀招一棒子打死一批，那就是通过shell脚本把secure log里面的登陆失败的IP添加到/etc/hosts.deny里面去。如果大家懒得自己写，可以参考这里的脚本（来自博客园非走不可的博客）：

```ts

#!/bin/bash

LIST=""

#过滤出协议，尝试连接主机的ip
LIST=$(cat /var/log/secure | grep "authentication failure" | awk '{print$14}' | sed -e 's/rhost=//g' -e 's/ /_/g' | uniq)

#Trusted Hosts
excludeList=( "192.168.30.55" )

function chkExcludeList()
{
for j in "${excludeList[@]}"; do
    if [[ "$1" == $j ]]; then
        return 10
    fi
done
return 11
}

#检查并追加到hosts.deny文件中
for i in $LIST; do
    chkExcludeList "$i"
        if [ $? != "10" ]; then
            if [ "$(grep $i /etc/hosts.deny)" = "" ]; then
                echo "ALL: $i : DENY" >> /etc/hosts.deny
            fi
        fi
done
```

通过crontab每几分钟运行一次这个shell脚本即可。


其他防暴力破解工具
--

denyhosts fail2ban是两个可以防破解的工具，我个人没有用过，不过如果大家不想用拉黑源IP的方案，可以试试fail2ban，安装简单。

最后祝大家用云主机玩的开心，玩的安全！
