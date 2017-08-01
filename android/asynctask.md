```
android在新版本中不允许UI线程访问网络；在android中UI线程中不能执行耗时太长的任务，否则会引发ANR。



不同线程间的通信，可以选择Handler或者AsyncTask解决线程间通信问题。这里主要介绍AsyncTask。
```

AsyncTask是一个抽象类含有三个参数， AsyncTask&lt;Params, Progress, Result&gt;，根据参数的名字可以知道分别为：输入参数，进度，与返回结果，如果你要使用它必须要集成他，或者采用匿名类的方式。

## Samples：

```
AsyncTask<Integer, Void, Integer> task = new AsyncTask<Integer, Void, Integer>() {

            @Override
            protected Integer doInBackground(Integer... params) {
                return params[0] * params[0];
            }

            @Override
            protected void onProgressUpdate(Void... values) {
                super.onProgressUpdate(values);
            }

            @Override
            protected void onCancelled() {
                super.onCancelled();
            }

            @Override
            protected void onPreExecute() {
                super.onPreExecute();
            }

            @Override
            protected void onPostExecute(Integer result) {
                Toast.makeText(AsyncTaskActivity.this, "Current Value = " + result, Toast.LENGTH_LONG).show();
                setTextValue("Current Value =" + result);
            }
        };
        if (Build.VERSION.SDK_INT >= 11) {
            task.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR,value);
        } else {
            task.execute(value);
        }
```

1： protected Integer doInBackground\(Integer... params\)是必须要重写的，因为它是在后台执行的，耗时逻辑和网络访问都可以放到这个里面，其中返回类型就是AsyncTask中的第三个参数的类型，输入参数是可变参数类型，与AsyncTask第一个参数保持一致，如果没有参数可以采用Void代替，V大写。

2：void onPreExecute\(\)  这个表示在后台线程执行之前，UI可以执行的操作，一般用来开始进度条，或者进行提醒。

3：void onProgressUpdate\(Void... values\) ，根据名字也可以看出是调整进度条的，输入的参数与AsyncTask第二个参数一致，用来在UI界面更新进度条。

4： void onPostExecute\(Integer result\) ，该函数表示执行完毕后，返回的内容，已经在UI线程执行了，可以用来更新界面。

最后来看执行的部分加入了一个判断，这是因为android刚推出AsyncTask的时候，采用的单线程方式，前一个执行完了后再执行下一个，后来在2.x版本中改为了多个线程，到3.0版本时又改为了一个线程。为了防止阻塞，在3.0以后的版本中指定执行的Executor。

## AsyncTask源码分析（Android4.4.2）

### AsyncTask内部通信依然使用Handler

```
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

    private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
        Params[] mParams;
    }

    @SuppressWarnings({"RawUseOfParameterizedType"})
    private static class AsyncTaskResult<Data> {
        final AsyncTask mTask;
        final Data[] mData;

        AsyncTaskResult(AsyncTask task, Data... data) {
            mTask = task;
            mData = data;
        }
    }
```

handleMessage内部有两个分支语句，MESSAGE\_POST\_RESULT，可以看出这个是处理结果的 ， result.mTask.finish\(result.mData\[0\]\)调用了

```
   private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
```

MESSAGE\_POST\_PROGRESS这个是处理进度条的，result.mTask.onProgressUpdate\(result.mData\)，这个对应了AsyncTask中的进度函数。

### Message发出

```
private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = sHandler.obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
```

在此处我们看到发送了消息，而调用这个的地方有两处：

```
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

    private void postResultIfNotInvoked(Result result) {
        final boolean wasTaskInvoked = mTaskInvoked.get();
        if (!wasTaskInvoked) {
            postResult(result);
        }
    }
```

可以发现第一处有一个熟悉的地方，doInBackground（mParams），这个就是我们执行后台程序的地方，另一处在postResultIfNotInvoked中调用。

在FutureTask中的run函数中：

```
public void run() {
        if (state != NEW ||!UNSAFE.compareAndSwapObject(this, runnerOffset, null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;//Callable 
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```

可以看到有一个Callable，调用了callable 的call函数，这个Callable函数就是AsyncTask构造函数中的mWorker ，call函数就是mWorker 中的call函数，在此调用了doInBackground（）函数， 其它一处是在cancel中掉用的，cancel调用了done，所以就算取消了任务执行也会调用到onPostExecute（）。

#### 调用FutureTask的地方:

```
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

在这个地方我们看到了exec.execute（mFuture），这个mFuture就是构造函数中的FutureTask。而executeOnExecutor被  task.executeOnExecutor\(AsyncTask.THREAD\_POOL\_EXECUTOR,value\)与  task.execute（value）调用，而这就是页面上开始调用的地方。到此为止分析了返回结果整个的调用流程，而进度条流程要简单很多就不进行分析了。

### 不指定Excutor时，AsyncTask如何顺序执行

我们在task.execute的内部发现实际调用的是executeOnExecutor\(sDefaultExecutor, params\)，这里面的sDefaultExecutor为SERIAL\_EXECUTOR，而SERIAL\_EXECUTOR为：

```
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

当调用execute的时候，首先将Runnable添加到了一个队列的尾部，第一次执行的时候mActive肯定为null，因此调用scheduleNext，获取到对头的Runnable执行，当执行完毕后再finnally中获取下一个执行对象。这样就能串行进行执行了，因此当前一个任务阻塞后，也会导致后面的所有任务阻塞，因此在3.0后版本执行时指定Executor。

最后我们来整理一下调用流程，execute调用FutureTask中的run函数，在run函数中调用callable的call函数，在call函数中调用postResult函数，在此发送message到handler，在handler处理返回的结果。

## 常见问题

1）能否在其他线程中调用AsncTask？如果可以调用，它能否更新界面？如果能，为什么在新开的线程中可以更新界面？

A1：可以在其他线程中调用AsncTask。

A2：它能更新UI界面，下面我们详细说说为什么能更新UI界面，正常情况下，一个线程都对应了一个Hander，每个Hander对应一个Looper，我们上面新开了一个线程，如要使用Handler，就要调用Looper.prepare\(\)函数,因为UI线程在ActivityThread起来的时候已经调用了prepare函数,所以在程序中我们不需要再次调用,但是AsyncTask函数中并没有任何地方调用Looper.prepare\(\),同时它还能更新界面就说明了这个Looper就是使用的MainLooper。我们看到AsyncTask中有一个init函数

```
public static void init() {
        sHandler.getLooper();
    }
```

这个Looper就是MainLooper，但是init这个函数是在什么地方调用的呐？如果是MainLooper那应该就是在UI线程中调用的，那又在UI线程的什么地方进行调用的？

```
public static void main(String[] args) {
       SamplingProfilerIntegration.start();

       // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        Environment.initForCurrentUser();

        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());

        Security.addProvider(new AndroidKeyStoreProvider());

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        AsyncTask.init();

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
}
```

我们查看ActivityThread的main函数中调用了AsyncTask.init\(\)，由此可以知道确实是在UI线程中进行调用的，因此在其他线程中调用AsyncTask，最终是与UI线程进行通信的。因此也能对界面进行更改。

