HandlerThread是个什么东西？查看类的定义时有这样一段话：

```
Handy class for starting a new thread that has a looper. The looper can then be used to create handler classes. Note that start() must still be called.
```
意思就是说：这个类的作用是创建一个包含looper的线程。
那么我们在什么时候需要用到它呢?加入在应用程序当中为了实现同时完成多个任务，所以我们会在应用程序当中创建多个线程。为了让多个线程之间能够方便的通信，我们会使用Handler实现线程间的通信。这个时候我们手动实现的多线程+Handler的简化版就是我们HandlerThrea所要做的事了。

下面我们首先看一下HandlerThread的基本用法：

```
HandlerThread mHandlerThread = new HandlerThread("myHandlerThreand");
        mHandlerThread.start();

        // 创建的Handler将会在mHandlerThread线程中执行
        final Handler mHandler = new Handler(mHandlerThread.getLooper()) {
            @Override
            public void handleMessage(Message msg) {
                Log.i("tag", "接收到消息：" + msg.obj.toString());
            }
        };

        title = (TextView) findViewById(R.id.title);
        title.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {

                Message msg = new Message();
                msg.obj = "11111";
                mHandler.sendMessage(msg);

                msg = new Message();
                msg.obj = "2222";
                mHandler.sendMessage(msg);
            }
        });
```

我们首先定义了一个HandlerThread对象，是直接通过new的方式产生的，查看其构造方法：

```
public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }
```
可以知道HandlerThread继承于Thread，所以说HandlerThread本质上是一个线程，其构造方法主要是做一些初始化的操作。

然后我们调用了mHandlerThread.start()方法，由上我们知道了HandlerThread类其实就是一个Thread，一个线程，所以其start方法内部调用的肯定是Thread的run方法，我们查看一下其run方法的具体实现：

```
@Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
```
我们发现其内部调用了Looper.prepate()方法和Loop.loop()方法，熟悉android异步消息机制的童鞋应当知道，在android体系中一个线程其实是对应着一个Looper对象、一个MessageQueue对象，以及N个Handler对象，具体可参考：<a href="http://blog.csdn.net/qq_23547831/article/details/50751687"> android源码解析之（二）-->异步消息机制</a>

所以通过run方法，我们可以知道在我们创建的HandlerThread线程中我们创建了该线程的Looper与MessageQueue；

这里需要注意的是其在调用Looper.loop()方法之前调用了一个空的实现方法：onLooperPrepared(),我们可以实现自己的onLooperPrepared（）方法，做一些Looper的初始化操作；

run方法里面当mLooper创建完成后有个notifyAll()，getLooper()中有个wait()，这是为什么呢？因为的mLooper在一个线程中执行，而我们的handler是在UI线程初始化的，也就是说，我们必须等到mLooper创建完成，才能正确的返回getLooper();wait(),notify()就是为了解决这两个线程的同步问题

然后我们调用了：

```
// 创建的Handler将会在mHandlerThread线程中执行
        final Handler mHandler = new Handler(mHandlerThread.getLooper()) {
            @Override
            public void handleMessage(Message msg) {
                Log.i("tag", "接收到消息：" + msg.obj.toString());
            }
        };
```
该Handler的构造方法中传入了HandlerThread的Looper对象，所以Handler对象就相当于含有了HandlerThread线程中Looper对象的引用。

然后我们调用handler的sendMessage方法发送消息，在Handler的handleMessge方法中就可以接收到消息了。

最后需要注意的是在我们不需要这个looper线程的时候需要手动停止掉；

```
protected void onDestroy() {
        super.onDestroy();
        mHandlerThread.quit();
    }

```

相对来说HandlerThread还是比较简单的，这里总结一下：

- HandlerThread本质上是一个Thread对象，只不过其内部帮我们创建了该线程的Looper和MessageQueue；

- 通过HandlerThread我们不但可以实现UI线程与子线程的通信同样也可以实现子线程与子线程之间的通信；

- HandlerThread在不需要使用的时候需要手动的回收掉；

另外对android源码解析方法感兴趣的可参考我的：
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50634435"> android源码解析之（一）-->android项目构建过程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50751687">android源码解析之（二）-->异步消息机制</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50803849">android源码解析之（三）-->异步任务AsyncTask</a>
