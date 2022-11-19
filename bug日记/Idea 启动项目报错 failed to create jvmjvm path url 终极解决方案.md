# Idea 启动项目报错 failed to create jvm:jvm path url 或Could not reserve enough space for xxxxKB object heap 终极解决方案

> 新老版本IDEA都有这样的问题。

当我们项目设置了JVM大小 -Xmx4096m ，但是配置文件中不够大就会产生这样的问题。当然你可能没有这样的设置，同样导致无法启动。

- 配置文件：

  用户目录下：*C:\Users\名字\.IntelliJIdea2019.3\config* 路径下生成一个文件：idea64.exe.vmoptions

  安装目录下：*bin* 路径下同样会有idea64.exe.vmoptions

例如这篇博客（https://blog.csdn.net/suhang1992/article/details/105694905）中提到的，删除配置文件，能解决大部分人的问题。



我们都知道 -Xmx4096m 设置了应用JVM大小，可能是我们就需要这么大，那怎么调整配置文件呢？打开这两个文件，可以看到格式大部分都如下：

```properties
-Xms128m
-Xmx4096m
-XX:ReservedCodeCacheSize=240m
-XX:+UseConcMarkSweepGC
-XX:SoftRefLRUPolicyMSPerMB=50
-ea
-XX:CICompilerCount=2
-Dsun.io.useCanonPrefixCache=false
-Djdk.http.auth.tunneling.disabledSchemes=""
-XX:+HeapDumpOnOutOfMemoryError
-XX:-OmitStackTraceInFastThrow
-Djdk.attach.allowAttachSelf=true
-Dkotlinx.coroutines.debug=off
-Djdk.module.illegalAccess.silent=true
-Dawt.useSystemAAFontSettings=lcd
-Dsun.java2d.renderer=sun.java2d.marlin.MarlinRenderingEngine
-Dsun.tools.attach.tmp.only=true

```

具体属性什么意思我就不多解释了，我们这边要调整的有`-XX:ReservedCodeCacheSize=240m`，这个值需要足够大。如果该值过小，可能还会导致IDEA启动不了，报这样的错：

![img](https://www.pianshen.com/images/553/20f94e7d51acfd913e946c069823d8a1.png)

两者的问题都一样，内存声明不够大，配置加大即可。

当然，启动项目报错还要考虑编译器的JVM内存大小：

![img](https://www.pianshen.com/images/83/8c4c9ccd44cfae73630368757af1eea3.png)

无论是gradle还是maven，都有这样的配置。