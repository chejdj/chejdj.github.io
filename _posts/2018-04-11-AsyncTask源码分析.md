---
title: AsyncTask源码分析
date: 2018-04-11 14:42:00
categories:
- Android
tags:
- Android
---

AsyncTask封装了线程池和Handler,它主要是为了方便我们在子线程当中去更新UI(大多数的情况下，是用它来更新进度条)，AsyncTask类并不适合做过多任务量的后台任务(过多任务量实际逻辑复杂度高)  
它主要有4个方法  
(1)onPreExecute()  主线程执行，在任务执行之前  
(2)doInBackground(Params...params) 执行任务  
(3)onProgressUpdate(Progress...values) 主线程执行，在doInBackground(Params...params)调用publicProgress通知任务执行进度  
(4)onPostExecute(Result result)主线程执行，任务执行后，返回的结果   

<!--more-->

#### 构造函数  
```
public AsyncTask(@Nullable Looper callbackLooper) {
      mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
      ? getMainHandler()
      : new Handler(callbackLooper);
```  

由这个构造函数可以得知，AsyncTask需要在UI线程当中，去实例化(不一定是UI线程，但是该线程必须要有Looper,handler需要绑定创建线程的Looper对象)  

```
mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    result = doInBackground(mParams); //调用了doInBackground(mParams)
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    postResult(result);
                }
                return result;
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
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
```  

初始化FutureTask对象和mWorker对象,mWorker作为参数传给了mFuture,可以看一下mWorker的call()方法，调用`doInBackground(mParams)`,然后调用`postResult(result)`最后返回结果。  

#### execute  
接下来我们看execute,执行函数，一般我们在创建AsyncTask对象的时候，调用execute执行  

```
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
}

public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
// Status.PENDING代表这个任务还没有执行，这个也是默认的初始值
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
        onPreExecute(); //任务执行之前调用
        mWorker.mParams = params;
        exec.execute(mFuture);//sDefaultExecutor 这个是传入的参数，类型为SerialExecutor
        return this; //传入的参数为mFuture  FutureTask类型
}
```  

这里是执行的过程，可以看到先进行了Status的判断，如果不是PENDING状态就会报错，我们可以这样理解，一个任务对应了一个AysncTask,只能执行一次，如果对同一个任务执行两次，第二次就会报错,不执行，接下来我们看一下这个`exec.execute(mFuture)`这个执行方法，`exec`默认为SerialExecutor类型。  

```
private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {   //把该FutureTask放入mTasks队列中
                public void run() {
                    try {
                        r.run();  //执行mFuture 的run方法
                    } finally {
                        scheduleNext();  
                    }
                }
            });
            if (mActive == null) { //注释1
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
我们可以看到这个SerialExecutor线程池，是一个线性执行的线程池，一次只能允许执行一个任务，`mActive`代表当前活跃的任务，只有在null的时候才会才会从mTasks队列中取出来执行。最后交给THREAD_POOL_EXECUTOR去执行。  
```
ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                sPoolWorkQueue, sThreadFactory);
        threadPoolExecutor.allowCoreThreadTimeOut(true);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
```  

这里贴一下THREAD_POOL_EXECUTOR初始化部分，可以看到就是一个正常的线程池，而SerialExecutor线程池存在的目的只是为了让任务一次只执行一个任务，所以说默认是一个任务一个任务接着执行的，如果想要并发，我们就需要我们自己构建一个线程池去执行他了。        

```
//再贴一下mWorker方法
mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    result = doInBackground(mParams);//执行doInBackground(mParams)
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    postResult(result);
                }
                return result;
            }
        };

//postRsult通过handler发送信息给主线程，调用onPostExecute
private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
}
//更新进度的代码
protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
}
```  
我们可以看到其实AsyncTask实现线程之间的转换，还是通过Handler去实现的，`postResult`和`publishProgress`都是，它会默认获取主线程的Handler。  
最后一张图总结一下  
![流程图](https://github.com/chejdj/chejdj.github.io/raw/master/assets/blog_image/2018-04-11/asynctask.png)

#### FutureTask  
看到这里，你会发现传入线程的是FutureTask类型，而不是我们熟悉的Runnable对象（FutureTask中又引入了Callable对象）其实FutureTask实现了RunnableFuture接口，而RunnableFuture有继承了Runnable,Future接口，接口也是可以向上转型，也就可以充当runnable执行，这里介绍一下Future  
**Future接口**  
Future接口代表异步计算的结果，通过Future接口提供的方法可以查看异步计算是否执行完成，或者等待执行结果并获取执行结果，同时还可以取消执行。Future接口的定义如下:
```
public interface Future<V> {
 boolean cancel(boolean mayInterruptIfRunning);
 boolean isCancelled();
 boolean isDone();
V get() throws InterruptedException, ExecutionException;
V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```  
**cancel()**:cancel()方法用来取消异步任务的执行。如果异步任务已经完成或者已经被取消，或者由于某些原因不能取消，则会返回false。如果任务还没有被执行，则会返回true并且异步任务不会被执行。如果任务已经开始执行了但是还没有执行完成，若mayInterruptIfRunning为true，则会立即中断执行任务的线程并返回true，若mayInterruptIfRunning为false，则会返回true且不会中断任务执行线程。  
**isCanceled()**:判断任务是否被取消，如果任务在结束(正常执行结束或者执行异常结束)前被取消则返回true，否则返回false。  
**isDone()**:判断任务是否已经完成，如果完成则返回true，否则返回false。需要注意的是：任务执行过程中发生异常、任务被取消也属于任务已完成，也会返回true。  
**get()**:获取任务执行结果，如果任务还没完成则会阻塞等待直到任务执行完成。如果任务被取消则会抛出CancellationException异常，如果任务执行过程发生异常则会抛出ExecutionException异常，如果阻塞等待过程中被中断则会抛出InterruptedException异常。  
**get(long timeout,Timeunit unit)**:带超时时间的get()版本，如果阻塞等待过程中超时则会抛出TimeoutException异常。  
我们的AsyncTask的任务也是可以被取消，传递任务执行结果，有没有很像的感觉。  
