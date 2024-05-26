距离golang 1.23发布还有两个月不到，按照惯例很快要进入1.23的功能冻结期了。在冻结期间不会再添加新功能，已经添加的功能不出大的意外一般也不会被移除。这正好可以让我们提前尝鲜这些即将到来的新特性。

今天要说的就是1.23中对`//go:linkname`指令的变更。这个新特性可以说和我的一次失误息息相关。

重要的事情得先写在前面：**`//go:linkname`指令官方并不推荐使用，且不保证任何向前或者向后兼容性，因此明智的做法是尽量别用**

牢记这一点之后，我们可以接着往下看了。至于为啥和“我”也就是本文的作者有关，我们先看完新版本带来的新变化再说。

## linkname指令是做什么的

简单的说，linkname指令用于向编译器和链接器传递信息。具体的含义根据用法可以分为三类。

第一类叫做“pull”，意思是拉取，使用方式如下：

```golang
import _ "unsafe" // 必须有这行才能用linkname

import _ "fmt" // 被拉取的包需要显式导入（除了runtime包）

//go:linkname my_func fmt.Println
func my_func(...any) (n int, err error)
```

这种用法的指令格式是`//go:linkname <指令下方的只有声明的函数或包级别变量名> <本包或者其他包中的有完整定义的函数或变量>`。

这个指令的作用就是告诉编译器和连接器，`my_func`的函数体直接使用`fmt.Println`的，`my_func`类似`fmt.Println`的别名，和它共享同一份代码，就像把指令第二个参数指定的函数和变量拉取下来给第一个参数使用一样。

正因如此，指令下方给出的声明必须和被拉取的函数/变量完全一致，否则很容易因为类型不匹配导致panic（是的没错，除非拉取的对象不存在，否则都不会出现编译错误）。

这个指令最恐怖的地方在于它能无视函数或者变量是否是export的，包私有的东西也能被拉取出来使用。因为这一点这种用法在早期的社区中很常见，比如很多人喜欢这么干：`//go:linkname myRand runtime.fastrand`，因为runtime提供了一个性能还不错的随机数实现，但没有公开出来，所以有人会用linkname指令把它导出为己所用，当然随着1.21的发布这种用法不再有任何意义了，请永远都不要去模仿。

第二种用法叫做“push”，即推送。形式上是下面这样：

```golang
import _ "unsafe" // 必须有这行才能用linkname

//go:linkname main.fastHandle
func fastHandle(input io.Writer) error {
    ...
}

// package main
func fastHandle(input io.Writer) error

// 后面main包中可以直接使用fastHandle
// 这种情况下需要在main包下创建一个空的asm文件（通常以.s作为扩展名），以告诉编译器fastHandle的定义在别处
```

在这种用法中，我们只需要把函数/变量名当作第一个参数传给指令，注意需要给出想用这个函数/变量的包的名字，这里是main。同时指令声明的变量或函数必须要在同包内有完整的定义，通常推荐直接把完整定义写在linkname指令下方。

这种用法是告诉编译器和链接器这个函数/变量的名字就是`xxx.yyy`，如果遇到这个函数就使用linkname指定的函数/变量的代码，这个模式下甚至能在本包定义别的包里的函数。

当然这种用法的语义作用更明显，它意味着这个函数会在任何地方被使用，修改它需要小心，因为改变了函数的行为可能会让其他调用它的代码出bug；修改了函数的签名则很可能导致运行时panic；删除了这个函数则会导致代码无法编译。

最后一类叫做“handshake”，即握手。他是把第一类和第二类方法结合使用：

```golang
package mypkg

import _ "unsafe" // 必须有这行才能用linkname

//go:linkname fastHandle
func fastHandle(input io.Writer) error {
    ...
}

package main

import (
    _ "unsafe" // 必须有这行才能用linkname
    _ "mypkg"
)

//go:linkname fastHandle mypkg.fastHandle 
func fastHandle(input io.Writer) error
```

“pull”的一方没什么区别，但“push”的一方不用再写包名，同时用来告诉编译器函数定义在别的地方的空的asm文件也不需要了。这种就像通讯协议中的“握手”，一方告诉编译器这边允许某个函数/变量被linkname操作，另一边则明确像编译器要求它要使用某个包的某个函数/变量。

通常“pull”和“push”应该成对出现，也就是你只应该使用“handshake”模式。

然而不幸的是，当前（1.22）的go语言支持“pull-only”的用法，即可以随便拉取任何包里的任何函数/变量，但不需要被拉取的对象使用“push”标记自己。而被linkname拉取的一方是完全无感知的。

这就导致了非常大的隐患。

## linkname带来的隐患

最大的隐患在于这个指令可以在不通知被拉取的packages的情况下随意使用包中私有的函数/变量。

举个例子：

```golang
// pkg/mymath/mymath.go
package mymath

func uintPow(n uint) uint {
    return n*n
}

// main.go
package main

import (
	"fmt"
	_ "linkname/pkg/mymath"
	_ "unsafe"
)

//go:linkname pow linkname/pkg/mymath.uintPow
func pow(n uint) uint

func main() {
	fmt.Println(pow(6))  // 36
}
```

正常来说，`uintPow`是不可能被外部使用的，然而通过linkname指令我们直接无视了接口的公开和私有，有什么就能用什么了。

这当然是非常危险的，比如我们把`uintPow`的参数类型改成string：

```golang
package mymath

func uintPow(n string) string {
	return n + n
}
```

这时候编译还是能正常编译，但运行的时候就会出现各种bug，在我的机器上表现是卡死和段错误。为什么呢？因为我们把uint强行传递了过去，但参数需要是string，类型对不上，自然会出现稀奇古怪的bug。这种在别的语言里是严重的类型相关的内存错误。

另外如果我们直接删了`uintPow`或者给他改个名，链接器会在编译期间报错：

```bash
$ go build

# linkname
main.main: relocation target linkname/pkg/mymath.uintPow not defined
```

而且我们导出的是私有函数，通常没人会认为自己写的私有级别的帮助函数会被导出到包外并被使用，因此在开发时大家都是保证公开接口的稳定性，私有的函数/变量是随时可以被大规模修改甚至删除的。

而linkname将这种在别的语言里最基本的规矩给粉碎了。

而且事实上也是如此，从1.18开始几乎每个版本都有因为编译器或者标准库内部的私有函数被修改/删除从而导致某些第三方库在新版本无法使用的问题，因为这些库在内部悄悄用`//go:linkname`用了一些未公开的功能。最近一次发生在广泛使用的知名json库上类似的问题可以在[这里](https://github.com/goccy/go-json/issues/506)看到。

## linkname的正面作用

既然这个指令如此危险，为什么还一直存在呢？答案是有不得不用的理由，其中一个就在启动go程序的时候。

我们来看下go的runtime里是怎么用linkname的：

```golang
// runtime/proc.go

//go:linkname main_main main.main
func main_main()

// runtime.main
// 所有go程序的入口
func main() {
    // 初始化runtime
    // 调用main.main
    fn := main_main // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime
    fn()
    // main退出后做清理工作
}
```

因为程序的入口在runtime里（要初始化runtime，比如gc等），所以入口函数必须在runtime包里。而我们又需要调用用户定义在main包里的main函数，但main包不能被import，因此只能靠linkname指令让链接器绕过所有编译器附加的限制来调用main函数。

这是目前在go自身的源代码里看到的唯一一处不得不使用“pull-only”模式的地方。

另外“handshake”模式也有存在的必要性，因为像runtime和reflect需要共享很多实现上的细节，因此reflect作为pull的一方，runtime作为push的一方，可以极大减少代码维护的复杂度。

除了上述这些情况，绝大数linkname的使用都可以算作*abuse*。

## golang1.23对linkname指令的改动

鉴于上述情况，golang核心团队决定限制linkname的使用。

第一个改动是标准库里新添加的包全部禁止使用linkname导出其中的内容，目前是通过黑名单实现的，1.23中新添加的几个包以及它们的internal依赖都在名单上，这样可以防止已有的linkname问题继续扩大。这对已有的代码也是完全无害的。

第二个变更时添加了新的ldflags: `-checklinkname=1`。1代表开启对linkname的限制，0代表维持1.22的行为不变。目前默认是0，但官方决定在1.23发布时默认值为1开启限制。个人建议尽量不要关闭这个限制。这个限制眼下只针对标准库，但按官方的说法效果好的话以后所有的代码不管标准库还是第三方都会启用限制。

最后也是最大的变动，禁止对标准库的 “pull-only” linkname指令，但允许“handshake”模式。

虽然go从来不保证linkname的向后兼容性，但这样还是会大量较大的破坏，因此官方已经对常见的go第三方库做了扫描，会把一些经常被人用linkname拉取的接口改成符合“handshake”模式的形式，这种改动只用加一行指令即可。而且该限制目前只针对标准库，其他第三方库暂时不受影响。

因为这个变更，下面的代码在1.23是无法编译通过的：

```golang
package main

import _ "unsafe"

//go:linkname corostart runtime.corostart
func corostart()

func main() {
	corostart()
}
```

因为`runtime.corostart`并不符合handshake模式，所以对它的linkname被禁止了：

```bash
$ go version

go version devel go1.23-13d36a9b46 Wed May 15 21:51:49 2024 +0000 windows/amd64

$ go build -ldflags=-checklinkname=1

# linkname
link: main: invalid reference to runtime.corostart
```

## linkname指令今后的发展

大趋势肯定是以后只允许handshake模式。不过作为过渡目前还是允许push模式的，并且官方应该会在进入功能冻结后把之前说的扫描到的常用的内部函数添加上linkname指令。

这里比较重要的是作为开发者的我们应该怎么办：

1. 1.23发布之后或者现在就开始利用`-checklinkname=1`排查代码，及时清除不必要的linkname指令。
2. 如果linkname指令非用不可，建议马上提issue或者熟悉go开发流程的立刻提pr补上handshake模式需要的指令，不过我不怎么推荐这种做法，因为内部api尤其是runtime以外的库的本来就不该随便被导出使用，没有一个强力的能说服所有人的理由，这些issue和pr多半不会被接受。
3. 向官方提案，尝试把你要用的私有api变成公开接口，这一步难度也很高，私有api之所以当初不公开一定是有原因的，现在再想公开可能性也不高。
4. 你的追求比较低，只要代码能跑就行，那可以在构建脚本里加上`-ldflags=-checklinkname=0`关闭限制，这样也许能岁月静好几个版本，直到某一天程序突然没法编译或者运行了一半被莫名其妙的panic打断。

4是万不得已时的保底方案，按优先度我推荐1 > 3 > 2的顺序去适配go1.23。2和3不仅仅适用于go标准库，常用的第三方库也可以。通过这些适配工作说不定也有机会让你成为go或者知名第三方库的贡献者。

从现在开始完全是来得及的，毕竟离1.23的第一个测试版发布还有一个月左右，离正式版发布还有两个月。而且方案2的修改并不算作新功能，不受功能冻结的影响。

当然，大部分开发者应该不用担心，比较linkname的使用是少数，一些主动使用linkname的库比如quic-go也知道兼容性问题，很小心地做了不同版本的适配，加上官方承诺的兜底这一对linkname指令的改动的影响应该比想象中小，但是是提高代码安全性的一大步。

## 说了这么多，和本文的作者有啥关系呢

那肯定有关系，老丢人了。

其实之所以会在开发窗口的中后期有这样大的变动，多半是因为我捅的篓子：前面也说过以前也有不少linkname引用的私有api变化导致的兼容问题，但要么影响范围很小要么作者及时适配使得这些问题没引起太大的波澜；但这次我的改动影响到了某个广泛应用的基础库，这个库用linkname指令引用了大量的内部api，更恐怖的是k8s也在用它，有人用master分支的go编译了一下k8s问题才被发现，要是没能及时发现的话会有一大堆软件在1.23测试版发布的时候出现兼容问题。其实在我的提交之前这些内部api已经变得面目全非了，但因为函数名字和字段类型没怎么变所以库的代码还能接着跑，直到我的提交打破了这一切。

当然问题要说大其实也不大，像那样大量使用linkname且没怎么适配版本的第三方库本身就不多，其次把变更的内部函数的签名还原之后问题很快就解决了，因此除了核心开发者和谷歌内部之外应该没多少人发觉这个问题。这也充分体现了linkname的危险性：在不算缺乏经验的我以及至少三位经验丰富的审核者的review下也没预料到这样功能简单且使用面极窄的内部私有函数会被linkname指令拉取出来使用。

后续库作者也说这些linkname引用的内部api其实很早之前就已经没啥用处了，他会尽快删除；实际上我跟踪了一下库代码发现这些被linkname导出的内部api除了设置了一些简单的flag值之外也确实没啥用处，flag值有些也没用上。

认识到这样的危险性后go官方自然不会坐视不管，官方以前应该也有类似想做限制的想法，这次也算是找到了合情合理的理由了，所以这回行动也意外的快，不到一星期从黑名单禁止导出新的库到linkname指令的检查都实现了。不出意外的话我们应该能在1.23看到一个更健壮的go以及它的标准库。

这样的问题怎么避免？答案是不可能，因为linkname能无视几乎一切限制私有函数/变量的办法，而且你也很难知道有哪些代码通过linkname访问了你写的函数/变量，因此只要一天不做限制类似这次问题的事故就会越来越多，总不可能让开发者每次改完代码都扫描一遍go语言编写的常见的项目吧。而且go的兼容性保证的是公开的接口和语法，内部实现的细节从来都不是也不应该是保证的对象。

我捅的这个篓子现在作为example被放在新提案里呢，虽说本质上用日本话讲叫“お互い様”（大家都有不对的地方），但作为广泛应用的编程语言也确实有需求和义务要兼容那些作为生态基石的应用广泛的第三方库，作为go的贡献者之一却忽视了这一点被结结实实地被上了一课也是应该的，算是经验教训了。。。

## 总结

最后总结就一句话：没事别用`//go:linkname`。。。。。。

想跟进这一变更的进展的话，可以看这个issue：<https://github.com/golang/go/issues/67401>
