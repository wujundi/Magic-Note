# 传智7天版Hadoop入门踩坑实录

标签（空格分隔）： Hadoop

---

[toc]

---

## 第一天 运行环境搭建与 Hadoop 单机 hello world

---

###1.准备Linux环境

#### 1.0 VMware设置
点击VMware快捷方式，右键打开文件所在位置 -> 双击vmnetcfg.exe -> VMnet1 host-only ->修改subnet ip 设置网段：192.168.1.0 子网掩码：255.255.255.0 -> apply -> ok
		回到windows --> 打开网络和共享中心 -> 更改适配器设置 -> 右键VMnet1 -> 属性 -> 双击IPv4 -> 设置windows的IP：192.168.1.110 子网掩码：255.255.255.0 -> 点击确定
		在虚拟软件上 --My Computer -> 选中虚拟机 -> 右键 -> settings -> network adapter -> host only -> ok	
		
>**xbb**:此处建议选择host only模式，后续会在HDFS管理界面出现理想的结果
如果选择NAT，后续虚拟机节点的Ip会识别为宿主机的localhost

#### 1.1修改主机名
		vim /etc/sysconfig/network
		
		NETWORKING=yes 
		HOSTNAME=itcast01    ###
		
>**xbb**：主机名称不能含有下划线“_”，否则将导致hadoop启动时不能识别主机名

#### 1.2修改IP
		两种方式：
##### 第一种：通过Linux图形界面进行修改（强烈推荐）
			进入Linux图形界面 -> 右键点击右上方的两个小电脑 -> 点击Edit connections -> 选中当前网络System eth0 -> 点击edit按钮 -> 选择IPv4 -> method选择为manual -> 点击add按钮 -> 添加IP：192.168.1.119 子网掩码：255.255.255.0 网关：192.168.1.1 -> apply
	
##### 第二种：修改配置文件方式（屌丝程序猿专用）
			vim /etc/sysconfig/network-scripts/ifcfg-eth0
			
			DEVICE="eth0"
			BOOTPROTO="static"           ###
			HWADDR="00:0C:29:3C:BF:E7"
			IPV6INIT="yes"
			NM_CONTROLLED="yes"
			ONBOOT="yes"
			TYPE="Ethernet"
			UUID="ce22eeca-ecde-4536-8cc2-ef0dc36d4a8c"
			IPADDR="192.168.1.44"       ###
			NETMASK="255.255.255.0"      ###
			GATEWAY="192.168.1.1"        ###
			
#### 1.3修改主机名和IP的映射关系
		vim /etc/hosts
			
		192.168.1.44	itcast01

	
#### 1.4关闭防火墙
		#查看防火墙状态
		service iptables status
		#关闭防火墙
		service iptables stop
		#查看防火墙开机启动状态
		chkconfig iptables --list
		#关闭防火墙开机启动
		chkconfig iptables off
	
#### 1.5重启Linux
		reboot
	
### 2.安装JDK
	2.1上传
	
	2.2解压jdk
		#创建文件夹
		mkdir /usr/java
		#解压
		tar -zxvf jdk-7u55-linux-i586.tar.gz -C /usr/java/
		
	2.3将java添加到环境变量中
		vim /etc/profile
		#在文件最后添加
		export JAVA_HOME=/usr/java/jdk1.7.0_55
		export PATH=$PATH:$JAVA_HOME/bin
	
		#刷新配置
		source /etc/profile
		
>费劲，直接yum安装很方便

### 3.安装Hadoop
#### 3.1上传hadoop安装包
	
#### 3.2解压hadoop安装包
		mkdir /cloud
		#解压到/cloud/目录下
		tar -zxvf hadoop-2.2.0.tar.gz -C /cloud/
		
#### 3.3修改配置文件（5个）
* 第一个：hadoop-env.sh
		#在27行修改
		export JAVA_HOME=/usr/java/jdk1.7.0_55
		
* 第二个：core-site.xml
```
<configuration>
	<!-- 指定HDFS老大（namenode）的通信地址 -->
	<property>
		<name>fs.defaultFS</name>
		<value>hdfs://itcast01:9000</value>
		<!-- 此处地址的主机名若换成IP将导致datanode无法启动 -->
	</property>
	<!-- 指定hadoop运行时产生文件的存储路径 -->
	<property>
		<name>hadoop.tmp.dir</name>
		<value>/cloud/hadoop-2.2.0/tmp</value>
    </property>
</configuration>
```	

* 第三个：hdfs-site.xml
```
<configuration>
	<!-- 设置hdfs副本数量 -->
	<property>
		<name>dfs.replication</name>
		<value>1</value>
	</property>
</configuration>
```		
* 第四个：mapred-site.xml.template 
需要重命名： mv mapred-site.xml.template mapred-site.xml
```
<configuration>
	<!-- 通知框架MR使用YARN -->
	<property>
		<name>mapreduce.framework.name</name>
		<value>yarn</value>
	</property>
</configuration>
```

* 第五个：yarn-site.xml
```
<configuration>
	<!-- reducer取数据的方式是mapreduce_shuffle -->
	<property>
		<name>yarn.nodemanager.aux-services</name>
		<value>mapreduce_shuffle</value>
	</property>
</configuration>
```	
#### 3.4将hadoop添加到环境变量
		vim /etc/profile
		
		export JAVA_HOME=/usr/java/jdk1.7.0_55
		export HADOOP_HOME=/cloud/hadoop-2.2.0
		export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin
	
		source /etc/profile
#### 3.5格式化HDFS（namenode）第一次使用时要格式化
		hadoop namenode -format
		
#### 3.6启动hadoop
		先启动HDFS
		sbin/start-dfs.sh
		
		再启动YARN
		sbin/start-yarn.shcd 
		
		也可以
		sbin/start-all.sh
		
#### 3.7验证是否启动成功
		使用jps命令验证
		27408 NameNode
		28218 Jps
		27643 SecondaryNameNode
		28066 NodeManager
		27803 ResourceManager
		27512 DataNode
	
		http://192.168.1.44:50070  (HDFS管理界面)
		在这个文件中添加linux主机名和IP的映射关系
		C:\Windows\System32\drivers\etc\hosts
		192.168.1.119	itcast
		
		http://192.168.1.44:8088 （MR管理界面）




