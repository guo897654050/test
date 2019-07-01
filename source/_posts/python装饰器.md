---
layout: posts
title: python装饰器
date: 2019-03-15 09:26:48
tags: python
categories: python
---

### 转:一种自顶而下的python装饰器的设计方法

昨天阅读了一篇关于装饰器的文章，读了感觉写的挺好的，自己在记录一遍加深印象。

#### python装饰器原理

python装饰器的常规写法:
```
@decorator
def func(8args, **kwargs)
    so_something()
```
这种写法只是一种语法糖，使得代码看起来更加简洁，在python解释器内部，函数`func`的调用被转换为下面的方式.
```
>>>func(a, ,b, c='value')
>>>decorated_func = decorator(func)
>>>decorated_func(a, b, c='value')
```
<!--more -->
#### 简单装饰器

简单的装饰器不带任何参数，例如，我们把装饰器命名为`timethis`
```
@timethis
def func(*args, **kwargs):
    pass
```
根据上文的分解，我们可以写出装饰器的模板代码
```
def timethis(func):
    def wrapper(*args, **kwargs):
        pass
    return wrapper
```
如此，装饰器的框架搭建好了，需要丰富下函数的逻辑。对于被装饰的`func`的调用，相当于对wrapper的调用。
```
def timethis(func)
    def wrapper(*args, **kwargs):
        result = func(*args, **kwargs)
        return result
    return wrapper
```
我们可以丰富函数`wrapper`的逻辑，我们的需求是打印`func`的调用时间,在'func'前后调用计时。
```
def timethis(func):
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(func.__name__, end-start)
        return result
    return wrapper
```
至此，完成一个打印函数事件的装饰器，不过在`func.__name__`这里，我们会返回wrapper，只需要在加上`@wrap(func)`即可返回正确的名称。测试一下

```
@timethis
def fibonacci(n):
    a = b = 1
    while n > 2:
        a, b = b, a+b
        n -= 1
    return b
fibonacci(100)
```

#### 带参数的装饰器

上面的装饰器简单些，不带参数，我们常看到带有参数的装饰器。
```
@logged(debug, name='example', message='message')
def func(*args, **kwargs):
    pass
```
`logged`返回一个装饰器，这个装饰器再去装饰`func`函数，因此`logged`的模板应为:
```
def logged(level, name=None, message=None):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            pass
        return wrapper
    return decorator
```
可见`wrapper`是最终被调用的函数,我们可以丰富`decorator`和`wrapper`这两个函数。
```
from functools import wraps
def logged(level, name=None, message=None):
    def decorator(func):
        logname = name if name else func.__module__
        logmsg = message if message else func.__name__
        @wraps
        def wrapper(*args, kwargs):
            print(level, logname, logmsg)
            result = func(*args, **kwargs)
        return wrapper
    return decorator
```
例子如下:
```
from functools import wraps
    ...: def logged(level, name=None, message=None):
    ...:     def decorator(func):
    ...:         logname = name if name else func.__module__
    ...:         logmsg = message if message else func.__name__
    ...:         @wraps(func)
    ...:         def wrapper(*args, **kwargs):
    ...:             print(level, logname, logmsg)
    ...:             result = func(*args, **kwargs)
    ...:             return result
    ...:         return wrapper
    ...:     return decorator

@logged('info', name='aaa', message='bbb')
def test(a,b):
    return a+b
    
test(1,3)
输出: info aaa bbb
out 4
```

#### 多功能装饰器

```
@logged
def func(*args, **args):
    pass
    
@logged(level, name='example', message='example')
def func(*args, **kwargs):
    pass
```
根据前面的分析，带有参数的装饰器和不带参数的装饰器是不同的，不带参数的装饰器返回的是被装饰的函数，而带参数的装饰器返回的是不带参数的装饰器(不太准确，被传递的func也算作参数),然后再返回被装饰的函数，多了一层相当于。需要用到`partial`。
```
from functools import wraps, partial

def logged(func=None, level='debug', name=None, message=None):
    if func is None:
        reutrn partial(logged, level=level, name=name, message=message)
    logname = name if name else func.__module__
    logmsg = message if message else func.__name__
    
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(level, lgoname, logmsg)
        return func(*args, **kwargs)
    return wrapper
    
```
这个装饰器以带参数的形式使用，这里的第一个参数`func`
值为`None`,此时我们用`partial`反返回了一个其他参数的固定装饰器，这个装饰器与不带参数的简单装饰器一样。
```
 #使用@logged装饰
>>>func(a, b, c)
>>> decorated_func = logged(func)  #可见func不为空
>>> decorated_func(a, b, c)

 #使用@logged(level=level, name='example', message='message' )
 相当于
>>> decorator = logged(level=level, name='example', message='message')
>>> decorated_func = decorator(func)
>>> decorated_fun(a,b,c)

 #例子如下
@logged
def test(a,b):
   reutrn a+b
test(1,2)
输出: None, __main__ test, 第一个是level默认为None

@logged(level='aa', name='bb', message='cc')
def test(a,b):
    return a+b
    
test(3,4)
输出: aa,bb,cc
```
例子    
设计一个装饰器函数retry,当被装饰的函数抛出指定的异常时候，函数会被重新调用，直到达到最大调用次数才重新抛出指定的异常，使用示例如下
```
@retry(times=10, traced_exception=ValueError,reraised_exception=CustomException)
def test():
    pass
```
其中，traced_exception为监控的异常，可以为None(默认),异常类，或者一个异常类的列表，如果为None，则监听所有的异常；如果指定了异常类，若函数调用抛出指定的异常，重新调用函数，直至返回成功或者达到最大尝试次数，此时重新抛出原异常(reraised_exception的值为None），或者抛出有reraised_exception指定的异常。

```
def retry(times=10, traced_exception=None, reraised_exception=None):
    def decorator(func)
        @wraps
        def wrapper(*args, **kwargs):
            n = times
            traced_all =  traced_exption is None
            traced_spec = traced_exption is not None
            while True:
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    traced = traced_spec and isinstance(e, traced_exception)    
                    reached_limits = n == 0
                    if not(trace_all or  traced) or reached_limits:
                        if reraise_exception is not None:
                            raise reraise_exception
                        raise
                    n -= 1
        return wrapper
    return decorator
```
