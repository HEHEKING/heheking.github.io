---
title: 大数据分析平台架设（一）
date: 2020/11/6 21:21:22
categories:
- 教程
tags:
- 大数据
- zookeeper
---

这一次我写的简单点，适用于已经有 hadoop 基础的人当笔记看。
<!-- more -->

详细版链接：

http://blog.fat-otaku.top:4000/2020/11/05/BigData2020v1.0/


## 一、准备

### （一）软件包

1. CentOS 7_x86_64_Minimal-1804
2. zookeeper-3.4.10
3. JDK 1.8
4. Hadoop 2.7.7
5. MySQL 5.7.25 (rpm-bundle)
6. hive 2.3.7

事实上，还可以配合 Spark + Scala 架设计算引擎。

Spark 和 Scala 的安装教程：



### （二）软件

1. VMware 15 或 其他虚拟机软件
2. WinSCP 或 其他ftp软件
3. putty 或 其他远程会话软件

### （三）初步架设

1. **准备三台 CentOS 7 主机**

   数据分析的集群架设共有3个节点（主节点x1，从节点x2）：master、slave1、slave2；

   但事实上需要额外准备从节点slave3，作为数据库节点来附加至集群，

   以达到动态增删节点的实现；

2. **确认虚拟机网络配置**

   是否可以ping通内外网，确认各节点内网地址

3. **检查ftp软件与会话软件是否成功**

## 二、基础环境搭建

### 1. 修改主机名

修改主机名，以便确认主机职能

```shell
# 修改主机名
hostnamectl set-hostname master
# 立即生效
bash
# 查看主机名
hostname
```

永久修改主机名：

```shell
# 编辑 network 文件
vi /etc/sysconfig/network
```

内容如下：

```
# Create by anaconda
NETWORKING=yes
HOSTNAME=master
```

### 2. 配置 hosts

配置 hosts 的目的在于，各个主机可以通过主机名直接访问到彼此

```shell
# 查看本机IP地址
ip addr
# 编辑 hosts
vi /etc/hosts
# 加入以下内容：
192.168.43.100 master
192.168.43.101 slave1
192.168.43.102 slave2

# 以实际主机IP为准
```

### 3. 关闭防火墙

```shell
# 关闭防火墙
systemctl stop firewalld
# 查看防火墙状态
systemctl status firewalld
```

### 4. 时间同步

集群架设中需要保证主机时间准确，需要进行时间同步

```shell
# 查看本机时间
date
# 设置本机时区，依次选择 5 9 1 ，即是 Asia-China-Beijing 时区
tzselect
```

进行时间同步，我们需要使用到 ntp 服务

在这里，我们将 master 作为 ntp 主机，其他从机作为客户端

```shell
# 安装 ntp
yum install -y ntp

# 此时修改 master 的 ntp 配置文件
vi /etc/ntpd.conf

修改为以下内容：
————————————————————————————————————————
server 127.127.1.0	# local clock
fudge 127.127.1.0 stratum 10	# stratum 设置为其他值亦可，范围为 0 ~ 15
————————————————————————————————————————

# 重启 ntp 服务
systemctl restart ntpd.sercice
# 同步某一 ntp 节点
ntpdate master
```

### 5. 配置 ssh 免密

产生公、私密匙

```shell
ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa
```

Id_dsa.pub 为公钥，id_dsa为私钥

```shell
# 进入之前创建密钥的 /.ssh 目录，配置 master 内回环
cd ~/.ssh
# 将公钥写入 authorized_keys
cat id_dsa.pub >> authorized_keys
# 在master节点上连接自己
ssh master
```

配置子节点，使 master 可以免密访问子节点

```shell
# 将 master 公钥复制到 slave
scp master:~/.ssh/id_dsa.pub ./master_dsa.pub
# 将 master 公钥追加至 authorized_keys
cat master_dsa.pub >> authorized_keys
```

首次连接，需要输入 yes 确认，输入 exit 可注销会话

### 6. 安装 jdk

运行 hadoop 需要 jdk 环境。

```shell
# 建立 JAVA_HOME
mkdir -p /usr/java
# 解压
tar -zxvf /opt/soft/jdk-8u171-linux-x64.tar.gz -C /usr/java/
# 配置环境变量
echo 'export JAVA_HOME=/usr/java/jdk1.8.0_171' >> /etc/profile
echo 'export CLASSPATH=$JAVA_HOME/lib/' >> /etc/profile
echo 'export PATH=$PATH:$JAVA_HOME/bin' >> /etc/profile
echo 'export PATH JAVA_HOME CLASSPATH' >> /etc/profile
# 更新环境变量
source /etc/profile
# 查看 java 版本
java -version
```

快速配置至从节点

```shell
# 分发 jdk
scp -r /usr/java/jdk1.8.0_171 slave1:/usr/java/
# 分发 profile
scp -r /etc/profile slave1:/etc/profile
```

## 二、部署 zookeeper

### 1. 安装

创建目录、并解压

```shell
# 创建目录
mkdir /usr/zookeeper
# 解压
tar -zxvf /opt/soft/zookeeper-3.4.10.tar.gz -C /usr/zookeeper/
```

配置 zoo.cfg 

```shell
# 进入 zookeeper 的安装目录
cd /usr/zookeeper/zookeeper-3.4.10/conf
# 复制 zoo_sample.cfg 重命名为 zoo.cfg
cp zoo_sample.cfg zoo.cfg
```

内容如下：

```wiki
tickTime=2000 
initLimit=10 
syncLimit=5 
# 临时数据路径
dataDir=/usr/zookeeper/zookeeper-3.4.10/zkdata 
clientPort=2181 
# 日志路径
dataLogDir=/usr/zookeeper/zookeeper-3.4.10/zkdatalog 
server.1=master:2888:3888 
server.2=slave1:2888:3888 
server.3=slave2:2888:3888
```

这时，我们需要返回上级路径，并创建 zkdata 、 zkdatalog 目录以存放数据

```shell
# 进入 zkdata
cd zkdata
# 快速创建 myid 文件
vi myid
```

<!--内容为server.*中本机对应的*-->

```
1
```

分发给从节点、

```shell
scp -r /usr/zookeeper slave1:/usr/
scp -r /usr/zookeeper slave2:/usr/
# 在分发过后，应修改 myid 文件
```

### 2. 配置环境变量

```shell
echo 'export ZOOKEEPER_HOME=/usr/zookeeper/zookeeper-3.4.10' >> /etc/profile
echo 'PATH=$PATH:$ZOOKEEPER_HOME/bin' >> /etc/profile
source /etc/profile
```

### 3. 启动集群

```shell
# 进入 zookeeper 程序入口所在目录
cd /usr/zookeeper/zookeeper-3.4.10/bin
# 启动所有节点
./zkServer.sh start
# 查看节点状态
./zkServer.sh status
```

此时应有 leader x1, follower x2

至此，zookeeper 架设完毕