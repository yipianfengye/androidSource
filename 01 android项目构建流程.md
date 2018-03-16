平时开发过程中我们通过android studio编写完成android项目之后直接点击 Run 'app'就可以在build/outputs/apk生成可以在android设备中安装的apk文件了，那么整个android源码的构建过程是怎么样的呢？

我们可以根据Google官方提供的流程图来具体了解构建的过程：
![image](http://img.blog.csdn.net/20160204114932917)

通常的构建过程就是如上图所示，下面是具体描述：

1.AAPT(Android Asset Packaging Tool)工具会打包应用中的资源文件，如AndroidManifest.xml、layout布局中的xml等，并将xml文件编译为二进制形式，当然assets文件夹中的文件不会被编译，图片及raw文件夹中的资源也会保持原来的形态，需要注意的是raw文件夹中的资源也会生成资源id。AAPT编译完成之后会生成R.java文件。

2.AIDL工具会将所有的aidl接口转化为java接口。

3.所有的java代码，包括R.java与aidl文件都会被Java编译器编译成.class文件。

4.Dex工具会将上述产生的.class文件及第三库及其他.class文件编译成.dex文件（dex文件是Dalvik虚拟机可以执行的格式），dex文件最终会被打包进APK文件。

5.ApkBuilder工具会将编译过的资源及未编译过的资源（如图片等）以及.dex文件打包成APK文件。

6.生成APK文件后，需要对其签名才可安装到设备，平时测试时会使用debug keystore，当正式发布应用时必须使用release版的keystore对应用进行签名。

7.如果对APK正式签名，还需要使用zipalign工具对APK进行对齐操作，这样做的好处是当应用运行时会减少内存的开销。 

另外对android源码解析方法感兴趣的可参考我的：
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50634435"> android源码解析之（一）-->android项目构建过程</a>
