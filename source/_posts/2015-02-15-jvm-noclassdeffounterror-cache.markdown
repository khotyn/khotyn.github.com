---
layout: post
title: "JVM 对 NoClassDefFoundError 的“缓存”"
date: 2015-02-15 20:44
comments: true
categories: [Java, JVM]
---

## 问题

今天在排查一个线上的问题，线上的一个应用在初始化一个类的静态字段的时候出现了 `NoClassDefFoundError`，并且在导致 `NoClassDefFoundError` 出现的根本原因消失后，后续再次尝试初始化这个类的时候，持续出现了 `NoClassDefFoundError`。

于是怀疑 JVM 是不是对一个类的 `NoClassDefFoundError` 做了缓存，在第一次加载这个类出现 `NoClassDefFoundError` 以后，后续再尝试加载就直接抛出 `NoClassDefFoundError`。


## 实验

为了证实自己的猜想，尝试设计了一个简单的实验，一个涉及三个类

```java
public class Test1 {
    static Test2 test2 = new Test2();
}
```

```java
public class Test2 {
}
```

```java
public class Test {
    public static void main(String... args) throws Exception {
        while(true) {
            System.out.println("================================");
            try {
                new Test1(); // 尝试实例化 Test1，触发 NoClassDefFoundError
            } catch (Throwable e) {
                e.printStackTrace();
                
                try {
                    Test.class.getClassLoader().loadClass("Test2"); // 尝试加载 Test2，用于证实当将 
                                                                    // Test2.class 拷贝到 ClassPath 下的时候，
                                                                    // Test2 就可以加载到了。
                } catch (Throwable ex) {
                    ex.printStackTrace();
                }
            }
            Thread.sleep(3000);
        }
    }
}
```

上述类的的作用是：Test2 是一个空的类，Test1 里面有一个 Test2 的静态成员。Test 是程序的主入口，在一个无限循环内部，不断地尝试去实例化 Test1，并且在加载 Test1 出现异常的时候，尝试加载一下 Test2。

实验的步骤是：

1. 编译以上类，运行 `javac Test.java`
2. 将生成出的 Test2.class 重命名成 Test2.class.bak
3. 运行 `java Test`，这个时候程序去加载 Test1 的时候，就会出现 `NoClassDefFoundError`，并且在尝试加载 Test2 的时候，会出现 `ClassNotFoundException`。
4. 将第二步重命名的 Test2.class.bak 该回成 Test2.class，这个时候程序去加载 Test1 的时候，就会出现 `NoClassDefFoundError`，在加载 Test2 的时候，不会出现 `ClassNotFoundException`。

实验的第二步的目的是为了程序在加载 Test1 的时候因为找不到 Test2 出现 `NoClassDefFoundError`，第四步是为了和第二步做对照，说明在后续程序可以加载到 Test2 的时候，在实例化 Test1 的时候，依旧出现 `NoClassDefFoundError`

在我的机器上，按照上面的方式去操作，结果如下：

![NoClassDefFoundError](/images/cnf.png)

结果正如预期，即使在后面 Test2 在 ClassPath 下的时候，`NoClassDefFoundError` 依旧出现，所以 JVM 里面肯定有地方对 `NoClassDefFoundError` 做了缓存。

## JVM 里面的实现

带着这个疑问，请教了部门里面的 JVM 专家，这个猜测得到了证实，并且他给出了 JVM 内部具体处理这段逻辑的代码，处理的代码在 JDK 的 `instanceKlass.cpp` 这个文件里面：

```java
bool instanceKlass::link_class_impl(
     instanceKlassHandle this_oop, bool throw_verifyerror, TRAPS) {
   // check for error state
   if (this_oop->is_in_error_state()) {
     ResourceMark rm(THREAD);
     THROW_MSG_(vmSymbols::java_lang_NoClassDefFoundError(),
                this_oop->external_name(), false);
   }
   // return if already verified
   if (this_oop->is_linked()) {
     return true;
   }
```

并且在 `instanceClass.hpp` 这个文件中，定义了类的 `_init_state`，其中，`is_in_error_state` 这个方法的定义如下：

```cpp
bool is_in_error_state() const           { return _init_state == initialization_error; }
```