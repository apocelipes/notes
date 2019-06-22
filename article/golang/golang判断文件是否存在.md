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

### 一点提示

<span style="color:red;">当我们的`FileExist`返回true时，其实文件并不一定存在。</span>

当我们对目标path中的某一部分没有可读权限时，`os.Lstat`和`syscall.Access`同样会返回error，不过这个error不会让`os.IsNotExist`返回true。

当文件不存在而你对文件所在的目录或者它的上层目录没有访问权限时，`FileExist`依旧会返回true，bug就在这时发生了。

所以重要的一点是<span style="color:red;">在判断文件是否存在前应该先判断自己对文件及其路径是否有访问权限</span>。
