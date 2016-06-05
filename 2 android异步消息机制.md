知乎上看了一篇非常不错的博文：<a href="http://zhuanlan.zhihu.com/kaede/20563936">有没有必要阅读ANDROID源码</a>
痛定思过，为了更好的深入android体系，决定学习android framework层源码，就从最简单的android异步消息机制开始吧。

**（一）Handler的常规使用方式**

```
public class MainActivity extends AppCompatActivity {

    public static final String TAG = MainActivity.class.getSimpleName();
    private TextView texttitle = null;

    /**
     * 在主线程中定义Handler，并实现对应的handleMessage方法
     */
    public static Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            if (msg.what == 101) {
                Log.i(TAG, "接收到handler消息...");
            }
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        texttitle = (TextView) findViewById(R.id.texttitle);
        texttitle.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                new Thread() {
                    @Override
                    public void run() {
                        // 在子线程中发送异步消息
                        mHandler.sendEmptyMessage(101);
                    }
                }.start();
            }
        });
    }
}
```

可以看出，一般handler的使用方式都是在主线程中定义Handler，然后在子线程中调用mHandler.sendEmptyMessage();方法，然么这里有一个疑问了，我们可以在子线程中定义Handler么？

**（二）如何在子线程中定义Handler？**

我们在子线程中定义Handler，看看结果:

```
texttitle.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                new Thread() {
                    @Override
                    public void run() {
                        Handler mHandler = new Handler() {
                            @Override
                            public void handleMessage(Message msg) {
                                if (msg.what == 101) {
                                    Log.i(TAG, "在子线程中定义Handler，并接收到消息。。。");
                                }
                            }
                        };
                    }
                }.start();
            }
        });
```
点击按钮并运行这段代码：
![这里写图片描述](http://img.blog.csdn.net/20160226180540827)

可以看出来在子线程中定义Handler对象出错了，难道Handler对象的定义或者是初始化只能在主线程中？
其实不是这样的，错误信息中提示的已经很明显了，在初始化Handler对象之前需要调用Looper.prepare()方法，那么好了，我们添加这句代码再次执行一次：

```
texttitle.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                new Thread() {
                    @Override
                    public void run() {
                        Looper.prepare();
                        Handler mHandler = new Handler() {
                            @Override
                            public void handleMessage(Message msg) {
                                if (msg.what == 101) {
                                    Log.i(TAG, "在子线程中定义Handler，并接收到消息。。。");
                                }
                            }
                        };
                    }
                }.start();
            }
        });
```
再次点击按钮执行该段代码之后，程序已经不会报错了，那么这说明初始化Handler对象的时候我们是需要调用Looper.prepare()的，那么主线程中为什么可以直接初始化Handler呢？

其实不是这样的，在App初始化的时候会执行ActivityThread的main方法：

```
public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
        SamplingProfilerIntegration.start();

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        Environment.initForCurrentUser();

        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());

        AndroidKeyStoreProvider.install();

        // Make sure TrustedCertificateStore looks in the right place for CA certificates
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

可以看到原来Looper.prepare()方法在这里调用了，所以在其他地方我们就可以直接初始化Handler了。

并且我们可以看到还调用了：Looper.loop()方法，通过参考阅读其他文章我们可以知道一个Handler的标准写法其实是这样的：

```
Looper.prepare();
Handler mHandler = new Handler() {
   @Override
   public void handleMessage(Message msg) {
      if (msg.what == 101) {
         Log.i(TAG, "在子线程中定义Handler，并接收到消息。。。");
       }
   }
};
Looper.loop();
```

**（三）查看Handler源码**
1）查看Looper.prepare()方法

```
// sThreadLocal.get() will return null unless you've called prepare().
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

/** Initialize the current thread as a looper.
      * This gives you a chance to create handlers that then reference
      * this looper, before actually starting the loop. Be sure to call
      * {@link #loop()} after calling this method, and end it by calling
      * {@link #quit()}.
      */
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```
可以看到Looper中有一个ThreadLocal成员变量，熟悉JDK的同学应该知道，当使用ThreadLocal维护变量时，ThreadLocal为每个使用该变量的线程提供独立的变量副本，所以每一个线程都可以独立地改变自己的副本，而不会影响其它线程所对应的副本。具体参考：<a href="http://blog.csdn.net/lufeng20/article/details/24314381">彻底理解ThreadLocal</a>
由此可以看出在每个线程中Looper.prepare()能且只能调用一次，这里我们可以尝试一下调用两次的情况。

```
/**
 * 这里Looper.prepare()方法调用了两次
*/
Looper.prepare();
Looper.prepare();
Handler mHandler = new Handler() {
   @Override
   public void handleMessage(Message msg) {
       if (msg.what == 101) {
          Log.i(TAG, "在子线程中定义Handler，并接收到消息。。。");
       }
   }
};
Looper.loop();
```
再次运行程序，点击按钮，执行该段代码：
![这里写图片描述](http://img.blog.csdn.net/20160226182500444)
可以看到程序出错，并提示prepare中的Excetion信息。

我们继续看Looper对象的构造方法，可以看到在其构造方法中初始化了一个MessageQueue对象：

```
private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```
***综上小结（1）：Looper.prepare()方法初始话了一个Looper对象并关联在一个MessageQueue对象，并且一个线程中只有一个Looper对象，只有一个MessageQueue对象。***

2）查看Handler对象的构造方法

```
public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```
可以看出在Handler的构造方法中，主要初始化了一下变量，并判断Handler对象的初始化不应再内部类，静态类，匿名类中，并且保存了当前线程中的Looper对象。
***综上小结（2）：Looper.prepare()方法初始话了一个Looper对象并关联在一个MessageQueue对象，并且一个线程中只有一个Looper对象，只有一个MessageQueue对象。而Handler的构造方法则在Handler内部维护了当前线程的Looper对象***

3）查看handler.sendMessage(msg)方法
一般的，我们发送异步消息的时候会这样调用：

```
mHandler.sendMessage(new Message());
```
通过不断的跟进源代码，其最后会调用：

```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```
原来msg.target就是Handler对象本身；而这里的queue对象就是我们的Handler内部维护的Looper对象关联的MessageQueue对象。查看messagequeue对象的enqueueMessage方法：

```
boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```
可以看到这里MessageQueue并没有使用列表将所有的Message保存起来，而是使用Message.next保存下一个Message，从而按照时间将所有的Message排序；

4）查看Looper.Loop()方法

```
/**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            msg.target.dispatchMessage(msg);

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
    }
```
可以看到方法的内容还是比较多的。可以看到Looper.loop()方法里起了一个死循环，不断的判断MessageQueue中的消息是否为空，如果为空则直接return掉，然后执行queue.next()方法：

```
Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
```

可以看到其大概的实现逻辑就是Message的出栈操作，里面可能对线程，并发控制做了一些限制等。获取到栈顶的Message对象之后开始执行：
```
msg.target.dispatchMessage(msg);
```

那么msg.target是什么呢？通过追踪可以知道就是我们定义的Handler对象，然后我们查看一下Handler类的dispatchMessage方法：

```
/**
     * Handle system messages here.
     */
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```
可以看到，如果我们设置了callback（Runnable对象）的话，则会直接调用handleCallback方法：

```
private static void handleCallback(Message message) {
        message.callback.run();
    }
```
即，如果我们在初始化Handler的时候设置了callback（Runnable）对象，则直接调用run方法。比如我们经常写的runOnUiThread方法：

```
runOnUiThread(new Runnable() {
            @Override
            public void run() {
                
            }
        });
```
看其内部实现：

```
public final void runOnUiThread(Runnable action) {
        if (Thread.currentThread() != mUiThread) {
            mHandler.post(action);
        } else {
            action.run();
        }
    }
```

而如果msg.callback为空的话，会直接调用我们的mCallback.handleMessage(msg)，即handler的handlerMessage方法。由于Handler对象是在主线程中创建的，所以handler的handlerMessage方法的执行也会在主线程中。


**综上可以知道：**
1）主线程中定义Handler，直接执行：

```
Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
               super.handleMessage(msg);
        }
};
```
而如果想要在子线程中定义Handler，则标准的写法为：

```
// 初始化该线程Looper，MessageQueue，执行且只能执行一次
                Looper.prepare();
                // 初始化Handler对象，内部关联Looper对象
                Handler mHandler = new Handler() {
                    @Override
                    public void handleMessage(Message msg) {
                        super.handleMessage(msg);
                    }
                };
                // 启动消息队列出栈死循环
                Looper.loop();
```

2）一个线程中只存在一个Looper对象，只存在一个MessageQueue对象，可以存在N个Handler对象，Handler对象内部关联了本线程中唯一的Looper对象，Looper对象内部关联着唯一的一个MessageQueue对象。

3）MessageQueue消息队列不是通过列表保存消息（Message）列表的，而是通过Message对象的next属性关联下一个Message从而实现列表的功能，同时所有的消息都是按时间排序的。

4）android中两个子线程相互交互同样可以通过Handler的异步消息机制实现，可以在线程a中定义Handler对象，而在线程b中获取handler的引用并调用sendMessage方法。

5）activity内部默认存在一个handler的成员变量，android中一些其他的异步消息机制的实现方法：
Handler的post方法：

```
mHandler.post(new Runnable() {
                    @Override
                    public void run() {

                    }
                });
```
查看其内部实现：

```
public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }
```
可以发现其内部调用就是sendMessage系列方法。。。

view的post方法：

```
public boolean post(Runnable action) {
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null) {
            return attachInfo.mHandler.post(action);
        }
        // Assume that post will succeed later
        ViewRootImpl.getRunQueue().post(action);
        return true;
    }
```
可以发现其调用的就是activity中默认保存的handler对象的post方法。

activity的runOnUiThread方法：

```
public final void runOnUiThread(Runnable action) {
        if (Thread.currentThread() != mUiThread) {
            mHandler.post(action);
        } else {
            action.run();
        }
    }
```
判断当前线程是否是UI线程，如果不是，则调用handler的post方法，否则直接执行run方法。


参考文章：
<br><a href="http://blog.csdn.net/guolin_blog/article/details/9991569">Android异步消息处理机制完全解析，带你从源码的角度彻底理解</a>
<br><a href="http://blog.csdn.net/yanbober/article/details/45936145"> Android异步消息处理机制详解及源码分析</a>

另外对android源码解析方法感兴趣的可参考我的：
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50634435"> android源码解析之（一）-->android项目构建过程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50751687">android源码解析之（二）-->异步消息机制</a>
