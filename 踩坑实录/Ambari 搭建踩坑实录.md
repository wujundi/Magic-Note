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











docker run -itd --name='ubuntu-ambari-1' -p 127.0.0.1:8080:8080 -p 127.0.0.1:8440:8440 -p 127.0.0.1:8441:8441 registry.cn-hangzhou.aliyuncs.com/wujundi/ubuntu-ambari-1

/usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java

mvn -B install jdeb:jdeb -DnewVersion=2.7.7.0.0 -DbuildNumber=388e072381e71c7755673b7743531c03a4d61be8 -DskipTests -Dpython.ver='python >= 2.6'  -Drat.skip=true -e
