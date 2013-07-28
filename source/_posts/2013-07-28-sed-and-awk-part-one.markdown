---
layout: post
title: "「Sed & awk」阅读笔记之 sed 基础"
date: 2013-07-28 16:39
comments: true
categories: [Programming, Shell]
---

之前写的一篇文章有提到采用 sed 来[匹配不包含连续字符串的行](../../../../blog/2013/07/24/match-line-not-contain-a-string/)，平时在做日志分析的时候也经常要用到 sed，但是仅仅用了 sed 的字符串替换的功能，没有系统地去学习过 sed 用法，这次找到一本叫[「sed & awk」](http://book.douban.com/subject/1741933/)的书，便花时间对 sed 做了系统的学习。

### sed 的执行方式

要了解 sed，必须了解 sed 的执行方式，sed 是一个行处理器，脱胎于 [ed](http://www.gnu.org/software/ed/manual/ed_manual.html)（ed 是一个行编辑器，awk 和 grep 也是基于 ed 的），简单地说，sed 的执行方式是这样的：**sed 会从输入的文本中读取一行，放到 pattern space 中，然后用 sed 脚本去处理，处理完后继续读取下一行，继续处理。**

假设我们有下面一段文本需要处理：

```
John Daggett, 341 King Road, Plymouth MA
Alice Ford, 22 East Broadway, Richmond VA
Orville Thomas, 11345 Oak Bridge Road, Tulsa OK
Terry Kalkas, 402 Lans Road, Beaver Falls PA
Eric Adams, 20 Post Road, Sudbury MA
Hubert Sims, 328A Brook Road, Roanoke VA
Amy Wilde, 334 Bayshore Pkwy, Mountain View CA
Sal Carpenter, 73 6th Street, Boston MA
```

我们要把文本中的 CA 替换成 California，OK 替换成 Oklahoma，于是我们写了下面一段 sed 脚本：

```
s/CA/California/
s/OK/Oklahoma/
```

那么 sed 的执行方式是这样的：

1. 先读取第一行 `John Daggett, 341 King Road, Plymouth MA` 到 pattern space
2. 然后执行脚本的第一行命令，将其中的 CA 替换成 California。
3. 然后执行脚本的第二行命令，将其中的 OK 替换成 Oklahoma。
4. 文本的第一行处理完毕，继续读取文本的下一行。
5. 继续第 2 步和第 3 步。

当然，这只是大部分情况下 sed 的执行方式，sed 的基本命令都是按照这种方式来执行的，一些高级命令可以改变 sed 的执行流。不过在了解这些 sed 命令之前，我们先了解下 sed 的地址选择器，它是很多命令的组成部分。

### sed 的地址选择器

默认的情况，sed 脚本会对文本的每一行做处理，但是有时候我们只希望我们的命令作用于特定的几行，这个时候，我们就可以用 sed 的地址选择器，sed 的地址选择器可以是一个正则表达式（sed 的正则表达式总是放在两个 `/` 中间），行号，或者地址符号（*这个是什么东西？我也不清楚*），具体的使用方式如下：

* 如果没有指定地址选择器，那么命令默认会应用在每一行上。
* 如果只有一个地址选择器，那么命令会作用在每一个符合这个地址选择器的行上。
* 如果是两个用 `,` 分割的地址，那么命令会先作用到第一个符合第一个地址选择器的行上，然后继续作用于后续的行，直到（包括）第一个符合第二个地址选择器的行为止。
* 地址选择器后面可以跟上一个 `!`，表示反向选择。

另外在一个地址选择器中，你可以用一对 `{}` 将多个命令包含在其中，下面我们来看一个例子：

```
/^\.TS/,/^\.TE/{
     /^$/d
     s/^\.ps 10/.ps 8/
     s/^\.vs 12/.vs 10/
}

```

这个 sed 脚本的第一行就是一个地址选择器，由 `,` 分开的两个地址选择器组成，都是正则表达式形式的，表示后面 `{}` 中的命令会从第一个以 `.TS` 开头的行一直作用到第一个以 `.TE` 开头的行为止。

### sed 的基本命令

了解完 sed 的地址选择器后，我们就可以继续了解 sed 的基本命令了。

#### 替换（substitution）

sed 的文本替换命令是我最常用的命令，它的语法是这样的：

```
[address]s/pattern/replacement/flags
```

它由这几个部分组成：

* 最前面是一个地址选择器，是可选的。
* 然后后面是一个命令 `s`，表示是替换命令。
* 后面紧跟一个正则表达式，表示要被替换的文本。
* 再后面是希望替换成的文本。
* 最后是标记位。

其中标记位可以是：

* n：一个从 1 到 512 的数字，表示只替换第 n 个符合 pattern 子串。
* g：默认情况下，替换命令只会替换一行中第一个符合 pattern 的子串，加上 `g` 以后会将行中所有符合 pattern 的子串都进行替换。
* p：将 pattern space 中的内容打印出来。
* w *file* ：将 pattern space 中的内容写到文件中。

举一个例子，假设我们要将第一个例子中的最后一行的 MA 换成 Massachusetts，就可以这样写：

```
$s/MA/Massachusetts/
```

其中的 `$` 是一个地址选择器，表示最后一行。

替换命令的替换文本基本上就是一个字符串，但是还是有一些特殊字符：`\`，`&` 和 `\n`，其中：

* `\`：转义，转义特殊字符。
* `&`：代表要被替换的文本，也就是符合 pattern 的子串。
* `\n`：当前面的正则表达式中有捕获部分的时候（即，正则表达式的 `()` 语法），可以在替换文本中用这种反向引用的方式进行引用。

这些特殊字符在当你需要将匹配的文本中的某些部分放到替换文本中的时候会特别有用。

#### 删除（delete）

删除命令很简单，它的语法是：

```
[address]d
```

前面可以带一个地址选择器，后面是一个 d，表示删除命令，举一个简单的例子，假设我们要筛选出不包含 `abc` 的行，可以这样写：

```
/abc/d
```

把包含 abc 的行全部都删除掉，这样就筛选出了不包含 `abc` 的行。

#### 追加，插入和变化（append，insert，change）

这三个命令的作用分别是：insert 将提供的文本插入到 pattern space 的当前行前面，append 将提供的文本追加到 pattern space 的当前行后面，change 命令替换 pattern space 中的内容，之所以将这三个命令放到一起说，是因为这三个命令需要将提供的文本放在命令的第二行，它们的语法分别是这样的：

append

    [line-address]a\
    text

insert
    [line-address]i\
    text

changae
    [address]c\
    text

来一个例子，我们有下面一段文本：

```
I want to see @f1(what will happen) if we put the
font change commands @f1(on a set of lines).
If I understand things (correctly), the @f1(third) line causes problems. (No?).
Is this really the case, or is it (maybe) just something else?
Let's test having two on a line @f1(here) and @f1(there) as
well as one that begins on one line and ends @f1(somewhere
on another line).
What if @f1(it is here) on the line?
Another @f1(one).
```

假设为了阅读的美观，我们希望段落之间能够空出一行来，段落结束的标记我们暂且简单地假设以 `.` 结尾，我们要做的就是在每一个以 `.` 结尾的行后面再插入一行，除了最后一行之外，那么我们的 sed 脚本就可以这么写：

```
$!{
	/\.$/a\
	\

}
```

来说明一下这段脚本，第一行是一个地址选择器，表示选择除最后一行以外的行，因为我们不希望在最后一个段落后面也加上一个空行，然后里面的命令是对所有的以 `.` 结尾的行运用 a 命令去追加一个空行。

#### 列出（list）

列出命令和打印命令其实很像，不同的是列出命令会将不可见字符给列出来，比如 windows 下的换行符，假设我们有下面一段文本（`^M` 可以用 Ctrl+V，然后 Ctrl+M 来输入）：

```
^M^M
^M
^M^M^M
```

上面一段文本用 `sed -n 'l'` 输出的内容是：

```
\r\r$
\r$
\r\r\r$
```

而用 `sed -n 'p'` 输出的内容是三个空空的行。

列出命令将不可见字符打印出来了，而打印命令则没有。

#### 转换（transform）

转换命令和替换命令听起来是一样，但是它们还是不同的，转换命令就像是多个 `tr` 命令用管道连在一起作用一样，它的语法是：

```
[address]y/abc/xyz/
```

转换命令的一个使用的场景就是大小写的转换：

```
y/abcdefghijklmnopqrstuvwxyz/ABCDEFGHIJKLMNOPQRSTUVWXYZ/
```

上面这个命令会将文本中所有的小写字母转换成大写字母。

#### 打印（print）

打印命令其实就是一个简单的 `p`，将 pattern space 中的内容打印出来，比如：

```
$!p
```

表示将除最后一行以外的内容全部打印出来。需要注意的是，默认情况下 sed 会将 pattern space 中的内容都打印出来，要关闭这个功能，可以加上一个 `-n` 参数，就像我在介绍列出命令的时候做的那样。

#### 打印行号

打印行号也就是一个简单的 `=` 号，大家可以去试一下，这里不再多讲了。

#### 下一个（next）

next 命令是一个 `n`，它的作用是将 pattern space 中的内容立即输出，然后将下一行读入到 pattern space 中，然后继续执行接下来的命令，比如：



    /H1/{
        n
    /^$/d
    }

就是先匹配到含有 H1 的行，然后将这一行打印出来，接着读取下一行到 pattern space，如果是空的话，就删除掉。

#### 读取和写入文件（read，write）

sed 的读取文件的功能可以将文件中的内容追加到 pattern space 后面，前面那个在段落后面添加空行的例子我们可以这么做：首先创建一个只包含一个空行的文件叫做 temp，接着我们就可以用下面的命令来达到我们的目的了：

```
$!{
    /\.$/r temp
}
```

sed 的写入文件的功能和读取文件的功能类似，语法是：

```
[address]w file
```

表示将 pattern space 中的内容写入到文件。

#### 退出（quit）

sed 的退出命令是让 sed 停止读取新的行，也停止输出，基本上就是让 sed 退出了，它的命令的语法是：

```
[line-address]q
```

它只能作用在单行的地址选择器上。

### 总结

sed 的基本命令相对来说还是比较简单的，最主要的还是要用好地址选择器，在下一篇中，我会介绍一些 sed 的高级命令。