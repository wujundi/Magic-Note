# Hive 源码调试采坑实录

[TOC]

## 1、搭建CDH quickstart VM
事实证明，CDH VM起来之后一段时间是最烧cpu的。启动后不要关闭虚拟机，以后的运行中还是非常和煦的，知识内存占用依然较高

## 2、配置远程调试
https://www.cnblogs.com/wy2325/p/5600232.html

## 3、下载和CDH环境同版本的Hive源码，build项目
CDH代码库，http://archive.cloudera.com/cdh5/cdh/5/
往下翻，有 hive-1.1.0-cdh5.13.0-src.tar.gz 文件（可搜索“hive-1.1.0-cdh5.13.0-”）：http://archive.cloudera.com/cdh5/cdh/5/

## 4、为mac安装jdk
       这东西原来是有的，不知道什么时候升级版本把mac自带的jdk弄没了
       所以到oracle下载了和CDH环境同版本的jdk，但是安装的时候报系统版本问题 https://blog.csdn.net/liutang090510/article/details/55073746
       原理是将安装包解压，修改版本控制函数，然后在重新打包。注意博客中的路径是有些不对的，重在理解思路与命令。

## 5、起远程调试，debug Hive 源码
代码快速格式化，alt + command + L
有关源码依赖报错的问题可以参考：https://my.oschina.net/kavn/blog/867314
Maven配置主副镜像，力求全面的加载依赖，maven镜像好像不能随意改动名称，否则会报错，玄学？找了一个现成的贴上去，好用，没敢再动：http://www.cnblogs.com/ae6623/p/4416256.html
为什么还是有提示找不到的类？
Maven配置已经没有什么好改的了，怀疑是代码和环境问题

出现跳转，虽然代码中还有好多报错的地方，但是好像不影响远程调试
另：代码行数匹配的不好，可能是代码版本不同的原因吗？

是的呢，相当于本地代码只是为了承接断点的符号，实际代码结构，行号，全部以线上为准。
一旦本地代码格式与线上不一致，就不好玩了

cdh hive 的 maven 项目是有 parent 的，也就是说，本质上说，这并不是一个独立的maven项目，我现在的做法是改掉了pom 中的parent，转而找到相关的maven坐标作为依赖，尽可能的保证依赖能够看到东西。 但是，这其实是解决不了根本问题的，本地的hive项目依然不是个独立的maven项目。将其改造成独立maven项目的工作留给以后吧

## 6、IDEA的debug方法
快速上手IDEA的debug功能：https://blog.csdn.net/qq_27093465/article/details/64124330