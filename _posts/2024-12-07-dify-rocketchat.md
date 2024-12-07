---
layout: post
title: "用Dify+LLM推送消息给RocketChat Webhook"
subtitle: "Push message to RocketChat Webhook using Dify and LLM"
author: "Yang"
header-style: text
tags:
  - LLM
  - Dify
  - RocketChat
  - webhook
---


引言
--

一直想实现一个每日定时给家里人和自己发送天气和其他定制消息的功能，最近研究了Dify，一个可以实现workflow和AI agent的工具，ta支持私有化部署，也支持官方网站直接试用。

先决条件如下：

* Dify账号(可以用Github和Google账户登录)。
* 一个可以接收incoming webhook的消息工具，这里我用RocketChat。
* 大语言模型的API key，这里我用Azure OpenAI的API key。


试用Dify
--

登录Dify，在'设置'-'模型供应商'里按情况填上自己的API Key等信息。

[https://cloud.dify.ai](https://cloud.dify.ai)

然后在'工作室'-'创建空白应用'处选择'工作流'并命名：

![](https://yangyanghoho.github.io/img/in-post/post-dify/d1.jpg)


调试工作流
--

进入新建的工作流，按照下图新建各个组件并将他们连起来：

![](https://yangyanghoho.github.io/img/in-post/post-dify/d2.jpg)

需要注意的是，天气查询工具需要去这个网址注册并申请一个免费的API key才能使用(注意这个API key需要一段时间才能生效可用，建议报错的话等几个小时再试)：

[https://home.openweathermap.org/users/sign_in](https://home.openweathermap.org/users/sign_in)

另外，好像有一个奇怪的bug，这个组件需要用户开启设置中模型里的OpenAI GPT-4o-mini模型的开关，即便我只用Azure OpenAI的模型。

天气组件支持用拼音填写城市名称，并通过API key访问拿到当前城市天气信息。

然后配置LLM将时间组件和天气组件的输出都作为LLM组件的输入，并配置要发给LLM的system消息，像这样：

![](https://yangyanghoho.github.io/img/in-post/post-dify/d3.jpg)

最后代码执行组件里，我们运行python代码，将LLM的输出作为输入并通过python发给RocketChat的webhook：

![](https://yangyanghoho.github.io/img/in-post/post-dify/d4.jpg)

其中的代码如下：

```ts
import urllib.request
import json
import ssl

#msg = "hello~"

def main(msg: str) -> dict:
    # 要发送的数据
    #msg = text
    data = json.dumps({"text": msg}).encode('utf-8')
    
    # 要发送请求的URL
    url = "https://your_domain.com/rocketchat/hooks/your_web_hook_url"
    
    # 请求头
    headers = {
        'Content-Type': 'application/json'
    }
    
    # 创建一个忽略证书错误的上下文
    context = ssl._create_unverified_context()
    
    # 创建请求对象
    request = urllib.request.Request(url, data=data, headers=headers, method='POST')
    
    # 发送请求并获取响应
    try:
        with urllib.request.urlopen(request, context=context) as response:
            response_data = response.read()
            print(response_data.decode('utf-8'))
            response_json = json.loads(response_data)
            response_json['success'] = str(response_json['success'])
            return response_json
    except urllib.error.URLError as e:
        print(e.reason)
```

这样到此我们的工作流基本配置完成。

关于RocketChat Webhook URL，请看下一节。


RocketChat Webhook
--

之前云服务器装的RocketChat版本太老了，这次直接卸载，停掉服务，用官方提供的Docker部署方法直接部署7.0版本。

[https://docs.rocket.chat/docs/deploy-with-docker-docker-compose](https://docs.rocket.chat/docs/deploy-with-docker-docker-compose)

需要注意的是，如果需要将RocketChat作为子域名，那么需要更改默认的.env文件里面的ROOT_URL为https://your_domain.com/rocketchat。

另外也建议配置RELEASE为特定的稳定版本，我设置成了7.0.0。

部署完成后，依然通过nginx做反向代理，具体方法参见本站那篇旧的安装RocketChat的文章。

安装向导和其他配置完成后，通过官方的指南新建入方向的Webhook，可以设置将消息推送给个人，也可以推送给频道：

[https://docs.rocket.chat/docs/integrations](https://docs.rocket.chat/docs/integrations)

将拿到的Webhook URL配置到Dify代码执行模块里python代码的url处即可。


测试工作流
--

点击工作流页面的发布，然后点击运行，工作流会从START模块开始运行。

结束后检查运行情况，排查报错等故障。

如果一切无误的话，手机的RocketChat APP应该可以收到关于今天日期的天气的推送了。


定时任务访问API
--

Dify工作流支持API调用，在工作流页面左侧点击'访问API'，在右侧点击'API密钥'，保存生成的API key。

在云服务器上新建shell脚本：

```ts
#!/bin/bash

DIFY_API="https://api.dify.ai/v1/workflows/run"

API_KEY="app-your_api_key"

curl -X POST $DIFY_API \
--header "Authorization: Bearer $API_KEY" \
--header 'Content-Type: application/json' \
--data '{
    "inputs": {},
    "response_mode": "blocking",
    "user": "bot"
}'
```

给脚本可执行权限后，可以直接执行脚本并测试。

通过后可以用crontab -e编辑定时任务，像这样：

```
0 0 * * * /home/yangza/agent/curl_dify.sh
```

因为我的服务器是UTC时间运行的，这样可以实现每天早8点推送消息给我的手机。


结语
--

虽然绕了一大圈，但是还是有点儿意思的。跑通了工作流还是很开心。

后续可以考虑推送更多类型的信息给自己和家人的手机。
