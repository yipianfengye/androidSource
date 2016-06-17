上一篇文章中我们介绍了android系统的截屏事件，由于截屏事件是一种系统全局处理事件，所以事件的处理逻辑不是在App中执行，而是在PhoneWindowManager中执行。而本文我们现在主要讲解android系统中HOME按键的事件处理，和截屏事件类似，这里的HOME按键也是系统级别的按键事件监听，所以其处理事件的逻辑也应该和截屏事件处理流程类似，从上一篇文章的分析过冲中我们不难发现，系统级别的按键处理逻辑其实都是在PhoneWindowManager中，所以HOME按键的处理逻辑也是在PhoneWindowManager的dispatchUnhandledKey方法中执行，那么我们就从dispatchUnhandleKey方法开始分析HOME按键的处理流程。

好吧我们看一下PhoneWindowManager的dispatchUnhandleKey方法的实现：

```
@Override
    public KeyEvent dispatchUnhandledKey(WindowState win, KeyEvent event, int policyFlags) {
        ...
        KeyEvent fallbackEvent = null;
        if ((event.getFlags() & KeyEvent.FLAG_FALLBACK) == 0) {
            final KeyCharacterMap kcm = event.getKeyCharacterMap();
            final int keyCode = event.getKeyCode();
            final int metaState = event.getMetaState();
            final boolean initialDown = event.getAction() == KeyEvent.ACTION_DOWN
                    && event.getRepeatCount() == 0;

            // Check for fallback actions specified by the key character map.
            final FallbackAction fallbackAction;
            if (initialDown) {
                fallbackAction = kcm.getFallbackAction(keyCode, metaState);
            } else {
                fallbackAction = mFallbackActions.get(keyCode);
            }

            if (fallbackAction != null) {
                if (DEBUG_INPUT) {
                    Slog.d(TAG, "Fallback: keyCode=" + fallbackAction.keyCode
                            + " metaState=" + Integer.toHexString(fallbackAction.metaState));
                }

                final int flags = event.getFlags() | KeyEvent.FLAG_FALLBACK;
                fallbackEvent = KeyEvent.obtain(
                        event.getDownTime(), event.getEventTime(),
                        event.getAction(), fallbackAction.keyCode,
                        event.getRepeatCount(), fallbackAction.metaState,
                        event.getDeviceId(), event.getScanCode(),
                        flags, event.getSource(), null);

                if (!interceptFallback(win, fallbackEvent, policyFlags)) {
                    fallbackEvent.recycle();
                    fallbackEvent = null;
                }

                if (initialDown) {
                    mFallbackActions.put(keyCode, fallbackAction);
                } else if (event.getAction() == KeyEvent.ACTION_UP) {
                    mFallbackActions.remove(keyCode);
                    fallbackAction.recycle();
                }
            }
        }

        if (DEBUG_INPUT) {
            if (fallbackEvent == null) {
                Slog.d(TAG, "No fallback.");
            } else {
                Slog.d(TAG, "Performing fallback: " + fallbackEvent);
            }
        }
        return fallbackEvent;
    }
```
通过查看源码，我们重点看一下dispatchUnhandledKey方法中调用的interceptFallback方法，关于HOME按键的处理逻辑也是在这个方法体中的，所以继续看一下interceptFallback方法的实现：

```
private boolean interceptFallback(WindowState win, KeyEvent fallbackEvent, int policyFlags) {
        int actions = interceptKeyBeforeQueueing(fallbackEvent, policyFlags);
        if ((actions & ACTION_PASS_TO_USER) != 0) {
            long delayMillis = interceptKeyBeforeDispatching(
                    win, fallbackEvent, policyFlags);
            if (delayMillis == 0) {
                return true;
            }
        }
        return false;
    }
```
通过分析源码我们知道关于HOME按键的处理逻辑主要是在interceptKeyBeforeDispatching方法的实现的，既然这样，我们看一下interceptKeyBeforeDispatching方法的实现：

```
@Override
    public long interceptKeyBeforeDispatching(WindowState win, KeyEvent event, int policyFlags) {
        ...
        // First we always handle the home key here, so applications
        // can never break it, although if keyguard is on, we do let
        // it handle it, because that gives us the correct 5 second
        // timeout.
        if (keyCode == KeyEvent.KEYCODE_HOME) {

            // If we have released the home key, and didn't do anything else
            // while it was pressed, then it is time to go home!
            if (!down) {
                cancelPreloadRecentApps();

                mHomePressed = false;
                if (mHomeConsumed) {
                    mHomeConsumed = false;
                    return -1;
                }

                if (canceled) {
                    Log.i(TAG, "Ignoring HOME; event canceled.");
                    return -1;
                }

                // If an incoming call is ringing, HOME is totally disabled.
                // (The user is already on the InCallUI at this point,
                // and his ONLY options are to answer or reject the call.)
                TelecomManager telecomManager = getTelecommService();
                if (telecomManager != null && telecomManager.isRinging()) {
                    Log.i(TAG, "Ignoring HOME; there's a ringing incoming call.");
                    return -1;
                }

                // Delay handling home if a double-tap is possible.
                if (mDoubleTapOnHomeBehavior != DOUBLE_TAP_HOME_NOTHING) {
                    mHandler.removeCallbacks(mHomeDoubleTapTimeoutRunnable); // just in case
                    mHomeDoubleTapPending = true;
                    mHandler.postDelayed(mHomeDoubleTapTimeoutRunnable,
                            ViewConfiguration.getDoubleTapTimeout());
                    return -1;
                }

                handleShortPressOnHome();
                return -1;
            }

            // If a system window has focus, then it doesn't make sense
            // right now to interact with applications.
            WindowManager.LayoutParams attrs = win != null ? win.getAttrs() : null;
            if (attrs != null) {
                final int type = attrs.type;
                if (type == WindowManager.LayoutParams.TYPE_KEYGUARD_SCRIM
                        || type == WindowManager.LayoutParams.TYPE_KEYGUARD_DIALOG
                        || (attrs.privateFlags & PRIVATE_FLAG_KEYGUARD) != 0) {
                    // the "app" is keyguard, so give it the key
                    return 0;
                }
                final int typeCount = WINDOW_TYPES_WHERE_HOME_DOESNT_WORK.length;
                for (int i=0; i<typeCount; i++) {
                    if (type == WINDOW_TYPES_WHERE_HOME_DOESNT_WORK[i]) {
                        // don't do anything, but also don't pass it to the app
                        return -1;
                    }
                }
            }

            // Remember that home is pressed and handle special actions.
            if (repeatCount == 0) {
                mHomePressed = true;
                if (mHomeDoubleTapPending) {
                    mHomeDoubleTapPending = false;
                    mHandler.removeCallbacks(mHomeDoubleTapTimeoutRunnable);
                    handleDoubleTapOnHome();
                } else if (mLongPressOnHomeBehavior == LONG_PRESS_HOME_RECENT_SYSTEM_UI
                        || mDoubleTapOnHomeBehavior == DOUBLE_TAP_HOME_RECENT_SYSTEM_UI) {
                    preloadRecentApps();
                }
            } else if ((event.getFlags() & KeyEvent.FLAG_LONG_PRESS) != 0) {
                if (!keyguardOn) {
                    handleLongPressOnHome(event.getDeviceId());
                }
            }
            return -1;
        }

        // Let the application handle the key.
        return 0;
    }
```
这里我们主要看一下对android系统HOME按键的处理逻辑，通过分析源码我们知道HOME按键进入launcher界面的主要逻辑是在handleShortPressOnHome();方法中执行的，所以我们继续看一下handleShortPressOnHome方法的实现。

```
private void handleShortPressOnHome() {
        // Turn on the connected TV and switch HDMI input if we're a HDMI playback device.
        getHdmiControl().turnOnTv();

        // If there's a dream running then use home to escape the dream
        // but don't actually go home.
        if (mDreamManagerInternal != null && mDreamManagerInternal.isDreaming()) {
            mDreamManagerInternal.stopDream(false /*immediate*/);
            return;
        }

        // Go home!
        launchHomeFromHotKey();
    }
```
可以看到在handleShortPressOnHome方法中调用了launchHomeFromHotKey方法，该方法的注释用于go home，所以继续看一下该方法的实现：

```
void launchHomeFromHotKey() {
        launchHomeFromHotKey(true /* awakenFromDreams */, true /*respectKeyguard*/);
    }
```
可以看到在launchHomeFromHotKey方法中我们又调用了launchHomeFromHotkey的重构方法，这样我们看一下这个重构方法的实现。

```
void launchHomeFromHotKey(final boolean awakenFromDreams, final boolean respectKeyguard) {
        if (respectKeyguard) {
            if (isKeyguardShowingAndNotOccluded()) {
                // don't launch home if keyguard showing
                return;
            }

            if (!mHideLockScreen && mKeyguardDelegate.isInputRestricted()) {
                // when in keyguard restricted mode, must first verify unlock
                // before launching home
                mKeyguardDelegate.verifyUnlock(new OnKeyguardExitResult() {
                    @Override
                    public void onKeyguardExitResult(boolean success) {
                        if (success) {
                            try {
                                ActivityManagerNative.getDefault().stopAppSwitches();
                            } catch (RemoteException e) {
                            }
                            sendCloseSystemWindows(SYSTEM_DIALOG_REASON_HOME_KEY);
                            startDockOrHome(true /*fromHomeKey*/, awakenFromDreams);
                        }
                    }
                });
                return;
            }
        }

        // no keyguard stuff to worry about, just launch home!
        try {
            ActivityManagerNative.getDefault().stopAppSwitches();
        } catch (RemoteException e) {
        }
        if (mRecentsVisible) {
            // Hide Recents and notify it to launch Home
            if (awakenFromDreams) {
                awakenDreams();
            }
            sendCloseSystemWindows(SYSTEM_DIALOG_REASON_HOME_KEY);
            hideRecentApps(false, true);
        } else {
            // Otherwise, just launch Home
            sendCloseSystemWindows(SYSTEM_DIALOG_REASON_HOME_KEY);
            startDockOrHome(true /*fromHomeKey*/, awakenFromDreams);
        }
    }
```
可以发现在方法中我们首先调用了ActivityManagerNative.getDefault().stopAppSwitches();该方法主要用于暂停后台的打开Activity的操作，避免打扰用户的操作。比如这时候我们在后台打开一个新的App，那么这时候由于要回到home页面，所以需要先延时打开。方法执行这个方法之后然后执行了sendCloseSystemWindows方法，该方法主要实现了对当前系统App页面的关闭操作，下面我们先看一下ActivityManagerNative.getDefault().stopAppSwitches();方法的实现，这里的ActivityManagerNative.getDefault我们在前面已经多次说过了其是一个Binder对象，是应用进程Binder客户端用于与ActivityManagerService之间通讯，所以这里最终调用的是ActivityManagerService的stopAppsSwitches方法，这样我们就继续看一下ActivityManagerService的stopAppsSwitches方法的实现。

```
@Override
    public void stopAppSwitches() {
        if (checkCallingPermission(android.Manifest.permission.STOP_APP_SWITCHES)
                != PackageManager.PERMISSION_GRANTED) {
            throw new SecurityException("Requires permission "
                    + android.Manifest.permission.STOP_APP_SWITCHES);
        }

        synchronized(this) {
            mAppSwitchesAllowedTime = SystemClock.uptimeMillis()
                    + APP_SWITCH_DELAY_TIME;
            mDidAppSwitch = false;
            mHandler.removeMessages(DO_PENDING_ACTIVITY_LAUNCHES_MSG);
            Message msg = mHandler.obtainMessage(DO_PENDING_ACTIVITY_LAUNCHES_MSG);
            mHandler.sendMessageDelayed(msg, APP_SWITCH_DELAY_TIME);
        }
    }
```
可以发现这里主要是发送了一个异步消息，并且msg.what为DO_PENDING_ACTIVITY_LAUNCHES_MSG，即跳转Activity，然后我们继续我们看一下mHandler的handleMessage方法当msg.what为DO_PENDING_ACTIVITY_LAUNCHES_MSG时的操作。而且我们可以发现这里的异步消息是一个延时的异步消息，延时的时间为APP_SWITCH_DELAY_TIME，我们可以看一下该变量的定义：

```
// Amount of time after a call to stopAppSwitches() during which we will
    // prevent further untrusted switches from happening.
    static final long APP_SWITCH_DELAY_TIME = 5*1000;
```
然后我们可以看一下mHander的handleMessage方法的具体实现：

```
case DO_PENDING_ACTIVITY_LAUNCHES_MSG: {
                synchronized (ActivityManagerService.this) {
                    mStackSupervisor.doPendingActivityLaunchesLocked(true);
                }
            } break;
```
可以发现这里直接调用了mStackSupervisor.doPendingActivityLaunchesLocked方法，好吧，继续看一下doPendingActivityLaunchesLocked方法的实现。

```
final void doPendingActivityLaunchesLocked(boolean doResume) {
        while (!mPendingActivityLaunches.isEmpty()) {
            PendingActivityLaunch pal = mPendingActivityLaunches.remove(0);
            startActivityUncheckedLocked(pal.r, pal.sourceRecord, null, null, pal.startFlags,
                    doResume && mPendingActivityLaunches.isEmpty(), null, null);
        }
    }
```
可以发现这里就是调用了startActivity的操作了，看过Activity启动流程的同学应该知道：<a href="http://blog.csdn.net/qq_23547831/article/details/51224992">android源码解析之（十四）-->Activity启动流程</a> 这里就是开始启动Activity了，所以当我们按下HOME按键的时候，后台的startActivity都会延时5秒钟执行...

然后回到我们的launchHomeFromHotKey方法，看一下launchHomeFromHotKey方法的实现。

```
void sendCloseSystemWindows(String reason) {
        PhoneWindow.sendCloseSystemWindows(mContext, reason);
    }
```
可以发现这里调用了PhoneWindow的静态方法sendCloseSystemWindow,继续看一下该方法的实现逻辑。

```
public static void sendCloseSystemWindows(Context context, String reason) {
        if (ActivityManagerNative.isSystemReady()) {
            try {
                ActivityManagerNative.getDefault().closeSystemDialogs(reason);
            } catch (RemoteException e) {
            }
        }
    }
```
看到这里，很明显了又是调用了Binder的进程间通讯，最终ActivityManagerService的closeSystemDialogs方法会被执行，所以我们继续看一下ActivityManagerService的closeSystemDialogs方法的实现。

```
@Override
    public void closeSystemDialogs(String reason) {
        enforceNotIsolatedCaller("closeSystemDialogs");

        final int pid = Binder.getCallingPid();
        final int uid = Binder.getCallingUid();
        final long origId = Binder.clearCallingIdentity();
        try {
            synchronized (this) {
                // Only allow this from foreground processes, so that background
                // applications can't abuse it to prevent system UI from being shown.
                if (uid >= Process.FIRST_APPLICATION_UID) {
                    ProcessRecord proc;
                    synchronized (mPidsSelfLocked) {
                        proc = mPidsSelfLocked.get(pid);
                    }
                    if (proc.curRawAdj > ProcessList.PERCEPTIBLE_APP_ADJ) {
                        Slog.w(TAG, "Ignoring closeSystemDialogs " + reason
                                + " from background process " + proc);
                        return;
                    }
                }
                closeSystemDialogsLocked(reason);
            }
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
```
可以发现其实在方法体中将关闭窗口的逻辑下发到了closeSystemDialogsLocked中，所以我们继续看一下closeSystemDialogsLocked方法的实现。

```
void closeSystemDialogsLocked(String reason) {
        Intent intent = new Intent(Intent.ACTION_CLOSE_SYSTEM_DIALOGS);
        intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                | Intent.FLAG_RECEIVER_FOREGROUND);
        if (reason != null) {
            intent.putExtra("reason", reason);
        }
        mWindowManager.closeSystemDialogs(reason);

        mStackSupervisor.closeSystemDialogsLocked();

        broadcastIntentLocked(null, null, intent, null, null, 0, null, null, null,
                AppOpsManager.OP_NONE, null, false, false,
                -1, Process.SYSTEM_UID, UserHandle.USER_ALL);
    }
```
可以发现在方法体中首先调用了mWindowManager.closeSystemDialogs方法，该方法就是关闭当前页面中存在的系统窗口，比如输入法，壁纸等：

```
@Override
    public void closeSystemDialogs(String reason) {
        synchronized(mWindowMap) {
            final int numDisplays = mDisplayContents.size();
            for (int displayNdx = 0; displayNdx < numDisplays; ++displayNdx) {
                final WindowList windows = mDisplayContents.valueAt(displayNdx).getWindowList();
                final int numWindows = windows.size();
                for (int winNdx = 0; winNdx < numWindows; ++winNdx) {
                    final WindowState w = windows.get(winNdx);
                    if (w.mHasSurface) {
                        try {
                            w.mClient.closeSystemDialogs(reason);
                        } catch (RemoteException e) {
                        }
                    }
                }
            }
        }
    }
```
讲过这样一层操作之后，我们就关闭了当前中存在的系统窗口。然后还是回到我们的launchHomeFromHotKey方法，我们发现在方法体的最后我们调用了startDockOrHome方法，这个方法就是实际的跳转HOME页面的方法了，我们可以具体看一下该方法的实现。

```
void startDockOrHome(boolean fromHomeKey, boolean awakenFromDreams) {
        if (awakenFromDreams) {
            awakenDreams();
        }

        Intent dock = createHomeDockIntent();
        if (dock != null) {
            try {
                if (fromHomeKey) {
                    dock.putExtra(WindowManagerPolicy.EXTRA_FROM_HOME_KEY, fromHomeKey);
                }
                startActivityAsUser(dock, UserHandle.CURRENT);
                return;
            } catch (ActivityNotFoundException e) {
            }
        }

        Intent intent;

        if (fromHomeKey) {
            intent = new Intent(mHomeIntent);
            intent.putExtra(WindowManagerPolicy.EXTRA_FROM_HOME_KEY, fromHomeKey);
        } else {
            intent = mHomeIntent;
        }

        startActivityAsUser(intent, UserHandle.CURRENT);
    }
```
可以发现我们在方法体中调用了createHomeDockIntent，这个方法的作用就是创建到达HOME页面的Intent对象，然后我们调用了startActivityAsUser方法，这样经过一系列的调用之后就调起了home页面的Activity，所以这时候系统就返回到了HOME页面。


总结：

- 系统也是在PhoneWindowManager中监听HOME按键的点击并进行处理；

- 系统监听到HOME按键之后会首先关闭相应的系统弹窗；

- 通过创建Intent对象，并调用startActivity方法使系统跳转到HOME页面；

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
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51465071">android源码解析（二十五）-->onLowMemory执行流程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51474288">android源码解析（二十六）-->截屏事件流程</a>
