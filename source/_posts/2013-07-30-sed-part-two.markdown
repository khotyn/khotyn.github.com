---
layout: post
title: "「Sed & Awk」阅读笔记之 Sed 高级命令"
date: 2013-07-30 21:41
comments: true
categories: [Programming, Shell]
---

上一篇文章中，我介绍了一下 [sed 的基础](../../../../2013/07/28/sed-and-awk-part-one/)，包括执行方式、地址选择器以及基本命令，在这一篇文章中，我们继续来了解一下 sed 的高级命令，之所以称它们为高级命令，是因为这些命令会改变 sed 的执行流，废话不说，我们来看看这些命令吧：

### 高级命令

#### N (Next)

这里要介绍的第一个命令是 `N`，它和我们前面介绍过的 `n` 命令很像，也是要读取下一行的内容，不同的是，`N` 读取下一行的内容并且将这些内容附加到 pattern space 当前的内容后面。这样，当你需要连着处理多行内容的时候，`N` 命令就会特别有用，比如，我们有下面一段文本：

```
one two three four
one two three 
four three two
three four
```

如果我们要把 `two three four` 替换成 `2 3 4`，注意例子中的 `two three four` 可能在不同的行中，那么我们就可以用 `N` 命令来处理：


```
N
s/\n/ /
s/two three four/2 3 4/
```

输出的内容为：

```
one 2 3 4 one two three
four three 2 3 4
```

这个结果不是我们想要的，不过的确是符合了上面的 sed 脚本的执行结果：

1. 首先，sed 脚本读取了文本的第一行，这个时候 pattern space 中的内容为 `one two three four`
2. 然后 sed 脚本执行 N 命令，将下一行读取并附加到当前的 pattern space 内容的后面，这个时候，pattern space 中的内容就变为 `one two three four\none two three`
3. 下一个命令，将换行符 `\n` 替换成一个空格，pattern space 中的内容为 `one two three four one two three`
4. 然后下一个命令，将 pattern space 中的 `two three four` 替换成 `2 3 4`，这个时候 pattern space 中的内容为 `one 2 3 4 one two three`
5. 到达脚本的结尾，输出 pattern space 中的内容，也就是我们输出内容的第一行。
6. 然后 sed 脚本读取文本的下一行，注意因为之前第二行已经被 `N` 命令读取了，所以 sed 脚本开始读取第三行，依旧按照前面的命令执行，最后就输出了输出内容中的第二行。

虽然这个结果不是我们想要的，不过算是了解了 N 的作用了。

#### D (Delete)

同样，前面我们介绍过 `d` 命令，它用来删除 pattern space 中的内容，并且读取下一行到 pattern space 中，sed 脚本也随之从头开始执行。`D` 命令和 `d` 命令稍微有点不同，`D` 命令会删除 pattern space 中的第一行的内容，它不会从文本中读取新的行进来，当然 sed 脚本还是会从头开始执行，如果我们有这么一个文本：

```
blank
blank


blank

blank


blank






blank
```

这个文本中的有些段落之间有多个空行，我们希望把多余的空行去掉，也就是如果段落之间有多个空行，就删掉只剩下一个，我们的 sed 脚本可以这么写：

```
/^$/{
	N
	/^\n$/D
}

```

这个脚本先匹配出空行，然后读取空行的下一行，如果两行都是空行的话，就把 pattern space 中的第一个空行删除掉，然后继续读取下一行到 pattern space 中，结果就是把多余的空行都删除掉，只剩下一个空行了。

#### P (Print)

`P` 命令和 `p` 命令也稍微不同，`P` 命令不像 `p` 命令那样会把 pattern space 中的所有内容打印出来，它只会将 pattern space 的第一行打印出来，这里就不做过多的介绍了。

#### h (hold), H (Hold), g (get), G (Get), x (exchange)

这里面有五个命令，之所以一起介绍是因为，这五个命令都是操作 hold space 的，之前我们已经知道了 pattern space 了，hold space 可以认为就是一个内容的临时存放点，你可以将 pattern space 中的内容放到 hold space 中，等到需要使用的时候再将 hold space 中的内容拿回到 pattern space 中，我们来看一下这五个命令的作用吧：

* h：将 pattern space 中的内容拷贝到 hold space 中，hold space 中原来的内容会被覆盖。
* H：将 pattern space 中的内容添加到 hold space 当前内容的后面。
* g：将 hold space 中的内容拷贝到 pattern space 中，pattern space 中原来的内容将会被覆盖。
* G：将 hold space 中的内容添加到 pattern space 中目前的内容后面。
* x：交换 pattern space 和 hold space 中的内容。

下面我们来看一个简单的例子：

```
1
2
11
22
111
222
```

现在我们要将上面的 1 和 2 的位置调换，就是先出现 2 再出现 1，我们的脚本可以这么写：

```
/1/{
	h
	d
}
/2/{
	G
}
```

这段脚本先匹配 1 所在的行，然后放到 hold space 中，将 pattern space 中的内容清除掉，然后匹配到 2 所在的行，将 hold space 中的内容添加到 pattern space 后面，这样，pattern space 中就是先有 2，再有 1 了。

最后，我们得到的结果就是：

```
2
1
22
11
222
111
```

#### b

`b` 命令是一个跳转命令，它是无条件的，它的语法是这样的：

```
[address]b [label]
```

[label] 是要跳转到的标签，你可以在 sed 脚本中用 `:` 开头来表示一个标签，比如下面的：

```
:start
s/start/end/g
b start
```

如果 `b` 后面不带参数，那么就表示直接跳到脚本的末尾了。

#### t

除了 `b` 这样一个跳转命令以外，sed 还有一个 `t` 的条件跳转命令，如果在当前行有一个替换被成功执行了，那么 `t` 就会跳转到特定的标签上，它的语法 `b` 是类似的。

```
[address]t [label]
```

看下面这段代码：

```
:begin
s/start/end/
t begin
```

这条 `t` 命令只有在当前行的 start 成功被替换成 end 的时候才会跳转到 :begin 标签那里。

### 总结

sed 的高级命令相对于基本命令来说不怎么常用，但是在处理特定的问题的时候，这些命令还是很有用的。不过，不管怎么说，sed 都不是一门完备的语言，所以其适用的问题域也是比较有限的，sed 最大的优势在于逐行处理文本上，用适当的工具处理适当的问题，才能发挥出工具最大的威力。