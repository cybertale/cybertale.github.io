---
layout: post
title: Linux内核List操作解析
author: 宋强
tags: Linux List
date: 2018-07-04 22:34 +0800
---

# 创建List

## 使用LIST_HEAD

```c++
static LIST_HEAD(tc_list);      //定义一个链表，tc_list是一个指针
```

LIST_HEAD的定义为：
```c++
#define LIST_HEAD_INIT(name) { &(name), &(name) }
 
#define LIST_HEAD(name) \
    struct list_head name = LIST_HEAD_INIT(name)
```

***花括号里面两个地址什么意思还不清楚。***

## 使用INIT_LIST_HEAD

先静态声明一个再使用INIT_LIST_HEAD进行初始化

```c++
struct list_head *tc_list;
INIT_LIST_HEAD(tc_list);
```

# 操作元素

list的节点基类要求就是包含一个list_head类型的元素，一般起名为node，之后内核会使用container_of解析子类，添加元素的时候我们直接给基类指针，例如子类定义为：

```c++
struct atmel_tc
{
        int index;
        struct list_head node;
}
 
struct atmel_tc *tc;  //Assuming that tc will be allocated as soon as possible.
```

# 添加元素
传输基类首地址，也就是node属性的地址：

```c++
list_add(&tc->node, &tc_list);  //Add an element after the specific head node. 
list_add_tail(&tc->node, &tc_list);   //Add an element before the specific head node.
```

# 删除元素

```c++
list_del(&tc->node);
```

# 移动元素

待补充。

# 替换元素

待补充。

# 遍历元素

待补充。
