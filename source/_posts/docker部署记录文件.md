---
layout: posts
title: docker部署记录文件
date: 2019-04-15 23:21:05
tags: docker
categories: docker
---
###<center>dokcer部署</center>

1. 首先把需要映射的文件放到某个固定文件夹下，如我的目录在`/opt/nginx-powerpred/`下面分别有四个文件夹,`/conf/nginx.conf`,`/log/access.log和/log/error.log`,`/sites-enabled/powerpred.cc`用来配置nginx,`/www/powerpred/*`为网页文件。
2. 其中powerpred的内容如下:
```
server {
    charset utf-8;
    listen 8000;
    server_name localhost;
    
    location /static {
	alias /var/www/static;
	}
    location / {
	proxy_set_header Host $host;
	proxy_pass http://unix:/tmp/windystreet.cn.socket;

	}
}
```
这里监听8000端口，是因为后面我们会把docker的8000端口映射到宿主机的80端口，那么在docker内监听的8000端口。
3.映射的命令，目录位于`/opt/nginx-powerpred/`,
```
 docker run -it -p 80:8000 
 -v $PWD/conf/nginx.conf:/ect/nginx/nginx.conf 
 -v $PWD/log/access.log:/var/log/nginx/access.log 
 -v $PWD/log/error.log:/var/log/nginx/error.log 
 -v $PWD/sites-enabled/powerpred.cc:/etc/nginx/sites-enabled/powerpred.cc 
 -v $PWD/www/powerpred:/opt/www/powerpred/ 15c /bin/bash

```
4.运行完上述命令，进入到容器内部，可以先`python3 manage.py collectstatic`收集静态文件，然后`service nginx start`,此时nginx启动了，再使用`gunicorn --bind unix:/tmp/windystreet.cn.socket ppower.wsgi:application`运行即可。
