最近在更新系统的时候发现pacman的命令行界面变了，我有很久没更新过设备上的Linux系统了，所以啥时候变的不好说。但这一变化成功勾起了我的好奇心。新版的更新进度界面如下：

![multibar](../../images/linux/terminal-countdown/multi_progressbar.gif)

新的更新进度界面能同时显示多个进度条，而且并没有依靠ncurses这个传统的TUI库。为啥我能断定没有用ncurses呢，因为用过这个库的人都会发现程序在绘制界面的时候会用背景色清屏，且退出后终端的内容会恢复成运行程序前的样子，而上述表现都不存在。

不借助专用的库却又能绘制出比较生动的效果，这难道不吸引人吗？

所以带着好奇心，我简单探索了实现的原理，并且用相同的原理做了个新东西：

![countdown](../../images/linux/terminal-countdown/countdown.gif)

这是一个在终端中显示倒计时的小玩具，原理和pacman的进度条是一样的，我并没有一比一去复现pacman的效果，那样其实和对着范本写作文一样略显无聊，所以我选择活用知识做个新玩具。

好了，我们先来复习下单个终端命令行的进度条是怎么实现的。

单个进度条的原理其实很简单，几乎所有的终端和终端模拟器都支持一些特殊的控制字符，比如`\n`表示新加一个空白行并把光标移动到这个新行的最左侧也就是开头处；`\r`则是将光标移动到当前行的开头处。

所以单个进度条的绘制过程一共只要两步：

1. 根据进度计算出当前进度条的样子，然后用打印函数输出，注意不能输出换行符`\n`；
2. 输出`\r`让光标回到行首，等待一段时间，重复步骤1，新的输出内容会覆盖掉老的。
3. 进度到了100%之后就可以输出一个换行符`\n`结束进度条的打印了。

最关键的地方也只有一处，新的输出内容的长度要大于或者等于老内容，否则老内容会残留在终端里。

人眼的要求很低，所以你甚至可以不必做到每秒xx次刷新，只要在一秒或几秒里更新几次就能让人觉得你的进度条动起来了。

所以一个最简单的例子可以是这样的：

```golang
package main

import (
	"bytes"
	"fmt"
	"time"
)

const width = 50

func main() {
	bar := bytes.Repeat([]byte{' '}, width)
	fmt.Println()
	for i := range 50 {
		bar[i] = '='
		fmt.Printf("[%s] % 3d%%\r", bar, (i+1)*2)
		time.Sleep(100 * time.Millisecond)
	}
    fmt.Println()
    fmt.Println("end")
}
```

这是效果：

![singlebar](../../images/linux/terminal-countdown/singlebar.gif)

但`\r`有个缺点，它只能回溯当前行，而且这个“行”是以终端显示为准的——即使你的输出并没有包含换行符但它的长度超过了终端显示的宽度导致需要“折行”，那么新折行出来的那行在终端显示中会被认为是一个新行，`\r`只会将光标放到这个新行的开头。

其实我最开始想利用折行加`\r`字符实现多行进度条，但很快就发现这条路是走不通的。显然pacman并没有使用`\r`或者说它还利用了一些其他的东西。

看源代码是最快的，而且简单搜索一下“progressbar”很快就能找到答案。我就不卖关子了，pacman实现多行进度条效果是利用了[ASNI转义序列](https://zh.wikipedia.org/zh-cn/ANSI%E8%BD%AC%E4%B9%89%E5%BA%8F%E5%88%97)。

> ANSI转义序列（ANSI escape sequences）是一种带内信号的转义序列标准，用于控制视频文本终端上的光标位置、颜色和其他选项。在文本中嵌入确定的字节序列，大部分以ESC转义字符和"["字符开始，终端会把这些字节序列解释为相应的指令，而不是普通的字符编码。

简单的说，转义序列就像一些命令，可以控制光标和终端的各种行为。

具体格式是：`转义序列开始字符参数1;参数2;...;参数N命令`。我们最常见的转义序列是颜色控制，让终端里的文字变成红色：`\033[0;31m`。其中`\033[`是转义序列的开始标志，`0;31`是命令`m`的两个参数，参数之间用空格分隔，最后一个参数紧贴着命令。

转义序列的支持程度要看终端和终端模拟器，好消息是我们需要用到的转义序列的被广泛支持的，我们要用它们来在行与行之间移动光标并绘制内容。

转义序列支持光标上下左右移动还支持直接清除整行的内容，这使得我们可以将终端当成一个画布：每个字符的位置相当于画布上的一个像素点（因此使用等宽字体效果显示会更好），坐标原点是程序运行开始后光标所在的位置，根据这个原点可以简单构建出一个平面坐标系，我们可以用一些特殊字符模拟点和线来绘制简单的图形。

我们要用的转义序列是这些：

1. `\033[nF`，将光标向上移动n行
2. `\033[nE`，将光标向下移动n行
3. `\033[nC`，将光标向后（右）移动n个字符
4. `\033[2K`，清除光标所在行的整个内容（2以外的参数可以选择只清除光标前/后的内容）
5. 转义字符之间可以组合使用，比如`\033[nE\033[mC`表示光标先向下移动n行然后再向右移动m个字符。

现在你应该明白那个倒计时是怎么画出来的了，核心技术点就是找到个合适的数字asciiart，然后根据每秒更新的内容在正确的位置上用上面的转义序列像画像素点一样把数字和分隔符画出来就行了。

说说其实一句话的事情，但做起来还是比较麻烦的，因为转义序列用的都是相对坐标，稍微算错一点相对位置显示效果就整个完蛋了，我也是调试了三四回才做到正确绘制的：

```golang
func (ar *ASCIIArtCharRender) RenderContent(duration time.Duration) {
	if len(ar.chars) > 0 {
		ar.chars = ar.chars[:0]
	}
	ar.chars = char.ConvertToChars(duration, char.ASCIIArtChars, ar.chars)
	for i := 0; i < char.MaxASCIIArtCharHeight(); i++ {
		util.CursorEraseEntireLine()
		fmt.Print(ar.chars[0][i])
		fmt.Print(" ")
		fmt.Print(ar.chars[1][i])
		fmt.Print("  ")
		fmt.Print(char.ASCIIArtChars[char.ASCIIArtColonIdx][i])
		fmt.Print("  ")
		fmt.Print(ar.chars[2][i])
		fmt.Print(" ")
		fmt.Print(ar.chars[3][i])
		fmt.Print("  ")
		fmt.Print(char.ASCIIArtChars[char.ASCIIArtColonIdx][i])
		fmt.Print("  ")
		fmt.Print(ar.chars[4][i])
		fmt.Print(" ")
		fmt.Print(ar.chars[5][i])
		fmt.Print("\n")
	}
}

func (ar *ASCIIArtCharRender) RenderFlashing() {
	util.CursorDownForward(1, 3+len(ar.chars[0][0])+1+len(ar.chars[1][0]))
	fmt.Print(" ")
	util.CursorForward(3 + len(ar.chars[2][0]) + 1 + len(ar.chars[3][0]) + 3)
	fmt.Print(" ")
	util.CursorDownForward(1, 2+len(ar.chars[0][0])+1+len(ar.chars[1][0]))
	fmt.Print("   ")
	util.CursorForward(2 + len(ar.chars[2][0]) + 1 + len(ar.chars[3][0]) + 2)
	fmt.Print("   ")

	util.CursorDownForward(2, 3+len(ar.chars[0][0])+1+len(ar.chars[1][0]))
	fmt.Print(" ")
	util.CursorForward(3 + len(ar.chars[2][0]) + 1 + len(ar.chars[3][0]) + 3)
	fmt.Print(" ")
	util.CursorDownForward(1, 2+len(ar.chars[0][0])+1+len(ar.chars[1][0]))
	fmt.Print("   ")
	util.CursorForward(2 + len(ar.chars[2][0]) + 1 + len(ar.chars[3][0]) + 2)
	fmt.Print("   ")
	// move to bottom
	util.CursorDown(1)
}
```

第一个函数是绘制时间用的数字的，为了简单我已经提前把数字的asciiart保存进了二维数组并且做到了等高，这样画的时候只要知道需要什么数字就行，剩下的就是逐行输出“像素点”。

第二个函数是用来绘制电子时钟数字分隔符的闪烁效果的，这个看上去就更乱了，因为需要在终端画布上大范围移动。

所以会者不难，纯体力活。

完整的代码可以在这找到：<https://github.com/apocelipes/ascii-count-down>，欢迎各位大佬的改进或者功能增强。

## 总结

TUI还是挺有意思的，好玩能学到东西而且很能消磨无聊的时间。

另外我觉得在之间看源码对答案之前，可以先自己思考一下并动手做做试验比如像我那样最先异想天开用折行去实现多行进度条。这样虽然浪费了点时间，但可以加深自己对新知识的理解和记忆。
