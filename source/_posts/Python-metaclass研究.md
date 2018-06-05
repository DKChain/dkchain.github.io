---
title: Python metaclass研究
date: 2018-06-04 09:51:02
tags:
category: 
- Python
- 面向对象
---

## 1. class is object
在绝大多数程序设计语言中，类是一种对象的抽象表达，Python自然也不例外:
```
class SimpleClass:
    pass

obj = SimpleClass()
print(obj)
```
上述代码执行后会将类的模块名，类名以及在内存中的地址打印出来，如下:
```
<__main__.SimpleClass object at 0x0000000005C95BA8>
```
不过在Python中，类本身也是一种对象，在上述代码定义好`SimpleClass`类后，我们可以试着将`SimpleClass`本身打印出来
```
print(SimpleClass)
# [Out] <class '__main__.SimpleClass'>
```
也就是说，当执行完`SimpleClass`的定义代码后，解释器会在内存中创建一个新的对象，名为`SimpleClass`。当然，由于类本身也是一种对象，自然对类的种种操作:赋值给变量，添加属性，作为参数传递
```
def func_print(arg):
    print(arg)

func_print(SimpleClass)
# [Out] <class '__main__.SimpleClass'>

print(hasattr(SimpleClass, 'attr'))
# [Out] False

SimpleClass.attr = 'value'
print(hasattr(SimpleClass, 'attr'))
# [Out] True
print(SimpleClass.attr)
# [Out] value

class_ = SimpleClass
print(class_)
# [Out] <class '__main__.SimpleClass'>

print(class_())
# [Out] <__main__.SimpleClass object at 0x0000000005CC7EF0>
```

## 2. type()
Python内置函数type通常会用来判断传入的参数的类型，如:
```
type(1)
# [Out] int

type('abc')
# [Out] str

def func():
    print("Hello")

type(func)
# [Out] function

s = SimpleClass()
type(s)
# [Out] __main__.SimpleClass

type(SimpleClass)
# [Out] type

print(type)
# [Out] <class 'type'>
```
从最后两段代码的执行结果可以看到，Python中**类**的**类型**是`type`,`type`也是一种类。而`type`的语法为
```
type(name, bases, dict)
```
其中`bases`和`dict`都不是必选参数，其中`bases`是一个tuple，表示该类继承哪些类，`dict`是成员变量的映射关系。在只有一个参数的时候，`type`会返回传入的参数的类型，在有两个以上参数时，会创建一个新的类，也就是说:
```
class A:
    a = 1
    def func(self, n):
        print(self.a*n)
```
和
```
A = type('A', (object,), dict(a = 1, func = lambda self,n:print(self.a*n)))
```
这两段代码是等价的，任取其中一段都能创建一个新的类A，可以用这个类A来创建对象。利用`type`，可以在代码运行期间动态创建类。

## 3. metaclass
metaclass可以控制类的创建行为，参照下述代码:
```
class ListMetaclass(type):
    def __new__(cls, name, bases, attrs):
        attrs['add'] = lambda self, value:self.append(value)
        return type.__new__(cls, name, bases, attrs)

class MyList(list, metaclass = ListMetaclass):
    pass

my_l = MyList()
my_l.add(1)
print(my_l)
# [Out] [1]

l = list()
l.add(1)
# [Out] 
#---------------------------------------------------------------------------
#AttributeError                            Traceback #(most recent call last)
#<ipython-input-59-001a7ae7d652> in <module>()
#----> 1 l.add(1)
#
#AttributeError: 'list' object has no attribute 'add'
```
从结果来看，定义`ListMetaclass`，然后在定义`MyList`时继承`list`类，并指定`metaclass`为`ListMetaclass`;接着创建`MyList`的实例`my_l`，并调用`add`函数添加一个新的对象;然后再创建`list`的实例`l`，并调用`add`函数，结果抛出`AttributeError`异常。

重头看一下整个代码，在一开始定义`ListMetaclass`的时候，定义了`__new__`函数
```
def __new__(cls, name, bases, attrs):
    attrs['add'] = lambda self, value: self.append(value)
    return type.__new__(cls, name, bases, attrs)
```
其中，`cls`是当前准备创建类的对象，`name`是类的名字，`bases`是类继承的父类的tuple，`attrs`是类的成员映射。
当我们定义(创建)`MyList`类时，会通过`ListMetaclass.__new__()`进行创建，在这个函数中，我们可以对类的定义进行修改，比如添加自定义函数`add`等，之后在调用`type`创建类并将其返回。

看起来有一点混乱?现在来理一下使用`metaclass`动态创建类并操作类的创建行为的流程
```
定义class
    |__指定metaclass
        |__调用metaclass.__new__
            |__完成metaclass的动作，返回新的类
```

## 4. 单例模式的实现
这种动态创建类的特性，在绝大多数情况下并不会用到，因为在很多场景下并不需要我们实现动态创建类的功能。不过，在很多框架里面却会反复用到这样的特性，比如说ORM(对象-关系映射);也可以利用这样的特性实现单例模式，具体参照如下代码:
```
class SingleTon(type):
    def __init__(self, *args, **kwargs):
        self._ins = None
        super(SingleTon, self).__init__(*args, **kwargs)

    def __call__(self, *args, **kwargs):
        if self._ins is None:
            self._ins = super(SingleTon, self).__call__(*args, **kwargs)
        return self._ins

class Any(metaclass = SingleTon):
    any_value = 1

a = Any()
b = Any()

print(id(a))
# [Out] 19184944
print(id(b))
# [Out] 19184944
```
上述代码便实现了单例模式，在第一次创建`Any`的实例`a`时，创建了新的对象，id为19184944，在第二次创建`Any`的实例`b`时，由于已经存在了实例`a`，便直接将实例`a`返回，所以此时`a`和`b`是指向同一个实例的不同的变量名。在这里可能会产生疑问:为什么在定义`SingleTon`类时，在`__call__`函数中进行了是否存在实例的判断，而不是用`__new__`函数?因为，metaclass是用来创建**类本身**，而不是用来创建**类构造的对象**的，这意味着`__new__`函数只会在解释器读取完`Any`类并将其创建的时候执行一次。在`__call__`被调用时，`self`就是类，形如`Any()`这样的代码会调用到`__call__`函数，在类`Any`看来，`__call__`中的`self`就是`<class '__main__.Any'>`，所以创建两个`Any`的实例，`__call__`就会被调用两次，所以，在`__call__`中对`self`操作，添加新的属性`_ins`用来保存一个`Any`的实例，并在每次构造是判断是否存在该实例来决定是否创建新的实例。

当然，使用`__new__`函数也能实现单例模式，不过在这里就应该在需要实现单例模式的类中重写`__new__`函数，参考代码如下:
```
class SingleTon:
    def __new__(cls, name, bases, attrs):
        if not hasattr(cls, '_ins'):
            cls._ins = super(SingleTon, cls).__new__(cls, name, bases, attrs)
        return cls._ins

a = SingleTon()
b = SingleTon()
print(id(a))
# [Out] 12696264
print(id(b))
# [Out] 12696264
```
如有不足之处，欢迎指正。
