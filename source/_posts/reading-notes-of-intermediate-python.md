---
title: 阅读Python进阶这本书
date: 2019-12-07 18:40:01
tags:
---


记录我在阅读<Python进阶>这本书时认为我确实不太熟悉的一些知识和技巧. 

## 0x01 args 和 *kwargs

* `*args` 是用来发送一个**非键值对**的可变数量的参数列表给一个函数.
* `**kwargs` 允许你将不定长度的**键值对**, 作为参数传递给一个函数.

## 0x02 用PDB进行调试

* `python -m pdb my_script.py`这会在脚本第一行指令处停止执行. 
* `pdb.set_trace()`可以在脚本内部设置断点
  * 除开跟`gdb`相似的`c`, `s`, `n`外还有以下两个的命令符号不相同
  * `w`:  显示当前正在执行的代码行的上下文信息
  * `a`: 打印当前函数的参数列表

## 0x03 生成器

* 可迭代对象

  Python中任意的对象, 只要它定义了可以返回一个迭代器的`__iter__`方法, 或者定义了可以支持下标索引的`__getitem__`方法(这些双下划线方法会在其他章节中全面解释), 那么它就是一个可迭代对象.

* 迭代器: 任意对象, 只要定义了`next`(Python2) 或者`__next__`方法, 它就是一个迭代器.

* 生成器: 生成器会在每次迭代时`yield`产生一个值, 不必将所有结果都直接分配到内存中, 能有效地减少资源消耗.

## 0x04 Map, Filter和Reduce

* `map`可以将一个函数映射到一个输入列表的所有元素

  ``` python
  items = [1, 2, 3, 4, 5]
  squared = list(map(lambda x: x**2, items))
  ```

* `filter`则根据函数来过滤列表中的元素

  ``` python
  number_list = range(-5, 5)
  less_than_zero = filter(lambda x: x < 0, number_list)
  ```

* `reduce`会根据函数将列表中的元素进行累积操作

  ``` python
  from functools import reduce
  product = reduce( (lambda x, y: x * y), [1, 2, 3, 4] )
  # Output: 24
  ```

## 0x05 装饰器

* 函数可以单独作为参数进行传参, 也可以返回一个函数, 在函数内定义函数(不过无法在函数外部访问内部函数)

* 使用装饰器来修饰函数, 其标准的模式如下:

  ``` python
  from functools import wraps
  
  def a_new_decorator(a_func):
      @wraps(a_func)
      def wrapTheFunction():
          print("wrap function")
          a_func()
      return wrapTheFunction
  
  @a_new_decorator
  def a_function_requiring_decoration():
      print("need decoration")
  ```

## 0x06 对象变动

* 在Python中当函数被定义时, 默认参数只会运算一次, 而不是每次被调用时都会重新运算.

* 永远不要定义可变类型变量为默认参数, 正确的做法应如下:

  ``` python
  def add_to(element, target=None):
      if target is None:
          target = []
      target.append(element)
      return target
  ```

## 0x07 slots魔法

`__slots__`可以限定为类分配固定的内存空间. 能有效地减少内存占用	

``` python
class MyClass(object):
  __slots__ = ['name', 'identifier']
  def __init__(self, name, identifier):
      self.name = name
      self.identifier = identifier
      self.set_up()
```

## 0x08 容器

* defaultdict: 不需要检查`key`是否存

* counter: 可以帮助我们针对某项数据进行计数

* namedtuple: 可以帮助像字典一样访问, 但它是不可变的

  ``` python
  Animal = namedtuple('Animal', 'name age type')
  perry = Animal(name="perry", age=31, type="cat")
  # 定义了一个名为Animal的命名元组, 它有name, age, type三个属性
  ```

## 0x09 一行式

* 可以使用`pprint`进行更漂亮的输出
* `cat file.json | python -m json.tool`能更好地输出文件中的json数据
* 脚本性能分析: `python -m cProfile my_script.py`
* csv转json: `    python -c "import csv,json;print json.dumps(list(csv.reader(open('csv_file.csv'))))"`
* 列表辗平: `itertools.chain.from_iterable`

## 0x10 for-else语句

``` python
for item in container:
    if search_something(item):
        # Found it!
        process(item)
        break
else:
    # Didn't find anything..
    not_found_in_container()
```

## 0x11 使用C扩展

使用`gcc -shared -Wl,-soname,adder -o adder.so -fPIC add.c`将c文件编译成链接库文件. 然后再python中类似如下进行调用

``` python
from ctypes import *

#load the shared object file
adder = CDLL('./adder.so')

#Find sum of integers
res_int = adder.add_int(4,5)

```

`SWIG`适合于当有一个`C/C++`代码需要被多种语言调用的情况

## 0x12 兼容Py2+3

* 对于py2程序, 可以用`__future__`来导入py3的一些特性

* 使用`as`来对引入的库进行重命名

  ``` python
  try:
      import urllib.request as urllib_request  # for Python 3
  except ImportError:
      import urllib2 as urllib_request  # for Python 2
  
  ```

## 0x13 协程

协程跟生成器相反, 它是数据的消费者. 比如以下代码

``` python
def grep(pattern):
    print("Searching for", pattern)
    while True:
        line = (yield)
        if pattern in line:
            print(line)
search = grep('coroutine')
next(search)
#output: Searching for coroutine

```

发送的值会被`yield`接收, 通过`next()`方法来响应`send()`方法, 这样就能启动协程执行`yield`表达式

我们可以通过调用`close()`方法来关闭一个协程, `search.close()`

## 0x14 函数缓存

函数缓存允许我们将一个函数对于给定参数的返回值缓存起来.  当一个I/O密集的函数被频繁使用相同的参数调用的时候, 函数缓存可以节约时间. 

在Python 3.2版本以前我们只有写一个自定义的实现. 在Python 3.2以后版本, 有个`lru_cache`的装饰器, 允许我们将一个函数的返回值快速地缓存或取消缓存.

## 0x15 上下文管理器

除开用`with`来保证文件的打开和关闭外, 上下文管理器的一个常见用例, 就是资源的加锁和解锁.

一个上下文管理器的类, 最起码要定义`__enter__`和`__exit__`方法. 来看下面这个类

``` python
class File(object):
    def __init__(self, file_name, method):
        self.file_obj = open(file_name, method)
    def __enter__(self):
        return self.file_obj
    def __exit__(self, type, value, traceback):
        print("Exception has been handled")
        self.file_obj.close()
        return True
with File('demo.txt', 'w') as opened_file:
    opened_file.write('Hola!')

```

python的底层是这样运行的:

1. `with`语句先暂存了`File`类的`__exit__`方法
2. 然后它调用`File`类的`__enter__`方法
3. `__enter__`方法打开文件并返回给`with`语句
4. 打开的文件句柄被传递给`opened_file`参数
5. 我们使用`.write()`来写文件
6. `with`语句调用之前暂存的`__exit__`方法
7. `__exit__`方法关闭了文件

在第4步和第6步之间, 如果发生异常, Python会将异常的`type`,`value`和`traceback`传递给`__exit__`方法. 如果`__exit__`返回的是`True`则表示异常被正确处理, 否则异常会被with抛出. 

还可以用装饰器和生成器来进行上下文管理, 对应有一个`contextlib`模块

``` python
from contextlib import contextmanager

@contextmanager
def open_file(name):
    f = open(name, 'w')
    yield f
    f.close()

```

