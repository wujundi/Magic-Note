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
* docker run -itd --privileged ubuntu，安装也没报什么错，service mysql start 的时候报了个错 ，cannot change directory to /nonexistent: No such file or directory 参考 [mysql 服务器关机后出现cannot change directory to /nonexistent: No such file or directory - 工大教务处 - 博客园 (cnblogs.com)](https://www.cnblogs.com/wjx-zjut/p/16367771.html) 解决了
* 重新安装 jdk，配置国内 apt 源，[(87条消息) Ubuntu22.04更换国内镜像源（阿里、网易163、清华、中科大）_ubuntu中科大镜像源_凯der苦练心态的博客-CSDN博客](https://blog.csdn.net/qq_42365082/article/details/127008698)，然后安装 apt install openjdk-8-jdk
* 安装 maven，apt install maven，然后再从之前的容器中把 setting.xml 搞过来
* 下载并解压了 spiderflow 的项目，参考 [关于Java：如何通过命令行启动spring-boot应用程序？ | 码农家园 (codenong.com)](https://www.codenong.com/47835901/) 把启动命令写到了 /home/start-up.sh 里面，这样就不用总是需要记住了，其中参考了 [Installation Guide for Ambari 2.7.7 - Apache Ambari - Apache Software Foundation](https://cwiki.apache.org/confluence/display/AMBARI/Installation+Guide+for+Ambari+2.7.7) 里面的 push 操作

CREATE USER 'bbb'@'%' IDENTIFIED BY '123456'；
