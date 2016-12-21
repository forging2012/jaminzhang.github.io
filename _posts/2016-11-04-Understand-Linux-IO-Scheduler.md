---
layout: post
title: 了解 Linux I/O 调度算法
description: "了解 Linux I/O 调度算法"
category: OS
avatarimg:
tags: [IO, Linux, Scheduler, cfq, deadline, noop, SSD]
duoshuo: true
---

# 引言
今天想了解下 Linux 下的 IO 调度算法。于是查看了相关资料，这里记录下。

# Linux 三种 IO 调度算法

<pre>
Linux IO 调度算法它要解决的核心问题是，如何提高块设备IO的整体性能？这一层也主要是针对机械硬盘结构而设计的。
众所周知，机械硬盘的存储介质是磁盘，磁头在盘片上移动进行磁道寻址，行为类似播放一张唱片。
这种结构的特点是，顺序访问时吞吐量较高，但是如果一旦对盘片有随机访问，那么大量的时间都会浪费在磁头的移动上，
这时候就会导致每次 IO 的响应时间变长，极大的降低 IO 的响应速度。
磁头在盘片上寻道的操作，类似电梯调度，如果在寻道的过程中，能把顺序路过的相关磁道的数据请求都“顺便”处理掉，
那么就可以在比较小影响响应速度的前提下，提高整体 IO 的吞吐量。这就是我们问什么要设计IO调度算法的原因。
在最开始的时期，Linux 把这个算法命名为 Linux 电梯算法。
目前在内核中默认开启了三种算法，其实严格算应该是两种，
因为第一种叫做 noop，就是空操作调度算法，也就是没有任何调度操作，
并不对 IO 请求进行排序，仅仅做适当的 IO 合并的一个 fifo 队列。

目前内核中默认的调度算法应该是 cfq，叫做完全公平队列调度。
这个调度算法人如其名，它试图给所有进程提供一个完全公平的 IO 操作环境。
它为每个进程创建一个同步 IO 调度队列，并默认以时间片和请求数限定的方式分配 IO 资源，
以此保证每个进程的 IO 资源占用是公平的，cfq 还实现了针对进程级别的优先级调度。

</pre>

<pre>
cfq 是通用服务器比较好的 IO 调度算法选择，对桌面用户也是比较好的选择。
但是对于很多 IO 压力较大的场景就并不是很适应，尤其是 IO 压力集中在某些进程上的场景。
因为这种场景我们需要更多的满足某个或者某几个进程的 IO 响应速度，
而不是让所有的进程公平的使用 IO，比如数据库应用。

deadline 调度（最终期限调度）就是更适合上述场景的解决方案。
deadline实现了四个队列，其中两个分别处理正常 read 和 write，按扇区号排序，进行正常 IO 的合并处理以提高吞吐量。
因为 IO 请求可能会集中在某些磁盘位置，这样会导致新来的请求一直被合并，可能会有其他磁盘位置的 IO 请求被饿死。
因此实现了另外两个处理超时 read 和 write 的队列，按请求创建时间排序，如果有超时的请求出现，就放进这两个队列，
调度算法保证超时（达到最终期限时间）的队列中的请求会优先被处理，防止请求被饿死。

noop 调度器是最简单的调度器。它本质上就是一个链表实现的 fifo 队列，并对请求进行简单的合并处理。
调度器本身并没有提供任何可疑配置的参数。

</pre>


# 各种调度器的应用场景选择

<pre>

根据以上几种 IO 调度算法的分析，我们应该能对各种调度算法的使用场景有一些大致的思路了。
从原理上看，cfq 是一种比较通用的调度算法，它是一种以进程为出发点考虑的调度算法，保证大家尽量公平。
deadline 是一种以提高机械硬盘吞吐量为思考出发点的调度算法，尽量保证在有 IO 请求达到最终期限的时候进行调度，
非常适合业务比较单一并且 IO 压力比较重的业务，比如数据库。
而 noop 呢？其实如果我们把我们的思考对象拓展到固态硬盘，那么你就会发现，无论 cfq 还是 deadline，
都是针对机械硬盘的结构进行的队列算法调整，而这种调整对于固态硬盘来说，完全没有意义。
对于固态硬盘来说，IO 调度算法越复杂，额外要处理的逻辑就越多，效率就越低。
所以，固态硬盘这种场景下使用 noop 是最好的，deadline 次之，而 cfq 由于复杂度的原因，无疑效率最低。

</pre>


# Ref
[Linux的IO调度](http://liwei.life/2016/03/14/linux_io_scheduler/)  