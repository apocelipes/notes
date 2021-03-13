tuple是c++11新增的数据结构，通过tuple我们可以方便地把各种不同类型的数据组合在一起。有了这样的数据结构我们就可以轻松模拟多值返回等技巧了。

tuple和其他的容器不同，标准库没有提供适用于tuple的迭代器，也没有提供tuple类型的迭代接口。所以当我们想要遍历tuple的时候只能自己动手了。

所以这篇文章我们会实现一个简单的接口用来遍历各种tuple，顺便一窥现代c++中的模板元编程。

<blockquote id="bookmark">
  <h4>本文索引</h4>
  <ul>
    <li><a href="#%E6%8E%A5%E5%8F%A3%E8%AE%BE%E8%AE%A1">接口设计</a></li>
    <li>
      <a href="#%E5%AE%9E%E7%8E%B0%E6%8E%A5%E5%8F%A3">实现接口</a>
      <ul>
        <li><a href="#%E5%88%9D%E6%AD%A5%E5%B0%9D%E8%AF%95">初步尝试</a></li>
        <li><a href="#%E9%80%9A%E7%94%A8%E7%9A%84%E5%8F%A4%E5%85%B8%E5%AE%9E%E7%8E%B0">通用的古典实现</a></li>
        <li><a href="#%E4%BD%BF%E7%94%A8%E7%BC%96%E8%AF%91%E6%9C%9F%E6%9D%A1%E4%BB%B6%E5%88%86%E6%94%AF">使用编译期条件分支</a></li>
        <li><a href="#%E5%8F%98%E9%95%BF%E6%A8%A1%E6%9D%BF%E5%8F%82%E6%95%B0%E9%94%99%E8%AF%AF%E7%9A%84%E8%A7%A3%E6%B3%95">变长模板参数——错误的解法</a></li>
        <li><a href="#%E6%8A%98%E5%8F%A0%E8%A1%A8%E8%BE%BE%E5%BC%8F%E4%BD%BF%E7%94%A8%E5%8F%98%E9%95%BF%E6%A8%A1%E6%9D%BF%E5%8F%82%E6%95%B0%E7%9A%84%E6%AD%A3%E7%A1%AE%E8%A7%A3%E6%B3%95">折叠表达式——使用变长模板参数的正确解法</a></li>
        <li><a href="#%E6%80%BB%E7%BB%93">总结</a></li>
      </ul>
    </li>
  </ul>
</blockquote>

## 接口设计

为什么要遍历tuple呢？通常我们确实不需要逐个遍历tuple的数据，通过使用get取出特定位置的元素就满足大部分的应用需求了。

但是，偶尔我们也会想要把某一个泛型算法应用到tuple的每一个成员上，虽然不多见但也确实有需求的场景存在。

然而get需要的索引只能是编译期常量，这导致我们无法依赖现有的循环语句去实现索引的递增，因此只有两条路可供选择：硬编码每一项索引和模板元编程。我是个不喜欢硬编码的人，所以我选择了后者。

把STL里的通用容器算法实现一遍工程量太大了，而且很明显一篇文章也讲不完。我决定实现标准库里的`for_each`，正好也契合今天的主题——遍历tuple。

标准库的`for_each`是这样的`template <class Iterator, class UnaryFunction> void for_each(Iterator first, Iterator last, UnaryFunction f)`，其中`UnaryFunction`的函数签名是`void fun(const Type &a)`。

通过`for_each`我们可以顺序遍历容器中的每一项数据，我们的`for_each_tuple`也将实现类似的功能——顺序遍历tuple的每一项元素。

不过前面已经提到了，tuple是没有迭代器的，因此我们的函数得改个样子：`template <class Tuple, class Functor> void for_each_tuple(const Tuple &, Functor &&)`。因为不能使用迭代器，所以我们传了tuple的引用进函数。

当然，c++17里tuple是constexpr类型，所以你还可以给我们的`for_each`加上constexpr。

## 实现接口

接口设计好了，下面我们就该实现它了。

我会介绍三种实现遍历tuple的方法，以及一种存在缺陷的方法，首先我们从最原始的方案开始。

### 初步尝试

距离c++11发布已经快整整十年了，想必大家也习惯于书写c++11的代码了。不过让我们把时间倒流回c++03时代，那时候既没有constexpr，也没有变长模板参数，相当的原始而蛮荒。

那么问题来了，那时候有tuple吗？当然有，boost里的tuple的历史可长了。

其中的秘诀就在于模板递归，这是一种经典的元编程手段，我们的foreach也需要借助这种技术。

现在我们来看一下不使用编译期计算和变长模板参数的原始方案：

```c++
template <typename Tuple, typename Functor, int Index>
void for_each_tuple_impl(Tuple &&t, Functor &&f)
{
    if (Index >= std::tuple_size<std::remove_reference_t<Tuple>>::value) {
        return;
    } else {
        f(std::get<Index>(t));
        for_each_tuple_impl<Tuple, Functor, Index+1>(std::forward<Tuple>(t), std::forward<Functor>(f));
    }
}

template <typename Tuple, typename Functor>
void for_each_tuple(Tuple &&t, Functor &&f)
{
    for_each_tuple_impl<Tuple, Functor, 0>(std::forward<Tuple>(t), std::forward<Functor>(f));
}
```

我们用`std::remove_reference_t`来把Tuple从引用类型转化为被引用的tuple的类型，原因是模板函数的右值引用参数会自动根据引用折叠的规则转换为左值引用或者右值引用，而我们不能从引用类型调用`std::tuple_size`获取tuple的长度。

整体的思路其实很简单，我们从0开始，每遍历处理完一项就让index+1，然后递归调用impl。如果了最后一个元素+1的位置，函数就返回。这样遍历就结束了。

注意f上的`std::forward`，我们用右值引用的目的是接受包括lambda在内的所有可调用对象，这些对象可以是一个lambda字面量，可以是一个具名的存储了lambda的变量，还以可以是函数指针或者任何重载了`template <typename T> void operator()(const T&)`运算符的类的实例。所以我们很难假设这么广范围内的可调用对象都是可以被复制的，所以保险起见我们使用了模板的右值引参数来将不可以复制的内容用右值引用捕获。当然因为移动语义会产生副作用，这点用户得自己负担，而我们也不得不对f使用`std::forward`进行完美转发。不过这样好处也不是没有，至少我们省去了很多不必要的复制。

然而当你满心欢喜地准备尝试运行自己杰作的时候，编译器给你浇了一头冷水：

```text
...
/usr/include/c++/10.2.0/tuple:1259:12: fatal error: template instantiation depth exceeds maximum of 900 (use '-ftemplate-depth=' to increase the maximum)
 1259 |     struct tuple_element<__i, tuple<_Head, _Tail...> >
      |            ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
compilation terminated.
```

报了一大堆错，甚至超过了屏幕的最大滚动高度（我设置的是10000行）。发生了什么呢？

稍微翻翻报错信息，我们发现了实际上是模板递归超过了允许的最大深度。可是我们不是已经给出了退出递归的条件了吗？

让我再来看看impl的代码：

```c++
template <typename Tuple, typename Functor, int Index>
void for_each_tuple_impl(Tuple &&t, Functor &&f)
{
    if (Index >= std::tuple_size<std::remove_reference_t<Tuple>>::value) {
        return;
    } else {
        f(std::get<Index>(t));
        for_each_tuple_impl<Tuple, Functor, Index+1>(std::forward<Tuple>(t), std::forward<Functor>(f));
    }
}
```

编译器在编译函数的时候是需要把所有条件分支都编译的，所以即使是在函数模板的实例达到退出递归的条件，else分支仍然会被编译，而在这个分支里模板会被不断递归实例化，最终超过允许的最大递归深度。

这里就引出了模板递归的一个重要规则：**我们应该用模板特化或是函数重载来实现递归的终止条件**

然而在这里我们既不是模板的特化也没有调用重载函数。

如果想利用函数重载的话并不现实，因为递归函数调用的形式是相同的，无法针对tuple的最后一个元素进行特殊处理。

而函数模板不支持部分特化，所以我们也很难实现一个针对tuple结尾的特化版本。

那怎么办呢？

### 通用的古典实现

既然函数模板不能满足要求，我们使用类模板不就行了。只要重载了`operator()`，使用起来也没多少区别。

所以一个真正通用的古典实现可以写出下面这样：

```c++
template <typename Tuple, typename Functor, int Start, int End>
struct classic_for_each_tuple_helper
{
    constexpr void operator()(const Tuple &t, Functor &&f)
    {
        f(std::get<Start>(t));
        classic_for_each_tuple_helper<Tuple, Functor, Start + 1, End>{}(t, std::forward<Functor>(f));
    }
};
```

我们首先实现了主模板，其中Start和End是tuple开始和结束的索引。每处理一个元素，我们就让Start加上1。

你可以想一想这里递归的停止条件是什么。

我们每次给Start递增1，那么最后我们的Start一定会等于甚至超过End。没错，这就是我们的停止条件：

```c++
template <typename Tuple, typename Functor, int End>
struct classic_for_each_tuple_helper<Tuple, Functor, End, End>
{
    constexpr void operator()(const Tuple &t, Functor &&f)
    {
        f(std::get<End>(t));
    }
};
```

我们没办法在模板参数列表里判断相等，那么最好的解决办法就是特化出Start和End都一样的特殊情况，这时候用一样的值End同时填入主模板的Start和End就行了。

特化的处理也很简单，我们直接把递归的语句删了就可以了。

想要使用这个帮助模板还需要一点代码，因为我可不想每次手动指定一大长串的tuple类型参数。

正好，利用函数模板我们可以自动进行类型推导：

```c++
template <typename Tuple, typename Functor>
constexpr void classic_for_each_tuple(const Tuple &t, Functor &&f)
{
    classic_for_each_tuple_helper<Tuple, Functor, 0, std::tuple_size_v<Tuple> - 1>{}(t, std::forward<Functor>(f));
}
```

这样我们就可以书写如下的代码了：

```c++
classic_for_each_tuple(std::make_tuple(1, 2, 3, "hello", "world", 3.1415, 2.7183), 
                        [](const auto &element) { /* work */ });
```

即使是`make_tuple`生成的临时对象，我们也可以自动推导出它的类型，所有粗活累活编译器都帮我们代劳了。

不过凡事总是有代价的，有得必有失。表面上我们实现了简单而漂亮的接口，但代价实际上是被转移到了底层：

```bash
$ nm a.out | grep classic_for_each_tuple_helper

00000000000031ae t _ZN29classic_for_each_tuple_helperISt5tupleIJiiiPKcS2_ddEEZ4mainEUlRKT_E2_Li0ELi6EEclERKS3_OS7_
00000000000034c8 t _ZN29classic_for_each_tuple_helperISt5tupleIJiiiPKcS2_ddEEZ4mainEUlRKT_E2_Li1ELi6EEclERKS3_OS7_
00000000000036a2 t _ZN29classic_for_each_tuple_helperISt5tupleIJiiiPKcS2_ddEEZ4mainEUlRKT_E2_Li2ELi6EEclERKS3_OS7_
000000000000391e t _ZN29classic_for_each_tuple_helperISt5tupleIJiiiPKcS2_ddEEZ4mainEUlRKT_E2_Li3ELi6EEclERKS3_OS7_
0000000000003a3e t _ZN29classic_for_each_tuple_helperISt5tupleIJiiiPKcS2_ddEEZ4mainEUlRKT_E2_Li4ELi6EEclERKS3_OS7_
0000000000003b6c t _ZN29classic_for_each_tuple_helperISt5tupleIJiiiPKcS2_ddEEZ4mainEUlRKT_E2_Li5ELi6EEclERKS3_OS7_
0000000000003be6 t _ZN29classic_for_each_tuple_helperISt5tupleIJiiiPKcS2_ddEEZ4mainEUlRKT_E2_Li6ELi6EEclERKS3_OS7_
```

我们的tuple有6个元素，所以我们生成了6个helper的实例。过多的模板实例会导致代码膨胀。

模板递归的另一个缺点是递归的最大深度有限制，在g++10.2上这个限制是900，也就是说超过900个元素的tuple我们是无法处理的，除非用编译器的命令行选项更改这一限制。不过通常也没人会写出有900多个元素的tuple。

虽然有些缺点，还需要工具类模板来实现遍历，但这是旧时代的c++唯一的选择。

### 使用编译期条件分支

好消息是我们可以用c++17了，c++17提供了编译期间计算的条件分支。一般形式如下：

```c++
if constexpr (编译期常量表达式) {
    work 1
} else {
    work 2
}
```

`if constexpr`最大的威力在于如果条件表达式为真，那么else里的语句根本不会被编译，反之亦然。

没错，我们在最开始遇到的问题就是if和else里的语句都会被编译，导致了模板的无限递归，现在我们可以用`if constexpr`解决问题了：

```c++
template <typename Tuple, typename Functor, int Index>
constexpr void for_each_tuple_impl(Tuple &&t, Functor &&f)
{
    if constexpr (Index >= std::tuple_size<std::remove_reference_t<Tuple>>::value) {
        return;
    } else {
        f(std::get<Index>(t));
        for_each_tuple_impl<Tuple, Functor, Index+1>(std::forward<Tuple>(t), std::forward<Functor>(f));
    }
}

template <typename Tuple, typename Functor>
constexpr void for_each_tuple(Tuple &&t, Functor &&f)
{
    for_each_tuple_impl<Tuple, Functor, 0>(std::forward<Tuple>(t), std::forward<Functor>(f));
}
```

这次当遍历完最后一个元素后函数会触发退出递归的条件，if constexpr会帮我们终止模板的递归。问题被干净利落地解决了。

虽然代码被进一步简化了，但是模板递归的两大问题依旧存在。

### 变长模板参数——错误的解法

现代c++有许多简化模板元编程的利器。如果说前面的`if constexpr`是编译期计算和模板不沾边，那下面要介绍的变长模板参数可就是如假包换的模板技巧了。

顾名思义，变长模板参数可以让我们在模板参数上指定任意数量的类型/非类型参数：

```c++
template <typename... Ts>
class tuple;
```

想要处理变长模板参数，在c++17之前还是得靠模板递归。所以我们是不是可以用变长模板参数获取tuple里每一个元素的类型呢？正好get也可以根据元素的类型来获取相应的数据。

于是新的实现产生了：

```c++
template <typename Tuple, typename Functor, typename First, typename... Ts>
constexpr void for_each_tuple2_impl(const Tuple& t, Functor &&f)
{
    f(std::get<First>(t));
    for_each_tuple2_impl<Tuple, Functor, Ts...>(t, std::forward<Functor>(f));
}

template <typename Tuple, typename Functor>
constexpr void for_each_tuple2_impl(const Tuple&, Functor &&)
{
    return;
}

template <typename Functor, typename... Ts>
constexpr void for_each_tuple2(const std::tuple<Ts...> &t, Functor &&f)
{
    for_each_tuple2_impl<std::tuple<Ts...>, Functor, Ts...>(t, std::forward<Functor>(f));
}
```

代码有些复杂，我会逐步讲解。

首先我们有两个`for_each_tuple2_impl`，不过别紧张，因为模板形参不同，所以这是两个不同的模板（函数模板没有部分特化）。又因为变长参数的实参数量可以为0，为了实例化的时候不会产生歧义，只能让第二个`for_each_tuple2_impl`不接受任何额外的模板参数。

接着我们看到`for_each_tuple2`，它的作用很简单，通过参数上的`std::tuple<Ts...>`自动推导出tuple元素的所有类型，然后存放在Ts里。习惯上我们给变长参数包的名字是以s结尾的，象征参数包里可能有不止一个类型参数。

接下来才是重头戏。当我们这样调用`for_each_tuple2_impl<std::tuple<Ts...>, Functor, Ts...>`时，实际上会展开成`for_each_tuple2_impl<std::tuple<Type1, Type2, ..., TypeN>, Functor, Type1, Type2, ..., TypeN.>`。

对应到我们的`template <typename Tuple, typename Functor, typename First, typename... Ts>`, First就会是Type1，而其他剩下来的类型又会被收集到`for_each_tuple2_impl`的Ts里，这样我们就分离出了第一次个tuple里的元素。

然后我们使用`std::get<First>(t)`获取到这个元素，然后递归重复上述步骤。

Ts的第一个参数会被逐个分离，最后一直到Ts和First都为空，这是递归就该结束了，所以我们写出了第二个`for_each_tuple2_impl`模板来处理这一情况。

因为tuple的类型参数列表的顺序和其中包含元素是对应的，所以我们可以实现遍历。

到目前为止我们的for_each工作得很好，然而当我们传入了`std::make_tuple(1,2,3)`，编译器又一次爆炸了。

好奇的你一定又在思考为什么了。不过这回你应该很快就有了头绪，`tuple<int,int,int>`，存在一样的类型参数，这时候`std::get`会不会不知道该获取的是哪个元素呢？

你猜对了，get的文档里是这么说的`Fails to compile unless the tuple has exactly one element of that type.`，意思是当某个类型A出现了不止一次时，使用`get<A>`会导致编译出错。

因此这个方案是无效的，我们不能保证tuple里总是不同类型的数据。

### 折叠表达式——使用变长模板参数的正确解法

不过别气馁，尝试失败也是模板元编程的乐趣之一。更何况现代c++里有相当多的实用工具可以加以利用，比如`integer_sequence`和折叠表达式。

折叠表达式用于按照给定的模式展开变长模板参数包，而`integer_sequence`则可以用来包含0-N的整数类型非类型模板参数，在我上一篇介绍[模板元编程的文章](https://www.cnblogs.com/apocelipes/p/11289840.html)里有介绍，这里不再赘述。

使用`integer_sequence`可以构建一个包含所有tuple元素索引的编译期整数常量序列，配合折叠表达式可以把这些索引展开利用：

```c++
template <typename Tuple, typename Functor, std::size_t... Is>
constexpr void for_each_tuple3_impl(const Tuple &t, Functor &&f, std::index_sequence<Is...>)
{
    // 展开成（f(std::get<0>(t))，f(std::get<1>(t))，...）
    (f(std::get<Is>(t)), ...);
}

template <typename Tuple, typename Functor>
constexpr void for_each_tuple3(const Tuple &t, Functor &&f)
{
    // std::make_index_sequence<std::tuple_size_v<Tuple>>产生一个index_sequence<0,1,2,..,N>
    for_each_tuple3_impl(t, std::forward<Functor>(f), std::make_index_sequence<std::tuple_size_v<Tuple>>());
}
```

这次不再是模板递归了，我们生成了所有元素的索引，然后教编译器硬编码了所有的get操作，形式上不太像但确确实实完成了遍历操作。这简单粗暴，同时也是三种方法中最直观的。

而且这个方案不会产生一大堆的模板实例，生成的二进制文件也是清爽干净的。同时因为不是递归，也不会受到递归深度限制的影响。

这就是现代c++在模板元编程上的威力。

### 总结

我们一共实现了三种遍历tuple的方法，从原始到现代，从复杂到简单。

同时我们还踩掉了一些坑，在今后的开发中只要留意类似的问题也能及时避免了。

当然，我写的方案仍有很大的提升空间，你可以自己进行尝试改进。

不过我最想说的还是现代c++真的极大简化了模板元编程，把模板元编程从一个复杂抽象的黑魔法变成了直观易于理解的开发技巧，应该有更多的人来体验使用现代c++的乐趣，c++已经脱胎换骨了。
