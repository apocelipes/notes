如果你正在寻找一款c++性能测试工具，那么这篇文章是不容错过的。

市面上的benchmark工具或多或少存在一些使用上的不便，那么是否存在一个使用简便又功能强大的性能测试工具呢？答案是[google/benchmark](https://github.com/google/benchmark)。

google/benchmark是一个由Google开发的基于googletest框架的c++ benchmark工具，它易于安装和使用，并提供了全面的性能测试接口。

下面我将介绍google/benchmark的安装并用一个简短的例子介绍它的简单使用。

<h2 id="install">安装google/benchmark</h2>
google/benchmark基于c++11标准和googletest框架，所以安装前需要先做一些准备工作。

首先是安装g++和cmake。

Debian/Ubuntu:

```bash
sudo apt install g++ cmake
```

Arch Linux/Manjaro Linux:

```bash
sudo pacman -s g++ cmake
```

确保你的g++版本在5.0以上，否则可能不能很好地支持c++11的某些特性。

然后是googletest框架，你可以选择单独安装，不过这里我选择将其作为benchmark源码树的依赖而不单独安装它，因为benchmark在编译安装时需要googletest但是在使用时并不需要，为了篇幅我们选择后者。

准备工作完成后选择一个合适的目录，然后运行下面的命令：

```bash
git clone https://github.com/google/benchmark.git
git clone https://github.com/google/googletest.git benchmark/googletest
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=RELEASE ../benchmark
make -j4
# 如果想全局安装就接着运行下面的命令
sudo make install
```

头文件会被安装至`/usr/local/include`，库文件会安装至`/usr/local/lib`。

现在安装完成了，我们来看看benchmark如何使用。

<h2 id="usage">google/benchmark的简单使用</h2>
我们的例子将会对比三种访问`std::array`容器内元素方法的性能，进而演示benchmark的使用方法。

先看代码：

```c++
#include <benchmark/benchmark.h>
#include <array>

constexpr int len = 6;

// constexpr function具有inline属性，你应该把它放在头文件中
constexpr auto my_pow(const int i)
{
    return i * i;
}

// 使用operator[]读取元素，依次存入1-6的平方
static void bench_array_operator(benchmark::State& state)
{
    std::array<int, len> arr;
    constexpr int i = 1;
    for (auto _: state) {
        arr[0] = my_pow(i);
        arr[1] = my_pow(i+1);
        arr[2] = my_pow(i+2);
        arr[3] = my_pow(i+3);
        arr[4] = my_pow(i+4);
        arr[5] = my_pow(i+5);
    }
}
BENCHMARK(bench_array_operator);

// 使用at()读取元素，依次存入1-6的平方
static void bench_array_at(benchmark::State& state)
{
    std::array<int, len> arr;
    constexpr int i = 1;
    for (auto _: state) {
        arr.at(0) = my_pow(i);
        arr.at(1) = my_pow(i+1);
        arr.at(2) = my_pow(i+2);
        arr.at(3) = my_pow(i+3);
        arr.at(4) = my_pow(i+4);
        arr.at(5) = my_pow(i+5);
    }
}
BENCHMARK(bench_array_at);

// std::get<>(array)是一个constexpr function，它会返回容器内元素的引用，并在编译期检查数组的索引是否正确
static void bench_array_get(benchmark::State& state)
{
    std::array<int, len> arr;
    constexpr int i = 1;
    for (auto _: state) {
        std::get<0>(arr) = my_pow(i);
        std::get<1>(arr) = my_pow(i+1);
        std::get<2>(arr) = my_pow(i+2);
        std::get<3>(arr) = my_pow(i+3);
        std::get<4>(arr) = my_pow(i+4);
        std::get<5>(arr) = my_pow(i+5);
    }
}
BENCHMARK(bench_array_get);

BENCHMARK_MAIN();
```

我们可以看到每一个benchmark测试用例都是一个类型为`std::function<void(benchmark::State&)>`的函数，其中`benchmark::State&`负责测试的运行及额外参数的传递。

随后我们使用`for (auto _: state) {}`来运行需要测试的内容，state会选择合适的次数来运行循环，时间的计算从循环内的语句开始，所以我们可以选择像例子中一样在for循环之外初始化测试环境，然后在循环体内编写需要测试的代码。

测试用例编写完成后我们需要使用`BENCHMARK(<function_name>);`将我们的测试用例注册进benchmark，这样程序运行时才会执行我们的测试。

最后是用`BENCHMARK_MAIN();`替代直接编写的main函数，它会处理命令行参数并运行所有注册过的测试用例生成测试结果。

示例中大量使用了constexpt，这是为了能在编译期计算出需要的数值避免对测试产生太多噪音。

然后我们编译测试程序：

```bash
g++ -Wall -std=c++14 -pthread benchmark_example.cpp -lbenchmark
```

benchmark需要链接`libbenchmark.so`，所以需要指定`-lbenchmark`，此外还需要thread的支持，因为libstdc++不提供thread的底层实现，我们需要pthread。另外不建议使用`-lpthread`，官方表示会出现兼容问题，在我这测试也会出现链接错误。

如果你是在Windows平台使用google/benchmark，那么你需要额外链接`shlwapi.lib`才能使benchmark正常编译和运行。详细信息在[这里](https://github.com/google/benchmark/issues/202)。

文件名最好在`-lbenchmark`之前，否则会出现链接找不到符号的问题。

编译好程序后就可以运行测试了：

![benchmark](../../images/c++benchmark/benchmark1)

显示的警告信息表示在当前系统环境有一些噪音（例如其他在运行的程序）可能导致结果不太准确，并不影响我们的测试。

在Windows上通常没有上述警告，如果你需要在Linux平台上去除相关警告的话，请参考[此处](https://github.com/google/benchmark#disabling-cpu-frequency-scaling)。

测试结果与预期基本相符，`std::get`最快，`at()`最慢。

以上就是google/benchmark的安装和简单使用，下一篇文章中我会介绍更多使用benchmark的技巧，如有错误欢迎指正。
