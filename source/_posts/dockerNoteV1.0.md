---
title: docker 基础
date: 2021/1/9 10:26:08
categories:
- 教程
tags:
- docker
- linux
toc:
- true
pic:
- facePic.png
---

2020.9

服务器环境为，阿里云的 Centos7 实例

## Docker 安装

https://docs.docker.com/engine/install/

### **Centos 环境下的安装**

https://docs.docker.com/engine/install/centos/

## Docker 的常用命令

### 1. 帮助命令

```shell
docker version  			# 显示docker的版本信息
docker info					# 显示docker的系统信息，包括镜像和容器
docker (command) --help		# 帮助命令
```

帮助文档地址：https://docs.docker.com/engine/reference/commandline/docker/



### 2. 镜像命令

```shell
docker images 		# 查看本地所有的镜像以及信息

# 解释
REPOSITORY 	仓库源
TAG 		标签
IMAGE ID	镜像的ID
CREATED 	镜像的创建时间
SIZE		镜像的大小

# 可选项
	-a, --all		# 列出所有镜像
	-q, --quiet		# 只显示镜像的ID
```

**docker search 搜索镜像**

```shell
docker search mysql		# 搜索 mysql

# 可选项
	--filter=STARS=3000		# 搜索出的镜像Stars大于3000
```

**docker pull 下载镜像**

```shell
docker pull 镜像名[:tag]	# 这里的tag指版本
# 如果不写tag，默认是latest
# docker image的核心 联合文件系统、分层下载

# 等价
docker pull mysql
docker pull docker.io/library/mysql:latest	# 真实地址

docker pull mysql:5.7	# 即下载 mysql 5.7 的版本（需要在镜像库中支持）
```

**联合文件系统**

在分层下载中，某些前置部分是共用的，docker在进行下载时不会下载已有的层，这极大的提高了存储效率。

{% asset_img image-20200909165017120.png %}

{% asset_img image-20200909165127351.png %}

如图所示，不同镜像共用的“层”被标识为Already exists（已存在）

**在这里这个结构有点类似git，这个以后再做理解吧。**

**docker rmi 删除镜像**

```shell
docker rmi -f 镜像id	# 删除镜像
docker rmi -f id1 id2 id3	# 删除多个镜像
docker rmi -f $(docker images -aq)	# 删除所有镜像
# 解释
	-f	递归
	$() 将参数传给 -f
# 在这个应用场景下一般只用$(docker images -q)
```

{% asset_img image-20200909165605624.png %}

### 3. 容器命令

有了镜像后，就可以创建容器了，需要一个centos的镜像来用于学习

相当于利用docker运行了一个centos系统，这比虚拟机更快一些。

**下载 Centos 镜像**

```shell
docker pull centos
```

**新建容器并启动**

```shell
docker run [可选参数] image
# 参数说明
--name="Name"	容器名为Name
-d				后台方式运行
-it				使用交互方式运行，进入容器查看内容
-p
    -p ip:主机端口:容器端口
    -p 主机端口:容器端口
-P				随机指定端口
```



**测试：启动并进入容器**

{% asset_img image-20200909170930233.png %}

我们先主机名已经变成了镜像id

此时，我们已处于容器内的环境

在该环境内输入 exit ，则关闭该容器，回归主机

**列出所有运行中容器**

```shell
docker ps

# 可选参数
-a		历史运行的容器 + 正在运行的容器
-n		显示 n 条最近创建的容器记录
-q		只显示容器的编号，即 CONTAINER ID
```

{% asset_img image-20200909171412343.png %}

**退出容器**

```shell
exit	# 容器停止并退出
Ctrl + P + Q	# 容器不停止退出，类似Screen Ctal + A + D
```

**删除容器**

```shell
docker rm 容器ID	# 删除容器
docker rm -f $(docker ps -aq)	# 删除所有容器
docker ps -a -q|xargs docker rm		#删除所有容器
```

**启动和停止容器的操作**

```shell
docker start 容器id		# 启动容器
docker restart 容器id		# 重启容器
docker stop 容器id		# 停止当前正在运行的容器
docker kill 容器id		# 杀死当前容器
```

当使用 docker run -d 跑centos时，发现 centos 自动停止了

docker 发现容器没有应用时，他会**自动关闭**（垃圾回收）

如，不带 /bin/bash 启动centos，或nginx

**查看日志**

{% asset_img image-20200909173300705.png %}

```shell
docker -t -f --tail 10 容器id
# 参数说明
-t	输出结果带时间戳
-f	follow log output，查看实时日志
--tail n	显示最近得到n条日志	
```

**查看容器内部进程信息**

```
docker top 容器id
```

{% asset_img image-20200909174323325.png %}

**查看容器元数据**

```
docker inspect 容器id
```

{% asset_img image-20200909183645448.png %}

**进入挂在后台的容器**

```shell
# exec实在容器内跑一个命令
docker exec -it 容器id /bin/bash

# 方法二
docker attach 容器id
```

{% asset_img image-20200910170509020.png %}

**从容器内拷贝文件到主机上**

```shell
docker cp 容器id:容器内路径 主机路径

# 测试
docker cp b78453025116:/home/test.java /home
```

{% asset_img image-20200910171035939.png %}

{% asset_img image-20200910171112723.png %}



### 4. 小结

**常用命令**

{% asset_img image-20200910172103068.png %}

{% asset_img image-20200910172307994.png %}



## Docker 安装 Nginx

**安装一个 Nginx 的步骤**

```
1. 搜索镜像	search
2. 下载镜像 pull
3. 新建一个容器并运行nginx	run
```

{% asset_img image-20200910172940171.png %}

{% asset_img image-20200910173655664.png %}


**端口转发**

{% asset_img image-20200910173145687.png %}

## Docker 部署 Tomcat

{% asset_img image-20200910174723154.png %}

由于 webapps 目录为空，因此该环境下 tomcat 可访问，但报错，将webapps.dist 的内容复制即可显示 tomcat 欢迎页

```
cp ./webapps.dist/* ./webapps
```



## 部署 ES + kibana

```shell
# es 暴露的端口较多、耗内存、数据一般需要放置在安全目录，需要挂载

docker run -d --name elasticsearch --net somenetwork -p 9300:9300 -e "discovery.type=single-node" elasticsearch:tag

# --net 为docker网络配置
```

如果不出意外，这玩意**非常的卡**。1核2G的服务器大概会当场猝死。

**使用 docker stats 查看 docker 状态**

```shell
docker stats
# 测试es是否运行成功，而不是假死
```

所以我们需要对 容器 进行配置，增加内存的限制

{% asset_img image-20200910181231832.png %}

在这里我们添加了如下参数来对 JVM 进行配置

```shell
-e ES_JAVA_OPTS="-Xms64m -Xmx512m"

# ES自带jdk
```

此时使用 crul 访问9200端口，ES 配置完毕

```
crul localhost:9200
```

## 可视化

- Portainer
- Rancher

**什么是Portainer？**

Docker 图形化界面管理工具，提供一个后台面板进行可视化操作。

**Quick start**

```
docker run -d -p 9000:9000 -p 8000:8000 --name portainer --restart always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer
```

文档：https://www.portainer.io/documentation/deploy-portainer-docker-swarm/

之后访问本机的9000端口，等待 Portainer 加载完毕后，可视化界面就会显示。

加载中画面：

{% asset_img image-20200910185237037.png %}


加载完毕后，进入初始化界面，对Portainer进行初始化配置。

## Docker 镜像原理

### 什么是镜像

镜像是一种轻量级、可执行的独立软件包，用来打包软件运行环境和基于运行环境开发的软件，它包含运行某个软件所需的所有内容，包括环境、环境变量、程序、配置文件和其他依赖。

这个独立软件包，可以被 Docker 运行。

### Docker 镜像加载

**UnionFS 联合文件系统**

{% asset_img image-20200910190118408.png %}

**镜像加载**

{% asset_img image-20200910190227832.png %}

**镜像占用的空间很小**

{% asset_img image-20200910190333976.png %}

**Docker 的镜像是分层的**

{% asset_img image-20200911142636807.png %}

{% asset_img image-20200911142908585.png %}

{% asset_img image-20200911143056702.png %}

我们在使用以下命令时，得到的json字符串中，有 RootFS 项存在，其中包含的 Layers 内容，就是**层**

```
docker image inspect redis:latest
```

{% asset_img image-20200911143435022.png %}

或者是

```
docker pull redis
```

{% asset_img image-20200911143351310.png %}




**镜像的特点**

Docker 的镜像都是**只读**的，当容器被启动时，会加入一个可写的层放置在镜像的顶部（所有层的最上方），用户基于该层进行操作。

这一层就是**容器层**，容器层以下的，叫做**镜像层**。

### 如何提交镜像

**Commit 镜像**

我们通过以下代码向docker提交一个容器作为新的副本。这个副本会被保存在**本地镜像库**内。

```shell
docker commit -m="副本的描述" -a="作者" 容器ID 目标镜像名:[tag]
# 该命令类似git

# 示例：
docker commit -a="Jhin" -m="Jhin`s tomcat with default settings" 7e119b82cff6 tomcatt:0.1
# 会在本地保存一个名为 tomcatt ，版本为0.1的镜像
```

{% asset_img image-20200911144709667.png %}


在这个镜像中，会将之前用户的操作，即**容器层**变为**镜像层**，即保存。

这一功能，类似于VMware中使用的**快照**。



## 数据卷

在使用 docker 的过程中，如果我们把数据都保存在容器中，如果将容器删除，数据就会丢失。	————需求：数据持久化

对于 MySQL 而言，如果因为失误操作导致容器被删除，那么就只能**删库跑路**了。

所以对于容器而言，我们需要一个更好的办法来保存数据。比如，容器需要一个数据共享的技术，我们将 Docker 运行过程中产生的数据，保存到本地。

这就是**卷**的技术，本质即目录的挂载，将容器内的目录挂载到物理机的文件系统上。这使得容器可以**同步**和**持久化**，同时可以使容器之间**数据共享**。这个过程，可以被认为是**双向绑定**。

### 数据卷的挂载

**方式一：使用命令挂载 -v**

```shell
docker run -it -v 主机路径:容器内路径
```

{% asset_img image-20200911151343085.png %}


此时，我们使用 docker inspect 查看容器信息，可得到：

{% asset_img image-20200911151600597.png %}


这里的 Mounts 就是目录挂载的信息。

示例：将容器中Mysql目录映射到本地

{% asset_img image-20200911161025934.png %}




**具名挂载**

{% asset_img image-20200911161953685.png %}


**匿名挂载**

{% asset_img image-20200911162403719.png %}


在这里可以砍到，VOLUME NAME 是随机的。

具名挂载相比匿名挂载，通过 **-v 卷名:容器内路径** 的方式指定了 volume name 的值

我们使用以下代码，可以检查卷。

```
docker volume inspect 卷名
```

{% asset_img image-20200911162701508.png %}


顺带一提，卷被保存在 docker 的工作目录下的 volumes 目录内。

一般情况下，docker 在 **/var/lib/docker** 这个位置下

{% asset_img image-20200911162817769.png %}


因此，我们在大多数情况下，均使用**具名挂载**的方式

{% asset_img image-20200911163030459.png %}


拓展：

{% asset_img image-20200911163214748.png %}


这里的权限，是针对容器的。

### DockerFile

**方式二：DockerFile提供了一个挂载的方式**

{% asset_img image-20200923191359740.png %}


DockerFile是用来构建docker镜像的构建文件，本质是命令脚本。通过这个脚本，我们可以构建一个镜像。

{% asset_img image-20200923184714170.png %}


编写一个DockerFIle

```shell
# 文件中的指令需要大写，如CMD，VOLUME，FROM
FROM centos
# 这里进行了两次匿名挂载吗，volume1/volume2都会在该镜像的根目录下，这是两次匿名挂载。
VOLUME ["volume1", "volume2"]
CMD echo "----end----"
CMD /bin/bash

# 这里的每一个命令，都是镜像的一层。
```

构建镜像

```shell
# 使用DockerFile构建镜像
docker build -f 路径 -t 名称:标签
```

我们可以看到，这里的build过程是分布完成的，这是根据我们编译的每一步脚本来构建镜像的覆盖层。

运行自己构建的镜像

```shell
# 获取镜像列表
docker images
# 运行构建的镜像
docker run it 镜像ID /bin/bash
# 查看镜像详细信息
docker inspect
```

可以在 Mounts --> Source 内看到之前匿名挂载的内容。

### 数据卷容器

在两个 mysql 数据库进行数据同步的应用场景下，我们可以使用数据卷容器来进行。

{% asset_img image-20200923190005526.png %}


多个 mysql 数据同步

{% asset_img image-20200923190735921.png %}


数据卷容器的生命周期一直会持续到没有容器使用为止，即所有使用该卷容器的镜像停止为止。

## DockerFile 详解

### 示例

**来自 GitHub 上的某一举例**

{% asset_img image-20200923191714649.png %}


### 构建过程

**初步了解**

1. 每个保留关键字（指令）都必须大写
2. 由上至下顺序执行
3. #表示注释
4. 每一个指令都会创建并提交一个新的镜像层

DockerFile是面向开发的。当需要制作镜像时，就需要编写对应的DockerFile。

Docker 镜像主键成为企业交付的标准。



**基础指令**

{% asset_img image-20200923193104235.png %}


某度上的图：

{% asset_img image-20200923193156251.png %}


```shell
FROM			# 基础镜像，构建镜像的起点。
MAINTAINER		# 镜像的维护者，姓名 + 邮箱
RUN				# Docker 镜像构建时需要的命令
ADD				# 添加内容，如依赖、库、环境、安装包、压缩包等资源
COPY			# 类似 ADD，将文件拷贝到镜像中
WORKDIR			# 镜像的工作目录
VOLUME			# 设置卷，挂载主机目录
EXPOSE			# 镜像暴露（对外、对宿主机）的端口
ENTRYPOINT		# 容器启动时需要执行的命令，可被追加
CMD				# 容器启动时需要执行的命令，只有最后一个会生效，可被替代
ONBUILD			# 当构建一个被继承的 DockerFile 时，会运行 ONBUILD 的指令
ENV				# 构建的时候设置环境变量
```

### 实战：测试

Docker Hub 中 99% 的镜像都是从 FROM scratch 开始的，然后配置需要的软件和环境。

{% asset_img image-20200923194453539.png %}

{% asset_img image-20200923195413491.png %}


同时，我们也可以使用 docker history 镜像id 来查看一个镜像的构建过程。

### 实战：Tomcat

1、准备镜像文件 tomcat 压缩包， jdk 的压缩包

{% asset_img image-20201003002645908.png %}


放置在当前目录下

2、编写 Dockerfile ，官方命名 Dockerfile ，build 时会自动寻找这个文件，就不需要 -f 来指定 dockerfile 了

{% asset_img image-20201003002908384.png %}


3、构建镜像

```shell
# docker build -t 镜像名
```

4、启动镜像

{% asset_img image-20201003003536478.png %}


5、随手写个webapp，访问测试

{% asset_img image-20201003003314550.png %}


至此，项目部署成功，测试完毕。

## 发布 Docker 镜像

> 发布到 DockerHub

1、地址：https://hub.docker.com/

2、注册一个账户，并尝试登陆

3、在服务器提交自己的镜像



使用 **docker login** 命令登陆

```shell
# docker login --help	可以帮助我们查询相关指令的用法
```

{% asset_img image-20201003004126313.png %}


{% asset_img image-20201003004140868.png %}


同时，我们也可以使用 **docker logout** 来登出

### Push 命令

若验证成功，则会显示 Login Succeeded

这时，我们可以通过 **docker push** 来上传我们的镜像

```shell
# docker push [your username]/[your custom image`s name]:[version tag]
```

### 发布到阿里云镜像服务器

1、登陆阿里云，找到容器镜像服务

2、创建命名空间，并创建容器镜像

3、查看容器镜像的基本信息，在这里我们可以获取一些基本信息以及操作指南

{% asset_img image-20201003004808540.png %}


此处，我们可以获得镜像的公网地址

{% asset_img image-20201003005025393.png %}


使用阿里云的镜像服务器，我们可以更好的管理镜像

## Docker 网络

### 理解 Docker0

清空所有的环境

```shell
docker rmi -f $(docker images -aq)
```

然后使用 **ip addr** 命令，查询一下本地网卡

{% asset_img image-20201003005557550.png %}


我们在这里看到了 lo、eth0、**docker0**

lo：即 localhost 本机回环地址

eth0：是阿里云的内网地址

**docker0**：docker 的桥接网络

两个容器之间的通讯：

{% asset_img image-20201003010011595.png %}


试着启动一个容器：

{% asset_img image-20201003010511669.png %}


> 原理

1、我们每启动一个 docker 容器，docker 都会给容器分配一个ip，我们只要安装了 docker ，就会有一个桥接网卡docker0，使用了 evth-pair 技术

**evth-pair** 是成对出现的虚拟设备接口，一端连着协议，一端连着彼此

OpenStac、Docker容器、OVS的链接都是使用了 evth-pair 技术

让我们多启动几个容器看看。

{% asset_img image-20201003010822779.png %}


在这里，网卡是成对增加的，我们在这里找到了262和264，不难推想，261和263分别是我们启动的容器内部的网卡。在262： **vethc96781@if261** 这个名称中，我们也能得到这一信息。

{% asset_img image-20201003013156293.png %}


在这样的网络环境下，容器和容器之间可以互相通讯，我们可以在容器内 ping 另一个容器的地址。

{% asset_img image-20201003011256223.png %}


因此，Docker 内网传发速度非常的快。

### --link 指定网络连接

我们可以在创建容器的时候，使用 --link 来指定网络连接

这会让我们可以通过 docker 的容器名来直接访问目标容器

{% asset_img image-20201003013211888.png %}


我们分别查看 tomcat02 、tomcat03 的信息（docker inspect tomcat02/tomcat03）

发现，？（执行了--link操作的tomcat03，拥有 links 相关属性，类似于 windows 系统下对 hosts 进行配置）

查看 tomcat03 的hosts：

{% asset_img image-20201003014035897.png %}


但是 --link 有很强的局限性，不适用于当前微服务，分布式的架设环境

### 自定义网络

```shell
# 列出当前的 docker 网络
docker network ls
```

{% asset_img image-20201003014332699.png %}


**网络模式：**

bridge：桥接 docker （默认）

host：和宿主机共享网络环境

none：不配置网络

container：容器网络连通（使用较少）

{% asset_img image-20201003014448468.png %}

{% asset_img image-20201003014844772.png %}


创建了docker 网络之后，我们可以使用 docker network inspect [网络名] 来查看信息

{% asset_img image-20201003014933477.png %}

{% asset_img image-20201003235649747.png %}


好处：

redis、mysql - 不同的集群使用不同的网络，保证集群是安全且健康的



在这里，我认为这个自定义网络与  docker0（默认）是彼此呈兄弟关系的。

### 网络连通

> 核心：--connect

{% asset_img image-20201004003922799.png %}


如字面意思，我们可以将一个容器连接至 docker 网络

{% asset_img image-20201004004009511.png %}


测试：将默认配置的容器连接至自定义的 mynet 网络

```shell
# 连接网络
docker network connect mynet tomcat01
# 查看网络信息
docker network inspect mynet
```

可以看到在containers的值中，多了tomcat01

{% asset_img image-20201004004332752.png %}


在这里我们看到，tomcat01 已被路由至统一子网段下。

这时，这三者可以互相 ping 通

### 实战：Redis 集群部署

> 集群部署的目的：高可用、负载均衡、分片

部署思路：建立一个专有网络，将每一台 redis 部署在这一网络中，健全、互通

创建脚本：

```shell
# 创建网卡
docker network create redis --subnet 172.38.0.0/16

# 通过脚本创建6台redis
for port in $(seq 1 6); \
do \
mkdir -p /mydata/redis/node-${port}/conf
touch /mydata/redis/node-${port}/conf/redis.conf
cat << EOF >/mydata/redis/node-${port}/conf/redis.conf
port 6379
bind 0.0.0.0
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
cluster-announce-ip 172.38.0.1${port}
cluster-announce-port 6379
cluster-announce-bus-port 16379
appendonly yes
EOF
done

docker run -p 637${port}:6379 -p 1637${port}:16379 --name redis-${port} \
-v /mydata/redis/node-${port}/data:/data \
-v /mydata/redis/node-${port}/conf/redis.conf:/etc/redis/redis.conf \
-d --net redis --ip 172.38.0.1${port} redis:5.0.9-alpine3.11 redis-server /etc/redis/redis.conf; \

# 创建集群
redis-cli --cluster create 172.38.0.11:6379 172.38.0.12:6379 172.38.0.13:6379 172.38.0.14:6379 172.38.0.15:6379 172.38.0.16:6379 --cluster-replicas 1
```

{% asset_img image-20201004005617328.png %}


PS：在使用 docker exec 进入 redis 容器内部操作的时候，/bin/bash 会失效，原因是 redis 容器使用最小核心，应使用 **/bin/sh** 来进入指令台

{% asset_img image-20201004010106201.png %}


{% asset_img image-20201004010138494.png %}


显示以上内容时，配置完毕

进入redis，使用 redis-cli 访问，查看cluster info、cluster nodes

{% asset_img image-20201004010529660.png %}


设置值，并停止值直接存储的主机，再次取值

{% asset_img image-20201004010436587.png %}


我们在4号机上得到了3号机的数据，3号机已被代替，架设成功
