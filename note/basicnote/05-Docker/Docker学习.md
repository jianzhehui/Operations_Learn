[TOC]



# docker学习

## 1 docker概念

Docker是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的容器中，然后发布到任何流行的Linux机器上，也可以实现虚拟化，容器是完全使用沙箱机制，相互之间不会有任何接口。

namespace实现隔离，Cgroup实现配额

一个容器就是一个进程



## 2 docker安装

使用opsx，华为云，清华大学镜像站等镜像安装

```shell
#!/bin/bash
# step 1: 安装必要的一些系统工具
sudo yum install -y yum-utils

# Step 2: 添加软件源信息
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce\
/linux/centos/docker-ce.repo

# Step 3: 安装Docker
sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Step 4: 开启Docker服务
sudo service docker start

bash 1.sh
```

配置镜像加速器

```shell
vi /etc/docker/daemon.json
```

```shell
{
"registry-mirrors": ["https://docker.m.daocloud.io"]
}
```

## 3 docker架构

Docker 架构主要由三部分组成：

Docker Client：使用命令行/工具，负责发送请求。

Docker Daemon：后台服务，负责构建镜像、创建/运行容器、管理网络和存储。

Docker Registry：镜像仓库（如 Docker Hub/私有仓库），用于存取镜像。

它们的关系：Client → 通过 API 和 Daemon 通信；Daemon → 从 Registry 拉取/推送镜像并运行容器。

## 4 docker命令

#### 4.1 服务相关命令

```shell
systemctl start docker
#启动docker服务

systemctl stop docker
#停止docker服务

systemctl restart docker
#重启docker服务

systemctl status docker
#查看docker服务状态

systemctl enable docker
#设置开机启动docker服务

systemctl daemon-reload
#重新加载配置
```

#### 4.2 镜像相关命令

```shell
docker images
#查看本地所有镜像

docker images -q
#查看所有镜像id

docker search 镜像名称
#从网络中查找需要镜像

docker pull 镜像名称:版本号
#从docker仓库下载镜像到本地，不指定版本号则为最新版本

docker rmi 镜像id
#删除指定镜像
```

#### 4.3 容器相关命令

```shell
docker ps 
#查看正在运行的容器

docker ps -a
#查看所有容器
```

```shell
docker run 参数
# -i：保存容器运行
# -t：为容器重新分配一个伪输入终端
# -d：以守护（后台）模式运行容器
# -it：创建的容器一般称为交互式容器，加入这两个参数后，容器创建后自动进入容器中，退出后，容器自动关闭
# -id：创建的容器一般称为守护式容器，需要使用docker exec进入容器，退出后，容器不会关闭
# --name：为创建的容器命名
# --restart：设置重启策略
```

```shell
docker exec
#进入容器

docker stop 容器名称
#停止容器

docker start 容器名称
#启动容器

docker rm 容器名称
#删除容器，如果容器正在运行需要停止后再删除

docker inspect 容器名称
#查看容器信息
```

## 5 容器数据卷

#### 5.1 数据卷概念及作用

数据卷概念

数据卷是宿主机中的一个目录或文件

当容器目录与数据卷目录绑定后，当修改其中的一个时，另一个会同步修改

一个数据卷可以同时被多个容器挂载

一个容器可以同时挂载多个数据卷

数据卷作用

容器数据持久化

外部机器与容器间接通信

容器之间数据交换

#### 5.2 配置数据卷

```shell
docker run -v /root/nginx:/usr/share/nginx/html		
#目录挂载

docker run -v ngconf:/etc/nginx
#卷映射
```

目录必须是绝对路径

如果目录不存在，会自动创建该名字的目录

可以挂载多个数据卷

#### 5.3 数据卷容器

如何实现多容器进行数据交换

多个容器挂载同一个数据卷

数据卷容器

配置数据卷容器

```shell
docker run -it --name 333 -v /volume nginx
#创建333数据卷容器，使用-v参数设置数据卷,不填写/volume左边的信息，系统会自动分配目录

docker run -it --name 111 --volume-from 333 nginx
docker run -it --name 111 --volume-from 333 nginx
#创建启动111，222容器，使用--volunme-from设置数据卷
```

```shell
docker inspect 333
#查看333
```

找到Mounts，查看源目录与目标目录

已经启动的容器无法再挂载文件

## 6 Dockerfile

#### 6.1 Docker镜像原理

linux文件系统由bootfs与rootfs两部分组成

bootfs包含bootloader（引导加载程序）和kernel（内核）

rootfs是root文件系统，包含典型linux系统中的标准目录和文件

不同linux发行版中，bootfs基本一样，rootfs不同

docker镜像是由特殊文件系统叠加而成

最底层是bootfs，并使用宿主机的bootfs

第二层是rootfs，称为base image

还可以再往上叠加其他镜像文件

当从一个镜像启动容器时，docker会在最顶层加载一个读写文件系统作为容器

docker镜像本质上是一个分层系统文件

#### 6.2 docker镜像如何制作

容器转为镜像

```shell
docekr commit 容器id 镜像名称:版本号
```

镜像无法直接传输，可以将镜像变为压缩文件后进行传输

```shell
docker save -o 压缩文件名称 镜像名称:版本号
docker load -i 压缩文件名称
```

#### 6.3 dockerfile概述

dockerfile是一个文本文件，其中包含了一条条指令，每一条指令构建一层，最终构建处一个新的镜像

案例 自定义centos7镜像，默认登录路径为/usr，可以使用vim

```shell
mkdir dockerfile
vi dockerfile/centos_file
```

```shell
FROM centos:7								#定义父镜像
MAINTAINER jzh								#定义作者信息
RUN yum install -y vim						#执行安装vim命令
WORKDIR /usr								#定义默认工作目录
CMD /bin/bash								#定义容器启动执行的命令
```

保存并退出

```shell
docker build -f docekrfile/centos_file \
			 -t jzh_centos:1		   \
			 .
# -f:指定dockerfile位置
# -t:设置新镜像的名称与版本
# .:在当前文件夹下进行编译，指定构建上下文
```

## 7 docker compose

#### 7.1 docker compose概述

docker compose是一个编排多容器分布式部署的工具，提供命令集管理容器化应用的完整开发

周期，包括服务构建，启动和停止。

使用步骤：

利用dockerfile定义运行环境镜像

使用docker-compose.yml定义组成应用的各服务

运行docker-compose up启动应用

#### 7.2 docker compose使用

以”在本机访问 [http://127.0.0.1:8080](http://127.0.0.1:8080/)如下:Do difficult but meaningful things and keep doing them !!!“为例

先在当前目录中创建两个文件：

compose.yml

```shell
services:
  nginx:
    image: nginx
    ports:
      - "8080:80"
    volumes:
      - ./index.html:/usr/share/nginx/html/index.html
```

index.html

```shell
Do difficult but meaningful things and keep doing them !!!
```

然后在该目录执行:

```shell
docker compose up -d
```

访问 http://127.0.0.1:8080就会看到这句话。停止并删除：

```shell
docker compose down
```

访问https://docs.docker.com/reference/compose-file/可以看到compose语法规则

#### 7.3 常用命令

```shell
docker compose up -d -
#启动（没有就创建）
docker compose down
#停止并删除
docker compose ps
#查看
docker compose logs -f
#日志
docker compose restart <服务名>
#重启某个服务
docker compose pull
#拉取新镜像
docker compose up -d --build
#重新构建再启动
docker compose up -f 文件名
#指定compose文件启动
```

## 8 docker网络

#### 8.1 3种网络模式

Bridge模式

这是 Docker 的默认模式。可以理解为电脑里装了一个虚拟路由器（docker0）。所有容器都插在这个

路由器上，它们有自己独立的 IP（内网 IP），与外界隔离。

特点：容器之间默认是通的，但与外部网络是隔绝的。

适用场景：绝大多数单机跑的应用（Web服务器、数据库等）。

 Host 模式

容器共用宿主机的网络

特点：容器没有自己的 IP，直接使用宿主机的 IP。如果容器监听 80 端口，就直接占用了你电脑的 80 

端口。

优点：网络性能最好。

缺点：容易端口冲突（你不能开两个都用 80 端口的容器），且安全性稍差。

None 模式

封闭模式，容器有自己的空间，但没有网卡，没有 IP，不能联网。

适用场景：极高安全性要求，只需要计算不需要联网的任务

#### 8.2 端口映射

因为默认的 Bridge 模式下，外部无法直接访问容器内部的 IP。需要将端口暴露使得外部能访问

参数：

```shell
-p 宿主机端口:容器端口

docker run -p 8080:80 nginx
#将nginx80端口映射为主机8080端口
```

随机端口

```shell
docker run -P nginx
#Docker会随机找一个没被使用的端口映射到容器的80端口
```

指定 IP 监听

```shell
docker run -p 127.0.0.1:3306:3306 nginx
#只有自己能连这个nginx，局域网其他人连不上。
```

UDP 协议

绝大多数服务是 TCP 的，但像 DNS 或某些游戏服务器需要 UDP。

```shell
docker run -p 53:53/udp dns-server
```

如何查看端口映射是否成功？

最常用的命令是：

```shell
docker ps
```

输出结果中的 PORTS 一栏会显示： 0.0.0.0:8080->80/tcp

这意味着：任何发往主机8080的流量，都会流向容器的80。

#### 8.3 容器间通信

自定义网络与DNS

Docker 允许创建一个自己的“虚拟局域网”，加入这个网络的容器可以通过容器名（Container Name）直接互相访问（自带 DNS 解析）。

创建网络

```shell
docker network create net
```

启动nginx1（加入网络）：

```shell
docker run -d --name 111 --network net nginx
```

启动nginx2（加入同一网络）：

```shell
docker run -d --name 222 --network net nginx
```

进入容器并访问

```shell
docker exec -it 111 bash
# 进入 my-web 容器内部测试

ping 222
```

#### 8.4 常用命令

列出当前所有的网络

```shell
docker network ls
```

创建一个新网络

```shell
docker network create <名字>
```

查看容器详情

```shell
docker network inspect <名字>
```

把一个正在运行的容器加入到某个网络中

```shell
docker network connect <网络名> <容器名>
```

删除所有未被使用的网络

```shell
docker network prune
```

