len是很常用的内置函数，可以测量字符串、slice、array、channel以及map的长度/元素个数。

不过你真的了解len吗？也许还有一些你不知道的小知识。

我们来看一道GO101的题目，这题也被[GO语言爱好者周刊](https://zhuanlan.zhihu.com/p/391245238)转载：

```golang
package main

import "fmt"

func main() {
    var x *struct {
        s [][32]byte
    }
    
    fmt.Println(len(x.s[99]))
}
```

题目问你这段代码的运行结果，选项有编译错误、panic、32和0。

我们分析一下，别看x的声明定义一大长串，实际上就是定义了一个有个`[][32]byte`的结构体，然后x是这个结构体的指针。

然后我们没有初始化x，所以x是一个值为`nil`的指针。看到这里你也许以及有答案了，对nil指针解引用访问它的成员s，那不就是panic嘛。即使引用x的成员合法，我们的s也没有初始化，访问没有初始化的slice也会panic。

然而这么想你就错了，代码的实际运行结果是32！

为什么呢？我们看看len的帮助文档：

> For some arguments, such as a string literal or a simple array expression, the result can be a constant. See the Go language specification's "Length and capacity" section for details.

这句话很重要，对于结果是数组的表达式，len可能会是一个编译期常量，而且数组类型的长度在编译期是可知的，所以熟悉c++的朋友大概会立刻想到这样的常量是不需要进行实际求值的，简单类型推导即可获得。不过口说无凭，我们看看spec里的描述：

> The expression len(s) is constant if s is a string constant. The expressions len(s) and cap(s) are constants if the type of s is an array or pointer to an array and the expression s does not contain channel receives or (non-constant) function calls; in this case s is not evaluated. Otherwise, invocations of len and cap are not constant and s is evaluated.
> 如果表达式是字符串常量那么len(s)也是常量。如果表达式s的类型是array或者array的指针，且表达式不是channel的接收操作或是函数调用，那么len(s)是常量，且表达式s不会被求值；否则len和cap会对s进行求值，其计算结果也不是一个常量。

其实说的很清楚了，但还有三点需要说明。

第一个是视为常量的表达式里为什么不能含有chan的接收操作和函数调用？

这个答案很简单，因为这两个操作都是使用这明确希望发生“副作用”的。特别是从chan里接收数据，还会导致goroutine阻塞，而我们的常量len表达式不会进行求值，这些你期望会发生的副作用便不会产生，会引发一些隐蔽的bug。

第二个是我们注意到了函数调用前用non-constant修饰了，这是什么意思？

按字面意思，一部分函数调用其实是可以在编译期完成计算被当成常量处理的，而另一些不可以。

在进一步深入之前我们先要看看golang里哪些东西是常量/常量表达式。

1. 首先是各种字面量以及对字面量的类型转换产生的值了，无需多说。
2. 一部分内置函数：len、cap、imag、real、complex，它们在参数是常量的时候本身也是常量表达式。
3. `unsafe.Sizeof`，因为类型的大小也是编译期就能确定的，所以它是常量表达式也很好理解。
4. 所有的常量之间的运算（加减乘除位运算等，除了非常量表达式函数的调用）都是常量表达式。

从上面的描述里可以看出两点，内置函数和`unsafe.Sizeof`的调用我们可以看成是constant function calls，所有常量表达式除了浮点数和复数表达式都可以在编译期完成计算。而其他函数比如用户自定义函数的调用，虽然仍然有可能在编译期被求值优化，但本身不属于常量表达式。所以语言标准会加以区分。比如下面这个：

```golang
func add(x, y int) int {
    return x + y
}

func main() {
    fmt.Println(add(512, 513)) // 1025
}
```

如果我们看生成的汇编，会发现求值已经完成，不需要调用add：

```assembly
MOVQ    $1025, (SP)
PCDATA  $1, $0
CALL    runtime.convT64(SB)
MOVQ    8(SP), AX
XORPS   X0, X0
MOVUPS  X0, ""..autotmp_16+64(SP)
LEAQ    type.int(SB), CX
MOVQ    CX, ""..autotmp_16+64(SP)
MOVQ    AX, ""..autotmp_16+72(SP)
NOP
MOVQ    os.Stdout(SB), AX
LEAQ    go.itab.*os.File,io.Writer(SB), CX
MOVQ    CX, (SP)
MOVQ    AX, 8(SP)
LEAQ    ""..autotmp_16+64(SP), AX
MOVQ    AX, 16(SP)
MOVQ    $1, 24(SP)
MOVQ    $1, 32(SP)
NOP
CALL    fmt.Fprintln(SB)
MOVQ    80(SP), BP
ADDQ    $88, SP
RET
```

很明显的，1025已经在编译期求值了，然而add的调用不是常量表达式，所以下面的代码会报错：

```golang
const number = add(512, 513) // error!!!

// example.go:11:7: const initializer add(512, 513) is not a constant
```

spec给出的实例是调用的内置函数，内置函数也只有在参数是常量的情况下被调用才算做常量表达式：

```golang
const (
	c4 = len([10]float64{imag(2i)})  // imag(2i) is a constant and no function call is issued
	c5 = len([10]float64{imag(z)})   // invalid: imag(z) is a (non-constant) function call
)
var z complex128
```

所以len的表达式里如果用了non-constant的函数调用，那么就len本身不能算是常量表达式了。

这就有了最后一个疑问，题目中的x不是常量，为什么len的结果是常量呢？

标准只说表达式里不能有chan的接收和非常量表达式的函数调用，没说其他的不可以。你也可以这么理解，表达式都有结果值，任何值除了无类型常量（这里显然不是）都是要有一个确定的类型的，只要这个类型是数组或者数组的指针，那len就能获得数组的长度，而这一切不需要s一定是常量表达式，编译器可以简单推导出表达式的值的类型。不能包含non-constant function calls和chan接收是我在第一点里解释的，杜绝所有可能的副作用发生从而保证即使不对表达式求值程序也是正确的，不包含这两个操作的表达式既可以是常量的也可以不是，所以这里我们能用`x.s[99]`作为len的参数。

说了这么多，只要len的参数类型为array或者array的指针并且符合要求，就不会进行求值，而题目里的表达式正好满足这点，所以虽然我们看起来是会导致panic的代码，但是本身并未进行实际求值，因此程序可以正常运行。另外cap也遵循同样的规则。

最后，还有个小测验，检验一下自己的学习吧：

```golang
// 以下哪些语句是正确的，哪些是错误的
var slice [][]*[10]int

const (
    a = len(slice[10000000000000][4]) // 1
    b = len(slice[1]) // 2
    c = len(slice) // 3
    d = len([1]int{1024}) // 4
    e = len([1]int{add(512, 512)}) // 5
    g = len([unsafe.Sizeof(slice)]int{}) // 6
    g = len([unsafe.Sizeof(slice)]int{int(unsafe.Sizeof(slice))}) // 7
)
```

##### 参考

<https://golang.org/ref/spec#Length_and_capacity>
