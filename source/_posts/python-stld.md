---
title: python-list-dict-tuple-set
date: 2021-01-18 08:24:28
tags: python
---

看到一篇讲解python独一无二特性的文章，如简洁性，容易学习和使用等特点，而最常与python打交道的就是对于`list, dict, tuple, set`的操作

<!--more-->


### list

#### 创建方法

在python中，直接的方括号就表示list，同时通过`list()`实例化方法也可以创建列表对象，提到方括号，最常用的莫过于强大的列表生成式

```python
>>>a = []
>>>a = list()
# list()接受一个参数，可以把tuple，set，dict的keys及其他迭代器生成列表
>>>b = list((1,2,3))
>>>b
[1,2,3]
>>>c = list({'name':'alice','age':18})
>>>c
['name','age']
```

webinfo-lab1中的一些很长的写法

```python
useful_words = [word for word in words if hobj.spell(word) and word not in stopwords.words('english')]
# 去除不通过拼写检查的单词和停用词
```


#### 列表索引

列表的索引-index操作，体现出了python的简洁性

```python
>>>a = list((1,4,10,'x'))
>>>a[:-1]
[1,4,10]
>>>a[::-1]
['x',10,4,1]
```

注意体会以下两种表达方式结果的不同

```python
>>>a = [1,2,3]
>>>a[1] = [4,5,6]
>>>a
[1,[4,5,6],3]
>>>b = [1,2,3]
>>>b[1:1] = [4,5,6]
>>>b
[1,4,5,6,2,3]
```

#### 列表的一些方法
```python
>>> a = [3.14, False, 'x', None]
>>> a.index('x')
2
>>> a.append([1,2,3])
>>> a
[3.14, False, 'x', None, [1, 2, 3]]
>>> a[-1].insert(1, 'ok')
>>> a
[3.14, False, 'x', None, [1, 'ok', 2, 3]]
>>> a.remove(False)
>>> a
[3.14, 'x', None, [1, 'ok', 2, 3]]
>>> a.pop(1)
'x'
>>> a
[3.14, None, [1, 'ok', 2, 3]]
>>> a.pop()
[1, 'ok', 2, 3]
>>> a
[3.14, None]
```

### set&dict

Python用花括号表示字典和集合两种对象：花括号内是**空的**，或者是键值对的，表示字典；花括号内是无重复元素的，表示集合。为了不引起误会，可以用dict()来生成字典，用set()来生成集合。

集合的使用频率较低，字典的一些主要操作有

判断是否在集合内

```python
>>> a = dict({'x':1, 'y':2, 'z':3})
>>> 'x' in a
True
```

有的时候我们需要直接引用并更改某个key-value对，这时候用`d['key'] = new`比较方便，如果不存在会直接创建；
但是如果只需要通过key应用`d['key']`，那么在key不存在时会用引用不存在报错，可以用get方法

```python
# dict.get(key, default=None)
# key -- 字典中要查找的键。
# default -- 如果指定键的值不存在时，返回该默认值。
a = dict()
a.get('age',18)

# 输出为18
```

>如果字典里面嵌套有字典，无法通过 get() 直接获取 value

dict类提供了keys()、values()和items()等三个方法分别返回字典的全部键、全部值和全部键值对的迭代器
同时如果只是为了遍历所有元素，可以这样写

```python
for key in a:
	print(key, a[key])
```

### tuple

元组要注意的第一点是：元组初始化时，如果只有单个参数，则必须在单个参数之后增加一个逗号（，），否则，初始化结果仅返回原参数

```python
>>> a = (5)
>>> a
5
>>> type(a)
<class 'int'>
>>> a = (5,)
>>> a
(5,)
>>> type(a)
<class 'tuple'>
```

元组的重要性质：不可变性

1. 内部元素不可改变
2. 常用`*`解包元组：`*args`
3. 元组可以hash，而列表不能hash，所以可以把元组添加到可hash的集合set中

举例：

```python
>>> def sum(a,b):
...    return a+b
...
>>> args = (4,5)
>>> sum(*args)
9
>>> sum(args)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: sum() missing 1 required positional argument: 'b'
```

把元组添加到集合中

```python
>>> s = {1,'x',(3,4,5)}
>>> s
{1, (3, 4, 5), 'x'}
>>> s = {1,'x',[3,4,5]}
Traceback (most recent call last):
  File "<pyshell#32>", line 1, in <module>
    s = {1,'x',[3,4,5]}
TypeError: unhashable type: 'list'
```