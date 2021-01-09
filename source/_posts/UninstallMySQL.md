---
title: CentOS 7 卸载 MySQL
date: 2020/12/1 0:47:01
categories:
- 教程
tags:
- linux
- mysql
---

最近遇到一些事，算是在这种问题上吃大亏了。

MySQL 因为流程化操作，很少会出现需要重装才能解决的问题。总的来说，在某些时候迅速的重置 MySQL 是非常有必要的。

「前有遗憾」
<!-- more -->

## 一、关闭 MySQL 服务

一般而言会出现以下情况：

```shell
# 通过 systemctl 停止 mysql 服务
systemctl stop mysqld
```

但在 mysql 因未知原因宕机时，这条指令经常会卡住，并不能有效关闭 mysql 的服务

```shell
# 查询所有进程，找到 mysql 的 PID
top
# 杀死进程
kill -9 [PID]
```

这时，MySQL 的服务已被关闭。

## 二、卸载 MySQL

MySQL 的安装，无非为 rpm 安装和 yum 安装

这里以 rpm 安装 mysql 5.7 为例

```shell
# 查询已安装的 mysql 包
rpm -qa | grep mysql
# 使用 rpm 命令卸载查询到的所有包
rpm -e [pkg-name] --nodeps
# 若想不使用 --nodeps 可以依次按下列顺序卸载
mysql-community-server
mysql-community-client
mysql-community-libs-compat
mysql-community-libs
mysql-community-common
# 当然你也可以使用 yum
yum remove mysql-*
```

一般情况下，我们安装的是以上五个包，特殊情况已环境为准。

若使用 yum 直接安装，则直接 yum remove mysql[:tag] 即可。但总是会卸载不完整，建议检查是否存在残留包。

## 三、重新安装 MySQL

进行安装即可

## 四、重置数据

重新安装 MySQL，我们需要删除相关的数据

MySQL 的一些配置、数据的路径

```
配置文件：/etc/my.cnf
日志文件：/var/log/mysqld.log
```

目录：使用 **find** 命令查找

```
find / -name mysql
```

得到以下目录

```shell
/var/lib/mysql or /usr/local/var/mysql
/usr/share/mysql
# 删除
rm -rf [path]
```

至此，MySQL 应该可以重新安装

## 五、常见问题

### 1. 初始化 MySQL 提示 "Could not open file '../mysqld.log'"

权限不足，无法保存日志信息。

```
sudo chown -R mysql:mysql /usr/local/mysql
```

### 2. [Warning] TIMESTAMP with implicit DEFAULT value is deprecated

不碍事，根据日志输入 **--explicit_defaults_for_timestamp** 或 配置 **my.ini** 即可

### 3. --initialize specified but the data directory has files in it

注意查看 **my.ini** 下的 datadir 目录，保证为空

说明 MySQL 数据目录未删除。

```shell
rm -rf /usr/local/var/mysql
```

