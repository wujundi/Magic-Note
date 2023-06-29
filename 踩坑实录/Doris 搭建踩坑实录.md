# Doris 搭建踩坑实录

* 按照官方教程进行 [通用编译 - Apache Doris](https://doris.apache.org/zh-CN/docs/dev/install/source-install/compilation-general)。不得不说，doris 这种提供专用编译镜像的行为，真的很友好。
* docker run -itd --name doris  apache/doris:build-env-for-1.2
* 上传到阿里云之后，编译结果可以在容器里面找到，docker run -itd --name='doris' registry.cn-hangzhou.aliyuncs.com/wujundi/centos-doris-1.2.4.1:latest

## 安装部署

整体的流程还是参考官方文档，[标准部署 - Apache Doris](https://doris.apache.org/zh-CN/docs/dev/install/standard-deployment)，但是这里如果遇到一些不会操作的地方，还是得需要更细致的SOP，所以我找到了 [(93条消息) 最新Doris安装部署（保姆级教程）_蜗牛大白牙的博客-CSDN博客](https://blog.csdn.net/u013618714/article/details/130743031) 这么一篇。

* 首先把doris这边编译出来的文件夹拷贝过来
* 然后，修改linux环境，关闭swap，注意，原来的容器当中，就没有 etc/fatab 文件，我这里建了一个没有内容的 etc/fatab，然后用 swapoff -a 在不重启之前，临时关掉 swap，但是遇到了报错 swapoff: Not superuser.  我怀疑是容器的 root 权限不够，我重新用 --privileged 参数启动一下试试，果然啊，swapoff -a 顺利执行

## FE

* 然后检查了一下 /home/doris-output/fe/conf/fe.conf，发现其实也没有什么要改动的
* 然后执行安装文件 sh /home/doris-output/fe/bin/start_fe.sh --daemon，过一会 vscode 提示 9020 端口已经投入使用，/home/doris-output/fe/log/fe.log 日志看起来没什么报错

## BE

* /home/doris-output/be/conf/be.conf 看起来也没有什么需要改的
* 然后，居然要使用 可以使用 mysql-client 连接到 FE。？？？这是什么操作？？？参考了 [centos7 yum仅安装mysql client客户端 - 参码踪 (shenmazong.com)](https://www.shenmazong.com/blog/1640010740671610880) 的操作，安装了  mysql client。我大概明白了，fe 自身相当于一个数据库，它会存储每一个 be 的 ip 和端口，而操作 fe 向其自身插入数据这件事情，是需要一个数据库链接客户端来做的，官方文档这里选择的工具就是 mysql client。mysql -h 127.0.0.1 -P 9030 -uroot 就看到进入到 Server version: 5.7.99 Doris version doris-1.2.4-1-Unknown 了。然后执行 ALTER SYSTEM ADD BACKEND "127.0.0.1:9050";
* 然后执行 sh /home/doris-output/be/bin/start_be.sh --daemon，报了一个错误 Please set vm.max_map_count to be 2000000 under root using 'sysctl -w vm.max_map_count=2000000'. 搜了两篇相关的文章，[(93条消息) apache doris数据库搭建（一）_Hello.Reader的博客-CSDN博客](https://blog.csdn.net/weixin_43114209/article/details/131395344) 和 [(93条消息) Doris集群的安装部署_doris安装部署_Xlucas的博客-CSDN博客](https://blog.csdn.net/paicMis/article/details/130178291)，好像也没有提到什么特别的解法，都是照做而已，那我也先照做试试。
* 再次启动，没有报错。
* 去 FE 节点来看 BE 状态 mysql -h 127.0.0.1 -P 9030 -uroot，然后 SHOW PROC '/backends'; 发现 Alive 字段是 false，报错信息是 (172.17.0.2)[INTERNAL_ERROR]actual backend local ip: 172.17.0.2
* 参考 [Doris浅略介绍 +部署+使用 (yii666.com)](https://www.yii666.com/blog/368463.html?action=onAll)，修改 BE 节点信息，先删除，alter system decommission backend "127.0.0.1:9050"，然后新加 ALTER SYSTEM ADD BACKEND "172.17.0.2:9050"
* 浏览器访问 [Apache Doris](http://127.0.0.1:8030/home) 有效，用户名和密码就是doris fe 节点数据库的用户名和密码，即默认是 root 密码是空
