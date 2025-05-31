最近料理老项目的时候被unsafe坑惨了，这里挑一个最不易察觉的错误记录一下。

这个问题几乎影响近几年来所有的go版本，为了方便讨论我就用最新版的1.24.3做例子了。

## 线上BUG

我们有一个收集集群信息的线上系统，这个系统有好几个数据源而且数据量比较大。众所周知Go语言总是会在一些关键性能点上拉跨，我们也遇到了，所以作为性能优化策略之一，我们在一部分数据处理逻辑里使用了unsafe以减少不必要的开销。

其中一个函数是利用unsafe把字符串转换成字符切片：

```golang
func getBytes(str string) (b []byte) {
	sh := (*reflect.StringHeader)(unsafe.Pointer(&str))
	bh := reflect.SliceHeader{
		Data: sh.Data,
		Len:  sh.Len,
		Cap:  sh.Len,
	}
	return *(*[]byte)(unsafe.Pointer(&bh))
}
```

尽管`reflect.StringHeader`和它的朋友`reflect.SliceHeader`都已经标记为废弃了，但因为1.0兼容性保证，这两个东西在1.24里依旧能用，而且还能正常工作。这个函数实际上把字符串底层的内存直接赋值给了slice，让slice和字符串共用这块内存来实现零复制零分配。

这个函数的风险在于如果这块共享的内存被修改了，这个修改会意外地被我们返回的slice看到。然而string在go里是不可变的，改变一个string的值只会重新分配一块内存存放新值，是不是我们多虑了？

事实上我们没有多虑，因为真的有这种意外会发生——线上系统出Bug了。

这个Bug其实很容易观察到，因为上线没多久我们就发现数据里出现了一些重复数据还有一些脏数据。当我们把版本回滚到做unsafe优化前，数据就彻底恢复了正常。

虽然问题现象很容易发现，但问题原因就很棘手了。因为如上面所说，字符串是不可变的，理论上从字符串里拿出来的切片应该也是不会被意外改变的，更何况我们用的字符串都是拼接出来的，也不存在共用字符串变量导致问题的可能。

然而排查了两天最终我们发现正是这个“拼接”导致的问题，在看了go编译器的源码之后我写了一段最小复现代码：

```golang
func main() {
	buffers := make([][]byte, 0)
	for i := range 5 {
		s1 := "test: "
		s2 := fmt.Sprintf("%02d", i)
		s3 := s1 + s2
		buffers = append(buffers, getBytes(s3))
	}
	for _, s := range buffers {
		fmt.Println(string(s))
	}
}
```

大部分会觉得这段代码会输出`test: 01\ntest: 02\n......`，然而这段代码会输出五个`test: 04`，如果不信可以自己在1.24环境下运行一次。当然1.22，1.23结果也是一样的。

## golang在我们用+拼接字符串时都做了什么

表达式`str1 + str2`做了哪些操作，我相信会写go的人都能答出来：创建了一个新字符串然后把str1和str2的内容拼接进新字符串的内存空间里。

表达上也许会有些出入，但大部分书、教程、视频甚至是语言标准都是这么说的。

遗憾的是标准只能规定语言的行为，并不能规定实现这个行为使用的技术细节。而问题正是出现这个技术细节上。

如果要代入技术细节的话，上面对于字符串拼接的表述并不全对。因为go编译器为了优化性能做了一些额外的处理：将小字符串尽量分配在栈上。

具体是实现代码在`runtime/string.go`中：

```golang
// The constant is known to the compiler.
// There is no fundamental theory behind this number.
const tmpStringBufSize = 32

type tmpBuf [tmpStringBufSize]byte

// concatstrings implements a Go string concatenation x+y+z+...
// The operands are passed in the slice a.
// If buf != nil, the compiler has determined that the result does not
// escape the calling function, so the string data can be stored in buf
// if small enough.
func concatstrings(buf *tmpBuf, a []string) string {
	idx := 0
	l := 0
	count := 0
	for i, x := range a {
		n := len(x)
		if n == 0 {
			continue
		}
		if l+n < l { // 检测长度是否有整数溢出
			throw("string concatenation too long")
		}
		l += n
		count++
		idx = i
	}
	if count == 0 {
		return ""
	}

	// If there is just one string and either it is not on the stack
	// or our result does not escape the calling frame (buf != nil),
	// then we can return that string directly.
	if count == 1 && (buf != nil || !stringDataOnStack(a[idx])) {
		return a[idx]
	}
	s, b := rawstringtmp(buf, l)
	for _, x := range a {
		n := copy(b, x)
		b = b[n:]
	}
	return s
}
```

这是负责实现字符串拼接的函数，它会先求出所有字符串拼接后的最终长度并顺手做了长度溢出检查，然后再调用`rawstringtmp`申请容纳新字符串的空间，最后把字符copy进内存里。所以`rawstringtmp`这个函数才是重头戏：

```golang
func rawstringtmp(buf *tmpBuf, l int) (s string, b []byte) {
	if buf != nil && l <= len(buf) {
		b = buf[:l]
		s = slicebytetostringtmp(&b[0], len(b))
	} else {
		s, b = rawstring(l)
	}
	return
}

// rawstring allocates storage for a new string. The returned
// string and byte slice both refer to the same storage.
// The storage is not zeroed. Callers should use
// b to set the string contents and then drop b.
func rawstring(size int) (s string, b []byte) {
	p := mallocgc(uintptr(size), nil, false)
	return unsafe.String((*byte)(p), size), unsafe.Slice((*byte)(p), size)
}
```

`rawstringtmp`会检测申请的长度是否超过参数buf的大小，如果没有就直接从buf里切片出一个新空间，如果超过了就直接走内存分配除一块新内存不再使用buf。

那么buf有多大？最上面两行告诉你了，`[32]byte`这么大，也就是32字节。

所以其实`str1 + str2`真正的拼接操作是下面这样的伪代码：

```golang
var tmpbuf1 [32]byte

runtime.concatstrings(&tmpbuf1, []string{str1, str2})

// 如果str1和str2加起来的长度≤32，那么就会利用tmpbuf1的内存生成新字符串
// 否则从heap里重新分配内存给新字符串用
```

因为目前go的编译器对大小符合限制要求的数组，一般总是分配在栈上不会逃逸，所以能复用`tmpbuf1`空间的字符串也是在栈上的，go就这样完成了小字符串尽量分配在栈上的优化。

这个优化有啥问题呢？一般没问题，因为go编译器总是会在`[]byte(str3)`这样的表达式中复制原字符串的内容，除非极少数编译器能100%确定转换出来的`[]byte`是只读的，因此完全不用担心栈上内存的生命周期以及`tmpbuf1`是否会被意外修改。

然而不巧的是，我的`getBytes`编译器并不会特殊处理，它会直接把`tmpbuf1`的内存拿出来，go当然不知道这块内存是从哪拿来的，而且我们还用了unsafe，这就进一步阻止了编译器的检测。分配在栈上的东西在函数调用结束之后生命周期就终止了，这就是为什么数据里会有垃圾值。

但这还没解释为什么会出现重复的值和最小复现代码中的现象。

其实也很简单，因为这个`tmpbuf`是编译器进行分配的，这个buf分配在了循环外面，所以每次循环拼接字符串都会改变buf的内容。伪代码如下：

```golang
for {
    str1 := xxxx
    str2 := xxx
    str3 := str1 + str2
}

// 等价于
var tmpbuf [32]byte
for {
    str1 := xxxx
    str2 := xxx
    str3 := runtime.concatstrings(&tmpbuf1, []string{str1, str2})
}
```

这么做无可厚非，因为buf设置在循环体外性能更好，而且前面说了在不用unsafe的时候编译器能正确处理buf的生命周期以及是否需要被复制。但我们的`getBytes`就出问题了，因为每次拼接出来的字符串大小都只要二十几字节，不到32，所以每次新字符串都使用tmpbuf的内存，而循环里创建的所以字符串和从字符串获得的slice都指向tmpbuf，这就是为什么会有重复数据的原因。

不巧的是或者说运气好的是我们的系统里两类问题全都遇上了。

## 修复bug

其中一种修复办法自然是让编译器把buf放到循环体里，然而这样容易破坏已有的代码，毕竟这个优化存在很久了，另外还有是否会导致大量栈空间被浪费的问题，如果你有耐心或者更好的做法可以去go的官方github上开提案并提交PR。

不过话说回来，官方以及明说了不会保证unsafe的兼容，也不保证unsafe的安全，因此出了问题官方并不负责。所以这个问题只能我们自己改业务代码。

改起来也简单，不用unsafe就行了，原先用unsafe是因为误解了字符串拼接一定会在heap上新申请内存，那样转成`[]byte`又得申请一遍内存还要复制数据，得不偿失。现在知道拼接会优先复用栈空间，想安全使用转换出来的`[]byte`还得分配一块堆内存并把数据复制进去。对比一下会发现内存分配的次数一样，只不过数据多copy了一次。数据复制是非常快的，只有内存分配才会是性能瓶颈，因此不用unsafe也不会有太大的问题。

修复后代码是这样的：

```diff
func main() {
	buffers := make([][]byte, 0)
	for i := range 5 {
		s1 := "test: "
		s2 := fmt.Sprintf("%02d", i)
		s3 := s1 + s2
-		buffers = append(buffers, getBytes(s3))
+		buffers = append(buffers, []byte(s3))
+       // getBytes(strings.Clone(s3)) 这样也行，效果类似，但代码更复杂还要引入额外依赖，不建议
	}
	for _, s := range buffers {
		fmt.Println(string(s))
	}
}
```

新代码更简洁，同时也不会有bug。修复代码上线后系统正常了，也没有明显的性能回退，问题圆满解决。

## 总结

这回要不是我有给go的runtime和编译器提交过修改大致对字符串拼接有个印象，恐怕很难定位到是什么问题成为悬案了。

这个问题在gccgo上也存在，现象也一模一样，至于tinygo之类的编译器没有进行尝试，但就算没问题也不代表应该这样用，上一个也说了这种unsafe优化其实效果不明显更不用说它不安全了。

另外还有个坏消息，1.25新版本的go不仅字符串拼接有优化，slice内容物总大小不大于32字节的时候也会有类似的优化，因此用unsafe获取slice内容的做法也不再安全了。

最一劳永逸的办法还是避免使用unsafe，俗话说常在河边走哪有不湿鞋，除非你精通go的编译器和runtime，否则还是别在项目代码里用unsafe了，我们也是踩了好几回坑之后终于下决心把能不用unsafe的地方都改用其他方法了，剩下的地方也在进行性能评估没有太大影响的话将会删除unsafe。
