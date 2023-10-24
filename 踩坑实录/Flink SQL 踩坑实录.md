# Flink SQL 踩坑实录

这里需要看一下 flink SQL 读写 kafka 的情况

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
