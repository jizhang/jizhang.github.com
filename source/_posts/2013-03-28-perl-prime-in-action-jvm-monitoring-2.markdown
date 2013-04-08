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

至于客户端，还请读者自行完成，可参考[相关文档](http://perldoc.perl.org/IO/Socket/INET.html)。

子进程
------

上述代码中有这样一个问题：当客户端建立了连接，但迟迟没有发送内容，那么服务端就会阻塞在`$line = <$client>`这条语句，无法接收其他请求。有三种解决方案：

1. 服务端读取信息时采用一定的超时机制，如果3秒内还不能读到完整的一行就断开连接。可惜Perl中并没有提供边界的方法来实现这一机制，需要自行使用`IO::Select`这样的模块来编写，比较麻烦。
2. 接受新的连接后打开一个子进程或线程来处理连接，这样就不会因为一个连接挂起而使整个服务不可用。
3. 使用非阻塞事件机制，当有读写操作时才会去处理。

这里我们使用第二种方案，即打开子进程来处理请求。

```perl
use IO::Socket::INET;

sub REAPER {
    my $pid;
    while (($pid = waitpid(-1, 'WNOHANG')) > 0) {
        print "SIGCHLD $pid\n";
    }
}

my $interrupted = 0;
sub INTERRUPTER {
    $interrupted = 1;
}

$SIG{CHLD} = \&REAPER;
$SIG{TERM} = \&INTERRUPTER;
$SIG{INT} = \&INTERRUPTER;

my $server = ...;

while (!$interrupted) {

    if (my $client = $server->accept()) {

        my $pid = fork();

        if ($pid > 0) {
            close($client);
            print "PID $pid\n";
        } elsif ($pid == 0) {
            close($server);

            my $line = <$client>;
            ...
            close($client);
            exit;

        } else {
            print "fork()调用失败\n";
        }
    }
}

close($server);
```

我们先看下半部分的代码。系统执行`fork()`函数后，会将当前进程的所有内容拷贝一份，以新的进程号来运行，即子进程。通过`fork()`的返回值可以知道当前进程是父进程还是子进程：大于0的是父进程；等于0的是子进程。子进程中的代码做了省略，执行完后直接`exit`。

上半部分的信号处理是做什么用的呢？这就是在多进程模型中需要特别注意的问题：僵尸进程。具体可以参考[这篇文章](http://shzhangji.com/blog/2013/03/27/fork-and-zombie-process/)。

而`$interrupted`变量则是用来控制程序是否继续执行的。当进程收到`SIGTERM`或`SIGINT`信号时，该变量就会置为真，使进程自然退出。

为何不直接使用`while (my $client = $server->accept()) {...}`呢？因为子进程退出时会向父进程发送`SIGCHLD`信号，而`accept()`函数在接收到任何信号后都会中断并返回空，使得`while`语句退出。
