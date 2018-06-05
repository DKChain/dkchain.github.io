---
title: 理解Python装饰器
date: 2018-06-04 11:42:24
tags:
category: 
- Python
- 函数式编程
---

## 1. 函数是对象
在Python中，函数也被实现为一种对象，这意味着函数也可以当做普通的变量一样处理，赋值，作为参数传递，作为返回值返回等等。由于内部将函数实现为对象，也被很多人诟病Python中的函数并不是"一等公民"却和函数式编程中的函数具有类似的特性。在SICP中，对"一等公民"的定义是
> Elements with the fewest restrictions are said to have first-class status. Some of the "rights and privileges" of first-class elements are:
> 1. They may be named by variables.
> 2. They may be passed as arguments to procedures.
> 3. They may be returned as the results of procedures.
> 4. They may be included in data structures.

即"一等公民"可以作为变量命名，可以作为参数用到procedures中，可以作为函数返回值返回，也可以包含在数据结构中。

## 2. 高阶函数
在维基百科中[高阶函数](https://zh.wikipedia.org/wiki/高阶函数)的定义如下:
> 在数学和计算机科学中，高阶函数是至少满足下列一个条件的函数：
> * 接受一个或多个函数作为输入 
> * 输出一个函数

也就是说，一个接受函数为参数或者返回一个函数的函数即为高阶函数，像Python内置的`map`、`filter`以及`wraps`等等都是高阶函数，我们常常会看到类似下面这种代码:
```
def square(x):
    return x*x
data = map(square, [0, 1, 2, 3])
print(data)
# Python2版本
# [Out] [0, 1, 4, 9]
# 如果上述代码运行在Python3环境下会是类似<map object at 0x00000000060596D8>的结果，这是Python3解释器将map的返回实现为生成器的原因，需要换一种方式
print(list(data))
# [Out] [0, 1, 4, 9]

def is_odd(n):
    return n%2==1

data = filter(is_odd, [0, 1, 2, 3, 4, 5, 6, 7])
print(list(data))
# [Out] [1, 3, 5, 7]
```

## 3. 闭包
维基百科中对[闭包](https://zh.wikipedia.org/wiki/闭包_(计算机科学))的定义如下:
>在计算机科学中，闭包（英语：Closure），又称词法闭包（Lexical Closure）或函数闭包（function closures），是引用了自由变量的函数。这个被引用的自由变量将和这个函数一同存在，即使已经离开了创造它的环境也不例外。所以，有另一种说法认为闭包是由函数和与其相关的引用环境组合而成的实体。闭包在运行时可以有多个实例，不同的引用环境和相同的函数组合可以产生不同的实例。

简单点来说，当我们在一个函数A内定义一个函数α，且在α中引用了函数A中的变量，而且函数A的返回值对α保有引用，那么这个时候便构成了一个闭包。也就是说，此时虽然函数A已经调用完成，但是依然可以通过它的返回值找到α的引用，比如说:
```
def func(step):
    n = 100
    def down():
        nonlocal n  # 此行代码在Python3版本有效，去掉此行会报错
        n -= step
        if n < 0:
            n = 0
        print(n)
    return down

foo = func(1)
bar = func(5)

foo()
# [Out] 99
foo()
# [Out] 98
bar()
# [Out] 95
```
在上述代码中，`foo`是一个闭包，`bar`是另一个闭包，为了能记录`n`的值，闭包必须要包含对`n`与`down`的引用，所以，闭包实际上是变量以及变量所处的环境的封装

## 4. 装饰器
说了这么多，终于来到了这篇文章的主角--装饰器。
装饰器是一个返回函数的高阶函数。利用装饰器，可以很方便的在执行某一个函数前做一些预处理，也可以在执行函数后执行一些预期的代码。
```
def clock(proc):
    def wrapper(*args, **kwargs):
        start = time.time()
        res = proc(*args, **kwargs)
        end = time.time()
        print("function %s execute %f seconds" % (proc.__name__, end-start))
        return res
    return wrapper

@clock
def func(n):
    time.sleep(n)
    print("wake up")

func(3)
# [Out] wake up
#       function func execute 3.001000 seconds
```
在这里，`func`函数定义前加了一行的`@clock`，等价于在定义`func`后执行`func = clock(func)`。

当然，装饰器本身也能有额外的参数
```
def clock(msg):
    def wrapper0(proc):
        def wrapper1(*args, **kwargs):
            print("Message from clock is '%s'" % msg)
            start = time.time()
            res = proc(*args, **kwargs)
            end = time.time()
            print("function %s execute %f seconds" % (proc.__name__, end-start))
            return res
        return wrapper1
    return wrapper0

@clock("Hello Lazier")
def func(n):
    time.sleep(n)
    print('wake up')

func(3)

# [Out] Message from clock is 'Hello Lazier'
#       wake up
#       function func execute 3.001000 seconds
```

## 5. 类装饰器
Python中的对象具有`__call__`函数，定义这个函数能够让对象和函数一样被调用。因此，在类中重写`__call__`函数即可实现类装饰器，例如:
```
class Decorator:
    def __init__(self, proc):
        print("Instance of Decorator init...")
        self.proc = proc
    
    def __call__(self, *args, **kwargs):
        print("Instance of Decorator call...")
        res = self.proc(*args, **kwargs)
        return res

@Decorator
def func():
    print("Hello")
# [Out] Instance of Decorator init...

func()
# [Out] Instance of Decorator call...
        Hello
```
当然，`@Decorator`的用法也可以用等价的python代码来实现。
```
func = Decorator(func)
```

## 6. 装饰器的顺序
一个函数可以使用多个装饰器，当函数有多个装饰器时，调用顺序为从上至下，即
```
@d0
@d1
@d2
def f():
    pass
```
等价于
```
f = d0(d1(d2(f)))
```

## 7. 使用装饰器可能产生的后果
从上面可以知道，装饰器是可以用`f = d(f)`的形式来等价实现的，这之中存在一个弊端，那就是`f`此时已经变成了`d`的返回值，也就是说，`__name__`和`__doc__`都会变成`d`的返回值的`__name__`和`__doc__`。
```
def d(proc):
    def wrapper(*args, **kwargs):
        return proc(*args, **kwargs)
    return wrapper

@d
def f():
    print("Hello")

print(f)
# [Out] <function d.<locals>.wrapper at 0x000000000609AA60>

print(f.__name__)
# [Out] wrapper
```
在某些时候，我们需要根据函数的名字来做判断，那么，用这样的方法自然无法满足我们的需求。Python内置的`wraps`可以帮助我们保留住函数原本的信息。(P.S. `wraps`在`functools`包中)
```
from functools import wraps
def d(proc):
    @wraps(proc)
    def wrapper(*args, **kwargs):
        return proc(*args, **kwargs)
    return wrapper

@d
def f():
    print("Hello")

print(f)
# [Out] <function f at 0x0000000004B58730>

print(f.__name__)
# [Out] f
```

利用好装饰器可以在用更少的代码实现更多的功能，并且也可以提高代码的复用性。设计优秀的装饰器对于调用者来说可以省去许多事情。

如有不足之处，欢迎指正。
