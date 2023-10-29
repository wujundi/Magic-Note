# Flink SQL 踩坑实录

这里需要看一下 flink SQL 读写 kafka 的情况

---

第一版本写入程序我是这样写的，但是报错了，但是暂时我还没有找到具体的报错在哪里

CREATETABLEifnotexists test_kafka_sink_1 (

  id int

  ,content STRING

  ,create_time TIMESTAMP

) WITH (

  'connector'='kafka',

  'topic'='test_topic_1',

  -- 'properties.bootstrap.servers' = 'indata-192-168-44-128:6667',

  -- 'properties.security.protocol' = 'SASL_PLAINTEXT',

  -- 'properties.sasl.mechanism' = 'GSSAPI',

  -- 'properties.sasl.kerberos.service.name' = 'kafka',

  -- 'properties.group.id' = 'dkl',

  -- 'scan.startup.mode' = 'earliest-offset',

  'value.format'='json'

);

insertinto test_kafka_sink_1

select id

    ,content

    ,create_time

  from cdc_mysql_source

在 FLINK completed jobs 里面是没有找到的，看起来是在 YARN 这里就出问题了

从 yarn 的 application 页面里面来看，说是 Failing this attempt.Diagnostics: [2023-10-23 11:07:04.710]Exception from container-launch.

在 jobmanager.log 里面找到了原因，链接是 [noah:19888/jobhistory/logs/noah:45454/container_1698048918289_0002_01_000001/container_1698048918289_0002_01_000001/hdfs/jobmanager.log/?start=0&amp;start.time=0&amp;end.time=9223372036854775807](http://noah:19888/jobhistory/logs/noah:45454/container_1698048918289_0002_01_000001/container_1698048918289_0002_01_000001/hdfs/jobmanager.log/?start=0&start.time=0&end.time=9223372036854775807)

具体的原因是 Caused by: org.apache.flink.table.api.ValidationException: Cannot discover a connector using option: 'connector'='kafka'

Caused by: org.apache.flink.table.api.ValidationException: Could not find any factory for identifier 'kafka' that implements 'org.apache.flink.table.factories.DynamicTableFactory' in the classpath.
Available factory identifiers are:

blackhole
datagen
doris
filesystem
mysql-cdc

看起来是因为 没有认出来 'connector'='kafka',

在 flink 源码里面找了一下，发现有这么一个模块，/opt/NOAH_source_reference/flink-1.15.3_md/flink-connectors 这下面就有 flink-connector-kafka

老规矩，找 jar 包，传 HDFS。然后重启应用再试试


---

新的报错是 Caused by: org.apache.flink.table.api.ValidationException: One or more required options are missing. Missing required options are: properties.bootstrap.servers。

看起来上面的那个问题解决了，那这回查一下 properties.bootstrap.servers 是什么，看起来好像是要指定kafka的服务器和端口.

加上 'properties.bootstrap.servers'='127.0.0.1:9092' 这个参数之后，再次启动

---

新的报错 

```
Caused by: org.apache.flink.client.program.ProgramInvocationException: The main method caused an error: Table sink 'default_catalog.default_database.test_kafka_sink_1' doesn't support consuming update and delete changes which is produced by node TableSourceScan(table=[[default_catalog, default_database, cdc_mysql_source]], fields=[id, content, create_time])

Caused by: org.apache.flink.table.api.TableException: Table sink 'default_catalog.default_database.test_kafka_sink_1' doesn't support consuming update and delete changes which is produced by node TableSourceScan(table=[[default_catalog, default_database, cdc_mysql_source]], fields=[id, content, create_time])

```

在 https://blog.csdn.net/weixin_42764556/article/details/115630239 这里大概找到了答案，错误1：doesn't support consuming update and delete changes which is produced by node TableSourceScan
解答：flink1.11之后引入了CDC（Change Data Capture，变动数据捕捉）阿里大神开源的，此次错误是因为Source源是mysql-cdc所以获取的数据类型为Changelog格式，所以在WITH kafka的时候需要指定format=debezium-json。

改完之后开始运行，这个问题得以解决。

---

那么问题就来到了下一步，如何看到 topic 里面的数据状态呢？好像看不到，那么就直接往下进行吧


---

下面来到了 flink sql 读取 kafka 的环节，读取 kafka 之后可以写入 doris

新建了 doris 表，新建了一个 kafka_2_doris 的任务

启动任务的时候，链接超时，YARN 管理页面也报出资源问题 Application is added to the scheduler and is not yet activated. Queue's AM resource limit exceeded，看来现在默认队列不行，还是得调整

检擦了一下 YARN 的信息，最大 46G，目前已经起了两个应用，占用 30G，这和 Minimum Container Size (Memory) 15G 是吻合的，所以怀疑是 YARN 按照 Minimum Container Size (Memory) 分配资源，导致容器最多只能运行 45/15 = 3 个应用，我调小一下试试。

重启之后，集群资源降低了。

重新启动任务，YARN占用内存总资源大概在 6G左右，此问题得到解决

---

启动之后继续报错 Caused by: org.apache.calcite.sql.validate.SqlValidatorException: No match found for function signature get_json_object(`<CHARACTER>`, `<CHARACTER>`)  看来是不认这个函数。查看了 https://developer.aliyun.com/ask/543695 之后，决定尝试换成 

JSON_EXTRACT 试试。坑，JSON_EXTRACT，也不行。从 https://ost.51cto.com/posts/17088 这里面得到了一个 MODULE 的概念，可以尝试引入 hive MODULE. 按照文章中的说法，我去 flink connector 模块找到了 /opt/NOAH_source_reference/flink-1.15.3_md/flink-connectors/flink-sql-connector-hive-3.1.2/target/flink-sql-connector-hive-3.1.2_2.12-1.15.3.jar ，看一下这个是不是。

要调试的话，还有点点麻烦，我先找找命令行怎么直接启动 flink sql client，bash 这个文件就可以 /usr/bigtop/3.2.0/usr/lib/flink/bin/sql-client.sh 测试了一下 load 模块的指令，也没啥问题。

现在的报错是 Caused by: java.lang.ClassNotFoundException: org.apache.hadoop.mapred.JobConf，查了一下 ，[flink sql org.apache.hadoop.mapred.JobConf_mob649e8157aaee的技术博客_51CTO博客](https://blog.51cto.com/u_16175447/7257465) 这里的说法是这个类是配置参数用的，在这里 [Flink 集成 Hive 踩坑过程总结（持续更新） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/545472819?utm_id=0) 找到了更多的信息，我也去找找 hadoop-mapreduce-client-core-*.*.*.jar，在 /usr/bigtop/3.2.0/usr/lib/hadoop/client/hadoop-mapreduce-client-core-3.3.4.jar 这里发现了，上传 HDFS，牛逼，报错真的变了，这个问题就算解决了。

---

新的报错是 Caused by: org.apache.flink.table.api.TableException: Cannot find table '`default_catalog`.`default_database`.`doris_test_table_1`' in any of the catalogs [default_catalog], nor as a temporary table.

好像是没有识别到 doris 表的库名。仔细一看，是 flink sql 里面定义的 sink 表和 insert 语句的 sink 表，表名没写成一样。

---

新的报错 Caused by: org.apache.flink.client.program.ProgramInvocationException: The main method caused an error: Property group.id is required when using committed offset for offsets initializer

在 sql 中 kafka 表的定义部分增加了 'properties.group.id'='flink_test_123456', 的参数

---

新报错 Caused by: java.lang.RuntimeException: Failed to get metadata for topics [test_topic_1].

Caused by: org.apache.flink.kafka.shaded.org.apache.kafka.common.errors.TimeoutException: Timed out waiting for a node assignment. Call: describeTopics。上网搜索找到了 [Flink读取kafka数据报错-CSDN博客](https://blog.csdn.net/QYHuiiQ/article/details/131544485) 这个。所以我在 ambari 这里做了修改，change listeners from PLAINTEXT://localhost:9092 to PLAINTEXT://127.0.0.1:9092

然后，任务倒是启动了，但是 doris 里面迟迟没有数据呀，而且 EFAK 也显示不出现有 topic 了，看起来 PLAINTEXT://127.0.0.1:9092 的影响还挺大，然后我换成了 PLAINTEXT://noah:9092，EFAK显示正常，Topic test_topic_1 的页面能看到数据抽样，说明写入 kafka 生效了。

doris 里面也有了，看起来数据的更新周期和check的时间间隔有关，现在设置的是 SET'execution.checkpointing.interval'='5s';
