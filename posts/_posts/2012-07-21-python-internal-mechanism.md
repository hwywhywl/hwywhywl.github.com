--- 
layout: post
title: python 内部机制的理解
tags:
- python
---

7.12 那天列了以下提纲，现在填吧。

1. Object -- 一个类继承与不继承object 的区别
2. setattr(obj, name, value) getattr(obj, name, default) 所遇所想
3. python 私有变量理解
4. 类中__new__(cls, *args, **kwarg)、__init__(self, *args, **kwargs)
   和
   元类中 __new__(cls, name, bases, **attrs)、__init__(cls, name, bases, **attrs)

Python 类派生object 与 不派生至 object 还是有很大的区别的，当然我这里并不想深入python类内部机制，知识想浅浅说明在实际应用中的对比，
当一个类不继承object是，会出现一些莫名其妙的状况，例如，你为一个类定义一个描述器

    
class Column:
    def __get__(self, instance, owner):
        print 'Enter __get__'
        return self

class Table:
    id = Column()

>>> class Column:
...     def __get__(self, instance, owner):
...         print 'Enter __get__'
...         return self
...
>>> class Table:
...     id = Column()
...
>>> Table.id
<__main__.Column instance at 0xfff378ec>
>>>
    

以上例子我使用数据库表和列的抽象成类表达，当你执行col = Table.id时候，并没有实现Column的描述器，也就是说并没有调用__get__函数，为什么呢，
我们对Column改造一下：
    
class Column(object):
    def __get__(self, instance, owner):
        print 'Enter __get__'
        return self

>>> class Column(object):
...     def __get__(self, instance, owner):
...         print 'Enter __get__'
...         return self
...
>>> Table.id
<__main__.Column instance at 0xfff378ec>
>>> class Table:
...     id = Column()
...
>>> Table.id
Enter __get__
<__main__.Column object at 0xfff3792c>
>>>

可以看到已经进入到描述器了。

我想以上可以直观见到继承与不继承object的区别，继承object的类可以使用python内部机制处理，如“魔术对象”
