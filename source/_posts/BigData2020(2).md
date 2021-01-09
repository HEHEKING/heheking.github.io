---
title: 大数据分析平台架设（二）
date: 2020/11/8 21:21:22
categories:
- 教程
tags:
- 大数据
- hadoop
---
前置：http://blog.fat-otaku.top:4000/2020/11/06/BigData2020(1)/

详细版：http://blog.fat-otaku.top:4000/2020/11/05/BigData2020v1.0/
<!-- more -->
## 三、安装 hadoop

### 1. 准备

创建目录

```shell
# 创建目录
mkdir /usr/hadoop
# 解压
tar -zxvf /opt/soft/hadoop-2.7.7.tar.gz -C /usr/hadoop
```

配置环境变量

```shell
echo 'export HADOOP_HOME=/usr/hadoop/hadoop-2.7.7' >> /etc/profile
echo 'export CLASSPATH=$CLASSPATH:$HADOOP_HOME/lib' >> /etc/profile
echo 'export PATH=$PATH:$HADOOP_HOME/bin' >> /etc/profile
```

更新环境变量

```shell
source /etc/profile
```

修改 hadoop 环境配置文件 hadoop-env.sh

修改其中的 JAVA_HOME 路径

```
export JAVA_HOME=/usr/java/jdk1.8.0_171
```

### 2. 配置文件

#### core-site.xml

```xml
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

#### yarn-site.xml

```xml
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

#### hdfs-site.xml

```xml
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

#### mapred-site.xml

```xml
<configuration>
	<property> 
   		<name>mapreduce.framework.name</name> 
   		<value>yarn</value> 
	</property>
</configuration>
```

#### 配置集群文件

在上述两个文件下创建 master / slaves 文件，内容分别为主、从节点的IP

master 文件内容：

```
master
```

slaves文件内容：

```
slave1
slave2
```

### 3. 分发 hadoop

#### scp 分发

```shell
scp -r /usr/hadoop slave1:/usr/ 
scp -r /usr/hadoop slave2:/usr/
```

#### namenode 格式化

```shell
hadoop namenode -format
```

#### 启动集群

```shell
cd /usr/hadoop/hadoop-2.7.7/sbin
./start-all.sh
```

#### 检查集群状态

```
jps
```

若配置无误，则在master节点中存在进程 `SecondaryNameNode`、`ResourceManager`、`NameNode`三个进程

至此，hadoop 已被架设

可访问 `http://master:50070/` 来测试

### 4. hdfs 简单应用

```
# 查看文件目录
hadoop fs -ls /
# 创建文件夹
hadoop fs -mkdir /newDir
# 上传文件
hadoop fs -mkdir -p /college
hadoop fs -put /root/college/loan.csv /college/
hadoop fs -ls /college/
```

### 5. hadoop 动态增删节点

**添加节点**

```
# 修改slaves文件、添加 
slave3

# 在slave3启动datanode/nodemanager
./hadoop-daemon.sh start datanode
./yarn-daemon.sh start nodemanager

# 刷新主节点
hdfs dfsadmin -refreshNodes

#均衡blocks
sbin/start-balancer.sh

#查看节点信息
hdfs dfsadmin -report
```

**删除节点**

```
# 拒绝列表excludes添加slave2
slave2

# 刷新主节点
hdfs dfsadmin -refreshNodes

# 查看节点信息
hdfs dfsadmin -report
若slave2状态为decommision in progress，则成功，执行完毕后会变为decommisioned

# 完全关闭进程
./hadoop-daemon.sh stop datanode
./yarn-daemon.sh stop nodemanager

# 均衡blocks
sbin/start-balancer.sh

# 强制active
hdfs haadmin -transitionToActive nn2

约十分钟左右，节点会转变为dead
```

## 四、高可用 hadoop

在一个典型的 HA 集群中，每个 NameNode都是一台独立的服务器，在任意时刻，只有一个 NameNode处于 active 状态，另一个处于 standby 状态。

active 状态的 NameNode负责 所有的客户端操作

standby 状态的 NameNode 处于从属地位，维护数据状态，随时准备切换，只进行数据同步，时刻准备提供服务。

### 1. 共享数据

NameNode 之间共享数据（ NFS 、 Quorum Journal Node ）

两个 NameNode 为了数据同步，会通过一组 JournalNodes 的独立进程进行相互通信

当 active 状态的 NameNode 的命名空间有任何修改时，会告知大部分的 JournalNodes 进程

当 standby 状态的 NameNode 有能力读取 JournalNodes中的变更信息，并且一直监控 edit log 的变化，并应用于自己的命名空间

### 2. 配置

需要 zookeeper 集群部署

所有节点都需要配置免密

### hdfs-site.xml

```xml
<configuration> 
  <!-- 指定NameService的名称 -->  
  <property> 
    <name>dfs.nameservices</name>  
    <value>mycluster</value> 
  </property>  
  <!-- 指定NameService下两个NameNode的名称 -->  
  <property> 
    <name>dfs.ha.namenodes.mycluster</name>  
    <value>nn1,nn2</value> 
  </property>  
  <!-- 分别指定NameNode的RPC通讯地址 -->  
  <property> 
    <name>dfs.namenode.rpc-address.mycluster.nn1</name>  
    <value>192.168.1.80:8020</value> 
  </property>  
  <property> 
    <name>dfs.namenode.rpc-address.mycluster.nn2</name>  
    <value>192.168.1.81:8020</value> 
  </property>  
  <!-- 分别指定NameNode的可视化管理界面的地址 -->  
  <property> 
    <name>dfs.namenode.http-address.mycluster.nn1</name>  
    <value>192.168.1.80:50070</value> 
  </property>  
  <property> 
    <name>dfs.namenode.http-address.mycluster.nn2</name>  
    <value>192.168.1.81:50070</value> 
  </property>  
  <!-- 指定NameNode编辑日志存储在JournalNode集群中的目录-->  
  <property> 
    <name>dfs.namenode.shared.edits.dir</name>  
  	<value>qjournal://192.168.1.80:8485;192.168.1.81:8485;192.168.1.82:8485/mycluster</value> 
  </property>
  <!-- 指定JournalNode集群存放日志的目录-->  
  <property> 
    <name>dfs.journalnode.edits.dir</name>  
    <value>/usr/hadoop/hadoop-2.9.0/journalnode</value> 
  </property>  
  <!-- 配置NameNode失败自动切换的方式-->  
  <property> 
    <name>dfs.client.failover.proxy.provider.mycluster</name>  
    <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value> 
  </property>  
  <!-- 配置隔离机制-->  
  <property> 
    <name>dfs.ha.fencing.methods</name>  
    <value>sshfence</value> 
  </property>  
  <!-- 由于使用SSH,那么需要指定密钥的位置-->  
  <property> 
    <name>dfs.ha.fencing.ssh.private-key-files</name>  
    <value>/root/.ssh/id_rsa</value> 
  </property>  
  <!-- 开启失败故障自动转移-->  
  <property> 
    <name>dfs.ha.automatic-failover.enabled</name>  
    <value>true</value> 
  </property>  
  <!-- 配置Zookeeper地址-->  
  <property> 
    <name>ha.zookeeper.quorum</name>  
    <value>192.168.1.80:2181,192.168.1.81:2181,192.168.1.82:2181</value> 
  </property>  
  <!-- 文件在HDFS中的备份数(小于等于DataNode) -->  
  <property> 
    <name>dfs.replication</name>  
    <value>3</value> 
  </property>  
  <!-- 关闭HDFS的访问权限 -->  
  <property> 
    <name>dfs.permissions.enabled</name>  
    <value>false</value> 
  </property>  
  <!-- 指定一个配置文件,使NameNode过滤配置文件中指定的host -->  
  <property> 
    <name>dfs.hosts.exclude</name>  
    <value>/usr/hadoop/hadoop-2.9.0/etc/hadoop/hdfs.exclude</value> 
  </property> 
</configuration>
```

### core-site.xml

```xml
<configuration> 
  <!-- Hadoop工作目录,用于存放Hadoop运行时产生的临时数据 -->
  <property> 
    <name>hadoop.tmp.dir</name>  
    <value>/usr/hadoop/hadoop-2.9.0/data</value> 
  </property>  
  <!-- 默认的NameNode,使用NameService的名称 -->  
  <property> 
    <name>fs.defaultFS</name>  
    <value>hdfs://mycluster</value> 
  </property>  
  <!-- 开启Hadoop的回收站机制,当删除HDFS中的文件时,文件将会被移动到回收站(/usr/<username>/.Trash),在指定的时间过后再对其进行删除,此机制可以防止文件被误删除 -->  
  <property> 
    <name>fs.trash.interval</name>  
    <!-- 单位是分钟 -->  
    <value>1440</value> 
  </property> 
</configuration> 
```

### yarn-site.xml

```xml
<configuration> 
  <!-- 配置Reduce取数据的方式是shuffle(随机) -->  
  <property> 
    <name>yarn.nodemanager.aux-services</name>  
    <value>mapreduce_shuffle</value> 
  </property>  
  <!-- 开启日志 -->  
  <property> 
    <name>yarn.log-aggregation-enable</name>  
    <value>true</value> 
  </property>  
  <!-- 设置日志的删除时间 -1:禁用,单位为秒 -->  
  <property> 
    <name>yarn.log-aggregation。retain-seconds</name>  
    <value>864000</value> 
  </property>  
  <!-- 设置yarn的内存大小,单位是MB -->  
  <property> 
    <name>yarn.nodemanager.resource.memory-mb</name>  
    <value>8192</value> 
  </property>  
  <!-- 设置yarn的CPU核数 -->  
  <property> 
    <name>yarn.nodemanager.resource.cpu-vcores</name>  
    <value>8</value> 
  </property>
  <!-- YARN HA配置 -->  
  <!-- 开启yarn -->  
  <property> 
    <name>yarn.resourcemanager.ha.enabled</name>  
    <value>true</value> 
  </property>  
  <!-- 指定yarn ha的名称 -->  
  <property> 
    <name>yarn.resourcemanager.cluster-id</name>  
    <value>cluster1</value> 
  </property>  
  <!-- 分别指定两个ResourceManager的名称 -->  
  <property> 
    <name>yarn.resourcemanager.ha.rm-ids</name>  
    <value>rm1,rm2</value> 
  </property>  
  <!-- 分别指定两个ResourceManager的地址 -->  
  <property> 
    <name>yarn.resourcemanager.hostname.rm1</name>  
    <value>192.168.1.80</value> 
  </property>  
  <property> 
    <name>yarn.resourcemanager.hostname.rm2</name>  
    <value>192.168.1.81</value> 
  </property>  
  <!-- 分别指定两个ResourceManager的Web访问地址 -->  
  <property> 
    <name>yarn.resourcemanager.webapp.address.rm1</name>  
    <value>192.168.1.80:8088</value> 
  </property>  
  <property> 
    <name>yarn.resourcemanager.webapp.address.rm2</name>  
    <value>192.168.1.81:8088</value> 
  </property>  
  <!-- 配置使用的Zookeeper集群 -->  
  <property> 
    <name>yarn.resourcemanager.zk-address</name>  
    <value>192.168.1.80:2181,192.168.1.81:2181,192.168.1.82:2181</value> 
  </property>  
  <!-- ResourceManager Restart配置 -->  
  <!-- 启用ResourceManager的restart功能,当ResourceManager重启时将会保存当前运行的信息到指定的位置,当重启成功后自动进行读取 -->  
  <property> 
    <name>yarn.resourcemanager.recovery.enabled</name>  
    <value>true</value> 
  </property>  
  <!-- ResourceManager Restart使用的存储方式(实现类) -->  
  <property> 
    <name>yarn.resourcemanager.store.class</name>  
    <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value> 
  </property>  
  <!-- ResourceManager重启时数据保存在Zookeeper中的目录 -->  
  <property> 
    <name>yarn.resourcemanager.zk-state-store.parent-path</name>  
    <value>/rmstore</value> 
  </property>  
  <!-- NodeManager Restart配置 -->  
  <!-- 启用NodeManager的restart功能,当NodeManager重启时将会保存当前运行时的信息到指定的位置,重启成功后自动进行读取 -->  
  <property> 
    <name>yarn.nodemanager.recovery.enabled</name>  
    <value>true</value> 
  </property>  
  <!-- NodeManager重启时数据保存在本地的目录 -->  
  <property> 
    <name>yarn.nodemanager.recovery.dir</name>  
    <value>/usr/hadoop/hadoop-2.9.0/data/rsnodemanager</value> 
  </property>  
  <!-- 配置NodeManager的RPC通讯端口 -->  
  <property> 
    <name>yarn.nodemanager.address</name>  
    <value>0.0.0.0:45454</value> 
  </property> 
</configuration>
```

值得一提的是：

ResourceManager Restart 使用的存储方式（实现类）

```markdown
# 保存在ZK当中
org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore
# 保存在HDFS中
org.apache.hadoop.yarn.server.resourcemanager.recovery.FileSystemRMStateStore
# 保存在本地
org.apache.hadoop.yarn.server.resourcemanager.recovery.LeveldbRMStateStore 
```

高可用 Hadoop 部分教程来源：

https://www.cnblogs.com/funyoung/p/9947105.html

### 启动 hadoop 高可用集群

#### 1. 启动所有节点的 JournalNode 

```
./hadoop-daemon.sh start journalnode
```

#### 2. 格式化第一个 NameNode 并启动

```
hadoop namenode -format
./hadoop-daemon.sh start namenode
```

#### 3. 第二个 NameNode 同步第一个 NameNode 的信息，并启动

```
hdfs namenode -bootstrapStandby
./hadoop-daemon.sh start namenode
```

#### 4. zookeeper 集群格式化

```
hdfs zkfc -formatZK
```

#### 5. 重启 hadoop

```
./stop-all.sh
./start-all.sh
```

至此，查看两个 namenode 节点的 50070 端口，即可访问