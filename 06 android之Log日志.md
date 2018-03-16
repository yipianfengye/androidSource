首先说点题外话，对于想学android framework源码的同学，其实可以在github中fork一份，具体地址：<a href="https://github.com/android/platform_frameworks_base">platform_frameworks_base</a>
这里面基本都是android framework层的源码了。而且最近发现了一个比较不错的github插件：OctoTree，它 是一个浏览器插件，它可以让你在Github 看代码时，左边栏会出现一个树状结构，就像我们在IDE 一样。当我们看一个项目的结构，或者想看具体的某个文件，这样就会很方便。
![image](http://img.blog.csdn.net/20160323143543308)

怎么样这样查看源代码的话是不是很方面？

好了说一下我们今天需要介绍的Log对象，它位于android framework层utils包下，是一个final class类：查看其具体定义：

```
public final class Log {

    /**
     * Priority constant for the println method; use Log.v.
     */
    public static final int VERBOSE = 2;

    /**
     * Priority constant for the println method; use Log.d.
     */
    public static final int DEBUG = 3;

    /**
     * Priority constant for the println method; use Log.i.
     */
    public static final int INFO = 4;

    /**
     * Priority constant for the println method; use Log.w.
     */
    public static final int WARN = 5;

    /**
     * Priority constant for the println method; use Log.e.
     */
    public static final int ERROR = 6;

    /**
     * Priority constant for the println method.
     */
    public static final int ASSERT = 7;

    private Log() {
    }

    /**
     * Send a {@link #VERBOSE} log message.
     * @param tag Used to identify the source of a log message.  It usually identifies
     *        the class or activity where the log call occurs.
     * @param msg The message you would like logged.
     */
    public static int v(String tag, String msg) {
        return println(LOG_ID_MAIN, VERBOSE, tag, msg);
    }

    /**
     * Send a {@link #VERBOSE} log message and log the exception.
     * @param tag Used to identify the source of a log message.  It usually identifies
     *        the class or activity where the log call occurs.
     * @param msg The message you would like logged.
     * @param tr An exception to log
     */
    public static int v(String tag, String msg, Throwable tr) {
        return println(LOG_ID_MAIN, VERBOSE, tag, msg + '\n' + getStackTraceString(tr));
    }

    /**
     * Send a {@link #DEBUG} log message.
     * @param tag Used to identify the source of a log message.  It usually identifies
     *        the class or activity where the log call occurs.
     * @param msg The message you would like logged.
     */
    public static int d(String tag, String msg) {
        return println(LOG_ID_MAIN, DEBUG, tag, msg);
    }

    /**
     * Send a {@link #DEBUG} log message and log the exception.
     * @param tag Used to identify the source of a log message.  It usually identifies
     *        the class or activity where the log call occurs.
     * @param msg The message you would like logged.
     * @param tr An exception to log
     */
    public static int d(String tag, String msg, Throwable tr) {
        return println(LOG_ID_MAIN, DEBUG, tag, msg + '\n' + getStackTraceString(tr));
    }

    /**
     * Send an {@link #INFO} log message.
     * @param tag Used to identify the source of a log message.  It usually identifies
     *        the class or activity where the log call occurs.
     * @param msg The message you would like logged.
     */
    public static int i(String tag, String msg) {
        return println(LOG_ID_MAIN, INFO, tag, msg);
    }

    /**
     * Send a {@link #INFO} log message and log the exception.
     * @param tag Used to identify the source of a log message.  It usually identifies
     *        the class or activity where the log call occurs.
     * @param msg The message you would like logged.
     * @param tr An exception to log
     */
    public static int i(String tag, String msg, Throwable tr) {
        return println(LOG_ID_MAIN, INFO, tag, msg + '\n' + getStackTraceString(tr));
    }

    /**
     * Send a {@link #WARN} log message.
     * @param tag Used to identify the source of a log message.  It usually identifies
     *        the class or activity where the log call occurs.
     * @param msg The message you would like logged.
     */
    public static int w(String tag, String msg) {
        return println(LOG_ID_MAIN, WARN, tag, msg);
    }

    /**
     * Send a {@link #WARN} log message and log the exception.
     * @param tag Used to identify the source of a log message.  It usually identifies
     *        the class or activity where the log call occurs.
     * @param msg The message you would like logged.
     * @param tr An exception to log
     */
    public static int w(String tag, String msg, Throwable tr) {
        return println(LOG_ID_MAIN, WARN, tag, msg + '\n' + getStackTraceString(tr));
    }

    /*
     * Send a {@link #WARN} log message and log the exception.
     * @param tag Used to identify the source of a log message.  It usually identifies
     *        the class or activity where the log call occurs.
     * @param tr An exception to log
     */
    public static int w(String tag, Throwable tr) {
        return println(LOG_ID_MAIN, WARN, tag, getStackTraceString(tr));
    }

    /**
     * Send an {@link #ERROR} log message.
     * @param tag Used to identify the source of a log message.  It usually identifies
     *        the class or activity where the log call occurs.
     * @param msg The message you would like logged.
     */
    public static int e(String tag, String msg) {
        return println(LOG_ID_MAIN, ERROR, tag, msg);
    }

    /**
     * Send a {@link #ERROR} log message and log the exception.
     * @param tag Used to identify the source of a log message.  It usually identifies
     *        the class or activity where the log call occurs.
     * @param msg The message you would like logged.
     * @param tr An exception to log
     */
    public static int e(String tag, String msg, Throwable tr) {
        return println(LOG_ID_MAIN, ERROR, tag, msg + '\n' + getStackTraceString(tr));
    }

    /**
     * Handy function to get a loggable stack trace from a Throwable
     * @param tr An exception to log
     */
    public static String getStackTraceString(Throwable tr) {
        if (tr == null) {
            return "";
        }

        // This is to reduce the amount of log spew that apps do in the non-error
        // condition of the network being unavailable.
        Throwable t = tr;
        while (t != null) {
            if (t instanceof UnknownHostException) {
                return "";
            }
            t = t.getCause();
        }

        StringWriter sw = new StringWriter();
        PrintWriter pw = new PrintWriter(sw);
        tr.printStackTrace(pw);
        pw.flush();
        return sw.toString();
    }

    /**
     * Low-level logging call.
     * @param priority The priority/type of this log message
     * @param tag Used to identify the source of a log message.  It usually identifies
     *        the class or activity where the log call occurs.
     * @param msg The message you would like logged.
     * @return The number of bytes written.
     */
    public static int println(int priority, String tag, String msg) {
        return println(LOG_ID_MAIN, priority, tag, msg);
    }

    /** @hide */ public static final int LOG_ID_MAIN = 0;
    /** @hide */ public static final int LOG_ID_RADIO = 1;
    /** @hide */ public static final int LOG_ID_EVENTS = 2;
    /** @hide */ public static final int LOG_ID_SYSTEM = 3;
    /** @hide */ public static final int LOG_ID_CRASH = 4;

    /** @hide */ @SuppressWarnings("unused")
    public static int println(int bufID,
            int priority, String tag, String msg) {
        return 0;
    }
}
```
可以看到其实final 类，所以我们不能通过继承Log类的方式实现自身的日志工具类，一般的我们可以通过定义Log成员变量的方式，封装Log工具方法；

在Log类中我们定义了六种日志级别，分别是：VERBOSE、DEBUG、INFO、WARN、ERROR、ASSERT等六种级别，但是我们平时使用的只有前五种，即VERBOSE,DEBUG,INFO,WARN,ERROR。

通过查看源代码我们发现Log类中所有的静态日志方法Log.v()，Log.d()，Log.i()，Log.w()，Log.e()等方法都是底层都是调用了println方法，然后在github的源码中查看，其实其内部调用的是println_native方法，也就是通过JNI调用底层的c++输出日志；

我们暂时只是分析到这里，至于底层的c++日志输出的具体实现不作分析，总结一下：

- Log.java是一个final类，所以我们不可以继承Log类来实现自己的日志框架，但是可以通过关联（保存Log成员变量）的方式实现自己的Log工具类；

- Log.java中定义了六种日志级别，但是我们通常只是使用其中的五种日志级别，分别对应着VERBOSE、DEBUG、INFO、WARN、ERROR，在具体的使用场景下具体分析；

- 有些同学对android自带的日志框架不太满意，主要是无法定位日志位置，这里可以查看我写的一篇实现自定义日志框架的文章：<a href="http://blog.csdn.net/qq_23547831/article/details/51707796">github项目解析（五）-->android日志框架</a>

日志可以个性化的展示相关信息：
![image](http://img.blog.csdn.net/20160323145153421)


另外对android源码解析方法感兴趣的可参考我的：
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50634435"> android源码解析之（一）-->android项目构建过程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50751687">android源码解析之（二）-->异步消息机制</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50803849">android源码解析之（三）-->异步任务AsyncTask</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50936584">android源码解析之（四）-->HandlerThread</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50958757">android源码解析之（五）-->IntentService</a>
