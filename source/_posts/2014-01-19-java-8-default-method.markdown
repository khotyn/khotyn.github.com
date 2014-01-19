---
layout: post
title: "Java 8 之 default method"
date: 2014-01-19 22:13
comments: true
categories: [Java, Programming]
---

如果进度正常，新版本的 Java，Java 8 将在三月份发布，Java 开发人员期待已久的 lambda 也将在 Java 8 中得到支持。目前，Java 8 的早期版本已经可以在 Java 的网站上下载到了，Intellij IDEA 也已经在其最新的版本支持了 Java 8。所以，最近花了点时间了解了一下 Java 8 中新增加的一些特性。

由于 lambda 的引入，Java 8 对原来的集合类做了大幅的更新，让集合操作可以支持 lambda 表达式。在看新的的集合类的代码的时候，发现了 java 8 似乎增加了一个新的方法描述符，比如在 `java.lang.Iterable` 里面就新加入了下面这个方法：

```
default void forEach(Consumer<? super T> action) {
    Objects.requireNonNull(action);
    for (T t : this) {
        action.accept(t);
    }
}
```

在方法的最前面，是一个 `default` 描述符。等等，Iterable 不是个接口吗，怎么有具体的实现代码了？

这个 default 就是在 java 8 中新引入的，它可以让你的接口有一个默认的实现，接口的实现类可以不用去实现 default method，比如，下面这段代码，是可以正常编译通过的：

```
class Impl implements A {

}

interface A {
    default String foo() {
        return "A";
    }
}
```

引入 default 的带来的一个好处就是在现有的接口上增加方法而不用让其实现修改代码，通过这种机制，Java 8 可以通过平滑的方式在原有的 Java 的 API 上引入 lambda 的支持。

那么，如果一个类实现了两个接口，这两个接口里面有方法签名相同的 default method，那运行的时候到底会选择哪一个？答案是编译不通过，如果出现这种情况，实现类必须实现 default method，以消除歧义，比如下面这样。

```
class MultiImpl implements A, B {

    /**
     * 由于 A，B 中都有 String foo() 接口，不知道要调用哪个，所以实现类必须实现一下
     *
     * @return
     */
    @Override
    public String foo() {
        return "C";
    }
}

interface A {
    default String foo() {
        return "A";
    }
}

interface B {
    default String foo() {
        return "B";
    }
}
```

当然，在的实现类中，也可以直接调用某个接口的 default method：

```
class MultiImplInvokeSuper implements A, B {
    @Override
    public String foo() {
        return B.super.foo();
    }
}
```
