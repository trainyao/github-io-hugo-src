---
author: "trainyao"
date: 2018-06-10 22:48:30
title: skynet里的定时器
linktitle: skynet里的定时器
weight: 10
categories: ["skynet"]
---

这周花了点时间看了 skynet 关于定时器功能的源码,以及扩展的了解了一下定时器实现的其他方案,写一遍博客总结一下.内容会分为下面几部分:

- skynet的定时器结构图
- skynet定时器执行过程
- 关于skynet定时器的一些思考
- 其他定时器方案
- linux定时器方案
- 总结


### 1. skynet 的定时器结构图

![skynet timer](../img/skynet/timer/skynettimer.png)

### 2. skynet 定时器执行过程

首先确定定时器里面的几个概念:

- `tick`: 指 skynet timer 线程每一次处理事件的循环.由于这个循环每隔 0.00025 秒会执行一次,很像时钟滴答滴答的走,很多书籍都称为 tick
- `skynet 启动时间戳`: 顾名思义,比如 1528613417 时间戳(北京时间 2018/6/10 14:50:17)代表 skynet 是这个时间点启动的,而且这个时间是不会变的.这个比较容易理解
- `系统启动时间数`: 单位 0.01 秒. 顾名思义,但是为什么说是时间数呢?因为这是一个时间的量,比如300,表示系统启动了3秒. skynet timer 线程在每一次 tick 都会去同步这个时间量
- `skynet 启动时间数`: 单位 0.01 秒. 与系统启动时间数类似,如 300,表示 skynet 启动了 3 秒,每次 tick skynet timer 线程也会去更新这个时间量

首先需要说明的是, skynet 定时器的时间刻度是百分秒(0.01秒).意味着 skynet 最小可以定义事件 0.01 秒后执行.这个精度对于游戏领域可以说是够用了.

skynet 初始化时会创建5个定时器的事件槽,分别是 Near,level0-3.每个定时器事件是按照触发时间(这个触发时间是以 skynet 启动时间数作为参照的.比如触发时间为 300,事件注册时 skynet 创建时间量为 100,表示注册这个事件在 2 秒后执行)为索引插入到对应的槽里面的.用索引的方式安排这些事件保证了查询和插入的性能( 时间复杂度为O(1) ).

那 skynet 是怎么安排这些事件应该放入哪个槽呢? 5个不同的槽又是什么关系呢?

skynet 根据触发时间得出事件应该插入到哪个槽,哪个索引.触发时间是一个 32 位的整型值. skynet 将这 32位分为 6 6 6 6 8 位每个区间.如果前 24 位有值,表示这个事件是比较延后触发的,skynet 将它们安排到 level 槽里面,表示这些事件暂时不关注,高 0-6 位有值的插入到 level 3 槽里,高 7-12 位有值的插入到 level 2 槽里,如此类推,高 19-24 位有值的,插入到 level 0 槽里.而高 24 位没有值,低 8 位有值的事件,skynet 认为是需要比较关注的事件,并以低 8 位为索引插入到 Near(near 可能就是比较近会发生的事件的意思)槽里.具体插入后的结果就像上面图中画的一样.

skynet timer 的每次 tick ,都会以系统时间为参照,检查这次 tick 落后了系统时间多少百分秒,然后将落后的系统时间同步回 skynet 启动时间数里,并执行这个时间数低 8 位为索引的 Near 槽里的事件.

你可能会有一个疑惑,假如我注册了一个 0x01 01 为触发时间的事件 A (这个事件应该被插入到 Level 0[0x01] 槽里),还有一个 0xff 为触发时间的事件 B (这个事件应该被插入到 Near[0xff]槽里),当时间走到 0xff 时,时间 B 会被顺利执行.但是当时间走到 0x01 01 时,事件 A 怎么办?它会被怎么执行?

skynet 在每个`高 24 有值的"整点"`,都会将对应的 Level 槽里的事件整理到 Near 槽里,然后执行.高 24 位有值的整点指 0x0f00 0x0100 0x1000 0000 这些时间点.如上面的例子,当时间走到 0x0100 的时候, skynet 会将 Level 0[0x01(0x0101 的`低 15-9 位`是 0x01 )]槽里的事件整理到 Near 槽里,即将事件 B 插入到 Near[0x01(0x0101 的`低 8-0 位`是0x01)] 槽里.时间再走 1 个百分秒,timer 线程就会执行 Near[0x01]槽里的事件,就会将事件 B 处理掉了.

这也是为什么上图在 Near[0xf0] 里会有事件的触发时间是 0xf0 ,也有的是 0x0f f0,0x0f f0 的事件是 timer 线程从 Level 0 槽里整理过去的.`只不过`,上图有一点不对的是, 0xf0和0x0f f0 时间不可能同时存在在 Near[0xf0] 里面,这里只是为了说明 Near 槽里面有可能出现的事件,忽略细节.而 Level 0 槽里,一个索引的列表会有低 8 位有各种值的事件包含在里面,等到 timer 线程走到整点的时候,再被处理整理到 Near 槽里面去.

上面就是 skynet 定时器整个工作的过程.用索引的方式保证了插入和查询的高效,而分槽的方式保证了触发事件的高效,着急的事件重点关注,不着急的事件延后处理.而按 66668 位每个区间来划分和注册事件,使用位操作作为哈希算法提升了插入和查找的性能.

### 3. 关于 skynet 定时器的一些思考

- timer 线程在处理 Near 槽一个索引的事件队列时,如果同时有事件插入到这个槽里,会发生什么呢?

timer 线程每次取出 Near 槽里需要执行的事件队列时,都会抢占一个自旋锁([源码位置](https://github.com/cloudwu/skynet/blob/master/skynet-src/skynet_timer.c#L174)),而在开始消费时释放这个锁([源码位置](https://github.com/cloudwu/skynet/blob/master/skynet-src/skynet_timer.c#L165)).这个时候 worker 线程可以抢占这个自旋锁,并往 Near 槽里面注册事件.而在[这里](https://github.com/cloudwu/skynet/blob/master/skynet-src/skynet_timer.c#L177),即下一次的 tick 循环,timer 线程会再次抢占自旋锁(这时不可能有事件注册进来)并尝试再消费这个 Near 槽索引里面的事件,并把这个事件消费掉,解决了上面的问题.(猜想这就是[这句注释](https://github.com/cloudwu/skynet/blob/master/skynet-src/skynet_timer.c#L176)的意思了吧)

### 4. 其他定时器方案

- 时间轮

时间轮和 skynet 的 66668 位划分事件的方案其实很类似,不过一个是位操作划分,一个是时钟刻度划分,并将下一刻度(天/小时/分钟)的事件安排到下一刻度的时间轮里面.但是时间轮按照小时/分钟/秒或者更小的刻度来分割,性能不如位操作来的好.

- 以最小堆位为基础的定时器

以最小堆位为基础的定时器每次消费最小堆根节点的事件,而注册时等同于节点插入最小堆.这个方案的优点是 timer 线程可以准确的在下一个事件的时间点被唤醒,而无事件时,timer 线程是挂起的.与 skynet 的方案对比起来会节省一点 cpu,因为 skynet 在没有事件的时候 timer 线程还是会空转.而这个方案的坏处是插入/删除节点时的算法时间复杂度位 O(lgn),性能不如 skynet的O(1) 来的好.

### 5. linux 定时器方案

- skynet 的定时器方案和 linux 的方案很相似

看完 skynet 的定时器方案后,就好奇大 linux 的定时器是怎么实现的,翻了一些博文才知道 skynet 的方案和 linux 的方案很相似,数据结构和算法都是根据 6 6 6 6 8 位每个区间来放置和消费定时器事件,应该还有一些细微的差别,暂时没有仔细的对比.

### 6. 总结

- skynet 的定时器方案与 linux 底层的定时器方案很类似,根据触发时间放置事件到对应的槽里面,保证了插入/删除节点的性能(时间复杂度为 O(1))
- skynet 启动了一个timer线程去循环消费事件槽里面的事件
- skynet 的定时器方案和基于最小堆的定时器方案应该是比较好的两种定时器方案,可以根据具体场景使用不同的方案

参考资料:

- [linux网络编程二十三：高性能定时器之时间堆](https://blog.csdn.net/jasonliuvip/article/details/24738605)
- [游戏后台定时器系统设计与实现](https://www.jianshu.com/p/5a973f3ac409)
- [深入剖析Linux内核定时器实现机制](https://blog.csdn.net/tianmohust/article/details/8707162)
- [基于Hash和多级时间轮：实现定时器的高效数据结构](http://www.lpnote.com/2017/11/16/hashed-and-hierarchical-timing-wheels/)
