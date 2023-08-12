## 准备基础环境

* docker run -itd --name='dolphin' --privileged -p 12345:12345 registry.cn-hangzhou.aliyuncs.com/wujundi/ubuntu-dolphinscheduler-3.1.5 /usr/sbin/init
* 一定要先 初始化数据库，否则 master 容器会因为读不到表而异常退出，无法正常启动
* docker compose 配置文件在项目的 deploy/docker 目录下面

```shell
# 如果需要初始化或者升级数据库结构，需要指定profile为schema
$ docker-compose --profile schema up -d

# 启动dolphinscheduler所有服务，指定profile为all
$ docker-compose --profile all up -d
```

* 后续过程中仍然遇到过访问不了的情况，一般就重启 api 容器就可以了
* 注意一下，在 wsl 环境下面，可以通过 1277.0.0.1:12345/dolphinscheduler/ui 访问，但是不能通过 172.18.254.162:12345/dolphinscheduler/ui，这一点没太搞懂，后面直接布在 linux 虚拟机器上不涉及这个问题，所以这里就每太深究了

## 环境联调部分
