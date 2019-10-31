---
title: python包导入核心概念理解
description: python包导入核心概念理解
categories:
 - python
tags:
 - python
 - 包导入
---

## python包导入核心概念理解

### 背景

平时使用python，没特殊关注包导入的机制， 这里列一下自己核心概念

### 核心概念


* 包不会主动导入 子包或者子模块， 需要在 `__init_`里定义，`__init__`主要是为了定义是包结构， 比如 `from . import xxxx， 或者全局导入到相对模块`

* `__All__`的命令主要是为了默认导入包地址， `from package import *`


* 模块导入会主动将模块的所有方法显示出来，但包导入不会主动暴露子包或者子模块

*  会将`__name__="main"`的目录加到sys.path目录
*  大部分情况下使用相对路径， 主要使用绝对路径，会使得项目变得僵硬， 如果顶层项目修改包名， 所有的绝对路径包名都要调整

* python3.0以后， 即使没有`__init__`, python仍然会导入包, 主要是因为存在 `命名空间包`
* 在执行模块导入时，Python 的导入系统会首先尝试从 sys.modules 查找。sys.modules 中是所有已导入模块的一个缓存，包括中间路径。即，假如 foo.bar.baz 被导入，那么，sys.modules 将包含进入 foo，foo.bar 和 foo.bar.baz 模块的缓存。其实一个 dict 类型，每个键都有自己的值，对应相应的模块对象。

### 主要参考资料：

* https://juejin.im/post/5da9a715e51d45249246ef91
* https://juejin.im/post/5da9a746518825668f167314
* https://python3-cookbook.readthedocs.io/zh_CN/latest/c10/p01_make_hierarchical_package_of_modules.html
* https://codingpy.com/article/python-import-101/
* http://liao.cpython.org/23createpackage/
* https://github.com/numpy/numpy/blob/master/numpy/__init__.py
* http://kuanghy.github.io/2018/08/05/python-module-import











