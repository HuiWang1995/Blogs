# graalvm 拯救生命，速速入手

标题很夸张，graalvm怎么就拯救生命了？把一个启动5-6秒的项目加速到3秒启动，不就是在拯救生命，拯救发际线吗？

我在上一篇博客["SpringBoot3.0工程建立"](https://blog.csdn.net/a17816876003/article/details/128379224)末尾启动了工程，其启动时间为6.2s，多次尝试也在5.5s以上。但是graalvm 可以使启动时间降低为3秒。

## graalvm官网下载

官网下载链接为 https://www.graalvm.org/downloads/

其会引导你跳转到Github下载https://github.com/graalvm/graalvm-jdk-downloader 但是网络情况不佳的时候是上不了的。

当前最新版为GraalVM Community 22.3，使用官网首页下载命令进行下载，会获得jdk 17版本的graalvm。当然，仍然需要能够连接到github。其包大小在284M，比一般JVM大许多，毕竟支持那么多语言运行。

```bash
bash <(curl -sL https://get.graalvm.org/jdk)
```

如果手动下载的包，使用tar -xzvf xxx.tar,gz 进行解压即可。使用命令的会自动解压，并尝试配置你的环境变量，JAVA_HOME。

进入其目录，应如下：

```bash
dark@dark:~/.jdks/gvm/graalvm-ce-java17-22.3.0$ ls -al
总用量 448
drwxrwxr-x 10 dark dark   4096 12月 20 00:41 .
drwxr-xr-x  3 dark dark   4096 12月 20 00:41 ..
drwxrwxr-x  2 dark dark   4096 12月 20 00:41 bin
drwxrwxr-x  5 dark dark   4096 12月 20 00:41 conf
-rw-rw-r--  1 dark dark   1611 10月 20 18:42 GRAALVM-README.md
drwxrwxr-x  3 dark dark   4096 12月 20 00:41 include
drwxrwxr-x  2 dark dark   4096 12月 20 00:41 jmods
drwxrwxr-x  6 dark dark   4096 12月 20 00:41 languages
drwxrwxr-x 72 dark dark   4096 12月 20 00:41 legal
drwxrwxr-x 13 dark dark   4096 12月 20 00:41 lib
-rw-rw-r--  1 dark dark  21035 12月 19 23:47 LICENSE_NATIVEIMAGE.txt
-rw-rw-r--  1 dark dark  23491 10月 20 18:42 LICENSE.txt
-rw-rw-r--  1 dark dark   4128 10月 20 19:09 release
-rw-rw-r--  1 dark dark 354225 10月 20 18:42 THIRD_PARTY_LICENSE.txt
drwxrwxr-x  9 dark dark   4096 12月 20 00:41 tools

```

## 验证JAVA版本

在graalvm的bin目录下执行命令验证java版本，当前的版本为17.0.5

```bash
dark@dark:~/.jdks/gvm/graalvm-ce-java17-22.3.0/bin$ ./java -version
openjdk version "17.0.5" 2022-10-18
OpenJDK Runtime Environment GraalVM CE 22.3.0 (build 17.0.5+8-jvmci-22.3-b08)
OpenJDK 64-Bit Server VM GraalVM CE 22.3.0 (build 17.0.5+8-jvmci-22.3-b08, mixed mode, sharing)

```



## IDE切换JAVA版本

1. 使用快捷键ctrl+shift+alt+s。或者依次点击：文件->项目结构，并在平台设置->SDK中加入我们刚解压好的graalvm，如图:

![image-20221220010306341](/home/dark/GitHub/Blogs/springboot/02/pic/01.png)

2. 在项目中，将sdk选择为我们刚引入的graalvm-17

![image-20221220010453460](/home/dark/GitHub/Blogs/springboot/02/pic/02.png)

## 启动项目

执行Application中的main方法来验证一下启动速度：

```shell
/home/dark/.jdks/gvm/graalvm-ce-java17-22.3.0/bin/java org.dark.migration.MigrationApplication

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.0.0)

2022-12-20T01:06:02.552+08:00  INFO 10606 --- [           main] o.dark.migration.MigrationApplication    : Starting MigrationApplication using Java 17.0.5 with PID 10606 (/home/dark/code/migration/target/classes started by dark in /home/dark/code/migration)
2022-12-20T01:06:02.561+08:00  INFO 10606 --- [           main] o.dark.migration.MigrationApplication    : No active profile set, falling back to 1 default profile: "default"
2022-12-20T01:06:03.276+08:00  INFO 10606 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.
2022-12-20T01:06:03.299+08:00  INFO 10606 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 12 ms. Found 0 JPA repository interfaces.
2022-12-20T01:06:03.941+08:00  INFO 10606 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2022-12-20T01:06:04.550+08:00  INFO 10606 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
2022-12-20T01:06:04.611+08:00  INFO 10606 --- [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]
2022-12-20T01:06:04.669+08:00  INFO 10606 --- [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.1.5.Final
2022-12-20T01:06:05.420+08:00  INFO 10606 --- [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
2022-12-20T01:06:05.822+08:00  INFO 10606 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2022-12-20T01:06:05.832+08:00  INFO 10606 --- [           main] o.dark.migration.MigrationApplication    : Started MigrationApplication in 3.832 seconds (process running for 4.358)

```

可以看到启动速度仅3.8s，有效挽救了博主的顶上三毛。
