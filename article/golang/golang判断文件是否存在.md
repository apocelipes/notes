判断一个文件是否存在是一个相当常见的需求，在golang中也有多种方案实现这一功能。

现在我们介绍其中两种最常用也是最简单的实现，第一种将是跨平台通用的，而第二种则在POSIX平台上通用。

## 跨平台实现

跨平台实现的思路很简单，如果某个文件不存在，那么使用`os.Lstat`就一定会返回error，只要判断error是否代表文件不存在即可。

也许你注意到了有些代码会使用`os.Open`来完成上述工作，不过最好不要这么做，因为虽然两者完成的功能没有区别，但open和stat的调用开销是不同的，后者要小于前者，而且对于判断文件是否存在，检查它的元数据要比直接尝试打开它更加合理。

那么来看看实现的代码：

```golang
func FileExist(path string) bool {
  _, err := os.Lstat(path)
  return !os.IsNotExist(err)
}
```

代码很简单，对于Windows/Linux/MacOS等是通用的，一般没有特殊需求我也比较推荐这种实现。

## POSIX平台实现

如果你的程序是面向POSIX平台的（例如UNIX、Linux等），那么还有一种更简单的方案——`syscall.Access`。

`syscall.Access`提供了用户检查文件元信息的手段，通常它被用来检查文件权限以及文件的存在性。

通过使用`syscall.F_OK`标志检查文件，如果不存在则会返回和`os.Lstat`一样的error：

```golang
func FileExist(path string) bool {
  err := syscall.Access(path, syscall.F_OK)
  return !os.IsNotExist(err)
}
```

这种实现的最大优势在于它简单而直观，但是它无法在Windows上使用。

### 一些提示

首先**当我们的`FileExist`返回true时，其实文件并不一定存在。**

当我们对目标path中的某一部分没有可读权限时，`os.Lstat`和`syscall.Access`同样会返回error，不过这个error不会让`os.IsNotExist`返回true。

当文件不存在而你对文件所在的目录或者它的上层目录没有访问权限时，`FileExist`依旧会返回true，bug就在这时发生了。所以重要的一点是**在判断文件是否存在前应该先判断自己对文件及其路径是否有访问权限**。

其次`syscall.Access`只会使用运行程序的用户的uid和gid，这会导致setuid之类的权限失效，通常来说这是没什么问题的，然而posix平台上一般都会考虑euid和egid，因此你可能需要使用`syscall.Faccessat`做代替。你需要在深思熟虑后使用合适的系统调用。

## 性能测试

最后我们看看两个方案的性能，我们以`os.Open`做为基准，分别测试先文件存在和不存在时的性能表现：

```golang
func checkWithOpen(path string) bool {
	f, err := os.Open(path)
	if err != nil {
		return false
	}
	f.Close()
	return true
}

func checkWithLstat(path string) bool {
	_, err := os.Lstat(path)
	return !os.IsNotExist(err)
}

func checkWithAccess(path string) bool {
  err := syscall.Access(path, syscall.F_OK)
	return !os.IsNotExist(err)
}

func BenchmarkNotExists(b *testing.B) {
	for range b.N {
		checkWithOpen("/home/apocelipes/no-")
	}
}

func BenchmarkNotExistsLstat(b *testing.B) {
	for range b.N {
		checkWithLstat("/home/apocelipes/no-")
	}
}

func BenchmarkNotExistsAccess(b *testing.B) {
	for range b.N {
		checkWithAccess("/home/apocelipes/no-")
	}
}

func BenchmarkExists(b *testing.B) {
	for range b.N {
		checkWithOpen("/home/apocelipes/.zshrc")
	}
}

func BenchmarkExistsLstat(b *testing.B) {
	for range b.N {
		checkWithLstat("/home/apocelipes/.zshrc")
	}
}

func BenchmarkExistsAccess(b *testing.B) {
	for range b.N {
		checkWithAccess("/home/apocelipes/.zshrc")
	}
}
```

这是结果：

```text
goos: linux
goarch: amd64
pkg: fileexiststest
cpu: Intel(R) Core(TM) i5-10200H CPU @ 2.40GHz
BenchmarkNotExists-4             1305411               939.3 ns/op            72 B/op          2 allocs/op
BenchmarkNotExistsLstat-4        1461896               833.8 ns/op           280 B/op          3 allocs/op
BenchmarkNotExistsAccess-4       1962312               614.7 ns/op            24 B/op          1 allocs/op
BenchmarkExists-4                 304419              3324 ns/op             128 B/op          3 allocs/op
BenchmarkExistsLstat-4           1331382               952.5 ns/op           232 B/op          2 allocs/op
BenchmarkExistsAccess-4          1793318               663.4 ns/op            24 B/op          1 allocs/op
PASS
ok      fileexiststest  11.210s
```

测试使用的文件系统类型是XFS。

可以看到open是最慢的，lstat比access慢了16%左右。从结果里也可以看到lstat需要额外返回一个`os.FileInfo`结构导致了额外的内存分配，所以整体上速度更慢。

但考虑到跨平台以及兼容性，使用`os.Lstat`是更长见的做法。
