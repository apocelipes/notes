对于无类型常量，可能大家是第一次听说，但这篇我就不放进[拾遗系列](golang拾遗：实现一个不可复制类型.md)里了。

因为虽然名字很陌生，但我们每天都在用，每天都有无数潜在的坑被埋下。包括我本人也犯过同样的错误，当时代码已经合并并发布了，当我意识到出了什么问题的时候为时已晚，最后不得不多了个合并请求留下了丢人的黑历史。

为什么我要提这种尘封往事呢，因为最近有朋友遇到了一样的问题，于是勾起了上面的那些“美好”回忆。于是我决定记录一下，一来备忘，二来帮大家避坑。

由于涉及各种隐私，朋友提问的代码没法放出来，但我可以给一个简单的复现代码，正如我所说，这个问题是很常见的：

```golang
package main

import "fmt"

type S string

const (
    A S = "a"
    B   = "b"
    C   = "c"
)

func output(s S) {
    fmt.Println(s)
}

func main() {
    output(A)
    output(B)
    output(C)
}
```

这段代码能正常编译并运行，能有什么问题？这里我就要提示你一下了，`B`和`C`的类型是什么？

你会说他们都是`S`类型，那你就犯了第一个错误，我们用发射看看：

```golang
fmt.Println(reflect.TypeOf(any(A)))
fmt.Println(reflect.TypeOf(any(B)))
fmt.Println(reflect.TypeOf(any(C)))
```

输出是：

```golang
main.S
string
string
```

惊不惊喜意不意外，常量的类型是由等号右边的值推导出来的(iota是例外，但只能处理整型相关的)，除非你显式指定了类型。

所以在这里B和C都是string。

那真正的问题来了，正如我在[这篇](golang拾遗：自定义类型和方法集.md)所说的，从原类型新定义的类型是独立的类型，不能隐式转换和赋值给原类型。

所以这样的代码就是错的：

```golang
func output(s S) {
    fmt.Println(s)
}

func main() {
    var a = "a" // string
    output(a)
}
```

编译器会报错。然而我们最开始的复现代码是没有报错的：

```golang
const (
    A S = "a"
    B   = "b"
    C   = "c"
)

func output(s S) {
    fmt.Println(s)
}
```

`output`函数只接受`S`类型的值，但我们的`B`和`C`都是string类型的，为什么这里可以编译通过还正常运行了呢？

这就要说到golang的坑点之一——**无类型常量了**。

## 什么是无类型常量

这个好理解，定义常量时没指定类型，那就是无类型常量，比如：

```golang
const (
    A S = "a"
    B   = "b"
    C   = "c"
)
```

这里A显式指定了类型，所以不是无类型常量；而B和C没有显式指定类型，所以就是无类型常量(untyped constant)。

## 无类型常量的特性

无类型常量有一些特性和其他有类型的常量以及变量不一样，得单独讲讲。

### 默认的隐式类型

正如下面的代码里我们看到的：

```golang
const (
    A = "a"
    B = 1
    C = 1.0
)

func main() {
    fmt.Println(reflect.TypeOf(any(A))) // string
    fmt.Println(reflect.TypeOf(any(B))) // int
    fmt.Println(reflect.TypeOf(any(C))) // float64
}
```

虽说我们没给这些常量指定某个类型，但他们还是有自己的类型，和初始化他们的字面量的默认类型相应，比如整数字面量是`int`，字符串字面量是`string`等等。

但只有一种情况下他们才会表现出自己的默认类型，也就是在上下文中没法推断出这个常量现在应该是什么类型的时候，比如赋值给空接口。

### 类型自动匹配

这个名字不好，是我根据它的表现起的，官方的名字叫`Representability`，直译过来是“代表性”。

看下这个例子：

```golang
const delta = 1 // untyped constant, default type is int
var num int64
num += delta
```

如果我们把`const`换成`var`，代码无法编译，会爆出这种错误：`invalid operation: num + delta (mismatched types int64 and int)`。

但为什么常量可以呢？这就是`Representability`或者说类型自动匹配在捣鬼。

按照官方的解释：`如果一个无类型常量的值是一个类型T的有效值，那么这个常量的类型就可以是类型T`。

举个例子，`int8`类型的所有合法的值是`[-128, 127)`，那么只要值在这个范围内的整数常量，都可以被转换成int8。

字符串类型同理，**所有用字符串初始化的无类型常量都可以转换成字符串以及那些基于字符串创建的新类型**。

这就解释了开头那段代码为什么没问题：

```golang
type S string

const (
    A S = "a"
    B   = "b"
    C   = "c"
)

func output(s S) {
    fmt.Println(s)
}

func main() {
    output(A) // A 本来就是 S，自然没问题
    output(B) // B 是无类型常量，默认类型string，可以表示成 S，没问题
    output(C) // C 是无类型常量，默认类型string，可以表示成 S，没问题
    // 下面的是有问题的，因为类型自动匹配不会发生在无类型常量和字面量以外的地方
    // s := "string"
    // output(s)
}
```

也就是说，在有明确给出类型的上下文里，无类型常量会尝试去匹配那个目标类型T，如果常量的值符合目标类型的要求，常量的类型就会变成目标类型T。例子里的`delta`的类型就会自动变成`int64`类型。

我没有去找为什么golang会这么设计，在c++、rust和Java里常量的类型就是从初始化表达式推导或显式指定的那个类型。

一个猜测是golang的设计初衷想让常量的行为表现和字面量一样。除了两者都有的类型自动匹配，另一个有力证据是golang里能作为常量的只有那些能做字面类型的类型（字符串、整数、浮点数、复数）。

无类型常量的类型自动匹配会带来很有限的好处，以及很恶心的坑。

## 无类型常量带来的便利

便利只有一个，可以少些几次类型转换，考虑下面的例子：

```golang
const factor = 2

var result int64 = int64(num) * factor / ( (a + b + c) / factor )
```

这样复杂的计算表达式在数据分析和图像处理的代码里是很常见的，如果我们没有自动类型匹配，那么就需要显式转换factor的类型，光是想想就觉得烦人，所以我也就不写显式类型转换的例子了。

有了无类型常量，这种表达式的书写就没那么折磨了。

## 无类型常量的坑

说完聊胜于无的好处，下面来看看坑。

一种常见的在golang中模拟enum的方法如下：

```golang
type ConfigType string

const (
    CONFIG_XML ConfigType = "XML"
    CONFIG_JSON = "JSON"
)
```

发现上面的问题了吗，没错，只有`CONFIG_XML`是`ConfigType`类型的！

但因为无类型常量有自动类型匹配，所以你的代码目前为止运行起来一点问题也没有，这也导致你没发现这个缺陷，直到：

```golang
// 给enum加个方法，现在要能获取常量的名字，以及他们在配置数组里的index
type ConfigType string

func (c ConfigType) Name() string {
    switch c {
    case CONFIG_XML:
        return "XML"
    case CONFIG_JSON:
        return "JSON"
    }
    return "invalid"
}

func (c ConfigType) Index() int {
    switch c {
    case CONFIG_XML:
        return 0
    case CONFIG_JSON:
        return 1
    }
    return -1
}
```

目前为止一切安好，然后代码炸了：

```golang
fmt.Println(CONFIG_XML.Name())
fmt.Println(CONFIG_JSON.Name()) // !!! error
```

编译器不乐意，它说：`CONFIG_JSON.Name undefined (type untyped string has no field or method Name)`。

为什么呢，因为上下文里没明确指定类型，`fmt.Println`的参数要求都是`any`，所以这里用了无类型常量的默认类型。当然在其他地方也一样，`CONFIG_JSON.Name()`这个表达式是无法推断出`CONFIG_JSON`要匹配成什么类型的。

这一切只是因为你少写了一个类型。

这还只是第一个坑，实际上因为只要是目标类型可以接受的值，就可以赋值给目标类型，那么出现这种代码也不奇怪：

```golang
const NET_ERR_MESSAGE = "site is unreachable"

func doWithConfigType(t ConfigType)

doWithConfigType(CONFIG_JSON)
doWithConfigType(NET_ERR_MESSAGE) // WTF???
```

一不小心就能把错得离谱的参数传进去，如果你没想到这点而做好防御的话，生产事故就理你不远了。

第一个坑还可以通过把常量定义写全每个都加上类型来避免，第二个就只能靠防御式编程凑活了。

看到这里，你也应该猜到我当年闯的是什么祸了。好在及时发现，最后补全声明 + 防御式编程在出事故前把问题解决了。

最后也许有人会问，golang实现enum这么折磨？没有别的办法了吗？

当然有，而且有不少，其中一个比较著名的是`stringer`: <https://pkg.go.dev/golang.org/x/tools/cmd/stringer>

这个工具也只能解决一部分问题，但至少比什么都做不了要强太多了。

## 总结

无类型常量会自动转换到匹配的类型，这会带来意想不到的麻烦。

一点建议：

1. 如果可以的话，尽量在定义常量时给出类型，尤其是你自定义的类型，int这种看情况可以不写
2. 尝试用工具去生成enum，一定要自己写过过瘾的话记得处理必然存在的例外情况。

这就是golang的大道至简，简单它自己，坑都留给你。

##### 参考

<https://go.dev/ref/spec#Representability>
