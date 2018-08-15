---
layout: post
title: Java learning notes - 3
date: 2018-07-26
categories:
- Java
tags:
- Java basic
---

Java基础知识总结-3:
<!-- more -->
## 类加载过程，双亲委派模型
1. 加载(Loading):
将java字节码数据从不同的数据源读取到JVM中，并映射为JVM认可的数据结构(Class对象)，这里的数据源可能是各种各样的形态，如jar文件、class文件，甚至是网络数据源等；
如果输入数据不是ClassFile结构，会抛出ClassFormatError；
2. 链接(Linking):
核心步骤，把原始类定义信息转化为JVM运行时信息
- 验证(Verification), 这是虚拟机安全的重要保障，JVM需要验证字节信息是否符合JAVA虚拟机规范，否则认为VerifyError，这样防止恶意信息和
不合规信息危害JVM的运行，验证阶段可能出发更多class的加载;
- 准备(Preparation), 创建类或者接口中的静态变量，并初始化静态变量的初始值，此处和显示初始化不同，重点在于分配内存空间，不会执行JVM指令;
- 解析(Resolution), 这一步会将常量池中的符号引用(symbolic reference)替换为直接引用，以及类、接口、方法和字段的解析;
3. 初始化(Initialization): 
这一步真正执行类初始化的代码逻辑，包括静态字段赋值的动作，以及执行类定义中的静态初始化块的逻辑，编译器在编译阶段将这部分逻辑准备好，
父类型的初始化逻辑优先于当前类型的逻辑。
4. 双亲委派模型
类加载器试图加载某个类型的时候，除非父加载器找不到相应类型，否则尽量将这个任务代理给当前加载器的父加载器去做，目的是避免重复加载
5. 两个class仅在字节码和加载器相同时才视为同一个class


## 运行时动态生成Java类
1. 字节码操纵框架： ASM、cglib、Javassist
2. 关键是由byte code生成class对象的过程, 考虑到类加载过程中，主要功能是defineClass方法实现了字节码数据到class对象的转换
3. 动态代理其实就是运行时生成class


## JVM内存区域划分
1. 程序计数器(PC, Program Counter Register). 在JVM规范中，每个线程都有它自己的程序计数器，并且任何时间一个线程只有一个方法在执行，也就是所谓的当前方法，程序计数器会存储当前线程正在执行的Java方法的JVM指令地址；若是本地方法，则是undefined;
2. Java虚拟机栈(Java Virtual Machine Stack). 每个线程创建时都会创建一个虚拟机栈，内部是一个个栈帧(Stack Frame)，对应一次次Java方法调用，栈帧中存储着局部变量表、操作数栈、动态链接、方法正常退出或异常退出的定义等;
3. Java堆(Heap). Java内存管理的核心区域，用来放置Java对象实例，几乎所有创建的Java对象实例都是被直接分配在堆上。堆被所有的线程共享，在虚拟机启动时，我们指定的"Xmx"之类的参数就是用来指定最大堆空间，堆根据不同的垃圾收集器有更进一步的划分，最有名的是新生代和老年代的划分;
4. 方法区(Method Area). 也是所有线程共享的区域，用来存储元数据(Meta data)，如类结构信息，以及对应的运行时常量池、字段、方法代码等,早期Hotspot JVM实现，很多人习惯于将方法区称为永久代(Permanent Generation)。Oracle JDK 8中将永久代移除，同时增加了元数据区(Metaspace);
5. 本地方法栈(Native Method Stack). 它和Java虚拟机栈非常类似，支持对本地方法的调用，也是每个线程创建一个，在Oracle Hotspot JVM中，本地方法栈和Java虚拟机栈是在同一块区域，这取决于具体实现，规范未强制。


## 堆内部是什么结构
1. 新生代
- 新生代是大部分对象创建和销毁的地方，在通常的Java应用中，绝大部分对象的生命周期都很短暂，其内部又分为Eden区域，作为对象初始分配的区域，两个Survivor，有时候也叫from、to区域，被用来放置从Minor GC中保留下来的对象;
- JVM会随意选取一个Survivor作为to区域，然后会在GC过程中进行区域间拷贝，也就是将Eden中存活下来的对象和from区域中的对象，拷贝到to区域这种设计为了防止内存的碎片化，并进一步清理无用对象;
- 从内存模型而不是垃圾收集的角度，对Eden区域继续划分，Hotspot JVM还有一个概念叫Thread Local Allocation Buffer(TLAB), 这是JVM为每个线程分配的私有缓存区域，否则，多线程同时分配内存时，为避免操作同意地址，可能需要使用加锁机制，进而影响分配速度，TLAB其实分配在Eden中;
2. 老年代
- 放置长生命周期的对象，通常都是从Survivor中拷贝过来的对象，通常，普通对象会被分配在TLAB上，如果对象较大，JVM会试图分配在Eden其他位置上，如果对象太大，完全无法在新生代找到足够长的连续空闲空间，JVM会直接分配到老年代;
3. 永久代
- 这部分就是早期Hotspot JVM的方法区实现，用于存储Java类元数据、常量池、Intern字符串缓存，JDK8之后就不存在永久代这块了;
4. 常用修改堆和内部大小的JVM参数
- 最大堆体积
``` java
-Xmx value
```
- 初始最小堆体积
``` java
-Xms value
```
- 老年代和新生代的比例
``` java
-XX:NewRatio=value(默认是3，老年代是新生代的3倍大)
```
- Eden和Survivor的比例
``` java
-XX:SurvivorRatio=value
```


## register-based VM and stack-based VM
1. 基于栈的虚拟机是操作数存储在栈上，通过pop操作数，执行指令，再push结果的虚拟机，代表有JVM、CPython、.NET CLR
2. 基于寄存器的虚拟机是将操作数直接存在寄存器上，执行指令，将结果写在另一个寄存器上的虚拟机，代表Lua、Dalvik
3. 栈虚拟机字节码占用空间较少，寄存器虚拟机执行效率较高
4. jvm操作数栈上long, double占8个字节, boolean byte short int float reference占4个字节(64位hotspot虚拟机)
5. Dalvik字节码2个字节，jvm字节码1个字节



## Java Virtual Machine Specification
1. Java Type
2. primitive type

| type    | bit    | sign    |  abv | default          | range               | note
| ------  | ------ | ------- | ---- | ---------------- | ------------------- | ----
| byte    | 8-bit  |  signed |    B | default 0        | [-2^ 7, 2^ 7 - 1]   | non
| short   | 16-bit |  signed |    S | default 0        | [-2^15, 2^15 - 1]   | non
| char    | 16-bit |unsigned |    C | default \u0000   | [    0,    65535]   | UTF-16
| int     | 32-bit |  signed |    I | default 0        | [-2^31, 2^31 - 1]   | non
| long    | 64-bit |  signed |    J | default 0        | [-2^63, 2^63 - 1]   | non
| float   | 32-bit |  signed |    F | default +0.0f    | ~[-3.4E38, 3.4E38]  | IEEE754
| double  | 64-bit |  signed |    D | default +0.0d    | ~[-1.8E308, 1.8E308]| IEEE754
| boolean | 32-bit |     non |    Z | default false(0) | {false, true}       | non

- 其实: boolean f = true; => iconst_1; istore_1; boolean数组映射为byte数组，操作使用byte数组的操作指令(baload, bastore)

3. Reference type
- class type 引用类的实例
- array type 引用数组
- interface type 引用实现了特定接口的类实例或数组
4. Run-Time Data Area
5. The pc Register
每个线程独有一个程序计数器(program counter), 任一时刻，每个JVM线程都在执行某个方法的代码，即当前线程的当前方法，若非native方法，pc会记录当前执行指令地址，native方法则为undefined
6. Java Virtual Machine Stack
每个JVM线程启动时都会创建有一个私有的JVM栈, JVM栈类似于C语言栈, 每调用一个方法就会创建一个栈帧(frames), 栈帧主要分为局部变量区、操作数栈; JVM栈允许设置大小xss; 若线程需求的内存超过JVM栈允许的大小抛出StackOverflowError(方法调用栈太深，超过栈的允许范围), 若JVM栈能够动态扩展，但没有足够的内存完成JVM栈的扩展，或者启动新线程时没有足够的内存初始化JVM栈, 抛出OutOfMemoryError(栈无法分配或无法扩展);
7. Heap
堆由所有JVM线程共享，用于创建类实例和数组对象，堆内存的释放由gc(garbage collector)处理; JVM 允许调整堆内存大小;
8. Method Area
方法区由所有JVM线程共享, 用于存储运行时常量池，静态字段、静态方法，方法代码数据; 逻辑上属于堆的一部分, 但规范不限制方法区的位置以及管理方法区内存的方式; JVM允许设置方法区的大小;
9. Run-Time Constant Pool
常量池是每个类或者接口常量池表在运行时的内存体现, 包含编译时就知道的字面值、运行时需要的字段引用，类似于传统语言的符号引用,但比这个更为宽泛; 常量池从方法区中分配内存, 当JVM创建class或者interface时创建对应的常量池;
10. Native Method Stacks


## 快速平方根倒数算法
1. IEEE754
32位float格式：

| 符号  | 指数(8)   | 有效数字(23)
| ----- | -------- | -----------------
| 0     | 01111100 | 01000000000000000000000

``` java
x = (-1)^Si·(1+m)·2^(E-B)
```

Si: Sign 符号位
m: Mantissa 有效数字的尾数, m∈[0, 1), m = 2^(-2) = 0.250
E: Exponent 偏移处理后的指数, E=124
B: 偏移量, 为了能表示[-127, 128]的指数, B=127
所以x = (1+0.250)·2^(-3) = 0.15625
2. code:
``` java
float Q_rsqrt( float number )
{
    long i;
    float x2, y;
    const float threehalfs = 1.5F;
    x2 = number * 0.5F;
    y  = number;
    i  = * ( long * ) &y;                       // evil floating point bit level hacking（对浮点数的邪恶位元hack）
    i  = 0x5f3759df - ( i >> 1 );               // what the fuck?（这他妈的是怎么回事？）
    y  = * ( float * ) &i;
    y  = y * ( threehalfs - ( x2 * y * y ) );   // 1st iteration （第一次迭代）
    //y  = y * ( threehalfs - ( x2 * y * y ) );   // 2nd iteration, this can be removed（第二次迭代，可以删除）
    return y;
}
```
3. explaination:
思路: 先求取近似值，然后再用牛顿迭代提高精度
证明: 
[https://zh.wikipedia.org/wiki/平方根倒数速算法]