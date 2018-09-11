---
layout: post
title: Limits of AsyncTask
date: 2016-02-29
categories:
- android
tags:
- AsyncTask
---
AsyncTask是android处理异步操作的工具类，使用也很方便，但存在一些限制，总结如下
<!--more-->

## AsyncTask使用限制：

* 1、AsyncTask类必须在主线程中加载，因为其中使用了Handler来postResult和publishProgress，Handler需在UI线程中创建；

* 2、AsyncTask实例需在主线程中创建；

* 3、execute方法需在主线程中调用；

* 4、不要直接调用onPreExecute、doInBackground、onProgressUpdate、onPostExecute方法；

* 5、一个任务只能被执行一次；

* 6、在Android1.6以前AsyncTask使用单线程串行执行任务，Android3.0以前使用线程池并行执行任务（默认5个并发，最多128个），Android3.0之后为避免并发错误又改用使用单线程串行执行任务，不过这个线程是在线程池中执行的，若需要并发执行任务，可以使用executeOnExecutor配合AsyncTask.THREAD_POOL_EXECUTOR或自定义线程池；

## AsyncTask工作原理（基于android-21）：
首先从execute方法开始，execute方法调用了executeOnExecutor，代码如下：
``` java
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}

public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
    if (mStatus != Status.PENDING) {
        switch (mStatus) {
            case RUNNING:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task is already running.");
            case FINISHED:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task has already been executed "
                        + "(a task can be executed only once)");
        }
    }

    mStatus = Status.RUNNING;

    onPreExecute();

    mWorker.mParams = params;
    exec.execute(mFuture);

    return this;
}
```

sDefaultExecutor(SerialExecutor)是一个串行的线程池，用于串行的执行任务，执行任务之前判断是否已经执行，所以一个任务只能执行一次，之后又调用了onPreExecute方法，所以这个方法是在UI线程中调用的，接着看SerialExecutor.execute方法：
``` java
private static class SerialExecutor implements Executor {
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
    Runnable mActive;

    public synchronized void execute(final Runnable r) {
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                    r.run();
                } finally {
                    scheduleNext();
                }
            }
        });
        if (mActive == null) {
            scheduleNext();
        }
    }

    protected synchronized void scheduleNext() {
        if ((mActive = mTasks.poll()) != null) {
            THREAD_POOL_EXECUTOR.execute(mActive);
        }
    }
}
```

其中会将mFuture(FutrueTask)包装成Runnale添加到mTasks的尾部，判断若没有任务在执行，就开始执行第一个任务，执行的任务是通过线程池来执行的，THREAD_POOL_EXECUTOR是AsyncTask中创建的线程池，从包装的Runnale中可以看出，只有一个任务执行完成之后才会执行下一个，所以AsyncTask默认是串行执行任务的，分析一下mFuture，它是在构造函数中创建的：
``` java
public AsyncTask() {
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            mTaskInvoked.set(true);

            Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
            //noinspection unchecked
            return postResult(doInBackground(mParams));
        }
    };

    mFuture = new FutureTask<Result>(mWorker) {
        @Override
        protected void done() {
            try {
                postResultIfNotInvoked(get());
            } catch (InterruptedException e) {
                android.util.Log.w(LOG_TAG, e);
            } catch (ExecutionException e) {
                throw new RuntimeException("An error occured while executing doInBackground()",
                        e.getCause());
            } catch (CancellationException e) {
                postResultIfNotInvoked(null);
            }
        }
    };
}
```

FutureTask的run方法最终会调用WorkerRunnable的call方法，当然这些方法是在线程池中执行的，即在子线程中执行的，在doInBackground执行完了之后，通过postResult将执行的结果发送给主线程，
``` java
private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = sHandler.obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}
```

sHandler是AsyncTask中创建的静态的Handler，所以AsyncTask需在ActivityThread中加载
``` java
private static class InternalHandler extends Handler {
    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult result = (AsyncTaskResult) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                // There is only one result
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
                result.mTask.onProgressUpdate(result.mData);
                break;
        }
    }
}
```

得到Result之后会调用finish方法，在finish中会将result传给onPostExecute做收尾工作，所以onPostExecute是在UI线程中执行的
``` java
private void finish(Result result) {
    if (isCancelled()) {
        onCancelled(result);
    } else {
        onPostExecute(result);
    }
    mStatus = Status.FINISHED;
}
```

分析完成。
