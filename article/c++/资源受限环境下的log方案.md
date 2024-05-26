资源受限环境：内存或者cpu性能不足，导致无法输出字符串（尤其是格式化的字符串）的环境，嵌入式上多见。

## 方案一：输出id

用id映射到对应的字符串：

```c++
enum class LogID {
    MSG1,
    MSG2,
    MSG3
};

log.info(ModuleID, LogID.MSG1);
```

信息格式可以是下面这样：

```text
|---------|
| 模块ID  |
|---------|
| 日志ID  |
|---------|
| 附加参数 |
|---------|
```

优点：
- 简单易懂
- 占用资源很少
- 生成的程序里可以不包含这些字符串，只保留ID

缺点：
- 类型不安全
- 容易typo
- 扩展不方便，尤其是在删除或者变更已存在的字符串信息的时候

## 方案2：输出字符串地址

直接输出字符串常量的地址，代替id，然后用addr2line去找到对应的字符串是什么。

优点：
- 资源占用小
- 扩展比id映射要容易一些

缺点：
- addr2line需要debug信息，这个在生产环境尤其是嵌入式设备上是没有的
- 没有任何规定要求内容相同的字符串常量一定要有相同的地址，因此不可靠

这个方法没啥使用价值。

## 方案3：输出字符串hash

利用constexpr用固定的hash算法计算出字符串常量的hash，然后把所有msg的hash对应关系保存到配置文件之类的东西里。

```c++
consteval uint2_t hash(const char *msg)
{
    ...
    return hashValue;
}

log(moduleID, hash("an error info"));
```

优点：
- 编译期间计算，对性能无影响
- 不像id是连续的，hash是非连续的，非常容易扩展

缺点：
- 有hash冲突，而且真冲突了基本没法解决，所以能不能用这个方案得看所有信息里有没有hash冲突的

## 方案4：模板元编程

```c++
template <typename CharT, CharT... chars>
struct string_constant {
    // 内部可以有string_view
    // 先用静态内联的数组把所有字符dump下来
    constexpr static CharT storage[sizeof...(chars)+1] = {chars...}; // +1 可以当成NUL结尾字符串用
    constexpr static std::string_view sv{storage, sizeof...(chars)};
};
// 因为内部有string_view所以可以依赖它来构筑各种比较运算符

template <typename T, T... chars>
constexpr string_constant<T, chars...> operator""_sc()
{
        return {};
}

using MSG1 = decltype("error info 1"_sc);
using MSG2 = decltype("error info 2"_sc);
```

这个是string literal operator templates，是GNU扩展。可以把字符串常量分解成一个个字符。相同的字符串常量必然有相同的类型。

原生的c++11只有raw literal，只能用于数字字面量：

```c++
template <char... Cs> 
??? operator "" _x()

10_x => operator ""_x<'1', '0'>();
3.14_x => operator ""_x<'3', '.', '1', '4'>();
```

使用：

```c++
template <typename T>
int log(T);

template <>
int log(MSG1) {return 1;}
template <>
int log(MSG2) {return 2;}

log("error info 1"_sc);
log("error info 2"_sc);
```

还是输出id那套，但现在类型安全了，`nm -uC`可以输出所有的字符串字面量模板，然后可以建立对应关系。

优点：
- 使用上很灵活，可以接近于真正的字符串
- 类型安全
- 可以方便得配合id或者hash使用

缺点：
- 是GNU扩展，可移植性欠佳。但clang和gcc都在gnu++14及更高版本支持。
- 模板暴胀
- 会让编译时间变长

##### 参考资料

<https://stackoverflow.com/questions/39795893/when-and-how-to-use-a-template-literal-operator>

<https://learn.microsoft.com/en-us/cpp/cpp/user-defined-literals-cpp?view=msvc-170#user-defined-literal-operator-signatures>

<https://www.youtube.com/watch?v=Dt0vx-7e_B0>
