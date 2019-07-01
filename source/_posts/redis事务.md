---
layout: posts
title: redis事务
date: 2019-02-25 09:58:14
tags: redis
categories: redis
---

### 什么是redis的基本事务
redis的基本事务(basic transaction)需要用到MULTI和EXEC命令，这种事务可以让一个客户端在不被其他客户端打断的情况下执行多个命令。和关系数据库那种可以在执行的过程中进行回滚(rollback)的事务不同，在redis里面，被MULTI和EXEC命令保卫的所有命令会一个接一个的执行，知道所有的命令执行完毕。当一个事务执行完毕后，才会处理其他客户端的命令。    
在redis里面执行事务，我们首先需要执行MULTI命令，然后输入我们想要在事务里面执行的命令，然后再执行EXEC命令。大概redis从一个客户端那里接收到MULTI命令，redis会将这个客户端发送的所有命令放到一个队列里面，直到这个客户端发送EXEC命令为止，然后redis就会在不被打断的情况下，一个接一个的执行存储在队列里面的命令。从语义上说，redis事务在pytho客户端上是以流水线(pipeline)实现的；对连接对象调用pipeline()方法创建一个事务，在这个事务正确使用的情况下，会用MULTI和EXEC包裹起一连串的多个命令。另外，为了减少redis与客户端之间的通信往返次数，执行多个命令时候的性能，python的redis客户端会存储起事务包含的多个命令，然后在事务执行时一次性将所有命令发送给redis。    
<!--more -->
```
conn = redis.Redis()

def notrans():
    print(conn.incr('notrans:')) #d对notrans计数器执行自增操作，并打印操作的执行结果
    time.sleep(0.1)
    conn.incr('notrans', -1)

if 1:
    for i in range(3):
        threading.Thread(target=notrans).start()
    time.sleep(5)
>>>1
>>>2
>>>3
    
##由于没有使用事务，所以三个线程在没有执行自减操作之前，对counter计数执行自增操作。如果我们需要在不受其他命令
##干扰的情况下，对计数器执行自增自减操作、利用事务

def trans():
    pipeline = conn.pipeline()  #创建一个事务型(transactional)流水线对象
    pipeline.incr('trans:')  #吧针对'trans:' 计数器的自增操作放入队列
    time.sleep(.1)
    pipeline.incr('trans:', -1)  #把针对'trans:'计数器的自减操作放入队列
    print(pipeline.execute()[0]) #执行事务包含的命令并打印自增的操作结果

if 1:
    for i in range(3):
        threading.Thread(target=trans).start()
    time.sleep(.5)
>>>1
>>>1
>>>1

## 尽管自增自减操作有一短延时，但通过使用事务，各个线程都可以在不被打断的情况下执行各自队列里面的命令
```

### 键的过期时间

在使用redis存储数据的时候，有些数据可能在某个时间点之后就不在使用了，用户可以显式的使用del删除，也可以通过redus的过期时间(expiration)特性来让一个键在给定的实现(timeout)之后自动删除。

- persist: persist key-name,移除键的过期时间
- ttl: ttl key-name,返回距离给定键的过期时间还有多少秒
- expire: expire key-name seconds,让key在多少seconds之后过期
- expireat: expireat keyname timestamp,将给定的键的过期时间设置为给定的unix时间戳
- pttl: pttl key-name,返回距离给定键的过期时间还有多少毫秒
- pexpire: pexpire key-name milliseconds,让键key-name在milliseconds毫秒之后过期
- pexpireat: pexpireat key-name timestamp-milliseconds,讲一个毫秒级的unix时间戳设置为给定键的过期时间
```
>>> conn.set('key', 'value')                    # 设置一个简单的字符串值，作为过期时间的设置对象。
True                                            #
>>> conn.get('key')                             #
'value'                                         #
>>> conn.expire('key', 2)                       # 如果我们为键设置了过期时间，那么当键过期后，
True                                            # 我们再尝试去获取键时，会发现键已经被删除了。
>>> time.sleep(2)                               #
>>> conn.get('key')                             #
>>> conn.set('key', 'value2')
True
>>> conn.expire('key', 100); conn.ttl('key')    # 还可以很容易地查到键距离过期时间还有多久。
True                                            #
100     
```












































































