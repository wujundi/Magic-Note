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
* 看streampark的后台日志，看到 /opt/apache-streampark_2.12-2.1.1-incubating-bin/logs/info.2023-08-09.log 里面打的都是 Deployment took more than 60 seconds. Please check if the requested resources are available in the YARN cluster，去网上找了一下，看到这个 [flink on yarn 模式下提示yarn资源不足问题分析-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1759954)，看起来需要检查和设置一些 YARN 队列的资源。一检查才发现，资源都被 spark 打满了，停掉 spark 之后，队列全腾出来，spark 咋这么有毒？
