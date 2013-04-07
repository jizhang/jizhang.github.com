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

正则表达式
----------

我们第一个任务是从“JVMPORT 2181”这样的字符串中提取“2181”这个端口号。解决方案当然是使用正则，而且Perl的强项之一正是文本处理：

```perl
my $line = 'JVMPORT 2181';
if ($line =~ /^JVMPORT ([0-9]+)$/) {
    print $1, "\n"; # 输出 2181
} else {
    print '匹配失败', "\n";
}
```

这里假设你知道如何使用正则表达式。

* `=~`运算符表示将变量和正则表达式进行匹配，如果匹配成功则返回真，失败则返回假。
* 匹配成功后，Perl会对全局魔术变量——`$0`至`$9`进行赋值，分别表示正则表达式完全匹配到的字符串、第一个子模式匹配到的字符串、第二个子模式，依此类推。
* `if...else...`是条件控制语句，其中`...} else if (...`可以简写为`...} elsif (...`。

调用命令行
----------

使用反引号（即大键盘数字1左边的按键）：

```perl
my $uname = `uname`;
print $unmae; # 输出 Linux
```

对于返回多行结果的命令，我们需要对每一行的内容进行遍历，因此会使用数组和`foreach`语句：

```perl
my $pid;
my $jvmport = '2181';
my @netstat = `netstat -lntp 2>/dev/null`;
foreach my $line (@netstat) {
    if ($line =~ /.*?:$jvmport\s.*?([0-9]+)\/java\s*$/) {
        $pid = $1;
        last;
    }
}
if ($pid) {
    print $pid, "\n";
} else {
    print '端口不存在', "\n";
}
```

* `$pid`变量的结果是2181端口对应的进程号。
* 这个正则可能稍难理解，但对照`netstat`的输出结果来看就可以了。
* `foreach`是循环语句的一种，用来遍历一个数组的元素，这里则是遍历`netstat`命令每一行的内容。注意，`foreach`可以直接用`for`代替，即`for my $line (@netstat) { ... }`。
* `last`表示退出循环。如果要进入下一次循环，可使用`next`语句。
