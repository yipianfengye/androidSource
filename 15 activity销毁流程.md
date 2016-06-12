继续我们的源码解析，上一篇文章我们介绍了Activity的启动流程，一个典型的场景就是Activity a 启动了一个Activity b，他们的生命周期回调方法是：
onPause(a) --> onCreate(b) --> onStart(b) --> onResume(b) --> onStop(a)
而我们根据源码也验证了这样的生命周期调用序列，那么Activity的销毁流程呢？它的生命周期的调用顺序又是这样的呢？

这里我们我做一个简单的demo，让一个Activity a启动Activity b，然后在b中调用finish()方法，它们的生命周期执行顺序是：
> onPause(b)
> onRestart(a)
> onStart(a)
> onResume(a)
> onStop(b)
> onDestory(b)

好吧，根据我们测试的生命周期方法的回调过程开始对Activity销毁流程的分析，一般而言当我们需要销毁Activity的时候都会调用其自身的finish方法，所以我们的流程开始是以finish方法开始的。

<br><strong><font size="6">一：请求销毁当前Activity

> <font color="red">
> MyActivity.finish()
> Activity.finish()
> ActivityManagerNative.getDefault().finishActivity()
> ActivityManagerService.finishActivity()
> ActivityStack.requestFinishActivityLocked()
> ActivityStack.finishActivityLocked()
> ActivityStack.startPausingLocked()
> </font>

首先我们在自己的Activity调用了finish方法，它实际上调用的是Activity的finish方法：

```
public void finish() {
    finish(false);
}
```
然后我们可以发现其调用了finish方法的重载方法，并且传递了一个参数值：

```
private void finish(boolean finishTask) {
        if (mParent == null) {
            int resultCode;
            Intent resultData;
            synchronized (this) {
                resultCode = mResultCode;
                resultData = mResultData;
            }
            if (false) Log.v(TAG, "Finishing self: token=" + mToken);
            try {
                if (resultData != null) {
                    resultData.prepareToLeaveProcess();
                }
                if (ActivityManagerNative.getDefault()
                        .finishActivity(mToken, resultCode, resultData, finishTask)) {
                    mFinished = true;
                }
            } catch (RemoteException e) {
                // Empty
            }
        } else {
            mParent.finishFromChild(this);
        }
    }
```
好吧，这个参数值似乎并没什么用。。。这里就不在讨论了，然后调用了ActivityManagerNative.getDefault().finishActivity方法，好吧，根据上一篇文章的介绍，我们知道了ActivityManagerNative是一个Binder对象，这里调用的方法最终会被ActivityManagerService执行，所以这了的finishActivity最终被执行的是ActivityManagerService.finishActivity方法，好吧，我们来看一下ActivityManagerService的finishActivity方法的执行逻辑。。。

```
@Override
public final boolean finishActivity(IBinder token, int resultCode, Intent resultData, boolean finishTask) {
     ...
     res = tr.stack.requestFinishActivityLocked(token, resultCode,resultData, "app-request", true);
     ...
}
```
这里我们可以发现，经过一系列逻辑判断之后，最终调用了ActivityStack的requestFinishActivityLocked方法，这里应该就是执行finish Activity的逻辑了。

```
final boolean requestFinishActivityLocked(IBinder token, int resultCode,
            Intent resultData, String reason, boolean oomAdj) {
        ActivityRecord r = isInStackLocked(token);
        if (DEBUG_RESULTS || DEBUG_STATES) Slog.v(TAG_STATES,
                "Finishing activity token=" + token + " r="
                + ", result=" + resultCode + ", data=" + resultData
                + ", reason=" + reason);
        if (r == null) {
            return false;
        }

        finishActivityLocked(r, resultCode, resultData, reason, oomAdj);
        return true;
    }
```
这个方法体里面又调用了finishActivityLocked方法，那我们继续看一下finishActivityLocked方法的实现：

```
final boolean finishActivityLocked(ActivityRecord r, int resultCode, Intent resultData,
            String reason, boolean oomAdj) {
        ...
        startPausingLocked(false, false, false, false);
		...
        return false;
    }
```
好吧，在这里调用了startPausingLocked方法，看名字应该是开始要执行Activity的onPause方法请求了，然后我们看一下startPausingLocked方法的实现：

```
final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping, boolean resuming, boolean dontWait) {
       ...
            try {
                EventLog.writeEvent(EventLogTags.AM_PAUSE_ACTIVITY,
                        prev.userId, System.identityHashCode(prev),
                        prev.shortComponentName);
                mService.updateUsageStats(prev, false);
                prev.app.thread.schedulePauseActivity(prev.appToken, prev.finishing,
                        userLeaving, prev.configChangeFlags, dontWait);
            } catch (Exception e) {
                // Ignore exception, if process died other code will cleanup.
                Slog.w(TAG, "Exception thrown during pause", e);
                mPausingActivity = null;
                mLastPausedActivity = null;
                mLastNoHistoryActivity = null;
            }
        ...
    }
```
这样从应用程序调用finish方法，ActivityManagerService接收请求并执行startPausingLocked方法。

<br><strong><font size="6">二：执行当前Activity的onPause方法

> <font color="red">
> IApplicationThread.schedulePauseActivity()
> ActivityThread.schedulePauseActivity()
> ActivityThread.sendMessage()
> ActivityThread.H.sendMessage()
> ActivityThread.H.handleMessage()
> ActivityThread.handlePauseActivity()
> ActivityThread.performPauseActivity()
> Instrumentation.callActivityOnPause()
> Activity.performPause()
> Activity.onPause()
> ActivityManagerNative.getDefault().activityPaused()
> ActivityManagerService.activityPaused()
> ActivityStack.activityPausedLocked()
> ActivityStack.completePauseLocked()
> </font>

在方法startPausingLocked中我们调用了：prev.app.thread.schedulePauseActivity这里实际上调用的是IApplicationThread的schedulePauseActivity方法，IApplicationThread也是一个Binder对象，它是ActivityThread中ApplicationThread的Binder client端，所以最终会调用的是ApplicationThread的schedulePauseActivity方法，好吧我们看一下ActivityThread的schedulePauseActivity方法的具体实现：

```
public final void schedulePauseActivity(IBinder token, boolean finished, boolean userLeaving, int configChanges, boolean dontReport) {
   sendMessage(
       finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,
       token, (userLeaving ? 1 : 0) | (dontReport ? 2 : 0),
                    configChanges);
}
```
然后调用了ActivityThread的sendMessage方法：

```
private void sendMessage(int what, Object obj, int arg1, int arg2) {
        sendMessage(what, obj, arg1, arg2, false);
    }
```
然后又回调了sendMessage的重载方法。。

```
private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
        if (DEBUG_MESSAGES) Slog.v(
            TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
            + ": " + arg1 + " / " + obj);
        Message msg = Message.obtain();
        msg.what = what;
        msg.obj = obj;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        if (async) {
            msg.setAsynchronous(true);
        }
        mH.sendMessage(msg);
    }
```
最终调用mH发送异步消息，然后在mH的handleMessge方法中处理异步消息并调用handlePauseActivity方法：

```
private void handlePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges, boolean dontReport) {
        ActivityClientRecord r = mActivities.get(token);
        if (r != null) {
            //Slog.v(TAG, "userLeaving=" + userLeaving + " handling pause of " + r);
            if (userLeaving) {
                performUserLeavingActivity(r);
            }

            r.activity.mConfigChangeFlags |= configChanges;
            performPauseActivity(token, finished, r.isPreHoneycomb());

            // Make sure any pending writes are now committed.
            if (r.isPreHoneycomb()) {
                QueuedWork.waitToFinish();
            }

            // Tell the activity manager we have paused.
            if (!dontReport) {
                try {
                    ActivityManagerNative.getDefault().activityPaused(token);
                } catch (RemoteException ex) {
                }
            }
            mSomeActivitiesChanged = true;
        }
    }
```
好吧，这里回调了performPauseActivity方法，上篇文章中我们已经分析过了这段代码：
> performPauseActivity()
> Instrumentation.callActivityOnPause()
> Activity.performPause()
> Activity.onPause()

这样我们就回调了第一个生命周期方法：onPause。。。

在handlePauseActivity方法中我们调用了ActivityManagerNative.getDefault().activityPaused(token)方法，好吧又是回调ActivityManagerService的方法，这样最终会调用ActivityManagerService的activityPaused方法：

```
@Override
    public final void activityPaused(IBinder token) {
        final long origId = Binder.clearCallingIdentity();
        synchronized(this) {
            ActivityStack stack = ActivityRecord.getStackLocked(token);
            if (stack != null) {
                stack.activityPausedLocked(token, false);
            }
        }
        Binder.restoreCallingIdentity(origId);
    }
```
这样，我们继续看一下activityPausedLocked方法的实现：

```
final void activityPausedLocked(IBinder token, boolean timeout) {
        ...
        completePauseLocked(true);
        ...
}
```
里面又经过一系列的逻辑判断之后，开始执行completePauseLocked方法：

```
private void completePauseLocked(boolean resumeNext) {
	...                   mStackSupervisor.resumeTopActivitiesLocked(topStack, null, null);
	...
    }
```
这样栈顶Activity的onPause操作就执行完成了，接下来就就是开始执行上一个Activity的onResume操作了。。。

</br><strong><font size="6">三：执行上一个Activity的onResume操作</font></strong>
这样调用了ActivityStackSupervisor.resumeTopActivitiesLocked方法。。，又开始调用这个方法，通过上一篇文章的介绍，我们知道这个方法实际上是执行Activity的初始化，我们看一下其具体的调用过程：
> <font color="red">
>ActivityStack.resumeTopActivityLocked()
>ActivityStack.resumeTopInnerLocked()
>IApplicationThread.scheduleResumeActivity()
>ActivityThread.scheduleResumeActivity()
>ActivityThread.sendMessage()
>ActivityTherad.H.sendMessage()
>ActivityThread.H.handleMessage()
>ActivityThread.H.handleResumeMessage()
>Activity.performResume()
>Activity.performRestart()
>Instrumentation.callActivityOnRestart()
>Activity.onRestart()
>Activity.performStart()
>Instrumentation.callActivityOnStart()
>Activity.onStart()
>Instrumentation.callActivityOnResume()
>Activity.onResume()
></font>

好吧，这个过程其实上一篇文章中已经做了介绍，这里不做过多的分析了，通过这样调用过程我们最终执行了当前栈顶Activity上一个Activity的onRestart方法，onStart方法，onResume方法等，下面我们将调用栈顶Activity的onStop方法，onDestory方法。


</br><strong><font size="6">四：执行栈顶Activity的销毁操作

><font color="red">
>Looper.myQueue().addIdleHandler(new Idler())
>ActivityManagerNative.getDefault().activityIdle()
>ActivityManagerService.activityIdle()
>ActivityStackSupervisor.activityIdleInternalLocked()
>ActivityStack.destroyActivityLocked()
>IApplicationThread.scheduleDestoryActivity()
>ActivityThread.scheduleDestoryActivity()
>ActivityThread.sendMessage()
>ActivityThread.H.sendMessage()
>ActivityThread.H.handleMessage()
>ActivityThread.handleDestoryActivity()
>ActivityThread.performDestoryActivity()
>Activity.performStop()
>Instrumentation.callActivityOnStop()
>Activity.onStop()
>Instrumentation.callActivityOnDestory()
>Activity.performDestory()
>Acitivity.onDestory()
>ActivityManagerNative.getDefault().activityDestoryed()
>ActivityManagerService.activityDestoryed()
>ActivityStack.activityDestoryedLocked()
></font>

我们在ActivityThread.handleResumeActivity方法中调用了Looper.myQueue().addIdleHandler(new Idler())，下面看一下这个方法的实现：

```
private class Idler implements MessageQueue.IdleHandler {
        @Override
        public final boolean queueIdle() {
            ActivityClientRecord a = mNewActivities;
            boolean stopProfiling = false;
            if (mBoundApplication != null && mProfiler.profileFd != null
                    && mProfiler.autoStopProfiler) {
                stopProfiling = true;
            }
            if (a != null) {
                mNewActivities = null;
                IActivityManager am = ActivityManagerNative.getDefault();
                ActivityClientRecord prev;
                do {
                    if (localLOGV) Slog.v(
                        TAG, "Reporting idle of " + a +
                        " finished=" +
                        (a.activity != null && a.activity.mFinished));
                    if (a.activity != null && !a.activity.mFinished) {
                        try {
                            am.activityIdle(a.token, a.createdConfig, stopProfiling);
                            a.createdConfig = null;
                        } catch (RemoteException ex) {
                            // Ignore
                        }
                    }
                    prev = a;
                    a = a.nextIdle;
                    prev.nextIdle = null;
                } while (a != null);
            }
            if (stopProfiling) {
                mProfiler.stopProfiling();
            }
            ensureJitEnabled();
            return false;
        }
    }
```
内部有一个queueIdle的回调方法，当它被添加到MessageQueue之后就会回调该方法，我们可以发现在这个方法体中调用了ActivityManagerNative.getDefault.activityIdle方法，通过上一篇文章以及上面的讲解，我们应该知道这了最终调用的是ActivityManagerService.activityIdle方法，好吧，这里看一下activityIdle方法的具体实现：

```
@Override
    public final void activityIdle(IBinder token, Configuration config, boolean stopProfiling) {
        final long origId = Binder.clearCallingIdentity();
        synchronized (this) {
            ActivityStack stack = ActivityRecord.getStackLocked(token);
            if (stack != null) {
                ActivityRecord r =
                        mStackSupervisor.activityIdleInternalLocked(token, false, config);
                if (stopProfiling) {
                    if ((mProfileProc == r.app) && (mProfileFd != null)) {
                        try {
                            mProfileFd.close();
                        } catch (IOException e) {
                        }
                        clearProfilerLocked();
                    }
                }
            }
        }
        Binder.restoreCallingIdentity(origId);
    }
```
可以发现这里又调用了ActivityStackSupervisor.activityIdleInternalLocked方法，然后我们看一下activityIdleInternalLocked方法的具体实现：

```
final ActivityRecord activityIdleInternalLocked(final IBinder token, boolean fromTimeout, Configuration config) {
    ....   
    stack.destroyActivityLocked(r, true, "finish-idle");
    ....    
}
```
可以看到这里调用ActivityStack.destroyActivityLocked方法，可以看一下其具体实现：

```
final boolean destroyActivityLocked(ActivityRecord r, boolean removeFromApp, String reason) {
      ...
      r.app.thread.scheduleDestroyActivity(r.appToken, r.finishing, r.configChangeFlags);
      ...      
}
```
好吧，这里又开始执行IApplicationThread.scheduleDestoryActivity方法，上文已经做了说明这里最终调用的是ActivityThread.scheduleDestroyActivity方法，好吧，看一下ActivityThread.scheduleDestryActivity方法的实现：

```
public final void scheduleDestroyActivity(IBinder token, boolean finishing, int configChanges) {
    sendMessage(H.DESTROY_ACTIVITY, token, finishing ? 1 : 0,
                    configChanges);
}
```
这里有开始执行sendMessage方法，通过一系列的调用sendMessage方法最终调用了handleDestroyActivity方法：

```
private void handleDestroyActivity(IBinder token, boolean finishing,
            int configChanges, boolean getNonConfigInstance) {
        ActivityClientRecord r = performDestroyActivity(token, finishing,
                configChanges, getNonConfigInstance);
        if (r != null) {
            cleanUpPendingRemoveWindows(r);
            WindowManager wm = r.activity.getWindowManager();
            View v = r.activity.mDecor;
            if (v != null) {
                if (r.activity.mVisibleFromServer) {
                    mNumVisibleActivities--;
                }
                IBinder wtoken = v.getWindowToken();
                if (r.activity.mWindowAdded) {
                    if (r.onlyLocalRequest) {
                        // Hold off on removing this until the new activity's
                        // window is being added.
                        r.mPendingRemoveWindow = v;
                        r.mPendingRemoveWindowManager = wm;
                    } else {
                        wm.removeViewImmediate(v);
                    }
                }
                if (wtoken != null && r.mPendingRemoveWindow == null) {
                    WindowManagerGlobal.getInstance().closeAll(wtoken,
                            r.activity.getClass().getName(), "Activity");
                }
                r.activity.mDecor = null;
            }
            if (r.mPendingRemoveWindow == null) {
                // If we are delaying the removal of the activity window, then
                // we can't clean up all windows here.  Note that we can't do
                // so later either, which means any windows that aren't closed
                // by the app will leak.  Well we try to warning them a lot
                // about leaking windows, because that is a bug, so if they are
                // using this recreate facility then they get to live with leaks.
                WindowManagerGlobal.getInstance().closeAll(token,
                        r.activity.getClass().getName(), "Activity");
            }

            // Mocked out contexts won't be participating in the normal
            // process lifecycle, but if we're running with a proper
            // ApplicationContext we need to have it tear down things
            // cleanly.
            Context c = r.activity.getBaseContext();
            if (c instanceof ContextImpl) {
                ((ContextImpl) c).scheduleFinalCleanup(
                        r.activity.getClass().getName(), "Activity");
            }
        }
        if (finishing) {
            try {
                ActivityManagerNative.getDefault().activityDestroyed(token);
            } catch (RemoteException ex) {
                // If the system process has died, it's game over for everyone.
            }
        }
        mSomeActivitiesChanged = true;
    }
```
可以看到这里调用了performDestroyActivity方法，用来执行Avtivity的onDestroy方法：

```
private ActivityClientRecord performDestroyActivity(IBinder token, boolean finishing,
            int configChanges, boolean getNonConfigInstance) {
       ...     
       r.activity.performStop();
       ...
       mInstrumentation.callActivityOnDestroy(r.activity);
	   ...
    }
```
然后调用了Activity.performStop()方法，查看performStop方法：

```
final void performStop() {
        ...
        mInstrumentation.callActivityOnStop(this);
        ...
}
```
然后调用了Instrumentation.callActivityOnStop()方法：

```
public void callActivityOnStop(Activity activity) {
        activity.onStop();
    }
```
好吧，终于调用了Activity的onStop方法。。。

我们继续看一下Instrumentation.callActivityOnDestroy()。。。。又是通过Instrumentation来调用Activity的onDestroy方法：

```
public void callActivityOnDestroy(Activity activity) {
    ...
    activity.performDestroy();
	...
}
```
然后看一下Activity的performDestroy()方法的实现：

```
final void performDestroy() {
        mDestroyed = true;
        mWindow.destroy();
        mFragments.dispatchDestroy();
        onDestroy();
        mFragments.doLoaderDestroy();
        if (mVoiceInteractor != null) {
            mVoiceInteractor.detachActivity();
        }
    }
```
O(∩_∩)O哈哈~，终于回调了Activity的onDestroy方法。。。。


</br></br></br>
<strong><font size="5">总结：

- Activity的销毁流程是从finish方法开始的

- Activity销毁过程是：onPause --> onRestart --> onStart --> onResume --> onStop --> onDestroy

- Activity的销毁流程是ActivityThread与ActivityManagerService相互配合销毁的

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
