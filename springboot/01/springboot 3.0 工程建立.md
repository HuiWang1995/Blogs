# springboot 3.0 工程建立



## 脚手架搭建

进入spring官网提供的https://start.spring.io/进行脚手架搭建。

选择 Maven进行包管理，语言选择JAVA，Spring Boot 版本选择3.0.0，JDK 版本选择17。并在右侧选择自己希望的依赖。结果如下图：

![image-20221219222456998](/home/dark/GitHub/Blogs/springboot/01/pic/01.png)



## 下载JDK 17

spring boot 依赖jdk 版本最低为17。可以在idea里自行下载，也可以自己选择需要的发行版下载。

可以在oracle 官网下载 https://www.oracle.com/java/technologies/downloads/

指定jdk17 https://www.oracle.com/java/technologies/downloads/#java17

linux下也可以使用命令直接下载到当前目录（linux下建议下载到 `~/.jdks/` 即当前用户主目录下的.jdks文件夹，idea的下载默认也在这个目录）

```shell
wget https://download.oracle.com/java/17/latest/jdk-17_linux-x64_bin.tar.gz
```

解压

```shell
tar -xzvf jdk-17.0.5_linux-x64_bin.tar.gz 
```



## 在IDE中打开

在点击`GENERATE`下载zip压缩包之后，在本地解压。再通过IDEA打开，选择JDK为jdk17(IDEA应该会为你自动检测到它的)。

等待Maven解析自动完成,时长取决于与中央仓库的连接网速。

再执行`mvn clean compile -U` 将所需依赖都拉到本地，同样取决于依赖的多寡与网速决定时间，首次构建需要一杯咖啡～。



## 新增项目配置

因为在依赖中添加了hibernate依赖，启动需要配置数据库连接信息。在resources/application.properties中添加如下配置：

```properties
# datasource config
spring.datasource.url=jdbc:mysql://localhost:3306/test?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai
spring.datasource.username=dark
spring.datasource.password=123456
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
```



## 项目启动

在main方法上点击启动即可：

```shell
/home/dark/.jdks/jdk-17.0.5/bin/java org.dark.migration.MigrationApplication

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.0.0)

2022-12-19T23:11:37.304+08:00  INFO 29737 --- [           main] o.dark.migration.MigrationApplication    : Starting MigrationApplication using Java 17.0.5 with PID 29737 (/home/dark/code/migration/target/classes started by dark in /home/dark/code/migration)
2022-12-19T23:11:37.311+08:00  INFO 29737 --- [           main] o.dark.migration.MigrationApplication    : No active profile set, falling back to 1 default profile: "default"
2022-12-19T23:11:38.468+08:00  INFO 29737 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.
2022-12-19T23:11:38.504+08:00  INFO 29737 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 21 ms. Found 0 JPA repository interfaces.
2022-12-19T23:11:39.634+08:00  INFO 29737 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2022-12-19T23:11:40.619+08:00  INFO 29737 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
2022-12-19T23:11:40.787+08:00  INFO 29737 --- [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.1.5.Final
2022-12-19T23:11:42.666+08:00  INFO 29737 --- [           main] o.dark.migration.MigrationApplication    : Started MigrationApplication in 6.239 seconds (process running for 7.069)

```

省略了一部分日志，可以看到tomcat启动成功，默认监听端口8080, hibernate启动了，并使用HikariPool连接了数据库。