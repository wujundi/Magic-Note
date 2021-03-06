﻿# Java 校招火力点侦察报告

标签（空格分隔）： 校招

---

## 套路
印象最深的项目是哪个？项目中有没有遇到啥难题？怎么解决的？
自己项目+自己项目不足
面试官不会主动问你具体项目，还是你自己讲为主，项目中的亮点也要主动和面试官提起
总的来说你需要了解简单的原理，具体每个问题面试官都问了我一下深入的其他问题
大学生活中有没有遇到什么挫折

## 算法
知道的排序算法，重点问了快排，快排的优化
快速排序和堆排序的优缺点，为什么？
熟练掌握八大排序
都有哪几种排序，都是怎么实现的
各种排序算法的时间复杂度和空间复杂度
第一题是要查找数组中的最小元素
最小堆
数据去重，海量数据去重，找出海量数据中前10个最大的数（数据有重复）
二分查找代码
查找算法（顺序查找、二分查找、二叉排序树、平衡二叉树、哈希法等等）
手写进制转换算法，求出一个数的二进制数1的个数
反转二叉树
平衡二叉树
红黑树的简单知识
哈夫曼树
了解哈夫曼树、b+/b-树、红黑树
说下平衡树
环状有序数组，中间剪断，寻找其中一个数值 
一串长字符串，两个一样的数字，寻找其中出现两次的数字。（两种实现方法） 
链表中如何判断有环路
掌握BFS/DFS、KMP、DP等等
能够实现栈与队列
深度优先遍历，广度优先遍历算法，在什么地方可以应用。
说下几个最短路算法和具体实现
动态规划

***

## Java

***

### 语法基础
java的抽象类和接口区别，抽象类可以有实例吗
int 4个字节，double 8个字节
java三大特性
java 4种方法修饰符
String类可以继承吗？
方法重载，何时用重载，为什么要使用重载？而不是把一个方法名字换成不同的
equals和hashcode的区别
java基本类型与引用类型的区别
Java中异常和错误的区别，列举五个运行时异常
Java动态代理
迭代器怎么用，手写代码
final(变量、方法、类) finally finalize
try catch finally 可不可以没有catch（try return,finally return）
static 关键字修饰的方法占不占内存，方法区的内存
手写java的8种基本数据类型
重写重载，重载返回类型不一样，其他都一样可以不？（不可以）
String StringBuffer StringBuilder的区别
对象判断采用hashcode判断对象是不是同一对象
serviable的序列化
什么情景下会用到反射（答注解、Spring配置文件、动态代理）
浅克隆与深克隆有什么区别
如何实现深克隆
equals跟hashCode有什么区别与联系
线程池的类型，固定大小的线程池内部是如何实现的，等待队列是用了哪一个队列实现
字符串内存分配问题，String s1 = "123" + "456"  和 String s1 = "123"； String s2 =  s1 + "456";  分别会创建多少个对象
Java 8函数式编程
回调函数，函数式编程，面向对象之间区别
Java 8中stream迭代的优势和区别
反射能够使用私有的方法属性吗和底层原理？
你谈谈Java8的一些新特性。
Servlet是线程安全的吗？
JSP和Servlet区别
手写jdbc连接过程

***

### 集合类
手写ArrayList
map底层实现，最好看源码
还有各种集合类的区别
hashmap的iterator读取时是否会读到另一个线程put的数据
concurrent包
hashmap和concurrenthashmap区别及两者的优缺点,
hashmao 扩容因子过大过小的缺点，扩容过程
翻转字符串
HashMap原理，自定义类型可以作为Key吗
Collection集合类中只能在Iterator中删除元素的原因
ArrayList扩充问题。add()方法的底层实现。 
各个容器的底层实现，比如arraylist，hashmap,set,底层的数据结构，画出结构图
hashmap与hashtable的区别
ArrayList与LinkedList的区别
ConcurrentHashMap和LinkedHashMap差异和适用情形
ConcurrentHashmap jdk1.8访问的时候是怎么加锁的，插入的时候是怎么加锁的（访问不加锁插入的时候对头结点加锁）
ArrayDeque的使用场景
ConcurrentHashmap源码
hashmap的存储过程，链表使用的循环链表还是双向链表
数组中Arrays.sort的排序方法是什么？
hashmap的实现原理 采用什么方法能保证每个bucket中的数据更均匀
hashmap底层实现原理，解决冲突的方式，还有没有其他方式（全域哈希）
hadhmap concurrenthashmap区别 synchronizedhashmap如何实现，之间的区别（锁的粒度不同）
hashmap存节点 怎么存
线程安全 java里面的实现方式
常见的线程安全的集合类
ConcurrentHashMap分段锁是如何实现的
什么时候会用到HashMap
HashMap的存放自定义类时，需要实现自定义类的什么方法
HashMap的负载因子
CopyOnWriteArrayList多线程安全吗
把你知道的java的concurrent包的技术全部说出来（volatile、锁重入，LinkedTransferQueue字节追加提高并发度技术，ConcurrentHaspMap结合volatile的happen-before读取优化）

***

### 多线程
threadlocal，
各种锁，
synchronized和lock
多线程中的wait和sleep区别，notify的作用
写一个生产者消费者队列的方法
多线程如何避免死锁
如何解决高并发问题？是否进行过相应程序的编写？
notify()工作原理
volitale的用途
说一下静态方法和普通方法同时加上synchronized有什么区别
java线程与进程差别
java 网络编程解决并发量
Thread中，ThreadLocal,Lock等
线程池，线程池有多少种，每种的特点
多线程实现方式，三种
说一说了解的锁(主要介绍可重入锁
ReentrantLock)
ArrayBlockingQueue源码
ReentrantLock底层(AQS)
多线程下有什么同步措施
Java 中的锁是怎么实现的、有什么锁
AtomicInteger实现原理(CAS)
你用的是多进程还是多线程；（多进程和多线程的区别）
Java无锁原理
悲观锁与乐观锁
说下死锁以及哪些情况可能会死锁
线程notify，wait，join的区别作用
wait跟sleep有什么区别
Thread跟Runnable有什么关系，分别用在什么场景
ThreadLocal为什么保证线程私有
synchronized可以替代读写锁吗？
知道线程的中断吗
当获取第一个获取锁之后，条件不满足需要释放锁应当怎么做？
既然线程调用中断方法不会停止程序，那么有什么用？
创建线程三种方式
CountDownLatch
ReentrantLock源码，AQS，synchronized实现，乐观锁和悲观锁，volatile，ThreadLocal原理，线程池工作流程

***

### I/O
java NIO
用过哪些java的I/O 
Java IO模型(BIO,NIO等)、Tomcat用的哪一种模型
java io用到什么设计模式
NIO和BIO的比较
NIO的DirectByteBuffer和HeapByteBuffer，同步和异步，阻塞和非阻塞

***

### 网络编程
socket通信是的应用过程
socket是靠什么协议支持的

***

### JVM
类加载器，双亲委派模型，热部署
jvm内存模型，内存结构、堆的分代算法、堆的分区、gc算法、gc过程。
java的内存模型，变量和实例存在哪。java栈的作用，java的堆存什么，方法区存什么。
java的分代回收。
热加载的原理
类加载的原理
java的堆和栈
内存泄漏发生在哪
jvm优化、内存管理、gc分析
JVM内存模型及调优
JVM解释编译执行过程
什么时候会回收对象，怎么判断这个对象可以被回收。对象的的几种生存状态。  
从他的编译之类，加载（忘记回答父类，子类，以及静态代码块，普通代码块的加载过程）  
自己写两个ClassLoader加载两个Spring bean能不能互相访问？（我也没懂这个问题是问什么）
CMS
Java 64 位的指针压缩
GC中可达性分析法，和引用计数法有什么不同？引用计数法有什么问题？
如何加快gc的速度 快速判断对象生死
说一下自旋锁
有没有遇见过内存溢出的情况
处理器指令优化有些什么考虑
同步等于可见性吗
那你怎么使用某种收集器呢？在哪个时候进行参数设置？
那你知道内存GC的时候，要使用Object的什么方法吗？这个方法在什么时候使用？
内存区域介绍及发生OOM的情况，GC过程，描述调优过程

***

## java EE 工具

### Spring + springMVC
常用框架的架构
spring类加载方式、实例保存在哪、aop ioc、反射机制
spring 
IOC
AOP以及动态代理
springmvc 
spring  的核心如：IOC AOP等，注入一个UserDaoImpl时，UserDaoImpl有几个实例，一个还是多个
spring 控制反转和依赖注入关系
spring的理解，注入的方式
如果让你自己实现一个orm框架会如何实现
Spring的注解是不是用反射实现的
Spring的注解要怎么开启
如果让你实现依赖注入，怎么实现？
了解SpringMVC与Struct2区别
了解SpringMVC请求流程
springmvc的流程 一个请求来了之后如何处理（handler链）
注解、反射、IOC理解
Springmvc 注解 流程
Spring 中哪些好的技术（IoC以及其他的），Spring有哪些缺点。
介绍一下spring cloud的组件
Spring Boot介绍
SpringMVC 接收到一个请求，是怎么处理的
你项目中拦截器是怎么使用的，你Controller类有哪些注解？
Spring的源码有看过吗？Spring用到哪些设计模式
问spring boot里用过哪些注解
手写springMVC的流程
Spring boot的starter简化了配置，如果想自定义一个starter该怎么做？
框架里的注解使用起来很方便，注解的原理是什么？具体是怎么产生作用的？怎么把几个注解合并成一个？ 
Spring的加载流程，Spring的源码中Bean的构造的流程
Spring中AOP有JDK和CGLIB代理，谈了谈两者区别。（不过感觉好像面试官想听到的是反射）

***

### Mybatis
JDBC连接的过程
mybatis
mybatis的#和$号区别
mybatis中 %与$的区别
ibatis跟hibernate的区别
ibatis是怎么实现映射的，它的映射原理是什么
mybatis与spring data JPA区别
框架封装jdbc受检异常的考虑和原因
Hibernate、Mybatis与JDBC区别

***

### Tomcat
部署项目时tomcat 的参数
tomcat的配置，堆得初始大小是多少
Tomcat类加载器模型
tocat 源码

***

### 数据库链接池
数据库连接池用的是什么
为什么要用数据库连接池，有什么好处。
mysql数据库连接池的驱动参数
数据库连接池如何防止失效
数据库连接池（druid）、线程池

***

### Redis
缓存数据库 比如redis
 redis缓存，redis的集群部署，热备份，主从备份，主从数据库，hash映射找到知道指定节点。
如何实现分布式缓存
浏览器的缓存机制
缓存问题，缓存与内存的区别
redis技术与zookeeper了解多少
redis五种数据类型，当散列类型的value值非常大的时候怎么进行压缩，用redis怎么实现摇一摇与附近的人功能，redis主从复制过程
项目中Redis是单机还是集群
Redis数据结构
Redis持久化机制
Redis的使用，Redis查找跟插入都很快，可以再考虑下Redis与MySQL的适用场景
什么时候适合用Redis作为主要的数据库
redis两种持久化方式
Redis数据结构
Redis散列是如何实现的
Redis持久化机制
项目中redis使用
有没有用过redis其他数据结构
排行榜可以使用redis哪种数据结构
redis的操作是不是原子操作
redis的五种数据结构
redis是怎么存储数据的
redis的配置文件（AOF&&Snapshot&&主从复制）
Redis的一致性哈希算法

***

### solr
Lucene底层实现原理，它的索引结构

***

### 时间调度
Quartz2的几种Trigger
对于这个quartz框架你如何在分布式服务中进行改造
分布式服务在多节点中如何保证事务的统一性，而不是时间上的一致性

***

## 设计模式
设计模式了解哪些，写一个观察者模式。实现两个接口，一个是主题一个是观察者，并写出对应方法
设计模式 工厂模式 单例模式 举例子
单例模式
写出线程安全的单例模式
Servlet的Filter用的什么设计模式
单例模式（双检锁模式）、简单工厂、观察者模式、适配器模式、职责链模式等等
手写单例模式
设计模式，装饰者模式画图
然后是画两个设计模式的UML图
框架源码中用到的设计模式

***

## Mysql
mysql引擎
索引
数据库的索引原理，b+树原理，trie树引申，二叉查找树的原理
实现层级树形结构 引擎 索引 查询优化
B+树怎么索引
对MySQL的了解，和oracle的区别
数据库的范式
mysql常用引擎有哪些，说说你对InnoDB了解
sql注入3种类型
唯一索引和普通索引差别
数据库的隔离级别
Mysql ACID具体
隔离级别如何实现
数据库索引，MyISAM、InnoDB的区别(一个B+树叶子节点存的地址，一个是直接存的数据)
MySQL索引底层实现
MySQL聚集索引与非聚集索引区别
你对数据库进行查询，发现查询很慢，对代码排查，代码没问题，你怎么对数据库进行排查；（聊了索引）
给你一个数据库，数据库里面数据很大（TB级），你怎么解决查询慢（性能优化）的问题；（分区技术）
分区的类型
数据库基本知识（存取控制、触发器、存储过程（了解作用）、游标（了解作用）
数据库优化能力（索引（了解其原理）、分区、分表以及SQL语句优化等）
并发控制（并发数据不一致性、事务隔离级别、乐观锁与悲观锁等）
mysql 数据库底层如何实现
B+树和B树的区别
插入节点怎么分裂
索引有哪些，用性别做联合索引有没有效果
设计数据库表，数据库设计一般设计成第几范式
mysql锁机制
varchar与text区别
如何设计多对多的表关系
mysql数据库char，varchar的区别
B+树索引与哈希索引有什么区别
如何查看索引的使用情况
innodb聚簇索引(实际就是聚集索引，听成计数索引，就说没听过)
mysql架构（连接池、查询器、优化器、缓冲器等）
数据源和连接池区别呢？
说一下数据库事务的四个特性，为什么mysql事务能保证失败回滚
mysql数据库的锁有多少种，怎么编写加锁的sql语句
mysql什么情况下会触发表锁
页锁、乐观锁、悲观锁
你能说一下数据库的主外关联吗？以及它有什么特性呢？

***
## SQL
sql语句，一个成绩表求出有不及格科目的同学
sql，group by的理解
有一条SQL语句很慢，如何优化
手写sql语句
数据库 SQL 问题，写了几条联结查询的语句，然后分析索引的使用原理之类的
问了一个这样的表(三个字段:姓名，id，分数)要求查出平均分大于80的id然后分数降序排序
Sql写的多吗？谈谈内连接和外连接吧。

***

## Linux
了解linux么，说一下linux的内核锁
linux的显示文件夹大小 ls -al
linux的查看端口状态 natstat加参数
linux的查看进程的启动时间 linux ps
linux查看进程并杀死
vim编辑器
常用的linux命令。查找某端口的进程与用户 
linux如何查看进程号
linux查看进程消耗的资源
top命令每个选项意义
问：怎么查看进程占用资源？   
问：怎么查看网络状态？   
问：查看磁盘读写吞吐量？   
问：查看80端口连接数。   

***

## 网络
http协议
tcp ip 七层模型 rest接口规范 get和post区别，长度，安全。
tcp ip的arp协议，两个同一网络的主机如何获得对方的mac地址。
tcp ip的四次挥手 子网掩码的作用
http的数据包格式
tcp包含ip么
tcp的数据包格式
三次握手四次分手以及wait_time三种差别
Http post和get差别，http状态码
Tcp与Udp
协议的划分，Tcp与Udp 网络层的ARP协议与ICMP协议
HTTP请求头包括哪些
路由器和交换机有什么区别
socket编程，怎么实现一个多人聊天室；（怎么设计、怎么实现）
http和https区别；(https = http + ssl)
HTTP请求方法（GET、HEAD、POST、PUT、DELETE、CONNECT、OPTIONS、TRACE）
GET与POST请求区别（根据笔试题的回答提问），POST请求运用，GET幂等的理解，GET请求URL显示，GET请求URL中为什么有“？”
当一个tcp监听了80端口后，Udp还能否监听80端口
PING位于哪一层
了解ICMP协议、ARP协议等
了解TCP协议（超时重传、流量控制（滑动窗口）、拥塞控制等等）
了解常见网络攻击（SQL注入、DDOS攻击、重放攻击、DNS欺骗等等）
网络编程的accept和connect
tcpip的四次挥手 几种状态，讲讲timewait和closewait
拥瑟避免实现算法
http2.0跟1.1有啥区别
tcp是看过书还是实际用过
TCP除了ESTABLISHED还有哪些状态
TCP 的三次握手、四次挥手，time_wait 状态
TCP为什么是四次挥手，其中TIME_WAIT和CLOSE_WAIT这两个阶段
仔细谈谈DNS解析
WebSocket长连接问题

***

## 操作系统
线程与进程区别
操作系统同步方式、通信方式
进程与线程的区别，多线程有可能会造成的问题(占用内存，资源竞争产生死锁啊之类的)
操作系统cpu调度算法
了解内存管理页面置换算法（LRU，Java中如何实现（LinkedHashMap））
了解进程间通信方式
了解死锁与饥饿区别
死锁的产生必要条件
了解如何预防死锁（银行家算法、破坏条件等等）
实现阻塞队列
生产者消费者模型实现
进程间通信的几种方式
32位系统的最大寻址空间？

***

## 架构实践
负载均衡、高并发、高可用的架构
项目中的数据库备份，主从数据库、集群
说说RESTful架构
docker文件系统
对docker的理解
了解输入网址之后到服务器整个过程
了解常见加密方式（MD5、SHA-1等）
了解分布式缓存、Zookeeper、阿里dubbo、Nginx等
负载均衡怎么实现
负载均衡里添加新的结点如何发现，局域网下 非局域网下
提升访问网页效率的方法(缓存:客户端缓存，cdn缓存，服务器缓存，多线程，负载均衡之类)
对nginx负载均衡有了解过吗
一个有500个用户的广播系统，你怎么做性能优化
你的并发项目有做过压测吗
写一个http请求的完整流程
项目中权限管理如何实现的
zookeeper的常用功能，自己用它来做什么，有没有复杂均衡与主从协调的经验


***
## 大数据
mapreduce流程
如何保证reduce接受的数据没有丢失，数据如何去重

***

## 云计算
了解云计算么，了解云容器docker么，容器和虚拟机的区别。