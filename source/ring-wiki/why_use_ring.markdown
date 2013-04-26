---
layout: page
title: "Ring 文档 -- 为什么要使用 Ring？"
date: 2013-04-25 21:07
comments: true
sharing: false
footer: true
---

使用 Ring 作为你的 Web 应用的基础有一系列的好处：

* 可以使用 Clojure 的函数和 map 来编写你的应用
* 让你的应用跑在一个能够进行自动重新加载的服务器上
* 可以将你的应用编译成 Java Servlet
* 可以将你的应用打包成 Java war 文件
* 利用大量现有的中间件
* 在 [Amazon Elastic Beanstalk](http://aws.amazon.com/elasticbeanstalk/) 或者 [Heroku](http://heroku.com/) 这样的云环境中部署你的应用

Ring 事实上已经是采用 Clojure 编写 Web 应用的一个标准了。高层的框架，诸如 [Compojure](http://compojure.org/)，[Moustache](https://github.com/cgrand/moustache) 和 [Noir](http://www.webnoir.org/) 都是基于 Ring 的。

尽管 Ring 只提供了底层的 API，但是如果你打算使用更高层的接口的话，了解 Ring 是如何工作的还是很有好处的。如果对 Ring 没有基本的了解，你就无法编写中间件，并且你会发现对你的应用进行 Debug 会变得更加困难。