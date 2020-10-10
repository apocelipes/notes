这是本系列的第一篇文章，golang拾遗主要是用来记录一些遗忘了的、平时从没注意过的golang相关知识。想做本系列的契机其实是因为疫情闲着在家无聊，网上冲浪的时候发现了zhuihu上的[go语言爱好者周刊](https://www.zhihu.com/column/polaris)和[Go 101](https://go101.org/article/101.html)，读之如醍醐灌顶，受益匪浅，于是本系列的文章就诞生了。拾遗主要是收集和golang相关的琐碎知识，当然也会对周刊和101的内容做一些补充说明。好了，题外话就此打住，下面该进入今天的正题了。

## 指针和接口

golang的类型系统其实很有意思，有意思的地方就在于类型系统表面上看起来众生平等，然而实际上却要分成普通类型（types）和接口（interfaces）来看待。普通类型也包含了所谓的引用类型，例如`slice`和`map`，虽然他们和`interface`同为引用类型，但是行为更趋近于普通的内置类型和自定义类型，因此只有特立独行的`interface`会被单独归类。

那我们是依据什么把golang的类型分成两类的呢？其实很简单，看类型能不能在编译期就确定以及调用的类型方法是否能在编译期被确定。

如果觉得上面的解释太过抽象的可以先看一下下面的例子：

```golang
package main

import "fmt"

func main(){
    m := make(map[int]int)
    m[1] = 1 * 2
    m[2] = 2 * 2
    fmt.Println(m)
    m2 := make(map[string]int)
    m2["python"] = 1
    m2["golang"] = 2
    fmt.Println(m2)
}
```

首先我们来看非interface的引用类型，`m`和`m2`明显是两个不同的类型，不过实际上在底层他们是一样的，不信我们用objdump工具检查一下：

```assembly
go tool objdump -s 'main\.main' a

TEXT main.main(SB) /tmp/a.go
  a.go:6  CALL runtime.makemap_small(SB)     # m := make(map[int]int)
  ...
  a.go:7  CALL runtime.mapassign_fast64(SB)  # m[1] = 1 * 2
  ...
  a.go:8  CALL runtime.mapassign_fast64(SB)  # m[2] = 2 * 2
  ...
  ...
  a.go:10 CALL runtime.makemap_small(SB)     # m2 := make(map[string]int)
  ...
  a.go:11 CALL runtime.mapassign_faststr(SB) # m2["python"] = 1
  ...
  a.go:12 CALL runtime.mapassign_faststr(SB) # m2["golang"] = 2
```

省略了一些寄存器的操作和无关函数的调用，顺便加上了对应的代码的原文，我们可以清晰地看到尽管类型不同，但map调用的方法都是相同的而且是编译期就已经确定的。如果是自定义类型呢？

```golang
package main

import "fmt"

type Person struct {
    name string
    age int
}

func (p *Person) sayHello() {
    fmt.Printf("Hello, I'm %v, %v year(s) old\n", p.name, p.age)
}

func main(){
    p := Person{
        name: "apocelipes",
        age: 100,
    }
    p.sayHello()
}
```

这次我们创建了一个拥有自定义字段和方法的自定义类型，下面再用objdump检查一下：

```assembly
go tool objdump -s 'main\.main' b

TEXT main.main(SB) /tmp/b.go
  ...
  b.go:19   CALL main.(*Person).sayHello(SB)
  ...
```

用字面量创建对象和初始化调用堆栈的汇编代码不是重点，重点在于那句`CALL`，我们可以看到自定义类型的方法也是在编译期就确定了的。

那反过来看看interface会有什么区别：

```golang
package main

import "fmt"

type Worker interface {
    Work()
}

type Typist struct{}
func (*Typist)Work() {
    fmt.Println("Typing...")
}

type Programer struct{}
func (*Programer)Work() {
    fmt.Println("Programming...")
}

func main(){
    var w Worker = &Typist{}
    w.Work()
    w = &Programer{}
    w.Work()
}
```

注意！编译这个程序需要禁止编译器进行优化，否则编译器会把接口的方法查找直接优化为特定类型的方法调用：

```assembly
go build -gcflags "-N -l" c.go
go tool objdump -S -s 'main\.main' c

TEXT main.main(SB) /tmp/c.go
  ...
  var w Worker = &Typist{}
    LEAQ runtime.zerobase(SB), AX
    MOVQ AX, 0x10(SP)
    MOVQ AX, 0x20(SP)
    LEAQ go.itab.*main.Typist,main.Worker(SB), CX
    MOVQ CX, 0x28(SP)
    MOVQ AX, 0x30(SP)
  w.Work()
    MOVQ 0x28(SP), AX
    TESTB AL, 0(AX)
    MOVQ 0x18(AX), AX
    MOVQ 0x30(SP), CX
    MOVQ CX, 0(SP)
    CALL AX
  w = &Programer{}
    LEAQ runtime.zerobase(SB), AX
    MOVQ AX, 0x8(SP)
    MOVQ AX, 0x18(SP)
    LEAQ go.itab.*main.Programer,main.Worker(SB), CX
    MOVQ CX, 0x28(SP)
    MOVQ AX, 0x30(SP)
  w.Work()
    MOVQ 0x28(SP), AX
    TESTB AL, 0(AX)
    MOVQ 0x18(AX), AX
    MOVQ 0x30(SP), CX
    MOVQ CX, 0(SP)
    CALL AX
  ...
```

这次我们可以看到调用接口的方法会去在runtime进行查找，随后`CALL`找到的地址，而不是像之前那样在编译期就能找到对应的函数直接调用。这就是interface为什么特殊的原因：interface是动态变化的类型。

可以动态变化的类型最显而易见的好处是给予程序高度的灵活性，但灵活性是要付出代价的，主要在两方面。

一是性能代价。动态的方法查找总是要比编译期就能确定的方法调用多花费几条汇编指令（mov和lea通常都是会产生实际指令的），数量累计后就会产生性能影响。不过好消息是通常编译器对我们的代码进行了优化，例如`c.go`中如果我们不关闭编译器的优化，那么编译器会在编译期间就替我们完成方法的查找，实际生产的代码里不会有动态查找的内容。然而坏消息是这种优化需要编译器可以在编译期确定接口引用数据的实际类型，考虑如下代码：

```golang
type Worker interface {
    Work()
}

for _, v := workers {
    v.Work()
}
```

因为只要实现了`Worker`接口的类型就可以把自己的实例塞进`workers`切片里，所以编译器不能确定v引用的数据的类型，优化自然也无从谈起了。

而另一个代价，确切地说其实应该叫陷阱，就是接下来我们要探讨的主题了。

## golang的指针

指针也是一个极有探讨价值的话题，特别是指针在reflect以及runtime包里的各种黑科技。不过放轻松，今天我们只用了解下指针的自动解引用。

我们把`b.go`里的代码改动一行：

```golang
p := &Person{
    name: "apocelipes",
    age: 100,
}
```

p现在是个指针，其余代码不需要任何改动，程序依旧可以正常编译执行。对应的汇编是这样的画风（当然得关闭优化）：

```assembly
p.sayHello()
    MOVQ AX, 0(SP)
    CALL main.(*Person).sayHello(SB)
```

对比一下非指针版本：

```assembly
p.sayHello()
    LEAQ 0x8(SP), AX
    MOVQ AX, 0(SP)
    CALL main.(*Person).sayHello(SB)
```

与其说是指针自动解引用，倒不如说是非指针版本先求出了对象的实际地址，随后传入了这个地址作为方法的接收器调用了方法。这也没什么好奇怪的，因为我们的方法是指针接收器：P。

如果把接收器换成值类型接收器：

```assembly
p.sayHello()
    TESTB AL, 0(AX)
    MOVQ 0x40(SP), AX
    MOVQ 0x48(SP), CX
    MOVQ 0x50(SP), DX
    MOVQ AX, 0x28(SP)
    MOVQ CX, 0x30(SP)
    MOVQ DX, 0x38(SP)
    MOVQ AX, 0(SP)
    MOVQ CX, 0x8(SP)
    MOVQ DX, 0x10(SP)
    CALL main.Person.sayHello(SB)
```

作为对比：

```assembly
p.sayHello()
    MOVQ AX, 0(SP)
    MOVQ $0xa, 0x8(SP)
    MOVQ $0x64, 0x10(SP)
    CALL main.Person.sayHello(SB)
```

这时候golang就是先检查指针随后解引用了。同时要注意，这里的方法调用是已经在编译期确定了的。

## 指向interface的指针

铺垫了这么久，终于该进入正题了。不过在此之前还有一点小小的预备知识需要提一下：

> A pointer type denotes the set of all pointers to variables of a given type, called the base type of the pointer.     --- go language spec

换而言之，只要是能取地址的类型就有对应的指针类型，比较巧的是在golang里引用类型是可以取地址的，包括interface。

有了这些铺垫，现在我们可以看一下我们的说唱歌手程序了：

```golang
package main

import "fmt"

type Rapper interface {
    Rap() string
}

type Dean struct {}

func (_ Dean) Rap() string {
    return "Im a rapper"
}

func doRap(p *Rapper) {
    fmt.Println(p.Rap())
}

func main(){
    i := new(Rapper)
    *i = Dean{}
    fmt.Println(i.Rap())
    doRap(i)
}
```

问题来了，小青年Dean能圆自己的说唱梦么？

很遗憾，编译器给出了反对意见：

```text
# command-line-arguments
./rapper.go:16:18: p.Rap undefined (type *Rapper is pointer to interface, not interface)
./rapper.go:22:18: i.Rap undefined (type *Rapper is pointer to interface, not interface)
```

也许`type *XXX is pointer to interface, not interface`这个错误你并不陌生，你曾经也犯过用指针指向interface的错误，经过一番搜索后你找到了一篇教程，或者是博客，有或者是随便什么地方的资料，他们都会告诉你不应该用指针去指向接口，接口本身是引用类型无需再用指针去引用。

其实他们只说对了一半，事实上只要把i和p改成接口类型就可以正常编译运行了。没说对的一半是指针可以指向接口，也可以使用接口的方法，但是要绕些弯路（当然，用指针引用接口通常是多此一举，所以听从经验之谈也没什么不好的）：

```golang
func doRap(p *Rapper) {
    fmt.Println((*p).Rap())
}

func main(){
    i := new(Rapper)
    *i = Dean{}
    fmt.Println((*i).Rap())
    doRap(i)
}
```

```bash
go run rapper.go 

Im a rapper
Im a rapper
```

神奇的一幕出现了，程序不仅没报错而且运行得很正常。但是这和golang对指针的自动解引用有什么区别呢？明明看起来都一样但就是第一种方案会报
找不到`Rap`方法？

为了方便观察，我们把调用语句单独抽出来，然后查看未优化过的汇编码：

```assembly
s := (*p).Rap()
  0x498ee1              488b842488000000        MOVQ 0x88(SP), AX
  0x498ee9              8400                    TESTB AL, 0(AX)
  0x498eeb              488b08                  MOVQ 0(AX), CX
  0x498eee              8401                    TESTB AL, 0(CX)
  0x498ef0              488b4008                MOVQ 0x8(AX), AX
  0x498ef4              488b4918                MOVQ 0x18(CX), CX
  0x498ef8              48890424                MOVQ AX, 0(SP)
  0x498efc              ffd1                    CALL CX
```

抛开手工解引用的部分，后6行其实和直接使用interface进行动态查询是一样的。真正的问题其实出在自动解引用上：

```assembly
p.sayHello()
    TESTB AL, 0(AX)
    MOVQ 0x40(SP), AX
    MOVQ 0x48(SP), CX
    MOVQ 0x50(SP), DX
    MOVQ AX, 0x28(SP)
    MOVQ CX, 0x30(SP)
    MOVQ DX, 0x38(SP)
    MOVQ AX, 0(SP)
    MOVQ CX, 0x8(SP)
    MOVQ DX, 0x10(SP)
    CALL main.Person.sayHello(SB)
```

不同之处就在于这个`CALL`上，自动解引用时的`CALL`其实是把指针指向的内容视作_普通类型_，因此会去静态查找方法进行调用，而指向的内容是interface的时候，编译器会去interface本身的数据结构上去查找有没有`Rap`这个方法，答案显然是没有，所以爆了`p.Rap undefined`错误。

那么interface的真实长相是什么呢，我们看看go1.15.2的实现：

```golang
// src/runtime/runtime2.go
// 因为这边没使用空接口，所以只节选了含数据接口的实现
type iface struct {
	tab  *itab
	data unsafe.Pointer
}

// src/runtime/runtime2.go
type itab struct {
	inter *interfacetype
	_type *_type
	hash  uint32 // copy of _type.hash. Used for type switches.
	_     [4]byte
	fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}

// src/runtime/type.go
type imethod struct {
	name nameOff
	ityp typeOff
}

type interfacetype struct {
	typ     _type
	pkgpath name
	mhdr    []imethod // 类型所包含的全部方法
}

// src/runtime/type.go
type _type struct {
	size       uintptr
	ptrdata    uintptr // size of memory prefix holding all pointers
	hash       uint32
	tflag      tflag
	align      uint8
	fieldAlign uint8
	kind       uint8
	// function for comparing objects of this type
	// (ptr to object A, ptr to object B) -> ==?
	equal func(unsafe.Pointer, unsafe.Pointer) bool
	// gcdata stores the GC type data for the garbage collector.
	// If the KindGCProg bit is set in kind, gcdata is a GC program.
	// Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
	gcdata    *byte
	str       nameOff
	ptrToThis typeOff
}
```

没有给出定义的类型都是对各种整数类型的typing alias。`interface`实际上就是存储类型信息和实际数据的`struct`，自动解引用后编译器是直接查看内存内容的（见汇编），这时看到的其实是`iface`这个普通类型，所以静态查找一个不存在的方法就失败了。而为什么手动解引用的代码可以运行？因为我们手动解引用后编译器可以推导出实际类型是interface，这时候编译器就很自然地用处理interface的方法去处理它而不是直接把内存里的东西寻址后塞进寄存器。

## 总结

其实也没什么好总结的。只有两点需要记住，一是interface是有自己对应的实体数据结构的，二是尽量不要用指针去指向interface，因为golang对指针自动解引用的处理会带来陷阱。

如果你对interface的实现很感兴趣的话，这里有个reflect+暴力穷举实现的[乞丐版](https://www.tapirgames.com/blog/golang-interface-implementation)。

理解了乞丐版的基础上如果有兴趣还可以看看真正的golang实现，数据的层次结构上更细化，而且有使用指针和内存偏移等的聪明办法，不说是否会有收获，起码研究起来不会无聊：P。
