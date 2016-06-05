android的异步任务体系中还有一个非常重要的操作类：AsyncTask，其内部主要使用的是java的线程池和Handler来实现异步任务以及与UI线程的交互。本文主要解析AsyncTask的的使用与源码。

首先我们来看一下AsyncTask的基本使用：

```
class MAsyncTask extends AsyncTask<Integer, Integer, Integer> {
        @Override
        protected void onPreExecute() {
            super.onPreExecute();
            Log.i(TAG, "onPreExecute...(开始执行后台任务之前)");
        }

        @Override
        protected void onPostExecute(Integer i) {
            super.onPostExecute(i);
            Log.i("TAG", "onPostExecute...(开始执行后台任务之后)");
        }

        @Override
        protected Integer doInBackground(Integer... params) {
            Log.i(TAG, "doInBackground...(开始执行后台任务)");
            return 0;
        }
    }
```
我们定义了自己的MAsyncTask并继承自AsyncTask；并重写了其中的是哪个回调方法：onPreExecute()，onPostExecute（），doInBackground();
然后开始调用异步任务：

```
new MAsyncTask().execute();
```

好了，下面我们开始分析异步任务的执行过程，首先查看一下异步任务的构造方法：

```
/**
     * Creates a new asynchronous task. This constructor must be invoked on the UI thread.
     */
    public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                Result result = doInBackground(mParams);
                Binder.flushPendingCommands();
                return postResult(result);
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
    }
```
咋一看AsyncTask的构造方法代码量还是比较多的，但是仔细一看其实这里面只是初始化了两个成员变量：mWorker和mFuture他们分别是：WorkerRunnable和FutureTask，熟悉java的童鞋应该知道这两个类其实是java里面线程池先关的概念。其具体用法大家可以在网上查询，这里具体的细节不在表述，重点是对异步任务整体流程的把握。

**总结：异步任务的构造方法主要用于初始化线程池先关的成员变量。**

接下来我们看一下execute方法：

```
@MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
```
这里发现该方法中添加一个@MainThread的注解，通过该注解，可以知道我们在执行AsyncTask的execute方法时，只能在主线程中执行，这里可以实验一下：

```
new Thread(new Runnable() {
                    @Override
                    public void run() {
                        Log.i("tag", Thread.currentThread().getId() + "");
                        new MAsyncTask().execute();
                    }
                }).start();
                Log.i("tag", "mainThread:" + Thread.currentThread().getId() + "");
```
然后执行，但是并没有什么区别，程序还是可以正常执行，我的手机的Android系统是Android5.0，具体原因尚未找到，欢迎有知道答案的童鞋可以相互沟通哈。但是这里需要主要的一个问题是：onPreExecute方法是与开始执行的execute方法是在同一个线程中的，所以如果在子线程中执行execute方法，一定要确保onPreExecute方法不执行刷新UI的方法，否则：

```
@Override
        protected void onPreExecute() {
            super.onPreExecute();
            title.setText("########");
            Log.i(TAG, "onPreExecute...(开始执行后台任务之前)");
        }
```

```
Process: com.example.aaron.helloworld, PID: 659
                                                                           android.view.ViewRootImpl$CalledFromWrongThreadException: Only the original thread that created a view hierarchy can touch its views.
                                                                               at android.view.ViewRootImpl.checkThread(ViewRootImpl.java:6981)
                                                                               at android.view.ViewRootImpl.requestLayout(ViewRootImpl.java:1034)
                                                                               at android.view.View.requestLayout(View.java:17704)
                                                                               at android.view.View.requestLayout(View.java:17704)
                                                                               at android.view.View.requestLayout(View.java:17704)
                                                                               at android.view.View.requestLayout(View.java:17704)
                                                                               at android.widget.RelativeLayout.requestLayout(RelativeLayout.java:380)
                                                                               at android.view.View.requestLayout(View.java:17704)
                                                                               at android.widget.TextView.checkForRelayout(TextView.java:7109)
                                                                               at android.widget.TextView.setText(TextView.java:4082)
                                                                               at android.widget.TextView.setText(TextView.java:3940)
                                                                               at android.widget.TextView.setText(TextView.java:3915)
                                                                               at com.example.aaron.helloworld.MainActivity$MAsyncTask.onPreExecute(MainActivity.java:53)
                                                                               at android.os.AsyncTask.executeOnExecutor(AsyncTask.java:587)
                                                                               at android.os.AsyncTask.execute(AsyncTask.java:535)
                                                                               at com.example.aaron.helloworld.MainActivity$1$1.run(MainActivity.java:40)
                                                                               at java.lang.Thread.run(Thread.java:818)
```
若在子线程中执行execute方法，那么这时候如果在onPreExecute方法中刷新UI，会报错，即子线程中不能更新UI。

继续看刚才的execute方法，我们可以发现其内部调用了executeOnExecutor方法：

```
@MainThread
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
可以看到其具体的内部实现方法里：首先判断当前异步任务的状态，其内部保存异步任务状态的成员变量mStatus的默认值为Status.PENDING,所以第一次执行的时候并不抛出这两个异常，那么什么时候回进入这个if判断并抛出异常呢，通过查看源代码可以知道，当我们执行了execute方法之后，如果再次执行就会进入这里的if条件判断并抛出异常，这里可以尝试一下：

```
final MAsyncTask mAsyncTask = new MAsyncTask();
        title.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                /*MLog.e("you have clicked the title textview!!!");
                Intent intent = new Intent(MainActivity.this, SecondActivity.class);
                startActivityForResult(intent, 101);*/


                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        Log.i("tag", Thread.currentThread().getId() + "");
                        mAsyncTask
                                .execute();
                    }
                }).start();
                Log.i("tag", "mainThread:" + Thread.currentThread().getId() + "");

            }
        });
```
这里我们可以看到我们定义了一个AsyncTask的对象，并且每次执行点击事件的回调方法都会执行execute方法，当我们点击第一次的时候程序正常执行，但是当我们执行第二次的时候，程序就崩溃了。若这时候第一次执行的异步任务尚未执行完成则会抛出异常：

```
Cannot execute task:the task is already running.
```
若第一次执行的异步任务已经执行完成，则会抛出异常：

```
Cannot execute task:the task has already been executed (a task can be executed only once)
```

继续往下看，在executeOnExecutor中若没有进入异常分之，则将当前异步任务的状态更改为Running，然后回调onPreExecute()方法，这里可以查看一下onPreExecute方法其实是一个空方法，主要就是为了用于我们的回调实现，同时这里也说明了onPreExecute（）方法是与execute方法的执行在同一线程中。

然后将execute方法的参数赋值给mWorker对象那个，最后执行exec.execute(mFuture)方法，并返回自身。

这里我们重点看一下exec.execute(mFuture)的具体实现，这里的exec其实是AsyncTask定义的一个默认的Executor对象：

```
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
```
那么，SERIAL_EXECUTOR又是什么东西呢？

```
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
```
继续查看SerialExecutor的具体实现：

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
可以发现其继承Executor类其内部保存着一个Runnable列表，即任务列表，在刚刚的execute方法中执行的exec.execute(mFuture)方法就是执行的这里的execute方法。
这里具体看一下execute方法的实现：
1）首先调用的是mTasks的offer方法，即将异步任务保存至任务列表的队尾
2）判断mActive对象是不是等于null，第一次运行是null，然后调用scheduleNext()方法
3）在scheduleNext()这个方法中会从队列的头部取值，并赋值给mActive对象，然后调用THREAD_POOL_EXECUTOR去执行取出的取出的Runnable对象。
4）在这之后如果再有新的任务被执行时就等待上一个任务执行完毕后才会得到执行，所以说同一时刻只会有一个线程正在执行。
5）这里的THREAD_POOL_EXECUTOR其实是一个线程池对象。

然后我们看一下执行过程中mWorker的执行逻辑：

```
mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                Result result = doInBackground(mParams);
                Binder.flushPendingCommands();
                return postResult(result);
            }
        };
```
可以看到在执行线程池的任务时，我们回调了doInBackground方法，这也就是我们重写AsyncTask时重写doInBackground方法是后台线程的原因。

然后在任务执行完毕之后会回调我们的done方法：

```
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

这里我们具体看一下postResultIfNotInvoked方法：

```
private void postResultIfNotInvoked(Result result) {
        final boolean wasTaskInvoked = mTaskInvoked.get();
        if (!wasTaskInvoked) {
            postResult(result);
        }
    }
```
其内部还是调用了postResult方法：

```
private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
```
这里可以看到起调用了内部的Handler对象的sendToTarget方法，发送异步消息，具体handler相关的内容可以参考：
<a href="http://blog.csdn.net/qq_23547831/article/details/50751687"> android源码解析之（二）-->异步消息机制</a>

追踪代码，可以查看AsyncTask内部定义了一个Handler对象：

```
 private static class InternalHandler extends Handler {
        public InternalHandler() {
            super(Looper.getMainLooper());
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
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
可以看到起内部的handleMessage方法，有两个处理逻辑，分别是：更新进入条和执行完成，这里的更新进度的方法就是我们重写AsyncTask方法时重写的更新进度的方法，这里的异步任务完成的消息会调用finish方法：

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
这里AsyncTask首先会判断当前任务是否被取消，若被取消的话则直接执行取消的方法，否则执行onPostExecute方法，也就是我们重写AsyncTask时需要重写的异步任务完成时回调的方法。

其实整个异步任务的大概流程就是这样子的，其中涉及的知识点比较多，这里总结一下：

- 异步任务内部使用线程池执行后台任务，使用Handler传递消息；

- onPreExecute方法主要用于在异步任务执行之前做一些操作，它所在线程与异步任务的execute方法所在的线程一致，这里若需要更新UI等操作，则execute方法不能再子线程中执行。

- 通过刚刚的源码分析可以知道异步任务一般是顺序执行的，即一个任务执行完成之后才会执行下一个任务。

- doInBackground这个方法所在的进程为任务所执行的进程，在这里可以进行一些后台操作。

- 异步任务执行完成之后会通过一系列的调用操作，最终回调我们的onPostExecute方法

- 异步任务对象不能执行多次，即不能创建一个对象执行多次execute方法。（通过execute方法的源码可以得知）

- 所有源码基于android23，中间有什么疏漏欢迎指正。


另外对android源码解析方法感兴趣的可参考我的：
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50634435"> android源码解析之（一）-->android项目构建过程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50751687">android源码解析之（二）-->异步消息机制</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50803849">android源码解析之（三）-->异步任务AsyncTask</a>
