`std::shared_ptr`智能指针是c++11一个相当重要的特性，可以极大地将开发者从资源申请/释放的繁重劳动中解放出来。

然而直到c++17前`std::shared_ptr`都有一个严重的限制，那就是它并不支持动态数组：

```c++
#include <memory>

std::shared_ptr<int[]> sp1(new int[10]()); // 错误，c++17前不能传递数组类型作为shared_ptr的模板参数
std::unique_ptr<int[]> up1(new int[10]()); // ok, unique_ptr对此做了特化

std::shared_ptr<int> sp2(new int[10]()); // 错误，可以编译，但会产生未定义行为，请不要这么做
```

`sp1`错误的原因很明显，然而`sp2`的就没有那么好找了，究其原因，是因为`std::shared_ptr`对非数组类型都使用`delete p`释放资源，显然这对于`new int[10]`来说是不对的，对它应该使用`delete [] p`。

其实c++17前的解决方案并不复杂，我们可以借助`std::default_delete`，它用于提供对应类型的正确的delete操作：

```c++
std::shared_ptr<int> sp3(new int[10](), std::default_delete<int[]>());
```

现在我们提供了正确的delete操作，可以放心地使用了。

不过这么做的缺点也是很明显的：

1. 我们想管理的值是int[]类型的，然而事实上传给模板参数的是int
2. 需要显示提供delete functor
3. 不能使用`std::make_shared`，无法保证异常安全
4. c++17前shared_ptr未提供`opreator[]`，所以当需要类似操作时不得不使用`sp3.get()[index]`的形式

事实上共享一片连续分配内存的需求是极为常见的，所以为了修正上述缺陷，c++17以及即将推出的c++2a对`std::shared_ptr`做了完善。

先说c++17的改进，shared_ptr增加了`opreator[]`，并可以使用`int[]`类的数组类型做模板参数，所以`sp3`的定义可以简化了：

```c++
std::shared_ptr<int[]> sp3(new int[10]());
```

对于访问分配的空间，可以将`sp3.get()[index]`替换为`sp3[index]`。看个具体的例子：

```c++
#include <iostream>
#include <memory>

int main()
{
    std::shared_ptr<int[]> sp(new int[5]());
    for (int i = 0; i < 5; ++i) {
        sp[i] = (i+1) * (i+1);
    }

    for (int i = 0; i < 5; ++i) {
        std::cout << sp[i] << std::endl;
    }
}
```

我们分配一个有5个int元素的动态数组，然后分别赋值1-5的平方，然后输出：

```bash
g++ -Wall -std=c++17 test.cpp
./a.out

1
4
9
16
25
```

使用被极大得简化了，然而还是有点问题，那就是无法使用`std::make_shared`，而我们除非指定自己的delete functor，否则我们应该尽量使用`std::make_shared`。

所以c++20对此做了改进：

```c++
auto up2 = std::make_unique<int[]>(10); // 从c++14开始，分配一个管理有10个int元素的动态数组的unique_ptr

// c++2a中你可以这样写，与上一句相似，只不过返回的是shared_ptr
auto sp3 = std::make_shared<int[]>(10);
```

在我的编译器上（GCC 8.2.1）还不能支持这一特性，所以很遗憾得不能提供演示了。

不过等c++2a(很可能就叫c++20)发布后`std::shared_ptr`就能安全而便捷地管理动态数组了。
