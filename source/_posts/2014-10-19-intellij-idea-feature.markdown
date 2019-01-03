---
layout: post
title: "Intellij IDEA 的一些使用技巧"
date: 2014-10-19 21:41
comments: true
categories: [Programming]
---

**所有的这些功能都是在 Intellij IDEA 14 中测试的，其他的版本不一定适用**

### 打开类的直接定位到某一行

在 Mac 下，IDEA 默认的打开类的快捷键是 `Command+O`，不过这个快捷键也有一些技巧。

第一个是可以在打开类的时候直接跳到某一行，比如下面这样：

![](https://pic.yupoo.com/khotyn/E8WysuL3/Pv6jh.png)

打开 String 这个类的同时直接跳转到 String 的第 40 行。

### 到某个类的某个方法

IDEA 的 Open Symbol 功能可以直接定位到某一个类的某一个方法，默认的快捷键是 `Option+Command+O`，如下：

![](https://pic.yupoo.com/khotyn/E8WRrUjB/I2Vom.png)

### 像 Sublime 那样多行编辑

以前要做多行编辑，总是现在 Sublime 里面先做好，然后再拷贝回到 IDEA 里面，现在知道了 IDEA 本身就自带这个功能，快捷键是 `Option+Shift+鼠标`，直接来看一个 gif 动画看来这个功能吧：

![](/images/select_multi_line.gif)

### Smart Code Completion

除了普通的代码补全的功能之外，IDEA 还提供了智能的不全功能，我们看下对比：

下面是基本的补全功能：

![](/images/basic_completion.gif)

这个是智能的补全功能：

![](/images/smart_completion.gif)

可以看到智能补全可以直接推断类型，把不符合类型的提示直接全部过滤，让我们可以更加高效地编写代码。

### 草稿

工作的时候我们经常会创建一些临时文件，在 IDEA 14 中，加入了一个非常有用的创建草稿的功能，Mac 下的快捷键是 `Command+Shift+N`，你可以在一个工程里面随意创建任意数量的草稿。

上面的这些是前几天参加 QCon 的时候听 IDEA 的一个 Session 知道的一些技巧。个人认为这个 Session 比很多其他的 Session 都更加有料