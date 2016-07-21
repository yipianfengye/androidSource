# androidSource

<img src="http://ac-mhke0kuv.clouddn.com/d01d862c0f04a1450315.png?imageView/1/w/220/h/120/q/100/format/png" width="1000" height="250">

知乎上看了一篇非常不错的博文：<a href="http://zhuanlan.zhihu.com/kaede/20563936">有没有必要阅读ANDROID源码</a>
看完之后痛定思过，平时所学往往是知其然然不知其所以然，所以为了更好的深入android体系，决定学习android framework层源码。这篇文章就是源码学习的汇总篇，包含学习源码的流程，文章列表等等，会根据学习的进度不定时更新。

在学习源码的时候容易进入一个误区就是只见树木不见森林，具体而言就是对某一个知识点扣的太死了，而忽略了整个流程，所以在我学习的过程中主要学习源码的执行流程而不纠结于细节，可能有的地方理解的不够深刻，有错误的地方希望大家指正。

在分析android源码的过程中我更希望以一种有序的分析过程来分framework的源码，这里我简单的以以下的源码流程来分析：

- <font color="red">异步消息机制源码

- <font color="red">系统核心进程启动流程源码

- <font color="red">应用进程启动流程源码

- <font color="red">apk解析与安装流程源码

- <font color="red">Activity启动销毁流程源码

- <font color="red">Activity绘制与销毁绘制流程源码

- <font color="red">Dialog,PopupWindow,Toast绘制取消绘制流程源码

- <font color="red">Activity其他成员方法执行流程源码

- <font color="red">系统按键处理流程源码

- Service启动销毁流程源码

- BroadcastReceiver流程源码

- ContextProvider流程源码

其中红色字体部分是我已经解析了的源码列表，黑色字体的流程是尚未解析的源码流程列表（PS：可能列表会随时更新奥）


<br>
**<font color="red">android源码解析系列文章列表（会根据解析过程随时更新文章列表）：**
<br>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50634435"> android源码解析之（一）-->android项目构建过程</a>
> 平时开发过程中我们通过android studio编写完成android项目之后直接点击 Run ‘app’就可以在build/outputs/apk生成可以在android设备中安装的apk文件了，那么整个android源码的构建过程是怎么样的呢？...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/50751687">android源码解析之（二）-->异步消息机制</a>
> 为了更好的深入android体系，决定学习android framework层源码，就从最简单的android异步消息机制开始吧。所以也就有了本文：android中的异步消息机制。本文主要从源码角度分析android的异步消息机制...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/50803849">android源码解析之（三）-->异步任务AsyncTask</a>
> android的异步任务体系中还有一个非常重要的操作类：AsyncTask，其内部主要使用的是java的线程池和Handler来实现异步任务以及与UI线程的交互。本文我们将从源码角度分析一下AsyncTask的基本使用和实现原理...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/50936584">android源码解析之（四）-->HandlerThread</a>
> 本文我们将讲解HandlerThread相关的概念。HandlerThread是个什么东西？查看类的定义时有这样一段话：...意思就是说：这个类的作用是创建一个包含looper的线程。 那么我们在什么时候需要用到它呢?

<br><a href="http://blog.csdn.net/qq_23547831/article/details/50958757">android源码解析之（五）-->IntentService</a>
> 本文我们将讲解IntentService相关的知识。什么是IntentService？简单来说IntentService就是一个自身含有消息循环的Service，首先它是一个service，所以service相关具有的特性他都有，同时他还有一些自身的属性，其内部封装了一个消息队列和一个HandlerThread，在其具体的抽象方法：onHandleIntent方法是运行在其消息队列线程中，废话不多说，我们来看其简单的使用方法...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/50963006">android源码解析之（六）-->Log日志API</a>
> 本文我们将介绍一下android中的日志API，其主要是我们即将需要介绍的Log对象，它位于android framework层utils包下，是一个final class类：查看其具体定义...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/50971968">android源码解析之（七）-->LruCache</a>
> android开发过程中经常会用到缓存，现在主流的app中图片等资源的缓存策略一般是分两级，一个是内存级别的缓存，一个是磁盘级别的缓存。 作为android系统的维护者google也开源了其缓存方案，LruCache和DiskLruCache。从android3.1开始LruCache已经作为android源码的一部分维护在android系统中，为了兼容以前的版本android的support-...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51104873">android源码解析之（八）-->Zygote进程启动流程</a>
> 大家都知道android系统的Zygote进程是所有的android进程的父进程，包括SystemServer和各种应用进程都是通过Zygote进程fork出来的。Zygote（孵化）进程相当于是android系统的根进程，后面所有的进程都是通过这个进程fork出来的，而Zygote进程则是通过linux系统的init进程启动的，也就是说，android系统中各种进程的启动方式init进程 –>...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51105171">android源码解析之（九）-->SystemServer进程启动流程</a>
> 上面一文中我们讲过android系统中比较重要的几个进程：init进程，Zygote进程，SystemServer进程已经各种应用进程，其中Zygote进程是整个android系统的根进程，包含SystemServer进程已经各种应用进程在内的进程都是通过Zygote进程fork出来的，具体可参见...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51112031">android源码解析之（十）-->Launcher启动流程</a>
> Launcher程序就是我们平时看到的桌面程序，它其实也是一个android应用程序，只不过这个应用程序是系统默认第一个启动的应用程序，这里我们就简单的分析一下Launcher应用的启动流程。不同的手机厂商定制android操作系统的时候都会更改Launcher的源代码，我们这里以android23的源码为例大致的分析一下Launcher的启动流程。 通过上一篇文章，我们知道SystemSe...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51119333">android源码解析之（十一）-->应用进程启动流程</a>
> 每一个android应用默认都是在他自己的linux进程中运行。android操作系统会在这个android应用中的组件需要被执行的时候启动这个应用进程，并且会在这个应用进程没有任何组件执行或者是系统需要为其他应用申请更多内存的时候杀死这个应用进程。所以当我们需要启动这个应用的四大组件之一的时候如果这个应用的进程还没有启动，那么就会先启动这个应用程序进程。本节主要是通过分析Activity的启动过程介绍应用程序进程的启动流程...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51203482">android源码解析之（十二）-->系统启动并解析Manifest的流程</a>
> 大家应该都知道，Android系统启动之后，我们就可以在一个应用中打开另一个从未打开过的应用，或者是在一个应用中发送广播，如果另外一个应用设置了这个广播的接收器，那么这个应用进程就会被启动并接收该广播并作出相应的处理，这样的例子很多，我们可以猜测到Android系统在启动的时候就会抓取到了系统中所有安装的应用信息（应该是解析apk文件的Manifest信息），即在Android系统的启动过程中就已经解析了系统中安装应用的androidManifest.xml文件并保存起来了，那么这个过程具体是如何的呢?...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51210682">android源码解析之（十三）-->apk安装流程</a>
> 上一篇文章中给大家分析了一下android系统启动之后调用PackageManagerService服务并解析系统特定目录，解析apk文件并安装的过程，这个安装过期实际上是没有图形界面的，底层调用的是我们平时比较熟悉的adb命令，那么我们平时安装apk文件的时候大部分是都过图形界面安装的，那么这种方式安装apk具体的流程是怎样的呢？下面我们就来具体看一下apk的具体安装过程，相信大家都知道如果我们想...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51224992">android源码解析之（十四）-->Activity启动流程</a>
> 好吧，终于要开始讲解Activity的启动流程了，Activity的启动流程相对复杂一下，涉及到了Activity中的生命周期方法，涉及到了Android体系的CS模式，涉及到了Android中进程通讯Binder机制等等，首先介绍一下Activity，这里引用一下Android guide中对Activity的介绍： An activity represents a single screen...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51232309">android源码解析之（十五）-->Activity销毁流程</a>
> 继续我们的源码解析，上一篇文章我们介绍了Activity的启动流程，一个典型的场景就是Activity a 启动了一个Activity b，他们的生命周期回调方法是： onPause(a) –> onCreate(b) –> onStart(b) –> onResume(b) –> onStop(a) 而我们根据源码也验证了这样的生命周期调用序列，那么Activity的销毁流程呢？它的生命周期...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51252082">android源码解析（十六）-->应用进程Context创建流程</a>
> 今天讲讲应用进程Context的创建流程，相信大家平时在开发过程中经常会遇到对Context对象的使用，Application是Context，Activity是Context，Service也是Context，所以有一个经典的问题是一个App中一共有多少个Context？所以这个问题的答案是Application + N个Activity + N个Service。还有就是我们平时在使用Contex...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51284556">android源码解析（十七）-->Activity布局加载流程</a>
> 好吧，终于要开始讲讲Activity的布局加载流程了，大家都知道在Android体系中Activity扮演了一个界面展示的角色，这也是它与android中另外一个很重要的组件Service最大的不同，但是这个展示的界面的功能是Activity直接控制的么？界面的布局文件是如何加载到内存并被Activity管理的？android中的View是一个怎样的概念？加载到内存中的布局文件是如何绘制出来的？...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51285804">android源码解析（十八）-->Activity布局绘制流程</a>
> 这篇文章是承接上一篇文章来写的，大家都知道Activity在Android体系中扮演者一个界面展示的角色，通过上一篇文章的分析，我们知道Activity是通过Window来控制界面的展示的，一个Window对象就是一个窗口对象，而每个Activity中都有一个相应的Window对象，所以说一个Activity对象也就可以说是一个窗口对象，而Window只是控制着界面布局文件的加载过程，那么界面布局...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51289456">android源码解析（十九）-->Dialog加载绘制流程</a>
> 其实Android系统中所有的显示控件（注意这里是控件，而不是组件）的加载绘制流程都是类似的，包括：Dialog的加载绘制流程，PopupWindow的加载绘制流程，Toast的显示原理等，上一篇文章中，我说在介绍了Activity界面的加载绘制流程之后，就会分析一下剩余几个控件的显示控制流程，这里我打算先分析一下Dialog的加载绘制流程...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51303072">android源码解析（二十）-->Dialog取消绘制流程</a>
> 上几篇文章中我们分析了Dialog的加载绘制流程，也分析了Acvityi的加载绘制流程，说白了Android系统中窗口的展示都是通过Window对象控制，通过ViewRootImpl对象执行绘制操作来完成的，那么窗口的取消绘制流程是怎么样的呢？这篇文章就以Dialog为例说明Window窗口是如何取消绘制的。 有的同学可能会问前几篇文章介绍Activity的加载绘制流程的时候为何没有讲...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51322574">android源码解析（二十一）-->PopupWindow加载绘制流程</a>
> 在前面的几篇文章中我们分析了Activity与Dialog的加载绘制流程，取消绘制流程，相信大家对Android系统的窗口绘制机制有了一个感性的认识了，这篇文章我们将继续分析一下PopupWindow加载绘制流程。 在分析PopupWindow之前，我们将首先说一下什么是PopupWindow？理解一个类最好的方式就是看一下这个类的定义，这里我们摘要了一下Android系统中...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51374627">android源码解析（二十二）-->Toast加载绘制流程</a>
> 前面我们分析了Activity、Dialog、PopupWindow的加载绘制流程，相信大家对整个Android系统中的窗口绘制流程已经有了一个比较清晰的认识了，这里最后再给大家介绍一下Toast的加载绘制流程。其实Toast窗口和Activity、Dialog、PopupWindow有一个不太一眼的地方，就是Toast窗口是属于系统级别的窗口，他和输入框等类似的，不属于某一个应用，即不属于某...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51382326">android源码解析（二十三）-->Android异常处理流程</a>
> 前面的几篇文章都是讲解的android中的窗口显示机制，包括Activity窗口加载绘制流程，Dialog窗口加载绘制流程，PopupWindow窗口加载绘制流程，Toast窗口加载绘制流程等等。整个Android的界面显示的原理都是类似的，都是通过Window对象控制View组件，实现加载与绘制流程。 这篇文章休息一下，不在讲解Android的窗口绘制机制，穿插的讲解一下Android系统的异...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51464535">android源码解析（二十四）-->onSaveInstanceState执行时机</a>
> 我们已经分析过Activity的启动流程，从中也分析了Activity的生命周期。而其中有一个生命周期方法:onSaveInstanceState方法，今天我们主要讲解一下onSaveInstanceState方法的执行时机。可能部分同学对Activity的onSaveInstanceState方法不是特别熟悉，这里我们简单介绍一下。onSaveInstanceState方法是Activity的...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51465071">android源码解析（二十五）-->onLowMemory执行流程</a>
> Android系统中一个个的App都是一个个不同的应用进程，拥有各自的JVM与运行时，每个App的进程可使用的内存大小都是固定的，当系统中App打开数量过多时，就会使Android系统的可用内存降低，对于当前正在使用的App而言，可能还需要继续申请系统内存，而我们的剩余系统内存已经不足以被当前App所申请了，这时候系统会自动的清理那些后台进程，进而释放出可用内存用于前台进程的使用，当然这里系统清理后台进程的算法不是我们讨论的重点。这里我们只是大概的分析Android系统回调Activity的onLowMemory方法的流程...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51474288">android源码解析（二十六）-->截屏事件流程</a>
> 今天这篇文章我们主要讲一下Android系统中的截屏事件处理流程。用过android系统手机的同学应该都知道，一般的android手机按下音量减少键和电源按键就会触发截屏事件（国内定制机做个修改的这里就不做考虑了）。那么这里的截屏事件是如何触发的呢？触发之后android系统是如何实现截屏操作的呢？带着这两个问题，开始我们的源码阅读流程。 我们知道这里的截屏事件是通过我们的按键操作触发的，所以这...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51475929">android源码解析（二十七）-->HOME事件流程</a>
> 上一篇文章中我们介绍了android系统的截屏事件，事件的处理逻辑不是在App中执行而是在PhoneWindowManager中执行，而本文我们现在主要讲解android系统中HOME按键的事件处理，和截屏事件类似，这里的HOME按键应该都是系统级别的按键事件监听，所以其处理事件的逻辑也应该和截屏事件处理流程类似，HOME按键的处理逻辑也是在PhoneWindowManager的...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51487978">android源码解析（二十八）-->电源开关机按键事件流程</a>
> 和截屏按键、HOME按键的处理流程类似，电源按键由于也是系统级别的按键，所以对其的事件处理逻辑是和截屏按键、HOME按键类似，不在某一个App中，而是在PhoneWindowManager的dispatchUnhandledKey方法中。所以和前面两篇类似，这里我们也是从PhoneWindowManager的dispatchUnhandledKey方法开始我们今天电源开关机按键的事件流程分析...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51513771">android源码解析（二十九）-->应用程序返回按键执行流程</a>
> 从这篇文章中我们开始分析android系统的事件分发流程，其实网上已经有了很多关于android系统的事件分发流程的文章，奈何看了很多但是印象还不是很深，所以这里总结一番。 android系统的事件分发流程分为很多部分： - Native层 --> ViewRootImpl层 --> DecorView层 --> Activity层 --> ViewGroup层 --> View层...

<br><a href="http://blog.csdn.net/qq_23547831/article/details/51530671"> android源码解析（三十）-->触摸事件分发流程</a>
> 前面一篇文章中我们分析了App返回按键的分发流程，从Native层到ViewRootImpl层到DocorView层到Activity层，以及在Activity中的dispatchKeyEvent方法中分发事件，最终调用了Activity的finish方法，即销毁Activity，所以一般情况下假如我们不重写Activity的onBackPress方法或者是onKeyDown方法，当我们按下并抬起...

让坚持成为一种习惯，慢慢努力中！！！
