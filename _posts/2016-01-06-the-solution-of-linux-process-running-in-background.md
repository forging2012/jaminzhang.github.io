---
layout: post
title: Linux 进程后台运行实现方法
description: "Linux 进程后台运行实现方法"
category: Linux
avatarimg:
tags: [Linux, Process, Background, nohup, screen, Shell]
duoshuo: true
---

# 引言

今天要写一个专用业务服务的维护脚本（包含启动、关闭、重启等功能），其中用一个专门的 Linux 普通用户来运行对应业务的服务进程。
通常要使业务进程后台运行，那么进程后台运行的方式有哪些呢？

# 方法1：nohup/setsid/&

临时使一个命令在后台运行

## 1.1 nohup

<pre> 

当用户注销（logout）或者网络断开时，终端会收到 HUP（hangup）信号从而关闭其所有子进程。
因此，我们的解决办法就有两种途径：要么让进程忽略 HUP 信号，要么让进程运行在新的会话里从而成为不属于此终端的子进程。

</pre>

nohup 的用途就是让提交的命令忽略 hangup 信号。
用法详细示例我就不写了，请 man nohup。

## 1.2 setsid

disown 能帮助我们来事后补救当前已经在运行了的作业。

<pre>

nohup 无疑能通过忽略 HUP 信号来使我们的进程避免中途被中断，但如果我们换个角度思考，
如果我们的进程不属于接受 HUP 信号的终端的子进程，那么自然也就不会受到 HUP 信号的影响了。
setsid 就能帮助我们做到这一点。

</pre>

这个的使用我比较少见到。

## 1.3 &

<pre>

将一个或多个命令包含在“()”中就能让这些命令在子 Shell 中运行中。  
当我们将"&"也放入“()”内之后，我们就会发现所提交的作业并不在作业列表中，也就是说，是无法通过 jobs 来查看的。  
新提交的进程的父 ID（PPID）为1（init 进程的 PID），并不是当前终端的进程 ID。
因此并不属于当前终端的子进程，从而也就不会受到当前终端的 HUP 信号的影响了。  

</pre>


# 方法2：disown

<pre>

如果事先在命令前加上 nohup 或者 setsid 就可以避免 HUP 信号的影响。
但是如果我们未加任何处理就已经提交了命令，该如何补救才能让它避免 HUP 信号的影响呢？  
这时想加 nohup 或者 setsid 已经为时已晚，只能通过作业调度和 disown 来解决这个问题了。

</pre>

这个的使用我比较少见到。

详细使用参考man disown和文章结尾的 Ref。

# 方法3：screen

screen 是在大批量操作时不二的选择了。

<pre>

场景：  
我们已经知道了如何让进程免受 HUP 信号的影响，
但是如果有大量这种命令需要在稳定的后台里运行，如何避免对每条命令都做这样的操作呢？  

解决方法：  
此时最方便的方法就是 screen 了。
简单的说，screen 提供了 ANSI/VT100 的终端模拟器，使它能够在一个真实终端下运行多个全屏的伪终端。
screen 的参数很多，具有很强大的功能。  

</pre>

详细使用参考man screen和文章结尾的Ref。

# 总结

<pre>

几种方法已经介绍完毕，我们可以根据不同的场景来选择不同的方案。
nohup/setsid 无疑是临时需要时最方便的方法，disown 能帮助我们来事后补救当前已经在运行了的作业，
而 screen 则是在大批量操作时不二的选择了。

</pre>

（我们的脚本中使用的是 1.3 中让进程在 Subshell 中使用 & 后台运行，当然另外也有其他的业务进程使用 nohup 和 screen 的方式来后台运行。）

# Ref
[Linux 技巧：让进程在后台可靠运行的几种方法](http://www.ibm.com/developerworks/cn/linux/l-cn-nohup/)  
[Linux Shell 的 & 、&& 、 ||](http://my.oschina.net/hanzhankang/blog/202754)  
