先说结论，lambda是不能重载的（至少到c++23依旧如此，以后会怎么样没人知道）。而且即使代码完全一样的两个lambda也会有完全不同的类型。

但虽然不能直接实现lambda重载，我们有办法去模拟。

在介绍怎么模拟之前，我们先看看c++里的functor是怎么重载的。

首先类的函数调用运算符是可以重载的，可以这样写：

```c++
struct Functor {
    bool operator()(int i) const
    {
        return i % 2 == 0;
    }

    bool operator()(const std::string &s) const
    {
        return s.size() % 2 == 0;
    }
};
```

在此基础上，c++11还引入了using的新用法，可以把基类的方法提升至子类中，子类无需手动重写就可直接使用这些基类的方法：

```c++
struct IntFunctor {
    bool operator()(int i) const
    {
        return i % 2 == 0;
    }
};

struct StrFunctor {
    bool operator()(const std::string &s) const
    {
        return s.size() % 2 == 0;
    }
};

struct Functor: IntFunctor, StrFunctor {
    // 不需要给出完整的签名，给出名字就可以了
    // 如果在基类中这个名字已经有重载，所有重载的方法也会被引入
    using IntFunctor::operator();
    using StrFunctor::operator();
};

auto f = Functor{};
```

现在`Functor`可以直接使用`bool operator()(const std::string &s)`和`bool operator()(int i)`了。

现在可以看看怎么模拟lambda重载了：我们知道c++标准要求编译器把lambda转换成类似上面的`Functor`的东西，因此也能使用上面的办法模拟重载。

但还有两个致命问题：第一是需要写明需要继承的lambda的类型，这个当然除了模板之外是做不到的；第二是继承的基类的数量得明确给出这限制了灵活性，但可以用c++11添加的新特性——变长模板参数来解决。

解决上面两个问题其实很简单，方案如下：

```c++
template <typename... Ts>
struct Functor: Ts...
{
    using Ts::operator()...;
};

auto f = Functor<StrFunctor, IntFunctor>{};
```

使用变长模板参数后就可以继承任意多的类了，然后再使用`...`在类的内部逐个引入基类的函数调用运算符。

这样把继承的对象从普通的类改成lambda就可以模拟重载。但是怎么做呢，前面说了我们没法直接拿到lambda的类型，用decltype的话又会非常啰嗦。

答案是可以依赖c++17的新特性：CTAD。简单得说就是可以提前指定规则，让编译器从构造函数或者符合要求的构造方式里推导需要的类型参数。于是可以这样写：

```c++
template <typename... Ts>
Functor(Ts...) -> Functor<Ts...>;
```

箭头左边的是构造函数，右边的是推导出来的类型。

现在又有疑问了，`Functor`里不是没定义过任何构造函数吗？是的，正是因为没有定义，使得`Functor`符合条件成为了“聚合”（aggregate）。“聚合”可以做聚合初始化，形式类似：`聚合{基类1初始化，基类2初始化, ...，成员变量1的值，成员变量2的值...}`。

作为一种符合要求的初始化方式，也可以使用CTAD，但形式上会用圆括号包起来导致看着像构造函数。另外对于聚合，c++20会自动生成和上面一样的CTAD规则无需再手写。

现在把所有代码组合起来：

```c++
template <typename... Ts>
struct Functor: Ts...
{
    using Ts::operator()...;
};

int main()
{
    const double num = 2.0;
    auto f = Functor{
        [](int i) { return i+1; },
        [&num](double d) { return d+num; },
        [s = std::string{}](const std::string &data) mutable {
            s = data + s;
            return s;
        }
    };

    std::cout << f(1) << '\n';
    std::cout << f(1.0) << '\n';
    std::cout << f("apocelipes!") << '\n';
    std::cout << f("Hello, ") << '\n';
    // Output:
    // 2
    // 3
    // apocelipes!
    // Hello, apocelipes!
}
```

有没有替代方案？c++17之后是有的，可以利用`if constexpr`或者`if consteval`对类型分别进行处理，编译器编译时会忽略其他分支，实际上这不是重载，但实现了类似的效果：

```c++
int main()
{
    auto f = []template <typename T>(T t) {
        if constexpr (std::is_same_v<T, int>) {
            return t + 1;
        }
        else if constexpr (std::is_same_v<T, std::string>) {
            return "Hello, " + t;
        }
        else {
            return t;
        }
    };
    std::cout << f(1) << '\n';
    std::cout << f("apocelipes") << '\n';
    std::cout << f(1.2) << '\n';
    // Output:
    // 2
    // Hello, apocelipes
    // 1.2
}
```

要注意的是这里的`f`本身并不是模板，f的`operator()`才是。这个方案除了啰嗦之外和上面靠继承的方案没有太大区别。

lambda重载有啥用呢？目前一大用处是可以简化`std::visit`的使用：

```c++
std::variant<int, long, double, std::string> v;
// 对v一顿操作
std::visit(Functor{
    [](int arg) { std::cout << arg << ' '; },
    [](long arg) { std::cout << arg << ' '; },
    [](double arg) { std::cout << std::fixed << arg << ' '; },
    [](const std::string& arg) { std::cout << std::quoted(arg) << ' '; }
}, v);
```

这个场景中需要一个callable对象，同时需要它的调用运算符有对应类型的重载，在这里不能直接用模板，所以我们的模拟lambda重载派上了用场。

如果要我推荐的话，我会选择继承的方式实现lambda重载，虽然一般不推荐使用多继承，但这里的多继承不会引发问题，而且可读性能获得很大提升，优势很明显，所以首选这种方案。
