---
layout: post
title: "Java 并发编程之 ConcurrentHashMap"
date: 2013-07-12 23:02
comments: true
categories: [Java, Programming, JUC]
---

**此篇文章是作者两年前发布在[黄金档](http://www.goldendoc.org/2011/06/juc_concurrenthashmap/)的文章。**

ConcurrentHashMap 是一个线程安全的 Hash Table，它的主要功能是提供了一组和 HashTable 功能相同但是线程安全的方法。ConcurrentHashMap 可以做到读取数据不加锁，并且其内部的结构可以让其在进行写操作的时候能够将锁的粒度保持地尽量地小，不用对整个 ConcurrentHashMap 加锁。

### ConcurrentHashMap 的内部结构

ConcurrentHashMap 为了提高本身的并发能力，在内部采用了一个叫做 Segment 的结构，一个 Segment 其实就是一个类 Hash Table 的结构，Segment 内部维护了一个链表数组，我们用下面这一幅图来看下 ConcurrentHashMap 的内部结构：

![image](http://pic.yupoo.com/goldendoc/Ba4GCFe1/nuEZ0.png)

从上面的结构我们可以了解到，ConcurrentHashMap 定位一个元素的过程需要进行两次 Hash 操作，第一次 Hash 定位到 Segment，第二次 Hash 定位到元素所在的链表的头部，因此，这一种结构的带来的副作用是 Hash 的过程要比普通的 HashMap 要长，但是带来的好处是写操作的时候可以只对元素所在的 Segment 进行加锁即可，不会影响到其他的 Segment，这样，在最理想的情况下，ConcurrentHashMap 可以最高同时支持 Segment 数量大小的写操作（刚好这些写操作都非常平均地分布在所有的 Segment 上），所以，通过这一种结构，ConcurrentHashMap 的并发能力可以大大的提高。

### Segment

我们再来具体了解一下Segment的数据结构：

```java
static final class Segment<K,V> extends ReentrantLock implements Serializable {
    transient volatile int count;
    transient int modCount;
    transient int threshold;
    transient volatile HashEntry<K,V>[] table;
    final float loadFactor;
}
```

详细解释一下 Segment 里面的成员变量的意义：

* count：Segment 中元素的数量
* modCount：对 table 的大小造成影响的操作的数量（比如 put 或者 remove 操作）
* threshold：阈值，Segment 里面元素的数量超过这个值依旧就会对 Segment 进行扩容
* table：链表数组，数组中的每一个元素代表了一个链表的头部
* loadFactor：负载因子，用于确定 threshold

### ConcurrentHashMap 的初始化

下面我们来结合源代码来具体分析一下 ConcurrentHashMap 的实现，先看下初始化方法：

```java
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
 
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
 
    // Find power-of-two sizes best matching arguments
    int sshift = 0;
    int ssize = 1;
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    segmentShift = 32 - sshift;
    segmentMask = ssize - 1;
    this.segments = Segment.newArray(ssize);
 
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    int cap = 1;
    while (cap < c)
        cap <<= 1;
 
    for (int i = 0; i < this.segments.length; ++i)
        this.segments[i] = new Segment<K,V>(cap, loadFactor);
}
```

CurrentHashMap 的初始化一共有三个参数，一个 initialCapacity，表示初始的容量，一个 loadFactor，表示负载参数，最后一个是 concurrentLevel，代表 ConcurrentHashMap 内部的 Segment 的数量，ConcurrentLevel 一经指定，不可改变，后续如果 ConcurrentHashMap 的元素数量增加导致 ConrruentHashMap 需要扩容，ConcurrentHashMap 不会增加 Segment 的数量，而只会增加 Segment 中链表数组的容量大小，这样的好处是扩容过程不需要对整个 ConcurrentHashMap 做 rehash，而只需要对 Segment 里面的元素做一次 rehash 就可以了。

整个 ConcurrentHashMap 的初始化方法还是非常简单的，先是根据 concurrentLevel 来 new 出 Segment，这里 Segment 的数量是不小于 concurrentLevel 的最大的 2 的指数，就是说 Segment 的数量永远是 2 的指数个，这样的好处是方便采用移位操作来进行 hash，加快 hash 的过程。接下来就是根据 intialCapacity 确定 Segment 的容量的大小，每一个 Segment 的容量大小也是 2 的指数，同样是为了加快 hash 的过程。

这边需要特别注意一下两个变量，分别是 segmentShift 和 segmentMask，这两个变量在后面将会起到很大的作用，假设构造函数确定了 Segment 的数量是 2 的 n 次方，那么 segmentShift 就等于 32 减去 n，而 segmentMask 就等于 2 的 n 次方减一。

### ConcurrentHashMap 的 get 操作

前面提到过 ConcurrentHashMap 的 get 操作是不用加锁的，我们这里看一下其实现：

```java
public V get(Object key) {
    int hash = hash(key.hashCode());
    return segmentFor(hash).get(key, hash);
}
```

看第三行，segmentFor 这个函数用于确定操作应该在哪一个 segment 中进行，几乎对 ConcurrentHashMap 的所有操作都需要用到这个函数，我们看下这个函数的实现：

```java
final Segment<K,V> segmentFor(int hash) {
    return segments[(hash >>> segmentShift) & segmentMask];
}
```

这个函数用了位操作来确定 Segment，根据传入的 hash 值向右无符号右移 segmentShift 位，然后和 segmentMask 进行与操作，结合我们之前说的 segmentShift 和 segmentMask 的值，就可以得出以下结论：假设 Segment 的数量是 2 的n次方，根据元素的 hash 值的高 n 位就可以确定元素到底在哪一个 Segment 中。

在确定了需要在哪一个 segment 中进行操作以后，接下来的事情就是调用对应的 Segment 的 get 方法：

```java
V get(Object key, int hash) {
    if (count != 0) { // read-volatile
        HashEntry<K,V> e = getFirst(hash);
        while (e != null) {
            if (e.hash == hash && key.equals(e.key)) {
                V v = e.value;
                if (v != null)
                    return v;
                return readValueUnderLock(e); // recheck
            }
            e = e.next;
        }
    }
    return null;
}
```

先看第二行代码，这里对 count 进行了一次判断，其中 count 表示 Segment 中元素的数量，我们可以来看一下 count 的定义：

```java
transient volatile int count;
```

可以看到 count 是 volatile 的，实际上这里里面利用了 volatile 的语义：

> 对volatile字段的写入操作happens-before于每一个后续的同一个字段的读操作。

因为实际上 put、remove 等操作也会更新 count 的值，所以当竞争发生的时候， volatile 的语义可以保证写操作在读操作之前，也就保证了写操作对后续的读操作都是可见的，这样后面 get 的后续操作就可以拿到完整的元素内容。

然后，在第三行，调用了 getFirst() 来取得链表的头部：

```java
HashEntry<K,V> getFirst(int hash) {
    HashEntry<K,V>[] tab = table;
    return tab[hash & (tab.length - 1)];
}
```

同样，这里也是用位操作来确定链表的头部，hash 值和 HashTable 的长度减一做与操作，最后的结果就是 hash 值的低 n 位，其中 n 是 HashTable 的长度以 2 为底的结果。

在确定了链表的头部以后，就可以对整个链表进行遍历，看第 4 行，取出 key 对应的 value 的值，如果拿出的 value 的值是 null，则可能这个 key，value 对正在 put 的过程中，如果出现这种情况，那么就加锁来保证取出的 value 是完整的，如果不是 null，则直接返回 value。

### ConcurrentHashMap 的 put 操作

看完了 get 操作，再看下 put 操作，put 操作的前面也是确定 Segment 的过程，这里不再赘述，直接看关键的 segment 的 put 方法：

```java
V put(K key, int hash, V value, boolean onlyIfAbsent) {
    lock();
    try {
        int c = count;
        if (c++ > threshold) // ensure capacity
            rehash();
        HashEntry<K,V>[] tab = table;
        int index = hash & (tab.length - 1);
        HashEntry<K,V> first = tab[index];
        HashEntry<K,V> e = first;
        while (e != null && (e.hash != hash || !key.equals(e.key)))
            e = e.next;
 
        V oldValue;
        if (e != null) {
            oldValue = e.value;
            if (!onlyIfAbsent)
                e.value = value;
        }
        else {
            oldValue = null;
            ++modCount;
            tab[index] = new HashEntry<K,V>(key, hash, first, value);
            count = c; // write-volatile
        }
        return oldValue;
    } finally {
        unlock();
    }
}
```

首先对 Segment 的 put 操作是加锁完成的，然后在第五行，如果 Segment 中元素的数量超过了阈值（由构造函数中的 loadFactor 算出）这需要进行对 Segment 扩容，并且要进行 rehash，关于 rehash 的过程大家可以自己去了解，这里不详细讲了。

第 8 和第 9 行的操作就是 getFirst 的过程，确定链表头部的位置。

第 11 行这里的这个 while 循环是在链表中寻找和要 put 的元素相同 key 的元素，如果找到，就直接更新更新 key 的 value，如果没有找到，则进入 21 行这里，生成一个新的 HashEntry 并且把它加到整个 Segment 的头部，然后再更新 count 的值。

### ConcurrentHashMap 的 remove 操作

Remove 操作的前面一部分和前面的 get 和 put 操作一样，都是定位 Segment 的过程，然后再调用 Segment 的 remove 方法：

```java
V remove(Object key, int hash, Object value) {
    lock();
    try {
        int c = count - 1;
        HashEntry<K,V>[] tab = table;
        int index = hash & (tab.length - 1);
        HashEntry<K,V> first = tab[index];
        HashEntry<K,V> e = first;
        while (e != null && (e.hash != hash || !key.equals(e.key)))
            e = e.next;
 
        V oldValue = null;
        if (e != null) {
            V v = e.value;
            if (value == null || value.equals(v)) {
                oldValue = v;
                // All entries following removed node can stay
                // in list, but all preceding ones need to be
                // cloned.
                ++modCount;
                HashEntry<K,V> newFirst = e.next;
                for (HashEntry<K,V> p = first; p != e; p = p.next)
                    newFirst = new HashEntry<K,V>(p.key, p.hash,
                                                  newFirst, p.value);
                tab[index] = newFirst;
                count = c; // write-volatile
            }
        }
        return oldValue;
    } finally {
        unlock();
    }
}
```

首先 remove 操作也是确定需要删除的元素的位置，不过这里删除元素的方法不是简单地把待删除元素的前面的一个元素的 next 指向后面一个就完事了，我们之前已经说过 HashEntry 中的 next 是 final 的，一经赋值以后就不可修改，在定位到待删除元素的位置以后，程序就将待删除元素前面的那一些元素全部复制一遍，然后再一个一个重新接到链表上去，看一下下面这一幅图来了解这个过程：

![image](http://pic.yupoo.com/goldendoc/Ba3OfBv8/medish.jpg)

假设链表中原来的元素如上图所示，现在要删除元素 3，那么删除元素 3 以后的链表就如下图所示：

![image](http://pic.yupoo.com/goldendoc/Ba3OfPQE/medish.jpg)

### ConcurrentHashMap 的 size 操作

在前面的章节中，我们涉及到的操作都是在单个 Segment 中进行的，但是 ConcurrentHashMap 有一些操作是在多个 Segment 中进行，比如 size 操作，ConcurrentHashMap 的 size 操作也采用了一种比较巧的方式，来尽量避免对所有的 Segment 都加锁。

前面我们提到了一个 Segment 中的有一个 modCount 变量，代表的是对 Segment 中元素的数量造成影响的操作的次数，这个值只增不减，size 操作就是遍历了两次 Segment，每次记录 Segment 的 modCount 值，然后将两次的 modCount 进行比较，如果相同，则表示期间没有发生过写入操作，就将原先遍历的结果返回，如果不相同，则把这个过程再重复做一次，如果再不相同，则就需要将所有的 Segment 都锁住，然后一个一个遍历了，具体的实现大家可以看 ConcurrentHashMap 的源码，这里就不贴了。