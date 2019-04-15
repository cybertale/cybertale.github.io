---
layout: post
title: Python常用BIF
author: 宋强
tags: Python
---

都说“人生苦短，我用Python”，这句话在我用ROS必须在C++和Python中选择一个的时候深有体会，前段时间在做ROS图传小车，所以又加深了对Python的学习，在这个文章里总结一下常用的BIF函数。

# dir(__builtins__)
使用这个语句可以查看系统所有的内置函数（BIF:Built-In-Functions）

# Help(<function-name>)
来查看对于内置函数的帮助。

# list(<iterative-objects>='')
创建一个列表
示例：
```python
movies = list(['a', 'b', 'c'])
```

# range(<start>=0, <stop>, <step>=1)
创建一个从start开始的，到stop结束的间隔为step的range实例（for循环用）。
示例：
```python
movies = range(10)
movies = range(1, 5)
movies = range(2, 8, 2)
```

# enumerate(<iterative-objects>, index=0)
创建一个对实例集合的枚举，可指定第一个对应的index（for循环用）。
示例：
```python
movies = enumerate(['a', 'b', 'c', 'd'])
movies = enumerate(['a', 'b', 'c', 'd'], 1)
```

# int(<string>)
将字符串parse为int型变量之后返回。
示例：
```python
print(int('10'))
```

# open(<filename>, mode='r', buffer=0)
以mode模式打开一个文件进行操作，缓存大小为buffer，可使用的mode有如下几种：
* r：读取。
* w：写入。
* a：附加。
* b：读写二进制文件。
* +：连续读写文件。
* U：通用的新输入，不可和w和+参数共同使用。

示例：
```python
open('E:/Users/Duke/Desktop/test.txt')
fileTest = open('E:/Users/Duke/Desktop/test.txt', 'w+')
fileTest = open('E:/Users/Duke/Desktop/test.txt', 'w+', 10)
```

# isinstance(object, type)
判断object是否为type类型。
示例：
```python
a = range(10)
isinstance(a, range)
```
