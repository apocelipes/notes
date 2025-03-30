写c/c++或者rust的开发者应该对条件编译不陌生，条件编译顾名思义就是在编译时让代码中的一部分生效或者失效，从而控制编译时的代码执行路径，进而影响编译出来的程序的行为。

这有啥用呢？通常在编写跨平台代码的时候有用。比如我想开发一个文件操作库，这个库有全平台统一的接口，然而各大操作系统提供的文件和文件系统api百花齐放，我们没法只用一套代码就让我们的库能在所有的操作系统上正常运行。

这时候就需要条件编译出场了，在Linux上我们只让适配了Linux的代码生效，在Windows上则只让Windows相关的代码生效其他失效。比如：

```c
#ifdef _Windows
typedef HFILE file_handle
#else
typedef int file_handle
#endif

file_handle open_file(const char *path)
{
    if (!path) {
#ifdef _Windows
        return invalid_handle;
#else
        return -1;
#endif
    }

#ifdef _Windows
    OFSTRUCT buffer;
    return OpenFile(path, &buffer, OF_READ);
#else
    return open(path, O_RDONLY|O_CLOEXEC);
#endif
}
```

在这个例子里，Windows和Linux的api完全不同，为了隐藏这种不同我们用条件编译在不同平台上定义出了一组相同的接口，这样我们就无需关心平台差异了。

从上面的例子也可以看出，c/c++实现条件编译最常用的是依靠宏。通过在编译时指定特定平台的标识，这些预编译宏就能自动把不需要的代码剔除不进行编译。c和c++中另一种实现条件编译的做法是依赖构建系统，我们不再使用预编译宏，但会为每个平台都编写一份代码：

```c
// open_file_windows.c
typedef HFILE file_handle

file_handle open_file(const char *path)
{
    if (!path) {
        return invalid_handle;
    }

    OFSTRUCT buffer;
    return OpenFile(path, &buffer, OF_READ);
}

// open_file_linux.c
typedef int file_handle

file_handle open_file(const char *path)
{
    if (!path) {
        return -1;
    }

    return open(path, O_RDONLY|O_CLOEXEC);
}
```

然后指定构建系统在编译Linux程序时只使用`open_file_linux.c`，在Windows上则只使用`open_file_windows.c`。这样同样可以把和当前平台无关的不兼容的代码排除掉。现在的构建系统如meson，cmake都可以轻松实现上述功能。

自称系统级的golang，自然也是支持条件编译的，而且它支持的方式是靠第二种——即依靠构建系统。

想要在golang中使用条件编译，也有两种办法。因为我们不使用宏，也没法在编译时给`go build`指定信息哪些代码不需要，所以需要一些手段来让go编译工具链识别出应该编译和应该忽略的代码。

第一种就是依赖文件后缀名。go的源代码文件的名字是有特殊规定的，符合下面格式的文件会被认为是在特定平台上需要被编译的文件：

```sh
name_{system}_{arch}.go
name_{system}_{arch}_test.go
```

其中`system`的取值和环境变量GOOS一样，常见的有`windows`、`linux`、`darwin`、`unix`，其中后缀是`unix`时文件会在Linux、bsd和darwin这些平台上编译。没有明确指定那么该文件就会在全平台有效，除非有额外指定我们后面会说的`build tag`。

`arch`的取值和GOARCH环境变量一样，都是常见的硬件平台比如`amd64`、`arm64`、`loong64`等等。有这些后缀的文件只会在为特定的硬件平台编译程序时才会生效并加入编译过程。如果没明确指定`arch`，则默认目标操作系统的所有支持的硬件平台上这个文件都会参与编译。

第一种方法简单易懂，但缺点也很明显，我们需要为每个平台都维护一份源代码文件，而且这些文件里必定会有很多重复的平台无关的代码，这对维护来说是个很大的负担。

所以第一种方案只适合那种平台间差异巨大的代码，一个典型的例子是go自己的runtime的代码，因为协程调度需要很多操作系统甚至硬件平台的功能做辅助，因此runtime在每个操作系统上出了自己的api之外差异很大，因此使用文件名后缀的形式分成多个文件维护是比较合适的。

第二种方法不再使用文件名后缀，而是依赖`build tag`这种东西来提示编译器哪些代码需要被编译。

`build tag`是go的一种编译指令，用于告诉编译器该文件需要在什么条件下才需要被编译：

```golang
//go:build 表达式
```

tag一般写在文件的开头（在版权声明之后）。其中表达式是一些tag的名字和简单的布尔运算符。比如：

```golang
//go:build !windows
表示文件在Windows以外的系统上才编译
//go:build linux && (arm64 || amd64)
表示在arm64或者amd64的Linux系统上才编译这个文件
//go:build ignore
特殊tag，表示文件不管在什么平台上都会被忽略，除非明确使用go run、go build或者go generate运行这个文件
//go:build 自定义tag名
表示只有在`go build -tags tag名`明确指定相同的tag名时才编译这个文件
```

预定义的tag的值其实就是前面文件名后缀那里提到过的`system`和`arch`。可以看到逻辑运算符和括号都可以使用，语义也和逻辑运算一样。使用tag的优点在于它可以让linux和Windows通用的逻辑出现在同一个文件里而不需要复制两份到`_windows.go`和`_linux.go`里。更重要的是它允许我们自定义编译tag。

能自定义tag的话玩法就很多了，我们来看个例子，一个可以在编译时指定日志输出级别的玩具程序，它的特点在于低于指定级别的日志不仅不会输出，而且连代码都不会存在，真正的做到零开销。

通常控制日志输出级别都是这么做的：

```golang
func DebugLog(msg ...any) {
    if level > DEBUG {
        return
    }
    ...
}
func InfoLog(msg ...any) {
    if level > INFO {
        return
    }
    ...
}
```

然而这不可避免的需要一次if判断，如果函数比较复杂的话还需要付出一次额外的函数调用开销。

使用条件编译可以消除这些开销，首先是处理debug级别的日志函数：

```golang
// file log_debug.go
//go:build debug || (!info && !warning)
package log

import "fmt"

func Debug(msg any) {
    fmt.Println("DEBUG:", msg)
}

// file log_no_debug.go
//go:build info || warning
package log

func Debug(_ any) {}
```

作为最低的级别，只有在指定了debug这个tag以及默认情况下才会生效，其他时间都是空函数。

info级别的处理是一样的，只有指定级别为debug和info时才生效：

```golang
// file log_info.go
//go:build !debug && !warning
package log

import "fmt"

func Info(msg any) {
	fmt.Println("INFO:", msg)
}

// file log_no_info.go
//go:build warning

package log

func Info(_ any) {}
```

最后是warning级别，这个级别的日志不管在什么时候都会输出，因此它不需要条件编译所以也不需要tag：

```golang
// file log_warning.go
package log

import "fmt"

func Warning(msg any) {
	fmt.Println("WARN:", msg)
}
```

最后是main函数：

```golang
package main

import "conditionalcompile/log"

func main() {
	log.Debug("A debug level message")
	log.Info("A info level message")
	log.Warning("A warning level message")
}
```

因为我们把不生效的函数都写成了空函数，因此编译器会在编译时发现这些空函数的调用什么都没做，因此直接忽略掉它们，所以运行的时候不会产生任何额外的开销。

下面简单做个测试：

```bash
$ go run

# 输出
DEBUG: A debug level message
INFO: A info level message
WARN: A warning level message

$ go run -tags info .

# 输出
INFO: A info level message
WARN: A warning level message

$ go run -tags warning .

# 输出
WARN: A warning level message
```

和我们预期的一致。不过我并不推荐你使用这个方法，因为它需要为每个日志函数编写两份代码，而且需要对编译tag做很复杂的逻辑运算，非常容易出错；而且运行时一次if判断一般也不会带来太多的性能开销，除非明确定位到了判断日志级别产生了不可接受的性能瓶颈，否则永远不要尝试使用上面的玩具代码。

不过生产实践里真的有使用自定义tag的例子：wire。

依赖注入工具wire让开发者把需要注入的依赖写入有特殊编译tag的源文件，这些源文件正常编译的时候不会被编译到程序里，使用wire工具生成注入代码的时候这些文件才会被识别，这样既可以正常实现依赖注入功能又不会对代码产生太大的影响。更具体的做法可以去看wire的使用教程。

至于选择在golang里选择哪种方式实现条件编译，这个得结合实际需求来看。至少像go自身的代码以及k8s中两种方法文件名后缀和build tag都有并行使用，最重要的选择依据还是要以方便自己和他人维护代码为准。
