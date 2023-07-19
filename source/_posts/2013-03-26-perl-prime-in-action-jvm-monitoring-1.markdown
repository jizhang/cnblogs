---
layout: post
title: "Perl 入门实战：JVM 监控脚本（上）"
date: 2013-03-26 23:00
comments: true
categories: Programming
tags: [perl, tutorial]
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
print $uname; # 输出 Linux
my $pid = '1234';
$line = `ps -ef | grep $pid`; # 支持管道符和变量替换
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

数组
----

下面我们要根据进程号来获取JVM的GC信息：（“2017”为上文获取到的进程号）

```bash
$ jstat -gc 2017
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       PC     PU    YGC     YGCT    FGC    FGCT     GCT
192.0  192.0   0.0    50.1   1792.0   682.8     4480.0     556.3    21248.0 9483.2      3    0.008   0      0.000    0.008
```

如何将以上输出结果转换为以下形式？

```
s0c 192.0
s1c 192.0
...
```

这时我们就需要更多地使用数组这一数据结构：

```perl
my $pid = 2017;
my @jstat = `jstat -gc $pid`;

$jstat[0] =~ s/^\s+|\s+$//;
$jstat[1] =~ s/^\s+|\s+$//;

my @kv_keys = split(/\s+/, $jstat[0]);
my @kv_vals = split(/\s+/, $jstat[1]);

my $result = '';
for my $i (0 .. $#kv_keys) {
    $result .= "$kv_keys[$i] $kv_vals[$i]\n";
}

print $result;
```

* 使用`$jstat[0]`获取数组的第一个元素，数组的下标从0开始，注意这里的`$`符号，而非`@`。
* `$#kv_keys`返回的是数组最大的下标，而非数组的长度。
* `for my $i (0 .. 10) {}`则是另一种循环结构，`$i`的值从0到10（含0和10）。

对于正则表达式，这里也出现了两个新的用法：

* `s/A/B/`表示将A的值替换为B，上述代码中是将首尾的空格去除；
* `split(A, B)`函数表示将字符串B按照正则A进行分割，并返回一个数组。

另外在学习过程中还发现了这样一种写法：

```perl
my @jstat;

$jstat[0] =~ s/^\s+|\s+$//;
$jstat[1] =~ s/^\s+|\s+$//;

map { s/^\s+|\s+$// } @jstat;
```

`map`函数会对数组中的每个元素应用第一个参数指向的函数（这里是一个匿名函数），当需要处理的数组元素很多时，这种是首选做法。具体内容读者可以自己去了解。

函数
----

我们可以用以下命令来获取指定进程的CPU和内存使用率：

```bash
$ ps -o pcpu,rss -p 2017
%CPU   RSS
 0.1 21632
```

格式和`jstat`是一样的，为了不再写一遍上文中的代码，我们可以将其封装为函数。

```perl
sub kv_parse {

    my @kv_data = @_;

    map { s/^\s+|\s+$// } @kv_data;

    my @kv_keys = split(/\s+/, $kv_data[0]);
    my @kv_vals = split(/\s+/, $kv_data[1]);

    my $result = '';
    for my $i (0 .. $#kv_keys) {
        $result .= "$kv_keys[$i] $kv_vals[$i]\n";
    }

    return $result;

}

my $pid = 2017;
my @jstat = `jstat -gc $pid`;
my @ps = `ps -o pcpu,rss -p $pid`;

print kv_parse(@jstat);
print kv_parse(@ps);
```

`sub`表示定义一个函数（subroutine），和其他语言不同的是，它没有参数列表，获取参数使用的是魔术变量`@_`：

```perl
sub hello {
    my $name1 = $_[0];
    $name1 = shift @_;
    my $name2 = shift(@_);
    my $name3 = shift;
}

hello('111', '222', '333');
hello '111', '222', '333';
&hello('111', '222', '333');
```

* `$_[0]`和`shift @_`返回的都是第一参数。不同的是，`shift`函数会将这个参数从`@_`数组中移除；
* `shift @_`和`shift(@_)`是等价的，因为调用函数时参数列表可以不加括号；
* `shift @_`和只写`shift`也是等价的，该函数若不指定参数，则默认使用`@_`数组。
* `&`符号也是比较特别的，主要作用有两个：一是告诉Perl解释器`hello`将是一个用户定义的函数，这样就不会和Perl原生关键字冲突；二是忽略函数原型（prototype）。具体可以参考这篇文章：[Subroutines Called With The Ampersand](https://www.socialtext.net/perl5/subroutines_called_with_the_ampersand)。

当传递一个数组给函数时，该数组不会被作为`@_`的第一个元素，而是作为`@_`本身。这也是很特别的地方。当传递多个数组，Perl会将这些数组进行拼接：

```perl
sub hello {
    for my $i (@_) {
        print $i;
    }
}

my @arr1 = (1, 2); # 使用圆括号定义一个数组，元素以逗号分隔。
my @arr2 = (3, 4);

hello @arr1, @arr2; # 输出 1234
```

小结
----

对于初学者来讲，本文的信息量可能有些大了。但如果你已经有一定的编程经验（包括Bash），应该可以理解这些内容。

Perl文化的特色是“不只一种做法来完成一件事情”，所以我们可以看到很多不同的写法。但也有一些是大家普遍接受的写法，所以也算是一种规范吧。

下一章我们会继续完成这个监控脚本。

PS：本文的示例代码可以从[Github](https://github.com/jizhang/blog-demo/tree/master/perl-jvm-monitor)中下载。
