今天讲一个常见的gc compiler（也就是官方版本的go编译器和runtime）在垃圾回收的扫描标记阶段做的优化。

我对这个优化的描述印象最深的是在bigcache的注释里，大致内容是如果map的键值都不包含指针，那么gc扫描的时候不管这个map多大都不会深入扫描map内部存储的数据，只检查map本身是否需要回收。

这么做的好处显然是可以让gc的扫描速度大大增加，从而减少gc对性能的损耗。

减少指针数量本身就是常见的优化手段，但让我感到好奇的是注释里说的“跳过”。跳过的依据究竟是什么，以及只有map存在这种跳过吗？

于是我进行了全面的搜索，结果除了复读bigcache里那段话的，没什么有用的发现。

于是这篇文章诞生了。

## 跳过扫描指的是什么

前置知识少不得。

简单的说，gc在检查对象是否存活的时候，除了对象本身，还要检查对象的子对象是否引用了其他对象，具体来说：

- 数组和slice的话指存储在里面的每一个元素是否存活，这里被存储的元素是数组/slice的子对象
- map的子对象就是里面存的键和值了
- struct的子对象是它的每一个字段

为了检查这些子对象是否引用了其他对象（关系到这些被引用的对象是否能被回收），gc需要深入扫描这些子对象。子对象越多需要扫描的东西就越多。而且这个过程是递归的，因为子对象也会有子对象，想象一下嵌套的数组或者map。

跳过扫描自然就是指跳过这些子对象的扫描，只需要检查对象本身即可的操作。

## 什么样的对象是可以跳过扫描的

这也是我的第一个疑问。跳过或不跳过的依据是什么，又或者是什么东西在控制这一过程。

bigcache告诉我们存有不包含指针的键值对的map是可以跳过的，那么具体情况是怎么样的呢？

找不到有用的资料，那只能看代码了，代码以Go 1.22.1为准。

首先应该想到的应该是从gc的代码开始看，于是很快就有了收获:

```go
// runtime/mgcmark.go
// 负责gc扫描的函数，还有个它的兄弟gcDrainN，代码差不多就不放了
func gcDrain(gcw *gcWork, flags gcDrainFlags) {
    ...
    // 先标记所有root对象，检查对象是否存活就是从这开始的
    if work.markrootNext < work.markrootJobs {
		for !(gp.preempt && (preemptible || sched.gcwaiting.Load() || pp.runSafePointFn != 0)) {
			markroot(gcw, job, flushBgCredit)
			// 检查自己是否需要被中断，需要的场合函数会直接跳到收尾工作然后返回
		}
	}

    // 从工作队列里拿需要扫描的对象进行处理
    for !(gp.preempt && (preemptible || sched.gcwaiting.Load() || pp.runSafePointFn != 0)) {
		b := gcw.tryGetFast() // 从工作队列拿对象
		scanobject(b, gcw)
        ...
	}
    ...
}
```

流程不考虑中断、数据统计和校验的话还是很简单的，就是先标记扫描的起点，然后从gcw这个工作队列里拿东西出来处理，直到工作队列里再也没数据了为止。

`markroot`也很简单，根据root对象的种类，它会调用`scanblock`或者`markrootSpans`。其中`scanblock`会调用`greyobject`来标记待处理的对象。因此稍微看看`markrootSpans`即可。

`markrootSpans`是用来处理那些存放设置了终结器的对象的内存的：

```golang
// runtime/mgcmark.go
func markrootSpans(gcw *gcWork, shard int) {
	...
	for i := range specialsbits {
		...
		for j := uint(0); j < 8; j++ {
			// 找到要处理的span（go内存使用的单位，你就当是“一块内存空间”就行）
			s := ha.spans[arenaPage+uint(i)*8+j]

			...
            lock(&s.speciallock)
			for sp := s.specials; sp != nil; sp = sp.next {
				if sp.kind != _KindSpecialFinalizer {
					continue
				}
				// don't mark finalized object, but scan it so we
				// retain everything it points to.
                // spf是终结器本身
				spf := (*specialfinalizer)(unsafe.Pointer(sp))
				// A finalizer can be set for an inner byte of an object, find object beginning.
				p := s.base() + uintptr(spf.special.offset)/s.elemsize*s.elemsize

				// p是设置了终结器的对象
                // 这里检查这个对象占用的内存上是否设置了跳过扫描的标记
                // 设置了的话就不要继续扫描对象自己的子对象了
				if !s.spanclass.noscan() {
					scanobject(p, gcw)
				}

				// 这个span本身就是root对象，所以剩下的直接用scanblock处理
				scanblock(uintptr(unsafe.Pointer(&spf.fn)), goarch.PtrSize, &oneptrmask[0], gcw, nil)
			}
			unlock(&s.speciallock)
		}
	}
}
```

其实很简单，依旧是找到所有的对象，然后进行处理。然而我们看到了有意思的东西：`s.spanclass.noscan()`。

看起来这和是否跳过扫描有关。

但我们先不深入这个方法，为什么？因为终结器是被特殊处理的，没看完`scanobject`和`greyobject`之前我们不能断言这个方法是否控制着对对象的扫描。（其实注释上我已经告诉你就是这个东西控制的了，但如果你自己跟踪代码的话头一次看到这段代码的时候是不知道的）

所以我们接着看`scanobject`，这个函数是扫描对象的子对象的：

```golang
// runtime/mgcmark.go
func scanobject(b uintptr, gcw *gcWork) {
	// 先拿到还没扫描过的内存
	s := spanOfUnchecked(b)
	n := s.elemsize
    // n 表示mspan里有几个对象，在被这个函数检查的时候肯定不能是0
	if n == 0 {
		throw("scanobject n == 0")
	}
	if s.spanclass.noscan() {
		// 如果内存设置了noscan标志，就报错
		throw("scanobject of a noscan object")
	}

	var tp typePointers
	if n > maxObletBytes {
		// 大内存分割成不同的块放进工作队列，这样能被并行处理
		if b == s.base() {
			// 分割后入队
			for oblet := b + maxObletBytes; oblet < s.base()+s.elemsize; oblet += maxObletBytes {
				if !gcw.putFast(oblet) {
					gcw.put(oblet)
				}
			}
		}

		// 获取类型信息
	} else {
		// 这里不重要
	}

	var scanSize uintptr
	for {
		var addr uintptr
        // 获取子对象
        // 整个循环的退出条件就是next不再返回子对象的时候（没东西可继续扫描了）
		if tp, addr = tp.nextFast(); addr == 0 {
			if tp, addr = tp.next(); addr == 0 {
				break
			}
		}

		// 拿到要处理的对象
		scanSize = addr - b + goarch.PtrSize
		obj := *(*uintptr)(unsafe.Pointer(addr))

		// 排除nil和指向当前对象自身的指针
        // 后者属于可以被回收的循环引用，当前对象能不能回收不受这个指针影响
        // 因为如果当前对象不可访问了，那么它的字段自然也是不可能被访问到的，两者均从root不可达
        // 而如果这个指针是可达的，那么当前对象的字段被引用，当前对象也是不需要回收的
        // 所以指向当前对象本身的指针字段不需要处理
		if obj != 0 && obj-b >= n {
			if obj, span, objIndex := findObject(obj, b, addr-b); obj != 0 {
				greyobject(obj, b, addr-b, span, gcw, objIndex)
			}
		}
	}
	...
}
```

这个函数长归长，条理还是清晰的：

1. 首先看看对象是否太大要把对象的内存分割成小块交给工作队列里的其他协程并行处理
2. 接着扫描所有子对象，用`greyobject`标记这些对象

因为这个函数本身已经是在扫描了，所以不太会有“跳过”的相关的逻辑，而且你也看到了把这个函数放在不需要扫描子对象的对象上调用时会触发throw，throw会导致程序报错并退出执行。

所以秘密就在`greyobject`里了。看看代码：

```golang
// runtime/mgcmark.go
func greyobject(obj, base, off uintptr, span *mspan, gcw *gcWork, objIndex uintptr) {
	...
	if useCheckmark {
		if setCheckmark(obj, base, off, mbits) {
			// Already marked.
			return
		}
	} else {
		...
		// If marked we have nothing to do.
		if mbits.isMarked() {
			return
		}
		mbits.setMarked()

		...

		// 如果内存被标记为不需要进一步扫描，则会跳过后续的流程（内存会被放进gc扫描的工作队列里等着被取出来扫描）
		if span.spanclass.noscan() {
			...
			return
		}
	}
	// 对象被放进工作队列等待扫描
}
```

这个函数会先检查对象是否已经被处理过，然后标记对象，接着检查span上的`noscan`标志，设置了的话就返回调用，没有设置说明需要被进一步扫描，于是被放进工作队列，等着`gcDrain`或者它兄弟来处理。

现在我们可以得出结论了，会不会跳过扫描，全部由内存上是否设置`noscan`标志来控制，设置了就可以跳过。

至于在这块内存上的是map还是slice还是struct，没关系。

## 跳过扫描的具体流程

看了上面的代码，我想信你一定是懵的，跳过具体发生的流程是什么样的呢？

没关系，我们看两个例子就知道了。

第一个例子是一个顶层的全局的可跳过扫描的对象A，介于我们还没说`noscan`会在什么情况下被设置，所以我们先忽略A的具体类型，只要知道它可以跳过扫描即可。

A的扫描流程是这样的：

1. gc开始运行，先标记root对象
2. A就是root之一，所以它要么被`scanblock`处理要么被`markrootSpan`处理
3. 假设A设置了终结器，又因为A是可跳过扫描子对象的，因此`markrootSpan`会直接调用`scanblock`
4. `scanblock`会调用`greyobject`处理内存里的对象
5. 因为A可跳过扫描，所以`greyobject`做完标记就返回了，A不会进入工作队列
6. A的扫描结束，整个流程上不会有`scanobject`的调用

A的例子相对简单，现在我们假设有个不是root对象的对象B，B本身不可跳过扫描，B有一个子对象C可以跳过扫描。我们来看看C的扫描流程：

1. 因为B并不是root对象，且不可跳过扫描，所以它作为某个root对象的子对象，现在肯定在gc工作队列里
2. `gcDrain`从队列里拿到了B，于是交给了`scanobject`处理
3. 我们假设B不是很大因此不会被分割（反正分割了也一样）
4. `scanobject`把每个B的子对象都用`greyobject`处理，C也不例外
5. 因为C可跳过扫描，所以`greyobject`做完标记就返回了，C不会进入工作队列
6. C的扫描结束，整个流程上不会有对C的`scanobject`的调用

这样基本涵盖了所有的情况，一些我没单独说的比如“可跳过对象E是不可跳过root对象D的子对象”这样的情况，实际上和情况2没什么区别。

现在对象的子对象扫描是这么跳过的我们也知道了，只剩一个疑问了：noscan标志是怎么设置的？

## noscan标志是怎么设置的

在深入之前，我们先来简单看下go的怎么分配内存的。完整讲解恐怕5篇长文也兜不住，所以我做些概念上的精简。

在go里，`mspan`是内存分配的基础单位，一个mspan上可以分配多个大小类似可以被归为一类的对象（比如13字节和14字节的对象都是一类，可以被分配到允许最大存储16字节对象的mspan上）。这个“类型”就叫mpan的`sizeclass`。一个简单的心智模型是把mspan当成一个能存大小相近的对象的列表。

为了加快内存分配，go会给每个线程预分配一块内存，然后按sizeclass分成多份，每份对应一个sizeclass的mspan。这个结构叫`mcache`。

当然了，总有对象的大小会超过所有mcache的sizeclass规定的范围，这个时候go就会像系统申请一大块内存，然后把内存交给mspan。

存储了span信息的比如sizeclass和noscan的结构叫`spanClass`。这个结构会作为字段存储在mspan的控制结构里。

知道了这些之后，我们就能看懂`s.spanclass.noscan()`了，它的意思就是检查mspan的spanclass信息是否设置了不需要扫描子对象的标志。

而创建spanclass只能用`makeSpanClass`这个函数：

```golang
// runtime/mheap.go
type spanClass uint8

func makeSpanClass(sizeclass uint8, noscan bool) spanClass {
	return spanClass(sizeclass<<1) | spanClass(bool2int(noscan))
}
```

现在问题简单了，我们只要追踪谁调用了这个函数就行，以及我们还知道额外的信息：这些调用者还需要从mcache或者系统申请内存获得mspan结构。这样一下范围就收缩了。

按上面的思路，我们很快就找到了go分配内存给对象的入口之一`mallocgc`：

```golang
// runtime/malloc.go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	...
    // size是指类型的大小
    // typ是需要创建的对象的类型信息，如果只是分配内存，typ要传nil
	// typ是否是空的或者typ是否包含有指针
	noscan := typ == nil || !typ.Pointers()
	if 如果size足够小可以从mspan上分配 {
		if size满足要求可以用tinyallocator分配 {
		} else {
			// 计算sizeclass（size对应到哪一类span）
			spc := makeSpanClass(sizeclass, noscan) // noscan是这里传进去的
			span = c.alloc[spc] // 从mcache拿mspan
            v := nextFreeFast(span) // 从mspan真正拿到可用的内存
			// 后面是把内存内容清零和维护gc信息等代码
		}
	} else {
		// 大对象分配
		// mcache.allocLarge也调用makeSpanClass(0, noscan)，然后用mheap.alloc根据span的信息从系统申请内存
		span = c.allocLarge(size, noscan) // noscan是这里传进去的
		// 后面是把内存内容清零和维护gc信息等代码
	}
}
```

即使sizeclass是一样的，因为noscan的值不一样，两个spanClass的值也是不一样的。对于可跳过扫描的大对象来说，会把为这个对象分配的内存标记为noscan；对于可跳过的小对象来说，会直接把这个小对象放在mcache提前分配的不需要深入扫描的内存区域上。

那么这个`mallocgc`又是谁调用的？答案太多了，因为new，make都会用到它。我们用slice和map做例子看看。

首先是slice。这个非常简单，创建slice的入口是`makeslice`：

```golang
// runtime/slice.go
func makeslice(et *_type, len, cap int) unsafe.Pointer {
	mem, overflow := math.MulUintptr(et.Size_, uintptr(cap))
	if overflow || mem > maxAlloc || len < 0 || len > cap {
		// NOTE: Produce a 'len out of range' error instead of a
		// 'cap out of range' error when someone does make([]T, bignumber).
		// 'cap out of range' is true too, but since the cap is only being
		// supplied implicitly, saying len is clearer.
		// See golang.org/issue/4085.
		mem, overflow := math.MulUintptr(et.Size_, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen()
		}
		panicmakeslicecap()
	}

	return mallocgc(mem, et, true)
}
```

slice中的元素的类型信息被传给了`mallocgc`。如果slice的元素不包含指针，那么slice是可以跳过扫描的。

map比较特殊，跳过扫描的是它的bucket，而bucket外界是看不到的：

```golang
// runtime/map.go
// 调用链：makemap -> makeBucketArray -> newarray -> mallocgc
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	base := bucketShift(b)
	nbuckets := base
	// For small b, overflow buckets are unlikely.
	// Avoid the overhead of the calculation.
	if b >= 4 {
		// Add on the estimated number of overflow buckets
		// required to insert the median number of elements
		// used with this value of b.
		nbuckets += bucketShift(b - 4)
		sz := t.Bucket.Size_ * nbuckets
		up := roundupsize(sz, !t.Bucket.Pointers())
		if up != sz {
			nbuckets = up / t.Bucket.Size_
		}
	}

	if dirtyalloc == nil {
        // t.Bucket.Pointers() 返回键值对中是否包含指针
		buckets = newarray(t.Bucket, int(nbuckets))
	} else {
		// dirtyalloc was previously generated by
		// the above newarray(t.Bucket, int(nbuckets))
		// but may not be empty.
		buckets = dirtyalloc
		size := t.Bucket.Size_ * nbuckets
		if t.Bucket.Pointers() {
			memclrHasPointers(buckets, size)
		} else {
			memclrNoHeapPointers(buckets, size)
		}
	}

	if base != nbuckets {
		// We preallocated some overflow buckets.
		// To keep the overhead of tracking these overflow buckets to a minimum,
		// we use the convention that if a preallocated overflow bucket's overflow
		// pointer is nil, then there are more available by bumping the pointer.
		// We need a safe non-nil pointer for the last overflow bucket; just use buckets.
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.BucketSize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.BucketSize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}

func newarray(typ *_type, n int) unsafe.Pointer {
	if n == 1 {
		return mallocgc(typ.Size_, typ, true)
	}
	mem, overflow := math.MulUintptr(typ.Size_, uintptr(n))
	if overflow || mem > maxAlloc || n < 0 {
		panic(plainError("runtime: allocation size out of range"))
	}
	return mallocgc(mem, typ, true)
}
```

可以看到要是键值对里都不包含指针的话，map就可以被跳过。

所以总结下，只要创建的对象不包含指针（例如数组/切片成员都是不包含指针的类型，map的键值对都不包含指针，结构体所有字段不包含指针）或者只是单纯分配块内存（`makeslicecopy`里分配一块内存然后再把数据copy进去的时候会判断element里包不包含指针，不包含的时候会传nil给`mallocgc`），noscan就会被设置。

现在所有的疑问都解决了：noscan是内存分配时根据类型信息来设置的；能跳过扫描的不只是map，符合条件的类型不管是slice、map还是struct都可以。

## 优化带来的提升

说了这么多，这个优化带来的提升有多少呢？

看个例子：

```golang
var a int64 = 1000

func generateIntSlice(n int64) []int64 {
	ret := make([]int64, 0, n)
	for i := int64(0); i < n; i++ {
		ret = append(ret, a)
	}
	return ret
}

func generatePtrSlice(n int64) []*int64 {
	ret := make([]*int64, 0, n)
	for i := int64(0); i < n; i++ {
		ret = append(ret, &a)
	}
	return ret
}

func BenchmarkGCScan1(b *testing.B) {
	defer debug.SetGCPercent(debug.SetGCPercent(-1)) // 测试期间禁止自动gc
	for i := 0; i < b.N; i++ {
		for j := 0; j < 20; j++ {
			generatePtrSlice(10000)
		}
		runtime.GC()
	}
}

func BenchmarkGCScan2(b *testing.B) {
	defer debug.SetGCPercent(debug.SetGCPercent(-1))
	for i := 0; i < b.N; i++ {
		for j := 0; j < 20; j++ {
			generateIntSlice(10000)
		}
		runtime.GC()
	}
}
```

我们分别创建20个包含10000个`int64`或者`*int64`的slice（两个类型在x64系统上都是8字节大小），然后手动触发一次GC。为了让结果更准确，我们还在测试开始前禁用了自动触发的gc，而且我们创建的slice的长度和slice里元素的大小都是一样的，所以总体来说结果应该比较接近真实的gc回收内存时的性能。

这是结果：

```text
goos: windows
goarch: amd64
cpu: Intel(R) Core(TM) i5-10200H CPU @ 2.40GHz
         │   old.txt   │               new.txt               │
         │   sec/op    │   sec/op     vs base                │
GCScan-8   379.0µ ± 2%   298.0µ ± 2%  -21.51% (p=0.000 n=10)

         │   old.txt    │            new.txt             │
         │     B/op     │     B/op      vs base          │
GCScan-8   1.563Mi ± 0%   1.563Mi ± 0%  ~ (p=0.438 n=10)

         │  old.txt   │            new.txt             │
         │ allocs/op  │ allocs/op   vs base            │
GCScan-8   20.00 ± 0%   20.00 ± 0%  ~ (p=1.000 n=10) ¹
¹ all samples are equal
```

内存用量大家都一样，但存指针的时候速度慢了五分之一。slice越大差距也会越大。可见跳过扫描带来的提升还是很大的。

另外少用指针还有助于增加数据的局部性，不仅仅是惠及gc扫描。

## 如何利用这一优化

最后我们看看如何利用这一优化。

少用指针可以减轻gc压力大家都知道，但有一些“不得不用”指针的时候。

以一个本地cache为例：

```golang
type Cache[K comparable, V any] struct {
    m map[K]*V
}

func (c *Cache[K, V]) Get(key K) *V {
    return c.m[key]
}

func (c *Cache[K, V]) Set(key Key, value *V) {
    c.m[key] = value
}
```

值需要用指针是有两个原因，一是map的元素不能取地址，如果我们想要cache里的数据可以自由使用的话那就不得不用临时变量加复制，这样如果我们想更新值的时候就会很麻烦；二是如果值很大的话复制带来的开销会很大，用cache就是想提升性能呢反过来下降了怎么行。

但这么做就会导致`Cache.m`里的每一个键值对要被扫描，如果键值对很多的话性能会十分感人。

这样看起来是“不得不用指针”的场景。真的是这样吗？考虑到cache本身就是空间换时间的做法，我们不妨再多用点空间：

```golang
type index = int

type Cache[K comparable, V any] struct {
    buf []V
    m map[K]index
}

func (c *Cache[K, V]) Get(key K) *V {
    idx, ok := c.m[key]
    if !ok {
        return nil
    }
    return &c.buf[idx] // 可以对slice里存的数据取地址
}

func (c *Cache[K, V]) Set(key Key, value V) {
    idx, ok := c.m[key]
    if !ok {
        // 新建
        c.m[key] = len(c.buf)
        c.buf = append(c.buf, value)
        return
    }
    // 覆盖已添加的
    c.buf[idx] = value
}
```

我们用一个slice来存所有的值，然后再把key映射到值在slice中的索引上。对于slice的元素，我们是可以取地址的，因此可以简单拿到值的指针，对于值的更新也可以基于这个`Get`拿到的指针，时间复杂度不变，简单又方便。

然后我们再来看，现在`buf`和`m`都没有指针了，只要`K`和`V`不包含指针，那么不管我们的cache里存了多少东西对gc来说都只要看看外层的`Cache`对象是否存活就够了。

但是这么做会有代价：

- Get会稍微慢一点，因为不仅要做额外的检查，还需要从两个不同的数据结构里拿数据，对缓存不友好
- 存数据进Cache的时候不可避免地需要一次复制
- Get返回的指针没有稳定性，在底层的buf扩容后就会失效
- 删除元素会很慢，这怪我们用了slice而且需要维护map里的映射关系，解决方法倒是不少，比如你可以把待删除元素和slice结尾的元素交换这样slice里的其他元素不用移动map也只要遍历一次，又比如你可以再多浪费点内存用墓碑标志来模拟删除或者干脆不提供删除功能（不好做就干脆不做，这是非常golang的做法👍）

顺带一提，前面说的bigcache就利用了类似的做法减轻了gc扫描压力。

所以我建议先用benchmark和trace确定gc是性能瓶颈之后再进行上面这样的优化，否则性能优化不了还会带来很多额外的烦恼。

## 总结

现在我们知道了对于不包含指针的对象，gc会跳过对它内部子对象的扫描，这个优化不止于map。

**接口虽然看起来不像指针，但其实它内部也有指针，因此接口是要被深入扫描的**。

另外还要强调一点，这个只是官方版本的go做的优化，不保证其他的编译器实现比如gccgo、tinygo会有类似的优化。但少用指针减轻gc压力是大多数语言的共识，这点不会错。

最后的最后，还是老话，过早的优化是万恶之源，但用这句话给自己低性能的设计找借口更是错上加错，性能问题靠数据说话，多做benchmark少做白日梦。
