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
