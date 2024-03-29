# Ambari 搭建踩坑实录

## 1、背景调研

* [CDH的免费午餐结束后，“免费”部署之路往何处去？ - HelloWorld开发者社区](https://www.helloworld.net/p/8083304775)

  * 指出 Cloudera 免费版本结束中之后，可以借助 Apache Ambari + Apache BigTop Stack 技术栈实现类似的功能

## 2、虚拟机搭建

* 3台 Ubuntu 22.04.1 桌面操作系统，因为有 GUI ，所以网络配置没有浪费太多时间

## 3、起步的文档准备

* 官网 [Ambari - (apache.org)](https://ambari.apache.org/)

## 4、前提环境搭建

Ambari 官网的 Start 中提到需要一些 "prerequisites"([Installation Guide for Ambari 2.7.7 - Apache Ambari - Apache Software Foundation](https://cwiki.apache.org/confluence/display/AMBARI/Installation+Guide+for+Ambari+2.7.7))，对于这些前置依赖，可以参考对应链接([Ambari Development - Apache Ambari - Apache Software Foundation](https://cwiki.apache.org/confluence/display/AMBARI/Ambari+Development))

当然，在文档之中其实 Ambari 也是给了大家抄作业的机会的，比如文档中提到"you can easily launch a VM that is preconfigured with all the tools that you need.  See the **Pre-Configured Development Environment** section in the [Quick Start Guide](https://cwiki.apache.org/confluence/display/AMBARI/Quick+Start+Guide)." 只不过这部分内容跳转过去是使用 VirtualBox 和命令行的，打包好的启动命令太黑盒了，我还是自己弄吧。

* 前置工作

  1. 安装JDK

     * 在寻找 java 安装位置的时候看到了 readlink 命令，挺有用的 [Ubuntu设置 JAVA_HOME具体方法-良许Linux教程网 (lxlinux.net)](https://www.lxlinux.net/6579.html)
     * 在配置 JAVA_HOME变量的时候遇到了一些问题，Ubuntu自带的vim无法正确的使用方向键，参考了 [(52条消息) 解决ubuntu20.04下vi编辑器方向键和退格键问题_zhoupenghui168的博客-CSDN博客_ubuntu vi 方向键](https://blog.csdn.net/zhoupenghui168/article/details/123499092) 予以解决
  2. 安装 maven

     * maven 的path还没有配置
       * 本身这个 maven 我是通过 apt 来安装的，安装之后 mvn 命令就有反映了，所以我理解这个 path 是不是没有必要再额外配置了，但是我又不能确定这个 path 在后面究竟会被用来干什么，所以又要搞
       * mvn -version 的时候，回复的消息里面有 "Maven home: /usr/share/maven"，所以我就用这个路径来配置 PATH 吧
       * 最终，/home/wujundi/.bashrc 文件中，增加了下三行

         ```
         export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
         export PATH=/usr/share/maven/bin/:$PATH
         # export _JAVA_OPTIONS="-Xmx2048m -XX:MaxPermSize=512m -Djava.awt.headless=true"
         ```
  3. 安装 python setuptools

     * 参考 [python安装setuptools的方法_编程开发_软件教程_脚本之家 (jb51.net)](https://www.jb51.net/softjc/158840.html)
     * apt-get install python-setuptools
  4. 安装 rpmbuild

     * 参考 [ubuntu系统上制作rpm包demo - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/524103077)
     * sudo apt-get install rpm
  5. 安装

     * 直接 sudo apt install g++ 提示报错 The following packages have unmet dependencies: libc6-dev : Depends: libc6 (= 2.35-0ubuntu3) but 2.35-0ubuntu3.1 is to be installed
     * 参考 [(54条消息) Ubuntu 22.04 LTS 解决 libc6-dev 缺少依赖 E: 软件包冲突的问题_肖典泽的博客-CSDN博客](https://blog.csdn.net/xdz233/article/details/127470733) 解决依赖问题，最终成功安装

## 5、下面正式开始搭建

指导链接就是官网的 Installation Guide  https://cwiki.apache.org/confluence/display/AMBARI/Installation+Guide+for+Ambari+2.7.7

1. 第一步，下载 tar.gz 格式的安装包，不得不说，国内的下载还是用 tuna 的镜像更快更方便
2. 第二部开始解压与安装

   ```
   tar xfvz apache-ambari-2.7.7-src.tar.gz
   cd apache-ambari-2.7.7-src
   mvn versions:set -DnewVersion=2.7.7.0.0 # 这里会执行一大波自动化的安装，最终 BUILD SUCCESS

   pushd ambari-metrics # 相当于高级版的 cd ambari-metrics
   mvn versions:set -DnewVersion=2.7.7.0.0 # 这里又会执行一大波自动化的安装，，最终 BUILD SUCCESS
   popd # 将当前目录更改为pushd命令最近存储的目录，在这个语境里，相当于 cd .. 了

   mvn -B clean install jdeb:jdeb -DnewVersion=2.7.7.0.0 -DbuildNumber=388e072381e71c7755673b7743531c03a4d61be8 -DskipTests -Dpython.ver="python >= 2.6" # 这句超出我的理解范畴了


   ```

* 打包过程中遇到了困难，卡在了Ambari Web 2.7.7.0.0，报错信息是：[ERROR] Failed to execute goal com.github.eirslett:frontend-maven-plugin:1.4:install-node-and-yarn (install node and yarn) on project ambari-web: Could not extract the Node archive: Could not extract archive: '/home/wujundi/.m2/repository/com/github/eirslett/node/4.5.0/node-4.5.0-linux-x64.tar.gz': EOFException -> [Help 1]
* 上网搜索了了之后，发现 Could not extract the Node archive: Could not extract archive 是一个比较普遍的报错，一般是由于 maven 库下载过程中的网络问题导致本地 maven 仓库对应的文件不全。解决办法一般就是删除掉报错对应的本地 maven 库文件，重新打包重新下载即可。
* 还是报错卡住，所以这里我转向Ambari的开发文档 [Ambari Development - Apache Ambari - Apache Software Foundation](https://cwiki.apache.org/confluence/display/AMBARI/Ambari+Development) ，尝试寻找有关安装的更多信息
* 结果在 Running Unit Tests 阶段就卡住了，提示没有npm，所以这里先把 npm 安装上，

## 6、我想在本地搭建环境可能是在做无用功，所以上面的步骤会挪到docker里面搞，顺便可以把docker学一学（尤其这玩意现在越来越多的被用到非专业的娱乐领域了，可能以后用得上）

* 所以新的思路就出来，用其中一个Ubuntu机器作为主力机安装docker，然后通过docker来搭建伪集群。为啥不直接用windows wsl？我怕搞的时候把环境搞崩，在VMware里面回滚起来没有风险。
* docker 安装成功，在ubuntu容器的基础上，拷贝进去了ambari的源码进行打包安装，之前 ambari-web部分的报错也通过安装npm解决掉了。
* 期间遇到了一个问题，打包的时候报错 Too many files with unapproved license，通过在maven打包命令中增加 -Drat.skip=true 参数解决了
* 接下来，在打包 anbari-view 的时候遇到的问题，报错是 package javax.ws.rs.core does not exist，暂时还没有弄好，更新了编译环境的 java 版本，从jdk11降级到jdk8，仍然有这个问题，那么我就怀疑是又是下载依赖包的时候没有下载全导致的，所以删除了所有本地仓库文件，全部重新从阿里云拉取，这次就把这个问题解决了
* 打包Storm的时候遇到问题，[INFO] Ambari Metrics Storm Sink (Legacy) 2.7.7.0.0 ....... FAILURE [ 13.035 s]，具体的报错是，[ERROR] Failed to execute goal on project ambari-metrics-storm-sink-legacy: Could not resolve dependencies for project org.apache.ambari:ambari-metrics-storm-sink-legacy:jar:2.7.7.0.0: Failed to collect dependencies at org.apache.storm:storm-core:jar:0.9.3.2.2.1.0-2340: Failed to read artifact descriptor for org.apache.storm:storm-core:jar:0.9.3.2.2.1.0-2340: Could not transfer artifact org.apache.storm:storm-core:pom:0.9.3.2.2.1.0-2340 from/to default-repository (http://repo.maven.apache.org/maven2): Transfer failed for http://repo.maven.apache.org/maven2/org/apache/storm/storm-core/0.9.3.2.2.1.0-2340/storm-core-0.9.3.2.2.1.0-2340.pom 501 HTTPS Required -> [Help 1]。这个问题真的是无解，mvn仓库里面根本就没有这个包，还好在网上找到了 [Ambari2.7.6源码编译 - shine-rainbow - 博客园 (cnblogs.com)](https://www.cnblogs.com/shine-rainbow/p/16149430.html) 这篇帖子，这里面博主附上了他的本地 repo 压缩包，然后又在一些 maven settings.xml 文件的修改之后，终于是把 Ambari Metrics Storm Sink (Legacy) 模块编译通过了
* 然后遇到了 Ambari Metrics Monitor 2.7.7.0.0 模块的编译报错 [ERROR] Failed to execute goal org.apache.maven.plugins:maven-antrun-plugin:1.7:run (psutils-compile) on project ambari-metrics-host-monitoring: An Ant BuildException has occured: exec returned: 1
  [ERROR] around Ant part ...`<exec failonerror="true" dir="/home/apache-ambari-2.7.7-src/ambari-metrics/ambari-metrics-host-monitoring/src/main/python/psutil" executable="/home/apache-ambari-2.7.7-src/ambari-metrics/ambari-metrics-host-monitoring/../../ambari-common/src/main/unix/ambari-python-wrap">`... @ 4:275 in /home/apache-ambari-2.7.7-src/ambari-metrics/ambari-metrics-host-monitoring/target/antrun/build-psutils-compile.xml ，看了一下报错的 /home/apache-ambari-2.7.7-src/ambari-metrics/ambari-metrics-host-monitoring/target/antrun/build-psutils-compile.xml 的文件内容，按照其中 dir 前往相应目录，按照 executable 作为终端命令，在其后面依此粘贴各个 value 的参数，参考 [ambari生成安装手记_ambari an ant buildexception has occured: exec ret_白乔的博客-CSDN博客](https://blog.csdn.net/bluejoe2000/article/details/54914840?locationNum=9&fps=1)，然后收到详细的报错 psutil/_psutil_linux.c:12:10: fatal error: Python.h: No such file or directory
  12 | #include <Python.h>
  |          ^~~~~~~~~~
  compilation terminated.
  error: command 'x86_64-linux-gnu-gcc' failed with exit status 1，看起来好像是 x86_64-linux-gnu-gcc 命令没能成功的执行 .c 文件，网上一搜，原来这是个普遍的问题，是没有安装python开发环境导致的，[(64条消息) xxx: fatal error: Python.h: No such file or directory 完美解决方案_TimeDoor的博客-CSDN博客](https://blog.csdn.net/caokun_8341/article/details/103275606)，所以就按照指令安装了环境
* 编译 ambari-server 模块的时候遇到了一个报错 /usr/bin/env: 'python': No such file or directory [ERROR] Failed to execute goal org.codehaus.mojo:exec-maven-plugin:1.2.1:exec (azuredb-gen) on project ambari-server: Command execution failed. Process exited with an error: 127 (Exit value: 127) -> [Help 1]
  org.apache.maven.lifecycle.LifecycleExecutionException: Failed to execute goal org.codehaus.mojo:exec-maven-plugin:1.2.1:exec (azuredb-gen) on project ambari-server: Command execution failed. 上网一查是一个通用的问题 [(66条消息) 解决：/usr/bin/env: ‘python’: No such file or directory_asjodnobfy的博客-CSDN博客](https://blog.csdn.net/qq_41550190/article/details/119804102)，然后就按照这个方式解决了，其实就是编译的时候，程序没有在 /usr/bin/ 目录下面找到名为 python 的符号链接
* 编译之后安装 ambari-server 的 deb 文件时候，一直报错 The following packages have unmet dependencies:
  ambari-server : Depends: python (>= 2.6) but it is not installable
  E: Unable to correct problems, you have held broken packages. 可是ubuntu自带的包根本就没有这个 python，只要输入 apt-get install python 就是提示 Package python is not available, but is referred to by another package. 试了很多方法，最终考虑到会不会是apt源的问题，在 [ubuntu更换apt源、pip源和conda源 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/366458982) 的指导之下，新加入了aliyun的apt源，然后，apt-get install python 就正常安装了，再然后，安装 ambari-server 的 deb 就命令就可以正常运行了
* 在 ambari-server start 阶段遇到了问题，一直启动不成功，查看 /var/log/ambari-server/ambari-server.out 日志，发现报错了，具体报错是 Caused by: java.io.IOException: ObjectIdentifier() -- data isn't an object ID (tag = 48)，上网查了一下 [(66条消息) Java Azure开发parseAlgParameters failed: ObjectIdentifier() -- data isn‘t an object ID (tag = 48)处理_风行無痕的博客-CSDN博客](https://blog.csdn.net/gmaaa123/article/details/125929375) 可能是 java版本的问题，没想都我选它提示默认的java包，它居然给我报错，真是服了，哈哈，这回重新运行了 ambari-server setup，在 jdk 的环节选了自定义的 jdk，然后参考 [Ubuntu 安装java8 并配置JAVA_HOME – 源码巴士 (code84.com)](https://code84.com/813098.html) 里面寻找 java_home 路径的方式，在 ambari-server setup 运行的过程中，填写了 java_home，运行之后 ps -aux 看到这回有对应的 pid 启动了
* 登录到页面之后，发现检索不出版本的信息，上网查到了这么个解释 https://blog.csdn.net/CREATE_17/article/details/128027911，官方文档里面找到了这样的说明 [Overview - Apache Ambari - Apache Software Foundation](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=38571133)，嗯，那就看看这些 `metainfo.xml` 吧。不看不知道，原来是所有的 `metainfo.xml 里面都是 <active>false</active>，我尝试把其中的一个改成 true，然后 ambari-server restart 了一下，结果页面上就有变化了，我以为问题已经得到解决了，但是。。。安装的时候HDP的 base url 全部都无法访问了，也就是说，把 HDP 版本放出来也没用。然后我放出了代码里面自带的 BIGTOP stack ,结果只有 redhat6的适配，而且url还访问不通，逼得我只能去 pull ambari 最新的源码，最新代码里面目前正在整合 bigtop 3.2.0 的 stack，同样的修改方式，这次页面上并没有出现 bigtop 3.2.0`
* 到这里为止，对于 anbari 2.7.7 的探索基本接近尾声了，2.7.7虽然能够成功编译与安装，但是最关键的 stack 还没有完成向 bigtop 的转变，而 HDP 对于历史版本的"断供"，让 ambari2.7.7 成为了一座空城。

# 7、重试 ambari 2.8.0

* ambari 的 git 项目已经在 3月底打上了 2.8.0 的 tag，pull 代码之后发现，stacks 已经换成了 bigtop 3.2.0，等一等官方发布 2.8.0 版本的安装文档吧
* 说曹操曹操到，官方wiki在20230331放出了[Installation Guide for Ambari 2.8.0 - Apache Ambari - Apache Software Foundation](https://cwiki.apache.org/confluence/display/AMBARI/Installation+Guide+for+Ambari+2.8.0) 2.8 的安装导引，线上的代码包还没有，所以我从 github 上下载了 tag 是 2.8.0 release 的源码压缩包
* 文档中说目前只支持 centos7 的环境，所以我就拉了 centos 的镜像，重头再来
* [INFO] Ambari Admin View .................................. FAILURE [ 10.168 s] [ERROR] Failed to execute goal org.codehaus.mojo:exec-maven-plugin:1.2.1:exec (Bower install) on project ambari-admin: Command execution failed.: Process exited with an error: 1 (Exit value: 1) -> [Help 1]
  org.apache.maven.lifecycle.LifecycleExecutionException: Failed to execute goal org.codehaus.mojo:exec-maven-plugin:1.2.1:exec (Bower install) on project ambari-admin: Command execution failed. 看起来像是 Bower install，更新了 yum 配置之后，安装了 npm 和 bower ,然后在执行到 bower 安装的时候也经常会报错，都是科学上网的问题导致 bower 安装一些包的时候拉不下来代码，安装好了也就没事了
* 

[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary for Ambari Main 2.8.0.0.0:
[INFO]
[INFO] Ambari Main ........................................ SUCCESS [  4.524 s]
[INFO] Apache Ambari Project POM .......................... SUCCESS [  0.290 s]
[INFO] Ambari Web ......................................... SUCCESS [01:43 min]
[INFO] Ambari Views ....................................... SUCCESS [  1.757 s]
[INFO] Ambari Admin View .................................. SUCCESS [ 13.978 s]
[INFO] ambari-utility ..................................... SUCCESS [06:41 min]
[INFO] Ambari Server SPI .................................. SUCCESS [ 22.502 s]
[INFO] Ambari Server ...................................... FAILURE [05:29 min]
[INFO] Ambari Functional Tests ............................ SKIPPED
[INFO] Ambari Agent ....................................... SKIPPED
[INFO] Ambari Service Advisor ............................. SKIPPED
[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  14:38 min
[INFO] Finished at: 2023-04-04T06:14:36Z
[INFO] ------------------------------------------------------------------------
[ERROR] Failed to execute goal on project ambari-server: Could not resolve dependencies for project org.apache.ambari:ambari-server:jar:2.8.0.0.0: Failed to collect dependencies at org.apache.ambari:ambari-serviceadvisor:jar:1.0.0.0-SNAPSHOT: Failed to read artifact descriptor for org.apache.ambari:ambari-serviceadvisor:jar:1.0.0.0-SNAPSHOT: Could not transfer artifact org.apache.ambari:ambari-serviceadvisor:pom:1.0.0.0-SNAPSHOT from/to maven-default-http-blocker (http://0.0.0.0/): Blocked mirror for repositories: [aliyun-repository (http://maven.aliyun.com/nexus/content/groups/public/, default, releases+snapshots)] -> [Help 1]
org.apache.maven.lifecycle.LifecycleExecutionException: Failed to execute goal on project ambari-server: Could not resolve dependencies for project org.apache.ambari:ambari-server:jar:2.8.0.0.0: Failed to collect dependencies at org.apache.ambari:ambari-serviceadvisor:jar:1.0.0.0-SNAPSHOT

在源代码里面搜了一下，发现ambari-server pom 里面对于 ambari-serviceadvisor 的坐标写的是     `<dependency>`
      `<groupId>`org.apache.ambari `</groupId>`
      `<artifactId>`ambari-serviceadvisor `</artifactId>`
      `<version>`1.0.0.0-SNAPSHOT `</version>`
      `<exclusions>`
        `<exclusion>`
          `<groupId>`commons-httpclient `</groupId>`
          `<artifactId>`commons-httpclient `</artifactId>`
        `</exclusion>`
      `</exclusions>`
    `</dependency>`
又搜了一下官方的 maven 坐标 https://mvnrepository.com/artifact/org.apache.ambari/ambari-serviceadvisor，看起来这不像是要用线上坐标的样子，所以我转而先编译 ambari-serviceadvisor 模块，看看是不是 ambari-server 是想以来这个本地项目的编译结果。

我尝试了配置上官方的 mvn 坐标还是没用，那么索性就手动的下 build ambari-serviceadvisor 模块，得到 2.8.0.0.0 的 ambari-serviceadvisor，然后把 ambari-server 的pom里面依赖 ambari-serviceadvisor 的版本直接替换成 2.8.0.0.0，结果就可以了。而且我还发现一个神奇的事情，当我把 ambari-server 的pom里面依赖 ambari-serviceadvisor 的版本直接替换成 2.8.0.0.0 之后，整个项目的编译顺序会随之调整，ambari-serviceadvisor 调整到 ambari-server 之前编译了，神奇。这样调整完就不需要单独手动 build ambari-serviceadvisor 了，直接在项目下 build 项目就可以了。

* ambari-server setup 的时候报了错误 /usr/sbin/ambari-server: line 34: buildNumber: unbound variable , 上网搜了一下，发现这位大佬是在安装之后去修改配置文件的，[Ambari2.7.6安装 - shine-rainbow - 博客园 (cnblogs.com)](https://www.cnblogs.com/shine-rainbow/p/16169205.html) ，那么间接说明这个 buildNumber 应该不是什么关键密钥，应该只是一个变量，所以我重新编译，并且在编译的时候就给这个变量赋值，mvn clean install rpm:rpm -DbuildNumber=test -DskipTests -Drat.skip=true -e，这样重新编译之后就好了、
* setup 的过程中遇到问题 Enter advanced database configuration [y/n] (n)? n
  Configuring database...
  Default properties detected. Using built-in database.
  Configuring ambari database...
  Checking PostgreSQL...
  Running initdb: This may take up to a minute.
  About to start PostgreSQL
  ERROR: Exiting with exit code 1.
  REASON: Unable to start PostgreSQL server. Exiting. 尝试单独启动 postgresql，遇到了这样的问题 Failed to get D-Bus connection: Operation not permitted failed to find PGDATA，然后查询到了 [(66条消息) Docker Centos7- Failed to get D-Bus connection: Operation not permitted failed to find PGDATA_成都-Python开发-王帅的博客-CSDN博客](https://blog.csdn.net/u012798683/article/details/108222670)，使用这些参数重新启动容器之后，ambari-server setup 过程中的问题得到了解决。但是这个解决方法挑环境，有时候行，有时候不行，最终按照 [docker中单容器开启多服务时systemctl引发的血案及破案过程 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/473252130) 里面提到的方法，替换了 systemctl，一劳永逸了

## bigtop的安装

#### allclean

* 按照 ambari 安装指导里面的命令来安装，会报错，Task 'bigtop-select-pkg' not found in root project 'bigtop'. 用 gradlew tasks 命令看了一下，好像确实是没有 bigtop-select-pkg，那就在执行命令中拿掉试试。
* 这个过程全部都要从 https://archive.apache.org 来下载源码包，那个网速一言难尽，这里遇到一个坑，用 aliyun 的 maven repo 会引起报错，我移除了之前配置的 setting.xml 文件之后，才顺利的编译下去
* 在编译 flink 的过程中报错 SEVERE: Step 'google-java-format' found problem in 'src/main/java/org/apache/flink...，参考 [flink-playgrounds官方案例使用注意事项 - maybe兔 - 博客园 (cnblogs.com)](https://www.cnblogs.com/silentgo/p/15603094.html) ,

```
<!--
                <executions>
                    <execution>
                        <id>spotless-check</id>
                        <phase>verify</phase>
                        <goals>
                            <goal>check</goal>
                        </goals>
                    </execution>
                </executions>
-->
```

把 dl 文件夹下面的 flink tar.gz 文件解压，注释掉其 pom 中的部分内容之后，重新打包，然后重新运行，这个解法更多的意义在于解放了思想，apache顶级项目的源码也没有你想的那么严丝合缝，如有必要就解压改代码，然后重新压缩。这里注意 gradle 每次执行都会从 dl 文件夹里里面取压缩文件重新解压，所以直接改解压后的文件是没用的，必须把改动做到 dl 文件夹的压缩包里面

#### toolchain

* 然后到了安装 bigtop 的环节，在安装依赖的 ./gradlew toolchain 环节报错 sudo: puppet: command not found，随之就在 [apache/bigtop: Mirror of Apache Bigtop (github.com)](https://github.com/apache/bigtop#for-developers-building-the-entire-distribution-from-scratch) 的指导下进行 Puppet 的安装，按照指导执行 bigtop_toolchain/bin/puppetize.sh 的时候一直不成功，所以进到这个脚本里面看了一下，原来是卡在了 gem 命令，需要安装 gem，而 gem 自带的源拉不下来代码，所以就需要先配置 gem 的国内源，详见 [(66条消息) gem 添加国内源_gem源_AdleyTales的博客-CSDN博客](https://blog.csdn.net/adley_app/article/details/111733908)，安装好之后，./gradlew toolchain 就可以运行了。
* 可以运行是可以运行，但是也比较麻烦，显示 gradle 源码包拉不下来，导致报错，我另外下载放到了对应的目录里面
* 然后是要修改我的 maven 安装文件，然后会报错 Notice: /Stage[main]/Bigtop_toolchain::Maven/File[/usr/local/maven]: Not removing directory; use 'force' to override
  Notice: /Stage[main]/Bigtop_toolchain::Maven/File[/usr/local/maven]: Not removing directory; use 'force' to override
  Error: Could not remove existing file
  Error: /Stage[main]/Bigtop_toolchain::Maven/File[/usr/local/maven]/ensure: change from directory to link failed: Could not remove existing file
* 所以我在想，toolchain 的目标场景应该是一个什么都没有的“裸机器”，对于已经搭建了一些东西的场景，是不是不太合适。

#### **Recommended build environments** docker

* 在 bigtop 的项目 readme 中提到，[apache/bigtop: Mirror of Apache Bigtop (github.com)](https://github.com/apache/bigtop#for-developers-building-the-entire-distribution-from-scratch)“Bigtop provides "development in the can" environments, using Docker containers. These have the build tools set by the toolchain, as well as the user and build environment configured and cached. All currently supported OSes could be pulled from official Bigtop repository at [https://hub.docker.com/r/bigtop/slaves/tags/](https://hub.docker.com/r/bigtop/slaves/tags/)”这听起来是官方的环境包，这个链接备用吧，暂时感觉这里提供的镜像可能都是大数据搭建的理想培养基，可以省去基础组件的安装过程，我准备拉这个环境来进行 bigtop 部分的编译。官方文档中的启动命令 dockers docker run --rm -u jenkins:jenkins -v `pwd`:/ws --workdir /ws bigtop/slaves:trunk-ubuntu-20.04 bash -l -c './gradlew allclean ; ./gradlew bigtop-groovy-pkg
* 参考 [(67条消息) BigTop3.2.0 大数据组件编译--组件编译_泽芯的博客-CSDN博客](https://blog.csdn.net/m0_48319997/article/details/128101667) 进行编译，相比于 ambari 官方的安装命令，这篇 csdn 文章中的命令能使用国内的源来下载，速度快了非常多，试错成本更低。命令基本就是  ./gradlew xxxx-rpm -PparentDir=/usr/bigtop，例如 ./gradlew hadoop-rpm -PparentDir=/usr/bigtop
* 编译 phoenix 的时候遇到了一个报错 [INFO] Phoenix Client ..................................... FAILURE [02:31 min] [ERROR] Failed to execute goal org.apache.maven.plugins:maven-shade-plugin:3.2.4:shade (default-shaded) on project phoenix-client-hbase-2.4: Error creating shaded jar: error in opening zip file /root/.m2/repository/com/google/guava/listenablefuture/9999.0-empty-to-avoid-conflict-with-guava/listenablefuture-9999.0-empty-to-avoid-conflict-with-guava-sources.jar -> [Help 1]，在这个报错之前日志里面报出了几个 warning ``[WARNING] Could not get sources for com.google.code.findbugs:jsr305:jar:2.0.1:compile [WARNING] Could not get sources for org.apache.hadoop.thirdparty:hadoop-shaded-protobuf_3_7:jar:1.1.1:compile[WARNING] Could not get sources for org.apache.hadoop.thirdparty:hadoop-shaded-guava:jar:1.1.1:compile``，看起来是没下载到东西，开启科学上网试试，还是一样，那么我就自己下载jar包试试，下载 jar 包的时候发现，maven 官方仓库都是有的，只是仓库地址是 https://repo1.maven.org/maven2，这个地址我之前没有配置到 setting 文件里面。然而配置上之后也没有锤子用。从 [(67条消息) guava listenablefuture版本号9999.0-empty-to-avoid-conflict-with-guava的原因_花落的速度的博客-CSDN博客](https://blog.csdn.net/q2878948/article/details/110038191) 里面了解到这个9999包是个空包，那么解题思路是不是让 shade 不要对他做解压就好了？通过 [Tomcat Caused by: java.util.zip.ZipException: error in opening zip file - Stack Overflow](https://stackoverflow.com/questions/54918087/tomcat-caused-by-java-util-zip-zipexception-error-in-opening-zip-file) 的方法，也证实了 listenablefuture-9999.0-empty-to-avoid-conflict-with-guava-sources.jar 确实会被识别成 broken，那么接下来就是如何排掉它了。后来我发现这个 source.jar 在官方仓库里面是没有的，只有阿里云有，那么，索性我就在编译之前把 setting 文件里面的 国内源注释掉，不去下载这个 source jar 自然也就不存在解压这个 source jar 的报错了。这个问题就这样解决了。
* 编译 kafka 的时候报了 Downloading https://services.gradle.org/distributions/gradle-6.8.1-all.zip Exception in thread "main" javax.net.ssl.SSLException: Read timed out 的错误。参考 [gradle plugins/repos/wrapper/tools 国内快速同步下载镜像 - 码农教程 (manongjc.com)](http://www.manongjc.com/detail/23-fsjnqxsbdjddfdt.html) 在 kafka 源码的 gradle/wrapper/gradle-wrapper.properties当中更改了distributionUrl的参数，换成国内源之后，文件下载没什么障碍，项目编译成功。回到 bigtop 的编译，注意重新打包kafka之后，还需要删掉 build 文件夹当中已经解压的kafka源码文件夹，否则就不会从 dl 文件夹 cp tar 包来运行了
* 编译 flink 的时候，在 Runtime web 模块遇到了一个报错 [ERROR] Failed to execute goal com.github.eirslett:frontend-maven-plugin:1.11.0:npm (npm run ci-check) on project flink-runtime-web: Failed to run task: 'npm run ci-check' failed. org.apache.commons.exec.ExecuteException: Process exited with an error: 126 (Exit value: 126) -> [Help 1]
  org.apache.maven.lifecycle.LifecycleExecutionException: Failed to execute goal com.github.eirslett:frontend-maven-plugin:1.11.0:npm (npm run ci-check) on project flink-runtime-web: Failed to run task。对于这个模块 [(67条消息) BigTop3.2.0 大数据组件编译--组件编译_泽芯的博客-CSDN博客](https://blog.csdn.net/m0_48319997/article/details/128101667?spm=1001.2014.3001.5502) 里面正好提到了对于 npm 版本的改动，估计就是这个问题，于是我就按照这里面写的做了改动。修改 flink-1.15.0/flink-runtime-web/pom.xm，在275行 nodeVersion改为v12.22.1，在276行 npmVersion改为6.14.12，果然就成功了，接下来就和上面一样，重新打包，然后用 bigtop 的 gradle 重新编译。
* 编译 tez 的时候遇到了问题 [ERROR] bower ESUDO         Cannot be run with sudo
  [ERROR]
  [ERROR] Additional error details:
  [ERROR] Since bower is a user command, there is no need to execute it with superuser permissions.
  [ERROR] If you're having permission errors when using bower without sudo, please spend a few minutes learning more about how your system should work and make any necessary repairs.
  [ERROR]
  [ERROR] http://www.joyent.com/blog/installing-node-and-npm
  [ERROR] https://gist.github.com/isaacs/579814
  [ERROR]
  [ERROR] You can however run a command with sudo using "--allow-root" option
  [INFO] ------------------------------------------------------------------------
  [INFO] Reactor Summary for tez 0.10.1:
  [INFO]
  [INFO] tez ................................................ SUCCESS [ 59.782 s]
  [INFO] hadoop-shim ........................................ SUCCESS [ 23.265 s]
  [INFO] tez-api ............................................ SUCCESS [ 11.686 s]
  [INFO] tez-build-tools .................................... SUCCESS [  0.174 s]
  [INFO] tez-common ......................................... SUCCESS [  3.358 s]
  [INFO] tez-runtime-internals .............................. SUCCESS [  2.994 s]
  [INFO] tez-runtime-library ................................ SUCCESS [  7.611 s]
  [INFO] tez-mapreduce ...................................... SUCCESS [  2.925 s]
  [INFO] tez-examples ....................................... SUCCESS [  2.023 s]
  [INFO] tez-dag ............................................ SUCCESS [  9.096 s]
  [INFO] tez-tests .......................................... SUCCESS [  3.791 s]
  [INFO] tez-ext-service-tests .............................. SUCCESS [  2.686 s]
  [INFO] tez-ui ............................................. FAILURE [01:53 min]
  [INFO] tez-plugins ........................................ SKIPPED
  [INFO] tez-protobuf-history-plugin ........................ SKIPPED
  [INFO] tez-yarn-timeline-history .......................... SKIPPED
  [INFO] tez-yarn-timeline-history-with-acls ................ SKIPPED
  [INFO] tez-yarn-timeline-cache-plugin ..................... SKIPPED
  [INFO] tez-yarn-timeline-history-with-fs .................. SKIPPED
  [INFO] tez-history-parser ................................. SKIPPED
  [INFO] tez-aux-services ................................... SKIPPED
  [INFO] tez-tools .......................................... SKIPPED
  [INFO] tez-perf-analyzer .................................. SKIPPED
  [INFO] tez-job-analyzer ................................... SKIPPED
  [INFO] tez-javadoc-tools .................................. SKIPPED
  [INFO] hadoop-shim-impls .................................. SKIPPED
  [INFO] hadoop-shim-2.8 .................................... SKIPPED
  [INFO] tez-dist ........................................... SKIPPED
  [INFO] Tez ................................................ SKIPPED
  [INFO] ------------------------------------------------------------------------
  [INFO] BUILD FAILURE
  [INFO] ------------------------------------------------------------------------
  [INFO] Total time:  04:03 min
  [INFO] Finished at: 2023-04-10T03:22:59Z
  [INFO] ------------------------------------------------------------------------
  [ERROR] Failed to execute goal com.github.eirslett:frontend-maven-plugin:1.4:bower (bower install) on project tez-ui: Failed to run task: 'bower install --allow-root=false' failed. org.apache.commons.exec.ExecuteException: Process exited with an error: 1 (Exit value: 1) -> [Help 1]
  [ERROR]
  [ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
  [ERROR] Re-run Maven using the -X switch to enable full debug logging.
  [ERROR]
  [ERROR] For more information about the errors and possible solutions, please read the following articles:
  [ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoFailureException
  [ERROR]
  [ERROR] After correcting the problems, you can resume the build with the command
  [ERROR]   mvn `<args>` -rf :tez-ui 按照之前那篇文章中的指导，把 apache-tez-0.10.1-src/tez-ui/pom.xml 第37行 allow-root-build改为--allow-root=true
* 接下来，还是 tez-ui 模块，下一个问题 [ERROR] bower file-saver.js#1.20150507.2            retry Download of https://github.com/Teleborder/FileSaver.js/archive/b7cf622909258086bc63ad764d08fcaed780ab42.tar.gz failed with EHTTP, trying with git..
  [INFO] bower file-saver.js#1.20150507.2         checkout b7cf622909258086bc63ad764d08fcaed780ab42
  [ERROR] bower file-saver.js#1.20150507.2          ECMDERR Failed to execute "git clone https://github.com/Teleborder/FileSaver.js.git /tmp/66fc555821c50e89a4942c6fb6f73aeb/bower/e75d8091393c079c0030df2840479819-6065-GscqbA --progress", exit code of #128 Dev Containers CLI: RPC pipe not configured. Message: {"args":["git-credential-helper","get"],"stdin":"protocol=https\nhost=github.com\n"} Dev Containers CLI: RPC pipe not configured. Message: {"args":["git-credential-helper","get"],"stdin":"protocol=https\nhost=github.com\n"} Dev Containers CLI: RPC pipe not configured. Message: {"args":["git-credential-helper","erase"],"stdin":"protocol=https\nhost=github.com\nusername=Username for 'https://github.com': \npassword=Password for 'https://Username for 'https://github.com': @github.com': \n"} Dev Containers CLI: RPC pipe not configured. Message: {"args":["git-credential-helper","erase"],"stdin":"protocol=https\nhost=github.com\nusername=Username for 'https://github.com': \npassword=Password for 'https://Username for 'https://github.com': @github.com': \n"} fatal: Authentication failed for 'https://github.com/Teleborder/FileSaver.js.git/'
  [ERROR]
  [ERROR] Additional error details:
  [ERROR] Dev Containers CLI: RPC pipe not configured. Message: {"args":["git-credential-helper","get"],"stdin":"protocol=https\nhost=github.com\n"}
  [ERROR] Dev Containers CLI: RPC pipe not configured. Message: {"args":["git-credential-helper","get"],"stdin":"protocol=https\nhost=github.com\n"}
  [ERROR] Dev Containers CLI: RPC pipe not configured. Message: {"args":["git-credential-helper","erase"],"stdin":"protocol=https\nhost=github.com\nusername=Username for 'https://github.com': \npassword=Password for 'https://Username for 'https://github.com': @github.com': \n"}
  [ERROR] Dev Containers CLI: RPC pipe not configured. Message: {"args":["git-credential-helper","erase"],"stdin":"protocol=https\nhost=github.com\nusername=Username for 'https://github.com': \npassword=Password for 'https://Username for 'https://github.com': @github.com': \n"}
  [ERROR] fatal: Authentication failed for 'https://github.com/Teleborder/FileSaver.js.git/'
  [INFO] ------------------------------------------------------------------------
  [INFO] BUILD FAILURE
  [INFO] ------------------------------------------------------------------------
  [INFO] Total time:  4.185 s
  [INFO] Finished at: 2023-04-10T13:11:33Z
  [INFO] ------------------------------------------------------------------------ 看起来是 http 下载不下来，转而尝试 git 下载，进而报出了没有人家这个库的 git 权限，我尝试自己去请求这网站，结果人家 https://github.com/Teleborder 已经是 [This organization has no public repositories.] 的状态了。我在源码里面搜索了一下，有个这么个记载 file-saver.js v1.20150507.2 (https://github.com/Teleborder/FileSaver.js) - Authored by Eli Grey ， github 上面搜索 [FileSaver.js](https://github.com/eligrey/FileSaver.js) 结果当中 star 最多的 是 [eligrey](https://github.com/eligrey)/[FileSaver.js](https://github.com/eligrey/FileSaver.js)，这么看起来，好像是同一个作者啊，所以我按照 [eligrey/FileSaver.js: An HTML5 saveAs() FileSaver implementation (github.com)](https://github.com/eligrey/FileSaver.js) 中给的命令进行了安装。npm install file-saver --save 然后 bower install file-saver。由于替换了 FileSaver 包，所以一些配置也需要跟着更改：1. git地址需要改 apache-tez-0.10.1-src/tez-ui/src/main/webapp/bower-shrinkwrap.json 文件中的 FileSaver git地址需要改掉；2. 版本号需要改 apache-tez-0.10.1-src/tez-ui/src/main/webapp/bower.json 里面 "file-saver.js": "1.20150507.2" 改成 "file-saver": "1.2.0"； 3. FileSaver.js 文件的安装位置发生了变化 apache-tez-0.10.1-src/tez-ui/src/main/webapp/ember-cli-build.js 中 app.import('bower_components/file-saver.js/FileSaver.js'); 改成 app.import('bower_components/file-saver/FileSaver.js'); 然后再 编译 tez-ui 模块就往下进行了。
* 另外，后续在编译 tez-ui 的时候又遇到了一些零散问题，基本都在 bower 安装的前后，有的是 phatformJs 下载超时，那么我就自己把对应的包复制到 /tmp/phantomjs 位置，有的是 Failed to run task: 'bower install --allow-root=true' failed. 往上一翻日志就是 bower 安装某个包的时候报错了，我就 bower install 手动安装一下。自动安装的实在是有点多，搞得我在 /home 目录下面写了个 shell 脚本来安装。
* 在模块内编译是通过了，但是在 bigtop gradle 执行的时候报了错误 patching file tez-ui/src/main/webapp/bower-shrinkwrap.json
  Hunk #1 FAILED at 2.
  1 out of 2 hunks FAILED -- saving rejects to file tez-ui/src/main/webapp/bower-shrinkwrap.json.rej
  patching file tez-ui/src/main/webapp/bower.json
  Hunk #1 FAILED at 22.
  1 out of 1 hunk FAILED -- saving rejects to file tez-ui/src/main/webapp/bower.json.rej
  patching file tez-ui/src/main/webapp/ember-cli-build.js
  Reversed (or previously applied) patch detected!  Assume -R? [n]
  Apply anyway? [n]
  Skipping patch.
  1 out of 1 hunk ignored -- saving rejects to file tez-ui/src/main/webapp/ember-cli-build.js.rej
  patching file tez-ui/src/main/webapp/bower-shrinkwrap.json
  Hunk #1 succeeded at 27 (offset 3 lines).
  Hunk #2 FAILED at 69.
  1 out of 2 hunks FAILED -- saving rejects to file tez-ui/src/main/webapp/bower-shrinkwrap.json.rej
  patching file tez-ui/src/main/webapp/bower.json
  Hunk #1 FAILED at 23.
  1 out of 1 hunk FAILED -- saving rejects to file tez-ui/src/main/webapp/bower.json.rej
  error: Bad exit status from /var/tmp/rpm-tmp.n1uMUz (%prep)
  Bad exit status from /var/tmp/rpm-tmp.n1uMUz (%prep)，原来上面提到的那个 tez 包下面的 FileSaver 的问题，bigtop 的开发人员也注意到了，并且在 home/bigtop-3.2.0/bigtop-packages/src/common/tez 准备了 diff 文件，要在编译的过程中“魔改”tez 的配置文件，而因为我们已经更改了这些文件，致使 diff 文件中的信息的匹配出现了问题。
* 编译 ranger 的时候提示没有找到，一看原来是 bigtop.bom 里面把 ranger 的部分注释掉了，放开注释就可以开始编译了。

/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.362.b08-1.el7_9.x86_64

docker run -itd --name='ambari-server' -p 8080:8080 -p 8440:8440 -p 8441:8441 registry.cn-hangzhou.aliyuncs.com/wujundi/centos-ambari-neo:runable

docker run -itd --name='ambari-server' --hostname='ambari-server' -p 8080:8080 -p 8440:8440 -p 8441:8441 registry.cn-hangzhou.aliyuncs.com/wujundi/centos-ambari-280:ready-for-install

docker run -itd --name='bigtop320' --privileged -p 2929:2929 registry.cn-hangzhou.aliyuncs.com/wujundi/bigtop3.2.0-centos-7-offical-build-env:ready-for-http /usr/sbin/init

* 最后，在暴露文件夹这块也不是很省心，官网的 python 工具经常不好用，最终换成了 httpd，配置文件在 /etc/httpd/conf/httpd.conf，直接在命令行里面 service httpd start 即可，就可以在浏览器访访问暴露的问价夹了 [Index of /bigtop](http://172.18.254.162:2929/bigtop/)

## UI安装阶段

* Select version 阶段认不出 bigto怕容器的 url ，报错 **Attention:** Please make sure all repository URLs are valid before proceeding. [Click **here** to retry.](javascript:void(null))
* 第一步，我想让容器能固定下来ip，这样方便后面的配置。参考了 [(69条消息) 七.Docker网络管理以及固定ip_docker 固定ip_dears-app的博客-CSDN博客](https://blog.csdn.net/u014295838/article/details/128522006)，和 [(69条消息) 如何修改docker容器的hostname_docker hostname_一木一石的博客-CSDN博客](https://blog.csdn.net/wendaowangqi/article/details/126283213)，把容器启动命令写成了 docker run -d --privileged=true --name='ambari-server' --network='subnet' --ip 177.188.0.101 -p 8080:8080 -p 8440:8440 -p 8441:8441 --hostname='ambari-server' --add-host=ambari-agent-2:177.188.0.102 --add-host=ambari-agent-3:177.188.0.103 registry.cn-hangzhou.aliyuncs.com/wujundi/centos-ambari-neo /sbin/init 这样，然后运行了 ambari-server，但是主机浏览器里面却打不开ammbari的主页。后来不管子网这个事情了，先试一下一个节点能不能顺利运行起来吧。如果一个节点不行，我就用三台IP固定的虚拟机来搞了，docker搞子网太搞心态了。
* 前面好多环节填了各种信息，真要安装的时候遇到报错 http://127.0.0.1:2929/repodata/repomd.xml: [Errno 14]curl#7-"Failed connect to 127.0.0.1:2929; Connection refused"，看来一个 docker 想通过 127.0.0.1 访问另一个 docker 的服务还是不太行啊

## 多台虚拟机环境下的部署

* 192.168.188.101
* 192.168.188.102
* 192.168.188.103
* ssh: connect to host 192.168.188.101 port 22: Connection refused，是因为刚装的 ubuntu 机器没有 ssh 服务，参考 [(72条消息) ssh: connect to host port 22: Connection refused_「已注销」的博客-CSDN博客](https://blog.csdn.net/LU_ZHAO/article/details/104698537)

单节点测试环境：docker run -itd --name='ambari-server' -p 8080:8080 -p 8440:8440 -p 8441:8441 --hostname='ambari-server' registry.cn-hangzhou.aliyuncs.com/wujundi/centos-ambari-neo:ready-for-install

docker run -itd --name='ambari-server' -p 8080:8080 -p 8440:8440 -p 8441:8441 --hostname='ambari-server' --add-host=ambari-server:192.168.188.101 --add-host=ambari-agent-2:192.168.188.102 --add-host=ambari-agent-3:192.168.188.103 registry.cn-hangzhou.aliyuncs.com/wujundi/centos-ambari-neo:ready-for-install

docker run -itd --name='ambari-agent' -p 8080:8080 -p 8440:8440 -p 8441:8441 --hostname='ambari-agent-2' --add-host=ambari-server:192.168.188.101 --add-host=ambari-agent-2:192.168.188.102 --add-host=ambari-agent-3:192.168.188.103 registry.cn-hangzhou.aliyuncs.com/wujundi/centos-ambari-neo:ready-for-install

docker run -itd --name='ambari-agent' -p 8080:8080 -p 8440:8440 -p 8441:8441 --hostname='ambari-agent-3' --add-host=ambari-server:192.168.188.101 --add-host=ambari-agent-2:192.168.188.102 --add-host=ambari-agent-3:192.168.188.103 registry.cn-hangzhou.aliyuncs.com/wujundi/centos-ambari-neo:runable

docker run -itd --name='bigtop320' -p 2929:2929 registry.cn-hangzhou.aliyuncs.com/wujundi/centos-bigtop-3.2.0:ready-for-http

* Confirm Hosts 步骤报错。详细 log 是

```
Warning: Permanently added 'ambari-server,192.168.188.101' (ECDSA) to the list of known hosts.
Permission denied (publickey,password).
SSH command execution finished
host=ambari-server, exitcode=255
Command end time 2023-04-22 14:55:09

ERROR: Bootstrap of host ambari-server fails because previous action finished with non-zero exit code (255)
ERROR MESSAGE: Warning: Permanently added 'ambari-server,192.168.188.101' (ECDSA) to the list of known hosts.
Permission denied (publickey,password).

STDOUT: 
Warning: Permanently added 'ambari-server,192.168.188.101' (ECDSA) to the list of known hosts.
Permission denied (publickey,password).
```

参考 [(72条消息) Ambari安装----Confirm Hosts Registering with the server failed解决办法_一只土肥圆的猿的博客-CSDN博客](https://blog.csdn.net/cp_panda_5/article/details/79993057) 发现是 ambari-agent 的配置文件里面有写死  server 的 hostname，所以参修改了 /etc/ambari-agent/conf/ambari-agent.ini 里面 [server] 模块下面的 hostname=ambari-server

* 官网上提到的 python -m SimpleHTTPServer 在容器运行在linux虚拟机的情况下经常会报错，所以我上网查了一下，改为用 httpd 来搞，参考了 [Linux搭建简单的http文件服务器 - cavan丶keke - 博客园 (cnblogs.com)](https://www.cnblogs.com/cavan2021/p/16498933.html) 和 [httpd/nginx文件共享服务 - 简书 (jianshu.com)](https://www.jianshu.com/p/ff787883c13a) 两个文章，终于把 output 文件夹成功地暴露在 http 请求中了
* 运行到安装程序的第9步，还是报错，

```
2023-04-23 15:15:56,369 - The 'hadoop-hdfs-datanode' component did not advertise a version. This may indicate a problem with the component packaging. However, the stack-select tool was able to report a single version installed (3.2.0). This is the version that will be reported.
2023-04-23 15:15:58,083 - The 'hadoop-hdfs-datanode' component did not advertise a version. This may indicate a problem with the component packaging. However, the stack-select tool was able to report a single version installed (3.2.0). This is the version that will be reported.
Traceback (most recent call last):
  File "/var/lib/ambari-agent/cache/stacks/BIGTOP/3.2.0/services/HDFS/package/scripts/datanode.py", line 172, in <module>
    DataNode().execute()
  File "/usr/lib/ambari-agent/lib/resource_management/libraries/script/script.py", line 355, in execute
    method(env)
  File "/var/lib/ambari-agent/cache/stacks/BIGTOP/3.2.0/services/HDFS/package/scripts/datanode.py", line 52, in install
    self.install_packages(env)
  File "/usr/lib/ambari-agent/lib/resource_management/libraries/script/script.py", line 876, in install_packages
    name = self.format_package_name(package['name'])
  File "/usr/lib/ambari-agent/lib/resource_management/libraries/script/script.py", line 569, in format_package_name
    return self.get_package_from_available(name)
  File "/usr/lib/ambari-agent/lib/resource_management/libraries/script/script.py", line 536, in get_package_from_available
    raise Fail("No package found for {0}(expected name: {1})".format(name, name_with_version))
resource_management.core.exceptions.Fail: No package found for hadoop_${stack_version}(expected name: hadoop_3_2_0)
```

仔细看了一下，script.py 是一个 python 脚本，貌似会遍历 bigtop 文件夹，并且会从文件名中通过正则来识别版本信息。那么第一个猜想就来了，是不是之前组件编译的时候选择了 ./gradlew hadoop-rpm -PparentDir=/usr/bigtop 的方式进而导致，和官方文档里面的 `./gradlew zookeeper-pkg` 这样的命令，致使最终的编译文件名称出现了差异呢？./gradlew clean zookeeper-pkg -Dbuildwithdeps=true -PparentDir=/usr/bigtop 重新编译之后，还是同样的一些文件。所以这里就怀疑是不是编译的结果有问题，按照 ambari 官方给的编译命令 ./gradlew allclean bigtop-groovy-pkg bigtop-jsvc-pkg bigtop-select-pkg bigtop-utils-pkg flink-pkg hadoop-pkg hbase-pkg hive-pkg kafka-pkg solr-pkg spark-pkg tez-pkg zeppelin-pkg zookeeper-pkg -Dbuildwithdeps=true -PparentDir=/usr/bigtop -PpkgSuffix 进行编译之后，就能进行下去了。

* 在安装过程中，安装到 hive 的时候报了一个错误，resource_management.core.exceptions.Fail: Failed to download file from http://ambari-server:8080/resources/mysql-connector-java.jar due to HTTP error: HTTP Error 404: Not Found，按照 [(73条消息) ambari错误及解决方案_smartsense2.7.6_董不懂22的博客-CSDN博客](https://blog.csdn.net/github_39319229/article/details/113052828) 的方式 yum install -y mysql-connector-java 然后 cp /usr/share/java/mysql-connector-java.jar /var/lib/ambari-server/resources/

## 部署成功

* 对外，单节点部署步骤
* docker run -itd --name='ambari-server' -p 8080:8080-p 8440:8440 -p 8441:8441 -p 9092:9092 --hostname='ambari-server' wujundi/centos-ambari-280:one-node
* docker exec -it ambari-server /bin/bash
* ambari-server start
* ambari-agent start
* 浏览器，http://{your-machine-ip}:8080，用户名 admin，密码 admin

docker run -itd --name='ambari' -p 8080:8080 -p 50070:50070 -p 8088:8088 -p 19888:19888 -p 3000:3000 -p 10002:10002 -p 16010:16010 -p 18081:18081 -p 9995:9995 -p 8082:8082 -p 8983:8983 -p 8440:8440 -p 8441:8441 -p 9092:9092 -p 4040:4040 -p 4041:4041 -p 9000:9000 -p 10086:10086 -p 5432:5432 -p 8081:8081 -p 8030:8030 -p 9020:9020 -p 9030:9030 -p 9010:9010 -p 9050:9050 -p 9060:9060 -p 8040:8040 -p 8060:8060 --hostname='ambari-server' --privileged registry.cn-hangzhou.aliyuncs.com/wujundi/centos-ambari-2.8.0:neo

## 环境联调部分

* 命令行执行hive报错：org.apache.hadoop.security.AccessControlException: Permission denied: user=root, access=WRITE, inode="/user":hdfs:hdfs:drwxr-xr-x，参考了 [(83条消息) 解决 Permission denied: user=root, access=WRITE, inode=“/user“:hdfs:supergroup:drwxr-xr-x_HiBoyljw的博客-CSDN博客](https://blog.csdn.net/HiBoyljw/article/details/120222455) 先是 su hdfs ，然后 输入hadoop fs -chmod 777 /user 来进行授权，就可以了
* 运行 spark-shell 报错，23/06/02 15:56:35 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
  23/06/02 15:56:35 WARN HiveConf: HiveConf of name hive.load.data.owner does not exist
  23/06/02 15:56:35 WARN DomainSocketFactory: The short-circuit local reads feature cannot be used because libhadoop cannot be loaded.
  23/06/02 15:56:36 WARN Utils: Service 'SparkUI' could not bind on port 4040. Attempting port 4041.
  23/06/02 15:56:37 WARN Client: Neither spark.yarn.jars nor spark.yarn.archive is set, falling back to uploading libraries under SPARK_HOME. 重启了一下 spark 组件，重启过程中，貌似自己就好了？然后重启完成又不行了，这是什么鬼？启动期间只要访问 http://127.0.0.1:4041 ，控制台和网页页面会同时显示报错

| URI:       | /jobs/                                                                                                                                           |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| STATUS:    | 500                                                                                                                                              |
| MESSAGE:   | java.util.NoSuchElementException: Failed to get the application information. If you are starting up Spark, please wait a while until it's ready. |
| SERVLET:   | org.apache.spark.ui.JettyUtils$$anon$1-23706db8                                                                                                  |
| CAUSED BY: | java.util.NoSuchElementException: Failed to get the application information. If you are starting up Spark, please wait a while until it's ready. |

### Caused by:

java.util.NoSuchElementException: Failed to get the application information. If you are starting up Spark, please wait a while until it's ready.
	at org.apache.spark.status.AppStatusStore.applicationInfo(AppStatusStore.scala:51)
	at org.apache.spark.ui.jobs.AllJobsPage.render(AllJobsPage.scala:276)
	at org.apache.spark.ui.WebUI.$anonfun$attachPage$1(WebUI.scala:90)
	at org.apache.spark.ui.JettyUtils$$anon$1.doGet(JettyUtils.scala:81)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:503)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:590)
	at org.sparkproject.jetty.servlet.ServletHolder.handle(ServletHolder.java:799)
	at org.sparkproject.jetty.servlet.ServletHandler$ChainEnd.doFilter(ServletHandler.java:1631)
	at org.apache.spark.ui.HttpSecurityFilter.doFilter(HttpSecurityFilter.scala:95)
	at org.sparkproject.jetty.servlet.FilterHolder.doFilter(FilterHolder.java:193)
	at org.sparkproject.jetty.servlet.ServletHandler$Chain.doFilter(ServletHandler.java:1601)
	at org.sparkproject.jetty.servlet.ServletHandler.doHandle(ServletHandler.java:548)
	at org.sparkproject.jetty.server.handler.ScopedHandler.nextHandle(ScopedHandler.java:233)
	at org.sparkproject.jetty.server.handler.ContextHandler.doHandle(ContextHandler.java:1434)
	at org.sparkproject.jetty.server.handler.ScopedHandler.nextScope(ScopedHandler.java:188)
	at org.sparkproject.jetty.servlet.ServletHandler.doScope(ServletHandler.java:501)
	at org.sparkproject.jetty.server.handler.ScopedHandler.nextScope(ScopedHandler.java:186)
	at org.sparkproject.jetty.server.handler.ContextHandler.doScope(ContextHandler.java:1349)
	at org.sparkproject.jetty.server.handler.ScopedHandler.handle(ScopedHandler.java:141)
	at org.sparkproject.jetty.server.handler.gzip.GzipHandler.handle(GzipHandler.java:763)
	at org.sparkproject.jetty.server.handler.ContextHandlerCollection.handle(ContextHandlerCollection.java:234)
	at org.sparkproject.jetty.server.handler.HandlerWrapper.handle(HandlerWrapper.java:127)
	at org.sparkproject.jetty.server.Server.handle(Server.java:516)
	at org.sparkproject.jetty.server.HttpChannel.lambda$handle$1(HttpChannel.java:400)
	at org.sparkproject.jetty.server.HttpChannel.dispatch(HttpChannel.java:645)
	at org.sparkproject.jetty.server.HttpChannel.handle(HttpChannel.java:392)
	at org.sparkproject.jetty.server.HttpConnection.onFillable(HttpConnection.java:277)
	at org.sparkproject.jetty.io.AbstractConnection$ReadCallback.succeeded(AbstractConnection.java:311)
	at org.sparkproject.jetty.io.FillInterest.fillable(FillInterest.java:105)
	at org.sparkproject.jetty.io.ChannelEndPoint$1.run(ChannelEndPoint.java:104)
	at org.sparkproject.jetty.util.thread.strategy.EatWhatYouKill.runTask(EatWhatYouKill.java:338)
	at org.sparkproject.jetty.util.thread.strategy.EatWhatYouKill.doProduce(EatWhatYouKill.java:315)
	at org.sparkproject.jetty.util.thread.strategy.EatWhatYouKill.tryProduce(EatWhatYouKill.java:173)
	at org.sparkproject.jetty.util.thread.strategy.EatWhatYouKill.produce(EatWhatYouKill.java:137)
	at org.sparkproject.jetty.util.thread.QueuedThreadPool.runJob(QueuedThreadPool.java:883)
	at org.sparkproject.jetty.util.thread.QueuedThreadPool$Runner.run(QueuedThreadPool.java:1034)

# NEO重装计划

## 捞 centos 镜像

* 第一步，先把之前版本的镜像捞起来，docker pull registry.cn-hangzhou.aliyuncs.com/wujundi/centos-ambari-2.8.0:ready-for-install，然后在这个版本之上，先安装 mysql 5.7 ，docker run -itd --name='ambari' -p 3306:3306 -p 8080:8080 -p 50070:50070 -p 8088:8088 -p 19888:19888 -p 3000:3000 -p 10002:10002 -p 16010:16010 -p 18081:18081 -p 9995:9995 -p 8082:8082 -p 8983:8983 -p 8440:8440 -p 8441:8441 -p 9092:9092 -p 4040:4040 -p 4041:4041 -p 9000:9000 -p 10086:10086 -p 5432:5432 -p 8081:8081 -p 8030:8030 -p 9020:9020 -p 9030:9030 -p 9010:9010 -p 9050:9050 -p 9060:9060 -p 8040:8040 -p 8060:8060  --hostname='ambari-server' --privileged registry.cn-hangzhou.aliyuncs.com/wujundi/centos-ambari-2.8.0:ready-for-install
* 后来我又想了想，既然要搞。。。要不干脆搞个大的， 参照 [apache/bigtop: Mirror of Apache Bigtop (github.com)](https://github.com/apache/bigtop) 里面提供的 [bigtop/slaves Tags | Docker Hub](https://hub.docker.com/r/bigtop/slaves/tags/)，直接拷贝一个 centos 环境下来。是的要重新搞了，就搞个彻彻底底，干干净净。
* 首先，选型，bigtop 3.2.0 提供的 centos 环境镜像有三个，分别对应 ppc64le、arm64 和 amd64，基于 [ppc64，ppc64le，ARM，AMD，X86，i386，x86_64（AMD64），AArch64的概念_ppc64el_ac.char的博客-CSDN博客](https://blog.csdn.net/sunny_day_day/article/details/108714946) 的解释，我在电脑上使用，应该是选用 AMD64 的，搞他！ docker pull bigtop/slaves:3.2.0-centos-7，然后把它传到阿里云，启一个新的 tag 叫 neo-new
* 启动它，docker run -itd --name='ambari' -p 8000:8000 -p 8080:8080 -p 50070:50070 -p 8088:8088 -p 19888:19888 -p 3000:3000 -p 10002:10002 -p 16010:16010 -p 18081:18081 -p 9995:9995 -p 8082:8082 -p 8983:8983 -p 8440:8440 -p 8441:8441 -p 9092:9092 -p 4040:4040 -p 4041:4041 -p 9000:9000 -p 10086:10086 -p 5432:5432 -p 8081:8081 -p 18030:18030 -p 19020:19020 -p 19030:19030 -p 19010:19010 -p 19050:19050 -p 19060:19060 -p 18040:18040 -p 18060:18060 -p 18088:18088 --hostname='ambari-server' --privileged registry.cn-hangzhou.aliyuncs.com/wujundi/centos-ambari-2.8.0:neo-prd
* 看一下 centos 版本，cat /etc/centos-release，返回 CentOS Linux release 7.9.2009 (Core) ，哦了

## 安装 mysql 5.7

* 检查是不是预装了 mysql，service mysql start，返回 Redirecting to /bin/systemctl start mysql.service，同时报出了 Failed to get D-Bus connection: Operation not permitted，这个之前已经遇到过，还是参照 [docker中单容器开启多服务时systemctl引发的血案及破案过程 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/473252130)，首先备份现有的 /bin/systemctl，cp /bin/systemctl /bin/systemctl.bak，然后替换systemctl，将其替换成github上的这个，执行 wget https://raw.githubusercontent.com/gdraheim/docker-systemctl-replacement/master/files/docker/systemctl.py -O /bin/systemctl，然后赋权 sudo chmod a+x /bin/systemctl。验证效果，再次运行 service mysql start，已经不再报 D-BUS 错误了。
* 安装新的 MySQL。这参考文档就茫茫多了，参考官方文档，[MySQL :: A Quick Guide to Using the MySQL Yum Repository](https://dev.mysql.com/doc/mysql-yum-repo-quick-guide/en/)，首先向yum仓库中增加mysql信息， wget https://dev.mysql.com/get/mysql80-community-release-el7-7.noarch.rpm，然后 sudo rpm -Uvh https://dev.mysql.com/get/mysql80-community-release-el7-7.noarch.rpm。然后选择要安装的版本，sudo yum-config-manager --disable mysql80-community ，和 sudo yum-config-manager --enable mysql57-community，然后再确认一下要安装的版本是不是就是想要的版本，yum repolist enabled | grep mysql，再然后执行安装就行了 sudo yum install mysql-community-server。
* 然后测试一下，mysql -uroot -p，由于没设置过密码，想着是不是可以不输入密码直接回车，结果报了个错误，ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: NO)，原来之前安装文档里面讲了，安装的时候默认为root设置了一个密码，可以通过 sudo grep 'temporary password' /var/log/mysqld.log 查看，用这个密码登录之后，ALTER USER 'root'@'localhost' IDENTIFIED BY '123456'; 结果提示 ERROR 1819 (HY000): Your password does not satisfy the current policy requirements，参考 [(94条消息) ERROR 1819 (HY000): Your password does not satisfy the current policy requirements_Stackflowed的博客-CSDN博客](https://blog.csdn.net/yuxielea/article/details/103800379)，先设置一个密码，用于继续后续操作 ALTER USER 'root'@'localhost' IDENTIFIED BY '#Mysql123'; 然后修改密码格式限制等级 SHOW VARIABLES LIKE 'validate_password%'; set global validate_password_policy=LOW; set global validate_password_length=4; SHOW VARIABLES LIKE 'validate_password%'。设置完成之后，最后重置密码 ALTER USER 'root'@'localhost' IDENTIFIED BY 'mysql';

## 安装 ambari 2.8.0

### 编译

* 参考官方教程 [Installation Guide for Ambari 2.8.0 - Apache Ambari - Apache Software Foundation](https://cwiki.apache.org/confluence/display/AMBARI/Installation+Guide+for+Ambari+2.8.0)
* 如前所述，官方还没有放出 2.8.0 的下载，需要从 github 里面，按照 tag 来下载，地址在这里 [Release release-2.8.0-rc0 · apache/ambari (github.com)](https://github.com/apache/ambari/releases/tag/release-2.8.0-rc0)，执行下载 wget https://github.com/apache/ambari/archive/refs/tags/release-2.8.0-rc0.tar.gz
* 然后就是解压和 mvn 编译，mvn clean install rpm:rpm -DskipTests -Drat.skip=true -e
* 没想到开着科学上网，还是会遇到 Failed to execute goal org.codehaus.mojo:exec-maven-plugin:1.2.1:exec (Bower install) on project ambari-admin: Command execution failed.: Process exited with an error: 1 (Exit value: 1) -> [Help 1] 我尝试单独打包 ambari-admin 模块，也是一个鸟样子。bower 的问题，报的是证书过期，在网上找了一下相关的问题，找到这么一个，[(94条消息) 解决 Ant Sword-HTTPS 证书失效（cert_has_expired）连不上问题_Adminxe的博客-CSDN博客 (coder.work)](https://www.coder.work/article/6892990)  ，还有 [Bower install CERT_UNTRUSTED Error - Stack Overflow](https://stackoverflow.com/questions/26375489/bower-install-cert-untrusted-error)，原来 bower 可以配置不用强校验证书的，哈哈哈哈。然后找一下 ambari-admin 里面这个配置文件在哪里 .bowerrc，通过文件目录寻找，发现在 /home/ambari-release-2.8.0-rc0/ambari-admin/src/main/resources/ui/admin-web/.bowerrc，改了之后，ambari-admin 的打包就顺利通过了，nice！然后，这样修改之后，在编译整个项目的时候，会致使 ambari-web 项目打包出错。所以这并不是根本上解决问题的办法。又参考了 [github.com/bower/bower/issues/2608](https://github.com/bower/bower/issues/2608) 和 [docker - How to refresh bower package certificate? - Stack Overflow](https://stackoverflow.com/questions/76089161/how-to-refresh-bower-package-certificate) ，我意识到，可能需要升级 node 和 npm 的版本就能解决 bower 报证书失效的问题，于是我参考 [Previous Releases | Node.js (nodejs.org)](https://nodejs.org/zh-cn/download/releases) 在 /home/ambari-release-2.8.0-rc0/ambari-admin/pom.xml 中将 `<nodeVersion>`v4.5.0 `</nodeVersion>` 和 `<npmVersion>`2.15.0 `</npmVersion>` 改成了 `<nodeVersion>`v12.22.12 `</nodeVersion>` 和 `<npmVersion>`6.14.16 `</npmVersion>` ，同时开启科学上网，编译安装，bower 部分非常丝滑，但是到了 Gulp build 的部分却报了错误，[INFO] --- exec-maven-plugin:1.2.1:exec (Gulp build) @ ambari-admin ---
  fs.js:36} = primordials; ReferenceError: primordials is not defined at fs.js:36:5 ，看起来 node 版本的升级，牵一发而动全身，查询了 [Previous Releases | Node.js (csdn.net)](https://blog.csdn.net/fairyier/article/details/118329465) 之后，发现问题可能是 node 和 npm 的版本太激进，导致和默认的 gulp 版本不太适配了，于是又降了一下，选择 `<nodeVersion>`v10.24.1 `</nodeVersion>` 和 `<npmVersion>`6.14.12 `</npmVersion>` 这个搭配方式，亲测成功。
* 改完之后，整体编译，编译到 ambari-web 的时候遇到报错， [ERROR] Failed to execute goal org.codehaus.mojo:exec-maven-plugin:1.2.1:exec (Brunch build) on project ambari-web: Command execution failed.: Process exited with an error: 137 (Exit value: 137) -> [Help 1] org.apache.maven.lifecycle.LifecycleExecutionException: Failed to execute goal org.codehaus.mojo:exec-maven-plugin:1.2.1:exec (Brunch build) on project ambari-web: Command execution failed. 经过尝试之后，明确了是 node 版本不一致引发的问题，注意刚刚 ambari-admin 那里改完 node 的版本之后，ambari-web 这边也得成对应的版本，在 /home/ambari-release-2.8.0-rc0/ambari-web/pom.xml 中，修改第 95 行，`<nodeVersion>`v10.24.1 `</nodeVersion>`，然后重新整体编译。
* 然后还有 serviceadvisor 的依赖问题，要把 ambari-server 的pom里面依赖 ambari-serviceadvisor 的版本直接替换成ambari-serviceadvisor 的版本号 2.8.0.0.0

### 安装rpm

* 安装编译号的 rpm 包 yum install /home/ambari-release-2.8.0-rc0/ambari-server/target/rpm/ambari-server/RPMS/x86_64/ambari-server-2.8.0.0-0.x86_64.rpm，顺利无报错。
* 然后进行 ambari-server 的设置，ambari-server setup，报了错误 /usr/sbin/ambari-server: line 34: buildNumber: unbound variable , 上网搜了一下，发现这位大佬是在安装之后去修改配置文件的，[Ambari2.7.6安装 - shine-rainbow - 博客园 (cnblogs.com)](https://www.cnblogs.com/shine-rainbow/p/16169205.html) ，那么间接说明这个 buildNumber 应该不是什么关键密钥，应该只是一个变量，所以我重新编译，并且在编译的时候就给这个变量赋值，mvn clean install rpm:rpm -DbuildNumber=test -DskipTests -Drat.skip=true -e，这样重新编译之后就好了
* 在 setup 的过程中，尝试在数据库环节，配置自己搭建的 mysql，遇到一个问题，需要 Enter full path to custom jdbc driver:，所以我还得下载一个 jdbc driver，官网地址是 [MySQL :: Download Connector/J](https://dev.mysql.com/downloads/connector/j/)，对应到具体的链接 wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-j-8.0.33.tar.gz，解压之后，我就把 /home/mysql-connector-j-8.0.33/mysql-connector-j-8.0.33.jar 这个文件的绝对路径填到了 Enter full path to custom jdbc driver: 后面。然后 提示我 ，WARNING: Before starting Ambari Server, you must run the following DDL directly from the database shell to create the schema: /var/lib/ambari-server/resources/Ambari-DDL-MySQL-CREATE.sql，看来自动建表什么的 ambari 是搞不了的，那算了，ambari 的元数据还是用它嵌入的 postgresql 吧。
* 接下来安装 ambari-agent ，yum install /home/ambari-release-2.8.0-rc0/ambari-agent/target/rpm/ambari-agent/RPMS/x86_64/ambari-agent-2.8.0.0-0.x86_64.rpm
* 然后修改 /etc/ambari-agent/conf/ambari-agent.ini，让 agent 认得 server 的ip，把 [server] hostname=localhost 改成 [server] hostname=ambari-server，这样一来以后在启动容器的时候指定容器的 hostname 为 ambari-server 就可以了。

### UI安装

* 先把 bigtop 数据源弄好，docker run -itd --name='bigtop320' --privileged -p 2929:2929 registry.cn-hangzhou.aliyuncs.com/wujundi/centos-bigtop-3.2.0:ready-for-http /usr/sbin/init，然后在容器中 service httpd start ，然后访问 [Index of /bigtop](http://172.18.254.162:2929/bigtop/)，这里的IP 172.18.254.162 是 WSL 的IP地址。
* 在安装到第6步骤的时候，有两个密码需要设置，分别是 Grafana Admin ，用户名是 admin，密码我设置的是 admin；Hive Database，用户名是 hive，密码我设置的是 hive。
* 在安装到第7步的时候，需要选择 hive 元数据的存放位置。这里如果选择了 new mysql 那就走回了老路，默认安装的是 mysql 5.6 ，到后边安装其他组件的时候就傻眼了。所以这次，我坚决选择 Existing MySQL，下面就提示我 To use MySQL with Hive, you must [download the https://dev.mysql.com/downloads/connector/j/ from MySQL](https://dev.mysql.com/downloads/connector/j/). Once downloaded to the Ambari Server host, run:ambari-server setup --jdbc-db=mysql --jdbc-driver=/path/to/mysql/mysql-connector-java.jar，于是我就照做，开始下载 mysql 的连接驱动。官网地址是 [MySQL :: Download Connector/J](https://dev.mysql.com/downloads/connector/j/)，对应到具体的链接 wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-j-8.0.33.tar.gz，解压之后驱动文件在 /home/mysql-connector-j-8.0.33/mysql-connector-j-8.0.33.jar，然后我执行 ambari-server setup --jdbc-db=mysql --jdbc-driver=/home/mysql-connector-j-8.0.33/mysql-connector-j-8.0.33.jar，然后还得把刚才设置的名叫 hive 的用户添加到 mysql 里面。执行 mysql -uroot -p，密码是之前设置 'mysql'，然后参考 [mysql怎么添加新的用户 - 老白网络 (yzktw.com.cn)](https://www.yzktw.com.cn/post/976271.html)，执行 CREATE USER 'hive'@'localhost' IDENTIFIED BY 'hive'; GRANT ALL PRIVILEGES ON * . * TO 'hive'@'localhost'; 又遇到密码政策的报错 ERROR 1819 (HY000): Your password does not satisfy the current policy requirements ，参考上文，又执行了一下，SHOW VARIABLES LIKE 'validate_password%'; set global validate_password_policy=LOW; set global validate_password_length=4; SHOW VARIABLES LIKE 'validate_password%'，然后再次 CREATE USER 'hive'@'localhost' IDENTIFIED BY 'hive'; GRANT ALL PRIVILEGES ON * . * TO 'hive'@'localhost'; create datebase hive;
* 然后运行页面上的 Test Connection 还是 fail，参考 [MySql链接url参数详解 - 下雨天MJ - 博客园 (cnblogs.com)](https://www.cnblogs.com/love-MJ/p/5980447.html) 修改了一下 Database URL ，加上了端口，改成了 jdbc:mysql://localhost:3306/hive，再次 Test ，通过。
* 然后就是在网页UI上，下一步下一步下一步。
* 搞定。
