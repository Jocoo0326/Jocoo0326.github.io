---
layout: post
title: Deep into ThreadPoolExecutor
date: 2017-06-06
categories:
- Java
tags:
- ThreadPoolExecutor,
- AQS
---

无论是网络请求，还是图片处理，但凡遇到频繁任务工作队列处理，都会用到线程，说到线程必要理解线程池，简单说线程池能够按需创建线程，并能够复用已创建的线程，避免没必要的创建销毁线程所带来的开销，同时又能够在任务执行完之后自动回收(队列超时)线程的一种池技术，下面是对ThreadPoolExecutor的理解
<!--more-->

#### 参数意义
- corePoolSize: 核心Worker线程的数量，可以理解为长期驻留的线程数目（除非设置了allowCoreThreadTimeOut）
- maximumPoolSize: 线程池最大Worker线程的数量，就是线程不够时能够创建最大线程数
- keepAliveTime: Worker线程结束之前的空闲时间
- unit: 时间的单位
- workQueue: 存放Runnable的阻塞队列
- threadFactory: 创建线程的工厂
- handler: 不能接受Runnable时的拒绝策略


#### 执行规则
1. 若currentThreadCount < corePoolSize 创建Worker线程，Worker线程会立即执行；
2. 若currentThreadCount >= corePoolSize 放入阻塞队列；
3. 队列已满后，若currentThreadCount <= maximumPoolSize 创建新的Worker线程。
4. 队列已满后，若currentThreadCount > maximumPoolSize 则reject
    

#### 为何能够复用线程?以及空闲超时的原理?
提交任务Runnable后，线程池会创建一个Worker线程，线程中while循环执行任务Runnable，线程执行完当前任务后，会从等待队列里获取一个任务并执行，如此就避免了重复创建线程，实际是一个线程执行多个runnable，线程的超时由队列的超时操作实现。


#### Thread和Runnable的理解
Runnable通常代表具体的业务逻辑，Thread代表操作系统线程的调度管理，早期java线程api将业务逻辑和线程创建调度管理混在一起，极为不便，就像HTTP请求还要处理TCP握手一样，很多框架的存在的意义也在于此，例如OKHTTP，用户用接口定义请求，然后执行，透明化HTTPS的细节


#### Executors常用线程池配置
1. newCachedThreadPool(), 通常用来处理大量短时间的工作任务，特点：试图缓存线程并重用，当无线程可用时， 创建新的线程执行任务；线程闲置60S后，自动移出线程池，长时间闲置不会消耗资源，corePoolSize为0，maximumPoolSize为Integer.MAX_VALUE，SynchronousQueue作为工作队列;
2. newFixedThreadPool(int nThreads), 重用指定数目(nThreads)的线程，使用LinkedBlockingQueue作为工作队列，任何时候最多只有nThreads个线程是活动的，任务超过nThreads后，任务会在工作队列中等待空闲线程，如果有工作线程退出，将会有新的线程被创建，以补足nThreads数目, corePoolSize为nThreads, maximumPoolSize为nThreads, keepAliveTime为0； 
3. newSingleThreadExecutor(), 它创建的是ScheduledExecutorService，支持定时或周期性的工作调度， 工作线程数目限制为1，所以任务都是被顺序执行，最多只会有一个任务处于活动状态，并且不允许改变线程池实例，避免改变线程数目；
4. newScheduledThreadPool(int corePoolSize), 同样是ScheduledExecutorService, 区别是会保持corePoolSize个工作线程;


#### 线程池大小选择策略
1. 如果我们的任务主要是计算，那么意味着CPU的处理能力是稀缺资源，我们不能够通过增大线程数提高计算能力， 因为线程越多，上下文切换的开销也越大，通常建议按照CPU核的数目N或N+1；
2. 如果是等待较多的任务，如I/O操作比较多，可以参考Brain Goetz推荐的计算方法:
 线程数 = CPU核数 x (1 + 平均等待时间/平均工作时间);
3. 实际可能受到各种系统资源限制影响，需要结合其他资源的使用，合理调整线程数量;


#### 代码分析
我们以Executors.newCachedThreadPool()为例，分析一下代码流程，首先是execute(Runnable command)方法
``` java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```
ctl的类型是AtomicInteger, 用一个整数表示线程池的运行时状态，包括rs(RunState)和wc(WokerCount), 同时使用一个AtomicInteger就实现对状态的原子操作, 设计简洁精妙
``` java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;

// Packing and unpacking ctl
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
```
回到execute方法，若当前Worker线程数量小于corePoolSize时, 则直接添加一个Worker线程, addWorker(command, true), 若成功则直接return
``` java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
    return workerStarted;
}
```
addWorker中首先通过两个嵌套的for循环检测边界条件：
1. 若此时rs大于等于SHUTDOWN(SHUTDOWN,STOP,TIDYING,TERMINATED)，则不能添加Worker, 除非一种情况: 当前rs处于SHUTDOWN, 但不能添加新的任务Runnable, 工作队列不能为空, 可以添加Worker完成队列中的任务;
2. currentWorkerCount不能超过corePoolSize或者maximumPoolSize之所以要循环做check是因为execute需满足并发操作，这里其实是spin + CAS(compare and swap)操作, 满足check条件之后便会创建一个Worker, 之后启动线程回到execute方法中，若当前Worker线程数量大于等于corePoolSize时, 尝试将runnable添加到工作队列, 此时分两种情况：
- 若添加任务成功, 但此时线程池已停止, 则移除任务并reject, 若没有停止但没有Worker线程，则创建一个Worker进行工作;
- 若添加任务失败，此时对应currentWorkerCount >= corePoolSize && workqueue is full， 则添加Worker线程, 若currentWorkerCount >= maximumPoolSize则reject

Executors.newCachedThreadPool()用的是SynchronousQueue, SynchronousQueue的offer总是返回false, 当线程数多于corePoolSize时总是添加Worker线程

下面看一下Worker线程是如何工作的
``` java
Worker(Runnable firstTask) {
    setState(-1); // inhibit interrupts until runWorker
    this.firstTask = firstTask;
    this.thread = getThreadFactory().newThread(this);
}

/** Delegates main run loop to outer runWorker. */
public void run() {
    runWorker(this);
}

final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                    (Thread.interrupted() &&
                    runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```
Worker继承自AbstractQueuedSynchronizer(AQS), AQS设计初衷是为了使用atomic int代表同步状态的同步器提供的基本实现, 支持FIFO等待队列, 支持独占模式和共享模式, JDK中ReentrantLock、CountDownLatch、Semaphore都是基于AQS实现, 子类实现tryAcquire、tryRelease, 用AQS主要是同步线程的中断控制状态, 比如setCorePoolSize改变corePoolSize时, 若workerCount > corePoolSize, 需要interruptIdleWorkers(), 此时tryLock成功说明线程空闲, 使用ReentrantLock此处会成功, 不合适
``` java
public void setCorePoolSize(int corePoolSize) {
    if (corePoolSize < 0)
        throw new IllegalArgumentException();
    int delta = corePoolSize - this.corePoolSize;
    this.corePoolSize = corePoolSize;
    if (workerCountOf(ctl.get()) > corePoolSize)
        interruptIdleWorkers();
    else if (delta > 0) {
        // We don't really know how many new threads are "needed".
        // As a heuristic, prestart enough new workers (up to new
        // core size) to handle the current number of tasks in
        // queue, but stop if queue becomes empty while doing so.
        int k = Math.min(delta, workQueue.size());
        while (k-- > 0 && addWorker(null, true)) {
            if (workQueue.isEmpty())
                break;
        }
    }
}
private void interruptIdleWorkers() {
    interruptIdleWorkers(false);
}
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```
runWorker就是firstTask或者从任务队列中获取一个任务，然后执行, 循环此过程, 执行任务的过程中lock, 表示工作状态,其他地方空闲中断操作便会失败(如上分析), 此处getTask时关键
``` java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```
同样是spin + CAS操作, 若线程池已处于STOP以上状态(TIDYING、TERMINATED)或SHUTDOWN且队列为空, 则将Worker数量-1, 并返回null, 返回null后会使Worker终止, 然后从队列中获取runnable, 若允许超时调用poll, 否则调用take, 队列的poll是会阻塞的, 阻塞keepAliveTime时长, 下次执行return null, Worker终止, 这就是线程池超时释放线程的原因。