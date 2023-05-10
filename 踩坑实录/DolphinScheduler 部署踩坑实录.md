## 准备基础环境

* docker run -itd --name='dolphin' --privileged -p 12345:12345 registry.cn-hangzhou.aliyuncs.com/wujundi/ubuntu-dolphinscheduler-315 /usr/sbin/init
* 一定要先 初始化数据库，否则 master 容器会因为读不到表而异常退出，无法正常启动

```shell
# 如果需要初始化或者升级数据库结构，需要指定profile为schema
$ docker-compose --profile schema up -d

# 启动dolphinscheduler所有服务，指定profile为all
$ docker-compose --profile all up -d
```

* 后续过程中仍然遇到过访问不了的情况，一般就重启 api 容器就可以了
