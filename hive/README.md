# Hive 2

## 介绍

### 前提

这篇介绍主要适用于 EMR 5.x（包含 Hive 2）。

EMR 6.x 版本包含 Hive 3，其配置和使用方式可能会有不同。

### SQL 引擎

Hive 是基于 HDFS 的一套 SQL 引擎。它把 HDFS 上的文件映射成数据库，并且让用户可以通过 SQL 语句查询。SQL 语句会被转译成底层的数据引擎（MapReduce、Tez 或者 Spark）的任务并执行，在执行完成后汇总返回结果。

这让用户可以用熟悉的 SQL 语言来在 HDFS 上存储和查询海量数据，提供了比较好的用户体验。

### 核心组件

Hive 的核心组件：

- 元数据存储服务（Metastore），用于操作元数据库，元数据库保存数据库、表结构、外部文件地址等，元数据可能保存在集群中，也可能保存在集群外（单独建立一个 MySQL，或者使用 Glue）
- 文件系统操作服务（File System），用于操作 HDFS 或兼容的文件系统（如 S3）
- 执行引擎服务（Execution Engine），用于把 SQL 语句转换成实际的分布式任务，对接 MapReduce、Tez 或 Spark 等引擎
  - 引擎再次把任务转换成 YARN 等资源相关的调度操作
- HiveServer，类似网关，用于把命令传递到 Hive 的各个组件
- Hive CLI，命令行工具
- Beeline，Hive 的 JDBC 驱动

在 Hive 2 里面，[Hive CLI 是一个厚客户端](https://docs.cloudera.com/HDPDocuments/HDP2/HDP-2.6.5/bk_data-access/content/beeline-vs-hive-cli.html)。用户启动的时候会感觉到它启动比较慢。此外，在 Hive CLI 内提交命令，Hive CLI 自身会直接去对接 Metastore 等服务的 API。这意味着要执行命令，Hive CLI 必须运行在可以直接访问到这些服务的环境中。

在 Hive 3 里面，Hive CLI 变成瘦客户端，主要是对接 Beeline，并且由 Beeline 再向 HiveServer 发送命令，最后才到 Metaservice 的服务。这样做的好处，就是 Hive CLI 可以在任意地方运行，只要能通过 JDBC 访问到 HiveServer 就可以进行操作。Hive CLI 变瘦之后启动也会变得很快。

Hue 默认使用 Beeline/JDBC 的方式来访问 Hive，不管对接的是 Hive 2 还是 Hive 3。

### 关于 MapReduce

目前已经不默认也不建议使用 MR 作为执行引擎。如果你尝试使用 `SET hive.execution.engine=mr`，那么有可能会遇到如下问题：

```
Provider org.apache.hadoop.hbase.security.token.AuthenticationTokenIdentifier not found
```

注意：这个错误只能在 Tracker 中看到，在控制台可能会显示通用的错误信息。这可能是因为默认 Hive 并没有引入 HBase 的 JAR 包。也有可能是因为[使用了低版本（5.31 和 5.32）的 EMR 并且打开了 EMR 集群的静态加密](https://docs.amazonaws.cn/en_us/emr/latest/ReleaseGuide/emr-release-5x.html)。

### 关于 Tez

Hive 2 开始默认使用 Tez 作为执行引擎，所以即便在创建 EMR 集群时不单独选择 Tez，也会安装 Tez。使用 `sudo yum list installed | grep tez` 或者 `ls /usr/lib/tez` 可以看到。

### 关于 Spark

Hive 也可以[使用 Spark 作为执行引擎](https://cwiki.apache.org/confluence/display/Hive/Hive+on+Spark%3A+Getting+Started)。不过，需要用户做[非常多的改动](https://stackoverflow.com/questions/56836812/spark-as-execution-engine-with-hive)，包括把 Spark 的 JAR 文件复制或者软连接到 Hive 的目录，并且特定版本的 Hive 只能确保和特定版本的 Spark 工作。

## 使用 Hive CLI（命令行工具）

### 运行 Hive CLI

在 EMR 中 Hive CLI 已经安装，可以直接运行。

```
hive
```

启动时间大概会在数秒到十数秒。如前面所说，这主要是因为在 Hive 2 中 Hive CLI 是厚客户端，会处理很多的逻辑，并且直接对接 Metastore API。

### 删除原来创建过的表

如果原来创建过表，可以先执行下面这个语句把这个表给删除。`DROP TABLE` 不会删除 S3 上的文件和数据。

```sql
DROP TABLE ny_taxi_test;
```

### 创建外部表

接下来我们创建一个外部表，这个表指向一个 S3 上的目录。

```sql
CREATE EXTERNAL TABLE ny_taxi_test (
	vendor_id int,
	lpep_pickup_datetime string,
	lpep_dropoff_datetime string,
	store_and_fwd_flag string,
	rate_code_id smallint,
	pu_location_id int,
	do_location_id int,
	passenger_count int,
	trip_distance double,
	fare_amount double,
	mta_tax double,
	tip_amount double,
	tolls_amount double,
	ehail_fee double,
	improvement_surcharge double,
	total_amount double,
	payment_type smallint,
	trip_type smallint
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
STORED AS TEXTFILE
LOCATION "s3://<YOUR-BUCKET>/input/";
```

注意创建外部表的时候 `LOCATION` 只能接受一个目录不能是文件。目录下的文件都会被扫描。

另外这个语句会直接把表创建在 `default` 数据库下。在没有用 `USE database_name` 选择数据库的情况下，默认会针对 `default` 数据库下的表进行操作。

如果选择了使用 Glue 作为元数据存储，则 Hive 上针对数据库和表的创建、编辑等操作都会翻译成 Glue 的 API 调用。这些 API 调用受制于 Glue 和 Lake Formation 的权限体系。我们可以在 Glue 控制台上看到我们创建的外部表，删除表时也会将其从 Glue 中删除。

### 查询

我们可以使用正常 SQL 语句进行查询。

```sql
SELECT DISTINCT rate_code_id FROM ny_taxi_test;
```

## 常用操作

### 输出 DEBUG 日志

`export HADOOP_ROOT_LOGGER=DEBUG,console`

### 关闭 DEBUG 日志

`export HADOOP_ROOT_LOGGER=WARN,console`

### 打开 Hive CLI 日志

`hive --hiveconf hive.root.logger=INFO,console`

## 参见

- [Hive 3 的一些架构变化](https://docs.cloudera.com/HDPDocuments/HDP3/HDP-3.1.0/hive-overview/content/hive-apache-hive-3-architectural-overview.html)，主要是：Tez 作为默认引擎，Hive CLI 变瘦，Beeline 作为唯一渠道，Metastore 独立化
- [Hive 常见命令参数](https://cwiki.apache.org/confluence/display/hive/gettingstarted)














