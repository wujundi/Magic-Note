# Docker 常用命令

* 拉取镜像 docker pull wujundi/mysql-server:born
* 查看本地镜像 docker images
### 按照镜像启动一个容器 
* docker run --name=mysql  -d -p 3306:3306 wujundi/mysql-server:born

i是交互式操作
t是一个终端
d指的是在后台运行
-P指在本地生成一个随机端口
-p指定端口映射 

### 进入一个容器
* docker exec -it mysql bash

### 退出一个容器
* exit


### 查看正在运行的容器 
* docker ps
* docker ps -a 查看所有容器

### 停止一个容器
* docker stop mysql

### 删除某个容器
* docker rm container_id

### 查看所有镜像
* docker images
### 删除某个镜像
* docker rmi

### 重命名镜像
* docker tag  image_id  docker_hub_REPOSITORY:TAG

### 上传镜像到 docker hub
* docker push  image_name