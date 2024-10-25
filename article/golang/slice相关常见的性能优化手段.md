介绍一些开发中常用的slice关联的性能优化手段。鉴于golang编译器本身捉鸡的优化能力，优化的成本就得分摊在开发者自己的头上了。

这篇文章会介绍的优化手段是下面这几样：

1. 创建slice时预分配内存
2. 操作slice前预分配内存
3. slice表达式中合理设置cap值
4. 添加多个零值元素的优化
5. 循环展开
6. 避免for-range复制数据带来的损耗
7. 边界检查消除
8. 并行处理slice
9. 复用slice的内存
10. 高效删除多个元素
11. 减轻GC扫描压力

这篇文章不会讨论缓存命中率和SIMD，我知道这两样也和slice的性能相关，但前者我认为是合格的开发者必须要了解的，网上优秀的教程也很多不需要我再赘述，后者除非性能瓶颈真的在数据吞吐量上否则一般不应该纳入考虑范围尤其在go语言里，所以这两个主题本文不会介绍。

最后开篇之前我还想提醒一下，性能瓶颈要靠测试和profile来定位，性能优化方案的收益和开销也需要性能测试来衡量，切记不可生搬硬套。

本文比较长，所以我建议可以挑自己感兴趣的内容看，有时间再通读。

<blockquote id="bookmark">
  <h4>本文索引</h4>
  <ul>
    <li><a href="#%E5%88%9B%E5%BB%BAslice%E6%97%B6%E9%A2%84%E5%88%86%E9%85%8D%E5%86%85%E5%AD%98">创建slice时预分配内存</a></li>
    <li><a href="#%E6%93%8D%E4%BD%9Cslice%E5%89%8D%E9%A2%84%E5%88%86%E9%85%8D%E5%86%85%E5%AD%98">操作slice前预分配内存</a></li>
    <li><a href="#slice%E8%A1%A8%E8%BE%BE%E5%BC%8F%E4%B8%AD%E5%90%88%E7%90%86%E8%AE%BE%E7%BD%AEcap%E5%80%BC">slice表达式中合理设置cap值</a></li>
    <li><a href="#%E5%90%91slice%E6%B7%BB%E5%8A%A0%E5%A4%9A%E4%B8%AA%E9%9B%B6%E5%80%BC%E5%85%83%E7%B4%A0%E7%9A%84%E4%BC%98%E5%8C%96">向slice添加多个零值元素的优化</a></li>
    <li><a href="#%E5%BE%AA%E7%8E%AF%E5%B1%95%E5%BC%80">循环展开</a></li>
    <li>
      <a href="#%E9%81%BF%E5%85%8Dfor-ranges%E5%A4%8D%E5%88%B6%E6%95%B0%E6%8D%AE%E5%B8%A6%E6%9D%A5%E7%9A%84%E6%8D%9F%E8%80%97">避免for-ranges复制数据带来的损耗</a>
      <ul>
        <li><a href="#%E9%81%BF%E5%85%8D%E5%A4%8D%E5%88%B6">避免复制</a></li>
        <li><a href="#%E9%81%8D%E5%8E%86%E5%AD%97%E7%AC%A6%E4%B8%B2%E7%9A%84%E6%97%B6%E5%80%99%E9%81%BF%E5%85%8D%E8%BD%AC%E6%8D%A2%E5%B8%A6%E6%9D%A5%E7%9A%84%E5%BC%80%E9%94%80">遍历字符串的时候避免转换带来的开销</a></li>
      </ul>
    </li>
    <li><a href="#bce%E8%BE%B9%E7%95%8C%E6%A3%80%E6%9F%A5%E6%B6%88%E9%99%A4">BCE边界检查消除</a></li>
    <li><a href="#%E5%B9%B6%E8%A1%8C%E5%A4%84%E7%90%86slice">并行处理slice</a></li>
    <li><a href="#%E5%A4%8D%E7%94%A8">复用</a></li>
    <li>
      <a href="#%E9%AB%98%E6%95%88%E5%88%A0%E9%99%A4%E5%A4%9A%E4%B8%AA%E5%85%83%E7%B4%A0">高效删除多个元素</a>
      <ul>
        <li><a href="#%E5%88%A0%E9%99%A4%E6%89%80%E6%9C%89%E5%85%83%E7%B4%A0">删除所有元素</a></li>
        <li><a href="#%E5%88%A0%E9%99%A4%E5%A4%B4%E9%83%A8%E6%88%96%E5%B0%BE%E9%83%A8%E7%9A%84%E5%85%83%E7%B4%A0">删除头部或尾部的元素</a></li>
        <li><a href="#%E5%88%A0%E9%99%A4%E5%9C%A8%E4%B8%AD%E9%97%B4%E4%BD%8D%E7%BD%AE%E7%9A%84%E5%85%83%E7%B4%A0">删除在中间位置的元素</a></li>
      </ul>
    </li>
    <li><a href="#%E5%87%8F%E8%BD%BBgc%E6%89%AB%E6%8F%8F%E5%8E%8B%E5%8A%9B">减轻GC扫描压力</a></li>
    <li><a href="#%E6%80%BB%E7%BB%93">总结</a></li>
  </ul>
</blockquote>

## 创建slice时预分配内存

预分配内存是最常见的优化手段，我会分为创建时和使用中两部分来讲解如何进行优化。

提前为要创建的slice分配足够的内存，可以消除后续添加元素时扩容产生的性能损耗。

具体做法如下：

```golang
s1 := make([]T, 0, 预分配的元素个数)

// 另一种不太常见的预分配手段，此时元素个数必须是常量
var arr [元素个数]T
s2 := arr[:]
```

很简单的代码，性能测试我就不做了。

前面说到添加元素时扩容产生的性能损耗，这个损耗分为两方面，一是扩容需要重新计算slice的cap，尤其是1.19之后采用更缓和的分配策略后计算量是有所增加的，另一方面在于重新分配内存，如果没能原地扩容的话还需要重新分配一块内存把数据移动过去，再释放原先的内存，添加的元素越多遇到这种情况的概率越大，这是相当大的开销。

另外slice采用的扩容策略有时候会造成浪费，比如下面这样：

```golang
func main() {
    var a []int
    for i := 0; i < 2048; i++ {
            a = append(a, i)
    }
    fmt.Println(cap(a)) // go1.22: 2560
}
```

可以看到，我们添加了2048个元素，但go最后给我们分配了2560个元素的内存，浪费了将近500个。

不过预分配不是万金油，有限定了的适用场景：

适用场景：

1. 明确知道slice里会有多少个元素的场景
2. 元素的个数虽然不确定，但严格限制再`[x, y]`的区间内，这时候可以选择设置预分配大小为`y+1`，当然x和y之间的差不能太大，像1和1000这种很明显是不应该进行预分配的，主要的判断依据是最坏情况下的内存浪费率。

除了上面两种情况，我不建议使用预分配，因为分配内存本身是要付出性能的代价的，不是上面两种场景时预分配都会不可避免的产生大量浪费，这些浪费带来的性能代价很可能会超过扩容的代价。

预分配内存还有另一个好处：如果分配的大小是常量或者常量表达式，则有机会被逃逸分析认定为大小合适分配在栈上，从而使性能更进一步提升。这也是编译器实现的，具体的代码如下：

```golang
// https://github.com/golang/go/blob/master/src/cmd/compile/internal/walk/builtin.go#L412

// walkMakeSlice walks an OMAKESLICE node.
func walkMakeSlice(n *ir.MakeExpr, init *ir.Nodes) ir.Node {
	l := n.Len
	r := n.Cap
	if r == nil {
		r = safeExpr(l, init)
		l = r
	}
	t := n.Type()
	if t.Elem().NotInHeap() {
		base.Errorf("%v can't be allocated in Go; it is incomplete (or unallocatable)", t.Elem())
	}
	if n.Esc() == ir.EscNone {
		if why := escape.HeapAllocReason(n); why != "" {
			base.Fatalf("%v has EscNone, but %v", n, why)
		}
		// 检查i是否是常量
		i := typecheck.IndexConst(r)
		if i < 0 {
			base.Fatalf("walkExpr: invalid index %v", r)
		}

		// 检查通过后创建slice临时变量，分配在栈上
	}

	// 逃逸了，这时候会生成调用runtime.makeslice的代码
    // runtime.makeslice用mallocgc从堆分配内存
}
```

栈上分配内存速度更快，而且对gc的压力也更小一些，但对象会在哪被分配并不是我们能控制的，我们能做的也只有创造让对象分配在栈上的机会仅此而已。

## 操作slice前预分配内存

从slices包进入标准库开始，操作现有的slice时也能预分配内存了。

当然之前也可以，不过得绕些弯路，有兴趣可以去看下`slices.Grow`是怎么做的。

通过简单的测试来看看效果：

```golang
func BenchmarkAppend(b *testing.B) {
	for i := 0; i < b.N; i++ {
		s := []int{1, 2, 3, 4, 5}
		for j := 0; j < 1024; j++ {
			s = append(s, j)
		}
	}
}

func BenchmarkAppendWithGrow(b *testing.B) {
	for i := 0; i < b.N; i++ {
		s := []int{1, 2, 3, 4, 5}
		s = slices.Grow(s, 1024)
		for j := 0; j < 1024; j++ {
			s = append(s, j)
		}
	}
}
```

这是结果，用benchstat进行了比较：

```text
goos: windows
goarch: amd64
cpu: Intel(R) Core(TM) i5-10200H CPU @ 2.40GHz
         │   old.txt   │               new.txt               │
         │   sec/op    │   sec/op     vs base                │
Append-8   4.149µ ± 3%   1.922µ ± 5%  -53.69% (p=0.000 n=10)

         │    old.txt    │               new.txt                │
         │     B/op      │     B/op      vs base                │
Append-8   19.547Ki ± 0%   9.250Ki ± 0%  -52.68% (p=0.000 n=10)

         │  old.txt   │              new.txt               │
         │ allocs/op  │ allocs/op   vs base                │
Append-8   8.000 ± 0%   1.000 ± 0%  -87.50% (p=0.000 n=10)
```

不仅速度快了一倍，内存也节约了50%，而且相比未用Grow的代码，优化过后的代码只需要一次内存分配。

性能提升的原因和上一节的完全一样：避免了多次扩容带来的开销。

同时节约内存的好处也和上一节一样是存在的：

```golang
func main() {
	s1 := make([]int, 10, 50) // 注意已经有一定的预分配了
	for i := 0; i < 1024; i++ {
		s1 = append(s1, i)
	}
	fmt.Println(cap(s1))  // 1280

	s2 := make([]int, 10, 50)
	s2 = slices.Grow(s3, 1024)
	for i := 0; i < 1024; i++ {
		s2 = append(s2, i)
	}
	fmt.Println(cap(s2))  // 1184
}
```

如例子所示，前者的内存利用率是80%，而后者是86.5%，**Grow虽然也是利用append的机制来扩容，但它可以更充分得利用内存，避免了浪费**。

也和上一节一样，使用前的预分配的适用场景也只有两个：

1. 明确知道会往slice里追加多少个元素的场景
2. 追加的元素的个数虽然不确定，但严格限制再`[x, y]`的区间内，这时候可以选择设置预分配大小为`y+1`。

另外如果是拼接多个slice，最好使用`slices.Concat`，因为它内部会用Grow预分配足够的内存，比直接用append快一些。这也算本节所述优化手段的一个活得例子。

## slice表达式中合理设置cap值

<https://github.com/golang/go/pull/64835>

在比较新的go版本里slice表达式是可以有第三个参数的，即cap的值，形式类似：`slice[start:end:capEnd]`。

注意我用了`capEnd`而不是cap，因为这个参数不是cap的长度，而是指新的slice最大可以访问到原数组或者slice的（索引-1）的元素。举个例子：`slice[1:2:3]`，这个表达式创建了一个新的切片，长度为`2-1`即1，可以访问到原切片的索引`3-1`即2的元素，因此新切片可以访问的元素实际上有`index 1`和`index 2`两个，cap为2。

为啥要加这个参数呢？因为可以限制切片访问的范围，避免意外地改变数据。

当然那么没有第三个参数的时候cap是怎么处理的呢？当然是相当于`cap(old slice) - start`了。

这和性能优化有什么关系呢？看个例子：

```golang
func noop(s []int) int {
	return s[1] + s[2]
}

func BenchmarkSlice(b *testing.B) {
	slice := []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	for i := 0; i < b.N; i++ {
		noop(slice[1:5])
	}
}

func BenchmarkSliceWithEqualCap(b *testing.B) {
	slice := []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	for i := 0; i < b.N; i++ {
		noop(slice[1:5:5])
	}
}
```

测试结果：

```text
goos: windows
goarch: amd64
cpu: Intel(R) Core(TM) i5-10200H CPU @ 2.40GHz
BenchmarkSlice-8                1000000000               0.3263 ns/op          0 B/op          0 allocs/op
BenchmarkSliceWithEqualCap-8    1000000000               0.3015 ns/op          0 B/op          0 allocs/op
```

如果用benchstat进行比较，平均来说使用`slice[1:5:5]`的代码要快3%左右。

事实上这里有一个go的小优化，当切片表达式里第二个参数和第三个参数一样的时候，cap可以不用额外计算，直接取之前算出来的length就行了。这会少几次内存访问和一个减法运算。

不信可以看看编译器的[代码](https://github.com/golang/go/blob/master/src/cmd/compile/internal/ssagen/ssa.go#L6083)：

```golang
// slice computes the slice v[i:j:k] and returns ptr, len, and cap of result.
// i,j,k may be nil, in which case they are set to their default value.
// v may be a slice, string or pointer to an array.
func (s *state) slice(v, i, j, k *ssa.Value, bounded bool) (p, l, c *ssa.Value) {
	t := v.Type
	var ptr, len, cap *ssa.Value
	switch {
	case t.IsSlice():
		ptr = s.newValue1(ssa.OpSlicePtr, types.NewPtr(t.Elem()), v)
        // 计算slice的len和cap
		len = s.newValue1(ssa.OpSliceLen, types.Types[types.TINT], v)
		cap = s.newValue1(ssa.OpSliceCap, types.Types[types.TINT], v)
	case t.IsString():
		// 省略，这里不重要
	case t.IsPtr():
		// 同上省略
	default:
		s.Fatalf("bad type in slice %v\n", t)
	}

	// 如果是s[:j:k]，i会默认设置为0
	if i == nil {
		i = s.constInt(types.Types[types.TINT], 0)
	}
    // 如果是s[i:]，则j设置为len(s)
	if j == nil {
		j = len
	}
	three := true
    // 如果是s[i:j:], 则k设置为cap(s)
	if k == nil {
		three = false
		k = cap
	}

	// 对i，j和k进行边界检查

	// 先理解成加减乘除的运算符就行
	subOp := s.ssaOp(ir.OSUB, types.Types[types.TINT])
	mulOp := s.ssaOp(ir.OMUL, types.Types[types.TINT])
	andOp := s.ssaOp(ir.OAND, types.Types[types.TINT])

	// Calculate the length (rlen) and capacity (rcap) of the new slice.
	// For strings the capacity of the result is unimportant. However,
	// we use rcap to test if we've generated a zero-length slice.
	// Use length of strings for that.
	rlen := s.newValue2(subOp, types.Types[types.TINT], j, i)
	rcap := rlen
	if j != k && !t.IsString() {
		rcap = s.newValue2(subOp, types.Types[types.TINT], k, i)
	}

	// 计算slice的内存从那里开始的，在这不重要忽略

	return rptr, rlen, rcap
}
```

整体没什么难的，所有切片表达式最终都会走到这个函数，这个函数会生产相应的opcode，这个opcode会过一次相对简单的优化，然后编译器根据这些的opcode生成真正的可以运行的程序。

重点在于`if j != k && !t.IsString()`这句，分支里那句`rcap = s.newValue2(subOp, types.Types[types.TINT], k, i)`翻译成普通的go代码的话相当于`rcap = k - i`，k的值怎么计算的在前面的注释里有写。这意味着切片表达式的二三两个参数如果值一样且不是string，那么会直接复用length而不需要额外的计算了。题外话，这里虽然我用了“计算”这个词，但实际是rcap和rlen还都只是表达式，真正的结果是要在程序运行的时候才能计算得到的，有兴趣的话可以自己研究一下go的编译器。

正是因为这个小小的优化带来了细微的性能提升。

当然，这些只是代码生成中的细节，只有这个原因的话我通常不会推荐这样的做法。

所以更重要的是在于前面提到的安全性：限制切片访问的范围，避免意外地改变数据。在此基础上不仅不会有性能下降还有小幅的上升，算是锦上添花。

适用场景：当切片的cap和length理论上长度应该相等时，最好都明确地进行设置，比如：`slice[i : j+2 : j+2]`这样。

上面这个场景估计能占到一半左右，当然还有很多不符合上述要求的场景，所以不要生搬硬套，一切以性能测试为准。

## 向slice添加多个零值元素的优化

往slice里添加“0”也有些小窍门，看看下面的测试：

```golang
func BenchmarkAppendZeros1(b *testing.B) {
	for i := 0; i < b.N; i++ {
		slice := []int{}
		slice = append(slice, []int{0, 0, 0, 0, 0}...)
	}
}

// 优化版本
func BenchmarkAppendZeros2(b *testing.B) {
	for i := 0; i < b.N; i++ {
		slice := []int{}
		slice = append(slice, make([]int, 5)...)
	}
}
```

测试结果：

```text
goos: windows
goarch: amd64
cpu: Intel(R) Core(TM) i5-10200H CPU @ 2.40GHz
              │   old.txt   │              new.txt               │
              │   sec/op    │   sec/op     vs base               │
AppendZeros-8   31.79n ± 2%   30.04n ± 2%  -5.50% (p=0.000 n=10)

              │  old.txt   │            new.txt             │
              │    B/op    │    B/op     vs base            │
AppendZeros-8   48.00 ± 0%   48.00 ± 0%  ~ (p=1.000 n=10) ¹
¹ all samples are equal

              │  old.txt   │            new.txt             │
              │ allocs/op  │ allocs/op   vs base            │
AppendZeros-8   1.000 ± 0%   1.000 ± 0%  ~ (p=1.000 n=10) ¹
¹ all samples are equal
```

一行代码，在内存用量没有变化的情况下性能提升了5%。

秘密依然在编译器里。

不管是`append(s1, s2...)`还是`append(s1, make([]T, length)...)`，编译器都有特殊的处理。

前者的流程是这样的：

1. 创建s2（如果s2是个slice的字面量的话）
2. 检查s1的cap，不够的情况下要扩容
3. 将s2的内容copy到s1里

使用make时的流程是这样的：

1. 检查s1的cap，不够的情况下要扩容
2. 对length长度的s1的空闲内存做memclr（将内存中的值全设置为0）

代码在这里：<https://github.com/golang/go/blob/master/src/cmd/compile/internal/walk/assign.go#L647>

性能提升的秘密在于：不用创建临时的slice，以及memclr做的事比copy更少也更简单所以更快。

而且显然`append(s1, make([]T, length)...)`的可读性也是更好的，可谓一举两得。

适用场景：需要往slice添加连续的零值的时候。

## 循环展开

用循环处理slice里的数据也是常见的需求，相比下一节会提到的for-range，普通循环访问数据的形式可以更加灵活，而且也不会受1.22改变range运行时行为的影响。

说到循环相关的优化，循环展开是绕不开的话题。顾名思义，就是把本来要迭代n次的循环，改成每轮迭代里处理比原先多m倍的数据，这样总的迭代次数会降为`n/m + 1`次。

这样为啥会更快呢？其中一点是可以少很多次循环跳转和边界条件的更新及比较。另一点是现代 CPU 都有一个叫做指令流水线的东西，它可以同时运行多条指令，如果它们之间没有数据依赖（后一项数据依赖前一项作为输入）的话，展开循环后意味着有机会让一部分指令并行从而提高吞吐量。

然鹅通常这不是程序员该关心的事，因为怎么展开循环，什么时候应该展开什么时候不应（循环展开后会影响到当前函数能否被内联等）都是一个有着良好的优化过程的编译器该做的。

你问go呢？那是自然没有的。在运行时性能和语言表现力之间，go选择了编译速度。编译得确实快，然而优化上就要眼前一黑了。

所以只能自己写了：

```golang
func loop(s []int) int {
	sum := 0
	for i := 0; i < len(s); i++ {
		sum += s[i]
	}
	return sum
}

func unroll4(s []int) int {
	sum := 0
	for i := 0; i < len(s); i += 4 {
		sum += s[i]
		sum += s[i+1]
		sum += s[i+2]
		sum += s[i+3]
	}
	return sum
}

func BenchmarkLoop(b *testing.B) {
	s := []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 35, 26, 27, 28, 29, 30, 31}
	for i := 0; i < b.N; i++ {
		loop(s)
	}
}

func BenchmarkUnroll4(b *testing.B) {
	s := []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 35, 26, 27, 28, 29, 30, 31}
	for i := 0; i < b.N; i++ {
		unroll4(s)
	}
}

func BenchmarkUnroll8(b *testing.B) {
	s := []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 35, 26, 27, 28, 29, 30, 31}
	for i := 0; i < b.N; i++ {
		unroll8(s)
	}
}
```

测试使用32个int的slice，首先和一个循环里处理四个数据的对比：

```text
goos: windows
goarch: amd64
cpu: Intel(R) Core(TM) i5-10200H CPU @ 2.40GHz
         │   old.txt   │               new.txt               │
         │   sec/op    │   sec/op     vs base                │
Unroll-8   9.718n ± 3%   3.196n ± 2%  -67.11% (p=0.000 n=10)

         │  old.txt   │            new.txt             │
         │    B/op    │    B/op     vs base            │
Unroll-8   0.000 ± 0%   0.000 ± 0%  ~ (p=1.000 n=10) ¹
¹ all samples are equal

         │  old.txt   │            new.txt             │
         │ allocs/op  │ allocs/op   vs base            │
Unroll-8   0.000 ± 0%   0.000 ± 0%  ~ (p=1.000 n=10) ¹
¹ all samples are equal
```

提升了将近67%，相当之大了。然后我们和一次处理8个数据的比比看：

```text
goos: windows
goarch: amd64
cpu: Intel(R) Core(TM) i5-10200H CPU @ 2.40GHz
         │   old.txt   │               new.txt               │
         │   sec/op    │   sec/op     vs base                │
Unroll-8   9.718n ± 3%   2.104n ± 1%  -78.34% (p=0.000 n=10)
```

这次提升了78%，相比一次只处理四个，处理8个的方法快了30%。

我这为了方便只处理了总数据量是每轮迭代处理数据数量整数倍的情况，非整数倍的时候需要借助“达夫设备”，在go里实现起来比较麻烦，所以偷个懒。不过鉴于循环展开带来的提升非常之大，如果确定循环处理slice的代码是性能瓶颈，不妨可以实现一下试试效果。

适用场景：slice的长度需要维持在固定值上，且长度需要时每轮迭代处理数据量的整数倍。

需要仔细性能测试的场景：如果单次循环需要处理的内容很多代码很长，那么展开的效果很可能是没有那么好的甚至起反效果，因为过多的代码会影响当前函数和当前代码调用的函数是否被内联以及局部变量的逃逸分析，前者会使函数调用的开销被放大同时干扰分支预测和流水线执行导致性能下降，后者则会导致不必要的逃逸同时降低性能和增加堆内存用量。

另外每次迭代处理多少个元素也没必要拘泥于4或者2的倍数什么的，理论上不管一次处理几个都会有显著的性能提升，实际测试也是如此，一次性处理3、5或者7个的效果和4或者8个时差不多，总体来说一次处理的越多提升越明显。但如果展开的太过火就会发展成为上面说的需要严格测试的场景了。所以我建议展开处理的数量最好别超过8个。

## 避免for-ranges复制数据带来的损耗

普通的循环结构提供了灵活的访问方式，但要是遍历slice的话我想大部分人的首选应该是for-ranges结构吧。

这一节要说的东西与其叫性能优化，到不如说应该是“如何避开for-ranges”的性能陷阱才对。

先说说陷阱在哪。

陷阱其实有两个，一个基本能避开，另一个得看情况才行。我们先从能完全避开的开始。

### 避免复制

第一个坑在于range遍历slice的时候，会把待遍历的数据复制一份到循环变量里，而且从1.22开始range的循环遍历每次迭代都会创建出一个新的实例，如果没注意到这点的话不仅性能下降还会使内存压力急剧升高。我们要做的就是避免不必要的复制带来的开销。

作为例子，我们用包含8个`int64`和1个string的结构体填充slice然后对比复制和不复制时的性能：

```golang
type Data struct {
	a, b, c, d, e, f, g, h int64
	text                   string
}

func generateData(n int) []Data {
	ret := make([]Data, 0, n)
	for i := range int64(n) {
		ret = append(ret, Data{
			a:    i,
			b:    i + 1,
			c:    i + 2,
			d:    i + 3,
			e:    i + 4,
			f:    i + 5,
			g:    i + 6,
			h:    i + 7,
			text: "测试",
		})
	}
	return ret
}

// 会导致额外复制数据的例子
func BenchmarkRanges1(b *testing.B) {
	data := generateData(100)
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		tmp := int64(0)
		for _, v := range data { // 数据被复制给循环变量v
			tmp -= v.a - v.h
		}
	}
}

// 避免了复制的例子
func BenchmarkRanges2(b *testing.B) {
	data := generateData(100)
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		tmp := int64(0)
		for i := range data { // 注意这两行
			v := &data[i]
			tmp -= v.a - v.h
		}
	}
}
```

结果：

```text
goos: windows
goarch: amd64
cpu: Intel(R) Core(TM) i5-10200H CPU @ 2.40GHz
         │   old.txt   │              new.txt               │
         │   sec/op    │   sec/op     vs base               │
Ranges-8   33.51n ± 2%   32.63n ± 1%  -2.41% (p=0.000 n=10)
```

使用指针或者直接通过索引访问可以避免复制，如结果所示，结构体越大性能的差异就越明显。此外新版本的go修改了range的语义，从以前会复用循环变量变成了每轮循环都创建新的循环变量，这会使一部分存在复制开销的for-range循环变得更慢。

适用场景：需要遍历每个元素，遍历的slice里的单项数据比较大且明确不需要遍历的数据被额外复制给循环变量的时候。

### 遍历字符串的时候避免转换带来的开销

字符串可能有点偏题了，但我们要说的这点也勉强和slice有关。

这个坑在于，range遍历字符串的时候会把字符串的内容转换成一个个rune，这一步会带来开销，尤其是字符串里只有ascii字符的时候。

写个简单例子看看性能损耗有多少：

```golang
func checkByte(s string) bool {
	for _, b := range []byte(s) {
		if b == '\n' {
			return true
		}
	}
	return false
}

func checkRune(s string) bool {
	for _, r := range s {
		if r == '\n' {
			return true
		}
	}
	return false
}

func BenchmarkRanges1(b *testing.B) {
	s := "abcdefghijklmnopqrstuvwxyz1234567890."
	for i := 0; i < b.N; i++ {
		checkRune(s)
	}
}

func BenchmarkRanges2(b *testing.B) {
	s := "abcdefghijklmnopqrstuvwxyz1234567890."
	for i := 0; i < b.N; i++ {
		checkByte(s)
	}
}
```

这是结果：

```text
goos: windows
goarch: amd64
cpu: Intel(R) Core(TM) i5-10200H CPU @ 2.40GHz
         │   old.txt   │               new.txt               │
         │   sec/op    │   sec/op     vs base                │
Ranges-8   36.07n ± 2%   23.95n ± 1%  -33.61% (p=0.000 n=10)
```

**把string转换成`[]byte`再遍历的性能居然提升了1/3**。换句话说如果你没注意到这个坑，那么就要白白丢失这么多性能了。

而且将string转换成`[]byte`是不需要额外分配新的内存的，可以直接复用string内部的数据，当然前提是不会修改转换后的slice，在这里我们把这个slice直接交给了range，它不会修改slice，所以转换的开销被省去了。

这个优化是从1.6开始的，有兴趣可以看看编译器的代码：<https://github.com/golang/go/blob/master/src/cmd/compile/internal/walk/convert.go#L316> （看代码其实还有别的针对这种转换的优化，比如字符串比较短的时候转换出来的`[]byte`会分配在栈上）

当然，如果你要处理ASCII以外的字符，比如中文汉字，那么这个优化就行不通了。

适用场景：需要遍历处理的字符串里的字符都在ASCII编码的范围内，比如只有换行符英文半角数字和半角标点的字符串。

## BCE边界检查消除

边界检查是指在访问slice元素、使用slice表达式、make创建slice等场景下检查参数的值是否超过最大限制以及是否会越界访问内存。这些检查是编译器根据编译时获得的信息添加到对应位置上的，检查的代码会在运行时被运行。

这个特性对于程序的安全非常重要。

那么是否只要是有上述表达式的地方就会导致边界检查呢？答案是不，因为边界检查需要取slice的长度或者cap然后进行比较，检查失败的时候会panic，整个造成有些花时间而且对分支预测不是很友好，总体上每个访问slice元素的表达式都添加检查会拖垮性能。

因此边界检查消除就顺理成章出现了——一些场景下明显index不可能有越界问题，那么检查就是完全不必要的。

如何查看编译器在哪里插入了检查呢？可以用下面这个命令：`go build -gcflags='-d=ssa/check_bce' main.go`。

以上一节的`unroll4`为例子：

```shell
$ go build -gcflags='-d=ssa/check_bce' main.go

# command-line-arguments
./main.go:8:11: Found IsInBounds
./main.go:9:11: Found IsInBounds
./main.go:10:11: Found IsInBounds
./main.go:11:11: Found IsInBounds
```

目前你会看到两种输出`IsInBounds`和`IsSliceInBounds`。两者都是插入边界检测的证明，检查的内容差不多，只有微小的差别，有兴趣可以看ssa怎么生成两者代码的：<https://github.com/golang/go/blob/master/src/cmd/compile/internal/ssa/rewriteAMD64.go#L25798>

那么这些检查怎么消除呢？具体来说可以分为好几种情况，但随着编译器的发展肯定会有不少变化，所以我不准备一一列举。

既然不列举，那肯定有大致通用的规则：如果使用index访问slice前的表达式里可以推算出当前index值不会越界，那么检查就能消除。

举几个例子：

```golang
s1 := make([]T, 10)
s1[9] // 常数索引值编译时就能判断是否越界，所以不需要插入运行时的检测。
_ = s1[i&6]   // 索引的值肯定在0-6之间，检查被消除

var s2 []int
_ = s2[:i] // 检查
_ = s2[:i] // 重复访问，消除边界检查
_ = s2[:i+1] // 检查
_ = s2[:i+1] // 重复的表达式，检查过了所以检查被消除

func f(s []int) int {
    if len(s) < 3 {
        panic("error")
    }

    return s[1] + s[2] // 前面的if保证了这两个访问一定不会越界，所以检查可以消除
}

// 一种通过临时变量避免多次边界检测的常用作法
func f2(s []int) int {
    tmp := s[:4:4] // 这里会边界检查。这里还利用了前面说的合理设置slice表达式的cap避免额外开销
    a := tmp[2] // tmp那里的检查保证了这里不会越界，因此不会再检查
    b := tmp[3] // 同上
    return a+b
}
```

我没列出所有例子，想看的可以去[这里](https://go101.org/article/bounds-check-elimination.html)。

当然有一些隐藏的不能消除检查的场景：

```golang
func f(s []int, i int) {
    if i < len(s) {
        fmt.Println(s[i]) // 消除不了，因为i是有符号整数，可能会小于0
    }
}

func f(s []int, i int) {
    if 0 < i && i < len(s) {
        fmt.Println(s[i+2]) // 消除不了，因为i是有符号整数，i+2万一发生溢出，索引值会因为绕回而变成负数
    }
}
```

有了这些知识，前面的`unroll4`有四次边界检查，实际上用不着这么多，因此可以改成下面这样：

```golang
func unroll4(s []int) int {
	sum := 0
	for i := 0; i < len(s); i += 4 {
		tmp := s[i : i+4 : i+4] // 只有这里会检查一次
		sum += tmp[0]
		sum += tmp[1]
		sum += tmp[2]
		sum += tmp[3]
	}
	return sum
}
```

这么做实际上还是会检查一次，能不能完全消除呢？

```golang
func unroll4(s []int) int {
	sum := 0
	for len(s) >= 4 {
		sum += s[0]
		sum += s[1]
		sum += s[2]
		sum += s[3]
        s = s[4:] // 忽略掉已经处理过的四个元素，而且因为len(s) >= 4，所以这里也不需要检查
	}
	return sum
}
```

这样检查就完全消除了，但多了一次slice的赋值。

然而我这的例子实在是太简单了，性能测试显示边界检查消除并没有带来性能提升，完全消除了检查的那个例子反而因为额外的slice赋值操作带来了轻微的性能下降（和消除到只剩一次检查的比较）。

如果想要看效果更明显的例子，可以参考[这篇博客](https://sourcegraph.com/blog/slow-to-simd)。

适用场景：能有效利用`len(slice)`的结果的地方可以尝试BCE。

其他场合需要通过性能测试来判断是否有提升以及提升的幅度。像这样既不像设置slice表达式cap值那样增强安全性又不像用make批量添加空值那样增加可读性的改动，个人认为除非真的是性能瓶颈而且没有其他优化手段，**否则提升低于5%的话建议不要做这类改动**。

## 并行处理slice

前面说到了循环展开，基于这一手段更进一步的优化就是并行处理了。这里的并行不是指SIMD，而是依赖goroutine实现的并行。

能并行的前提是slice元素的处理不会互相依赖，比如`s[1]`的处理依赖于`s[0]`的处理结果这样的。

在能确定slice的处理可以并行后，就可以写一些并行代码了，比如并行求和：

```golang
func Sum(s []int64) int64 {
	// 假设s的长度是4000
	var sum atomic.Int64
	var wg sync.WaitGroup
	// 每个goroutine处理800个
	for i := 0; i < len(s); i += 800 {
		wg.Add(1)
		go func(ss []int) {
			defer wg.Done()
			var ret int64
			for j := range ss {
				ret += ss[j]
			}
			sum.Add(ret)
		}(s[i: i+800])
	}
	wg.Wait()
	return sum.Load()
}
```

很简单的代码。和循环展开一样，需要额外料理数量不够一次处理的剩余的元素。

另外协程的创建销毁以及数据的同步都是比较耗时的，如果slice里元素很少的话并行处理反而得不偿失。

适用场景：slice里元素很多、对元素的处理可以并行互不干扰，还有重要的一点，golang程序可以使用超过一个cpu核心保证代码真正可以“并行”运行。

## 复用

复用slice是个常见的套路，其中复用`[]byte`是最为常见的。

复用可以利用`sync.Pool`，也可以像下面这样：

```golang
buf := make([]byte, 1024)
for {
	read(buf)
	...
	// reuse
	buf = buf[:0]
}
```

其中`buf = buf[:0]`使得slice的cap不变，length清零，这样就可以复用slice的内存了。使用`sync.Pool`时也需要这样使slice的长度为零。

此外使用`sync.Pool`时还要注意slice的尺寸不能太大，否则同样会增加gc负担。一般来说超过1M大小的slice是不建议存进去的，当然还得结合项目需求和性能测试才能决定尺寸的上限。

适用场景：你的slice内存可以反复被使用（最好是能直接重用连清理都可以不做的那种，清理会让优化效果打点折扣）并且多次创建slice确实成为了性能瓶颈时。

## 高效删除多个元素

删除元素也是常见需求，这里我们也要分三种情况来讨论。

这三种情况都包含在标准库的`slices.Delete`里了，所以比起自己写我更推荐你用标准库。因此本节没有适用场景这一环境，但每一小节针对一些特殊场景给出了相应的建议。

### 删除所有元素

如果删除元素后也不打算复用slice了，直接设置为nil就行。

如果还要复用内存，利用我们在[复用](#复用)那节里提到的`s := s[:0]`就行，不过光这样还不够，为了防止内存泄漏还得把删除的元素全部清零，在1.21前我们只能这么做：

```golang
func deleteSlice[T any, S ~[]T](s S) S {
	var zero T
	for i := range s {
		s[i] = zero
	}
	return s[:0]
}
```

1.21之后我们有了clear内置函数，代码可以大幅简化：

```golang
func deleteSlice[T any, S ~[]T](s S) S {
	clear(s)
	return s[:0]
}
```

两种写法的性能是一样的，因为go专门对for-range循环写入零值做了优化，效果和直接用clear一样：

```golang
func BenchmarkClear(b *testing.B) {
	for i := 0; i < b.N; i++ {
		var a = [...]uint64{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1}
		clear(a[:])
	}
}

func BenchmarkForRange(b *testing.B) {
	for i := 0; i < b.N; i++ {
		var a = [...]uint64{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1}
		for j := range a {
			a[j] = 0
		}
	}
}
```

```text
goos: windows
goarch: amd64
cpu: Intel(R) Core(TM) i5-10200H CPU @ 2.40GHz
BenchmarkClear-8      	1000000000	         0.2588 ns/op	       0 B/op	       0 allocs/op
BenchmarkForRange-8   	1000000000	         0.2608 ns/op	       0 B/op	       0 allocs/op
```

但是如果循环的形式不是for-range，那么就吃不到这个优化了：

```golang
func BenchmarkClear(b *testing.B) {
	for i := 0; i < b.N; i++ {
		var a = [...]uint64{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1}
		clear(a[:])
	}
}

func BenchmarkForLoop(b *testing.B) {
	for i := 0; i < b.N; i++ {
		var a = [...]uint64{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1}
		for j := 0; j < 20; j++ {
			a[j] = 0
		}
	}
}
```

```text
goos: windows
goarch: amd64
cpu: Intel(R) Core(TM) i5-10200H CPU @ 2.40GHz
BenchmarkClear-8     	1000000000	         0.2613 ns/op	       0 B/op	       0 allocs/op
BenchmarkForLoop-8   	173418799	         7.088 ns/op	       0 B/op	       0 allocs/op
```

速度相差一个数量级。对“循环写零”优化有兴趣的可以在这看到是这么实现的：[arrayclear](https://github.com/golang/go/blob/master/src/cmd/compile/internal/walk/range.go#L532)。这个优化对map也有效果。

我们可以简单对比下置空为nil和clear的性能：

```golang
func BenchmarkDeleteWithClear(b *testing.B) {
	for i := 0; i < b.N; i++ {
		a := []uint64{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1}
		clear(a)
		a = a[:0]
	}
}

func BenchmarkDeleteWithSetNil(b *testing.B) {
	for i := 0; i < b.N; i++ {
		a := []uint64{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1}
		a = a[:] // 防止编译器认为a没有被使用
		a = nil
	}
}
```

从结果来看只是删除操作的话没有太大区别：

```text
BenchmarkDeleteWithClear-8      1000000000               0.2592 ns/op          0 B/op          0 allocs/op
BenchmarkDeleteWithSetNil-8     1000000000               0.2631 ns/op          0 B/op          0 allocs/op
```

所以选用哪种方式主要取决于你后续是否还要复用slice的内存，需要复用就用clear，否则直接设为nil。

### 删除头部或尾部的元素

删除尾部元素是最简单的，最快的方法只有`s = s[:index]`这一种。注意别忘了要用`clear`清零被删除的部分。

这个方法唯一的缺点是被删除的部分的内存不会释放，通常这没有坏处而且能在新添加元素时复用这些内存，但如果你不会再复用这些内存并且对浪费很敏感，那只能分配一个新slice然后把要留下的元素复制过去了，但要注意这么做的话会慢很多而且在删除的过程中要消费更多内存（因为新旧两个slice得同时存在）。

删除头部元素的选择就比较多了，常见的有两种（我们需要保持元素之间的相对顺序）：`s = s[index+1:]`或者`s = append(s[:0], s[index+1:]...)`。

前者是新建一个slice，底层数组起始为止指向原先slice的`index+1`处，注意虽然底层数组被复用了，但cap实际上是减小的，而且被删除部分的内存没有机会再被复用了。这种方法需要在删除前先把元素清零。

后一种则不会创建新的slice，它把`index+1`开始的元素平移到了slice的头部，这样也是删除了头部的元素（被覆盖掉了）。使用这种方案不需要主动清零元素，你要是不放心移动后尾部剩下的空间也可以选择使用clear但一般不建议。

理论上前者真正地浪费了内存但性能更好，不过性能始终要用benchmark来证明：

```golang
func BenchmarkClearWithReSlice(b *testing.B) {
	for i := 0; i < b.N; i++ {
		a := []uint64{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1}
		// 删除头部7个元素
		clear(a[:7])
		a = a[7:]
	}
}

func BenchmarkClearWithAppend(b *testing.B) {
	for i := 0; i < b.N; i++ {
		a := []uint64{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1}
		a = append(a[:0], a[7:]...)
	}
}
```

测试结果显示确实第一种方法快：

```text
BenchmarkClearWithReSlice-8     1000000000               0.2636 ns/op          0 B/op          0 allocs/op
BenchmarkClearWithAppend-8      100000000               10.82 ns/op            0 B/op          0 allocs/op
```

Append慢了一个数量级，即使memmove已经得到了相当多的优化，在内存里移动数据还是很慢的。

在实际应用中应该根据内存利用效率和运行速度综合考虑选择合适的方案。

### 删除在中间位置的元素

删除中间部分的元素还要保持相对顺序，能用的办法就只有移动删除部分后面的元素到前面进行覆盖这一种办法：

```golang
s := append(s[:index], s[index+n:]...)
```

这个方法也不需要主动clear被删除元素，因为它们都被覆盖掉了。利用append而不是for循环除了前面说的for循环优化差之外还有代码更简洁和能利用memmove这两个优势。

因为方法唯一没啥参照物，所以性能就不测试了。

## 减轻GC扫描压力

简单的说，尽量不要在slice里存放大量的指针或者包含指针的结构体。指针越多gc在扫描对象时需要做的工作就越多，最后会导致性能下降。

更具体的解释和性能测试可以看[这篇](./gc的内部优化.md#优化带来的提升)。

适用场景：无特殊需求且元素大小不是特别大的，存值优于存指针。

作为代价，如果选择了存值，得小心额外的复制导致的开销。

## 总结

按个人经验来看，使用频率最高的几个优化手段依次是预分配内存、避免for-ranges踩坑、slice复用、循环展开。从提升来看这几个也是效果最明显的。

编译器的优化不够给力的话就只能自己想办法用这些优化技巧了。

有时候也可以利用逃逸分析规则来做优化，但正如[这篇文章](./为什么不应该过度关注逃逸分析.md)所说，绝大多数情况下你都不应该考虑逃逸分析。

还有另外一条路：给go编译器共享代码提升编译产物的性能。虽然阻力会很大，但我还是相信有大佬一定能做到的。这也是我为什么会把编译器怎么做优化的代码贴出来，抛砖引玉嘛。

还有最重要的一点：性能问题不管是定位还是优化，**都必须以性能测试为依据**，切记不可光靠“经验”和没有事实依据支撑的“推论”。

最后我希望这篇文章能成为大家优化性能时的趁手工具，而不是面试时背的八股文。
