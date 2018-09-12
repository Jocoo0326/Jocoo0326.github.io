---
layout: post
title: Java中synchronized关键字的实现原理
date: 2017-03-26
categories:
- Java
tags:
- synchronized
---

synchronized提供了Java中同步原语的实现
<!-- more -->
锁操作一直都被认为是相当昂贵的，但jdk1.6之后，对synchronized进行了多种优化，引入了偏向锁、轻量级锁、锁消除等机制
### 基本使用
synchronized关键字使用点：
1. 修饰实例方法
2. 修饰类方法
3. 修饰同步块
经编译后同步块对应monitorenter和monitorexit字节码，对应同步块的开始和结束，值得注意的是，javac会默认生成一个异常表，用于保证monitorexit必会执行；修饰方法时，将方法的flags中添加ACC_SYNCHRONIZED标志;
修饰实例方法时以实例作为锁对象， 修饰类方法时以类的class作为锁对象， 修饰同步块时以给定对象作为锁对象

### 实现原理
JDK1.5之前synchronized属于重量级锁，基于操作系统的Mutex lock实现，线程的阻塞和恢复需要陷入内核态，但经研究发现很多同步的场景并非是非此即彼的，这由共享变量的竞争情况而定，因此锁优化引入了偏向锁、轻量级锁、锁消除等机制，这些实现都与Java对象头息息相关

### Java对象内存布局
Java对象在内存中的布局分为三个部分：对象头、实例数据、填充数据
![](/img/2017-03-26-Java-object-memory-structure-01.png)
1. Mark word
存储对象hashcode、GC年龄、锁标记、偏向锁、轻量级锁、重量级锁信息
![32位JVM](/img/2017-03-26-Java-object-memory-structure-02.png)
2. Class pointer
指向实例在方法区中对应类型的入口
3. Array length
如果对象是数组类型则存在

### 偏向锁
假设：大部分场景中只有一个线程使用锁
目的：消除无竞争情况下同步开销，CAS操作都不要
偏向锁偏向于第一个获取锁的线程，若该锁没有被其他线程获取，则持有偏向锁的线程永远不需要同步，第一次获取偏向锁时使用CAS更新Mark word中信息，以后同一线程再次进入同步块都不用进行任何同步操作

### 轻量级锁
假设：大部分的锁，在整个同步周期内不存在竞争，即多个线程轮流获取锁
目的：轮流获取锁的情况下，使用CAS操作避免使用互斥量的开销
首先在当前线程的栈帧中建立一个锁记录(Lock record), 将当前的Mark word拷贝进去，然后使用CAS操作将Mark word更新为指向Lock Record指针，如果成功，这个线程就持有了对象的轻量级锁，并且更新Mark word标志位；如果失败了，虚拟机会检查Mark word是否指向当前线程的栈帧，如果是，说明当前线程已有对象锁，接着执行同步块，否则，说明有其他线程争用同一个锁，那么轻量级锁将膨胀为重量级锁，锁标记变为10，Mark word指向互斥量的指针

### 重量级锁
操作系统底层的mutex lock