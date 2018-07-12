# 专题：当我在玩Hive的时候，我究竟在干什么？

***
[TOC]
***

## 一、Hive 解决了什么问题

大量数据的压力
大量数据的存储问题 -> HDFS
大量数据的计算问题 -> MapReduce框架下编写的程序
所以为了进行海量数据的计算，我们夜以继日的撸着各个 Mapper 和 Reducer 的子类。。。
绝对不是的

Hive，使开发人员使用 sql ，像操作关系型数据库一样，进行海量数据的计算

## 二、Hive 是怎么解决这个问题的

构建一个解决方案。
将 HDFS 上指定的文件，解析成逻辑上的表
使用一种类似 MySQL 的 sql 方言
将 sql 语句表达的含义，转换为底层的 MR 程序
读取、计算、新建HDFS文件，从而实现 sql 操作 data file 的愿景

## 三、Hive 解决这个问题的过程*

### 3-1 运行环境说明
1、macOS
2、VMware Fusion 
3、cdh quick start vm 
> 什么是CDH? 
> 基于Hadoop稳定版本的企业级解决方案提供商
>  "Cloudera成立于2008年，由硅谷顶尖企业的优秀人士创立，其中包括Google（Christophe Bisciglia），Yahoo! （Amr Awadallah），Oracle（Mike Olson）和Facebook（Jeff Hammerbacher）。Hadoop的共同发明人Doug Cutting于2009年加入公司担任首席架构师，至今任担任该职务。"
> 为什么选择 cdh quick start vm?
> 真正的零配置，快速拥有运行环境，零踩坑风险
官方文档介绍及下载地址：https://www.cloudera.com/documentation/enterprise/latest/topics/cloudera_quickstart_vm.html

4、IDEA
5、hive源码(需要与运行环境的代码保持一致)
> CDH代码库，http://archive.cloudera.com/cdh5/cdh/5/
往下翻，有 hive-1.1.0-cdh5.13.0-src.tar.gz 文件（可搜索“hive-1.1.0-cdh5.13.0-”）

6、java运行环境

### 3-2 远程调试原理与方法
#### java远程调试原理
远程调试基于JDWP（Java Debug Wire Protocol），这是 JVM 定义并设计的调试器和应用之间通信的协议，不同的版本虚拟机的具体实现方式不尽相同
关于java调试详尽的原理，推荐大家看下：https://blog.csdn.net/qq_33406377/article/details/50340055

#### IDEA 远程调试 Hive
1、应用端开启hive调试模式：hive --debug
命令行返回，Listening for transport dt_socket at address: 8000

2、idea打开Hive源码，其实无需build项目，设置方法参照：https://www.cnblogs.com/wy2325/p/5600232.html

3、cli下Clidriver.main()为程序入口，打上断点，开启debug后，进入调试状态

### 3-3 Hive各模块关系与一条sql语句的运行过程主线
1、Hive架构
图示：https://blog.csdn.net/yanshu2012/article/details/54944943
有图可知，对于从命令行执行hive语句的使用场景来说，cli 与 driver 是两个主要的模块

2、Hive源码学习参考
https://segmentfault.com/a/1190000002766035
https://segmentfault.com/a/1190000002774731

3、debug代码，1+1=2
  select 1+1;


### 3-4 sql语句深层次运行原理(TODO)
参见美团点评技术团队博文：https://tech.meituan.com/hive-sql-to-mapreduce.html


## 四、知道了这个过程之后，我们又能做些什么？