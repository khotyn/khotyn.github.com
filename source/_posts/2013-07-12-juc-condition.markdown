---
layout: post
title: "Java 并发编程 J.U.C 之 Condition"
date: 2013-07-12 07:36
comments: true
categories: [Java, Programming, JUC]
---

**此篇文章是作者两年前发布在[黄金档](http://www.goldendoc.org/2011/06/juc_condition/)的文章。**

在上一篇中，我们了解了下 [J.U.C 的锁的获取与释放的过程](http://blog.khotyn.com/blog/2013/07/10/juc-lock-acquire-release/)，这个过程主要通过在 A.Q.S 中维持一个等待队列来实现，其中我们也提到了，在 A.Q.S 中除了一个等待队列之外，还有一个 Condition 队列，在了解 Condition 队列之前，先来看一下 Condition 是怎么回事：

> The synchronizer framework provides a ConditionObject class for use by synchronizers that maintain exclusive synchronization and conform to the Lock interface. Any number of condition objects may be attached to a lock object, providing classic monitor-style await, signal, and signalAll operations, including those with timeouts, along with some inspection and monitoring methods.

上面的这一段内容摘自 Doug Lea 的 [AQS 论文](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)，从上面这一段话可以看出，Condition 主要是为了在 J.U.C 框架中提供和 Java 传统的监视器风格的 wait，notify 和 notifyAll 方法类似的功能，那么先来解释一下这三个方法的作用：


* Object.wait() 方法：使当前线程释放 Object 上的监视器并且挂起，直到有另外的线程调用 Object.notify() 方法或者 Object.notifyAll() 方法唤醒当前线程，当被唤醒后，Object.wait() 方法会尝试重新获取监视器，成功获取后继续往下执行。注意 Object.wait() 方法只有在当前线程持有 Object 的监视器的时候才能够调用，不然会抛出异常。
* Object.notify() 方法：用于唤醒另外一个调用了 Object.wait() 方法的线程，如果有多个都调用了 Object.wait() 方法，那么就会选择一个线程去 notify()，具体选择哪一个和具体的实现有关，当前线程在调用 Object.notify() 方法以后会就释放Object的监视器，和 wait() 方法一样，Object.notify() 方法只有在当前线程持有 Object 的监视器的时候才能够调用，不然就会抛出异常。
* Object.notifyAll() 方法：唤醒所有调用了 Object.wait() 方法的线程，如果有多个线程调用了 Object.wait() 方法，那么就会引发这些线程之间的竞争，最后谁成功获取到 Object 的监视器和具体的实现有关，当前线程在调用 Object.notifyAll() 方法以后会就释放 Object 的监视器，和 wait() 方法一样，Object.notifyAll() 方法只有在当前线程只有 Object 的监视器的时候才能够调用，不然就会抛出异常。

那么 Condition 是如何实现 wait，notify 和 notifyAll 方法的功能呢？我们接下来看：

在 Condition 中，wait，notify 和 notifyAll 方法分别对应了 await，signal 和 signalAll 方法，当然 Condition 也提供了超时的、不可被中断的 await() 方法，不过我们主要还是看一看 await，notify 和 notifyAll 的实现，先看 await：

### await 方法

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

整个 await 的过程如下：

* 在第 2 行处，如果当前线程被中断，则抛出中断异常。
* 在第 4 行处，将节点加入到 Condition 队列中去，这里如果 lastWaiter 是 cancel 状态，那么会把它踢出 Condition 队列。
* 在第 5 行处，调用 tryRelease，释放当前线程的锁
* 在第 7 行处，判断节点是否在等待队列中（signal 操作会将 Node 从 Condition 队列中拿出并且放入到等待队列中去），如果不在等待队列中了，就 park 当前线程，如果在，就退出循环，这个时候如果被中断，那么就退出循环
* 在第 12 行处，这个时候线程已经被 signal() 或者 signalAll() 操作给唤醒了，退出了 4 中的 while 循环，尝试再次获取锁，调用 acquireQueued 方法。

可以看到，这个 await 的操作过程和 Object.wait() 方法是一样，只不过 await() 采用了 Condition 队列的方式实现了 Object.wait() 的功能。

### signal 和 signalAll 方法

在了解了 await 方法的实现以后，signal 和 signalAll 方法的实现就相对简单了，先看看 signal 方法：

```java
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```

这里先判断当前线程是否持有锁，如果没有持有，则抛出异常，然后判断整个 condition 队列是否为空，不为空则调用 doSignal 方法来唤醒线程，看看 doSignal 方法都干了一些什么：

```java
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```

这个 while 循环的作用就是将 firstWaiter 往 Condition 队列的后面移一位，并且唤醒 first，看看 while 循环中 tranferForSignal：

```java
final boolean transferForSignal(Node node) {
    /*
     * If cannot change waitStatus, the node has been cancelled.
     */
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
 
    /*
     * Splice onto queue and try to set waitStatus of predecessor to
     * indicate that thread is (probably) waiting. If cancelled or
     * attempt to set waitStatus fails, wake up to resync (in which
     * case the waitStatus can be transiently and harmlessly wrong).
     */
    Node p = enq(node);
    int c = p.waitStatus;
    if (c > 0 || !compareAndSetWaitStatus(p, c, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

这段代码的作用就是修改 Node 的 waitStatus 为 0，然后将 Node 插入到等待队列中，并且唤醒 Node。

signalAll 和 signal 方法类似，主要的不同在于它不是调用 doSignal 方法，而是调用 doSignalAll 方法：

```java
private void doSignalAll(Node first) {
    lastWaiter = firstWaiter  = null;
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        transferForSignal(first);
        first = next;
    } while (first != null);
}
```

这个方法就相当于把 Condition 队列中的所有 Node 全部取出插入到等待队列中去。

### 总结

在了解了 await，signal 和 signalAll 方法的实现以后，我们再来通过一副 gif 动画来看一看这一个整体的过程：

![image](http://farm3.staticflickr.com/2888/9263654699_6b959eecb2_o.gif)