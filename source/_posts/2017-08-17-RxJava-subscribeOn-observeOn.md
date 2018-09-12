---
layout: post
title: RxJava的subscribeOn和observeOn区别与原理
date: 2017-08-17
categories:
- android
tags:
- RxJava
---

RxJava代码语义清晰，数据产生、变换、消费一目了然，极大的提高了代码的可读性、维护性，同时还提供了另外一个特性------线程切换
<!-- more -->
RxJava用于线程切换的主要有2个操作符：subscribeOn和observeOn

### 区别
* subscrieOn
指定Observable在特定的调度器上发射数据
![](http://reactivex.io/documentation/operators/images/subscribeOn.c.png)
- observeOn
指定Observer在特定的调度器上接收Observable的数据
![](http://reactivex.io/documentation/operators/images/observeOn.c.png)
observeOn在收到错误通知时会立即回调observer的onError方法，即使之前还有未消费的数据，onError会在它们之前被传递
![](http://reactivex.io/documentation/operators/images/observeOn.e.png)

### 特性
* subscribeOn
Observable的subscribe方法将在操作符链条中第一个subscribeOn指定的调度器上执行，就算出现多个subscribeOn操作符也是如此
* observeOn
observeOn会将直接紧跟的后续操作符在其指定的调度器上执行
例如：
``` java
    Observable.create((ObservableOnSubscribe<String>) emitter -> {
      System.out.println(Thread.currentThread());
      String s = "from 1 created";
      System.out.println(s);
      emitter.onNext(s);
      emitter.onComplete();
    })
        .subscribeOn(Schedulers.computation())
        .map(s -> {
          System.out.println(Thread.currentThread());
          System.out.println(s + " in map");
          return s;
        })
        .subscribeOn(Schedulers.io())
        .observeOn(Schedulers.io())
        .filter(s -> {
          System.out.println(Thread.currentThread());
          System.out.println(s + " in filter");
          return true;
        })
        .subscribeOn(Schedulers.newThread())
        .observeOn(Schedulers.computation())
        .subscribe(s -> {
          System.out.println(Thread.currentThread());
          System.out.println(s + " in observer");
        });
    try {
      Thread.sleep(100);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
```
结果：
```
Thread[RxComputationThreadPool-1,5,main]
from 1 created
Thread[RxComputationThreadPool-1,5,main]
from 1 created in map
Thread[RxCachedThreadScheduler-2,5,main]
from 1 created in filter
Thread[RxComputationThreadPool-2,5,main]
from 1 created in observer
```

### 源码分析
首先看subscribeOn，只是将原Observable封装成ObservableSubscribeOn
``` java
public final Observable<T> subscribeOn(Scheduler scheduler) {
	ObjectHelper.requireNonNull(scheduler, "scheduler is null");
	return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
}
```
那么在发生subscribe时会调用其subscribeActual方法
``` java
public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {
    final Scheduler scheduler;

    public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
        super(source);
        this.scheduler = scheduler;
    }

    @Override
    public void subscribeActual(final Observer<? super T> observer) {
        final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer);

        observer.onSubscribe(parent);

        parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
    }

    static final class SubscribeOnObserver<T> extends AtomicReference<Disposable> implements Observer<T>, Disposable {

        private static final long serialVersionUID = 8094547886072529208L;
        final Observer<? super T> downstream;

        final AtomicReference<Disposable> upstream;

        SubscribeOnObserver(Observer<? super T> downstream) {
            this.downstream = downstream;
            this.upstream = new AtomicReference<Disposable>();
        }

        @Override
        public void onSubscribe(Disposable d) {
            DisposableHelper.setOnce(this.upstream, d);
        }

        @Override
        public void onNext(T t) {
            downstream.onNext(t);
        }

        @Override
        public void onError(Throwable t) {
            downstream.onError(t);
        }

        @Override
        public void onComplete() {
            downstream.onComplete();
        }

        @Override
        public void dispose() {
            DisposableHelper.dispose(upstream);
            DisposableHelper.dispose(this);
        }

        @Override
        public boolean isDisposed() {
            return DisposableHelper.isDisposed(get());
        }

        void setDisposable(Disposable d) {
            DisposableHelper.setOnce(this, d);
        }
    }

    final class SubscribeTask implements Runnable {
        private final SubscribeOnObserver<T> parent;

        SubscribeTask(SubscribeOnObserver<T> parent) {
            this.parent = parent;
        }

        @Override
        public void run() {
            source.subscribe(parent);
        }
    }
}
```
在订阅时会将后续Observer封装成SubscribeOnObserver(parent字段)，并且将上游Observable(也就是source字段)订阅parent的操作封装成SubscribeTask(Runnable)，最后使用调度器scheduler安排Runnable的执行，也就是将上游Observable的订阅操作放到一个线程中执行，Observable将在那个线程产生数据并发射给Observer；

如果出现多个subscribeOn的情形，将在第一个出现的subscibeOn所指定的调度器中执行，因为多个操作符拼接的过程本质是封装Observable的过程，自上而下，新调用的操作符会封装上一个Observable，产生新的Observable，直到最终调用subscribe(observer)方法，而调用subscribe方法的过程本质是封装Observer的过程，自下而上，依次调用前面封装的Observable的subscribeActual方法，其中会封装Observer，最终顶层的Observable看到的是一层一层封装的Observer，所以数据会流经各个Observer（外层Observer的onNext调用里层Observer的onNext），所以第一个subscribeOn封装的ObservableSubscribeOn的subscribeActual最后调用，最终订阅操作也就在其指定的调度器中执行，当然如果出现多个subscribeOn中夹杂observeOn，那就以这个observeOn为分割线；

再看下observeOn，首先封装原Observable为ObservableObserveOn
``` java
public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
	ObjectHelper.requireNonNull(scheduler, "scheduler is null");
	ObjectHelper.verifyPositive(bufferSize, "bufferSize");
	return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
}
```
在ObservableObserveOn中，会将Observer封装成ObserveOnObserver，目的是在收到上游数据后先缓存，并在新的线程中将数据转发给下游Observer，即下游Observer就在调度器指定的线程中接收到数据
``` java
public final class ObservableObserveOn<T> extends AbstractObservableWithUpstream<T, T> {
    final Scheduler scheduler;
    final boolean delayError;
    final int bufferSize;
    public ObservableObserveOn(ObservableSource<T> source, Scheduler scheduler, boolean delayError, int bufferSize) {
        super(source);
        this.scheduler = scheduler;
        this.delayError = delayError;
        this.bufferSize = bufferSize;
    }

    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        if (scheduler instanceof TrampolineScheduler) {
            source.subscribe(observer);
        } else {
            Scheduler.Worker w = scheduler.createWorker();

            source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
        }
    }

    static final class ObserveOnObserver<T> extends BasicIntQueueDisposable<T>
    implements Observer<T>, Runnable {

        private static final long serialVersionUID = 6576896619930983584L;
        final Observer<? super T> downstream;
        final Scheduler.Worker worker;
        final boolean delayError;
        final int bufferSize;

        SimpleQueue<T> queue;

        Disposable upstream;

        Throwable error;
        volatile boolean done;

        volatile boolean disposed;

        int sourceMode;

        boolean outputFused;

        ObserveOnObserver(Observer<? super T> actual, Scheduler.Worker worker, boolean delayError, int bufferSize) {
            this.downstream = actual;
            this.worker = worker;
            this.delayError = delayError;
            this.bufferSize = bufferSize;
        }

        @Override
        public void onSubscribe(Disposable d) {
            if (DisposableHelper.validate(this.upstream, d)) {
                this.upstream = d;
                if (d instanceof QueueDisposable) {
                    @SuppressWarnings("unchecked")
                    QueueDisposable<T> qd = (QueueDisposable<T>) d;

                    int m = qd.requestFusion(QueueDisposable.ANY | QueueDisposable.BOUNDARY);

                    if (m == QueueDisposable.SYNC) {
                        sourceMode = m;
                        queue = qd;
                        done = true;
                        downstream.onSubscribe(this);
                        schedule();
                        return;
                    }
                    if (m == QueueDisposable.ASYNC) {
                        sourceMode = m;
                        queue = qd;
                        downstream.onSubscribe(this);
                        return;
                    }
                }

                queue = new SpscLinkedArrayQueue<T>(bufferSize);

                downstream.onSubscribe(this);
            }
        }

        @Override
        public void onNext(T t) {
            if (done) {
                return;
            }

            if (sourceMode != QueueDisposable.ASYNC) {
                queue.offer(t);
            }
            schedule();
        }

        @Override
        public void onError(Throwable t) {
            if (done) {
                RxJavaPlugins.onError(t);
                return;
            }
            error = t;
            done = true;
            schedule();
        }

        @Override
        public void onComplete() {
            if (done) {
                return;
            }
            done = true;
            schedule();
        }

        @Override
        public void dispose() {
            if (!disposed) {
                disposed = true;
                upstream.dispose();
                worker.dispose();
                if (getAndIncrement() == 0) {
                    queue.clear();
                }
            }
        }

        @Override
        public boolean isDisposed() {
            return disposed;
        }

        void schedule() {
            if (getAndIncrement() == 0) {
                worker.schedule(this);
            }
        }

        void drainNormal() {
            int missed = 1;

            final SimpleQueue<T> q = queue;
            final Observer<? super T> a = downstream;

            for (;;) {
                if (checkTerminated(done, q.isEmpty(), a)) {
                    return;
                }

                for (;;) {
                    boolean d = done;
                    T v;

                    try {
                        v = q.poll();
                    } catch (Throwable ex) {
                        Exceptions.throwIfFatal(ex);
                        disposed = true;
                        upstream.dispose();
                        q.clear();
                        a.onError(ex);
                        worker.dispose();
                        return;
                    }
                    boolean empty = v == null;

                    if (checkTerminated(d, empty, a)) {
                        return;
                    }

                    if (empty) {
                        break;
                    }

                    a.onNext(v);
                }

                missed = addAndGet(-missed);
                if (missed == 0) {
                    break;
                }
            }
        }

        void drainFused() {
            int missed = 1;

            for (;;) {
                if (disposed) {
                    return;
                }

                boolean d = done;
                Throwable ex = error;

                if (!delayError && d && ex != null) {
                    disposed = true;
                    downstream.onError(error);
                    worker.dispose();
                    return;
                }

                downstream.onNext(null);

                if (d) {
                    disposed = true;
                    ex = error;
                    if (ex != null) {
                        downstream.onError(ex);
                    } else {
                        downstream.onComplete();
                    }
                    worker.dispose();
                    return;
                }

                missed = addAndGet(-missed);
                if (missed == 0) {
                    break;
                }
            }
        }

        @Override
        public void run() {
            if (outputFused) {
                drainFused();
            } else {
                drainNormal();
            }
        }

        boolean checkTerminated(boolean d, boolean empty, Observer<? super T> a) {
            if (disposed) {
                queue.clear();
                return true;
            }
            if (d) {
                Throwable e = error;
                if (delayError) {
                    if (empty) {
                        disposed = true;
                        if (e != null) {
                            a.onError(e);
                        } else {
                            a.onComplete();
                        }
                        worker.dispose();
                        return true;
                    }
                } else {
                    if (e != null) {
                        disposed = true;
                        queue.clear();
                        a.onError(e);
                        worker.dispose();
                        return true;
                    } else
                    if (empty) {
                        disposed = true;
                        a.onComplete();
                        worker.dispose();
                        return true;
                    }
                }
            }
            return false;
        }

        @Override
        public int requestFusion(int mode) {
            if ((mode & ASYNC) != 0) {
                outputFused = true;
                return ASYNC;
            }
            return NONE;
        }

        @Nullable
        @Override
        public T poll() throws Exception {
            return queue.poll();
        }

        @Override
        public void clear() {
            queue.clear();
        }

        @Override
        public boolean isEmpty() {
            return queue.isEmpty();
        }
    }
}
```
所以Observable最本质的调用过程就是top-level Observable将数据发射给各个操作符创建的Observer，直至bottom-level Observer；
