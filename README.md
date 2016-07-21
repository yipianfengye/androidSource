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
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51475929">android源码解析（二十七）-->HOME事件流程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51487978">android源码解析（二十八）-->电源开关机按键事件流程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51513771">android源码解析（二十九）-->应用程序返回按键执行流程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51530671"> android源码解析（三十）-->触摸事件分发流程</a>

让坚持成为一种习惯，慢慢努力中！！！
