> 转载请标明出处：<a href="http://blog.csdn.net/qq_23547831/article/details/51105171">一片枫叶的专栏</a>

上面一文中我们讲过android系统中比较重要的几个进程：init进程，Zygote进程，SystemServer进程已经各种应用进程，其中Zygote进程是整个android系统的根进程，包含SystemServer进程已经各种应用进程在内的进程都是通过Zygote进程fork出来的，具体可参见：<a href="http://blog.csdn.net/qq_23547831/article/details/51104873"> android源码解析之（八）-->Zygote进程启动流程</a>
那么SystemServer进程是做什么用的呢？

其实SystemServer进程主要的作用是在这个进程中启动各种系统服务，比如ActivityManagerService，PackageManagerService，WindowManagerService服务，以及各种系统性的服务其实都是在SystemServer进程中启动的，而当我们的应用需要使用各种系统服务的时候其实也是通过与SystemServer进程通讯获取各种服务对象的句柄的。

由上一篇文章我们知道SystemServer进程其实也是有Zygote进程fork出来的，并且执行其main方法，那么这里我们以android23的源码为例，看一下SystemServer的main方法的执行逻辑：

```
/**
     * The main entry point from zygote.
     */
    public static void main(String[] args) {
        new SystemServer().run();
    }
```
这里比较简单，只是new出一个SystemServer对象并执行其run方法，查看SystemServer类的定义我们知道其实final类型的，所以我们一般不能重写或者继承。

然后我们查看run方法：

```
private void run() {
        // If a device's clock is before 1970 (before 0), a lot of
        // APIs crash dealing with negative numbers, notably
        // java.io.File#setLastModified, so instead we fake it and
        // hope that time from cell towers or NTP fixes it shortly.
        if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
            Slog.w(TAG, "System clock is before 1970; setting to 1970.");
            SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
        }

        // If the system has "persist.sys.language" and friends set, replace them with
        // "persist.sys.locale". Note that the default locale at this point is calculated
        // using the "-Duser.locale" command line flag. That flag is usually populated by
        // AndroidRuntime using the same set of system properties, but only the system_server
        // and system apps are allowed to set them.
        //
        // NOTE: Most changes made here will need an equivalent change to
        // core/jni/AndroidRuntime.cpp
        if (!SystemProperties.get("persist.sys.language").isEmpty()) {
            final String languageTag = Locale.getDefault().toLanguageTag();

            SystemProperties.set("persist.sys.locale", languageTag);
            SystemProperties.set("persist.sys.language", "");
            SystemProperties.set("persist.sys.country", "");
            SystemProperties.set("persist.sys.localevar", "");
        }

        // Here we go!
        Slog.i(TAG, "Entered the Android system server!");
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_SYSTEM_RUN, SystemClock.uptimeMillis());

        // In case the runtime switched since last boot (such as when
        // the old runtime was removed in an OTA), set the system
        // property so that it is in sync. We can't do this in
        // libnativehelper's JniInvocation::Init code where we already
        // had to fallback to a different runtime because it is
        // running as root and we need to be the system user to set
        // the property. http://b/11463182
        SystemProperties.set("persist.sys.dalvik.vm.lib.2", VMRuntime.getRuntime().vmLibrary());

        // Enable the sampling profiler.
        if (SamplingProfilerIntegration.isEnabled()) {
            SamplingProfilerIntegration.start();
            mProfilerSnapshotTimer = new Timer();
            mProfilerSnapshotTimer.schedule(new TimerTask() {
                @Override
                public void run() {
                    SamplingProfilerIntegration.writeSnapshot("system_server", null);
                }
            }, SNAPSHOT_INTERVAL, SNAPSHOT_INTERVAL);
        }

        // Mmmmmm... more memory!
        VMRuntime.getRuntime().clearGrowthLimit();

        // The system server has to run all of the time, so it needs to be
        // as efficient as possible with its memory usage.
        VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);

        // Some devices rely on runtime fingerprint generation, so make sure
        // we've defined it before booting further.
        Build.ensureFingerprintProperty();

        // Within the system server, it is an error to access Environment paths without
        // explicitly specifying a user.
        Environment.setUserRequired(true);

        // Ensure binder calls into the system always run at foreground priority.
        BinderInternal.disableBackgroundScheduling(true);

        // Prepare the main looper thread (this thread).
        android.os.Process.setThreadPriority(
                android.os.Process.THREAD_PRIORITY_FOREGROUND);
        android.os.Process.setCanSelfBackground(false);
        Looper.prepareMainLooper();

        // Initialize native services.
        System.loadLibrary("android_servers");

        // Check whether we failed to shut down last time we tried.
        // This call may not return.
        performPendingShutdown();

        // Initialize the system context.
        createSystemContext();

        // Create the system service manager.
        mSystemServiceManager = new SystemServiceManager(mSystemContext);
        LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);

        // Start services.
        try {
            startBootstrapServices();
            startCoreServices();
            startOtherServices();
        } catch (Throwable ex) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting system services", ex);
            throw ex;
        }

        // For debug builds, log event loop stalls to dropbox for analysis.
        if (StrictMode.conditionallyEnableDebugLogging()) {
            Slog.i(TAG, "Enabled StrictMode for system server main thread.");
        }

        // Loop forever.
        Looper.loop();
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```
好吧，代码比较多，慢慢看。。。

```
if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
            Slog.w(TAG, "System clock is before 1970; setting to 1970.");
            SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
        }
```
首先判断系统当前时间，若当前时间小于1970年1月1日，则一些初始化操作可能会处所，所以当系统的当前时间小于1970年1月1日的时候，设置系统当前时间为该时间点。

然后代码：

```
if (!SystemProperties.get("persist.sys.language").isEmpty()) {
            final String languageTag = Locale.getDefault().toLanguageTag();

            SystemProperties.set("persist.sys.locale", languageTag);
            SystemProperties.set("persist.sys.language", "");
            SystemProperties.set("persist.sys.country", "");
            SystemProperties.set("persist.sys.localevar", "");
        }
```
主要是设置系统的语言环境等；
下面的主要是设置虚拟机运行内存，加载运行库，设置SystemServer的异步消息，具体的异步消息机制可参见：<a href="http://blog.csdn.net/qq_23547831/article/details/50751687"> android源码解析之（二）-->异步消息机制</a>

然后下面的代码是：

```
// Initialize the system context.
        createSystemContext();

        // Create the system service manager.
        mSystemServiceManager = new SystemServiceManager(mSystemContext);
        LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);

        // Start services.
        try {
            startBootstrapServices();
            startCoreServices();
            startOtherServices();
        } catch (Throwable ex) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting system services", ex);
            throw ex;
        }
```

首先调用createSystemContext()方法：

```
private void createSystemContext() {
        ActivityThread activityThread = ActivityThread.systemMain();
        mSystemContext = activityThread.getSystemContext();
        mSystemContext.setTheme(android.R.style.Theme_DeviceDefault_Light_DarkActionBar);
    }
```
可以看到在SystemServer进程中也存在着Context对象，并且是通过ActivityThread.systemMain方法创建context的，这一部分的逻辑以后会通过介绍Activity的启动流程来介绍，这里就不在扩展，只知道在SystemServer进程中也需要创建Context对象。

然后通过SystemServiceManager的构造方法创建了一个新的SystemServiceManager对象，我们知道SystemServer进程主要是用来构建系统各种service服务的，而SystemServiceManager就是这些服务的管理对象。

然后调用：

```
LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
```
是将SystemServiceManager对象保存SystemServer进程中的一个数据结构中。

最后开始执行：

```
// Start services.
        try {
            startBootstrapServices();
            startCoreServices();
            startOtherServices();
        } catch (Throwable ex) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting system services", ex);
            throw ex;
        }
```
里面主要涉及了是三个方法：
startBootstrapServices() 主要用于启动系统Boot级服务
startCoreServices() 主要用于启动系统核心的服务
startOtherServices() 主要用于启动一些非紧要或者是非需要及时启动的服务

下面我们重点介绍这三个启动服务的方法，包括启动那些系统服务已经如何启动系统服务等。

首先看一下startBootstrapServices方法：

```
private void startBootstrapServices() {
        // Wait for installd to finish starting up so that it has a chance to
        // create critical directories such as /data/user with the appropriate
        // permissions.  We need this to complete before we initialize other services.
        Installer installer = mSystemServiceManager.startService(Installer.class);

        // Activity manager runs the show.
        mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);

        // Power manager needs to be started early because other services need it.
        // Native daemons may be watching for it to be registered so it must be ready
        // to handle incoming binder calls immediately (including being able to verify
        // the permissions for those calls).
        mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);

        // Now that the power manager has been started, let the activity manager
        // initialize power management features.
        mActivityManagerService.initPowerManagement();

        // Manages LEDs and display backlight so we need it to bring up the display.
        mSystemServiceManager.startService(LightsService.class);

        // Display manager is needed to provide display metrics before package manager
        // starts up.
        mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);

        // We need the default display before we can initialize the package manager.
        mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);

        // Only run "core" apps if we're encrypting the device.
        String cryptState = SystemProperties.get("vold.decrypt");
        if (ENCRYPTING_STATE.equals(cryptState)) {
            Slog.w(TAG, "Detected encryption in progress - only parsing core apps");
            mOnlyCore = true;
        } else if (ENCRYPTED_STATE.equals(cryptState)) {
            Slog.w(TAG, "Device encrypted - only parsing core apps");
            mOnlyCore = true;
        }

        // Start the package manager.
        Slog.i(TAG, "Package Manager");
        mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
        mFirstBoot = mPackageManagerService.isFirstBoot();
        mPackageManager = mSystemContext.getPackageManager();

        Slog.i(TAG, "User Service");
        ServiceManager.addService(Context.USER_SERVICE, UserManagerService.getInstance());

        // Initialize attribute cache used to cache resources from packages.
        AttributeCache.init(mSystemContext);

        // Set up the Application instance for the system process and get started.
        mActivityManagerService.setSystemProcess();

        // The sensor service needs access to package manager service, app ops
        // service, and permissions service, therefore we start it after them.
        startSensorService();
    }
```

首先执行：

```
Installer installer = mSystemServiceManager.startService(Installer.class);
```
mSystemServiceManager是系统服务管理对象，在main方法中已经创建完成，这里我们看一下其startService方法的具体实现：

```
public <T extends SystemService> T startService(Class<T> serviceClass) {
        final String name = serviceClass.getName();
        Slog.i(TAG, "Starting " + name);

        // Create the service.
        if (!SystemService.class.isAssignableFrom(serviceClass)) {
            throw new RuntimeException("Failed to create " + name
                    + ": service must extend " + SystemService.class.getName());
        }
        final T service;
        try {
            Constructor<T> constructor = serviceClass.getConstructor(Context.class);
            service = constructor.newInstance(mContext);
        } catch (InstantiationException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service could not be instantiated", ex);
        } catch (IllegalAccessException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service must have a public constructor with a Context argument", ex);
        } catch (NoSuchMethodException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service must have a public constructor with a Context argument", ex);
        } catch (InvocationTargetException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service constructor threw an exception", ex);
        }

        // Register it.
        mServices.add(service);

        // Start it.
        try {
            service.onStart();
        } catch (RuntimeException ex) {
            throw new RuntimeException("Failed to start service " + name
                    + ": onStart threw an exception", ex);
        }
        return service;
    }
```
可以看到我们通过反射器构造方法创建出服务类，然后添加到SystemServiceManager的服务列表数据中，最后调用了service.onStart()方法，因为我们传递的是Installer.class，我们这里我们查看一下Installer的onStart方法：

```
@Override
    public void onStart() {
        Slog.i(TAG, "Waiting for installd to be ready.");
        mInstaller.waitForConnection();
    }
```
很简单就是执行了mInstaller的waitForConnection方法，这里简单介绍一下Installer类，该类是系统安装apk时的一个服务类，继承SystemService（系统服务的一个抽象接口），我们需要在启动完成Installer服务之后才能启动其他的系统服务。
然后查看waitForConnection（）方法：

```
public void waitForConnection() {
        for (;;) {
            if (execute("ping") >= 0) {
                return;
            }
            Slog.w(TAG, "installd not ready");
            SystemClock.sleep(1000);
        }
    }
```
通过追踪代码可以发现，其在不断的通过ping命令连接Zygote进程（SystemServer和Zygote进程通过socket方式通讯，其他进程通过Binder方式通讯）；

**总结：**
在开始执行启动服务之前总是会先尝试通过socket方式连接Zygote进程，在成功连接之后才会开始启动其他服务。

继续来看startBootstrapServices方法：

```
// Activity manager runs the show.
        mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);
```
这段代码主要是用于启动ActivityManagerService服务，并为其设置SysServiceManager和Installer。ActivityManagerService是系统中一个非常重要的服务，Activity，service，Broadcast，contentProvider都需要通过其余系统交互。

首先看一下Lifecycle类的定义：

```
public static final class Lifecycle extends SystemService {
        private final ActivityManagerService mService;

        public Lifecycle(Context context) {
            super(context);
            mService = new ActivityManagerService(context);
        }

        @Override
        public void onStart() {
            mService.start();
        }

        public ActivityManagerService getService() {
            return mService;
        }
    }
```
可以看到其实ActivityManagerService的一个静态内部类，在其构造方法中会创建一个ActivityManagerService，通过刚刚对Installer服务的分析我们知道，SystemServiceManager的startService方法会调用服务的onStart()方法，而在Lifecycle类的定义中我们看到其onStart（）方法直接调用了mService.start()方法，mService是Lifecycle类中对ActivityManagerService的引用，所以我们可以看一下ActivityManagerService的start方法的实现：

```
private void start() {
        Process.removeAllProcessGroups();
        mProcessCpuThread.start();

        mBatteryStatsService.publish(mContext);
        mAppOpsService.publish(mContext);
        Slog.d("AppOps", "AppOpsService published");
        LocalServices.addService(ActivityManagerInternal.class, new LocalService());
    }
```
由于ActivityManagerService的创建过程比较复杂这里不做过多的分析了，主要是在其构造方法中初始化了一些变量。

然后是启动PowerManagerService服务：

```
mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
```
启动方式跟上面的ActivityManagerService服务相似都会调用其构造方法和onStart方法，PowerManagerService主要用于计算系统中和Power相关的计算，然后决策系统应该如何反应。同时协调Power如何与系统其它模块的交互，比如没有用户活动时，屏幕变暗等等。

然后是启动LightsService服务

```
mSystemServiceManager.startService(LightsService.class);
```
主要是手机中关于闪光灯，LED等相关的服务；也是会调用LightsService的构造方法和onStart方法；

然后是启动DisplayManagerService服务

```
mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);
```
主要是手机显示方面的服务；

然后是启动PackageManagerService，该服务也是android系统中一个比较重要的服务，包括多apk文件的安装，解析，删除，卸载等等操作。

```
Slog.i(TAG, "Package Manager");
        mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
        mFirstBoot = mPackageManagerService.isFirstBoot();
        mPackageManager = mSystemContext.getPackageManager();
```
可以看到PackageManagerService服务的启动方式与其他服务的启动方式有一些区别，直接调用了PackageManagerService的静态main方法，这里我们看一下其main方法的具体实现：

```
public static PackageManagerService main(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        PackageManagerService m = new PackageManagerService(context, installer,
                factoryTest, onlyCore);
        ServiceManager.addService("package", m);
        return m;
    }
```
可以看到也是直接使用new的方式创建了一个PackageManagerService对象，并在其构造方法中初始化相关变量，最后调用了ServiceManager.addService方法，主要是通过Binder机制与JNI层交互，这里不再扩展。

然后启动UserManagerService和SensorService，至此startBootstrapServices方法执行完成。

然后查看startCoreServices方法：

```
private void startCoreServices() {
        // Tracks the battery level.  Requires LightService.
        mSystemServiceManager.startService(BatteryService.class);

        // Tracks application usage stats.
        mSystemServiceManager.startService(UsageStatsService.class);
        mActivityManagerService.setUsageStatsManager(
                LocalServices.getService(UsageStatsManagerInternal.class));
        // Update after UsageStatsService is available, needed before performBootDexOpt.
        mPackageManagerService.getUsageStatsIfNoPackageUsageInfo();

        // Tracks whether the updatable WebView is in a ready state and watches for update installs.
        mSystemServiceManager.startService(WebViewUpdateService.class);
    }
```
可以看到这里启动了BatteryService（电池相关服务），UsageStatsService，WebViewUpdateService服务等。

最后看一下startOtherServices方法，主要用于启动系统中其他的服务，代码很多，这里就不贴代码了，启动的流程和ActivityManagerService的流程类似，会调用服务的构造方法与onStart方法初始化变量。

总结：

- SystemServer进程是android中一个很重要的进程由Zygote进程启动；

- SystemServer进程主要用于启动系统中的服务；

- SystemServer进程启动服务的启动函数为main函数；

- SystemServer在执行过程中首先会初始化一些系统变量，加载类库，创建Context对象，创建SystemServiceManager对象等之后才开始启动系统服务；

- SystemServer进程将系统服务分为三类：boot服务，core服务和other服务，并逐步启动

- SertemServer进程在尝试启动服务之前会首先尝试与Zygote建立socket通讯，只有通讯成功之后才会开始尝试启动服务；

- 创建的系统服务过程中主要通过SystemServiceManager对象来管理，通过调用服务对象的构造方法和onStart方法初始化服务的相关变量；

- 服务对象都有自己的异步消息对象，并运行在单独的线程中；

另外对android源码解析方法感兴趣的可参考我的：
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50634435"> android源码解析之（一）-->android项目构建过程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50751687">android源码解析之（二）-->异步消息机制</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50803849">android源码解析之（三）-->异步任务AsyncTask</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50936584">android源码解析之（四）-->HandlerThread</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50958757">android源码解析之（五）-->IntentService</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50963006">android源码解析之（六）-->Log</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50971968">android源码解析之（七）-->LruCache</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51104873">android源码解析之（八）-->Zygote进程启动流程</a>

