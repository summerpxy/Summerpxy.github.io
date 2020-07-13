---
layout: post
title: "python中的装饰器"
date:  2020-7-14 12:06:00  
categories: python
---

python的装饰器是用来装饰函数的。这是什么意思呢？**假如我们有一个函数，这个函数的功能不能满足我们现有的需求，那么我们可以通过装饰器在这个函数执行前执行后做一些我们需要的操作(如果函数本身功能不满足，那就直接修改方法体了，不需要装饰器帮忙)**。

#### **1. 简单装饰器**
装饰器的语法糖是使用`@`符号表示，装饰器本身也是一个函数，只不过参数是函数而已。

```python
def decor_function(func):
    def wrapper_function():
        print("[%s] %s() called" % (ctime(), func.__name__))
        return func()
    return wrapper_function

@decor_function
def my_func():
    print("Hello world")

...
my_func()
```
`decor_function`也就是我们的装饰器函数，它对原有的函数进行包装，返回一个包装过的函数`wrapper_function`。使用`@`修饰过的函数`my_func`，返回的函数实际上是装饰器返回的函数`wrapper_function`.

```shell
[Mon Jul  9 17:07:40 2018] my_func() called
Hello world
```

#### **2. 修饰含有参数的函数**
函数定义可以使用任意的参数，那么装饰器函数如何处理呢？其实很简单，使用`*args`和`**kargs`就可以方便的调用了，只需要在装饰器函数的返回的函数中将参数传递给被修饰的函数就可以了。

```python
def decor_function(func):
    def wrapper_function(*args, **kargs):
        print("[%s] %s() called" % (ctime(), func.__name__))
        return func(*args, **kargs)
    return wrapper_function
  
@decor_function
def my_func_with_param(name):
    print("Hello", name)

my_func_with_param("Joe")
```

```shell
[Mon Jul  9 17:12:58 2018] my_func_with_param() called
Hello Joe
```

#### **3. 装饰函数带参数**

装饰器函数本身也是可以带参数的，使用参数，可以根据具体的场景添加不同的功能实现。

```python
def decor_function_with_parm(level):
    if level == "info":
        logging.info("info message logged")
    elif level == "error":
        logging.error("error message logged")
    else:
        logging.debug("debug message logged")

    def wrapper_outter_func(func):
        def wrapper_inner_func(*args, **kargs):
            func(*args, **kargs)
        return wrapper_inner_func

    return wrapper_outter_func


@decor_function_with_parm(level="info")
def my_func2(name):
    print("Hello,", name)


my_func2("Joe")
```
带参数的装饰器函数写起来比较麻烦，因为需要处理的参数比较多，**一般最外层的函数处理装饰器参数，接下来的函数处理`func`，最后一层函数用来处理被修饰的函数的参数。**

#### **4. 多重修饰**
一个函数可以被多个装饰器修饰，like this

```python
@decor_function_with_parm(level="info")
@decor_function
def my_func():
    print("Hello world")

```
执行的顺序是：

```python

f = decor_function_with_parm(level='info', decor_function(my_func()))
```

#### **5.使用类来处理**
类的`__call__()`方法可以把类当成函数来处理，所以类也可以用做装饰器

```python
class Decor:
    def __init__(self, func):
        print("__init__ method called")
        self.func = func

    def __call__(self, *args, **kargs):
        print("__call__ method called")
        self.func(*args, **kargs)


@Decor
def func(name):
    print("func called")
    print("Hello,",name)


func("joe")

```
使用类做装饰器时，init函数中添加被修饰函数的引用，在call函数中处理参数。

```shell
__init__ method called
__call__ method called
func called
Hello, joe
```

#### **6.保留函数的元信息**
被修饰之后的函数，它的元信息都消失，被替换的wrapper函数代替。python中提供了`functools.wraps`来保存函数的元信息。`wraps`本身也是个装饰器

```python
def decor_function(func):
    @wraps(func)
    def wrapper_function(*args, **kargs):
        print("[%s] %s() called" % (ctime(), func.__name__))
        print(func.__name__)
        return func(*args, **kargs)
    return wrapper_function
   
@decor_function
def my_func_with_param(name):
    print("Hello", name)

```

```shell
[Mon Jul  9 18:16:11 2018] my_func_with_param() called
my_func_with_param
Hello joe
```

参考：[理解 Python 装饰器看这一篇就够了][1]


[1]: https://foofish.net/python-decorator.html