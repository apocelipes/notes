本文将带你走进python3.7的新特性dataclass，通过本文你将学会dataclass的使用并避免踏入某些陷阱。

<blockquote id="bookmark">
  <ul>
    <li><a href="#abstract">dataclass简介</a></li>
    <li><a href="#using">dataclass的使用</a>
      <ul>
        <li><a href="#using-define">定义一个dataclass</a></li>
        <li><a href="#using-decorator">深入dataclass装饰器</a></li>
        <li><a href="#using-field">数据类的基石——dataclasses.field</a></li>
        <li><a href="#using-funcs">一些常用函数</a></li>
        <li><a href="#using-inheritance">dataclass继承</a></li>
      </ul>
    </li>
    <li><a href="#summary">总结</a></li>
  </ul>
</blockquote>

<h2 id="abstract">dataclass简介</h2>
dataclass的定义位于[PEP-557](https://www.python.org/dev/peps/pep-0557/)，根据定义一个dataclass是指“一个带有默认值的可变的namedtuple”，广义的定义就是有一个类，它的属性均可公开访问，可以带有默认值并能被修改，而且类中含有与这些属性相关的类方法，那么这个类就可以称为dataclass，再通俗点讲，dataclass就是一个含有数据及操作数据方法的容器。

乍一看可能会觉得这个概念不就是普通的class么，然而还是有几处不同：
1. 相比普通class，dataclass通常不包含私有属性，数据可以直接访问
2. dataclass的repr方法通常有固定格式，会打印出类型名以及属性名和它的值
3. dataclass拥有`__eq__`和`__hash__`魔法方法
4. dataclass有着模式单一固定的初始化方式，或是需要重载运算符，而普通class通常无需这些工作

基于上述原因，通常自己实现一个dataclass是繁琐而无聊的，而dataclass单一固定的行为正适合程序为我们自动生成，于是`dataclasses`模块诞生了。

配合类型注解语法，我们可以轻松生成一个实现了`__init__`，`__repr__`，`__cmp__`等方法的dataclass：
```python
from dataclasses import dataclass

@dataclass
class InventoryItem:
    '''Class for keeping track of an item in inventory.'''
    name: str
    unit_price: float
    quantity_on_hand: int = 0

    def total_cost(self) -> float:
        return self.unit_price * self.quantity_on_hand
```
同时使用dataclass也有一些好处，它比namedtuple更灵活。同时因为它是一个常规的类，所以你可以享受继承带来的便利。

<h2 id="using">dataclass的使用</h2>
我们分x步介绍dataclass的使用，首先是如何定义一个dataclass。

<h3 id="using-define">定义一个dataclass</h3>
`dataclasses`模块提供了一个装饰器帮助我们定义自己的数据类：
```python
@dataclass
class Lang:
    """a dataclass that describes a programming language"""
    name: str = 'python'
    strong_type: bool = True
    static_type: bool = False
    age: int = 28
```
我们定义了一个描述某种程序语言特性的数据类——`Lang`，在接下来的例子中我们都会用到这个类。

在数据类被定义后，会根据给出的类型注解生成一个如下的初始函数：
```python
def __init__(self, name: str='python',
            strong_type: bool=True,
            static_type: bool=False,
            age: int=28):
    self.name = name
    self.strong_type = strong_type
    self.static_type = static_type
    self.age = age
```
可以看到初始化操作都已经自动生成了，让我们试用一下：
```python
>>> Lang()
Lang(name='python', strong_type=True, static_type=False, age=28)
>>> Lang('js', False, False, 23)
Lang(name='js', strong_type=False, static_type=False, age=23)
>>> Lang('js', False, False, 23) == Lang()
False
>>> Lang('python', True, False, 28) == Lang()
True
```
例子中可以看出`__repr__`和`__eq__`方法也已经为我们生成了，如果没有其他特殊要求的话这个dataclass已经具备了投入生产环境的能力，是不是很神奇？

<h3 id='using-decorator'>深入dataclass装饰器</h3>
dataclass的魔力源泉都在`dataclass`这个装饰器中，如果想要完全掌控dataclass的话那么它是你必须了解的内容。

装饰器的原型如下：
```python
dataclasses.dataclass(*, init=True, repr=True, eq=True, order=False, unsafe_hash=False, frozen=False)
```
`dataclass`装饰器将根据类属性生成数据类和数据类需要的方法。

我们的关注点集中在它的`kwargs`上：

| key | 含义 |
| ------ | ------ |
| init | 指定是否自动生成`__init__`，如果已经有定义同名方法则忽略这个值，也就是指定为True也不会自动生成 |
| repr | 同init，指定是否自动生成`__repr__`；自动生成的打印格式为`class_name(arrt1:value1, attr2:value2, ...)` |
| eq | 同init，指定是否生成`__eq__`；自动生成的方法将按属性在类内定义时的顺序逐个比较，全部的值相同才会返回True |
| order | 自动生成`__lt__`，`__le__`，`__gt__`，`__ge__`，比较方式与eq相同；如果order指定为True而eq指定为False，将引发`ValueError`；如果已经定义同名函数，将引发`TypeError` |
| unsafehash | 如果是False，将根据eq和frozen参数来生成`__hash__`:<br/> 1. eq和frozen都为True，`__hash__`将会生成<br/>2. eq为True而frozen为False，`__hash__`被设为`None`<br/>3. eq为False，frozen为True，`__hash__`将使用超类（object）的同名属性（通常就是基于对象id的hash）<br/>当设置为True时将会根据类属性自动生成`__hash__`，然而这是不安全的，因为这些属性是默认可变的，这会导致hash的不一致，所以除非能保证对象属性不可随意改变，否则应该谨慎地设置该参数为True |
| frozen | 设为True时对field赋值将会引发错误，对象将是不可变的，如果已经定义了`__setattr__`和`__delattr__`将会引发`TypeError` |

有默认值的属性必须定义在没有默认值的属性之后，和对kw参数的要求一样。

上面我们偶尔提到了field的概念，我们所说的数据类属性，数据属性实际上都是被field的对象，它代表着一个数据的实体和它的元信息，下面我们了解一下`dataclasses.field`。

<h3 id="using-field">数据类的基石——dataclasses.field</h3>
先看下field的原型：
```python
dataclasses.field(*, default=MISSING, default_factory=MISSING, repr=True, hash=None, init=True, compare=True, metadata=None)
```
通常我们无需直接使用，装饰器会根据我们给出的类型注解自动生成field，但有时候我们也需要定制这一过程，这时`dataclasses.field`就显得格外有用了。

default和default_factory参数将会影响默认值的产生，它们的默认值都是None，意思是调用时如果为指定则产生一个为None的值。其中default是field的默认值，而default_factory控制如何产生值，它接收一个无参数或者全是默认参数的`callable`对象，然后用调用这个对象获得field的初始值，之后再将default（如果值不是MISSING）复制给`callable`返回的这个对象。

举个例子，对于list，当复制它时只是复制了一份引用，所以像dataclass里那样直接复制给实例的做法的危险而错误的，为了保证使用list时的安全性，应该这样做：
```python
@dataclass
class C:
    mylist: List[int] = field(default_factory=list)
```
当初始化`C`的实例时就会调用`list()`而不是直接复制一份list的引用：
```python
>>> c1 = C()
>>> c1.mylist += [1,2,3]
>>> c1.mylist
[1, 2, 3]
>>> c2 = C()
>>> c2.mylist
[]
```
数据污染得到了避免。

init参数如果设置为False，表示不为这个field生成初始化操作，dataclass提供了hook——`__post_init__`供我们利用这一特性：
```python
@dataclass
class C:
    a: int
    b: int
    c: int = field(init=False)

    def __post_init__(self):
        self.c = self.a + self.b
```
`__post_init__`在`__init__`后被调用，我们可以在这里初始化那些需要前置条件的field。

repr参数表示该field是否被包含进repr的输出，compare和hash参数表示field是否参与比较和计算hash值。metadata不被dataclass自身使用，通常让第三方组件从中获取某些元信息时才使用，所以我们不需要使用这一参数。

如果指定一个field的类型注解为`dataclasses.InitVar`，那么这个field将只会在初始化过程中（`__init__`和`__post_init__`）可以被使用，当初始化完成后访问该field会返回一个`dataclasses.Field`对象而不是field原本的值，也就是该field不再是一个可访问的数据对象。举个例子，比如一个由数据库对象，它只需要在初始化的过程中被访问：
```python
@dataclass
class C:
    i: int
    j: int = None
    database: InitVar[DatabaseType] = None

    def __post_init__(self, database):
        if self.j is None and database is not None:
            self.j = database.lookup('j')

c = C(10, database=my_database)
```
这个例子中会返回`c.i`和`c.j`的数据，但是不会返回`c.database`的。

<h3 id="using-funcs">一些常用函数</h3>
`dataclasses`模块中提供了一些常用函数供我们处理数据类。

使用`dataclasses.asdict`和`dataclasses.astuple`我们可以把数据类实例中的数据转换成字典或者元组：
```python
>>> from dataclasses import asdict, astuple
>>> asdict(Lang())
{'name': 'python', 'strong_type': True, 'static_type': False, 'age': 28}
>>> astuple(Lang())
('python', True, False, 28)
```

使用`dataclasses.is_dataclass`可以判断一个类或实例对象是否是数据类：
```python
>>> from dataclasses import is_dataclass
>>> is_dataclass(Lang)
True
>>> is_dataclass(Lang())
True
```

<h3 id="using-inheritance">dataclass继承</h3>
python3.7引入dataclass的一大原因就在于相比namedtuple，dataclass可以享受继承带来的便利。

`dataclass`装饰器会检查当前class的所有基类，如果发现一个dataclass，就会把它的字段按顺序添加进当前的class，随后再处理当前class的field。所有生成的方法也将按照这一过程处理，因此如果子类中的field与基类同名，那么子类将会无条件覆盖基类。子类将会根据所有的field重新生成一个初始化函数，并在其中初始化基类。

看个例子：
```python
@dataclass
class Python(Lang):
    tab_size: int = 4
    is_script: bool = True

>>> Python()
Python(name='python', strong_type=True, static_type=False, age=28, tab_size=4, is_script=True)

@dataclass
class Base:
    x: float = 25.0
    y: int = 0

@dataclass
class C(Base):
    z: int = 10
    x: int = 15

>>> C()
C(x=15, y=0, z=10)
```
`Lang`的field被`Python`继承了，而`C`中的`x`则覆盖了`Base`中的定义。

没错，数据类的继承就是这么简单。

<h2 id="summary">总结</h2>
合理使用dataclass将会大大减轻开发中的负担，将我们从大量的重复劳动中解放出来，这既是dataclass的魅力，不过魅力的背后也总是有陷阱相伴，最后我想提几点注意事项：
- dataclass通常情况下是unhashable的，因为默认生成的`__hash__`是`None`，所以不能用来做字典的key，如果有这种需求，那么应该指定你的数据类为frozen dataclass
- 小心当你定义了和`dataclass`生成的同名方法时会引发的问题
- 当使用可变类型（如list）时，应该考虑使用`field`的`default_factory`
- 数据类的属性都是公开的，如果你有属性只需要初始化是使用而不需要在其他时候被访问，请使用`dataclasses.InitVar`

只要避开这些陷阱，dataclass一定能成为提高生产力的利器。

##### 参考

[https://docs.python.org/3.7/library/dataclasses.html](https://docs.python.org/3.7/library/dataclasses.html)

[https://www.python.org/dev/peps/pep-0557](https://www.python.org/dev/peps/pep-0557)
