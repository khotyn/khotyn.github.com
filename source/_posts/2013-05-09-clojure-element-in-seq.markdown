---
layout: post
title: "Clojure 如何判断一个序列中是否存在某个元素"
date: 2013-05-09 21:17
comments: true
categories: [Clojure]
---

最近一直在看 Clojure，经常碰到的一个问题是怎么判断一个序列中是否存在某个元素。对于这个问题的第一反应就是用`contains?`来判断，但是`contains?`的第二个参数是`key`而不是元素的值，对于 vector 或者 array 这样的数据结构不能做判断：

```bash
user=> (contains? '(101 102 103 104) 101)
false
```

另一个方法就是采用 set 配合 some 来做判断：

```bash
user=> (some #{101} '(101 102 103 104))
101
```

`some` 的函数原型是 `(some prev coll)`，它会对 coll 中的元素依次应用 prev 进行测试，返回第一个为真的元素，而用一个 set 进行测试，返回就是第一个包含在这个 set 中的元素，因此上面的那段表达式就返回了 101。

这样，只要上面这个表达式返回的不是`nil`，就表示元素在序列中了。