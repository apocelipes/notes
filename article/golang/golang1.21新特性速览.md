经过了半年左右的开发，golang 1.21 在今天早上正式发布了。

这个版本中有不少重要的新特性和变更，尤其是在泛型相关的代码上。

因为有不少大变动，所以建议等第一个patch版本也就是1.21.1出来之后再进行升级，以免遇到一些意外的bug带来麻烦。

好了，一起来看看1.21带来的新特性吧。

<blockquote id="bookmark">
  <h4>本文索引</h4>
  <ul>
    <li><a href="#%E6%96%B0%E7%9A%84%E5%86%85%E7%BD%AE%E5%87%BD%E6%95%B0">新的内置函数</a></li>
    <li><a href="#%E7%B1%BB%E5%9E%8B%E6%8E%A8%E5%AF%BC%E6%9B%B4%E5%8A%A0%E6%99%BA%E8%83%BD">类型推导更加智能</a></li>
    <li><a href="#panic%E7%9A%84%E8%A1%8C%E4%B8%BA%E5%8F%98%E5%8C%96">panic的行为变化</a></li>
    <li><a href="#modules%E7%9A%84%E5%8F%98%E5%8C%96">modules的变化</a></li>
    <li><a href="#%E5%8C%85%E5%88%9D%E5%A7%8B%E5%8C%96%E9%A1%BA%E5%BA%8F%E7%9A%84%E6%94%B9%E5%8F%98">包初始化顺序的改变</a></li>
    <li><a href="#%E7%BC%96%E8%AF%91%E5%99%A8%E5%92%8Cruntime%E7%9A%84%E5%8F%98%E5%8C%96">编译器和runtime的变化</a></li>
    <li>
      <a href="#%E6%96%B0%E6%A0%87%E5%87%86%E5%BA%93">新标准库</a>
      <ul>
        <li><a href="#logslog%E5%92%8Ctestingslogtest">log/slog和testing/slogtest</a></li>
        <li><a href="#slices%E5%92%8Cmaps">slices和maps</a></li>
        <li><a href="#cmp">cmp</a></li>
      </ul>
    </li>
    <li>
      <a href="#%E5%B7%B2%E6%9C%89%E7%9A%84%E6%A0%87%E5%87%86%E5%BA%93%E7%9A%84%E5%8F%98%E5%8C%96">已有的标准库的变化</a>
      <ul>
        <li><a href="#bytes">bytes</a></li>
        <li><a href="#context">context</a></li>
        <li><a href="#cryptosha256">crypto/sha256</a></li>
        <li><a href="#net">net</a></li>
        <li><a href="#reflect">reflect</a></li>
        <li><a href="#runtime">runtime</a></li>
        <li><a href="#sync">sync</a></li>
        <li><a href="#errors">errors</a></li>
        <li><a href="#unicode">unicode</a></li>
      </ul>
    </li>
    <li><a href="#%E5%B9%B3%E5%8F%B0%E6%94%AF%E6%8C%81%E5%8F%98%E5%8C%96">平台支持变化</a></li>
    <li><a href="#%E6%80%BB%E7%BB%93">总结</a></li>
  </ul>
</blockquote>

## 新的内置函数

1.21添加了三个新的内置函数：`min`、`max`和`clear`。

`min`、`max`如其字面意思，用了选出一组变量里（数量大于等于1，只有一个变量的时候就返回那个变量的值）最大的或者最小的值。两个函数定义是这样的：

```golang
func min[T cmp.Ordered](x T, y ...T) T
func max[T cmp.Ordered](x T, y ...T) T
```

注意那个类型约束，这是新的标准库里提供的，原型如下：

```golang
type Ordered interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
    ~float32 | ~float64 |
    ~string
}
```

也就是说只有基于所有除了map，chan，slice以及复数之外的基本类型的变量才能使用这两个函数。或者换句话说，只有可以使用`<`、`>`、`<=`、`>=`、`==`和`!=`进行比较的类型才可以使用min和max。

有了min和max，可以把许多自己手写的代码替换成新的内置函数，可以少写一些帮助函数。而且使用新的内置函数还有一个好处，在变量个数比较少的时候还有编译器的优化可用，比自己写min函数性能上要稍好一些。

使用上也很简单：

```golang
maxIntValue := max(0, 7, 6, 5, 4, 3, 2, 1) // 7 type int
minIntValue := min(8, 7, 6, 5, 4, 3, 2, 1) // 1 type int
```

目前max和min都不支持slice的解包操作：`max(1, numbers...)`。

对于clear来说事情比min和max复杂。clear只接受slice和map，如果是对泛型的类型参数使用clear，那么类型参数的type set必须是map或者slice，否则编译报错。

clear的定义如下：

```golang
func clear[T ~[]Type | ~map[Type]Type1](t T)
```

对于参数是map的情况，clear会删除所有map里的元素（不过由于golang的map本身的特性，map存储的数据会被正常销毁，但map已经分配的空间不会释放）：

```golang
func main() {
        m := map[int]int{1:1, 2:2, 3:3, 4:4, 5:5}
        fmt.Println(len(m))  // 5
        clear(m)
        fmt.Println(len(m))  // 0
}
```

然而对于slice，它的行为又不同了：会把slice里所有元素变回零值。看个例子：

```golang
func main() {
        s := make([]int, 0, 100) // 故意给个大的cap便于观察
        s = append(s, []int{1, 2, 3, 4, 5}...)
        fmt.Println(s) // [1 2 3 4 5]
        fmt.Println(len(s), cap(s)) // len: 5; cap: 100
        clear(s)
        fmt.Println(s) // [0 0 0 0 0]
        fmt.Println(len(s), cap(s)) // len: 5; cap: 100
}
```

这个就比较反直觉了，毕竟clear首先想到的应该是把所有元素删除。那它的意义是什么呢？对于map来说意义是很明确的，但对于slice来说就有点绕弯了：

**slice的真实大小是cap所显示的那个大小，如果只是用`s := s[:0]`来把所有元素“删除”，那么这些元素实际上还是留在内存里的，直到s本身被gc回收或者往s里添加新元素把之前的对象覆盖掉，否则这些对象是不会被回收掉的，这一方面可以提高内存的利用率，另一方面也会带来泄露的问题（比如存储的是指针类型或者包含指针类型的值的时候，因为指针还存在，所以被指向的对象始终有一个有效的引用导致无法被回收），所以golang选择了折中的办法：把所有已经存在的元素设置成0值**

如果想安全的正常删除slice的所有元素，有想复用slice的内存，该怎么办？答案是：

```golang
s := make([]T, 0, 100) // 故意给个大的cap便于观察
s = append(s, []T{*new(T), *new(T)}...)

clear(s)
s = s[:0]
```

如果没有内置函数clear，那么我们得自己循环一个个赋值处理。而有clear的好处是，编译器会直接用memset把slice的内存里的数据设置为0，比循环会快很多。有兴趣的可以看看clear在slice上的实现：[代码在这](https://github.com/golang/go/blob/master/src/runtime/mbarrier.go#L330) 。

## 类型推导更加智能

其实就是bug修复，以前类似这样的代码在某些情况下没法正常进行推导：

```
func F[T ~E[], E any](t T, callable func(E))

func generic[E any](e E) {}
F(t, generic) // before go1.21: error; after go1.21: ok
```

理论上只要能推导出E的类型，那么`F`和`generic`的所有类型参数都能推导出来，哪怕`generic`本身是个泛型函数。以前想正常使用就得这么写：`F(t, generic[Type])`。

所以与其说是新特性，不如说是对类型推导的bug修复。

针对类型推导还有其他一些修复和报错信息的内容优化，但这些都没上面这个变化大，所以恕不赘述。

## panic的行为变化

1.21开始如果goroutine里有panic，那么这个goroutine里的defer里调用的recover必然不会返回nil值。

这导致了一个问题：recover的返回值是传给panic的参数的值，`panic(nil)`这样的代码怎么办？

先要提醒一下，`panic(nil)`本身是无意义的，且会导致recover的调用方无法判断究竟发生了什么，所以一直是被各类linter包括`go vet`命令警告的。然而这么写语法上完全正确，所以只有警告并不能解决问题。

解决办法是，如果现在使用`panic(nil)`或者`panic(值为nil的接口)`，recover会收到一个新类型的error：`*runtime.PanicNilError`。

总体上算是解决了问题，然而它把有类型的但值是nil的接口也给转换了，虽然从最佳实践的角度来讲panic一个空值的接口是不应该的，但多少还是会给使用上带来一些麻烦。

所以目前想要禁用这一行为的话，可以设置环境变量：`export GODEBUG=panicnil=1`。如果go.mod里声明的go版本小于等于1.20，这个环境变量的功能自动启用。

对于modules的变化，会在下一节讲解。

## modules的变化

最大的变化是现在mod文件里写的go版本号的意义改变了。

变成了：mod文件里写的go的版本意味着这个mod最低支持的golang版本是多少。

比如：

```golang
module github.com/apocelipes/flatmap

go 1.21.0
```

意味着这个modules最低要求go的版本是`go1.21.0`，而且可以注意到，现在patch版本也算在内里，所以一个声明为`go 1.21.1`的modules没法被1.21.0版本的go编译。

这么做的好处是能严格控制自己的程序和库可以在哪些版本的golang上运行，且可以推动库版本和golang本身版本的升级。

如果严格按照官方要求使用语义版本来控制自己的modules的话，这个改动没有什么明显的坏处，唯一的缺点是只有1.21及更高版本的go工具链才有这样的功能。

这个变更对`go.work`文件同样适用。

## 包初始化顺序的改变

现在按新的顺序来初始化包：

1. 把所有的packages按导入路径进行排序（字符串字典顺序）存进一个列表
2. 按要求和顺序找到列表里第一个符合条件的package，要求是这个package所有的import的包都已经完成初始化
3. 初始化这个找到的包然后把它移出列表，接着重复第二步
4. 列表为空的时候初始化流程结束

这样做的好处是包的初始化顺序终于有明确的标准化的定义了，坏处有两点：

1. 以前的程序如果依赖于特定的初始化顺序，那么在新版本会出问题
2. 现在可以通过修改package的导入路径（主要能改的部分是包的名字）和控制导入的包来做到让A包始终在B包之前初始化，因此B包的初始化流程里可以对A包公开出来的接口或者数据进行修改，这样做耦合性很高也不安全，尤其是B包如果是某个包含恶意代码的包的话。

我们能做的只有遵守最佳实践：不要依赖特定的包直接的初始化顺序；以及在使用某个第三方库前要仔细考量。

## 编译器和runtime的变化

runtime的变化上，gc一如既往地得到了小幅度优化，现在对于gc压力较大的程序来说gc延迟和内存占用都会有所减少。

cgo也有优化，现在cgo函数调用最大可以比原先快一个数量级。

编译器的变化上比较显著的是这个：PGO已经可以正式投入生产使用。[使用教程](https://go.dev/doc/pgo)。

PGO可以带来6%左右的性能提升，1.21凭借PGO和上个版本的优化现在不仅没有了泛型带来的编译速度减低，相比以前还有细微提升。

还有最后一个变化，这个和编译器关系：现在没有被使用的全局的map类型的变量（需要达到一定大小，且初始化的语句中没有任何副作用会产生），现在编译完成的程序里不会在包含这个变量。因为map本身占用内存且初始化需要花费一定时间（map越大花的时间越多）。这个好处是很实在的，既可以减小产生的二进制可执行文件的大小，又可以提升运行性能。但有个缺点，如果有什么程序要依赖编译好的可执行文件内部的某些数据的话，这个变更可能会带来麻烦，普通用户可以忽略这点。

## 新标准库

这个版本添加了大把的新标准库，一起来看看。

### log/slog和testing/slogtest

官方提供的结构化日志库。

可以通过实现`slog.Handler`来定义自己的日志处理器，可以把日志转换成json等格式。标准库自带了很多预定义的处理器，比如json的：

```golang
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
logger.Info("hello", "count", 3)
// {"time":"2023-08-09T15:28:26.000000000+09:00","level":"INFO","msg":"hello","count":3}
```

简单得说，就是个简化版的zap，如果想使用最基础的结构化日志的功能，又不想引入zap这样的库，那么slog是个很好的选择。

testing/slogtest里有帮助函数用来测试自己实现的日志处理器是否符合标准库的要求。

### slices和maps

把`golang.org/x/exp/slices`和`golang.org/x/exp/maps`引入了标准库。

slices库提供了排序、二分查找、拼接、增删改查等常用功能，sort这个标准库目前可以停止使用用slices来替代了。

maps提供了常见的对map的增删改查拼接合并等功能。

两个库使用泛型，且针对golang的slice和map进行了细致入微的优化，性能上比自己写的版本有更多优势，比标准库sort更是有数量级的碾压。

这两个库本来1.20就该被接收进标准库了，但因为需要重新设计api和进行优化，所以拖到1.21了。

### cmp

这个也是早该进入标准库的，但拖到了现在。随着slices、maps和新内置函数都进入了新版本，这个库想不接收也不行了。

这个库一共有三个东西：`Ordered`、`Less`、`Compare`。

最重要的是`Ordered`，它是所有可以使用内置运算符进行比较的类型的集合。

`Less`和`Ordered`顾名思义用来比大小的，且只能比`Ordered`类型的大小。

之所以还有单独造出这两个函数，是因为他们对Nan有检查，比如：

```golang
// Less reports whether x is less than y.
// For floating-point types, a NaN is considered less than any non-NaN,
// and -0.0 is not less than (is equal to) 0.0.
func Less[T Ordered](x, y T) bool {
	return (isNaN(x) && !isNaN(y)) || x < y
}
```

**所以在泛型函数里不知道要比较的数据的类型是不是有float的时候，用cmp里提供的函数是最安全的**。这就是他俩存在的意义。

但如果可以100%确定没有float存在，那么就不应该用`Less`等，应该直接用运算符去比较，因为大家都看到，Less和直接比较相比效率是较低的。

## 已有的标准库的变化

因为是速览，所以我只挑重点说。

### bytes

`bytes.Buffer`添加了`AvailableBuffer`和`Available`两个方法，分别返回目前可用的buf切片和可用的长度。主要可以配合`strconv.AppendInt`来使用，直接把数据写入buffer对应的内存里，可以提升性能。**不要对`AvailableBuffer`返回的切片扩容，否则必然踩坑**。

### context

新的`context.WithoutCancel`会把原来的`context.Context`复制一份，并去除cancel函数，这意味着原先被复制的上下文取消了这个新的上下文也将继续存在。例子：

```golang
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    newCtx := context.WithoutCancel(ctx)
    cancel()
    <-ctx.Done() // ok, ctx has cancled.
    <-newCtx.Done() // error: dead lock!
}
```

之所以会死锁，是因为`newCtx`没有被取消，Done返回的chan会永远阻塞住。而且更根本的，`newCtx`无法被取消。

新增了`context.WithDeadlineCause`和`context.WithTimeoutCause`，可以增加超时上下文被取消时的信息：

```golang
d := time.Now().Add(shortDuration)
ctx, cancel := context.WithDeadline(context.Background(), d, &MyError{"my message"})
cancel()
context.Cause(ctx) // --> &MyError{"my message"}
```

虽然不如`context.WithCancelCause`灵活，但也很实用。

### crypto/sha256

现在在x86_64平台上计算sha256会尽量利用硬件指令（simd和x86_64平台的SHA256ROUND等指令），这带来了3-4倍的性能提升。

### net

现在golang在Linux上已经初步支持Multipath TCP。有关Multipath TCP的信息可以在这查阅：<https://www.multipath-tcp.org/>

### reflect

ValueOf现在会根据逃逸分析把值分配在栈上，以前都是直接分配到堆上的。对于比较小的类型来说可以获得10%以上的性能提升。利好很多使用反射的ORM框架。

新增了`Value.Clear`，对应第一节的clear内置函数，如果type不是map或者slice的话这个函数和其他反射的方法一样会panic。

### runtime

最值得一提的变化是新增了`runtime.Pinner`。

它的能力是可以让某个go的对象不会gc回收，一直到`Unpin`方法被调用。这个是为了方便cgo代码里让c使用go的对象而设计的。

**不要滥用这个接口，如果想告诉gc某个对象暂时不能回收，应该正确使用`runtime.KeepAlive`**。

runtime/trace现在有了很大的性能提升，因此观察程序行为的时候开销更小，更接近程序真实的负载。

### sync

添加了`OnceFunc`、`OnceValue`、`OnceValues`这三个帮助函数。主要是为了简化代码。

1.21前：

```golang
var initFlag sync.Once

func GetSomeThing() {
    initFlag.Do(func(){
        真正的初始化逻辑
    })
    // 后续处理
}
```

现在变成：

```golang
var doInit = sync.OnceFunc(func(){
    真正的初始化逻辑
})

func GetSomeThing() {
    doInit()
    // 后续处理
}
```

新代码要简单点。

`OnceValue`、`OnceValues`是函数带返回值的版本，支持一个和两个返回值的函数。

### errors

新增了`errors.ErrUnsupported`。这个错误表示当前操作系统、硬件、协议、或者文件系统不支持某种操作。

目前os，filepath，syscall，io里的一些函数已经会返回这个错误，可以用`errors.Is(err, errors.ErrUnsupported)`来检查。

### unicode

升级到了最新的Unicode 15.0.0。

## 平台支持变化

新增了wasip1支持，这是一个对WASI（WebAssembly System Interface）的初步支持。

对于macOS，go1.21需要macOS 10.15 Catalina及以上版本。

龙芯上golang现在支持将代码编译为c的动态和静态链接库，基本上在龙芯上已经可以尝试投入生产环境了。

## 总结

上面就是我认为需要关注一下的go1.21的新特性，不是所有的变化都罗列在其中。

比如实验性支持的新的`for-range`语义我就没写，这个还在早期实验阶段，后续可能会被砍掉也可能行为上有极大变化，所以现在花精力关注不是很值得。

如果对其他变更有兴趣，可以看完整的[发版日志](https://go.dev/doc/go1.21)。

不过还是开头说的那句话，先别急着在生产环境上升级，可以等到第一个patch版本出来了再说。
