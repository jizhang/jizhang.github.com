---
layout: post
title: "Perl入门实战：JVM监控脚本（上）"
date: 2013-03-26 23:00
comments: true
categories: Tutorial
tags: [perl]
published: false
---

由于最近在搭建Zabbix监控服务，需要制作各类监控的模板，如iostat、Nginx、MySQL等，因此会写一些脚本来完成数据采集的工作。又因为近期对Perl语言比较感兴趣，因此决定花些时间学一学，写一个脚本来练练手，于是就有了这样一份笔记。

需求描述
--------

我们将编写一个获取JVM虚拟机状态信息的脚本：

1. 启动一个服务进程，通过套接字接收形如“JVMPORT 2181”的请求；
2. 执行`netstat`命令，根据端口获取进程号；
3. 执行`jstat`命令获取JVM的GC信息；`jstack`获取线程信息；`ps -o pcpu,rss`获取CPU和内存使用情况；
4. 将以上信息返回给客户端；

之所以需要这样一个服务是因为Zabbix Agent会运行在zabbix用户下，无法获取运行在其他用户下的JVM信息。

此外，Zabbix Agent也需要编写一个脚本来调用上述服务，这个在文章末尾会给出范例代码。

<!-- more -->
