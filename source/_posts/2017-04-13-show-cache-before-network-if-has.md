---
layout: post
title: 使用RxJava协同加载本地缓存与网络数据
date: 2017-04-13
categories:
- android
tags:
- RxJava
---

项目中我们常常遇到这种情况：首页上面的数据做了缓存，首次打开app时，若存在缓存，则先加载缓存，同时拉取网络数据，加载完毕后界面再显示网络数据，如何实现呢？
<!-- more -->

### 分析
显然我们应该有两个Observable，一个是cacheObservable用于从缓存中获取数据，另一个是networkObservable用于从网络加载数据，我们应该先从cache获取数据再从network获取数据，一般情况下前者快于后者，但也有异常情况，network快于cache，也有可能cache失败了，所以正确的策略应该是：若cache先返回则加载cache数据，再加载network数据，若network先返回则加载network数据，此时忽略cache数据；

### 实现
我们可以组合使用publish + merge + takeUntil三个操作符，先看代码：
``` java
final Observable<DataModel> cacheObservable = getCacheObservable();
final Observable<DataModel> networkObservable = getNetworkObservable();
SimpleDisposableObserver disposableObserver =
        networkObservable.publish(new Function<Observable<DataModel>, ObservableSource<DataModel>>() {
            @Override
            public ObservableSource<DataModel> apply(@NonNull Observable<DataModel> localLifeModelObservable) throws Exception {
                return Observable.merge(localLifeModelObservable, 
                        cacheObservable.takeUntil(localLifeModelObservable));
            }
        })
        .observeOn(AndroidSchedulers.mainThread())
        .subscribeWith(new SimpleDisposableObserver<DataModel>(mView) {
            @Override
            public void onNext(DataModel localLifeModel) {
                mView.showContent(localLifeModel);
            }

            @Override
            public void onError(Throwable e) {
                super.onError(e);
                mView.showWhenError();
            }
        });

addSubscribe(disposableObserver);
```

### publish操作符
将原Observable转换成一个可连接的Observable，接收一个Function作为参数，目的是转换原Observable的数据，实际是一个map操作，将转换后的数据分发给后续订阅的Observer
![](http://reactivex.io/documentation/operators/images/publishConnect.c.png)
### merge操作符
将多个Observable合并成一个Observable，不做任何转换，合并多个Observable发射的数据，就像这些数据是同一个Observable发出的一样
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/merge.c.png)
### takeUntil操作符
发射原Observable的数据，直到第二个Observable发射一个数据或停止发射或发射一个错误为止
![](https://raw.github.com/wiki/ReactiveX/RxJava/images/rx-operators/takeUntil.png)
### 总结
``` java
cacheObservable.takeUntil(networkObservable);
```
所以上面的代码的作用是：若cacheObservable先于networkObservable发射数据，则取cacheObservable发射的数据，若networkObservable先于cacheObservable发射数据，则取消cacheObservable，不发射数据
``` java
Observable.merge(networkObservable, 
        cacheObservable.takeUntil(networkObservable));
```
merge后的作用是：若cacheObservable先于networkObservable发射数据，则先取cacheObservable发射的数据，后取networkObservable发射的数据，若networkObservable先于cacheObservable发射数据，则只取networkObservable发射的数据
``` java
networkObservable.publish(new Function<Observable<DataModel>, ObservableSource<DataModel>>() {
    @Override
    public ObservableSource<DataModel> apply(@NonNull Observable<DataModel> localLifeModelObservable) throws Exception {
        return Observable.merge(localLifeModelObservable, 
                cacheObservable.takeUntil(localLifeModelObservable));
    }
});
```