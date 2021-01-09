---
title: 大数据分析平台架设（三）
date: 2020/11/10 14:13:06
categories:
- 教程
tags:
- 大数据
- hive
---
前置：http://blog.fat-otaku.top:4000/2020/11/08/BigData2020(2)/

详细版：http://blog.fat-otaku.top:4000/2020/11/05/BigData2020v1.0/
<!-- more -->
## 五、hive 部署

直接上**远程模式**配置，详细介绍看详细版相关部分

### 准备

```shell
# 创建目录
mkdir /usr/hive
# 解压缩
tar -zxvf /opt/soft/apache-hive-2.3.7-bin.tar.gz -C /usr/hive

# 配置更新环境变量
echo 'export HIVE_HOME=/usr/hive/apache-hive-2.3.7-bin' >> /etc/profile
echo 'export PATH=$PATH:$HIVE_HOME/bin' >> /etc/profile
source /etc/profile
```

### 解决依赖

拷贝**事先准备好**的 MySQL 驱动 **mysql-connector-java**

并复制某路径的 **jline-2.12.jar**

```shell
# 将 mysql 驱动加入依赖库目录
cp /opt/soft/mysql-connector-java-5.1.47-bin.jar /usr/hive/apache-hive-2.3.7-bin/lib
# 将 jline 加入 hadoop的 yarn 依赖库目录
cp /usr/hive/apache-hive-2.3.7-bin/lib/jline-2.12.jar /usr/hadoop/hadoop-2.7.7/share/hadoop/yarn/lib
```

### 修改 **hive-env.sh**

```
# Set HADOOP_HOME to point to a specific hadoop install directory
HADOOP_HOME=/usr/hadoop/hadoop-2.7.7

# Hive Configuration Directory can be controlled by:
export HIVE_CONF_DIR=/usr/hive/apache-hive-2.3.7-bin/conf

# Folder containing extra libraries required for hive compilation/execution can be controlled by:
export HIVE_AUX_JARS_PATH=/usr/hive/apache-hive-2.3.7-bin/lib
```

分发 hive 至**服务端节点** slave1

```
scp -r /usr/hive  slave1:/usr
```

### 配置 **hive-site.xml**

master（客户端） 的配置：

```xml
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

slave1（服务端）的配置：

```xml
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

slave2的作用是数据库节点，因此**不需要**安装hive环境。

### 修改 hive 日志信息

文件位置：

**$(HIVE_HOME)/conf/hive-log4j2.properties.template**

默认日志存放位置：

**/tmp/$(user.name)** 的 **hive.log**

### 安装 MySQL

使用**远程模式**，我们需要在在数据节点（slave2）上部署 Mysql

#### 准备

卸载 MariaDB

```shell
# 首先获取mariadb的包名
rpm -qa | grep mariadb
# 会得到mariadb-libs-x.x
rpm -e mariadb-libs-x.x --nodeps
```

使用 rpm -ivh 命令依次安装以下 mysql 包

1. `mysql-community-common`
2. `mysql-community-libs`
3. `mysql-community-libs-compat`
4. `mysql-community-client`
5. `mysql-community-server`

#### 配置 MySQL

**初始化 MySQL**

```shell
# 以 mysql 用户初始化数据库，并设置密码为空
/usr/sbin/mysqld --initialize-insecure --user=mysql
```

若使用 **mysqld --initialize-insecure** 来初始化数据库，则会在

**/var/log/mysqld.log** 下生成一个登录 MySQL 的随机密码。

**启动 MySQL**

```shell
# 启动 mysql 服务
systemctl start mysqld
# 开机自启
systemctl enable mysqld
```

**登录 MySQL**

```shell
# 默认密码为空的登录
mysql -u root
# 带密码的登录
mysql -u root -p 123456
# 更改密码
```

**更改密码**

```mysql
# 设置密码强度为低级
set global validate_password_policy=0
# 更改密码
alter user 'root'@'localhost' identified by '123456'
```

**更改远程登录权限**

```shell
use mysql;
update user set host='%' where host='localhost';
# 应用配置
flush pricileges;
# 查询结果，若 host 字段为 % 则修改成功
select user,host from user;
# 允许远程连接
grant all privileges on *.* to 'root'@'%' with grant option
flush pricileges;
```

## 六、启动 hive

### 初始化服务

由于使用 **远程模式** ，所以在 **服务端节点** 初始化数据库即可

```shell
schematool -dbType mysql -initSchema
```

若显示 **schemaTool completed** 则表示初始化成功

**启动 metastore 服务：**

**hive --service metastore** + & 可运行在后台

或者使用 screen -S 创建一个视窗也可以

### 启动 hive 服务

在 **客户端节点** 开启hive服务

```
hive
```

至此，hive 架设完毕

### hive 数据导入

将本地数据导入至对应表

`load data local inpath '/root/college/loan/csv' into table loan`

统计表数据，结果写入本地  `/root/college000/01`  中

```sql
insert overwrite local directory '/root/college000/01'
row format delimited fields terminated by '\t'
select count(*) from loan
```

直接读取 **csv** 文件

```
load data local inpath 'linux的csv文件路径' into table '表名';
```

