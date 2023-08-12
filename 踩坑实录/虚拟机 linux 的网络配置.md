# 虚拟机 linux 的网络配置

标签（空格分隔）： linux

---

[toc]

***

## linux系统网络配置步骤
1、setup配置IP地址

2、修改网卡配置文件，修改 ONBOOT=yes

3、重启网络服务：service network restart

4、如果需要，修改 uuid，保证其唯一性
（1）删除网卡配置文件中 MAC地址信息
（2）删除网卡与MAC地址绑定文件：rm -rf /etc/udev/ruled.d/70-persistent-net.rules
（3）重新启动系统

## 虚拟机的网络连接方式
* 桥接：与真实机相同的网关，需要单独的IP地址

* NAT：使用 VMware8 网卡，可以使用公网

* Host-only：使用 VMware1 网卡，不能使用公网





