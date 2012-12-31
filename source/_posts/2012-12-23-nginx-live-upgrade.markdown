---
layout: post
title: "Nginx热升级"
date: 2012-12-23 22:11
comments: true
categories: Ops
tags: [nginx]
published: false
---

Nginx热升级
===========

系统管理员可以使用Nginx提供的信号机制来对其进行维护，比较常用的是`kill -HUP <master pid>`命令，它能通知Nginx使用新的配置文件启动工作进程，并逐个关闭旧进程，完成平滑切换。当需要对Nginx进行版本升级或增减模块时，为了不丢失请求，可以结合使用`USR2`、`WINCH`等信号进行平滑过度，达到热升级的目的。如果中途遇到问题，也能立刻回退至原版本。

操作步骤
--------

1、备份原Nginx二进制文件；

2、编译新Nginx源码，安装路径需与旧版一致；

3、向主进程发送`USR2`信号，Nginx会启动一个新版本的master进程和工作进程，和旧版一起处理请求：

4、向原Nginx主进程发送`WINCH`信号，它会逐步关闭旗下的工作进程（主进程不退出），这时所有请求都会由新版Nginx处理：

5、如果这时需要回退，可向原Nginx主进程发送`HUP`信号，它会重新启动工作进程， **仍使用旧版配置文件** 。尔后可以将新版Nginx进程杀死（使用`QUIT`、`TERM`、或者`KILL`）：

6、如果不需要回滚，可以将原Nginx主进程杀死，至此完成热升级。

切换过程中，Nginx会将旧的`.pid`文件重命名为`.pid.oldbin`文件，并在旧进程退出后删除。

原理简介
--------

### 多进程模式下的请求分配方式

Nginx默认工作在多进程模式下，即主进程（master process）启动后完成配置加载和端口绑定等动作，`fork`出指定数量的工作进程（worker process），这些子进程会持有监听端口的文件描述符（fd），并通过在该描述符上添加监听事件来接受连接（accept）。

### 信号的接收和处理

Nginx主进程在启动完成后会进入等待状态，负责响应各类系统消息，如SIGCHLD、SIGHUP、SIGUSR2等。

```c
// src/os/unix/ngx_process_cycle.c
void
ngx_master_process_cycle(ngx_cycle_t *cycle)
{
    sigset_t           set;

    sigemptyset(&set);
    sigaddset(&set, SIGCHLD);
    sigaddset(&set, ngx_signal_value(NGX_RECONFIGURE_SIGNAL));
    sigaddset(&set, ngx_signal_value(NGX_CHANGEBIN_SIGNAL));

    if (sigprocmask(SIG_BLOCK, &set, NULL) == -1) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "sigprocmask() failed");
    }

    for ( ;; ) {

        sigsuspend(&set); // 等待信号

        // 信号回调函数定义在 src/os/unix/ngx_process.c 中，
        // 它只负责设置全局变量，实际处理逻辑在本文件中。
        if (ngx_change_binary) { 
            ngx_change_binary = 0;
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "changing binary");
            ngx_new_binary = ngx_exec_new_binary(cycle, ngx_argv);
        }
        
    }
}
```

上述代码中的`ngx_exec_new_binary`函数会调用`execve`系统函数运行一个新的Nginx主进程，并将当前监听端口的文件描述符通过环境变量的方式传递给新的主进程，这样新的主进程`fork`出的子进程就同样能够对监听端口添加事件回调，接受连接，从而使得新老Nginx共同处理用户请求。

其它方式
--------

### Keepalived主备切换

### Tengine动态模块

Nginx信号汇总
-------------

