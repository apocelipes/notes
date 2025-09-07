有一个古老的c++问题：`struct Empty{}; sizeof(Empty);` 请问Empty的大小是多少。

很多新手会回答0，但稍有经验的开发者会说出正确答案，大小至少是1字节。

这看起来很奇怪，但这是语言规范决定的：c++要求同一类型的不同实例对象必须拥有完全不同的地址，如果Empty的大小是0，那么想象一下一个元素类型是Empty的数组，这个数组的连续存储空间里很可能不同的Empty会重叠在一起，从而导致它们违反前面对于拥有不同地址的规定。最简单最省事的做法就是让这种看起来大小应该为0的类型占据一字节的内存，从而确保每个实例都有独立的地址。而且语言规范也是要求这样去做的，它要求所有零大小的类型除了位域都必须占至少一字节的内存。

这么做当然带来了很多弊端，所以c++20新增了属性`[[no_unique_address]]`来解决问题。

不过在介绍这个属性之前，我们还得回顾一点基础知识。

## 基础回顾

c++的知识是一环套一环的，所以基础回顾环节少不了。我们需要回顾三个小知识点：什么是空类型、什么是空基类优化、空类型对内存对齐的影响。

首先回顾的是“空类型是什么”。

空类型，或者用语言规范里的叫法“zero size”，是指那些符合标准布局的、没有虚基类虚函数、没有非静态数据成员的类型。如果存在继承关系，则类型的每一层继承关系上涉及的类型也都必须符合前面提到的条件，这样的类型可以被视作是空类型。union不在此范围之内。

简单的说，下面三个类都可以被认为是空的：

```c++
struct A {
    static constexpr int i = 0; // 这是静态数据成员，不影响类型为zero size
};

struct B {};

struct C: A {}; // 自己和基类都符合要求

int main()
{
    static_assert(std::is_empty_v<A>);
    static_assert(std::is_empty_v<B>);
    static_assert(std::is_empty_v<C>);
}
```

`std::is_empty`是c++11新增的用于判断类型是否是zero size的接口。我们可以看到，没有非静态数据成员没有虚函数且基类也符合同样条件的类型都会被认为是空类型。

概念还是很容易理解的，不过标准并没有把话说死，在后面标准紧接着指出任何编译器觉得应该是空类型的东西也可以算作空类型。换句话说除了标准规定的少数情况，还有不少类型是否为空是具体平台和编译器共同影响的。

第二个要回顾的是“空类型对内存对齐的影响”。在复习空基类优化之前我们需要知道优化的动机，而动机来自于空类型对内存对齐的影响。

我们现在都知道因为c++对象地址的限制，空类型需要占用至少一字节的内存。这会让程序付出代价：

```c++
struct Empty {};

struct A {
    long number;
    Empty e;
};

static_assert(sizeof(A) > sizeof(long));
```

A的大小至少为2个long类型的大小。为什么呢，因为c++有内存对齐的规则，类的对齐长度以所有非静态数据成员中对齐长度最大的为准，这里我们有两个非静态数据成员，number和e，number的长度是`sizeof(long)`，而它的对齐长度要求也是`sizeof(long)`，e的长度和对齐要求都是1，`sizeof(long)`一定大于1，所以最后类型A要求每个字段都以`sizeof(long)`为基准进行对齐，作为最后一个字段的e，前面的字段number正好有一个long类型那么长，而自己后面又没有其他字段，按对齐要求这时候需要在自己后面填充`sizeof(long) - 1`个字节的填充物。最后A的整体大小会是两个long那么大。

实际上我们用不到Empty占用的内存里的内容，通常我们使用空类型是为了利用其类方法或者静态数据，但却要为了这一字节付出内存占用上的代价。类型变成两倍大意味着高速缓存里能存下的同类型数据至少减少一半，对于频繁访问这类数据的程序来说这是显著的性能损失。

c++为了践行“不支付不必要的运行时代价”，提出了EBO——空基类优化（Empty Base Optimization）这一方案。

空基类优化，是指当基类为空类型，派生类的第一个非静态数据成员的类型和基类不一样，继承不是虚拟继承的时候，这个空类型的基类可以不占用任何存储空间。

举个例子，还是前面的A：

```c++
struct Empty {};
struct A : Empty {
    long number;
};

static_assert(sizeof(A) == sizeof(long))
```

正常情况下基类也需要在派生类的内存空间内占据一部分地盘，但因为空基类优化，这一字节的占用就免除了。空基类优化也适用于多继承：

```c++
struct Empty1 {};
struct Empty2 {};
struct A : Empty1, Empty2 {
    long number;
};

static_assert(sizeof(A) == sizeof(long))
```

通过继承，我们也可以复用作为基类的空类型的静态数据和类方法，同时又不用支付存储的代价。

对于不满足要求的类型，比如第一个数据成员的类型和基类相同，这时候空基类优化就不生效了：

```c++
struct Empty {};
struct A: Empty {
    Empty e;
};

static_assert(sizeof(A) > sizeof(Empty));
```

A至少有两个Empty那么大。因为在一部分平台上基类的内存是紧挨着派生类的数据成员的，如果第一个数据成员的类型和基类相同，那么继续应用空基类优化就会导致基类和第一个数据成员发生重叠（基类的大小是0对其取地址通常会得到和派生类或者派生类数据成员相同的地址），这违反了c++对于同类型的不同对象地址必须不同的规定。

空基类优化在标准库里用的很多，比如Hasher、各种迭代器以及allocator，都是使用了空基类优化来复用方法同时减小存储负担的。

另外还有一个比较知名的空基类优化应用：compressed_pair，这是`std::pair`的变体，它在元素为空类型的时候可以不占用额外的内存，原理就是利用了空基类优化。这种容器常见的第三方c++模板库中都有提供，比如boost。

## 新属性no_unique_address

空基类优化看似解决了问题，然而继承本身会引来新的问题。

继承最大的问题在于派生类和基类的关系是`is-a`，即派生类从分类上是基类的某种延伸或者说派生类和基类直接有着相似的结构和操作方法。但如果我们只是想复用空类型中的方法或者干脆为了避免内存占用而使用空基类优化，则会打破这种`is-a`关系。

考虑一下上一节说到的`compressed_pair`，再能利用`no_unique_address`之前它的实现是这样的：

```c++
template <class _T1, class _T2>
class compressed_pair : private __compressed_pair_elem<_T1, 0>, private __compressed_pair_elem<_T2, 1> {
public:
  // NOTE: This static assert should never fire because __compressed_pair
  // is *almost never* used in a scenario where it's possible for T1 == T2.
  // (The exception is std::function where it is possible that the function
  //  object and the allocator have the same type).
  static_assert(
      (!is_same<_T1, _T2>::value),
      "__compressed_pair cannot be instantiated when T1 and T2 are the same type; "
      "The current implementation is NOT ABI-compatible with the previous implementation for this configuration");

  using _Base1 _LIBCPP_NODEBUG = __compressed_pair_elem<_T1, 0>;
  using _Base2 _LIBCPP_NODEBUG = __compressed_pair_elem<_T2, 1>;
  ...
};
```

`__compressed_pair_elem`是元素的包装器，用来提供元素的访问方法，以及在元素大小是0的时候让自己的大小也为0，方便利用空基类优化：

```c++
template <class _Tp, int _Idx, bool _CanBeEmptyBase = is_empty<_Tp>::value && !__libcpp_is_final<_Tp>::value>
struct __compressed_pair_elem {
  using _ParamT         = _Tp;
  using reference       = _Tp&;
  using const_reference = const _Tp&;

  _LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR explicit __compressed_pair_elem(__default_init_tag) {}
  _LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR explicit __compressed_pair_elem(__value_init_tag) : __value_() {}
  ...
  其他一些构造函数，这里省略

  _LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR_SINCE_CXX14 reference __get() _NOEXCEPT { return __value_; }
  _LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR const_reference __get() const _NOEXCEPT { return __value_; }

private:
  _Tp __value_;
};

// 注意下面这个为了对象大小是0的部分特化模板
template <class _Tp, int _Idx>
struct __compressed_pair_elem<_Tp, _Idx, true> : private _Tp {
  using _ParamT         = _Tp;
  using reference       = _Tp&;
  using const_reference = const _Tp&;
  using __value_type    = _Tp;

  _LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR explicit __compressed_pair_elem() = default;
  _LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR explicit __compressed_pair_elem(__default_init_tag) {}
  _LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR explicit __compressed_pair_elem(__value_init_tag) : __value_type() {}
  其他一些构造函数，这里省略

  _LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR_SINCE_CXX14 reference __get() _NOEXCEPT { return *this; }
  _LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR const_reference __get() const _NOEXCEPT { return *this; }
  // 注意这里，没有任何数据成员，所以这个模板类的实例大小也是零，这个模板实例化出来的都是空类型
};
```

对于这些代码，最直观的感受就是长。对于模板用的不多的开发者来说这东西还会沾点难懂。但最重要的问题在于这一继承关系阐述了这样一个情况：pair是(is-a)一种pair自己的元素。很荒诞，

鉴于利用空基类优化的代码又长又复杂，还会违背继承关系的原则，c++20接受了`[[no_unique_address]]`的提案，提供了一种不利用继承同时又能让不同类型的实例对象内存空间发生折叠的技术。

顾名思义，被`[[no_unique_address]]`修饰的东西可以没有自己独立的地址。具体来说这个属性只能用在类的非静态数据成员上，且根据字段是否是空类型会有不同的效果：

1. 如果是空类型，则这个字段可以和其他的类非静态数据成员或者基类的内存空间重叠在一起，也就是这个字段本身不再占用内存，对这个字段取地址也会得到类的其他数据成员或者基类的地址。
2. 如果不为空，则这个字段后面因为内存对齐留下的空间可以被其他类成员利用。

对于非空类型来说，这个属性没有什么明显的效果，因为目前只要相邻的字段大小和对齐合适，就会自动利用前一个字段因为对齐而留下的空间。这个属性只是有限度的放宽了“相邻”这个限制，但类的成员还有offset偏移量这个限制需要遵守，所以很难在非空类型字段上看到这个属性带来的影响。

而对于空类型，这个属性的影响就大了，举个例子：

```c++
struct Empty {};
struct A {
    long number;
    [[no_unique_address]] Empty e;
};

static_assert(sizeof(A) == sizeof(long));

#include <cstddef>

int main()
{
    std::cout << offsetof(A, e) << '\n'; // GCC和Clang上都是0，如果不加属性这个值会是4或8
}
```

利用`[[no_unique_address]]`，我们可以让e和number共享内存空间，e不再占用1字节的额外内存，所以A只有一个long那么大。这是对于内存占用的影响。

第二个影响是对`[[no_unique_address]]`修饰的成员取地址和计算偏移量。被修饰的字段的地址和偏移量是不确定的。标准规定对于被修饰的成员，取地址和计算偏移量都是合法的，但没规定取到的地址和偏移量具体应该是什么，只是说可能是其他类成员变量或者基类的地址。换个说法，标准的意思就是取地址是合法的，但得到的值是不确定的。这是一种ABI变更，不仅A的大小改变了，A的成员的内存布局也发生了很大的变化。

`[[no_unique_address]]`虽然让被修饰字段的内存可以和其他对象重叠，但仍然需要遵守c++关于相同类型的不同对象需要有不同地址的规定：

```c++
struct Empty1 {};
struct Empty2 {};
struct A {
    long number;
    [[no_unique_address]] Empty1 e1;
    [[no_unique_address]] Empty2 e2;
};

struct B {
    long number;
    [[no_unique_address]] Empty1 e1;
    [[no_unique_address]] Empty1 e2;
};

static_assert(sizeof(A) == sizeof(long));
static_assert(sizeof(B) > sizeof(long));
```

注意B中我们的`e1`和`e2`类型相同，为了不违反规则，`e1`和`e2`中有一个是要有自己的独立的内存空间的，另一个可以和其他类型的字重叠。至于那个字段有独立空间哪个字段重叠，这个完全由编译器决定。而类型不同，则两个字段都可以和别的字段发生重叠，因此都不占额外的内存空间。

最后一点，如果类中只有一个非静态数据成员，且这个成员有空类型，那么`[[no_unique_address]]`也不会生效：

```c++
struct Empty {};
struct A {
    [[no_unique_address]] Empty e;
};

struct B {
    Empty e;
};

static_assert(sizeof(A) == 1);
static_assert(sizeof(A) == sizeof(B));
```

属性`[[no_unique_address]]`提供了一种比空基类优化更简单更清晰的方式让空类型不再占用额外的内存。

## no_unique_address的应用

如果你的代码不是很在意ABI稳定性的话，很多空基类优化可以转换成更简单`[[no_unique_address]]`。

我们还是拿前文中的libcxx的`compressed_pair`举例子，转换后的代码如下：

```c++
struct compressed_pair {
    _LIBCPP_NO_UNIQUE_ADDRESS __attribute__((__aligned__(::std::__compressed_pair_alignment<T2>))) T1 Initializer1;
    // 内存对齐填充
    _LIBCPP_NO_UNIQUE_ADDRESS T2 Initializer2;
    // 内存对齐填充
};
```

`_LIBCPP_NO_UNIQUE_ADDRESS`是个宏，会被替换成`[[no_unique_address]]`或者`[[msvc::no_unique_address]]`，因为号称完全支持c++20的MSVC实际上没有正确实现`[[no_unique_address]]`这个属性，所以在MSVC上必须使用编译器自己实现的效果类似的属性，包装代码在`llvm-project/libcxx/include/__config`里：

```c++
#  if __has_cpp_attribute(msvc::no_unique_address)
// MSVC implements [[no_unique_address]] as a silent no-op currently.
// (If/when MSVC breaks its C++ ABI, it will be changed to work as intended.)
// However, MSVC implements [[msvc::no_unique_address]] which does what
// [[no_unique_address]] is supposed to do, in general.
#    define _LIBCPP_NO_UNIQUE_ADDRESS [[msvc::no_unique_address]]
#  else
     // __no_unique_address__是clang和gcc实现的[[no_unique_address]]
#    define _LIBCPP_NO_UNIQUE_ADDRESS [[__no_unique_address__]]
#  endif
```

整体代码要比利用空基类优化的那版简单很多。同时，这个实现也不会有奇怪的继承关系了。

除此之外libcxx里还有很多类似的使用例，在不影响运行时效率的前提下大幅简化了代码。

## 总结

`[[no_unique_address]]`让空类型的类数据成员有机会不再占用额外的内存空间，从而减轻了因为地址规定带来的性能影响，同时还让空基类优化代码得到了简化的机会。

不过这个属性会破坏ABI兼容性，所以重构的时候要慎重。然而它带来的好处是很实在的，所以libcxx在去年用这个属性重构了一大堆的代码，并且在文档里注明了哪些东西的ABI兼容被破坏了。对于开发者来说这是阵痛，但对于长期维护来说是利大于弊的。

关于这个属性以及对于c++语言规范的影响，可以看这里：<https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0840r2.html>
