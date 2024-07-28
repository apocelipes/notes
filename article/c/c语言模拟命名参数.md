最近在书里看到的，让c语言去模拟其他语言里有的命名函数参数。觉得比较有意思所以记录一下。

## 目标

众所周知c语言里是没有命名函数参数这种东西的，形式参数虽然有自己的名字，但传递的时候并不能通过这个名字来指定参数的值。

而支持命名参数的语言，比如python里，我们能让代码达到这种效果：

```python
def k_func(a, b, c):
    print(f'{a=}, {b=}, {c=}')

k_func(1, 2, 3)        # Output: a=1, b=2, c=3
k_func(c=1, b=3, a=2)  # Output: a=2, b=3, c=1
```

我们想要的类似于`k_func(c=1, b=3, a=2)`的效果，至少表现形式上要接近。虽说c的语法并不支持这样的表达式，但我们有办法模拟。

## 实现

我们假设有这样一个c语言的函数，现在我们想模拟命名参数传递：

```c
int func(const char *text, unsigned int length, int width, int height, double weight)
{
    int printed = 0;
    printed += printf("text: %s\n", text);
    printed += printf("length: %d\n", length);
    printed += printf("width X height: %d X %d\n", width, height);
    printed += printf("weight: %g\n", weight);
    return printed;
}
```

模拟的关键在于如何完成名字到参数的映射。而且我们函数的五个参数有四种不同的类型，所以这个映射还得是异构的。

在不借助第三方库的情况下，第一个能想到的应该是enum加`void*`数组。enum可以完成名字到数组索引的映射，`void*`可以保存异构的数据。

这个方案的缺点很多：如果想要在length和width之间加个参数，我们很可能就需要修改所有的映射代码；比如`void*`可以接受任何数据类型的指针，所以我们几乎没有办法确保类型安全，想象一下如果有人给text传了个int的指针会发生什么。所以这个方案并不推荐。

能容纳异构的数据，同时还能给这些数据名字的东西，实际上在c里非常常见，那就是结构体。我们要选择的就是基于结构体的方案。

首先我们来定义这个结构体：

```c
typedef struct func_arguments {
    const char *text;
    unsigned int length;
    int width;
    int height;
    double weight;
} func_arguments;
```

字段的顺序是无所谓的，你可以根据情况来任意调整。现在我们可以根据字段名来设置值了。现在还缺一环，只有结构体是没用的，我们需要把结构体的字段传递给函数才行。所以我们要写一个帮助函数：

```c
int func_wrapper(func_arguments fa)
{
    // 根据需要还可以加入参数校验等逻辑
    return func(fa.text, fa.length, fa.width, fa.height, fa.weight);
}
```

我们需要的工具基本上都在这了，然而现在和命名参数传递还有不少差距，因为我们需要这样写代码：

```c
func_arguments fa;
fa.text = "text";
fa.length = 4;
fa.width = fa.height = 8;
fa.weight = 10.0;
func_wrapper(fa);
```

不仅形式上差远了，代码还很繁琐，所以我们还得借助一下c99的复合字面量+指定初始化器的新语法来模拟命名参数传递：

```c
func_wrapper((func_arguments){ .text = "text", .length = 4, .width = 8, .height = 8, .weight = 10.0 });
```

c99允许在初始化时使用`.字段名`的形式给字段设置值，没有指定的字段则初始化成零值，c99还允许符合要求的字面量类型转换成数组/结构体。利用新语法我们就能写出上面的代码了。

现在形式上确实很接近了，但还是显得有点啰嗦。这时候就得依赖宏了。c的宏可以实现文本替换和简单的类型分发，所以可以用它来把一些看起来不合法的表达式转换成正常的c语言代码。

首先声明，不要滥用宏，尤其是像下面那样，这里只是充当一下记录而不是教你生产实践。

用宏可以这样写：

```c
#define k_func(...) func_wrapper((func_arguments){ __VA_ARGS__ })
```

这里用了另一个c99的新语法，可变长参数宏，三个点意味着宏可以接受用逗号分隔的任意个参数，而`__VA_ARGS__`会原样替换成这些参数。因此我们只需要让宏的参数里是正确的指定初始化器就行了：

```c
k_func(.text="text", .length=4);
// 宏完成替换后等价于 func_wrapper((func_arguments){ .text = "text", .length = 4 });
```

是不是很神奇？

有人可能会担心我们参数传递时复制了整个结构体，这会不会带来效率问题？通常这不会带来什么问题，现在编译器一般都能做到省略大部分不必要的复制，另外如果对象比较小的话复制通常也不会带来太大的开销，什么叫小很难定义，以我个人的经验来看尺寸比两个cacheline小的通常都可以算是“小”。

如果还是不放心，也可以简单得把参数类型改成结构体指针：

```c
int func_wrapper(const func_arguments *fa)
{
    return func(fa->text, fa->length, fa->width, fa->height, fa->weight);
}

#define k_func(...) func_wrapper(&(func_arguments){ __VA_ARGS__ })
```

使用方法是一样的。注意宏里的`&`，这会分配一个auto生命周期的func_arguments变量（通常在栈上）然后再取它的指针。现在你可以不用担心了。不过我一般不推荐这么写，除非你经过性能测试后发现参数复制真的导致了性能问题。

## 缺陷

奇迹和魔法都不是免费的，所以上面像变魔术的代码也是有代价的。

第一个缺陷比较小，那就是字段名字前必须加上点。如果这么写：`printf("%d\n", k_func(.text="text", length=4))`，注意length前我们不小心把点给漏了。编译器会爆出一个不明所以的报错：

```text
test.c: In function ‘main’:
test.c:31:45: error: ‘length’ undeclared (first use in this function)
   31 |         printf("%d\n", k_func(.text="text", length=4));
      |                                             ^~~~~~
test.c:27:52: note: in definition of macro ‘k_func’
   27 | #define k_func(...) func_wrapper((func_arguments){ __VA_ARGS__ })
      |                                                    ^~~~~~~~~~~
test.c:31:45: note: each undeclared identifier is reported only once for each function it appears in
   31 |         printf("%d\n", k_func(.text="text", length=4));
      |                                             ^~~~~~
test.c:27:52: note: in definition of macro ‘k_func’
   27 | #define k_func(...) func_wrapper((func_arguments){ __VA_ARGS__ })
      |                                                    ^~~~~~~~~~~
```

它会告诉你`length`这个名字从来没被定义过而不是告诉你漏了个`.`。平时看东西不认真的人就要受罪了，因为要找这个点得花上一番功夫。

第二个缺陷是没有语法提示和自动补全。毕竟宏可以把任何符合规则的文本替换进去，至于替换完怎么样就管不到了，所以想要对`k_func`的参数进行提示和补全是不太容易的，而且实验下来目前没有ide和编辑器能帮我把字段名自动补全。不过问题倒也不大，因为万一写错了的话编译器会给出准确的报错，只不过开发效率会降低一些。

第三个缺陷在于可以写出这样的东西：`printf("%d\n", k_func(.length=10, .text="text", .length=4))`。我们指定了两次length字段的值，语法上这是允许的，length的值会被最右边的那个覆盖掉。但这显然不符合我们的要求，而且前面说了因为没有自动补全和语法提示，我们不小心把height写成了weight也很难察觉到。更糟糕的是这种覆盖在gcc下需要指定`-Wextra`才能看到一个不痛不痒的警告。同样的情况在python下会直接收到一个语法错误的异常`keyword argument repeated`。

前两个缺陷还有办法克服，最后一个是没有任何办法的，只能靠更高级别的警告设置和人力检查了。

## 总结

正常来说我们做到`func_wrapper`那步就足够，后面的宏没啥意义纯粹是为了在形式上模拟python的命名参数而做的。

除了用结构体包装，写一个包装函数并调换参数的顺序或者给出默认值也是常见的做法，但这种做法很快会让接口的数量失控，大量相似的接口会让代码变得臭不可闻，所以我更推荐用结构体。

最后学习新语法还是很有用的，因为很多新语法好好利用的话可以有效提升扩展性和你的开发效率。
