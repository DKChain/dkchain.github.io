---
title: 协程&&gevent
date: 2018-06-13 21:28:46
tags:
category: 
- Python
- 多线程
- 协程
---

## 1. 协程
通常我们会认为一个函数的执行会是从头到尾，每次的执行的行为几乎是一样的，但是这样的感觉在协程中并不适用，协程，我认为是一个可以被中断以及被重新唤起的函数。在协程的执行过程中，我们可以在需要的地方将协程中断，并转而去执行别的操作，这里与多线程有着显著的区别。在多线程中，会是多个线程同时执行，而协程则始终只有一个线程，只是反复在多个函数之间切换，并且每个函数都会保留上次的执行状态。
```
# 通常函数调用
def func_a():
    print("x")
    print("y")
    print("z")

def func_b():
    print(1)
    print(2)
    print(3)

func_a()
func_b()

# [Out] 
#       x
#       y
#       z
#       1
#       2
#       3


# 协程
def func_a():
    print("x")
    yield
    print("y")
    print("z")

def func_b():
    print(1)
    print(2)
    yield
    print(3)

a = func_a()
b = func_b()

next(a)
next(b)
next(b)
next(a)
# [Out]
#       x
#       1
#       2
#       3
#       y
#       z

# 在第二次调用b和a时，会捕获到StopIteration异常，如果不想看到这个异常使用try语句忽略即可
```
上面是一个简单的例子，可以看到在使用协程的时候会与一般的函数调用有所区别，首先在第一次调用`a = func_a()`时，并没有打印出`x`，而是将其返回值赋值给了`a`,其次用到了`next`函数，并且在调用`next`时会将对应的函数的代码执行。我们可以直接将`a`打印出来，得到如下结果:
```
print(a)
# [Out] <generator object func_a at 0x000002E619E9E780>

print(func_a)
# [Out] <function func_a at 0x000002E619EB9D90>
```
`a`是`generator`, `func_a`是`function`。也就是说，这时调用`func_a`并不会直接调用到`func_a`中的代码，而是得到一个`generator`，通过`next`函数才能执行其中的代码。

## 2. 生产者消费者模式
通常实现生产者消费者模式会使用多线程，消费者线程监听资源池，等待生产者往资源池中写入资源，为了避免出现死锁的问题，需要设置锁。如果使用协程来实现，在生产者写入资源之后直接调用消费者使用资源即可，避免了锁的问题
```
import time
def consume():
    msg = ''
    while True:
        n = yield msg
        if not n:
            return
        print("Consume %d" % n)
        time.sleep(1)
        msg = 'ok'
 
def produce(c):
    c.__next__()
    for i in range(1, 5):
        print("Produce %d" % i)
        r = c.send(i)
        print("Consumer said: %s" % r)
    c.close()

c = consume()
produce(c)
```
这是一个简单的生产者消费者模式，`produce`接受参数`consume`，在接受到`consume`时，首先使用`__next__`函数对他进行调用，此时`consume`的下一次调用入口为`n = yield msg`，当`produce`使用`send`函数将`i`传入`consume`时，`consume`中的`n`的值即为`i`;最后循环至第五次时推出循环，关闭consume。

## 3. gevent
gevent是第三方协程库，通过greenlet实现协程，原理是当一个greenlet遇到IO操作时则切换到其他的greenlet，待IO操作完成后再在某个时机切换回来继续执行，这样可以使程序在很多时候都是在处理计算操作而不是等待IO。

Demo
```
from gevent import monkey
monkey.patch_socket()
monkey.patch_ssl()
import gevent
import request

def proc(url):
    print("Access %s..." % url)
    rep = requests.get(url)
    print("Length of response from %s is %d" % (url, len(rep.text)))

gevent.joinall([
    gevent.spawn(proc, 'https://www.baidu.com'),
    gevent.spawn(proc, 'https://github.com'),
    gevent.spawn(proc, 'https://www.bilibili.com')
])
# [Out] Access https://www.baidu.com...
#       Access https://github.com...
#       Access https://www.bilibili.com...
#       Length of response from https://www.baidu.com is 2443
#       Length of response from https://www.bilibili.com is 24268
#       Length of response from https://github.com is 59430
```
上述代码最终的输出结果"Access..."部分几乎是同时打印出来的，因为requests的操作是IO操作，因此直接被暂时搁置了。值得注意的小细节是，`monkey.patch_socket()`和`monkey.patch_ssl()`这两行代码，由于切换greenlet时是在遇到IO操作时进行，因此gevent会对Python内置的库进行一些修改，为了避免出现网络请求上的异常需要将两行代码包含进来，具体需要patch哪些部分请根据实际需求来操作。

## 4. 多线程与协程的不同
这里我排除Python中的GIL带来的影响，单从多线程与协程本身的特性来分析。
|    | 多线程    |  协程  |
|:--:|:--------:|:------:|
|代码复杂度|需要考虑访问共享数据时带来的麻烦，复杂度较高|单线程处理，不存在数据共享带来的问题，复杂度较低
|切换的代价|线程切换由CPU进行调度，两个线程的切换需要将线程A的状态保存，并重新载入线程B的状态，在不同的操作系统中表现不一样，通常会涉及到页表切换，TLB刷新等操作|在用户态上做切换，涉及到堆栈信息的复制，开销很小|
|创建与销毁|不同的操作系统对线程的创建开销不尽相同，但是都会为线程分配堆栈与内存空间|协程本质上就是函数，与调用函数的开销相同|
从上面来看，协程在代码复杂度，创建以及切换的开销上都比使用多线程要来的优秀，**但是**，如果说协程的主流程中存在阻塞的情况那么整个协程都会被阻塞，如果说整个程序只有计算的话，协程并不见得比多线程有优势，相反，在IO密集型的场景下，协程比之多线程要效率高一点，因此，具体使用哪一种来实现系统应当根据实际情况来设计，而不是盲目认为协程比多线程更优或是多线程比协程更快。
如有不足之处，欢迎指正。

