# EMR / Hadoop


## Hadoop 生态

### 存储层

- HDFS = 分布式文件存储系统，如果可以提供兼容的 API 也可以把别的存储介质和协议映射成 HDFS（比如 S3）

### 数据界面

- Hive = SQL 引擎，用 SQL 查询 HDFS 上的数据库和表，这些数据库和表以文件形式存在
- Pig = 高阶数据处理语言（类比 SQL）引擎
- Presto = 分布式 SQL 查询引擎，类似 Hive，但不限于 Hadoop，也可以查 MySQL 等其他数据库

### 数据库

- HBase = 基于 HDFS 的 NoSQL 数据库，由 Google BigTable 发展而来

### 数据搬迁

- Sqoop = 批量把关系型数据库中
- Flink = 流式数据处理

### 执行引擎

- MapReduce = 老式执行引擎
- Tez = 执行引擎
- Spark = 基于内存的执行引擎；常存在于 Hadoop 生态内，但本身不依赖 Hadoop

### 编排引擎

在编排引擎中经常会看到「DAG」这个词，全称是「Directed Acyclic Graph」，即「有向非循环图」。「Directed」 指的是有方向，或者说有先后顺序，比如要等 A 步骤执行完了再执行 B，再执行 C 步骤。「Acyclic」指的是步骤之间不形成「循环」，比如 A 执行完到 B，到 C，然后又回到 A，就形成了循环，这种循环在 DAG 中不允许。

编排引擎通常使用 DAG 的模式来组织工作流，整体的逻辑会比较简单。

- Oozie = 大概可以理解为 Hadoop 生态的 `cron`，此外也增加了很多工作流编排的功能，比如先做什么，再做什么，出错了怎么重试，哪里再循环一下，最后产出一个什么结果，诸如此类
  - Airflow = Airbnb 推出的编排引擎，主要应用于数据在不同系统和组件之间的搬迁，不依赖 Hadoop
  - NiFi = 与 Airflow 类似；不依赖于 Hadoop

### 流式引擎

通常来说，我们会把处理引擎分成批处理（Batch）和流式处理（Stream）。两种引擎的运行方式，优化逻辑和支持的输入输出方式都有所不同。

现在比较新潮的看法（大概从 Flink 开始）是世界上只有一种数据，只不过有的「有边界」（Bounded）有的「没有边界」（Unbounded）。有边界的数据起点和终点，类似一个文件或者数据库，而所谓没有边界的数据，实际上就是持续产出的流式数据。

现在，大部分流式引擎都可以处理有边界和没有边界的数据。

- Flink = 从德国几所大学的研究项目中催生出来的开源项目，类似 Spark 的处理引擎，但是更倾向于流式处理，对流式数据的支持更好；不依赖 Hadoop
- Kafka = LinkedIn 推出的「发布-订阅」（Pub-sub）式数据流，主打超高性能，适合需要高效收集海量数据的场景，比如收集 IoT 数据、API 调用日志或者运营指标等；不依赖 Hadoop
- Storm = Twitter 推出的流式处理引擎；不依赖 Hadoop


## CDP、CDH、HDP

下面这些东西指的是一个东西。

- CDP = Cloudera Data Platform
- CDH = Cloudera Distributed Hadoop
- HDP = Hortonworks Data Platform

这个发行版基于 Hadoop 做了一些安全和性能上的增强，包含了特定的一些应用。Cloudera 和 Hortonworks 两家公司已经合并，然后 CDH 和 HDP 合并，目前称作 CDP。