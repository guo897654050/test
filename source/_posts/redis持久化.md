---
layout: posts
title: redis持久化
date: 2019-02-25 15:13:24
tags: redis
catrgories: redis
---

### 持久化选项
redis提供了两种不同的持久化方法来将数据存储到硬盘中。一种方法叫做快照(snapshotting)，他可以将某一时刻的所有数据写入硬盘。另一种叫做只追加文件(appedn-only file,AOF),他会在执行写命令时，将被执行的写命令复制到硬盘中，这两种持久化方法即可以同时使用也可以单独使用。    
将内存的数据存储到硬盘的主要原因是为了在之后重用数据，或是为了防止系统故障将数据备份到另一个远程位置

#### 快照持久化
创建快照的方法
- 客户端可以通过向redis发送BGSAVE命令来创建一个快照，对于支持BGSAVE命令的平台来说，redis会调用fork来创建一个子进程，然后子进程负责将快照写入硬盘，而父进程继续处理命令请求。
- 客户端还可以通过向redis发送SAVE命令来创建一个快照，接到SAVE命令的redis服务器在快照创建完毕之前不再响应任何其他请求。SAVE命令并不常用，通常只会在没有足够内存去执行BGSAVE的命令情况下，或者等待持久化操作完毕后也无所谓的情况下，才会使用SAVE.
- 如果用户设置了save的配置选项，比如`save 60 10000`，那么redis最近一次创建快照之后开始算起，当60s之后有10000次写入，这个条件被满足以后，redis会自动触发BGSAVE命令。如果用户设置了多个save配置，当任意一个save配置条件被满足，都会触发BGSAVE。
- 当redis通过SHUTDOWN命令接收到关闭服务器的请求时，或者接受到标准的TERM信号时，会执行一个SAVE命令，阻塞所有客户端，不在执行刻画段发送的任何命令。并在SAVE命令之后关闭服务器。
- 当一个redis服务器链接另一个redis服务器，并向对方发送SYNC命令来开始一次复制操作的时候，如果服务器目前没有执行BGSAVE操作，或者主服务器并非刚刚执行完BGSAVE操作，那么主服务器会执行BGSAVE。     
<!-- more -->
在只使用快照持久化来保存数据时，如果系统真的发生崩溃，那么用户将丢失最近一次生成快照之后的所有数据。因此快照持久化只适用于那些即使丢失一部分数据也不会造成问题的应用程序。


### AOF持久化
AOF持久化会将被执行的写命令写到AOF文件的末尾，以此来记录数据的变化。因此，redis只要从头到尾重新执行一次AOF文件包含的所有写命令，就可以恢复AOF文件所记录的数据集。appendfsync配置选项对AOF文件的同步频率的影响。
- always: 每个redis写命令都会同步写入硬盘，这样会严重降低redis的速度。
- everysec: 每秒执行一次同步，显式的将多个命令同步到硬盘
- no: 让操作系统决定何时同步



#### 重写/压缩AOF文件
假如选择everysec，由于redis会不断的执行写命令记录到AOF文件中，随着redis的不断运行，AOF文件会不断增大，甚至用完硬盘所有空间。为了解决此问题，用户可以向redis发送BGREWRITEAOF，这个命令会移除AOF文件的冗余来重写AOF文件。`同快照持久化一样，重写需要用到子进程，由于快照持久化存在内存占中和性能问题，在AOF持久化同样存在`，如果不加以控制，AOF的文件会比快照文件大好几倍。    
同快照持久化一样可以通设置save选项来自动执行BGSAVE一样，AOF持久化可以通过设置'auto-aof-rewrite-perventage'选项和`auto-aof-rewrite-min-size`选项来自动执行BGREWRITEAOF，例如，设置`auto-aof-rewrite-percentage 100`和`auto-aof-rewrite-min-size 64mb`，并且启用了AOF持久化，那么当AOF文件大于64mb，并且AOF文件的体积比上次重写之后体积增加了一倍，redis自动执行BGREWRITEAOF。    
无论使用AOF持久化还是快照持久化，将数据持久化到硬盘上是十分有必要的，除此外，用户还需对持久化的文件进行备份，这样才能避免数据的丢失。但随着负载量的上升，或者数据的完整性越来越重要，用户可能需要复制特性。

#### 复制(replication)
复制可以让其他服务器拥有一个不断更新的数据副本，从而使得拥有数据副本的服务器可以处理客户端发送的请求。关系数据库通常会使用一个主服务器(master)向多个从服务器(salve)发送更新，并使用从服务器开处理所有请求。redis同理。    
复制的过程如下:

<div style="width: 200px; margin: auto">![示例图片](/images/AOF-replication.bmp)</div>

从服务器的设置:用户可以通过配置选项SLAVEOF host port来讲一个redis服务器设置为服务器，也可以通过向正在运行的redis服务器发送SLAVEOF命令来将其设置为从服务器。
- 如果使用SALVEOF配置选项设置，那么redis启动时会载入当前可用的任何快照或者AOF文件，然后连接主服务器完成上图的复制过程。
- 如果使用SALVEOF命令，那么redis会立即尝试连接主服务器，并在连接成功后开始上图的复制过程。

当多个从服务器尝试链接同一个主服务器，有时候可重用已有的快照文件。
<div style="width: 200px; margin: auto">![示例图片](/images/AOF-replication.bmp)</div>

#### 检验硬盘写入
为了验证主服务器已经把数据发送至从服务器，用户需要再向主服务器写入一个唯一的虚构值(unique dummy value)，然后通过检查虚构值是否存在于从服务器来判断数据是否到达从服务器。但判断数据是否写入硬盘则相对困难，对于每秒同步一次的AOF文件，可通过等待一秒来确定数据是否写入硬盘。更节约时间的做法是检查INFO命令的输出结果中aof_pending_bio_fsync的属性是否为0，如果是的话，那么就表示服务器保存至硬盘。在向主服务器写入数据后，用户可以将二者的链接作为参数，然后执行检查。代码
```
import time
import uuid

##wait_for_sync
def wait_for_sync(mconn, sconn):
    identifier = str(uuid.uuid4()) #添加随机的令牌利用uuid生成
    #print(identifier)
    mconn.zadd('sync:wait', identifier, time.time())  #将令牌添加至主服务器
    
    while not sconn.info()['master_link_status'] != 'up': #如果有必要，等待从服务器完成同步
        time.sleep(.001)
        
    while not sconn.zscore('sync:wait', identifier): #等待从服务器接受数据更新
        time.sleep(.001)
        
    deadline = time.time() + 1.01   #最多等待1s
    while time.time() < deadline:
        if sconn.info()['aof_pending_bio_fsync'] == 0: #检查数据是否存入了硬盘，等于0则存入
            break
        time.sleep(.001)
    mconn.zrem('sync:wait', identifier)
    mconn.zremrangebyscore('sync:wait', 0, time.time()-900) #清理刚创建的新令牌以及之前可能留下的旧令牌,900s
    
    #zremrange的使用
    #     redis> ZRANGE salary 0 -1 WITHSCORES          # 显示有序集内所有成员及其 score 值
    # 1) "tom"
    # 2) "2000"
    # 3) "peter"
    # 4) "3500"
    # 5) "jack"
    # 6) "5000"

    # redis> ZREMRANGEBYSCORE salary 1500 3500      # 移除所有薪水在 1500 到 3500 内的员工
    # (integer) 2

    # redis> ZRANGE salary 0 -1 WITHSCORES          # 剩下的有序集成员
    # 1) "jack"
    # 2) "5000"
```
其中info命令提供了大量与redis服务器有关的信息，比如内存占用量，，客户端连接数，每个数据库包含的键的数量，上次创建快照文件之后执行的命令的数量等等。     
为了确保操作正确执行wait_for_sync()函数会首先确认从服务器连接上主服务器，然后检查自己添加到等待同步有序集合里面的值是否存在从服务器，在发现值已经存在于从服务器，函数会检查从服务器写入缓冲区的状态，并在1s之内，等待从服务器将缓冲区的所有数据存硬盘。孙然函数等待1s，但是大部分同步会在很短的时间内完成。最后确认数据保存到硬盘，函数会执行一些清理操作。



































