lambda即使代码一模一样，类型也是完全不同的。

```c++
template <typename T, auto = [](){}>
struct unique_t {
    using base_t = T;
};

typedef unique_t<int> A;
typedef unique_t<int> B;

int main()
{
    std::cout << std::boolalpha;
    std::cout << std::is_same_v<unique_t<int>, unique_t<int>> << '\n'; // false
    std::cout << std::is_same_v<unique_t<int>, A> << '\n'; // false
    std::cout << std::is_same_v<unique_t<int>, B> << '\n'; // false
    std::cout << std::is_same_v<A, B> << '\n'; // false
    std::cout << std::is_same_v<A, A> << '\n'; // true
    std::cout << std::is_same_v<B, B> << '\n'; // true
}
```

GCC下这段代码会产生下面这些类型：

```c++
std::is_same_v<unique_t<int, {lambda()#2}{}>, unique_t<int, {lambda()#3}{}> >
std::is_same_v<unique_t<int, {lambda()#2}{}>, unique_t<int, {lambda()#2}{}> >
std::is_same_v<unique_t<int, {lambda()#3}{}>, unique_t<int, {lambda()#3}{}> >
std::is_same_v<unique_t<int, {lambda()#4}{}>, unique_t<int, {lambda()#5}{}> >
std::is_same_v<unique_t<int, {lambda()#6}{}>, unique_t<int, {lambda()#2}{}> >
std::is_same_v<unique_t<int, {lambda()#7}{}>, unique_t<int, {lambda()#3}{}> >
```

这意味着lambda并不轻量，随手乱写会造成代码膨胀并浪费内存。比如find_if，大量的lambda不仅会生成大量不同的lambda类型，还会生成大量的find_if实例，代码会显著膨胀，不过实测性能影响很小（GCC上连续六次字面量调用在未开优化的情况下性能差异也不到2%，开启优化后几乎没有差异），而且clang和gcc在开启优化时能识别一部分类似情况，这使得实际使用时影响没有理论上大。
