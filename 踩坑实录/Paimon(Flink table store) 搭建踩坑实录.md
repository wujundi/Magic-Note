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

---



USE CATALOG paimon_catalog;

CREATE TABLE paimon_test_table (
   id          BIGINT PRIMARY KEY NOT ENFORCED
  ,content     STRING
  ,create_time TIMESTAMP
);
