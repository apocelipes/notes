最近在重温《c++程序设计新思维》这本经典著作，感慨颇多。由于成书较早，书中很多元编程的例子使用c++98实现的。而如今c++20即将带着concept，Ranges等新特性一同到来，不得不说光阴荏苒。在c++11之后，得益于新标准很多元编程的复杂技巧能被简化了，STL也提供了诸如`<type_traits>`这样的基础设施，c++14更是大幅度扩展了编译期计算的适用面，这些都对元编程产生了不小的影响。今天我将使用书中最简单也就是最基础的元容器`TypeList`来初步介绍现代c++在元编程领域的魅力。

<blockquote id="bookmark">
  <h4>本文索引</h4>
  <ul>
    <li><a href="#什么是typelist">什么是TypeList</a></li>
    <li><a href="#typelist的定义">TypeList的定义</a></li>
    <li>
      <a href="#元函数的实现">元函数的实现</a>
      <ul>
        <li><a href="#length元函数求list长度">Length元函数求list长度</a></li>
        <li><a href="#typeat获取索引位置上的类型">TypeAt获取索引位置上的类型</a></li>
        <li><a href="#indexof获得指定类型在list中的索引">IndexOf获得指定类型在list中的索引</a></li>
        <li><a href="#append为typelist添加元素">Append为TypeList添加元素</a></li>
        <li><a href="#erase和eraseall删除元素">Erase和EraseAll删除元素</a></li>
        <li><a href="#noduplicates去除所有重复type">NoDuplicates去除所有重复type</a></li>
        <li><a href="#replace和replaceall">Replace和ReplaceAll</a></li>
        <li><a href="#derived2front将派生类型移动至list前部">Derived2Front将派生类型移动至list前部</a></li>
        <li><a href="#元函数实现总结">元函数实现总结</a></li>
      </ul>
    </li>
    <li>
      <a href="#示例">示例</a>
      <ul>
        <li><a href="#自制tuple">自制tuple</a></li>
        <li><a href="#简化工厂模式">简化工厂模式</a></li>
      </ul>
    </li>
    <li><a href="#总结">总结</a></li>
  </ul>
</blockquote>

## 什么是TypeList

TypeList顾名思义，是一个存储和操作type的list，你没有看错，存储的是type（类型信息）而不是data。

这些被存储的type也被称为元数据，存储的它们的TypeList也被称为元容器。

那么，我们存储了这些元数据有什么用呢？答案是用处很多，比如tuple，工厂模式，这两个后面会举例；还可用来实现CRTP技巧（一种元编程技巧），线性化继承结构等，这些也在原书中有详细的演示。

不过光看我在上面的解释多半是理解不了什么是TypeList以及它有什么用的，不过没关系，元编程本身就是高度抽象的脑力活动，只有多读代码勤思考才能有所收获。下面我就展示如何使用现代c++实现一个TypeList，以及对c++11以前的古典版本做些简单的对比。

## TypeList的定义

最初的问题是我们要如何存储类型呢？数据可以存变量，单是type和data的不同的东西，怎么办？

聪明的你可能以及想到了，我们可以让模板参数成为type信息的容器。

但是紧接着第二个问题来了，所谓list它的元素数量是固定的，但是直到c++11以前，模板参数的数量都是固定的，那么怎么办？

其实也很简单，参考普通list的链表实现法，我们也可以用相同的思想去构造一个“异质链表”：

```c++
template <typename T, typename U>
struct TypeList {
    typedef T Head;
    typedef U Tail;
};
```

这就是最简单的定义，其中，T是一个普通的类型，而U则是一个普通类型或TypeList。创建TypeList是这样的：

```c++
// 创建unsigned char和signed char的list
typedef TypeList<unsigned char, signed char> TypedChars;
// 现在我们把char也添加进去
typedef TypeList<char, TypeList<unsigned char, signed char> > Chars;
// 创建int，short，long，long long的list
typedef TypeList<int, TypeList<short, TypeList<long, long long> > > Ints;
```

可以看到，通过TypeList环环相扣，我们就能把所有的类型都存储在一个模板类组成的链表里了。但是这种实现的弊端有很多：

1. 首先是定义类型不方便，上面的第三个例子中，仅仅为了4个元素的list我们就要写出大量的嵌套代码，可读性大打折扣；
2. 原书中提到，为了简化定义，Loki库提供了`TYPELIST_N`这个宏，但是它是硬编码的，而且最大只支持50个元素，硬编码在程序员的世界里始终是丑陋的，更不用说还存在硬编码的数量上限，而且这么做也违反了“永远不要复读你自己”的原则，不过对于c++98来说只能如此；
3. 我们没办法清晰得表示只有一个元素或是没有元素的list，所以我们只能引入一个空类`NullType`来表示list的某一位上没有数据存在，比如：`TypeList<char, NullType>`或`TypeList<NullType, NullType>`，当然，你特化出单参数的TypeList也只是换汤不换药。
4. 无法有效得表示list的结尾，除非像上面一样使用NullType最为终结标志。

好在现代c++有变长模板，上述限制大多都不存在了：

```c++
template <typename...> struct TypeList;

template <typename Head, typename... Tails>
struct TypeList<Head, Tails...> {
    using head = Head;
    using tails = TypeList<Tails...>;
};

// 针对空list的特化
template <>
struct TypeList<> {};
```

通过变长模板，我们可以轻松定义任意长度的list：

```c++
using NumericList = TypeList<short, unsigned short, int, unsigned int, long, unsigned long>;
```

同时，我们特化出了空的TypeList，现在我们可以用它作为终止标记，而不用引入新的类型。如果你对变长模板不熟悉，可以搜索相关的资料，cnblogs上就有很多优质教程，介绍这个语法特性已经超过了本文的讨论范畴。

当然，变长模板也不是百利而无一害的，首先变长模板的参数包始终可以解包出空包，这会导致模板的偏特化和主模板发生歧义，因此在处理一些元函数（编译期计算出某些元数据的模板类就叫做元函数，概念来自于boost.mpl）的时候就要格外小心；其次，虽然我们方便了类型定义和部分的处理，但是向list头部添加数据就很困难了，参考下面的例子：

```c++
// TL1是一个包含int和long的list，现在我们在头部添加一个short
// 古典实现很简单
using New = TypeList<short, TL1>;

// 而现代的实现就没那么轻松了
// using New = TypeList<short, TL1>; 这么做是错的
```

问题出在哪？`...`运算符只能对参数包进行解包扩展，而TL1是一个类型，不是参数包，但是我们有需要把TL1包含的参数拿出来，于是问题就出现了。

对于这种需求我们只能使用一个元函数来解决，这是现代化方法为数不多的缺憾之一。

## 元函数的实现

定义了TypeList，接下来是定义各种元函数了。

也许你会疑惑为什么不把元函数定义为模板类的内部静态constexpr函数呢？现代c++不是已经具备强大的编译期计算能力了吗？

答案是否定的，编译期函数只能计算数值常量，而我们的元数据还包括了type，这时函数处理不了的。

不过话也不能说死，因为在处理数值常量的地方constexpr的作用还是很大的，后面我也会用constexpr函数辅助元函数。

### Length元函数求list长度

最常见的需求就是求出TypeList中存放了多少个元素，当然这也是实现起来最简单的需求。

先来看看古典技法，所谓古典技法就是让模板递归特化，依靠偏特化和特化来确定退出条件达到求值的目的。

因为编译期很难存储下迭代需要的中间状态，因此我们不得不依赖这种像递归函数般的处理技巧：

```c++
template <typename TList> struct Length; // 主模板，为下面的偏特化服务

template <>
struct Length<TypeList<>> {
    static constexpr int value = 0;
}

template <typename Head, typename... Types>
struct Length<TypeList<Head, Types...>> {
    static constexpr int value = Length<Types...>::value + 1;
};
```

解释一下，`static constexpr int value`是c++17的新特性，这种变量将会被视为类内的静态inline变量，可以就地初始化（c++11）。否则你可能需要将值定义为匿名的enum，这也是常见的元编程技巧之一。

我们从参数包的第一个参数开始逐个处理，遇到空包就返回0结束递归，然后从底层逐步返回，每一层都让结果+1，因为每一层代表了有一个type。

其实我们可以用c++11的新特性——sizeof...操作符，它可以直接返回参数包中参数的个数：

```c++
template <typename... Types>
struct Length<TypeList<Types...>> {
    static constexpr int value = sizeof...(Types);
};
```

使用现代c++的代码简单明了，因为参数包总是可以展开为空包，这时候value为0，还可以少写一个特化。

### TypeAt获取索引位置上的类型

list上第二个常见的操作就是通过index获取对应位置的数据。为了和c++的使用习惯相同，我们规定TypeList的索引也是从0开始。

在Python中你可以这样引用list的数据`list_1[3]`，但是我们并不会给元容器创建实体，元容器和元函数都是配合typedef或其他编译期手段实现编译期计算的，只需要用到它的类型本身和类型别名。因此我们只能这样操作元容器：`using res = typename TypeAt<TList, 3>::type`。

有了元函数的调用形式，我们可以开始着手实现了：

```c++
template <typename TList, unsigned int index> struct TypeAt;
template <typename Head, typename... Args>
struct TypeAt<TypeList<Head, Args...>, 0> {
    using type = Head;
};

template <typename Head, typename... Args, unsigned int i>
struct TypeAt<TypeList<Head, Args...>, i> {
    static_assert(i < sizeof...(Args) + 1, "i out of range");
    using type = typename TypeAt<TypeList<Args...>, i - 1>::type;
};
```

首先还是声明主模板，具体的实现交给偏特化。

虽然c++已经支持编译期在constexpr函数中进行迭代操作了，但是对于模板参数包我们至今不能实现直接的迭代，即使是c++17提供的折叠表达式也只是实现了参数包在表达式中的就地展开，远远达不到迭代的需要。因此我们不得不用老办法，从第一个参数开始，逐渐减少参数包中参数的数量，在减少了index个后这次偏特化的模板中，index一定是0， 而Head就一定是我们需要的类型，将它设置为type即可，而上层的元函数只需要不断减少index的值，并把Head从参数包中去除，将剩下的参数和index传递给下一层的元函数TypeAt即可。

顺带一提，static_assert不是必须的，因为你传递了不合法的索引，编译器会直接检测出来，但是在我这（g++ 8.3, clang++ 8.0.1, vs2017）编译器对此类问题发出的抱怨实在是难以让人类去阅读，所以我们使用static_assert来明确报错信息，而其余的信息比如不合法的index是多少，编译器会给你提示。

如果你不想越界报错而是返回NullType，那么可以这样写：

```c++
template <typename Head, typename... Args>
struct TypeAt<TypeList<Head, Args...>, 0> {
    using type = Head;
};

template <typename Head, typename... Args, unsigned int i>
struct TypeAt<TypeList<Head, Args...>, i> {
    // 如果i越界就返回NullType
    using type = typename TypeAt<TypeList<Args...>, i - 1>::type;
};

// 越界后的退出条件
template <unsigned int i>
struct TypeAt<TypeList<>, i> {
    using type = NullType;
};
```

因为不想越界后报错，所以我们要提供越界之后参数包为空的退出条件，在参数包处理完后就会立即使用这个新的特化，返回NullType。

聪明的读者也许会问为什么不用SFINAE，没错，在类模板和它的偏特化中我们也可以在模板参数列表或是类名后的参数列表中使用enable_if实现SFINAE，但是这里存在两个问题，一是类名后的参数列表必须要能推演出模板参数列表里的所有项，二是类名后的参数列不能和其他偏特化相同，同时也要符合主模板的调用方式。有了如上限制，利用SFINAE就变得无比困难了。（当然如果你能找到利用SFINAE的实现，也可以通过回复告诉我，大家可以相互学习；不清楚SFINAE是什么的读者，可以参阅cppreference上的简介，非常的通俗易懂）

当然这么做的话静态断言就要被忍痛割爱了，为了接口表现的丰富性，Loki的作者将不报错的TypeAt单独实现为了不同的元函数：

```c++
template <typename TList, unsigned int Index> struct TypeAtNonStrict;
template <typename Head, typename... Args>
struct TypeAtNonStrict<TypeList<Head, Args...>, 0> {
    using type = Head;
};

template <typename Head, typename... Args, unsigned int i>
struct TypeAtNonStrict<TypeList<Head, Args...>, i> {
    using type = typename TypeAtNonStrict<TypeList<Args...>, i - 1>::type;
};

template <unsigned int i>
struct TypeAtNonStrict<TypeList<>, i> {
    using type = Null;
};
```

### IndexOf获得指定类型在list中的索引

IndexOf的套路和TypeAt差不多，只不过这里的递归不用扫描整个参数包（逐个按顺序处理参数包，是不是和扫描一样呢），只需要匹配到Head和待匹配类型相同，就返回0；如果不匹配就像TypeAt中那样递归调用元函数，对其返回结果+1，因为结果在本层之后，所以需要把本层加进索引里，递归调用返回后逐渐向前相加最终的结果就是类型所在的index（从0开始）。

IndexOf一个重要的功能就是判断某个类型是否在TypeList中。

如果处理完参数包仍然找不到对应类型呢？这时候对空的TypeList做个特化返回-1就行，当然前面的偏特化元函数也需要对这种情况做处理。

现在我们来看下IndexOf的调用形式：“IndexOf<TList, int>::value”

现在我们就照着这个形式实现它：

```c++
template <typename TList, typename T> struct IndexOf;
template <typename Head, typename... Tails, typename T>
struct IndexOf<TypeList<Head, Tails...>, T> {
private:
    // 为了避免表达式过长，先将递归的结果起了别名
    using Result = IndexOf<TypeList<Tails...>, T>;
public:
    // 如果类型相同就返回，否则检查递归结果，-1说明查找失败，否则返回递归结果+1
    static constexpr int value =
            std::is_same_v<Head, T> ? 0 :
            (Result::value == -1 ? -1 : Result::value + 1);
};

// 终止条件，没找到对应类型
template <typename T>
struct IndexOf<TypeList<>, T> {
    static constexpr int value = -1;
};
```

因为有了c++11的type_traits的帮助，我们可以偷懒少写了一个类似这样的偏特化：

```c++
template <typename... Tails, typename T>
struct IndexOf<TypeList<T, Tails...>, T> {
    static constexpr int value = 0;
};
```

然而现代c++的威力远不止如此，前面我们说过不能对参数包实现迭代，但是我们可以借助折叠表达式、constexpr函数，编译期容器这三者，将参数包中每一个参数映射到编译期容器中，之后便可以对编译期容器进行迭代操作，避免了递归偏特化。

当然，这种方案只是证明了c++的可能性，真正实现起来比递归的方式要麻烦的多，性能可能也并不会比递归好多少（当然都是编译期的计算，不会付出运行时代价），而且需要一个完全支持c++14，至少支持c++17折叠表达式的编译期（vs2019可以设置使用clang，原生的编译器对c++17的支持有点惨不忍睹）。

技术的关键是c++14的`std::array`和`std::index_sequence`。

前者是我们需要使用的编译期容器（vector也许以后也会成为编译期容器，编译期的动态内存分配已经进入c++20），后者负责把一串数字映射为模板参数包，以便折叠表达式展开。（折叠表达式仍然可以参考cppreference上的解释）

`std::index_sequence`的一个示例：

```c++
using Ints = std::make_index_sequence<5>; // 产生std::index_sequence<0, 1, 2, 3, 4>

// 将一串数字传递给模板，重新映射为变长模板参数
template <typename T, std::size_t... Nums>
void some_func(T, std::index_sequence<Nums...>) {/**/}

some_func("test", Ints{}); // 这时Nums包含<0, 1, 2, 3, 4>
```

这个用法看着很像元编程的惯用法之一的标签分派，但是仔细看的话两者不是同一种技巧，暂时没有发现这种技巧的具体名字，因此我们就暂时称其为“整数序列映射”。

有了这些前置知识，现在可以看实现了：

```c++
template <typename TList, typename T> struct IndexOf2;
template <typename T, typename... Types>
struct IndexOf2<TypeList<Types...>, T> {
    using Seq = std::make_index_sequence<sizeof...(Types)>;
    static constexpr int index()
    {
        std::array<bool, sizeof...(Types)> buf = {false};
        set_array(buf, Seq{});
        for (int i = 0; i < sizeof...(Types); ++i) {
            if (buf[i] == true) {
                return i;
            }
        }
        return -1;
    }

    template <typename U, std::size_t... Index>
    static constexpr void set_array(U& arr, std::index_sequence<Index...>)
    {
        ((std::get<Index>(arr) = std::is_same_v<T, Types>), ...);
    }
};

// 空TypeList单独处理，简单返回-1即可，因为list里没有任何东西自然只能返回-1
template <typename T>
struct IndexOf2<TypeList<>, T> {
    static constexpr int index()
    {
        return -1;
    }
};
```

其中index很好理解，首先初始化一个array，随后将参数包的每个参数的状态映射到array里，之后循环找到第一个true的index，整个过程都在编译期进行。

问题在于`set_array`里，里面究竟发生了什么呢？

首先是我们前面提到的整数序列映射，Index在映射后是`{0, 1, 2, ..., len_of(Array) - 1}`，接着被折叠表达式展开为：

```c++
(
    (std::get<0>(arr) = std::is_same_v<T, Types_0>),
    (std::get<1>(arr) = std::is_same_v<T, Types_1>),
    (std::get<2>(arr) = std::is_same_v<T, Types_2>),
    ...,
    (std::get<len_of(Array) - 1>(arr) = std::is_same_v<T, Types_(len_of(Array) - 1>)),
)
```

真实的展开是类似`Arg1, (Arg2, (Arg3, Arg4))`这种，为了可读性我把括号省略了，反正在这里执行顺序并不影响结果。

get会返回array中指定的index的内容的引用，因此我们可以对它赋值，`Types_N`则是从左至右被依次展开的参数，这样不借助递归就将参数包中所有的参数处理完了。

不过本质上方案B还是舍近求远式的杂耍，实用性并不高，但是它充分展示了现代c++给模板元编程带来的可能性。

### Append为TypeList添加元素

看完前面几个元函数你可能已经觉得有点累了，没事我们看个简单的放松一下。

Append可以在TypeList前添加元素（虽然这个操作严格来说不叫Append，但后面经常要用而且实现类似，所以请允许我把它当作特殊的Append），在TypeList后面添加元素或是其他TypeList中的所有元素。

调用形式如下：

```c++
Append<int, TList>::result_type;
Append<TList, long>::result_type;
Append<TList1, TList2>::result_type;
```

借助变长模板实现起来颇为简单：

```c++
template <typename, typename> struct Append;
template <typename... TList, typename T>
struct Append<TypeList<TList...>, T> {
    using result_type = TypeList<TList..., T>;
};

template <typename T, typename... TList>
struct Append<T, TypeList<TList...>> {
    using result_type = TypeList<T, TList...>;
};

template <typename... TListLeft, typename... TListRight>
struct Append<TypeList<TListLeft...>, TypeList<TListRight...>> {
    using result_type = TypeList<TListLeft..., TListRight...>;
};
```

### Erase和EraseAll删除元素

顾名思义，Erase负责删除第一个匹配的type，EraseAll删除所有匹配的type，它们有着一样的调用形式：

```c++
Erase<TList, int>::result_type;
EraseAll<TList, long>::result_type;
```

Erase的算法也比较简单，利用了递归，先在本层查找，如果匹配就返回去掉Head的TypeList，否则对剩余的部分继续调用Erase：

```c++
template <typename TList, typename T> struct Erase;
template <typename Head, typename... Tails, typename T>
struct Erase<TypeList<Head, Tails...>, T> {
    using result_type = typename Append<Head, typename Erase<TypeList<Tails...>, T>::result_type >::result_type;
};

// 终止条件1，删除匹配的元素
template <typename... Tails, typename T>
struct Erase<TypeList<T, Tails...>, T> {
    using result_type = TypeList<Tails...>;
};

// 终止条件2，未发现要删除的元素
template <typename T>
struct Erase<TypeList<>, T> {
    using result_type = TypeList<>;
};
```

注意模板的第一个参数必须是一个TypeList。

如果Head和T不匹配时，我们需要借助Append把Head粘回TypeList，这是在[定义](#typelist%e7%9a%84%e5%ae%9a%e4%b9%89)那节提到的弊端之一，因为我们不可能直接展开TypeList类型，它不是变长模板的参数包。后面的几个元函数中都需要用到Append来完成相同的工作，与传统的链式实现相比这一点确实不够优雅。

有了Erase，实现EraseAll就简单很多了，我们只需要在终止条件1那里不终止，而是对剩下的list继续进行EraseAll即可：

```c++
template <typename TList, typename T> struct EraseAll;
template <typename Head, typename... Tails, typename T>
struct EraseAll<TypeList<Head, Tails...>, T> {
    using result_type = typename Append<Head, typename EraseAll<TypeList<Tails...>, T>::result_type >::result_type;
};

// 这里不会停止，而是继续把所有匹配的元素删除
template <typename... Tails, typename T>
struct EraseAll<TypeList<T, Tails...>, T> {
    using result_type = typename EraseAll<TypeList<Tails...>, T>::result_type;
};

template <typename T>
struct EraseAll<TypeList<>, T> {
    using result_type = TypeList<>;
};
```

有了Erase和EraseAll，下面去除重复元素的元函数也就能实现了。

### NoDuplicates去除所有重复type

NoDuplicates也许看起来会很复杂，其实不然。

NoDuplicates算法只需要三步：

1. 先对去除Head之后的TypeList进行NoDuplicates操作，形成L1；现在保证L1里没有重复元素
2. 对L1进行删除所有Head的操作，形成L2，因为L1里可能会有和Head相同的元素；
3. 最后将Head添加回TypeList

步骤1中递归的调用还会重复相同的步骤，这样最后就确保了TypeList中不会有重复的元素出现。这个元函数也是较为常用的，比如你肯定不会想在抽象工厂模板类中出现两个相同的类型，这不正确也没有必要。

调用形式为：

```c++
NoDuplicates<TList>::result_type;
```

按照步骤实现算法也不难：

```c++
template <typename TList> struct NoDuplicates;
template <>
struct NoDuplicates<TypeList<>> {
    using result_type = TypeList<>;
};

template <typename Head, typename... Tails>
struct NoDuplicates<TypeList<Head, Tails...>> {
private:
    // 保证L1中没有重复的项目
    using L1 = typename NoDuplicates<TypeList<Tails...>>::result_type;
    // 删除L1中所有和Head相同的项目，L1中已经没有重复，所以最多只会有一项和Head相同，Erase就够了
    using L2 = typename Erase<L1, Head>::result_type;
public:
    // 把Head添加回去
    using result_type = typename Append<Head, L2>::result_type;
};
```

在处理L1时我们只使用了Erase，注释已经给出了原因。

### Replace和ReplaceAll

除了删除，偶尔我们也希望将某些type替换成新的type。

这里我只讲解Replace的实现，Replace和ReplaceAll的区别就像Erase和EraseAll，因此不再赘述。

Replace其实就是翻版的Erase，只不过它并不删除匹配的Head，而是将其替换成了新类型。

```c++
template <typename TList, typename Old, typename New> struct Replace;
template <typename T, typename U>
struct Replace<TypeList<>, T, U> {
    using result_type = TypeList<>;
};

template <typename... Tails, typename T, typename U>
struct Replace<TypeList<T, Tails...>, T, U> {
    using result_type = typename Append<U, TypeList<Tails...>>::result_type;
};

template <typename Head, typename... Tails, typename T, typename U>
struct Replace<TypeList<Head, Tails...>, T, U> {
    using result_type = typename Append<Head, typename Replace<TypeList<Tails...>, T, U>::result_type>::result_type;
};
```

### Derived2Front将派生类型移动至list前部

前面的元函数基本都是将参数包分解为Head和Tails，然后通过递归依次处理，但是现在描述的算法就有些复杂了。

通过给定一个Base，我们希望TypeList中所有Base的派生类都能出现在list的前部，位置先于Base，这在你处理继承的层次结构时会很有帮助，当然我们后面是示例中没有使用此功能，不过作为一个比较重要的接口，我们还是需要进行一定的了解的。

首先想要将派生类移动到前端就需要先找出在list末尾上的派生类型，我们使用一个帮助类的元函数MostDerived来实现：

```c++
template <typename TList, typename Base> struct MostDerived;
// 终止条件，找不到任何派生类就返回Base自己
template <typename T>
struct MostDerived<TypeList<>, T> {
    using result_type = T;
};

template <typename Head, typename... Tails, typename T>
struct MostDerived<TypeList<Head, Tails...>, T> {
private:
    using candidate = typename MostDerived<TypeList<Tails...>, T>::result_type;
public:
    using result_type = std::conditional_t<std::is_base_of_v<candidate, Head>, Head, candidate>;
};
```

首先我们递归调用MostDerived，结果保存为candidate，这是Base在去除Head之后的list中最深层次的派生类或是Base自己，然后我们判断Head是否是candidate的派生类，如果是就返回Head，否则返回candidate，这样就可以得到最末端的派生类类型了。

`std::conditional_t`则是c++11的type_traits提供的基础设施之一，通过bool值返回类型，有了它我们就可以省去自己实现Select的工夫了。

完成帮助元函数后就可以着手实现Derived2Front了：

```c++
template <typename TList> struct Derived2Front;
template <>
struct Derived2Front<TypeList<>> {
    using result_type = TypeList<>;
};

template <typename Head, typename... Tails>
struct Derived2Front<TypeList<Head, Tails...>> {
private:
    using theMostDerived = typename MostDerived<TypeList<Tails...>, Head>::result_type;
    using List = typename Replace<TypeList<Tails...>, theMostDerived, Head>::result_type;
public:
    using result_type = typename Append<theMostDerived, List>::result_type;
};
```

算法步骤不复杂，先找到最末端的派生类，然后将去除头部的TypeList中与最末端派生类相同的元素替换为Head，最后我们把最末端的派生类添加在处理过的TypeList的最前面，就完成了派生类从末端移动到前端。

### 元函数实现总结

通过这些元函数的示例，我们可以看到现代c++对于元编程有了更多的内建支持，利用新的标准库和语言特性我们可以少写很多代码，也可以实现在c++11之前看似根本不可能的任务。

当然现代c++也带来了自己独有的问题，比如边长模板参数包无法直接迭代，这导致了我们大多数时间仍然需要依赖递归和偏特化这样的古典技法。

然而不可否认的是，随着语言的进化，c++进行元编程的难度在不断下降，元编程的能力和代码的表现力也越来越强了。

## 示例

我想通过两个示例来更好地展示TypeList和现代c++的威力。

第一个例子是个简陋的tuple类型，模仿了标准库。

第二个例子是工厂类，传统的工厂模式要么避免不了复杂的继承结构，要么避免不了大量的硬编码导致扩展困难，我们使用TypeList来解决这些问题。

### 自制tuple

首先是我们的玩具tuple，之所以说它简陋是因为我们只选择实现了get这一个接口，并且标准库的tuple并不是向我们这样实现的，因此这里的tuple只是一个演示用的玩具罢了。

首先是我们用来存储数据的节点：

```c++
template <typename T>
struct Data {
    explicit Data(T&& v): value_(std::move(v))
    {}
    T value_;
};
```

接着我们实现Tuple：

```c++
template <typename... Args>
class Tuple: private Data<Args>... {
    using TList = TypeList<Args...>;
public:
    explicit Tuple(Args&&... args)
    : Data<Args>(std::forward<Args>(args))... {}

    template <typename Target>
    Target& get()
    {
        static_assert(IndexOf<TList, Target>::value != -1, "invalid type name");
        return Data<Target>::value_;
    }

    template <std::size_t Index>
    auto& get()
    {
        static_assert(Index < Length<TList>::value, "index out of range");
        return get<typename TypeAt<TList, Index>::type>();
    }

    // const的重载
    template <typename Target>
    const Target& get() const
    {
        static_assert(IndexOf<TList, Target>::value != -1, "invalid type name");
        return Data<Target>::value_;
    }

    template <std::size_t Index>
    const auto& get() const
    {
        static_assert(Index < Length<TList>::value, "index out of range");
        return get<typename TypeAt<TList, Index>::type>();
    }
};

// 空Tuple的特化
template <>
class Tuple<> {};
```

我们的Tuple实现地简单暴力，通过private继承，我们就可以同时存储多种不同的数据，引用的时候只需要`Data<type>.value_`，因此我们的第一个get很容易就实现了，只需要检查TypeList中是否存在对应类型即可。

但是标准库的get还有第二种形式：`get<1>()`。对于第一种get，事实上我们不借助TypeList也能实现，但是对于第二种我们就不得不借助TypeList的力量了，因为我们除了利用元容器记录type的出现顺序之外别无办法（这也是为什么标准库不会这样实现tuple的原因之一）。因此我们利用`TypeAt`元函数找到对应的类型后再获取它的值。

另外标准库不使用这种形式最重要的原因就是如果你在tuple里存储了2个以上相同type的数据，会报错，很容易想到是为什么。

所以类似的技术更适合用于`variant`这样的对象，不过这里只是举例所以我们忽略了这些问题。

下面是一些简单的测试：

```c++
Tuple<int, double, std::string> t{1, 1.2, "hello"};
std::cout << t.get<std::string>() << std::endl;
t.get<std::string>() = "Hello, c++!";
std::cout << t.get<2>() << std::endl;
std::cout << t.get<1>() << std::endl;
std::cout << t.get<0>() << std::endl;

// Output:
// hello
// Hello, c++!
// 1.2
// 1
```

### 简化工厂模式

假设我们有一个`WidgetFactory`，用来创建不同风格的Widgets，Widgets的种类有很多，例如Button，Label等：

```c++
class WidgetFactory {
public:
    virtual CreateButton() = 0;
    virtual CreateLabel() = 0;
    virtual CreateToolBar() = 0;
};

// 风格1
class KDEFactory: public WidgetFactory {
public:
    CreateButton() override;
    CreateLabel() override;
    CreateToolBar() override;
};

// 风格2
class GnomeFactory: public WidgetFactory {
public:
    CreateButton() override;
    CreateLabel() override;
    CreateToolBar() override;
};

// 使用
WidgetFactory* factory = new KDEFactory;
factory->CreateButton(); // KDE button
delete factory;
factory = new GnomeFactory;
factory->CreateButton(); // Gnome button
```

这种实现有两个问题，一是如果增加/改变/减少产品，那么需要改动大量的代码，容易出错；二是创建不同种类的widget的代码通常是较为相似的，所以我们在这里需要不断复读自己，这通常是bug的根源之一。

较为理想的形式是什么呢？如果widget构造过程相同，只是参数上有差别，你可能已经想到了，我们有变长模板和完美转发：

```c++
class WidgetFactory {
public:
    template <typename T, typename... Args>
    auto Create(Args&&... args)
    {
        return new T(std::forward<Args>(args)...);
    }
};
```

这样我们可以通过`Create<KDEButton>(...)`来创建不同的对象了，然而这已经不是一个工厂了，我们创建工厂的目的之一就是为了限制产品的种类，现在我们反而把限制解除了！

那么这么解决呢？答案还是TypeList，通过TypeList限制产品的种类：

```c++
template <typename... Widgets>
class WidgetFactory {
    // 我们不需要重复的类型
    using WidgetList = NoDuplicates<TypeList<Widgets...>>::result_type;
public:
    template <typename T, typename... Args>
    auto Create(Args&&... args)
    {
        static_assert(IndexOf<WidgetList, T>::value != -1, "unknow type");
        return new T(std::forward<Args>(args)...);
    }
};

using KDEFactory = WidgetFactory<KDEButton, KDEWindow, KDELabel, KDEToolBar>;
using GnomeFactory = WidgetFactory<GnomeLabel, GnomeButton>;
```

现在如果我们想增加或改变某一个工厂的产品，只需要修改有限数量的代码即可，而且我们在限制了产品种类的同时将重复的代码进行了抽象集中。同时，类型检查都是编译期处理的，无需任何的运行时代价！

当然，这样简化的坏处是灵活性的降低，因为不同工厂现在实质是不同的不相关类型，不可能通过`Base*`或`Base&`关联起来，不过对于接口相同但是类型相同的对象，我们还是可以依赖模板实现静态分派，这只是设计上的取舍而已。

## 总结

这篇文章只是对模板元编程的入门级探讨，旨在介绍如果使用现代c++简化元编程和泛型编程任务。

本文虽然不能带你入门元编程，但是可以让你对元编程的概念有一个整体的概览，对深入的学习是有帮助的。
