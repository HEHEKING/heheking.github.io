---
title: yum 换源
date: 2020/11/13 1:54:27
categories:
- 教程
tags:
- yum
---
yum 换源
<!-- more -->

## 1. 备份本地源

防止出现意外还可以还原回来

```
cd /etc/yum.repos.d/
cp /CentOS-Base.repo /CentOS-Base-repo.bak
```

## 2. 准备 yum 源  文件

示例：下载阿里源

```
wget http://mirrors.aliyun.com/repo/Centos-7.repo
```

## 3. 清理旧包

```
yum clean all
```

## 4. 替换 yum 源  文件

```
mv Centos-7.repo CentOS-Base.repo
```

## 5. 更新 yum

```
yum makecache
yum update
```

