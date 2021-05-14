最近在使用constexpr的时候无意中踩了个小坑。

下面给个小示例：

```c++
#include <iostream>

constexpr int n = 10;
constexpr char *msg = "Hello, world!";

int main()
{
    for (auto i = 0; i < n; ++i) {
        std::cout << msg << std::endl;
    }
}
```

constexpr应该是大家很熟悉的东西了，也是最常用的c++11新特性之一。和宏相比除了更强的类型安全之外，constexpr还带来了编译期计算。

上面的代码相当简单，我们循环输出“Hello, world!”这个字符串10次。

这么简单的代码还有讨论的必要吗？一开始我也是这么想的，然而当我们编译运行的时候却会得到下面这样的警告：

```bash
$ g++ --version
                      
g++ (GCC) 10.2.0
Copyright (C) 2020 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.


$ g++ -std=c++17 -Wall -Wextra test.cpp

test.cpp:4:23: 警告: ISO C++ forbids converting a string constant to ‘char*’ [-Wwrite-strings]
    4 | constexpr char *msg = "Hello, world!";
      |                       ^~~~~~~~~~~~~~~
```

这段信息的意思是c++不允许把字符串字面量赋值给`char*`。

然而对于constexpr，[文档](https://en.cppreference.com/w/cpp/language/constexpr)中是这么写的：

> A constexpr specifier used in an object declaration implies const.

这里的object你可以理解为变量，意思是constexpr修饰的变量都会隐式添加一个const限定符。

也就是说：

```c++
// T 是任意类型
constexpr T a = xxx;
// 不考虑其他因素，在类型上等价于：
T const a = xxx;
```

我们这里的`T`实际上可以填任意类型，包括指针。这不是说明我们的指针变量有const吗？

眼尖的读者大概已经知道答案了：constexpr添加的是顶层const。

所以我们的代码实际上是这样的：

```c++
// 原本的代码
constexpr char *msg = "Hello, world!";
// 实际上的效果
char * const msg = "Hello, world!";
```

下面一行的`msg`实际上是一个指向`char`的指针常量，而我们可以通过它任意修改被指向的字符串（当然这是未定义行为）。指针常量意味着我们不能把这个指针重新指向其他的对象，这个const作用在指针本身上，因此叫做顶层const。

而字符串常量的类型是`const char[N]`，在表达式里退化为`const char *`，这表示一个指向常量字符串的指针，这里的const的底层的，因为它作用于被指向的对象而不是我们的指针自身。

对于顶层const，赋值的时候是可以被去除的，而底层const则不行，这就是为什么编译器会弹出警告的原因了。

正确的做法也很简单，牢记constexpr不是const的等价替代品，它只会添加顶层const，不会添加底层const。

所以constexpr的字符串常量应该这样写：

```c++
constexpr const char *p = "Hello, world!";
```

或者你的编译环境支持c++17，我更推荐你这样写：

```c++
#include <string_view>

constexpr std::string_view msg = "Hello, world!";
```

使用`string_view`之后就不会出现上面的顶层/底层const的坑了。所以在现代c++里能不用裸指针就尽量不要用。

##### 参考

<https://stackoverflow.com/questions/54258241/warning-iso-c-forbids-converting-a-string-constant-to-char-for-a-static-c>
