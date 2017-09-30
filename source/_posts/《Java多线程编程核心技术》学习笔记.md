---
title: 《Java多线程编程核心技术》学习笔记
author: 汪博全
tags:
- Java
- 连载中
categories:
- 技术
- Java
date: 2017-03-01 21:33:00
---

> 连载中

<!-- more -->

# 第一章 Java多线程技能
## 1.1 进程和多线程的概念及线程的优点
见操作系统
## 1.2 使用多线程
方法1:继承Thread类，覆写run()方法
方法2:实现Runnable接口，覆写run()方法

Thread类实现了Runnable接口

CPU以不确定的方式调用线程中的run方法

线程安全：
概念见操作系统，在方法前加入synchronized关键字可对方法加锁，使这段代码变成“互斥区”，从而实现线程安全。
当一个线程想要执行同步方法里的代码时，线程首先尝试去拿这把锁，如果能够拿到这把锁，那么这个线程就可以执行synchronize里的代码。如果不能拿到这把锁，那么这个线程就会不断地尝试拿这把锁，直到能够拿到为止，而且是有多个线程同时去争抢这把锁。

## 1.3 currentThread()方法
该方法可返回代码段正在被哪个线程调用的信息。

## 1.4 isAlive()方法
方法isAlive()的作用是测试线程是否处于活动状态。
活动状态：线程已经启动且尚未终止的状态

例子：

```java
public class Run {

    public static void main(String[] args){
        CountOperate c = new CountOperate();
        c.start();
        Thread t1 = new Thread(c);
        System.out.println("main begin t1 isAlive=" + t1.isAlive());
        t1.setName("A");
        t1.start();
        System.out.println("main end t1 isAlive=" + t1.isAlive());
    }

    public class CountOperate extends Thread{

        public CountOperate(){
            System.out.println("CountOperate---begin");
            System.out.println("Thread.currentThread().getName()=" + Thread.currentThread().getName());//获取线程名
            System.out.println("Thread.currentThread().isAlive()=" + Thread.currentThread().isAlive()); //查看线程是否存活
            System.out.println("this.getName=" + this.getName());
            System.out.println("this.isAlive()=" + this.isAlive());
            System.out.println("CountOperate---end ");
            System.out.println("Thread.currentThread()==this :"+ (Thread.currentThread() == this));
        }

        @Override
        public void run() {
            System.out.println("run---begin");
            System.out.println("Thread.currentThread().getName=" + Thread.currentThread().getName());
            System.out.println("Thread.currentThread().isAlive()" + Thread.currentThread().isAlive());
            System.out.println("Thread.currentThread()==this :"+ (Thread.currentThread() == this));
            System.out.println("this.getName()=" + this.getName());
            System.out.println("this.isAlive()=" + this.isAlive());
            System.out.println("run --- end");
        }
    }
}
```

打印结果：

```
CountOperate---begin
Thread.currentThread().getName()=main
Thread.currentThread().isAlive()=true
this.getName=Thread-0
this.isAlive()=false
CountOperate---end
Thread.currentThread()==this :false
main begin t1 isAlive=false
main end t1 isAlive=true
run---begin
Thread.currentThread().getName=A
Thread.currentThread().isAlive()true
Thread.currentThread()==this :false
this.getName()=Thread-0
this.isAlive()=false
run --- end
```

根据打印结果可以知道调用CountOperate构造函数的是main线程，因此打印出
Thread.currentThread().getName()=main
Thread.currentThread().isAlive()=true

而此时还没有启动CountOperate子线程所以打印出
this.getName=Thread-0
this.isAlive()=false

此时this代表的是CountOperate对象实例，所以
Thread.currentThread()==this :false

这里比较让人疑惑的是“this.getName() = Thread-0”，这个Thread-0是什么东西？？？
通过查看Thread源码发现，在Thread类的构造方法中，会自动给name赋值，赋值代码：

```java
public Thread() {
    init(null, null，"Thread-" + nextThreadNum(), 0);
}
```

然后执行到:
Thread t1 = new Thread(c);
System.out.println("main begin t1 isAlive=" + t1.isAlive());
t1.setName("A");
t1.start();

打印：
Thread.currentThread().getName=A
Thread.currentThread().isAlive()true
Thread.currentThread()==this :false
this.getName()=Thread-0
this.isAlive()=false
说明此时的this和Thread.currentThread()指向不是同一个线程实例

也就是说，this指向的还是new CountOperate()创建的那个线程实例，而不是new Thread(thread)创建的那个实例即t1。
查看源代码可以知道。

```java
public Thread(Runnable target) {
    init(null, target，"Thread-" + nextThreadNum(), 0);
}
```

实际上new Thread(thread)会将thread应用的对象绑定到一个private变量target上，
在t1被执行的时候即t1.run()被调用的时候，它会调用target.run()方法，也就是说它是直接调用thread对象的run方法，
再确切的说，在run方法被执行的时候，this.getName()实际上返回的是target.getName()，而Thread.currentThread().getName()实际上是t1.getName()。

[参考](http://www.cnblogs.com/huangyichun/p/6071625.html)

## 1.5 sleep()方法
方法sleep()的作用是在指定的毫秒数内让当前“正在执行的线程”休眠（暂停执行），这个“正在执行的线程”是指Thread.currentThread()返回的线程。

## 1.6 getId()方法
取得线程的唯一标识

## 1.7 停止线程
三种方法
(1)使用退出标志，使线程正常退出，也就是当run方法完成后线程终止。
(2)使用stop方法强制终止线程，但是不推荐这种方法，因为stop和suspend及resume一样，都是作废过期的方法，使用它们可能产生不可预料的结果。
(3)使用interrupt方法中断线程。

interrupt()
调用interrupt()方法仅仅是在当前线程中打了一个停止的标记，并不是真的停止线程。

如何判断线程的状态是不是停止的
(1) this.interrupted()：测试当前线程是否已经中断（当前线程指运行this.interrupted()方法的线程），执行后具有将状态标志置清除为false的功能。
线程的中断状态由该方法清除，换句话说，如果连续两次调用该方法，则第二次调用将返回false（在第一次调用已清除了其中的中断状态之后，且第二次调用检验完中断状态前，当前线程再次中断的情况除外）
(2) this.isInterrupted()：测试线程是否已经中断，但不清除状态标志。

使用interrupt方法中断线程——异常法
eg：

```java
public class MyThread extends Thread {
    @Override
    public void run() {
        super.run();
        try {
            for (int i = 0; i < 50000; i++) {
                if (this.interrupted()) {
                    throw new InterruptedException();
                }
                // do something
            }
        } catch(InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

使用interrupt方法中断线程——return
eg：

```java
public class MyThread extends Thread {
    @Override
    public void run() {
       for (int i = 0; i < 50000; i++) {
           if (this.interrupted()) {
               return;
           }
           // do something
       }
    }
}
```

建议使用“异常法”来实现线程的停止，因为在catch块中可以对异常的信息进行相关的处理，而且使用异常流能更好、更方便地控制程序的运行流程，不至于代码中出现很多个return，造成污染。

使用stop()暴力停止线程可能会产生线程安全问题，不要使用。

## 1.8 暂停线程
在Java多线程中，可以使用suspend()方法暂停线程，使用resume()方法恢复线程的执行。
缺点：可能造成死锁和线程安全问题

## 1.9 yield方法
yield()方法的作用是放弃当前的CPU资源，将它让给其他的任务去占用CPU执行时间，但是放弃的时间不确定，有可能刚刚放弃，马上又获得时间片。

## 1.10 线程的优先级
设置线程的优先级使用setPriority()方法，在Java中，线程的优先级分为1 ~ 10这10个等级。
继承性：线程的优先级具有继承性，比如A线程启动B线程，则B线程的优先级与A是一样的
规则性：高优先级线程总是大部分先执行完，但不代表高优先级线程全部先执行完。
随机性：优先级高的线程不一定每一次都先执行完

## 1.11 守护线程
在Java线程中有两种线程：用户线程和守护(Daemon)线程
守护线程是一种特殊的线程，它的特性有陪伴的含义，当进程中不存在非守护线程了，则守护线程自动销毁。守护线程的作用是为其他线程的运行提供便利服务，最典型的应用是GC（垃圾回收器）

# 第二章 对象及变量的并发访问
## 2.1 synchronized同步方法
关键字synchronized取得的锁都是对象锁，而不是把一段代码或方法当做锁，哪个线程先执行带synchronized关键字的方法，哪个线程就持有该方法所属对象的锁Lock，那么其他线程只能呈等待状态，前提是多个线程访问的是同一个对象。但是如果多个线程访问多个对象，则JVM会创建多个锁。

调用用关键字synchronized声明的方法一定是排队运行的

synchronized锁重入
关键字synchronized拥有锁重入的功能，也就是在使用synchronized时，当一个线程得到一个对象锁后，再次请求次对象锁时是可以再次得到该对象的锁的。这也证明在一个synchronized方法/块的内部调用本类的其他synchronized方法/块时，是永远可以得到锁的。如果不可锁重入的话，就会造成死锁。

当一个线程执行的代码出现异常时，其所持有的锁会自动释放。

## 2.2 synchronized同步语句块
当两个并发线程访问同一个对象object中的synchronized(this)同步代码块时，一段时间内只能有一个线程被执行。

synchronized关键字可以锁任意对象 synchronized(非this对象x)

锁非this对象有一定的优点：如果在一个类中有很多个synchronized方法，这时虽然能实现同步，但会受到阻塞，所以影响运行效率；但如果使用同步代码块锁非this对象，则synchronized代码块中的程序与同步方法是异步的，不与其他锁this同步方法争抢this锁。

**静态同步synchronized方法与synchronized(class)代码块**
关键字synchronized还可以应用在static静态方法上，如果这样写，那是对当前的*.java文件对应的Class类进行持锁。同步synchronized(class)代码块的作用其实和synchronized static方法的作用一样。

数据类型String的常量池特性
在JVM中具有String常量池缓存的功能，下列代码的打印值为true

```java
String a = "a";
String b = "b";
System.out.println(a==b)
```

将synchronized(string)同步块与String联合使用时，要注意常量池带来的一些意外。

## 2.3 volatile关键字
**在特殊情况下的死循环问题**
有以下代码
```java
public class RunThread extends Thread {
    private boolean isRunning = true;
    public boolean isRunning() {
        return isRunning;
    }
    public void setRunning(boolean isRunning) {
        this.isRunning = isRunning;
    }
    @Override
    public void run() {
        System.out.println("进入run了");
        while(isRunning == true) {
        }
        System.out.println("线程被停止了");
    }
}

public class Run {
    public static void main(String[] args) {
        try {
            RunThread thread = new RunThread();
            thread.start();
            Thread.sleep(1000);
            thread.setRunning(false);
            System.out.println("已经赋值为false");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

执行后的结果
```
进入run了
线程被停止了
已经赋值为false
```

但是如果JVM设置为Server服务器环境中，代码会陷入死循环。

原因：
正常环境下变量private boolean isRunning = true;存在于公共堆栈及线程的私有堆栈中。在JVM被设置为-server模式时为了线程运行的效率，线程一直在私有堆栈中取得isRunning的值是true。而代码thread.setRunning(false);虽然被执行，更新的却是公共堆栈中的isRuning变量值false，所以一直处于死循环状态。

解决：
这个问题其实就是私有堆栈中的值和公共堆栈中的值不同步造成。解决这样的问题就要使用volatile关键字了，它的作用就是当线程访问isRunning这个变量时，强制从公共堆栈中进行取值。

**synchronized与volatile比较**
（1）关键字volatile是线程同步的轻量级实现，所以volatile性能肯定比synchronized要好，并且volatile只能修饰于变量，而synchronized可以修饰方法，以及代码块。随着JDK新版本的发布，synchronized关键字在执行效率上得到很大提升，在开发中使用synchronized关键字的比率还是比较大的
（2）多线程访问volatile不会发生阻塞，而synchronized会出现阻塞。
（3）volatile能保证数据的可见性，但是不能保证原子性；而synchronized可以保证原子性，也可间接保证可见性，因为它会将私有内存和公共内存中的数据做同步。

**原子类**
除了在i++操作中使用synchronized关键字实现同步外，还可以使用AtomicInteger原子类实现

# 第三章 线程间通信

**未完待续...**
