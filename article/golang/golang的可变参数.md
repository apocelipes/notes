在实际开发中，总有一些函数的参数个数是在编码过程中无法确定的，比如我们最常用的`fmt.Printf`和`fmt.Println`：

```golang
fmt.Printf("一共有%v行%v列\n", rows, cols)
fmt.Println("共计大小:", size)
```

当你需要实现类似的接口时，就需要我们的**可变参数**出场了。

## golang的可变参数

可变参数就是一个占位符，你可以将1个或者多个参数赋值给这个占位符，这样不管实际参数的数量是多少，都能交给可变参数来处理，我们看一下可变参数的声明：

```golang
func Printf(format string, a ...interface{}) (n int, err error)
func Println(a ...interface{}) (n int, err error)
```

可变参数使用`name ...Type`的形式声明在函数的参数列表中，而且需要是参数列表的最后一个参数，这点与其他语言类似；

可变参数在函数中将转换为对应的`[]Type`类型，所以我们可以像使用slice时一样来获取传给函数的参数们；

有一点值得注意，golang的可变参数不需要强制绑定参数的出现。

举个例子，我想在c语言中实现一个求和任意个整数的函数sum：

```c
int sum(int num, ...) {
    // todo
}
```

我们只有先指定至少一个固定的形参（num）才能使用`...`可变参数，在golang中是不需要这样做的：

```golang
func sum(nums ...int) int {
    //todo
}
```

这也是golang语法简洁的其中一个体现。

## 传递参数给...可变参数

传递参数给带有可变参数的函数有两种形式，第一种与通常的参数传递没有什么区别，拿上一节的sum举个例子：

```golang
sum(1, 2, 3)
sum(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
```

除了参数的个数是动态变化的之外和普通的函数调用是一致的。

第二种形式是使用`...运算符`以`变量...`的形式进行参数传递，这里的变量必须是与可变参数类型相同的slice，而不能是其他类型（没错，数组也不可以），看个例子：

```golang
numbers := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
sum(numbers...) // 和sum(1, 2, 3, 4, 5, 6, 7, 8, 9. 10)等价
```

这种形式最常用的地方是在内置函数append里：

```golang
result := []int{1, 3}
data := []int{5, 7, 9}
result = append(result, data...) // result == []int{1, 3, 5, 7, 9}
```

是不是和python的解包操作很像，没错，大部分情况下你可以把`...运算符`当做是golang的unpack操作，不过有几点不同还是要注意的：

第一，只能对slice类型使用`...运算符`：

```golang
arr := [...]int{1, 2, 3, 4, 5}
sum(arr...) // 编译无法通过
```

你会见到这样的报错信息：`cannot use arr (type [5]int) as type []int in argument to sum`

这是因为可变参数实际是个slice，`...运算符`是个语法糖，它把前面的slice直接复制给可变参数，而不是先解包成独立的n个参数再传递，这也是为什么我只说`...运算符`看起来像unpack的原因。

第二个需要注意的地方是不能把独立传参和`...运算符`混用，再看个例子：

```golang
slice := []int{2, 3, 4, 5}
sum(1, slice...) // 无法通过编译
```

 这次你会见到一个比较长的报错：

```
too many arguments in call to sum
    have (number, []int...)
    want (...int)
```

这是和前面所说的原因是一样的，`...运算符`将不定参数直接替换成了slice，这样就导致前一个独立给出的参数不再算入可变参数的范围内，使得函数的参数列表从`(...int)`变成了`(int, ...int)`，最终使得函数类型不匹配编译失败。

正确的做法也很简单，不要混合使用`...运算符`给可变参数传参即可。

第三，如果是把slice解包，golang不会创建新的slice，而是复用被解包的slice，将那个slice传递给函数。看个例子：

```golang
func handle(numbers ...int) int {
    // 先对收到的数字做些处理，注意这里直接修改了numbers
    for i := range numbers {
        numbers[i] *= 2
    }
    return sum(numbers...)
}

slice := []int{1,2,3,4}
fmt.Println(slice) // []int{1,2,3,4}
handle(slice...)
fmt.Println(slice) // []int{2,4,6,8}
```

可以看到`slice`被意外修改了。原因是在这种调用中，`slice`会被直接传给`numbers`，不会复制生成新的slice，所以导致函数内部的修改可以反映到函数之外。

想解决这种问题，一是让接受可变长参数的函数别去修改自己收到的参数，二是如果要修改，就先自己copy一份参数，然后修改copy出来的副本。

## 总结

读了这篇文章，再加上一些简单的联系，我相信你们一定也能掌握golang可变参数的使用。

##### 参考：

<https://go.dev/ref/spec#Passing_arguments_to_..._parameters>

<https://go.dev/doc/effective_go#append>
