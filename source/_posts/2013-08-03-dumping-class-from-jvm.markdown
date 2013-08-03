---
layout: post
title: "从 JVM 中 dump class 的几种方法"
date: 2013-08-03 14:04
comments: true
categories: [Java, Programming, JVM]
---

前几天在 HotCode 的用户群里面，有同学问起“如何将 JVM 中的 class dump 出来”，当时我下意识的回答就是“可以在 JVM 启动的时候挂一个 agent 上去，然后通过 Instrumentation API 在 class 加载的时候做拦截，把类 dump 出来。”，今天无聊在翻 [R 大](http://weibo.com/rednaxelafx)的[博客](http://rednaxelafx.iteye.com/blog/727938)的时候，发现还可以通过 sa-jdi.jar 里面的一个类做 dump，这里就集中介绍一下这几个方法，然后介绍我在 sa-jdi.jar 基础上改的一个小工具。

### 采用 classLoader.getResourceAsStream()

将一个类从 JVM 中 dump 出来，最简单的方法当然就是直接从 jar 包中把对应的 class 文件找到，然后 dump 出来了，我们可以用 `classLoader` 的 `getResourceAsStream` 来做：

```java
ClassLoader loader = Thread.currentThread().getContextClassLoader();
InputStream in = loader.getResourceAsStream("com/khotyn/Test.class");
```

拿到 InputStream 后，你就可以随便玩了。

这个方法简单是简单，但是缺点也很明显，在有些 Java 程序中，类不一定是从 Class 文件中过来，有些是在运行时生成的，有些则在载入到 JVM 之前被增强过，所以这个方法有些类是 dump 不出来的，有些类则 dump 出来不是你想要的。

### 采用 javaagent

另外一个方法是通过在 JVM 启动的时候挂在一个 javaagent，然后用 Instrucmentation API 在类被加载到虚拟机之前做拦截，参考代码如下：

```java
package com.khotyn.test;

import java.io.File;
import java.io.IOException;
import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.lang.instrument.Instrumentation;
import java.security.ProtectionDomain;

import org.apache.commons.io.FileUtils;

/**
 * A demo to demonstrate how to use JVM ti to dump class file from JVM.
 * 
 * @author khotyn.huangt 13-8-3 PM2:21
 */
public class AgentMain {

    public static void premain(String agentArgs, Instrumentation inst) {
        inst.addTransformer(new ClassFileTransformer() {

            @Override
            public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined,
                                    ProtectionDomain protectionDomain, byte[] classfileBuffer)
                                                                                              throws IllegalClassFormatException {
                try {
                    FileUtils.writeByteArrayToFile(new File("/tmp/" + className.replace('.', '/') + ".class"),
                                                   classfileBuffer);
                } catch (IOException e) {
                    // Quite
                }
                return null;
            }
        });
    }
}
```

将上面的代码打成一个 jar 包（*注意把依赖的 apache commons.io 也打入，也可以直接下载我的 demo 工程：[agentDumpClass.zip](http://pan.baidu.com/share/link?shareid=778648109&uk=607430891)*），然后在 jar 包的 `META-INF/MANIFEST.MF` 中填上如下的内容：

```
Manifest-Version: 1.0
Boot-Class-Path: agentDump.jar
Built-By: apple
Build-Jdk: 1.7.0_17
Class-Path: lib/commons-io-2.4.jar
Premain-Class: com.khotyn.test.AgentMain
Created-By: Apache Maven
Can-Redefine-Classes: true
Archiver-Version: Plexus Archiver
```

然后你就可以执行类似于下面的命令来进行 dump 了：

```
java -javaagent:/Users/apple/workspace/agentDumpClass/target/agentDump.jar Test
```

由于 premain 方法是在 java 程序的 main 方法执行之前执行的，所以这个方法几乎可以拦截到所有的类，另外，由于注册的 ClassFileTransformer 是 ClassLoader 加载 class 之后，JVM 定义 class 之前被执行的，所以无论是在运行时生成的类，还是经过增强后的类，这个方法都能够 dump 出来，比第一种方法要强很多。然而这个方法还是有一些缺点：

* 需要在 JVM 启动时增加特别的参数。
* 只能随 class 被加载进行 dump，不能随时进行 dump

### 采用 sa-jdi.jar 的 ClassDump 工具

这个方式是 R 大在博客中介绍的方法，可以说是最强大的方法，不像前面的两个方法，这个方法可以在 JVM 进程外执行，且像 javaagent 的那个方法一样，都可以将运行时和被增强过的类 dump 出来，非常方便，至于具体的用法大家就直接看 [R 大的文章](http://rednaxelafx.iteye.com/blog/727938)吧。

### 改进 ClassDump

ClassDump 工具虽然强大，但是命令略显繁琐，特别是当你只需要 dump 特定的类的时候，还需要专门写一个 ClassFilter 的实现类，这么好的工具，应该直接做成命令行工具才好，于是我修改了 ClassDump 的代码，让它可以支持正则表达式的方式来对需要 dump 的类进行过滤，改进后的 ClassDump 放在了我的 github 仓库上：[https://github.com/khotyn/tools](https://github.com/khotyn/tools)

大家可以直接用下面的方式来使用这个改进版：

```
sudo classDump 17118 'com.khotyn.*' dump
```

classDump 这个命令的第一个参数是目标 JVM 的 PID，第二个参数是一个正则表达式，表示你所要 Dump 出来的类，第三个参数可选，是 dump 的目录。

但是这个工具有一个缺点，就是目前只能在 Mac 下用（因为我用 Mac，呵呵，我把修改后的类直接打入到了 Mac 的 jdk 的 sa-jdi.jar 下面），不过要做其他的平台的也很简单啦，只要按照以下步骤来打包出自己的 sa-jdi.jar 就可以：

* 下载我修改过的两个类：[ClassDump.class](http://pan.baidu.com/share/link?shareid=780474506&uk=607430891)，[RegexClassFilter.class](http://pan.baidu.com/share/link?shareid=782905423&uk=607430891)
* 从 jdk 目录下拷贝一份 sa-jdi.jar 出来
* 用下面的命令将修改过的两个类打到 sa-jdi.jar 中去：`jar uf sa-jdi.jar sun/jvm/hotspot/tools/jcore/ClassDump.class sun/jvm/hotspot/tools/jcore/RegexClassFilter.class`
* 然后配合仓库中的 classDump 脚本就可以用了。