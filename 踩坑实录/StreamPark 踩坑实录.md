## StreamPark 踩坑实录

* 在 [Apache StreamPark (incubating)](https://streampark.apache.org/zh-CN/download/) 选择源码进行下载
* 所以我找到了 StreamPark [安装部署 | Apache StreamPark (incubating)](https://streampark.apache.org/zh-CN/docs/user-guide/deployment)
* 这里涉及到一个环境变量的问题，之前安装 ambari 的时候并没有注意环境变量，我是真的不知道 ambari 在安装的时候是往哪些目录里面安装的，所以这里我只能重新搞这两个镜像，这次一定要在安装界面截图保存。

docker run -itd --name='bigtop320' -p 2929:2929 registry.cn-hangzhou.aliyuncs.com/wujundi/centos-bigtop-3.2.0:ready-for-http

docker run -itd --name='ambari-server' --hostname='ambari-server' -p 8080:8080 -p 8440:8440 -p 8441:8441 registry.cn-hangzhou.aliyuncs.com/wujundi/centos-ambari-280:ready-for-install，然后我还发现。安装流程走到后面，其实会有一个 blueprint 确认的步骤，并且还可以下载 blueprint.zip 压缩包，这个压缩包里面的 json 文件详细的记录了各个组件的配置文档模板，还有各种各样的参数。我在里面搜索 HADOOP_HOME 发现其实有很多行都有这个关键字，看前后文的注释也能看出来，各个组件安装的时候，好像也经常需要先 export 一下这个 HADOOP_HOME 变量，由此我更加相信，ambari 一定是设置过 HADOOP_HOME 的。那么在哪里呢？我又想到像之前操作 HDFS 的时候，就需要切换到专有的用户 hdfs，然后才能操作。那么，会不会 ambari 在安装各个组件的时候，其实也是以各个特定用户的身份进行安装的呢？如果是这样的话，那么 /home 下属的各个文件夹，是不是正式这些用户的家目录呢？按其中时候会有 .bashrc 这类文件承担着配置系统变量的任务呢？答案是，没有。home下的这些文件夹虽然看起来像是用户家目录，但是其中的 .bashrc 等几个文件的内容却空空如也。

于是，我想到了，搜索大法，我用 vs code 从根目录开始，搜索全部带有 HADOOP_HOME 的文件，然后，我就打开了新世界的大门。不仅找到了很多 HADOOP_HOME 的赋值信息，还发现之前在 blueprint.json 里面见到的很多变量，比如 {{hadoop_home}}、{{hadoop_yarn_home}} 这些，在实际的文件中是已经被替换成文件目录了的。然后我从 root/.bashrc 里面看到有 source/etc/profile，进而找到了 /etc/profile，这里也是之前配置 JAVA_HOME 地方，按照 [安装部署 | Apache StreamPark (incubating)](https://streampark.apache.org/zh-CN/docs/user-guide/deployment/) 里面提到了，我配置了这几个环境变量，并且把他们的值替换成了在docker中搜索到的实际值。看起来都挺规整的，基本是 /usr/bigtop/3.2.0/usr/lib/ 下面的一级目录，只有 hbase 特殊一些，我搜到的是 export HBASE_HOME=/usr/bigtop/current/hbase-client，但我感觉是不是也和其他的是一个规则，应该是 exportHBASE_HOME=/usr/bigtop/3.2.0/usr/lib/hbase？？？先用后者吧，有问题记得回来看这一条。

* 在数据库的设置环节，参照之前 ambari 安装页面上的信息，修改了 driver-class-name: com.mysql.jdbc.Driver，并且依然沿用了 按照 [(73条消息) ambari错误及解决方案_smartsense2.7.6_董不懂22的博客-CSDN博客](https://blog.csdn.net/github_39319229/article/details/113052828) 的方式 yum install -y mysql-connector-java ,然后按照 StreamPark 的要求复制了一份到它的lib目录 cp /usr/share/java/mysql-connector-java.jar /home/apache-streampark_2.12-2.1.0-incubating-bin/lib/。
* 参考 [Linux下通过shell进MySQL执行SQL或导入脚本 - zifeiy - 博客园 (cnblogs.com)](https://www.cnblogs.com/zifeiy/p/9981253.html) 执行数据初始化脚本。mysql < mysql-schema.sql ，报错 ERROR 1067 (42000) at line 28: Invalid default value for 'create_time'，搜了一下 [(86条消息) mysql为datetime类型的字段设置默认值current_timestamp，引发 Invalid default value for 错误_mysql datetime current_一滴水的眼泪的博客-CSDN博客](https://blog.csdn.net/qq_35112567/article/details/111677052) 所可能是版本太低，mysql --version 一下，果然 Ver 15.1 Distrib 5.5.68-MariaDB, for Linux (x86_64) using readline 5.1，那么，不动现有版本的情况下能解吗？参照 [mysql - MariaDB 中的日期时间 current_timestamp - IT工具网 (coder.work)](https://www.coder.work/article/2468851)，我把 current_timestamp 都改成了 current_timestamp() 的写法，结果还是报错
* 醉了，转而试试 postgresql，参考 [PostgreSQL基本使用方法 - 简书 (jianshu.com)](https://www.jianshu.com/p/814fc0f880b8)，psql < xxxx.sql 或者 psql -f xxxx.sql 都可以。执行倒是也不报错，但是怎么也看不到数据库里面数据有变化呢？在我导入SQL之后，我尝试在 psql 里面重新执行见表语句，提示我表已经存在，那么，建到哪里去了呢？在psql里面使用 \d 终于看到了，原来这个 public 库并不会真正建一个库，所以在 \l 命令的时候，不会限制 public ，而 \d 时候可以直接看到这些表。
* 在 streampark 的 bin 问价夹下面执行 startup.sh 遇到了报错 ERROR: streampark.workspace.local: "/opt/streampark_workspace" is invalid path, Please reconfigure in application.yml 和 NOTE: "streampark.workspace.local" Do not set under APP_HOME(/home/apache-streampark_2.12-2.1.0-incubating-bin). Set it to a secure directory outside of APP_HOME. 可能是因为目前还没有 /opt/streampark_workspace目录，有点搓哦，没有的话不能自动创建一下吗？创建了目录之后，问题解决。成功启动
* 最后，访问 10000 端口，没反应，丢
* 在 /home/apache-streampark_2.12-2.1.0-incubating-bin/logs/streampark.out 看了后台日志，org.postgresql.util.PSQLException: FATAL: Ident authentication failed for user "postgres"，感觉是没有 postgres 用户的权限，再往深了看，Caused by: org.postgresql.util.PSQLException: Connection to localhost:5432 refused. Check that the hostname and port are correct and that the postmaster is accepting TCP/IP connections. 这么说是端口没有映射的问题？映射了端口，也还是报 Ident authentication failed for user "postgres" 的错误。参考了 [【PostgreSQL】FATAL: Ident authentication failed for user - 简书 (jianshu.com)](https://www.jianshu.com/p/d41d09fe18e2) 找到了配置文件在 /var/lib/pgsql/data/pg_hba.conf，然后修改了一下，确实，这次就不再报相同的错误了。
* 新的报错是，database "streampark" does not exist，看起来是数据初始化做的不太对。我手动创建了数据库，然后再启动的时候又报错 relation "t_flink_app" does not exist，所以，是不是整套的见表和数据灌入都需要换到这个库？postgresql 有个模式的概念，库>模式>表，现在的状态是，库是 postgres，模式 public ,表是各个表名。然后 /home/apache-streampark_2.12-2.1.0-incubating-bin/conf/application-pgsql.yml 里面配置的链接属性，url 看起来是在查 streampark 库，但是之前的初始化 SQL，好像没有换库的操作，都是直接干，那么表和数据，其实就灌到了 postgresql 库下面。解决办法就是在见表和数据初始化的 SQL 最前面都加上 \c streampark，先转换到 streampark 库，再见表或者插入数据。
* 这些之后都可以了，在启动，日志报错是 Web server failed to start. Port 10000 was already in use. 看来只剩下端口问题了。然后我修改了 /home/apache-streampark_2.12-2.1.0-incubating-bin/conf/application.yml 里面的 port，修改成了 10086
* 登录到UI之后，我就开始配置页面里面的各种变量。首先是 FLINK_HOME，在容器里面搜了半天我也搜不到哪里实例化了FLINK_HOME，干脆，参考其他组件的位置，直接在 /etc/profile 里面配置了 export FLINK_HOME=/usr/bigtop/3.2.0/usr/lib/flink，streampark UI 页面里面，我对 FLINK_HOME 配置的也是 /usr/bigtop/3.2.0/usr/lib/flink
* Streampark 里面，Setting -> Flink Cluter 里面需要配置执行模式，选择了 yarn 模式之后，它就开始检查环境了，我这边还得手动启动 ambari 里面的服务
* 水 Streampark 页面的时候，在进入 System -> Token Management 的时候会有报错，internal server error: ### Error querying database. Cause: org.postgresql.util.PSQLException: ERROR: relation "t_access_token" does not exist Position: 31 ### The error may exist in class path resource [mapper/system/AccessTokenMapper.xml] ### The error may involve org.apache.streampark.console.system.mapper.AccessTokenMapper.page-Inline ### The error occurred while setting parameters ### SQL: select count(*) as total from t_access_token t1 join t_user t2 on t1.user_id = t2.user_id ### Cause: org.postgresql.util.PSQLException: ERROR: relation "t_access_token" does not exist Position: 31 ; bad SQL grammar []; nested exception is org.postgresql.util.PSQLException: ERROR: relation "t_access_token" does not exist Position: 31。看了一下 postgresql 的表里面，还真的没有 t_access_token 表，检查了一下初始化脚本，对 t_access_token ，有建表语句，没有灌数语句。发现是原来脚本里面的 create sequence if not exists "public"."streampark_t_access_token_id_seq" 报错，ERROR:  syntax error at or near "not" LINE 20: create sequence if not exists "public"."streampark_t_access_...导致后续 t_access_token 的建表语句失败。
  去掉 if not exists 之后就好了，推测是版本问题，可能不支持 if not exists。我从建表的sql里面摘出来了 t_access_token 相关的部分，建了一个 pgsql-schema-fix.sql 脚本，然后执行了一把 psql < pgsql-schema-fix.sql。再回到界面，问题解决

## NEO第二遍安装

* [Apache StreamPark (incubating)](https://streampark.apache.org/zh-CN/download/) 页面，找到 apache 的下载 cdn
* 下载，解压
* 参考 [安装部署 | Apache StreamPark (incubating)](https://streampark.apache.org/zh-CN/docs/user-guide/deployment/)，通过在容器内继续搜索，确定各个组件的 HOME 变量的地址，新增到 /etc/profile ，然后 source /etc/profile。
* 修改 /home/apache-streampark_2.11-2.1.1-incubating-bin/conf/application.yml，修改 profiles.active 为 mysql，修改 server:port 为 10086
* 创建目录 mkdir /opt/streampark_workspace，对应的是 /home/apache-streampark_2.11-2.1.1-incubating-bin/conf/application.yml 里面配置的 workspace: local: 后面配置的值
* 修改 /home/apache-streampark_2.11-2.1.1-incubating-bin/conf/application-mysql.yml，把数据库链接密码改掉
* 下载 MySQL jdbc 驱动，官网地址是 [MySQL :: Download Connector/J](https://dev.mysql.com/downloads/connector/j/)，对应到具体的链接 wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-j-8.0.33.tar.gz，解压之后，cp /home/mysql-connector-j-8.0.33/mysql-connector-j-8.0.33.jar /home/apache-streampark_2.11-2.1.1-incubating-bin/lib
* 初始化数据库 mysql -uroot -p < /home/apache-streampark_2.11-2.1.1-incubating-bin/script/schema/mysql-schema.sql
* 导入数据 mysql -uroot -p < /home/apache-streampark_2.11-2.1.1-incubating-bin/script/data/mysql-data.sql
* sh /home/apache-streampark_2.11-2.1.1-incubating-bin/bin/startup.sh

## 平台的使用

* 先到 /StreamPark/Setting 下面设置基础信息，配置 FLINK_HOME 的时候遇到了报错，java.lang.UnsupportedOperationException: The current Scala version of StreamPark is 2.11.12, but the scala version of Flink to be added is 2.12, which does not match, Please check
  at org.apache.streampark.console.core.entity.FlinkEnv.doSetVersion(FlinkEnv.java:81)，看了一下 [(96条消息) StreamX添加Flink引擎时对scala版本的映射选择_flink scala 版本映射_江畔独步的博客-CSDN博客](https://blog.csdn.net/liuwei0376/article/details/124936342)，原来人家项目的源码就给了两套包，一套for scala 2.11，一套for scala 2.12，所以我重新下载了对应 2.12 版本 scala 的包，然后从之前的文件夹中，把修改好的配置文件拷贝过来，重新运行项目，这次 flink home 添加成功了。
* 编辑任务，点击 Verify 的时候会报错，SQL check error，internal server error: No match found，到后台 /opt/apache-streampark_2.12-2.1.1-incubating-bin/logs/info.2023-08-02.log 里面看到了实际的报错日志

2023-08-0213:10:06.171 StreamPark [XNIO-1 task-1] INFO  o.a.s.c.b.h.GlobalExceptionHandler:54 - Internal server error：

java.lang.IllegalStateException: No match found

    at java.util.regex.Matcher.group(Matcher.java:536)

    at org.apache.streampark.common.conf.FlinkVersion.majorVersion$lzycompute(FlinkVersion.scala:99)

    at org.apache.streampark.common.conf.FlinkVersion.majorVersion(FlinkVersion.scala:93)

    at org.apache.streampark.common.conf.FlinkVersion.toString(FlinkVersion.scala:137)

    at java.lang.String.valueOf(String.java:2994)

* 这看起来有点像正则匹配的报错啊。。顺着报错，我下载了源码，追到了 /opt/apache-streampark-2.1.1-incubating-src/streampark-common/src/main/scala/org/apache/streampark/common/conf/FlinkVersion.scala 里面，代码里面各种命令转了一圈，是要执行 java -classpath /usr/bigtop/3.2.0/usr/lib/flink/lib/flink-dist-1.15.3.jar org.apache.flink.client.cli.CliFrontend --version，并且通过返回的值进行正则匹配。
* 我尝试手动执行这个命令，遇到了报错 SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
  SLF4J: Defaulting to no-operation (NOP) logger implementation
  SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
  Exception in thread "main" java.lang.RuntimeException: The configuration directory was not specified. Please specify the directory containing the configuration file through the 'FLINK_CONF_DIR' environment variable.
  at org.apache.flink.client.cli.CliFrontend.getConfigurationDirectoryFromEnv(CliFrontend.java:1197)
  at org.apache.f-incubating-src/streampark-flink/streampark-flink-shims/streampark-fl。其中的'shims'字样，在 streampark的报错里面也又见过，看来方向是对的。从[windows运行FLINK出现 FLINK_CONF_DIR问题__Mr. White的博客-CSDN博客](https://blog.csdn.net/weixin_43918355/article/details/118083976) 处受到启发，补充配置了一个 FLINK_CONF_DIR 的环境变量 exportFLINK_CONF_DIR=${FLINK_HOME}/conf，source /etc/profile 然后再试试，不报错了，但是也没有显示版本，而是打印的
  SLF4J: Defaulting to no-operation (NOP) logger implementation
  SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
  Version: `<unknown>`, Commit ID: DeadD0d0
* 我试了一下 flink -v 返回的也是 Version: `<unknown>`, Commit ID: DeadD0d，要不我看看 flink -v 读取的究竟是什么，我给手动写死一下？通过 whereis flink，找到了 /usr/bin/flink ，里面写的是 exec /usr/bigtop/3.2.0/usr/lib/flink/bin/flink "@"，参考 [sell编程常用知识点_exec &#34;$@_shelutai的博客-CSDN博客](https://blog.csdn.net/shelutai/article/details/122739544)，这就是将 -v 作为参数传了过来
* 按照番外部分的经验来看的话，是不是 version 显示不出来也是 jar 包的问题？从 /usr/bigtop/3.2.0/usr/lib/flink/bin/flink 来看，最后一句是 exec "${JAVA_RUN}"$JVM_ARGS$FLINK_ENV_JAVA_OPTS"${log_setting[@]}"-classpath"`manglePathList "$CC_CLASSPATH:$INTERNAL_HADOOP_CLASSPATHS"`"org.apache.flink.client.cli.CliFrontend"$@" 这句话里除了一些变量之外，关键的类就是 org.apache.flink.client.cli.CliFrontend 了，去源码里面找找。源码在 /home/bigtop-3.2.0/dl/flink-1.15.3/flink-clients/src/main/java/org/apache/flink/client/cli/CliFrontend.java 这里还确实又 switch 里面在处理 -v 和 --version。这个类使用了 /home/bigtop-3.2.0/dl/flink-1.15.3/flink-runtime/src/main/java/org/apache/flink/runtime/util/EnvironmentInformation.java 这里面就定义了那个【unknown】和【DeadD0d0】。这个类里面的环境信息最终是读取的 /home/bigtop-3.2.0/dl/flink-1.15.3/flink-runtime/src/main/resources-filtered/.flink-runtime.version.properties，也就是编译后的 /home/bigtop-3.2.0/dl/flink-1.15.3/flink-runtime/target/classes/.flink-runtime.version.properties。
* 但是这个配置文件，在 bigtop 的打包里面到底在哪里，我不太知道。只能开启搜索大法了。但是并没有搜到。
* 那就拷贝过来吧，放到了 /usr/bigtop/3.2.0/usr/lib/flink/conf/.flink-runtime.version.properties 这里，重启 flink，还是不行
* 我去 bigtop 打包 的 flink 的 rpm 逐个解压，rpm2cpio xxxx.rpm | cpio -div，结果发现里面的东西就和 /usr/bigtop/3.2.0/usr/lib/flink 这里没有什么区别。如果实在不行。。。我打算修改源码进行硬编码了。
* 我把源码里面的 /home/bigtop-3.2.0/dl/flink-1.15.3/flink-clients/src/main/java/org/apache/flink/client/cli/CliFrontend.java 的 -v 部分的取值给改了，改成暴力写死 "1.15.3"，然后打包重新走 bigtop 的编译流程。说是简单一句话，重新打包恐生变数啊。
* bigtop 编译的 hadoop 阶段报错，[INFO] error triple-beam@1.4.1: The engine "node" is incompatible with this module. Expected version ">= 14.0.0". Got "12.22.1"，看起来 node 要安装的部分组件其所需的版本已经迭代了。这种问题还真有解法，[error The engine “node“ is incompatible with this module. Expected version “^12.20.0 || ＞=14“. Got “_徐同保的博客-CSDN博客](https://blog.csdn.net/xutongbao/article/details/123071192) 比如这里提到的 yarn config set ignore-engines true。那我要怎么在 bigtop 打包的时候把这个命令做进去呢？在 hadoop 源码里面搜到 triple-beam@1.3.0，也就是说人家本来要安装的是 1.3.0 的版本，那么为什么实际安装的时候变成了 1.4.1 了呢？报错的前的一条是 angular-route@1.6.10，这个玩意在 /home/bigtop-3.2.0/dl/hadoop-3.3.4-src/hadoop-yarn-project/hadoop-yarn/hadoop-yarn-applications/hadoop-yarn-applications-catalog/hadoop-yarn-applications-catalog-webapp/package.json 里面的配置是 "angular-route": "~1.6.4" 这个写法是啥意思？参考 [node-npm 依赖包版本号~和^的区别_【南汐】前端的博客-CSDN博客](https://blog.csdn.net/weixin_55639808/article/details/129747114)，这个写法确实会拉到 1.6.xx 里面最新的包，这不是坑人么。。。你怎么能确定人家小版本迭代就能符合你的打包要求？我决定把配置里面的 ~ 和 ^ 都去掉试试。然而改了之后，angular-route 确实是 1.6.4 了，但是 triple-beam@1.4.1。要不我显式指定 triple-beam@1.3.0？试了一下也没有什么用。这么推测下来是依赖的这几个包他们的上有也有依赖 triple-beam，但是应该是没有强限制其小版本，致使出现现在引用到了 triple-beam@1.4.1 的情况，最终参考了 [nodejs前端项目 如何显式指定某个依赖的版本 resolutions 字段 + npm-force-resolutions 插件 package-lock.json_锦天的博客-CSDN博客](https://blog.csdn.net/wuyujin1997/article/details/129098567) 在 package.json 里面增加了 "resolutions": {"triple-beam":"1.3.0"}，顺利过关！！！
* bigtop 顺利编译完毕之后，我准备在现有的 ambari 里面先删除 flink，然后再重新安装 flink。先把 bigtop 的文件暴露服务启动起来，然后把 ambari 的全部服务都启动起来，然后再 ambari 的 UI 页面上直接就可以操作。重新安装之后 flink -v 还是 unknown。这个流程根本就没有读取 bigtop。。就是表面上删掉了 flink 的服务，实际上对于代码不会有本质的改变。要动真格的，还得是重装。
* 重装回来之后还是报错，不过报的不是同一个错误了。现在是

2023-08-0802:20:06.291 StreamPark [XNIO-1 task-3] ERRORo.a.s.c.c.s.impl.FlinkSqlServiceImpl:205 - verifySql invocationTargetException: java.lang.reflect.InvocationTargetException

    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)

    at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)

    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)

    at java.lang.reflect.Method.invoke(Method.java:498)

    at org.apache.streampark.console.core.service.impl.FlinkSqlServiceImpl.lambda$verifySql$0(FlinkSqlServiceImpl.java:199)

    at org.apache.streampark.flink.proxy.FlinkShimsProxy$.$anonfun$proxyVerifySql$1(FlinkShimsProxy.scala:157)

    at org.apache.streampark.common.util.ClassLoaderUtils$.runAsClassLoader(ClassLoaderUtils.scala:38)

    at org.apache.streampark.flink.proxy.FlinkShimsProxy$.proxyVerifySql(FlinkShimsProxy.scala:157)

    at org.apache.streampark.flink.proxy.FlinkShimsProxy.proxyVerifySql(FlinkShimsProxy.scala)

    at org.apache.streampark.console.core.service.impl.FlinkSqlServiceImpl.verifySql(FlinkSqlServiceImpl.java:192)

    at org.apache.streampark.console.core.service.impl.FlinkSqlServiceImpl\$\$FastClassBySpringCGLIB$$d657a89c.invoke(\<generated>)

    at org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:218)

.....

Caused by: java.lang.NoClassDefFoundError: org/apache/calcite/sql/validate/SqlConformance

    at org.apache.streampark.flink.core.FlinkSqlValidator.verifySql(FlinkSqlValidator.scala)

    ... 138 more

Caused by: java.lang.ClassNotFoundException: org.apache.calcite.sql.validate.SqlConformance

    at java.net.URLClassLoader.findClass(URLClassLoader.java:387)

* 按照上面的报错，找到了这么一个类 /opt/apache-streampark-2.1.1-incubating-src/streampark-console/streampark-console-service/src/main/java/org/apache/streampark/console/core/service/impl/FlinkSqlServiceImpl.java，它所加载引用的类是 apache-streampark-2.1.1-incubating-src/streampark-flink/streampark-flink-shims/streampark-flink-shims-base/src/main/scala/org/apache/streampark/flink/core/FlinkSqlValidator.scala。报错里面最上层是 Caused by: java.lang.ClassNotFoundException: org.apache.calcite.sql.validate.SqlConformanceCaused by: java.lang.ClassNotFoundException: org.apache.calcite.sql.validate.SqlConformance，说的就是这个 FlinkSqlValidator.scala 里面定义的。我打算通过 maven 打包的方式把这个jar包搞下来，然后放到 lib 目录看看效果。FlinkSqlValidator.scala 所在的子项目中并没有再pom中出现 calcite。而且子项目编译出来的 jar 包已经存在在 lib 目录当中了。那看起来就是缺的应该是其上游的东西，以至于都不需要打在包内。再仔细看，verifysql() 方法里面 calcite 的加载好像通过 private[this] val FLINK112_CALCITE_PARSER_CLASS= "org.apache.flink.table.planner.calcite.CalciteParser" 和 private[this] val FLINK113_PLUS_CALCITE_PARSER_CLASS="org.apache.flink.table.planner.parse.CalciteParser" 这么两个变量来的，莫非又是 flink 的 jar 包不全？果然在flink源码里面找到了 /home/bigtop-3.2.0/dl/flink-1.15.3/flink-table/flink-table-planner/src/main/java/org/apache/flink/table/planner/parse/CalciteParser.java 这个类和 /opt/NOAH_source_reference/flink-1.15.3_md/flink-table/flink-table-planner/src/main/java/org/apache/flink/table/planner/parse/CalciteParser.java这个类 ，这个模块的编译结果在 /home/bigtop-3.2.0/dl/flink-1.15.3/flink-table/flink-table-planner/target/flink-table-planner_2.12-1.15.3.jar 这里。然后就好了，ambari老哥啊，你到底是少拷贝了多少 flink 的 jar 包啊。。。

## 番外

* 番外，因为这个 flink -v 的异常，我注意到 flink 管理页面的异常，这和 [基于 Flink CDC 构建 MySQL 和 Postgres 的 Streaming ETL — CDC Connectors for Apache Flink® documentation (ververica.github.io)](https://ververica.github.io/flink-cdc-connectors/master/content/%E5%BF%AB%E9%80%9F%E4%B8%8A%E6%89%8B/mysql-postgres-tutorial-zh.html) 里面给出的 Flink Web UI 可是不太一样，所以我怀疑，是不是我的 flink 环境本身还有问题？
* 于是，我执行了 bash /usr/bigtop/3.2.0/usr/lib/flink/bin/sql-client.sh 果然也没有成功，报错是 find: ‘/usr/bigtop/3.2.0/usr/lib/flink/opt’: No such file or directory
  find: ‘/usr/bigtop/3.2.0/usr/lib/flink/opt’: No such file or directory
  [ERROR] Flink SQL Client JAR file 'flink-sql-client*.jar' neither found in classpath nor /opt directory should be located in /usr/bigtop/3.2.0/usr/lib/flink/opt.
* 所以我先建了一个 opt/ 的文件夹
* 然后我跑回到 bigtop 镜像里面，重新编译了 flink 终于在 /home/bigtop-3.2.0/dl/flink-1.15.3/flink-table/flink-sql-client/target/flink-sql-client-1.15.3.jar 这里找到了编译出来的 jar 包。然后把它拷贝到了 /usr/bigtop/3.2.0/usr/lib/flink/opt 下面
* 再执行，报了新的错，哈哈 Exception in thread "main" org.apache.flink.table.client.SqlClientException: Failed to parse the Python command line options.
  at org.apache.flink.table.client.cli.CliOptionsParser.getPythonConfiguration(CliOptionsParser.java:387)
  at org.apache.flink.table.client.cli.CliOptionsParser.parseEmbeddedModeClient(CliOptionsParser.java:276)
  at org.apache.flink.table.client.SqlClient.startClient(SqlClient.java:181)
  at org.apache.flink.table.client.SqlClient.main(SqlClient.java:161)
  Caused by: java.lang.ClassNotFoundException: org.apache.flink.python.util.PythonDependencyUtils
  at java.net.URLClassLoader.findClass(URLClassLoader.java:387)
  at java.lang.ClassLoader.loadClass(ClassLoader.java:418)
  at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:352)
  at java.lang.ClassLoader.loadClass(ClassLoader.java:351)
  at java.lang.Class.forName0(Native Method)
  at java.lang.Class.forName(Class.java:348)
  at org.apache.flink.table.client.cli.CliOptionsParser.getPythonConfiguration(CliOptionsParser.java:376)
  ... 3 more
* 顺着报错找到 flink-1.15.3/flink-table/flink-sql-client/src/main/java/org/apache/flink/table/client/cli/CliOptionsParser.java 这个类，看代码，它是需要调用 "org.apache.flink.python.util.PythonDependencyUtils" 的，那我去 flink 源码里找找有没有对应模块，下面有没有编译出来的 jar 包。然后我找到了这个 /home/bigtop-3.2.0/dl/flink-1.15.3/flink-python/target/flink-python_2.12-1.15.3.jar，然后我把它拷贝到了 /usr/bigtop/3.2.0/usr/lib/flink/lib/flink-python_2.12-1.15.3.jar。在运行就成功啦，哈哈哈哈，开心。
