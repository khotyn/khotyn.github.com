---
layout: post
title: "Guice 集合注入"
date: 2014-08-16 18:47
comments: true
categories: [Programming]
---

Guice 的初学者在使用 Guice 往一个类中注入一个集合注入的时候，肯定有感觉到非常地不自然（这里的不自然我觉得一定程度上是不符合 Guice 给人的初印象），由于最近在项目中也在使用 Guice，所以在这里对 Guice 的集合注入做一个记录。

#### 一、使用 Guice 的扩展 guice-multibindings

Guice 的文档上关于 Guice 注入的最简单的例子应该就是：

```java
bind(Interface.java).to(Implementation.java);
```

我们希望在使用 Guice 做集合注入的时候肯定也是希望使用类似的 API 做注入，不过可惜的是，Guice 的核心里面并没有提供类似的 API 让我们可以使用来注入集合。

所幸的是，Guice 提供了一个扩展的包 `guice-multibindings` 使用和 Guice 最原始的 API 类似的方式来做注入。

需要使用这个扩展的包，使用 Maven 的话，可以在项目中加入如下的依赖：

```
<dependency>
    <groupId>com.google.inject.extensions</groupId>
    <artifactId>guice-multibindings</artifactId>
    <version>3.0</version>
</dependency>
```

`guice-multibindings` 主要使用了两种方式来注入，一种是注入一个 Set：

```java
Multibinder<CheckHandler> checkAdapter = Multibinder.newSetBinder(binder(), CheckHandler.class);
checkAdapter.addBinding().to(InstalledCheckHandler.class);
```

首先创建一个 `checkAdapter`，然后往这个 Multibinder 中，我们可以添加任意多的 `CheckHandler` 的实现。

另一种方式是注入一个 Map：

```java
MapBinder<String, CheckHandler> mapBinder = MapBinder.newMapBinder(binder(), String.class, CheckHandler.class);
mapBinder.addBinding("Hello").to(InstalledCheckHandler.class);
```

Map 的注入方式和 Set 的注入方式非常类似。不过奇怪的一点是，Guice 并没有提供注入 List 的方法，**这是值得思考的一点**。

#### 二、使用 `@Provides` 来注入

看了第一种方法，我们可以看到，上面的这种方法并不能注入一个 List，不过，我们还是有办法来注入一个 List 的，就是使用一个 `@Provides` 注解，比如在我们的 Guice Module 的类里面加入一下的代码：

```java
@Provides
List<BindingAdapter> provideBindingAdapter() {
    List<BindingAdapter> bindingAdapters = new ArrayList<BindingAdapter>();
    List<OsgiServiceHolder<BindingAdapter>> bindingAdapterHolders = OsgiFrameworkUtils
        .getServices(bundleContext, BindingAdapter.class);

    for (OsgiServiceHolder<BindingAdapter> bindingAdapterHolder : bindingAdapterHolders) {
        final BindingAdapter bindingAdapter = bindingAdapterHolder.getService();

        if (bindingAdapter != null) {
            bindingAdapters.add(bindingAdapter);
        }
    }

    return bindingAdapters;

}
```

这样，我们就可以从其他的地方拿到对应的类的实例，然后放到一个 List 中，通过 Guice 注入给其他的类了。

上面的这两种方式其实各有优劣。一般情况下，我觉得选择第一种就可以了，毕竟，第一种方法的类的实例是由 Guice 来生成的。选择第二种方式的场景我觉得可能有：

* 类的实例是从其他的地方来的，比如上面的例子中，是从 OSGi 来的。
* 简单类型的类，比如一个 String 的 List。