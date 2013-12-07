---
layout: post
title: "Google Guava 之 EventBus"
date: 2013-12-07 19:49
comments: true
categories: [Programming]
---

Google 的 Guava 库是一个 Java 程序员必须了解的库，它提供了一些非常强大的功能，比如函数式风格的集合操作，Cache Builder 等等的功能，另外 Google Guava 还提供了一个非常方便的观察者模式的实现：EventBus。这篇文章就来介绍一下 EventBus 的使用。

### EventBus 对象

在举例说明 EventBus 的使用方式之前，我们先来看一下 EventBus 对象，EventBus 对象整个负责了观察者模式监听者的注册，事件的分发，所以，在使用 EventBus 的时候，你就省去了非常多的工作，你只要去使用 EventBus 就可以了，不用再去自己实现一个 Publisher 的类，使用 EventBus 的第一步就是你需要一个 EventBus 的实例：

```java
EventBus eventBus = new EventBus();
```

### 注册监听者

使用 EventBus 监听事件，只需要在你的处理事件的方法上添加一个 `@Subscribe` 注解就可以：

```java
static class Subscriber {
    @Subscribe
    public void subscribe(Event event) {
        System.out.println(event.getWord());
    }
}
```

这里的事件对象 `Event` 可以是任何的对象，可以是 `Object`，但是也可以是任何你自定义的消息对象。

建立一个类以后，就可以往 EventBus 中注册 Subscriber：

```java
eventBus.register(new Subscriber());
```

### 分发事件

在注册完事件后，就可以去分发事件了，分发的代码非常简单：

```java
eventBus.post(new Event("Hello world"));
```

这样，所有的注册在 EventBus 中的监听者，只要它的监听方法的参数是 `Event` 或者 `Event` 的超类，那么都会收到事件。

### 结论

EventBus 作为一个 In-JVM 的观察者模式的实现，非常使用，使用起来非常简单，可以减少不少的工作，建议在项目中可以多多使用。