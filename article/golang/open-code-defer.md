open code就是指不需要runtime介入，在每个函数的退出点都自动执行defer指定的函数。（panic的时候还是需要runtime的）这样开销很小，就像自己在每个return前写函数调用语句一样。

open code defer的数据都分配在栈上。

for-loop以及迭代器里的defer分配在堆上，且会加入runtime维护的defer链表，由runtime负责在函数退出前执行。

如果函数里出现分配在堆的defer，则其他open code defer不再open，调用也由runtime负责，但仍然分配在堆上。

defer是否在if/switch里不会影响它open。

性能对比（循环4次，次数越多效果越明显）：

```golang
func BenchmarkNoDefer(b *testing.B) {
	names, _ := os.ReadDir("./data")
	for range b.N {
		for _, name := range names {
			file, err := os.Open("./data/" + name.Name())
			if err != nil {
				panic(err)
			}
			if file.Name() == "needpanic" {
				file.Close()
				panic("needpanic")
			}
			stat, err := file.Stat()
			if err != nil {
				file.Close()
				panic(err)
			}
			if stat.Size() > 0 {
				file.Close()
				panic("big")
			}
			file.Close()
		}
	}
}

func BenchmarkForLoopDefer(b *testing.B) {
	names, _ := os.ReadDir("./data")
	for range b.N {
		for _, name := range names {
			file, err := os.Open("./data/" + name.Name())
			if err != nil {
				panic(err)
			}
			defer file.Close()
			if file.Name() == "needpanic" {
				panic("needpanic")
			}
			stat, err := file.Stat()
			if err != nil {
				panic(err)
			}
			if stat.Size() > 0 {
				panic("big")
			}
		}
	}
}
```

```text
BenchmarkNoDefer-8                  5736            197743 ns/op            2924 B/op         20 allocs/op
BenchmarkForLoopDefer-8             6152            346955 ns/op            3178 B/op         27 allocs/op
```

循环里的defer更慢更占内存而且资源还会延迟释放。

不想延迟释放也不想付出loop里用for的代价，可以把循环体包装成函数，这样defer就能open code了：

```golang
func helper(name string) {
	file, err := os.Open(name)
	if err != nil {
		panic(err)
	}
	defer file.Close()
	if file.Name() == "needpanic" {
		panic("needpanic")
	}
	stat, err := file.Stat()
	if err != nil {
		panic(err)
	}
	if stat.Size() > 0 {
		panic("big")
	}
}

func BenchmarkForLoopOptimize(b *testing.B) {
	names, _ := os.ReadDir("./data")
	for range b.N {
		for _, name := range names {
			helper("./data/" + name.Name())
		}
	}
}
```

```text
BenchmarkNoDefer-8                  6044            199649 ns/op            2923 B/op         20 allocs/op
BenchmarkForLoopDefer-8             6326            370525 ns/op            3180 B/op         27 allocs/op
BenchmarkForLoopOptimize-8          6111            197746 ns/op            2923 B/op         20 allocs/op
```

使用绑定外部变量的闭包效果一样。
