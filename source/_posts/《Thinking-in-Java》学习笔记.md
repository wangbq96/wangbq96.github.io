---
title: 《Thinking in Java》学习笔记
author: 汪博全
tags:
- Java
- 连载中
categories:
- 技术
- Java
date: 2017-3-13 14:41:00
---

> 连载中

<!-- more -->

# 1 字符串

String对象是不可变的
String类中每一个看起来会修改String值的方法，实际上都是创建了一个全新的String对象

String的"+"操作是通过StringBuilder对象做为中间对象实现的

如果重写类的toString()方法中使用了循环，最好自己创建一个StringBuilder对象

Java SE5之前使用StringBuffer，是线程安全的

**toString()递归调用**：
有以下代码

```java
@Override
public String toString() {
    return " xxx " + this + "\n";
}

```

执行toString方法时会发生异常，因为编译器会尝试把this转换为String，即调用类的toString()方法，从而发生了递归调用

格式化输出printf() 类java

# 2 容器类
JAVA的容器---List,Map,Set

```
Collection
|-List
| |-LinkedList
| |-ArrayList
| |-Vector
|　 |-Stack
|-Set
```

```
Map
|-Hashtable
|-HashMap
|-WeakHashMap
```

1）Collection：一个独立元素的序列，这些元素都服从一条或者多条规则。List必须按照插入的顺序保存元素，而set不能有重复的元素。Queue按照排队规则来确定对象产生的顺序（通常与它们被插入的顺序相同）。
2）Map：一组成对的“键值对”对象，允许你使用键来查找值。

注：
1、java.util.Collection 是一个集合接口。它提供了对集合对象进行基本操作的通用接口方法。Collection接口在Java 类库中有很多具体的实现。Collection接口的意义是为各种具体的集合提供了最大化的统一操作方式。
2、java.util.Collections 是一个包装类。它包含有各种有关集合操作的静态多态方法。此类不能实例化，就像一个工具类，服务于Java的Collection框架。
　　
## 2.1 List接口

List是有序的Collection，使用此接口能够精确的控制每个元素插入的位置。用户能够使用索引（元素在List中的位置，类似于数组下标）来访问List中的元素，这类似于Java的数组。

实现List接口的常用类有LinkedList，ArrayList，Vector和Stack。

### 2.1.1 LinkedList类
LinkedList实现了List接口，允许null元素。此外LinkedList提供额外的get，remove，insert方法在 LinkedList的首部或尾部。这些操作使LinkedList可被用作堆栈（stack），队列（queue）或双向队列（deque）。

注意:LinkedList没有同步方法。如果多个线程同时访问一个List，则必须自己实现访问同步。一种解决方法是在创建List时构造一个同步的List：

```java
List list = Collections.synchronizedList(new LinkedList(…));
```

### 2.1.2 ArrayList类
ArrayList实现了可变大小的数组。它允许所有元素，包括null。size，isEmpty，get，set方法运行时间为常数。但是add方法开销为分摊的常数，添加n个元素需要O(n)的时间。其他的方法运行时间为线性。每个ArrayList实例都有一个容量（Capacity），即用于存储元素的数组的大小。这个容量可随着不断添加新元素而自动增加，但是增长算法并没有定义。当需要插入大量元素时，在插入前可以调用ensureCapacity方法来增加ArrayList的容量以提高插入效率。

和LinkedList一样，ArrayList也是非同步的（unsynchronized）。一般情况下使用这两个就可以了，因为非同步，所以效率比较高。
如果涉及到堆栈，队列等操作，应该考虑用List，对于需要快速插入，删除元素，应该使用LinkedList，如果需要快速随机访问元素，应该使用ArrayList。

### 2.1.3 Vector类
Vector非常类似ArrayList，但是Vector是同步的。由Vector创建的Iterator，虽然和ArrayList创建的 Iterator是同一接口，但是，因为Vector是同步的，当一个 Iterator被创建而且正在被使用，另一个线程改变了Vector的状态（例 如，添加或删除了一些元素），这时调用Iterator的方法时将抛出 ConcurrentModificationException，因此必须捕获该 异常。

### 2.1.4 Stack 类
Stack继承自Vector，实现一个后进先出的堆栈。Stack提供5个额外的方法使得Vector得以被当作堆栈使用。基本的push和pop方 法，还有 peek方法得到栈顶的元素，empty方法测试堆栈是否为空，search方法检测一个元素在堆栈中的位置。Stack刚创建后是空栈。

### 2.1.5 ArrayList与LinkedList

1.ArrayList是实现了基于动态数组的数据结构，LinkedList基于链表的数据结构。
2.对于随机访问get和set，ArrayList优于LinkedList，因为LinkedList要移动指针。
3.对于新增和删除操作add和remove，LinkedList比较占优势，因为ArrayList要移动数据。

## 2.2 Set接口

Set是一种不包含重复的元素的Collection，即任意的两个元素e1和e2都有e1.equals(e2)=false，Set最多有一个null元素。 Set的构造函数有一个约束条件，传入的Collection参数不能包含重复的元素。
Set容器类主要有HashSet和TreeSet等。
　　
### 2.2.1 HashSet类
Java.util.HashSet类实现了Java.util.Set接口。

* 它不允许出现重复元素；
* 不保证集合中元素的顺序
* 允许包含值为null的元素，但最多只能有一个null元素。

### 2.2.2 TreeSet
TreeSet描述的是Set的一种变体：可以实现排序等功能的集合，它在讲对象元素添加到集合中时会自动按照某种比较规则将其插入到有序的对象序列中，并保证该集合元素组成的对象序列时刻按照“升序”排列。


## 2.3 Map集合接口
Map没有继承Collection接口，Map提供key到value的映射。一个Map中不能包含相同的key，每个key只能映射一个value。Map接口提供3种集合的视图，Map的内容可以被当作一组key集合，一组value集合，或者一组key-value映射。

### 2.3.1 Hashtable类
Hashtable继承Map接口，实现一个key-value映射的哈希表。任何非空（non-null）的对象都可作为key或者value。**添加数据使用put(key, value)，取出数据使用get(key)，这两个基本操作的时间开销为常数。**Hashtable通过initial capacity和load factor两个参数调整性能。通常缺省的load factor 0.75较好地实现了时间和空间的均衡。增大load factor可以节省空间但相应的查找时间将增大，这会影响像get和put这样的操作。

由于作为key的对象将通过计算其散列函数来确定与之对应的value的位置，因此任何作为key的对象都必须实现hashCode和equals方法。hashCode和equals方法继承自根类Object，如果你用自定义的类当作key的话，要相当小心，按照散列函数的定义，如果两个对象相同，即obj1.equals(obj2)=true，则它们的hashCode必须相同，但如果两个对象不同，则它们的hashCode不一定不同，如果两个不同对象的hashCode相同，这种现象称为冲突，冲突会导致操作哈希表的时间开销增大，所以尽量定义好的hashCode()方法，能加快哈希表的操作。

如果相同的对象有不同的hashCode，对哈希表的操作会出现意想不到的结果（期待的get方法返回null），要避免这种问题，只需要牢记一条：要同时复写equals方法和hashCode方法，而不要只写其中一个。

### 2.3.2 HashMap类
HashMap和Hashtable类似，不同之处在于HashMap是非同步的，并且允许null，即null value和null key，但是将HashMap视为Collection时（values()方法可返回Collection），其迭代子操作时间开销和HashMap的容量成比例。因此，如果迭代操作的性能相当重要的话，不要将HashMap的初始化容量设得过高，或者load factor过低。　　

－JDK1.0 引入了第一个关联的集合类HashTable，它是线程安全的。 HashTable的所有方法都是同步的。
－JDK2.0 引入了HashMap，它提供了一个不同步的基类和一个同步的包装器synchronizedMap。synchronizedMap被称为 有条件的线程安全类。
－JDK5.0 java.util.concurrent包中引入对Map线程安全的实现ConcurrentHashMap，比起synchronizedMap，它提供了更高的灵活性。同时进行的读和写操作都可以并发地

### 2.3.3 WeakHashMap类
WeakHashMap是一种改进的HashMap，它对key实行“弱引用”，如果一个key不再被外部所引用，那么该key可以被GC回收。

### 2.3.4 HashTable和HashMap

* 继承不同。 <br>
public class Hashtable extends Dictionary implements Map <br>
public class HashMap extends AbstractMap implements Map <br>
* Hashtable 中的方法是同步的，而HashMap中的方法在缺省情况下是非同步的。在多线程并发的环境下可以直接使用Hashtable，但是要使用HashMap的话就要自己增加同步处理了。
* Hashtable中，key和value都不允许出现null值。在HashMap中，null可以作为键，这样的键只有一个；可以有一个或多个键所对应的值为null。当get()方法返回null值时，即可以表示 HashMap中没有该键，也可以表示该键所对应的值为null。因此，在HashMap中不能由get()方法来判断HashMap中是否存在某个键， 而应该用containsKey()方法来判断。
* 两个遍历方式的内部实现上不同。Hashtable、HashMap都使用了 Iterator。而由于历史原因，Hashtable还使用了Enumeration的方式 。
* 哈希值的使用不同，HashTable直接使用对象的hashCode。而HashMap重新计算hash值。
* Hashtable和HashMap它们两个内部实现方式的数组的初始大小和扩容的方式。HashTable中hash数组默认大小是11，增加的方式是 old*2+1。HashMap中hash数组的默认大小是16，而且一定是2的指数。
