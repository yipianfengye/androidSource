上一篇文章中给大家分析了一下android系统启动之后调用PackageManagerService服务并解析系统特定目录，解析apk文件并安装的过程，这个安装过期实际上是没有图形界面的，底层调用的是我们平时比较熟悉的adb命令，那么我们平时安装apk文件的时候大部分是都过图形界面安装的，那么这种方式安装apk具体的流程是怎样的呢？

下面我们就来具体看一下apk的具体安装过程，相信大家都知道如果我们想在代码里执行apk的安装，那么一般都是这样：

```
Intent intent = new Intent(Intent.ACTION_VIEW);
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
intent.setDataAndType(Uri.parse("file://" + path),"application/vnd.android.package-archive");
context.startActivity(intent);
```
这样，我们就会打开安装apk文件的程序并执行安装逻辑了，那么这段代码具体是打开那个activity呢？好吧，从这个问题开始，我们来解析apk的安装流程...

这里跟大姐简单介绍一下android的源码，平时我们使用的android.jar里面的java源码只是android系统源码的一部分，还有好多源码并没有打入到android.jar中，这里为大家推荐一个android源码的地址：https://github.com/android
里面根据android系统的不同模块包含了许多android模块的源码。
![这里写图片描述](http://img.blog.csdn.net/20160421165129710)

这里我们找到platform_packages_apps_packageinstaller库，这里面就是android系统安装程序的源码了。
![这里写图片描述](http://img.blog.csdn.net/20160421165234648)

这里我们找到其androidManifest.xml，然后我们来看一下其具体的定义：

```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
          package="com.android.packageinstaller" coreApp="true">
 
    <original-package android:name="com.android.packageinstaller" />
 
    ...
 
    <application android:label="@string/app_name"
            android:allowBackup="false"
            android:theme="@style/Theme.DialogWhenLarge"
            android:supportsRtl="true">
 
        <activity android:name=".PackageInstallerActivity"
                android:configChanges="orientation|keyboardHidden|screenSize"
                android:excludeFromRecents="true">
            <intent-filter>
                <action android:name="android.intent.action.VIEW" />
                <action android:name="android.intent.action.INSTALL_PACKAGE" />
                <category android:name="android.intent.category.DEFAULT" />
                <data android:scheme="file" />
                <data android:mimeType="application/vnd.android.package-archive" />
            </intent-filter>
            <intent-filter>
                <action android:name="android.intent.action.INSTALL_PACKAGE" />
                <category android:name="android.intent.category.DEFAULT" />
                <data android:scheme="file" />
                <data android:scheme="package" />
            </intent-filter>
            <intent-filter>
                <action android:name="android.content.pm.action.CONFIRM_PERMISSIONS" />
                <category android:name="android.intent.category.DEFAULT" />
            </intent-filter>
        </activity>
 
        <activity android:name=".InstallAppProgress"
                android:configChanges="orientation|keyboardHidden|screenSize"
                android:exported="false" />
 
        <activity android:name=".UninstallerActivity"
                android:configChanges="orientation|keyboardHidden|screenSize"
                android:excludeFromRecents="true"
                android:theme="@style/Theme.AlertDialogActivity">
            <intent-filter android:priority="1">
                <action android:name="android.intent.action.DELETE" />
                <action android:name="android.intent.action.UNINSTALL_PACKAGE" />
                <category android:name="android.intent.category.DEFAULT" />
                <data android:scheme="package" />
            </intent-filter>
        </activity>
 
        <activity android:name=".UninstallAppProgress"
                android:configChanges="orientation|keyboardHidden|screenSize"
                android:exported="false" />
 
        <activity android:name=".permission.ui.GrantPermissionsActivity"
                android:configChanges="orientation|keyboardHidden|screenSize"
                android:excludeFromRecents="true"
                android:theme="@style/GrantPermissions">
            <intent-filter>
                <action android:name="android.content.pm.action.REQUEST_PERMISSIONS" />
                <category android:name="android.intent.category.DEFAULT" />
            </intent-filter>
        </activity>
 
        <activity android:name=".permission.ui.ManagePermissionsActivity"
                  android:configChanges="orientation|keyboardHidden|screenSize"
                  android:excludeFromRecents="true"
                  android:label="@string/app_permissions"
                  android:theme="@style/Settings"
                  android:permission="android.permission.GRANT_RUNTIME_PERMISSIONS">
            <intent-filter>
                <action android:name="android.intent.action.MANAGE_PERMISSIONS" />
                <action android:name="android.intent.action.MANAGE_APP_PERMISSIONS" />
                <action android:name="android.intent.action.MANAGE_PERMISSION_APPS" />
                <category android:name="android.intent.category.DEFAULT" />
            </intent-filter>
        </activity>
 
        <activity android:name=".permission.ui.OverlayWarningDialog"
                android:excludeFromRecents="true"
                android:theme="@android:style/Theme.DeviceDefault.Light.Dialog.NoActionBar" />
 
        <provider android:name=".wear.WearPackageIconProvider"
                  android:authorities="com.google.android.packageinstaller.wear.provider"
                  android:grantUriPermissions="true"
                  android:exported="true" />
 
        <activity android:name=".permission.ui.wear.WarningConfirmationActivity"
                  android:permission="android.permission.GRANT_RUNTIME_PERMISSIONS"
                  android:theme="@style/Settings"/>
    </application>
 
</manifest>
```
好吧，这里我们大概看一下Activity的定义，这里我们重点看一下PackageInstallerActivity的定义：

```
<activity android:name=".PackageInstallerActivity"
                android:configChanges="orientation|keyboardHidden|screenSize"
                android:excludeFromRecents="true">
            <intent-filter>
                <action android:name="android.intent.action.VIEW" />
                <action android:name="android.intent.action.INSTALL_PACKAGE" />
                <category android:name="android.intent.category.DEFAULT" />
                <data android:scheme="file" />
                <data android:mimeType="application/vnd.android.package-archive" />
            </intent-filter>
            <intent-filter>
                <action android:name="android.intent.action.INSTALL_PACKAGE" />
                <category android:name="android.intent.category.DEFAULT" />
                <data android:scheme="file" />
                <data android:scheme="package" />
            </intent-filter>
            <intent-filter>
                <action android:name="android.content.pm.action.CONFIRM_PERMISSIONS" />
                <category android:name="android.intent.category.DEFAULT" />
            </intent-filter>
        </activity>
```
恩？这里不就是我们刚刚定义的启动安装Apk activity的intent filter？好吧，所以说一开始我们调用的startActivity其实启动的就是PackageInstallerActivity，那么下面我们就看一下PackageInstellerActivity的具体实现：

```
@Override
    protected void onCreate(Bundle icicle) {
        super.onCreate(icicle);

        mPm = getPackageManager();
        mInstaller = mPm.getPackageInstaller();
        mUserManager = (UserManager) getSystemService(Context.USER_SERVICE);

        final Intent intent = getIntent();
        if (PackageInstaller.ACTION_CONFIRM_PERMISSIONS.equals(intent.getAction())) {
            final int sessionId = intent.getIntExtra(PackageInstaller.EXTRA_SESSION_ID, -1);
            final PackageInstaller.SessionInfo info = mInstaller.getSessionInfo(sessionId);
            if (info == null || !info.sealed || info.resolvedBaseCodePath == null) {
                Log.w(TAG, "Session " + mSessionId + " in funky state; ignoring");
                finish();
                return;
            }

            mSessionId = sessionId;
            mPackageURI = Uri.fromFile(new File(info.resolvedBaseCodePath));
            mOriginatingURI = null;
            mReferrerURI = null;
        } else {
            mSessionId = -1;
            mPackageURI = intent.getData();
            mOriginatingURI = intent.getParcelableExtra(Intent.EXTRA_ORIGINATING_URI);
            mReferrerURI = intent.getParcelableExtra(Intent.EXTRA_REFERRER);
        }

        final boolean unknownSourcesAllowedByAdmin = isUnknownSourcesAllowedByAdmin();
        final boolean unknownSourcesAllowedByUser = isUnknownSourcesEnabled();

        boolean requestFromUnknownSource = isInstallRequestFromUnknownSource(intent);
        mInstallFlowAnalytics = new InstallFlowAnalytics();
        mInstallFlowAnalytics.setContext(this);
        mInstallFlowAnalytics.setStartTimestampMillis(SystemClock.elapsedRealtime());
        mInstallFlowAnalytics.setInstallsFromUnknownSourcesPermitted(unknownSourcesAllowedByAdmin
                && unknownSourcesAllowedByUser);
        mInstallFlowAnalytics.setInstallRequestFromUnknownSource(requestFromUnknownSource);
        mInstallFlowAnalytics.setVerifyAppsEnabled(isVerifyAppsEnabled());
        mInstallFlowAnalytics.setAppVerifierInstalled(isAppVerifierInstalled());
        mInstallFlowAnalytics.setPackageUri(mPackageURI.toString());

        if (DeviceUtils.isWear(this)) {
            showDialogInner(DLG_NOT_SUPPORTED_ON_WEAR);
            mInstallFlowAnalytics.setFlowFinished(
                    InstallFlowAnalytics.RESULT_NOT_ALLOWED_ON_WEAR);
            return;
        }

        final String scheme = mPackageURI.getScheme();
        if (scheme != null && !"file".equals(scheme) && !"package".equals(scheme)) {
            Log.w(TAG, "Unsupported scheme " + scheme);
            setPmResult(PackageManager.INSTALL_FAILED_INVALID_URI);
            mInstallFlowAnalytics.setFlowFinished(
                    InstallFlowAnalytics.RESULT_FAILED_UNSUPPORTED_SCHEME);
            finish();
            return;
        }

        final PackageUtil.AppSnippet as;
        if ("package".equals(mPackageURI.getScheme())) {
            mInstallFlowAnalytics.setFileUri(false);
            try {
                mPkgInfo = mPm.getPackageInfo(mPackageURI.getSchemeSpecificPart(),
                        PackageManager.GET_PERMISSIONS | PackageManager.GET_UNINSTALLED_PACKAGES);
            } catch (NameNotFoundException e) {
            }
            if (mPkgInfo == null) {
                Log.w(TAG, "Requested package " + mPackageURI.getScheme()
                        + " not available. Discontinuing installation");
                showDialogInner(DLG_PACKAGE_ERROR);
                setPmResult(PackageManager.INSTALL_FAILED_INVALID_APK);
                mInstallFlowAnalytics.setPackageInfoObtained();
                mInstallFlowAnalytics.setFlowFinished(
                        InstallFlowAnalytics.RESULT_FAILED_PACKAGE_MISSING);
                return;
            }
            as = new PackageUtil.AppSnippet(mPm.getApplicationLabel(mPkgInfo.applicationInfo),
                    mPm.getApplicationIcon(mPkgInfo.applicationInfo));
        } else {
            mInstallFlowAnalytics.setFileUri(true);
            final File sourceFile = new File(mPackageURI.getPath());
            PackageParser.Package parsed = PackageUtil.getPackageInfo(sourceFile);

            // Check for parse errors
            if (parsed == null) {
                Log.w(TAG, "Parse error when parsing manifest. Discontinuing installation");
                showDialogInner(DLG_PACKAGE_ERROR);
                setPmResult(PackageManager.INSTALL_FAILED_INVALID_APK);
                mInstallFlowAnalytics.setPackageInfoObtained();
                mInstallFlowAnalytics.setFlowFinished(
                        InstallFlowAnalytics.RESULT_FAILED_TO_GET_PACKAGE_INFO);
                return;
            }
            mPkgInfo = PackageParser.generatePackageInfo(parsed, null,
                    PackageManager.GET_PERMISSIONS, 0, 0, null,
                    new PackageUserState());
            mPkgDigest = parsed.manifestDigest;
            as = PackageUtil.getAppSnippet(this, mPkgInfo.applicationInfo, sourceFile);
        }
        mInstallFlowAnalytics.setPackageInfoObtained();

        //set view
        setContentView(R.layout.install_start);
        mInstallConfirm = findViewById(R.id.install_confirm_panel);
        mInstallConfirm.setVisibility(View.INVISIBLE);
        PackageUtil.initSnippetForNewApp(this, as, R.id.app_snippet);

        mOriginatingUid = getOriginatingUid(intent);

        // Block the install attempt on the Unknown Sources setting if necessary.
        if (!requestFromUnknownSource) {
            initiateInstall();
            return;
        }

        // If the admin prohibits it, or we're running in a managed profile, just show error
        // and exit. Otherwise show an option to take the user to Settings to change the setting.
        final boolean isManagedProfile = mUserManager.isManagedProfile();
        if (!unknownSourcesAllowedByAdmin
                || (!unknownSourcesAllowedByUser && isManagedProfile)) {
            showDialogInner(DLG_ADMIN_RESTRICTS_UNKNOWN_SOURCES);
            mInstallFlowAnalytics.setFlowFinished(
                    InstallFlowAnalytics.RESULT_BLOCKED_BY_UNKNOWN_SOURCES_SETTING);
        } else if (!unknownSourcesAllowedByUser) {
            // Ask user to enable setting first
            showDialogInner(DLG_UNKNOWN_SOURCES);
            mInstallFlowAnalytics.setFlowFinished(
                    InstallFlowAnalytics.RESULT_BLOCKED_BY_UNKNOWN_SOURCES_SETTING);
        } else {
            initiateInstall();
        }
    }
```
这里我们主要先看一下PackageInstallerActivity的onCreate方法：

```
@Override
    protected void onCreate(Bundle icicle) {
        super.onCreate(icicle);

        mPm = getPackageManager();
        mInstaller = mPm.getPackageInstaller();
        mUserManager = (UserManager) getSystemService(Context.USER_SERVICE);

        final Intent intent = getIntent();
        if (PackageInstaller.ACTION_CONFIRM_PERMISSIONS.equals(intent.getAction())) {
            final int sessionId = intent.getIntExtra(PackageInstaller.EXTRA_SESSION_ID, -1);
            final PackageInstaller.SessionInfo info = mInstaller.getSessionInfo(sessionId);
            if (info == null || !info.sealed || info.resolvedBaseCodePath == null) {
                Log.w(TAG, "Session " + mSessionId + " in funky state; ignoring");
                finish();
                return;
            }

            mSessionId = sessionId;
            mPackageURI = Uri.fromFile(new File(info.resolvedBaseCodePath));
            mOriginatingURI = null;
            mReferrerURI = null;
        } else {
            mSessionId = -1;
            mPackageURI = intent.getData();
            mOriginatingURI = intent.getParcelableExtra(Intent.EXTRA_ORIGINATING_URI);
            mReferrerURI = intent.getParcelableExtra(Intent.EXTRA_REFERRER);
        }

        final boolean unknownSourcesAllowedByAdmin = isUnknownSourcesAllowedByAdmin();
        final boolean unknownSourcesAllowedByUser = isUnknownSourcesEnabled();

        boolean requestFromUnknownSource = isInstallRequestFromUnknownSource(intent);
        mInstallFlowAnalytics = new InstallFlowAnalytics();
        mInstallFlowAnalytics.setContext(this);
        mInstallFlowAnalytics.setStartTimestampMillis(SystemClock.elapsedRealtime());
        mInstallFlowAnalytics.setInstallsFromUnknownSourcesPermitted(unknownSourcesAllowedByAdmin
                && unknownSourcesAllowedByUser);
        mInstallFlowAnalytics.setInstallRequestFromUnknownSource(requestFromUnknownSource);
        mInstallFlowAnalytics.setVerifyAppsEnabled(isVerifyAppsEnabled());
        mInstallFlowAnalytics.setAppVerifierInstalled(isAppVerifierInstalled());
        mInstallFlowAnalytics.setPackageUri(mPackageURI.toString());

        if (DeviceUtils.isWear(this)) {
            showDialogInner(DLG_NOT_SUPPORTED_ON_WEAR);
            mInstallFlowAnalytics.setFlowFinished(
                    InstallFlowAnalytics.RESULT_NOT_ALLOWED_ON_WEAR);
            return;
        }

        final String scheme = mPackageURI.getScheme();
        if (scheme != null && !"file".equals(scheme) && !"package".equals(scheme)) {
            Log.w(TAG, "Unsupported scheme " + scheme);
            setPmResult(PackageManager.INSTALL_FAILED_INVALID_URI);
            mInstallFlowAnalytics.setFlowFinished(
                    InstallFlowAnalytics.RESULT_FAILED_UNSUPPORTED_SCHEME);
            finish();
            return;
        }

        final PackageUtil.AppSnippet as;
        if ("package".equals(mPackageURI.getScheme())) {
            mInstallFlowAnalytics.setFileUri(false);
            try {
                mPkgInfo = mPm.getPackageInfo(mPackageURI.getSchemeSpecificPart(),
                        PackageManager.GET_PERMISSIONS | PackageManager.GET_UNINSTALLED_PACKAGES);
            } catch (NameNotFoundException e) {
            }
            if (mPkgInfo == null) {
                Log.w(TAG, "Requested package " + mPackageURI.getScheme()
                        + " not available. Discontinuing installation");
                showDialogInner(DLG_PACKAGE_ERROR);
                setPmResult(PackageManager.INSTALL_FAILED_INVALID_APK);
                mInstallFlowAnalytics.setPackageInfoObtained();
                mInstallFlowAnalytics.setFlowFinished(
                        InstallFlowAnalytics.RESULT_FAILED_PACKAGE_MISSING);
                return;
            }
            as = new PackageUtil.AppSnippet(mPm.getApplicationLabel(mPkgInfo.applicationInfo),
                    mPm.getApplicationIcon(mPkgInfo.applicationInfo));
        } else {
            mInstallFlowAnalytics.setFileUri(true);
            final File sourceFile = new File(mPackageURI.getPath());
            PackageParser.Package parsed = PackageUtil.getPackageInfo(sourceFile);

            // Check for parse errors
            if (parsed == null) {
                Log.w(TAG, "Parse error when parsing manifest. Discontinuing installation");
                showDialogInner(DLG_PACKAGE_ERROR);
                setPmResult(PackageManager.INSTALL_FAILED_INVALID_APK);
                mInstallFlowAnalytics.setPackageInfoObtained();
                mInstallFlowAnalytics.setFlowFinished(
                        InstallFlowAnalytics.RESULT_FAILED_TO_GET_PACKAGE_INFO);
                return;
            }
            mPkgInfo = PackageParser.generatePackageInfo(parsed, null,
                    PackageManager.GET_PERMISSIONS, 0, 0, null,
                    new PackageUserState());
            mPkgDigest = parsed.manifestDigest;
            as = PackageUtil.getAppSnippet(this, mPkgInfo.applicationInfo, sourceFile);
        }
        mInstallFlowAnalytics.setPackageInfoObtained();

        //set view
        setContentView(R.layout.install_start);
        mInstallConfirm = findViewById(R.id.install_confirm_panel);
        mInstallConfirm.setVisibility(View.INVISIBLE);
        PackageUtil.initSnippetForNewApp(this, as, R.id.app_snippet);

        mOriginatingUid = getOriginatingUid(intent);

        // Block the install attempt on the Unknown Sources setting if necessary.
        if (!requestFromUnknownSource) {
            initiateInstall();
            return;
        }

        // If the admin prohibits it, or we're running in a managed profile, just show error
        // and exit. Otherwise show an option to take the user to Settings to change the setting.
        final boolean isManagedProfile = mUserManager.isManagedProfile();
        if (!unknownSourcesAllowedByAdmin
                || (!unknownSourcesAllowedByUser && isManagedProfile)) {
            showDialogInner(DLG_ADMIN_RESTRICTS_UNKNOWN_SOURCES);
            mInstallFlowAnalytics.setFlowFinished(
                    InstallFlowAnalytics.RESULT_BLOCKED_BY_UNKNOWN_SOURCES_SETTING);
        } else if (!unknownSourcesAllowedByUser) {
            // Ask user to enable setting first
            showDialogInner(DLG_UNKNOWN_SOURCES);
            mInstallFlowAnalytics.setFlowFinished(
                    InstallFlowAnalytics.RESULT_BLOCKED_BY_UNKNOWN_SOURCES_SETTING);
        } else {
            initiateInstall();
        }
    }
```
可以发现，在onCreate方法中，首先执行一些初始化操作，获取PackageManager和Installer、UserManager等对象，然后会根据当前Intent的信息最一些逻辑判断并弹出消息弹窗，我们可以看一下具体的消息弹窗类型：

```
private static final int DLG_BASE = 0;
    private static final int DLG_UNKNOWN_SOURCES = DLG_BASE + 1;
    private static final int DLG_PACKAGE_ERROR = DLG_BASE + 2;
    private static final int DLG_OUT_OF_SPACE = DLG_BASE + 3;
    private static final int DLG_INSTALL_ERROR = DLG_BASE + 4;
    private static final int DLG_ALLOW_SOURCE = DLG_BASE + 5;
    private static final int DLG_ADMIN_RESTRICTS_UNKNOWN_SOURCES = DLG_BASE + 6;
    private static final int DLG_NOT_SUPPORTED_ON_WEAR = DLG_BASE + 7;
```
可以发现当分析Intent对象的时候，如果可以得到这样几种结果：不知道apk的来源，package信息错误，存储空间不够，安装时报，来源正确，允许未知来源的apk文件，在wear上不支持等，这样根据不同的消息类型会弹出不同的消息弹窗：

```
@Override
    public Dialog onCreateDialog(int id, Bundle bundle) {
        switch (id) {
        case DLG_UNKNOWN_SOURCES:
            return new AlertDialog.Builder(this)
                    .setTitle(R.string.unknown_apps_dlg_title)
                    .setMessage(R.string.unknown_apps_dlg_text)
                    .setNegativeButton(R.string.cancel, new DialogInterface.OnClickListener() {
                        public void onClick(DialogInterface dialog, int which) {
                            Log.i(TAG, "Finishing off activity so that user can navigate to settings manually");
                            finish();
                        }})
                    .setPositiveButton(R.string.settings, new DialogInterface.OnClickListener() {
                        public void onClick(DialogInterface dialog, int which) {
                            Log.i(TAG, "Launching settings");
                            launchSecuritySettings();
                        }
                    })
                    .setOnCancelListener(this)
                    .create();
        case DLG_ADMIN_RESTRICTS_UNKNOWN_SOURCES:
            return new AlertDialog.Builder(this)
                    .setTitle(R.string.unknown_apps_dlg_title)
                    .setMessage(R.string.unknown_apps_admin_dlg_text)
                    .setPositiveButton(android.R.string.ok, new DialogInterface.OnClickListener() {
                        public void onClick(DialogInterface dialog, int which) {
                            finish();
                        }
                    })
                    .setOnCancelListener(this)
                    .create();
        case DLG_PACKAGE_ERROR :
            return new AlertDialog.Builder(this)
                    .setTitle(R.string.Parse_error_dlg_title)
                    .setMessage(R.string.Parse_error_dlg_text)
                    .setPositiveButton(R.string.ok, new DialogInterface.OnClickListener() {
                        public void onClick(DialogInterface dialog, int which) {
                            finish();
                        }
                    })
                    .setOnCancelListener(this)
                    .create();
        case DLG_OUT_OF_SPACE:
            // Guaranteed not to be null. will default to package name if not set by app
            CharSequence appTitle = mPm.getApplicationLabel(mPkgInfo.applicationInfo);
            String dlgText = getString(R.string.out_of_space_dlg_text,
                    appTitle.toString());
            return new AlertDialog.Builder(this)
                    .setTitle(R.string.out_of_space_dlg_title)
                    .setMessage(dlgText)
                    .setPositiveButton(R.string.manage_applications, new DialogInterface.OnClickListener() {
                        public void onClick(DialogInterface dialog, int which) {
                            //launch manage applications
                            Intent intent = new Intent("android.intent.action.MANAGE_PACKAGE_STORAGE");
                            intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                            startActivity(intent);
                            finish();
                        }
                    })
                    .setNegativeButton(R.string.cancel, new DialogInterface.OnClickListener() {
                        public void onClick(DialogInterface dialog, int which) {
                            Log.i(TAG, "Canceling installation");
                            finish();
                        }
                })
                  .setOnCancelListener(this)
                  .create();
        case DLG_INSTALL_ERROR :
            // Guaranteed not to be null. will default to package name if not set by app
            CharSequence appTitle1 = mPm.getApplicationLabel(mPkgInfo.applicationInfo);
            String dlgText1 = getString(R.string.install_failed_msg,
                    appTitle1.toString());
            return new AlertDialog.Builder(this)
                    .setTitle(R.string.install_failed)
                    .setNeutralButton(R.string.ok, new DialogInterface.OnClickListener() {
                        public void onClick(DialogInterface dialog, int which) {
                            finish();
                        }
                    })
                    .setMessage(dlgText1)
                    .setOnCancelListener(this)
                    .create();
        case DLG_ALLOW_SOURCE:
            CharSequence appTitle2 = mPm.getApplicationLabel(mSourceInfo);
            String dlgText2 = getString(R.string.allow_source_dlg_text,
                    appTitle2.toString());
            return new AlertDialog.Builder(this)
                    .setTitle(R.string.allow_source_dlg_title)
                    .setMessage(dlgText2)
                    .setNegativeButton(R.string.cancel, new DialogInterface.OnClickListener() {
                        public void onClick(DialogInterface dialog, int which) {
                            setResult(RESULT_CANCELED);
                            finish();
                        }})
                    .setPositiveButton(R.string.ok, new DialogInterface.OnClickListener() {
                        public void onClick(DialogInterface dialog, int which) {
                            SharedPreferences prefs = getSharedPreferences(PREFS_ALLOWED_SOURCES,
                                    Context.MODE_PRIVATE);
                            prefs.edit().putBoolean(mSourceInfo.packageName, true).apply();
                            startInstallConfirm();
                        }
                    })
                    .setOnCancelListener(this)
                    .create();
        case DLG_NOT_SUPPORTED_ON_WEAR:
            return new AlertDialog.Builder(this)
                    .setTitle(R.string.wear_not_allowed_dlg_title)
                    .setMessage(R.string.wear_not_allowed_dlg_text)
                    .setPositiveButton(R.string.ok, new DialogInterface.OnClickListener() {
                        public void onClick(DialogInterface dialog, int which) {
                            setResult(RESULT_OK);
                            finish();
                        }
                    })
                    .setOnCancelListener(this)
                    .create();
       }
       return null;
   }
```
消息弹窗的主要作用，用于提示用户当前安装apk文件的特性。都知道android系统在android apk文件之前会解析器manifest文件，这个操作也是早onCreate方法中执行的：

```
PackageParser.Package parsed = PackageUtil.getPackageInfo(sourceFile);
```
我们具体看一下getPackageInfo方法的实现：

```
public static PackageParser.Package getPackageInfo(File sourceFile) {
        final PackageParser parser = new PackageParser();
        try {
            PackageParser.Package pkg = parser.parseMonolithicPackage(sourceFile, 0);
            parser.collectManifestDigest(pkg);
            return pkg;
        } catch (PackageParserException e) {
            return null;
        }
    }
```
好吧，到了这里是不是代码变得很熟悉了？parseMonolithicPackage就是我们上一节分析的android系统解析manifest文件的过程，具体的可参考：http://blog.csdn.net/qq_23547831/article/details/51203482

而collectManifestDigest方法，我们这里简单的介绍一下，其主要是要争apk的签名是否正确。好吧通过这两部我们就把apk文件的manifest和签名信息都解析完成并保存在了Package中。


接着往下走，在所有的解析完成之后我们会在onCreate方法中执行initiateInstall();方法，刚方法的主要作用是初始化安装。

```
private void initiateInstall() {
        String pkgName = mPkgInfo.packageName;
        // Check if there is already a package on the device with this name
        // but it has been renamed to something else.
        String[] oldName = mPm.canonicalToCurrentPackageNames(new String[] { pkgName });
        if (oldName != null && oldName.length > 0 && oldName[0] != null) {
            pkgName = oldName[0];
            mPkgInfo.packageName = pkgName;
            mPkgInfo.applicationInfo.packageName = pkgName;
        }
        // Check if package is already installed. display confirmation dialog if replacing pkg
        try {
            // This is a little convoluted because we want to get all uninstalled
            // apps, but this may include apps with just data, and if it is just
            // data we still want to count it as "installed".
            mAppInfo = mPm.getApplicationInfo(pkgName,
                    PackageManager.GET_UNINSTALLED_PACKAGES);
            if ((mAppInfo.flags&ApplicationInfo.FLAG_INSTALLED) == 0) {
                mAppInfo = null;
            }
        } catch (NameNotFoundException e) {
            mAppInfo = null;
        }

        mInstallFlowAnalytics.setReplace(mAppInfo != null);
        mInstallFlowAnalytics.setSystemApp(
                (mAppInfo != null) && ((mAppInfo.flags & ApplicationInfo.FLAG_SYSTEM) != 0));

        // If we have a session id, we're invoked to verify the permissions for the given
        // package. Otherwise, we start the install process.
        if (mSessionId != -1) {
            startInstallConfirm();
        } else {
            startInstall();
        }
    }
```
好吧，这里面有调用了startInstallConfirm方法，然后我们看一下startInstallConfirm方法的实现:

```
private void startInstallConfirm() {
	...
	//初始化安装确认界面
	...
}
```
好吧，这个方法的实现比较简单，主要的实现逻辑就是现实该activity的用户界面，平时我们安装某一个应用的时候会弹出一个安装确认页面，还有一个确认和取消按钮，有印象么？其实就是在这里执行的界面初始化操作。

好吧，一般情况下在apk安装确认页面，我们会点击确认按钮执行安装逻辑吧？那么这里我们找一下确认按钮的点击事件：

```
public void onClick(View v) {
        if (v == mOk) {
            if (mOkCanInstall || mScrollView == null) {
                mInstallFlowAnalytics.setInstallButtonClicked();
                if (mSessionId != -1) {
                    mInstaller.setPermissionsResult(mSessionId, true);

                    // We're only confirming permissions, so we don't really know how the
                    // story ends; assume success.
                    mInstallFlowAnalytics.setFlowFinishedWithPackageManagerResult(
                            PackageManager.INSTALL_SUCCEEDED);
                    finish();
                } else {
                    startInstall();
                }
            } else {
                mScrollView.pageScroll(View.FOCUS_DOWN);
            }
        } else if (v == mCancel) {
            // Cancel and finish
            setResult(RESULT_CANCELED);
            if (mSessionId != -1) {
                mInstaller.setPermissionsResult(mSessionId, false);
            }
            mInstallFlowAnalytics.setFlowFinished(
                    InstallFlowAnalytics.RESULT_CANCELLED_BY_USER);
            finish();
        }
    }
```
很明显了，这里当我们点击确认按钮的时候会执行startInstall方法，也就是开始执行安装逻辑：

```
private void startInstall() {
        // Start subactivity to actually install the application
        Intent newIntent = new Intent();
        newIntent.putExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO,
                mPkgInfo.applicationInfo);
        newIntent.setData(mPackageURI);
        newIntent.setClass(this, InstallAppProgress.class);
        newIntent.putExtra(InstallAppProgress.EXTRA_MANIFEST_DIGEST, mPkgDigest);
        newIntent.putExtra(
                InstallAppProgress.EXTRA_INSTALL_FLOW_ANALYTICS, mInstallFlowAnalytics);
        String installerPackageName = getIntent().getStringExtra(
                Intent.EXTRA_INSTALLER_PACKAGE_NAME);
        if (mOriginatingURI != null) {
            newIntent.putExtra(Intent.EXTRA_ORIGINATING_URI, mOriginatingURI);
        }
        if (mReferrerURI != null) {
            newIntent.putExtra(Intent.EXTRA_REFERRER, mReferrerURI);
        }
        if (mOriginatingUid != VerificationParams.NO_UID) {
            newIntent.putExtra(Intent.EXTRA_ORIGINATING_UID, mOriginatingUid);
        }
        if (installerPackageName != null) {
            newIntent.putExtra(Intent.EXTRA_INSTALLER_PACKAGE_NAME,
                    installerPackageName);
        }
        if (getIntent().getBooleanExtra(Intent.EXTRA_RETURN_RESULT, false)) {
            newIntent.putExtra(Intent.EXTRA_RETURN_RESULT, true);
            newIntent.addFlags(Intent.FLAG_ACTIVITY_FORWARD_RESULT);
        }
        if(localLOGV) Log.i(TAG, "downloaded app uri="+mPackageURI);
        startActivity(newIntent);
        finish();
    }
```
可以发现，点击确认按钮之后我们调用启用了一个新的Activity-->InstallAppProgress，这个Activity主要用于执行apk的安装逻辑了。

```
@Override
    public void onCreate(Bundle icicle) {
        super.onCreate(icicle);
        Intent intent = getIntent();
        mAppInfo = intent.getParcelableExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO);
        mInstallFlowAnalytics = intent.getParcelableExtra(EXTRA_INSTALL_FLOW_ANALYTICS);
        mInstallFlowAnalytics.setContext(this);
        mPackageURI = intent.getData();

        final String scheme = mPackageURI.getScheme();
        if (scheme != null && !"file".equals(scheme) && !"package".equals(scheme)) {
            mInstallFlowAnalytics.setFlowFinished(
                    InstallFlowAnalytics.RESULT_FAILED_UNSUPPORTED_SCHEME);
            throw new IllegalArgumentException("unexpected scheme " + scheme);
        }

        mInstallThread = new HandlerThread("InstallThread");
        mInstallThread.start();
        mInstallHandler = new Handler(mInstallThread.getLooper());

        IntentFilter intentFilter = new IntentFilter();
        intentFilter.addAction(BROADCAST_ACTION);
        registerReceiver(
                mBroadcastReceiver, intentFilter, BROADCAST_SENDER_PERMISSION, null /*scheduler*/);

        initView();
    }
```
可以发现InstallAppProcess这个Activity的onCreate方法中主要初始化了一些成员变量，并调用initView方法，我们在iniTView方法中可以看到：

```
void initView() {
	...
	mInstallHandler.post(new Runnable() {
                @Override
                public void run() {
                    doPackageStage(pm, params);
                }
            });
	...
}
```
经过一些view的初始化操作之后调用了doPackageStage方法，该方法主要是通过调用PackageInstaller执行apk文件的安装，这里就不在详细的介绍了，在apk文件安装完成之后PackageInstaller会发送一个安装完成的广播，刚刚我们在onCreate方法中注册了一个广播接收器，其可以用来接收apk安装完成的广播：

```
private final BroadcastReceiver mBroadcastReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            final int statusCode = intent.getIntExtra(
                    PackageInstaller.EXTRA_STATUS, PackageInstaller.STATUS_FAILURE);
            if (statusCode == PackageInstaller.STATUS_PENDING_USER_ACTION) {
                context.startActivity((Intent)intent.getParcelableExtra(Intent.EXTRA_INTENT));
            } else {
                onPackageInstalled(statusCode);
            }
        }
    };
```
这样apk安装完成之后，这里的广播接收器会接收到广播并执行onPackageInstalled方法，执行后续的处理逻辑，那么我们来看一下onPackageInstalled方法的具体实现逻辑：

```
void onPackageInstalled(int statusCode) {
        Message msg = mHandler.obtainMessage(INSTALL_COMPLETE);
        msg.arg1 = statusCode;
        mHandler.sendMessage(msg);
    }
```
好吧，这里是发送Handler异步消息，我们来看一下异步消息的处理逻辑：

```
private Handler mHandler = new Handler() {
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case INSTALL_COMPLETE:
                    mInstallFlowAnalytics.setFlowFinishedWithPackageManagerResult(msg.arg1);
                    if (getIntent().getBooleanExtra(Intent.EXTRA_RETURN_RESULT, false)) {
                        Intent result = new Intent();
                        result.putExtra(Intent.EXTRA_INSTALL_RESULT, msg.arg1);
                        setResult(msg.arg1 == PackageInstaller.STATUS_SUCCESS
                                ? Activity.RESULT_OK : Activity.RESULT_FIRST_USER,
                                        result);
                        finish();
                        return;
                    }
                    // Update the status text
                    mProgressBar.setVisibility(View.INVISIBLE);
                    // Show the ok button
                    int centerTextLabel;
                    int centerExplanationLabel = -1;
                    LevelListDrawable centerTextDrawable =
                            (LevelListDrawable) getDrawable(R.drawable.ic_result_status);
                    if (msg.arg1 == PackageInstaller.STATUS_SUCCESS) {
                        mLaunchButton.setVisibility(View.VISIBLE);
                        centerTextDrawable.setLevel(0);
                        centerTextLabel = R.string.install_done;
                        // Enable or disable launch button
                        mLaunchIntent = getPackageManager().getLaunchIntentForPackage(
                                mAppInfo.packageName);
                        boolean enabled = false;
                        if(mLaunchIntent != null) {
                            List<ResolveInfo> list = getPackageManager().
                                    queryIntentActivities(mLaunchIntent, 0);
                            if (list != null && list.size() > 0) {
                                enabled = true;
                            }
                        }
                        if (enabled) {
                            mLaunchButton.setOnClickListener(InstallAppProgress.this);
                        } else {
                            mLaunchButton.setEnabled(false);
                        }
                    } else if (msg.arg1 == PackageInstaller.STATUS_FAILURE_STORAGE){
                        showDialogInner(DLG_OUT_OF_SPACE);
                        return;
                    } else {
                        // Generic error handling for all other error codes.
                        centerTextDrawable.setLevel(1);
                        centerExplanationLabel = getExplanationFromErrorCode(msg.arg1);
                        centerTextLabel = R.string.install_failed;
                        mLaunchButton.setVisibility(View.INVISIBLE);
                    }
                    if (centerTextDrawable != null) {
                    centerTextDrawable.setBounds(0, 0,
                            centerTextDrawable.getIntrinsicWidth(),
                            centerTextDrawable.getIntrinsicHeight());
                        mStatusTextView.setCompoundDrawablesRelative(centerTextDrawable, null,
                                null, null);
                    }
                    mStatusTextView.setText(centerTextLabel);
                    if (centerExplanationLabel != -1) {
                        mExplanationTextView.setText(centerExplanationLabel);
                        mExplanationTextView.setVisibility(View.VISIBLE);
                    } else {
                        mExplanationTextView.setVisibility(View.GONE);
                    }
                    mDoneButton.setOnClickListener(InstallAppProgress.this);
                    mOkPanel.setVisibility(View.VISIBLE);
                    break;
                default:
                    break;
            }
        }
    };
```
可以发现，当apk安装完成之后，我们会更新UI，显示完成和打开按钮，是不是和我们平时安装apk的逻辑对应上了？这时候我们可以看一下这两个按钮的点击事件。

```
public void onClick(View v) {
        if(v == mDoneButton) {
            if (mAppInfo.packageName != null) {
                Log.i(TAG, "Finished installing "+mAppInfo.packageName);
            }
            finish();
        } else if(v == mLaunchButton) {
            startActivity(mLaunchIntent);
            finish();
        }
    }
```
好吧，比较简单，点击完成按钮，直接finish掉这个activity，点击打开，则直接调用startActivity启动安装的应用，然后直接finish自身。

**总结：**

- 代码中执行intent.setDataAndType(Uri.parse("file://" + path),"application/vnd.android.package-archive");可以调起PackageInstallerActivity；

- PackageInstallerActivity主要用于执行解析apk文件，解析manifest，解析签名等操作；

- InstallAppProcess主要用于执行安装apk逻辑，用于初始化安装界面，用于初始化用户UI。并调用PackageInstaller执行安装逻辑；

- InstallAppProcess内注册有广播，当安装完成之后接收广播，更新UI。显示apk安装完成界面；

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

