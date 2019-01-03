---
layout: post
title: "jvmtop 介绍和实现分析"
date: 2013-11-02 15:36
comments: true
categories: [Programming, Java]
---

### 简介

jvmtop 是一个分析工具，顾名思义，它是一个针对 jvm 的 工具，展示的方式和 unix 的 top 命令相似。

jvmtop 的项目地址是：[jvmtop](https://code.google.com/p/jvmtop/)，安装 jvmtop 除了项目地址上的方式以外，还可以通过 jenv 安装：`jenv install jvmtop`。

jvmtop 提供了两个视图，一个是概览视图，可以展示出当前机器的所有的 JVM 的情况，命令是 

```
jvmtop.sh
```

显示出的信息类似下面这样：

![image](https://pic.yupoo.com/khotyn/DhvVumOY/medish.jpg)

其中，各个字段的意义分别如下：

* PID：进程 ID
* MAIN-CLASS：main 类的名字
* HPCUR：当前被使用的 heap 的大小
* HPMAX：最大可用的 heap 的大小
* NHCUR：当前被使用的非 heap 大小（比如：perm gen）
* NHMAX：最大可用的非 heap 大小
* CPU：CPU 的使用情况
* GC：消耗在 GC 上的时间比例
* VM：JVM 的提供者，大版本号，小版本号，图中的意思是 Apple 提供的 JDK 6U51 版本。
* USERNAME：当前的用户名
* \#T：线程数量
* DL：是否有现成发生死锁

还有一个视图是详情视图，展示一个 JVM 的详细情况，使用的命令如下：

```
jvmtop.sh <pid>
```

显示的信息类似下面这样：

![image](https://pic.yupoo.com/khotyn/Dhw0sotX/dTwsh.png)

其中，各个字段的意义如下：

* TID：线程 ID
* NAME：线程名
* STATE：线程状态
* CPU：线程当前的 CPU 占用情况
* TOTALCPU：从线程被创建开始总体的 CPU 占用情况
* BLOCKBY：阻塞这个线程的线程 ID

更加详细的用法大家可以用下面的用 `jvmtop.sh -h` 来查看。

### 实现

jvmtop 的实现相对来说还是比较简单的，整个 jvmtop 才 14 个类。

![image](https://pic.yupoo.com/khotyn/DhwgtqPY/IYHUw.png)

其中 JvmTop.java 是入口类。

jvmtop 在启动后，会首先用 `sun.jvmstat.monitor.*` 下面的类以及 `com.sun.tools.attach.VirtualMachine` 获取到当前机器的所有的 JVM，然后通过 attachment api 将 `management-agent.jar` 这个 agent 加载到目标 JVM 上，这样，通过 JMX，就可以拿到当前的 JVM 的各种信息了，具体各个信息需要用什么样的 MBean 去拿，大家可以看对应的源代码。

其实，如果需要一个 JVM 的静态的信息，比如，PID，MAIN-CLASS，JVM-ARGS 等等静态信息，直接用 `sun.jvmstat.monitor.*` 下的 API 就可以，只有需要动态信息的时候，我们才需要通过 attachment api 把 JMX 的功能打开，通过各种 MBean 去获取这些信息。如果后续需要实现类似的功能，也可以通过这样的思路去做。
