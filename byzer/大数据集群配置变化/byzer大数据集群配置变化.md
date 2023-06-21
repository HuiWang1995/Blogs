# byzer大数据集群配置变化

<img src="https://www.byzer.org/static/media/Byzer_With_Slogan_Color_v1.0.409c7b62.svg" alt="byzer"/>

因cdh升级为cdp后，原本的小集群被大佬收回去到大集群中了，byzer所用的集群配置就需要发生修改。另外，因为大集群的权限配置更为严格，byzer修改配置时就会遇到这些问题。

## 配置下载
从 cloudera manager 中下载hadoop、yarn、hive的配置文件，并覆盖到byzer engine 配置的spark的conf下。

## 修改hosts
cdp集群中的主机都以主机名作为配置，故需要修改/etc/hosts 文件，把对应的主机名与IP映射补上。

## 安装用户切换为hive用户
原先部署在/root/apps目录中，则使用root启动，byzer会直接使用root去做hive的连接。以下几个问题是在调整用户时遇到的。

### byzer启动失败
Permission denied: user=root（具体用户名）,access=WRITE, inode="/user":hdfs:supergroup:drwxr-xr-x

这是因为这个用户没权限，需要授权，或者换个用户

### byzer 任务执行失败
Permission denied: user=root（具体用户名）,access=READ_EXECUTE, inode="/user/xxx/xxx":hive:hive:drwxr-x--x

也是因为权限不足导致的。

### byzer 启动无限输出日志
INFO Client: Application report for application_xxxxxxx_xxxx (state:ACCEPTED)

是任务提交到了yarn，但是未被执行，属于资源分配不足，或者已经有人用这个用户把资源用完了。
可以暂时的byzer部署目录下的 conf/byzer.properties.override 中修改配置将占用资源减小尝试重启。