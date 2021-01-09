---
title: 大数据平台架设
date: 2020/11/5 20:46:25
categories:
- 教程
tags:
- 大数据
- zookeeper
- hadoop
- hive
toc:
- true
---

**文章篇幅很长**
<!-- more -->
## 一、准备

### （一）运行环境准备

1. CentOS 7_x86_64_Minimal-1804
2. zookeeper-3.4.10
3. JDK 1.8
4. Hadoop 2.7.7
5. Spark 2.4.5 (for Hadoop 2.7)
6. Scala 2.11.12
7. MySQL 5.7.25 (rpm-bundle)
8. hive 2.3.7

软件包准备就绪后，放置在虚拟机的`/opt/soft/`路径下

### （二）操作软件

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

### （一）修改主机名

（三节点均执行、以master节点为例）

1. 使用root账户登录

2. 修改主机名：

`hostnamectl set-hostname master`

3. 立即生效

`bash`

4. 永久修改主机名

编辑并保存：`/etc/sysconfig/network `

内容如下：

```
#Create by anaconda
NETWORKING=yes
HOSTNAME=master
```

5. 重启计算机后查看是否生效

重启计算机：`reboot`

查看主机名：`hostname`

### （二）配置hosts文件

（三节点均配置）

1. 查看本机IP地址

`ip addr`

2. 修改hosts文件

文件路径：`/etc/hosts`

文件尾加入以下内容

```
192.168.43.100 master
192.168.43.101 slave1
192.168.43.102 slave2

# 以实际IP地址为准
```

### （三）关闭防火墙

（三节点均配置）

1. 关闭防火墙

`systemctl stop firewalld`

2. 查看防火墙状态

`systemctl status firewalld`

### （四）时间同步

​	集群架设中需要保证主机时间准确，各节点时区一致，需要同步	网络时间。

1. 查看本机时间

`date`

2. 设置本机时区（三节点均执行）

`tzselect`

然后依次选择 5 Asia 、9 China 、1 Beijing 即可

3. 安装ntp（三节点均执行）

`yum install -y ntp`

hadoop集群对于时间要求很高，集群内主机要经常同步。

我们使用ntp网络协议进行同步，master作为ntp服务器，其余当做ntp客户端

4. 在master上修改ntp配置文件

文件路径：`/etc/ntp.conf`

修改为：

```
server 127.127.1.0	# local clock
fudge 127.127.1.0 stratum 10	# stratum 设置为其他值亦可，范围为 0 ~ 15
```

5. 重启ntp服务

`systemctl restart ntpd.service`

6. 其他节点进行同步

`ntpdate master`

7. *若无外网三节点则直接设定同一时间

`date -s 10:00`

### （五）配置ssh免密

​		SSH 主要通过 RSA 算法来产生公钥与私钥，在数据传输过程中对数据进行加密来保障数 据的安全性和可靠性，公钥部分是公共部分，网络上任一结点均可以访问，私钥主要用于对数据进行加密，以防他人盗取数据。总而言之，这是一种非对称算法，想要破解还是非常有难度的。hadoop 集群的各个结点之间需要进行数据的访问，被访问的结点对于访问用户结点的可靠性必须进行验证，hadoop 采用的是 ssh 的方法通过密钥验证及数据加解密的方式进行远程安全登录操作，当然，如果 hadoop 对每个结点的访问均需要进行验证，其效率将会大大降低，所以才需要配置 SSH 免密码的方法直接远程连入被访问结点，这样将大大提高访问效率。

1. 产生公、私密匙（三节点均执行）

```
ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa
```

2. 复制 authorized_keys 文件 (仅 master)

Id_dsa.pub 为公钥，id_dsa为私钥，我们需要将公匙复制。

(在`/.ssh`路径下操作)

`cat id_dsa.pub >> authorized_keys`

在master节点上连接自己，也叫作ssh内回环

`ssh master`

3. 子节点配置（仅子节点）

为了实现这个功能，两个 slave 结点的公钥文件中必须要包含主结点的公钥信息，这样当 master 就可以顺利安全地访问这两个 slave 结点了。

slave1 结点通过 scp 命令远程登录 master 结点，并复制 master 的公钥文件到当前的目录下，且重命名为 master_das.pub，这一过程需要密码验证。

```
# 将 master 公钥复制到 slave
scp master:~/.ssh/id_dsa.pub ./master_dsa.pub
# 将 master 公钥追加至 authorized_keys
cat master_dsa.pub >> authorized_keys
```

4. master测试首次连接

`ssh slave1`

在首次连接时需要输入`yes`确认连接，首次连接后输入`exit`可注销会话

5. 操作其他节点

### （六）安装JDK

（三节点均执行）

1. 建立软件路径`/usr/java`

```
mkdir -p /usr/java

tar -zxvf /opt/soft/jdk-8u171-linux-x64.tar.gz -C /usr/java/
```

注意环境所在的路径

查看当前路径:`pwd`

2. 配置环境变量 (修改 profile 文件)

文件位置：`/etc/profile`

添加以下内容：

```
export JAVA_HOME=/usr/java/jdk1.8.0_171 
export CLASSPATH=$JAVA_HOME/lib/ 
export PATH=$PATH:$JAVA_HOME/bin 
export PATH JAVA_HOME CLASSPATH 
```

3. 更新环境变量

`source /etc/profile`

4. 查看java版本

`java -version`

5. *快速配置至从节点

profile文件与java环境是通用的

所以我们可以用scp命令直接传给从节点

`scp -r /usr/java/jdk1.8.0_171 slave1:/usr/java/`

`scp -r /etc/profile slave1:/etc/profile`

事实上更换profile时直接用ftp软件复制粘贴来的更快。。。

### （七）zookeeper安装

1. 创建目录并解压环境

```
mkdir /usr/zookeeper
tar -zxvf /opt/soft/zookeeper-3.4.10.tar.gz -C /usr/zookeeper/
```

2. 配置 `zoo.cfg` 文件

进入路径`cd /usr/zookeeper/zookeeper-3.4.10/conf`

复制 `zoo_sample.cfg` 并重命名为`zoo.cfg`

```
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

返回上级目录，并创建`zkdata`、 `zkdatalog`两个文件夹以存放数据

3. 创建`myid`文件

进入`zkdata`文件夹，并创建文件`myid`

在这里使用`vi myid`可快捷创建

内容为

<!--内容为server.*中本机对应的*-->

```
1
```

4. 分发给从节点

```
scp -r /usr/zookeeper slave1:/usr/
scp -r /usr/zookeeper slave2:/usr/
```

5. 从节点修改`myid`文件中的id

6. 配置环境变量

添加以下内容

```
export ZOOKEEPER_HOME=/usr/zookeeper/zookeeper-3.4.10  
PATH=$PATH:$ZOOKEEPER_HOME/bin
```

更新环境变量

`source /etc/profile`

7. 启动zookeeper集群

`cd /usr/zookeeper/zookeeper-3.4.10`

启动所有节点

`bin/zkServer.sh start`

三节点均启动完毕后，查看状态

`bin/zkServer.sh status`

应有`leader`x1、`follower`x2

至此，zookeeper架设完成

## 三、 安装hadoop

### （一）准备

1. 创建环境目录并解压

`mkdir /usr/hadoop`

`tar -zxvf /opt/soft/hadoop-2.7.7.tar.gz -C /usr/hadoop`

2. 配置环境变量，添加以下内容（三节点均执行）

```
export HADOOP_HOME=/usr/hadoop/hadoop-2.7.7
export CLASSPATH=$CLASSPATH:$HADOOP_HOME/lib 
export PATH=$PATH:$HADOOP_HOME/bin
```

更新环境变量

`source /etc/profile`

3. 修改hadoop环境配置文件`hadoop-env.sh`

修改其中的JAVA_HOME配置

`export JAVA_HOME=/usr/java/jdk1.8.0_171`

### （二）配置文件

1. 配置`core-site.xml`文件

```
<configuration>
<property>
	<name>fs.default.name</name>
	<value>hdfs://master:9000</value>
</property>

<property>
	<name>hadoop.tmp.dir</name>
	<value>/usr/hadoop/hadoop-2.7.7/hdfs/tmp</value>
	<description>A base for other temporary directories.</description>
</property>

<property>
	<name>io.file.buffer.size</name>
	<value>131072</value>
</property>

<property>
	<name>fs.checkpoint.period</name>
	<value>60</value>
</property>

<property>
	<name>fs.checkpoint.size</name>
	<value>67108864</value>
</property>
</configuration>
```

2. 配置`yarn-site.xml`文件

注意下列几个`.address`的端口，之后会用浏览器访问测试架设是否成功，尤其是18088端口。

```
<configuration> 
<property> 
	<name>yarn.resourcemanager.address</name> 
	<value>master:18040</value> 
   </property>
<property> 
	<name>yarn.resourcemanager.scheduler.address</name> 
	   <value>master:18030</value> 
   </property>
	<property> 
		<name>yarn.resourcemanager.webapp.address</name> 
	   <value>master:18088</value> 
	</property> 
   <property> 
	<name>yarn.resourcemanager.resource-tracker.address</name> 
	   <value>master:18025</value> 
</property> 
   <property> 
	<name>yarn.resourcemanager.admin.address</name> 
	   <value>master:18141</value> 
</property> 
   <property> 
	<name>yarn.nodemanager.aux-services</name> 
	   <value>mapreduce_shuffle</value> 
	</property> 
   <property> 
		<name>yarn.nodemanager.auxservices.mapreduce.shuffle.class</name> 
	<value>org.apache.hadoop.mapred.ShuffleHandler</value> 
   </property> 
</configuration>
```

3. 在同路径下配置`master`、`slaves`文件，添加主、从节点的IP

master文件内容：

```
master
```

slaves文件内容：

```
slave1
slave2
```

4. 配置`hdfs-site.xml`文件

同样注意一下的几个端口，会用浏览器访问

```
<configuration> 
<property> 
	<name>dfs.replication</name> 
	<value>2</value> 
   </property> 
   <property> 
		<name>dfs.namenode.name.dir</name> 
		<value>file:/usr/hadoop/hadoop-2.7.3/hdfs/name</value> 
		<final>true</final> 
	</property> 
	<property> 
		<name>dfs.datanode.data.dir</name> 
		<value>file:/usr/hadoop/hadoop-2.7.3/hdfs/data</value> 
	   <final>true</final> 
	</property> 
	<property> 
	<name>dfs.namenode.secondary.http-address</name> 
	   <value>master:9001</value> 
	</property> 
	<property> 
	<name>dfs.webhdfs.enabled</name> 
		<value>true</value> 
	</property> 
   <property> 
	<name>dfs.permissions</name> 
		<value>false</value> 
	</property> 
</configuration>
```

5. 配置`mapred-site.xml`文件

```
<configuration>
<property> 
		<name>mapreduce.framework.name</name> 
		<value>yarn</value> 
</property>
</configuration>
```

### （三）分发hadoop

​	master节点中已经配置好了hadoop，通过scp命令将hadoop分发给从节点。（直接用ftp远程复制也无所谓）

1. 用scp分发

```
scp -r /usr/hadoop slave1:/usr/ 
scp -r /usr/hadoop slave2:/usr/
```

因为之前在配置master时，已经对三节点profile文件配置过，所以不需要配置环境变量。

更新环境变量

`source /etc/profile`

2. 在master中格式化hadoop

`hadoop namenode -format`

格式化完成后，进入路径：

`cd /usr/hadoop/hadoop-2.7.7/sbin`

启动hadoop集群：

`./start-all.sh`

检查启动情况：

`jps`

若配置无误，则在master节点中存在进程 `SecondaryNameNode`、`ResourceManager`、`NameNode`三个进程。

（`QuorumPeerMain`是zookeeper的进程）

在子节点中，则有`DataNode`、`NodeManager`两个进程

3. 测试

使用浏览器访问主节点地址的50070端口

`http://192.168.43.100:50070/`

该页面是hdfs的web管理页面，若显示无误，则大概率架设成功

### （四）hdfs的简单应用

1. 查看文件目录

`hadoop fs -ls /    #(最开始创建的是一个空的文件系统所以什么也没有) `

2. 创建文件夹  `helloWorld`

`hadoop fs -mkdir /a   # (在 hdfs 上传到 a 文件夹)`

3. 上传数据

```
# 创建目录
hadoop fs -mkdir -p /college
# 上传文件
hadoop fs -put /root/college/loan.csv /college/
hadoop fs -ls /college/
```



## 四、安装spark

### （一）安装scala环境

1. 创建目录、解压tar包

2. 配置环境变量

```
export SCALA_HOME=/usr/scala/scala-2.11.12
export PATH=$SCALA_HOME/bin:$PATH
```

更新环境变量

3. 测试

`scala -version`

4. 分发scala至子节点

`scp -r /usr/scala slave1:/usr/`

### （二）安装spark

1. 创建目录、解压tar包

2. 配置环境变量

```
export SPARK_HOME=/usr/spark/spark-2.4.5-bin-hadoop2.7
export PATH=$SPARK_HOME/bin:$PATH
```

3. 修改`spark-env.sh`

文件位置：`/usr/spark/spark-2.4.5-bin-hadoop2.7/conf`

复制一份`spark-env.sh.template`并重命名

内容如下：

```
#!/usr/bin/env bash

export SPARK_MASTER_IP=master 
export SCALA_HOME=/usr/scala/scala-2.11.12 
export SPARK_WORKER_MEMORY=8g 
export JAVA_HOME=/usr/java/jdk1.8.0_171 
export HADOOP_HOME=/usr/hadoop/hadoop-2.7.7
export HADOOP_CONF_DIR=/usr/hadoop/hadoop-2.7.7/etc/hadoop
```

4. 配置spark从节点（修改slaves文件）

同样复制一遍.template模板，重命名为`slaves`

内容就是从节点的IP

```
slave1
slave2
```

5. 分发spark

```
scp -r /usr/spark slave1:/usr/ 
scp -r /usr/spark slave2:/usr/
```

记得配置并更新环境变量

6. 测试spark环境

先启动hadoop环境

`/usr/hadoop/hadoop-2.7.7/sbin/start-all.sh`

然后开启spark集群

`/usr/spark/spark-2.4.5-bin-hadoop2.7/sbin/start-all.sh`

浏览器访问master节点的8080端口即可，这是Spark-Web的界面

`http://192.168.43.100:8080/`

访问成功则说明部署完毕

7. 测试spark-shell环境

进入spark-shell

`spark-shell`

spark-shell 此时进入的是 scala 环境的 spark 交互模式

输入命令进入 python 环境下的 spark 交互模式

`pyspark`

至此我们的spark on yarn环境架设完毕

## 五、Something

### （一）配置文件合集

1. `/etc/profile`

```
export JAVA_HOME=/usr/java/jdk1.8.0_171 
export CLASSPATH=$JAVA_HOME/lib/ 
export PATH=$PATH:$JAVA_HOME/bin 
export PATH JAVA_HOME CLASSPATH

export ZOOKEEPER_HOME=/usr/zookeeper/zookeeper-3.4.14
PATH=$PATH:$ZOOKEEPER_HOME/bin

export HADOOP_HOME=/usr/hadoop/hadoop-2.7.7
export CLASSPATH=$CLASSPATH:$HADOOP_HOME/lib 
export PATH=$PATH:$HADOOP_HOME/bin

export SCALA_HOME=/usr/scala/scala-2.11.12
export PATH=$SCALA_HOME/bin:$PATH

export SPARK_HOME=/usr/spark/spark-2.4.5-bin-hadoop2.7
export PATH=$SPARK_HOME/bin:$PATH

export HIVE_HOME=/usr/hive/apache-hive-2.3.7-bin
export PATH=$PATH:$HIVE_HOME/bin
```

2. `/etc/sysconfig/network-scripts/ifcfg-ens33`

这里所填写的信息可以通过在物理机windows环境的cmd中输入`ipconfig`获取

```
TYPE=Ethernet
# 静态IP
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
DEVICE=ens33
ONBOOT=yes
# 指定本机IP
IPADDR=192.168.43.100
# 所处网络的子网掩码
NETMASK=255.255.255.0
# 所处网络的网关
GATEWAY=192.168.43.1
# DNS
DNS1=8.8.8.8
DNS2=114.114.114.114
```

### （二）一些问题

1. 无法使用 vim 或 wget

获取通过yum安装的软件

`yum list installed`

查询指定软件是否安装，如是否安装wget（即在已安装的软件中筛选出wget）

`yum list installed | grep wget`

### （三）修改本地源

1. 备份当前yum源

防止出现意外还可以还原回来

```
cd /etc/yum.repos.d/
cp /CentOS-Base.repo /CentOS-Base-repo.bak
```

2. 下载yum源repo文件

示例：下载阿里源

`wget http://mirrors.aliyun.com/repo/Centos-7.repo`

3. 清理旧包

`yum clean all`

4. 设置默认源

把下载下来阿里云repo文件设置成为默认源

`mv Centos-7.repo CentOS-Base.repo`

5. 生成yum缓存并更新yum源

```
yum makecache
yum update
```



## 六、安装hive

​		hive是基于[Hadoop](https://baike.baidu.com/item/Hadoop/3526507)的一个[数据仓库](https://baike.baidu.com/item/数据仓库/381916)工具，用来进行数据提取、转化、加载，这是一种可以存储、查询和分析存储在Hadoop中的大规模数据的机制。hive数据仓库工具能将结构化的数据文件映射为一张数据库表，并提供[SQL](https://baike.baidu.com/item/SQL/86007)查询功能，能将[SQL语句](https://baike.baidu.com/item/SQL语句/5714895)转变成[MapReduce](https://baike.baidu.com/item/MapReduce/133425)任务来执行。Hive的优点是学习成本低，可以通过类似SQL语句实现快速MapReduce统计，使MapReduce变得更加简单，而不必开发专门的MapReduce应用程序。hive是十分适合数据仓库的统计分析和[Windows](https://baike.baidu.com/item/Windows/165458)注册表文件。

### （一）运行模式

​	hive拥有三种运行模式

1. 内嵌模式

元数据保村在内嵌的derby中，允许一个会话链接，尝试多个会话链接时会报错。

在使用这种模式时，每次使用前需要初始化

`schematool -dbType derby -initSchema`

随后即可在该路径下访问hive

2. 本地模式

本地安装MySQL替代derby存储元数据。
由于元数据的获取需要访问MySQL，所以这就要求每一个用户必须要有对MySQL的访问权利。

3. 远程模式

以本地模式为基础。
MySQL数据库所在的节点提供metastore service服务，其他节点可以连接该服务来获取元数据信息。
各种客户端通过 beeline 来连接，连接之前无需知道数据库的用户名和密码。

### （二）配置hive环境

**直接上远程模式的配置**

1. 创建目录，解压tar包

2. 配置并更新环境变量

3. 拷贝MySQL驱动`mysql-connector-java`

由于hive默认并不存在MySQL驱动，所以需要将准备好的`mysql-connector-java-5.1.47-bin.jar`拷贝至`/usr/hive/apache-hive-2.3.7-bin/lib`

同时，将同路径下的`jline-2.12.jar`包保存至本地

将该文件拷贝至`/usr/hadoop/hadoop-2.7.7/share/hadoop/yarn/lib`路径下

4. 修改`hive-env.sh`文件

完整内容如下

```
# Set HADOOP_HOME to point to a specific hadoop install directory
HADOOP_HOME=/usr/hadoop/hadoop-2.7.7

# Hive Configuration Directory can be controlled by:
export HIVE_CONF_DIR=/usr/hive/apache-hive-2.3.7-bin/conf

# Folder containing extra libraries required for hive compilation/execution can be controlled by:
export HIVE_AUX_JARS_PATH=/usr/hive/apache-hive-2.3.7-bin/lib
```

5. 分发hive至服务端节点

`scp -r /usr/hive  slave1:/usr/`

6. 配置`hive-site.xml`

master的配置（客户端）：

```
<configuration>
<!--数据仓库默认路径-->
<property>
	<name>hive.metastore.warehouse.dir</name>
	<value>/usr/hive/apache-hive-2.3.7-bin/warehouse</value>
	<description>location of default database for the warehouse</description>
</property>

<!--使用本地服务连接hive，默认为true，因master作为客户机，设置为false-->
<property>
	<name>hive.metastore.local</name>
	<value>false</value>
</property>

<!--连接数据库-->
<property>
	<name>hive.metastore.uris</name>
	<value>thrift://slave1:9083</value>
	<description>Thrift URI for the remote metastore. Used by metastore client to connect to remote metastor</description>
</property>
</configuration>
```

slave1的配置（服务端）：

```
<configuration>
<!-- 连接元数据库-->
<property>
	<name>javax.jdo.option.ConnectionURL</name>
	<value>jdbc:mysql://slave2:3306/hivedb?createDatabaseIfNotExist=true&amp;useSSL=false</value>
	<description>
	  JDBC connect string for a JDBC metastore.
	  To use SSL to encrypt/authenticate the connection, provide database-specific SSL flag in the connection URL.
	  For example, jdbc:postgresql://myhost/db?ssl=true for postgres database.
	</description>
</property>

<!--连接数据库驱动-->
<property>
	<name>javax.jdo.option.ConnectionDriverName</name>
	<value>com.mysql.jdbc.Driver</value>
	<description>Driver class name for a JDBC metastore</description>
</property>

<!--连接数据库用户名-->
<property>
	<name>javax.jdo.option.ConnectionUserName</name>
	<value>root</value>
	<description>Username to use against metastore database</description>
</property>

<!--连接数据库密码-->
<property>
	<name>javax.jdo.option.ConnectionPassword</name>
	<value>123456</value>
	<description>password to use against metastore database</description>
</property>

<!--默认数据仓库路径，即default数据库的存储位置-->
<property>
	<name>hive.metastore.warehouse.dir</name>
	<value>/usr/hive/apache-hive-2.3.7-bin/warehouse</value>
	<description>location of default database for the warehouse</description>
</property>

<!--客户端显示当前数据库名称信息-->
<property>
	<name>hive.cli.print.current.db</name>
	<value>true</value>
	<description>Whether to include the current database in the Hive prompt.</description>
</property>

<!--客户端显示当前查询表的头信息-->
<property>
	<name>hive.cli.print.header</name>
	<value>true</value>
	<description>Whether to print the names of the columns in query output.</description>
</property>

<!--官方推荐设置此项，以解决多并发读取失败的问题（HIVE-4762）-->
<property>
	<name>datanucleus.autoStartMechanism</name>
	<value>SchemaTable</value>
</property>

<!--再配置元数据认证为false，否则启动时会报以下异常
	Caused by: MetaException(message:Version information not found in metastore. )-->
<property>
   <name>hive.metastore.schema.verification</name>
   <value>false</value>
   <description>  
	Enforce metastore schema version consistency.  
	True: Verify that version information stored in metastore matches with one from Hive jars.  Also disable automatic schema migration attempt. Users are required to manully migrate schema after Hive upgrade which ensures proper metastore schema migration. (Default)  
	False: Warn if the version information stored in metastore doesn't match with one from in Hive jars.  
	</description>  
</property>

<!--配置 io 临时目录，否则会报异常-->
<property>
	<name>system:java.io.tmpdir</name>
	<value>/usr/hive/apache-hive-2.3.7-bin/hive_tmp/</value>
</property>
<!--系统用户名-->
<property>
	<name>system:user.name</name>
	<value>root</value>
</property>
</configuration>
```

slave2的作用是数据库节点，因此不需要安装hive环境。

### （三）修改hive配置

1. hive运行日志信息配置

在hive中，使用的是Log4j来输出日志，默认情况下，CLI是不能将日志信息输出到控制台的。

在hive 0.13.0 之前的版本，默认的日志级别是WARN，在hive 0.13.0 之后，默认的日志级别是INFO。

默认的日志存放在 `/tmp/$(user.name)` 文件夹的 `hive.log` 文件内。

在 `$(HIVE_HOME)/conf/hive-log4j2.properties.template` 文件中记录了hive日志的存储情况，我们可以通过修改以上两个参数的值来重新设置hive log日志的存放地址

## 七、安装MySQL

**hive使用远程模式，即MySQL只需要在数据节点slave2上部署即可**

### （一）准备

1. 卸载MariaDB

因为CentOS 7 系统自带了MariaDB，这与MySQL是冲突的，我们需要先卸载MariaDB。

```
# 首先获取mariadb的包名
rpm -qa | grep mariadb
# 会得到mariadb-libs-x.x
rpm -e mariadb-libs-x.x --nodeps
```

2. 使用 `rpm -ivh` 命令一次安装以下组件

`-ivh`：`--install--verbose--hash`

1. `mysql-community-common`
2. `mysql-community-libs`
3. `mysql-community-libs-compat`
4. `mysql-community-client`
5. `mysql-community-server`

### （二）配置MySQL

1. 初始化MySQL数据库

​		安装好MySQL后，我们需要初始化数据库，初始化和启动数据库时最好不要使用root用户，而是使用mysql用户启动

初始化指令：

`/usr/sbin/mysqld --initialize-insecure --user=mysql`

​		对于MySQL 5.7.6以后的5.7系列版本，MySQL也可以使用`mysqld --initialize`初始化数据库，该命令会在`/var/log/mysqld.log`文件中生成一个登陆MySQL的随机密码，而`mysqld --initialize-insecure`命令**不会生成随机密码**，而是**设置MySQL的密码为空**。

2. 启动MySQL服务

使用如下命令开启MySQL服务，让其在后台运行

`/usr/sbin/mysqld --user=mysql &`

加"&"使脚本放到后台运行

执行后，系统会打印出命令执行的PID（进程号）

重载所有修改过的配置文件：`systemctl daemon-reload`

开启MySQL服务：`systemctl start mysqld`

开机自启服务：`systemctl enable mysqld`

3. 登陆MySQL

使用root用户无密码登陆MySQL

`mysql -uroot`

4. 重置root用户密码为123456

`mysql> alter user 'root'@'localhost' identified by '123456';`

若显示`Query OK, X rows affected (x.xx sec)`则表示操作成功

成功后 `exit` 或 `quit` 退出MySQL，重新登录验证密码

`mysql -uroot -p`

若显示`Enter Password >`则表示密码设置成功，输入密码登录即可

5. MySQL密码安全测试

设置密码强度为低级：

`set global validate_password_policy=0`

设置标准密码的长度：

`set global validate_password_length=4`

### （三）增加远程登录权限

1. 修改user表

```
mysql > use mysql;
# 查询用户信息
mysql > select user,host from user;
```

通过授权法实现远程连接，将host字段的值改为%就表示在任何客户端机器上都能以root用户登录到MySQL服务器，简易在开发时设置为%。

```
mysql > update user set host='%' where host='localhost';
# 刷新配置信息
mysql > flush pricileges;
```

再次查询用户信息后，若host字段为%，则修改成功

2. 授权及生效

允许远程连接：

`grant all privileges on *.* to 'root'@'%' with grant option`

刷新权限：

`flush privileges`

## 八、 启动hive服务

### （一）初始化服务

1. 服务端节点初始化元数据库

因为使用hive的远程模式，所以在服务端节点初始化数据库即可

`schematoll -dbType mysql -initSchema`

若显示 `schemaTool completed` 则表示初始化成功

2. 服务端节点启动metastore服务

`hive --service metastore` + & 可运行在后台

### （二）测试

​	1. 测试

​		master节点作为客户端开启hive服务

​		直接输入：`hive` 指令以启动客户端

​	至此，hive服务架设完毕

### （三）hive的简单应用

​	即HQL

1. 建库、建表

与SQL没有太大差别

```
create database newdb;
create table newtable(...);
```

2. 数据导入

将本地数据导入至对应表

`load data local inpath '/root/college/loan/csv' into table loan`

统计表数据，结果写入本地  `/root/college000/01`  中

```
insert overwrite local directory '/root/college000/01'
row format delimited fields terminated by '\t'
select count(*) from loan
```

### （四）数据分析简单示例

1. 在场景：整体借款 的数据中，计算信用得分Prosper Score对于借款的影响。

（以信用得分为变量，统计借款次数）

即了解大多数借款人的信用得分

```
select ProsperScore, count(*) as sum from loan
group by ProsperScore
order by sum by desc;
```

2. 找出借款较容易的行业前5及对应借款次数；

（借款次数）

```
select occupation, count(*) as sum from loan
group by occupation
oreder by sum by desc limit 5;
```

3. 根据Apriori关联规则算法的原理找出与违约最多的（借款状态，后项）之间的关联度最强的职业（前项），并计算出其支持与置信度。要求如下：

支持度写到本地  `/root/college009/`  中（保留五位小数）

置信度写到本地  `/root/college010/`  中（保留五位小数）

4. 关联规则

关联规则：用于表示数据内隐含的关联性

定义一个关联规则：

A → B

其中A和B表示的是两个互斥时间，A称为前因（Antecedent），B称为后果（Consequent），上述关联规则表示A会导致B。

关联规则分析（Association Rule Analysis）就是为了发掘购物数据背后的商机而诞生的。

例如：购买尿布的人往往会购买啤酒

公式参考如下：

前项：A	后项：B

支持度：表示同时包含A和B的事务占所有事务的比例。如果用P(A)表示使用A事务的比例。Support=P(A&B)

置信度：表示使用包含A的事务中同时包含B事务的比例，即同时包含A和B的事务占包含A事务的比例。Confidence=P(A&B)/P(A)

## 九、数据分析

### （一）操作环境

1. Python 3.7（Anaconda 3）
2. 可使用Jupyter notebook 或其他 IDE
3. pandas（统计分析、预处理）、matplotlib（可视化）、sklearn（建模）等数据分析工具库

### （二）操作步骤

1. 需求分析
2. 数据获取
3. 数据预处理
4. 分析与建模
5. 模型评价与优化
6. 部署

### （三）简单操作

1. 创建csv文件

```python
import pandas as pd

ls = [['张三','男',16],['李四','男',17],['刘红','女',9]]
title = ['name', 'sex', 'age']
dataFrame = pd.DataFrame(columns=name, data=ls)
# index=false 声明dataframe中数据体左侧的序号列不随数据写入csv文件内
dataFrame.to_csv('./hello.csv', index = False)
```

读取csv文件

```
dataFrame = pd.read_csv('hello.csv', header = 0, encoding = 'utf-8')
```

数据处理完毕后，将csv文件导入hdfs，并使用hive处理即可。

hive导入csv文件进数据库：

```
load data local inpath 'linux的csv文件路径' into table '表名';
```