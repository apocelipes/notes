两个东西名字很像，但没有任何关系，使用场景也不一样。

## NotImplemented

NotImplemented不是异常，不能直接抛出和捕获，否则会得到报错：`TypeError: exceptions must derive from BaseException`。

用途：二元运算符在不支持对参数进行运算时需要返回NotImplemented，解释器遇到这个返回值时会尝试进行fallback：

```python
a: A = A()
b: B = B()

def f(a, b):
    return a == b
```

这个表达式实际运行的时候是这样的：

```python
if (result := a.__eq__(b)) is NotImplemented:
    return b.__eq__(a)
else:
    return result
```

不同运算符有不同转换方式。

这个值是单例。这个值不能参与bool运算或是转换成bool值。

应用：自定义运算符时；`functools.total_ordering`。

## NotImplementedError

是异常，是RuntimeError的子类型。

用途：抽象类的方法里返回这个；还未完工的类的方法或者函数里可以抛出这个异常。

不应该使用的情况：某个方法或者函数不应该被使用的时候不应该抛出这个，应该直接删除方法或者函数，或者不能删除时设置为None，或者加上废弃警告：

```python
def create_app(self, *args, **kwargs):
    warnings.warn("create_app() is deprecated; use __call__().", warnings.DeprecationWarning)
    return self(*args,**kwargs) 
```
