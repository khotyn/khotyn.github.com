---
layout: post
title: "Java并发编程J.U.C之锁的获取与释放"
date: 2013-07-10 20:14
comments: true
categories: [Programming, JUC, Java]
---

**此篇文章是作者两年前发表在[黄金档](http://www.goldendoc.org/2011/06/lock_acquire_release/)的文章。**

上一篇文章中，我们对 [J.U.C 的一些大概的情况](http://www.goldendoc.org/2011/05/juc/)做了了解，在这一篇文章我们将来以 ReentrantLock 为例，来分析一下锁的获取和释放的过程，让大家能够对锁的获取和释放的整体过程有一个了解。

### 一、锁的获取

先看下 ReentrantLock 的 lock() 方法，整个方法只有一行，调用 acquire 方法，看看 acquire 方法的实现：

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

这段代码的实现也是比较简洁，先尝试一次 tryAcquire 操作，如果失败，则把当前线程加入到同步队列中去，这个时候可能会反复的阻塞与唤醒这个线程，直到后续的 tryAcquire（看 acquireQueued 的实现）操作成功。

再看看 tryAcquire 的实现：

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (isFirst(current) &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

这段代码是尝试获取锁的过程，它先判断当前的 AQS 的 state 值，如果为 0，则表示该锁没有被持有过，如果这个时候同步队列是空的或者当前线程就是在同步队列的头部，那么修改 state 的值，并且设置排他锁的持有线程为当前线程

如果大于 0，则判断当前线程是否是排他锁的持有线程，如果是，那么把 state 值加 1（注意 state 是 int 类型的，所以 state 的最大值是就是 int 的最大值）

如果第一次 tryAcquire() 操作失败，那么就把当前线程加入到等待队列中去，看 addWaiter() 方法：

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```

这段代码中先尝试了一下了下 enq() 方法中等待队列不为空的情况，如果失败，再调用 enq() 方法将当前线程加入等待队列，enq() 的过程我们已经在上一篇文章中讲过了，不再赘述。

最后在当前线程被加入到等待队列中去以后，再调用 acquireQueued 去获取锁，看看 acquireQueued 的代码：

```java
final boolean acquireQueued(final Node node, int arg) {
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } catch (RuntimeException ex) {
        cancelAcquire(node);
        throw ex;
    }
}
```

这段代码中拿到当前线程在同步队列中的前面一个节点，如果这个节点是是头部，那么马上进行一次 tryAcquire 操作，如果操作成功，那么把当前线程弹出队列，整个操作就此结束。如果这个节点不是头部或者说 tryAcquire 操作失败的话，那么就判断是不是要将当前线程给阻塞掉（shouldParkAfterFailedAcquire）方法：判断当前线程是否应该被阻塞掉，实际上判断的是当前线程的前一个节点的状态，如果前一个节点的状态小于 0（condition 或者 signal），那么返回 true，阻塞当前线程；如果前一个节点的状态大于 0（cancelled），则向前遍历，直到找到一个节点状态不大于 0 的节点，并且将中间的 cancelled 状态的节点全部踢出队列；如果前一个节点的状态等于 0，那么将其状态置为 -1（signal），并且返回 false，等待下一次循环的时候再阻塞。

**整个锁的获取过程就是这样，我们再来总结一下整个过程**：acquire() 方法会先调用一次 tryAcquire 方法获取一次锁，如果失败，则把当前线程加入到等待队列中去，然后再调用 acquireQueued 获取锁，acquireQueued 在当前节点不在头部的时候会把当前线程的前一个结点的状态置为 SIGNAL，然后阻塞当前线程。当当前线程到了队列的头部的时候，那么获取锁的操作就会成功返回。

### 二、锁的释放

首先，我们知道在 acquireQueued 方法中，如果一个线程成功获取到了锁，那么它就应该是整个等待队列的 head 节点，然后，我们再来看一看 unlock() 方法，和 lock() 方法一样，unlock() 方法也是只有一行代码，直接调用 release() 方法，我们看看 release() 方法的实现：

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

这个过程首先调用 tryRelease 方法，如果锁已经完全释放，那么就唤醒下一个节点，先来看看 tryRelease 方法：

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

这段代码首先获取当前 AQS 的 state 状态并且将其值减一，如果结果等于 0（锁已经被完全释放），那么将排他锁的持有线程置为 null。将 AQS 的 state 状态置为减一后的结果。

然后再看看唤醒继任节点的代码：

```
private void unparkSuccessor(Node node) {
    compareAndSetWaitStatus(node, Node.SIGNAL, 0);
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

这段代码先清除当前节点的 waitStatus 为 0，然后判断下一个节点是不是 null 或者 cancelled 的状态，如果是，则从队列的尾部往前开始找，找到一个非 cancelled 状态的节点，最后唤醒这个节点。

**最后，总结一下释放操作的整个过程**：其实整个释放过程就做了两件事情，一个是将state值减1，然后就是判断锁是否被完全释放，如果被完全释放，则唤醒继任节点。

### 三、整体过程描述

看了上面的锁的获取与释放操作以后，整体过程还是比较清晰的，在文章的最后，我们把获取与释放操作串在一起在简单看一下：

* 获取锁的时候将当前线程放入同步队列，并且将前一个节点的状态置为 signal 状态，然后阻塞
* 当这个节点的前一个节点成功获取到锁，前一个节点就成了整个同步队列的 head。
* 当前一个节点释放锁的时候，它就唤醒当前线程的这个节点，然后当前线程的节点就可以成功获取到锁了
* 这个时候它就到整个队列的头部了，然后 release 操作的时候又可以唤醒下一个。