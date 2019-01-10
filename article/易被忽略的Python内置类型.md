Python中的内置类型是我们开发中最常见的，很多人都能熟练的使用它们。

然而有一些内置类型确实不那么常见的，或者说往往会被我们忽略，所以这次的主题就是带领大家重新认识这些“不同寻常”的内置类型。
（注意：本文基于python3，不会包含任何python2相关内容）

## frozenset
不可变集合（frozenset）与普通的set一样，只不过它的元素是不可变的，因此诸如`add`，`remove`，`update`等可以添加/删除/改变集合内元素的方法是不存在的，换句话说一旦frozenset建立后你将不再可能更改集合内的元素。其他的方法与set一致：
```python
>>> frozen = frozenset([1, 1, 2, 3, 4, 5, 6, 6])
frozenset({1, 2, 3, 4, 5, 6})
>>> frozen | {1, 2, 3, 7, 8}
frozenset({1, 2, 3, 4, 5, 6, 7, 8})
>>> frozen ^ {1, 2, 3, 7, 8}
frozenset({4, 5, 6, 7, 8})
```

## range
`range`事实上相当得常见，所以你也许会奇怪我为什么把它列出来。

其实原因很简单，因为大部分人熟悉`range`的使用，但并不清楚range到底是什么。返回迭代器？返回一个可迭代对象？`range`本身又是什么呢？

答案揭晓：
```python
>>> range
<class 'range'>
```
是的，`range`是个class！所以当我们使用`for i in range(1, 10)`这样的代码时，实际上我们遍历了一个`range`对象，而`range`也实现了可迭代对象需要的`__iter__`魔法方法，所以它自身是可迭代对象：
```python
>>> range.__iter__
<slot wrapper '__iter__' of 'range' objects>
```
因此，`range`既不返回迭代器，也不返回其他可迭代对象，而是返回的自己。

## bytearray
`bytearray`一般情况下并不常见，它主要为了可以实现原地修改bytes对象而出现，因为bytes和str一样是不可变对象，例如这样是非法的：
```python
>>> b = '测试用例a'.encode('utf8')
>>> b[-1] = 98 # change 'a' -> 'b'
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: 'bytes' object does not support item assignment
```
而当我们把bytes的内容复制给`bytearray`时就可以进行原地修改了：
```python
>>> array = bytearray(b)
>>> array[-1] = 98
>>> array.decode('utf8')
测试用例b
```
`bytearray`对象没有字面常量，因此只能通过构造函数创建，它有着和bytes一样的方法，只是可变以及多了一些序列对象的特性。如果要创建一个`bytearray`可以有如下的几种方法：
- `bytearray()`返回一个空的`bytearray`对象
- `bytearray(10)`创建一个长度为10且内容被0填充的`bytearray`
- `bytearray(iterable)`会将可迭代对象的内容转换成bytes然后存入对象中
- `bytearray(b'Hi!')`将已有的二进制数据复制进对象

另外`bytearray`还提供了`fromhex`和`hex`方便将数据以16进制的形式输入输出：
```python
>>> array.hex()
'e6b58be8af95e794a8e4be8b62'
>>> bytearray().fromhex('e6b58be8af95e794a8e4be8b62').decode('utf8')
'测试用例b'
```

## memoryview
`memoryview`提供了直接访问对象内存的机制，只要目标对象支持[buffer protocol](https://docs.python.org/3/c-api/buffer.html#bufferobjects)，例如`bytes`和`bytearray`。

`memoryview`有个称为“元素”的概念，也就是对象规定的最小的内存单元，比如`bytes`和`bytearray`的最小内存单元就是一个byte，具体取决于对象的实现。

`len(view)`通常等于`len(view.tolist())`，也就是等于view的“元素”数量。如果`view.ndim == 0`，那么整个view的内存会被视作一个整体，len会返回1，如果`view.ndim == 1`那么就正常返回“元素”的个数。`view.itemsize`会返回单个“元素”的大小。单位是byte。

`view.readonly`表示当前的`memoryview`是否是只读的，例如`bytes`对象的view就是只读的，`view.readonly`的值为`True`。是否只读取决于被引用的对象是否可变以及对buffer protocol的实现。

对于使用完毕的`memoryview`应该尽快调用其`release()`方法释放资源，而且部分对象在被view引用时会自动进行一些限制，比如`bytearray`会禁止调整大小，及时释放view是资源可以解除这些限制。

结合示例可以更清晰地了解这些特性：
```python
>>> data = bytearray(b'abcefg')
>>> v = memoryview(data)
>>> v.readonly
False
>>> v[0] = ord(b'z')
>>> data
bytearray(b'zbcefg')
>>> v[1:4] = b'123'
>>> data
bytearray(b'z123fg')
>>> v[2:3] = b'spam'
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ValueError: memoryview assignment: lvalue and rvalue have different structures
>>> v[2:6] = b'spam'
>>> data
bytearray(b'z1spam')
```

## dict-views
准确的说，这不是一种类型，而是一种概念。然而typing里仍然将其视为一种类型，所以也就罗列在此了。

概念：`返回自dict.keys()`,`dict.values`()和`dict.items()`的对象被称作`dict-views`。

对于views对象，可以使用len，成员检测，它本身也是可迭代对象：
```python
>>> dishes = {'eggs': 2, 'sausage': 1, 'bacon': 1, 'spam': 500}
>>> keys = dishes.keys()
>>> values = dishes.values()

>>> # iteration
>>> n = 0
>>> for val in values:
...     n += val
>>> print(n)
504

>>> # keys and values are iterated over in the same order (insertion order)
>>> list(keys)
['eggs', 'sausage', 'bacon', 'spam']
>>> list(values)
[2, 1, 1, 500]

>>> # view objects are dynamic and reflect dict changes
>>> del dishes['eggs']
>>> del dishes['sausage']
>>> list(keys)
['bacon', 'spam']

>>> # set operations
>>> keys & {'eggs', 'bacon', 'salad'}
{'bacon'}
>>> keys ^ {'sausage', 'juice'}
{'juice', 'sausage', 'bacon', 'spam'}
```
从例子中可以看出，views保持着元素的插入顺序（插入顺序的保证从python3.6开始）以及views动态反应了key/value的插入和删除以及修改，因此在某些场景下views对象是相当有用的。

## The Ellipsis Object (...)
`...`不是一个类型，不过算是一个内置对象。

它没什么特殊的含义，仅表示省略，通常被用在type hints中：
```python
>>> ...
Ellipsis
>>> from typing import Callable
>>> func: Callable[..., None] = lambda x,y:print(x*y)
```
func是一个没有返回值的函数，参数列表没有做任何限制。

你也可以写成`Ellipsis`，两者是等价的，不过显然是`...`这种形式更简单明了。

以上就是这些容易被忽略和遗忘的内置类型，如有错误和疏漏欢迎指出。

#### 参考：
https://docs.python.org/3/library/stdtypes.html

https://docs.python.org/3/c-api/buffer.html#bufferobjects
