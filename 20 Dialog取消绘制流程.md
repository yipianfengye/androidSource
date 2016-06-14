上几篇文章中我们分析了Dialog的加载绘制流程，也分析了Acvityi的加载绘制流程，说白了Android系统中窗口的展示都是通过Window对象控制，通过ViewRootImpl对象执行绘制操作来完成的，那么窗口的取消绘制流程是怎么样的呢？这篇文章就以Dialog为例说明Window窗口是如何取消绘制的。

有的同学可能会问前几篇文章介绍Activity的加载绘制流程的时候为何没有讲Activity的窗口取消流程，这里说明一下。那是因为当时说明的重点是Activity的加载与绘制流程，而取消绘制流程由于混杂在Activity的生命周期管理，可能不太明显，所以这里将Window窗口的取消绘制流程放在Dialog中，其实他们的取消绘制流程都是相似的，看完Dialog的取消绘制流程之后，再看一下Activity的取消绘制流程就很简单了。

还记得我们上一篇文章关于Dialog的例子么？我们通过AlertDialog.Builder创建了一个AlertDialog，并通过Activity中的按钮点击事件来显示这个AlertDialog，而在AlertDialog中定义了一个“知道了”按钮，点击这个按钮就会触发alertDialog.cancel方法，通过执行这个方法，我们的alertDialog就不在显示了，很明显的，cancel方法执行过程中就执行了取消绘制的逻辑，这里我们先看一下我们的例子核心代码：

```
title.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                AlertDialog.Builder builder = new AlertDialog.Builder(MainActivity.this.getApplication());
                builder.setIcon(R.mipmap.ic_launcher);
                builder.setMessage("this is the content view!!!");
                builder.setTitle("this is the title view!!!");
                builder.setView(R.layout.activity_second);
                builder.setPositiveButton("知道了", new DialogInterface.OnClickListener() {
                    @Override
                    public void onClick(DialogInterface dialog, int which) {
                        alertDialog.cannel();
                    }
                });
                alertDialog = builder.create();
                alertDialog.show();
            }
        });
```
这里的title就是我们自己的Activity中的一个TextView，通过注册这个TextView的点击事件，来显示一个AlertDialog，通过注册AlertDialog中按钮的点击事件，执行alertDialog的cancel方法。

好吧，看一下Dialog的cannel方法的具体实现：

```
public void cancel() {
        if (!mCanceled && mCancelMessage != null) {
            mCanceled = true;
            // Obtain a new message so this dialog can be re-used
            Message.obtain(mCancelMessage).sendToTarget();
        }
        dismiss();
    }
```
可以看到方法体中，若当前Dialog没有取消，并且设置了取消message，则调用Message.obtain(mCancel).sendToTarget()，前面已经分析过这里的sendToTarget方法会回调我们注册的异步消息处理逻辑：

```
public void setOnCancelListener(final OnCancelListener listener) {
        if (mCancelAndDismissTaken != null) {
            throw new IllegalStateException(
                    "OnCancelListener is already taken by "
                    + mCancelAndDismissTaken + " and can not be replaced.");
        }
        if (listener != null) {
            mCancelMessage = mListenersHandler.obtainMessage(CANCEL, listener);
        } else {
            mCancelMessage = null;
        }
    }
```
可以看到如果我们在初始化AlertDialog.Builder时，设置了setOnCancelListener，那么我们就会执行mListenersHandler的异步消息处理，好吧，这里看一下mListenersHandler的定义：

```
private static final class ListenersHandler extends Handler {
        private WeakReference<DialogInterface> mDialog;

        public ListenersHandler(Dialog dialog) {
            mDialog = new WeakReference<DialogInterface>(dialog);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case DISMISS:
                    ((OnDismissListener) msg.obj).onDismiss(mDialog.get());
                    break;
                case CANCEL:
                    ((OnCancelListener) msg.obj).onCancel(mDialog.get());
                    break;
                case SHOW:
                    ((OnShowListener) msg.obj).onShow(mDialog.get());
                    break;
            }
        }
    }
```
好吧，这里调用的是设置的OnCancelListener的onCancel方法，也就是说我们调用dialog.cancel方法时首先会判断dialog是否调用了setOnCancelListener若设置了，则先调用OnCancelListener的onCancel方法，然后再次执行dismiss方法，若我们没有为Dialog.Builder设置OnCancelListener那么cancel方法和dismiss方法是等效的。

这样，我们来看一下dismiss方法的实现逻辑：

```
public void dismiss() {
        if (Looper.myLooper() == mHandler.getLooper()) {
            dismissDialog();
        } else {
            mHandler.post(mDismissAction);
        }
    }
```
可以看到，这里首先判断当前线程的Looper是否是主线程的Looper（由于mHandler是在主线程中创建的，所以mHandler.getLooper返回的是主线程中创建的Looper对象），若是的话，则直接执行dismissDialog()方法，否则的话，通过mHandler发送异步消息至主线程中，简单来说就是判断当前线程是否是主线程，若是主线程则执行dismissDialog方法否则发送异步消息，我们看一下mHandler对异步消息的处理机制，由于这里的mDismissAction是一个Runnable对象，所以这里直接看一下mDismissAction的定义：
```
private final Runnable mDismissAction = new Runnable() {
        public void run() {
            dismissDialog();
        }
    };
```
好吧，这里的异步消息最终也是调用的dismissDialog方法。。。。

所以无论我们执行的cancel方法还是dismiss方法，无论我们方法是在主线程执行还是子线程中执行，最终调用的都是dismissDialog方法，那么就看一下dismissDialog是怎么个执行逻辑。
```
void dismissDialog() {
        if (mDecor == null || !mShowing) {
            return;
        }

        if (mWindow.isDestroyed()) {
            Log.e(TAG, "Tried to dismissDialog() but the Dialog's window was already destroyed!");
            return;
        }

        try {
            mWindowManager.removeViewImmediate(mDecor);
        } finally {
            if (mActionMode != null) {
                mActionMode.finish();
            }
            mDecor = null;
            mWindow.closeAllPanels();
            onStop();
            mShowing = false;

            sendDismissMessage();
        }
    }
```
好吧，看样子代码还不是特别多，方法体中，首先判断当前的mDector是否为空，或者当前Dialog是否在显示，若为空或者没有在显示，则直接return掉，也就是说当前我们的dialog已经不再显示了，则我们不需要往下在执行。

然后我们调用了mWindow.isDestroyed()方法，判断Window对象是否已经被销毁，若已经被销毁，则直接return，并打印错误日志。

然后我们调用了mWindowManager.removeViewImmediate(mDector)，这里的mDector是我们Dialog窗口的根布局，看这个方法的名字应该就是Dialog去除根布局的操作了，可以看一下这个方法的具体实现。前几篇文章中我们已经分析过了这里的mWindowManager其实是WindowManagerImpl的实例，所以这里的removeViewImmediate方法应该是WindowManagerImpl中的方法，我们看一下它的具体实现：

```
@Override
    public void removeViewImmediate(View view) {
        mGlobal.removeView(view, true);
    }
```
可以发现，这里它调用了mGlobal.removeView方法，而这里的mGlobal是WindowManagerGlobal的实例，所以我们再看一下WIndowManagerGlobal中removeView的实现逻辑:

```
public void removeView(View view, boolean immediate) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }

        synchronized (mLock) {
            int index = findViewLocked(view, true);
            View curView = mRoots.get(index).getView();
            removeViewLocked(index, immediate);
            if (curView == view) {
                return;
            }

            throw new IllegalStateException("Calling with view " + view
                    + " but the ViewAncestor is attached to " + curView);
        }
    }
```
可以发现，这里在获取了保存的mDector组件之后，又调用了removeViewLocked方法，我们在看一下这个方法的具体实现逻辑：

```
private void removeViewLocked(int index, boolean immediate) {
        ViewRootImpl root = mRoots.get(index);
        View view = root.getView();

        if (view != null) {
            InputMethodManager imm = InputMethodManager.getInstance();
            if (imm != null) {
                imm.windowDismissed(mViews.get(index).getWindowToken());
            }
        }
        boolean deferred = root.die(immediate);
        if (view != null) {
            view.assignParent(null);
            if (deferred) {
                mDyingViews.add(view);
            }
        }
    }
```
看到了么，我们获取了mDector组件的ViewRootImpl，然后调用了其的die方法，通过这个方法实现Window组件的销毁流程。

```
boolean die(boolean immediate) {
        // Make sure we do execute immediately if we are in the middle of a traversal or the damage
        // done by dispatchDetachedFromWindow will cause havoc on return.
        if (immediate && !mIsInTraversal) {
            doDie();
            return false;
        }

        if (!mIsDrawing) {
            destroyHardwareRenderer();
        } else {
            Log.e(TAG, "Attempting to destroy the window while drawing!\n" +
                    "  window=" + this + ", title=" + mWindowAttributes.getTitle());
        }
        mHandler.sendEmptyMessage(MSG_DIE);
        return true;
    }
```
可以看到在方法体中有调用了doDie方法，看名字应该就是真正执行window销毁工作的方法了，我们在看一下doDie方法的具体实现：

```
void doDie() {
        checkThread();
        if (LOCAL_LOGV) Log.v(TAG, "DIE in " + this + " of " + mSurface);
        synchronized (this) {
            if (mRemoved) {
                return;
            }
            mRemoved = true;
            if (mAdded) {
                dispatchDetachedFromWindow();
            }

            if (mAdded && !mFirst) {
                destroyHardwareRenderer();

                if (mView != null) {
                    int viewVisibility = mView.getVisibility();
                    boolean viewVisibilityChanged = mViewVisibility != viewVisibility;
                    if (mWindowAttributesChanged || viewVisibilityChanged) {
                        // If layout params have been changed, first give them
                        // to the window manager to make sure it has the correct
                        // animation info.
                        try {
                            if ((relayoutWindow(mWindowAttributes, viewVisibility, false)
                                    & WindowManagerGlobal.RELAYOUT_RES_FIRST_TIME) != 0) {
                                mWindowSession.finishDrawing(mWindow);
                            }
                        } catch (RemoteException e) {
                        }
                    }

                    mSurface.release();
                }
            }

            mAdded = false;
        }
        WindowManagerGlobal.getInstance().doRemoveView(this);
    }
```
可以看到方法体中，首先调用了checkThread方法，介绍Activity的绘制流程的时候有过介绍，判断当前执行代码的线程，若不是主线程，则抛出异常：

```
void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }
```
我们顺着doDie的方法往下看，又调用了dispatchDetachedFromWindow()方法，这个方法主要是销毁Window中的各中成员变量，临时变量等

```
void dispatchDetachedFromWindow() {
        if (mView != null && mView.mAttachInfo != null) {
            mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(false);
            mView.dispatchDetachedFromWindow();
        }

        mAccessibilityInteractionConnectionManager.ensureNoConnection();
        mAccessibilityManager.removeAccessibilityStateChangeListener(
                mAccessibilityInteractionConnectionManager);
        mAccessibilityManager.removeHighTextContrastStateChangeListener(
                mHighContrastTextManager);
        removeSendWindowContentChangedCallback();

        destroyHardwareRenderer();

        setAccessibilityFocus(null, null);

        mView.assignParent(null);
        mView = null;
        mAttachInfo.mRootView = null;

        mSurface.release();

        if (mInputQueueCallback != null && mInputQueue != null) {
            mInputQueueCallback.onInputQueueDestroyed(mInputQueue);
            mInputQueue.dispose();
            mInputQueueCallback = null;
            mInputQueue = null;
        }
        if (mInputEventReceiver != null) {
            mInputEventReceiver.dispose();
            mInputEventReceiver = null;
        }
        try {
            mWindowSession.remove(mWindow);
        } catch (RemoteException e) {
        }

        // Dispose the input channel after removing the window so the Window Manager
        // doesn't interpret the input channel being closed as an abnormal termination.
        if (mInputChannel != null) {
            mInputChannel.dispose();
            mInputChannel = null;
        }

     mDisplayManager.unregisterDisplayListener(mDisplayListener);

        unscheduleTraversals();
    }
```
可以看到我们在方法中调用了mView.dispatchDetachedFromWindow方法，这个方法的作用就是将mView从Window中detach出来，我们可以看一下这个方法的具体实现：

```
void dispatchDetachedFromWindow() {
        AttachInfo info = mAttachInfo;
        if (info != null) {
            int vis = info.mWindowVisibility;
            if (vis != GONE) {
                onWindowVisibilityChanged(GONE);
            }
        }

        onDetachedFromWindow();
        onDetachedFromWindowInternal();

        InputMethodManager imm = InputMethodManager.peekInstance();
        if (imm != null) {
            imm.onViewDetachedFromWindow(this);
        }

        ListenerInfo li = mListenerInfo;
        final CopyOnWriteArrayList<OnAttachStateChangeListener> listeners =
                li != null ? li.mOnAttachStateChangeListeners : null;
        if (listeners != null && listeners.size() > 0) {
            // NOTE: because of the use of CopyOnWriteArrayList, we *must* use an iterator to
            // perform the dispatching. The iterator is a safe guard against listeners that
            // could mutate the list by calling the various add/remove methods. This prevents
            // the array from being modified while we iterate it.
            for (OnAttachStateChangeListener listener : listeners) {
                listener.onViewDetachedFromWindow(this);
            }
        }

        if ((mPrivateFlags & PFLAG_SCROLL_CONTAINER_ADDED) != 0) {
            mAttachInfo.mScrollContainers.remove(this);
            mPrivateFlags &= ~PFLAG_SCROLL_CONTAINER_ADDED;
        }

        mAttachInfo = null;
        if (mOverlay != null) {
            mOverlay.getOverlayView().dispatchDetachedFromWindow();
        }
    }
```
其中onDetachedFromWindow方法是一个空的回调方法，这里我们重点看一下onDetachedFromWindowInternal方法：

```
protected void onDetachedFromWindowInternal() {
        mPrivateFlags &= ~PFLAG_CANCEL_NEXT_UP_EVENT;
        mPrivateFlags3 &= ~PFLAG3_IS_LAID_OUT;

        removeUnsetPressCallback();
        removeLongPressCallback();
        removePerformClickCallback();
        removeSendViewScrolledAccessibilityEventCallback();
        stopNestedScroll();

        // Anything that started animating right before detach should already
        // be in its final state when re-attached.
        jumpDrawablesToCurrentState();

        destroyDrawingCache();

        cleanupDraw();
        mCurrentAnimation = null;
    }
```
onDetachedFromWindowInternal方法的方法体也不是特别长，都是一些调用函数，这里看一下destropDrawingCache方法，这个方法主要是销毁View的缓存Drawing，我们来看一下具体实现：

```
public void destroyDrawingCache() {
        if (mDrawingCache != null) {
            mDrawingCache.recycle();
            mDrawingCache = null;
        }
        if (mUnscaledDrawingCache != null) {
            mUnscaledDrawingCache.recycle();
            mUnscaledDrawingCache = null;
        }
    }
```
这里的mDrawingCache其实就是一个Bitmap类型的成员变量，而这里调用的recycler和置空操作其实就是把View中执行draw方法之后缓存的bitmap清空。

这里需要说明的是，我们View组件的最终显示落实是通过draw方法实现绘制的，而我们的draw方法的参数是一个Canvas，这是一个画布的对象，通过draw方法就是操作这个对象并显示出来，而Canvas对象之所以能够实现显示的效果是因为其内部保存着一个Bitmap对象，通过操作Canvas对象实质上是操作Canvas对象内部的Bitmap对象，而View组件的显示也就是通过这里的Bitmap来实现的。

而我们上文中置空了bitmap对象就相当于把View组件的显示效果置空了，就是相当于我们取消了View的draw方法的执行效果，继续回到我们的dispatchDetachedFromWindow方法，在执行了mView.dispatchDetachedFromWindow()方法之后，又调用了mView = null;方法，这里设置mView为空，这样我们有取消了View的meature和layouot的执行效果。

这样经过一系列的操作之后我们的Dialog的取消绘制流程就结束了，现在我们来看一下Activity的取消绘制流程。还记得我们“Activity的销毁流程”么？可参考：<a href="http://blog.csdn.net/qq_23547831/article/details/51232309">android源码解析之（十五）-->Activity销毁流程</a>
当我们调用activity的finish方法的时候回调用ActivityThread的handleDestroyActivity方法，我们来看一下这个方法的实现：

```
private void handleDestroyActivity(IBinder token, boolean finishing,
            int configChanges, boolean getNonConfigInstance) {
        ...
        wm.removeViewImmediate(v);
        ...            
    }
```
可以看到这里调用了这里调用了wm.removeViewImmediate方法，这个方法不就是我们刚刚分析Dialog销毁绘制流程的起始方法么？以后的逻辑都是详细的，这样我们就实现了Activity的取消绘制流程。

总结：

- 窗口的取消绘制流程是相似的，包括Activity和Dialog等；

- 通过调用WindowManager.removeViewImmediate方法，开始执行Window窗口的取消绘制流程；

- Window窗口的取消绘制流程，通过清空bitma撤销draw的执行效果，通过置空View撤销meature和layout的执行效果；

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
