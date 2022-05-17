# es应用笔记1-es部署

> 偷懒起见，用docker部署elasticsearch以及kibana。 为了结合实际使用，部署7.6.2版本。

参考：[docker部署es](https://www.jianshu.com/p/5df149854878)

## 部署es

### 拉取镜像

```shell
docker pull docker.elastic.co/elasticsearch/elasticsearch:7.6.2
```

### 启动

```shell
docker run -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.6.2
```

## 部署kibana

```shell
docker pull docker.elastic.co/kibana/kibana:7.6.2
```

## 启动

```shell
docker run --link 17c987d18621 -p 5601:5601 kibana:7.6.2
```

​		`e75e146b116b`为es的容器ID。

​		实际会发现kibana启动失败，那是因为配置连接es的地址不对。进入终端修改。

```shell
docker exec -t -i hardcore_agnesi /bin/bash
```

```shell
vi config/kibana.yml
```

​		将elasticsearch.hosts修改为本机IP的ES，IP使用本机的wsl的IP也行。（ip通过ipconfig在win终端下）

```yaml
elasticsearch.hosts: [ "http://192.168.43.214:9200" ]
```

​		之后重启kibana即可。

