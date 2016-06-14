在前面的几篇文章中我们分析了Activity与Dialog的加载绘制流程，取消绘制流程，相信大家对Android系统的窗口绘制机制有了一个感性的认识了，这篇文章我们将继续分析一下PopupWindow加载绘制流程。

在分析PopupWindow之前，我们将首先说一下什么是PopupWindow？理解一个类最好的方式就是看一下这个类的定义，这里我们摘要了一下Android系统中PopupWindow的类的说明：

> A popup window that can be used to display an arbitrary view. The popup window is a floating container that appears on top of the current
 activity.

一个PopupWindow能够被用于展示任意的View，PopupWindow是一个悬浮的容易展示在当前Activity的上面。
简单来说PopupWindow就是一个悬浮在Activity之上的窗口，可以用展示任意布局文件。

在说明PopupWindow的加载绘制机制之前，我们还是先写一个简单的例子用于说明一下PopupWindow的简单用法。

```
public static View showPopupWindowMenu(Activity mContext, View anchorView, int layoutId) {
        LayoutInflater inflater = (LayoutInflater) mContext.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        View view = inflater.inflate(layoutId, null);
        popupWindow = new PopupWindow(view, DisplayUtil.dip2px(mContext, 148), WindowManager.LayoutParams.WRAP_CONTENT);
        popupWindow.setBackgroundDrawable(mContext.getResources().getDrawable(R.drawable.menu_bg));
        popupWindow.setFocusable(true);
        popupWindow.setOutsideTouchable(true);

        int[] location = new int[2];
        anchorView.getLocationOnScreen(location);
        popupWindow.setAnimationStyle(R.style.popwin_anim_style);
        popupWindow.showAtLocation(anchorView, Gravity.NO_GRAVITY,
                location[0] - popupWindow.getWidth() + anchorView.getWidth() - DisplayUtil.dip2px(mContext, 12),
                location[1] + anchorView.getHeight() - DisplayUtil.dip2px(mContext, 10));

        popupWindow.setOnDismissListener(new PopupWindow.OnDismissListener() {
            @Override
            public void onDismiss() {
                popupWindow = null;
            }
        });
        return view;
    }
```
可以看到我们首先通过LayoutInflater对象将布局文件解析到内存中View对象，然后创建了一个PopupWindow对象，可以看到传递了三个参数，一个是View对象，一个是PopupWindow的宽度和高度。

这里就是PopupWindow的初始化流程的开始了，好吧，我们来看一下PopupWindow的构造方法的实现：

```
public PopupWindow(View contentView, int width, int height) {
        this(contentView, width, height, false);
    }
```
可以看到这里调用了PopupWindow的重载构造方法，好吧，继续看一下这个重载构造方法的实现逻辑：

```
public PopupWindow(View contentView, int width, int height, boolean focusable) {
        if (contentView != null) {
            mContext = contentView.getContext();
            mWindowManager = (WindowManager) mContext.getSystemService(Context.WINDOW_SERVICE);
        }

        setContentView(contentView);
        setWidth(width);
        setHeight(height);
        setFocusable(focusable);
    }
```
这里首先根据传入的View是否为空做了一下判断，若不为空，则初始化成员变量,Context和mWindowManager，可以发现这里的mContext对象就是传入的View组件中保留的Context对象，这里的mWindowManager是应用进程创建的时候注册的服务本地接口。然后调用了setContentView方法，这里就是为PopupWindow的contentView赋值。然后后面调用的setWidth、setHeight、setFocusable方法都是为PopupWindow的成员变量，width，height，focusable等赋值，这样PopupWindow的构造方法就执行完成了。

我们继续回到我们的例子代码中，在后续的代码中我们调用了：popupWindow.setBackgroundDrawable、popupWindow.setFocusable、PopupWindow.setOutsideTouchable、
PopupWindow.setAnimationStyle等方法，初始化了PopupWindow中的相关成员变量，最后我们调用了popupWindow.showAtLocation方法用于展示PopupWindow，这里我们具体看一下showAtLocation的实现逻辑：

```
public void showAtLocation(View parent, int gravity, int x, int y) {
        showAtLocation(parent.getWindowToken(), gravity, x, y);
    }
```
可以发现，这里调用了showAtLocation的重载函数，这样我们继续看一下这个重载函数的实现方式：

```
public void showAtLocation(IBinder token, int gravity, int x, int y) {
        if (isShowing() || mContentView == null) {
            return;
        }

        TransitionManager.endTransitions(mDecorView);

        unregisterForScrollChanged();

        mIsShowing = true;
        mIsDropdown = false;

        final WindowManager.LayoutParams p = createPopupLayoutParams(token);
        preparePopup(p);

        // Only override the default if some gravity was specified.
        if (gravity != Gravity.NO_GRAVITY) {
            p.gravity = gravity;
        }

        p.x = x;
        p.y = y;

        invokePopup(p);
    }
```
可以看到通过调用createPopupLayoutParams方法创造了WindowManager.LayoutParams对象，然后又调用了preparePopup方法，可以看一下preparePopup方法的具体实现：

```
private void preparePopup(WindowManager.LayoutParams p) {
        if (mContentView == null || mContext == null || mWindowManager == null) {
            throw new IllegalStateException("You must specify a valid content view by "
                    + "calling setContentView() before attempting to show the popup.");
        }

        // The old decor view may be transitioning out. Make sure it finishes
        // and cleans up before we try to create another one.
        if (mDecorView != null) {
            mDecorView.cancelTransitions();
        }

        // When a background is available, we embed the content view within
        // another view that owns the background drawable.
        if (mBackground != null) {
            mBackgroundView = createBackgroundView(mContentView);
            mBackgroundView.setBackground(mBackground);
        } else {
            mBackgroundView = mContentView;
        }

        mDecorView = createDecorView(mBackgroundView);

        // The background owner should be elevated so that it casts a shadow.
        mBackgroundView.setElevation(mElevation);

        // We may wrap that in another view, so we'll need to manually specify
        // the surface insets.
        final int surfaceInset = (int) Math.ceil(mBackgroundView.getZ() * 2);
        p.surfaceInsets.set(surfaceInset, surfaceInset, surfaceInset, surfaceInset);
        p.hasManualSurfaceInsets = true;

        mPopupViewInitialLayoutDirectionInherited =
                (mContentView.getRawLayoutDirection() == View.LAYOUT_DIRECTION_INHERIT);
        mPopupWidth = p.width;
        mPopupHeight = p.height;
    }
```
preparePopup方法的参数是WindowManager.LayoutParams，然后设置了PopupWindow中的几个比较重要的成员变量，首先看一下mBackgroundView的初始化过程：

```
if (mBackground != null) {
            mBackgroundView = createBackgroundView(mContentView);
            mBackgroundView.setBackground(mBackground);
        } else {
            mBackgroundView = mContentView;
        }
```
可以发现如果我们设置了mBackground变量也就是我们在初始化的时候执行了popupWindow的setBackgound方法，那么我们这里执行的就是if分之，这里看一下createBackgourndView的具体执行逻辑：

```
private PopupBackgroundView createBackgroundView(View contentView) {
        final ViewGroup.LayoutParams layoutParams = mContentView.getLayoutParams();
        final int height;
        if (layoutParams != null && layoutParams.height == ViewGroup.LayoutParams.WRAP_CONTENT) {
            height = ViewGroup.LayoutParams.WRAP_CONTENT;
        } else {
            height = ViewGroup.LayoutParams.MATCH_PARENT;
        }

        final PopupBackgroundView backgroundView = new PopupBackgroundView(mContext);
        final PopupBackgroundView.LayoutParams listParams = new PopupBackgroundView.LayoutParams(
                ViewGroup.LayoutParams.MATCH_PARENT, height);
        backgroundView.addView(contentView, listParams);

        return backgroundView;
    }
```
可以看到，createBackgroundView的执行逻辑就是在参数contentView的外面一层包裹一层PopupBackgroundView，而这里的PopupBackgroundView值我们自定义的FrameLayout的子类，重写了其onCreateDrawableState方法。

继续回到我们的preparePopup方法，这里我们又调用了createDecorView方法初始化mDectorView变量，我们可以看一下createDecorView的具体实现：

```
private PopupDecorView createDecorView(View contentView) {
        final ViewGroup.LayoutParams layoutParams = mContentView.getLayoutParams();
        final int height;
        if (layoutParams != null && layoutParams.height == ViewGroup.LayoutParams.WRAP_CONTENT) {
            height = ViewGroup.LayoutParams.WRAP_CONTENT;
        } else {
            height = ViewGroup.LayoutParams.MATCH_PARENT;
        }

        final PopupDecorView decorView = new PopupDecorView(mContext);
        decorView.addView(contentView, ViewGroup.LayoutParams.MATCH_PARENT, height);
        decorView.setClipChildren(false);
        decorView.setClipToPadding(false);

        return decorView;
    }
```
可以发现这里也是给参数contentView外面包裹了一层PopupDecorView，这里的PopupDecorView也是我们自定义的FrameLayout的子类，PopupDecorView的源码比较多，这里就不都贴出来了，这里具体看一下其onTouchEvent方法的实现：

```
@Override
        public boolean onTouchEvent(MotionEvent event) {
            final int x = (int) event.getX();
            final int y = (int) event.getY();

            if ((event.getAction() == MotionEvent.ACTION_DOWN)
                    && ((x < 0) || (x >= getWidth()) || (y < 0) || (y >= getHeight()))) {
                dismiss();
                return true;
            } else if (event.getAction() == MotionEvent.ACTION_OUTSIDE) {
                dismiss();
                return true;
            } else {
                return super.onTouchEvent(event);
            }
        }
```
可以发现其重写了onTouchEvent时间，这样我们在点击popupWindow外面的时候就会执行pupopWindow的dismiss方法，取消PopupWindow。

好吧，继续回到我们的showAsDropDown方法，在执行完成preparePopup方法之后又调用了invokePopup方法，这里的方法应该就是具体执行PopupWindow的加载与显示逻辑了。这里我们具体看一下其实现逻辑：

```
private void invokePopup(WindowManager.LayoutParams p) {
        if (mContext != null) {
            p.packageName = mContext.getPackageName();
        }

        final PopupDecorView decorView = mDecorView;
        decorView.setFitsSystemWindows(mLayoutInsetDecor);

        setLayoutDirectionFromAnchor();

        mWindowManager.addView(decorView, p);

        if (mEnterTransition != null) {
            decorView.requestEnterTransition(mEnterTransition);
        }
    }
```
我们看到这里我们调用了mWindowManager.addView方法，看过我们前面几篇关于Dialog和Activity的加载与现实流程的同学应该知道这里的addView其实是我们布局绘制的流程，这里的mWindowManager是我们在调用PopupWIndow的构造函数的时候初始化的，其调用的是：

```
if (mWindowManager == null && mContentView != null) {
            mWindowManager = (WindowManager) mContext.getSystemService(Context.WINDOW_SERVICE);
        }
```
而这里的mContext.getSystemService是一个接口其具体的实现是在ContextImpl中实现的，所以这里我们看一下ContextImpl的getSystemService的实现：

```
@Override
    public Object getSystemService(String name) {
        return SystemServiceRegistry.getSystemService(this, name);
    }
```
好吧，在ContextImpl中的getSystemService方法又调用了SystemServiceRegister中的静态方法getSystemService，这样我们再看看一下在SystemServiceRegister是如何实现的。

```
public static Object getSystemService(ContextImpl ctx, String name) {
        ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
        return fetcher != null ? fetcher.getService(ctx) : null;
    }
```
这里发现服务对象的获取就是通过一个SYSTEM_SERVICE_FETCHERS的map数据结构获取的，那么这个map对象的数据是何时填充的呢？通过查看源码我们发下在SystemServiceRegister中有一段静态代码主要用于注册本地服务接口，其中关于windowManagerService本地服务的代码如下：

```
registerService(Context.WINDOW_SERVICE, WindowManager.class,
                new CachedServiceFetcher<WindowManager>() {
            @Override
            public WindowManager createService(ContextImpl ctx) {
                return new WindowManagerImpl(ctx.getDisplay());
            }});
```
好吧，原来我们通过mContext.getSystemService获取的WindowManager其实际上是一个WindowManagerImpl对象，而我们调用的addView就是WindowManagerImpl的addView方法。

这样就回到了我们前几篇讲解的内容上了，通过调用WindowManagerImpl实现了布局文件的绘制流程。。。。

好了，经过上面的一系列的操作我们分析完了PopupWindow的加载绘制流程，其和Dialog，Activity的加载绘制流程类似，都是通过Window对象控制布局文件的加载与绘制流程。

总结：

- PopupWindow的界面加载绘制流程也是通过Window对象实现的；

- PopupWindow内部保存的mWindowManager对象通过ContextImpl中获取，并且取得的是WindowManagerImpl对象；

- PopupWindow通过为传入的View添加一层包裹的布局，并重写该布局的点击事件，实现点击PopupWindow之外的区域PopupWindow消失的效果；

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
