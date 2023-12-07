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

首先，测试 bigtop320 镜像是否可以启动

1、docker pull registry.cn-hangzhou.aliyuncs.com/wujundi/centos-noah:bigtop-repo-ready-for-http

2、sudo docker run -itd --name='bigtop320' --privileged -p 2929:2929 registry.cn-hangzhou.aliyuncs.com/wujundi/centos-noah:bigtop-repo-ready-for-http /usr/sbin/init

3、浏览器 http://127.0.0.1:2929/bigtop/

如果遇到不行的情况，需要重新安装httpd，先 yum remove httpd，然后 yum install httpd，放心历史版本的配置文件会自动重命名保留的。亲测刚开始启动无效，重装就OK了

---

基础环境搭建

1、docker pull registry.cn-hangzhou.aliyuncs.com/wujundi/centos-noah:bigtop-offical-recommended-build-environment-in-centos-for-release-3.2.0

2、docker run -itd --name='noah' --hostname='noah' -e TZ=Asia/Shanghai -p 8000:8000 -p 8080:8080 -p 50070:50070 -p 8088:8088 -p 19888:19888 -p 3000:3000 -p 10002:10002 -p 16010:16010 -p 18081:18081 -p 9995:9995 -p 8082:8082 -p 8983:8983 -p 8440:8440 -p 8441:8441 -p 9092:9092 -p 4040:4040 -p 4041:4041 -p 9000:9000 -p 10086:10086 -p 5432:5432 -p 8081:8081 -p 18030:18030 -p 19020:19020 -p 19030:19030 -p 19010:19010 -p 19050:19050 -p 19060:19060 -p 18040:18040 -p 18060:18060 -p 18088:18088 -p 2181:2181 -p 18048:18048 -p 18085:18085 -p 8020:8020  --privileged registry.cn-hangzhou.aliyuncs.com/wujundi/centos-noah:bigtop-offical-recommended-build-environment-in-centos-for-release-3.2.0

3、备份 /bin/systemctl，mv /bin/systemctl /bin/systemctl.bak

4、wget https://raw.githubusercontent.com/gdraheim/docker-systemctl-replacement/master/files/docker/systemctl.py -O /bin/systemctl

5、sudo chmod a+x /bin/systemctl

6、cd /opt & wget https://dev.mysql.com/get/mysql80-community-release-el7-11.noarch.rpm

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


---

ambari 安装

1、cd /opt & wget https://github.com/apache/ambari/archive/refs/tags/release-2.8.0-rc0.tar.gz

2、tar xfvz release-2.8.0-rc0.tar.gz

3、cd ambari-release-2.8.0-rc0

4、修改 ambari-release-2.8.0-rc0/ambari-admin/pom.xml，修改 `<nodeVersion>`v10.24.1 `</nodeVersion>` 和 `<npmVersion>`6.14.12 `</npmVersion>`

5、修改 ambari-release-2.8.0-rc0/ambari-web/pom.xml，修改 `<nodeVersion>`v10.24.1 `</nodeVersion>`

6、修改 ambari-release-2.8.0-rc0/ambari-server/pom.xml，将 ambari-serviceadvisor 的版本修改为 2.8.0.0.0

7、mvn clean install rpm:rpm -DskipTests -Drat.skip=true
