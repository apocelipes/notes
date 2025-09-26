目前golang 1.26的各种新特性还在开发中，不过其中一个在开发完成之前就已经被官方拿到台面上进行宣传了——内置函数new功能扩展。

每个新特性其实都有它的背景故事，没有需求的驱动也就不会有新特性的诞生。所以在介绍这个新特性之前我们先来了解下是什么样的场景催生了这个功能。

如果你经常浏览一些大型的go项目，尤其是那些需要频繁和JSON、GRPC或者yaml打交道的项目，比如k8s，你会发现这些代码库会提供一些和下面代码类似的帮助函数：

```go
func getPointerValue[T any](v T) *T {
	return &v
}
```

这个是我用泛型改写的，代码库里通常都是`getIntPointerValue(int) *int`这样非泛型函数。函数的作用很简单，返回指向自己参数的指针。但这样简单的三行代码有什么用呢？

用处有好几个，第一个是在json或者rpc里有时候我们会用指针的nil来表示这个值没有生效，和字段类型的零值做区分，但这使得给字段赋值变麻烦了：

```go
type Data struct {
    Num *uint
}

d := &Data{}
d.Num = &12345 // 编译错误
d.Num = getPointerValue(12345)
```

这行代码`d.Num = &12345`是语法错误，因为在golang里规定不能对字面量以及常量取地址。不仅如此，类似`d.Num = &getNum()`这样的代码也是无法编译的，因为go也规定了不能对右值取地址。

如果没有帮助函数，我们需要用一个中间变量接住这些值，然后再把这个中间变量的指针赋值给结构体的字段。

第二个作用在于防止潜在的内存泄漏：

```go
type BigStruct struct {
    // 100个其他字段
    Num int
}

bigObj := &BigStruct{....}
bigSlice := make([]int, 1024)

d1.Num = &bigObj.Num
d2.Num = &bigSlice[1000]
```

猜猜如果`d1`和`d2`需要很长时间才能被释放会发生什么。答案是`bigObj`和`bigSlice`也会一直存在不被释放，因为golang中结构体、数组/切片只要还有指针指向自己的字段或者元素，那么整个结构体和数组/切片的内存都不能被释放。换句话说因为你的Data结构体持有了一个8字节的指针，会导致它背后十几KB的内存一直没法释放，尽管这些内存中的99%你完全用不到。这在比较宽泛的定义上已经属于是内存泄漏了。

所以这时候帮助函数就起作用了。`getPointerValue`的参数不是指针，因此会把传进来的值拷贝一份，然后再取拷贝出来的新变量的指针，这样就不会有指针指向那些大对象的字段或者元素了，这些大对象也可以尽快得到释放从而不会浪费内存。

背景故事到此结束，到这里其实你也能猜出new被扩展的新功能大致是什么了。

new在1.26中获得的新功能是可以接受一个表达式，它会复制表达式的结果到同类型的变量里并返回指向这个变量的指针。

看个例子：

```go
new(1234) // *int, 指向的值是1234

func getString() string {
    return "apocelipes"
}
new(getString()) // *string, 指向的值是"apocelipes"

s := "Hello, "
new(s + getString() + "!") // *string, 指向的值是表达式的结果"Hello, apocelipes!"
```

功能很简单，相当于把上面的帮助函数`getPointerValue`集成到了现有的内置函数`new`里。这能让我们简化一些代码。

不过按照go团队以往的做法，如果只是简化代码的话其实是不会在原有的内置函数上新增功能的。现在这么做了说明还有额外的好处——性能。

我们看个性能测试：

```go
func BenchmarkOld(b *testing.B) {
	for b.Loop() {
		p := getPointerValue(123)
		if p == nil || *p != 123 {
			b.Fatal()
		}
	}
}

func BenchmarkNew(b *testing.B) {
	for b.Loop() {
		p := new(123)
		if p == nil || *p != 123 {
			b.Fatal()
		}
	}
}
```

这段代码需要master分支上的go编译器才能正常编译运行，我使用的版本是`go1.26-devel_d7a38adf4c`。

结果：

```console
goos: darwin
goarch: arm64
pkg: newnew
cpu: Apple M4
BenchmarkOld-10         200946866                5.698 ns/op           8 B/op          1 allocs/op
BenchmarkOld-10         215901068                5.548 ns/op           8 B/op          1 allocs/op
BenchmarkOld-10         215598393                5.568 ns/op           8 B/op          1 allocs/op
BenchmarkNew-10         755338909                1.590 ns/op           0 B/op          0 allocs/op
BenchmarkNew-10         754724300                1.589 ns/op           0 B/op          0 allocs/op
BenchmarkNew-10         754838827                1.592 ns/op           0 B/op          0 allocs/op
```

可以看到使用帮助函数要额外多分配一次内存，速度也更慢。这是因为golang的逃逸分析主要保证内存安全，而在优化上比较保守，所以在处理我们的帮助函数时哪怕这个函数已经被内联，编译器还是会选择分配一块堆内存再返回指向这块内存的指针。换句话说，编译器不够“聪明”。

但内置函数就不一样了，内置函数是被编译器特殊处理的，new会被编译器改写：

```go
p1 := new(int)
// 改写成
// var tmp int
// p1 := &tmp

p2 := new(12345)
// 改写成
// var tmp int
// tmp = copy 12345
// p2 := &tmp
```

可以看到new是先在当前作用域里创建一个临时变量，然后再把表达式的结果复制进去的。全程没有其他的函数调用。

对于改写后的代码，逃逸分析有充足的信息来决定改写产生的`tmp`应该分配在栈上还是堆上，比起帮助函数来说获得了更多的优化机会，因此性能也更好。

所以官方才有底气提前宣传，毕竟不仅解决了痛点，还有额外的收获。

## 总结

1.26开始内置函数new的参数除了能接受一个类型名称，现在还可以接收任意的表达式了。

在新版本中我们可以直接利用内置函数new不需要写帮助函数了，同时还能收获更高的性能。

当然，1.26的新特性开发窗口还没结束，不能保证最终发布的功能和文章里介绍的一模一样，但看官方这架势这个新特性大概率是板上钉钉了，先用这篇文章尝个鲜也未尝不可。
