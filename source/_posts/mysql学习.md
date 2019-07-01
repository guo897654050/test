---
title: mysql学习
date: 2018-12-20 21:18:40
tags: mysql
categories: mysql
---

## <center>mysql学习</center>

### mysql安装

1.首先是安装mysql，我自己数在ubuntu服务器下面。步骤如下：

```
sudo apt-get install mysql-server
sudo apt-get install mysql-client
sudo apt-get install libmysqlclient-dev
```

安装完毕后，一般情况下都是输入命令

```
mysql -v
```
显示出版本即代表安装成功.mysql端口占用的3306端口，查看方法如下。

```
sudo netstat -anpt | grep mysql
```
<!--more -->
### mysql编辑配置文件

2.编辑位于/etc/mysql/mysql.conf.d/mysqld.cnf

```
sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
```

注释掉bind-address=127.0.0.1。这样远程也可以访问了。退出。

```
sudo service mysql restart
```
### 进入mysql的命令行

3.进入mysql服务

```
mysql -u root -p
```

会提示输入密码，输入即可。ok,但是我们最好还是新建一个用户,mysql不区分大小写，我一般用小写。

```
cereate user 'username'@'host' identified by 'password'
```

- username: 你创建的用户名
- host: 指定你从那个主机可以登录，本地用户可以设为localhost，如果想让用户从任意远程主机登录，可以设为通配符%。
- password: 该用户的登录密码

```
create user 'wind'@'%' identified by 'password'  #新建用户
drop user 'wind'@'%';  #删除用户
```
### 创建mysql新用户

4.此时我们创建完新的用户了，我们可以看一下当前的数据库。

```
show databases; #mysql语句均是以;作为结束
create testdb;  #随便创建一个名为testdb的数据库
```

此时我们是没有任务数据库的。参照易佰教程，我们可以导入数据。
加入我们sql数据文件存放于/home/guoxy/Desktop/mysql/yibai.sql

```
source /home/guoxy/Desktop/mysql/yibai.sql
```

便会创建名为yibaidb的数据库。

```
show databases;  #会显示我们的yibai.db
use yibaidb;  #会显示database change
```

我们想看看这个yibaidb的数据表是啥样的。
```

show tables;  #会显示所有的数据表的名字

```
假如我们想修改某个表的字段名字。
```
alter table 表名 旧字段名 新字段名 新数据类型;
```
### 为新建用户分配数据库

5.上述的操作均是在root用户下操作，为了安全我们应当为新建的wind用户分配一个数据库

```
grant all on yibaidb.* to 'wind'@'%';  #给权限
revoke all on yibaidb.* to 'wind'@'%'; #撤销权限
```

这表明把yibaidb的所有的表的权限都赋给了wind用户。

### 新建用户的登录

6.此时我们可以退出root用户登录wind用户

```
mysql -u wind -p 
```

输入密码进入，查看数据库及数据表

```
show databases;
use yibaidb;
show tables;
```

如果想查看表的字段，在show tables;之后会看到表名，

```
desc tablename;
```

便可以看到表的字段类型。    

### 自己如何建立数据表

7.自己新建table的方法:如新建person表和favourite_food表。

```
CREATE TABLE person
(person_id SMALLINT UNSIGNED,
fname VARCHAR(20),
lname VARCHAR(20),
gender ENUM('M','F'),
birth_data DATE,
street VARCHAR(20),
city VARCHAR(20),
country VARCHAR(20),
postal_code VARCHAR(20),
CONSTRAINT pk_person PRIMARY KEY (person_id)
);

CREATE TABLE favorite_food
(person_id SMALLINT UNSIGNED,
food VARCHAR(20),
CONSTRAINT pk_favorite_food PRIMARY KEY (person_id, food),
CONSTRAINT fk_favorite_food_person_id FOREIGN KEY (person_id)
REFERENCES person (person_id)
);
```
### mysql的查询语句的使用

8.mysql的查询，之前自己做过django的网站，用的是封装好sql的查询语言，不过sql的也不难理解，一般来说都是select 表的字段名 from 表 where 条件。便可以显示出来

```
select * from preson where name='libai';
```

从person表找出所有名字为libai的人。
两个表的查询,连接了employee表和department表，且employ表简写为e，department表简写为d

```
select e.emp_id, e.fname, e.lname, d.name from employee as e inner join department as d
 on e.dept_id=d.dept_id
```

有时候我们想把一些表的字段名提取出来单独存下来，方便以后的查询。

```
create view employee_vm as select emp_id, fname, lname, year(start_date) as start_year from employee
```

这时，

```
show tables;
```

会发现多了一个employee_vm的表，其中的数据便是我们选择的部分。    
用in操作符来代表多个或的连接。

```
select account_id, producr_cd, cust_id, acail_balance from accont where
product_cd in ('CHK','SAV','CD','MM')
```

等同于

```
select account_id, producr_cd, cust_id, acail_balance from accont where
product_cd='CHK' or product_cd='SAV' or producr_cd='CD' or product_cd='MM'
```

可以看出方便了很多。    
mysql也可以通过正则来查询，个人正则不是特别了解，只记住点简单的。此外mysql查询可以通过%和_来查询，前者代表通配符，即多个字符，而_只代表一个字符。例如

```
select lname from employee where lname like '_a%e%';
```

表明查找lname第二字符为a并且后面至少存在一个e的名字。具体的查询方法也看了一些，就不一一详细叙述了，后面继续学习。





































































