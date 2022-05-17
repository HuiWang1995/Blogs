# es学习笔记2-es组件

> es生态组件简介
>
> The Elastic Stack components are shown in the following diagram. It is not necessary to include all of them in your solution. Some components are general-purpose and can be used outside the Elastic Stack without using any other components.
>
> 组件非必须使用，有些可单独独立使用在堆栈外。

![image-20220321194414906](pic\2\1.png)

## Elasticsearch

​		Elasticsearch is at the heart of the Elastic Stack. It stores all your data and provides search and analytic capabilities in a scalable way.  

## Logstash
​		Logstash helps centralize event data such as logs, metrics, or any other data in any format. It can perform a number of transformations before sending it to a stash of your choice. It is a key component of the Elastic Stack, used to centralize the collection and transformation processes inyour data pipeline.

​		Logstash is a server-side component. Its role is to centralize the collection of data from a wide number of inputsources in a scalable way, and transform and send the data to an output of your choice. Typically, the output is sent to Elasticsearch, but Logstash is capable of sending it to a wide variety of outputs. Logstash has a plugin-based, extensible architecture. It supports three types of plugin: input plugins, filter plugins, and output plugins. Logstash has a collection of 200+ supported plugins and the count is ever increasing.
​		Logstash is an excellent general-purpose data flow engine that helps in building real-time, scalable data pipelines.

​		logstash 是服务器端的组件，用来收集数据，可以做转换。它是插件，可扩展的架构，有输入、过滤、输出类型的插件，现在已经有200多中插件，且还在增加。是一个通用的数据流引擎。

## Beats
​		Beats is a platform of open source lightweight data shippers. Its role is complementary to Logstash. Logstash is a server-side component, whereas Beats has a role on the client side. Beats consists of a core library, libbeat, which provides an API for shipping data from the source, configuring the input options, and implementing logging. Beats is installed on machines that are not part of server-side components such as Elasticsearch, Logstash, or Kibana. These agents reside on non-cluster nodes, which are sometimes called edge nodes.

​		Beats是一个开源的轻量数据托运人插件平台。与Logstash形成互补，它往往装在边缘节点，进行搬运数据。官方的Beats有：Packetbeat, Filebeat, Metricbeat, Winlogbeat, Audiobeat, Heartbeat. 



## Kibana
​		Kibana is the visualization tool for the Elastic Stack, and can help you gain powerful insights about your data in Elasticsearch. It is often called a window into the Elastic Stack. It offers many visualizations including histograms, maps, line charts, time series, and more. You can build visualizations with just a few clicks and interactively explore data. It lets you build beautiful dashboards by combining different visualizations, sharing with others, and exporting high-quality reports.
​		Kibana also has management and development tools. You can manage settings and configure X‑Pack security features for Elastic Stack. Kibana also has development tools that enable developers to build and test REST API requests.

​		Kibana是可视化工具，提高我们的数据洞察力。可以通过少量的交互式点击创建图表。同时它有管理、开发工具，并且可以配置X-Pack安全功能。

## X-Pack

​		X-Pack adds essential features to make the Elastic Stack production-ready. It adds security, monitoring, alerting, reporting, graph, and machine learning capabilitiesto the Elastic Stack.

### Security
​		The security plugin within X-Pack adds authentication and authorization capabilities to Elasticsearch and Kibana so that only authorized people can access data, and they can only see what they are allowed to. 

The security extension also lets you configure fields and document-level security with the licensed version.

​		登录权限、数据权限限制。可以精确到属性、文档级别的许可证版本。

### Monitoring
​		You can monitor your Elastic Stack components so that there is no downtime. The monitoring component in X-Pack lets you monitor your Elasticsearch clusters and Kibana.
​		You can monitor clusters, nodes, and index-level metrics. The monitoring plugin maintains a history of performance so you can compare current metrics with past metrics. It also has a capacity planning feature.

​		监控ES、Kibana。可以看到历史记录，并且有容量规划能力。

### Alerting
​		X-Pack has sophisticated alerting capabilities that can alert you in multiple possible ways when certain conditions are met. It gives tremendous flexibility in terms of when, how, and who to alert.

### Graph
​		Graph lets you explore relationships in your data. Data in Elasticsearch is generally perceived as a flat list of entities without connections to other entities. This relationship opens up the possibility of new use cases. 

​		Graph consists of the Graph API and a UI within Kibana, that let you explore this relationship. 

​		Graph可以探索数据之间的关系。由Graph API 和Kibaba的UI组成。

### Machine learning
​		X-Pack has a machine learning module, which is for learning from patterns within data. Machine learning is a vast field that includes supervised learning, unsupervised learning, reinforcement learning, and other specialized areas such as deep learning. The machine learning module within X-Pack is limited to anomaly detection in time series data, which fallsunder the unsupervised learning branch of machine learning.

​		X-Pack 的机器学习模块有多种领域，但限制在检测时序数据的异常。

## Elastic Cloud
​		Elastic Cloud is the cloud-based, hosted, and managed setup of the Elastic Stack components. The service is provided by Elastic (https://www.elastic.co/), which is behind the development of Elasticsearch and other Elastic Stack components. All Elastic Stack components are open source except X-Pack (and Elastic Cloud). Elastic, the company, provides services for Elastic Stack components including training, development, support, and cloud hosting.

​		ES的服务提供商提供的云服务，包含培训、开发、运维等服务。（X-Pack不开源）。



## 单词记录

- centralize  集中
- stash 存放;储藏
- server-side 服务器端
- general-purpose 通用的
- shippers 承运商，托运人
- complementary 互补的
- edge nodes 边缘节点
- insights 洞察力
- interactively 交互地
- sophisticated 复杂的
- tremendous 极好的
- perceived 被...视为
- anomaly 异常事物

