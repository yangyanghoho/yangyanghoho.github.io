---
layout: post
title: "利用Python2发送UDP报文"
subtitle: "Using Python to test network reachability"
author: "Yang"
header-style: text
tags:
  - Python
  - UDP
  - Linux
---

上个月遇到某些源端口的UDP报文对端无法收到问题，但是又能ping通，一头雾水。

后来想到用Linux命令构建UDP socket，但是不是很好用，不能绑定指定的源端口。

于是开始研究用python发送，还是比较简单。代码如下：

```python
import sys,socket
 
UDP_SRC_IP = "39.1.1.2"
UDP_IP = "39.1.1.1"
UDP_SRC_PORT = int(sys.argv[1])
UDP_DEST_PORT = 2123
MESSAGE = "Hello, World!"
 
print "UDP target IP:", UDP_IP
print "UDP source port:", UDP_SRC_PORT
print "UDP target port:", UDP_DEST_PORT
print "message:", MESSAGE
 
sock = socket.socket(socket.AF_INET,
                     socket.SOCK_DGRAM)
sock.bind((UDP_SRC_IP,UDP_SRC_PORT))
sock.sendto(MESSAGE, (UDP_IP, UDP_DEST_PORT))
```

测试网络联通性时，运行下面命令即可。

```ts
python2 send_udp.py 12345
```

执行之前需要在对端设备抓包，确定是否收到。

如果端口较多，可以用for循环实现，也很简单，不在此处赘述了。
