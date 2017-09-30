---
title: HashMap的线程安全问题
author: 汪博全
tags:
- Java
- 已完结
categories:
- 技术
- Java
date: 2017-03-08 22:33:00
---

> 连载中

<!-- more -->

# 为什么HashMap是线程不安全的

总说 HashMap 是线程不安全的，不安全的，不安全的，那么到底为什么它是线程不安全的呢？要回答这个问题就要先来简单了解一下 HashMap 源码中的使用的存储结构(这里引用的是 Java 8 的源码，与7是不一样的)和它的扩容机制。

<!-- more -->

## HashMap的内部存储结构

下面是 HashMap 使用的存储结构:

```java
transient Node<K,V>[] table;

static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
}
```

可以看到 HashMap 内部存储使用了一个 Node 数组(默认大小是16)，而 Node 类包含一个类型为 Node 的 next 的变量，也就是相当于一个链表，所有根据 hash 值计算的 bucket 一样的 key 会存储到同一个链表里(即产生了冲突)

## HashMap的自动扩容机制

HashMap 内部的 Node 数组默认的大小是16，假设有100万个元素，那么最好的情况下每个 hash 桶里都有62500个元素，这时get(),put(),remove()等方法效率都会降低。为了解决这个问题，HashMap 提供了自动扩容机制，当元素个数达到数组大小 loadFactor 后会扩大数组的大小，在默认情况下，数组大小为16，loadFactor 为0.75，也就是说当 HashMap 中的元素超过16\0.75=12时，会把数组大小扩展为2*16=32，并且重新计算每个元素在新数组中的位置。


## 为什么线程不安全

个人觉得 HashMap 在并发时可能出现的问题主要是两方面,首先如果多个线程同时使用put方法添加元素，而且假设正好存在两个 put 的 key 发生了碰撞(根据 hash 值计算的 bucket 一样)，那么根据 HashMap 的实现，这两个 key 会添加到数组的同一个位置，这样最终就会发生其中一个线程的 put 的数据被覆盖。第二就是如果多个线程同时检测到元素个数超过数组大小* loadFactor ，这样就会发生多个线程同时对 Node 数组进行扩容，都在重新计算元素位置以及复制数据，但是最终只有一个线程扩容后的数组会赋给 table，也就是说其他线程的都会丢失，并且各自线程 put 的数据也丢失。
关于 HashMap 线程不安全这一点，《Java并发编程的艺术》一书中是这样说的：

> HashMap 在并发执行 put 操作时会引起死循环，导致 CPU 利用率接近100%。因为多线程会导致 HashMap 的 Node 链表形成环形数据结构，一旦形成环形数据结构，Node 的 next 节点永远不为空，就会在获取 Node 时产生死循环。

# 如何线程安全的使用HashMap

了解了 HashMap 为什么线程不安全，那现在看看如何线程安全的使用 HashMap。这个无非就是以下三种方式：

* Hashtable
* ConcurrentHashMap
* Synchronized Map

```java
//Hashtable
Map<String, String> hashtable = new Hashtable<>();

//synchronizedMap
Map<String, String> synchronizedHashMap = Collections.synchronizedMap(new HashMap<String, String>());

//ConcurrentHashMap
Map<String, String> concurrentHashMap = new ConcurrentHashMap<>();
```

## Hashtable
HashTable 源码中是使用 synchronized 来保证线程安全的，比如下面的 get 方法和 put 方法：

```java
public synchronized V get(Object key) {
    // 省略实现
}
public synchronized V put(K key, V value) {
    // 省略实现
}
```

所以当一个线程访问 HashTable 的同步方法时，其他线程如果也要访问同步方法，会被阻塞住，效率很低，现在基本不会选择它了。

## ConcurrentHashMap
ConcurrentHashMap(简称CHM)是在Java 1.5作为Hashtable的替代选择新引入的，是concurrent包的重要成员。在Java 1.5之前，如果想要实现一个可以在多线程和并发的程序中安全使用的Map,只能在HashTable和synchronized Map中选择，因为HashMap并不是线程安全的。但再引入了CHM之后，我们有了更好的选择。CHM不但是线程安全的，而且比HashTable和synchronizedMap的性能要好。相对于HashTable和synchronizedMap锁住了整个Map，CHM只锁住部分Map。CHM允许并发的读操作，同时通过同步锁在写操作时保持数据完整性。

### Java中ConcurrentHashMap的实现

CHM引入了分割，并提供了HashTable支持的所有的功能。在CHM中，支持多线程对Map做读操作，并且不需要任何的blocking。这得益于CHM将Map分割成了不同的部分，在执行更新操作时只锁住一部分。根据默认的并发级别(concurrency level)，Map被分割成16个部分，并且由不同的锁控制。这意味着，同时最多可以有16个写线程操作Map。试想一下，由只能一个线程进入变成同时可由16个写线程同时进入(读线程几乎不受限制)，性能的提升是显而易见的。但由于一些更新操作，如put(),remove(),putAll(),clear()只锁住操作的部分，所以在检索操作不能保证返回的是最新的结果。

另一个重要点是在迭代遍历CHM时，keySet返回的iterator是弱一致和fail-safe的，可能不会返回某些最近的改变，并且在遍历过程中，如果已经遍历的数组上的内容变化了，不会抛出ConcurrentModificationExceptoin的异常。

CHM默认的并发级别是16，但可以在创建CHM时通过构造函数改变。毫无疑问，并发级别代表着并发执行更新操作的数目，所以如果只有很少的线程会更新Map，那么建议设置一个低的并发级别。另外，CHM还使用了ReentrantLock来对segments加锁。

### Java中ConcurrentHashMap putifAbsent方法的例子

很多时候我们希望在元素不存在时插入元素，我们一般会像下面那样写代码

```java
synchronized(map){
  if (map.get(key) == null){
      return map.put(key, value);
  } else{
      return map.get(key);
  }
}
```
上面这段代码在HashMap和HashTable中是好用的，但在CHM中是有出错的风险的。这是因为CHM在put操作时并没有对整个Map加锁，所以一个线程正在put(k,v)的时候，另一个线程调用get(k)会得到null，这就会造成一个线程put的值会被另一个线程put的值所覆盖。当然，你可以将代码封装到synchronized代码块中，这样虽然线程安全了，但会使你的代码变成了单线程。CHM提供的putIfAbsent(key,value)方法原子性的实现了同样的功能，同时避免了上面的线程竞争的风险。

### 什么时候使用ConcurrentHashMap

CHM适用于读者数量超过写者时，当写者数量大于等于读者时，CHM的性能是低于Hashtable和synchronized Map的。这是因为当锁住了整个Map时，读操作要等待对同一部分执行写操作的线程结束。CHM适用于做cache,在程序启动时初始化，之后可以被多个请求线程访问。正如Javadoc说明的那样，CHM是HashTable一个很好的替代，但要记住，CHM的比HashTable的同步性稍弱。

## SynchronizedMap
源码

```java
// synchronizedMap方法
public static <K,V> Map<K,V> synchronizedMap(Map<K,V> m) {
       return new SynchronizedMap<>(m);
}

// SynchronizedMap类
private static class SynchronizedMap<K,V> implements Map<K,V>, Serializable {
    private static final long serialVersionUID = 1978198479659022715L;

    private final Map<K,V> m;     // Backing Map
    final Object mutex;        // Object on which to synchronize

    SynchronizedMap(Map<K,V> m) {
        this.m = Objects.requireNonNull(m);
        mutex = this;
    }

    SynchronizedMap(Map<K,V> m, Object mutex) {
        this.m = m;
        this.mutex = mutex;
    }

    public int size() {
        synchronized (mutex) {return m.size();}
    }
    public boolean isEmpty() {
        synchronized (mutex) {return m.isEmpty();}
    }
    public boolean containsKey(Object key) {
        synchronized (mutex) {return m.containsKey(key);}
    }
    public boolean containsValue(Object value) {
        synchronized (mutex) {return m.containsValue(value);}
    }
    public V get(Object key) {
        synchronized (mutex) {return m.get(key);}
    }
    public V put(K key, V value) {
        synchronized (mutex) {return m.put(key, value);}
    }
    public V remove(Object key) {
        synchronized (mutex) {return m.remove(key);}
    }
    // 省略其他方法
}
```

从源码中可以看出调用 synchronizedMap() 方法后会返回一个 SynchronizedMap 类的对象，而在 SynchronizedMap 类中使用了 synchronized 同步关键字来保证对 Map 的操作是线程安全的。

## 性能对比
CHM 性能是明显优于 Hashtable 和 SynchronizedMap 的,CHM 花费的时间比前两个的一半还少

# 参考资料
[如何线程安全的使用 HashMap](http://yemengying.com/2016/05/07/threadsafe-hashmap/)
