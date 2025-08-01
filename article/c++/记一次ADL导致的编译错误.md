这篇文章主要讲讲c++的ADL，顺便说说为什么很多c++的IDE都会让你尽量不要include用不上的头文件。

和其他c++文章一样，这篇也会有基础回顾环节，所以不用担心看不懂，但读者最好还是得有c++的基础知识并且对c++11之后的内容有所了解。

好了，下面我们进入正题吧。

## 偶遇报错

最近工作收尾有了不少空闲时间，于是准备试试手头环境的编译器对新标准的支持，以便选择合适的时机给自己的几个项目做个升级。

虽然有现成的工具的网站可以查询编译器对新标准的支持情况，但这些网站给的信息还是不够详细，有时候得写些例子手动编译做测试。我是个懒人，所以我不愿意花时间自己写，而AI又对新标准理解的不够透彻，可能是语料太少的缘故，总是写出点离谱的东西。无奈之下我只能去网上找现成的吃了，cppreference是个不错的选择，用的人很多而且比较权威，更棒的是对于新特性它一般都给出了示例代码，这正中我的下怀。

于是我搬了这样一段代码进行测试，预想中要么编译成功要么新特性不支持导致编译失败：

```c++
#include <array>
#include <iostream>
#include <list>
#include <ranges>
#include <string>
#include <tuple>
#include <vector>

void print(auto const rem, auto const& range)
{
    for (std::cout << rem; auto const& elem : range)
        std::cout << elem << ' ';
    std::cout << '\n';
}

int main()
{
    auto x = std::vector{1, 2, 3, 4};
    auto y = std::list<std::string>{"α", "β", "γ", "δ", "ε"};
    auto z = std::array{'A', 'B', 'C', 'D', 'E', 'F'};

    print("Source views:", "");
    print("x: ", x);
    print("y: ", y);
    print("z: ", z);

    print("\nzip(x,y,z):", "");

    for (std::tuple<int&, std::string&, char&> elem : std::views::zip(x, y, z))
    {
        std::cout << std::get<0>(elem) << ' '
                  << std::get<1>(elem) << ' '
                  << std::get<2>(elem) << '\n';

        std::get<char&>(elem) += ('a' - 'A'); // modifies the element of z
    }

    print("\nAfter modification, z: ", z);
}
```

很简单的代码，测试一下c++23的`ranges::views::zip`，如果要报错那么多半也是和这个zip有关。

然而事实出人意料：

```console
$ clang++ -std=c++23 -Wall test.cpp
test.cpp:23:5: error: call to 'print' is ambiguous
   23 |     print("x: ", x);
      |     ^~~~~
test.cpp:9:6: note: candidate function [with rem:auto = const char *, range:auto = std::vector<int>]
    9 | void print(auto const rem, auto const& range)
      |      ^
/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include/c++/v1/print:343:28: note: candidate function [with _Args = <std::vector<int> &>]
  343 | _LIBCPP_HIDE_FROM_ABI void print(format_string<_Args...> __fmt, _Args&&... __args) {
      |                            ^
test.cpp:24:5: error: call to 'print' is ambiguous
   24 |     print("y: ", y);
      |     ^~~~~
test.cpp:9:6: note: candidate function [with rem:auto = const char *, range:auto = std::list<std::string>]
    9 | void print(auto const rem, auto const& range)
      |      ^
/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include/c++/v1/print:343:28: note: candidate function [with _Args = <std::list<std::string> &>]
  343 | _LIBCPP_HIDE_FROM_ABI void print(format_string<_Args...> __fmt, _Args&&... __args) {
      |                            ^
test.cpp:25:5: error: call to 'print' is ambiguous
   25 |     print("z: ", z);
      |     ^~~~~
test.cpp:9:6: note: candidate function [with rem:auto = const char *, range:auto = std::array<char, 6>]
    9 | void print(auto const rem, auto const& range)
      |      ^
/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include/c++/v1/print:343:28: note: candidate function [with _Args = <std::array<char, 6> &>]
  343 | _LIBCPP_HIDE_FROM_ABI void print(format_string<_Args...> __fmt, _Args&&... __args) {
      |                            ^
test.cpp:38:5: error: call to 'print' is ambiguous
   38 |     print("\nAfter modification, z: ", z);
      |     ^~~~~
test.cpp:9:6: note: candidate function [with rem:auto = const char *, range:auto = std::array<char, 6>]
    9 | void print(auto const rem, auto const& range)
      |      ^
/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include/c++/v1/print:343:28: note: candidate function [with _Args = <std::array<char, 6> &>]
  343 | _LIBCPP_HIDE_FROM_ABI void print(format_string<_Args...> __fmt, _Args&&... __args) {
      |                            ^
4 errors generated.
```

print函数报错了，和zip完全不相关，难道说cppreference上例子会有这么明显的错误？但检查了一下`print`也只用到了早就支持的c++20的语法并不存在错误，而且换成gcc和Linux上的clang18之后都能正常编译。

这还只是第一个点异常，仔细阅读报错信息就会发现第二点了：我们没有导入c++23的新标准库`<print>`，为什么我们自定义的`print`会和`std::print`冲突呢？

看到这里是不是已经按耐不住自己想转投Rust的心了？不过别急，尽管报错很离奇但原因没那么复杂，听我慢慢解释。

## 基础回顾

基础回顾是c++博客少不了的环节，因为语法太多太琐碎，不回顾下容易看不懂后续的内容。

### 限定和非限定名称

第一个要回顾的是`限定名称`和`非限定名称`这两个概念。国内有时候也会把非限定名称叫做无限定名称，我觉得后者更符合中文的语用习惯，不过我这儿一直非限定非限定的习惯了所以就不改了。

如果要照着标准规范念经，那可有得念了，所以我会有通俗易懂的方式解释，这样多少会和真正的标准有那么点出入，还请语言律师们海涵。

简单的说，c++里如果一个标识符光秃秃的，比如`print`，那么它是非限定名称；而如果一个名字前面包含命名空间限定符，比如`::print, std::print, classA::print`，那么它是限定名称。

他俩有啥区别呢？限定名称的限定指的是指定了这标识符出现在那个命名空间/类里，编译器只能去限定的地方查找，没找到就是编译错误。而非限定名称，因为没限制编译器去哪找这个标识符，所以编译器会从当前作用域开始，一路往上走查找每个父作用域/类以找到这个标识符，注意同级的命名空间/类不会进行搜索。

举个例子：

```c++
#include <iostream>

namespace A {
    int a = 1;
    int b = 2;
    namespace B {
        int b = 3;

        void print()
        {
            std::cout << b << '\n'; // 非限定名称，就近找到A::B::b
            std::cout << a << '\n'; // 非限定名称，找到父命名空间的A::a
            std::cout << A::b << '\n'; // 限定名称，直接找到A::b
            // 下面这行会报错，因为使用了限定名称，只允许编译器搜索B，B中没有a
            // std::cout << B::a << '\n';
        }
    }
}

int main()
{
    A::B::print(); // 这也是限定名称
    // 输出 3 1 2
}
```

顺带一提每个编译单元都有一个默认存在的匿名的命名空间，所有没有明确定义在其他命名空间中的标识符都会被归入这个匿名的命名空间。举个例子，前文里我们定义的`print`函数就是在这个匿名的命名空间中，这个空间和`std`是平级关系。

非限定名称可以让程序员以自然的方式引入外层作用域的名字，而限定名称则提供了一个防止名称冲突的机制。

### ADL

理解了限定和非限定名称，下面我们再看看这行代码：

```c++
std::cout << A::b << '\n';
```

注意那个`<<`，c++允许进行运算符重载，所以它的真身其实是`std::ostream& operator<<(...)`，并且这个运算符是定义在`std`这个命名空间中的。

因为我们没有限定运算符的命名空间（按照运算符当前的调用方式我们也没法进行限定），所以编译器会从当前作用域开始逐层往上查找。但我们的代码中没有定义过这个运算符，std则不在非限定名称的搜索范围内，理论上编译器不应该报错说找不到`operator<<`吗？

事实上程序可以正常编译，因为c++还有另外一套名称查找策略，叫ADL——Argument Dependent Lookup。

简单的说，如果一个函数/运算符是非限定名称，而它的实际参数的类型所在的命名空间里定义有同名的函数，那么编译器就会把这个和实参类型在同一空间的函数当成这个非限定名称指代的函数/运算符。当然真实环境下编译器还得考虑可见性和函数重载决议，这里我们不细究了。

还是以上面那行代码为例，虽然我们没有重载`<<`，但`<iostream>`里有在std里重载，而我们的实际参数是`std::cout`，类型是`std::ostream&`，所以ADL会去命名空间std中查找是否有符合调用形式的`operator<<`，编译器会发现正好有完全合适的运算符存在，所以编译成功不会报错。

另外ADL只适用于函数和运算符（也算一种特殊的函数），lambda、functor等东西触发不了ADL。

ADL最大的用处是方便了运算符重载的使用。否则，我们不得不写很多`std::operator<<(a, b)`这样的代码，这既繁琐又不符合自然习惯。此外c++还有一些基于ADL的惯用法，例如我之前介绍过的[copy-and-swap](./做个地道的c++程序猿：copy%20and%20swap惯用法.md)惯用法。

不过除了少数正面作用，ADL更多的时候是个trouble maker，本文开头那个报错就是活生生的例子。

## 报错原因

复习完基础我们再看报错信息：

```console
test.cpp:23:5: error: call to 'print' is ambiguous
   23 |     print("x: ", x);
      |     ^~~~~
test.cpp:9:6: note: candidate function [with rem:auto = const char *, range:auto = std::vector<int>]
    9 | void print(auto const rem, auto const& range)
      |      ^
/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include/c++/v1/print:343:28: note: candidate function [with _Args = <std::vector<int> &>]
  343 | _LIBCPP_HIDE_FROM_ABI void print(format_string<_Args...> __fmt, _Args&&... __args) {
      |                            ^
```

我们的`x,y,z`都是std里的容器类的实例，`print`是非限定名称，于是非限定名称的查找触发，找到了我们定义的print，ADL也被触发，因为编译器要找出所有可行的函数或者函数模板然后用重载决议确定调用哪一个，于是c++23的新函数`std::print`被找到。

不巧的是两个函数虽然参数形式不太一样，但谁也不比谁更特殊化，导致出现调用的二义性，编译器不知道该用我们的模板函数还是标准库的，报错了。

正是ADL把我们不需要的函数加入了重载决议过程，cppreference上那段代码才会报错。

## 排查和处理

首先要排查问题是谁引起的。

看起来锅全是ADL的，但引入了`<print>`的家伙其实要分一半的锅，因为不引入这东西我们的代码里是没有`std::print`的，编译器就算用了ADL也不会看到这个干扰项。

那么多头文件，一个个看是看不完根本看不完。不过我们能缩小范围。

`std::print`是输出相关的，标准库实际上有一定要求不能随便乱include文件，所以我们可以先锁定`<iostream>`；其次标准库的容器有时候会对一些模板做特殊化，这些特殊化的模板当然也能被ADL找出来，所以容器的头文件也需要检查，万一他们特殊处理了`std::print`也说不定，不过鉴于vector，array，list都报错了，那说明我们只需要看其中一个就行，我选择`<array>`，因为比起另外两个`std::array`的结构更简单功能相对也少一些，所以代码也相对更少更方便检查。

我先检查了`<array>`和它include的所有文件，并未发现`<print>`。

所以我又检查了`<iostream>`，bingo，罪魁祸首是它include的`<ostream>`：

```cpp
#if _LIBCPP_STD_VER >= 23
#  include <__ostream/print.h>
#endif
```

检测到在用c++23就导入`<__ostream/print.h>`，而这个头文件里直接`#include <print>`了。

原因找到，现在该想想如何修复了。

修起来也简单，要么让我们自定义的print更加特殊使其在重载决议中胜出，要么使用限定名称直接屏蔽掉std，或者干脆给函数改个名字。

我只是想试试编译器支不支持新的`ranges`函数，懒劲发作不想动脑子，所以选了第二种，毕竟加个`::`就完事了：

```diff
int main()
{
    auto x = std::vector{1, 2, 3, 4};
    auto y = std::list<std::string>{"α", "β", "γ", "δ", "ε"};
    auto z = std::array{'A', 'B', 'C', 'D', 'E', 'F'};

-   print("Source views:", "");
-   print("x: ", x);
-   print("y: ", y);
-   print("z: ", z);
+   ::print("Source views:", "");
+   ::print("x: ", x);
+   ::print("y: ", y);
+   ::print("z: ", z);

-   print("\nzip(x,y,z):", "");
+   ::print("\nzip(x,y,z):", "");

    for (std::tuple<int&, std::string&, char&> elem : std::views::zip(x, y, z))
    {
        std::cout << std::get<0>(elem) << ' '
                  << std::get<1>(elem) << ' '
                  << std::get<2>(elem) << '\n';

        std::get<char&>(elem) += ('a' - 'A'); // modifies the element of z
    }

-   print("\nAfter modification, z: ", z);
+   ::print("\nAfter modification, z: ", z);
}
```

修改后的代码可以用g++和clang正常编译，不再会报错。

## 为什么不能乱include

现代C++ IDE一般都会在你include没用的头文件时给出提示或警告，这不仅仅是因为会拖累编译速度。

上面的例子告诉你了：include了没用的东西有时候会影响c++的名称查找导致莫名其妙的错误。

但话说回来，同样的代码g++并未报错，为啥呢，因为g++用的libstdc++直接实现了`std::print`对`std::ostream`的重载，而没`#include <print>`，事实上从libstdc++的代码来看这个include也没有必要。Linux上的clang除非特殊指定否则和g++用的同一套标准库代码，所以没有报错。macOS上的clang用的是libcxx，就遇上问题了。

当然我没看libcxx的代码不好说它这个include是对是错，也许它的代码里不得不这样做也未可知。

## 总结

c++就像古神，要不是我正好熟悉这块的语言规则好奇心也比较重，这个诡异的报错就要让我陷入疯狂了。

cppreference上的例子如果有人有兴趣可以尝试下修改，推荐选择给`print`函数重命名这个方案，这也是为社区做贡献的一次好机会。链接在这里：[link](https://en.cppreference.com/w/cpp/ranges/zip_view.html)

当然我懒抽筋了，这个机会就让给有缘人喽。
