隐式类型转换可以说是我们的老朋友了，在代码里我们或多或少都会依赖c++的隐式类型转换。

然而不幸的是隐式类型转换也是c++的一大坑点，稍不注意很容易写出各种奇妙的bug。

因此我想借着本文来梳理一遍c++的隐式类型转换，复习的同时也避免其他人踩到类似的坑。

<blockquote id="bookmark">
  <h4>本文索引</h4>
  <ul>
    <li><a href="#%E4%BB%80%E4%B9%88%E6%98%AF%E9%9A%90%E5%BC%8F%E7%B1%BB%E5%9E%8B%E8%BD%AC%E6%8D%A2">什么是隐式类型转换</a></li>
    <li>
      <a href="#%E5%9F%BA%E7%A1%80%E5%9B%9E%E9%A1%BE">基础回顾</a>
      <ul>
        <li><a href="#%E7%9B%B4%E6%8E%A5%E5%88%9D%E5%A7%8B%E5%8C%96">直接初始化</a></li>
        <li><a href="#%E5%A4%8D%E5%88%B6%E5%88%9D%E5%A7%8B%E5%8C%96">复制初始化</a></li>
        <li><a href="#%E7%B1%BB%E5%9E%8B%E6%9E%84%E9%80%A0%E6%97%B6%E7%9A%84%E9%9A%90%E5%BC%8F%E8%BD%AC%E6%8D%A2">类型构造时的隐式转换</a></li>
      </ul>
    </li>
    <li>
      <a href="#%E9%9A%90%E5%BC%8F%E8%BD%AC%E6%8D%A2%E6%98%AF%E5%A6%82%E4%BD%95%E5%B7%A5%E4%BD%9C%E7%9A%84">隐式转换是如何工作的</a>
      <ul>
        <li><a href="#%E6%A0%87%E5%87%86%E8%BD%AC%E6%8D%A2">标准转换</a></li>
        <li><a href="#%E7%94%A8%E6%88%B7%E8%87%AA%E5%AE%9A%E4%B9%89%E8%BD%AC%E6%8D%A2">用户自定义转换</a></li>
        <li><a href="#%E9%9A%90%E5%BC%8F%E8%BD%AC%E6%8D%A2%E5%BA%8F%E5%88%97">隐式转换序列</a></li>
      </ul>
    </li>
    <li>
      <a href="#%E9%9A%90%E5%BC%8F%E8%BD%AC%E6%8D%A2%E5%BC%95%E5%8F%91%E7%9A%84%E9%97%AE%E9%A2%98">隐式转换引发的问题</a>
      <ul>
        <li><a href="#%E5%BC%95%E7%94%A8%E7%BB%91%E5%AE%9A">引用绑定</a></li>
        <li><a href="#%E6%95%B0%E7%BB%84%E9%80%80%E5%8C%96">数组退化</a></li>
        <li><a href="#%E4%B8%A4%E6%AD%A5%E8%BD%AC%E6%8D%A2">两步转换</a></li>
      </ul>
    </li>
    <li>
      <a href="#%E6%80%BB%E7%BB%93">总结</a>
      <ul>
        <li><a href="#%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99">参考资料</a></li>
      </ul>
    </li>
  </ul>
</blockquote>


## 什么是隐式类型转换

借用标准里的话来说，就是当你只有一个类型T1，但是当前表达式需要类型为T2的值，如果这时候T1自动转换为了T2那么这就是隐式类型转换。

如果你觉得太抽象的话可以看两个例子，首先是最常见的混用数值类型：

```c++
int a = 0;
long b = a + 1; // int 转换为 long

if (a == b) {
    // 默认的operator==需要a的类型和b相同，因此也发生转换
}
```

int转成long是向上转换，通常不会有太大问题，而long到int则很可能导致数据丢失，因此要尽量避免后者。

第二个例子是自定义类型到标量类型的转换：

```c++
std::shared_ptr<int> ptr = func();
if (ptr) { // 这里会从shared_ptr转换成bool
    // 处理数据
}
```

因为提供了用户自定义的隐式类型转换规则，所以我们可以很简单地去判断智能指针是否为空。在这里if表达式里需要bool，因此ptr转换为了bool，这又被叫做语境转换。

理解了什么是隐式类型转换转换之后我们再来看看那些不允许进行隐式转换的语言，比如golang：

```golang
var a int32 = 0;
var b int64 = 1;

fmt.Println(a + b) // error!
fmt.Println(int64(a) + b)
```

编译器会告诉你类型不同无法运算。一个更灾难性的例子如下：

```golang
sleepDuration := 2.5
time.Sleep( time.Duration(float64(time.Millisecond) * ratio) ) // 休眠2.5ms
```

本身是非常简单的代码，然而多层嵌套式的类型转换带来了杂音，代码可读性严重下降。

这种形式的类型转换被称为显式类型转换，在c++里是这样的：

```c++
A a{1};
B b = static_cast<B>(a);
```

`static_cast`被用于将某个类型转换到其相关的类型，需要用户指明待转换到的类型，除此之外还有`const_cast`等cast，它们负责了c++中的显式类型转换。

由此可见隐式类型转换转换可以简化代码的书写。不过简化不是没有代价的，我们细细说来。

## 基础回顾

在正式介绍隐式类型转换之前，我们先要回顾一下基础知识，放轻松。

### 直接初始化

首先是类的直接初始化。

顾名思义，就是显式调用类型的构造函数进行初始化。举个例子：

```c++
struct A {
    A() = default;
    A(const A&) = default;
    A(int) {}
};

// 这是默认初始化: A a; 注意区分

A a1{}; // c++11的列表初始化
// 不能写出A a2()，因为这会被认为是函数声明
A a2(1);
A a3(a2); // 没错，显式调用复制构造函数也是直接初始化

auto a4 = static_cast<A>(1);
```

需要注意的是a4，用`static_cast`转换成类型T的这一步也是直接初始化。

这种初始化方式有什么用呢？直接初始化会考虑全部的构造函数，而不会忽略explicit修饰的构造函数。

显式地调用构造函数进行直接初始化实际上是显式类型转换的一种。

### 复制初始化

除去默认初始化和直接初始化，剩下的会导致复制的基本都是复制初始化，典型的如下：

```c++
A func() {
    return A{}; // 返回值会被复制初始化
}

A a5 = 1; // 先隐式转换，再复制初始化

void func2(A a) {} // 非引用的参数传递也会进行复制构造
```

然而类似`A a6 = {1}`的表达式却不是复制初始化，这是复制列表初始化，会直接选择合适的非explicit构造函数进行初始化，而不用创建临时量再进行复制。

复制初始化又起到什么作用呢？

首先想到的是这样可以创造某个对象的副本，没错，不过还有一个更重要的作用：

**如果想要某个类型T1的value能进行到T2的隐式转换，两个类型必须满足这个表达式的调用`T2 v2 = value`。**

而这个形式的表达式正是复制初始化表达式。至于具体的原因，我们马上就会在下一节看到。

### 类型构造时的隐式转换

在进入本节前我们看一道经典的面试题：

```c++
std::string s = "hello c++";
```

请问创建了几个string呢？如果你脱口而出1个，那么面试官八成会狡黠一笑，让你回家等通知去了。

那么答案是什么呢？是1个或者2个。什么，你逗我呢？

先别急，我们分情况讨论。首先是c++11之前。

在c++11前题目里的表达式实际上会导致下面的行为：

1. 首先`"hello c++"`是`const char[N]`类型的，不过它在表达式中于是退化成`const char *`
2. 然后因为s实际上是处于“声明即定义”的表达式中，因此适用的只有复制构造函数，而不是重载的=
3. 因此等号的右半边必须也是`string`类型
4. 因为正好有从`const char *`到`string`的转换规则，因此把它转换成合适的类型
5. 转换完会返回一个新的`string`的临时量，它会作为参数调用复制构造函数
6. 复制构造函数调用完成后s也就创建完毕了。

在这里我们暂且忽略了string的写时复制等黑科技，整个过程创建了s和一个临时量，一共两个string。

很快c++11就出现了，同时还带来了移动语义，然而结果并没有改变：

1. 前面步骤相同，字符串字面量隐式转换成string，创建了一个临时量
2. 临时量是个右值，所以绑定给右值引用，因此移动构造函数被选择
3. 临时量里的数据移动到s里，s创建完成

移动语义减少了不必要的内部数据的复制，但是临时量还是会被创建的。

有进捣鼓编译器的朋友可能要说了，编译器是不生成这个临时量的。是这样的，编译器会用复制省略（copy elision）优化这段代码。

是的，复制省略在c++11里就已经被提到了，不过那时候它是可选的，并不强制编译器支持这一优化。因此你在GCC和clang上观察到的不一定能代表全部的c++编译器的情况，所以我们仍以标准为基础推演了理论上的行为。

到目前为止答案都是2，然而很快有意思的事情发生了——复制省略在c++17里成为了被标准化的行为。

在c++17里除非必要，否则临时量（现在叫做右值的结果对象，一个右值只有在实际需要存在一个临时变量的情况下才会创建一个临时变量，这个过程叫做实质化，创建出来的那个临时量就是该右值的结果对象）不会被创建，换而言之，`T obj = expr`这样的形式会以expr产生结果直接调用合适的构造函数，而不会进行临时量的创建和复制构造函数的调用，不过为了保证语义的完整性，复制构造函数仍然被要求是可访问的，毕竟类本身不允许复制构造的话复制初始化本身就是不正确的，不能因为复制省略而导致错误的代码被编译通过。

所以现在过程变成了下面这样子：

1. 编译器发现表达式是string的复制初始化
2. 右侧是表达式会隐式转换产生一个string的纯右值用于初始化同一类型的s
3. 判断复制构造函数是否可用，然后发现符合复制省略的条件
4. 寻找string里是否有符合要求的构造函数
5. 找到了`string::string(const char *)`，于是直接调用
6. s初始化完成

因此，在c++17下只会创建一个string对象，这比移动语义更加高效。这也是为什么我说题目的答案既可以是1也可以是2的原因。

同时我们还发现，在复制构造时的类型转换不管复制有没有被省略都是存在的，只不过换了一个形式，这就是我们后面要讲的内容。

## 隐式转换是如何工作的

复习完基础知识，我们可以进入正题了。

隐式转换可以分为两个部分，标准定义的转换和用户自定义的转换。我们先来看看它们是什么。

### 标准转换

也就是编译器里内置的一些类型转换规则，比如数组退化成指针，函数转换成函数指针，特定语境下要求的转换（if里要求bool类型的值），整数类型提升，数值转换，数据类型指针到void指针的转换，nullptr_t到数据类型指针的转换等。

底层const和volatie也可以被转换，只不过只能添加不能减少，可以把`T*`转换成`const T*`，但反过来是不可以的。

这些转换基本都是针对标量类型和数组这种内置的聚合类型的。

如果想要指定自定义类型的转换规则，则需要编写用户自定义类型转换的接口了。

### 用户自定义转换

说了这么多，也该看看用户自定义转换了。

用户能控制的自定义转换接口一共也就两个，转换构造函数和用户定义转换函数。

转换构造函数就是只类似`T(T2)`这样的构造函数，它拥有一个显式的T2类型的参数，通过这个构造函数可以实现从T2转换类型至T1的效果。

用户定义转换函数是类似`operator T2()`这样的类方法，注意不需要指定返回值。通过它可以实现从T1转换到T2。可转换的类型包括自身T1（还可附加cv限定符，或者引用）、T1的基类（或引用）以及void。

举个例子：

```c++
struct A {};

struct B {
    // 转换构造函数
    B(int);
    B(const A&);

    // 用户定义转换函数，不需要显式指定返回值
    operator A();
    operator int();
}
```

上面的B自定义了转换规则，既可以从int和A转换成B，也可以从B转换成int和A。

不难看出规则是这样的：

```text
T  <---转换构造函数---  其他类型

T  ---用户定义转换函数--->  其他类型
```

这里的转换构造函数是指没有`explicit`限定的，有的话就不能用于隐式类型转换。

从c++11开始`explicit`还可以用于用户定义的转换函数，例如：

```c++
template <typename T>
struct SmartPointer {
    //...
    T *ptr = nullptr;
    // 方便判断指针是否为空
    explicit operator bool() {
        return ptr != nullptr;
    }
};

SmartPointer<int> p = func();
if (p) {
    p << 1; // 这是不允许的
}
```

这样的类型转换函数只能用于显式初始化以及特定语境要求的类型转换（比如if里的条件表达式要求返回bool值，这算隐式转换的一种），因此可以避免注释标注的那种语义错误。因此这类转换函数也无法用于其他的隐式转换。

c++11开始函数可以自动推导返回值，模板和自动推到也可以用于自定义的转换函数：

```c++
template <typename T>
struct SmartPointer {
    //...
    T *ptr = nullptr;
    explicit operator bool() {
        return ptr != nullptr;
    }

    // 配合模板参数
    operator T*() {
        return ptr;
    }

    /* 自动推到返回值，与上一个同义
    operator auto() {
        return ptr;
    }
    */
};

SmartPointer<int> p = func();
int *p1 = p;
```

最后用户自定义的转换函数还可以是虚函数，但是只有从基类的引用或指针进行派发的时候才会调用子类实现的转换函数：

```c++
struct D;
struct B {
    virtual operator D() = 0;
};
struct D : B
{
    operator D() override { return D(); }
};
 
int main()
{
    D obj;
    D obj2 = obj; // 不调用 D::operator D()
    B& br = obj;
    D obj3 = br; // 通过虚派发调用 D::operator D() 
}
```

用户定义转换函数不能是类的静态成员函数。

### 隐式转换序列

了解完标准内置的转换规则和用户自定义的转换规则，我们该看看隐式转换的工作机制了。

对于需要进行隐式转换的上下文，编译器会生成一个隐式转换序列：

1. 零个或一个由标准转换规则组成的标准转换序列，叫做初始标准转换序列
2. 零个或一个由用户自定义的转换规则构成的用户定义转换序列
3. 零个或一个由标准转换规则组成的标准转换序列，叫做第二标准转换序列

对于隐式转换发生在构造函数的参数上时，第二标准转换序列不存在。

初始标准转换序列很好理解，在调用用户自定义转换前先把值的类型处理好，比如加上cv限定符：

```c++
struct A {};
struct B {
    operator A() const;
};

const B b;
const A &a = b;
```

初始标准转换序列会把值先转换成适当的形式以供用户转换序列使用，在这里`operator A() const`希望传进来的this是`const B*`类型的，而对b直接取地址只能得到`B*`，正好标准转换规则里有添加底层const的规则，所以适用。

如果值的类型正好，不需要任何预处理，那么初始标准转换序列不会做任何多余的操作。

如果第一步还不能转换出合适的类型，那么就会进入用户定义转换序列。

如果类型是直接初始化，那么只会调用转换构造函数；如果是复制初始化或者引用绑定，那么转换构造函数和用户定义转换函数会根据重载决议确定使用谁。另外如果转换函数不是const限定的，那么在两者都是可行函数时优先选择转换函数，比如`operator A();`这样的，否则会报错有歧义（GCC 10.2上测试显示有歧义的时候会选择转换构造函数，clang++11.0和标准描述一致）。这也是我们复习了几种初始化有什么区别的原因，因为类的构造形式不同结果也可能会不同。

选择好一个规则后就可以进入下一步了。

如果是在构造函数的参数上，那么隐式转换到此就结束了。除此之外我们需要进行第三部。

第三部是针对用户转换序列处理后的值的类型做一些善后工作。之所以不允许在构造函数的参数上执行这一步是因为防止过度转换后和用户转换规则产生循环。

举个例子：
```c++
struct A
{
    operator int() const;
};

A a;
bool b = a;
```

在这里a只能转换成int，而为了偷懒我们直接把a隐式转换成bool，问题来了，初始标准转换序列把`A*`转换成了`const A*`（作为this，类方法的隐式参数），用户转换序列把`const A*`转换为了int，int和bool是完全不同的类型，怎么办呢？

这就用上第二标准转换序列了，这里是数值转换，int转成bool。

不过上面只是个例子，请不要这么写，因为在实际代码中会出现问题：

```c++
template <typename T>
struct SmartPointer {
    //...
    T *ptr = nullptr;
    operator bool() {
        return ptr != nullptr;
    }

    T& operator*() {
        return *ptr;
    }
};

auto ptr = get_smart_pointer();
if (ptr) {
    // ptr 是int*的包装，现在我们想取得ptr指向的值
    int value = p;
    // ...
}
```

上面的代码不会有任何编译错误，然而它将引发严重的运行时错误。

为什么呢？因为如注释所说我们想取得指针指向的值，然而我们忘记解引用了！实际上因为要转换成int，隐式转换序列里是这样的：

1. 初始标准转换序列  ----->  当前类型已经调用用户转换序列的要求了，什么都不做
2. 用户定义转换序列  ----->  和int最接近的有转换关系的类型只有bool了，调用这个
3. 第二标准转换序列  ----->  得到了bool，目标的int，正好有规则可用，进行转换

因此你的value只会有两种值，0和1。这就是隐式转换带来的第一个大坑，而上面代码反应出的问题叫做“安全bool（safe bool）”问题。

好在我们可以用`explicit`把它踢出转换序列：

```c++
template <typename T>
struct SmartPointer {
    //...
    T *ptr = nullptr;
    explicit operator bool() {
        return ptr != nullptr;
    }
};
```

这样当再写出`int value = p`的时候编译器就能及时发现并报错啦。

第二标准转换序列的本意是帮我们善后，毕竟类的编写者很难面面俱到，然而也正是如此带来了一些坑点。

还有另外一点要注意，标准规定了如果用户转换序列转换出了一个左值（比如一个左值引用），而最终转换目标的右值引用，那么标准转换中的左值转换为右值的规则不可用，程序是无法通过编译的，比如：

```c++
struct A
{
    operator int&();
};

int&& b = A();
```

编译上面的代码，g++会奖励你一句`cannot bind rvalue reference of type ‘int&&’ to lvalue of type ‘int’`。

如果隐式转换序列里一个可行的转换都没有呢？那很遗憾，只能编译报错了。

## 隐式转换引发的问题

现在我们已经知道隐式转换的工作方式了。而且我们也看到了隐式类型转换是如何闯祸的。

下面将要介绍隐式类型转换闯了祸怎么善后，以及怎么防患于未然。

是时候和实际应用碰撞出点火花了。

### 引用绑定

第一个问题是和引用相关的。不过与其说是隐式转换惹的祸倒不如说是引用绑定自身的坑。

我们知道对于一个类型T，可以有这几种引用类型：

- `T&`，T的引用，只能绑定到T类型的左值
- `const T&`，const T的引用，可以绑定到T的左值和右值，以及const T的左值和右值
- `T&&`，T的右值引用，只能绑定到T类型的右值
- `const T&&`，一般来说见不到，然而当你对一个`const T&`使用`std::move`就能得到这东西了

引用必须在声明的同时进行初始化，所以下面这样的代码应该是大家再熟悉不过的了：

```c++
int num = 0;
const int &a = num;
int &b = num;
int &&c = 100;
const int &d = 100;
```

新的问题出现了，考虑一下如下代码的运行结果：

```c++
int a = 10;
long &b = a;
std::cout << b << std::endl;
```

不是10吗？还真不是：

```text
c.cpp: In function ‘int main()’:
c.cpp:6:11: error: cannot bind non-const lvalue reference of type ‘long int&’ to an rvalue of type ‘long int’
    6 | long &b = a;
      | 
```

报错说得很清楚了，一个普通的左值引用不能绑定到一个右值上。因为a是int，b是long，所以a想赋值给b就必须先隐式转换成long。

隐式转换除非是转成引用类型，否则一般都是右值，所以这里报错了。解决办法也很简单：

```c++
long b1 = a;
const long &b2 = a;
```

要么直接复制构造一个新的long类型变量，值类型的变量可以从右值初始化；要么使用const左值引用，因为它能绑定到右值。

扩展一下，函数的参数传递也是如此：

```c++
void func(unsigned int &)
{
    std::cout << "lvalue reference" << std::endl;
}

void func(const unsigned int &)
{
    std::cout << "const lvalue reference" << std::endl;
}

int main()
{
    int a = 1;
    func(a);
}
```

结果是“const lvalue reference”，这也是为什么很多教程会叫你尽量多使用const lvalue引用的原因，因为除了本身的类型T，这样的函数还可以通过隐式类型转换接受其他能转换成T的数据做参数，并且相比创建一个对象并复制初始化成参数，应用的开销更小。当然右值最优先匹配的是右值引用，所以如果有形如`void func(unsigned int &&)`的重载存在，那么这个重载会被调用。

最典型的应用非下面的例子莫属了：

```c++
template <typename... Args>
void format_and_print(const std::string &s, Args&&... args)
{
    // 实现格式化并打印出结果
}

std::string info = "%d + %d = %d\n";
format_and_print(info, 2, 2, 4);
format_and_print("%d * %d = %d\n", 2, 2, 4);
```

只要能隐式转换成string，就能直接调用我们的函数。

最重要的一点，隐式类型转换产生的通常是右值。（当然显式类型转换也一样，不过在隐式转换的时候更容易忘了这点）

### 数组退化

同样是隐式转换带来的经典问题：数组在求值表达式中退化成指针。

你能给出下面代码的输出吗：

```c++
void func(int arr[])
{
    std::cout << (sizeof arr) << std::endl;
}

int main()
{
    int a[100] = {0};
    std::cout << (sizeof a) << std::endl;
    func(a);
}
```

在我的amd64 Linux上使用GCC 10.2编译运行的结果是400和8，后者其实是该系统上`int*`的大小。因为sizeof不求值而函数参数传递是求值的，所以数组退化成了指针。

这样的隐式转换带来的坏处是什么呢？答案是数组的长度丢失了。假如你不知道这一点，在函数中仍然用sizeof去求数组的大小，那么难免不会出问题。

解决办法有很多，比如最简单的借助模板：

```c++
template <std::size_t N>
void func(int (&arr)[N])
{
    std::cout << (sizeof arr) << std::endl; // 400
    std::cout << N << std::endl; // 100
}
```

现在N是100，而sizeof会返回400，因为sizeof一个引用会返回引用指向的类型的大小，这里是`int [100]`。

一个更简单也更为现代c++推崇的做法是放弃原始数组，把它当做沉重的历史包袱丢弃掉，转而使用`std::array`和即将到来的`std::span`。这些更现代化的数组替代品可以更好得代替原始数组而不会发生诸如隐式转换成指针等问题。

### 两步转换

还有不少教程会告诉你在隐式转换的时候超过一次的类型转换是不可以的，我习惯把这种问题叫做“两步转换”。

为什么叫两步转换呢？假如我们有ABC三个类型，A可以转B，B可以转C，他们是单步的转换，而如果我们需要把A转成C，就需要先把A转成B，因为A不能直接转换成C，因此形成了一个转换链：`A -> B -> C`，其中进行了两次类型转换，我称其为两步转换。

下面是一个典型的“两步转换”：

```c++
struct A{
    A(const std::string &s): _s{s} {}
    std::string _s;
};

void func(const A &s)
{
    std::cout << s._s << std::endl;
}

int main()
{
    func("two-steps-implicit-conversion");
}
```

我们知道`const char*`能隐式转换到string，然后string又可以隐式转换成A：`const char* -> string -> A`，而且函数参数是个常量左值引用，应该能绑定到隐式转换产生的右值。然而用g++编译代码会是下面的结果：

```text
test.cpp: In function 'int main()':
test.cpp:15:10: error: invalid initialization of reference of type 'const A&' from expression of type 'const char [30]'
   15 |     func("two-steps-implicit-conversion");
      |          ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
test.cpp:8:20: note: in passing argument 1 of 'void func(const A&)'
    8 | void func(const A &s)
      |           ~~~~~~~~~^

```

果然报错了。可是这真的是因为两步转换带来的结果吗？我们稍微改一改代码：

```c++
struct A{
    A(bool b)
    {
        _s = b ? "received true" : "received false";
    }
    std::string _s;
};

void func(const A &s)
{
    std::cout << s._s << std::endl;
}

int main()
{
    int num = 0;
    func(num); // received false
    unsigned long num2 = 100;
    func(num2); // received true
}
```

这次不仅编译通过，而且指定`-Wall -Wextra`也不会有任何警告，输出也是正常的。

那就怪了，这里的两次调用分别是`int -> bool -> A`和`unsigned long -> bool -> A`，很明星的两步转换，怎么就是合法的正常代码呢？

其实答案早在[隐式转换序列](#隐式转换序列)那节就告诉过你了：

_一个隐式类型转换序列包括一个初始标准转换序列、一个用户定义转换序列、一个第二标准转换序列_

也就是说不存在什么两步转换问题，本身转换序列最少可以转换1次，最多可以三次。两次转换当然没问题了。

唯一会触发问题的是出现了两次用户定义转换，因为隐式转换序列里只允许一次用户定义转换，语言标准也规定了不允许出现多余一次的用户定义转换：

> At most one user-defined conversion (constructor or conversion function) is implicitly applied to a single value. -- 12.3 Conversions \[class.conv\]

所以这条转换链：`const char* -> string -> A` 是有问题的，因为从字符串字面量到string和string到A都是用户定义转换。

而`int -> bool -> A`和`unsigned long -> bool -> A`这两条是没问题的，第一次转换是初始标准转换序列完成的，第二次是用户定义转换，整个过程合情合理。

由此看来教程们只说对了一半，“两步转换”的症结在于一次隐式转换中不能出现两次用户定义的类型转换，这个问题叫做“两步自定义转换”更恰当。

用户定义的类型转换只能出现在自定义类型中，这其中包括了标准库。所以换句话说，当你有一条`A -> B -> C`这样的隐式转换链的时候，如果其中有两个都是自定义类型，那么这个隐式转换是错误的。

唯一的解决办法就是把第一次发生的用户自定义转换改成显式类型转换：

```c++
struct A{
    A(const std::string &s): _s{s} {}
    std::string _s;
};

void func(const A &s)
{
    std::cout << s._s << std::endl;
}

int main()
{
    func(std::string{"two-steps-implicit-conversion"});
}
```

现在隐式转换序列里只有一次自定义转换了，问题也就不会发生了。

## 总结

相信现在你已经彻底理解c++的隐式类型转换了，常见的坑应该也能绕过了。

但我还是得给你提个醒，尽量不要去依赖隐式类型转换，多用`explicit`和各种显式转换，少想当然。

_Keep It Simple and Stupid._

##### 参考资料

<https://zh.cppreference.com/w/cpp/language/copy_elision>

<http://www.cplusplus.com/doc/tutorial/typecasting/>

<https://en.cppreference.com/w/cpp/language/implicit_conversion>

<https://stackoverflow.com/questions/26954276/second-standard-conversion-sequence-of-user-defined-conversion>

<https://stackoverflow.com/questions/48576011/why-does-const-allow-implicit-conversion-of-references-in-arguments/48576055>

<https://zh.cppreference.com/w/cpp/language/cast_operator>

<https://www.nextptr.com/tutorial/ta1211389378/beware-of-using-stdmove-on-a-const-lvalue>

<https://en.cppreference.com/w/cpp/language/reference_initialization>

<https://stackoverflow.com/questions/12847272/multiple-implicit-conversions-on-custom-types-not-allowed>
