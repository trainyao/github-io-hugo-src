---
author: "trainyao"
date: 2018-06-04 07:15:34
linktitle: skynet 概览
title: skynet 概览
weight: 10
---

这个星期以来，由于工作需要使用skynet，于是看了5天skynet的源码。收获很多，于是周末写篇博文总结一下这周以来的学习。

这5天的学习过程可以概括为5句话：（因为5天嘛）

1. 代码结构，从main函数开始
2. skynet启动过程，worker线程工作过程
3. skynet启动过程，skynet数据结构熟悉，
4. worker回调函数注册，worker的c层回调lua逻辑原理，
5. lua层逻辑协程的使用熟悉，服务之间通过消息交互的过程

看了5天后回过头来发现，其实了解的还停留在skynet的核心部分，还没有走出核心。因为skynet还包含很多其他内容，比如他的消息机制的各种api、socket支持、loginserver gateserver、datacenter、harbor等等。。。

不过问题也不大，毕竟核心比较重要，后续还要继续学习。

那下面就来总结一下这5天都看到来什么内容。有可能因为我了解尚浅，写出来的和实际的skynet有出入。不过没关系，后续会回过头来再看看。

下面的总结会按照这个结构：

- 源码目录结构
- skynet的结构图以及工作过程

那么，开始吧。

## 1. 源码目录结构

可以用一个段落来说明

``` shell
$ ll
drwxr-xr-x  26 root  staff     884  6  1 06:30 .
drwxr-xr-x   5 root  staff     170  6  2 07:07 ..
drwxr-xr-x  12 root  staff     408  6  1 07:10 .git
-rw-r--r--   1 root  staff      94  6  1 05:36 .gitignore
-rw-r--r--   1 root  staff      96  6  1 05:36 .gitmodules
drwxr-xr-x   8 root  staff     272  6  1 22:25 .idea
drwxr-xr-x   6 root  staff     204  6  1 05:36 3rd -- 第三方c库目录，如lua jmalloc
-rw-r--r--   1 root  staff   12384  6  1 05:58 CMakeLists.txt
-rw-r--r--   1 root  staff   11953  6  1 05:36 HISTORY.md
-rw-r--r--   1 root  staff    1085  6  1 05:36 LICENSE
-rw-r--r--   1 root  staff    3484  6  1 05:36 Makefile
-rw-r--r--   1 root  staff    1085  6  1 05:36 README.md
drwxr-xr-x   7 root  staff     238  6  1 05:58 cmake-build-debug
drwxr-xr-x  10 root  staff     340  6  1 05:39 cservice -- 可被skynet核心加载的so动态库服务
drwxr-xr-x  33 root  staff    1122  6  1 05:36 examples -- skynet提供的一些lua逻辑的例子
drwxr-xr-x  14 root  staff     476  6  1 05:39 luaclib -- 可被lua加载的skynet so库
drwxr-xr-x  12 root  staff     408  6  1 07:10 lualib -- lua skynet库
drwxr-xr-x  22 root  staff     748  6  1 05:55 lualib-src -- ./luaclib 里的so库的源文件
-rw-r--r--   1 root  staff     876  6  1 05:36 platform.mk
drwxr-xr-x  23 root  staff     782  6  1 06:57 service -- 可被skynet加载的lua服务
drwxr-xr-x   8 root  staff     272  6  1 05:36 service-src - ./cservice 里的so文件的源文件
-rwxr-xr-x   1 root  staff  343772  6  1 05:38 skynet -- skynet可执行文件
drwxr-xr-x  41 root  staff    1394  6  1 05:36 skynet-src -- ./skynet 的源文件
drwxr-xr-x   3 root  staff     102  6  1 05:38 skynet.dSYM
drwxr-xr-x  38 root  staff    1292  6  1 05:36 test -- 单元测试文件
```

从文件目录分析可以看出，skynet这个框架的内容既`包含了c语言的实现`（主要是c语言），也`包含了lua逻辑`（一些配置部分，或者可以被替代或者修改的逻辑，如服务加载器launcher.lua的逻辑, 或skynet启动的逻辑bootstrap.lua）。

将这两部分东西分离开的好处是一些与skynet核心无关的逻辑可以被动态的修改，或者skynet用户可以根据自己的需要去改动这部分逻辑，比较灵活。

其次,skynet除了可以加载`lua文件`作为服务(这主要是通过launcher服务实现的),也可以加载`so文件`作为服务(这种服务在skynet内核里面称为module, 如snlua服务和logger服务).由于这部分功能属于skynet内核的功能,使用时需要按照skynet内核定义的api来使用.

还有,skynet里还有一份lua源代码,是用来编译`可以被lua引用的so库`的源文件引用的,因为这部分用了lua的capi

## 2. skynet的结构图以及工作过程

![skynet结构图](../img/skynet/overview/skynet结构.png)

- actor模型

skynet是一个游戏基础设施框架. skynet使用了actor模型,引入了服务的概念. 在skynet中不同的服务就是不同的actor, 游戏中解耦的逻辑在skynet中都以服务的形式存在, 并通过消息进行通信,而skynet要做的,就是调度并处理这些信息. 由于服务之间需要频繁的进行消息通讯,skynet使用单进程多线程的模型,优点在于方便服务之间通讯,避免进程间通讯的性能损耗.

skynet启动时,会创建多个worker线程(数量可配置),并让这些线程运转起来.而这些线程的工作,就是消费全局一级消息队列里的消息.worker从消息队列里弹出一个指针,该指针指向的是一个服务里的二级消息队列.worker循环消费二级消息队列里面的消息,将它们分发到服务的lua层逻辑初始化时定义好的lua消息处理回调函数里,直到二级队列消息都消费完.当消息都消费完后,worker线程会进入睡眠状态,等待条件变量唤醒重新工作.

- 消息处理的逻辑,是用lua写的.
    
在skynet加载并启动服务时,都会注册一个处理消息的回调函数.由于这个处理的函数是用lua编写的,实际上skynet的worker线程在处理二级消息队列里面的消息时,会从c层回调lua层的这个信息处理回调函数.

- 不同服务的消息处理是异步的,同一个服务里面的消息处理也是异步的

worker在消费一级队列的时候,会争抢一个自旋锁来弹出二级消息队列的指针.所以实际上一个worker线程只会处理一个服务的消息,不同服务的消息处理异步进行.skynet用了lua的协程特性,一个消息到达lua层后,主协程会创建一个工作协程来处理.所以一个服务里的消息处理逻辑就算是阻塞的也不怕浪费cpu资源,lua会异步处理多个消息.

- skynet框架内封装了很多工具的API

skynet有自己的线程使用方式,在支持其他工具时(如mysql)需要重写协议的实现.因为这样做才能保持服务在使用工具时依然受skynet控制,并且保持高效.

作为一个基础设施框架,skynet封装了很多游戏中的逻辑开发需要用到的工具,比如广播,和数据中心.这些都还没细看具体是怎么实现的,后面会发文章单独看他们的实现.

