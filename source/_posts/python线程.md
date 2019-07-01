---
title: python线程
date: 2019-01-16 12:01:00
tags: python
categories: python
---
### 进程与线程的关系

线程是属于进程的，线程运行在进程空间内，同一进程所产生的线程共享同一内存空间，当进程退出时，该进程所产生的所有线程都会被强制清除并且退出。。线程可与属于同一进程的其他线程共享进程的全部资源。


### python的线程

Threading用于提供线程相关的操作，线程是应用程序中的最小单元。

#### 1.threading模块

threading模块是对thread模块的二次封装，提供了更方便的api来处理线程。
```
import threading
import time
def work(num):
    time.sleep(1)
    print("the number is {}".format(num))
   #return

for i in range(20):
    t = threading.Thread(target=work, args=(i,), name="t{}".format(i))
    t.start()
```
<!-- more -->
这里创建了20个线程，然后控制器就交给了cpu，cpu根据指定算法进行调度，分片执行命令。    
方法说明:    
1.t.start():激活线程    
2.t.getName():获取线程名称    
3.t.setName():设置线程名称    
4.t.name:获取或者设置线程名称    
5.t.is_alive():判断线程是否为激活状态    
6.t.isAlive():判断线程是否为激活状态    
7.t.setDaemon():设置为后台线程或者前台线程(默认为False),通过设定一个布尔值设置线程是否为守护线程，必须在start()方法之后才可以使用。如果是后台线程，主线程执行过程中，后台线程也在进行，主线程执行完毕，无论后台线程成功与否，均停止。如果是前台线程，主线程执行完毕，前台线程也在执行，主线程执行完毕后，等待前台线程也执行完毕后，程序停止。    
8.t.isDaemon():判断是否为守护线程    
9.t.ident:获取线程的标识符，线程标识符是一个非零整数，只有在调用了start()方法之后属性才有效,否则返回None。    
10.t.join():逐个执行每个线程，执行完毕后往下执行，该方法使得多线程无意义。    
11.t.run():线程被cpu调度后自动执行run方法。

#### 2.线程锁threading.RLock和threading.Lock

由于线程之间是随机调度，并且每个线程可能执行n条执行后，cpu接着执行其他线程，为了保证数据的准确性，引入了锁的概念，可能出现的问题如下:    
假如列表A的所有元素都为0,当一个线程从前向后打印所有元素，另一个线程从后向前修改列表的元素为1，那么输出的时候，列表的元素会一部分为0，一部分为1，导致了数据的不一致。锁的出现解决了这个问题。    
```
import threading
import time

global_num = 0
lock = threading.RLock()

def Func():
    lock.acquire()    #获得锁
    global global_num
    global_num += 1
    time.sleep(1)
    print(global_num)
    lock.release()    #释放锁

for i in range(10):
    t = threading.Thread(target=Func,)
    t.start()
```
#### 3.thrading.RLock和thrading.Lock
RLock允许在同一线程被多次require，而Lock却不允许，如果使用RLock，那么acquire和release必须成对出现。
```
import threading
lock = threading.Lock()
lock.require()
lock.acquire()  ##产生了死锁
lock.release()
loack.release()

rlock = threading.RLock()
rlock.require()
rlock.require() #在同一线程内，程序不会堵塞
rlock.release()
rlock.release()
```

#### 4.threding.Event

python线程的事件用于主线程控制其他线程的执行，事件主要提供了三个方法:set,wait,clear。    
事件处理的机制:全局定义了一个`Flag`,如果`Flag`的值为False，那么当程序执行event.wait方法就会堵塞，如果`Flag`为True,那么event.wait方法时不会堵塞。

- clear: 将`Flag`设置为False
- set: 将`Flag`设置为True
- Event.isSet():判断标识位是否为True
```
import threading

def do(event):
    print('start')
    print('event:',event.isSet())
    event.wait()
    print('execute')
    
event_obj = threading.Event()
for i in range(10):
    #event_obj.set()
    t = threading.Thread(target=do, args=(event_obj,))
    t.start()
    
event_obj.clear()
inp = input("input:")
if inp == 'true':
    event_obj.set()
```
当线程执行的时候，如果Flag为False，则线程会阻塞，若为True，线程不会阻塞，提供了本地和远程的并发性。

#### 5.threading.Condition

一个condition变量总是与某些类型的锁相关联，这个可以使用默认的情况创建一个，当几个condition变量必须共享变量和同一个锁的时候是很有用的，锁是condition对象的一部分，没有必要分别跟踪.    

condition变量服从上下文管理协议，with语句封闭之前可以取得和锁的联系，acquire()和release()会调用与锁相关的方法。   

其他和锁关联的方法必须被调用,wait()方法会释放锁，当另一个线程使用notify()或者notify_all()唤醒它之前会一直阻塞，一旦被唤醒，wait()会重新获得 锁并返回。

Condition实现了一个condition变量，这个condition变量
允许一个或者多个线程等待，直到他们被另一个线程通知。如果lock参数，被给定一个非空的值，那么他必须是一个lock或者Rlock对象，他用来做底层锁，否则会创建一个新的Rlock对象，用来做底层锁。

- wait(timeout=None):等待通知，或者等待设定的超时事件，当调用这个wait()方法时，如果调用他的线程没有得到锁，会抛出RuntimeError异常。wait()释放锁以后，在调用相同条件下的另一个进程用notify()或者notify_all()叫醒之前，会一直阻塞，wait()可指定超时时间。

如果有等待的线程，notify()方法会唤醒一个在等待的condition变量的线程，notify_all()则会唤醒所有在等待的condition变量的线程。
```
import threading
import time

def consumer(cond):
    with cond:
        print("consumer before wait")
        cond.wait()
        print("consumer after wait")

def producter(cond):
    with cond:
        print("producter beofre notifyAll")
        cond.notifyAll()
        print("producter after notifyAll")
        
condition = threading.Condition()
c1 = threading.Thread(name='c1', target=consumer, args=(condition,))
c2 = threading.Thread(name='c2', target=consumer, args=(condition,))
p1 = threading.Thread(name='p1', target=producter, args=(condition,))

c1.start()
time.sleep(2)
c2.start()
time.sleep(2)
p1.start()

输出:
consumer before wait
consumer before wait
producter beofre notifyAll
producter after notifyAll
consumer after wait
consumer after wait
```

#### 6.queue模块

Queue就是队列，他是线程安全的，举例来说，我们去麦当劳吃饭，前台负责把做好的饭卖给顾客，顾客去前台领取做好的饭，这里的前台相当于我们的队列，厨师做好饭通过前台传递给顾客，所谓单项队列。这个模型也叫生产者-消费者模型。
```
import queue
import threading
import time

class Threadingpool():
    def __init__(self, max_num=10):
        self.queue = queue.Queue(max_num)
        for i in range(max_num):
            self.queue.put(threading.Thread)
    
    def getThreading(self):
        return self.queue.get()
    
    def addThreading(self):
        return self.queue.put(threading.Thread)
    
def func(p, i):
    time.sleep(1)
    print(i)
    p.addThreading()
    
if __name__ == '__main__':
    p = Threadingpool()
    for i in range(11):
        thread = p.getThreading()
        print(thread)
        t = thread(target= func, args=(p, i))
        t.start()

```


























































































