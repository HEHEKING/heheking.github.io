---
title: docker 部署 apache
date: 2020/11/23 16:09:01
categories:
- 教程
tags:
- docker
- apache
---

官方文档：https://hub.docker.com/_/httpd
<!-- more -->

部署 latest 版本：

```shell
docker run -dit --name apache -p 2333:80 -v /root/apache:/usr/local/apache2/htdocs/ httpd
```

部署 2.4 版本：

```shell
docker run -dit --name apache2 -p 2333:80 -v /root/apache:/usr/local/apache2/htdocs/ httpd:2.4
```

