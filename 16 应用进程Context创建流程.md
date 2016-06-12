今天讲讲应用进程Context的创建流程，相信大家平时在开发过程中经常会遇到对Context对象的使用，Application是Context，Activity是Context，Service也是Context，所以有一个经典的问题是一个App中一共有多少个Context？

这个问题的答案是Application + N个Activity + N个Service。


还有就是我们平时在使用Context过程中许多时候由于使用不当，可能会造成内存泄露的情况等等，这个也是需要我们注意的。这里有篇不错的文章：
<a href="http://blog.csdn.net/feiduclear_up/article/details/47356289"> Android Context 是什么？</a>

好吧，什么叫应用进程Context呢？这是指的是Application所代表的Context的创建流程，还记得我们前几篇写的应用进程创建流程么？
<a href="http://blog.csdn.net/qq_23547831/article/details/51119333"> android源码解析之（十一）-->应用进程启动流程</a>
最后我们得出结论，应用进程的起始方法是ActivityThread.main方法，好吧，

由于还未讲解Service相关知识，这里暂时讲解一下Activity与Application中Context对象的创建过程。

首先我们就从ActivityThread.main方法开始看一下Application的创建流程。。。

```
public static void main(String[] args) {
        ...
        ActivityThread thread = new ActivityThread();
        thread.attach(false);
        ...
    }
```
这里我们发现在方法体中我们创建了一个ActivityThread对象并执行了attach方法：

```
private void attach(boolean system) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {
            ViewRootImpl.addFirstDrawHandler(new Runnable() {
                @Override
                public void run() {
                    ensureJitEnabled();
                }
            });
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            final IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                // Ignore
            }
            // Watch for getting close to heap limit.
            BinderInternal.addGcWatcher(new Runnable() {
                @Override public void run() {
                    if (!mSomeActivitiesChanged) {
                        return;
                    }
                    Runtime runtime = Runtime.getRuntime();
                    long dalvikMax = runtime.maxMemory();
                    long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
                    if (dalvikUsed > ((3*dalvikMax)/4)) {
                        if (DEBUG_MEMORY_TRIM) Slog.d(TAG, "Dalvik max=" + (dalvikMax/1024)
                                + " total=" + (runtime.totalMemory()/1024)
                                + " used=" + (dalvikUsed/1024));
                        mSomeActivitiesChanged = false;
                        try {
                            mgr.releaseSomeActivities(mAppThread);
                        } catch (RemoteException e) {
                        }
                    }
                }
            });
        } else {
          ...  
        }
    }
```
这里看一下重点实现，我们可以发现在方法体中调用了ActivityManagerNative.getDefault().attachApplication(mAppThread)
看过我的前几篇文章的童鞋应该知道这里就是一个Binder进程间通讯，其实上执行的是ActivityManagerService.attachApplication方法，具体的可以参考前几篇文章的介绍，好吧，既然这样我们看一下ActivityManagerService.attachApplication方法的具体实现。

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
然后这里面又调用了attachApplicationLocked方法：

```
private final boolean attachApplicationLocked(IApplicationThread 	  thread, int pid) {

        ...
        thread.bindApplication(processName, appInfo, providers, app.instrumentationClass, profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace, isRestrictedBackupMode || !normalMode, app.persistent, new Configuration(mConfiguration), app.compat,
getCommonServicesLocked(app.isolated),
mCoreSettingsObserver.getCoreSettingsLocked());
        ...
```
可以看到这里面又调用了IApplication.bindApplication，从方法名称中我们可以看出这里应该是绑定Application的方法，跟上面的ActivityManangerNative类似的，前面几篇文章中我们已经做过介绍，IApplicationThread是ActivityThread中ApplicationThread binder对象的客户端，所以这里最终调用的是ApplicationThread的bindApplication方法，既然这样，我们来看一下ApplicationThread的bindApplication的实现：

```
public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableOpenGlTrace, boolean isRestrictedBackupMode, boolean persistent,
                Configuration config, CompatibilityInfo compatInfo, Map<String, IBinder> services,
                Bundle coreSettings) {

            if (services != null) {
                // Setup the service cache in the ServiceManager
                ServiceManager.initServiceCache(services);
            }

            setCoreSettings(coreSettings);

            /*
             * Two possible indications that this package could be
             * sharing its runtime with other packages:
             *
             * 1.) the sharedUserId attribute is set in the manifest,
             *     indicating a request to share a VM with other
             *     packages with the same sharedUserId.
             *
             * 2.) the application element of the manifest has an
             *     attribute specifying a non-default process name,
             *     indicating the desire to run in another packages VM.
             *
             * If sharing is enabled we do not have a unique application
             * in a process and therefore cannot rely on the package
             * name inside the runtime.
             */
            IPackageManager pm = getPackageManager();
            android.content.pm.PackageInfo pi = null;
            try {
                pi = pm.getPackageInfo(appInfo.packageName, 0, UserHandle.myUserId());
            } catch (RemoteException e) {
            }
            if (pi != null) {
                boolean sharedUserIdSet = (pi.sharedUserId != null);
                boolean processNameNotDefault =
                (pi.applicationInfo != null &&
                 !appInfo.packageName.equals(pi.applicationInfo.processName));
                boolean sharable = (sharedUserIdSet || processNameNotDefault);

                // Tell the VMRuntime about the application, unless it is shared
                // inside a process.
                if (!sharable) {
                    VMRuntime.registerAppInfo(appInfo.packageName, appInfo.dataDir,
                                            appInfo.processName);
                }
            }

            AppBindData data = new AppBindData();
            data.processName = processName;
            data.appInfo = appInfo;
            data.providers = providers;
            data.instrumentationName = instrumentationName;
            data.instrumentationArgs = instrumentationArgs;
            data.instrumentationWatcher = instrumentationWatcher;
            data.instrumentationUiAutomationConnection = instrumentationUiConnection;
            data.debugMode = debugMode;
            data.enableOpenGlTrace = enableOpenGlTrace;
            data.restrictedBackupMode = isRestrictedBackupMode;
            data.persistent = persistent;
            data.config = config;
            data.compatInfo = compatInfo;
            data.initProfilerInfo = profilerInfo;
            sendMessage(H.BIND_APPLICATION, data);
        }
```
好吧，最后调用了ActivityThread.sendMessage()...

```
private void sendMessage(int what, Object obj) {
        sendMessage(what, obj, 0, 0, false);
    }
```
然后我们看一下其sendMessage的重载方法：

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
可以发现这里调用了mH的sendMessage方法，最后通过Handler的异步消息机制被mH的handleMessage方法处理，然后根据Message.what选择处理分支，最终调用了ActivityThread的handleBindApplication方法。

```
private void handleBindApplication(AppBindData data) {
        ...
        // 创建Instrumentation
        if (data.instrumentationName != null) {
            InstrumentationInfo ii = null;
            try {
                ii = appContext.getPackageManager().
                    getInstrumentationInfo(data.instrumentationName, 0);
            } catch (PackageManager.NameNotFoundException e) {
            }
            if (ii == null) {
                throw new RuntimeException(
                    "Unable to find instrumentation info for: "
                    + data.instrumentationName);
            }

            mInstrumentationPackageName = ii.packageName;
            mInstrumentationAppDir = ii.sourceDir;
            mInstrumentationSplitAppDirs = ii.splitSourceDirs;
            mInstrumentationLibDir = ii.nativeLibraryDir;
            mInstrumentedAppDir = data.info.getAppDir();
            mInstrumentedSplitAppDirs = data.info.getSplitAppDirs();
            mInstrumentedLibDir = data.info.getLibDir();

            ApplicationInfo instrApp = new ApplicationInfo();
            instrApp.packageName = ii.packageName;
            instrApp.sourceDir = ii.sourceDir;
            instrApp.publicSourceDir = ii.publicSourceDir;
            instrApp.splitSourceDirs = ii.splitSourceDirs;
            instrApp.splitPublicSourceDirs = ii.splitPublicSourceDirs;
            instrApp.dataDir = ii.dataDir;
            instrApp.nativeLibraryDir = ii.nativeLibraryDir;
            LoadedApk pi = getPackageInfo(instrApp, data.compatInfo,
                    appContext.getClassLoader(), false, true, false);
            ContextImpl instrContext = ContextImpl.createAppContext(this, pi);

            try {
                java.lang.ClassLoader cl = instrContext.getClassLoader();
                mInstrumentation = (Instrumentation)
                    cl.loadClass(data.instrumentationName.getClassName()).newInstance();
            } catch (Exception e) {
                throw new RuntimeException(
                    "Unable to instantiate instrumentation "
                    + data.instrumentationName + ": " + e.toString(), e);
            }

            mInstrumentation.init(this, instrContext, appContext,
                   new ComponentName(ii.packageName, ii.name), data.instrumentationWatcher,
                   data.instrumentationUiAutomationConnection);

            if (mProfiler.profileFile != null && !ii.handleProfiling
                    && mProfiler.profileFd == null) {
                mProfiler.handlingProfiling = true;
                File file = new File(mProfiler.profileFile);
                file.getParentFile().mkdirs();
                Debug.startMethodTracing(file.toString(), 8 * 1024 * 1024);
            }

        } else {
            mInstrumentation = new Instrumentation();
        }

		...
		/ If the app is being launched for full backup or restore, bring it up in
            // a restricted environment with the base application class.
            Application app = data.info.makeApplication(data.restrictedBackupMode, null);
            mInitialApplication = app;
		...
        try {
           mInstrumentation.onCreate(data.instrumentationArgs);
         }
         catch (Exception e) {
                throw new RuntimeException(
                    "Exception thrown in onCreate() of "
                    + data.instrumentationName + ": " + e.toString(), e);
            }

            try {
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                if (!mInstrumentation.onException(app, e)) {
                    throw new RuntimeException(
                        "Unable to create application " + app.getClass().getName()
                        + ": " + e.toString(), e);
                }
            }
        } finally {
            StrictMode.setThreadPolicy(savedPolicy);
        }
    }
```
这个方法的方法体比较长，我们挑重点的看，可以看到方法体中系统通过反射机制创建了Instrumentation对象，并执行了init方法，执行了Insrtumentation对象的初始化。然后我们调用了LockedApk.makeApplication方法创建了Application对象，我们来看一下其具体的实现逻辑：

```
public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        if (mApplication != null) {
            return mApplication;
        }

        Application app = null;

        String appClass = mApplicationInfo.className;
        if (forceDefaultAppClass || (appClass == null)) {
            appClass = "android.app.Application";
        }

        try {
            java.lang.ClassLoader cl = getClassLoader();
            if (!mPackageName.equals("android")) {
                initializeJavaContextClassLoader();
            }
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
        } catch (Exception e) {
            if (!mActivityThread.mInstrumentation.onException(app, e)) {
                throw new RuntimeException(
                    "Unable to instantiate application " + appClass
                    + ": " + e.toString(), e);
            }
        }
        mActivityThread.mAllApplications.add(app);
        mApplication = app;

        if (instrumentation != null) {
            try {
                instrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                if (!instrumentation.onException(app, e)) {
                    throw new RuntimeException(
                        "Unable to create application " + app.getClass().getName()
                        + ": " + e.toString(), e);
                }
            }
        }

        // Rewrite the R 'constants' for all library apks.
        SparseArray<String> packageIdentifiers = getAssets(mActivityThread)
                .getAssignedPackageIdentifiers();
        final int N = packageIdentifiers.size();
        for (int i = 0; i < N; i++) {
            final int id = packageIdentifiers.keyAt(i);
            if (id == 0x01 || id == 0x7f) {
                continue;
            }

            rewriteRValues(getClassLoader(), packageIdentifiers.valueAt(i), id);
        }

        return app;
    }
```
可以发现这里也是以反射的机制创建了Application对象，并创建了一个ContextImpl对象，并将Application与ContextImpl建立关联。。。

继续回到我们的ActivityThread的handleBindApplication方法，在创建了Application对象之后我们调用了Instrumentation的onCreate方法，然后调用了Instrumentation的callApplicationOnCreate方法，我们来看一下其具体实现：

```
public void callApplicationOnCreate(Application app) {
        app.onCreate();
    }
```
咋样？原来Application的onCreate生命周期方法是在这里回调滴啊。

这样我们整个Application的创建执行流程就讲解完了。

总结：

- 应用进程启动 --> 创建Instrumentation --> 创建Application对象 --> 创建Application相关的ContextImpl对象；

- ActivityThread.main方法--> ActivityManagerService.bindApplication方法 --> ActivityThread.handleBindApplication --> 创建Instrumentation，创建Application；

- 每个应用进程对应一个Instrumentation，对应一个Application；

- Instrumentation与Application都是通过java反射机制创建；

- Application创建过程中会同时创建一个ContextImpl对象，并建立关联；

<br><br><br>
接下来我们来看一下Acitivty中的Context创建流程，大家都知道我们Activity的具体创建过程是在ActivityThread的performLaunchActivity,可参见：<a href="http://blog.csdn.net/qq_23547831/article/details/51224992"> android源码解析之（十四）-->Activity启动流程</a>，这里我们看一下其具体的实现：

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

        try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);
			...
            if (activity != null) {
                Context appContext = createBaseContextForActivity(r, activity);
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                r.activity = activity;
                r.stopped = true;
                if (!r.activity.mFinished) {
                    activity.performStart();
                    r.stopped = false;
                }
                if (!r.activity.mFinished) {
                    if (r.isPersistable()) {
                        if (r.state != null || r.persistentState != null) {
                            mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                    r.persistentState);
                        }
                    } else if (r.state != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                    }
                }
                ...
        return activity;
    }
```
这里简要说明一下，Activity也是系统通过反射机制创建的，然后我们通过LockedApk.makeApplication创建一个Application，通过查看源码我们知道若这时候LockedApk中的mApplication不为空则直接返回当前的mApplication又因为当我们创建应用进程的时候Application已经被创建，所以当创建Activity的时候这时候Application肯定不为空，所以这时候返回的就是应用进程创建的时候创建的Application，这也从侧面说明了一个应用进程对应着一个Application。然后我们通过createBaseContextForActivity创建了一个ContextImpl对象。

```
private Context createBaseContextForActivity(ActivityClientRecord r, final Activity activity) {
        int displayId = Display.DEFAULT_DISPLAY;
        try {
            displayId = ActivityManagerNative.getDefault().getActivityDisplayId(r.token);
        } catch (RemoteException e) {
        }

        ContextImpl appContext = ContextImpl.createActivityContext(
                this, r.packageInfo, displayId, r.overrideConfig);
        appContext.setOuterContext(activity);
        Context baseContext = appContext;

        final DisplayManagerGlobal dm = DisplayManagerGlobal.getInstance();
        // For debugging purposes, if the activity's package name contains the value of
        // the "debug.use-second-display" system property as a substring, then show
        // its content on a secondary display if there is one.
        String pkgName = SystemProperties.get("debug.second-display.pkg");
        if (pkgName != null && !pkgName.isEmpty()
                && r.packageInfo.mPackageName.contains(pkgName)) {
            for (int id : dm.getDisplayIds()) {
                if (id != Display.DEFAULT_DISPLAY) {
                    Display display =
                            dm.getCompatibleDisplay(id, appContext.getDisplayAdjustments(id));
                    baseContext = appContext.createDisplayContext(display);
                    break;
                }
            }
        }
        return baseContext;
    }
```
可以发现这里创建了一个ContextImpl对象，并通过ContextImpl的setOuterContext方法，让该ContextImpl持有了Activity的引用，继续往下看，我们调用了activity.attach方法，查看一下该方法的实现逻辑：

```
final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor) {
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);

        mWindow = new PhoneWindow(this);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
        mUiThread = Thread.currentThread();

        mMainThread = aThread;
        mInstrumentation = instr;
        mToken = token;
        mIdent = ident;
        mApplication = application;
        mIntent = intent;
        mReferrer = referrer;
        mComponent = intent.getComponent();
        mActivityInfo = info;
        mTitle = title;
        mParent = parent;
        mEmbeddedID = id;
        mLastNonConfigurationInstances = lastNonConfigurationInstances;
        if (voiceInteractor != null) {
            if (lastNonConfigurationInstances != null) {
                mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
            } else {
                mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
                        Looper.myLooper());
            }
        }

        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;
    }
```
除了一下初始化操作之外，还调用了attachBaseContext方法，让Activity持有了ContextImpl的引用，这样就相当于Activity与ContextImpl对象相互持有了对方的引用，并且Activity是继承与Context。

总结：

- Activity中创建ContextImpl对象的具体实现在ActivityThread的performLauncherAcitivty方法中；

- Activity的创建伴随着ContextImpl的创建，二者相互持有对方的引用；

- 创建Activity --> 创建Activity相关ContextImpl对象；

- 创建应用进程 --> 创建Application --> 创建Application相关ContextImpl对象；

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
