在这篇文章中我们将初步体验对qss的使用。并对在goqt中使用qss时的注意事项进行说明。
那么事不宜迟，现在开始我们的qss之旅吧。

## QSS语法入门

qss是一种与css3相似的控制Qt组件的样式表，它有着与css3相似的语法，或者在某种意义上它可以说是对css3进行某些特化后的子集。

在日常开发中，Qt控件自身的外观有时很难满足我们的需要，这时候一般会有两种常见的解决方案，第一种是通过重写<code>paintEvent</code>来实现控件的自绘，这种方式最灵活，然而学习和使用成本也是最高的；另一种则是使用qss，通过qss控制widgets的外观表达。而qss的强大之处在于它不仅简单易学，而且功能强大，对自定义组件也能提供良好的支持。

下面我们就来简略学习一下qss的语法。
首先是选择器，和css3相同，qss需要用选择器来确定对哪些控件起作用：

### QSS选择器:

| 名称             | 形式/示例            | 说明                                                                       |
| ---------------- | -------------------- | -------------------------------------------------------------------------- |
| 万用选择器       | *                    | 顾名思义，选择所有的组件，包括自定义组件                                   |
| 类型选择器       | QLabel               | 选择所有和选择器指定类型相同的组件，如果是继承自该组件的类型则也会被选择   |
| 属性选择器       | QLabel[flat="false"] | 选择具有某一qss属性的指定类型的控件，指定继承类型的派生类的实例不会被选择  |
| class选择器      | .QLabel              | 与类型选择器相似，不过它不会选择派生类型的实例                             |
| ID选择器         | QLabel#name          | 通过setObjectName给控件的实例设置名字，通过这个名字选择该实例              |
| 子控件选择器     | QDialog QLabel       | 将选择父控件中的所有指定子控件                                             |
| 直接子控件选择器 | QDialog > QLabel     | 选择父控件的所有直接子控件，如果指定的子控件包含在其他子控件中将不会被选择 |

当需要同时使用多个选择器时，可以像这样`SelectorA, SelectorB, ...`。

### 状态选择器

虽然文档上这么称呼，实际上就和css3的伪类选择器差不多。它可以选择处于某种状态的控件。

| 名称           | 形式/示例                  | 说明                               |
| -------------- | -------------------------- | ---------------------------------- |
| 状态选择器     | QPushButton:hover          | 当控件处于指定的状态时会被选择     |
| 非状态选择器   | QPushButton:!hover         | 当控件不处于指定状态时会被选择     |
| 嵌套状态选择器 | QPushButton:hover:!checked | 当控件拥有全部指定的状态才会被选择 |

通过状态选择器，我们可以更细致的控制widgets。

### 局部选择器

它的真名叫Sub-Controls，意思是某个widget的某个局部组件，比如下拉框的下拉按钮，所以我们简称叫它局部选择器。

它的形式很简单：`QComboBox::drop-down`

通过两个冒号指定局部组件的名字，这些名字的Qt内置的。对局部选择器选中的内容修改属性将会影响局部组件的外观表现。

所有可用的局部组件名称都在Qt的文档中，有需要的可以进行查阅。

### 属性指定

看了那么多选择器，我们再来看看如何指定属性。

```css3
QLabel {
  font-size: 20px;
  color: #00ff00;
}

QMainWindow {
  background-image: url(bg.jpg);
}

QComboBox::down-arrow:pressed {
  position: relative;
  top: 1px;
  left: 1px;
}
```

可以看到，与css基本无异，值得注意的是定位相关属性只能用在局部组件上。

### 自定义组件和qss继承

首先明确一点，qss与css不同，子控件不会自动继承父控件的qss。所以想对子控件的样式进行调整就必须明确的用选择器选择子控件。如果你想让子控件从父控件继承qss属性，需要如下代码：

```C++
QCoreApplication::setAttribute(Qt::AA_UseStyleSheetPropagationInWidgetStyles, true);
```

对于我们继承各种widget而来的自定义控件，可以在选择器里像原生控件一样指定，例如：

```css3
MyWidget {
  font-size: 10px;
}

MyWidget:hover {
  backgorund: lightblue;
}
```

而在goqt中会有所不同，因为golang的继承是用“包含”做的一种模拟，你看起来是继承了某个widget，实际上只是在struct里包含了某个widget的实例而已，所以我们使用子控件的名字来定义qss是不起效果的，正确的做法如下：

```golang
type MyLabel struct {
  widgets.QLabel
  // some method
}

type MyWidget struct {
  widgets.QWidget
  // some method
}

// 错误做法
// myLabel.setStyleSheet("MyLabel{color:red;}")
// myWidget.setStyleSheet("MyWidget{backgorund:blue}")

// 正确做法
myLabel.setStyleSheet("QLabel{color:red;}")
myWidget.setStyleSheet("QWidget{backgorund:blue}")
```

不用担心这样做会引发冲突，因为如上文所提，每个widget的qss都是独立的，所以在不做特别设置时完全没有任何问题。

## 接下来学什么

现在我们已经学习了QSS的基础语法，相信聪明的你已经开始查阅文档或者动手实践了。

在下一篇文章中我们将会接触一个使用了QSS的简单组件的实现，你也可以先看看[源代码](https://github.com/apocelipes/schannel-qt5/blob/master/widgets/color_label.go)做一下预习。

如果觉得以上的代码对你有所帮助，也欢迎star我的项目：[https://github.com/apocelipes/schannel-qt5](https://github.com/apocelipes/schannel-qt5)

##### 参考

Qt Assistant: Qt Style Sheets

Qt Assistant: Qt Style Sheets Reference
