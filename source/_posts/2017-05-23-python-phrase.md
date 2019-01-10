---
title: python基础语法
date: 2017-05-23 23:17:02
tags:
- python
---

mark下
- [入门贴](https://zhuanlan.zhihu.com/p/23561159)
- [python 与 机器学习](https://zhuanlan.zhihu.com/p/24500257)
- [python 3.x 基础学习资料](https://zhuanlan.zhihu.com/p/24249743?refer=passer)

# python3 改变

## urlparse.urljoin
python3对urllib和urllib2进行了重构，拆分成了urllib.request, urllib.response, urllib.parse, urllib.error等几个子模块

# 对比java
1. python中没有类似java中的interface的概念

# list  常用操作 
`[]`

## list 操作符
| 表达式        | 结果           | 描述      |
| :-------------: |:-------------:|:-------------:|
|`len([1, 2, 3])`|3|长度|
|`[1, 2, 3] + [4, 5, 6]`|[1, 2, 3, 4, 5, 6]|组合|
|`['Hi!'] * 4`|['Hi!', 'Hi!', 'Hi!', 'Hi!']|重复|
|`3 in [1, 2, 3]`|True|元素是否存在于列表中|
|`for x in [1, 2, 3]: print x`|1 2 3|迭代|

## list 截取
定义原始样例list `L = ['v1', 'v2', 'v3']` 

| 表达式        | 结果           | 描述    |
| :-------------: |:-------------:|:-------------:|
|`L[2]`|'v3'|读取列表中第三个元素|
|`L[-2]`|'v2'|读取列表中倒数第二个元素|
|`L[1:]`|['v2', 'v3']|从第二个元素开始截取列表|

## 反转list
以下source代表list，ie: `source=[1,2,3,4,5,6,7,8,9]`
- 使用`reversed()`函数
```
b=list(reversed(source))
```
- 使用`sorted()`函数
```
c=sorted(source,cmp=None, key=None, reverse=True)  
```
- 使用分片
```
d=source[::-1]
```
  **其中`[::-1]`代表从后向前取值，每次步进值为1**

# 默认参数，单星号参数，双星号参数
- 默认值函数参数
第一个有默认值的参数后的每一个参数都必须提供默认值。
传参时，可以直接传参，也可以以“默认值参数名=value”的形式传参。
- 单星号函数参数
单星号函数参数接收的参数组成一个**`元组 '()'`**。
- 双星号函数参数
双星号函数参数接收的参数组成一个**`字典 '{}'`**。

## 一个星号（*）的函数参数
- 不传参数
参数位置被解析为空元组`()`
- 传多个值
多个参数被解析为一个元组
- 传多个元组
解析后的元组中的每个元素都是元组
- 参数是元组，并希望作为星号参数的参数值（不被再套一层元组）
传入元组参数前加**一个星号 `*`**

## 两个星号（**）的函数参数
- 不传参数
参数位置被解析为空map`{}`
- 传多个值，每个参数格式为 `key = val`
每个参数会作为一个kv保存在map `{}`中
- 参数传入map，作为双星号参数的参数值
传入map参数前加**两个星号 `**`**

# 常用描述

## from...import & import...as
- `from...import`
`from A import b` 相当于 
```
import A
b = A.b
```
- `import...as`
`import A as B` 给A库定义一个别名B

## lambda & def 差别
最大差别: `lambda可以定义一个匿名函数，而def定义的函数必须有一个名字`
与JavaScript不同的是，Python中匿名函数与非匿名函数需要使用不同的语法来定义。这是因为：
`lambda是一个expression，不是一个statement`
因此lambda表达式可以出现在def无法出现的地方。比如list comprehension。
lambda的函数体是一个简单的表达式而不是语句块。
所以lambda中没有return语句。也不能使用if, while等等

## `if __name__ == '__main__': `作用及原理
python文件两种使用方法
- 直接作为脚本执行
- import到其他的python脚本中被调用（模块重用）
`if __name__ == '__main__':` 保证只有在**直接作为脚本时才能执行，import到其他脚本时不会执行**
具体含义见特殊变量定义

--- 

# python中的下划线
- `_xxx`
保护变量，只有类对象和子类对象自己能访问到这些变量；
- `__xxx`
私有成员，意思是只有类对象自己能访问，连子类对象也不能访问到这个数据。
- `__xxx__`
系统定义名称


## `__name__` 
1. 直接被执行时，`__name__`为含`.py`尾缀的文件名
2. import时，`__name__`等于模块名称
## `__main__` 当前执行文件的文件名
## `__init` 构造函数
第一个参数永远是`self`，表示创建的实例本身，因此，在`__init__`方法内部，就可以把各种属性绑定到`self`，因为`self`就指向创建的实例本身

# if-else 单行写法

```list
if a>b:
    c = a
else:
    c = b

# 等同于

c = a if a>b else b
```

---

# python 用法

## list

### 作为Stack
- .append()
- .pop()

### 作为Queue
- .append()
- .popleft()

### List Comprehensions
[List comprehensions](https://docs.python.org/3/tutorial/datastructures.html#list-comprehensions)
**List comprehensions provide a concise way to create lists**
以下三种方法等价
- 方法1
```
squares = []
  for x in range(10):
    squares.append(x**2)
squares
>>> [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
```
- 方法2
```
squares = list(map(lambda x: x**2, range(10)))
```
- 方法3
```
squares = [x**2 for x in range(10)]
```
#### Nested List Comprehensions
```
[[row[i] for row in matrix] for i in range(4)]
```

## defaultdict & lambda

### 基础用法
```
x = defaultdict(lambda: 0)
# 可以简写为
x = defaultdict(int)
```
**when I call x[k] for a nonexistent key k (such as a statement like v=x[k]), the key-value pair (k,0) will be automatically added to the dictionary, as if the statement x[k]=0 is first executed.**


### 进阶用法
```
y = defaultdict(lambda: defaultdict(lambda: 0))
```
**when you do y`["ham"]["spam"]`, the key "`ham`" is inserted in y if it does not exist. The value associated with it becomes a defaultdict in which "`spam`" is automatically inserted with a value of 0.
I.e., y is a kind of "two-tiered" defaultdict. If "ham" not in y, then evaluating y["ham"]["spam"] is like doing
**

## `()`,`[]`,`{}`
### `()`
代表tuple元组
### `[]`
1. 代表list列表数据，列表是一种可变的序列
2. 字符串后边表示截取指定长度的字符串
`line = '+' + ('+------' * self.width + '+')[1:]`，最终实现目的是输出`+------+------+------+------+`
这不是最优的写法，可以调整为`line = '+' + ('------+' * self.width)`
### `{}`
代表dict字典类型。Set集合类型
```
{x: x**2 for x in (2, 4, 6)}
basket = {'apple', 'orange', 'apple', 'pear', 'orange', 'banana'}
```


---

# 定义函数
- 函数定义
**通过`def`定义函数**
```
def functionname(params):
    statement1
    statement2
```
- 默认参数
```
def test(a , b=-99):
  if a > b:
    return True
  else:
    return False
```
	默认参数使用限制
	> 1. 默认参数后边不能再有普通参数
	> 2. 默认值只被赋值一次（比如data=[]，则整个代码生命周期中，都只会创建一次数组）
```
	def f(a, data=[]):
	  data.append(a)
	  return data
	...
	>>> print(f(1))
	[1]
	>>> print(f(2))
	[1, 2]
	>>> print(f(3))
	[1, 2, 3]
```

- 强制关键字参数

```
def hello(*, name='User'):
  print("Hello", name)

hello(name='shiyanlou')
```

- 文档字符串 docstrings（说明如何使用代码）

```
#!/usr/bin/env python3
import math

def longest_side(a, b):
    """
    Function to find the length of the longest side of a right triangle.

    :arg a: Side a of the triangle
    :arg b: Side b of the triangle

    :return: Length of the longest side c as float
    """
    return math.sqrt(a*a + b*b)
```

- 高阶函数
高阶函数（Higher-order function）或仿函数（functor）是内部至少含有一个以下步骤的函数：
> 使用一个或多个函数作为参数
> 返回另一个函数作为输出

Python 里的任何函数都可以作为高阶函数。

```
def high(func, value):
  return func(value)
lst = high(dir, int)
print(lst[-3:])
['imag', 'numerator', 'real']
```

- map函数
`map`是一个在 Python 里非常有用的高阶函数。
> 它接受一个函数和一个序列（迭代器）作为输入，然后对序列（迭代器）的每一个值应用这个函数，返回一个序列（迭代器），其包含应用函数后的结果。

```
  lst = [1, 2, 3, 4, 5]
  def square(num):
    return num * num
  ...
  print(list(map(square, lst)))
  [1, 4, 9, 16, 25]
```

- any函数
任何一个为`True`，则返回`True`

---

# 数据结构

## 元组 tuple
- 由数个都好分割的值组成 
`a = 'Fedora', 'ShiYanLou', 'Kubuntu', 'Pardus'`
- 可以对任何一个元组执行拆封操作并赋值给多个变量
`x, y = divmod(15,2)`
- 元组是**不可变类型**
- 创建只含有一个元素的元组，在值后面跟一个逗号
`b = 321,`

## 字符串
一些有意思的地方
- 分几行输入字符串
```
print("""\
  Usage: thingy [OPTIONS]
    -h              Display this usage message
    -H hostname     Hostname to connect to
""")
>> Usage: thingy [OPTIONS]
>>   -h                Display this usage message
>>   -H hostname       Hostname to connect to
```
- python支持回文
```
#!/usr/bin/env python3
s = input("Please enter a string: ")
z = s[::-1]
if s == z:
    print("The string is a palindrome")
else:
    print("The string is not a palindrome")
```
- 单词计数
```
#!/usr/bin/env python3
s = input("Enter a line: ")
print("The number of words in the line are %d" % (len(s.split(" "))))
```

## 集合
*无序不重复元素的集*
集合支持的操作
- union：联合
- intersection：交集
- difference：差集
- symmetric difference：对称差集

集合表示方法
- 大括号{}
`basket = {'apple', 'orange', 'apple', 'pear', 'orange', 'banana'}`
- set()
`a = set('abracadabra')`

## k-v （字典）
- 创建
`data = {'kushal':'Fedora', 'kart_':'Debian', 'Jace':'Mac'}`
- 新增
`data['parthan'] = 'Ubuntu'`
- 删除
`del data['kushal']`
- 判断是否存在
`'ShiYanLou' in data`
- 从包含键值对的元组中创建
`dict((('Indian','Delhi'),('Bangladesh','Dhaka')))`
- 遍历
```
for x, y in data.items():
  print("{} uses {}".format(x, y))
```
- 不存在则添加默认v
```
data = {}
data.setdefault('names', []).append('Ruby')
```
- 查找的key不存在时返回默认值
`data.get('foo', 0)`

## 遍历
- 遍历列表（或任何序列类型）的同时获得元素索引值 `enumerate()`
```
for i, j in enumerate(['a', 'b', 'c']):
  print(i, j)
# 其中i是索引
# ...
# 0 a
# 1 b
# 2 c
```
- 同时遍历两个序列类型 `zip()`
```
a = ['Pradeepto', 'Kushal']
b = ['OpenSUSE', 'Fedora']
for x, y in zip(a, b):
  print("{} uses {}".format(x, y))
```

--- 

# 异常

## 异常定义

| 名称        | 描述           |
| :-------------: |:-------------:|
|NameError|试图访问一个未定义的变量|
|TypeError|当操作或函数应用于不适当类型的对象时引发，一个常见的例子是对整数和字符串做加法|

## 处理异常
```
try:
    statements to be inside try clause
    statement2
    statement3
    ...
except ExceptionName:
    statements to evaluated in case of ExceptionName happens
```

## 抛出异常 `raise`
```
raise ValueError("A value error happened.")
```

## 清理行为 `finally`
```
try:
  raise KeyboardInterrupt
finally:
  print('Goodbye, world!')
```

---

# 类

在Python中，所有数据类型都可以视为对象，当然也可以自定义对象。自定义的对象数据类型就是面向对象中的类（Class）的概念。

## 定义类 & 类继承
```
class nameoftheclass(parent_class, parent_class_2, ...):
    statement1
    statement2
    statement3
```

## 构造方法 `__init__`
```
def __init__(self):
  statement
  statement
```

## 属性（attributes）读取
> 在 Python 里请不要使用属性（attributes）读取方法（getters 和 setters）

## 装饰器 @property（对指定的属性的set或get方法增加filter）
```
class Account(object):

    @property
    def amount(self):
        """账号余额（美元）"""
        return self.__amt

    @amount.setter
    def amount(self, value):
        if value < 0:
            print("Sorry, no negative amount in the account.")
            return
        self.__amt = value
```

---

# 模块
> 模块是包括 Python 定义和声明的文件。文件名就是模块名加上 .py 后缀。

## `__name__`
可以由全局变量 __name__ 得到模块的模块名（一个字符串）。

模块示例，文件名为 `bars.py`，可以通过`import model_name`导入模块
```
"""
Bars Module，File name is bars.py
============
这是一个打印不同分割线的示例模块
"""
def starbar(num):
    """打印 * 分割线

    :arg num: 线长
    """
    print('*' * num)

def hashbar(num):
    """打印 # 分割线

    :arg num: 线长
    """
    print('#' * num)

def simplebar(num):
    """打印 - 分割线

    :arg num: 线长
    """
    print('-' * num)
```

## 常用模块
| 名称        | 描述           |
| ------------- |:-------------|
|os|提供与操作系统相关的功能|
|requests|Requests 唯一的一个非转基因的 Python HTTP 库，人类可以安全享用。警告：非专业使用其他 HTTP 库会导致危险的副作用，包括：安全缺陷症、冗余代码症、重新发明轮子症、啃文档症、抑郁、头疼、甚至死亡。|
|collections|包含一些很好用的数据结构|
|unittest|python 单元测试，异常测试|

### Counter
- 有助于 hashable 对象计数的 dict 子类。它是一个无序的集合，其中 hashable 对象的元素存储为字典的键，它们的计数存储为字典的值，计数可以为任意整数，包括零和负数。
- `elements()` 的方法，其返回的序列中，依照计数重复元素相同次数，元素顺序是无序的。
- `most_common()` 方法返回最常见的元素及其计数，顺序为最常见到最少。
```
>>> from collections import Counter
>>> import re
>>> path = '/usr/lib/python3.4/LICENSE.txt'
>>> words = re.findall('\w+', open(path).read().lower())
>>> Counter(words).most_common(10)
[('the', 80), ('or', 78), ('1', 66), ('of', 61), ('to', 50), ('and', 48), ('python', 46), ('in', 38), ('license', 37), ('any', 37)]

>>> c = Counter(a=4, b=2, c=0, d=-2)
>>> list(c.elements())
['b','b','a', 'a', 'a', 'a']

>>> Counter('abracadabra').most_common(3)
[('a', 5), ('r', 2), ('b', 2)]
```

# 包
含有 `__init__.py` 文件的目录可以用来作为一个包，目录里的所有 `.py` 文件都是这个包的子模块

如果 `__init__.py` 文件内有一个名为 `__all__` 的列表，那么只有在列表内列出的名字将会被公开

--- 

# 迭代器
Python 迭代器（Iterators）对象在遵守迭代器协议时需要支持如下两种方法。
- `__iter__()`，返回迭代器对象自身。这用在 `for` 和 `in` 语句中。
- `__next__()`，返回迭代器的下一个值。如果没有下一个值可以返回，那么应该抛出 `StopIteration` 异常。

# 生成器

# 生成器表达式 

# 闭包 Closures （嵌套函数）

闭包（Closures）是由另外一个函数返回的函数。我们使用闭包去除重复代码。在下面的例子中我们创建了一个简单的闭包来对数字求和。


```
>>> def add_number(num):
...     def adder(number):
...         #adder 是一个闭包
...         return num + number
...     return adder
...
>>> a_10 = add_number(10)
>>> a_10(21)
31
>>> a_10(34)
44
>>> a_5 = add_number(5)
>>> a_5(3)
8
```


---

# 文件操作 `open()`

## 文件打开
`open()` 函数打开文件。
第一个参数是文件路径或文件名
第二个是文件的打开模式。

模式通常是下面这样的：
`r`: **默认方式** 以只读模式打开，你只能读取文件但不能编辑/删除文件的任何内容
`w`: 以写入模式打开，如果文件存在将会删除里面的所有内容，然后打开这个文件进行写入
`a`: 以追加模式代开，写入到文件中的任何数据将自动添加到末尾

## 文件关闭 `close()`

> 需要保证显式关闭，因为能够打开的文件句柄是有限的

## 文件读取 `read()`

- `read()`: 全部读完
- `read(size)`: 读取指定长度
- `readline()`: 读取一行

## 文件写入 `write()`

> 会覆盖当前存储的信息

## 安全的操作文件 `with`
> 使用`with`，可以保证文件自动关闭

  ```
  with open('sample.txt') as fobj:
    for line in fobj:
      print(line, end = '')
  ```

## linux下查询当前CPU信息 `lscpu`
只读方式读取 `/proc/cpuinfo` 中的信息

---

# 便利方法
## 内建函数
- `type()`
查看任意变量的数据类型

- `range()`

- `__doc__`
查看方法的doc
