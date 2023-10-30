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
