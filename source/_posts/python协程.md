---
title: python协程
date: 2019-01-16 20:25:31
tags: python
categories: python
---
线程和进程的操作是由程序出发系统接口，最后的执行者是系统，协程的操作则是程序员。

协程存在的意义:对于多线程应用，cpu通过切片的方式来切换线程间的执行，线程切换时需要耗时(保存状态，下次继续)。协程，则只用一个线程，在一个线程中规定代码块的执行顺序。

协程适用的场景:当程序中存在大量不需要cpu的操作时(io)，适用于协程。

event loop是协程执行的控制点，如果希望执行协程，就需要用到他们。

event loop提供了如下特性:
- 注册，执行，取消延时调用(异步函数)
- 创建用于通信的client与server协议(工具)
- 创建和别的程序通信的子进程和协议(工具)
- 把函数送入线程池

<!-- more -->
```
import asyncio

async def cor1():
    print('cor1 start')
    await cor2()
    print('cor1 end')
    
async def cor2():
    print('cor2')
    
loop = asyncio.get_event_loop() #async启动默认的event loop
loop.run_until_complete(cor1())  #这个函数是阻塞执行的，直到所有的异步函数执行完成
loop.close() #关闭event loop

输出:
cor1 start
cor2
cor1 end
```

#### 1.greenlet
```
import greenlet
 
 
def fun1():
    print("12")
    gr2.switch()
    print("56")
    gr2.switch()
 
def fun2():
    print("34")
    gr1.switch()
    print("78")
 
 
gr1 = greenlet.greenlet(fun1)
gr2 = greenlet.greenlet(fun2)
gr1.switch()

输出:
12
34
56
78
```

#### 2.gevent
```
import gevent
 
def fun1():
    print("www.baidu.com")   # 第一步
    gevent.sleep(0)
    print("end the baidu.com")  # 第三步
 
def fun2():
    print("www.zhihu.com")   # 第二步
    gevent.sleep(0)
    print("end th zhihu.com")  # 第四步
 
gevent.joinall([
    gevent.spawn(fun1),
    gevent.spawn(fun2),
])

输出:
www.baidu.com
www.zhihu.com
end the baidu.com
end th zhihu.com
```

### io操作
IO在计算机中指Input/Output，也就是输入和输出。由于程序和运行时数据是在内存中驻留，由CPU这个超快的计算核心来执行，涉及到数据交换的地方，通常是磁盘、网络等，就需要IO接口。

比如打开一个网页，网络io操作，我们需要先发送数据(output)去访问网页的服务器，然后接受网页的数据(input)。

IO编程中，Stream（流）是一个很重要的概念，可以把流想象成一个水管，数据就是水管里的水，但是只能单向流动。Input Stream就是数据从外面（磁盘、网络）流进内存，Output Stream就是数据从内存流到外面去。对于浏览网页来说，浏览器和新浪服务器之间至少需要建立两根水管，才可以既能发数据，又能收数据.

由于CPU和内存的速度远远高于外设的速度，所以，在IO编程中，就存在速度严重不匹配的问题。举个例子来说，比如要把100M的数据写入磁盘，CPU输出100M的数据只需要0.01秒，可是磁盘要接收这100M数据可能需要10秒，怎么办呢？有两种办法

第一种是CPU等着，也就是程序暂停执行后续代码，等100M的数据在10秒后写入磁盘，再接着往下执行，这种模式称为同步IO；

另一种方法是CPU不等待，只是告诉磁盘，“您老慢慢写，不着急，我接着干别的事去了”，于是，后续代码可以立刻接着执行，这种模式称为异步IO。

很明显，使用异步IO来编写程序性能会远远高于同步IO，但是异步IO的缺点是编程模型复杂。想想看，你得知道什么时候通知你“汉堡做好了”，而通知你的方法也各不相同。如果是服务员跑过来找到你，这是回调模式，如果服务员发短信通知你，你就得不停地检查手机，这是轮询模式。总之，异步IO的复杂度远远高于同步IO。
























