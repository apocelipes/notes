c++23新增了一些智能指针适配器，用来扩展和简化智能指针的使用。

这次主要介绍的是`std::out_ptr`和`std::inout_ptr`。这两个适配器用法和实现都很简单，但网上的文档都比较抱歉，还缺少一些比较重要的部分，因此单开一篇文章记录一下。

## out_ptr

首先从功能最简单的`out_ptr`讲起。

`std::out_ptr`其实是一个函数，返回一个类型为`std::out_ptr_t`的智能指针适配器，函数签名如下：

```c++
#include <memory>

template< class Pointer = void, class Smart, class... Args >
auto out_ptr( Smart& s, Args&&... args );
```

这个函数主要是把各种智能指针包装成output parameter，以方便现有的接口使用，尤其是一些用c语言写的函数。

在继续之前我们先来复习一下output parameter是什么。这东西又叫传出参数，一次就是函数会把一部分数据写进自己的参数里返回给调用者。

通过参数返回是因为c语言和c++11之前的c++不支持多值返回也没有类似tuple这样方便的数据结构，导致函数无法直接返回两个以上的值，所以需要用一种额外的传递数据的方式。

比如我在[以前的博客](./glibc自带哈希表的用例及性能测试.md)中提到的hsearch：`int hsearch_r(ENTRY item, ACTION action, ENTRY **retval, struct hsearch_data *htab)`。这个函数用来在哈希表里创建或者查找数据，查找失败的时候会返回错误码，而查找成功的时候函数返回0并把找到的数据设置给`retval`。这个`retval`就是output parameter，承载了函数除了错误码之外的返回数据。

c++里现在很少用指针类型作为output parameter了，但还有更本地化的做法——引用：`int func(const char *name, Data &retval)`。

这类函数有几个特点：

1. 不在乎output parameter里有什么值
2. 函数调用期间完全享有output parameter和其资源的所有权
3. 函数返回后output parameter通常被设置为新值

在c++提倡少用裸指针的今天，我们越来越习惯使用shared_ptr和unique_ptr，但不管哪种智能指针都很难直接适配上面这些函数，看个例子就明白了：

```c++
int get_data(const std::string &name, Data **retval)
{
    if (!check_name(name)) {
        return ErrCheckFailed;
    }
    *retval = make_data(name);
    return 0;
}

// 使用裸指针
Data *data_ptr = nullptr;
if (auto err = get_data("name", &data_ptr); err != 0) {
    错误处理
} else {
    这里可以使用data_ptr
}
```

使用裸指针的时候代码比较简单，我们再来看看使用智能指针的时候：

```c++
std::unique_ptr<Data> resource;

Data *data_ptr = nullptr;
if (auto err = get_data("name", &data_ptr); err != 0) {
    错误处理
} else {
    resource.reset(data_ptr);
    这里可以使用resource
}
```

代码会变得啰嗦，而且如果我们忘记了调用reset，那么资源就可能泄漏了；还有最重要的一点，我们主动使用了裸指针，而这正是我们想避免的。

这时候就需要`out_ptr`了。`out_ptr`生成的适配器会先放弃智能指针持有资源的所有权并将旧资源释放，因为如前面所说我们要调用的函数会接管资源的所有权，接着构造出的`std::out_ptr_t`有自动的类型转换方法，可以把智能指针转换成我们需要的`T**`交给函数使用，最后在函数调用结束之后再把新的资源设置回智能指针。

所以上面的例子可以改成：

```c++
std::unique_ptr<Data> resource;
if (auto err = get_data("name", std::out_ptr(resource)); err != 0) {
    错误处理
} else {
    这里可以使用resource，无需reset
}
```

除了代码更简洁，`out_ptr`还保证异常安全，即使在调用`get_data`的过程中抛出了异常，也不会出现资源泄漏。

利用`out_ptr`我们可以在使用智能指针的同时兼容老旧接口。

## out_ptr和shared_ptr

如果只看函数签名，很多人会觉得`out_ptr`也可以直接配合`std::shared_ptr`使用，然而现实是多变的：

```c++
struct Data {
    std::string name;
};

int get_data(const std::string &name, Data **retval)
{
    if (name == "")
        return 1;
    *retval = new Data{name};
    return 0;
}

int main()
{
    std::shared_ptr<Data> resource;
    if (auto err = get_data("apocelipes", std::out_ptr(resource)); err != 0)
        std::cerr << "error\n";
    else
        std::cout << "success, name: " << resource->name << "\n";
}
```

上面的代码无法通过编译：

```console
$ clang++ -std=c++23 test.cpp

In file included from test.cpp:2:
In file included from /Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/v1/memory:948:
/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/v1/__memory/out_ptr.h:38:17: error: static assertion failed due to requirement '!__is_specialization_v<std::shared_ptr<Data>, shared_ptr> || sizeof...(_Args) > 0': Using std::shared_ptr<> without a deleter in std::out_ptr is not supported.
   38 |   static_assert(!__is_specialization_v<_Smart, shared_ptr> || sizeof...(_Args) > 0,
      |                 ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/v1/__memory/out_ptr.h:93:10: note: in instantiation of template class 'std::out_ptr_t<std::shared_ptr<Data>, Data *>' requested here
   93 |   return std::out_ptr_t<_Smart, _Ptr, _Args&&...>(__s, std::forward<_Args>(__args)...);
      |          ^
test.cpp:19:48: note: in instantiation of function template specialization 'std::out_ptr<void, std::shared_ptr<Data>>' requested here
   19 |     if (auto err = get_data("apocelipes", std::out_ptr(resource)); err != 0)
      |                                                ^
1 error generated.
```

报错虽然很长但只要关注前几行就行了，错误的原因很明显，`std::shared_ptr`要配合`out_ptr`使用就必须显示提供deleter。

这是因为对于`std::shared_ptr`，deleter并不是类型的一部分，通常是我们通过构造函数或者reset方法穿进去的，为了能100%正确释放资源，我们需要手动把合适的deleter传进去；相对地deleter是`std::unique_ptr`类型的一部分，out_ptr可以直接从类型参数里得到合适的deleter从而正确释放资源。

这也是为什么`out_ptr`还有变长参数，这些参数就是为了`std::shared_ptr`或者其他有特殊要求的类似智能指针准备的。

好在上面的代码稍作修改就能正常使用：

```diff
int main()
{
    std::shared_ptr<Data> resource;
-   if (err = get_data("apocelipes", std::out_ptr(resource)); err != 0)
+   if (err = get_data("apocelipes", std::out_ptr(resource, std::default_delete<Data>{})); err != 0)
        std::cerr << "error\n";
    else
        std::cout << "success, name: " << resource->name << "\n";
}
```

`std::default_delete<T>`会调用`delete`或者`delete[]`来释放资源，正好我们这里可以利用它。shared_ptr平时也默认使用的这个。

修改很简单，但网上讲这点的文档不多，因此多记一笔。另外基于`out_ptr`会临时转移所有权这点来看，共享所有权模型的`std::shared_ptr`其实并不适合使用`out_ptr`，虽然标准没有禁止甚至还要求额外做检测（用于初始化shared_ptr），但我仍然建议把`std::shared_ptr`和`std::out_ptr`一起使用看做一种坏味道，尽量避免这种用例。

## inout_ptr

`inout_ptr`的名字比较抽象，但只是在`out_ptr`的基础上加了个“in”而已。它会返回一个`std::inout_ptr_t`类型的对象，函数签名如下：

```c++
#include <memory>

template< class Pointer = void, class Smart, class... Args >
auto inout_ptr( Smart& s, Args&&... args );
```

这个“in”是指使用output parameter的函数在重新设置参数的值之前会先使用他们，因此这些函数的特点是：

1. **非常在乎output parameter里有什么值，根据这些值执行不同的操作**
2. 函数调用期间完全享有output parameter和其资源的所有权
3. 函数返回后output parameter**不变**或者被设置为新值

还是看例子，我们对`Data`增加一个`update_data`函数，如果name是`recreate`则删除原来的对象重新创建一个：

```c++
int update_data(Data **data)
{
    if (data == nullptr || *data == nullptr)
        return 1;
    if ((*data)->name == "recreate") {
        delete *data;
        *data = new Data{"apocelipes"};
        return 2; // 代表已修改
    }
    return 0;
}
```

现实中没人这么写代码，但存在很多类似的c接口，而且我们也很难控制第三方库的代码质量，难免不会遇上类似的东西。如果想在这种接口上用智能指针，那只能说有福了：

```c++
auto resource = std::make_unique<Data>("recreate");

Data *ptr = resource.get();
resource.release(); // 释放所有权，但不释放资源
if (auto code = update_data(&ptr); code == 1)
    std::cerr << "error\n";
else if (code == 2) {
    resource.reset(ptr);
    std::cout << "updated, name: " << resource->name << "\n";
} else {
    resource.reset(ptr);
    std::cout << "updated, name: " << resource->name << "\n";
}
```

可以看到代码会变得很复杂，而且一但忘记使用reset就会内存错误。这时候我们就需要`inout_ptr`帮忙了。

`inout_ptr`整体上和`out_ptr`差不多，都是让出资源的所有权然后重新把函数返回的值设置回去，但还有几个差异：

1. 前面说过需要`inout_ptr`的函数是需要参数的值的，因此构造`inout_ptr_t`时之后放弃资源的所有权，不会像`out_ptr`那样释放资源本身
2. 资源的释放是调用的函数的责任，`inout_ptr`只会把函数返回出来的值重新设置回智能指针

用`inout_ptr`改写后的代码如下：

```c++
auto resource = std::make_unique<Data>("recreate");

if (auto code = update_data(std::inout_ptr(resource)); code == 1)
    std::cerr << "error\n";
else if (code == 2) {
    std::cout << "updated, name: " << resource->name << "\n";
} else {
    std::cout << "updated, name: " << resource->name << "\n";
}
```

代码看起来清爽多了。

另外虽然`inout_ptr`也有变长参数，但标准明确规定它不能配合`std::shared_ptr`使用，这些参数`std::unique_ptr`用不上，是预留给其他的第三方的类似指针对象使用的。

## 注意事项

除了`std::shared_ptr`配合`out_ptr`使用时需要传入deleter，还有一个注意事项。

两个适配器都不建议这么用：

```c++
auto out = std::out_ptr(resource);
func(out);
```

因为他们都是在析构函数里重新设置智能指针的值，如果绑定到一个局部变量或者其他存储器的变量上，函数调用结束就无法把正确的值重新设置回智能指针，这会导致严重的内存错误。

唯一建议的用法是直接使用`out_ptr`和`inout_ptr`的返回值：`func(std::out_ptr(resource))`，这样函数调用结束后表达式结束，返回值作为表达式中创建的临时变量会被析构，这样智能指针的值就被正常设置了。

尽管只要在转换操作符上加上一点限制就能避免误用，但标准考虑到了各种边缘情形，最终没有添加限制，所以我们只能牢记这条注意事项避免踩坑了。

## 总结

说实话这两个适配器有很浓的给c库函数擦屁股的意味，甚至标准文档上直接拿`fopen_s`做例子了，我们看下它的函数声明就能秒懂：`errno_t fopen_s( FILE *restrict *restrict streamptr, const char *restrict filename, const char *restrict mode );`。

另外这两个适配器虽然叫智能指针适配器，但也可以对普通裸指针使用，不过我不推荐这种用法。

最后虽然它们的用法都比较偏，但真要用的时候还都有用，所以了解一下总是没坏处的。而且它们的源代码也很简单，有兴趣可以看看libcxx的实现，虽然相比其他家的有点啰嗦，但可读性很强：

out_ptr: <https://github.com/llvm/llvm-project/blob/main/libcxx/include/__memory/out_ptr.h>

inout_ptr: <https://github.com/llvm/llvm-project/blob/main/libcxx/include/__memory/inout_ptr.h>

##### 参考资料

<https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/n4950.pdf> p.643
