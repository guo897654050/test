---
title: 记tomcat的部署与apache的反向代理
date: 2018-12-27 18:06:39
tags: 服务器
categories: 服务器
---

### tomcat的部署

首先，先说一下tomcat的部署，tomcat的部署的文件位于/conf/server.xml中，其中默认的端口为8080，一般来说http访问都是80端口，那么只需要把8080端口改成80端口即可。    
其次，更改为https部署，首选我们可以去阿里云申请免费的的ssl证书，那么在server.xml中配置
```
<Connector port="443"
    protocol="org.apache.coyote.http11.Http11Protocol"
    SSLEnabled="true"
    scheme="https"
    secure="true"
    keystoreFile="/home/guoxy/product-sites/apache-tomcat-7.0.88/conf/cert/1616181_xuyunrui.cn.pfx"
    keystoreType="PKCS12"
    keystorePass="EClj60j3"
    clientAuth="false"
    SSLProtocol="TLSv1+TLSv1.1+TLSv1.2"
    ciphers="TLS_RSA_WITH_AES_128_CBC_SHA,TLS_RSA_WITH_AES_256_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256,TLS_RSA_WITH_AES_128_CBC_SHA256,TLS_RSA_WITH_AES_256_CBC_SHA256"/>

```
当着样部署好以后，那么就可以利用https访问了。

<!-- more -->
### 反向代理tomcat的端口

先说下为什么要反向代理。首先，原来的服务起的80端口apache已经使用了，供其他的网页使用。此时如果tomcat也是用80端口，那么就会发现端口已经被占用，无法进行绑定。所以google了一下，找到的方法是利用apache来代理tomcat的端口即可。例如假如tomcat运行在8080端口，那么我们在apache下新建一个.conf文件来反代xx.com:8080即可。    
说一下具体的操作:

- 首先网上找的一堆教程都是修改httpd.conf，但是在ubuntu下的apache2下，根本没有这个文件。我们的网站的配置文件均位于
```
/etc/apache2/site-available/
```
我们在此新建tomcat.conf.配置如下
```
<VirtualHost *:80>
        # The ServerName directive sets the request scheme, hostname and port that
        # the server uses to identify itself. This is used when creating
        # redirection URLs. In the context of virtual hosts, the ServerName
        # specifies what hostname must appear in the request's Host: header to
        # match this virtual host. For the default virtual host (this file) this
        # value is not decisive as it is used as a last resort host regardless.
        # However, you must set it for any further virtual host explicitly.
        #ServerName www.example.com

        #ServerAdmin webmaster@localhost
        #DocumentRoot /var/www/html
        #配置站点的域名
        ServerName beta.steering.ai
        #配置站点的管理员信息
        #ServerAdmin admin@xuyunrui.com

        #off表示开启反向代理，on表示开启正向代理
        ProxyRequests Off
        ProxyMaxForwards 100
        ProxyPreserveHost On
        #这里表示要将现在这个虚拟主机跳转到本机的8080端口
        ProxyPass / http://127.0.0.1:8080/
        ProxyPassReverse / http://127.0.0.1:8080/
        <Proxy *>
            Order Deny,Allow
            Allow from all
        </Proxy>
        # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
        # error, crit, alert, emerg.
        # It is also possible to configure the loglevel for particular
        # modules, e.g.
        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        # For most configuration files from conf-available/, which are
        # enabled or disabled at a global level, it is possible to
        # include a line for only one particular virtual host. For example the
        # following line enables the CGI configuration for this host only
        # after it has been globally disabled with "a2disconf".
        #Include conf-available/serve-cgi-bin.conf
</VirtualHost>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet

```
上面的信息都已经注释了含义，首先要开一个80端口的虚拟主机。

- servername即为我们的域名.
- ProxyRequests Off #off表示开启反向代理，on表示开启正向代理
-  ProxyPass / http://127.0.0.1:8080/ && ProxyPassReverse / http://127.0.0.1:8080/是指要代理的网址以及反代的资源网址。
-  后面的就是常规配置了。
-  但是一般来说我们为了安全性，都是把网站配置成https的访问形式，所以需要重新配置tomcat.conf文件
```
<VirtualHost *:443>
        # The ServerName directive sets the request scheme, hostname and port that
        # the server uses to identify itself. This is used when creating
        # redirection URLs. In the context of virtual hosts, the ServerName
        # specifies what hostname must appear in the request's Host: header to
        # match this virtual host. For the default virtual host (this file) this
        # value is not decisive as it is used as a last resort host regardless.
        # However, you must set it for any further virtual host explicitly.
        #ServerName www.example.com

        #ServerAdmin webmaster@localhost
        #DocumentRoot /var/www/html
        #配置站点的域名
        ServerName beta.steering.ai
        #配置站点的管理员信息
        #ServerAdmin admin@xuyunrui.com
        #ssl 这里写ssl证书的位置文件
        SSLEngine on
        SSLCertificateFile "/etc/steering.ai/STAR_steering_ai.crt"
        SSLCertificateKeyFile "/etc/steering.ai/STAR_steering_ai.key"

        #off表示开启反向代理，on表示开启正向代理
        ProxyRequests Off
        ProxyMaxForwards 100
        ProxyPreserveHost On
        #这里表示要将现在这个虚拟主机跳转到本机的8080端口
        ProxyPass / http://127.0.0.1:8080/
        ProxyPassReverse / http://127.0.0.1:8080/
        <Proxy *>
            Order Deny,Allow
            Allow from all
        </Proxy>
        # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
        # error, crit, alert, emerg.
        # It is also possible to configure the loglevel for particular
        # modules, e.g.
        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        # For most configuration files from conf-available/, which are
        # enabled or disabled at a global level, it is possible to
        # include a line for only one particular virtual host. For example the
        # following line enables the CGI configuration for this host only
        # after it has been globally disabled with "a2disconf".
        #Include conf-available/serve-cgi-bin.conf
</VirtualHost>
```
也就是加上了ssl的证书，所以配置反代还是代理原来的8080端口，而不是配置tomcat的8443端口进行https访问。

#### docker的部署

这里我把tomcat的文件部署到docker中，Dockerfile如下
```
FROM ubuntu:16.04
COPY ["jdk-6u45-linux-x64.bin", "."]
RUN  apt-get update && apt-get install -y vim
RUN  cp jdk-6u45-linux-x64.bin /usr/local
RUN  cd /usr/local && chmod +x jdk-6u45-linux-x64.bin
RUN  cd /usr/local && ./jdk-6u45-linux-x64.bin
ENV JAVA_HOME=/usr/local/jdk1.6.0_45
ENV CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV PATH=$PATH:$JAVA_HOME/bin
RUN cd /home && mkdir -p /guoxy/product-sites/apache-tomcat-7.0.88
COPY apache-tomcat-7.0.88 /home/guoxy/product-sites/apache-tomcat-7.0.88
WORKDIR /home/guoxy/product-sites/apache-tomcat-7.0.88/bin
CMD ./start.sh
```
相应的文件都有，这里主要是java的jdk和tomcat的文件。执行
```
docker build -t tomcat:v1 .
```
这时候吧镜像上传到自己dockerhub，需要给自己的镜像打上tag
```
docker login #先登录
docker tag tomcat:company guoxy/tomcat:company  #其中guoxy为自己的仓库名
docker push guoxy/tomcat:company
```
此时镜像上传到自己的dockerhub。运行映射一下端口
```
docker run -it -p 8080:8080 tomcat:company /bin/bash
```
按下Ctrl+P+Q使容器在后台运行。再次进入
```
docker exec -it tomcat:company /bin/bash 进入如果没启动在bin目录下./startup.sh
```