# arthas 源码构建

## git下载代码
git clone https://github.com/alibaba/arthas.git

若github被墙，可以在gitee搜索下载

## maven clean

可以在项目目录执行 mvn clean ， ide可以执行界面执行

![image-20221121215001904](/home/dark/GitHub/Blogs/arthas/pic/01-01.png)

## maven package

可以在项目目录执行mvn package

![image-20221121215224227](/home/dark/GitHub/Blogs/arthas/pic/01-02.png)



## 问题记录

### javah 命令不存在

pom文件位于/arthas/arthas-vmtool/pom.xml，属于arthas-vmtool模块，这个模块中有c代码，编译后需要java native使用，所以需要使用javah 命令，但是在高版本的jdk中已经使用`javac -h`进行代替，所以在java环境的bin目录中已经不存在javah 命令（初始版本使用jdk11编译会报错）。

```bash
[ERROR] Failed to execute goal org.codehaus.mojo:native-maven-plugin:1.0-alpha-11:javah (javah) on project arthas-vmtool: Error running javah command: Error executing command line: Error while executing process. Cannot run program "javah" (in directory "/home/dark/IdeaProjects/arthas/arthas-vmtool"): error=2, 没有那个文件或目录 -> [Help 1]
[ERROR] 
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
```

为了解决没有javah 命令，我切换为jdk8，并检查bin目录下的确存在javah，则问题得以解决。但是其他模块却需要更高版本的java，所以查阅文档后，发现`native-maven-plugin`可以配置参数`javahPath`指定javah命令的位置，所以我在使用jdk11时，指定了jdk8的javah命令，在pom.xml中如下改造：

```xml
<plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>native-maven-plugin</artifactId>
                <version>1.0-alpha-11</version>
                <extensions>true</extensions>
                <configuration>
                    <!--<javahPath>${java.home}/../bin/javah</javahPath>-->
                    <!-- 指定你的javah 命令的位置 -->
                    <javahPath>/home/dark/.jdks/jdk1.8.0_202/bin/javah</javahPath>
                    <javahIncludes>
                        <javahInclude>
                            <className>arthas.VmTool</className>
                        </javahInclude>
                    </javahIncludes>
                    <jdkIncludePath>${project.basedir}/src/main/native/head</jdkIncludePath>
                    <javahOS>${os_name}</javahOS>
                    <sources>
                        <source>
                            <directory>src/main/native/src</directory>
                            <fileNames>
                                <fileName>jni-library.cpp</fileName>
                            </fileNames>
                 javahPath       </source>
                    </sources>

                    <compilerProvider>generic-classic</compilerProvider>
                    <compilerExecutable>g++</compilerExecutable>
                    <compilerStartOptions>
                        <compilerStartOption>${os_arch_option}</compilerStartOption>
                        <compilerStartOption>-fpic</compilerStartOption>
                        <compilerStartOption>-shared</compilerStartOption>
                        <compilerStartOption>-o</compilerStartOption>
                    </compilerStartOptions>

                    <linkerOutputDirectory>target</linkerOutputDirectory>
                    <linkerExecutable>g++</linkerExecutable>
                    <linkerStartOptions>
                        <linkerStartOption>${os_arch_option}</linkerStartOption>
                        <linkerStartOption>-fpic</linkerStartOption>
                        <linkerStartOption>-shared</linkerStartOption>
                        <!-- <linkerStartOption>-o</linkerStartOption> -->
                    </linkerStartOptions>
                    <linkerEndOptions>
                        <linkerEndOption>-o ${project.build.directory}/${lib_name}</linkerEndOption>
                    </linkerEndOptions>
                </configuration>
                <executions>
                    <execution>
                        <id>javah</id>
                        <phase>compile</phase>
                        <goals>
                            <goal>javah</goal>
                            <goal>initialize</goal>
                            <goal>compile</goal>
                            <goal>link</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
```

### jfr包找不到

jfr是jdk下的包，但是低版本（jdk 8）中许多类不存在，导致许多类不存在报错，在构建arthas-core模块会失败。

![image-20221121220605187](/home/dark/GitHub/Blogs/arthas/pic/01-03.png)

所以要设置jdk环境为jdk 11，如下图：

![image-20221121220725052](/home/dark/GitHub/Blogs/arthas/pic/01-04.png)