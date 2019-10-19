---
layout: default
title: Python  Import 和 From import
excerpt: 由于Python  Import 和 From import 的不同机制，会导致很多预料不到的错误。总结归纳了一下可能的问题 和 解决办法
category: Python
---
> 由于Python  Import 和 From import 的不同机制，会导致很多预料不到的错误。总结归纳了一下可能的问题 和 解决办法

-------------------------------

问题：
1. 命名空间污染
2. 循环import

### 命名空间污染
```python
# a.py
def say():
    print 'Hi, Here is a'
# b.py
def say():
    print 'Hi, Here is b'
from a import *
if __name__ == '__main__':
    say()
```
执行结果：
```
Hi, Here is a
```
原理： 
- import a:  将 "a" 引入到当前名字空间。 
- from a import * ： 不将 "a" 引入，而将 a module 中所有的属性的名字引入。虽然这样 可以直接调用 a module 中的方法，但也可能覆盖了其他同名的方法或属性。 慎用

### 循环import
```python
# module_1/a.py
from module_2 import b
print 'Here is a'
# module_2/b.py
from module_1 import a
print 'Here is b'
# test.py
import module_1.a
if __name__ == '__main__':
    print ''
```
执行 test.py 会发现如下错误：
```
Traceback (most recent call last):
  File "C:/PycharmProjects/Study/mytest.py", line 4, in <module>
    import module_1.a
  File "C:\PycharmProjects\Study\module_1\a.py", line 4, in <module>
    from module_2 import b
  File "C:\PycharmProjects\Study\module_2\b.py", line 4, in <module>
    from module_1 import a
ImportError: cannot import name a
```
 但如果把a b两个文件中的 from...import 换成 import， 就不会这个错误。这是为什么呢？
原因是 


- import 语句执行时， 会检查下 需要import的module 在不在 sys.modules 里。
不在： create  一个新的entry 在sys.modules 里，然后去执行需要导入module里的代码块 直到执行完毕
在 ： 直接返回。而不管这个module是否完全加载（即该module中代码都被执行过） 
-  from ... import 语句执行时，会要求被导入的module 已经被执行编译过。 
这就解释了为什么 在 module2.b module 中，执行 from...import a 是被（a 没有执行完毕）， import moudle_1.a 可以成功（a module已经在之前被放入 sys.modules 里）。

在pycharm里debug打断点验证过。

慎用from....import. 
