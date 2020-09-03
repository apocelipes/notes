三元运算符广泛存在于其他语言中，比如：

python：

```python
val = trueValue if expr else falseValue
```

javascript：

```js
const val = expr ? trueValue : falseValue
```

c、c++：

```c++
const char *val = expr ? "trueValue" : "falseValue";
```

然而，被广泛支持的三目运算符在golang中却是不存在的！如果我们写出类似下面的代码：

```golang
val := expr ? "trueValue" : "falseValue"
```

那么编译器就该抱怨了：`invalid character U+003F '?'`。意思是golang中不存在`?`这个运算符，编译器不认识而且非字母数字下划线也不能用做变量名，自然也就当作是非法字符了。

然而这是为什么呢，其实官方给出了[解释](https://golang.org/doc/faq#Does_Go_have_a_ternary_form)，这里简单引用一下：

> The reason ?: is absent from Go is that the language's designers had seen the operation used too often to create impenetrably complex expressions. The if-else form, although longer, is unquestionably clearer. A language needs only one conditional control flow construct.

> golang中不存在?:运算符的原因是因为语言设计者已经预见到三元运算符经常被用来构建一些极其复杂的表达式。虽然使用if进行替代会让代码显得更长，但这毫无疑问可读性更强。一个语言只需要有一种条件判断结构就足够了。

毫无疑问，这是在golang“大道至简”的指导思想下的产物。

这段话其实没问题，因为某些三元运算符的使用场景确实会降低代码的可读性：

```js
const status = (type===1?(agagin===1?'再售':'已售'):'未售')

const word = (res.distance === 0) ? 'a'
    : (res.distance === 1 && res.difference > 3) ? 'b'
    : (res.distance === 2 && res.difference > 5 && String(res.key).length > 5) ? 'c'
    : 'd';
```

乍一看确实很复杂，至少第二个表达式不花个20秒细看可能没法理清控制流程（想象一下当缩进错位或是完全没有缩进的时候）。

如果把它们直接转化成if语句是这样的：

```js
let status = ''
if (type === 1) {
    if (again === 1) {
        status = '再售'
    } else {
        status = '已售'
    }
} else {
    status = '未售'
}

let word = ''
if (res.distance === 0) {
    word = 'a'
} else {
    if (res.distance === 1 && res.difference > 3) {
        word = 'b'
    } else {
        if (res.distance === 2 && res.difference > 5 && String(res.key).length > 5) {
            word = 'c'
        } else {
            word = 'd'
        }
    }
}
```

看起来并没有多少的改善，特别是例2，三层嵌套，不管是谁review到这段代码难免不会抱怨你几句。

然而事实上这些代码是可以简化的，考虑到三元运算符总是会给变量一个值，因此最后的`else`其实可以看作是变量的默认值，于是代码可以这么写：

```js
let status = '未售'
if (type === 1) {
    if (again === 1) {
        status = '再售'
    } else {
        status = '已售'
    }
}

let word = 'd'
if (res.distance === 0) {
    word = 'a'
} else {
    if (res.distance === 1 && res.difference > 3) {
        word = 'b'
    } else {
        if (res.distance === 2 && res.difference > 5 && String(res.key).length > 5) {
            word = 'c'
        }
    }
}
```

其次，对于例2，显然可以使用`else if`来清除嵌套结构：

```js
let word = 'd'
if (res.distance === 0) {
    word = 'a'
} else if (res.distance === 1 && res.difference > 3) {
    word = 'b'
} else if (res.distance === 2 && res.difference > 5 && String(res.key).length > 5) {
    word = 'c'
}
```

现在再来看，显然使用`if`语句的版本的可读性更高，逻辑也更清晰（通过去除嵌套）。

然而事实也不尽然。除了用三元运算符表达流程控制之外，事实上更常见更广泛的一个应用是如下这样的表达式：

```js
const val = expr ? trueValue : falseValue

const func = (age) => age > 18 ? '成年人' : '未成年人'
```

类似上述通过一个简短的条件表达式来确定变量的值，在开发中的出现频率是相当高的。这时三元运算符的意图更清晰，可读性也较if语句更高，特别是配合匿名函数（lambda表达式）使用可以极大简化我们的代码。

对此python的解决之道是之支持上述的简化版三元表达式，同时表达式不支持嵌套，达到了扬长避短的目的。不过代价是编译器的相关实现会复杂化。

而对于golang来说一个简单的能只通过单遍扫描即可完成ast构建的编译器是其保持急速的构建速度的秘诀之一，为了这样简单的功能增加编译器实现的复杂度是不可接受的。同时由于golang“大道至简”的哲学，能用现有语法结构解决的问题，自然不会再添加新的语法。

不过还是有办法的，虽然不推荐：

```golang
func If(cond bool, a, b interface{}) {
    if cond {
        return a
    }

    return b
}

age := 20
val := If(age > 18, "成年人", "未成年人").(string)
```

不推荐这么做是有几个原因：

1. 使用接口导致性能下降
2. 需要强制的类型断言
3. 不管三元表达式还是if语句，对于不会到达的分支是不会计算的，也就是惰性计算;而给函数传递参数时每一个表达式都会被计算

最后总结一下：

三元运算符的优点：

- 对于简短的表达式使用三元运算符表意更清晰，特别是在习惯了线性阅读三元运算符表达式之后
- 不需要中间状态（例如第一个例子中的let变量可以替换为const，代码更健壮），心智负担更低
- 没有中间状态也就意味着更少或完全没有副作用，代码更易跟踪和维护

但三元运算符也有明显的缺点：

- 对于复杂逻辑代码可读性较差（例如第一个例子中的status，需要在trueValue的位置进行进一步的条件判断时）
- 容易被滥用，很多人将其用于替代if语句或是简化复杂的if嵌套，这会导致上一条中所描述的结果
- 条件分支只能为表达式，不支持多条语句

所以这是一个见仁见智的问题，总之只能入乡随俗了。

##### 参考

<https://juejin.im/post/6844903561759850510>

<https://www.it-swarm.dev/zh/javascript/%E6%9B%BF%E4%BB%A3js%E4%B8%AD%E7%9A%84%E5%B5%8C%E5%A5%97%E4%B8%89%E5%85%83%E8%BF%90%E7%AE%97%E7%AC%A6/1055944752/>

<https://golang.org/doc/faq#Does_Go_have_a_ternary_form>
