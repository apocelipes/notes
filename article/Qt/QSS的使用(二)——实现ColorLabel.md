在[上一篇文章](https://www.cnblogs.com/apocelipes/p/9959763.html)中，我们已经了解了QSS的基础使用，现在我们将会看到一个简单的例子来加深对QSS的理解。

## 需求分析

我们想要在界面中让文本显示出指定的颜色，现在有几种方案：

1. 使用paintEvent手动计算文字大小和位置，然后绘制
2. 利用QLabel可以识别HTML标签的特性实现彩色文字
3. 利用QSS+QLabel实现彩色文字

我们逐一分析这三种方案的利弊。

首先是paintEvent的方案，这是三种方案中最灵活却也是最复杂的，通过重绘事件可以最大限度的发挥其灵活性，但对于字体大小的计算以及文字对齐的控制都需要自行处理，如此一来工程量不可谓不大，显然对于我们只是希望让文字显示出指定颜色的简单需求来说实现成本过高了。

其次是使用HTML标签的方案，这种方案的好处在于QLabel已经帮我们处理了文字的绘制和对齐，我们只需要将合适的HTML内容添加进去即可，虽然灵活性不如第一种，但是其简便性是显而易见的，例如：

```python
label = QLabel(r'<font color="#ff0000">Qt</font>')
label.show()
```

这将会显示一个红色的“Qt”文字。

然而该方案的缺点也是显而易见的，虽然QLabel识别了HTML标签并且不会明文显示，但它确是实际存在于label中的：

```python
print(label.text()) # '<font color="#ff0000">Qt</font>'
```

当我们需要设置/获取label中的文字时，就不得不想办法去除这个HTML标签，又或者当我们想要修改label的颜色时，就不得不对text做更复杂的处理，显然，我们的需求中也会包含以上类似的场景，所以这种看起来简单实际上暗含了复杂性的方案我也没有采用。事实上开发中经常会出现这种坑，所以在方案选择时三思而后行总是有好处的。

最后一种是QSS+QLabel的方案，也是目前我采用的方案。你可能已经猜到了，这种方案兼具QLabel的实用和QSS的简单，也不会在内部保存多余的信息，在牺牲部分灵活性的前提下是最简单也是最合适的解决方案，接下来我们就详细了解下这种方案的实现。

## ColorLabel的实现

所有的代码在[这里](https://github.com/apocelipes/schannel-qt5/blob/master/widgets/color_label.go)，具体使用可以在我的[项目](https://github.com/apocelipes/schannel-qt5)中看到。

首先是定义默认颜色和QSS模板，模板用于后续的颜色设置：

```golang
var (
    // 控制颜色的qss模板
    colorStyle = "QLabel{color:%s;}"
    // 默认颜色-黑色
    defaultStyle = "QLabel{color:black;}"
)
```

其实空字符串""就表示使用系统自带样式，然而我这里为了统一就选用了黑色。

ColorLabel组件的定义，继承自QLabel，并保存自己的样式：

```golang
// ColorLabel 使用QSS显示彩色文字
type ColorLabel struct {
    widgets.QLabel

    // color style sheet
    defaultColor string
}
```

接着是构造函数，函数的功能在注释中写的比较清楚了，关键在于它调用的两个方法：

```golang
// NewColorLabelWithColor 生成colorlabel，设置default color为color
// color为空则设置为黑色
// color可以是颜色对应的名字，例如"black", "green"
// 也可以是16进制的RGB值，例如 #ffffff, #ff08ff, #000000
func NewColorLabelWithColor(text, color string) *ColorLabel {
    l := NewColorLabel(nil, 0)

    l.SetDefaultColor(color)
    l.SetDefaultColorText(text)

    return l
}
```

`SetDefaultColor`用于给ColorLabel设置默认的颜色，而`SetDefaultColorText`则和`setText`槽一样，给label设置文字，并使用默认指定的颜色显示，通过这两个方法我们可以处理绝大部分的使用场景，现在来看看它们的实现：

```golang
// SetDefaultColor 设置defaultColor
// color为""时设置为黑色
// 不会改变现有text内容的颜色
func (l *ColorLabel) SetDefaultColor(color string) {
    if color == "" {
        l.defaultColor = defaultStyle
        return
    }

    l.defaultColor = fmt.Sprintf(colorStyle, color)
}

// SetDefaultColorText 设置新的text值，并使其显示设置的default color
func (l *ColorLabel) SetDefaultColorText(text string) {
    l.SetText(text)
    l.SetStyleSheet(l.defaultColor)
}
```

当color为空字符串时使用默认颜色，否则设置为color指定的颜色，color可以是颜色的名字/16进制值。值得一提的是，修改默认颜色并不会影响已经显示的文字，如果想改变已经显示的文字的颜色，需要使用`ChangeColor`。

`SetDefaultColorText`则先使用`SetText`设置文字，随后添加QSS，因为这中间间隔相当短所以先添加QSS还是后添加不会有明显可见的区别，而且事件循环也会尽量将两次调用产生的重绘事件合并。

有时候我们也需要中途修改ColorLabel的颜色，或者颜色和文字一起修改，这时上面的方法就满足不了我们了，比如在[这段代码](https://github.com/apocelipes/schannel-qt5/blob/master/widgets/ssr_switch_panel.go#L156)里。

所以我们也实现了这些功能：

```golang
// ChangeColor 改变现有text的颜色
// 并且设置defaultColor为新的颜色
// color为""时设置为defaultStyle
func (l *ColorLabel) ChangeColor(color string) {
    l.SetDefaultColor(color)
    text := l.Text()
    l.SetDefaultColorText(text)
}

// SetColorText 用color显示新的text
// color为""时显示defaultStyle
func (l *ColorLabel) SetColorText(text, color string) {
    var style string
    if color == "" {
        style = defaultStyle
    } else {
        style = fmt.Sprintf(colorStyle, color)
    }

    l.SetText(text)
    l.SetStyleSheet(style)
}
```

`ChangeColor`改变了已显示文字的颜色，并设置label默认的颜色为新的颜色。`SetColorText`则显示指定颜色的文字，不会影响label的默认设置。

有了这些方法，我们就能方便地设置文字的颜色了，而且因为我们继承自QLabel，所以可以使用QLabel提供的方法对文字的显示做更进一步的控制。

通过这个小例子，我们已经对QSS的实际使用有了较为具体的印象，在实际应用中QSS的强大功能将会为我们提供许多的便利，掌握QSS的使用会使你的开发技能更上一层楼。

欢迎大家提出意见，也欢迎大家关注[我的项目](https://github.com/apocelipes/schannel-qt5)。
