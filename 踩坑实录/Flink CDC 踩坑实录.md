# FLINK CDC 踩坑实录

* 官网地址 [CDC Connectors for Apache Flink® (ververica.github.io)](https://ververica.github.io/flink-cdc-connectors/)
* 文档地址 [CDC Connectors for Apache Flink® — CDC Connectors for Apache Flink® documentation (ververica.github.io)](https://ververica.github.io/flink-cdc-connectors/master/)，其中快速上手的章节是中英双语各有一份的。
* 下载项目 [Releases · ververica/flink-cdc-connectors (github.com)](https://github.com/ververica/flink-cdc-connectors/releases) 并且编译，这里选择的是 2.4.0 版本，编译过程就是简单的 mvn clean install -Dmaven.test.skip=true。
* flink cdc 官方案例里面是直接写入 ES，NOAH里面暂时没有引入es，所以我准备先直接灌入 doris 看看，这样可以同时测试 【flink cdc】 和 【flink 向 doris 的写入】，参考文章是 [使用 Flink CDC 实现 MySQL 数据实时入 Apache Doris - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/425938765)
* 这个过程当中需要使用到 flink-doris-connector，看了一下 [Apache Doris 系列： 基础篇-Flink SQL写入Doris_flink sql doris_修破立生的博客-CSDN博客](https://blog.csdn.net/weixin_47298890/article/details/127001562)，这里使用 java 代码套壳的 flink sql，其中 pom 文件里面给出了 flink connector 的依赖是这样的，所以我去 doris 项目的编译结果里面找找看，但是没找到
  `<!-- flink-doris-connector -->`
  `<dependency>`
  `<groupId>`org.apache.doris `</groupId>`
  `<artifactId>`flink-doris-connector-1.14_2.12 `</artifactId>`
  `<version>`1.1.0 `</version>`
  `</dependency>`
* 从 doris 的官方文档可以搜索到对于 flink doris connector 的信息，[Flink Doris Connector - Apache Doris](https://doris.apache.org/zh-CN/docs/dev/ecosystem/flink-doris-connector)，其中给出了下载 jar 包的链接 https://repo.maven.apache.org/maven2/org/apache/doris/，下载下来 wget https://repo.maven.apache.org/maven2/org/apache/doris/flink-doris-connector-1.15/1.4.0/flink-doris-connector-1.15-1.4.0.jar，然后放置到 /usr/bigtop/3.2.0/usr/lib/flink/lib/
* 很惊喜的是，doris 的文档[Flink Doris Connector - Apache Doris](https://doris.apache.org/zh-CN/docs/dev/ecosystem/flink-doris-connector) 里面居然有 “使用FlinkSQL通过CDC接入Doris示例”，粘贴到 stream park 上修改之后，提交的时候报错，internal server error: No match found，然后我发现自带的 Flink SQL Demo 任务也报错。

-- enable checkpoint

SET'execution.checkpointing.interval'='10s';

CREATETABLE test_cdc (

    id INT,

    content STRING,

    create_time STRING,

    PRIMARYKEY (id) NOT ENFORCED

  ) WITH (

    'connector'='mysql-cdc',

    'hostname'='localhost',

    'port'='3306',

    'username'='root',

    'password'='mysql',

    'database-name'='spiderflow',

    'table-name'='test_table'

  );

-- 支持同步insert/update/delete事件

CREATETABLE doris_sink_test (

    id INT,

    content STRING,

    create_time STRING,

)

WITH (

  'connector'='doris',

  'fenodes'='127.0.0.1:8030',

  'table.identifier'='database.table',

  'username'='root',

  'password'='',

  'sink.properties.format'='json',

  'sink.properties.read_json_by_line'='true',

  'sink.enable-delete'='true',  -- 同步删除事件

  'sink.label-prefix'='doris_label'

);

insertinto doris_sink_test

select  id

    ,content

    ,create_time

  from test_cdc;

* doris 见表语句可以参考 [数据模型 - Apache Doris](https://doris.apache.org/zh-CN/docs/dev/data-table/data-model)
* 创建的 flink sql 任务代码请见 /opt/noah_flink.sql
* 任务编辑好之后，先 release ，相当于发布；然后 start，启动任务。
* 然后遇到了报错 Caused by: org.apache.flink.client.deployment.ClusterDeploymentException: Couldn't deploy Yarn Application Cluster. Caused by: java.io.FileNotFoundException: File does not exist: hdfs://noah:8020/streampark/flink/flink/plugins 晚上查到其他组件也会有类似的报错，都说是需要在 HDFS 上面创建对应的目录。hadoop fs -ls /streampark/flink/flink 发现，确实没有 plugins 这个目录，su hsfs，然后 hadoop fs -mkdir /streampark/flink/flink/plugins，这回再试试。这回任务顺利启动了。
* 然后还是报错，检查了一下是 cdc 的 jar 包没拷贝过来，这扯不扯，从 /opt/flink-cdc-connectors-release-2.4.0/flink-sql-connector-mysql-cdc/target/flink-sql-connector-mysql-cdc-2.4.0.jar 拷贝到了 /usr/bigtop/3.2.0/usr/lib/flink/lib/flink-sql-connector-mysql-cdc-2.4.0.jar 还是没用
* 看streampark的后台日志，看到 /opt/apache-streampark_2.12-2.1.1-incubating-bin/logs/info.2023-08-09.log 里面打的都是 Deployment took more than 60 seconds. Please check if the requested resources are available in the YARN cluster，去网上找了一下，看到这个 [flink on yarn 模式下提示yarn资源不足问题分析-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1759954)，看起来需要检查和设置一些 YARN 队列的资源。一检查才发现，资源都被 spark 打满了，停掉 spark 之后，队列全腾出来，spark 咋这么有毒？因为 spark 并不是这个环节的主角，所以这里我们先不探究 YARN 队列和 SPARK 队列的适配关系。直接先把 spark 关掉。
* 再次启动，Application Id 和 JobManager URL 都有了，最终 Run Status 还是 FAILED 了，这个是什么原因呢？这次更惨，都没什么日志打出来，费了半天劲，在 YARN 这边找到了应用的详情，[Application application_1691502672438_0012](http://noah:8088/cluster/app/application_1691502672438_0012)，logs 点开有一段是 Log Type: jobmanager.log 下面有一行是 Showing 4096 bytes of 91291 total. Click [here](http://noah:19888/jobhistory/logs/noah:45454/container_1691502672438_0012_01_000001/container_1691502672438_0012_01_000001/hdfs/jobmanager.log/?start=0&start.time=0&end.time=9223372036854775807) for the full log. 点到这个 here 里面去，就到了 [noah:19888/jobhistory/logs/noah:45454/container_1691502672438_0012_01_000001/container_1691502672438_0012_01_000001/hdfs/jobmanager.log/?start=0&amp;start.time=0&amp;end.time=9223372036854775807](http://noah:19888/jobhistory/logs/noah:45454/container_1691502672438_0012_01_000001/container_1691502672438_0012_01_000001/hdfs/jobmanager.log/?start=0&start.time=0&end.time=9223372036854775807)。具体的报错是

```
Caused by: org.apache.flink.table.api.ValidationException: Multiple factories for identifier 'default' that implement 'org.apache.flink.table.delegation.ExecutorFactory' found in the classpath.

Ambiguous factory classes are:

org.apache.flink.table.planner.delegation.DefaultExecutorFactory
org.apache.flink.table.planner.loader.DelegateExecutorFactory
```

* 看了一下，这俩类，一个是 /opt/flink-1.15.3_md/flink-table/flink-table-planner/src/main/java/org/apache/flink/table/planner/delegation/DefaultExecutorFactory.java，另一个是 /opt/flink-1.15.3_md/flink-table/flink-table-planner-loader/src/main/java/org/apache/flink/table/planner/loader/DelegatePlannerFactory.java我之前确实手贱，往 flink 的 lib 目录里面拷贝过 /usr/bigtop/3.2.0/usr/lib/flink/lib/flink-table-planner_2.12-1.15.3.jar 这么个玩意。我先把它名称改掉试试，怎么折腾 lib 目录的 jar 包也没有效果，我看问题不在这里，转而搜索一下，这个 jar 包还在哪里有，find / -name 'flink-table-planner-loader-1.15.3.jar' 发现 /hadoop/yarn/local/filecache/17/flink-table-planner-loader-1.15.3.jar 这里还有一处意外的备份。结果意外的发现 /hadoop/yarn/local/filecache/ 这里面全是yarn缓存的 jar 包，怪不得我修改 lib 没用，合着人家就根本没从那里面拿了。全删掉，重新测。结果已测试，/hadoop/yarn/local/filecache/ 里面又填满了，这些文件究竟是从哪里读取来的呢？从 [yarn部署模式依赖预上传设置_yarn.provided.lib.dirs yarn.provided.usrlib.dir_PONY LEE的博客-CSDN博客](https://blog.csdn.net/weixin_38251332/article/details/125372181) 里面收到启发，这些 jar 包可能是 hadoop 先加载到 hdfs ，而后 yarn 从 hdfs 拿的，看了一下 hadoop fs -ls /streampark/flink/flink/lib，还真是这些 jar 包。hadoop fs -rm /streampark/flink/flink/lib/flink-table-planner-loader-1.15.3.jar 删除之后，再次删除 /hadoop/yarn/local/filecache/ 下面的文件。重新启动，问题解决。报错换了。
* 新的报错 Caused by: java.lang.UnsupportedOperationException: class org.apache.calcite.sql.SqlIdentifier: DATETIME，看了 [flink读取mysql表中的时间字段error java.time.LocalDateTime cannot be cast to java.sql.Timestamp_笔墨新城的博客-CSDN博客](https://blog.csdn.net/weixin_43975771/article/details/123049377) 和 [flink1.16flinksql在创建表时如何把 mysql datetime3映射为 Date？-问答-阿里云开发者社区-阿里云 (aliyun.com)](https://developer.aliyun.com/ask/535998)，准备替换成 TIMESTAMP 类型。过了，报错又换了
* 新的报错是 Caused by: org.apache.flink.table.api.ValidationException: Cannot discover a connector using option: 'connector'='mysql-cdc'......Caused by: org.apache.flink.table.api.ValidationException: Could not find any factory for identifier 'mysql-cdc' that implements 'org.apache.flink.table.factories.DynamicTableFactory' in the classpath. 这个类是 flink 的，在 /opt/flink-1.15.3_md/flink-table/flink-table-common/src/main/java/org/apache/flink/table/factories/DynamicTableFactory.java 这里。我就把对应的编译结果 /opt/flink-1.15.3_md/flink-table/flink-table-common/target/flink-table-common-1.15.3.jar 拷贝到了 /usr/bigtop/3.2.0/usr/lib/flink/lib/flink-table-common-1.15.3.jar。同时，对应的 identifier 也找到了，在 flink-cdc-connectors 项目里，具体是 /opt/flink-cdc-connectors-release-2.4.0/flink-connector-mysql-cdc/src/main/java/com/ververica/cdc/connectors/mysql/table/MySqlTableSourceFactory.java。检查了一下，jar包就是 flink-connector-mysql-cdc-2.4.0.jar，没有自动传到 hdfs 上，这里手动搞一把 hadoop fs -put /usr/bigtop/3.2.0/usr/lib/flink/lib/flink-connector-mysql-cdc-2.4.0.jar /streampark/flink/flink/lib/。good，修改生效了，报错又换了。
* 这次的报错是 Caused by: java.lang.NoClassDefFoundError: com/ververica/cdc/debezium/utils/ResolvedSchemaUtils...... Caused by: java.lang.ClassNotFoundException: com.ververica.cdc.debezium.utils.ResolvedSchemaUtils。这个类在 /opt/flink-cdc-connectors-release-2.4.0/flink-connector-debezium/src/main/java/com/ververica/cdc/debezium/utils/ResolvedSchemaUtils.java 对应的 jar 包是 /opt/flink-cdc-connectors-release-2.4.0/flink-connector-debezium/target/flink-connector-debezium-2.4.0.jar，老规矩，拷贝到lib，上传hdfs,hadoop fs -put /usr/bigtop/3.2.0/usr/lib/flink/lib/flink-connector-debezium-2.4.0.jar /streampark/flink/flink/lib/，顺利，报错又换了。
* 这次的报错是 Caused by: org.apache.doris.flink.exception.ConnectedFailedException: Connect to http://127.0.0.1:8030/api/test_db/doris_test_table/_schemafailed, status code is -1.......Caused by: java.net.ConnectException: Connection refused (Connection refused)。我想起来 doris 的 8030 端口被我改成 18030 了。看来，doris 的官方 jar 包是用不了了，需要自行改端口，然后编译。下载源码 [Releases · apache/doris-flink-connector (github.com)](https://github.com/apache/doris-flink-connector)，下载下来的文件名叫 1.4.0.tar.gz，可以，非常简洁。然后，在源码里面搜索 8030，一搜一大把，全是硬代码写死的，得一个一个改。改好，编译，然后把 jar 包拷贝到了 /usr/bigtop/3.2.0/usr/lib/flink/lib/flink-doris-connector-1.15-1.4.0-SNAPSHOT.jar，然后再修改 hdfs 上的 jar 包。hadoop fs -put /usr/bigtop/3.2.0/usr/lib/flink/lib/flink-doris-connector-1.15-1.4.0-SNAPSHOT.jar /streampark/flink/flink/lib/，hadoop fs -rm /streampark/flink/flink/lib/flink-doris-connector-1.15-1.4.0.jar，然后再清空 /hadoop/yarn/local/filecache/。重新见检查任务的时候我发现，原来我的 flink sql 里面给 doris 指定的端口是 8030，哈哈哈，这就尴尬了，那到底是 jar 包的问题，还是我 flink sql 的问题，反正两边都改了，万无一失吧。重新启动，改动生效，报错换了。
* 新的报错是 Caused by: java.lang.NoClassDefFoundError: org/apache/kafka/connect/source/SourceRecord......Caused by: java.lang.ClassNotFoundException: org.apache.kafka.connect.source.SourceRecord。看起来报错的问题是在kafka的连接上面了。通过 [根据类查找缺少的jar包，在已有jar包内查找类 - 一杯半盏 - 博客园 (cnblogs.com)](https://www.cnblogs.com/slankka/p/16938678.html) 这里来看，好i想还是和这个 debezium 有关系，我翻翻源码，在 /opt/flink-cdc-connectors-release-2.4.0/flink-connector-debezium 下面，好几个类都 importorg.apache.kafka.connect.source.SourceRecord; 但是 pom 文件里面没发现。所以我决定再去 bigtop 的编码编译结果里面找一找，对应的类在 /home/bigtop-3.2.0/dl/kafka-2.8.1-src/connect/api/src/main/java/org/apache/kafka/connect/source/SourceRecord.java ，kafka 的编译不是用 maven，而是在源码目录下面执行 ./gradlew idea，了解到 kafka 默认只有通过 /usr/bigtop/3.2.0/usr/lib/kafka/bin/kafka-topics.sh 在命令行有一些用户交互之后，我服了，我需要一个UI管理页面，所以我去看了尚硅谷的培训材料，里面提到了 EFAK，培训班推荐至少是大众货，搞他。
* 搞好了 EFAK，发现 kafka 的信息能正常显示，那么意味着 kafka 没问题。那么是需要我把 kafka 的 jar 包传到 HDFS 上面去？这个包在这里 /usr/bigtop/3.2.0/usr/lib/kafka/libs/connect-api-2.8.1.jar，于是就 hadoop fs -put /usr/bigtop/3.2.0/usr/lib/kafka/libs/connect-api-2.8.1.jar /streampark/flink/flink/lib/ 这把任务成功进入了 RUNNING 状态，点击 Application Name 直接能进入到 flink 后台页面，查看运行状态。里面的找到了 Exception 信息。
* 新的报错是 org.apache.flink.util.FlinkException: Global failure triggered byOperatorCoordinatorfor'Source: cdc_mysql_source[1] -> DropUpdateBefore[2] -> doris_sink[3]: Writer -> doris_sink[3]: Committer' (operator cbc357ccb763df2852fee8c4fc7d55f2)......Causedby: java.lang.NoClassDefFoundError: io/debezium/relational/history/DatabaseHistory
