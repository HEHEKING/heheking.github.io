---
title: nginx 反向代理和跨域
date: 2020/12/2 22:40:44
categories:
- 教程
tags:
- nginx
- linux
---
不多BB，直接上配置文件 **nginx.conf**
<!-- more -->

```
http {
	server {
		listen 80;
		server_name localhost;
	   
		root  html;
		index index.html;
		
		location /api/ {
			proxy_pass http://localhost:8080;
			
			# 指定跨域方法，*表示所有
			add_header Access-Control-Allow-Methods *;
			
			# 预检命令的缓存，如果不缓存会发送两次请求
			add_header Access-Control-Max_Age 3600;
			
			# 带cookie请求需要加上该字段
			add_header Access-Control-Allow-Credentials true;
			
			# 表示允许这个域跨域调用（客户端发送请求的域名和端口）
			add_header Access-Control-Allow-Origin $http_origin;
			
			# 表示请求头的字段 动态获取
			add_header Access-Control-Allow-Headers $http_access_control_request_headers;
			
			# OPTIONS预检命令，预检命令通过时才发出请求
			if ($request_method = OPTIONS){
				return 200;
			}
		}
	   
		location /ofs/ {
			proxy_pass http://localhost:8080;
		}
	   
		location /admin/api/ {
			proxy_pass http://localhost:8080;
		}
	}
}
```

nginx 会监听请求，并根据配置文件进行转发。

上述配置文件中，将对 /api/ 路径及子路径的代理至 8080 端口，并代替接口布置了跨域。

/ofs/ 和 /admin/api/ 的路径则只进行代理。