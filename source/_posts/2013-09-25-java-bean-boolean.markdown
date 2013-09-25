---
layout: post
title: "Java Bean 的布尔类型属性获取问题"
date: 2013-09-25 18:38
comments: true
categories: [Programming, Java]
---

### Velocity 对 Java Bean 中布尔类型的属性的获取问题

今天朋友遇到一个问题，是 Velocity 下面一个 `Boolean` 类型的变量在模板上没有办法输出，我大致简化一下这个问题，现在我们有一个简单的 Java Bean：

```java
public class SimpleBean {
    private Boolean hasKatong = false;

    public Boolean isHasKatong() {
        return hasKatong;
    }

    public void setHasKatong(Boolean hasKatong) {
        this.hasKatong = hasKatong;
    }
}
```

然后有一个简单的模板：

```java
$simpleBean.hasKatong
```

模板合并的代码如下：

```java
public static void main(String[] args) throws Exception {
    Properties p = new Properties();
    p.put("resource.loader", "class");
    p.put("class.resource.loader.class",
        "org.apache.velocity.runtime.resource.loader.ClasspathResourceLoader");
    Velocity.init(p);
    VelocityContext context = new VelocityContext();
    context.put("simpleBean", new SimpleBean());
    Template template = Velocity.getTemplate("mytemplate.vm");
    StringWriter sw = new StringWriter();
    template.merge(context, sw);
    System.out.println(sw.toString());
}
```

大家猜测一下这段代码的输出会是什么？可能大多数人都会认为是 `false`，但是在 Velocity 1.5 下，这段代码不会输出任何东西，反而会有一个 Warning：

>  INFO: Null reference [template 'mytemplate.vm', line 1, column 1] : $simpleBean.hasKatong cannot be resolved.

而在 Velocity 1.7 下，输出就是如大家所预测的那样，是 `false`。

具体的分析过程并不复杂，Velocity 1.5 和 1.7 在寻找 isXXXX 这样的方法的时候处理稍微有一点不一样，具体在 `BooleanPropertyExecutor` 这个类上，在找到方法，对方法的返回值的判断上有一点不一样，1.5 是这样的：

```java
if (isAlive()) {
    if (getMethod().getReturnType() != Boolean.TYPE) {
        setMethod(null);
    }
}
```

而 1.7 的是这样的：

```java
if (isAlive())
{
    if( getMethod().getReturnType() != Boolean.TYPE &&
        getMethod().getReturnType() != Boolean.class )
    {
        setMethod(null);
    }
}
```

可以看到，1.7 中增加了对返回值是 `Boolean` 的支持，而 1.5 只支持返回值是 `boolean` 的方法，那么既然知道了问题的根本原因，解决方法就显而易见了，要么将 `hasKatong` 这个属性的类型从 `Boolean` 改成 `boolean`，要么修改下 velocity 的模板，将属性获取直接改成方法调用：`$simpleBean.isHasKatong()`。

### Java Bean 规范对布尔类型属性的定义

当然，照理说像 velocity 这样的著名开源组件，不应该在这种问题上犯错误，然后我看了一下 Java Bean 的规范：
![image](http://pic.yupoo.com/khotyn/DbKXKmwa/medish.jpg)其实这段话已经说的很清楚了，只有原生类型的 `boolean` 的 Accessor 方法才能够用 **is** 前缀，其他的都用 get，其实在 JDK 的 Introspector 的实现中，也是这样处理的。
那么，这么看来，Velocity 1.5 的处理是正确的，那么 1.7 增加对 `Boolean` 的支持是为什么呢？其实，Java Bean 的规范在 `is` 这种 Accessor 的规定上，是有点不怎么符合开发人员的直觉的，很多人都会在这个问题上纠结：**`Boolean` 类型的属性的 Accessor 是不是应该用 is 开头？**，我觉得大部分人的直觉对这个答案的回答应该都是*是*，所以 Velocity 这样处理只不过是顺着大多数人的直觉的意思罢了，无可厚非。