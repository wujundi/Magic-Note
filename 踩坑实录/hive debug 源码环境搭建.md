# 实验条件准备

[toc]

## 1、安装 vmware fusion
 host 管理工具推荐安装 SwitchHosts

## 2、虚拟机网络设置
本机 vmnet8 ip 为 172.16.67.1
虚拟机中 ip 为 172.16.67.101
子网掩码 255.255.255.0
gateway 及 dns server 172.16.67.2(根据 172.16.67.1 来填，这个是vmware中固定的模式)

## 3、导入 cloudera quick start vm
下载
解压
安装过程可以参考这篇博客 https://blog.csdn.net/wiborgite/article/details/78731944
网络参数设置参考 虚拟机网络设置， ip 设置为 172.16.67.167


## 4、准备对应 hive 版本的源码
下载 
http://archive.cloudera.com/cdh5/cdh/5/
解压之后
参照：https://blog.csdn.net/xiao_jun_0820/article/details/72152579?utm_source=blogxgwz2
进行打包
发现报错，无法打包
报错信息为 “Non-resolvable parent POM for org.apache.hive:hive:1.1.0-cdh5.13.0: Failure to find com.cloudera.cdh:cdh-root:pom:5.13.0 in http://maven.aliyun.com/nexus/content/groups/public was cached in the local repository”
，初步理解为本地没有对应的 pom 依赖文件。
一般情况下， maven  会到对应网址下载相关文件的，但是如果修改过仓库信息，或者网络环境不是很好，可能会造成下载失败，这里以 org.apache.hive:hive:1.1.0-cdh5.13.0 为例，说下手动操作的方式：前往 
maven 仓库寻找 com.cloudera.cdh:cdh-root:pom:5.13.0 的信息，按照坐标信息找到这个 https://mvnrepository.com/artifact/com.cloudera.cdh/cdh-root，
戳进某一个版本里面的 Files，即：https://repository.cloudera.com/artifactory/cloudera-repos/com/cloudera/cdh/cdh-root/，
下载对应版本的 pom 文件，在用户的 .m2 路径下，顺着 /repository/com/cloudera/cdh/cdh-root/5.13.0 找去，把下载的 pom 文件放置在里面。




