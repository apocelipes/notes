这是golang拾遗系列的第六篇。这个系列主要用来记录一些平时不常见的知识点，偶尔也会实现些有意思的小功能，比如这篇。

golang拾遗系列目录：

1. [golang拾遗：指针和接口](./golang拾遗：指针和接口.md)
2. [golang拾遗：为什么我们需要泛型](./golang拾遗：为什么我们需要泛型.md)
3. [golang拾遗：嵌入类型](./golang拾遗：嵌入类型.md)
4. [golang拾遗：内置函数len的小知识](./golang拾遗：内置函数len的小知识.md)
5. [golang拾遗：自定义类型和方法集](./golang拾遗：自定义类型和方法集.md)
6. **golang拾遗：实现一个不可复制类型**

在本篇中我们将实现一个无法被复制的类型，顺便加深对引用类型、值传递以及指针的理解。

阅读本文前需要你拥有一定的前置知识，包括掌握基本的golang语法，能理解并应用接口，对sync包下的内容有粗略的了解。如果你准备好了，就可以接着往下看了。

#### 本文索引
- [如何复制一个对象](#%E5%A6%82%E4%BD%95%E5%A4%8D%E5%88%B6%E4%B8%80%E4%B8%AA%E5%AF%B9%E8%B1%A1)
- [为什么要禁止复制](#%E4%B8%BA%E4%BB%80%E4%B9%88%E8%A6%81%E7%A6%81%E6%AD%A2%E5%A4%8D%E5%88%B6)
- [运行时检测实现禁止复制](#%E8%BF%90%E8%A1%8C%E6%97%B6%E6%A3%80%E6%B5%8B%E5%AE%9E%E7%8E%B0%E7%A6%81%E6%AD%A2%E5%A4%8D%E5%88%B6)
  - [初步尝试](#%E5%88%9D%E6%AD%A5%E5%B0%9D%E8%AF%95)
  - [更好的实现](#%E6%9B%B4%E5%A5%BD%E7%9A%84%E5%AE%9E%E7%8E%B0)
  - [性能](#%E6%80%A7%E8%83%BD)
  - [优点和缺点](#%E4%BC%98%E7%82%B9%E5%92%8C%E7%BC%BA%E7%82%B9)
- [静态检测实现禁止复制](#%E9%9D%99%E6%80%81%E6%A3%80%E6%B5%8B%E5%AE%9E%E7%8E%B0%E7%A6%81%E6%AD%A2%E5%A4%8D%E5%88%B6)
  - [利用Locker接口不可复制实现静态检测](#%E5%88%A9%E7%94%A8locker%E6%8E%A5%E5%8F%A3%E4%B8%8D%E5%8F%AF%E5%A4%8D%E5%88%B6%E5%AE%9E%E7%8E%B0%E9%9D%99%E6%80%81%E6%A3%80%E6%B5%8B)
  - [优点和缺点](#%E4%BC%98%E7%82%B9%E5%92%8C%E7%BC%BA%E7%82%B9)
- [更进一步](#%E6%9B%B4%E8%BF%9B%E4%B8%80%E6%AD%A5)
  - [利用package和interface进行封装](#%E5%88%A9%E7%94%A8package%E5%92%8Cinterface%E8%BF%9B%E8%A1%8C%E5%B0%81%E8%A3%85)
  - [优点和缺点](#%E4%BC%98%E7%82%B9%E5%92%8C%E7%BC%BA%E7%82%B9)
- [总结](#%E6%80%BB%E7%BB%93)

## 如何复制一个对象

不考虑IDE提供的代码分析和`go vet`之类的静态分析工具，golang里几乎所有的类型都能被复制。

```golang
// 基本标量类型和指针
var i int = 1
iCopy := i
str := "string"
strCopy := str

pointer := &i
pointerCopy := pointer
iCopy2 := *pointer // 解引用后进行复制

// 结构体和数组
arr := [...]int{1, 2, 3}
arrCopy := arr

type Obj struct {
    i int
}
obj := Obj{}
objCopy := obj
```

除了这些，golang还有函数和引用类型（slice、map、interface），这些类型也可以被复制，但稍有不同：

```golang
func f() {...}
f1 := f
f2 := f1

fmt.Println(f1, f2) // 0xabcdef 0xabcdef 打印出来的值是一样的
fmt.Println(&f1 == &f2) // false 虽然值一样，但确实是两个不同的变量
```

这里并没有真正复制处三份`f`的代码，`f1`和`f2`均指向`f`，f的代码始终只会有一份。map、slice和interface与之类似：

```golang
m := map[int]string{
    0: "a",
    1: "b",
}
mCopy := m // 两者引用同样的数据
mCopy[0] := "unknown"
m[0] == "unknown" // True
// slice的复制和map相同
```

interface是比较另类的，它的行为要分两种情况：

```golang
s := "string"
var i1 any = s
var i2 any = s
// 当把非指针和接口类型的值赋值给interface，会导致原来的对象被复制一份

s := "string"
var i1 any = s
var i2 any = i2
// 当把接口赋值给接口，底层引用的数据不会被复制，i1会复制s，i2此时和i1共有一个s的副本

ss := "string but pass by pointer"
var i3 any = &ss
var i4 any = i3
// i3和i4均引用ss，此时ss没有被复制，但指向ss的指针的值被复制了两次
```

上面的结果会一定程度上被编译优化干扰，比如少数情况下编译器可以确认赋值给接口的值从来没被修改并且生命周期不比源对象长，则可能不会进行复制。

所以这里有个小提示：如果要赋值给接口的数据比较大，那么最好以指针的形式赋值给接口，复制指针比复制大量的数据更高效。

## 为什么要禁止复制

从上一节可以看到，允许复制时会在某些情况下“闯祸”。比如：

1. 浅拷贝的问题很容易出现，比如例子里的map和slice的浅拷贝问题，这可能会导致数据被意外修改
2. 意外复制了大量数据，导致性能问题
3. 在需要共享状态的地方错误的使用了副本，导致状态不一致从而产生严重问题，比如`sync.Mutex`，复制一个锁并使用其副本会导致死锁
4. 根据业务或者其他需求，某类型的对象只允许存在一个实例，这时复制显然是被禁止的

显然在一些情况下禁止复制是合情合理的，这也是为什么我会写这篇文章。

但具体情况具体分析，不是说复制就是万恶之源，什么时候该支持复制，什么时候应该禁止，应该结合自己的实际情况。

## 运行时检测实现禁止复制

想在别的语言禁止某个类型被复制，方法有很多，用c++举一例：

```c++
struct NoCopy {
    NoCopy(const NoCopy &) = delete;
    NoCopy &operator=(const NoCopy &) = delete;
};
```

可惜在golang里不支持这么做。

另外，因为golang没有运算符重载，所以很难在赋值的阶段就进行拦截，所以我们的侧重点在于“复制之后可以尽快检测到”。

所以我们先实现在对象被复制后报错的功能。虽然不如c++编译期就可以禁止复制那样优雅，但也算实现了功能，至少不什么都没有要强一些。

### 初步尝试

那么如何直到对象是否被复制了？很简单，看它的地址就行了，地址一样那必然是同一个对象，不一样了那说明复制出一个新的对象了。

顺着这个思路，我们需要一个机制来保存对象第一次创建时的地址，并在后续进行比较，于是第一版代码诞生了：

```golang
import "unsafe"

type noCopy struct {
	p uintptr
}

func (nc *noCopy) check() {
	if uintptr(unsafe.Pointer(nc)) != nc.p {
		panic("copied")
	}
}
```

逻辑比较清晰，每次调用`check`来检查当前的调用者的地址和保存地址是否相同，如果不同就panic。

为什么没有创建这个类型的方法？因为我们没法得知自己被其他类型创建时的地址，所以这块得让其他使用`noCopy`的类型代劳。

使用的时候需要把`noCopy`嵌入自己的struct，注意不能以指针的形式嵌入：

```golang
type SomethingCannotCopy struct {
	noCopy
    ...
}

func (s *SomethingCannotCopy) DoWork() {
	s.check()
	fmt.Println("do something")
}

func NewSomethingCannotCopy() *SomethingCannotCopy {
	s := &SomethingCannotCopy{
        // 一些初始化
    }
    // 绑定地址
	s.noCopy.p = unsafe.Pointer(&s.noCopy)
	return s
}
```

注意初始化部分的代码，在这里我们需要把`noCopy`对象的地址绑定进去。现在可以实现运行时检测了：

```golang
func main() {
    s1 := NewSomethingCannotCopy()
    pointer := s1
    s1Copy := *s1 // 这里实际上进行了复制，但需要调用方法的时候才能检测到
    pointer.DoWork() // 正常打印出信息
    s1Copy.DoWork() // panic
}
```

解释下原理：当`SomethingCannotCopy`被复制的时候，`noCopy`也会被复制，因此复制出来的`noCopy`的地址和原先的那个是不一样的，但他们内部记录的`p`是一样的，这样当被复制出来的`noCopy`对象调用check方法的时候就会触发panic。这也是为什么不要用指针形式嵌入它的原因。

功能实现了，但代码实在是太丑，而且耦合严重：只要用了`noCopy`，就必须在创建对象的同时初始化noCopy的实例，noCopy的初始化逻辑会侵入到其他对象的初始化逻辑中，这样的设计是不能接受的。

### 更好的实现

那么有没有更好的实现？答案是有的，而且在标准库里。

标准库的信号量`sync.Cond`是禁止复制的，而且比Mutex更为严格，因为复制它比复制锁更容易导致死锁和崩溃，所以标准库加上了运行时的动态检查。

主要代码如下：

```golang
type Cond struct {
    // L is held while observing or changing the condition
    L Locker
    ...
    // 复制检查
    checker copyChecker
}

// NewCond returns a new Cond with Locker l.
func NewCond(l Locker) *Cond {
        return &Cond{L: l}
}

func (c *Cond) Signal() {
    // 检查自己是否被复制
    c.checker.check()
    runtime_notifyListNotifyOne(&c.notify)
}

```

`checker`实现了运行时检测是否被复制，但初始化的时候并不需要特殊处理这个`checker`，这是用了什么手法做到的呢？

看代码：

```golang
type copyChecker uintptr

func (c *copyChecker) check() {
    if uintptr(*c) != uintptr(unsafe.Pointer(c)) && // step 1
            !atomic.CompareAndSwapUintptr((*uintptr)(c), 0, uintptr(unsafe.Pointer(c))) && // step 2
            uintptr(*c) != uintptr(unsafe.Pointer(c)) { //step 3
        panic("sync.Cond is copied")
    }
}
```

看着很复杂，连原子操作都来了，这都是啥啊。但别怕，我给你捋一捋就明白了。

首先是checker初始化之后第一次调用：

1. 当check第一次被调用，c的值肯定是0，而这时候c是有真实的地址的，所以`step 1`失败，进入`step 2`；
2. 用原子操作把c的值设置成自己的地址值，注意只有c的值是0的时候才能完成设置，因为这里c的值是0，所以交换成功，`step 2`是`False`，判断流程直接结束；
3. 因为不排除还有别的goroutine拿着这个checker在做检测，所以`step 2`是会失败的，这是要进入`step 3`；
4. `step 3`再次比较c的值和它自己的地址是否相同，相同说明多个goroutine共用了一个checker，没有发生复制，所以检测通过不会panic。
5. 如果`step 3`的比较发现不相等，那么说明被复制了，直接panic

然后我们再看其他情况下checker的流程：

1. 这时候c的值不是0，如果没发生复制，那么`step 1`的结果是`False`，判断流程结束，不会panic；
2. 如果c的值和自己的地址不一样，会进入`step 2`，因为这里c的值不为0，所以表达式结果一定是`True`，所以进入`step 3`；
3. `step 3`和`step 1`一样，结果是`True`，地址不同说明被复制，这时候if里面的语句会执行，因此panic。

搞得这么麻烦，其实就是为了能干干净净地初始化。这样任何类型都只需要带上`checker`作为自己的字段就行，不用关心它是这么初始化的。

还有个小问题，为什么设置checker的值需要原子操作，但读取就不用呢？

因为读取一个uintptr的值，在现代的x86和arm处理器上只要一个指令，所以要么读到过时的值要么读到最新的值，不会读到错误的或者写了一半的不完整的值，对于读到旧值的情况（主要出现在第一次调用check的时候），还有`step 3`做进一步的检查，因此不会影响整个检测逻辑。而“比较并交换”显然一条指令做不完，如果在中间步骤被打断那么整个操作的结果很可能就是错的，从而影响整个检测逻辑，所以必须要用原子操作才行。

那么在读取的时候也使用`atomic.Load`行吗？当然行，但**一是这么做仍然避免不了`step 3`的检测，可以思考下是为什么**；二是原子操作相比直接读取会带来性能损失，在这里不使用原子操作也能保证正确性的情况下这是得不偿失的。

### 性能

因为是运行时检测，所以我们得看看会对性能带来多少影响。我们使用改进版的checker。

```golang
type CheckBench struct {
    num uint64
    checker copyChecker
}

func (c *CheckBench) CheckCopy() {
	c.checker.check()
	c.num++
}

// 不进行检测
func (c *CheckBench) NoCheck() {
	c.num++
}

func BenchmarkCheckBench_NoCheck(b *testing.B) {
	c := CheckBench{}
	for i := 0; i < b.N; i++ {
		for j := 0; j < 50; j++ {
			c.NoCheck()
		}
	}
}

func BenchmarkCheckBench_WithCheck(b *testing.B) {
	c := CheckBench{}
	for i := 0; i < b.N; i++ {
		for j := 0; j < 50; j++ {
			c.CheckCopy()
		}
	}
}
```

测试结果如下：

```text
cpu: Intel(R) Core(TM) i5-10200H CPU @ 2.40GHz
BenchmarkCheckBench_NoCheck-8           17689137                68.36 ns/op
BenchmarkCheckBench_WithCheck-8         17563833                66.04 ns/op
```

几乎可以忽略不计，因为我们这里没有发生复制，所以几乎每次检测都是通过的，这对cpu的分支预测非常友好，所以性能损耗几乎可以忽略。

所以我们给cpu添点堵，让分支预测没那么容易：

```golang
func BenchmarkCheckBench_WithCheck(b *testing.B) {
	for i := 0; i < b.N; i++ {
		c := &CheckBench{}
		for j := 0; j < 50; j++ {
			c.CheckCopy()
		}
	}
}

func BenchmarkCheckBench_NoCheck(b *testing.B) {
	for i := 0; i < b.N; i++ {
		c := &CheckBench{}
		for j := 0; j < 50; j++ {
			c.NoCheck()
		}
	}
}
```

现在分支预测没那么容易了而且要多付出初始化时使用atomic的代价，测试结果会变成这样：

```text
cpu: Intel(R) Core(TM) i5-10200H CPU @ 2.40GHz
BenchmarkCheckBench_WithCheck-8         15552717                74.84 ns/op
BenchmarkCheckBench_NoCheck-8           26441635                44.74 ns/op
```

差不多会慢40%。当然，实际的代码不会有这么极端，所以最坏可能也只会产生20%的影响，通常不太会成为性能瓶颈，运行时检测是否有影响还需结核profile。

### 优点和缺点

优点：

1. 只要调用check，肯定能检查出是否被复制
2. 简单

缺点：

1. 所有的方法里都需要调用check，新加方法忘了调用的话就无法检测
2. 只能在被复制出来的新对象那检测到复制操作，原先那个对象上check始终是没问题的，这样不是严格禁止了复制，但大多数时间没问题，可以接受
3. 如果只复制了对象没调用任何对象上的方法，也无法检测到复制，这种情况比较少见
4. 有潜在性能损耗，虽然很多时候可以得到充分优化损耗没那么夸张

## 静态检测实现禁止复制

动态检测的缺点不少，能不能像c++那样编译期就禁止复制呢？

### 利用Locker接口不可复制实现静态检测

也可以，但得配合静态代码检测工具，比如自带的`go vet`。看下代码：

```golang
// 实现sync.Locker接口
type noCopy struct{}
func (*noCopy) Lock() {}
func (*noCopy) Unlock() {}

type SomethingCannotCopy struct {
    noCopy
}
```

这样就行了，不需要再添加其他的代码。解释下原理：任何实现了`sync.Locker`的类型都不应该被拷贝，静态代码检测会检测出这些情况并报错。

所以类似下边的代码都是无法通过静态代码检测的：

```golang
func f(s SomethingCannotCopy) {
    // 报错，因为参数会导致复制
    // 返回SomethingCannotCopy也是不行的
}

func (s SomethingCannotCopy) Method() {
    // 报错，因为非指针类型接收器会导致复制
}

func main() {
    s := SomethingCannotCopy{}
    sCopy := s // 报错
    sInterface := any(s) // 报错
    sPointer := &s // OK
    sCopy2 := *sPointer // 报错
    sInterface2 := any(sPointer) // OK
    sCopy3 := *(sInterface2.(*SomethingCannotCopy)) // 报错
}
```

基本上涵盖了所以会产生复制操作的地方，基本能在编译期完成检测。

如果跳过`go vet`，直接使用`go run`或者`go build`，那么上面的代码可以正常编译并运行。

如果不想意外实现`Locker`接口，一种更常见的做法是使用下划线字段而不是嵌入字段：

```golang
type Obj struct {
    _ noCopy
    data int
}
```

因为复制结构体的时候每个字段也会被复制，这时候`noCopy`类型的字段的复制就会被静态检测工具检测到。同时因为是下划线字段，这样的字段不会实际占用内存，内部和外部也都访问不到，因此类型本身也不会意外实现`Locker`接口。当然代价也不是没有的，这样的类型没法简单地使用字面量初始化，想要在本package之外的地方被使用则需要额外的函数来帮忙生成实例。

### 优点和缺点

因为只有静态检测，因此没有什么运行时开销，所以性能这节就不需要费笔墨了。主要来看下这种方案的优缺点。

优点：

1. 实现非常简单，代码很简练，基本无侵入性
2. 依赖静态检测，不影响运行时性能
3. golang自带检测工具：go vet
4. 可检测到的case比运行时检测多

缺点：

1. 最大的缺点，尽管静态检测会报错，但仍然可以正常编译执行
2. 不是每个测试环境和CI都配备了静态检测，所以很难强制保证类型没有被复制
3. 不使用下划线字段会导致类型实现`sync.Locker`，然而很多时候我们的类型并不是类似锁的资源，使用这个接口只是为了静态检测，这会带来代码被误用的风险
4. 使用下划线字段不会有意外实现接口的问题，但会使类型的创建变得复杂

标准库也使用的这套方案，建议仔细阅读这个[issue](https://github.com/golang/go/issues/8005#issuecomment-190753527)里的讨论。

标准库里还有使用这个方案的实例：<https://github.com/golang/go/blob/master/src/sync/atomic/type.go#L68>

## 更进一步

看过运行时检测和静态检测两种方案之后，我们会发现这些做法多少都有些问题，不尽如人意。

所以我们还是要追求一种更好用的，更符合golang风格的做法。幸运的是，这样的做法是存在的。

### 利用package和interface进行封装

首先我们创建一个worker包，里面定义一个`Worker`接口，包中的数据对外以`Worker`接口的形式提供：

```golang
package worker

import (
	"fmt"
)

// 对外只提供接口来访问数据
type Worker interface {
	Work()
}

// 内部类型不导出，以接口的形式供外部使用
type normalWorker struct {
	// data members
}
func (*normalWorker) Work() {
	fmt.Println("I am a normal worker.")
}
func NewNormalWorker() Worker {
	return &normalWorker{}
}

type specialWorker struct {
	// data members
}
func (*specialWorker) Work() {
	fmt.Println("I am a special worker.")
}
func NewSpecialWorker() Worker {
	return &specialWorker{}
}
```

worker包对外只提供`Worker`接口，用户可以使用`NewNormalWorker`和`NewSpecialWorker`来生成不同种类的worker，用户不需要关心具体的返回类型，只要使用得到的`Worker`接口即可。

这么做的话，在worker包之外是看不到`normalWorker`和`specialWorker`这两个类型的，所以没法靠反射和类型断言取出接口引用的数据；因为我们传给接口的是指针，因此源数据不会被复制；同时我们在第一节提到过，把一个接口赋值给另一个接口（worker包之外你只能这么做），底层被引用的数据不会被复制，因此在包外始终不会在这两个类型上产生复制的行为。

因此下面这样的代码是不可能通过编译的：

```golang
func main() {
    w := worker.NewSpecialWorker()
    // worker.specialWorker 在worker包以外不可见，因此编译错误
    wCopy := *(w.(*worker.specialWorker))
    wCopy.Work()
}
```

### 优点和缺点

这样就实现了worker包之外的禁止复制，下面来看看优缺点。

优点：

1. 不需要额外的静态检查工具在编译代码前执行检查
2. 不需要运行时动态检测是否被复制
3. 不会实现自己不需要的接口类型导致污染方法集
4. 符合golang开发中的习惯做法

缺点：

1. 并没有让类型本身不可复制，而是靠封装屏蔽了大部分可能导致复制的情况
2. 这些worker类型在包内是可见的，如果在包内修改代码时不注意可能会导致复制这些类型的值，所以要么包内也都用Woker接口，要么参考上一节添加静态检查
3. 有些场景下不需要接口或者因为性能要求苛刻而使用不了接口，这种做法就行不通了，比如标准库sync里的类型为了性能大部分都是暴露出来给外部直接使用的

综合来说，这种方案是实现成本最低的。

## 总结

现在我们有三种方式防止我们的类型被复制：

1. 运行时检测
2. 静态代码检测
3. 通过接口封装避免暴露类型，从而避免被复制

一共三种方案，选择困难症仿佛要发作了。别着急，我们一起看看标准库是怎么做的：

1. 标准库的`sync.Cond`同时使用了方案一和方案二，因为设计者确实很不希望条件变量被复制
2. `sync.Mutex`、`sync.Pool`和`sync.WaitGroup`使用了方案二，需要配合`go vet`
3. 方案三在标准库中应用最广泛，然而多数是处于设计和封装的考虑，并不是为了禁止copy，但复制`crypto`包下的那些`Hash`和`Cipher`确实没什么意义会带来误用，正好借着方案三避免了这些问题

综合来看首选的应该是方案三；但也有需要使用方案二的时候，比如sync包中的那些同步机构；使用最少的是方案一，尽可能地不要设计出类似的代码。

还有一点需要注意，如果你的类型里有字段是`sync.Pool`、`sync.WaitGroup`、`sync.RWMutex`、`sync.Mutex`、`sync.Cond`、`sync.Map`或`sync.Once`，那么这个类型本身也是不可复制的，也不需要额外实现禁止复制的功能，因为那些字段自带了。

最后，我只想说golang的语言技能实在是太简陋了，想只依赖语言特性实现禁止复制的功能不太现实，更多的还是需要靠“设计”。
