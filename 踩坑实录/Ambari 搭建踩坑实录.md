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

在打包过程中遇到了困难，卡在了Ambari Web 2.7.7.0.0，报错信息是：

[ERROR] Failed to execute goal com.github.eirslett:frontend-maven-plugin:1.4:install-node-and-yarn (install node and yarn) on project ambari-web: Could not extract the Node archive: Could not extract archive: '/home/wujundi/.m2/repository/com/github/eirslett/node/4.5.0/node-4.5.0-linux-x64.tar.gz': EOFException -> [Help 1]
