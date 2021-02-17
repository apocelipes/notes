今天是假期最后一天，明天起大家也要陆续复工了。golang1.16也在今天正式发布了。

原定计划是2月1号年前发布的，不过迟到也是golang的老传统了，正好也趁着最后的假期快速预览一下golang1.16的新特性吧。

<blockquote id="bookmark">
  <h4>本文索引</h4>
  <ul>
    <li><a href="#%E8%AF%AD%E8%A8%80%E5%85%A7%E5%BB%BA%E7%9A%84%E8%B5%84%E6%BA%90%E5%B5%8C%E5%85%A5%E6%94%AF%E6%8C%81">语言內建的资源嵌入支持</a></li>
    <li><a href="#%E6%94%AF%E6%8C%81arm64">支持arm64</a></li>
    <li>
      <a href="#go-modules%E7%9A%84%E6%96%B0%E7%89%B9%E6%80%A7">go modules的新特性</a>
      <ul>
        <li><a href="#go111module%E7%8E%B0%E5%9C%A8%E9%BB%98%E8%AE%A4%E4%B8%BAon">GO111MODULE现在默认为on</a></li>
        <li><a href="#go-build%E4%B8%8D%E5%9C%A8%E6%9B%B4%E6%94%B9mod%E7%9B%B8%E5%85%B3%E6%96%87%E4%BB%B6">go build不再更改mod相关文件</a></li>
        <li><a href="#go-install%E7%9A%84%E5%8F%98%E5%8C%96">go install的变化</a></li>
        <li><a href="#%E6%96%B0%E7%9A%84govcs%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F">新的GOVCS环境变量</a></li>
        <li><a href="#%E7%9B%B8%E5%AF%B9%E8%B7%AF%E5%BE%84%E5%AF%BC%E5%85%A5%E4%B8%8D%E5%9C%A8%E8%A2%AB%E5%85%81%E8%AE%B8">相对路径导入不再被允许</a></li>
      </ul>
    </li>
    <li>
      <a href="#%E6%A0%87%E5%87%86%E5%BA%93%E7%9A%84%E5%8F%98%E5%8C%96">标准库的变化</a>
      <ul>
        <li><a href="#testing">testing</a></li>
        <li><a href="#ioutils%E5%8C%85%E5%B7%B2%E7%BB%8F%E5%BA%9F%E5%BC%83">ioutils包已经废弃</a></li>
        <li><a href="#tcp%E5%8D%8A%E8%BF%9E%E6%8E%A5%E9%98%9F%E5%88%97%E6%89%A9%E5%AE%B9">tcp半连接队列扩容</a></li>
        <li><a href="#%E9%87%8D%E5%A4%A7%E6%9B%B4%E6%96%B0iofs">重大更新io/fs</a></li>
      </ul>
    </li>
    <li><a href="#%E5%85%B6%E4%BB%96%E6%94%B9%E8%BF%9B">其他改进</a></li>
  </ul>
</blockquote>

## 语言內建的资源嵌入支持

之前市面上已经有很多把今天文件嵌入golang二进制程序的工具了，这次golang官方将这一功能加入了`embed`标准库，从语言层面上提供了支持。

我之前以及写了embed的使用教程，可以看[这里](https://www.cnblogs.com/apocelipes/p/13907858.html)。

这儿还有一篇官方推荐的[教程](https://blog.carlmjohnson.net/post/2021/how-to-use-go-embed/)。

## 支持arm64

m1芯片可谓是最近的焦点，golang自然也不会落下。

在golang1.16中官方已经支持`darwin/arm64`平台，cgo和编译成c语言可调用的动态/静态链接库的功能也已支持。同样受益的还有bsd家族的arm64版本。

现在可以在新版mac上尝试golang了。

不过plugin模式的支持仍在进行中，想要完整支持arm64还需要一段时间。

## go modules的新特性

本次更新依旧带来了许多modules的新特性。

### GO111MODULE现在默认为on

1.16开始默认启用modules，这在1.15的时候已经预告过了。现在GO111MODULE的默认值为on。

不过golang还是提供了一个版本的适应期，如果你还不习惯modules，可以把GO111MODULE设置回auto。在1.17中这个环境变量将会被删除。

都1202年了，也该学学go modules怎么用了。

### go build不再更改mod相关文件

以前的教程里我提到过go build会自动下载依赖，这会更新mod文件。

现在这一行为被禁止了。想要安装、更新依赖只能使用go get命令，go build和go test将不会再做这类工作。

### go install的变化

go install在1.16中也有了不小的变化。

首先是通过go install my.module/tool@1.0.0 这样在module末尾加上版本号，可以在不影响当前mod的依赖的情况下安装golang程序。

go install是未来唯一可以安装golang程序的命令，go get的编译安装功能现在可以靠`-d`选项关闭，而未来编译安装功能会从go get移除。

也就是说go的命令各司其职，不再长臂管辖了。

### 新的GOVCS环境变量

新的GOVCS环境变量指定了golang用什么版本控制工具下载源代码。

其格式为：`GOVCS=<module prefix>:<tool name>,[<module prefix>:<tool name>, ...]`

其中module prefix为github.com等，而tool name就是版本控制工具的名字，比如git，svn。

一个更具体的例子是：`GOVCS=github.com:git,evil.com:off,*:git|hg`

module prefix也可以用`*`通配任何模块的前缀。

tool name还可以设置为all和off，all代表允许使用任何可用的工具，而off则表示不允许使用任何版本控制工具。

不过现在设置为off的模块的代码仍然可能会被下载。

更多的细节可以参考`go help vcs`。

### 相对路径导入不再被允许

golang1.16开始禁止import导入的模块以`.`开头，模块路径中也不允许出现任何非ASCII字符，所以下面的代码不再合法：

```golang
import (
    "./tools/factory"
    "../models/user"
    "some.pkg.com/杀马特/音乐工厂"
)
```

对非ASCII字符一如既往的不友好，不过也只能按规矩办事了。

## 标准库的变化

golang1.16除了对标准库进行通常的功能更新和修复，还引入了一些重大变化。

### testing

testing包主要的变化是在测试用例里调用`os.Exit(0)`会从程序正常终止变成测试失败。

比如这个：

```golang
package main

import (
    "os"
    "testing"
)

func TestXXX(t *testing.T) {
    t.Log("exit")
    os.Exit(0)
}
```

现在会是这样的输出：

```bash
$ go test -v a_test.go

=== RUN   TestXXX
    a_test.go:9: exit
--- FAIL: TestXXX (0.00s)
panic: unexpected call to os.Exit(0) during test [recovered]
        panic: unexpected call to os.Exit(0) during test

goroutine 18 [running]:
testing.tRunner.func1.2(0x51b920, 0x56cc28)
        /usr/local/go/src/testing/testing.go:1144 +0x332
testing.tRunner.func1(0xc000082600)
        /usr/local/go/src/testing/testing.go:1147 +0x4b6
panic(0x51b920, 0x56cc28)
        /usr/local/go/src/runtime/panic.go:965 +0x1b9
os.Exit(0x0)
        /usr/local/go/src/os/proc.go:68 +0x6d
command-line-arguments.TestXXX(0xc000082600)
        /tmp/a_test.go:10 +0x76
testing.tRunner(0xc000082600, 0x54df18)
        /usr/local/go/src/testing/testing.go:1194 +0xef
created by testing.(*T).Run
        /usr/local/go/src/testing/testing.go:1239 +0x2b3
FAIL    command-line-arguments  0.004s
FAIL
```

### ioutils包已经废弃

1.16已经标记`io/ioutil`为废弃，函数被转移到了os和io这两个包里，具体见下表：

| ioutil旧函数 | 新函数 |
| --- | --- |
| Discard | io.Discard |
| NopCloser | io.NopCloser |
| ReadAll | io.ReadAll |
| ReadDir | os.ReadDir |
| ReadFile | os.ReadFile |
| WriteFile | os.WriteFile |
| TempDir | os.MkdirTemp |
| TempFile | os.CreateTemp |

现在开始可以做移植了。

### tcp半连接队列扩容

在Linux kernel 4.1以前，golang设置tcp的listen队列的长度是从/proc/sys/net/core/somaxconn获取的，通常为4096。

而在4.1以后golang会直接设置半连接队列的长度为`2^32 - 1`也就是4294967295。

更大的半连接队列意味着可以同时处理更多的新加入请求，而且不用再读取配置文件性能也会略微提升。

### 重大更新io/fs

1.16除了支持嵌入静态资源外，最大的变化就是引入了io/fs包。

golang认为文件的io操作是依赖于文件系统（filesystem，fs）的，所以决定模仿Linux的vfs做一套基于fs的io接口。

这样做的目的有三个：

1. os包应该专注于和系统交互而不是包含一部分io接口
2. io包和os包分别包含了io接口的一部分，导致互相依赖职责不清晰
3. 可以把有关联的一部分文件或者数据组成虚拟文件系统，供通用接口处理提升程序的可扩展性，比如zip打包的文件

所以io/fs诞生了。

fs包中主要包含了下面几种数据类型（都是接口类型）：

| 名称 | 作用 |
| --- | --- |
| FS | 文件系统的抽象，有一个Open方法用来从FS打开获取文件数据 |
| DirEntry | 描述目录项目（包含目录自身）的数据结构 |
| File | 描述文件数据的结构，包含Stat，Read，Close方法 |
| ReadDirFile | 在File的基础上支持ReadDir，可以代表目录自身 |
| FileMode | 描述文件类型，比如是通常文件还是套接字或者是管道 |
| FileInfo | 文件的元数据，例如创建时间等 |

其中有一些接口和os包中的同名，实际上是os包引入fs包后起的别名。

对于FS，还有以下的扩展，以便增量描述文件系统允许的操作：

| 名称 | 作用 |
| --- | --- |
| GlobFS | 增加Glob方法，可以用通配符查找文件 |
| ReadDirFS | 增加ReadDir方法，可以遍历目录 |
| ReadFileFS | 增加ReadFile方法，可以用文件名读取文件所有内容 |
| StatFS | 增加Stat方法，可以获得文件/目录的元信息 |
| SubFS | 增加Sub方法，Sub方法接受一个文件/目录的名字，从这个名字作为根目录返回一个新的文件系统对象 |

fs包还提供了诸如Glob，WalkDir等传统的文件操作接口。

fs的主要威力在于处理zip、tar文件，以及http的文件接口时可以大幅简化代码。而且新的`embed`静态资源嵌入也是依赖fs实现的。

因为只是速览的缘故，无法详尽介绍io/fs包，你可以参考golang的文档或[这篇文章](https://benjamincongdon.me/blog/2021/01/21/A-Tour-of-Go-116s-iofs-package/)做进一步了解。

## 其他改进

其他的改进包括Unicode更新到了13.0、新增加了runtime/metrics包已提供更好更规范的运行时信息等。

同时1.16优化了链接器，现在它在linux/amd64上比1.15快了20-25%，内存占用减少了5-15%。

在Windows上已经全面支持了地址空间布局随机化（ASLR），此前不支持将golang编译为dll时启用ASLR。

本次更新中语言本身没有什么变化。

更多信息可以查看[golang1.16 release notes](https://golang.org/doc/go1.16)
