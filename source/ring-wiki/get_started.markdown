---
layout: page
title: "Ring 文档 -- 开始"
date: 2013-04-25 21:07
comments: true
sharing: false
footer: true
---

用 Leiningen 创建一个新项目：

```bash
$ lein new hello-world
$ cd hello-world
```

在 `project.clj` 中添加 ring-core 和 ring-jetty-adapter 依赖。

```clojure
(defproject hello-world "1.0.0-SNAPSHOT"
  :description "FIXME: write"
  :dependencies [[org.clojure/clojure "1.4.0"]
                 [ring/ring-core "1.1.6"]
                 [ring/ring-jetty-adapter "1.1.6"]])
```

接下来，编辑 `src/hello_world/core.clj`，添加一个基本的 handler。

```clojure
(ns hello-world.core)

(defn handler [request]
  {:status 200
   :headers {"Content-Type" "text/html"}
   :body "Hello World"})
```

现在，我们可以将这个 handler 和一个 adapter 连接在一起了。使用 Leiningen 开启一个 REPL：

```bash
$ lein repl
```

然后在 REPL 中，使用你的 handler 来运行 Jetty adapter。

```bash
=> (use 'ring.adapter.jetty)
=> (use 'hello-world.core)
=> (run-jetty handler {:port 3000})
```

这样，一个 Web 服务器就在 [http://localhost:3000/](http://localhost:3000/) 跑起来了。