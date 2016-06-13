前面两篇文章，我们分析了Activity的布局文件加载、绘制流程，算是对整个Android系统中界面的显示流程有了一个大概的了解，其实Android系统中所有的显示控件（注意这里是控件，而不是组件）的加载绘制流程都是类似的，包括：Dialog的加载绘制流程，PopupWindow的加载绘制流程，Toast的显示原理等，上一篇文章中，我说在介绍了Activity界面的加载绘制流程之后，就会分析一下剩余几个控件的显示控制流程，这里我打算先分析一下Dialog的加载绘制流程。

可能有的同学问这里为什么没有Fragment？其实严格意义上来说Fragment并不是一个显示控件，而只是一个显示组件。为什么这么说呢？其实像我们的Activity，Dialog，PopupWindow以及Toast类的内部都管理维护着一个Window对象，这个Window对象不但是一个View组件的集合管理对象，它也实现了组件的加载与绘制流程，而我们的Fragment组件如果看过源码的话，严格意义上来说，只是一个View组件的集合并通过控制变量实现了其特定的生命周期，但是其由于并没有维护Window类型的成员变量，所以其不具备组件的加载与绘制功能，因此其不能单独的被绘制出来，这也是我把它称之为组件而不是控件的原因。（在分析完这几个控件的加载绘制流程之后，有时间的话，也会分析一下Fragment的相关源码）

好吧，开始我们今天关于Dialog的讲解，相信大家在平时的开发过程中经常会使用到Dialog弹窗，使用Dialog可以在Activity弹出弹窗，确认消息等。为了更好的分析Dialog的源码，我们这里暂时写一个简单的demo，看一下Dialog的使用实例。

```
title.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {

                AlertDialog.Builder builder = new AlertDialog.Builder(MainActivity.this);
                builder.setIcon(R.mipmap.ic_launcher);
                builder.setMessage("this is the content view!!!");
                builder.setTitle("this is the title view!!!");
                builder.setView(R.layout.activity_second);
                builder.setPositiveButton("知道了", new DialogInterface.OnClickListener() {
                    @Override
                    public void onClick(DialogInterface dialog, int which) {
                        alertDialog.dismiss();
                    }
                });
                alertDialog = builder.create();
                alertDialog.show();
            }
        });
```
我们在Activity中获取一个textView组件，并监听TextView的点击事件，并在点击事件中，初始化一个AlertDialog弹窗，并执行AlertDialog的show方法展示弹窗，在弹窗中定义一个按钮，并监听弹窗按钮的点击事件，若用户点击了弹窗的按钮，则执行AlertDialog的dismiss方法，取消展示AlertDialog。好吧，我们来看一下这个弹窗弹出的展示结果：
![这里写图片描述](http://img.blog.csdn.net/20160501105319191)
可以看到我们定义的icon，title，message和button都已经显示出来了，这时候我们点击弹窗按钮知道了，这时候弹窗就会消失了。

一般我们使用Dialog的大概流程都是这样的，可能定制Dialog的时候有一些定制化的操作，但是基本操作流程还是这样的。

那么我们先来看一下AlertDialog.Builder的构造方法，这里的Builder是AlertDialog的内部类，用于封装AlertDialog的构造过程，看一下Builder的构造方法：

```
public Builder(Context context) {
            this(context, resolveDialogTheme(context, 0));
        }
```
好吧，这里调用的是Builder的重载构造方法：

```
public Builder(Context context, int themeResId) {
            P = new AlertController.AlertParams(new ContextThemeWrapper(
                    context, resolveDialogTheme(context, themeResId)));
        }
```
那么这里的P是AlertDialog.Builder中的一个AlertController.AlertParams类型的成员变量，可见在这里执行了P的初始化操作。

```
public AlertParams(Context context) {
            mContext = context;
            mCancelable = true;
            mInflater = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        }
```
可以看到这里主要执行了AlertController.AlertParams的初始化操作，初始化了一些成员变量。这样执行了一系列操作之后我们的代码：

```
AlertDialog.Builder builder = new AlertDialog.Builder(MainActivity.this);
```
就已经执行完成了，然后我们调用了builder.setIcon方法，这里看一下setIcon方法的具体实现：

```
public Builder setIcon(@DrawableRes int iconId) {
            P.mIconId = iconId;
            return this;
        }
```
可以看到AlertDialog的Builder的setIcon方法，这里执行的就是给类型为AlertController.AlertParams的P的mIconId赋值为传递的iconId，并且这个方法返回的类型就是Builder。

然后我们调用了builder.setMessage方法，可以看一下builder.setMessage方法的具体实现：

```
public Builder setMessage(CharSequence message) {
            P.mMessage = message;
            return this;
        }
```
好吧，这里跟setIcon方法的实现逻辑类似，都是给成员变量的mMessage赋值为我们传递的Message值，且和setIcon方法类似的，这个方法返回值也是Builder。

再看一下builder.setTitle方法：

```
public Builder setTitle(CharSequence title) {
            P.mTitle = title;
            return this;
        }
```
可以发现builder的setIcon、setMessage、setTitle等方法都是给Builder的成员变量P的icon，message，title赋值。

然后我们看一下builder.setView方法：

```
public Builder setView(int layoutResId) {
            P.mView = null;
            P.mViewLayoutResId = layoutResId;
            P.mViewSpacingSpecified = false;
            return this;
        }
```
可以发现这里的setView和setIcon，setMessage，setTitle等方法都是类似的，都是将我们传递的数据值赋值给Builder的成员变量P。

然后我们调用了builder.setPositiveButton方法：

```
builder.setPositiveButton("知道了", new DialogInterface.OnClickListener() {
                    @Override
                    public void onClick(DialogInterface dialog, int which) {
                        alertDialog.dismiss();
                    }
                });
```
好吧，这里我们看一下builder的setPositiveButton的源码：

```
public Builder setPositiveButton(CharSequence text, final OnClickListener listener) {
            P.mPositiveButtonText = text;
            P.mPositiveButtonListener = listener;
            return this;
        }
```
好吧，可以发现跟上面几个方法还是类似的，都是为Builder的成员变量P的相应成员变量赋值。。。

上面的几行代码我们都是调用的builder.setXXX等方法，主要就是为Builder的成员变量P的相应成员变量值赋值。并且setXX方法返回值都是Builder类型的，因此我们可以通过消息琏的方式连续执行：

```
builder.setIcon().setMessage().setTitle().setView().setPositiveButton()...
```
这样代码显得比较简洁，set方法的执行顺序是没有固定模式的，这里多说一下，这种编程方式很优秀，平时我们在设计构造类工具类的时候也可以参考这种模式，构造类有不同的功能或者特性，并且都不是必须的，我们可以通过set方法设置不同的特性值并返回构造类本身。

然后我们调用了builder.create方法，并且这个方法返回了AlertDialog。

```
public AlertDialog create() {
            // Context has already been wrapped with the appropriate theme.
            final AlertDialog dialog = new AlertDialog(P.mContext, 0, false);
            P.apply(dialog.mAlert);
            dialog.setCancelable(P.mCancelable);
            if (P.mCancelable) {
                dialog.setCanceledOnTouchOutside(true);
            }
            dialog.setOnCancelListener(P.mOnCancelListener);
            dialog.setOnDismissListener(P.mOnDismissListener);
            if (P.mOnKeyListener != null) {
                dialog.setOnKeyListener(P.mOnKeyListener);
            }
            return dialog;
        }
```
可以看到这里首先构造了一个AlertDialog，我们可以看一下这个构造方法的具体实现：

```
AlertDialog(Context context, @StyleRes int themeResId, boolean createContextThemeWrapper) {
        super(context, createContextThemeWrapper ? resolveDialogTheme(context, themeResId) : 0,
                createContextThemeWrapper);

        mWindow.alwaysReadCloseOnTouchAttr();
        mAlert = new AlertController(getContext(), this, getWindow());
    }
```
可以看到这里首先调用了super的构造方法，而我们的AlertDialog继承于Dialog，所以这里执行的就是Dialog的构造方法，好吧，继续看一下Dialog的构造方法：

```
Dialog(@NonNull Context context, @StyleRes int themeResId, boolean createContextThemeWrapper) {
        if (createContextThemeWrapper) {
            if (themeResId == 0) {
                final TypedValue outValue = new TypedValue();
                context.getTheme().resolveAttribute(R.attr.dialogTheme, outValue, true);
                themeResId = outValue.resourceId;
            }
            mContext = new ContextThemeWrapper(context, themeResId);
        } else {
            mContext = context;
        }

        mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);

        final Window w = new PhoneWindow(mContext);
        mWindow = w;
        w.setCallback(this);
        w.setOnWindowDismissedCallback(this);
        w.setWindowManager(mWindowManager, null, null);
        w.setGravity(Gravity.CENTER);

        mListenersHandler = new ListenersHandler(this);
    }
```
可以发现在Dialog的构造方法中直接直接构造了一个PhoneWindow，并赋值给Dialog的成员变量mWindow，从这里可以看出其实Dialog和Activity的显示逻辑都是类似的，都是通过对应的Window变量来实现窗口的加载与显示的。然后我们执行了一些Window对象的初始化操作，比如设置回调函数为本身，然后调用了Window类的setWindowManager方法，并传入了WindowManager，可以发现这里的WindowManager对象是通过方法：

```
mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
```
获取的，而我们的context传入的是Activity对象，所以这里的WindowManager对象其实和Activity获取的WindowManager对象是一致的。然后我们看一下window类的setWindowManager方法：

```
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
            boolean hardwareAccelerated) {
        mAppToken = appToken;
        mAppName = appName;
        mHardwareAccelerated = hardwareAccelerated
                || SystemProperties.getBoolean(PROPERTY_HARDWARE_UI, false);
        if (wm == null) {
            wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
        }
        mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
    }
```
可以看到跟Activity的Window对象的windowManager的获取方式是相同的，都是通过new的方式创建一个新的WindowManagerImpl对象。好吧，继续回到我们的AlertDialog的构造方法中，在构造方法中，我们除了调用Dialog的构造方法之外还执行了：

```
mAlert = new AlertController(getContext(), this, getWindow());
```
相当于初始化了AlertDiaog的成员变量mAlert。

继续回到我们的AlertDialog.Builder.create方法，在创建了一个AlertDialog之后，又执行了P.apply(dialog.mAlert)；
我们知道这里的P是一个AlertController.AlertParams的变量，而dialog.mAlert是我们刚刚创建的AlertDialog中的一个AlertController类型的变量，我们来看一下apply方法的具体实现：

```
ublic void apply(AlertController dialog) {
            if (mCustomTitleView != null) {
                dialog.setCustomTitle(mCustomTitleView);
            } else {
                if (mTitle != null) {
                    dialog.setTitle(mTitle);
                }
                if (mIcon != null) {
                    dialog.setIcon(mIcon);
                }
                if (mIconId != 0) {
                    dialog.setIcon(mIconId);
                }
                if (mIconAttrId != 0) {
                    dialog.setIcon(dialog.getIconAttributeResId(mIconAttrId));
                }
            }
            if (mMessage != null) {
                dialog.setMessage(mMessage);
            }
            if (mPositiveButtonText != null) {
                dialog.setButton(DialogInterface.BUTTON_POSITIVE, mPositiveButtonText,
                        mPositiveButtonListener, null);
            }
            if (mNegativeButtonText != null) {
                dialog.setButton(DialogInterface.BUTTON_NEGATIVE, mNegativeButtonText,
                        mNegativeButtonListener, null);
            }
            if (mNeutralButtonText != null) {
                dialog.setButton(DialogInterface.BUTTON_NEUTRAL, mNeutralButtonText,
                        mNeutralButtonListener, null);
            }
            if (mForceInverseBackground) {
                dialog.setInverseBackgroundForced(true);
            }
            // For a list, the client can either supply an array of items or an
            // adapter or a cursor
            if ((mItems != null) || (mCursor != null) || (mAdapter != null)) {
                createListView(dialog);
            }
            if (mView != null) {
                if (mViewSpacingSpecified) {
                    dialog.setView(mView, mViewSpacingLeft, mViewSpacingTop, mViewSpacingRight,
                            mViewSpacingBottom);
                } else {
                    dialog.setView(mView);
                }
            } else if (mViewLayoutResId != 0) {
                dialog.setView(mViewLayoutResId);
            }
        }
```
看到了么？就是我们在初始化AlertDialog.Builder的时候设置的icon、title、message赋值给了AlertController.AlertParams，这里就是将我们初始化时候设置的属性值赋值给我们创建的Dialog对象的mAlert成员变量。。。。

继续我们的AlertDialog.Builder.create方法，在执行了AlertController.AlertParams.apply方法之后又调用了：

```
dialog.setCancelable(P.mCancelable);
```
可以发现这个也是AertController.AlertParams的一个成员变量，我们在初始化AlertDialog.Builder的时候也可以通过设置builder.setCancelable赋值，由于该属性为成员变量，所以默认值为false，而我们并没有通过builder.setCancelable修改这个属性值，所以这里设置的dialog的cancelable的值为false。然后我们的create方法有设置了dialog的cancelListener和dismissListener并返回了我们创建的Dialog对象。这样我们就获取到了我们的Dialog对象，然后我们调用了dialog的show方法用于显示dialog，好吧，这里我们看一下show方法的具体实现：

```
public void show() {
        if (mShowing) {
            if (mDecor != null) {
                if (mWindow.hasFeature(Window.FEATURE_ACTION_BAR)) {
                    mWindow.invalidatePanelMenu(Window.FEATURE_ACTION_BAR);
                }
                mDecor.setVisibility(View.VISIBLE);
            }
            return;
        }

        mCanceled = false;
        
        if (!mCreated) {
            dispatchOnCreate(null);
        }

        onStart();
        mDecor = mWindow.getDecorView();

        if (mActionBar == null && mWindow.hasFeature(Window.FEATURE_ACTION_BAR)) {
            final ApplicationInfo info = mContext.getApplicationInfo();
            mWindow.setDefaultIcon(info.icon);
            mWindow.setDefaultLogo(info.logo);
            mActionBar = new WindowDecorActionBar(this);
        }

        WindowManager.LayoutParams l = mWindow.getAttributes();
        if ((l.softInputMode
                & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION) == 0) {
            WindowManager.LayoutParams nl = new WindowManager.LayoutParams();
            nl.copyFrom(l);
            nl.softInputMode |=
                    WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION;
            l = nl;
        }

        try {
            mWindowManager.addView(mDecor, l);
            mShowing = true;
    
            sendShowMessage();
        } finally {
        }
    }
```
方法体的内容比较多，我们慢慢看，由于一开始mShowing变量用于表示当前dialog是否正在显示，由于我们刚刚开始调用执行show方法，所以这里的mShowing变量的值为false，所以if分支的内容不会被执行，继续往下看：

```
if (!mCreated) {
            dispatchOnCreate(null);
        }
```
mCreated这个控制变量控制dispatchOnCreate方法只被执行一次，由于我们是第一次执行，所以这里会执行dispatchOnCreate方法，好吧，我们看一下dispatchOnCreate方法的执行逻辑：

```
void dispatchOnCreate(Bundle savedInstanceState) {
        if (!mCreated) {
            onCreate(savedInstanceState);
            mCreated = true;
        }
    }
```
好吧，可以看到代码的执行逻辑很简单就是回调了Dialog的onCreate方法，那么onCreate方法内部又执行了那些逻辑呢？由于我们创建的是AlertDialog对象，该对象继承于Dialog，所以我们这时候需要看一下AlertDialog的onCreate方法的执行逻辑：

```
@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mAlert.installContent();
    }
```
可以看到这里面除了调用super.onCreate方法之外就是调用了mAlert.installContent方法，而这里的super.onCreate方法就是调用的Dialog的onCreate方法，Dialog的onCreate方法只是一个空的实现逻辑，所以我们具体来看一下mAlert.installContent的实现逻辑。

```
public void installContent() {
        /* We use a custom title so never request a window title */
        mWindow.requestFeature(Window.FEATURE_NO_TITLE);
        int contentView = selectContentView();
        mWindow.setContentView(contentView);
        setupView();
        setupDecor();
    }
```
可以看到这里实现Window窗口的页面设置布局初始化等操作，这里设置了mWindow对象为NO_TITLE，然后通过调用selectContentView设置Window对象的布局文件。

```
private int selectContentView() {
        if (mButtonPanelSideLayout == 0) {
            return mAlertDialogLayout;
        }
        if (mButtonPanelLayoutHint == AlertDialog.LAYOUT_HINT_SIDE) {
            return mButtonPanelSideLayout;
        }
        // TODO: use layout hint side for long messages/lists
        return mAlertDialogLayout;
    }
```
可以看到这里通过执行selectContentView方法返回布局文件的id值，这里的默认值是mAlertDialogLayout。看过Activity布局加载流程（<a href="http://blog.csdn.net/qq_23547831/article/details/51284556">android源码解析（十七）-->Activity布局加载流程</a>）的童鞋应该知道，从这个方法开始我们就把指定布局文件的内容加载到内存中的Window对象中。我们这里看一下具体的布局文件。

```
mAlertDialogLayout = a.getResourceId(
                R.styleable.AlertDialog_layout, R.layout.alert_dialog);
```
也就是R.layout.alert_dialog的布局文件，有兴趣的同学可以看一下该布局文件的源码，O(∩_∩)O哈哈~

继续回到我们的installContent方法，在执行了mWindow.setContentView方法之后，又调用了setupView方法和setupDector方法，这两个方法的主要作用就是初始化布局文件中的组件和Window对象中的mDector成员变量，这里就不在详细的说明。

然后回到我们的show方法，在执行了dispatchOnCreate方法之后我们又调用了onStart方法，这个方法主要用于设置ActionBar，这里不做过多的说明，然后初始化WindowManager.LayoutParams对象，并最终调用我们的mWindowManager.addView()方法。

O(∩_∩)O哈哈~，到了这一步大家如果看了上一篇Acitivty布局绘制流程的话，就应该知道顺着这个方法整个Dialog的界面就会被绘制出来了。

最后我们调用了sendShowMessage方法，可以看一下这个方法的实现：

```
private void sendShowMessage() {
        if (mShowMessage != null) {
            // Obtain a new message so this dialog can be re-used
            Message.obtain(mShowMessage).sendToTarget();
        }
    }
```
这里会发送一个Dialog已经显示的异步消息，该消息最终会在ListenersHandler中的handleMessage方法中被执行：

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
由于我们的msg.what = SHOW,所以会执行OnShowListener.onShow方法，那么这个OnShowListener是何时赋值的呢？还记得我们构造AlertDialog.Builder么？

```
alertDialog.setOnShowListener(new DialogInterface.OnShowListener() {
                    @Override
                    public void onShow(DialogInterface dialog) {

                    }
                });
```
这样就为我们的AlertDialog.Builder设置了OnShowListener，可以看一下setOnShowListener方法的具体实现：

```
public void setOnShowListener(OnShowListener listener) {
        if (listener != null) {
            mShowMessage = mListenersHandler.obtainMessage(SHOW, listener);
        } else {
            mShowMessage = null;
        }
    }
```
这样就为我们的Dialog中的mListenersHandler构造了Message对象，并且当我们在Dialog中发送showMessage的时候被mListenersHandler所接收。。。。


注：
这里说一下我们平时开发中若创建的Dialog使用的Context对象不是Activity，就会报出：

```
Process: com.example.aaron.helloworld, PID: 11948                                                                         android.view.WindowManager$BadTokenException: Unable to add window -- token null is not for an application
at android.view.ViewRootImpl.setView(ViewRootImpl.java:690)
at android.view.WindowManagerGlobal.addView(WindowManagerGlobal.java:282)
at android.view.WindowManagerImpl.addView(WindowManagerImpl.java:69)
at android.app.Dialog.show(Dialog.java:298)
at com.example.aaron.helloworld.MainActivity$1.onClick(MainActivity.java:59)
at android.view.View.performClick(View.java:4811)
at android.view.View$PerformClick.run(View.java:20136)
at android.os.Handler.handleCallback(Handler.java:815)
at android.os.Handler.dispatchMessage(Handler.java:104)
at android.os.Looper.loop(Looper.java:194)
at android.app.ActivityThread.main(ActivityThread.java:5552)
at java.lang.reflect.Method.invoke(Native Method)
at java.lang.reflect.Method.invoke(Method.java:372)
at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:964)
at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:759)
```
的异常，这是由于WindowManager.addView方法最终会调用ViewRootImpl.setView方法，而这时候会有mToken的检查，若我们传入的Context对象不是Activity，这时候的mToken为空，就会出现上述问题。。。


总结：

- Dialog和Activity的显示逻辑是相似的都是内部管理这一个Window对象，用WIndow对象实现界面的加载与显示逻辑；

- Dialog中的Window对象与Activity中的Window对象是相似的，都对应着一个WindowManager对象；

- Dialog相关的几个类：Dialog，AlertDialog，AlertDialog.Builder，AlertController，AlertController.AlertParams，其中Dialog是窗口的父类，主要实现Window对象的初始化和一些共有逻辑，而AlertDialog是具体的Dialog的操作实现类，AlertDialog.Builder类是AlertDialog的内部类，主要用于构造AlertDialog，AlertController是AlertDialog的控制类，AlertController.AlertParams类是控制参数类；

- 构造显示Dialog的一般流程，构造AlertDialog.Builder，然后设置各种属性，最后调用AlertDialog.Builder.create方法获取AlertDialog对象，并且create方法中会执行，构造AlertDialog，设置dialog各种属性的操作。最后我们调用Dialog.show方法展示窗口，初始化Dialog的布局文件，Window对象等，然后执行mWindowManager.addView方法，开始执行绘制View的操作，并最终将Dialog显示出来；

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




