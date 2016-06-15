前面的几篇文章都是讲解的android中的窗口显示机制，包括Activity窗口加载绘制流程，Dialog窗口加载绘制流程，PopupWindow窗口加载绘制流程，Toast窗口加载绘制流程等等。整个Android的界面显示的原理都是类似的，都是通过Window对象控制View组件，实现加载与绘制流程。

这篇文章休息一下，不在讲解Android的窗口绘制机制，穿插的讲解一下Android系统的异常处理流程。O(∩_∩)O哈哈~

开发过android项目的童鞋对android系统中错误弹窗，force close应该不陌生了，当我们的App异常崩溃时，就会弹出一个force close的弹窗，告诉我们App崩溃，以及一下简易的错误信息：
![这里写图片描述](http://img.blog.csdn.net/20160512110851449)

那么这里的force close弹窗是如何弹出的呢？

还有我们在开发App的过程中，经常会自定义Application，自定义UncaughtExceptionHandler实现App的全局异常处理，那么这里的UncaughtExceptionHandler是如何实现对异常的全局处理的呢？（可参考：<a href="http://blog.csdn.net/qq_23547831/article/details/41725069"> 在Android中自定义捕获Application全局异常</a>）

带着这两个问题，我们开始今天的异常流程分析。

在android应用进程的启动流程中我们在经过一系列的操作之后会调用RuntimeInit.zygoteInit方法（可参考：<a href="http://blog.csdn.net/luoshengyang/article/details/6747696">Android应用程序进程启动过程的源代码分析</a>）

而我们也是从这里开始分析我们的Android系统异常处理流程，好了，让我们先来看一下zygoteInit方法的具体实现：

```
public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application from zygote");

        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "RuntimeInit");
        redirectLogStreams();

        commonInit();
        nativeZygoteInit();
        applicationInit(targetSdkVersion, argv, classLoader);
    }
```
可以看到在方法体中我们调用了commonInit方法，这个方法是用于初始化操作的，继续看一下commonInit方法的实现：

```
private static final void commonInit() {
        ...
        Thread.setDefaultUncaughtExceptionHandler(new UncaughtHandler());
		...
    }
```
可以看到在这里我们调用了Thread.setDefaultUncaughtExceptionHandler方法，这样当我们的进程出现异常的时候，异常信息就会被我们新创建的UncaughtHandler所捕获。

看过我们前面写过的关于Android全局异常处理文章的童鞋应该知道，我们实现对Android异常全局处理的操作也是通过设置Thread.setDefaultUncaughtExceptionHandler来实现的，具体可参考：<a href="http://blog.csdn.net/qq_23547831/article/details/41725069"> 在Android中自定义捕获Application全局异常</a>
所以Android系统默认的异常信息都会被这里的UncaughtHandler所捕获并被其uncaughtException方法回调，所以若我们不重写Thread.setDefaultUncaughtExceptionHandler方法，那么这里的UncaughtHandler就是我们默认的异常处理操作 这样我们看一下UncaughtHandler的具体实现：

```
private static class UncaughtHandler implements Thread.UncaughtExceptionHandler {
        public void uncaughtException(Thread t, Throwable e) {
            try {
                // Don't re-enter -- avoid infinite loops if crash-reporting crashes.
                if (mCrashing) return;
                mCrashing = true;

                if (mApplicationObject == null) {
                    Clog_e(TAG, "*** FATAL EXCEPTION IN SYSTEM PROCESS: " + t.getName(), e);
                } else {
                    StringBuilder message = new StringBuilder();
                    message.append("FATAL EXCEPTION: ").append(t.getName()).append("\n");
                    final String processName = ActivityThread.currentProcessName();
                    if (processName != null) {
                        message.append("Process: ").append(processName).append(", ");
                    }
                    message.append("PID: ").append(Process.myPid());
                    Clog_e(TAG, message.toString(), e);
                }

                // Bring up crash dialog, wait for it to be dismissed
                ActivityManagerNative.getDefault().handleApplicationCrash(
                        mApplicationObject, new ApplicationErrorReport.CrashInfo(e));
            } catch (Throwable t2) {
                try {
                    Clog_e(TAG, "Error reporting crash", t2);
                } catch (Throwable t3) {
                    // Even Clog_e() fails!  Oh well.
                }
            } finally {
                // Try everything to make sure this process goes away.
                Process.killProcess(Process.myPid());
                System.exit(10);
            }
        }
    }
```
这里uncaughtException方法最终会被执行异常信息的处理，我们看一下在这里我们调用了ActivityManagerNative.getDefault().handleApplicationCrash方法，看过我们前面Activity启动流程的童鞋应该知道这里的ActivityManagerNative其实是ActivityManagerService的Binder客户端，而这里的handleApplicationCrash方法最终会调用的是ActivityManagerService的handleApplicationCrash方法。最后在finally分之中，我们调用了Process.killProcess(Process.myPid)和System.exit(10)，这样我们的应用进程就会退出了。

然后我们在这里先简单的分析一下Binder的数据传输过程，看一下handleApplicationCrash方法具体做了哪些事，首先看一下ActivityManagerNative的getDefault方法是如何实现的？

```
static public IActivityManager getDefault() {
        return gDefault.get();
    }
```
可以发现，其是一个静态方法，并执行了gDefault.get方法，我们在看一下gDefault.get方法的实现逻辑：

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
可以发现这里返回一个IActivityManager类型的am对象，而这个am对象是通过调用asInterface方法创建的，我们再来看一下这个asInterface方法的实现逻辑。

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
可以发现该方法最终返回的是一个ActivityManagerProxy对象，所以ActivityManagerNative.getDefault()方法最终返回的是一个ActivityManagerProxy对象，我们再来看一下ActivityManagerProxy的handleApplicationCrash方法。

```
public void handleApplicationCrash(IBinder app,
            ApplicationErrorReport.CrashInfo crashInfo) throws RemoteException
    {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(app);
        crashInfo.writeToParcel(data, 0);
        mRemote.transact(HANDLE_APPLICATION_CRASH_TRANSACTION, data, reply, 0);
        reply.readException();
        reply.recycle();
        data.recycle();
    }
```
这里就是具体的Binder传输数据的逻辑了，这里ActivityManagerNative最为Binder的clent端，而我们的ActivityManagerService同样是继承与ActivityManagerNative，最为Binder的server端，通过传输最终ActivityManagerService的handleApplicationCrash方法会被执行。

```
public void handleApplicationCrash(IBinder app, ApplicationErrorReport.CrashInfo crashInfo) {
        ProcessRecord r = findAppProcess(app, "Crash");
        final String processName = app == null ? "system_server"
                : (r == null ? "unknown" : r.processName);

        handleApplicationCrashInner("crash", r, processName, crashInfo);
    }
```
可以看到在ActivityManagerService的handleApplicationCrash方法中我们调用了handleApplicationCreashInner方法，这样我们继续看一下handleApplicationCrashInner方法的实现。

```
void handleApplicationCrashInner(String eventType, ProcessRecord r, String processName,
            ApplicationErrorReport.CrashInfo crashInfo) {
        EventLog.writeEvent(EventLogTags.AM_CRASH, Binder.getCallingPid(),
                UserHandle.getUserId(Binder.getCallingUid()), processName,
                r == null ? -1 : r.info.flags,
                crashInfo.exceptionClassName,
                crashInfo.exceptionMessage,
                crashInfo.throwFileName,
                crashInfo.throwLineNumber);

        addErrorToDropBox(eventType, r, processName, null, null, null, null, null, crashInfo);

        crashApplication(r, crashInfo);
    }
```
可以发现在handleApplicationCrashInner方法中主要调用了两个方法addErrorToDropBox和crashApplication，我们首先看一下addErrorToDropBox方法的实现逻辑。

```
public void addErrorToDropBox(String eventType,
            ProcessRecord process, String processName, ActivityRecord activity,
            ActivityRecord parent, String subject,
            final String report, final File logFile,
            final ApplicationErrorReport.CrashInfo crashInfo) {
        // NOTE -- this must never acquire the ActivityManagerService lock,
        // otherwise the watchdog may be prevented from resetting the system.

        final String dropboxTag = processClass(process) + "_" + eventType;
        final DropBoxManager dbox = (DropBoxManager)
                mContext.getSystemService(Context.DROPBOX_SERVICE);

        // Exit early if the dropbox isn't configured to accept this report type.
        if (dbox == null || !dbox.isTagEnabled(dropboxTag)) return;

        final StringBuilder sb = new StringBuilder(1024);
        appendDropBoxProcessHeaders(process, processName, sb);
        if (activity != null) {
            sb.append("Activity: ").append(activity.shortComponentName).append("\n");
        }
        if (parent != null && parent.app != null && parent.app.pid != process.pid) {
            sb.append("Parent-Process: ").append(parent.app.processName).append("\n");
        }
        if (parent != null && parent != activity) {
            sb.append("Parent-Activity: ").append(parent.shortComponentName).append("\n");
        }
        if (subject != null) {
            sb.append("Subject: ").append(subject).append("\n");
        }
        sb.append("Build: ").append(Build.FINGERPRINT).append("\n");
        if (Debug.isDebuggerConnected()) {
            sb.append("Debugger: Connected\n");
        }
        sb.append("\n");

        // Do the rest in a worker thread to avoid blocking the caller on I/O
        // (After this point, we shouldn't access AMS internal data structures.)
        Thread worker = new Thread("Error dump: " + dropboxTag) {
            @Override
            public void run() {
                if (report != null) {
                    sb.append(report);
                }
                if (logFile != null) {
                    try {
                        sb.append(FileUtils.readTextFile(logFile, DROPBOX_MAX_SIZE,
                                    "\n\n[[TRUNCATED]]"));
                    } catch (IOException e) {
                        Slog.e(TAG, "Error reading " + logFile, e);
                    }
                }
                if (crashInfo != null && crashInfo.stackTrace != null) {
                    sb.append(crashInfo.stackTrace);
                }

                String setting = Settings.Global.ERROR_LOGCAT_PREFIX + dropboxTag;
                int lines = Settings.Global.getInt(mContext.getContentResolver(), setting, 0);
                if (lines > 0) {
                    sb.append("\n");

                    // Merge several logcat streams, and take the last N lines
                    InputStreamReader input = null;
                    try {
                        java.lang.Process logcat = new ProcessBuilder("/system/bin/logcat",
                                "-v", "time", "-b", "events", "-b", "system", "-b", "main",
                                "-b", "crash",
                                "-t", String.valueOf(lines)).redirectErrorStream(true).start();

                        try { logcat.getOutputStream().close(); } catch (IOException e) {}
                        try { logcat.getErrorStream().close(); } catch (IOException e) {}
                        input = new InputStreamReader(logcat.getInputStream());

                        int num;
                        char[] buf = new char[8192];
                        while ((num = input.read(buf)) > 0) sb.append(buf, 0, num);
                    } catch (IOException e) {
                        Slog.e(TAG, "Error running logcat", e);
                    } finally {
                        if (input != null) try { input.close(); } catch (IOException e) {}
                    }
                }

                dbox.addText(dropboxTag, sb.toString());
            }
        };

        if (process == null) {
            // If process is null, we are being called from some internal code
            // and may be about to die -- run this synchronously.
            worker.run();
        } else {
            worker.start();
        }
    }
```
可以看到方法体很长，但是逻辑比较简单，在方法体最后通过判断应用进程是否为空（是否被销毁）来执行worker.run方法或者是worker.start方法，这里的worker是一个Thread对象，而在我们的worker对象的run方法中主要的执行逻辑就是将崩溃信息写入系统log中，所以addErrorToDropBox方法的主要执行逻辑就是讲App的崩溃信息写入系统log中。。。。

继续回到我们的handleApplicationCrashInner方法中，看一下crashApplication方法是如何实现的。

```
private void crashApplication(ProcessRecord r, ApplicationErrorReport.CrashInfo crashInfo) {
        long timeMillis = System.currentTimeMillis();
        String shortMsg = crashInfo.exceptionClassName;
        String longMsg = crashInfo.exceptionMessage;
        String stackTrace = crashInfo.stackTrace;
        if (shortMsg != null && longMsg != null) {
            longMsg = shortMsg + ": " + longMsg;
        } else if (shortMsg != null) {
            longMsg = shortMsg;
        }

        AppErrorResult result = new AppErrorResult();
        synchronized (this) {
            if (mController != null) {
                try {
                    String name = r != null ? r.processName : null;
                    int pid = r != null ? r.pid : Binder.getCallingPid();
                    int uid = r != null ? r.info.uid : Binder.getCallingUid();
                    if (!mController.appCrashed(name, pid,
                            shortMsg, longMsg, timeMillis, crashInfo.stackTrace)) {
                        if ("1".equals(SystemProperties.get(SYSTEM_DEBUGGABLE, "0"))
                                && "Native crash".equals(crashInfo.exceptionClassName)) {
                            Slog.w(TAG, "Skip killing native crashed app " + name
                                    + "(" + pid + ") during testing");
                        } else {
                            Slog.w(TAG, "Force-killing crashed app " + name
                                    + " at watcher's request");
                            if (r != null) {
                                r.kill("crash", true);
                            } else {
                                // Huh.
                                Process.killProcess(pid);
                                killProcessGroup(uid, pid);
                            }
                        }
                        return;
                    }
                } catch (RemoteException e) {
                    mController = null;
                    Watchdog.getInstance().setActivityController(null);
                }
            }

            final long origId = Binder.clearCallingIdentity();

            // If this process is running instrumentation, finish it.
            if (r != null && r.instrumentationClass != null) {
                Slog.w(TAG, "Error in app " + r.processName
                      + " running instrumentation " + r.instrumentationClass + ":");
                if (shortMsg != null) Slog.w(TAG, "  " + shortMsg);
                if (longMsg != null) Slog.w(TAG, "  " + longMsg);
                Bundle info = new Bundle();
                info.putString("shortMsg", shortMsg);
                info.putString("longMsg", longMsg);
                finishInstrumentationLocked(r, Activity.RESULT_CANCELED, info);
                Binder.restoreCallingIdentity(origId);
                return;
            }

            // Log crash in battery stats.
            if (r != null) {
                mBatteryStatsService.noteProcessCrash(r.processName, r.uid);
            }

            // If we can't identify the process or it's already exceeded its crash quota,
            // quit right away without showing a crash dialog.
            if (r == null || !makeAppCrashingLocked(r, shortMsg, longMsg, stackTrace)) {
                Binder.restoreCallingIdentity(origId);
                return;
            }

            Message msg = Message.obtain();
            msg.what = SHOW_ERROR_MSG;
            HashMap data = new HashMap();
            data.put("result", result);
            data.put("app", r);
            msg.obj = data;
            mUiHandler.sendMessage(msg);

            Binder.restoreCallingIdentity(origId);
        }

        int res = result.get();

        Intent appErrorIntent = null;
        synchronized (this) {
            if (r != null && !r.isolated) {
                // XXX Can't keep track of crash time for isolated processes,
                // since they don't have a persistent identity.
                mProcessCrashTimes.put(r.info.processName, r.uid,
                        SystemClock.uptimeMillis());
            }
            if (res == AppErrorDialog.FORCE_QUIT_AND_REPORT) {
                appErrorIntent = createAppErrorIntentLocked(r, timeMillis, crashInfo);
            }
        }

        if (appErrorIntent != null) {
            try {
                mContext.startActivityAsUser(appErrorIntent, new UserHandle(r.userId));
            } catch (ActivityNotFoundException e) {
                Slog.w(TAG, "bug report receiver dissappeared", e);
            }
        }
    }
```
可以发现在方法体中我们调用了mUiHandler.sendMessage(msg)，其中mUiHandler是一个在主线程中创建的Handler对象，而这里的msg是一个what值为SHOW_ERROR_MSG的消息，这句话的本质就是向Ui线程中发送一个异步消息。我们来看一下mUiHander的处理逻辑。

在mUiHandler的handeMessage方法中，根据what值得不同，执行了如下逻辑：

```
case SHOW_ERROR_MSG: {
                HashMap<String, Object> data = (HashMap<String, Object>) msg.obj;
                boolean showBackground = Settings.Secure.getInt(mContext.getContentResolver(),
                        Settings.Secure.ANR_SHOW_BACKGROUND, 0) != 0;
                synchronized (ActivityManagerService.this) {
                    ProcessRecord proc = (ProcessRecord)data.get("app");
                    AppErrorResult res = (AppErrorResult) data.get("result");
                    if (proc != null && proc.crashDialog != null) {
                        Slog.e(TAG, "App already has crash dialog: " + proc);
                        if (res != null) {
                            res.set(0);
                        }
                        return;
                    }
                    boolean isBackground = (UserHandle.getAppId(proc.uid)
                            >= Process.FIRST_APPLICATION_UID
                            && proc.pid != MY_PID);
                    for (int userId : mCurrentProfileIds) {
                        isBackground &= (proc.userId != userId);
                    }
                    if (isBackground && !showBackground) {
                        Slog.w(TAG, "Skipping crash dialog of " + proc + ": background");
                        if (res != null) {
                            res.set(0);
                        }
                        return;
                    }
                    if (mShowDialogs && !mSleeping && !mShuttingDown) {
                        Dialog d = new AppErrorDialog(mContext,
                                ActivityManagerService.this, res, proc);
                        d.show();
                        proc.crashDialog = d;
                    } else {
                        // The device is asleep, so just pretend that the user
                        // saw a crash dialog and hit "force quit".
                        if (res != null) {
                            res.set(0);
                        }
                    }
                }

                ensureBootCompleted();
            }
```
可以看到在方法体中我们创建了一个AppErrorDialog对象，并执行了show方法，这样该Dialog就会被显示出来。而这里的Dialog的显示内容就是：App already has crash dialog: ....

O(∩_∩)O哈哈~，原来我们App崩溃的时候弹出昂的异常提示框就是在这里弹出来的。这里对AppErrorDialog不做过多的介绍，在其的构造方法中，调用了如下的代码：

```
// After the timeout, pretend the user clicked the quit button
        mHandler.sendMessageDelayed(
                mHandler.obtainMessage(FORCE_QUIT),
                DISMISS_TIMEOUT);
```
这里的常量DISMISS_TIME = 5 * 60 * 1000，也就是五分钟，相当于这里发送了一个延时异步消息五分钟之后取消崩溃弹窗的显示。所以我们的App若果崩溃之后不主动取消弹窗，崩溃弹窗也会默认在五分钟之后取消。

好吧，文章开头我们所提到的两个问题我们已经解决掉一个了，force close弹窗是如何弹出来的，相信大家已经有所了解了，其实第二个问题也已经说明了，我们知道系统默认的App异常处理流程就是从Thread.setDefaultUncaughtExceptionHandler(new UncaughtHandler());开始的，并创建了自己的UncaughtHandler对象，那么我们接管系统默认的异常处理逻辑其实也就是从Thread.setDefaultUncaughtExceptionHandler开始的，并重写其uncaughtException方法，那么App异常信息就会被我们自定义的UncaughtHandler所捕获，捕获之后奔溃信息的记录与上报就可以做定制了。。。

这样我们就大概分析完成了Android系统的异常处理流程。O(∩_∩)O哈哈~

总结：

- App应用进程启动时会经过一系列的调用，执行Thread.setDefaultUncaughtExceptionHandler方法，创建默认的UncaughtHandler异常处理对象。

- 默认的UncaughtHandler异常处理对象，在其回调方法uncaughtException方法中会执行弹窗异常弹窗的操作，这也就是我们原生的force close弹窗，并且弹窗如果不主动取消的话，会在五分钟内默认取消。

- 自定义App的全局异常处理逻辑，需要接管UncaughtHandler，也就是创建自身的UncaughtHandler对象，并调用Thread.setDefaultUncaughtExceptionHandler方法，接管默认的异常处理逻辑。

- force close弹窗，弹窗的时候App应用可能已经退出，该弹窗的弹窗是SystemServer进程中的ActivityManagerService服务控制的。

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
