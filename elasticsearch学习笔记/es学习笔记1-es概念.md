# es学习笔记1-es概念

> 书：《[Learning Elastic Stack 7.0 : distributed search, analytics, and visualization using Elasticsearch, Logstash, Beats, and Kibana](https://zh.pb1lib.org/book/11034550/bfdc9f)》

## 简介

​		为了应付大量数据，且传统的关系型数据库无法存储。尤其是全是搜索和分析的应用和BI(business intelligence )应用。

​		es生态组件有：Kibana, Logstash, Beats, X-Pack, and Elasticsearch.其中Elasticsearch是es的心脏，kibina是窗口，logstash和beats是数据导入的帮手。X-pack提供强大的功能，包括监控、告警、安全、图形。

### What is Elasticsearch 

> Elasticsearch is a real-time, distributed search and analytics engine that is horizontally scalable and capable of solving a wide variety of use cases. At the heart of the Elastic Stack, it centrally stores your data so you can discover the expected and uncover the unexpected.

​		es是一个实时的、分布式搜索、计算引擎。可水平扩展，解决大量案例。

### 非严格的数据结构

> Elasticsearch does not impose a strict structure on your data; you can store any JSON documents.JSON documents are first-class citizens in Elasticsearch as opposed to rows and columns in a relational database. A document is roughly equivalent to a record in a relational database table. 

​		es不会严格要求你数据的结构，可以存储任何json文档。 json文档是es的头等公民，而不是关系型数据库的的行、列。

> Often the nature of data is very dynamic, requiring support for new or dynamic columns. JSON documents naturally support this type of data. 

​		自然的数据（结构）往往是动态的。json文档自然的支持这种数据。

### 搜索能力

> The core strength of Elasticsearch lies in its text-processing capabilities. Elasticsearch is great at searching, especially full-text searches. 

​		核心能力在于文本处理能力。es擅长搜索，尤其是全文搜索。

### 分析

> Apart from searching, the second most important functionalstrength of Elasticsearch is analytics. Yes, what was originally known as just a full-text search engine is now used as an analytics engine in a variety of use cases. Many organizations are running analytics solutions powered by Elasticsearch in production.

​		查询能力也被当作是分析引擎，在很多案例中。

### 丰富的客户端支持和rest api

​		超过20种语言的客户端，以及rest api。

### 易于操作、方便扩展

> Horizontal scalability is the ability to scale a system horizontally by starting up multiple instances of the same type rather than making one instance more and more powerful. Vertical scaling is about upgrading a single instance by adding more processing power (by increasing the number of CPUs or CPU cores), memory, or storage capacity. There is a practical limit to how much a system can be scaled vertically due to cost and other factors, such as the availability of higher-end hardware. 

​		水平扩展和垂直扩展都是支持的，传统的数据库往往只能垂直扩展（加CPU、内存、磁盘等配置）。

### 近乎实时查询

> Typically, data is available for queries within a second after being indexed (saved). Not all big data storage systemsare real-time capable. Elasticsearch allows you to index thousands to hundreds of thousands of documents per second and makes them available for searching almost immediately.

​		ES一秒建立索引上千到数十万的文本，并可立即搜索。

### 优化的飞快

> Elasticsearch uses Apache Lucene as its underlying technology. By default, Elasticsearch indexes all the fields of your documents. This is extremely invaluable as you can query or search by any field in your records. You will never be in a situation in which you think, If only I had chosen tocreate an index on this field. Elasticsearch contributors have leveraged Apache Lucene to its best advantage, and thereare other optimizations that make it lightning-fast

​		ES底层技术是 apache lucene。默认将文档中所有属性建立索引。



## 单词记录

- emergence 出现
- massive 大量的
- technology 技术
- ecosystem 生态系统
- visualization 形象化（图形化）
- production-ready 生产就绪
- tremendous 巨大的
- underlying technology 底层技术
- advantage 优点

