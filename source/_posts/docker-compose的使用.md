---
layout: posts
title: docker-compose的使用
date: 2019-04-29 17:50:06
tags: docker
categories: docker
---

## <center>记录docker-compose的使用</center>

1.使用docker-compose的目的: 我么一般把web和数据库进行分离，每个应用程序作为一个docker的镜像来使用，这也是docker官方推荐的做法。

2.docker-compose是官方的编排应用。官方有django的设置。今天你测试了以下自己的环境。记录一下。

3.首先我们使用的是django和postgresql数据库。然后需要拉取postgresql数据库。先看一下`docker-compose.yaml`文件。

```
version: '3'

services:
  db:
    image: postgres #依赖的镜像，若没有则会自动从dockerhub获取
    volumes:
      - ./postgres-data:/var/lib/postgresql/data #挂载数据库文件，因为如果不挂载，那么容易起一停止数据就没了。运行的数据都会同步到这里
  web:
    build: .  #这部分是值相当于docker build，会自动寻找Dockerfile，.代表当前目录。
    command: python manage.py runserver 0.0.0.0:8000  #启动命令。
    volumes:
      - .:/code  #也是挂载web的一些代码
    ports:
      - "8000:8000"  #端口映射
    depends_on:
      - db  #这里是依赖，那么就是先启动了db
```
<!-- more -->
4.由于使用的pqsql，所以我们在settings需要更改。
如下
```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'postgres',
        'USER': 'postgres',
        'HOST': 'db',
        'PORT': 5432,
    }
}
```
5. 这个时候已经可以启动了，并且使用的是pqsql，因为我需要导入一些测试数据。先运行这写代码,在包含yml的文件的目录下运行
```
docker-compose config ##检查yml文件格式
docker-compose config --service  ##显示service
docker-compose run web python manage.py migrate
docker-compose run web python manage.py loaddata test_auth.json
docker-compose run web python manage.py loaddata test_management.json
docker-comnpose run web python manage.py loaddata test_location.json
docker-compose up ##启动
```
6. 晚上加个班，加上一层nginx,Dockerfile如下
```
FROM nginx


#对外暴露端口
EXPOSE 8005

RUN rm /etc/nginx/conf.d/default.conf

ADD localhost.conf  /etc/nginx/conf.d/


```
localhost.conf如下

```
server {
    charset utf-8;
    listen 8005;
    server_name 127.0.0.1;
    
    error_log /var/nginx_error.log;
    access_log /var/nginx_access.log;

    location /static {
	alias /var/www/static/guacamole;
    }
    location / {
	proxy_set_header Host $host;
	proxy_pass http://web:8000;  ##注意这里写service web的名字，找了好久的错误
    }
}

```
最后修改docker-compose.yml如下
```
version: '3'

services:
  db:
    image: postgres
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
  web:
    build: .
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/code
    depends_on:
      - db
  nginx:
    build: ./nginx
    volumes:
      - /var/www/static/guacamole:/var/www/static/guacamole
    ports:
      - "8005:8005"
    depends_on:
      - web
```
最后执行 `docker-compose up`即可，访问127.0.01:8005便可以访问到服务。

7.等待套一层gunicorn
8.有个bug，有时候数据库最后启动导致服务启动不了，等待查看原因。































