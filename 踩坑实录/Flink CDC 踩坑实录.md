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
* 然后遇到了报错 Caused by: org.apache.flink.client.deployment.ClusterDeploymentException: Couldn't deploy Yarn Application Cluster. Caused by: java.io.FileNotFoundException: File does not exist: hdfs://noah:8020/streampark/flink/flink/plugins 网上查到其他组件也会有类似的报错，都说是需要在 HDFS 上面创建对应的目录。hadoop fs -ls /streampark/flink/flink 发现，确实没有 plugins 这个目录，su hsfs，然后 hadoop fs -mkdir /streampark/flink/flink/plugins，这回再试试。这回任务顺利启动了。
* 然后还是报错，检查了一下是 cdc 的 jar 包没拷贝过来，这扯不扯，从 /opt/flink-cdc-connectors-release-2.4.0/flink-sql-connector-mysql-cdc/target/flink-sql-connector-mysql-cdc-2.4.0.jar 拷贝到了 /usr/bigtop/3.2.0/usr/lib/flink/lib/flink-sql-connector-mysql-cdc-2.4.0.jar 还是没用
* 看streampark的后台日志，看到 /opt/apache-streampark_2.12-2.1.1-incubating-bin/logs/info.2023-08-09.log 里面打的都是 Deployment took more than 60 seconds. Please check if the requested resources are available in the YARN cluster，去网上找了一下，看到这个 [flink on yarn 模式下提示yarn资源不足问题分析-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1759954)，看起来需要检查和设置一些 YARN 队列的资源。一检查才发现，资源都被 spark 打满了，停掉 spark 之后，队列全腾出来，spark 咋这么有毒？因为 spark 并不是这个环节的主角，所以这里我们先不探究 YARN 队列和 SPARK 队列的适配关系。直接先把 spark 关掉。
* 再次启动，Application Id 和 JobManager URL 都有了，最终 Run Status 还是 FAILED 了，这个是什么原因呢？这次更惨，都没什么日志打出来，费了半天劲，在 YARN 这边找到了应用的详情，[Application application_1691502672438_0012](http://noah:8088/cluster/app/application_1691502672438_0012)，logs 点开有一段是 Log Type: jobmanager.log 下面有一行是 Showing 4096 bytes of 91291 total. Click [here](http://noah:19888/jobhistory/logs/noah:45454/container_1691502672438_0012_01_000001/container_1691502672438_0012_01_000001/hdfs/jobmanager.log/?start=0&start.time=0&end.time=9223372036854775807) for the full log. 点到这个 here 里面去，就到了 [noah:19888/jobhistory/logs/noah:45454/container_1691502672438_0012_01_000001/container_1691502672438_0012_01_000001/hdfs/jobmanager.log/?start=0&amp;start.time=0&amp;end.time=9223372036854775807](http://noah:19888/jobhistory/logs/noah:45454/container_1691502672438_0012_01_000001/container_1691502672438_0012_01_000001/hdfs/jobmanager.log/?start=0&start.time=0&end.time=9223372036854775807)。具体的报错是

```
Caused by: org.apache.flink.table.api.ValidationException: Multiple factories for identifier 'default' that implement 'org.apache.flink.table.delegation.ExecutorFactory' found in the classpath.

Ambiguous factory classes are:

org.apache.flink.table.planner.delegation.DefaultExecutorFactory
org.apache.flink.table.planner.loader.DelegateExecutorFactory
```

* 看了一下，这俩类，一个是 /opt/flink-1.15.3_md/flink-table/flink-table-planner/src/main/java/org/apache/flink/table/planner/delegation/DefaultExecutorFactory.java，另一个是 /opt/flink-1.15.3_md/flink-table/flink-table-planner-loader/src/main/java/org/apache/flink/table/planner/loader/DelegatePlannerFactory.java 我之前确实手贱，往 flink 的 lib 目录里面拷贝过 /usr/bigtop/3.2.0/usr/lib/flink/lib/flink-table-planner_2.12-1.15.3.jar 这么个玩意。我先把它名称改掉试试，怎么折腾 lib 目录的 jar 包也没有效果，我看问题不在这里，转而搜索一下，这个 jar 包还在哪里有，find / -name 'flink-table-planner-loader-1.15.3.jar' 发现 /hadoop/yarn/local/filecache/17/flink-table-planner-loader-1.15.3.jar 这里还有一处意外的备份。结果意外的发现 /hadoop/yarn/local/filecache/ 这里面全是yarn缓存的 jar 包，怪不得我修改 lib 没用，合着人家就根本没从那里面拿了。全删掉，重新测。结果已测试，/hadoop/yarn/local/filecache/ 里面又填满了，这些文件究竟是从哪里读取来的呢？从 [yarn部署模式依赖预上传设置_yarn.provided.lib.dirs yarn.provided.usrlib.dir_PONY LEE的博客-CSDN博客](https://blog.csdn.net/weixin_38251332/article/details/125372181) 里面收到启发，这些 jar 包可能是 hadoop 先加载到 hdfs ，而后 yarn 从 hdfs 拿的，看了一下 hadoop fs -ls /streampark/flink/flink/lib，还真是这些 jar 包。hadoop fs -rm /streampark/flink/flink/lib/flink-table-planner-loader-1.15.3.jar 删除之后，再次删除 /hadoop/yarn/local/filecache/ 下面的文件。重新启动，问题解决。报错换了。
* 新的报错 Caused by: java.lang.UnsupportedOperationException: class org.apache.calcite.sql.SqlIdentifier: DATETIME，看了 [flink读取mysql表中的时间字段error java.time.LocalDateTime cannot be cast to java.sql.Timestamp_笔墨新城的博客-CSDN博客](https://blog.csdn.net/weixin_43975771/article/details/123049377) 和 [flink1.16flinksql在创建表时如何把 mysql datetime3映射为 Date？-问答-阿里云开发者社区-阿里云 (aliyun.com)](https://developer.aliyun.com/ask/535998)，准备替换成 TIMESTAMP 类型。过了，报错又换了
* 新的报错是 Caused by: org.apache.flink.table.api.ValidationException: Cannot discover a connector using option: 'connector'='mysql-cdc'......Caused by: org.apache.flink.table.api.ValidationException: Could not find any factory for identifier 'mysql-cdc' that implements 'org.apache.flink.table.factories.DynamicTableFactory' in the classpath. 这个类是 flink 的，在 /opt/flink-1.15.3_md/flink-table/flink-table-common/src/main/java/org/apache/flink/table/factories/DynamicTableFactory.java 这里。我就把对应的编译结果 /opt/flink-1.15.3_md/flink-table/flink-table-common/target/flink-table-common-1.15.3.jar 拷贝到了 /usr/bigtop/3.2.0/usr/lib/flink/lib/flink-table-common-1.15.3.jar。同时，对应的 identifier 也找到了，在 flink-cdc-connectors 项目里，具体是 /opt/flink-cdc-connectors-release-2.4.0/flink-connector-mysql-cdc/src/main/java/com/ververica/cdc/connectors/mysql/table/MySqlTableSourceFactory.java。检查了一下，jar包就是 flink-connector-mysql-cdc-2.4.0.jar，没有自动传到 hdfs 上，这里手动搞一把 hadoop fs -put /usr/bigtop/3.2.0/usr/lib/flink/lib/flink-connector-mysql-cdc-2.4.0.jar /streampark/flink/flink/lib/。good，修改生效了，报错又换了。
* 这次的报错是 Caused by: java.lang.NoClassDefFoundError: com/ververica/cdc/debezium/utils/ResolvedSchemaUtils...... Caused by: java.lang.ClassNotFoundException: com.ververica.cdc.debezium.utils.ResolvedSchemaUtils。这个类在 /opt/flink-cdc-connectors-release-2.4.0/flink-connector-debezium/src/main/java/com/ververica/cdc/debezium/utils/ResolvedSchemaUtils.java 对应的 jar 包是 /opt/flink-cdc-connectors-release-2.4.0/flink-connector-debezium/target/flink-connector-debezium-2.4.0.jar，老规矩，拷贝到lib，上传hdfs,hadoop fs -put /usr/bigtop/3.2.0/usr/lib/flink/lib/flink-connector-debezium-2.4.0.jar /streampark/flink/flink/lib/，顺利，报错又换了。
* 这次的报错是 Caused by: org.apache.doris.flink.exception.ConnectedFailedException: Connect to http://127.0.0.1:8030/api/test_db/doris_test_table/_schemafailed, status code is -1.......Caused by: java.net.ConnectException: Connection refused (Connection refused)。我想起来 doris 的 8030 端口被我改成 18030 了。看来，doris 的官方 jar 包是用不了了，需要自行改端口，然后编译。下载源码 [Releases · apache/doris-flink-connector (github.com)](https://github.com/apache/doris-flink-connector)，下载下来的文件名叫 1.4.0.tar.gz，可以，非常简洁。然后，在源码里面搜索 8030，一搜一大把，全是硬代码写死的，得一个一个改。改好，编译，然后把 jar 包拷贝到了 /usr/bigtop/3.2.0/usr/lib/flink/lib/flink-doris-connector-1.15-1.4.0-SNAPSHOT.jar，然后再修改 hdfs 上的 jar 包。hadoop fs -put /usr/bigtop/3.2.0/usr/lib/flink/lib/flink-doris-connector-1.15-1.4.0-SNAPSHOT.jar /streampark/flink/flink/lib/，hadoop fs -rm /streampark/flink/flink/lib/flink-doris-connector-1.15-1.4.0.jar，然后再清空 /hadoop/yarn/local/filecache/。重新见检查任务的时候我发现，原来我的 flink sql 里面给 doris 指定的端口是 8030，哈哈哈，这就尴尬了，那到底是 jar 包的问题，还是我 flink sql 的问题，反正两边都改了，万无一失吧。重新启动，改动生效，报错换了。
* 新的报错是 Caused by: java.lang.NoClassDefFoundError: org/apache/kafka/connect/source/SourceRecord......Caused by: java.lang.ClassNotFoundException: org.apache.kafka.connect.source.SourceRecord。看起来报错的问题是在kafka的连接上面了。通过 [根据类查找缺少的jar包，在已有jar包内查找类 - 一杯半盏 - 博客园 (cnblogs.com)](https://www.cnblogs.com/slankka/p/16938678.html) 这里来看，好像还是和这个 debezium 有关系，我翻翻源码，在 /opt/flink-cdc-connectors-release-2.4.0/flink-connector-debezium 下面，好几个类都 importorg.apache.kafka.connect.source.SourceRecord; 但是 pom 文件里面没发现。所以我决定再去 bigtop 的编码编译结果里面找一找，对应的类在 /home/bigtop-3.2.0/dl/kafka-2.8.1-src/connect/api/src/main/java/org/apache/kafka/connect/source/SourceRecord.java ，kafka 的编译不是用 maven，而是在源码目录下面执行 ./gradlew idea，了解到 kafka 默认只有通过 /usr/bigtop/3.2.0/usr/lib/kafka/bin/kafka-topics.sh 在命令行有一些用户交互之后，我服了，我需要一个UI管理页面，所以我去看了尚硅谷的培训材料，里面提到了 EFAK，培训班推荐至少是大众货，搞他。
* 搞好了 EFAK，发现 kafka 的信息能正常显示，那么意味着 kafka 没问题。那么是需要我把 kafka 的 jar 包传到 HDFS 上面去？这个包在这里 /usr/bigtop/3.2.0/usr/lib/kafka/libs/connect-api-2.8.1.jar，于是就 hadoop fs -put /usr/bigtop/3.2.0/usr/lib/kafka/libs/connect-api-2.8.1.jar /streampark/flink/flink/lib/ 这把任务成功进入了 RUNNING 状态，点击 Application Name 直接能进入到 flink 后台页面，查看运行状态。里面的找到了 Exception 信息。
* ----------> CHECK LINE
* 新的报错是 org.apache.flink.util.FlinkException: Global failure triggered byOperatorCoordinatorfor'Source: cdc_mysql_source[1] -> DropUpdateBefore[2] -> doris_sink[3]: Writer -> doris_sink[3]: Committer' (operator cbc357ccb763df2852fee8c4fc7d55f2)......Causedby: java.lang.NoClassDefFoundError: io/debezium/relational/history/DatabaseHistory at com.ververica.cdc.connectors.mysql.source.config.MySqlSourceConfigFactory.createConfig(MySqlSourceConfigFactory.java:304。这debe有点眼熟，在 flink cdc 的源码里面找到了它信息,发现好几处都 import 了它，再对比报错，看起来是 mysql cdc connector 报的错误。那么参照 [根据类查找缺少的jar包，在已有jar包内查找类 - 一杯半盏 - 博客园 (cnblogs.com)](https://www.cnblogs.com/slankka/p/16938678.html) 这个找包大法，我也来试一试搜索 c:DatabaseHistory g:io.debezium，搜到的并且在 /opt/NOAH_source_reference/flink-cdc-connectors-release-2.4.0/pom.xml 里面找到了 debezium.version 是 1.9.7.Final，那么就是 [io.debezium : debezium-core : 1.9.7.Final - Maven Central Repository Search](https://search.maven.org/artifact/io.debezium/debezium-core/1.9.7.Final/jar) 这个了。我把它下载并放置在了 /usr/bigtop/3.2.0/usr/lib/flink/lib/debezium-core-1.9.7.Final.jar 这里。但是这样不行，我再放到 HDFS 看看，hadoop fs -put /usr/bigtop/3.2.0/usr/lib/flink/lib/debezium-core-1.9.7.Final.jar /streampark/flink/flink/lib/ , 哦吼，报错换了
* 新的报错是 Causedby: java.lang.NoClassDefFoundError: io/debezium/connector/mysql/MySqlConnectorConfig at com.ververica.cdc.connectors.mysql.source.config.MySqlSourceConfig.`<init>`(MySqlSourceConfig.java:117) 意外的发现一个事情，在 yarn 里面点击 flink 任务的 application master 链接，也会跳转到 flink 的任务 ui 那里。搜 c:MySqlConnectorConfig g:io.debezium.connector.mysql ，找到了 [io.debezium : debezium-connector-mysql : 1.9.7.Final - Maven Central Repository Search](https://search.maven.org/artifact/io.debezium/debezium-connector-mysql/1.9.7.Final/jar) 版本还是选 1.9.7.Final，放置在 /usr/bigtop/3.2.0/usr/lib/flink/lib/debezium-connector-mysql-1.9.7.Final.jar 之后在启动任务，没啥卵用。得传到 HDFS 才行 hadoop fs -put /usr/bigtop/3.2.0/usr/lib/flink/lib/debezium-connector-mysql-1.9.7.Final.jar /streampark/flink/flink/lib/。好的，报错又换了
* 新的报错是 Causedby: java.lang.NoClassDefFoundError: org/apache/kafka/common/KafkaException...... Causedby: java.lang.ClassNotFoundException: org.apache.kafka.common.KafkaException，搜 c:KafkaException g:org.apache.kafka，ambari 的 kafka 版本是 2.8.1，后来一看，原来在 /usr/bigtop/3.2.0/usr/lib/kafka/libs/kafka-clients-2.8.1.jar 就有，搞吧 hadoop fs -put /usr/bigtop/3.2.0/usr/lib/kafka/libs/kafka-clients-2.8.1.jar /streampark/flink/flink/lib/
* 新报错 Causedby: java.lang.NoClassDefFoundError: Could not initialize class io.debezium.connector.mysql.MySqlConnectorConfig，这个类刚刚搞过了，这次报这个 Couldnot initialize 是怎么个情况呢？在代码里面搜索 debezium-connector-mysql 的时候，发现 /opt/NOAH_source_reference/flink-cdc-connectors-release-2.4.0/flink-sql-connector-mysql-cdc/pom.xml 也用到了，那我把这块对应的 jar 包也传到 HDFS 上试一下，hadoop fs -put /usr/bigtop/3.2.0/usr/lib/flink/lib/flink-sql-connector-mysql-cdc-2.4.0.jar /streampark/flink/flink/lib/，然后好了，报错换了。
* 新的报错 Causedby: java.lang.NoClassDefFoundError: org/antlr/v4/runtime/TokenStream......Causedby: java.lang.ClassNotFoundException: org.antlr.v4.runtime.TokenStream，搜索 c:TokenStream g:org.antlr.v4，找到 \<artifactId\>antlr-runtime\</artifactId\> ,搜索了一下 flink 源码，里面好几处 maven 坐标写的是 3.5.2 版本，下载，上传 hadoop fs -put /opt/NOAH_files/antlr-runtime-3.5.2.jar /streampark/flink/flink/lib/ 结果没啥用。换个思路，我在源码里面直接搜 org.antlr.v4.runtime.TokenStream，发现只有 /opt/NOAH_source_reference/flink-1.15.3_md/flink-table/flink-table-code-splitter/target/generated-sources/antlr4/org/apache/flink/table/codesplit/JavaLexer.java 这里出现了一模一样的 import ，那么它所在的项目中的 pom 里面对于 antlr 有关的部分是啥呢，是 \<artifactId\>antlr4-runtime\</artifactId\> 及 \<version\>${antlr4.version}\</version\> 而 antlr4.version 的赋值是 4.7 ，那么就去搜一下，找到了 [org.antlr : antlr4-runtime : 4.7 - Maven Central Repository Search](https://search.maven.org/artifact/org.antlr/antlr4-runtime/4.7/jar)，上传 hadoop fs -put /opt/NOAH_files/antlr4-runtime-4.7.jar /streampark/flink/flink/lib/，为了防止冲突，我再把之前上传的 antlr-runtime-3.5.2.jar 删掉。重新启动任务。报错换了。
* 新的报错，在 flink 后台页面的 Exception tab 里面，内容是 Causedby: java.lang.VerifyError: Badreturntype ExceptionDetails: Location: io/debezium/connector/mysql/antlr/MySqlAntlrDdlParser.createNewParserInstance(Lorg/antlr/v4/runtime/CommonTokenStream;)Lorg/antlr/v4/runtime/Parser; @5: areturn Reason:Type'io/debezium/ddl/parser/mysql/generated/MySqlParser' (current frame, stack[0]) is not assignable to 'org/antlr/v4/runtime/Parser' (from method signature) CurrentFrame:bci: @5 flags: { } locals: { 'io/debezium/connector/mysql/antlr/MySqlAntlrDdlParser', 'org/antlr/v4/runtime/CommonTokenStream' } stack: { 'io/debezium/ddl/parser/mysql/generated/MySqlParser' }Bytecode: 0x0000000: 2a2b b601 18b0 at io.debezium.connector.mysql.MySqlDatabaseSchema.`<init>`(MySqlDatabaseSchema.java:104) at com.ververica.cdc.connectors.mysql.debezium.DebeziumUtils.createMySqlDatabaseSchema(DebeziumUtils.java:105)，参考了[flink redis connector 报错Caused by: java.lang.VerifyError: Bad return type_Thomas杨大炮的博客-CSDN博客](https://blog.csdn.net/weixin_42598278/article/details/129609783)，怀疑是字段类型写的有问题，所以检查了一下，暂时去掉了可能引起争议的时间类型，只保留 int 类型的 id 字段，试试看。还是一样的报错，那么看起来就不是sql里面写的类型的问题了。通过这个 [proguard混淆后出现java.lang.VerifyError: Bad return type错误_proguard 混淆后 启动报错_码上致富的博客-CSDN博客](https://blog.csdn.net/mashangzhifu/article/details/123155567) 案例，可以看出 Reason 的意思是，某个地方需要的是 org/antlr/v4/runtime/Parser，但是实际传入的却是 io/debezium/ddl/parser/mysql/generated/MySqlParser 因此出现了问题。

  * 我搜索了一下本地代码，在 /opt/NOAH_source_reference/flink-cdc-connectors-release-2.4.0/flink-connector-mysql-cdc/src/main/java/io/debezium/connector/mysql/antlr/listener/DefaultValueParserListener.java 里面找到了 importio.debezium.ddl.parser.mysql.generated.MySqlParser; 同时，我又下载了 debezium 的源码，在 opt/NOAH_source_reference/debezium-1.9.7.Final/* 里面，也找到了大量的带有 import io.debezium.ddl.parser.mysql.generated.MySqlParser; 的类，自此可以知道 MySqlParser 和 flink CDC 还有 debezium-1.9.7.Final 相关。
  * 那么接下来，来看另一半，org/antlr/v4/runtime/Parser 都和谁有关呢？同样搜索了一下，是在 opt/NOAH_source_reference/antlr4-4.7/* 和 opt/NOAH_source_reference/debezium-1.9.7.Final/* 下的个别类。
  * 再接下来。我从 HDFS 里面看了一下目前都上传了哪些 jar， hadoop fs -ls /streampark/flink/flink/lib/，和上面两点相关的有 antlr4-runtime-4.7.jar、debezium-connector-mysql-1.9.7.Final.jar、debezium-core-1.9.7.Final.jar、flink-connector-debezium-2.4.0.jar
  * 注意报错之前是进行到这里 at io.debezium.connector.mysql.MySqlDatabaseSchema.`<init>`(MySqlDatabaseSchema.java:104) at com.ververica.cdc.connectors.mysql.debezium.DebeziumUtils.createMySqlDatabaseSchema(DebeziumUtils.java:105) 经搜索，这类是在 opt/NOAH_source_reference/flink-cdc-connectors-release-2.4.0 里面被 import 的。在 /opt/NOAH_source_reference/flink-cdc-connectors-release-2.4.0/flink-connector-mysql-cdc/src/main/java/io/debezium/connector/mysql/antlr/listener/DefaultValueParserListener.java 里面 import 了 io.debezium.ddl.parser.mysql.generated.MySqlParser。并且在整个 flink-cdc-connectors-release-2.4.0 里面没有搜到 org.antlr.v4.runtime.Parser
* 为了解决 Causedby: java.lang.NoClassDefFoundError: org/antlr/v4/runtime/TokenStream......Causedby: java.lang.ClassNotFoundException: org.antlr.v4.runtime.TokenStream 而引入了 antlr4-runtime-4.7.jar，然后这里的 org/antlr/v4/runtime/Parser 又和 FLINK cdc 里面的东西对不上，那么是不是说，antlr4 就不应该被引入进来，或者说，在 java.lang.ClassNotFoundException: org.antlr.v4.runtime.TokenStream 之前，就又什么地方的版本出现了问题？

  * 把 antlr-runtime-3.5.2.jar 重新上传到hdfs，把 4.0 版本去掉 hadoop fs -rm /streampark/flink/flink/lib/antlr4-runtime-4.7.jar ，同时清除掉 /hadoop/yarn/local/filecache/ 下的各个目录，确保yarn重新拉取文件
  * 在本文档从上到下再看一下，都做过什么修改，引入过什么jar包，除去一些本来就是从 flink 源码或者从 bigtop 3.2当中获得的版本之外，最大的疑点来到了 flink cdc 的版本上，官网这里看 [Overview — CDC Connectors for Apache Flink® documentation (ververica.github.io)](https://ververica.github.io/flink-cdc-connectors/release-2.4/content/about.html#supported-flink-versions)，flink 1.5.* 可以适配 fkunk cdc 的 2.3 和 2.4 版本，我下载了两个版本的源码，其中对于 debezium 是有差异的，flink cdc 2.3 使用的是 debezium 1.6.4.Final，flink cdc 2.4 使用的是 debezium 1.9.7.Final。
  * 所以，尝试1，将 flink cdc 相关的包更改 2.3 版本，debezium 相关的包更改为 1.6.4.Final 版本
    * 换了 flink-connector-mysql-cdc-2.3.0.jar、flink-sql-connector-mysql-cdc-2.3.0.jar、flink-connector-debezium-2.3.0.jar、debezium-connector-mysql-1.6.4.Final.jar、debezium-core-1.6.4.Final.jar
    * 换了之后，还是报错 Causedby: java.lang.ClassNotFoundException: org.antlr.v4.runtime.tree.ParseTree，看来 antlr.v4 是躲不过去了。那再把 antlr4-runtime-4.7.jar 加回来试试
    * 报错 Causedby: java.lang.VerifyError: Badreturntype ExceptionDetails: Location: io/debezium/connector/mysql/antlr/MySqlAntlrDdlParser.createNewLexerInstance(Lorg/antlr/v4/runtime/CharStream;)Lorg/antlr/v4/runtime/Lexer; @5: areturn Reason: Type'io/debezium/ddl/parser/mysql/generated/MySqlLexer' (current frame, stack[0]) is not assignable to 'org/antlr/v4/runtime/Lexer' (from method signature) CurrentFrame: bci: @5 flags: { } locals: { 'io/debezium/connector/mysql/antlr/MySqlAntlrDdlParser', 'org/antlr/v4/runtime/CharStream' } stack: { 'io/debezium/ddl/parser/mysql/generated/MySqlLexer' } Bytecode: 0x0000000: 2a2b b600 58b0 at io.debezium.connector.mysql.MySqlDatabaseSchema.`<init>`(MySqlDatabaseSchema.java:101) at com.ververica.cdc.connectors.mysql.debezium.DebeziumUtils.createMySqlDatabaseSchema(DebeziumUtils.java:98) 这个报错和之前的报错非常相似，但是报错的具体类是有些微差别的
    * 和版本降级之前的报错非常相似，综合两个报错一起来看的话，好像是 io/debezium/ddl/parser/mysql/* 这边的东西和 /antlr/v4/runtime/* 这里要求的类型不太一致。更详细一点说是 /opt/NOAH_source_reference/debezium-1.9.7.Final/debezium-connector-mysql/src/main/java/io/debezium/connector/mysql/antlr/MySqlAntlrDdlParser.java 里面29、30两行 import 的东西。
    * 在里面又找了一下，有个 /opt/NOAH_source_reference/debezium-1.9.7.Final/debezium-ddl-parser/src/main/antlr4/io/debezium/ddl/parser/mysql/generated 下面正好就有 MySqlLexer.g4 和 MySqlParser.g4 两个文件，这，，，有点太巧合了吧，那是不是缺的就是这个 /opt/NOAH_source_reference/debezium-1.9.7.Final/debezium-ddl-parser 模块的编译结果呢？maven 中央仓库里面还真搜到了 [io.debezium : debezium-ddl-parser : 1.9.7.Final - Maven Central Repository Search](https://search.maven.org/artifact/io.debezium/debezium-ddl-parser/1.9.7.Final/jar)，下载下来放到了HDFS里面 hadoop fs -put /opt/NOAH_files/debezium-ddl-parser-1.9.7.Final.jar /streampark/flink/flink/lib/，这回再运行一下看看，生效了，报错换了。
* 新的报错是 Causedby: java.lang.NoSuchMethodError: io.debezium.connector.mysql.MySqlConnection$MySqlConnectionConfiguration.`<init>`(Lio/debezium/config/Configuration;Ljava/util/Properties;)V at com.ververica.cdc.connectors.mysql.debezium.DebeziumUtils.createMySqlConnection(DebeziumUtils.java:85) 报错 at 的类在 /opt/NOAH_source_reference/flink-cdc-connectors-release-2.4.0/flink-connector-mysql-cdc/src/main/java/com/ververica/cdc/connectors/mysql/debezium/DebeziumUtils.java，看起来它 import io.debezium.config.Configuration; 没找到东西，搜了一下代码，看起来是要找 /opt/NOAH_source_reference/debezium-1.9.7.Final/debezium-core/src/main/java/io/debezium/config/Configuration.java 这个东西，搜一下对应的 jar 包吧，在 [io.debezium : debezium-core : 1.9.7.Final - Maven Central Repository Search](https://search.maven.org/artifact/io.debezium/debezium-core/1.9.7.Final/jar)，结果发现这个 jar 包已经放到 HDFS 上面了呀。重新理解一下报错，它应该是在找一个方法，这个方法使用的入参是 Configuration 和 Properties，结果没找到，所以报错了。这么说的话，应该是指第86行 new MySqlConnection.MySqlConnectionConfiguration(dbzConfiguration, jdbcProperties)); 的时候，这个类看起来是在 /opt/NOAH_source_reference/flink-cdc-connectors-release-2.4.0/flink-connector-mysql-cdc/src/main/java/io/debezium/connector/mysql/MySqlConnection.java 的第 579 行的public MySqlConnectionConfiguration(Configurationconfig, PropertiesjdbcProperties)，对应的 jar 包已经上传 HDFS了呀（/streampark/flink/flink/lib/flink-connector-mysql-cdc-2.4.0.jar）搜索的时候我发现，包路径叫 io.debezium.connector.mysql.MySqlConnection 的类，在两个地方都有，分别是 /opt/NOAH_source_reference/debezium-1.9.7.Final/debezium-connector-mysql/src/main/java/io/debezium/connector/mysql/MySqlConnection.java(下文简称A) 和 /opt/NOAH_source_reference/flink-cdc-connectors-release-2.4.0/flink-connector-mysql-cdc/src/main/java/io/debezium/connector/mysql/MySqlConnection.java（下文简称B），并且 A 里面是没有目标形式的 MySqlConnectionConfiguration 方法的，只有 B 里面才有。尝试一下把A对应的jar包从 HDFS 上面拿掉，但是前面之所以加这个 jar 包是报错找不到 io/debezium/connector/mysql/MySqlConnectorConfig 类了呀，先试一试吧，不行，拿掉是真之前那个错呀.

  * 尝试1，参考 [有遇到过这个Flink CDC 问题的大佬没 两个包里的类冲突 但又不能删 删了别的类就找不到 咋？-问答-阿里云开发者社区-阿里云 (aliyun.com)](https://developer.aliyun.com/ask/517815)，看一下把 flink-connector-mysql-cdc-2.4.0.jar 换成 flink-sql-connector-mysql-cdc.jar 包能不能解决问题，不太行，还是报一样的错误
  * 尝试2，在 flink cdc 编译的时候直接把所需的jar包打进来，这个办法也有其他人想到，[java.lang.NoSuchMethodError: io.debezium.connector.mysql.MySqlConnection$MySqlConnectionConfiguratio_我是一只猴子的博客-CSDN博客](https://blog.csdn.net/qq_41864443/article/details/132479176)，并且从 [我的架包往集群上丢运行就报错，兄弟们知道这个原因吗？ -问答-阿里云开发者社区-阿里云 (aliyun.com)](https://developer.aliyun.com/ask/458495) 里面得知flink cdc 里面 flink-sql-connector* 的这些 jar 本身就是 fat jar，我用 jd-gui 也打开看了一下确实有 MySqlConnection，而且还是带有 publicMySqlConnectionConfiguration(Configurationconfig, PropertiesjdbcProperties) 的。那么接下来我就把 debezium 相关的 jar 包下掉试试
    * 删掉 debezium-connector-mysql-1.9.7.Final.jar、debezium-core-1.9.7.Final.jar、debezium-ddl-parser-1.9.7.Final.jar
    * 启动之后报错是 Causedby: java.lang.NoClassDefFoundError: Couldnot initialize class io.debezium.connector.mysql.MySqlConnectorConfig at com.ververica.cdc.connectors.mysql.source.config.MySqlSourceConfig.`<init>`(MySqlSourceConfig.java:117)，可是我用反编译，看包里面已经是在了呀。
    * 参考 [Java 打包 FatJar 方法小结 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/43238220)，/opt/NOAH_source_reference/flink-cdc-connectors-release-2.4.0/flink-sql-connector-mysql-cdc/pom.xml 里面使用的就是 maven-shade-plugin，但是看起来好像并没有 relocate debezium，那么这里 include 到的包，如果 HDFS 已经上传了单独了，我下都拿掉，包括 flink-connector-debezium-2.4.0.jar、antlr4-runtime-4.7.jar、connect-api-2.8.1.jar、kafka-clients-2.8.1.jar
    * 报错换了，看起来起作用了。看起来之前和 debezium 相关的那些 jar 包不需要引入，因为 flink-sql-connector-mysql-cdc-2.4.0.jar 是一个 all in one 思路下的 fat jar，所以只要把这个 jar 包放到 HDFS 就好了。
* 新的报错是 Causedby: org.apache.flink.util.FlinkRuntimeException: Cannot read the binlog filename and position via 'SHOW MASTER STATUS'. Make sure your server is correctly configured，看起来已经是在尝试读 binlog 了，参考 [mysql开启bin log_mysql开启binlog-CSDN博客](https://blog.csdn.net/weixin_46039745/article/details/132223528)，在 /etc/my.cnf 里面增加参数之后，重启 mysqld 服务
* 报错换了，Causedby: java.net.ConnectException: Connection refused (Connection refused)，开了 switchhost 之后，问题解决了。
* 加上 content 字段之后报错 Causedby: org.apache.doris.flink.exception.DorisRuntimeException: stream load error: [INTERNAL_ERROR]too many filtered rows, see more in http://172.17.0.2:18040/api/_load_error_log?file=__shard_0/error_log_insert_stmt_714ccabc617d3b09-9898b25fab573697_714ccabc617d3b09_9898b25fab573697，这里的 172.17.0.2 是容器的ip，得换成 127.0.0.1 然后访问，页面中的报错是 Reason: column_name[content], the length of input is too long than schema. first 32 bytes of input str: [{"results":[{"nat":"ES","gender"] schema length: 1024; actual length: 1161; . src line [];，搜索了一下，[Apache Doris FAQs-Apache Doris 文档-面试哥 (mianshigee.com)](https://www.mianshigee.com/tutorial/Doris/faq.md)，再看看之前的 doris 见表语句，确实是 content VARCHAR(1024) COMMENT "抓取信息" 了，这回改成 10240 试一下，问题解决。
* 然后去 doris 里面看到数据有了，开心，自此 flink cdc 就算走通了。包含了整数、长字符串、时间类型三个字段。


noah重装

---

没想到重装的时候i遇到的问题还会不一样

---

准备新建一个 flink_cdc_to_doris_test 任务，结果在 Verify 环节报错，无法提交

在 /opt/NOAH_streampark/apache-streampark_2.12-2.1.1-incubating-bin/logs/info.2023-12-19.log 里面查看日志，是一个空指针错误。

java.lang.NullPointerException: null

    at org.apache.streampark.console.core.controller.FlinkSqlController.verify(FlinkSqlController.java:64)

找到是这个类 /opt/NOAH_source_reference/apache-streampark-2.1.1-incubating-src/streampark-console/streampark-console-service/src/main/java/org/apache/streampark/console/core/controller/FlinkSqlController.java

这基本是啥也看不出来呀，/opt/NOAH_streampark/apache-streampark_2.12-2.1.1-incubating-bin/logs/error.2023-12-19.log 这里的信息多一些，除了反射的这些最终给到了 Caused by: java.lang.NoClassDefFoundError: org/apache/calcite/sql/validate/SqlConformance 

Caused by: java.lang.ClassNotFoundException: org.apache.calcite.sql.validate.SqlConformance
