上篇文章中我们分析了Activity的onSaveInstanceState方法执行时机，知道了Activity在一般情况下，若只是执行onPause方法则不会执行onSaveInstanceState方法，而一旦执行了onStop方法就会执行onSaveInstanceState方法，具体的信息，可以参见onSaveInstanceState方法执行时机：<a href="http://blog.csdn.net/qq_23547831/article/details/51464535">android源码解析（二十四）-->onSaveInstanceState执行时机</a> 这篇文章中同样的我们分析一下Actvity（当然不只是Activity，同样包含Servier，ContentProvider，Application等）的另一个内部方法：onLowMemory。该方法主要用于当前系统可用内存比较低的时候回调使用。

这里简单介绍一下Android系统的内存分配机制。Android系统中一个个的App都是一个个不同的应用进程，拥有各自的JVM与运行时，每个App的进程可使用的内存大小都是固定的，当系统中App打开数量过多时，就会使Android系统的可用内存降低，对于当前正在使用的App而言，可能还需要继续申请系统内存，而我们的剩余系统内存已经不足以被当前App所申请了，这时候系统会自动的清理那些后台进程，进而释放出可用内存用于前台进程的使用，当然这里系统清理后台进程的算法不是我们讨论的重点。这里我们只是大概的分析Android系统回调Activity的onLowMemory方法的流程。

通过前面关于Activity的启动流程分析我们知道ActivityManagerService是整个Android系统的管理中枢，负责Activity，Servier等四大组件的启动与销毁等工作，同样的对于应用进程的管理工作也是在ActivityMaangerServier中完成的，我们知道android系统中有两个比较重要的进程Zygote进程和SystemServer进程，其中Zygote进程是整个Android系统的根进程，其他所有的进程都是通过Zygote进程fork出来的。而SystemServer进程则用于运行各种服务，为其他的应用进程提供各种功能接口等，在前面我们分析过SystemServer进程的启动流程（参考：<a href="http://blog.csdn.net/qq_23547831/article/details/51105171"> android源码解析之（九）-->SystemServer进程启动流程</a>）其中在SystemServer的startBootService方法中我们调用了：

```
// Set up the Application instance for the system process and get started.
        mActivityManagerService.setSystemProcess();
```
方法，看其注释说明，说的是为System进程初始化Application实例，这里我们可以看一下该方法的具体实现：

```
public void setSystemProcess() {
        try {
            ServiceManager.addService(Context.ACTIVITY_SERVICE, this, true);
            ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
            ServiceManager.addService("meminfo", new MemBinder(this));
            ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
            ServiceManager.addService("dbinfo", new DbBinder(this));
            if (MONITOR_CPU_USAGE) {
                ServiceManager.addService("cpuinfo", new CpuBinder(this));
            }
            ServiceManager.addService("permission", new PermissionController(this));
            ServiceManager.addService("processinfo", new ProcessInfoService(this));

            ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
                    "android", STOCK_PM_FLAGS);
            mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());

            synchronized (this) {
                ProcessRecord app = newProcessRecordLocked(info, info.processName, false, 0);
                app.persistent = true;
                app.pid = MY_PID;
                app.maxAdj = ProcessList.SYSTEM_ADJ;
                app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
                synchronized (mPidsSelfLocked) {
                    mPidsSelfLocked.put(app.pid, app);
                }
                updateLruProcessLocked(app, false, null);
                updateOomAdjLocked();
            }
        } catch (PackageManager.NameNotFoundException e) {
            throw new RuntimeException(
                    "Unable to find android system package", e);
        }
    }
```
这里简单介绍一下ServierManager是一个管理服务的服务，而其addServier方法就是注册各种服务（服务注册到JNI层，具体的关于是如何注册到JNI层的这里暂不做过多的解释）。可以发现在方法体中我们注册了名称为：memInfo的服务MemBinder，MemBinder是一个Binder类型的服务，主要用于检测系统内存情况，这里可以看一下其具体的实现逻辑：

```
static class MemBinder extends Binder {
        ActivityManagerService mActivityManagerService;
        MemBinder(ActivityManagerService activityManagerService) {
            mActivityManagerService = activityManagerService;
        }

        @Override
        protected void dump(FileDescriptor fd, PrintWriter pw, String[] args) {
            if (mActivityManagerService.checkCallingPermission(android.Manifest.permission.DUMP)
                    != PackageManager.PERMISSION_GRANTED) {
                pw.println("Permission Denial: can't dump meminfo from from pid="
                        + Binder.getCallingPid() + ", uid=" + Binder.getCallingUid()
                        + " without permission " + android.Manifest.permission.DUMP);
                return;
            }

            mActivityManagerService.dumpApplicationMemoryUsage(fd, pw, "  ", args, false, null);
        }
    }
```
查看源码，我们可以发现MemBinder类继承于Binder类也就是说其实一个Binder类型的服务，并且有一个成员方法dump，该方法主要用于执行shell命令，当系统可用内存比较低的时候就会执行了该方法，然后回调到ActivityManagerService中的killAllBackground方法，下面我们重点看一下killAllBackground方法的具体实现：

```
@Override
    public void killAllBackgroundProcesses() {
        ...
           doLowMemReportIfNeededLocked(null);
        ...
        } finally {
            Binder.restoreCallingIdentity(callingId);
        }
    }
```
可以看到这个方法体中会执行doLowMemReportIfNeededLocked方法，该方法是做什么的呢?我们继续看一下doLowMemReportIfNeededLoced方法的实现：

```
final void doLowMemReportIfNeededLocked(ProcessRecord dyingProc) {
        ...
        scheduleAppGcsLocked();
        ...
    }
```
好吧，在这个方法中我们又调用了scheduleAppGcsLocked方法，这样我们就继续看一下scheduleAppGcsLocked方法的实现逻辑：

```
/**
     * Schedule the execution of all pending app GCs.
     */
    final void scheduleAppGcsLocked() {
        mHandler.removeMessages(GC_BACKGROUND_PROCESSES_MSG);

        if (mProcessesToGc.size() > 0) {
            // Schedule a GC for the time to the next process.
            ProcessRecord proc = mProcessesToGc.get(0);
            Message msg = mHandler.obtainMessage(GC_BACKGROUND_PROCESSES_MSG);

            long when = proc.lastRequestedGc + GC_MIN_INTERVAL;
            long now = SystemClock.uptimeMillis();
            if (when < (now+GC_TIMEOUT)) {
                when = now + GC_TIMEOUT;
            }
            mHandler.sendMessageAtTime(msg, when);
        }
    }
```
可以发现这里执行的逻辑就是通过mHandler发送一个msg.what为GC_BACKGROUND_PROCESSES_MSG的异步消息，这样消息体最终会被mHandler的handleMessage方法所执行，继续看一下mHandler的handleMessage方法的执行逻辑：

```
case GC_BACKGROUND_PROCESSES_MSG: {
                synchronized (ActivityManagerService.this) {
                    performAppGcsIfAppropriateLocked();
                }
            } break;
```
在mHandler的handleMessage方法中，首先会判断msg的what是否为GC_BACKGROUND_PROCESSES_MSG，然后会执行performAppGcsIfAppropriateLocked方法，这样我们继续看一下performAppGcsIfAppropriateLocked方法的实现：

```
/**
     * If all looks good, perform GCs on all processes waiting for them.
     */
    final void performAppGcsIfAppropriateLocked() {
        if (canGcNowLocked()) {
            performAppGcsLocked();
            return;
        }
        // Still not idle, wait some more.
        scheduleAppGcsLocked();
    }
```
可以发现这里首先判断是否能够执行gc操作，若不能继续执行上面的scheduleAppGcsLocked方法，然后继续执行发送异步消息的逻辑，直到变量canGcNowLocked为true，并执行performAppGcsLocked方法，然后return掉，这样我们继续跟踪代码，看一下performAppGcsLocked方法的执行逻辑：

```
/**
     * Perform GCs on all processes that are waiting for it, but only
     * if things are idle.
     */
    final void performAppGcsLocked() {
        final int N = mProcessesToGc.size();
        if (N <= 0) {
            return;
        }
        if (canGcNowLocked()) {
            while (mProcessesToGc.size() > 0) {
                ProcessRecord proc = mProcessesToGc.remove(0);
                if (proc.curRawAdj > ProcessList.PERCEPTIBLE_APP_ADJ || proc.reportLowMemory) {
                    if ((proc.lastRequestedGc+GC_MIN_INTERVAL)
                            <= SystemClock.uptimeMillis()) {
                        // To avoid spamming the system, we will GC processes one
                        // at a time, waiting a few seconds between each.
                        performAppGcLocked(proc);
                        scheduleAppGcsLocked();
                        return;
                    } else {
                        // It hasn't been long enough since we last GCed this
                        // process...  put it in the list to wait for its time.
                        addProcessToGcListLocked(proc);
                        break;
                    }
                }
            }

            scheduleAppGcsLocked();
        }
    }
```
可以发现该方法经过一系列的逻辑判断之后会执行performAppGcLocked方法，我们继续看一下该方法的实现：

```
/**
     * Ask a given process to GC right now.
     */
    final void performAppGcLocked(ProcessRecord app) {
        try {
            app.lastRequestedGc = SystemClock.uptimeMillis();
            if (app.thread != null) {
                if (app.reportLowMemory) {
                    app.reportLowMemory = false;
                    app.thread.scheduleLowMemory();
                } else {
                    app.thread.processInBackground();
                }
            }
        } catch (Exception e) {
            // whatever.
        }
    }
```
可以发现最终执行的是app.thread.scheduleLowMemory方法，而这里的app.thread是ActivityThread.ApplicationThread对象，所以这里最终是通过Binder进程间通讯，执行的是ActivityThread.ApplicationThread的scheduleLowMemory方法，好吧让我们看一下ActivityThread.ApplicationThread的scheduleLowMemory
方法的实现逻辑...

```
@Override
        public void scheduleLowMemory() {
            sendMessage(H.LOW_MEMORY, null);
        }
```
在ActivityThread中的scheduleLowMemory方法中并没有执行额外逻辑，而是直接调用了sendMessage方法，继续跟踪方法的执行：

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
可以发现在sendMessage方法中最终通过一个Handler类型的mH成员变量发送一个异步消息，这样异步消息最终会被mH的handleMessage方法执行。。。。，经过查看源代码我们知道在mH的handleMessage方法中最终调用的是handleLowMemory方法：

```
final void handleLowMemory() {
        ArrayList<ComponentCallbacks2> callbacks = collectComponentCallbacks(true, null);

        final int N = callbacks.size();
        for (int i=0; i<N; i++) {
            callbacks.get(i).onLowMemory();
        }

        // Ask SQLite to free up as much memory as it can, mostly from its page caches.
        if (Process.myUid() != Process.SYSTEM_UID) {
            int sqliteReleased = SQLiteDatabase.releaseMemory();
            EventLog.writeEvent(SQLITE_MEM_RELEASED_EVENT_LOG_TAG, sqliteReleased);
        }

        // Ask graphics to free up as much as possible (font/image caches)
        Canvas.freeCaches();

        // Ask text layout engine to free also as much as possible
        Canvas.freeTextLayoutCaches();

        BinderInternal.forceGc("mem");
    }
```
可以发现这里通过遍历ComponentCallbacks2并执行了其onLowMemory方法，那么这里的ComponentCallBacks2是什么呢？这里我们查看一下collectComponentCallbacks方法的实现逻辑。

```
ArrayList<ComponentCallbacks2> collectComponentCallbacks(
            boolean allActivities, Configuration newConfig) {
        ArrayList<ComponentCallbacks2> callbacks
                = new ArrayList<ComponentCallbacks2>();

        synchronized (mResourcesManager) {
            final int NAPP = mAllApplications.size();
            for (int i=0; i<NAPP; i++) {
                callbacks.add(mAllApplications.get(i));
            }
            final int NACT = mActivities.size();
            for (int i=0; i<NACT; i++) {
                ActivityClientRecord ar = mActivities.valueAt(i);
                Activity a = ar.activity;
                if (a != null) {
                    Configuration thisConfig = applyConfigCompatMainThread(
                            mCurDefaultDisplayDpi, newConfig,
                            ar.packageInfo.getCompatibilityInfo());
                    if (!ar.activity.mFinished && (allActivities || !ar.paused)) {
                        // If the activity is currently resumed, its configuration
                        // needs to change right now.
                        callbacks.add(a);
                    } else if (thisConfig != null) {
                        // Otherwise, we will tell it about the change
                        // the next time it is resumed or shown.  Note that
                        // the activity manager may, before then, decide the
                        // activity needs to be destroyed to handle its new
                        // configuration.
                        if (DEBUG_CONFIGURATION) {
                            Slog.v(TAG, "Setting activity "
                                    + ar.activityInfo.name + " newConfig=" + thisConfig);
                        }
                        ar.newConfig = thisConfig;
                    }
                }
            }
            final int NSVC = mServices.size();
            for (int i=0; i<NSVC; i++) {
                callbacks.add(mServices.valueAt(i));
            }
        }
        synchronized (mProviderMap) {
            final int NPRV = mLocalProviders.size();
            for (int i=0; i<NPRV; i++) {
                callbacks.add(mLocalProviders.valueAt(i).mLocalProvider);
            }
        }

        return callbacks;
    }
```
可以发现该方法最终返回类型为ArrayList<ComponentCallbacks2>类型的callBacks而我们的callBacks中保存的是我们应用进程中的Activity，Service，Provider已经Application等。咦？Activity，Service，Provider，Application都是ComponentCallBacks2类型的么？我们看一看一下具体的定义：

Actvity的类定义：

```
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback
```
Service的类定义：

```
public abstract class Service extends ContextWrapper implements ComponentCallbacks2
```

ContentProvider的类定义：

```
public abstract class ContentProvider implements ComponentCallbacks2
```
Application的类定义：

```
public class Application extends ContextWrapper implements ComponentCallbacks2
```
可以发现其都是继承与ComponentCalbacks2，所以其都可以被当做是ComponentCallbacks2类型的变量。而同样是四大组件的BroadcastReceiver，我们可以下其类定义：

```
public abstract class BroadcastReceiver
```
可以看到其并未继承与ComponentCallbacks2，所以并未执行，所以通过这样的分析，我们知道了，最终应用程序中的Activity，Servier，ContentProvider，Application的onLowMemory方法会被执行。而由于我们是在系统内存紧张的时候会执行killAllBackground方法进而通过层层条用执行Activity、Service、ContentProvider、Application的onLowMemory方法，所以我们可以在这些组件的onLowMemory方法中执行了一些清理资源的操作，释放一些内存，尽量保证自身的应用进程不被杀死。

总结：

- 系统在JNI层会时时检测内存变量，当内存过低时会通过kiilbackground的方法清理后台进程。

- 经过层层的调用过程最终会执行Activity、Service、ContentProvider、Application的onLowMemory方法。

- 可以在组件的onLowMemory方法中执行一些清理资源的操作，释放内存防止进程被杀死。

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
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51464535">android源码解析（二十四）-->onSaveInstanceState执行时机</a>

