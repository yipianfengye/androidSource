前面我们分析了Activity、Dialog、PopupWindow的加载绘制流程，相信大家对整个Android系统中的窗口绘制流程已经有了一个比较清晰的认识了，这里最后再给大家介绍一下Toast的加载绘制流程。

其实Toast窗口和Activity、Dialog、PopupWindow有一个不太一样的地方，就是Toast窗口是属于系统级别的窗口，他和输入框等类似的，不属于某一个应用，即不属于某一个进程，所以自然而然的，一旦涉及到Toast的加载绘制流程就会涉及到进程间通讯，看过前面系列文章的同学应该知道，Android间的进程间通讯采用的是Android特有的Binder机制，所以Toast的加载绘制流程也会涉及到Binder进程间通讯。

Toast的显示流程其实内部还是通过Window的窗口机制实现加载绘制的，只不过由于是系统级别的窗口，在显示过程中涉及到了进程间通讯等机制。

下面我们来具体看一下Toast窗口的简单使用。

```
Toast.makeText(context, msg, Toast.LENGTH_SHORT).show();
```
上面的代码是Toast的典型使用方式，通过makeText方法创建出一个Toast对象，然后调用show方法将Toast窗口显示出来。

下面我们来看一下makeText方法的具体实现：

```
public static Toast makeText(Context context, CharSequence text, @Duration int duration) {
        Toast result = new Toast(context);

        LayoutInflater inflate = (LayoutInflater)
                context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        View v = inflate.inflate(com.android.internal.R.layout.transient_notification, null);
        TextView tv = (TextView)v.findViewById(com.android.internal.R.id.message);
        tv.setText(text);
        
        result.mNextView = v;
        result.mDuration = duration;

        return result;
    }
```
方法体不是很长，在makeText方法中，我们首先通过Toast对象的构造方法，创建了一个新的Toast对象，这样我们就先来看一下Toast的构造方法做了哪些事。

```
public Toast(Context context) {
        mContext = context;
        mTN = new TN();
        mTN.mY = context.getResources().getDimensionPixelSize(
                com.android.internal.R.dimen.toast_y_offset);
        mTN.mGravity = context.getResources().getInteger(
                com.android.internal.R.integer.config_toastDefaultGravity);
    }
```
可以看到这里初始化了Toast对象的成员变量mContext和mTN，这里的mContext是一个Context类型的成员变量，那mTN是什么东西呢？

```
private static class TN extends ITransientNotification.Stub
```
从类的源码定义来看，我们知道TN是一个继承自ITransientNotification.Stub的类，这里我们暂时只用知道他的继承关系就好了，知道其是一个Binder对象，可以用于进程间通讯，然后回到我们的makeText方法，在调用了Toast的构造方法创建了Toast对象之后，我们又通过context.getSystemService方法获取到LayoutInflater，然后通过调用LayoutInflater的inflate方法加载到了Toast的布局文件：

```
LayoutInflater inflate = (LayoutInflater)
                context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        View v = inflate.inflate(com.android.internal.R.layout.transient_notification, null);
```
这里我们可以看一下布局文件的具体代码：

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:background="?android:attr/toastFrameBackground">

    <TextView
        android:id="@android:id/message"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_weight="1"
        android:layout_gravity="center_horizontal"
        android:textAppearance="@style/TextAppearance.Toast"
        android:textColor="@color/bright_foreground_dark"
        android:shadowColor="#BB000000"
        android:shadowRadius="2.75"
        />

</LinearLayout>
```
可以发现Toast加载的布局文件只有一个LinearLayout布局，并且只包含一个TextView组件。。。。

然后我们通过调用：

```
TextView tv = (TextView)v.findViewById(com.android.internal.R.id.message);
        tv.setText(text);
        
        result.mNextView = v;
        result.mDuration = duration;

        return result;
```
初始化了布局文件，Toast的mNextView和mDuration成员变量并返回Toast类型的result对象。这样我们的Toast对象就构造完成了。

然后我们回到我们的Toast.show方法，调用完这个方法之后就准备开始显示Toast窗口了，我们来具体看一下show方法的具体实现：

```
public void show() {
        if (mNextView == null) {
            throw new RuntimeException("setView must have been called");
        }

        INotificationManager service = getService();
        String pkg = mContext.getOpPackageName();
        TN tn = mTN;
        tn.mNextView = mNextView;

        try {
            service.enqueueToast(pkg, tn, mDuration);
        } catch (RemoteException e) {
            // Empty
        }
    }
```
首先判断我们的mNextView是否为空，为空的话，显示逻辑就无法进行了，所以这里判断如果mNextView为空的话，就直接抛出异常，不在往下执行。。。。

然后我们执行了：

```
INotificationManager service = getService();
```
这里的INotificationManager是服务器端NotificationManagerService的Binder客户端，我们可以看一下getService方法的实现方式：

```
static private INotificationManager getService() {
        if (sService != null) {
            return sService;
        }
        sService = INotificationManager.Stub.asInterface(ServiceManager.getService("notification"));
        return sService;
    }
```
这里获取了INotificationManager对象，然后我们调用了service.enqueueToast方法，并传递了package，TN对象，duration等参数，这里实际执行的是NotificationManagerService的内部类的INotificationManager.Stub的enqueueToast方法，而我们的NoticationManagerService是在SystemServer进程中执行的，这里的底层其实是通过Binder机制传输数据的，具体的Binder机制相关知识可自行学习。。

好吧，我们在看一下INotificationManager.Stub的enqueueToast方法的具体实现：

```
@Override
        public void enqueueToast(String pkg, ITransientNotification callback, int duration)
        {
            ...
            synchronized (mToastQueue) {
                int callingPid = Binder.getCallingPid();
                long callingId = Binder.clearCallingIdentity();
                try {
                    ToastRecord record;
                    int index = indexOfToastLocked(pkg, callback);
                    // If it's already in the queue, we update it in place, we don't
                    // move it to the end of the queue.
                    if (index >= 0) {
                        record = mToastQueue.get(index);
                        record.update(duration);
                    } else {
                        // Limit the number of toasts that any given package except the android
                        // package can enqueue.  Prevents DOS attacks and deals with leaks.
                        if (!isSystemToast) {
                            int count = 0;
                            final int N = mToastQueue.size();
                            for (int i=0; i<N; i++) {
                                 final ToastRecord r = mToastQueue.get(i);
                                 if (r.pkg.equals(pkg)) {
                                     count++;
                                     if (count >= MAX_PACKAGE_NOTIFICATIONS) {
                                         Slog.e(TAG, "Package has already posted " + count
                                                + " toasts. Not showing more. Package=" + pkg);
                                         return;
                                     }
                                 }
                            }
                        }

                        record = new ToastRecord(callingPid, pkg, callback, duration);
                        mToastQueue.add(record);
                        index = mToastQueue.size() - 1;
                        keepProcessAliveLocked(callingPid);
                    }
                    // If it's at index 0, it's the current toast.  It doesn't matter if it's
                    // new or just been updated.  Call back and tell it to show itself.
                    // If the callback fails, this will remove it from the list, so don't
                    // assume that it's valid after this.
                    if (index == 0) {
                        showNextToastLocked();
                    }
                } finally {
                    Binder.restoreCallingIdentity(callingId);
                }
            }
        }
```
可以发现我们首先将我们的ToastRecord（Toast对象在server端的对象）保存到一个List列表mToastQueue中，然后调用了showNextToastLocked方法，这样我们在看一下showNextToastLocked方法的具体实现。

```
void showNextToastLocked() {
        ToastRecord record = mToastQueue.get(0);
        while (record != null) {
            if (DBG) Slog.d(TAG, "Show pkg=" + record.pkg + " callback=" + record.callback);
            try {
                record.callback.show();
                scheduleTimeoutLocked(record);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Object died trying to show notification " + record.callback
                        + " in package " + record.pkg);
                // remove it from the list and let the process die
                int index = mToastQueue.indexOf(record);
                if (index >= 0) {
                    mToastQueue.remove(index);
                }
                keepProcessAliveLocked(record.pid);
                if (mToastQueue.size() > 0) {
                    record = mToastQueue.get(0);
                } else {
                    record = null;
                }
            }
        }
    }
```
这里主要执行了record.callback.show方法，而这里的callback对象就是我们创建Toast对象的时候传递的TN对象，显然的，这了的show方法就是我们的Toast内部类TN的show方法，然后我们调用了scheduleTimeoutLocked方法，这里先看一下scheduleTimeoutLocked方法的实现。

```
private void scheduleTimeoutLocked(ToastRecord r)
    {
        mHandler.removeCallbacksAndMessages(r);
        Message m = Message.obtain(mHandler, MESSAGE_TIMEOUT, r);
        long delay = r.duration == Toast.LENGTH_LONG ? LONG_DELAY : SHORT_DELAY;
        mHandler.sendMessageDelayed(m, delay);
    }
```
可以发现这里发送了一个异步消息，并且这里的异步消息是在duration时间之后发送的，也就是说我们在Toast端传递的duration参数就是这里的message消息delay发送的时间，而我们发送MESSAGE_TIMEOUT异步消息之后最终会被方法handleTimeout执行。

```
private void handleTimeout(ToastRecord record)
    {
        if (DBG) Slog.d(TAG, "Timeout pkg=" + record.pkg + " callback=" + record.callback);
        synchronized (mToastQueue) {
            int index = indexOfToastLocked(record.pkg, record.callback);
            if (index >= 0) {
                cancelToastLocked(index);
            }
        }
    }
```
好吧，方法体里面又调用了cancelToastLocked方法，然后我们看一下cancelToastLocked方法的实现：

```
void cancelToastLocked(int index) {
        ToastRecord record = mToastQueue.get(index);
        try {
            record.callback.hide();
        } catch (RemoteException e) {
            Slog.w(TAG, "Object died trying to hide notification " + record.callback
                    + " in package " + record.pkg);
            // don't worry about this, we're about to remove it from
            // the list anyway
        }
        mToastQueue.remove(index);
        keepProcessAliveLocked(record.pid);
        if (mToastQueue.size() > 0) {
            // Show the next one. If the callback fails, this will remove
            // it from the list, so don't assume that the list hasn't changed
            // after this point.
            showNextToastLocked();
        }
    }
```
好吧，这里又是调用了record.callback.hide方法，显然的这里的hide方法和刚刚的show方法是相似的，都是调用的Toast内部类TN的hide方法，所以这里可以看出Toast的显示与隐藏操作都是在Toast内部类TN的show和hide方法实现的，然后我们调用了:

```
mToastQueue.remove(index);
```
清除这个Toast对象，并继续执行showNextToastLocked方法，直到mToastQueue的大小为0。。。

这样关于Toast窗口的显示与隐藏操作都是在Toast内部类TN的show方法和hide方法中，我们先看一下TN内部类的show方法的具体实现：

```
@Override
        public void show() {
            if (localLOGV) Log.v(TAG, "SHOW: " + this);
            mHandler.post(mShow);
        }
```
好吧，这里也是发送一个异步消息，我们看一下Runnable类型的mShow的定义。

```
final Runnable mShow = new Runnable() {
            @Override
            public void run() {
                handleShow();
            }
        };
```
可以看到再其run方法中调用了handleShow方法，继续看handleShow方法的实现逻辑。

```
public void handleShow() {
            if (localLOGV) Log.v(TAG, "HANDLE SHOW: " + this + " mView=" + mView
                    + " mNextView=" + mNextView);
            if (mView != mNextView) {
                // remove the old view if necessary
                handleHide();
                mView = mNextView;
                Context context = mView.getContext().getApplicationContext();
                String packageName = mView.getContext().getOpPackageName();
                if (context == null) {
                    context = mView.getContext();
                }
                mWM = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
                // We can resolve the Gravity here by using the Locale for getting
                // the layout direction
                final Configuration config = mView.getContext().getResources().getConfiguration();
                final int gravity = Gravity.getAbsoluteGravity(mGravity, config.getLayoutDirection());
                mParams.gravity = gravity;
                if ((gravity & Gravity.HORIZONTAL_GRAVITY_MASK) == Gravity.FILL_HORIZONTAL) {
                    mParams.horizontalWeight = 1.0f;
                }
                if ((gravity & Gravity.VERTICAL_GRAVITY_MASK) == Gravity.FILL_VERTICAL) {
                    mParams.verticalWeight = 1.0f;
                }
                mParams.x = mX;
                mParams.y = mY;
                mParams.verticalMargin = mVerticalMargin;
                mParams.horizontalMargin = mHorizontalMargin;
                mParams.packageName = packageName;
                if (mView.getParent() != null) {
                    if (localLOGV) Log.v(TAG, "REMOVE! " + mView + " in " + this);
                    mWM.removeView(mView);
                }
                if (localLOGV) Log.v(TAG, "ADD! " + mView + " in " + this);
                mWM.addView(mView, mParams);
                trySendAccessibilityEvent();
            }
        }
```
好吧，在handleShow方法中经过一系列的初始化操作，初始化mWN对象，初始化mView对象，初始化了mParams对象，然后调用了mWM的addView方法，到了这里大家应该就很熟悉了（不熟悉的同学可以看一下Activity的加载绘制流程等文章
<a href="http://blog.csdn.net/qq_23547831/article/details/51285804"> android源码解析（十八）-->Activity布局绘制流程</a>&nbsp;&nbsp;  
<a href="http://blog.csdn.net/qq_23547831/article/details/51284556"> android源码解析（十七）-->Activity布局加载流程</a>）通过这个方法就实现了Toast窗口的显示逻辑。

继续看一下TN的hide方法：

```
@Override
        public void hide() {
            if (localLOGV) Log.v(TAG, "HIDE: " + this);
            mHandler.post(mHide);
        }
```
好吧，和show方法类似，也是发送了一个异步消息，这里看一下Runnable类型的mHide对象的定义：

```
final Runnable mHide = new Runnable() {
            @Override
            public void run() {
                handleHide();
                // Don't do this in handleHide() because it is also invoked by handleShow()
                mNextView = null;
            }
        };
```
可以发现在其run方法中调用了handleHide方法，显然的，与show方法类似，这里的handleHide方法也是执行Toast窗口销毁的逻辑：

```
public void handleHide() {
            if (localLOGV) Log.v(TAG, "HANDLE HIDE: " + this + " mView=" + mView);
            if (mView != null) {
                // note: checking parent() just to make sure the view has
                // been added...  i have seen cases where we get here when
                // the view isn't yet added, so let's try not to crash.
                if (mView.getParent() != null) {
                    if (localLOGV) Log.v(TAG, "REMOVE! " + mView + " in " + this);
                    mWM.removeView(mView);
                }

                mView = null;
            }
        }
```
可以发现，在方法体重调用了mWM.removeView(mView),又是熟悉的代码，通过执行这里的removeView方法，我们可以实现Toast窗口的销毁流程，至此我们就分析完了Toast窗口的显示与销毁流程。

总结：

- Toast是一个系统窗口，Toast在显示与销毁流程设计到进程间通讯（Binder机制实现）

- Toast的show方法首先会初始化一个Toast对象，然后将内部对象TN与duration传递给NotificationManagerService，并在NotificationManagerService端维护一个Toast对象列表。

- NotificationManagerService接收到Toast的show请求之后，保存Toast对象并回调Toast.TN的show方法具体实现Toast窗口的显示逻辑。

- Toast窗口的显示与销毁机制与Activity、Dialog、PopupWIndow都是类似的，都是通过WIndow对象实现的。

- NotificationManagerService端在执行show方法执行会发送一个异步消息用于销毁Toast窗口，这个异步消息会在duration时间段之后发出，这样，在设置Toast显示的时间就会被传递到NotificationManagerService端，并在这段时间之后发送异步消息销毁Toast窗口。

另外对android源码解析方法感兴趣的可参考我的：
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50634435"> android源码解析之（一）-->android项目构建过程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50751687">android源码解析之（二）-->异步消息机制</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50803849">android源码解析之（三）-->异步任务AsyncTask</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50936584">android源码解析之（四）-->HandlerThread</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50958757">android源码解析之（五）-->IntentService</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50963006">android源码解析之（六）-->Log</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50971968">android源码解析之（七）-->LruCache</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51104873">android源码解析之（八）-->Zygote进程启动流程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51105171">android源码解析之（九）-->SystemServer进程启动流程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51112031">android源码解析之（十）-->Launcher启动流程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51119333">android源码解析之（十一）-->应用进程启动流程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51203482">android源码解析之（十二）-->系统启动并解析Manifest的流程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51210682">android源码解析之（十三）-->apk安装流程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51224992">android源码解析之（十四）-->Activity启动流程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51232309">android源码解析之（十五）-->Activity销毁流程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51252082">android源码解析（十六）-->应用进程Context创建流程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51284556">android源码解析（十七）-->Activity布局加载流程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51285804">android源码解析（十八）-->Activity布局绘制流程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51289456">android源码解析（十九）-->Dialog加载绘制流程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51303072">android源码解析（二十）-->Dialog取消绘制流程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51322574">android源码解析（二十一）-->PopupWindow加载绘制流程</a>
