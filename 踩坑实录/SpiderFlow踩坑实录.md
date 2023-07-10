# Spider-Flow踩坑实录

* 先说一下选型背景吧。爬虫对于整个数据处理demo来说是数据源头，不管后续是进行功能验证，还是进行各种数据的处理，都需要先采集到数据。对于这个功能，其实我不需要很强大或者刁钻的定制化能力，因为我需要的数据都是开放数据，或者会寻求免费接口。对于数据抓取来说，简单，好维护，才是我的目标。因此调研的时候看了 webmagic、crawlab、spider-flow 等等组件。webmagic 属于爬虫框架，衍生的webUI项目默认是把爬取的数据存放在本地文件；rawlab 更类似于爬虫任务的调度平台，对于爬虫本身还是需要通过python来编写的；spider-flow 可以通过配置与图形化的方式创建爬虫任务，并且存储方式也是mysql，这看起来比较理想。
* 首先构建docker 镜像，这里还是以 ubuntu 镜像作为基础，docker run -itd --name='spider' --privileged -p 8088:8088 -p 3306:3306 registry.cn-hangzhou.aliyuncs.com/wujundi/ubuntu-22:latest
* 安装mysql 的时候有报错 debconf: unable to initialize frontend: Dialog
  debconf: (No usable dialog-like program is installed, so the dialog based frontend cannot be used. at /usr/share/perl5/Debconf/FrontEnd/Dialog.pm line 78.)
  debconf: falling back to frontend: Readline
  /usr/sbin/invoke-rc.d: 308: test: xinactive: unexpected operator
  /usr/sbin/invoke-rc.d: 308: test: xinactive: unexpected operator
  /usr/sbin/invoke-rc.d: 317: test: xinactive: unexpected operator
  /usr/sbin/invoke-rc.d: 317: test: xinactive: unexpected operator
  invoke-rc.d: policy-rc.d denied execution of stop.
  update-alternatives: using /etc/mysql/mysql.cnf to provide /etc/mysql/my.cnf (my.cnf) in auto mode
  Renaming removed key_buffer and myisam-recover options (if present)
  mysqld will log errors to /var/log/mysql/error.log
  mysqld is running as pid 3294
  /usr/bin/deb-systemd-helper: error: systemctl preset failed on mysql.service: No such file or directory
  ERROR:systemctl:  getty-static.service: Service Executable path is not absolute.
  ERROR:systemctl:  initrd-cleanup.service: Service Executable path is not absolute.
  ERROR:systemctl:  initrd-parse-etc.service: Service Executable path is not absolute.
  ERROR:systemctl:  initrd-switch-root.service: Service Executable path is not absolute.
  ERROR:systemctl:  systemd-ask-password-console.service: Service Executable path is not absolute.
  ERROR:systemctl:  systemd-ask-password-wall.service: Service Executable path is not absolute.
  ERROR:systemctl:  systemd-boot-system-token.service: Service Executable path is not absolute.
  ERROR:systemctl: systemd-exit.service: a .service file without [Service] section
  ERROR:systemctl:  systemd-halt.service: Service Executable path is not absolute.
  ERROR:systemctl:  systemd-journal-flush.service: Service Executable path is not absolute.
  ERROR:systemctl:  systemd-journal-flush.service: Service Executable path is not absolute.
  ERROR:systemctl:  systemd-kexec.service: Service Executable path is not absolute.
  ERROR:systemctl:  systemd-machine-id-commit.service: Service Executable path is not absolute.
  ERROR:systemctl:  systemd-networkd.service: Service Executable path is not absolute.
  ERROR:systemctl: systemd-poweroff.service: a .service file without [Service] section
  ERROR:systemctl: systemd-reboot.service: a .service file without [Service] section
  ERROR:systemctl:  systemd-sysext.service: Service Executable path is not absolute.
  ERROR:systemctl:  systemd-sysext.service: Service Executable path is not absolute.
  ERROR:systemctl:  systemd-sysusers.service: Service Executable path is not absolute.
  ERROR:systemctl:  systemd-tmpfiles-clean.service: Service Executable path is not absolute.
  ERROR:systemctl:  systemd-tmpfiles-setup-dev.service: Service Executable path is not absolute.
  ERROR:systemctl:  systemd-tmpfiles-setup.service: Service Executable path is not absolute.
  /usr/sbin/invoke-rc.d: 308: test: xinactive: unexpected operator
  /usr/sbin/invoke-rc.d: 308: test: xinactive: unexpected operator
  /usr/sbin/invoke-rc.d: 317: test: xinactive: unexpected operator
  /usr/sbin/invoke-rc.d: 317: test: xinactive: unexpected operator
  disabled
  invoke-rc.d: policy-rc.d denied execution of start.
  Setting up libcgi-pm-perl (4.54-1) ...
  Setting up libhtml-template-perl (2.97-1.1) ...
  Setting up mysql-server (8.0.33-0ubuntu0.22.04.2) ...
  Setting up libcgi-fast-perl (1:2.15-1) ...
  Processing triggers for libc-bin (2.35-0ubuntu3.1) ... 参考 [(87条消息) docker报错：invoke-rc.d: policy-rc.d denied execution of start._jiangjiane的博客-CSDN博客](https://blog.csdn.net/jiangjiang_jian/article/details/88823372) 操作，修改了 /usr/sbin/policy-rc.d。结果还是没什么用，看起来 mysql 对安装环境好挑剔哦。。。那我用原始 ubuntu 镜像来安装试试看。
* docker run -itd --privileged ubuntu，安装也没报什么错，service mysql start 的时候报了个错 ，cannot change directory to /nonexistent: No such file or directory 参考 [mysql 服务器关机后出现cannot change directory to /nonexistent: No such file or directory - 工大教务处 - 博客园 (cnblogs.com)](https://www.cnblogs.com/wjx-zjut/p/16367771.html) 解决了。
* 重新安装 jdk，配置国内 apt 源，[(87条消息) Ubuntu22.04更换国内镜像源（阿里、网易163、清华、中科大）_ubuntu中科大镜像源_凯der苦练心态的博客-CSDN博客](https://blog.csdn.net/qq_42365082/article/details/127008698)，然后安装 apt install openjdk-8-jdk
* 安装 maven，apt install maven，然后再从之前的容器中把 setting.xml 搞过来，
* 下载并解压了 spiderflow 的项目，修改 /home/spider-flow-0.5.0/spider-flow-web/src/main/resources/application.properties 里面的数据库配置，并且去 mysql 里面创建账号 CREATE USER 'spiderflow'@'%' IDENTIFIED BY 'spiderflow';
* 然后用 /home/spider-flow-0.5.0/db/spiderflow.sql 做数据库的初始化，mysql < /home/spider-flow-0.5.0/db/spiderflow.sql
* OK，下面，点火！cd /home/spider-flow-0.5.0，然后 mvn spring-boot:run，然后报错了 [ERROR] Failed to execute goal org.springframework.boot:spring-boot-maven-plugin:2.0.7.RELEASE:run (default-cli) on project spider-flow: Unable to find a suitable main class, please add a 'mainClass' property -> [Help 1]，嗯，看起来不是在这个目录下启动？
* cd /home/spider-flow-0.5.0/spider-flow-web/src/main/java/org/spiderflow/，然后 mvn spring-boot:run，然后报错 [ERROR] No plugin found for prefix 'spring-boot' in the current project and in the plugin groups [org.apache.maven.plugins, org.codehaus.mojo] available from the repositories [local (/root/.m2/repository), aliyun-nexus (https://maven.aliyun.com/nexus/content/groups/public/)] -> [Help 1]
* 看 [springboot命令行启动的方法详解 - 第一PHP社区 (php1.cn)](https://www.php1.cn/detail/springboot_MingL_818fc6bc.html) 的说法，是不是我没有编译？我现在项目根目录下面 mvn clean install，然后报错 [ERROR] Failed to execute goal on project spider-flow-web: Could not resolve dependencies for project org.spiderflow:spider-flow-web:jar:0.5.0: The following artifacts could not be resolved: org.spiderflow:spider-flow-redis:jar:0.5.0, org.spiderflow:spider-flow-mongodb:jar:0.5.0, org.spiderflow:spider-flow-selenium:jar:0.5.0, org.spiderflow:spider-flow-ocr:jar:0.5.0, org.spiderflow:spider-flow-oss:jar:0.5.0, org.spiderflow:spider-flow-mailbox:jar:0.5.0: Could not find artifact org.spiderflow:spider-flow-redis:jar:0.5.0 in apache-repository (https://repo.maven.apache.org/maven2/) -> [Help 1] 。我去看 /home/spider-flow-0.5.0/spider-flow-web/pom.xml，一看都是什么玩意，自己搞了一堆插件项目，还没包含在主项目里面，百度了一下，发现这些插件是作为独立项目存储在 gitee 的。。。真的是有那么一点点无语。我先把这些依赖注释掉，看下能怎么样。编译是没有问题了
* 然后 cd cd /home/spider-flow-0.5.0/spider-flow-web，再然后 mvn spring-boot:run，报错 Caused by: com.mysql.jdbc.exceptions.jdbc4.MySQLNonTransientConnectionException: Public Key Retrieval is not allowed，看起来启动正常了，就是数据库权限有点问题，参考了 [(87条消息) MySQL 8.0 Public Key Retrieval is not allowed 错误的原因及解决方法_mysql public key_Stone Lio的博客-CSDN博客](https://blog.csdn.net/u011447905/article/details/121441165) ，修改密码加密方式 ALTER USER 'spiderflow'@'%' IDENTIFIED WITH mysql_native_password BY 'spiderflow';。然后变成了报错 Caused by: com.mysql.jdbc.exceptions.jdbc4.MySQLSyntaxErrorException: Access denied for user 'spiderflow'@'%' to database 'spiderflow'，参考 [(87条消息) MySQL出现Access denied for user ‘xxx‘@‘%‘ to database ‘xxxx‘问题_access denied for user to database_田培融的博客-CSDN博客](https://blog.csdn.net/u011296165/article/details/122195845)，原来新建数据库之后，还有个授权用户的动作。GRANT ALL PRIVILEGES ON spiderflow.* to 'spiderflow'@'%';
* 这把运行成功了！！！！！[SpiderFlow](http://127.0.0.1:8088/)
* 上手配置爬虫也还算方便吧，为了不影响系统本身的元数据，我为抓取的数据单独建了一个数据库 spiderflow_data，建库以据以及用户赋权都和上面一样。然后 spiderflow 自带了几个爬虫示例，对于抓取 API 来说，是足够了。目前我是用一个免费的随机数 API [Random Data API (random-data-api.com)](https://random-data-api.com/documentation) 来获取的用户信息

## NEO安装

* 从 github [Releases · ssssssss-team/spider-flow (github.com)](https://github.com/ssssssss-team/spider-flow/releases) 下载源码包
* 解压，进行数据初始化，mysql -uroot -p < /home/spider-flow-0.5.0/db/spiderflow.sql
* 修改 /home/spider-flow-0.5.0/spider-flow-web/src/main/resources/application.properties，更改里面的数据库链接密码，修改端口 server.port=18088
* 编辑 /home/spider-flow-0.5.0/spider-flow-web/pom.xml , 把 spider-flow-redis、spider-flow-mongodb、spider-flow-selenium、spider-flow-ocr、spider-flow-oss、spider-flow-mailbox 这几个依赖全都注释掉
* 编译项目 mvn clean install
* cd /home/spider-flow-0.5.0/spider-flow-web 之后，mvn spring-boot:run
*
