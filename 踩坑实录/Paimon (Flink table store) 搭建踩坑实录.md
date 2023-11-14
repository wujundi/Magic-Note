# Paimon 踩坑实录

* 官方网站 [Apache Paimon](https://paimon.apache.org/)
* 到目前的稳定版本文档 [Flink | Apache Flink Table Store](https://nightlies.apache.org/flink/flink-table-store-docs-release-0.3/docs/engines/flink/)
* 在集群上找到 flink 的安装目录，首先 whereis flink，然后 which flink，就可以知道究竟 flink 命令是在执行哪个文件了，这个文件一般都是一个链接，相当于windows中的快捷方式，这里就顺藤摸瓜，找到最后实际的目录是 /usr/bigtop/3.2.0/usr/lib/flink。
* 重新在 bigtop3.2.0的编译结果里面确定了flink的版本是 1.15.3，从 [Flink | Apache Flink Table Store](https://nightlies.apache.org/flink/flink-table-store-docs-release-0.3/docs/engines/flink/) 下载了 flink 1.15 对应的jar包，放在了 /usr/bigtop/3.2.0/usr/lib/flink/lib
* 额，然后好像就完事了，可以测试了？吗？

## 用 API 的方式太原始了，我需要一个 Flink sql 的 ui 提交页面

* 参见 StreamPark 踩坑实录

## 哈哈，我又回来了，StreamPark 已成，Flink SQL 也已经调通了，现在再来搞 Paimon

今非昔比了哥，现在 Paimon 已经正式独立了，文档地址 https://paimon.apache.org/docs/0.5/

---

从这里下载 jar 包 https://paimon.apache.org/docs/0.5/engines/flink/

妹的，下载贼慢，我自己从 github 下载源码编译吧。

编译好了，把 /opt/NOAH_source_reference/incubator-paimon-release-0.5.0-incubating/paimon-flink/paimon-flink-1.15/target/paimon-flink-1.15-0.5.0-incubating.jar 上传到 HDFS

---

然后对照着 https://paimon.apache.org/docs/0.5/engines/flink/ 和 https://paimon.apache.org/docs/0.5/concepts/file-operations/ 来进行操作

阿里云这边也有个文档，主要是针对云上环境的，胜在每一步是在干什么比较详细 https://www.alibabacloud.com/help/zh/emr/emr-on-ecs/user-guide/integrate-paimon-with-flink?spm=a2c63.p38356.0.0.658b1a45Ld7v5T

还有一些民间博客，https://manor.blog.csdn.net/article/details/132001087 和 https://blog.csdn.net/weixin_42354436/article/details/131634720，胜在参数可借鉴吧

---

在 sql-client 里面执行

CREATE CATALOG paimon_catalog WITH (
    'type'='paimon',
    'warehouse'='hdfs://noah:8020/paimon/paimon_catalog'
);

遇到报错 java.lang.ClassNotFoundException: org.apache.hadoop.hdfs.HdfsConfiguration

我把 hadoop-hdfs-3.3.4.jar 拷贝到了 /usr/bigtop/3.2.0/usr/lib/flink/lib/ 下面再试一下，还是不行

从 bigtop 里面捞源码看吧 docker run -itd --name='bigtop320' --privileged -p 2929:2929 registry.cn-hangzhou.aliyuncs.com/wujundi/centos-noah:bigtop-repo-ready-for-http /usr/sbin/init

还真找到了 /home/bigtop-3.2.0/dl/hadoop-3.3.4-src/hadoop-hdfs-project/hadoop-hdfs-client/src/main/java/org/apache/hadoop/hdfs/HdfsConfiguration.java，对应的模块是 hadoop-hdfs-client

把 /usr/bigtop/3.2.0/usr/lib/hadoop-hdfs/hadoop-hdfs-client-3.3.4.jar 复制到了 /usr/bigtop/3.2.0/usr/lib/flink/lib/hadoop-hdfs-client-3.3.4.jar，然后重新启动 sql-client 创建 catalog，报错换了

---

新的报错是 java.lang.ClassNotFoundException: com.ctc.wstx.io.InputBootstrapper

尝试1：查了一下 https://blog.csdn.net/love_Java123/article/details/89711237 可能是这个 woodstox-core-\**.*\*.\*.jar

又看到了 https://www.modb.pro/db/436117，猜测着大概是 hadoop 那边需要的jar包，最终在 /usr/bigtop/3.2.0/usr/lib/hadoop/lib/woodstox-core-5.3.0.jar 这里找到了，拷贝到 。然后重新启动 sql-client 创建 catalog，报错换了

---

尝试2：https://www.modb.pro/db/436117 这个里面说可能是 HADOOP_CLASSPATH 设置的问题，我又检查了一下 root/.bashrc 和 /etc/profile，还真是，没有配置过 HADOOP_CLASSPATH 变量，而且缺的那个 jar 就在这里 /usr/bigtop/3.2.0/usr/lib/hadoop/lib/woodstox-core-5.3.0.jar，所以我就在 /etc/profile 添加了一行 exportHADOOP_CLASSPATH=/usr/bigtop/3.2.0/usr/lib/hadoop/lib，重新 source /etc/profile 再执行 sql-client.sh

不过好像并没有生效，难道是需要重启 flink 服务才能生效么？重启了也还是一个鸟样子，服气了，还是走尝试1吧

---

新的报错是 java.lang.ClassNotFoundException: org.codehaus.stax2.XMLInputFactory2。往上查到 https://blog.csdn.net/lzh754413563/article/details/107394607，看起来和刚刚那个问题类似，从 /usr/bigtop/3.2.0/usr/lib/hadoop/lib/stax2-api-4.2.1.jar 拷贝到 /usr/bigtop/3.2.0/usr/lib/flink/lib，再试一次

---

新的报错 java.lang.ClassNotFoundException: org.apache.hadoop.thirdparty.com.google.common.base.Precondition，通过 https://www.freesion.com/article/34651309978/ 看，好像是缺少 guava 的 jar，把 /usr/bigtop/3.2.0/usr/lib/hadoop/lib/guava-27.0-jre.jar 复制过去之后，没有什么变化。

我注意到这个类不是单纯的 guava，而是 hadoop.thirdparty 下面的，那么，会不会是 /usr/bigtop/3.2.0/usr/lib/hadoop/lib/hadoop-shaded-guava-1.1.1.jar 呢？果然啊，果然，报错又换了

---

新的报错是 java.lang.ClassNotFoundException: org.apache.commons.configuration2.Configuration，参考了 https://blog.csdn.net/qq_43408367/article/details/129476937，应该是需要 commons-configuration 这个 jar

---

新的报错是 java.lang.ClassNotFoundException: org.apache.hadoop.util.PlatformName，参考 https://blog.csdn.net/Xgx120413/article/details/51889743，应该是需要 hadoop-auth

---

新的报错是 java.lang.ClassNotFoundException: org.apache.commons.logging.LogFactory，参考 https://blog.csdn.net/Leoon123/article/details/125816907，应该是需要 commons-logging

---

新的报错是

[ERROR] Could not execute SQL statement. Reason:
org.apache.flink.table.api.ValidationException: Unable to create catalog 'paimon_catalog'.

Catalog options are:
'type'='paimon'
'warehouse'='hdfs://noah:8020/paimon/paimon_catalog'

同一个 sql session 再次执行语句时，报错换成了  java.lang.NoClassDefFoundError: Could not initialize class org.apache.hadoop.fs.FileSystem

hadoop 源码的类是 /home/bigtop-3.2.0/dl/hadoop-3.3.4-src/hadoop-common-project/hadoop-common/src/main/java/org/apache/hadoop/fs/FileSystem.java，看起来需要 hadoop-common 的 jar，可是 /usr/bigtop/3.2.0/usr/lib/flink/lib/hadoop-common-3.3.4.jar 已经在了呢。

那么这个 ValidationException 是指用户不对么？换 hdfs 用户再试一下。不行一个鬼样子。

那我先在HDFS里面创建这个目录再试一下。结果还是一样的报错 。

真猜不出来了，找一下 log 吧，/etc/flink/conf/flink-conf.yaml 这里记录了 env.log.dir: /var/log/flink，去看看

具体在 /var/log/flink/flink-root-sql-client-noah.log

看起来具体的报错是 Caused by: java.lang.ClassNotFoundException: Class org.apache.flink.fs.shaded.hadoop3.org.apache.hadoop.security.JniBasedUnixGroupsMappingWithFallback not found

在 flink 的源码里面搜索，找到了 /opt/NOAH_source_reference/flink-1.15.3_md/flink-filesystems/flink-fs-hadoop-shaded/src/main/resources/core-default-shaded.xml 这里，把对应的 jar 包 /opt/NOAH_source_reference/flink-1.15.3_md/flink-filesystems/flink-fs-hadoop-shaded/target/flink-fs-hadoop-shaded-1.15.3.jar 拷贝到 /usr/bigtop/3.2.0/usr/lib/flink/lib，试一下

而且发现了一个更要命的，/var/log/flink/flink-root-sql-client-noah.log 好像只针对部分报错才会打日志，手动清空其内容之后，迟迟没有新的日志，所以，刚刚看到的那个 Caused by: java.lang.ClassNotFoundException: Class org.apache.flink.fs.shaded.hadoop3.org.apache.hadoop.security.JniBasedUnixGroupsMappingWithFallback not found 的报错，也不是当前的报错，而是之前尝试引入 flink-fs-hadoop-shaded-1.15.3 之后产生的报错。

---

整一个不知所措，重启一下试试吧，还是老样子。

真的是没办法了，直接上线试试，看看通过 YARN 的日志能不能看到具体的报错原因

---

CREATETABLEifnotexists test_kafka_sink_1 (

  id int

  ,content STRING

  ,create_time TIMESTAMP

) WITH (

  'connector'='kafka',

  'topic'='test_topic_1',

  'properties.bootstrap.servers'='noah:9092',

  -- 'properties.security.protocol' = 'SASL_PLAINTEXT',

  -- 'properties.sasl.mechanism' = 'GSSAPI',

  -- 'properties.sasl.kerberos.service.name' = 'kafka',

  'properties.group.id'='flink_paimon_test',

  'scan.startup.mode'='earliest-offset',

  'value.format'='debezium-json'-- 接 flink-cdc 不能用 json，必须指定为 debezium-json

);

CREATECATALOG paimon_catalog WITH (

    'type'='paimon'

    ,'warehouse'='hdfs://noah:8020/paimon/paimon_catalog'

);

USECATALOG paimon_catalog;

-- create a word count table

CREATETABLEifnotexists paimon_test_db.paimon_test_table (

    id intPRIMARYKEYNOT ENFORCED

    ,content STRING

    ,create_time TIMESTAMP

);

-- paimon requires checkpoint interval in streaming mode

SET'execution.checkpointing.interval'='10 s';

-- write streaming data to dynamic table

INSERTINTO paimon_test_table

SELECT  id

    ,content

    ,create_time

  FROM default_catalog.default_database.test_kafka_sink_1

;

---

报错 Caused by: org.apache.flink.table.catalog.exceptions.CatalogException: Paimon Catalog only supports paimon tables, but you specify  'connector'= 'kafka' when using Paimon Catalog，调整 kafka table 的位置

---

报错 Caused by: org.apache.flink.table.catalog.exceptions.TableAlreadyExistException: Table (or view) default.paimon_test_table already exists in Catalog paimon_catalog. 也算一个好消息，看来 paimon_catalog 是已经建成了，hadoop fs -ls /paimon/paimon_catalog/ 一下发现确实，已经有文件了 /paimon/paimon_catalog/default.db。sql 里面增加一下 if not exists 应该就好了。

---

报错 Caused by: org.apache.calcite.runtime.CalciteContextException: From line 4, column 8 to line 4, column 24: Object 'test_kafka_sink_1' not found，因为中途换了 catalog，所以不认识 test_kafka_sink_1 了，from 这个地方改成了 FROM default_catalog.default_database.test_kafka_sink_1 这样的形式。

---

还真是环境问题，放到线上跑。。。就OK了。

---

读取 paimon 数据，写入 doris

-- switch to streaming mode
SET 'execution.runtime-mode' = 'streaming';
-- enable checkpoint
SET 'execution.checkpointing.interval' = '5s';

-- 支持同步insert/update/delete事件
CREATE TABLE doris_sink (
  id int
  ,content STRING
  ,create_time TIMESTAMP
)
WITH (
  'connector' = 'doris',
  'fenodes' = '127.0.0.1:18030',
  'table.identifier' = 'test_db.doris_test_table',
  'username' = 'root',
  'password' = '',
  'sink.properties.format' = 'json',
  'sink.properties.read_json_by_line' = 'true',
  'sink.enable-delete' = 'true',  -- 同步删除事件
  'sink.label-prefix' = 'doris_label'
);

CREATE CATALOG paimon_catalog WITH (
    'type'='paimon'
    ,'warehouse'='hdfs://noah:8020/paimon/paimon_catalog'
);

insert into doris_sink
select id
       ,content
       ,null as create_time
  FROM paimon_catalog.paimon_test_db.paimon_test_table
;

---

报错：Caused by: org.apache.calcite.sql.validate.SqlValidatorException: Object 'paimon_catalog' not found

难道还要回回都显示的创建 catalog 么？

CREATECATALOG paimon_catalog WITH (

    'type'='paimon'

    ,'warehouse'='hdfs://noah:8020/paimon/paimon_catalog'

);

---

通车仪式，撒花庆祝！！！！

伴随着第一条数据从接口采集入mysql，通过 flink cdc 流经 kafka、经过 paimon、最终落入 doris，忙活了大半年的个人项目终于完成了“通车仪式”！！！

踩坑无数，整晚与这茫茫多的失败日志为伍。

后面整理一下使用文档，把它发到 github 上，希望可以帮助到更多的人。

自此，流式数仓的“硬件”已经齐备，接下来就是我 sql boy 的老本行了。

龙珠已经集齐，我要开始召唤神龙了！！！
