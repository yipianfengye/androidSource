什么是IntentService？简单来说IntentService就是一个含有自身消息循环的Service，首先它是一个service，所以service相关具有的特性他都有，同时他还有一些自身的属性，其内部封装了一个消息队列和一个HandlerThread，在其具体的抽象方法：onHandleIntent方法是运行在其消息队列线程中，废话不多说，我们来看其简单的使用方法：

- 定义一个IntentService

```
public class MIntentService extends IntentService{

    public MIntentService() {
        super("");
    }

    @Override
    protected void onHandleIntent(Intent intent) {
        Log.i("tag", intent.getStringExtra("params") + "  " + Thread.currentThread().getId());
    }

}
```

- 在androidManifest.xml中定义service

```
<service
            android:name=".MIntentService"
            />
```

- 启动这个service

```
title.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent = new Intent(MainActivity.this, MIntentService.class);
                intent.putExtra("params", "ceshi");
                startService(intent);
            }
        });
```

可以发现当点击title组件的时候，service接收到了消息并打印出了传递过去的intent参数，同时显示onHandlerIntent方法执行的线程ID并非主线程，这是为什么呢？

下面我们来看一下service的源码：

```
public abstract class IntentService extends Service {
    private volatile Looper mServiceLooper;
    private volatile ServiceHandler mServiceHandler;
    private String mName;
    private boolean mRedelivery;

    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }

    /**
     * Creates an IntentService.  Invoked by your subclass's constructor.
     *
     * @param name Used to name the worker thread, important only for debugging.
     */
    public IntentService(String name) {
        super();
        mName = name;
    }

    /**
     * Sets intent redelivery preferences.  Usually called from the constructor
     * with your preferred semantics.
     *
     * <p>If enabled is true,
     * {@link #onStartCommand(Intent, int, int)} will return
     * {@link Service#START_REDELIVER_INTENT}, so if this process dies before
     * {@link #onHandleIntent(Intent)} returns, the process will be restarted
     * and the intent redelivered.  If multiple Intents have been sent, only
     * the most recent one is guaranteed to be redelivered.
     *
     * <p>If enabled is false (the default),
     * {@link #onStartCommand(Intent, int, int)} will return
     * {@link Service#START_NOT_STICKY}, and if the process dies, the Intent
     * dies along with it.
     */
    public void setIntentRedelivery(boolean enabled) {
        mRedelivery = enabled;
    }

    @Override
    public void onCreate() {
        // TODO: It would be nice to have an option to hold a partial wakelock
        // during processing, and to have a static startService(Context, Intent)
        // method that would launch the service & hand off a wakelock.

        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }

    @Override
    public void onStart(Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

    /**
     * You should not override this method for your IntentService. Instead,
     * override {@link #onHandleIntent}, which the system calls when the IntentService
     * receives a start request.
     * @see android.app.Service#onStartCommand
     */
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

    @Override
    public void onDestroy() {
        mServiceLooper.quit();
    }

    /**
     * Unless you provide binding for your service, you don't need to implement this
     * method, because the default implementation returns null. 
     * @see android.app.Service#onBind
     */
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    /**
     * This method is invoked on the worker thread with a request to process.
     * Only one Intent is processed at a time, but the processing happens on a
     * worker thread that runs independently from other application logic.
     * So, if this code takes a long time, it will hold up other requests to
     * the same IntentService, but it will not hold up anything else.
     * When all requests have been handled, the IntentService stops itself,
     * so you should not call {@link #stopSelf}.
     *
     * @param intent The value passed to {@link
     *               android.content.Context#startService(Intent)}.
     */
    @WorkerThread
    protected abstract void onHandleIntent(Intent intent);
}
```

怎么样，代码还是相当的简洁的，首先通过定义我们可以知道IntentService是一个Service，并且是一个抽象类，所以我们在继承IntentService的时候需要实现其抽象方法：onHandlerIntent。

下面看一下其onCreate方法：

```
@Override
    public void onCreate() {
        // TODO: It would be nice to have an option to hold a partial wakelock
        // during processing, and to have a static startService(Context, Intent)
        // method that would launch the service & hand off a wakelock.

        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }
```
我们可以发现其内部定义一个HandlerIThread（本质上是一个含有消息队列的线程）具体可参考：<a href="http://blog.csdn.net/qq_23547831/article/details/50936584">android源码解析之（四）-->HandlerThread</a>
然后用成员变量维护其Looper和Handler，由于其Handler关联着这个HandlerThread的Looper对象，所以Handler的handMessage方法在HandlerThread线程中执行。

然后我们发现其onStartCommand方法就是调用的其onStart方法，具体看一下其onStart方法：

```
@Override
    public void onStart(Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }
```
很简单就是就是讲startId和启动时接受到的intent对象传递到消息队列中处理，那么我们具体看一下其消息队列的处理逻辑：

```
private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }
```
可以看到起handleMessage方法内部执行了两个逻辑一个是调用了其onHandlerIntent抽象方法，通过分析其onCreate方法handler对象的创建过程我们知道其handler对象是依附于HandlerThread线程的，所以其handeMessage方法也是在HandlerThread线程中执行的，从而证实了我们刚刚例子中的一个结论，onHandlerIntent在子线程中执行。
然后调用了stopSelf方法，这里需要注意的是stopSelf方法传递了msg.arg1参数，从刚刚的onStart方法我们可以知道我们传递了startId，参考其他文章我们知道，由于service可以启动N次，可以传递N次消息，当IntentService的消息队列中含有消息时调用stopSelf(startId)并不会立即stop自己，只有当消息队列中最后一个消息被执行完成时才会真正的stop自身。


通过上面的例子与相关说明，我们可以知道：

- IntentService是一个service，也是一个抽象类；

- 继承IntentService需要实现其onHandlerIntent抽象方法；

- onHandlerIntent在子线程中执行；

- IntentService内部保存着一个HandlerThread、Looper与Handler等成员变量，维护这自身的消息队列；

- 每次IntentService后台任务执行完成之后都会尝试关闭自身，但是当且仅当IntentService消息队列中最后一个消息被执行完成之后才会真正的stop自身；


另外对android源码解析方法感兴趣的可参考我的：
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50634435"> android源码解析之（一）-->android项目构建过程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50751687">android源码解析之（二）-->异步消息机制</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50803849">android源码解析之（三）-->异步任务AsyncTask</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50936584">android源码解析之（四）-->HandlerThread</a>
