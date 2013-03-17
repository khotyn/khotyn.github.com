---
layout: post
title: "Clojure 学习笔记：一"
date: 2013-03-17 20:44
comments: true
categories: [Notes, Clojure]
---

### 一、 为什么学 Clojure

Clojure 一直是我想去学习的一门语言，从去年开始就想学，但是我一直忍着没学，没学的原因一方面是想看一看自己是不是三分钟热度，过了一段时间就不再对它感兴趣了。另一方面，我更想去多学一些计算机底层的技术，因为我自认为基础并不太好，未来会成为个人发展的一个障碍。

但是，从现在看来，我似乎对 Clojure 并不是脑子一热就想去学，而有一些更深层的原因。我想学 Clojure 一个主要的原因是因为最近一段时间一直在思考语言的抽象维度，表达能力的问题，对这个问题我虽然有一些感觉，但是因为了解的语言非常有限，所以一直都没有抓到问题的本质，找不到问题的答案是什么，而多了解一门不同类型的语言或许可以在一定程度上帮助我思考这个问题，如果这门语言和 Java 越不同，能给我带来的思维的转换就越大，显然 Lisp 会是很好的选择。而 Clojure 正是基于 JVM 的一种 Lisp 方言，对做 Java 开发的我来说再好不过。

### 二、Clojure 是一门什么样的语言

面对这个问题，我的脑子中一下子塞入了各种名词：

- 不变量
- 函数式编程
- S 表达式 
- 一堆括号
- 宏

现在对 Clojure 的认识还很模糊，这个问题我希望能够一直带在整个学习过程中，希望在经过一段时间的学习以后可以回答这个问题。

### 三、环境安装

在 Mac 下可以直接通过 brew 来安装：

```
brew install clojure
```

Clojure 官网也提供了其他的安装方式：[http://clojure.org/getting_started](http://clojure.org/getting_started)

编辑器我选择是 Intellij IDEA，安装上一个 La Clojure 的插件即可，这个插件提供了代码高亮、格式化等功能，并且针对 Clojure 括号略多的情况，给括号提供了彩虹色的高亮，非常贴心。另外 La Clojure 内置的 REPL 在语法高亮、光标的移动方面都比 Clojure 自带的要好。

### 四、学习资料

关于 Clojure 的书比较著名的有三本：

- [The Joy of Clojure](http://book.douban.com/subject/4743790/)
- [Programming Clojure](http://book.douban.com/subject/7915128/)
- [Clojure Programming](http://book.douban.com/subject/6715378/)

目前这三本书中 The Joy of Clojure 正在翻译中，Programming Clojure 和 Clojure Programming 已经翻译好，可以在各大网站上预订了，想看中文版的朋友可以考虑去买纸质书。

我选择了 Clojure Programming 这本书，没什么原因，刚好之前下载了这本书的英文版，看着看着就一直再看了。

### 五、下一篇

这第一篇笔记没有什么关于 Clojure 的具体内容，主要作用在于提醒自己学习要不忘初衷，下一篇中我会开始整理出一些具体内容出来，敬请期待。