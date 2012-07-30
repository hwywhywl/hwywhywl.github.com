--- 
layout: post
title: python 如何判断生成器类型
tags:
- python
---

头疼如何判断生成器变量，例如定义如下生成器

<pre><code>

def _generator():
    yield 1

gen = _generator()

>>> type(gen)
<type 'generator'>

>>> isinstance(gen, generator)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
NameError: name 'generator' is not defined

>>> dir(types)
['BooleanType', 'BufferType', 'BuiltinFunctionType', 'BuiltinMethodType', 'ClassType', 'CodeType', 'ComplexType', 'DictProxyType', 'DictType', 'DictionaryType', 'EllipsisType', 'FileType', 'FloatType', 'FrameType', 'FunctionType', 'GeneratorType', 'GetSetDescriptorType', 'InstanceType', 'IntType', 'LambdaType', 'ListType', 'LongType', 'MemberDescriptorType', 'MethodType', 'ModuleType', 'NoneType', 'NotImplementedType', 'ObjectType', 'SliceType', 'StringType', 'StringTypes', 'TracebackType', 'TupleType', 'TypeType', 'UnboundMethodType', 'UnicodeType', 'XRangeType', '__builtins__', '__doc__', '__file__', '__name__', '__package__']

</code></pre>

发现types 里面有一个GeneratorType，遂尝试

<pre><code>

>>> isinstance(gen, types.GeneratorType)
True

</code></pre>

其实还可以用inspect.isgenerator

详细可查考stackoverflow ： [How to check if an object is a generator object in python?](http://stackoverflow.com/questions/6416538/how-to-check-if-an-object-is-a-generator-object-in-python)
