---
layout: post
title: "Perl入门实战：JVM监控脚本（下）"
date: 2013-03-28 15:28
comments: true
categories: Tutorial
tags: [perl]
published: false
---

套接字
------

使用套接字（Socket）进行网络通信的基本流程是：

* 服务端：监听端口、等待连接、接收请求、发送应答；
* 客户端：连接服务端、发送请求、接收应答。

```perl
use IO::Socket::INET;

my $server = IO::Socket::INET->new(
    LocalPort => 10060,
    Type => SOCK_STREAM,
    Reuse => 1,
    Listen => SOMAXCONN
) || die "服务创建失败\n";

while (my $client = $server->accept()) {

    my $line = <$client>;
    chomp($line);

    if ($line =~ /^JVMPORT ([0-9]+)$/) {
        print "RECV $1\n";
        print $client "OK\n";
    } else {
        print "ERROR $line\n";
        print $client "ERROR\n";
    }

    close($client);
}

close($server);
```

<!--more-->

* `IO::Socket::INET`是一个内置模块，`::`符号用来分隔命名空间。
* `->new`运算符是用来创建一个类的实例的，这涉及到面向对象编程，我们暂且忽略。
* `(key1 => value1, key2 => value2)`是用来定义一个哈希表的，也就是键值对。这里是将哈系表作为参数传递给了`new`函数。请看以下示例。对于哈系表的进一步操作，我们这里暂不详述。

```perl
sub hello {
    my %params = @_;
    print "Hello, $params{'name'}!\n";
}

hello('name' => 'Jerry'); # 输出 Hello, Jerry!
```

* `while (...) {...}`是另一种循环结构，当圆括号的表达式为真就会执行大括号中的语句。
* `$server->accept()`表示调用`$server`对象的`accept()`函数，用来接受一个连接。执行这个函数时进程会阻塞（进入睡眠），当有连接过来时才会唤醒，并将该连接赋值给`$client`变量。
* `<...>`运算符表示从文件中读取一行，如：

```perl
open my $fd, '<', '/proc/diskstats';
while (my $line = <$fd>) {
    print $line;
}
```

由于套接字也可以作为文件来看待，所以就能使用`<...>`运算符。关于`open`函数和其他文件操作，读者可参考[这篇文章](http://perl5maven.com/open-and-read-from-files)。

* `chomp()`函数用来将字符串末尾的换行符去掉。它的用法也比较奇特，不是`$line = chomp($line)`，而是`chomp($line)`，这里`$line`是一次引用传递。
* 细心的读者会发现，第二句`print`增加了`$client`，可以猜到它是用来指定`print`的输出目标。默认情况下是标准输出。

我们打开两个终端，一个终端执行服务端，另一个终端直接用Bash去调用。

```bash
# 客户端
$ echo 'JVMPORT 2181' | nc 127.0.0.1 10060
OK
$ echo 'hello' | nc 127.0.0.1 10060
ERROR

# 服务端
$ ./socket-server.pl
RECV 2181
ERROR hello
```

至于客户端，还请读者自行完成，可参考[相关文档](http://perldoc.perl.org/IO/Socket/INET.html);
