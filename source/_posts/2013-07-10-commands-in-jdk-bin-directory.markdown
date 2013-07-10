---
layout: post
title: "JDK bin 目录下的常见命令"
date: 2013-07-10 21:05
comments: true
categories: [Java, Programming]
published: false
---

作为一个 Java 开发，每天都需要和 JDK 打交道，而每一个 Java 开发都应该熟悉 JDK bin 目录下提供的一些命令。

### java 和 javac

在 JDK bin 目录下众多的命令中，最常用的当然就是 java 和 javac 了，java 命令无需多言，就是启动一个 Java 虚拟机，而 javac 命令当然就是编译 java 文件了，这里面多说一句，如果你在用 javac 编译一个 java 文件的时候用 jps（后面会提到）查看当前的 java 虚拟机进程，你会看到这样的内容：

```
52639 com.sun.tools.javac.Main -Dapplication.home=/Library/Java/JavaVirtualMachines/jdk1.7.0_17.jdk/Contents/Home -Xms8m
```

看到这行输出相比大家已经意识到了，其实 javac 也是用 java 实现的，是不是有一点自举的味道？哈哈。

### jps

除了 java 和 javac，我平时用到最多的命令就是 jps，这个命令顾名思义，就是 ps 的 java 版本，列出 java 虚拟机的进程，单单执行 jps 这个命令，输出非常简单

```
52427 RemoteMavenServer
52408
52761 Jps
```

前面是 java 虚拟机的进程号，后面是 java 程序 main 函数所在类的类名（眼尖的朋友可能已经注意到了，这个输出里面有一个 Jps，是的，没错，jps 也是用 Java 实现的）。很多时候，这么简单的输出可能不能满足我们的需求，你可以加上特定的参数来获得更多的信息：

* -l：输出全限定类名。
* -v：输出虚拟机的启动参数。
* -q：只输出进程 ID，不输出其他内容。

我平时使用 jps 都会加上 -l 和 -v（`jps -lv`），这样输出的信息基本上就能满足日常的需求了，知道了虚拟机的入口类是什么，也知道启动虚拟机的参数，够了。