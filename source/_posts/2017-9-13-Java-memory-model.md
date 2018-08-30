---
layout: post
title: Java内存模型
date: 2017-09-13
categories:
- Java
tags:
- JMM
---

现代CPU的发展经历了, 从发展晶体管数量到提高多核心并行运行效率的发展历程, 但从串行运行切换到并行运行似乎没有想象中的那么简单, 现代CPU的运行速度和周边存储设备的速度相差几个数量级, 所以引入了高速缓存解决此问题, 不过同时也遇到新的问题: 缓存一致性问题.
<!-- more -->
比如多个核心同时对同一块内存区域做出修改, 当把修改后的数据回写至主内存时已谁的数据为准呢?不同的架构定义了不同的协议: MSI、MESI、MOSI、Synapse等等;

![](/img/2017-09-13-Java_memory-model-01.png)

除此之外, 为了提高运行效率, 实践中还会出现三种情况: 处理器乱序执行、编译器重排序、内存系统重排序, 为了保证并发条件下, 内存数据的安全性, 并屏蔽底层硬件和操作系统的细节, 达到各平台下并发一致性, Java定义了内存模型;

Java内存模型定义了多线程情况下内存访问的规则

## 编译器重排序
Java编译器优化时必须遵守as-if-serial原则, 通俗地讲, 就是在单线程情况下, 编译器优化后的重排序的执行结果和顺序执行的结果保持一致, 但数据之间存在依赖时, 编译器不能调整指令之间的顺序, 那何为数据依赖呢?主要有3种情况:
1. Read after Write
2. Write after Read
3. Write after Write

## happens-before
Java内存模型核心概念happens-before原则, __happens-before原则描述了两个操作的可见性, 如果A操作happens-before操作B, 那么A操作的结果对B操作可见__, 具体原则:
1. 线程内执行的操作happens-before后面的操作, 保证基本的顺序规则;
2. 对volatile变量的写操作happens-before对该变量的读操作;
3. 线程的start()方法happens-before线程内的第一个动作;
4. 线程的最后一个动作happens-before他的终止事件(我们可以通过Thread.join()或者Thread.isAlive()的返回值判断线程是否已经终止);
5. 对线程的interrupt调用happens-before线程收到中断事件的发生;
6. 对象的unlock操作happens-before对同一个对象的lock操作;
7. 一个对象的构造函数完成happens-before该对象的finalize方法;
8. 传递性: A操作happens-before操作B, B操作happens-before操作C, 则A操作happens-before操作C;
时钟顺序上的先后并不能多线程的安全性, happens-before强调的是可见性的先后性.

## volatile理解
volatile实现了synchronized一半的语义, 即可见性, 对volatile变量的写happens-before对volatile变量的读, 同时还禁止指令重排序, volatile适用于那些对变量的修改不依赖于变量当前状态的情景, 典型的应用场景是用bool值控制线程的运行终止状态, 另一个线程改变标志的值, 因为一个线程对volatile变量的写操作会强制回写主内存, 同时invalidate其他线程的工作内存中的缓存行, 当线程读取volatile变量时由于缓存失效, 便会到主内存读取最新数据, 所以volatile变量的读操作代价很低, 接近于普通变量的读操作, 但写操作开销却很大, 所以volatile适用于读多写少的情景, 此时可以用volatile节省读操作的开销(读操作直接读, 写操作加同步锁), volatile不具有原子性。

``` java
private volatile boolean shouldShutdown = false;

public void shutdown() {
  shouldShutdown = true;
}

public void work() {
  while (!shouldShutdown) {
    doWork();
  }
}
```


``` java
private volatile int i;

public int getValue() {
  return i;
}

public synchronized void setValue(int v) {
    i = v;
}
```

## JMM的底层实现
Java内存模型底层是通过内存屏障实现的, 对于JIT编译器, 它会在需要happens-before关系的代码中插入相应的读读、读写、写读、写写内存屏障, 对于volatile变量：
- 对该变量的写操作之后，编译器会插入一个写屏障
- 对该变量的读操作之前，编译器会插入一个读屏障
内存屏障有两个作用：
- 禁止指令重排序
- 强制将写入高速缓存的脏数据写入主内存，或者invalidating缓存中的数据，从主内存读取
内存屏障能够在变量的写操作之后，保证其他线程对volatile变量的修改对当前线程可见，或者当前线程的修改对其他线程可见，换句话说，写屏障会通过强制刷新处理器缓存让其他线程拿到最新数据。