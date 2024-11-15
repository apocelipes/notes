`static_cast`和c-style转换都是一种直接初始化，因此它们优先使用构造函数，没有可用的构造函数时才使用转换运算符。

```c++
#include <iostream>

struct B;

struct A {
    A() = default;
    A(const struct B&) {
        std::cout << "called A::A(const struct B&)\n";
    }
};

struct B {
    B() = default;
    operator A() const {
        std::cout << "called B::operator A()\n";
        return A{};
    }
};

int main()
{
    B b;
    [[maybe_unused]] auto a1 = static_cast<A>(b);
    [[maybe_unused]] auto a2 = (A)b;
}
```

两者都调用`A::A(const struct B&)`，其他转换下因为有两个可选函数，会导致二义性编译会报错。

如果未定义`A::A(const struct B&)`，则使用B的转换运算符，但如果`A::A(const struct B&)`被删除，则编译报错：

```c++
struct A {
    A() = default;
    A(const struct B&) = delete;
};
```

报错：

```text
a.cpp: In function ‘int main()’:
a.cpp:21:48: error: use of deleted function ‘A::A(const B&)’
   21 |     [[maybe_unused]] auto a1 = static_cast<A>(b);
      |                                                ^
a.cpp:7:5: note: declared here
    7 |     A(const struct B&) = delete;
      |     ^
```

通常定义其中一个就行。

#### 参考

<https://en.cppreference.com/w/cpp/language/cast_operator>

<https://en.cppreference.com/w/cpp/language/direct_initialization>
