--- 
layout: post
title: 理解python变量和内存管理 
tags:
- python
- memory
---

你是否曾经比较过python 变量和 C 变量有什么异同？例如，当你在C中做一次分配变量空间，实际上创建一块内存,并且你可以把变量的值放到这块内存。

    `int a = 1;`

上面的表达式，你可以想象在成有计算机生成一个盒子a，盒子口是打开着的n，并且可以用来装东西，可以把1[你可以想象成1为要装的东西]装入盒子里面，
盒子能装大小(int)[也可想象成载重量]0xFFFFFFFF ~ 0x0FFFFFFF

![a=1](http://python.net/~goodger/projects/pycon/2007/idiomatic/a1box.png)

当你执行

    `a = 2;`

相当于你把2这个东西装进了a这个箱子，原先箱子里面的东西[这里指1]就没有了。

![a=2](http://python.net/~goodger/projects/pycon/2007/idiomatic/a2box.png)       

    `int b = a`

相当于有一个盒子b，现在把一个跟a里面一样的东西放到b盒子中，注意并不是从a中拿出来放到b盒子中。

以上是C语言中的变量的理解, C语言中变量你可以更多地想象成盒子。


在Python中变量更像是为一个东西打上标签，当你在Python声明一个变量的时候，随之Python给这个变量打上了标签。

    `a = 1`

以上表达式只不过是给1打上一个标签，这个标签名为：a

![tag=1](http://python.net/~goodger/projects/pycon/2007/idiomatic/a1tag.png)

当你改变一个变量的值的时候,Python 只不过是把原来的值（东西）从标签中摘走。

    `a = 2`

![tag=2](http://python.net/~goodger/projects/pycon/2007/idiomatic/a2tag.png)
![tag=2](http://python.net/~goodger/projects/pycon/2007/idiomatic/1.png)

如上图，把1这个东西从标签a中摘走，然后把标签a套入2这个东西，一点像 ”冠名“的意思，这知道这个词好不好理解。

    `b = a`

![tag=b](http://python.net/~goodger/projects/pycon/2007/idiomatic/ab2tag.png)

这里可以理解为：（1）找到标签为a的东西 （2）再在这个东西上套上b这个标签。
所以这个东西就有两个标签了。   

####浅淡Python的内存管理

从上面可以看到，对相同变量值进行赋值的时候，在内存中只有一次拷贝，例如：a,b,c的值都等于10的时候，在内存中并不是占用3块内存存放10这个值，而只有占用了一块
内存存放10，当你更新a的值时，例如a=11,此时另外分配一块新的内存放11这个变量。

你亦可以尝试做下一下实验在python解释器中输入一下：

    `>>> a_list = [1] * 10
     >>> a_list
     [1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
     >>> [id(value) for value in a_list]
     [2147598144L, 2147598144L, 2147598144L, 2147598144L, 2147598144L, 2147598144L, 2147598144L, 2147598144L, 2147598144L, 2147598144L]`

你可以看到列表里面每一个值地址都是一样的(2147598144L)

    `>>> a_foo = foo()
     >>> b_foo = foo()
     >>> id(b_foo)
     4294145420L
     >>> id(a_foo)
     4294145484L
     >>> b_list = [1, 2, 3, 5, 4]
     >>> a_list = [1, 2, 3, 4, 5]
     >>> id(b_list)
     4294488428L
     >>> id(a_list)
     4294145068L
     >>> for item in b_list:
     ...     print "b_list's item " + str(item) + ' ' + str(id(item))

     b_list's item 1 2147598144
     b_list's item 2 2147598132
     b_list's item 3 2147598120
     b_list's item 5 2147598096
     b_list's item 4 2147598108

     >>> for item in a_list:
     ...     print "a_list's item " + str(item) + ' ' + str(id(item))
     ...
     a_list's item 1 2147598144
     a_list's item 2 2147598132
     a_list's item 3 2147598120
     a_list's item 4 2147598108
     a_list's item 5 2147598096`

以上可以看出a_foo,b_foo 对象实例地址是不一样的，虽然他们是相同的对象。这是跟前面有所不同的，当你创建相同对象的实例的时候,没有对象都有独立的地址,除非他们使用的是单例，
immutable类型的对象例如： str, int, float 将是同一个地址。

以上译文出处：[Understanding Python variables and Memory Management](http://foobarnbaz.com/2012/07/08/understanding-python-variables/)
