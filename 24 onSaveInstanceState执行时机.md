我们已经分析过Activity的启动流程，从中也分析了Activity的生命周期。而其中有一个生命周期方法:onSaveInstanceState方法，今天我们主要讲解一下onSaveInstanceState方法的执行时机。
可能部分同学对Activity的onSaveInstanceState方法不是特别熟悉，这里我们简单介绍一下。onSaveInstanceState方法是Activity的成员方法，主要用于在Activity销毁时保存Activity相关的对象信息，而其执行的时机不是我们主动调用的，而是Android系统的framework帮忙调用的，而其调用的时机，可以参考android系统的介绍：

> This method is called before an activity may be killed so that when it comes back some time in the future it can restore its state.  For example, if activity B is launched in front of activity A, and at some point activity A is killed to reclaim resources, activity A will have a chance to save the current state of its user interface via this method so that when the user returns to activity A, the state of the user interface can be restored via {@link #onCreate} or {@link #onRestoreInstanceState}.

可以发现onSaveInstanceState方法会在Activity将要被kill的时候执行。O(∩_∩)O哈哈~，可能跟以前讲解的内容不是太对，我们看过不少文章都是说onSaveInstanceStatex方法会在Activity容易被销毁的时候执行。那么这里明明说的是当Activity被销毁的时候就会执行onSaveInstanceState方法，那么具体的情况是如何的呢?我们具体看一下源码吧，哈哈。

通过分析Activity的生命周期方法，我们知道onSaveInstanceState方法在onPause方法之后执行在onStop方法之前执行。这里我们首先看一下onPause方法的源码逻辑。

Activity在执行onPause方法的时候回回调ActivityThread的handlePauseActivity方法，不太熟悉的同学可以参考:<a href="http://blog.csdn.net/qq_23547831/article/details/51224992"> android源码解析之（十四）-->Activity启动流程</a>，文章中有对Activity生命周期的详细讲解。

好吧，先具体看一下ActivityThread.handlePauseActivity的源码：

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
在方法体中我们除了执行一些其他的操作，然后在handlePauseActivity方法体中调用了performPauseActivity方法，这个方法就是具体执行回调pauseActivity操作的方法，既然这样我们在看一下performPauseActivity方法的实现：

```
final Bundle performPauseActivity(IBinder token, boolean finished,
            boolean saveState) {
        ActivityClientRecord r = mActivities.get(token);
        return r != null ? performPauseActivity(r, finished, saveState) : null;
    }
```
可以发现在performPauseActivity方法中首先判断ActivityClientRecord是否为空，然后又调用了performPauseActivity方法的重载方法：

```
final Bundle performPauseActivity(ActivityClientRecord r, boolean finished,
            boolean saveState) {
        ...
        if (!r.activity.mFinished && saveState) {
            callCallActivityOnSaveInstanceState(r);
        }
        ...
    }
```
可以发现，这里调用了callCallActivityOnSaveInstanceState方法，看名称可以发现这里应该回调的是Activity的onSaveInstanceState方法，但是这里执行之前有一个条件判断，首先会判断这里的Activity是否被finish？应为这时候刚刚执行onPause方法所以这里的mFinished变量为false，所以判断执行callCallActivityOnSaveInstanceState方法只要需要通过saveState变量来判断了，而这里的saveState方法是performPauseActivity方法传递过来的。。。。好吧，我们来看一下调用performPauseActivity方法时saveState变量是如何赋值的。回到我们的handlePauseActivity方法，看一下performPauseActivity方法是如何调用的：

```
performPauseActivity(token, finished, r.isPreHoneycomb());
```
可以发现saveState boolean变量是通过r.isPreHoneycomb方法赋值的，这里我们看一下IsPreHoneycomb方法是如何实现的：

```
public boolean isPreHoneycomb() {
            if (activity != null) {
                return activity.getApplicationInfo().targetSdkVersion
                        < android.os.Build.VERSION_CODES.HONEYCOMB;
            }
            return false;
        }
```
可以发现当我们的App设置的targetSdk版本号小于android versionCode 11也就是android3.0的时候返回为true，其他的时候返回为false，也就是说当我们App设置的targetVersion大于android3.0的时候才会执行callCallActivityOnSaveInstanceState方法，好吧，继续看一下callCallActivityOnSaveInstanceState方法是如何实现的：

```
private void callCallActivityOnSaveInstanceState(ActivityClientRecord r) {
        r.state = new Bundle();
        r.state.setAllowFds(false);
        if (r.isPersistable()) {
            r.persistentState = new PersistableBundle();
            mInstrumentation.callActivityOnSaveInstanceState(r.activity, r.state,
                    r.persistentState);
        } else {
            mInstrumentation.callActivityOnSaveInstanceState(r.activity, r.state);
        }
    }
```
可以发现方法体主要调用了mInstrumentation的callActivityOnSaveInstanceState方法，既然这样，我们再来看一下callActivityOnSaveInstanceState方法：

```
public void callActivityOnSaveInstanceState(Activity activity, Bundle outState,
            PersistableBundle outPersistentState) {
        activity.performSaveInstanceState(outState, outPersistentState);
    }
```
这里方法体中又回调了Activity的performSaveInstanceState方法。。。

```
final void performSaveInstanceState(Bundle outState) {
        onSaveInstanceState(outState);
        saveManagedDialogs(outState);
        mActivityTransitionState.saveState(outState);
        if (DEBUG_LIFECYCLE) Slog.v(TAG, "onSaveInstanceState " + this + ": " + outState);
    }
```
可以看到这里回调了Activity的onSaveInstanceState方法，这样经过一系列的方法回调之后我们就执行了onSaveInstanceState方法。

这样我们当只执行onPause方法的时候一般通过设置targetVersion控制是否执行onSaveInstanceState方法，当设置的targetVersionCode大于android3.0的时候默认不会执行onSaveInstanceState方法。


然后我们看一下当Activity执行onStop方法的时候是否会执行onSaveInstanceState方法，通过之前分析的Activity的启动流程，我们知道Actvitiy执行onStop方法会回调ActivityThread的handleStopActivity，这样我们先看一下handleStopActivity方法的实现：

```
private void handleStopActivity(IBinder token, boolean show, int configChanges) {
        ActivityClientRecord r = mActivities.get(token);
        r.activity.mConfigChangeFlags |= configChanges;

        StopInfo info = new StopInfo();
        performStopActivityInner(r, info, show, true);

        if (localLOGV) Slog.v(
            TAG, "Finishing stop of " + r + ": show=" + show
            + " win=" + r.window);

        updateVisibility(r, show);

        info.activity = r;
        info.state = r.state;
        info.persistentState = r.persistentState;
        mH.post(info);
        mSomeActivitiesChanged = true;
    }
```
然后我们发现在方法performStopActivity方法中调用了performStopActivityInner方法，我们继续看一下performStopActivityInner方法的实现：

```
private void performStopActivityInner(ActivityClientRecord r,
            StopInfo info, boolean keepShown, boolean saveState) {
        ...
            if (!r.activity.mFinished && saveState) {
                if (r.state == null) {
                    callCallActivityOnSaveInstanceState(r);
                }
            }
            ...
    }
```
可以发现还是通过saveState变量来控制是否调用onSaveInstanceState，而这里的saveState变量是在performStopActivityInner方法调用的时候传递的，回到我们的handleStopActivity方法中关于performStopActivityInner调用的代码：

```
performStopActivityInner(r, info, show, true);
```
好吧，这里直接传值为true，这样我们执行Activity的stop方法一定执行onSaveInstanceState方法。


总结

- onSaveInstanceState方法是Activity的生命周期方法，主要用于在Activity销毁时保存一些信息。

- 当Activity只执行onPause方法时（Activity a打开一个透明Activity b）这时候如果App设置的targetVersion大于android3.0则不会执行onSaveInstanceState方法。

- 当Activity执行onStop方法时，通过分析源码我们知道调用onSaveInstanceState的方法直接传值为true，所以都会执行onSaveInstanceState方法。

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
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51374627">android源码解析（二十二）-->Toast加载绘制流程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51382326">android源码解析（二十三）-->Android异常处理流程</a>

