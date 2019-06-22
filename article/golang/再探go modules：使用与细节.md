还有半个月go1.12就要发布了。这是首个将go modules纳入正式支持的稳定版本。

距离go modules随着go1.11正式面向广大开发者进行体验也已经过去了半年，这段时间go modules也发生了一些变化，借此机会我想再次深入探讨go modules的使用，同时对这个新生包管理方案做一些思考。

<blockquote id="bookmark">
  <h4>本文索引</h4>
  <ul>
    <li>
      <a href="#vcs-semver">版本控制和语义化版本</a>
      <ul>
        <li><a href="#vcs">控制包版本</a></li>
        <li>
          <a href="#semver">语义化版本</a>
          <ul>
            <li><a href="#semver-what">何谓语义化</a></li>
            <li><a href="#semver-why">为何使用语义化版本</a></li>
            <li><a href="#semver-how">语义化版本带来的影响</a></li>
            <li><a href="#semver-think">一点思考</a></li>
          </ul>
        </li>
      </ul>
    </li>
    <li>
      <a href="#replace">replace的限制</a>
      <ul>
        <li><a href="#replace-local">本地包替换</a></li>
        <li><a href="#replace-indirect">顶层依赖与间接依赖</a></li>
        <li><a href="#replace-limit">限制</a></li>
      </ul>
    </li>
    <li>
      <a href="#release">发布go modules</a>
      <ul>
        <li><a href="#release-gosum">go.sum不是锁文件</a></li>
        <li><a href="#release-vendor">使用vendor目录</a></li>
        <li><a href="#release-version">注意包的版本</a></li>
      </ul>
    </li>
    <li><a href="#summary">小结</a></li>
  </ul>
</blockquote>

<h2 id="vcs-semver">版本控制和语义化版本</h2>
包的版本控制总是一个包管理器绕不开的古老话题，自然对于我们的go modules也是这样。

我们将学习一种新的版本指定方式，然后深入地探讨一下golang官方推荐的`semver`即语义化版本。

<h3 id="vcs">控制包版本</h3>
在讨论go get进行包管理时我们曾经讨论过如何对包版本进行控制（[文章在此](https://www.cnblogs.com/apocelipes/p/9534885.html)），支持的格式如下：
```text
vX.Y.Z-pre.0.yyyymmddhhmmss-abcdefabcdef
vX.0.0-yyyymmddhhmmss-abcdefabcdef
vX.Y.(Z+1)-0.yyyymmddhhmmss-abcdefabcdef
vX.Y.Z
```
在go.mod文件中我们也需要这样指定，否则go mod无法正常工作，这带来了2个痛点：
1. 目标库需要打上符合要求的tag，如果tag不符合要求不排除日后出现兼容问题（目前来说只要正确指定tag就行，唯一的特殊情况在下一节介绍）
2. 如果目标库没有打上tag，那么就必须毫无差错的编写大串的版本信息，大大加重了使用者的负担

基于以上原因，现在可以直接使用commit的hash来指定版本，如下：

```text
# 使用go get时
go get github.com/mqu/go-notify@ef6f6f49

# 在go.mod中指定
module my-module

require (
  // other packages
  github.com/mqu/go-notify ef6f6f49
)
```

随后我们运行`go build`或`go mod tidy`，这两条命令会整理并更新go.mod文件，更新后的文件会是这样：

```text
module my-module

require (
    github.com/mattn/go-gtk v0.0.0-20181205025739-e9a6766929f6 // indirect
    github.com/mqu/go-notify v0.0.0-20130719194048-ef6f6f49d093
)
```

可以看到hash信息自动扩充成了符合要求的版本信息，今后可以依赖这一特性简化包版本的指定。

对于hash信息只有两个要求：

1. 指定hash信息时不要在前面加上`v`，只需要给出commit hash即可
2. hash至少需要8位，与git等工具不同，少于8位会导致go mod无法找到包的对应版本，推荐与go mod保持一致给出12位长度的hash

然而这和我们理想中的版本控制方式似乎还是有些出入，是不是觉得。。。有点不直观？接下来介绍的语义化版本也许能带来一些改观。

<h3 id="semver">语义化版本</h3>
golang官方推荐的最佳实践叫做`semver`，这是一个简称，写全了就是`Semantic Versioning`，也就是语义化版本。

<h4 id='semver-what'>何谓语义化</h4>
通俗地说，就是一种清晰可读的，明确反应版本信息的版本格式，更具体的规范在[这里](https://semver.org/lang/zh-CN/)。

如规范所言，形如`vX.Y.Z`的形式显然比一串hash更直观，所以golang的开发者才会把目光集中于此。

<h4 id="semver-why">为何使用语义化版本</h4>
`semver`简化版本指定的作用是显而易见的，然而仅此一条理由显然有点缺乏说服力，毕竟改进后的版本指定其实也不是那么麻烦，对吧？

那么为何要引入一套新的规范呢？

我想这可能与golang一贯重视工程化的哲学有关：
> 不要删除导出的名称，鼓励标记的复合文字等等。如果需要不同的功能，添加 新名称而不是更改旧名称。如果需要完整中断，请创建一个带有新导入路径的新包。 -go modules wiki

通过`semver`对版本进行严格的约束，可以最大程度地保证向后兼容以及避免“breaking changes”，而这些都是golang所追求的。两者一拍即合，所以go modules提供了语义化版本的支持。

<h4 id="semver-how">语义化版本带来的影响</h4>
如果你使用和发布的包没有版本tag或者处于1.x版本，那么你可能体会不到什么区别，因为go mod所支持的格式从始至终是遵循`semver`的，主要的区别体现在`v2.0.0`以及更高版本的包上。

> “如果旧软件包和新软件包具有相同的导入路径，则新软件包必须向后兼容旧软件包。” - go modules wiki

正如这句话所说，相同名字的对象应该向后兼容，然而按照语义化版本的约定，当出现`v2.0.0`的时候一定表示发生了重大变化，很可能无法保证向后兼容，这时候应该如何处理呢？

答案很简单，我们为包的导入路径的末尾附加版本信息即可，例如：

```text
module my-module/v2

require (
  some/pkg/v2 v2.0.0
  some/pkg/v2/mod1 v2.0.0
  my/pkg/v3 v3.0.1
)
```

格式总结为`pkgpath/vN`，其中`N`是大于1的主要版本号。在代码里导入时也需要附带上这个版本信息，如`import "some/pkg/v2"`。如此一来包的导入路径发生了变化，也不用担心名称相同的对象需要向后兼容的限制了，因为golang认为不同的导入路径意味着不同的包。

不过这里有几个例外可以不用参照这种写法：

1. 当使用`gopkg.in`格式时可以使用等价的`require gopkg.in/some/pkg.v2 v2.0.0`
2. 在版本信息后加上`+incompatible`就可以不需要指定`/vN`，例如：`require some/pkg v2.0.0+incompatible`
3. 使用go1.11时设置`GO111MODULE=off`将取消这种限制，当然go1.12里就不能这么干了

除此以外的情况如果直接使用v2+版本将会导致go mod报错。

v2+版本的包允许和其他不同大版本的包同时存在（前提是添加了`/vN`），它们将被当做不同的包来处理。

另外`/vN`并不会影响你的仓库，不需要创建一个v2对应的仓库，这只是go modules添加的一种附加信息而已。

当然如果你不想遵循这一规范或者需要兼容现有代码，那么指定`+incompatible`会是一个合理的选择。不过如其字面意思，go modules不推荐这种行为。

<h4 id="semver-think">一点思考</h4>
眼尖的读者可能已经发现了，`semver`很眼熟。

是的，`REST api`是它的最忠实用户，像`xxx.com/api/v2/xxx`的最佳实践我们恐怕都司空见惯了，所以golang才会要求v2+的包使用`pkg/v2`的形式。然而把`REST api`的最佳实践融合进包管理器设计，真的会是又一个最佳实践吗？

我觉得未必如此，一个显而易见的缺点就在于向后兼容上，主流的包管理器都只采用`semver`的子集，最大的原因在于如果只提供对版本的控制，而把先后兼容的责任交由开发者/用户相对于强行将无关的信息附加在包名上来说可能会造成一定的迷惑，但是这种做法可以最大限度的兼容现有代码，而golang则需要修改mod文件，修改引入路径，分散的修改往往导致潜在的缺陷，考虑到现有的golang生态这一做法显得不那么明智。同时将版本信息绑定进包名对于习惯了传统包管理器方案的用户（npm，pip）来说显得有些怪异，可能需要花上一些额外时间适应。

不过检验真理的标准永远都是实践，随着go1.12的发布我们最终会见分晓，对于go modules现在是给予耐心提出建议的阶段，评判还为时尚早。

<h2 id="replace">replace的限制</h2>
`go mod edit -replace`无疑是一个十分强大的命令，但强大的同时它的限制也非常多。

本部分你将看到两个例子，它们分别阐述了本地包替换的方法以及顶层依赖与间接依赖的区别，现在让我们进入第一个例子。

<h3 id="replace-local">本地包替换</h3>
replace除了可以将远程的包进行替换外，还可以将本地存在的modules替换成任意指定的名字。

假设我们有如下的项目：

```bash
tree my-mod

my-mod
├── go.mod
├── main.go
└── pkg
    ├── go.mod
    └── pkg.go
```

其中main.go负责调用`my/example/pkg`中的`Hello`函数打印一句“Hello”，`my/example/pkg`显然是个不存在的包，我们将用本地目录的`pkg`包替换它，这是main.go：

```golang
package main

import "my/example/pkg"

func main() {
    pkg.Hello()
}
```

我们的pkg.go相对来说很简单：

```golang
package pkg

import "fmt"

func Hello() {
    fmt.Println("Hello")
}
```

重点在于go.mod文件，虽然不推荐直接编辑mod文件，但在这个例子中与使用`go mod edit`的效果几乎没有区别，所以你可以尝试自己动手修改my-mod/go.mod：

```text
module my-mod

require my/example/pkg v0.0.0

replace my/example/pkg => ./pkg
```

至于pkg/go.mod，使用`go mod init`生成后不用做任何修改，它只是让我们的pkg成为一个module，因为replace的源和目标都只能是go modules。

因为被replace的包首先需要被require（wiki说本地替换不用指定，然而我试了报错），所以在my-mod/go.mod中我们需要先指定依赖的包，即使它并不存在。对于一个会被replace的包，如果是用本地的module进行替换，那么可以指定版本为`v0.0.0`(对于没有使用版本控制的包只能指定这个版本)，否则应该和替换包的指定版本一致。

再看`replace my/example/pkg => ./pkg`这句，与替换远程包时一样，只是将替换用的包名改为了本地module所在的绝对或相对路径。

一切准备就绪，我们运行`go build`，然后项目目录会变成这样：

```bash
tree my-mod

my-mod
├── go.mod
├── main.go
├── my-mod
└── pkg
    ├── go.mod
    └── pkg.go
```

那个叫my-mod的文件就是编译好的程序，我们运行它：

```bash
./my-mod
Hello
```

运行成功，`my/example/pkg`已经替换成了本地的`pkg`。

同时我们注意到，使用本地包进行替换时并不会生成go.sum所需的信息，所以go.sum文件也没有生成。

本地替换的价值在于它提供了一种使自动生成的代码进入go modules系统的途径，毕竟不管是go tools还是rpc工具，这些自动生成代码也是项目的一部分，如果不能纳入包管理器的管理范围想必会带来很大的麻烦。

<h3 id="replace-indirect">顶层依赖与间接依赖</h3>
如果你因为`golang.org/x/...`无法获取而使用replace进行替换，那么你肯定遇到过问题。明明已经replace的包为何还会去未替换的地址进行搜索和下载？

解释这个问题前先看一个go.mod的例子，这个项目使用的第三方模块使用了`golang.org/x/...`的包，但项目中没有直接引用它们：

```text
module schanclient

require (
    github.com/PuerkitoBio/goquery v1.4.1
    github.com/andybalholm/cascadia v1.0.0 // indirect
    github.com/chromedp/chromedp v0.1.2
    golang.org/x/net v0.0.0-20180824152047-4bcd98cce591 // indirect
)
```

注意`github.com/andybalholm/cascadia v1.0.0`和`golang.org/x/net v0.0.0-20180824152047-4bcd98cce591`后面的`// indirect`，它表示这是一个间接依赖。

间接依赖是指在当前module中没有直接import，而被当前module使用的第三方module引入的包，相对的顶层依赖就是在当前module中被直接import的包。如果二者规则发生冲突，那么顶层依赖的规则覆盖间接依赖。

在这里`golang.org/x/net`被`github.com/chromedp/chromedp`引入，但当前项目未直接import，所以是一个间接依赖，而`github.com/chromedp/chromedp`被直接引入和使用，所以它是一个顶层依赖。

而我们的replace命令只能管理顶层依赖，所以在这里你使用`replace golang.org/x/net => github.com/golang/net`是没用的，这就是为什么会出现go build时仍然去下载`golang.org/x/net`的原因。

那么如果我把`// indirect`去掉了，那么不就变成顶层依赖了吗？答案当然是不行。不管是直接编辑还是`go mod edit`修改，我们为go.mod添加的信息都只是对`go mod`的一种提示而已，当运行`go build`或是`go mod tidy`时golang会自动更新go.mod导致某些修改无效，简单来说<mark>一个包是顶层依赖还是间接依赖，取决于它在本module中是否被直接import，而不是在go.mod文件中是否包含`// indirect`注释</mark>。

<h3 id="replace-limit">限制</h3>
replace唯一的限制是它只能处理顶层依赖。

这样限制的原因也很好理解，因为对于包进行替换后，通常不能保证兼容性，对于一些使用了这个包的第三方module来说可能意味着潜在的缺陷，而允许顶层依赖的替换则意味着你对自己的项目有充足的自信不会因为replace引入问题，是可控的。相当符合golang的工程性原则。

也正如此replace的适用范围受到了相当的限制：

1. 可以使用本地包替换将生成代码纳入go modules的管理
2. 对于直接import的顶层依赖，可以替换不能正常访问的包或是过时的包
3. go modules下import不再支持使用相对路径导入包，例如`import "./mypkg"`，所以需要考虑replace

除此之外的replace暂时没有什么用处，当然以后如果有变动的话说不定可以发挥比现在更大的作用。

<h2 id="release">发布go modules</h3>
本部分将讨论如何发布你的modules到github等开源仓库以供他人使用，放心这是相对来说最轻松的一部分。

<h3 id="release-gosum">go.sum不是锁文件</h3>
也许你知道npm的package-lock.json的作用，它会记录所有库的准确版本，来源以及校验和，从而帮助开发者使用正确版本的包。通常我们发布时不会带上它，因为package.json已经够用，而package-lock.json的内容过于详细反而会对版本控制以及变更记录等带来负面影响。

如果看到go.sum文件的话，也许你会觉得它和package-lock.json一样也是一个锁文件，那就大错特错了。go.sum不是锁文件。

更准确地来说，go.sum是一个构建状态跟踪文件。它会记录当前module所有的顶层和间接依赖，以及这些依赖的校验和，从而提供一个可以100%复现的构建过程并对构建对象提供安全性的保证。

go.sum同时还会保留过去使用的包的版本信息，以便日后可能的版本回退，这一点也与普通的锁文件不同。所以go.sum并不是包管理器的锁文件。

因此我们应该把go.sum和go.mod一同添加进版本控制工具的跟踪列表，同时需要随着你的模块一起发布。如果你发布的模块中不包含此文件，使用者在构建时会报错，同时还可能出现安全风险（go.sum提供了安全性的校验）。

<h3 id="release-vendor">使用vendor目录</h3>
golang一直提供了工具选择上的自由性，如果你不喜欢go mod的缓存方式，你可以使用`go mod vendor`回到`godep`或`govendor`使用的`vendor`目录进行包管理的方式。

当然这个命令并不能让你从godep之类的工具迁移到go modules，它只是单纯地把go.sum中的所有依赖下载到vendor目录里，如果你用它迁移godep你会发现vendor目录里的包回合godep指定的产生相当大的差异，所以请务必不要这样做。

我们举第一部分中用到的项目做例子，使用`go mod vendor`之后项目结构是这样的：

```bash
tree my-module

my-module
├── go.mod
├── go.sum
├── main.go
└── vendor
    ├── github.com
    │   ├── mattn
    │   │   └── go-gtk
    │   │       └── glib
    │   │           ├── glib.go
    │   │           └── glib.go.h
    │   └── mqu
    │       └── go-notify
    │           ├── LICENSE
    │           ├── README
    │           └── notify.go
    └── modules.txt
```

可以看到依赖被放入了vendor目录。

接下来使用`go build -mod=vendor`来构建项目，因为在go modules模式下go build是屏蔽vendor机制的，所以需要特定参数重新开启vendor机制:

```bash
go build -mod=vendor
./my-module
a notify!
```

构建成功。当发布时也只需要和使用godep时一样将vendor目录带上即可。

<h3 id="release-version">注意包版本</h3>
其实这是第一部分的老生常谈，当你发布一个v2+版本的库时，需要进行以下操作：
1. 将`module my-module`改成`module my-module/v2`
2. 将源代码中使用了v2+版本包的import语句从`import "my-module"`改为`import "my-module/v2"`
3. 仔细检查你的代码中所有`my-module`包的版本是否统一，修改那些不兼容的问题
4. 在changelog中仔细列出所有breaking changes
5. 当然，如果你觉得前面四步过于繁琐，注明你的用户需要指定`+incompatible`是一个暂时性的解决方案。

注意以上几点的话发布go modules也就是一个轻松的工作了。

<h2 id="summary">小结</h2>
相比godep和vendor机制而言，go modules已经是向现代包管理器迈出的坚实一步，虽然还有不少僵硬甚至诡异的地方，但是个人还是推荐在go1.12发布后考虑逐步迁移到go modules，毕竟有官方的支持，相关issues的讨论也很活跃，不出意外应该是go包管理方案的最终答案，现在花上一些时间是值得的。

当然包管理是一个很大的话题，就算本文也只是讲解了其中的一二，以后我也许有时间会介绍更多go modules相关的内容。

总之go modules还是一个新兴事物，包管理器是一个需要不断在实践中完善的工具，如果你有建设性的想法请尽量向官方反馈。

go modules的官方wiki也上线一段时间了，这篇文件基本上是与其结合的查漏补缺，同时也夹杂了一些个人见解，所以难免有所错误疏漏，欢迎指正。

##### 参考

[go modules wiki](https://github.com/golang/go/wiki/Modules)
