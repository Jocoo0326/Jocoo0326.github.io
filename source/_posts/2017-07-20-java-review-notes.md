---
layout: post
title: Java复习笔记 - 2
date: 2017-07-20
categories:
- Java
tags:
- Java basic
---

Java基础知识总结-2:
<!-- more -->
## HashTable、HashMap、TreeMap、LinkedHashMap有何区别
1. HashTable是早期Java提供的哈希表实现，本身同步，不支持null键和值，initialCapacity=11，loadFactor=0.75，rehash之后newCapacity=(oldCapacity << 1) + 1，HashMap扩容后为原来2倍
2. HashMap是应用更为广泛的哈希表实现，行为大致和HashTable一致，主要区别在于HashMap非同步，支持null键和值，通常情况下，HashMap进行put和get可以达到常数时间的性能，所以它是绝大部分利用键值对存储场景的首选
3. HashMap的initialCapacity=16，loadFactor=0.75
4. TreeMap则是基于红黑树的一种提供顺序访问的Map，和HashMap不同，它的get、put、remove之类的操作都是O(logn)的时间复杂度，具体顺序可以由指定的Comparator来决定，或者根据键的自然顺序来判断
5. HashMap并发环境可能出现无限循环(桶内的链表变成了环形链表导致, resize并发导致)
6. HashMap的性能表现非常依赖于哈希码的有效性，所以hashCode和equals的一些基本约定：
- equals相等，hashCode一定要相等
- 重写了hashCode也要重写equals
- hashCode需要保持一致性，状态改变返回的哈希值仍然要一致
- equals的对称、反射、传递等特性
7. HashMap的hash(Object key):
``` java
int h;
return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16); // 将高位数据与低位数据合并，可以有效减少碰撞
index = (tab.length - ) & hash;
```
8. ConcurrentHashMap基于lock实现锁分段技术，首先将数据分段，为每个段分配一把锁，当一个线程占用锁访问其中一段数据时，其他段的数据也能被其他线程访问，ConcurrentHashmap不仅保证了多线程环境下的数据访问安全性，性能上也有长足的提升
9. LinkedHashMap基于HashMap实现，提供了插入顺序和访问顺序功能，通过accessOrder设置，通过双链表自身维护了访问的顺序，提供原生LRUCache功能实现
10. TODO 红黑树


## 如何保证集合是线程安全的?ConcurrentHashMap如何实现高效的线程安全?
1. Java提供了不同层面的线程安全支持。传统集合框架内部，除了HashTable等同步容器，还提供了同步包装器(Synchronized Wrapper)，我们可以调用Collections工具类提供的包装方法，获取一个同步容器(i.e: Collections.synchronizedMap etc)，但非常粗粒度的方式(利用自身作mutex)，性能低下
2. 并发包提供的线程安全容器类
- 各种并发容器，ConcurrentHashMap、CopyOnWriteArrayList
- 各种线程安全队列(Queue/Deque)，ArrayBlockingQueue、SynchronousQueue
- 各种有序容器的线程安全版本
3. 利用Unsafe的CAS(Compare and swap，CPU原子指令)实现无锁并发机制(free-lock)，线程更新时判断内存值是否与期望值一致，若是说明没有其他线程修改过，则更新新值，否则返回失败，重试进行，直至成功，最重要的是CAS是CPU原子指令，CAS操作通常配合while无限循环


## Java提供了哪些io方式?NIO如何实现多路复用?
1. 传统java.io基于流模型实现，提供输入输出流，读取写入字节或字符流，属于同步阻塞io，缺点是io效率和扩展性存在局限性
2. Java1.4引入NIO框架，提供了Channel、Selector、Buffer等新的抽象，可以构建多路复用、同步非阻塞IO程序，同时提供了更接近操作系统底层的高性能数据操作方式
3. Java7中，NIO进一步改进，也就是NIO2，引入了异步非阻塞IO，基于事件和回调机制
4. select模式是使用一个线程做监听，而bio每次来一个链接都要做线程切换，所以节省的时间在线程切换上
5. Selector管理channel，channel关心一种事件，当channel接受到某事件时，selector.select()方法会被通知，进而处理IO操作
6. Linux上依赖epoll机制，windows依赖iocp


## ThreadPoolExecutor的理解
1. 参数意义:
- corePoolSize: 核心Worker线程的数量，可以理解为长期驻留的线程数目（除非设置了allowCoreThreadTimeOut）
- maximumPoolSize: 线程池最大Worker线程的数量，就是线程不够时能够创建最大线程数
- keepAliveTime: Worker线程结束之前的空闲时间
- unit: 时间的单位
- workQueue: 存放Runnable的阻塞队列
- threadFactory: 创建线程的工厂
- handler: 不能接受Runnable时的拒绝策略
2. 执行规则:
- 若currentThreadCount < corePoolSize 创建core线程，core线程会立即执行；
- 若currentThreadCount >= corePoolSize 放入阻塞队列；
- 队列已满后，若currentThreadCount < maximumPoolSize 创建新的线程。
3. 为什么能够复用线程?以及空闲超时的原理?
提交任务Runnable后，线程池会创建一个Worker线程，线程中while循环执行任务，线程执行完当前任务后，会从等待队列里获取一个任务并执行，如此就避免了重复创建线程，实际是一个线程执行多个runnable， 线程的超时由队列的超时操作实现。
4. Thread和Runnable的理解
Runnable通常代表具体的业务逻辑，Thread代表操作系统线程的调度管理，早起java线程api将业务逻辑和线程创建调度管理混在一起，极为不便，就像HTTP请求还要处理TCP握手一样，很多框架的存在的意义也在于此，例如OKHTTP，用户用接口定义请求，然后执行，透明化HTTPS的细节
5. Executors常用线程池配置
- newCachedThreadPool(), 通常用来处理大量短时间的工作任务，特点：试图缓存线程并重用，当无线程可用时，创建新的线程执行任务；线程闲置60S后，自动移出线程池，长时间闲置不会消耗资源，corePoolSize为0，SynchronousQueue作为工作队列;
- newFixedThreadPool(int nThreads), 重用指定数目(nThreads)的线程，使用LinkedBlockingQueue作为工作队列，任何时候最多只有nThreads个线程是活动的，任务超过nThreads后，任务会在工作队列中等待空闲线程，如果有工作线程退出，将会有新的线程被创建，以补足nThreads数目；
- newSingleThreadExecutor(), 它创建的是ScheduledExecutorService，支持定时或周期性的工作调度，工作线程数目限制为1，所以任务都是被顺序执行，最多只会有一个任务处于活动状态，并且不允许改变线程池实例，避免改变线程数目；
- newScheduledThreadPool(int corePoolSize), 同样是ScheduledExecutorService, 区别是会保持corePoolSize个工作线程;
6. 线程池大小选择策略
- 如果我们的任务主要是计算，那么意味着CPU的处理能力是稀缺资源，我们不能够通过增大线程数提高计算能力，因为线程越多，上下文切换的开销也越大，通常建议按照CPU核的数目N或N+1；
- 如果是等待较多的任务，如I/O操作比较多，可以参考Brain Goetz推荐的计算方法：线程数 = CPU核数 x (1 + 平均等待时间/平均工作时间);
- 实际可能受到各种系统资源限制影响，需要结合其他资源的使用，合理调整线程数量;


## Synchronized和ReentrantLock有什么区别？
1. Synchronized是Java内建同步机制，提供了互斥的语义和可见性(volatile)，一个线程获取锁，其他试图获取锁的线程只能等待或阻塞
2. ReentrantLock是再入锁，语义和Synchronized基本相同，通过调用lock方法获取锁，书写灵活，一般配合try-catch-finally，并在finally中调用unlock释放锁，当线程已获取了锁，lock方法会立即返回
3. Reentrantlock提供更细粒度的同步操作，可以提供公平性(等待时间长的线程优先获取锁)，定义条件
4. 通过lock.isHeldByCurrentThread可以判断当前线程是否拥有这个锁


## Java并发包工具类
1. 主要特点
- 提供了比synchronized更加高级的同步结构，包括CountDownLatch、CyclicBarrier、Semaphore，可以在更多实际场景下使用；
- 并发容器类，如ConcurrentHashMap、CopyOnWriteArrayList
- 并发队列实现类，如ArrayBlockingQueue, LinkedListBlockingQueue, PriorityBlockingQueue, SynchronousQueue
- 强大的Executor框架，可以创建各种不同类型的线程池，调度任务运行等
2. 目的：
- 完成业务逻辑
- 提高吞吐量
3. Semaphore
- 一种计数器，可以控制一定数量的permit，以限制通用资源的访问
- acquire/release基本操作, acquire获取permit则执行，否则阻塞, release释放permit
4. CountDownLatch和CyclicBarrier
- CountDownLatch不可以重置，无法重用，CyclicBarrier可以重用
- CountDownLatch基本操作是countDown/await, await会阻塞等待countDown达到足够的次数，不管在哪个线程countDown, CountDownLatch通常用于线程间等待操作
- CyclicBarrier可以指定多个线程达到公共障碍点(common barrier point)前互相等待，然后一起执行，barrier重置


## ThreadLocal的理解
1. 线程局部存储，提供一个线程独立的局部变量存储机制；
2. 通过Thread类中ThreadLocal.ThreadLocalMap threadLocals变量实现，每个线程独有此变量，这是一个类似于HashMap的结构，内部用Entry[]存储数据，
    每个Entry是一个WeakReference<ThreadLocal<?>>扩展类，ThreadLocal作为key，欲存储的变量作为value，通过ThreadLocal.get()/set()方法操作当前线程关联的
    map结构中对应的Entry数据键值对；
3. 一个ThreadLocal对应一个线程局部变量，若多个线程均需此局部变量，则ThreadLocal会被多个线程引用


## CAS和AQS
1. CAS --CompareAndSwap, Unsafe提供的内部操作，基于CPU特定指令，属轻量级操作指令，实现free-lock机制的基础;
2. AQS --AbstractQueuedSynchronizer, 基于FIFO队列实现的同步器，ReentrantLock, CountDownLatch, ThreadPoolExecutor$Worker等都是基于AQS实现;
3. Free-lock高并发的基础-CAS，AQS是JAVA提供的封装CAS的实现;