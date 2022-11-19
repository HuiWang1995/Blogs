# sshpass 命令踩坑记录--未获取环境变量

> 近日，由于某个厂商的安全流量分析的探针导致了java进程假死，在使用jstack命令却报虚拟机版本不匹配的问题。几番查询，发现未使用该用户配置在.bash_profile下的java环境变量。而是操作系统预装的openJdk。

## 不执行用户目录下.bash_profile
因此，通过root用户 yum 删除了openJdk。然引起了gitee企业版流水线执行主机部署阶段报出异常 java命令找不到。

## 不执行/etc/profile

接着，通过root用户在/etc/profile中配置了java环境变量。然流水线仍然报错，java命令找不到。

## 对比预装openJdk

通过 `whereis java`命令，对比了其他还保留预装openJdk的机器，发现java命令的位置在/usr/bin/下。

## 建立软链接

通过 **`ln -s 被链接的源文件 链接文件`**的方式将java命令链接为/usr/bin/java。之后流水线就正常运行了。



## 三者对比

通过个人电脑，对三种情况（root/个人用户/sshpass）进行了比较，得到如下结果：

```shell
wanglh@dark:~$ su - root
请输入密码:
验证成功
root@dark:~# echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin
root@dark:~# exit
注销
wanglh@dark:~$ echo $PATH
/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games:/sbin:/usr/sbin:/snap/bin
wanglh@dark:~$ sshpass -p 密码 ssh -o 'StrictHostKeyChecking=no' wanglh@127.0.0.1 'echo $PATH'
/usr/local/bin:/usr/bin:/bin:/usr/games

```

环境变量果然是sshpass得到的最少。

## 登录方式的不同

linux 登录分 交互式登录、非交互式登录。我们正常登录终端是交互式登录，使用脚本去执行就是非交互式登录。

例如我们在切换用户时使用的`su - username`、`su username`即是不同的方式，带着“-”符号的为交互式的方式登录，会清空当前的环境变量，并去执行初始化脚本。而后者则是继承当前的环境变量。

可参考https://blog.csdn.net/gui951753/article/details/79154496

## 总结

java命令会受到环境变量的影响，使用了其他版本的jdk。最好，环境中只存在一个java环境。

另外，该厂商给出意见，对于流水线中的脚本，可以指明java命令的绝对路径，防止此类问题。
