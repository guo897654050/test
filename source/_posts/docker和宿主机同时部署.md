---
layout: posts
title: docker和宿主机同时部署
date: 2019-04-16 21:36:25
tags: docker
categories: docker
---
今天完成的是在docker中部署一个服务，然后宿主机也部署一个服务。将二者回顾一下。

问题:使用docker部署的时候，不需要宿主机的nginx来工作，但是当有俩服务，如果docker还用80端口映射，那么宿主机的web服务的nginx则无法工作，那么此时我们需要映射到其他端口，再用宿主的nginx来代理反代docker的服务。

1.在docker中部署一个nginx服务，上文已经介绍过了，在docker中我们可以像宿主机一样来部署nginx，在`/etc/nginx/sites-enabled/`下新建一个文件，命名为powerped.cc，我们监听的是内容就和上文一样，然后需要映射为81:8000，把docker的8000端口映射到宿主机的81端口，proxy_pass在我们测试时，可以填写127.0.0.1:81,然后回到网页的目录执行`python3 manage.py runserver 127.0.0.1:81`，如果网页可以访问了，说明可以，我们再使用gunicorn启动，`gunicorn --bind unix:/tmp/windystreet.cn.socket ppower.wsgi:application`，同时powerpred.cc的proxy_pass的地址也改为`http://unix:/tmp/windystreet.cn.socket;`,保证一致。
<!-- more -->

2.在宿主机部署另一个web服务，此时我们把映射的docker的81的端口使用nginx代理，同时也代理另一个web服务叫guacamole(web名字而已)，在宿主机的`/etc/nginx/sites-enabled/`新建guacamole，同时新建powrepred，二者均监听80端口。powerpred即需要反代docker的服务，内容如下
```
server {
	charset utf-8;
	listen 80;
	server_name windystreet.cn;
	
	location / {
		proxy_set_header Host $host;
		proxy_pass http://127.0.0.1:81;
	}
}
```
由于我们映射到宿主机的81端口，反代地址为127.0.0.1:81,且静态文件已经由docker的nginx代理完毕。    
guacamole的内容如下：
```
server {
        charset utf-8;
        listen 80;
        server_name common.windystreet.cn;

        location /static {
                alias /var/www/static/guacamole;
        }
        location / {
                proxy_set_header Host $host;
                proxy_pass http://unix:/tmp/common.windystreet.cn.socket;
        }
}

```
这里依然监听80端口，但是代理的位置不同。
`service nginx start`
我们切换到gucamole的目录下`gunicorn --bind unix:/tmp/common.windystreet.cn.socket guacamole.wsgi:application`启动，
然后分别访问'windystreet.cn'和'common.windystreet.cn'会发现这是两个不同的服务。














