---
title: redis
date: 2019-01-04 11:24:29
tags: redis
catrgories: redis
---

#### 字符串

在redis中，字符串可以存储三种类型的值    
- 字节串(byte string)
- 整数
- 浮点数

简单的自增命令，自减命令    
- incr: 用法incr key-name，将键存储的值加1    
- decr: 用法decr key-name，将键存储的值减1    
- incrby: 用法 incrby key-name amount，将键对应的值加上amount    
- decrby: 用法 decrby key-name amount,将键对应的值减去amount    
- incrbyfloat: 用法incrbyfloat ket-name amount，将键存储的值加上浮点数amount，这个命令在redis2.6以上可用。    

例子，在python解释器运行:
```
>>> conn = redis.Redis()
>>> conn.get('key')  #尝试获取一个不存在的键值，得到None，终端不会显示
>>> conn.incr('key')  #可对不存在的键来进行自增操作
1
>>> conn.incr('key', 15)
16
>>> conn.decr('key', 5)
11
>>> conn.get('key')  #尝试获取一个键时，命令以字符串格式返回被存储的整数
11
>>> conn.set('key', '20')
True
>>> conn.incr('key', 5)
25
```
<!-- more -->
其他命令:   
- append: append key-name value，将提供的值value追加到给定键key-name当前存储值的末尾
- getrange: getrange ket-name start end，获取一个由偏移量start-end内的所有字符组成的子串，start从0递增。
- setrange: setrange key-name start value，将从start偏移量开始的子串设置为value。
- getbit: getbit key-name offset，将字节串看做二进制串(bit string)，并返回串中偏移量为offset的二进制的值。
- setbit: setbit keyname offset value，将字节串看为二进制串，并将串中偏移量为offset的值设为value

例子:
```
>>> conn = redis.Redis()
>>> conn.append('newstring', 'hello ')
6
>>> conn.get('newstring')
'hello '
>>> conn.getrange('newstring', 0, 2)
'hel'
>>> conn.setrange('newstring', 6, 'world')
11
>>> conn.setbit('anohter-key', 2, 1)
0
>>> conn.setbit('another-key', 7, 1)  #redis存储二进制位按照从高到低进行排列
0
>>> conn.get('another-key')
'!'
```

#### 列表

redis列表允许用户从序列的两端弹出或者推入元素。获取元素，执行列表操作，列表还可以用来存储任务信息，最近浏览过的文章和常用的联系人信息。常用命令如下

- rpush: rpush key-name value [value...],将一个值或多个值从列表右端推入
- lpush: 同上，从左边推入
- rpop:  rpop key-name value,将一个值从列表右端弹出，并返回
- lpop: 同上，从左边
- lindex: lindex key-name offset,返回列表中偏移量为offset的元素
- lrange: lrange key-name start end,返回列表中偏移量从start开始到end结束的所有元素
- ltrim: ltrim key-name start end,对列表进行修剪，只保留start到end之间的元素，包括start和end

例子:
```
>>> conn.rpush('list-key', 'last')          # 在向列表推入元素时，
1L                                          # 推入操作执行完毕之后会返回列表当前的长度。
>>> conn.lpush('list-key', 'first')         # 可以很容易地对列表的两端执行推入操作。
2L
>>> conn.rpush('list-key', 'new last')
3L
>>> conn.lrange('list-key', 0, -1)          # 从语义上来说，列表的左端为开头，右端为结尾。
['first', 'last', 'new last']               #
>>> conn.lpop('list-key')                   # 通过重复地弹出列表左端的元素，
'first'                                     # 可以按照从左到右的顺序来获取列表中的元素。
>>> conn.lpop('list-key')                   #
'last'                                      #
>>> conn.lrange('list-key', 0, -1)
['new last']
>>> conn.rpush('list-key', 'a', 'b', 'c')   # 可以同时推入多个元素。
4L
>>> conn.lrange('list-key', 0, -1)
['new last', 'a', 'b', 'c']
>>> conn.ltrim('list-key', 2, -1)           # 可以从列表的左端、右端或者左右两端删减任意数量的元素。
True                                        #
>>> conn.lrange('list-key', 0, -1)          #
['b', 'c']
```
其他命令：    
- blpop: blpop key-name [key-name] timeout,从第一个非空列表中弹出位于最左端的元素，或者在timeout秒之内阻塞并等待可弹出的元素出现。
- brpop: brpop key-name [key-name] timeout,同上，但是弹出位于最右端的元素。
- rpoplpush: rpoplpush source-key dest-key,从source-key列表中弹出位于最右端的元素，将这个元素推如到desk-key的最左端，并返回这个元素
- brpoplpush: brpoplpush source-key dest-key,从source-key列表弹出最右端的元素，然后将这个元素推入到dest-key的最左端，并向用户返回这个元素，若source-key为空，那在timeout秒内阻塞并等待可弹出的元素出现。    

例子,带有b可以设置timeout:
```
>>> conn.rpush('list', 'item1')             # 将一些元素添加到两个列表里面。
1                                           #
>>> conn.rpush('list', 'item2')             #
2                                           #
>>> conn.rpush('list2', 'item3')            #
1                                           #
>>> conn.brpoplpush('list2', 'list', 1)     # 将一个元素从一个列表移动到另一个列表，
'item3'                                     # 并返回被移动的元素。
>>> conn.brpoplpush('list2', 'list', 1)     # 当列表不包含任何元素时，阻塞弹出操作会在给定的时限内等待可弹出的元素出现，并在时限到达后返回None（交互终端不会打印这个值）。
>>> conn.lrange('list', 0, -1)              # 弹出“list2”最右端的元素，
['item3', 'item1', 'item2']                 # 并将弹出的元素推入到“list”的左端。
>>> conn.brpoplpush('list', 'list2', 1)
'item2'
>>> conn.blpop(['list', 'list2'], 1)        # BLPOP命令会从左到右地检查传入的列表，
('list', 'item3')                           # 并对最先遇到的非空列表执行弹出操作。
>>> conn.blpop(['list', 'list2'], 1)        #
('list', 'item1')                           #
>>> conn.blpop(['list', 'list2'], 1)        #list空了，那么弹出list2的元素
('list2', 'item2')                          #
>>> conn.blpop(['list', 'list2'], 1)        #等待1s，没有元素可以弹出
```

对于阻塞弹出命令和弹出并且推入命令，最常见的技术消息传递(messageing)和任务队列(task queue)的开发

#### 集合

redis的集合以无序的方式来存储各个互不相同的元素，用户可以快速的对集合执行添加元素，移除元素，以及检验一个元素是否存在于集合的操作。常用的命令如下:

- sadd: sadd key-name item [item...],将一个或者多个元素添加到集合中，并返回添加元素中原本不存在于集合里面的元素数量。
- srem: srem key-name item [item...],从集合里面移除一个或者多个元素，并返回被移除元素的数量
- sismember: sismember key-name item,检查item元素是否存在与集合key-name中
- scard: scard key-name,返回集合中包含元素的数量
- smembers: smembers key-name,返回集合所包含的元素
- srandmember: srandmember key-name [count],从集合中随机返回一个或多个元素，当count为正数，命令返回的元素不会重复，当count为负数，命令返回的随机元素可能会重复。返回的数目为count的绝对值，若大于set所有元素的数量，则全部返回，负数则重复。
- spop: spop key-name,从集合里面移除并且返回一个随机元素
- smove: smove source-key dest-key item,如果集合source-key包含item,那么从集合source-key移除item，并且将item加入到dest-key中，如果item被成功移除，反应会1，否则返回0

示例如下:
```
>>> conn.sadd('set-key', 'a', 'b', 'c')         # SADD命令会将那些目前并不存在于集合里面的元素添加到集合里面，
3                                               # 并返回被添加元素的数量。
>>> conn.srem('set-key', 'c', 'd')              # srem函数在元素被成功移除时返回True，
True                                            # 移除失败时返回False；
>>> conn.srem('set-key', 'c', 'd')              # 注意这是Python客户端的一个bug，
False                                           # 实际上Redis的SREM命令返回的是被移除元素的数量，而不是布尔值。
>>> conn.scard('set-key')                       # 查看集合包含的元素数量。
2                                               #
>>> conn.smembers('set-key')                    # 获取集合包含的所有元素。
set(['a', 'b'])                                 #
>>> conn.smove('set-key', 'set-key2', 'a')      # 可以很容易地将元素从一个集合移动到另一个集合。
True                                            #
>>> conn.smove('set-key', 'set-key2', 'c')      # 在执行SMOVE命令时，
False                                           # 如果用户想要移动的元素不存在于第一个集合里，
>>> conn.smembers('set-key2')                   # 那么移动操作就不会执行。
set(['a'])   
```

如下是用于组合和处理多个集合的redis命令
- sdiff: sdiff key-name [key-name],返回存在与第一个集合而不存在于其他集合的元素。(数学的差集运算)。
- sdiffstore: sdiff dest-key key-name [key-name],将那些存在与第一个集合，而不存在于其他集合的元素存储到dest-key中。
- sinter: sinter key-name [key-name],返回存在于所有集合中的元素，数学的交集运算
- sinterstore: sinter dest-key key-name [key-name],同上，不过存储下来
- sunion: sunion key-name [key-name],数学的并集
- sunionstore: sunionstore dest-key key-name [key-name],交集并存储

例子如下:
```
>>> conn.sadd('skey1', 'a', 'b', 'c', 'd')  # 首先将一些元素添加到两个集合里面。
4                                           #
>>> conn.sadd('skey2', 'c', 'd', 'e', 'f')  #
4                                           #
>>> conn.sdiff('skey1', 'skey2')            # 计算从第一个集合中移除第二个集合所有元素之后的结果。
set(['a', 'b'])                             #
>>> conn.sinter('skey1', 'skey2')           # 还可以找出同时存在于两个集合中的元素。
set(['c', 'd'])                             #
>>> conn.sunion('skey1', 'skey2')           # 可以找出两个结合中的所有元素。
set(['a', 'c', 'b', 'e', 'd', 'f'])         #
```

#### 散列

redis的三列可以让用户将多个键值对存储带一个redis里面，从功能上来说，redis为散列值提供了一些和字符串相同的特性。是的散列适用于将一些相关的数据存储到一起，我们可以吧这些数据聚集作为关系数据库的行，或者文档中存储的文档。    
常用命令如下:
- hmget: hmget key-name key [key...],从散列里面获取一个或者多个键的值
- hmset: hmset key-name key value [key,value...],为散列里面的一个键或者多个键来设置值
- hdel: hdel key-name key [key...],删除散列六面的一个或者多个键值对，返回成功找到并删除键值的数目。
- hlen: hlen key-name,返回散列包含的键值对的数目    
例子如下:
```
>>> conn.hmset('hash-key', {'k1':'v1', 'k2':'v2', 'k3':'v3'})   # 使用HMSET命令可以一次将多个键值对添加到散列里面。
True                                                            #
>>> conn.hmget('hash-key', ['k2', 'k3'])                        #  使用HMGET命令可以一次获取多个键的值。
['v2', 'v3']                                                    #
>>> conn.hlen('hash-key')                                       # HLEN命令通常用于调试一个包含非常多键值对的散列。
3                                                               #
>>> conn.hdel('hash-key', 'k1', 'k3')                           # HDEL命令在成功地移除了至少一个键值对时返回True，
True 
```
其他的常用批量操作的命令:    
- hexists: hexists key-name key,检查给定键是否存在与散列之中
- hkeys: hkeys key-name,获取散列包含的所有键
- hvals: hvals key-name,获取散列包含的所有值
- hgetall: hgetall key-name,获取散列包含的所有键值对
- hincrby: hincrby key-name key increment,将键key的值加上整数increment
- hincrfloat: hincrfloat key-name key increment,将键的值加上浮点数increment    

例子如下:
```
>>> conn.hmset('hash-key2', {'short':'hello', 'long':1000*'1'}) # 在考察散列的时候，我们可以只取出散列包含的键，而不必传输大的键值。
True                                                            #
>>> conn.hkeys('hash-key2')                                     #
['long', 'short']                                               #
>>> conn.hexists('hash-key2', 'num')                            # 检查给定的键是否存在于散列中。
False                                                           #
>>> conn.hincrby('hash-key2', 'num')                            # 和字符串一样，
1L                                                              # 对散列中一个尚未存在的键执行自增操作时，
>>> conn.hexists('hash-key2', 'num')                            # Redis会将键的值当作0来处理。
True  
```

#### 有序集合

常用的有序集合的命令:
- zadd: zadd key-name score number [score number],将带有给定值的成员添加到有序集合里面
- zrem: zrem key-name member [member...],从有序集合移除给定的成员，并返回被移除成员的数目
- zcard: zcard key-name,返回有序集合包含的成员的数目
- zincrby: zincrby key-name increment member,将member的值加上increment
- zcount: zcount key-name min max,返回分值介于min和max之间的成员
- zrank: zrank key-name min max,返回成员member在key-name的排名
- zscore: zscore key-name member ,返回成员member的值
- zrange: zrange key-name start end [withscores],返回有序集合中排名介于start和end之间的成员，如果除了给定的可选的withscore选项，那么连成员的分值一并返回

例子如下:
```
>>> conn.zadd('zset-key', {'a': 3, 'b': 2, 'c': 1})   # 在Python以字典输入
3                                                   # 这跟Redis标准的先输入分值、后输入成员的做法正好相反。
>>> conn.zcard('zset-key')                          # 取得有序集合的大小可以让我们在某些情况下知道是否需要对有序集合进行修剪。
3                                                   #
>>> conn.zincrby('zset-key', 'c', 3)                # 跟字符串和散列一样，
4.0                                                 # 有序集合的成员也可以执行自增操作。
>>> conn.zscore('zset-key', 'b')                    # 获取单个成员的分值对于实现计数器或者排行榜之类的功能非常有用。
2.0                                                 #
>>> conn.zrank('zset-key', 'c')                     # 获取指定成员的排名（排名以0为开始），
2                                                   # 之后可以根据这个排名来决定ZRANGE的访问范围。
>>> conn.zcount('zset-key', 0, 3)                   # 对于某些任务来说，
2L                                                  # 统计给定分值范围内的元素数量非常有用。
>>> conn.zrem('zset-key', 'b')                      # 从有序集合里面移除成员和添加成员一样容易。
True                                                #
>>> conn.zrange('zset-key', 0, -1, withscores=True) # 在进行调试时，我们通常会使用ZRANGE取出有序集合里包含的所有元素，
[('a', 3.0), ('c', 4.0)]    
```
其他常用:    
- zrevrank: zrevrank key-name member,返回有序集合成员member所处的位置，成员按照从大到小排列。
- zrevrange: zrevrange key-name start stop [withscores],返回有序集合给定范围内的成员，成员按照分值由大到小排列。
- zrangebyscore: zrangebyscore key min max [withscores] [limit offset count],获取有序集合分值介于min和max之间，并按照分值从大到小的顺序返回他们
- zremrangebyrank: zremrangebyrank key-name start stop,移除有序集合排名介于start-stop之间的所有成员。
- zremrangebyscore: zremrangebyscore key-name min max,移除有序集合中分值介于min-max之间的所有成员。
- zinterstore: zinterstore dest-key key-count key [key...],给给定的有序集合执行集合的交集运算
- zunionstore: zunionstore dest-key key-count key [key...],并集操作。

```
>>> def publisher(n):
...     time.sleep(1)                                                   # 函数在刚开始执行时会先休眠，让订阅者有足够的时间来连接服务器并监听消息。
...     for i in range(n):
...         conn.publish('channel', i)                                  # 在发布消息之后进行短暂的休眠，
...         time.sleep(1)                                               # 让消息可以一条接一条地出现。
...
>>> def run_pubsub():
...     threading.Thread(target=publisher, args=(3,)).start()           # 启动发送者线程发送三条消息。
...     pubsub = conn.pubsub()                                          # 创建发布与订阅对象，并让它订阅给定的频道。
...     pubsub.subscribe(['channel'])                                   #
...     count = 0
...     for item in pubsub.listen():                                    # 通过遍历pubsub.listen()函数的执行结果来监听订阅消息。
...         print(item)                                                  # 打印接收到的每条消息。
...         count += 1                                                  # 在接收到一条订阅反馈消息和三条发布者发送的消息之后，
...         if count == 4:                                              # 执行退订操作，停止监听新消息。
...             pubsub.unsubscribe()                                    #
...         if count == 5:                                              # 当客户端接收到退订反馈消息时，
...             break                                                   # 需要停止接收消息。
...
>>> run_pubsub()                                                        # 实际运行函数并观察它们的行为。
{'pattern': None, 'type': 'subscribe', 'channel': 'channel', 'data': 1L}# 在刚开始订阅一个频道的时候，客户端会接收到一条关于被订阅频道的反馈消息。
{'pattern': None, 'type': 'message', 'channel': 'channel', 'data': '0'} # 这些结构就是我们在遍历pubsub.listen()函数时得到的元素。
{'pattern': None, 'type': 'message', 'channel': 'channel', 'data': '1'} #
{'pattern': None, 'type': 'message', 'channel': 'channel', 'data': '2'} #
{'pattern': None, 'type': 'unsubscribe', 'channel': 'channel', 'data':  # 在退订频道时，客户端会接收到一条反馈消息，
0L} 
```


curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.23.0/minikube-darwin-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/


