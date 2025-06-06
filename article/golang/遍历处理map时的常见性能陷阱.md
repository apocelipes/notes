最近一直在重构优化老系统，所以性能优化相关的文章会比较多。

这次的是有关循环处理map时的性能优化。预分配内存之类的大家都知道的就不多说了，今天来讲点大伙不知道的。

要讲的一共有三点，而且都和循环处理map有关。

## 不要用for-range循环清空map

这里要讨论的“清空”是指删除map中所有键值对，但保留map里已分配的内存供下次复用。

如果只是想释放map并且不再需要复用，那么`map1 = nil`或者`map2 = map[T]U{}`就足够了。

内置函数`delete`可以帮我们实现删除键值对但保留它们在map中的内存空间，通常我们会这么写：

```golang
for key := range Map1 {
    delete(Map1, key)
}
```

这种模式化的代码太常见，以至于go编译器专门对其做了优化，只要形式上符合上述代码片段的，编译器都会把循环优化成`runtime.mapclear(Map1)`，使用runtime内置的map清理函数将map清空，这比靠循环遍历删除要快很多倍。

看到这里你可能会说这不是挺好吗，为什么不让用了呢？

因为现在有更好的替代方案了——内置函数`clear`。

`clear`应用在map上时本身就会调用`runtime.mapclear(...)`，在性能上和循环方法大致一样而且只快不慢。因为两者最终生成代码差不多，性能测试也就没多大意义了，所以这里不做性能测试。

`clear`还有另一个好处，它更容易让包含它的函数被内联。

这是什么意思呢？go的编译器实际上在编译时要分很多个步骤，粗略地讲go代码在真正开始生成机器码之前，得先经过`内联 -> 逃逸分析 -> 语法树改写`这样几个阶段。上文说的for循环删除优化在`语法树改写`这个阶段完成。

这就带来了一个问题，相比一个简单的`clear`函数调用，编译器认为for循环这种操作更“重量级”，一个函数拥有的“重”操作越多，那么这个函数被内联优化的可能性就越小。

我们看个例子：

```golang
func RangeClearMap() {
	for k := range bigMap {
		delete(bigMap, k)
	}
	for k := range bigPtrMap {
		delete(bigPtrMap, k)
	}
	for k := range smallMap {
		delete(smallMap, k)
	}
	for k := range smallPtrMap {
		delete(smallPtrMap, k)
	}
	for k := range bigMapIntKey {
		delete(bigMapIntKey, k)
	}
	for k := range bigPtrMapIntKey {
		delete(bigPtrMapIntKey, k)
	}
	for k := range smallMapIntKey {
		delete(smallMapIntKey, k)
	}
	for k := range smallPtrMapIntKey {
		delete(smallPtrMapIntKey, k)
	}
}

func BuiltinClearMap() {
	clear(bigMap)
	clear(bigPtrMap)
	clear(smallMap)
	clear(smallPtrMap)
	clear(bigMapIntKey)
	clear(bigPtrMapIntKey)
	clear(smallMapIntKey)
	clear(smallPtrMapIntKey)
}
```

同样是清空8个map，一个用循环，一个用内置函数。我们看下内联分析的结果：

```bash
$ go build -gcflags=-m=2 2>&1|grep -E '(Range|Builtin)ClearMap'
./main.go:8:6: can inline RangeClearMap with cost 64 as: func() { for loop; for loop; for loop; for loop; for loop; for loop; for loop; for loop }
./main.go:35:6: can inline BuiltinClearMap with cost 16 as: func() { clear(bigMap); clear(bigPtrMap); clear(smallMap); clear(smallPtrMap); clear(bigMapIntKey); clear(bigPtrMapIntKey); clear(smallMapIntKey); clear(smallPtrMapIntKey) }
```

其中`cost`就是衡量一个函数里的操作有多“重”的数值标准，超过一定的cost，函数就无法内联。可以看到，使用循环会比使用`clear`内置函数重整整四倍。虽然最后因为两个函数都很简单所以被内联展开，但碰上更复杂一点的函数，显然使用clear能有更多的冗余。

尽管编译器最终会把两者优化成一样的对runtime的map清理函数的调用，但对for循环的优化在内联处理之后，因此for不仅让代码更长，也更容易错失内联优化的机会，而失去内联优化进而会影响逃逸分析从而损失更多性能，你可以在我[以前的博客](./为什么不应该过度关注逃逸分析.md)里看到内联和逃逸分析对内联的影响。

`clear()`是go1.21添加的，因此只要你在用的go版本大于等于1.21，我推荐你尽量使用clear而不是for-range循环来清空map。

## 遍历访问map时的陷阱

遍历处理map中的元素也是常见操作，不过不像循环删除，编译器在这种代码上并没有什么优化。

最常见的写法是这样的：

```golang
for key, value := range Map1 {
    func1(key, value)
    func2(key, value)
}
```

这时候，一部分开发者会觉得每次循环都得复制一次value，尤其是1.22开始循环变量每轮都是新变量，这种操作是不是会很慢也很占用内存？毕竟在遍历slice的时候确实有这些问题，那么能不能采用优化slice遍历的相同方法来优化map遍历呢：

```golang
for key := range Map1 {
    func1(key, Map1[key])
    func2(key, Map1[key])
}
```

现在不用复制value了，性能应该获得提升了吧？真的是这样吗？

我说过很多次，性能优化不能靠想象，要靠benchmark来说话，所以我们来做个性能测试。

我们测试大value和小value在两种循环下的表现，另外还会额外测试一下map里存放指针的情况，测试用的value类型主要是下面两种：

```golang
// 128字节
type BigObject struct {
	n1, n2, n3, n4, n5, n6, n7, n8, n9, n10 int64
	s1, s2, s3                              string
}

func NewBigObjectPtr() *BigObject {
	return &BigObject{
		n1:  randNum(),
		n2:  randNum(),
		n3:  randNum(),
		n4:  randNum(),
		n5:  randNum(),
		n6:  randNum(),
		n7:  randNum(),
		n8:  randNum(),
		n9:  randNum(),
		n10: randNum(),
		s1:  genRandStr(),
		s2:  genRandStr(),
		s3:  genRandStr(),
	}
}

// 32字节
type SmallObject struct {
	n1, n2 int64
	s1     string
}

func NewSmallObject() SmallObject {
	return SmallObject{
		n1: randNum(),
		n2: randNum(),
		s1: genRandStr(),
	}
}

const table = "abcdefgh01234567"

// 生成固定16字符长度的随机字符串
func genRandStr() string {
	buf := make([]byte, 16)
	num := rand.Uint64()
	for i := range buf {
		buf[i] = table[num&0xf]
		num >>= 4
	}
	return string(buf)
}

func randNum() int64 {
	for {
		num := rand.Int64()
		if num != 0 {
			return num
		}
	}
}
```

我们一共分8种case进行测试：

```golang
var (
	bigMap            = map[string]BigObject{}
	bigPtrMap         = map[string]*BigObject{}
	smallMap          = map[string]SmallObject{}
	smallPtrMap       = map[string]*SmallObject{}
)

func init() {
	for i := range int64(100) {
		strKey := fmt.Sprintf("Key:%03d", i)
		bigMap[strKey] = NewBigObject()
		bigPtrMap[strKey] = NewBigObjectPtr()
		smallMap[strKey] = NewSmallObject()
		smallPtrMap[strKey] = NewSmallObjectPtr()
	}
}
```

每个map里都填充100个元素，元素的值随机生成，不过我限制了字符串都是等长的，这是为了结果的准确性。

测试也比较简单：

```golang
func BenchmarkBigObjectLoopCopy(b *testing.B) {
	for b.Loop() {
		for _, v := range bigMap {
			if v.n1 == 0 || v.n2 == 0 {
				panic("error")
			}
		}
	}
}

func BenchmarkBigObjectPtrLoopCopy(b *testing.B) {
	for b.Loop() {
		for _, v := range bigPtrMap {
			if v.n1 == 0 || v.n2 == 0 {
				panic("error")
			}
		}
	}
}

func BenchmarkBigObjectLoopKey(b *testing.B) {
	for b.Loop() {
		for k := range bigMap {
			if bigMap[k].n1 == 0 || bigMap[k].n2 == 0 {
				panic("error")
			}
		}
	}
}

func BenchmarkBigObjectPtrLoopKey(b *testing.B) {
	for b.Loop() {
		for k := range bigPtrMap {
			if bigPtrMap[k].n1 == 0 || bigPtrMap[k].n2 == 0 {
				panic("error")
			}
		}
	}
}

func BenchmarkSmallObjectLoopCopy(b *testing.B) {
	for b.Loop() {
		for _, v := range smallMap {
			if v.n1 == 0 || v.n2 == 0 {
				panic("error")
			}
		}
	}
}

func BenchmarkSmallObjectPtrLoopCopy(b *testing.B) {
	for b.Loop() {
		for _, v := range smallPtrMap {
			if v.n1 == 0 || v.n2 == 0 {
				panic("error")
			}
		}
	}
}

func BenchmarkSmallObjectLoopKey(b *testing.B) {
	for b.Loop() {
		for k := range smallMap {
			if smallMap[k].n1 == 0 || smallMap[k].n2 == 0 {
				panic("error")
			}
		}
	}
}

func BenchmarkSmallObjectPtrLoopKey(b *testing.B) {
	for b.Loop() {
		for k := range smallPtrMap {
			if smallPtrMap[k].n1 == 0 || smallPtrMap[k].n2 == 0 {
				panic("error")
			}
		}
	}
}
```

8种case就是大value的map和小value的分别用`k,v := range map`遍历和只用key遍历，看看性能差异。

```text
goos: windows
goarch: amd64
pkg: maptraps
cpu: Intel(R) Core(TM) i7-14650HX
BenchmarkBigObjectLoopCopy-24            1866486               648.0 ns/op             0 B/op          0 allocs/op
BenchmarkBigObjectPtrLoopCopy-24         2531185               474.5 ns/op             0 B/op          0 allocs/op
BenchmarkBigObjectLoopKey-24              779468              1487 ns/op               0 B/op          0 allocs/op
BenchmarkBigObjectPtrLoopKey-24           691897              1616 ns/op               0 B/op          0 allocs/op
BenchmarkSmallObjectLoopCopy-24          2355145               506.1 ns/op             0 B/op          0 allocs/op
BenchmarkSmallObjectPtrLoopCopy-24       2504696               479.6 ns/op             0 B/op          0 allocs/op
BenchmarkSmallObjectLoopKey-24            771307              1560 ns/op               0 B/op          0 allocs/op
BenchmarkSmallObjectPtrLoopKey-24         735915              1514 ns/op               0 B/op          0 allocs/op
```

结果很是让人诧异，自认为的不需要复制value所以访问更快的结论是错的，而且错的离谱：复制value的做法性能是只用key遍历的3倍！

为啥会这样呢？因为你掉进hashmap的陷阱里了，我就直接说原因了：

1. 以`for k,v := range m`遍历map时，键值对是被顺序访问的，这对缓存命中和cpu的模式预测更友好，性能会比用key进行hash的随机访问要好；
2. 使用`m[k]`访问时需要计算key的hash，这一步是比较耗费计算资源的，哪怕新版本换了swissmap之后也一样，而for-range循环不需要计算hash值。hash计算对性能的影响可以看我[这篇博客](../c++/性能优化陷阱之hash真的比strcmp快吗.md)讲解。

这两点就是hashmap的陷阱，本来在少量多次或者随机查找等模式下算不上太大的问题，但在循环遍历的情形下影响就会放大，最后导致出现3倍以上的性能差异。

所以当你要遍历处理每一个value的时候，最好用`for k,v := range m`或者`for _,v := range m`。

不过也有例外：如果你的value比较大，你需要遍历key，符合过滤条件的key的value才需要处理，这种时候那些无用的复制就会成为性能绊脚石了。不过还是老话，先做benchmark再谈优化。

## 复制map时的性能陷阱

最后一个性能陷阱埋在复制map时。

这里说的“复制”是浅复制，把键值对复制到一个新map里去，里面的指针或者slice都是浅拷贝的。

还是先上常见写法，实际上在1.21之前你也只能这么写：

```golang
m2 := make(map[T]U, len(m1))
for k, v := range m1 {
    m2[k] = v
}
```

编译器同样也不会对这种代码有特殊优化，循环会老老实实地执行。

到了1.21，我们可以用`maps.Clone`做一样的事情，而且这个标准库函数也会在底层调用runtime里的专用map复制函数，性能杠杠的。

我们准备一下性能测试，map样本沿用上一节里的：

```golang
func BenchmarkMapClone(b *testing.B) {
	b.Run("range-smallMap", func(b *testing.B) {
		for b.Loop() {
			m := make(map[string]SmallObject, len(smallMap))
			for k, v := range smallMap {
				m[k] = v
			}
		}
	})
	b.Run("range-smallPtrMap", func(b *testing.B) {
		for b.Loop() {
			m := make(map[string]*SmallObject, len(smallPtrMap))
			for k, v := range smallPtrMap {
				m[k] = v
			}
		}
	})
	b.Run("range-bigMap", func(b *testing.B) {
		for b.Loop() {
			m := make(map[string]BigObject, len(bigMap))
			for k, v := range bigMap {
				m[k] = v
			}
		}
	})
	b.Run("range-bigPtrMap", func(b *testing.B) {
		for b.Loop() {
			m := make(map[string]*BigObject, len(bigPtrMap))
			for k, v := range bigPtrMap {
				m[k] = v
			}
		}
	})
	b.Run("clone-smallMap", func(b *testing.B) {
		for b.Loop() {
			maps.Clone(smallMap)
		}
	})
	b.Run("clone-smallPtrMap", func(b *testing.B) {
		for b.Loop() {
			maps.Clone(smallPtrMap)
		}
	})
	b.Run("clone-bigMap", func(b *testing.B) {
		for b.Loop() {
			maps.Clone(bigMap)
		}
	})
	b.Run("clone-bigPtrMap", func(b *testing.B) {
		for b.Loop() {
			maps.Clone(bigPtrMap)
		}
	})
}
```

在这里如果你用的go版本是1.24，那么你会踩到第一个陷阱，对没错，我说了有三种陷阱，没说只有三个哦。

1.24的`maps.clone`实现有问题，会有严重的性能回退，所以你可以看到它和循环复制性能没有差距，甚至有时候还更慢一点：

```text
enchmarkMapClone/range-smallMap-24         	  466128	      2505 ns/op	    6568 B/op	       3 allocs/op
BenchmarkMapClone/range-smallPtrMap-24      	  561308	      1910 ns/op	    3496 B/op	       3 allocs/op
BenchmarkMapClone/range-bigMap-24           	  278541	      4342 ns/op	   19112 B/op	       3 allocs/op
BenchmarkMapClone/range-bigPtrMap-24        	  594208	      1914 ns/op	    3496 B/op	       3 allocs/op
BenchmarkMapClone/clone-smallMap-24         	  418908	      2956 ns/op	    6616 B/op	       4 allocs/op
BenchmarkMapClone/clone-smallPtrMap-24      	  497443	      2438 ns/op	    3544 B/op	       4 allocs/op
BenchmarkMapClone/clone-bigMap-24           	  252829	      4735 ns/op	   19160 B/op	       4 allocs/op
BenchmarkMapClone/clone-bigPtrMap-24        	  525180	      2438 ns/op	    3544 B/op	       4 allocs/op
```

具体是什么样的问题我就不深入讲解了，因为是偷懒导致的很无聊的问题。好在这个问题会在1.25修复，修复代码已经在主分支上了，因此我们可以用`go version go1.25-devel_3fd729b2a1 Sat May 24 08:48:53 2025 -0700 windows/amd64`来测试：

```text
goos: windows
goarch: amd64
pkg: maptraps
cpu: Intel(R) Core(TM) i7-14650HX
BenchmarkMapClone/range-smallMap-24               464791              2498 ns/op            6568 B/op          3 allocs/op
BenchmarkMapClone/range-smallPtrMap-24            603405              1947 ns/op            3496 B/op          3 allocs/op
BenchmarkMapClone/range-bigMap-24                 252979              4346 ns/op           19112 B/op          3 allocs/op
BenchmarkMapClone/range-bigPtrMap-24              617744              1939 ns/op            3496 B/op          3 allocs/op
BenchmarkMapClone/clone-smallMap-24              1000000              1020 ns/op            6616 B/op          4 allocs/op
BenchmarkMapClone/clone-smallPtrMap-24           2010072               600.7 ns/op          3544 B/op          4 allocs/op
BenchmarkMapClone/clone-bigMap-24                 398769              3032 ns/op           19160 B/op          4 allocs/op
BenchmarkMapClone/clone-bigPtrMap-24             1912401               647.5 ns/op          3544 B/op          4 allocs/op
```

修复后结果就和1.22以及1.23一样了。总得来说`maps.Clone`虽然多浪费了一点内存，但速度是循环复制的1.5~3倍。

所以要复制map的时候，尽量去用`maps.Clone`，这样就能避开循环复制慢这第二个陷阱。

## 总结

golang果然还是那个golang，大道至简的外皮下往往暗藏杀机。

真要说什么原则的话，那就是如果有对应的标准库函数/内置函数，那就用，尽量少在map上直接用循环。

还有我说过无数次的，性能优化要靠benchmark，切记不要依赖经验去预判，陷阱二就是我用benchmark找出来的“预判”失误。
