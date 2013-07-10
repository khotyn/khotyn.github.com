---
layout: post
title: "Bash Job Control"
date: 2013-07-10 19:43
comments: true
categories: [Programming, Bash]
---

这两天在给 [HotCode](https://github.com/khotyn/hotcode) 写[测试脚本](https://github.com/khotyn/hotcode/blob/master/test.sh)，其中用到了 Bash 的任务控制功能，说白了就是把一个任务放到后台，拉到前台之类的操作，这里做一下总结。

### 开启任务控制

首先，如果要在 Bash 脚本中使用 Bash 的任务控制功能，必须先执行下面这个命令：

```bash
set -m
```

这个命令的意思其实就是开启 Bash 的任务控制功能，见 Bash Manual 的 [The Set Buildin](https://www.gnu.org/software/bash/manual/html_node/The-Set-Builtin.html)。

### 把任务放到后台执行

要把任务放到后台非常容易，只要在命令后面加入一个 `&` 即可，像这样：

```bash
bash test.sh &
```

命令的输出类似于这样：

```
[1] 51668
```

其中前面的 `[1]` 代表的是这个任务的任务编号，而 `51668` 则是这个任务的进程 ID。


### 查看在后台执行的任务的执行情况

`jobs` 命令可以查看在后台执行的任务的情况

它的输出类似于下面这样：

```
[1]  + running    bash test.sh
```

前面的 `[1]` 就是任务编号，后面的 `running` 表示任务当前运行的状态。

`jobs` 命令后面可以加一些参数，比如下面这些：

```bash
jobs -r ###只输出正在运行的任务
jobs -s ###只输出已经执行完的任务
```

具体看 [Job Control Buildins](https://www.gnu.org/software/bash/manual/html_node/Job-Control-Builtins.html#Job-Control-Builtins)。

### 把任务调度到前台

要把任务调度到前台只要用 `fg` 命令就可以，后面加上一个任务编号表示要把哪个后台任务调度到前台：

```
fg %1
```

上面这个命令就是把任务编号为 1 的任务调度到前台。

Bash 的任务调度还是比较简单的，不过的确是非常地有用。

