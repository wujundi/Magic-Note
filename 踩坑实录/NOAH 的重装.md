# NOAH的重装

这真是一个艰难的决定，在本地机器环境崩溃之后，我只能重装系统，然后伴随着重装系统的ip变化，容器也无法启动了。

我尝试了很多办法调试容器和ambari配置文件中对ip的配置，都没有效果。

主要矛头指向两个部分：

1、现在的镜像内还有一些残存的，ambari-server 版本时候配置的影子（虽然我不知道是怎么搞进来的，我印象中做noah的时候应该已经是完全的重装重装了）

2、之前在安装的 bigtop 镜像的时候，我使用了当时 wsl  的 ip （baseurl=http://172.18.254.162:2929/bigtop/），这个 IP 就像幽灵一样，在每一次 ambari 启动的时候都会将其作为 yum 源去扫描。或者说，ambari 不是每次启动都需要从 bigtop 镜像重新安装一遍，但是它却每次都需要找得到那台机器才行。那我把 bigtop 机器的 ip 写死了，这就非常尴尬，总不能每次基础环境更换，就不能复用容器了吧。

综上，我决定还是要再次大搞一番。

本次安装必须遵照两个原则：

1、所有环节都 100% 的重新安装，杜绝一些不明所以的配置变量

2、所有环节都必须实现通用化，禁止一切与基础环境绑定的事项。

好的，开干

---

## 基础环境搭建(systemctl+mysql)

1、docker pull registry.cn-hangzhou.aliyuncs.com/wujundi/centos-noah:bigtop-offical-recommended-build-environment-in-centos-for-release-3.2.0

2、docker run -itd --name='noah' --hostname='noah' -e TZ=Asia/Shanghai -p 8000:8000 -p 8080:8080 -p 50070:50070 -p 8088:8088 -p 19888:19888 -p 3000:3000 -p 10002:10002 -p 16010:16010 -p 18081:18081 -p 9995:9995 -p 8082:8082 -p 8983:8983 -p 8440:8440 -p 8441:8441 -p 9092:9092 -p 4040:4040 -p 4041:4041 -p 9000:9000 -p 10086:10086 -p 5432:5432 -p 8081:8081 -p 18030:18030 -p 19020:19020 -p 19030:19030 -p 19010:19010 -p 19050:19050 -p 19060:19060 -p 18040:18040 -p 18060:18060 -p 18088:18088 -p 2181:2181 -p 18048:18048 -p 18085:18085 -p 8020:8020  --privileged registry.cn-hangzhou.aliyuncs.com/wujundi/centos-noah:bigtop-offical-recommended-build-environment-in-centos-for-release-3.2.0

3、备份 /bin/systemctl，mv /bin/systemctl /bin/systemctl.bak

4、wget https://raw.githubusercontent.com/gdraheim/docker-systemctl-replacement/master/files/docker/systemctl.py -O /bin/systemctl

5、sudo chmod a+x /bin/systemctl

6、cd /opt && wget https://dev.mysql.com/get/mysql80-community-release-el7-11.noarch.rpm

7、sudo yum localinstall mysql80-community-release-el7-11.noarch.rpm

8、sudo yum-config-manager --disable mysql80-community

9、sudo yum-config-manager --enable mysql57-community

10、确认一下要安装的版本 yum repolist all | grep mysql 和 yum repolist enabled | grep mysql

11、执行安装就行了 sudo yum install mysql-community-server

12、service mysqld start

13、查看 root 用户的默认密码 **sudo** grep **'temporary password'** /var/log/mysqld**.**log

14、mysql -uroot -p

15、ALTER USER 'root'@'localhost' IDENTIFIED BY '#Mysql123';

16、SHOW VARIABLES LIKE 'validate_password%';

17、set global validate_password_policy=LOW;

18、set global validate_password_length=4;

19、SHOW VARIABLES LIKE 'validate_password%'

20、设置完成之后，最后重置密码 ALTER USER 'root'@'localhost' IDENTIFIED BY 'mysql';

21、use mysql; update user set host ='%' where user='root'; FLUSH PRIVILEGES;

---

## ambari rpm 安装

1、cd /opt && wget https://github.com/apache/ambari/archive/refs/tags/release-2.8.0-rc0.tar.gz

2、tar xfvz release-2.8.0-rc0.tar.gz

3、cd ambari-release-2.8.0-rc0

4、修改 ambari-release-2.8.0-rc0/ambari-admin/pom.xml

```
修改  <nodeVersion>v10.24.1</nodeVersion>
和    <npmVersion>6.14.12</npmVersion>
```

5、修改 ambari-release-2.8.0-rc0/ambari-web/pom.xml，

```
修改 <nodeVersion>v10.24.1</nodeVersion>
```

6、修改 ambari-release-2.8.0-rc0/ambari-server/pom.xml，将 ambari-serviceadvisor 的版本修改为 2.8.0.0.0

7、mvn clean install rpm:rpm -DbuildNumber=test -DskipTests -Drat.skip=true -e

8、yum install /opt/ambari-release-2.8.0-rc0/ambari-server/target/rpm/ambari-server/RPMS/x86_64/ambari-server-2.8.0.0-0.x86_64.rpm

9、jdk 环节，选择自定义 jdk

```shell
[root@xxx]# which java
/usr/bin/java
[root@xxx]# ls -lrt /usr/bin/java
lrwxrwxrwx. 1 root root 22 7月  23 14:43 /usr/bin/java -> /etc/alternatives/java
[root@xxx]# ls -lrt /etc/alternatives/java
lrwxrwxrwx. 1 root root 73 7月  23 14:43 /etc/alternatives/java -> /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java
找到java路径后，去掉最后的/jre/bin/java，在profile文件中进行修改
```

10、参考 https://code84.com/813098.html ，在 /etc/profile 里增加 JAVA_HOME 变量

11、source /etc/profile

12、ambari-server setup

13、yum install /opt/ambari-release-2.8.0-rc0/ambari-agent/target/rpm/ambari-agent/RPMS/x86_64/ambari-agent-2.8.0.0-0.x86_64.rpm

## bigtop320 安装文件的引入

1、docker pull registry.cn-hangzhou.aliyuncs.com/wujundi/centos-noah:bigtop-repo-ready-for-http

2、sudo docker run -itd --name='bigtop320' --privileged   registry.cn-hangzhou.aliyuncs.com/wujundi/centos-noah:bigtop-repo-ready-for-http /usr/sbin/init

4、将 bigtop 镜像中编译后 bigtop 项目的 output 文件夹拷贝到 noah 容器的  /opt/NOAH_repositories/ 目录下

5、yum install httpd

6、修改 /etc/httpd/conf/httpd.conf

```
修改 Listen 2929
修改 Alias /bigtop /opt/NOAH_repositories/bigtop-release-3.2.0/output，
修改 <Directory "/opt/NOAH_repositories/bigtop-release-3.2.0/output">
修改 Options Indexes FollowSymLinks
增加 ServerSignature Off
```

7、浏览器 http://127.0.0.1:2929/bigtop/

8、service httpd start

---

## ambari web 页面安装

第1步里面，base url 一定不能写死某个特定的ip，这里可以用127.0.0.1

第7步里面，需要选择 hive 元数据的存放位置。这里如果选择了 new mysql 那就走回了老路，默认安装的是 mysql 5.6 ，到后边安装其他组件的时候就傻眼了。所以坚决选择 Existing MySQL

1、点开页面上提示的下载mysql连接驱动的网站，选择 Platform Independent，这样可以下载到 tar.gz，否则都是各种安装包格式。

2、 cd /opt && wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-j-8.2.0.tar.gz

3、tar -zxvf mysql-connector-j-8.2.0.tar.gz

4、mkdir NOAH_mysql_connector && mv mysql-connector-j-8.2.0 NOAH_mysql_connector/mysql-connector-j-8.2.0

5、ambari-server setup --jdbc-db=mysql --jdbc-driver=/opt/NOAH_mysql_connector/mysql-connector-j-8.2.0/mysql-connector-j-8.2.0.jar

6、service mysqld start

7、mysql -uroot -p

```
SHOW VARIABLES LIKE 'validate_password%';
set global validate_password_policy=LOW;
set global validate_password_length=4;
SHOW VARIABLES LIKE 'validate_password%';
CREATE USER 'hive'@'localhost' IDENTIFIED BY 'hive';
GRANT ALL PRIVILEGES ON * . * TO 'hive'@'localhost';
create database hive;
```



8、页面上继续操作，修改 Database URL 为 jdbc:mysql://localhost:3306/hive

9、到 ALL CONFIGURATIONS 页面的时候，修改 hadoop.proxyuser.* 为 root；修改 yarn.scheduler.minimum-allocation-mb 为 1 MB

---

## ambari 安装之后，mysql、postgressql 连接/查看权限的调整

1、mysql -uroot -p

```
use mysql;
select host, user from user;
SHOW VARIABLES LIKE 'validate_password%';
set global validate_password_policy=LOW;
set global validate_password_length=4;
SHOW VARIABLES LIKE 'validate_password%';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'mysql' WITH GRANT OPTION;
FLUSH PRIVILEGES;
exit;

```

2、serviceambari-serverstart

3、su - postgres

4、psql

```
\c ambari
grant postgres to ambari;
grant all privileges on database ambari to ambari;
\q

```

(后续可以使用默认账户ambari，默认密码bigdata连接 postgresql，来查看ambari元数据)

---

## flink sql-client 的调整

1、修改 /etc/profile，新增

```
export HADOOP_CONF_DIR=/etc/hadoop/conf
export HADOOP_CLASSPATH=/usr/bigtop/3.2.0/usr/lib/hadoop/lib
export HADOOP_HOME=/usr/bigtop/3.2.0/usr/lib/hadoop
export HADOOP_HDFS_HOME=/usr/bigtop/3.2.0/usr/lib/hadoop-hdfs
export HADOOP_MAPRED_HOME=/usr/bigtop/3.2.0/usr/lib/hadoop-mapreduce
export HADOOP_YARN_HOME=/usr/bigtop/3.2.0/usr/lib/hadoop-yarn
export HIVE_HOME=/usr/bigtop/3.2.0/usr/lib/hive
# export HBASE_HOME=/usr/bigtop/current/hbase-client # what i found
export HBASE_HOME=/usr/bigtop/3.2.0/usr/lib/hbase # what i thought it should be , comparing with the others
export FLINK_HOME=/usr/bigtop/3.2.0/usr/lib/flink
# export FLINK_CONF_DIR=${FLINK_HOME}/conf
export FLINK_CONF_DIR=/etc/flink/conf
export KE_HOME=/opt/kafka-eagle-bin-3.0.1/efak-web-3.0.1

export PATH=$PATH:$HADOOP_CONF_DIR:$HADOOP_CLASSPATH:$HADOOP_HOME:$HADOOP_HDFS_HOME:$HADOOP_MAPRED_HOME:$HADOOP_YARN_HOME
export PATH=$PATH:$HIVE_HOME:$HBASE_HOME:$FLINK_HOME:$FLINK_CONF_DIR:$KE_HOME
```

2、source /etc/profile

3、cd /usr/bigtop/3.2.0/usr/lib/flink && cp -r lib/ lib_bak/

4、下载 flink 1.15.3 源码，并编译（实际安装时候拷贝了历史文件）

5、mkdir /usr/bigtop/3.2.0/usr/lib/flink/opt

6、cp /opt/NOAH_source_reference/flink-1.15.3_md/flink-table/flink-sql-client/target/flink-sql-client-1.15.3.jar /usr/bigtop/3.2.0/usr/lib/flink/opt

7、cp /opt/NOAH_source_reference/flink-1.15.3_md/flink-python/target/flink-python_2.12-1.15.3.jar /usr/bigtop/3.2.0/usr/lib/flink/lib

5、cd /usr/bigtop/3.2.0/usr/lib/flink/bin && bash start-cluster.sh

6、cd /usr/bigtop/3.2.0/usr/lib/flink/bin && bash sql-client.sh

7、exit;

## spiderflow 的安装

1、mkdir /opt/NOAH_spiderflow

2、把原来的文件夹拷贝过来

3、mysql -uroot -p < /opt/opt_bak/data_dump/spiderflow.sql（把原来的数据dump导入，这样就不需要重复配置任务了。）

4、在 /opt/start_up.sh 里面添加

```
java -jar /opt/NOAH_spiderflow/spider-flow-0.5.0/spider-flow-web/target/spider-flow.jar &
```

## doris 的安装

1、mkdir /opt/NOAH_doris

2、把原来的文件夹拷贝过来

3、在 /opt/start_up.sh 里面添加

```
swapoff -a
/opt/NOAH_doris/doris-1.2.4.1/output/fe/bin/start_fe.sh --daemon
sysctl -w vm.max_map_count=2000000
/opt/NOAH_doris/doris-1.2.4.1/output/be/bin/start_be.sh --daemon
```


---

---

上一版本 NOAH 镜像可以查看 docker run -itd --name='bak' registry.cn-hangzhou.aliyuncs.com/wujundi/centos-noah:stream-warehouse-dev-20231118
