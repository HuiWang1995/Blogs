# docker快速集群部署redis

> 快速创建集群，方便开发测试。3步完成集群创建。

## 准备配置模板

- redis-cluster.tmpl

```shell
port ${PORT}
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
cluster-announce-ip 172.17.0.1
cluster-announce-port ${PORT}
cluster-announce-bus-port 1${PORT}
appendonly yes

```

## 生成配置文件

- 脚本redis-create-config.sh

  8010 8015表示从8010端口到8015端口

```shell
for port in $(seq 8010 8015);
do
  mkdir -p ./${port}/conf
  PORT=${port} envsubst < ./redis-cluster.tmpl > ./${port}/conf/redis.conf
  mkdir -p ./${port}/data;
done

```

执行脚本后生成目录结构：

> appendonly.aof, dump.rdb, nodes.conf为运行后产物

```shell
├── 8010
│   ├── conf
│   │   └── redis.conf
│   └── data
│       ├── appendonly.aof
│       ├── dump.rdb
│       └── nodes.conf
├── 8011
│   ├── conf
│   │   └── redis.conf
│   └── data
│       ├── appendonly.aof
│       ├── dump.rdb
│       └── nodes.conf
├── 8012
│   ├── conf
│   │   └── redis.conf
│   └── data
│       ├── appendonly.aof
│       ├── dump.rdb
│       └── nodes.conf
├── 8013
│   ├── conf
│   │   └── redis.conf
│   └── data
│       ├── appendonly.aof
│       ├── dump.rdb
│       └── nodes.conf
├── 8014
│   ├── conf
│   │   └── redis.conf
│   └── data
│       ├── appendonly.aof
│       ├── dump.rdb
│       └── nodes.conf
├── 8015
│   ├── conf
│   │   └── redis.conf
│   └── data
│       ├── appendonly.aof
│       ├── dump.rdb
│       └── nodes.conf

```

## 创建Redis容器

- 脚本redis-create-container.sh

> 其中/home/wanglh/config/redis-cluster/为执行以上所有脚本的当前路径，为我当前计算机的路径，改成你自己的。

```shell
for port in $(seq 8010 8015); \
do \
   docker run -it -d -p ${port}:${port} -p 1${port}:1${port} \
  --privileged=true -v /home/wanglh/config/redis-cluster/${port}/conf/redis.conf:/usr/local/etc/redis/redis.conf \
  --privileged=true -v /home/wanglh/config/redis-cluster/${port}/data:/data \
  --restart always --name redis-${port} --net bridge \
  --sysctl net.core.somaxconn=1024 redis redis-server /usr/local/etc/redis/redis.conf; \
done

```

- 创建完成

  执行脚本，创建完成后，使用docker ps查看redis集群容器

```shell
wanglh@dark:~/config/redis-cluster$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                        NAMES
63b6e9c02d48        redis               "docker-entrypoint.s…"   6 days ago          Up 3 seconds        0.0.0.0:8015->8015/tcp, 6379/tcp, 0.0.0.0:18015->18015/tcp   redis-8015
d3e6e1002999        redis               "docker-entrypoint.s…"   6 days ago          Up 3 seconds        0.0.0.0:8014->8014/tcp, 6379/tcp, 0.0.0.0:18014->18014/tcp   redis-8014
4d342165a1a5        redis               "docker-entrypoint.s…"   6 days ago          Up 3 seconds        0.0.0.0:8013->8013/tcp, 6379/tcp, 0.0.0.0:18013->18013/tcp   redis-8013
4bbf3fa9d5d8        redis               "docker-entrypoint.s…"   6 days ago          Up 4 seconds        0.0.0.0:8012->8012/tcp, 6379/tcp, 0.0.0.0:18012->18012/tcp   redis-8012
c1f8c8b4ea64        redis               "docker-entrypoint.s…"   6 days ago          Up 4 seconds        0.0.0.0:8011->8011/tcp, 6379/tcp, 0.0.0.0:18011->18011/tcp   redis-8011
13889ce1bba2        redis               "docker-entrypoint.s…"   6 days ago          Up 5 seconds        0.0.0.0:8010->8010/tcp, 6379/tcp, 0.0.0.0:18010->18010/tcp   redis-8010

```

## 其他脚本

​		本文使用到了创建配置文件脚本redis-create-config.sh，创建Redis容器脚本redis-create-container.sh。但是启动、关闭容器仍需手动命令，图方便，便提供以下脚本。

### 启动脚本

redis-start.sh

```shell
for port in $(seq 8010 8015); \
do \
   docker start redis-${port}; \
done
```

### 关闭脚本

redis-stop.sh

```shell
for port in $(seq 8010 8015); \
do \
   docker stop redis-${port}; \
done
```

### 重启脚本

redis-restart.sh

```shell
for port in $(seq 8010 8015); \
do \
   docker restart redis-${port}; \
done
```

### 禁止开机自启动脚本

redis-disable-autostart.sh

```shell
for port in $(seq 8010 8015); \
do \
   docker update --restart=no redis-${port}; \
done
```

### 删除容器脚本

redis-rm-container.sh

```shell
sh redis-stop.sh
for port in $(seq 8010 8015); \
do \
   docker rm redis-${port}; \
done
```





