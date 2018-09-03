---
layout: post
title: 使用RxJava防止button快速点击
date: 2017-04-13
categories:
- android
tags:
- RxJava
---

项目中很多场景需要防止重复点击，比如提交订单等与支付相关的操作时，要避免重复点击重复提交表单，使用RxJava可以优雅的实现
<!-- more -->
### 传统实现方式
以往我们有很多实现方式，比较多的是使用系统时间比较的方式，如下代码
``` java
public abstract class OnMultiClickListener implements View.OnClickListener{
    // 两次点击按钮之间的点击间隔不能少于1000毫秒
    private static final int MIN_CLICK_DELAY_TIME = 1000;
    private static long lastClickTime;

    public abstract void onMultiClick(View v);

    @Override
    public void onClick(View v) {
        long curClickTime = System.currentTimeMillis();
        if((curClickTime - lastClickTime) >= MIN_CLICK_DELAY_TIME) {
            // 超过点击间隔后再将lastClickTime重置为当前点击时间
            lastClickTime = curClickTime;
            onMultiClick(v);
        }
    }
}
```
### 使用RxJava
我们使用throttleFirst操作符可以实现，代码
``` java
Disposable d = RxView.clicks(tv).throttleFirst(1, TimeUnit.SECONDS).subscribe(new Consumer<Object>() {
    @Override
    public void accept(Object o) throws Exception {
    // do something
    }
});
```
### throttleFirst作用
发射原Observable的数据，并启动一个时间窗口，在这个时间窗口内的其他数据将被忽略
![](https://raw.github.com/wiki/ReactiveX/RxJava/images/rx-operators/throttleFirst.png)
使用和作用很好懂，我们看下实现加深理解

### throttleFirst源码分析
首先throttleFirst将原Observable分装成ObservableThrottleFirstTimed对象
``` java
public final Observable<T> throttleFirst(long windowDuration, TimeUnit unit) {
    return throttleFirst(windowDuration, unit, Schedulers.computation());
}
public final Observable<T> throttleFirst(long skipDuration, TimeUnit unit, Scheduler scheduler) {
    ObjectHelper.requireNonNull(unit, "unit is null");
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    return RxJavaPlugins.onAssembly(new ObservableThrottleFirstTimed<T>(this, skipDuration, unit, scheduler));
}
```
当Observable发生订阅时会调用ObservableThrottleFirstTimed的subscribeActual, 其中只不过是封装了Observer为DebounceTimedObserver
``` java
public final class ObservableThrottleFirstTimed<T> extends AbstractObservableWithUpstream<T, T> {
    final long timeout;
    final TimeUnit unit;
    final Scheduler scheduler;

    public ObservableThrottleFirstTimed(ObservableSource<T> source,
            long timeout, TimeUnit unit, Scheduler scheduler) {
        super(source);
        this.timeout = timeout;
        this.unit = unit;
        this.scheduler = scheduler;
    }

    @Override
    public void subscribeActual(Observer<? super T> t) {
        source.subscribe(new DebounceTimedObserver<T>(
                new SerializedObserver<T>(t),
                timeout, unit, scheduler.createWorker()));
    }

    static final class DebounceTimedObserver<T>
    extends AtomicReference<Disposable>
    implements Observer<T>, Disposable, Runnable {
        private static final long serialVersionUID = 786994795061867455L;

        final Observer<? super T> actual;
        final long timeout;
        final TimeUnit unit;
        final Scheduler.Worker worker;

        Disposable s;

        volatile boolean gate;

        boolean done;

        DebounceTimedObserver(Observer<? super T> actual, long timeout, TimeUnit unit, Worker worker) {
            this.actual = actual;
            this.timeout = timeout;
            this.unit = unit;
            this.worker = worker;
        }

        @Override
        public void onSubscribe(Disposable s) {
            if (DisposableHelper.validate(this.s, s)) {
                this.s = s;
                actual.onSubscribe(this);
            }
        }

        @Override
        public void onNext(T t) {
            if (!gate && !done) {
                gate = true;

                actual.onNext(t);

                Disposable d = get();
                if (d != null) {
                    d.dispose();
                }
                DisposableHelper.replace(this, worker.schedule(this, timeout, unit));
            }


        }

        @Override
        public void run() {
            gate = false;
        }

        @Override
        public void onError(Throwable t) {
            if (done) {
                RxJavaPlugins.onError(t);
            } else {
                done = true;
                DisposableHelper.dispose(this);
                actual.onError(t);
            }
        }

        @Override
        public void onComplete() {
            if (!done) {
                done = true;
                DisposableHelper.dispose(this);
                worker.dispose();
                actual.onComplete();
            }
        }

        @Override
        public void dispose() {
            DisposableHelper.dispose(this);
            worker.dispose();
            s.dispose();
        }

        @Override
        public boolean isDisposed() {
            return DisposableHelper.isDisposed(get());
        }
    }
}
```
当上游Observable给DebounceTimedObserver发射数据时，onNext方法会回调
``` java
@Override
public void onNext(T t) {
	if (!gate && !done) {
		gate = true;
		actual.onNext(t);
		Disposable d = get();
		if (d != null) {
			d.dispose();
		}
		DisposableHelper.replace(this, worker.schedule(this, timeout, unit));
	}
}
```
只是在给下游Observer发射数据之后设置开关gate=true，同时启动一个延迟任务, 延迟任务中会将开关关闭gate=false
``` java
@Override
public void run() {
	gate = false;
}
```
就这么简单。
