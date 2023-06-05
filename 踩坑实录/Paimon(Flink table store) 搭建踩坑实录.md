# Paimon 踩坑实录

* 官方网站 [Apache Paimon](https://paimon.apache.org/)
* 到目前的稳定版本文档 [Flink | Apache Flink Table Store](https://nightlies.apache.org/flink/flink-table-store-docs-release-0.3/docs/engines/flink/)
* 在集群上找到 flink 的安装目录，首先 whereis flink，然后 which flink，就可以知道究竟 flink 命令是在执行哪个文件了，这个人间一般都是一个链接，相当于windows中的快捷方式，这里就顺藤摸瓜，找到最后实际的目录是 /usr/bigtop/3.2.0/usr/lib/flink。
* 重新在 bigtop3.2.0的编译结果里面确定了flink的版本是 1.15.3，从 [Flink | Apache Flink Table Store](https://nightlies.apache.org/flink/flink-table-store-docs-release-0.3/docs/engines/flink/) 下载了 flink 1.15 对应的jar包，放在了 /usr/bigtop/3.2.0/usr/lib/flink/lib
* 额，然后好像就完事了，可以测试了？吗？

## 用 API 的方式太原始了，我需要一个 Flink sql 的 ui 提交页面

* 所以我找到了 StreamPark [安装部署 | Apache StreamPark (incubating)](https://streampark.apache.org/zh-CN/docs/user-guide/deployment)
* 这里涉及到一个环境变量的问题，之前安装 ambari 的时候并没有注意环境变量，我是真的不知道 ambari 在安装的时候是往哪些目录里面安装的，所以这里我只能重新搞这两个镜像，这次一定要在安装界面截图保存。

docker run -itd --name='bigtop320' -p 2929:2929 registry.cn-hangzhou.aliyuncs.com/wujundi/centos-bigtop-3.2.0:ready-for-http

docker run -itd --name='ambari-server' --hostname='ambari-server' -p 8080:8080 -p 8440:8440 -p 8441:8441 registry.cn-hangzhou.aliyuncs.com/wujundi/centos-ambari-280:ready-for-install
