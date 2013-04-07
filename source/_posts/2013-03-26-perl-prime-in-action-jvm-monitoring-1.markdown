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

Hello, world!
-------------

还是要不免俗套地来一个helloworld，不过我们的版本会稍稍丰富些：

```perl
#!/usr/bin/perl
use strict;
my $name = 'Jerry';
print "Hello, $name!\n"; # 输出 Hello, Jerry!
```

将该文件保存为`hello.pl`，可以用两种方式执行：

```bash
$ perl hello.pl
Hello, Jerry!
$ chmod 755 hello.pl
$ ./hello.pl
Hello, Jerry!
```

* 所有的语句都以分号结尾，因此一行中可以有多条语句，但并不提倡这样做。
* `use`表示加载某个模块，加载[`strict`模块](http://search.cpan.org/~rjbs/perl-5.16.3/lib/strict.pm)表示会对当前文件的语法做出一些规范和约束。比如将`my $name ...`前的`my`去掉，执行后Perl解释器会报错。建议坚持使用该模块。
* `$name`，一个Perl变量。`$`表示该变量是一个标量，可以存放数值、字符串等基本类型。其它符号有`@`和`%`，分别对应数组和哈希表。
* `my`表示声明一个变量，类似的有`our`、`local`等，将来接触到变量作用域时会了解。
* 字符串可以用单引号或双引号括起来，区别是双引号中的变量会被替换成实际值以及进行转移，单引号则不会。如`'Hello, $name!\n'`中的`$name`和`\n`会按原样输出，而不是替换为“Jerry”和换行符。
* `print`语句用于将字符串输出到标准输出上。
* `#`表示注释。

