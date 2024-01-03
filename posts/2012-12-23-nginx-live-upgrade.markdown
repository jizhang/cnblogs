---
layout: post
title: "Nginx 热升级"
date: 2012-12-23 22:11
comments: true
categories: Programming
tags: [nginx]
published: true
---

系统管理员可以使用Nginx提供的信号机制来对其进行维护，比较常用的是`kill -HUP <master pid>`命令，它能通知Nginx使用新的配置文件启动工作进程，并逐个关闭旧进程，完成平滑切换。当需要对Nginx进行版本升级或增减模块时，为了不丢失请求，可以结合使用`USR2`、`WINCH`等信号进行平滑过度，达到热升级的目的。如果中途遇到问题，也能立刻回退至原版本。

操作步骤
--------

1、备份原Nginx二进制文件；

2、编译新Nginx源码，安装路径需与旧版一致；

3、向主进程发送`USR2`信号，Nginx会启动一个新版本的master进程和工作进程，和旧版一起处理请求：

```bash
prey:~ root# ps -ef|grep nginx
 127     1   nginx: master process /usr/local/nginx-1.2.4/sbin/nginx
 129   127   nginx: worker process
prey:~ root# kill -USR2 127
prey:~ root# ps -ef|grep nginx
 127     1   nginx: master process /usr/local/nginx-1.2.4/sbin/nginx
 129   127   nginx: worker process  
5180   127   nginx: master process /usr/local/nginx-1.2.4/sbin/nginx
5182  5180   nginx: worker process  
```

<!-- more -->

4、向原Nginx主进程发送`WINCH`信号，它会逐步关闭旗下的工作进程（主进程不退出），这时所有请求都会由新版Nginx处理：

```bash
prey:~ root# kill -WINCH 127
prey:~ root# ps -ef|grep nginx
 127     1   nginx: master process /usr/local/nginx-1.2.4/sbin/nginx
5180   127   nginx: master process /usr/local/nginx-1.2.4/sbin/nginx
5182  5180   nginx: worker process
```

5、如果这时需要回退，可向原Nginx主进程发送`HUP`信号，它会重新启动工作进程， **仍使用旧版配置文件** 。尔后可以将新版Nginx进程杀死（使用`QUIT`、`TERM`、或者`KILL`）：

6、如果不需要回滚，可以将原Nginx主进程杀死，至此完成热升级。

```bash
prey:~ root# kill 127
prey:~ root# ps -ef|grep nginx
5180     1   nginx: master process /usr/local/nginx-1.2.4/sbin/nginx
5182  5180   nginx: worker process  
```

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
        // 它只负责设置全局变量，实际处理逻辑在这里。
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

除了使用上述方式进行Nginx热升级，还可以选择以下两种方式：

### Keepalived主备切换

Nginx用作LB时一般会用[Keepalived](http://www.keepalived.org/)做热备，即DNS解析指向一个虚拟IP（VIP），主备服务器上分别启动Keepalived进程，当Master健康检查失败，Slave会自动抢夺VIP，完成切换。

在进行热升级时就可以使用这种方式，在Slave上进行Nginx升级，然后关闭Master的Keepalived进程，完成VIP的漂移。测试完成后可以对继续对Master进行升级操作，或选择回滚。

### Tengine动态模块

当需要增加Nginx模块时，必须对Nginx源码进行重新编译，然后采用上面提到的方式进行热升级。Nginx官网上说未来并无打算增加动态模块加载的功能，至少1.x中不会。

如果你愿意使用[Tengine](http://tengine.taobao.org/)，淘宝开发的一个Nginx分支，它提供了动态模块加载（DSO）功能。这里简单介绍一下使用方法：

#### 动态添加内部模块

以`http_sub_module`为例，到Nginx源码目录执行以下命令：

```bash
$ ./configure --with-http_sub_module=shared
$ make
$ sudo make dso_install
```

它会将编译好的`ngx_http_sub_filter_module.so`文件复制到`/usr/local/nginx/modules`目录下。随后在Nginx配置文件中的最外层添加以下内容：

```
dso {
    load ngx_http_sub_filter_module.so;
}
```

执行`sbin/nginx -s reload`，就能完成模块的加载。

#### 第三方模块

对于第三方模块，Tengine提供了`dso_tool`命令，能够一步安装：

```bash
sbin/dso_tool --add-module=/home/dso/lua-nginx-module
```

Nginx信号汇总
-------------

以下内容译自：http://wiki.nginx.org/CommandLine

### 主进程支持的信号

* `TERM`, `INT`: 立刻退出
* `QUIT`: 等待工作进程结束后再退出
* `KILL`: 强制终止进程
* `HUP`: 重新加载配置文件，使用新的配置启动工作进程，并逐步关闭旧进程。
* `USR1`: 重新打开日志文件
* `USR2`: 启动新的主进程，实现热升级
* `WINCH`: 逐步关闭工作进程

### 工作进程支持的信号

* `TERM`, `INT`: 立刻退出
* `QUIT`: 等待请求处理结束后再退出
* `USR1`: 重新打开日志文件

### nginx -s signal 支持的信号

* `stop`: 等价于`TERM`, `INT`
* `quit`: `QUIT`
* `reopen`: `USR1`
* `reload`: `HUP`

### 使用方法

```bash
$ sbin/nginx -s reload
$ kill -HUP $(cat logs/nginx.pid)
```
