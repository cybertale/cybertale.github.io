---
layout: post
title: python - logging
author: 宋强
tags: python
date: 2018-06-20 23:51 +0800
---

直接使用类似

```python
logging.warning('Watch out!')
```

就可以输出，总共有几种，描述如下：

|                     | gate                    |
|---------------------|-------------------------|
| logging.info()      | 调试信息                |
| logging.debug()     | 调试信息                |
| logging.warn()      | log程序应该能解决的警告 |
| logging.warning     | log程序无法解决的警告   |
| logging.error()     | 错误                    |
| logging.exception() | 异常                    |
| logging.critical()  | 非常严重，系统停止      |

logging模块存在一个默认等级，INFO、DEBUG、WARNING、ERROR、CRITICAL，只有在这个等级和比这个等级高的错误会输出，默认是WARNING，

增加修改方式。

# 输出到文件

```python
import logging
logging.basicConfig(filename='example.log',level=logging.DEBUG)
logging.debug('This message should go to the log file')
logging.info('So should this')
logging.warning('And this, too')
```

# 覆盖log可以增加打开模式信息

```python
logging.basicConfig(filename='example.log', filemode='w', level=logging.DEBUG)
```

# logging支持格式化输出

```python
logging.warning('%s before you %s', 'Look', 'leap!')
```

想要更改整个输出信息的格式的话，可以使用：

```python
logging.basicConfig(format='%(levelname)s:%(message)s', level=logging.DEBUG)
```

输出形如：

```
DEBUG:This message should appear on the console
INFO:So should this
WARNING:And this, too
```

增加时间段：

```python
logging.basicConfig(format='%(asctime)s %(message)s', datefmt='%m/%d/%Y %I:%M:%S %p')
logging.warning('is when this event was logged.')
```

输出形如：

```
12/12/2010 11:46:36 AM is when this event was logged.
```

可以不指定datefmt。

可以格式化的选项：

* name – The name of the logger used to log the event represented by this LogRecord. Note that this name will always have this value, even though it may be emitted by a handler attached to a different (ancestor) logger.
* level – The numeric level of the logging event (one of DEBUG, INFO etc.) Note that this is converted to two attributes of the LogRecord: levelno for the numeric value and levelname for the corresponding level name.
* pathname – The full pathname of the source file where the logging call was made.
* lineno – The line number in the source file where the logging call was made.
* msg – The event description message, possibly a format string with placeholders for variable data.
* args – Variable data to merge into the msg argument to obtain the event description.
* exc_info – An exception tuple with the current exception information, or None if no exception information is available.
* func – The name of the function or method from which the logging call was invoked.
* sinfo – A text string representing stack information from the base of the stack in the current thread, up to the logging call.