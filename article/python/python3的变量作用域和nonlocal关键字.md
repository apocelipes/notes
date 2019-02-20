也许你已经觉得自己可以熟练使用python并能胜任许多开发任务，所以这篇文章是在浪费你的时间。不过别着急，我们先从一个例子开始：
```python
i = 0
def f():
  print(i)
  i += 1
  print(i)

f()
print(i)
```
猜猜看输出是什么？你会说不就是0，1，1么，真的是这样吗？
```bash
> python test.py
Traceback (most recent call last):
  File "a.py", line 7, in <module>
    f()
  File "a.py", line 3, in f
    print(i)
UnboundLocalError: local variable 'i' referenced before assignment
```
这是为什么？如果你还不清楚产生错误的原因，那就请继续往下阅读吧！

<blockquote id="bookmark">
  <h4>本文索引</h4>
  <ul>
    <li>
      <a href="#legb原则">LEGB原则</a>
    </li>
    <li>
      <a href="#名字隐藏和暂时性死区">名字隐藏和暂时性死区</a>
    </li>
    <li>
      <a href="#消除暂时性死区">消除暂时性死区</a>
    </li>
    <li>
      <a href="#使用nonlocal声明闭包作用域变量">使用nonlocal声明闭包作用域变量</a>
    </li>
    <li>
      <a href="#总结">总结</a>
    </li>
  </ul>
</blockquote>

## LEGB原则
变量的作用域，这是一个老生常谈的问题了。

在python中作用域规则可以简单的归纳为`LEGB原则`，也就是说，对于一个变量`name`，首先会从当前的作用域开始查找，如果它不在函数里那就从global开始，没找到就查找builtin作用域，如果它位于函数中，就先从local作用域查找，接着如果当前的函数是一个闭包，那么就查找外层闭包的作用域，也就是规则中的`E`，接着是global和builtin，如果都没找到`name`这个变量，则抛出`NameError`。

那么我们来看一段代码：
```python
i = 100
def f():
  print(i)
```
在这段代码中，print位于builtin作用域，i位于global，那么：
1. 在函数f中找不到这两个名字，所以从local向上查找，
2. 首先f不是闭包，因此跳过闭包作用域的查找，
3. 然后查找global，找到了i，但print还未找到，
4. 然后查找builtin，找到了print的builtin模块里的一个函数。

至此名字查找结束，调用找到的函数，输出结果100。

现在你可能更加疑惑了，既然查找规则按照`LEGB`的方向进行，那么test.py中的f不就应该找到i为global中的变量吗，为什么会报错呢？

## 名字隐藏和暂时性死区
在揭晓答案之前，我们先复习一下名字隐藏。

它是指一个声明在局部作用中的名字会隐藏外层作用域中的同名的对象。许多语言都遵守这一特性，python也不例外。

那么暂时性死区是什么呢？这是es6的一个概念，当你在局部作用域中定义了一个非全局的名字时，这个名字会绑定在当前作用域中，并将外部作用域的同名对象隐藏：
```js
var i = 'hello'
function f() {
  i = 'world'
  let i
}
```
这段代码中函数中的i被绑定在局部作用域（也就是函数体内）中，在绑定的作用域中可见，并将外部的名字隐藏，而对一个未声明的局部变量赋值会导致错误，所以上面的代码会引发`ReferenceError: i is not defined`。

对于python来说也是一样的问题，python代码在执行前首先会被编译成字节码，这就会导致某些时候实际执行的程序会和我们看到的产生出入。不过我们有`dis`模块帮忙，它可以输出python对象的字节码，下面我们就来看下经过编译后的`f`：
```text
> dis(f)

2           0 LOAD_GLOBAL              0 (print)
            2 LOAD_FAST                0 (i)
            4 CALL_FUNCTION            1
            6 POP_TOP

3           8 LOAD_CONST               1 ('a')
           10 STORE_FAST               0 (i)

4          12 LOAD_GLOBAL              0 (print)
           14 LOAD_FAST                0 (i)
           16 CALL_FUNCTION            1
           18 POP_TOP
           20 LOAD_CONST               0 (None)
           22 RETURN_VALUE
```
字节码的解释在[这里](https://docs.python.org/3/library/dis.html#python-bytecode-instructions)。

其中`LOAD_FAST`和`STORE_FAST`是读取和存储local作用域的变量，我们可以看到，i变成了局部作用域的变量！而对i的赋值早于i的定义，所以报错了。

产生这种现象的原因也很简单，python对函数的代码是独立编译的，如果未加说明而在函数内对一个变量赋值，那么就认为你定义了一个局部变量，从而把外部的同名对象屏蔽了。这么做无可厚非，毕竟python没有独立的声明一个局部变量的语法，但结果就会造成我们看到的类似暂时性死区的现象。所以请允许我把es6的概念套用在python身上。

## 消除暂时性死区
既然知道问题的症结在于python无法区分局部变量的声明和定义，那么我们就来解决它。

对于一个可以区分声明和定义的语言来说是没有这种烦恼的，比如c：
```c
int i = 0;
void f(void)
{
  i++;
  printf("%d\n", i); // 1
  const char *i = "hello";
  printf("%s\n", i); // "hello"
}
```
python中不能这么做，但是我们可以换一个思路，声明一个变量是全局作用域的，这样不就解决了吗？

`global`运算符就是为了这个目的而存在的，它声明一个变量始终是全局作用域的变量，因此只要存在global声明，那么当前作用域里的这个名字就是一个对同名全局变量的引用。改进后的函数如下：
```python
def f():
  global i
  print(i)
  i += 1
  print(i)
```
现在运行程序就会是你想要的结果了：
```bash
> python test.py
0
1
1
```
如果你还是不放心，那么我们再来看看字节码：
```text
> dis(f)

3           0 LOAD_GLOBAL              0 (print)
            2 LOAD_GLOBAL              1 (i)
            4 CALL_FUNCTION            1
            6 POP_TOP

4           8 LOAD_CONST               1 ('a')
           10 STORE_GLOBAL             1 (i)

5          12 LOAD_GLOBAL              0 (print)
           14 LOAD_GLOBAL              1 (i)
           16 CALL_FUNCTION            1
           18 POP_TOP
           20 LOAD_CONST               0 (None)
           22 RETURN_VALUE
```
对于i的存取已经由`LOAD_GLOBAL`和`STORE_GLOBAL`接手了，没问题。

当然`global`也有它的局限性：
- 一旦声明global，那么这个名字始终是global作用域的一个变量，不可以再是局部变量
- 名字必须存在于global里，因为python在运行时进行名字查找，所以你的变量在global里找不到的话对它的引用将会出错
- 接上一条，因为global限定了名字查找的范围，所以像闭包作用域的变量就找不到了

事实上需要引用非global名字的需求是极其常见的，因此为了解决global的不足，python3引入了`nonlocal`

## 使用nonlocal声明闭包作用域变量
假设我们有一个需求，一个函数需要知道自己被调用了多少次，最简单的实现就是使用闭包：
```python
def closure():
  count = 0
  def func():
    # other code
    count += 1
    print(f'I have be called {count} times')

  return func
```
还是老问题，这样写对吗？

答案是不对，你又制造暂时性死区啦！
```python
>>> f=closure()
>>> f()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 5, in func
UnboundLocalError: local variable 'count' referenced before assignment
```
这时候就要`nonlocal`出场了，它声明一个名字位于闭包作用域中，如果闭包作用域中未找到就报错。

所以修正后的函数如下：
```python
def closure():
  count = 0
  def func():
    # other code
    nonlocal count
    count += 1
    print(f'I have be called {count} times')

  return func
```
测试一下：
```python
>>> f=closure()
>>> f()
I have be called 1 times
>>> f()
I have be called 2 times
>>> f()
I have be called 3 times
>>> f2=closure()
>>> f2()
I have be called 1 times
```
现在可以正常使用和修改闭包作用域的变量了。

## 总结
当然，在函数里修改外部变量往往会导致潜在的缺陷，但有时这样做又是对的，所以希望你在好好了解作用域规则的前提下合理地利用它们。

作用域规则可以总结为下：
1. 名字查找按照LEGB规则进行，如果当前代码在global中则从global作用域开始查找，否则从local开始
2. builtin作用域中是内置类型和函数，所以它们总是能被找到，前提是不要在局部作用域中对它们赋值
3. global中存放着所有定义在当前模块和导入的名字
4. local是局部作用域，存放在形成局部作用于的代码中有赋值行为的名字
5. 闭包作用域是闭包函数的外层作用域，里面可以存放一些自定义的状态
6. global声明一个名字在global作用域中
7. nonlocal声明一个名字在闭包作用域中
8. 最重要的一条，当你在能产生局部作用域的代码中对一个名字进行赋值，那么这个名字就会被认为是一个local作用域的变量从而屏蔽其他作用域中的同名对象

只要记住这些规则你就可以和因作用域引起的各种问题说再见了。而且理解了这些规则还会为你探索更深层次的python打下坚实的基础，所以请将它牢记于心。
