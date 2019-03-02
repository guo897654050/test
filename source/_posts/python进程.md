---
title: python进程
date: 2019-01-16 12:02:16
tags: python
categories: python
---
#### 1.multiprocessing模块
直接从侧面用multiprocess替换线程使用GIL的方式，由于这一点，multiprocessing模块可以让程序员在给定的机器上充分使用cpu，在multiprocessing中，通过multiprocessing对象生成进程，然后调用其start()方法。
```
from multiprocessing import Process

def func(name):
    print('hello', name)
    
if __name__ == '__main__':
    p = Process(target=func, args=('guoguoguo',))
    p.start()
    p.join()
```
<!-- more -->
在使用并发设计的时候尽可能的避免数据共享，尤其使用多进程的时候，如果真的需要共享数据，multiprocessing提供了两种方式：
#### 1.1 multiprocessing，Array，Value
数据可以使用value或者Array存在共享内存中。
```
from multiprocessing import Array,Value,Process
 
def func(a,b):
    a.value = 3.333333333333333
    for i in range(len(b)):
        print(-b[i])
        b[i] = -b[i]
        print('b[i]',b[i])
 
 
if __name__ == "__main__":
    num = Value('d',0.0)
    arr = Array('i',range(11))
 
 
    c = Process(target=func,args=(num,arr))
    d= Process(target=func,args=(num,arr))
    c.start()
    d.start()
    c.join()
    d.join()
 
    print(num.value)
    for i in arr:
        print(i)
```

#### 1.2multiprocessing, Manager
由manager()返回的manager提供list，dict，Namespace，Lock，RLock，Semmaphore，BoundedSemaphore,Condition,Event,Barrier,Queue,Value,Array类型的支持。
```
from multiprocessing import Process, Manager

def func(d, l):
    d['name'] = 'zhangchunhua'
    d['age'] = 21
    d['job'] = 'teacher'
    l.reverse()
    
if __name__ == '__main__':
    with Manager() as man:
        d = man.dict()
        l = man.list(range(10))
        print(l)
        p = Process(target=func, args=(d, l))
        p.start()
        p.join()
        print(d)
        print(l)
```

#### 2.进程池
Poll类描述了一个工作进程池，投几种不同的方法让任务卸载工作进程。

进程池内部维护一个进程序列，当使用时去进程池获取一个进程，如果进程池没有可以使用的进程，那么程序就会等待，直到进程池中有可用的进程为止。

我们可以用Poll类来创建一个进程池，展开提交的任务给进程池，例
```
from multiprocessing import Process,Manager,Pool

def f1(i):
    time.sleep(0.5)
    print(i)
    return i+100

if __name__ == '__main__':
    pool = Pool(5)
    for i in range(1,31):
        pool.apply(func=f1, args=(i,))
        
    pool.close()
    pool.join()

###apply_async
def f1(i):
    time.sleep(0.5)
    print(i)
    return i+100

def f2(arg):
    print(arg)
    
if __name__ == '__main__':
    pool = Pool(5)
    for i in range(1,31):
        pool.apply_async(func=f1, args=(i,), callback=f2)
    pool.close()
    pool.join()
```
一个进程池对象可以控制工作进程池那些工作可以被提交，它支持超时和回调的异步结果，有一个类似map的实现。

- processes:使用的工作进程的数量，如果为processess是None，那么使用os.cpu_count()返回的数量。
- initializer: 如果initializer为None，那么每一个工作进程在开始时候会调用initializer(*initargs)
- maxtasksperchild:工作进程退出之前可以完成的任务数，完成后用一个新的工作进程来代替原进程，来让闲置的资源被释放。maxtasksperchild默认是None，意味着只要pool存在工作进程就一直存活。
- context: 用在制定工作进程启动时的上下文，一般使用multiprocesssing.Pool()或者一个context对象的Pool方法来创建一个池，两种方法都适当的设置了context。

#### 2.1进程池方法
- apply(func[, args[,kwds]]):使用arg和kwds参数调用func函数，结果返回前会一直阻塞，由于这个原因，apply_async()更适合并发执行，此外，func函数仅被pool中的一个进程运行。
- apply_async:(func[,args[,kwds[,callback[,error_callback]]]]):apply()方法的一个变体，会返回一个结果对象。如果callback被指定，那么callback可以接受一个参数然后被调用，调用失败时，则用error_callback。callback硬背立即执行完成，否则处理结果的线程会被阻塞。
- close():组织更多的任务提交到pool，待任务完成后，工作进程会退出。
- terminate():不管任务是否完成，立即停止工作进程，在对pool进行垃圾回收的时候，会立即调用terminate
- join():wait工作线程的退出，在调用join()前，必须调用close()或者terminate()，这样是因为被终止的进程需要被父进程调用wait(join等价于wait)，否则会成为僵尸进程。





























































