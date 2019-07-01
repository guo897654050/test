---
layout: posts
title: Docker CI CD
date: 2019-05-15 11:30:07
tags: docker
categories: docker
---
### 记录docker的持续集成(CI)和自动部署(CD)


#### docker+travis实现持续集成(CI)

1.在github新建仓库，加入一个`Dockerfile`，内容自定义。

2.在travis中添加这个仓库，登录travis-ci.com，然后登录github选中仓库即可。

3.使用`tavis`需要编辑一个`.travis.yml`，内容如下
```
language: bash
dist: xenial
services:
- docker
before_script:
- echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
script:
- echo "test code"
after_success:
- docker build -t guoxy/alpine .
- docker push guoxy/alpine

```
<!-- more -->
这里只是很简单的测试文件，由于我们需要使用`docker`，所以在`serives`中天机`docker`，`before_script`是在运行之前先登录`dockerhub`的账号。    
其中`$DOCKER_PASSWORD`和`DOCKER_USERNAME`我们可以在travis选中的这个仓库的settings中添加作为环境变量。    
我们在持续集成中，一般把需要运行测试的代码添加到script中,以django为例的话，一般执行`python manage.py migrate`和`python manage.py test`来运行迁移和执行我们的单元测试代码。在运行成功后即`after_success`我们重新build了这个镜像，然后推送到我们的`dockerhub`的仓库。这里我们实现了持续集成，相当于我们每次提交了代码，完成了单元测试，然后travis会自动把我们的代码重新打包成镜像并且推送的我们的`dockerhub`。然后我们需要做的事是从服务器自动拉取更新过的镜像，然后运行，完成自动部署的部分。

#### docker+travis实现自动部署

1. 首先我们要在本机上安装ruby的gem，利用gem安装travis
```
sudo apt-get install ruby
sudo apt-get install ruby-dev
sudo gem install travis
```
2. 此时可以进行登录，这里卡了一个问题，由于travis-ci有俩网站，所以我们一定要看自己是在`travis-ci.org`结尾的网站还是`travis-ci.com`,非常重要，否则后面会出bug，我自己用的就是`travis-ci.com`，一般也都是这个。
```
travis login --com  

然后输入你的github的账号和密码
```

3.然后我们需要把我们服务器的私钥传到travis上才能让其远程登录服务器。  
```
travis encrypt-file ~/.ssh/id_rsa --pro --com --add  #com网站后缀,add会把加密的私钥作为环境变量传送到travis的setting中可以查看。
```
这一步之后我们会发现有了我们之前加入的四个环境变量。`.travis.yml`的内容如下
```
language: bash
dist: xenial
services:
- docker
before_script:
- echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
script:
- echo "test code"
after_success:
- docker build -t guoxy/alpine .
- docker push guoxy/alpine
- ssh guoxiaoyang@101.201.68.24 -o StrictHostKeyChecking=no "sh ~/travis-remote/test.sh"
env:
  global:
    secure: jjMhZ/k9Yv+XjeUQxjC1lE2cOPJ8MTgmckJ8M7YCWcIn2u2lb+H+n/2CaTQUjpL3K9Q/OlePKQjioL1Qh2FGxi7x+8p2HpTVXxamrPSXF3DCkNBiWWRcvA8rBrTLztmap45qMyhNyNGxjiOOvS6PnNRjE29BJZd3Lmyu06Y7s/MiwGGrc12frs8m/Z/LahNg+oJeeZQilw795K6/X5gen2aRWaAlsk6c2FKbNU3Gh26ifMwdlgAYZ51Z7W2b610tT9DkV1r/nztLGd+1Y6PSSJUTd9q5R/Nm7xlHPyj9Pwhhyz6yiQ8Px3irKDh3iFBpzH2ylDoUr/9wPIwMsWg2Tb2DoF5LOjvcRVWRoBdOJAiQIdxXN2euPLkJBA/FEv1oPngBOKAljJn+BdKBq5PVGCdXBt8BAdCr8tjvlgT1YJdSFuz9wYZLyX9+WKUbDt5H4QwkMZ0Ro/xFx6u3yPnBf5bCkrgJzkpmLgakckFaU/uHEdp11p+SAQ35n/FVImnZgz9GiHWA6g7lTTvLt7U//KI+ZRT8n39fSDiS378x8ORqqnFcdAMzZAYWr26uVYTS/d6DpHWZlqLrIG0UPhy6+HRZa3U4OfYW4toe1u1SHWn9JuAA9ejFzvn3M4T/PDOcTEx/NsMIfzjsjfabWiPUoQ7Di8Hd6jYBxm+e9gBtYIM=
addons:
- ssh_known_hosts: 101.201.68.24
before_install:
- openssl aes-256-cbc -K $encrypted_78105f82ade5_key -iv $encrypted_78105f82ade5_iv
  -in id_rsa.enc -out ~/.ssh/id_rsa -d
- chmod 600 ~/.ssh/id_rsa

```
因为 travis 第一次登录远程服务器会出现 SSH 主机验证，这边会有一个主机信任问题。官方给出的方案是添加 addons 配置：
```
addons:
- ssh_known_hosts: 服务器ip
```
ssh那一步使我们登录远程服务器，并且执行相应的脚本。`-o`代表第一次舞曲输入确认的yes选项。

关于ssh远程免密码登录记录如下:
我们本地先生成一对秘钥，默认存放于`~/.ssh/`，我们把公钥上传至服务器
```
ssh-copy-id -i ~/.ssh/id_rsa.pub username@ip
然后在服务器端会生成authorized_keys的文件
```
然后登录即可
```
ssh username@ip
```
自己的自动部署的脚本名为`test.sh`，内容如下
```
#!/bin/bash
echo "Hello World"
cat ./.secret | docker login --username "guoxy" --password-stdin
docker stop deploy-test
docker rm $(docker ps -q)
docker container rm deploy-test
docker pull guoxy/ubuntu-with-vi:v1
docker run -it -d --name deploy-test guoxy/ubuntu-with-vi:v1

```
以上都是用测试的代码，不过也算完整的跑了一遍CI,CD,后面添加自己的web代码，再跑一遍，看看是否有问题。








































