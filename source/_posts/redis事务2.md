---
layout: posts
title: redis事务2
date: 2019-02-25 22:39:06
tags: redis
categories: redis
---
### redis事务
为了保证数据的正确性，确认一点:在多个客户端同时处理相同的数据时候，不谨慎的操作会导致数据出错。下面介绍使用redis事务来防止数据出错的方法，以及事务来提升性能的方法。    
redis的事务和传统关系数据库事务并不同，在传统关系数据库中，用户先向数据库服务器发送BEGIN，然后执行各个相互一致(consistent)的写操作和读操作，最后用户选择COMMIT来确认之前所做的更改，或者发送ROLLBACK来放弃修改。    
在redis也可一处理一连串相互一致的读操作和写操作。redis的事务以MULTI开始，之后跟着用户传入的多个命令，最后以EXEC为结束。但是这种简单的操作在调用EXEC命令之前不会执行任何实际的操作，所以用户没办法根据读到的数据做决定。    
以例子来说明。下图展示了游戏中用于表示用户信息和用户包裹的结构，用户信息存储在散列用户报国使用一个集合来表示。
<!-- more -->
<div style="width: 200px; margin: auto">![示例图片](/images/redis-transcation1.bmp)</div>

买卖市场的需求:一个用户(卖家)可以将商品放到市场上进行销售，当一个用户(买家)购买时，卖家会收到钱。为了将被销售商品的全部信息存储到市场里面，我们将会把商品id和卖家id拼接在一起，并将拼接结果作为成员存储到市场的有序集合如下图所示

<div style="width: 200px; margin: auto">![示例图片](/images/redis-transcation2.bmp)</div>

#### 将商品放到市场上销售

为了将商品放到市场上销售，程序出了用到MULTI和EXEC命令之外，需要配合使用WATCH命令，有时甚至会用到UNWATCH和DISCARD(丢弃)。在用户使用WATCH对键进行监听后，直到用户执行EXEC这段时间里，如果有其他的客户端抢险对任何被监视的键进行了替换，更新，删除等操作，那么当用户尝试执行EXEC命令的时候。事务失败并返回一个错误(之后用户可以选择重试事务或者放弃事务)。通过使用WATCH，MULTI/EXEC，UNWATCH/DISCARD等命令，程序可以在执行某些重要操作的时候。通过确保自己正在使用的数据没有发生变化来避免出错。    
什么是DISCARD？ unwatch命令可以再WATCH命令之后，在MULTI命令执行之前进行重置(reset)。同样DISCARD命令也可以在MULTI命令执行之后，在EXEC执行之前对连接进行重置。这也意味着，用户在使用WATCH监视一个键或者多个键，接着使用MULTI开始一个新的事务，并将多个命令入队到事务队列之后，仍然可以通过发送DISCARD命令来取消WATCH命令并清空所有已经已经入队的命令。

在讲一件商品放到市场上机型销售的时候，程序需要将被销售的商品添加到记录市场正在销售商品的有序集合里面，并且再添加操作执行的过程中，监视卖家的包裹以确保被销售的商品确实存在u卖家的包裹当中。
```
def list_item(conn, itemid, sellerid, pric):
    inventory = "inventory:%s"%sellerid #包裹的集合
    item = "%s.%s"%(itemid, sellerid) 
    end = time.time() + 5
    pipe = conn.pipeline()
    while time.time() < end:
        try:
            pipe.watch(inventory)  #监视用户的包裹
            if not pipe.sismember(inventory, itemid) #检查用户是否仍然持有被销售的产品
                pipe.unwatch(inventory)  #没有的话，取消监视并返回空
                return None
            pipe.multi()
            pipe.zadd("market:", item, price)  #把被销售的产品添加到市场中
            pipe.srem("inventory", itemid)
            pipe.execute()
            return None
        except redis.exceptions.WatchError:
            pass
    return None
```
list_item()函数，首先执行一些初始化，然后对卖家的包裹进行监视，验证卖家的商品是否处于卖家的包裹中，如果是，函数就会把商品添加到市场，并从卖家包裹移除这个商品，，在使用watch对包裹进行监视的过程中，如果包裹被更新或修改，程序受到错误并进行重试。

#### 购买商品

首先对市场以及卖家的信息进行监控，获取买家的钱数以及商品的售价，并检查钱够不够。不够，取消事务。够的话，先把买家的钱转给卖家，然后将售出的商品移动至买家的包裹，并将该商品从市场中移除。当买家的个人信息或者市场的商品信息变动，返回错误，最大重试时间10s
```
def purchase_item(conn, buyerid, itemid, sellerid, lprice):
    buyer = "users:%s" %buyerid
    seller = "users:%s" %sellerid
    item = "%s.%s" %(itemid, sellerid)
    inventory = "inventory:%s" %buyerid
    end = time.time() + 10
    pipe = conn.pipeline()
    
    while time.time() < end + 10:
        try:
            pipe.watch("market:", buyer)　　#监视两个，一个market,一个购买者的信息
            
            price = pipe.zscore("market:", item)
            funds = int(pipe.hget(buyer, "funds"))
            if price != lprice or price > funds:  #检查买家购买的商品的价格是否发生变化及用户是否有钱购买
                pipe.unwatch()
                return None
            pipe.multi()
            pipe.hincrby(seller, "funds", int(price))
            pipre.hincrby(buyer, "funds", int(-price))
            pipe.sadd(inventory, itemid)
            pipe.zrem("market:", item)
            pipe.excute()
            reutrn True
            
        except redis.exceptions.WatchError:
            pass
    return False
    
```

补充:为什么ｒｅｄｉｓ没有实现典型的加锁功能？　　　　
在访问以写入为目的数据的时候，关系库会对被访问的数据进行加锁，直到事务被提交或者回滚。如果其他用户尝试对被加锁的数据进行写入，该客户端将会被阻塞。直到第一个事务执行完，持有锁的可花旦越慢，等待时间越长。　　　　
为了减少等待时间，redis并不会在watch的时候对数据进行加锁，知乎自爱数据被其他客户端抢先修改的情况下通知执行了watch的客户端，这种做法成为乐观锁，而关系数据库的称为悲观锁。乐观锁在实际中也有效，因为客户端永远不必花时间去等待第一个已经取得锁的用户，只需要在自己的事务中失败并且重试即可。


### 非事务性流水线操作

事务性质-被，multi和exec包裹的命令在执行时候不会被其他客户端打扰。    
在需要执行大量命令的情况下，及时命令不需要放在事务中执行，单位了一次发送所有的命令来减少通信次数，用户也可能吧命令包裹在multi和exec之间。但是multi和exec也会消耗资源，可能导致其他命令延迟执行。

pipe = conn.pipeline()

如果用户在执行pipeline()传入True作为参数，或者不传，客户端将使用multi和exec包裹命令。如果传入Flase作为参数，同样客户端会想执行事务那样收集用户所需的所有命令，只是不再用multi和exec包裹命令。如果用户需要向redis一次发送多个命令，且对这些命令来说，一个命令的执行结果不影响另一个命令的输入，且这些命令需要以事务方式执行，可通过piepeline()传入fasle来提升redis的性能。

之前的例子update_token函数，负责记录用户最近浏览的商品记忆访问过的页面，并跟新用户的cookie，每次执行都会调用2-5个redis命令，是的客户端和redis产生2-5次通信往返。
```
def update_token(conn, token, user, item=None):
    #获取时间戳
    timestamp = time.time()
    #维持令牌与已经登录的用户的映射
    conn.hset('login:', token, user)
    #记录令牌最后一次出现的时间
    conn.zadd('recent:', token, timestamp)
    if item:
        #记录用户浏览过的商品
        conn.zadd('viewed:' + token, item, timestamp)
        #移除旧的记录，只保留过去的25个商品,移除0 - -26区间的成员，0和-26都是闭区间
        conn.zremrangebyrank('viewed:' + token, 0 , -26)
        #新增,zincrby key -5 member 让member的score减去5，每次减去-1，若key不存在，相当于zadd key increament member
        #浏览单最多的商品将排列在索引0的位置上
        conn.zincrby('viewed:', item, -1)
```
流水线
```
def update_token_pipeline(conn, toekn, user, item=None):
    timestamp = time.time()
    pipe = conn.pipeline(False)
    pipe.hset('login:', token, user)
    pipe.zadd('recent', token, timestamp)
    if item:
        pipe.zadd('viewed:' + token, item, timestamp)
        pipe.zremrangebyrank('viewed:' + token, 0， -26)
        pipe.zincr('viewed:', item, -1)
    pipe.execute()
```

减少了通信往返次数，提升了redis性能，

























































































