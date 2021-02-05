# skywalking02 - skywalking安装

- skywalking的安装教程在网上已经很多了,我觉得没必要重复写,分普通安装和docker安装,以及常用配置进行介绍\引入链接.建议按照官网进行安装.

## 官网文档

https://github.com/apache/skywalking/tree/master/docs

## 整体安装步骤

1. 安装分析后端OAP
   - 需先安装好持久层的组件,或使用H2内存数据库
2. 安装UI界面
3. 安装Agent
   - 先准备好一个被代理的应用

## 普通安装

[skywalking学习之路---skywalking环境从零搭建部署](https://www.cnblogs.com/jackion5/p/10604189.html)

## docker安装

[Docker部署SkyWalking](https://blog.csdn.net/jamel_litoo/article/details/109771940#DockerSkyWalking_0)

## 后端配置文件简单介绍

- 配置有很多,这里只介绍和部署相关的一部分.

- 配置文件位于 config/下,文件名为application.yml

### 集群配置

[官网集群部署文档](https://github.com/apache/skywalking/blob/master/docs/en/setup/backend/backend-cluster.md)

  ```yaml
  cluster:
    standalone: #单击部署
    #zookeeper: #zookeeper集群部署
    #kubernetes: #kubernetes集群部署
    # ... 省略许多,集群部署集群的方式有很多
  ```

### 存储模块配置
[官网存储模块配置文档](https://github.com/apache/skywalking/blob/master/docs/en/setup/backend/backend-storage.md)

- 默认是es作为存储模块,为了方便实验,也可以选择h2的形式.也可以在官网文档中看具体的es的配置.当然,像MySQL\TiDB也是支持的

```yaml
storage:
  selector: ${SW_STORAGE:h2}
  h2:
    driver: org.h2.jdbcx.JdbcDataSource
    url: jdbc:h2:mem:skywalking-oap-db
    user: sa
```

  

### IP端口配置

[官网IP端口配置文档](https://github.com/apache/skywalking/blob/master/docs/en/setup/backend/backend-ip-port.md)

```yaml
core:
  default:
    restHost: 0.0.0.0
    restPort: 12800
    restContextPath: /
    gRPCHost: 0.0.0.0
    gRPCPort: 11800
```


