# Linux Service 启动超时问题排查

> 今天同事遇到个奇怪的问题。在使用linux service 来做java进程宕机重启时，正常的用 systemctl restart service_name 或者 stop、start 时均正常。而几乎每次真的让linux 去执行reboot 命令模拟宕机重启时，Java进程都会启动失败。

## 表象

​	java进程启动时能够正常的看到启动成功，注册到注册中心，或是有去查数据库，但是会冷不丁的来个loaderBalance的线程池关闭，是info级别的。没有任何报错的堆栈日志。

## timeout

​	经过他的几次演示，首先我猜测可能是linux 系统oom killer 之类的进程将其杀死，虽然不应该在启动启动时出现这样的事，但是还是通过dmesg 命令查看了一下。 并没有什么异常的。
​	然后，通过 systemctl status service_name 发现，他下面标记的状态是timeout。
​	通过百度的话，你会发现帖子、博客都在说，默认超时是无限的。我们猜测我们服务器并不是这样，因此，将其改为超时时间10分钟，即 10 min 。**重启后验证启动成功**。

## 莫轻信网上配置

​	不同的linux发行版，不同的版本号，其默认的配置都可能发生变化，还看6年前的博客，只能提供方向，还是要查阅更官方的才合适，最后我们查了，我们服务器默认的是90秒。这个对一个较大的java进程来说，从系统启动开始计时，超时倒也是很正常了。

