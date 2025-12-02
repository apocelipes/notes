在类UNIX系统上，可执行文件和shell脚本一般都是不带后缀名的，操作系统内置的程序加载器会自动检测文件的权限和内容是否是一个可执行的程序。这么做的好处是可以在输入命令的时候少打很多字。坏处自然是不对文件做彻底的检查就无法确定其是否是可执行文件，这会带来一些安全问题。

Linux则更进一步，提供了一套叫`binfmt_misc`的机制让用户自定义哪些格式的文件是可执行文件，进一步提升了系统灵活性。

这篇文章就简单讲解一下Linux的`binfmt_misc`的工作原理和应用。阅读这篇文章需要一些知识储备：

1. 会简单的Linux操作
2. 知道什么是shell脚本
3. 简单了解过Python、c/c++、Go、Shell或者js等任何一门能进行Linux编程的语言

当然上面这些都只需要简单了解即可，下面就进入正文吧。

## 什么是binfmt_misc

`binfmt_misc`全称是“Miscellaneous Binary Format”，它提供了一种用户接口，可以让用户注册自定义的可执行文件格式给内核。

内核在执行程序时会先检查用户和程序文件的权限，然后让程序加载器根据规则加载并执行程序，`binfmt_misc`所做的就是添加用户的自定义规则到加载器的规则集合中，使得除了传统意义上的可执行文件（ELF文件或者有Shebang的脚本）之外的其他文件也可以直接被执行。

举个例子，在Linux模拟Windows环境运行exe程序的模拟器wine，可以通过`binfmt_misc`机制把exe文件对应的执行规则注册进加载器的规则集合，之后用户就可以像使用普通的Linux程序一样直接执行exe文件，内核会检查到wine注册的规则，自动调用wine来模拟运行exe程序。

知道`binfmt_misc`是什么之后，下面我们来看看`binfmt_misc`提供的用户注册接口。

## binfmt_misc的用户接口

说是用户接口，但因为涉及到操作内核数据以及影响整个系统的行为，所以`binfmt_misc`提供的接口都需要root权限，接口的操作结果会对所有用户立即生效。

`binfmt_misc`接口的操作结果只在系统运行中生效，关机重启之后之前人工注册的规则就消失了，所以有持久化需求的需要主动把注册命令写入启动脚本之类的东西里。

`binfmt_misc`提供的接口不是系统调用，也不是特殊的命令，而是在`/proc/sys/fs/binfmt_misc`目录下的一系列文件，通过读取和写入这些文件，可以实现注册规则、删除规则、暂停规则、查看规则状态等操作。

接口文件主要有这几个：

1. `/proc/sys/fs/binfmt_misc/register`，一个不能读取只能写入的文件，写入固定格式的数据可以注册规则到内核
2. `/proc/sys/fs/binfmt_misc/status`，可读可写的文件，读取时获取当前内核是否开启`binfmt_misc`机制，返回值是`enabled/disabled`；写入则可以关闭或重新开启`binfmt_misc`，允许写入的值只有`0`（关闭binfmt_misc）、`1`（重新打开）、`-1`（删除所有注册规则）。
3. `/proc/sys/fs/binfmt_misc/<rule-name>`，所有注册的规则都会生成一个和规则名相同的文件，读取整个文件会获得规则的详细信息，写入则可以控制整个规则，允许写入的值有`0`（暂时让规则失效）、`1`（重新生效）、`-1`（删除这个规则，对应的文件也会在写入完成之后被删除）。

这些接口都比较简单，你可以通过命令行或者任意一种可以读写系统文件的编程语言来操作它们。

接口中最核心的是`/proc/sys/fs/binfmt_misc/register`，向它写入数据才能完成我们自定义规则的注册。

注册规则的数据格式是`:name:type:offset:magic:mask:interpreter:flags`，每个部分都以冒号开头，字段可以省略但前导冒号需要保留，比如我们想省略mask和flags字段，就得写成`:name:type:offset:magic::interpreter:`。下面解释一下每个字段的意义：

1. name，规则的名称，目录下生成的虚拟文件的名字也是它，所以name中不能包含`/`，也不能出现一些其他在文件名中不允许出现的字符。不同规则直接不能重名。
2. type，设置以哪种方式识别文件，支持两个选项`M`和`E`，其中`M`表示通过文件头来识别文件，`E`则表示通过扩展名来识别文件。
3. offset，只在type是`M`时才有效，表示读取文件头信息时需要从文件开头跳过多少个字节。
4. magic，文件头对应的二进制数据或者文件扩展名（扩展名不包含`.`），对于一些特殊数据比如`\0`，`\n`需要转换成`\x00`和`\x0a`。
5. mask，只在type是`M`时才生效。mask会和magic进行`&`位运算，运算结果会作为识别文件所用的依据。这是因为一部分文件的文件头特征数据是不连续的，比如HEIC文件文件头的前八个字节和第13到16字节的内容是固定的，但第9-12个字节可以是`HEIF`或者`HEIC`，我们可以用mask把第12个字节过滤掉，这样就不要写两条大致内容重复的规则了。
6. interpreter，负责执行这种文件的程序的绝对路径，当前准备执行的文件的路径或者描述符会作为第一个参数传给这个程序。
7. flags，控制程序执行行为的选项，注意这不是传给interpreter的。选项可以传递多个。

flags的常用选项有：

1. P，如果给了这个参数，加载器会在`argv[0]`之后添加一个可执行文件的完整路径。
2. O，默认情况下可执行文件的完整路径会作为命令行参数被传递给interpreter，启用这个选项后会打开可执行文件并把文件描述符通过auxv数组传递给interpreter。
3. C，新的进程不会从interpreter继承权限，比如setuid，权限会从待执行的可执行文件本身获取。
4. F，加载器会立即打开interpreter然后用fexecve/execveat执行程序，这通常被用在需要和容器交互的程序上。

只看文字描述可能有点抽象，我们看几个具体的例子：

第一个例子我们注册一条规则，使用python3执行扩展名为`.py3`的文件。

```console
$ echo ':py3:E::py3::/usr/bin/python3:' > /proc/sys/fs/binfmt_misc/register
$ ls /proc/sys/fs/binfmt_misc

py3  register  status

$ cat /proc/sys/fs/binfmt_misc/py3

enabled
interpreter /usr/bin/python3
flags:
extension .py3

$ echo 'print("hello binfmt_misc!")' > /tmp/test.py3
$ chmod +x /tmp/test.py3
$ /tmp/test.py3

hello binfmt_misc!
```

如果不注册规则就直接执行`.py3`文件，Linux会直接报错。

第二个例子是用文件头内容识别可执行文件，我们的可执行文件不会有文件扩展名，但会有`-- binfmt_lua\n`这样的文件头，文件内容是正常的lua脚本，但解释器我们会使用luajit：

```console
$ echo ':luajit.exec:M:3:binfmt_lua\x0a::/usr/bin/luajit:' > /proc/sys/fs/binfmt_misc/register
$ ls /proc/sys/fs/binfmt_misc

luajit.exec  py3  register  status

$ echo -e '-- binfmt_lua\nprint([[hello from luajit with binfmt_misc]])' > /tmp/testlua
$ chmod +x /tmp/testlua
$ /tmp/testlua

hello from luajit with binfmt_misc
```

在这个例子中对于非显示的ascii字符换行符，我们把它转换成了`\x0a`，并使用offset跳过了表示注释的三个字符`-- `。

看完两个例子我想大家应该掌握`binfmt_misc`的基本用法了。

`binfmt_misc`有几个小限制还需要注意：

1. proc的接口需要主动挂载才会出现，好在主流发行版都以及自动帮我们处理挂载了
2. 规则字符串总长度不能超过1920字节
3. 使用文件头探测文件时，offset+len(magic)不能超过128字节
4. interpreter长度不能超过127字节

当然，正常使用的情况下其实很难遇到这些限制。

## binfmt_misc的工作原理

工作原理其实很简单，整个调用链路是这样的：

```text
用户通过命令行或者GUI上点击准备允许文件A -->
程序加载器先判断文件是否是ELF或者是否有Shebang -->
都不符合则遍历binfmt_misc规则，根据每条规则检查文件内容 -->
找到第一条匹配的规则后，加载器修改命令行参数，把A传递给interpreter -->
加载器加载并运行interpreter
```

整个链路非常直观，而且你可以发现这个规则是允许递归的，也就是说interpreter也可以是用`binfmt_misc`注册的自定义可执行文件。不过实际生产中很少有人这么做，因为调用链越长越容易出问题，排查错误也会变得更困难。

对于传递给interpreter和实际可执行文件的参数会这么处理，我们可以写个小程序来看下：

```golang
import (
    "fmt"
    "os"
)

func main() {
    for i, arg := range os.Args {
        fmt.Printf("idx: %d, arg: %s\n", i, arg)
    }
}
```

编译这个程序并起名叫myinterp，然后注册规则`echo ':myinterp:E::myi::/home/apocelipes/myinterp:' > /proc/sys/fs/binfmt_misc/register`。

现在我们创建一个空的`test.myi`，然后运行：

```console
$ ./test.myi

idx: 0, arg: /home/apocelipes/myinterp
idx: 1, arg: ./test.myi

$ ./test.myi --test1 --test2

idx: 0, arg: /home/apocelipes/myinterp
idx: 1, arg: ./test.myi
idx: 2, arg: --test1
idx: 3, arg: --test2
```

可以看到我们的程序被调用了，被执行的可执行文件的路径会被作为`argv[1]`传入，其余的命令行选项会被依次传进来。

flags的`P`和`O`会对命令行选项产生影响，首先是`P`：

```console
$ test.myi --test1 --test2

idx: 0, arg: /home/apocelipes/myinterp
idx: 1, arg: /home/apocelipes/go/bin/test.myi
idx: 2, arg: test.myi
idx: 3, arg: --test1
idx: 4, arg: --test2
```

我们把`test.myi`移动到了`$PATH`中，这样就不需要指定完整路径了，现在在启用`P`标志时加载器会把被执行文件的完整路径添加在`argv[1]`的位置上。这是为了方便我们的解释器可以获取被执行文件的路径从而进行处理。

`O`的演示比较复杂，因为它并不会影响命令行参数，而是通过auxv传递打开的描述符，所以我们用c++重写解释器：

```c++
#include <iostream>
#include <cstdio>
#include <sys/auxv.h>
#include <unistd.h>

int main()
{
    std::cout << "pid: " << getpid() << "\n";
    unsigned long execfd = getauxval(AT_EXECFD);
    std::cout << "fd: " << execfd << "\n";

    auto file = fdopen(execfd, "r");
    if (file == nullptr) {
        std::perror("fdopen");
        return 1;
    }
    char buf[1024] = {0};
    std::fgets(buf, 1024, file);
    std::cout << "fd data: " << buf;
    if (std::fclose(file) != 0) {
        std::perror("fclose");
        return 1;
    }
}
```

我们用`fdopen`和`fclose`来检验收到的fd是否有效，并读取其内容：

```console
$ echo -1 > /proc/sys/fs/binfmt_misc/myinterp
$ echo ':myinterp:E::myi::/home/apocelipes/myinterp:O' > /proc/sys/fs/binfm
t_misc/register
$ echo 'test data' > /home/apocelipes/go/bin/test.myi
$ test.myi --test1 --test2

pid: 4821
fd: 3
fd data: test data
```

程序没有报错说明fd是有效的，读取到的内容也是我们之前写入的，值为3通常意味着这是进程中除了标准输入输出之外第一个打开的文件。

到此我想大家应该都了解`binfmt_misc`的工作原理了。不过在介绍应用之前，我还要先介绍一个和它很相似的东西——Shebang。

## Shebang

Shebang中文名又叫“释伴”，是写在脚本文件开头第一行的特殊指令，可以让操作系统调用特定的程序来执行这个脚本。它不光听着和binfmt_misc很像，其实Shebang的实现代码也在binfmt_misc里。不过两者终究只是有点像，具体行为上还是有区别的。

Shebang必须出现在脚本文件的开头，以`#!`开始，以换行符结束，具体格式是：`#![零个一个或多个空格]/path/to/interpreter 参数1 参数2 ...\n`。

程序加载器会找到路径指定的解释器，然后把脚本文件所在路径添加在其他参数之后传递给解释器。我们接着用前面golang写的小程序作为解释器，这回我们编写一个带有Shebang的脚本：

```bash
#! /home/apocelipes/myinterp --test1 --test2
echo hello
```
运行效果如下：

```console
$ chmod +x ./myscript
$ ./myscript

idx: 0, arg: /home/apocelipes/myinterp
idx: 1, arg: --test1 --test2
idx: 2, arg: ./myscript
```

可以看到所有参数合并成了一个，并作为第一个参数传递给了解释器，脚本路径则是最后一个参数。解释器的参数选项是可以省略的，这时候解释器之后收到一个参数也就是脚本所在路径。因此编写解释器的时候要根据参数数量自己处理命令行选项。

shebang总体上比binfmt_misc简单很多，也是日常工作中使用最多的。

一个把Shebang利用到极致的例子是字节跳动编写的ffmpeg rust绑定库里的脚本：

```rust
#!/bin/sh
#![allow(unused_attributes)] /*
OUT=/tmp/tmp && rustc "$0" -0 ${0UT} && exec ${OUT} $@ || exit $? #*/

use std::process::Command;
use std::io::Result;
use std::path::PathBuf;
use std::fs;

fn mkdir(dir_name: &str) →> Result<()> {
    fs::create_dir(dir_name)
}

fn main () {
    // 省略
}
```

这是合法的rust代码，rust编译器会忽略Shebang，其余的代码都是合法的rust代码或者注释。同时这也是合法的shell脚本，因为shell是解释执行的，在执行到第三行后程序要么exit退出执行要么exec切换到编译好的程序上了，尽管后面的rust内容都不是合法的shell代码，但只要不执行到它们脚本就不会报错。这是一个非常巧妙的利用Shebang把rust当脚本使用的例子。

当然，通过`binfmt_misc`这个例子可以进一步被简化，但Shebang的可移植性更强。

## binfmt_misc的应用

`binfmt_misc`的用处很多，比如前文提到的wine等模拟器会注册类似`:DOSWin:M::MZ::/usr/bin/wine:`的规则，让操作系统可以执行exe程序。Ubuntu也会注册`Python3.x`之类的规则，让python解释器去运行`.pyc`文件。

除此之外binfmt_misc还有一些妙用。比如可以让我们把`.go`代码文件当作脚本来运行。

首先我们写一个脚本编译通过命令行参数传入的代码生成可执行文件，然后再执行这个编译出来的程序：

```bash
#!/bin/bash
filename="/tmp/go-${RANDOM}.bin"
# $1 是传入的脚本所在路径，我们的注册规则需要使用P flag
go build -o "$filename" "$1"
# 跳过前两个参数，第一个参数是可执行文件路径，第二个参数是可执行文件在命令行里的名字，剩下的才是要传递给脚本的参数
"$filename" "${@:3}"
rm "$filename"
```

脚本起名叫`mygointerp`，然后我们给`.go`文件注册一条规则：`:golang-script:E::go::/home/apocelipes/mygointerp:P`。

最后写一个简单的go脚本：

```golang
import (
    "fmt"
    "os"
)

func main() {
    fmt.Println("script start")
    for i, arg := range os.Args {
        fmt.Printf("idx: %d, arg: %s\n", i, arg)
    }
    fmt.Println("script end")
}
```

运行：

```console
$ chmod +x goscript.go
$ ./goscript.go --test1 --test2

script start
idx: 0, arg: /tmp/go-21972.bin
idx: 1, arg: --test1
idx: 2, arg: --test2
script end
```

运行良好

我知道，大多数时候使用`go run`会更简单，这个例子只是用来说明编译型语言的代码文件也可以通过`binfmt_misc`机制像脚本一样方便地使用。

相比上一节提到的rust+Shebang的例子，这个利用`binfmt_misc`的例子可以让开发者专注于go代码本身，不需要在同一份源代码文件中兼顾两种不同的语言，缺点是需要额外的配置且可移植性不如Shebang。

## 总结

`binfmt_misc`机制提供了用户自定义可执行文件的能力，善加利用可有效提升生产力。

但如果滥用则会带来安全问题，病毒和木马会获得更多感染系统的机会。

最后如果规则注册太多，不仅排查问题会变得困难，还会拖慢程序加载执行的速度，所以凡事都有度，切不可滥用。

作为一个Zsh的扩展，可以`alias -s 扩展名=运行文件的程序`，直接输入文件名即可执行，文件也不需要有执行权限。
