# Go内存模型

[介绍](#介绍)
[建议](#建议)
[先行发生原则](#先行发生原则)
[同步](#同步)
&nbsp;&nbsp;&nbsp;&nbsp;[初始化](#初始化)
&nbsp;&nbsp;&nbsp;&nbsp;[Go协程的创建](#Go协程的创建)
&nbsp;&nbsp;&nbsp;&nbsp;[Go协程的销毁](#Go协程的销毁)
&nbsp;&nbsp;&nbsp;&nbsp;[香奈儿通信](#香奈儿通信)
&nbsp;&nbsp;&nbsp;&nbsp;[锁](#锁)
&nbsp;&nbsp;&nbsp;&nbsp;[Once](#Once)
[错误的同步](#错误的同步)

## 介绍