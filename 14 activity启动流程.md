好吧，终于要开始讲解Activity的启动流程了，Activity的启动流程相对复杂一下，涉及到了Activity中的生命周期方法，涉及到了Android体系的CS模式，涉及到了Android中进程通讯Binder机制等等，

首先介绍一下Activity，这里引用一下Android guide中对Activity的介绍：

> An activity represents a single screen with a user interface. For example, an email application might have one activity that shows a list of new emails, another activity to compose an email, and another activity for reading emails. Although the activities work together to form a cohesive user experience in the email application, each one is independent of the others. As such, a different application can start any one of these activities (if the email application allows it). For example, a camera application can start the activity in the email application that composes new mail, in order for the user to share a picture.

英文不太好，这里就不献丑了，这里介绍的Activity的大概意思就是说，activity在Android系统中代表的就是一个屏幕，一个App就是由许多个不同的Acitivty组成的，并且不同进程之间的Activity是可以相互调用的。

在介绍Activity的启动流程之前，我们先介绍几个概念：

- Activity的生命周期

> protected void onCreate(Bundle savedInstanceState);
> protected void onRestart();
> protected void onStart();
> protected void onResume();
> protected void onPause();
> protected void onStop();
> protected void onDestory();
> 以上为Activity生命周期中的各个时期的回调方法，在不同的方法中我们可以执行不同的逻辑。
关于Activity生命周期的详细介绍可以参考：<a href="http://blog.csdn.net/qq_23547831/article/details/41693807"> Android activity的生命周期</a>

- Activity的启动模式

> activity启动时可以设置不同的启动模式，主要是：standrand，singleTop，singleTask，instance等四种启动模式，不同的启动模式在启动Activity时会执行不同的逻辑，系统会按不同的启动模式将Activity存放到不同的activity栈中。
关于Activity启动模式的详细介绍，可以参考：<a href="http://blog.csdn.net/qq_23547831/article/details/46534693"> Android任务和返回栈完全解析</a>

- Activity的启动进程

> 在Manifest.xml中定义Activity的时候，Activity默认是属于进程名称为包名的进程的，当然这时候是可以指定Activity的启动进程，所以在Activity启动时首先会检测当前Activity所属的进程是否已经启动，若进程没有启动，则首先会启动该进程，并在该进程启动之后才会执行Activity的启动过程。

- Intent启动Activity的方式
> Intent启动Activity分为两种，显示启动和隐士启动，显示启动就是在初始化Intent对象的时候直接引用需要启动的Activity的字节码，显示引用的好处就是可以直接告诉Intent对象启动的Activity对象不需要执行intent filter索引需要启动哪一个Activity，但是显示引用不能启动其他进程的Activity对象，因为无法获取其他进程的Activity对象的字节码，而隐式启动则可以通过配置Intent Filter启动其他进程的Activity对象，因此在应用内，我们一般都是使用显示启动的方式启动Activity，而如果需要启动其他应用的Activity时，一般使用隐式启动的方式。

- Android Framework层的CS模式
通过前几篇文章的介绍我们知道android系统在启动过程中会执行这样的逻辑：
        Zygote进程 --> SystemServer进程 --> 各种系统服务 --> 应用进程
在Actvity启动过程中，其实是应用进程与SystemServer进程相互配合启动Activity的过程，其中应用进程主要用于执行具体的Activity的启动过程，回调生命周期方法等操作，而SystemServer进程则主要是调用其中的各种服务，将Activity保存在栈中，协调各种系统资源等操作。

- Android系统进程间通讯Binder机制
Android系统存了Zygote进程和SystemServer进程以及各种应用进程等，为了能够实现各种进程之间的通讯，Android系统采用了自己的进程间通讯方式Binder机制。其中主要涉及到了四种角色：Binder Client，Binder Server，Binder Manager， Binder driver。各种角色之间的关系可以参考下面这张图的介绍：
![这里写图片描述](http://img.blog.csdn.net/20160423105932970)

好吧，前面我们介绍了一些Activity启动过程中需要的相关知识点，下面我们开始Activity启动流程的讲解。。。。

还记得前面我们讲过的Launcher启动流程么？可以参考：<a href="http://blog.csdn.net/qq_23547831/article/details/51112031">android源码解析之（十）-->Launcher启动流程</a>
在这篇文章中我们说Launcher启动之后会将各个应用包名和icon与app name保存起来，然后执行icon的点击事件的时候调用startActivity方法：

```
@Override
    protected void onListItemClick(ListView l, View v, int position, long id) {
        Intent intent = intentForPosition(position);
        startActivity(intent);
    }

protected Intent intentForPosition(int position) {
        ActivityAdapter adapter = (ActivityAdapter) mAdapter;
        return adapter.intentForPosition(position);
    }

public Intent intentForPosition(int position) {
            if (mActivitiesList == null) {
                return null;
            }

            Intent intent = new Intent(mIntent);
            ListItem item = mActivitiesList.get(position);
            intent.setClassName(item.packageName, item.className);
            if (item.extras != null) {
                intent.putExtras(item.extras);
            }
            return intent;
        }
```
可以发现，我们在启动Activity的时候，执行的逻辑就是创建一个Intent对象，然后初始化Intent对象，使用隐式启动的方式启动该Acvitity，这里为什么不能使用显示启动的方式呢？

这是因为Launcher程序启动的Activity一般都是启动一个新的应用进程，该进程与Launcher进程不是在同一个进程中，所以也就无法引用到启动的Activity字节码，自然也就无法启动该Activity了。

继续，我们查看startActivity方法的具体实现：

</br><strong><font size="6" >一:开始请求执行启动Activity</font><strong>
> <font color="red">
> MyActivity.startActivity()
> Activity.startActivity()
> Activity.startActivityForResult
> Instrumentation.execStartActivty
> ActivityManagerNative.getDefault().startActivityAsUser()
> </font>

在我们的Activity中调用startActivity方法，会执行Activity中的startActivity
```
@Override
    public void startActivity(Intent intent) {
        this.startActivity(intent, null);
    }
```
然后在Activity中的startActivity方法体里调用了startActivity的重载方法，这里我们看一下其重载方法的实现：

```
@Override
    public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }
```
由于在上一步骤中我们传递的Bunde对象为空，所以这里我们执行的是else分支的逻辑，所以这里调用了startActivityForResult方法，并且传递的参数为intent和-1.

<font color="#A52A2A">**注意：**通过这里的代码我们可以发现，其实我们在Activity中调用startActivity的内部也是调用的startActivityForResult的。那么为什么调用startActivityForResult可以在Activity中回调onActivityResult而调用startActivity则不可以呢？可以发现其主要的区别是调用startActivity内部调用startActivityForResult传递的传输requestCode值为-1，也就是说我们在Activity调用startActivityForResult的时候传递的requestCode值为-1的话，那么onActivityResult是不起作用的。
实际上，经测试requestCode的值小于0的时候都是不起作用的，所以当我们调用startActivityForResult的时候需要注意这一点。</font>

好吧，我们继续往下看，startActivityForResult方法的具体实现：

```
public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
        if (mParent == null) {
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
                // If this start is requesting a result, we can avoid making
                // the activity visible until the result is received.  Setting
                // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
                // activity hidden during this time, to avoid flickering.
                // This can only be done when a result is requested because
                // that guarantees we will get information back when the
                // activity is finished, no matter what happens to it.
                mStartedActivity = true;
            }

            cancelInputsAndStartExitTransition(options);
            // TODO Consider clearing/flushing other event sources and events for child windows.
        } else {
            if (options != null) {
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                // Note we want to go through this method for compatibility with
                // existing applications that may have overridden it.
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
    }
```
可以发现由于我们是第一次启动Activity，所以这里的mParent为空，所以会执行if分之，然后调用mInstrumentation.execStartActivity方法，并且这里需要注意的是，有一个判断逻辑：

```
if (requestCode >= 0) {
	mStartedActivity = true;
}
```
通过注释也验证了我们刚刚的说法即，调用startActivityForResult的时候只有requestCode的值大于等于0，onActivityResult才会被回调。

然后我们看一下mInstrumentation.execStartActivity方法的实现。在查看execStartActivity方法之前，我们需要对mInstrumentation对象有一个了解？什么是Instrumentation？Instrumentation是android系统中启动Activity的一个实际操作类，也就是说Activity在应用进程端的启动实际上就是Instrumentation执行的，那么为什么说是在应用进程端的启动呢？实际上acitivty的启动分为应用进程端的启动和SystemServer服务进程端的启动的，多个应用进程相互配合最终完成了Activity在系统中的启动的，而在应用进程端的启动实际的操作类就是Intrumentation来执行的，可能还是有点绕口，没关系，随着我们慢慢的解析大家就会对Instrumentation的认识逐渐加深的。

可以发现execStartActivity方法传递的几个参数：
this，为启动Activity的对象；
contextThread，为Binder对象，是主进程的context对象；
token，也是一个Binder对象，指向了服务端一个ActivityRecord对象；
target，为启动的Activity；
intent，启动的Intent对象；
requestCode，请求码；
options，参数；

这样就调用了Imstrument.execStartActivity方法了：

```
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        ...
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess();
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```
我们发现在这个方法中主要调用ActivityManagerNative.getDefault().startActivity方法，那么ActivityManagerNative又是个什么鬼呢？查看一下getDefault()对象的实现：

```
static public IActivityManager getDefault() {
        return gDefault.get();
    }
```
好吧，相当之简单直接返回的是gDefault.get()，那么gDefault又是什么呢？

```
private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
    };
```
可以发现启动过asInterface()方法创建，然后我们继续看一下asInterface方法的实现：

```
static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IActivityManager in =
            (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }

        return new ActivityManagerProxy(obj);
    }
```
好吧，最后直接返回一个ActivityManagerProxy对象，而ActivityManagerProxy继承与IActivityManager，到了这里就引出了我们android系统中很重要的一个概念：Binder机制。我们知道应用进程与SystemServer进程属于两个不同的进程，进程之间需要通讯，android系统采取了自身设计的Binder机制，这里的ActivityManagerProxy和ActivityManagerNative都是继承与IActivityManager的而SystemServer进程中的ActivityManagerService对象则继承与ActivityManagerNative。简单的表示：
Binder接口 --> ActivityManagerNative/ActivityManagerProxy --> ActivityManagerService；

这样，ActivityManagerNative与ActivityManagerProxy相当于一个Binder的客户端而ActivityManagerService相当于Binder的服务端，这样当ActivityManagerNative调用接口方法的时候底层通过Binder driver就会将请求数据与请求传递给server端，并在server端执行具体的接口逻辑。需要注意的是Binder机制是单向的，是异步的，也就是说只能通过client端向server端传递数据与请求而不同等待服务端的返回，也无法返回，那如果SystemServer进程想向应用进程传递数据怎么办？这时候就需要重新定义一个Binder请求以SystemServer为client端，以应用进程为server端，这样就是实现了两个进程之间的双向通讯。

好了，说了这么多我们知道这里的ActivityManagerNative是ActivityManagerService在应用进程的一个client就好了，通过它就可以滴啊用ActivityManagerService的方法了。

继续往下卡，我们调用的是：

```
int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
```
这里通过我们刚刚的分析，ActivityManagerNative.getDefault()方法会返回一个ActivityManagerProxy对象，那么我们看一下ActivityManagerProxy对象的startActivity方法：

```
public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        data.writeString(callingPackage);
        intent.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeStrongBinder(resultTo);
        data.writeString(resultWho);
        data.writeInt(requestCode);
        data.writeInt(startFlags);
        if (profilerInfo != null) {
            data.writeInt(1);
            profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
        } else {
            data.writeInt(0);
        }
        if (options != null) {
            data.writeInt(1);
            options.writeToParcel(data, 0);
        } else {
            data.writeInt(0);
        }
        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        int result = reply.readInt();
        reply.recycle();
        data.recycle();
        return result;
    }
```
这里就涉及到了具体的Binder数据传输机制了，我们不做过多的分析，知道通过数据传输之后就会调用SystemServer进程的ActivityManagerService的startActivity就好了。

以上其实都是发生在应用进程中，下面开始调用的ActivityManagerService的执行时发生在SystemServer进程。

<br/><strong><font size="6">二：ActivityManagerService接收启动Activity的请求</font></strong>

><font color="red">
> ActivityManagerService.startActivity() 
> ActvityiManagerService.startActivityAsUser() 
> ActivityStackSupervisor.startActivityMayWait() 
> ActivityStackSupervisor.startActivityLocked() 
> ActivityStackSupervisor.startActivityUncheckedLocked() 
> ActivityStackSupervisor.startActivityLocked() 
> ActivityStackSupervisor.resumeTopActivitiesLocked()
> ActivityStackSupervisor.resumeTopActivityInnerLocked() 
> </font>

好吧，代码量比较大，慢慢看，首先看一下ActivityManagerService.startActivity的具体实现；

```
@Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, options,
            UserHandle.getCallingUserId());
    }
```
可以看到，该方法并没有实现什么逻辑，直接调用了startActivityAsUser方法，我们继续看一下startActivityAsUser方法的实现：

```
@Override
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options, int userId) {
        enforceNotIsolatedCaller("startActivity");
        userId = handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(), userId,
                false, ALLOW_FULL_ONLY, "startActivity", null);
        // TODO: Switch to user app stacks here.
        return mStackSupervisor.startActivityMayWait(caller, -1, callingPackage, intent,
                resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
                profilerInfo, null, null, options, false, userId, null, null);
    }
```
可以看到这里只是进行了一些关于userid的逻辑判断，然后就调用mStackSupervisor.startActivityMayWait方法，下面我们来看一下这个方法的具体实现：

```
final int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, WaitResult outResult, Configuration config,
            Bundle options, boolean ignoreTargetSecurity, int userId,
            IActivityContainer iContainer, TaskRecord inTask) {
		    ...

            int res = startActivityLocked(caller, intent, resolvedType, aInfo,
                    voiceSession, voiceInteractor, resultTo, resultWho,
                    requestCode, callingPid, callingUid, callingPackage,
                    realCallingPid, realCallingUid, startFlags, options, ignoreTargetSecurity,
                    componentSpecified, null, container, inTask);
		    ...
            return res;
    }
```
这个方法中执行了启动Activity的一些其他逻辑判断，在经过判断逻辑之后调用startActivityLocked方法：

```
final int startActivityLocked(IApplicationThread caller,
            Intent intent, String resolvedType, ActivityInfo aInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode,
            int callingPid, int callingUid, String callingPackage,
            int realCallingPid, int realCallingUid, int startFlags, Bundle options,
            boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
            ActivityContainer container, TaskRecord inTask) {
        int err = ActivityManager.START_SUCCESS;

        ...
        err = startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, true, options, inTask);

        ...
        return err;
    }
```
这个方法中主要构造了ActivityManagerService端的Activity对象-->ActivityRecord，并根据Activity的启动模式执行了相关逻辑。然后调用了startActivityUncheckedLocked方法：

```
final int startActivityUncheckedLocked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, int startFlags,
            boolean doResume, Bundle options, TaskRecord inTask) {
        ...
        ActivityStack.logStartActivity(EventLogTags.AM_CREATE_ACTIVITY, r, r.task);
        targetStack.mLastPausedActivity = null;
        targetStack.startActivityLocked(r, newTask, doResume, keepCurTransition, options);
        if (!launchTaskBehind) {
            // Don't set focus on an activity that's going to the back.
            mService.setFocusedActivityLocked(r, "startedActivity");
        }
        return ActivityManager.START_SUCCESS;
    }
```
startActivityUncheckedLocked方法中只要执行了不同启动模式不同栈的处理，并最后调用了startActivityLocked的重载方法：

```
final void startActivityLocked(ActivityRecord r, boolean newTask,
            boolean doResume, boolean keepCurTransition, Bundle options) {
		...
        if (doResume) {
            mStackSupervisor.resumeTopActivitiesLocked(this, r, options);
        }
    }
```
这个startActivityLocked方法主要执行初始化了windowManager服务，然后调用resumeTopActivitiesLocked方法：

```
boolean resumeTopActivitiesLocked(ActivityStack targetStack, ActivityRecord target,
            Bundle targetOptions) {
        if (targetStack == null) {
            targetStack = mFocusedStack;
        }
        // Do targetStack first.
        boolean result = false;
        if (isFrontStack(targetStack)) {
            result = targetStack.resumeTopActivityLocked(target, targetOptions);
        }

        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            final ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = stacks.get(stackNdx);
                if (stack == targetStack) {
                    // Already started above.
                    continue;
                }
                if (isFrontStack(stack)) {
                    stack.resumeTopActivityLocked(null);
                }
            }
        }
        return result;
    }
```
可以发现经过循环逻辑判断之后，最终调用了resumeTopActivityLocked方法：

```
final boolean resumeTopActivityLocked(ActivityRecord prev) {
        return resumeTopActivityLocked(prev, null);
    }
```
然后调用：

```
final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
        if (mStackSupervisor.inResumeTopActivity) {
            // Don't even start recursing.
            return false;
        }

        boolean result = false;
        try {
            // Protect against recursion.
            mStackSupervisor.inResumeTopActivity = true;
            if (mService.mLockScreenShown == ActivityManagerService.LOCK_SCREEN_LEAVING) {
                mService.mLockScreenShown = ActivityManagerService.LOCK_SCREEN_HIDDEN;
                mService.updateSleepIfNeededLocked();
            }
            result = resumeTopActivityInnerLocked(prev, options);
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }
        return result;
    }
```
继续调用resumeTopActivityInnerLocked方法：

```
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, Bundle options) {

		...
		if (mResumedActivity != null) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Pausing " + mResumedActivity);
            pausing |= startPausingLocked(userLeaving, false, true, dontWaitForPause);
        }        ...
        return true;
    }
```

经过一系列处理逻辑之后最终调用了startPausingLocked方法，这个方法作用就是让系统中栈中的Activity执行onPause方法。

<br><strong><font size="6">三：执行栈顶Activity的onPause方法

><font color="red">
> ActivityStack.startPausingLocked()
>IApplicationThread.schudulePauseActivity()
>ActivityThread.sendMessage()
>ActivityThread.H.sendMessage();
>ActivityThread.H.handleMessage()
>ActivityThread.handlePauseActivity()
>ActivityThread.performPauseActivity()
>Activity.performPause()
>Activity.onPause()
>ActivityManagerNative.getDefault().activityPaused(token)
>ActivityManagerService.activityPaused()
>ActivityStack.activityPausedLocked()
>ActivityStack.completePauseLocked()
>ActivityStack.resumeTopActivitiesLocked()
>ActivityStack.resumeTopActivityLocked()
>ActivityStack.resumeTopActivityInnerLocked()
>ActivityStack.startSpecificActivityLocked
></font>

好吧，方法比较多也比较乱，首先来看startPausingLocked方法：

```
final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping, boolean resuming,
            boolean dontWait) {
        ...
        if (prev.app != null && prev.app.thread != null) {
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Enqueueing pending pause: " + prev);
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
        } else {
            mPausingActivity = null;
            mLastPausedActivity = null;
            mLastNoHistoryActivity = null;
        }
        ...
    }
```
可以看到这里执行了pre.app.thread.schedulePauseActivity方法，通过分析不难发现这里的thread是一个IApplicationThread类型的对象，而在ActivityThread中也定义了一个ApplicationThread的类，其继承了IApplicationThread，并且都是Binder对象，不难看出这里的IAppcation是一个Binder的client端而ActivityThread中的ApplicationThread是一个Binder对象的server端，所以通过这里的thread.schedulePauseActivity实际上调用的就是ApplicationThread的schedulePauseActivity方法。

<font color="red">这里的ApplicationThread可以和ActivityManagerNative对于一下：
通过ActivityManagerNative -->   ActivityManagerService实现了应用进程与SystemServer进程的通讯
通过AppicationThread          <--    IApplicationThread实现了SystemServer进程与应用进程的通讯</font>

然后我们继续看一下ActivityThread中schedulePauseActivity的具体实现：

```
public final void schedulePauseActivity(IBinder token, boolean finished,
                boolean userLeaving, int configChanges, boolean dontReport) {
            sendMessage(
                    finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,
                    token,
                    (userLeaving ? 1 : 0) | (dontReport ? 2 : 0),
                    configChanges);
        }
```
发送了PAUSE_ACTIVITY_FINISHING消息，然后看一下sendMessage的实现方法：

```
private void sendMessage(int what, Object obj, int arg1, int arg2) {
        sendMessage(what, obj, arg1, arg2, false);
    }
```
调用了其重载方法：

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
最终调用了mH的sendMessage方法，mH是在ActivityThread中定义的一个Handler对象，主要处理SystemServer进程的消息，我们看一下其handleMessge方法的实现：

```
public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                ...
                case PAUSE_ACTIVITY_FINISHING:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
                    handlePauseActivity((IBinder)msg.obj, true, (msg.arg1&1) != 0, msg.arg2,
                            (msg.arg1&1) != 0);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
				...
}
```
可以发现其调用了handlePauseActivity方法：

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
然后在方法体内部通过调用performPauseActivity方法来实现对栈顶Activity的onPause生命周期方法的回调，可以具体看一下他的实现：

```
final Bundle performPauseActivity(IBinder token, boolean finished,
            boolean saveState) {
        ActivityClientRecord r = mActivities.get(token);
        return r != null ? performPauseActivity(r, finished, saveState) : null;
    }
```
然后调用其重载方法：

```
final Bundle performPauseActivity(ActivityClientRecord r, boolean finished,
            boolean saveState) {
        ...
        mInstrumentation.callActivityOnPause(r.activity);
        ...

        return !r.activity.mFinished && saveState ? r.state : null;
    }
```
这样回到了mInstrumentation的callActivityOnPuase方法：

```
public void callActivityOnPause(Activity activity) {
        activity.performPause();
    }
```
呵呵，原来最终回调到了Activity的performPause方法：

```
final void performPause() {
        mDoReportFullyDrawn = false;
        mFragments.dispatchPause();
        mCalled = false;
        onPause();
        mResumed = false;
        if (!mCalled && getApplicationInfo().targetSdkVersion
                >= android.os.Build.VERSION_CODES.GINGERBREAD) {
            throw new SuperNotCalledException(
                    "Activity " + mComponent.toShortString() +
                    " did not call through to super.onPause()");
        }
        mResumed = false;
    }
```
终于，太不容易了，回调到了Activity的onPause方法，哈哈，Activity生命周期中的第一个生命周期方法终于被我们找到了。。。。也就是说我们在启动一个Activity的时候最先被执行的是栈顶的Activity的onPause方法。记住这点吧，面试的时候经常会问到类似的问题。

然后回到我们的handlePauseActivity方法，在该方法的最后面执行了ActivityManagerNative.getDefault().activityPaused(token);方法，这是应用进程告诉服务进程，栈顶Activity已经执行完成onPause方法了，通过前面我们的分析，我们知道这句话最终会被ActivityManagerService的activityPaused方法执行。

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
可以发现，该方法内部会调用ActivityStack的activityPausedLocked方法，好吧，继续看一下activityPausedLocked方法的实现：

```
final void activityPausedLocked(IBinder token, boolean timeout) {
	        ...
                if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to PAUSED: " + r
                        + (timeout ? " (due to timeout)" : " (pause complete)"));
                completePauseLocked(true);
            ...
    }
```
然后执行了completePauseLocked方法：

```
private void completePauseLocked(boolean resumeNext) {
        ...

        if (resumeNext) {
            final ActivityStack topStack = mStackSupervisor.getFocusedStack();
            if (!mService.isSleepingOrShuttingDown()) {
                mStackSupervisor.resumeTopActivitiesLocked(topStack, prev, null);
            } else {
                mStackSupervisor.checkReadyForSleepLocked();
                ActivityRecord top = topStack.topRunningActivityLocked(null);
                if (top == null || (prev != null && top != prev)) {
                    // If there are no more activities available to run,
                    // do resume anyway to start something.  Also if the top
                    // activity on the stack is not the just paused activity,
                    // we need to go ahead and resume it to ensure we complete
                    // an in-flight app switch.
                    mStackSupervisor.resumeTopActivitiesLocked(topStack, null, null);
                }
            }
        }
        ...
    }
```
经过了一系列的逻辑之后，又调用了resumeTopActivitiesLocked方法，又回到了第二步中解析的方法中了，这样经过
resumeTopActivitiesLocked --> 
ActivityStack.resumeTopActivityLocked() --> 
resumeTopActivityInnerLocked --> 
startSpecificActivityLocked
好吧，我们看一下startSpecificActivityLocked的具体实现：

```
void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);

        r.task.stack.setLaunchTime(r);

        if (app != null && app.thread != null) {
            try {
                if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                        || !"android".equals(r.info.packageName)) {
                    // Don't add this if it is a platform component that is marked
                    // to run in multiple processes, because this is actually
                    // part of the framework so doesn't make sense to track as a
                    // separate apk in the process.
                    app.addPackage(r.info.packageName, r.info.applicationInfo.versionCode,
                            mService.mProcessStats);
                }
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }

            // If a dead object exception was thrown -- fall through to
            // restart the application.
        }

        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
    }
```
可以发现在这个方法中，首先会判断一下需要启动的Activity所需要的应用进程是否已经启动，若启动的话，则直接调用realStartAtivityLocked方法，否则调用startProcessLocked方法，用于启动应用进程。
这样关于启动Activity时的第三步骤就已经执行完成了，这里主要是实现了对栈顶Activity执行onPause
方法，而这个方法首先判断需要启动的Activity所属的进程是否已经启动，若已经启动则直接调用启动Activity的方法，否则将先启动Activity的应用进程，然后在启动该Activity。

<br/><strong><font size="6">四：启动Activity所属的应用进程</font></strong>

关于如何启动应用进程，前面的一篇文章已经做了介绍，可参考：<a href="http://blog.csdn.net/qq_23547831/article/details/51119333"> android源码解析之（十一）-->应用进程启动流程</a> 这里在简单的介绍一下

><font color="red">
>ActivityManagerService.startProcessLocked()
>Process.start()
>ActivityThread.main()
>ActivityThread.attach()
>ActivityManagerNative.getDefault().attachApplication()
>ActivityManagerService.attachApplication()
></font>

好吧，首先看一下startProcessLocked()方法的具体实现：

```
private final void startProcessLocked(ProcessRecord app,
            String hostingType, String hostingNameStr) {
        startProcessLocked(app, hostingType, hostingNameStr, null /* abiOverride */,
                null /* entryPoint */, null /* entryPointArgs */);
    }
```
然后回调了其重载的startProcessLocked方法：

```
private final void startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
            ...
            boolean isActivityProcess = (entryPoint == null);
            if (entryPoint == null) entryPoint = "android.app.ActivityThread";
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "Start proc: " +
                    app.processName);
            checkTime(startTime, "startProcess: asking zygote to start proc");
            Process.ProcessStartResult startResult = Process.start(entryPoint,
                    app.processName, uid, uid, gids, debugFlags, mountExternal,
                    app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                    app.info.dataDir, entryPointArgs);
            checkTime(startTime, "startProcess: returned from zygote!");
            ...
    }
```
可以发现其经过一系列的初始化操作之后调用了Process.start方法，并且传入了启动的类名“android.app.ActivityThread”:

```
public static final ProcessStartResult start(final String processClass,
                                  final String niceName,
                                  int uid, int gid, int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String[] zygoteArgs) {
        try {
            return startViaZygote(processClass, niceName, uid, gid, gids,
                    debugFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, zygoteArgs);
        } catch (ZygoteStartFailedEx ex) {
            Log.e(LOG_TAG,
                    "Starting VM process through Zygote failed");
            throw new RuntimeException(
                    "Starting VM process through Zygote failed", ex);
        }
    }
```
然后调用了startViaZygote方法：

```
private static ProcessStartResult startViaZygote(final String processClass,
                                  final String niceName,
                                  final int uid, final int gid,
                                  final int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String[] extraArgs)
                                  throws ZygoteStartFailedEx {
        synchronized(Process.class) {
            ...

            return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
        }
    }
```
继续查看一下zygoteSendArgsAndGetResult方法的实现：

```
private static ProcessStartResult zygoteSendArgsAndGetResult(
            ZygoteState zygoteState, ArrayList<String> args)
            throws ZygoteStartFailedEx {
        try {
            /**
             * See com.android.internal.os.ZygoteInit.readArgumentList()
             * Presently the wire format to the zygote process is:
             * a) a count of arguments (argc, in essence)
             * b) a number of newline-separated argument strings equal to count
             *
             * After the zygote process reads these it will write the pid of
             * the child or -1 on failure, followed by boolean to
             * indicate whether a wrapper process was used.
             */
            final BufferedWriter writer = zygoteState.writer;
            final DataInputStream inputStream = zygoteState.inputStream;

            writer.write(Integer.toString(args.size()));
            writer.newLine();

            int sz = args.size();
            for (int i = 0; i < sz; i++) {
                String arg = args.get(i);
                if (arg.indexOf('\n') >= 0) {
                    throw new ZygoteStartFailedEx(
                            "embedded newlines not allowed");
                }
                writer.write(arg);
                writer.newLine();
            }

            writer.flush();

            // Should there be a timeout on this?
            ProcessStartResult result = new ProcessStartResult();
            result.pid = inputStream.readInt();
            if (result.pid < 0) {
                throw new ZygoteStartFailedEx("fork() failed");
            }
            result.usingWrapper = inputStream.readBoolean();
            return result;
        } catch (IOException ex) {
            zygoteState.close();
            throw new ZygoteStartFailedEx(ex);
        }
    }
```
可以发现其最终调用了Zygote并通过socket通信的方式让Zygote进程fork除了一个新的进程，并根据我们刚刚传递的"android.app.ActivityThread"字符串，反射出该对象并执行ActivityThread的main方法。这样我们所要启动的应用进程这时候其实已经启动了，但是还没有执行相应的初始化操作。

为什么我们平时都将ActivityThread称之为ui线程或者是主线程，这里可以看出，应用进程被创建之后首先执行的是ActivityThread的main方法，所以我们将ActivityThread成为主线程。

好了，这时候我们看一下ActivityThread的main方法的实现逻辑。

```
public static void main(String[] args) {
        ...
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
在main方法中主要执行了一些初始化的逻辑，并且创建了一个UI线程消息队列，这也就是为什么我们可以在主线程中随意的创建Handler而不会报错的原因，这里提出一个问题，大家可以思考一下：子线程可以创建Handler么？可以的话应该怎么做？
然后执行了ActivityThread的attach方法，这里我们看一下attach方法执行了那些逻辑操作。

```
private void attach(boolean system) {
	...
	final IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                // Ignore
            }
	...
}
```
刚刚我们已经分析过ActivityManagerNative是ActivityManagerService的Binder client，所以这里调用了attachApplication实际上就是通过Binder机制调用了ActivityManagerService的attachApplication，具体调用的过程，我们看一下ActivityManagerService是如何实现的：

```
@Override
    public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
    }
```
可以发现其回调了attachApplicationLocked方法，我们看一下这个方法的实现逻辑。

```
private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {

        ...
        // See if the top visible activity is waiting to run in this process...
        if (normalMode) {
            try {
                if (mStackSupervisor.attachApplicationLocked(app)) {
                    didSomething = true;
                }
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
                badApp = true;
            }
        }
        ...

        return true;
    }
```
该方法执行了一系列的初始化操作，这样我们整个应用进程已经启动起来了。终于可以开始activity的启动逻辑了，O(∩_∩)O哈哈~

<br/><strong><font size="6">五：执行启动Acitivity</font></strong>

><font color="red">
>ActivityStackSupervisor.attachApplicationLocked()
>ActivityStackSupervisor.realStartActivityLocked()
>IApplicationThread.scheduleLauncherActivity()
>ActivityThread.sendMessage()
>ActivityThread.H.sendMessage()
>ActivityThread.H.handleMessage()
>ActivityThread.handleLauncherActivity()
>ActivityThread.performLauncherActivity()
>Instrumentation.callActivityOnCreate()
>Activity.onCreate()
>ActivityThread.handleResumeActivity()
>ActivityThread.performResumeActivity()
>Activity.performResume()
>Instrumentation.callActivityOnResume()
>Activity.onResume()
>ActivityManagerNative.getDefault().activityResumed(token)
></font>

首先看一下attachApplicationLocked方法的实现：

```
boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
        final String processName = app.processName;
        boolean didSomething = false;
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = stacks.get(stackNdx);
                if (!isFrontStack(stack)) {
                    continue;
                }
                ActivityRecord hr = stack.topRunningActivityLocked(null);
                if (hr != null) {
                    if (hr.app == null && app.uid == hr.info.applicationInfo.uid
                            && processName.equals(hr.processName)) {
                        try {
                            if (realStartActivityLocked(hr, app, true, true)) {
                                didSomething = true;
                            }
                        } catch (RemoteException e) {
                            Slog.w(TAG, "Exception in new application when starting activity "
                                  + hr.intent.getComponent().flattenToShortString(), e);
                            throw e;
                        }
                    }
                }
            }
        }
        if (!didSomething) {
            ensureActivitiesVisibleLocked(null, 0);
        }
        return didSomething;
    }
```
可以发现其内部调用了realStartActivityLocked方法，通过名字可以知道这个方法应该就是用来启动Activity的，看一下这个方法的实现逻辑：

```
final boolean realStartActivityLocked(ActivityRecord r,
            ProcessRecord app, boolean andResume, boolean checkConfig)
            throws RemoteException {

        ...
            app.forceProcessStateUpTo(mService.mTopProcessState);
            app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                    new Configuration(stack.mOverrideConfig), r.compat, r.launchedFromPackage,
                    task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
                    newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);
		...
        return true;
    }
```
可以发现与第三步执行栈顶Activity onPause时类似，这里也是通过调用IApplicationThread的方法实现的，这里调用的是scheduleLauncherActivity方法，所以真正执行的是ActivityThread中的scheduleLauncherActivity，所以我们看一下ActivityThread中的scheduleLauncherActivity的实现：

```
@Override
        public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
                CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
                int procState, Bundle state, PersistableBundle persistentState,
                List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
                boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {

            updateProcessState(procState, false);

            ActivityClientRecord r = new ActivityClientRecord();

            r.token = token;
            r.ident = ident;
            r.intent = intent;
            r.referrer = referrer;
            r.voiceInteractor = voiceInteractor;
            r.activityInfo = info;
            r.compatInfo = compatInfo;
            r.state = state;
            r.persistentState = persistentState;

            r.pendingResults = pendingResults;
            r.pendingIntents = pendingNewIntents;

            r.startsNotResumed = notResumed;
            r.isForward = isForward;

            r.profilerInfo = profilerInfo;

            r.overrideConfig = overrideConfig;
            updatePendingConfiguration(curConfig);

            sendMessage(H.LAUNCH_ACTIVITY, r);
        }
```
好吧，还是那套逻辑，ActivityThread接收到SystemServer进程的消息之后会通过其内部的Handler对象分发消息，经过一系列的分发之后调用了ActivityThread的handleLaunchActivity方法：

```
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {

        Activity a = performLaunchActivity(r, customIntent);

        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            Bundle oldState = r.state;
            handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed);

            
        }
        ...
    }
```
可以发现这里调用了performLauncherActivity，看名字应该就是执行Activity的启动操作了。。。

```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ...

Activity activity = null;
        try {
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }
        ...

        activity.mCalled = false;
        if (r.isPersistable()) {
           mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
        } else {
           mInstrumentation.callActivityOnCreate(activity, r.state);
        }
        ...
		if (!r.activity.mFinished) {
                    activity.performStart();
                    r.stopped = false;
                }
        ...
        return activity;
    }
```
可以发现这里我们需要的Activity对象终于是创建出来了，而且他是以反射的机制创建的，现在还不太清楚为啥google要以反射的方式创建Activity，先不看这些，然后在代码中其调用Instrumentation的callActivityOnCreate方法。

```
public void callActivityOnCreate(Activity activity, Bundle icicle,
            PersistableBundle persistentState) {
        prePerformCreate(activity);
        activity.performCreate(icicle);
        postPerformCreate(activity);
    }
```
然后执行activity的performCreate方法。。。。好吧，都转晕了。。。

```
final void performCreate(Bundle icicle) {
        onCreate(icicle);
        mActivityTransitionState.readState(icicle);
        performCreateCommon();
    }
```
O(∩_∩)O哈哈~，第二个生命周期方法出来了，onCreate方法。。。。

在回到我们的performLaunchActivity方法，其在调用了mInstrumentation.callActivityOnCreate方法之后又调用了activity.performStart();方法，好吧，看一下他的实现方式：

```
final void performStart() {
        mActivityTransitionState.setEnterActivityOptions(this, getActivityOptions());
        mFragments.noteStateNotSaved();
        mCalled = false;
        mFragments.execPendingActions();
        mInstrumentation.callActivityOnStart(this);
        if (!mCalled) {
            throw new SuperNotCalledException(
                "Activity " + mComponent.toShortString() +
                " did not call through to super.onStart()");
        }
        mFragments.dispatchStart();
        mFragments.reportLoaderStart();
        mActivityTransitionState.enterReady(this);
    }
```
好吧，还是通过Instrumentation调用callActivityOnStart方法：

```
public void callActivityOnStart(Activity activity) {
        activity.onStart();
    }
```
然后是直接调用activity的onStart方法，第三个生命周期方法出现了，O(∩_∩)O哈哈~

还是回到我们刚刚的handleLaunchActivity方法，在调用完performLaunchActivity方法之后，其有吊用了handleResumeActivity方法，好吧，看名字应该是回调Activity的onResume方法的。

```
final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();
        mSomeActivitiesChanged = true;

        // TODO Push resumeArgs into the activity for consideration
        ActivityClientRecord r = performResumeActivity(token, clearHide);

        if (r != null) {
            final Activity a = r.activity;

            if (localLOGV) Slog.v(
                TAG, "Resume " + r + " started activity: " +
                a.mStartedActivity + ", hideForNow: " + r.hideForNow
                + ", finished: " + a.mFinished);

            final int forwardBit = isForward ?
                    WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;

            // If the window hasn't yet been added to the window manager,
            // and this guy didn't finish itself or start another activity,
            // then go ahead and add the window.
            boolean willBeVisible = !a.mStartedActivity;
            if (!willBeVisible) {
                try {
                    willBeVisible = ActivityManagerNative.getDefault().willActivityBeVisible(
                            a.getActivityToken());
                } catch (RemoteException e) {
                }
            }
            if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (a.mVisibleFromClient) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                }

            // If the window has already been added, but during resume
            // we started another activity, then don't yet make the
            // window visible.
            } else if (!willBeVisible) {
                if (localLOGV) Slog.v(
                    TAG, "Launch " + r + " mStartedActivity set");
                r.hideForNow = true;
            }

            // Get rid of anything left hanging around.
            cleanUpPendingRemoveWindows(r);

            // The window is now visible if it has been added, we are not
            // simply finishing, and we are not starting another activity.
            if (!r.activity.mFinished && willBeVisible
                    && r.activity.mDecor != null && !r.hideForNow) {
                if (r.newConfig != null) {
                    r.tmpConfig.setTo(r.newConfig);
                    if (r.overrideConfig != null) {
                        r.tmpConfig.updateFrom(r.overrideConfig);
                    }
                    if (DEBUG_CONFIGURATION) Slog.v(TAG, "Resuming activity "
                            + r.activityInfo.name + " with newConfig " + r.tmpConfig);
                    performConfigurationChanged(r.activity, r.tmpConfig);
                    freeTextLayoutCachesIfNeeded(r.activity.mCurrentConfig.diff(r.tmpConfig));
                    r.newConfig = null;
                }
                if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward="
                        + isForward);
                WindowManager.LayoutParams l = r.window.getAttributes();
                if ((l.softInputMode
                        & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
                        != forwardBit) {
                    l.softInputMode = (l.softInputMode
                            & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
                            | forwardBit;
                    if (r.activity.mVisibleFromClient) {
                        ViewManager wm = a.getWindowManager();
                        View decor = r.window.getDecorView();
                        wm.updateViewLayout(decor, l);
                    }
                }
                r.activity.mVisibleFromServer = true;
                mNumVisibleActivities++;
                if (r.activity.mVisibleFromClient) {
                    r.activity.makeVisible();
                }
            }

            if (!r.onlyLocalRequest) {
                r.nextIdle = mNewActivities;
                mNewActivities = r;
                if (localLOGV) Slog.v(
                    TAG, "Scheduling idle handler for " + r);
                Looper.myQueue().addIdleHandler(new Idler());
            }
            r.onlyLocalRequest = false;

            // Tell the activity manager we have resumed.
            if (reallyResume) {
                try {
                    ActivityManagerNative.getDefault().activityResumed(token);
                } catch (RemoteException ex) {
                }
            }

        } else {
            // If an exception was thrown when trying to resume, then
            // just end this activity.
            try {
                ActivityManagerNative.getDefault()
                    .finishActivity(token, Activity.RESULT_CANCELED, null, false);
            } catch (RemoteException ex) {
            }
        }
    }
```

可以发现其resumeActivity的逻辑调用到了performResumeActivity方法，我们来看一下performResumeActivity是如何实现的。

```
public final ActivityClientRecord performResumeActivity(IBinder token,
            boolean clearHide) {
        ActivityClientRecord r = mActivities.get(token);
        if (localLOGV) Slog.v(TAG, "Performing resume of " + r
                + " finished=" + r.activity.mFinished);
        if (r != null && !r.activity.mFinished) {
            if (clearHide) {
                r.hideForNow = false;
                r.activity.mStartedActivity = false;
            }
            try {
                r.activity.onStateNotSaved();
                r.activity.mFragments.noteStateNotSaved();
                if (r.pendingIntents != null) {
                    deliverNewIntents(r, r.pendingIntents);
                    r.pendingIntents = null;
                }
                if (r.pendingResults != null) {
                    deliverResults(r, r.pendingResults);
                    r.pendingResults = null;
                }
                r.activity.performResume();

                EventLog.writeEvent(LOG_AM_ON_RESUME_CALLED,
                        UserHandle.myUserId(), r.activity.getComponentName().getClassName());

                r.paused = false;
                r.stopped = false;
                r.state = null;
                r.persistentState = null;
            } catch (Exception e) {
                if (!mInstrumentation.onException(r.activity, e)) {
                    throw new RuntimeException(
                        "Unable to resume activity "
                        + r.intent.getComponent().toShortString()
                        + ": " + e.toString(), e);
                }
            }
        }
        return r;
    }
```
在方法体中，最终调用了r.activity.performResume();方法，好吧，这个方法是Activity中定义的方法，我们需要在Activity中查看这个方法的具体实现：

```
final void performResume() {
        ...
        mInstrumentation.callActivityOnResume(this);
        ...
    }
```
好吧，又是熟悉的味道，通过Instrumentation来调用了callActivityOnResume方法。。。

```
public void callActivityOnResume(Activity activity) {
        activity.mResumed = true;
        activity.onResume();
        
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    am.match(activity, activity, activity.getIntent());
                }
            }
        }
    }
```
O(∩_∩)O哈哈~，第四个生命周期方法出现了，onResume方法。。。

终于回调onResume方法了，这时候我们的界面应该已经展示出来了，照理来说我们的Activity应该已经启动完成了，但是还没有，哈哈，别着急。

有一个问题，Activity a 启动 Activity b 会触发那些生命周期方法？
你可能会回答？b的onCreate onStart方法，onResume方法 a的onPause方法和onStop方法，咦？对了onStop方法还没回调呢，O(∩_∩)O哈哈~，对了缺少的就是对onStop方法的回调啊。

好吧，具体的逻辑我们下一步再说

<br><strong><font size="6">六：栈顶Activity执行onStop方法

><font color="red">
>Looper.myQueue().addIdleHandler(new Idler())
>Idler.queueIdle()
>ActivityManagerNative.getDefault().activityIdle()
>ActivityManagerService.activityIdle()
>ActivityStackSupervisor.activityIdleInternalLocked()
>ActivityStack.stopActivityLocked()
>IApplicationThread.scheduleStopActivity()
>ActivityThread.scheduleStopActivity()
>ActivityThread.sendMessage()
>ActivityThread.H.sendMessage()
>ActivityThread.H.handleMessage()
>ActivityThread.handleStopActivity()
>ActivityThread.performStopActivityInner()
>ActivityThread.callCallActivityOnSaveInstanceState()
>Instrumentation.callActivityOnSaveInstanceState()
>Activity.performSaveInstanceState()
>Activity.onSaveInstanceState()
>Activity.performStop()
>Instrumentation.callActivityOnStop()
>Activity.onStop()
></font>

回到我们的handleResumeActivity方法，在方法体最后有这样的一代码：

```
Looper.myQueue().addIdleHandler(new Idler());
```

这段代码是异步消息机制相关的代码，我们可以看一下Idler对象的具体实现：

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
这样当Messagequeue执行add方法之后就会回调其queueIdle()方法，我们可以看到在方法体中其调用了ActivityManagerNative.getDefault().activityIdle()，好吧，熟悉了Binder机制以后我们知道这段代码会执行到ActivityManagerService的activityIdle方法：

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
然后在activityIdle方法中又调用了ActivityStackSupervisor.activityIdleInternalLocked方法：

```
final ActivityRecord activityIdleInternalLocked(final IBinder token, boolean fromTimeout,
            Configuration config) {
        ...

        // Stop any activities that are scheduled to do so but have been
        // waiting for the next one to start.
        for (int i = 0; i < NS; i++) {
            r = stops.get(i);
            final ActivityStack stack = r.task.stack;
            if (stack != null) {
                if (r.finishing) {
                    stack.finishCurrentActivityLocked(r, ActivityStack.FINISH_IMMEDIATELY, false);
                } else {
                    stack.stopActivityLocked(r);
                }
            }
        }

        ...

        return r;
    }
```
可以发现在其中又调用了ActivityStack.stopActivityLocked方法：

```
final void stopActivityLocked(ActivityRecord r) {
        if (DEBUG_SWITCH) Slog.d(TAG_SWITCH, "Stopping: " + r);
        if ((r.intent.getFlags()&Intent.FLAG_ACTIVITY_NO_HISTORY) != 0
                || (r.info.flags&ActivityInfo.FLAG_NO_HISTORY) != 0) {
            ...
                r.app.thread.scheduleStopActivity(r.appToken, r.visible, r.configChangeFlags);
             ...
        }
    }
```
好吧，又是相同的逻辑通过IApplicationThread.scheduleStopActivity,最终调用了ActivityThread.scheduleStopActivity()方法。。。。

```
public final void scheduleStopActivity(IBinder token, boolean showWindow,
                int configChanges) {
           sendMessage(
                showWindow ? H.STOP_ACTIVITY_SHOW : H.STOP_ACTIVITY_HIDE,
                token, 0, configChanges);
        }
```
然后执行sendMessage方法，最终执行H（Handler）的sendMessage方法，并被H的handleMessge方法接收执行handleStopActivity方法。。。

```
private void handleStopActivity(IBinder token, boolean show, int configChanges) {
        ...
        performStopActivityInner(r, info, show, true);
        ...
    }
```
然后我们看一下performStopActivityInner的实现逻辑：

```
private void performStopActivityInner(ActivityClientRecord r,
            StopInfo info, boolean keepShown, boolean saveState) {
	        ...
            // Next have the activity save its current state and managed dialogs...
            if (!r.activity.mFinished && saveState) {
                if (r.state == null) {
                    callCallActivityOnSaveInstanceState(r);
                }
            }

            if (!keepShown) {
                try {
                    // Now we are idle.
                    r.activity.performStop();
                } catch (Exception e) {
                    if (!mInstrumentation.onException(r.activity, e)) {
                        throw new RuntimeException(
                                "Unable to stop activity "
                                + r.intent.getComponent().toShortString()
                                + ": " + e.toString(), e);
                    }
                }
                r.stopped = true;
            }
        }
    }
```
好吧，看样子在这个方法中执行了两个逻辑，一个是执行Activity的onSaveInstance方法一个是执行Activity的onStop方法，我们先看一下callCallActivityOnSaveInstanceState的执行逻辑：

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
好吧，又是通过Instrumentation来执行。。。

```
public void callActivityOnSaveInstanceState(Activity activity, Bundle outState,
            PersistableBundle outPersistentState) {
        activity.performSaveInstanceState(outState, outPersistentState);
    }
```
又间接调用了Activity的performSaveInstanceState方法：

```
final void performSaveInstanceState(Bundle outState) {
        onSaveInstanceState(outState);
        saveManagedDialogs(outState);
        mActivityTransitionState.saveState(outState);
        if (DEBUG_LIFECYCLE) Slog.v(TAG, "onSaveInstanceState " + this + ": " + outState);
    }
```
呵呵，这里调用到了，我们以前经常会重写的onSaveInstanceState方法。

然后我们看一下performStopActivityInner中调用到的Activity方法的performStop方法：

```
final void performStop() {
        mDoReportFullyDrawn = false;
        mFragments.doLoaderStop(mChangingConfigurations /*retain*/);

        if (!mStopped) {
            if (mWindow != null) {
                mWindow.closeAllPanels();
            }

            if (mToken != null && mParent == null) {
                WindowManagerGlobal.getInstance().setStoppedState(mToken, true);
            }

            mFragments.dispatchStop();

            mCalled = false;
            mInstrumentation.callActivityOnStop(this);
            if (!mCalled) {
                throw new SuperNotCalledException(
                    "Activity " + mComponent.toShortString() +
                    " did not call through to super.onStop()");
            }

            synchronized (mManagedCursors) {
                final int N = mManagedCursors.size();
                for (int i=0; i<N; i++) {
                    ManagedCursor mc = mManagedCursors.get(i);
                    if (!mc.mReleased) {
                        mc.mCursor.deactivate();
                        mc.mReleased = true;
                    }
                }
            }

            mStopped = true;
        }
        mResumed = false;
    }
```
还是通过Instrumentation来实现的，调用了它的callActivityOnStop方法。。

```
public void callActivityOnStop(Activity activity) {
        activity.onStop();
    }
```
O(∩_∩)O哈哈~，最后一个生命周期方法终于出来了，onStop().....

<br><br><br><font size="5">总结：</font>

- Activity的启动流程一般是通过调用startActivity或者是startActivityForResult来开始的

- startActivity内部也是通过调用startActivityForResult来启动Activity，只不过传递的requestCode小于0

- Activity的启动流程涉及到多个进程之间的通讯这里主要是ActivityThread与ActivityManagerService之间的通讯

- ActivityThread向ActivityManagerService传递进程间消息通过ActivityManagerNative，ActivityManagerService向ActivityThread进程间传递消息通过IApplicationThread。

- ActivityManagerService接收到应用进程创建Activity的请求之后会执行初始化操作，解析启动模式，保存请求信息等一系列操作。

- ActivityManagerService保存完请求信息之后会将当前系统栈顶的Activity执行onPause操作，并且IApplication进程间通讯告诉应用程序继承执行当前栈顶的Activity的onPause方法；

- ActivityThread接收到SystemServer的消息之后会统一交个自身定义的Handler对象处理分发；

- ActivityThread执行完栈顶的Activity的onPause方法之后会通过ActivityManagerNative执行进程间通讯告诉ActivityManagerService，栈顶Actiity已经执行完成onPause方法，继续执行后续操作；

- ActivityManagerService会继续执行启动Activity的逻辑，这时候会判断需要启动的Activity所属的应用进程是否已经启动，若没有启动则首先会启动这个Activity的应用程序进程；

- ActivityManagerService会通过socket与Zygote继承通讯，并告知Zygote进程fork出一个新的应用程序进程，然后执行ActivityThread的mani方法；

- 在ActivityThead.main方法中执行初始化操作，初始化主线程异步消息，然后通知ActivityManagerService执行进程初始化操作；

- ActivityManagerService会在执行初始化操作的同时检测当前进程是否有需要创建的Activity对象，若有的话，则执行创建操作；

- ActivityManagerService将执行创建Activity的通知告知ActivityThread，然后通过反射机制创建出Activity对象，并执行Activity的onCreate方法，onStart方法，onResume方法；

- ActivityThread执行完成onResume方法之后告知ActivityManagerService onResume执行完成，开始执行栈顶Activity的onStop方法；

- ActivityManagerService开始执行栈顶的onStop方法并告知ActivityThread；

- ActivityThread执行真正的onStop方法；

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
