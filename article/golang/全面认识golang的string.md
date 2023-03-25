string我们每天都在使用，可是对于string的细节问题你真的了解吗？

今天我们先以一个问题开篇。

你能猜到下面代码的输出吗？

```golang
package main

import (
    "fmt"
)

func main() {
    s := "测试"
    fmt.Println(s)
    fmt.Println(len(s))
    fmt.Println(s[0])
    for _, v := range s {
        fmt.Println(v)
    }
}
```

谜底揭晓：

```text
测试
6
230
27979
35797
```

是不是觉得很奇怪？明明是2个汉字，为啥长度是6？为啥s[0]是个数字，又为啥长度是6却只循环了两次，而且输出的也是数字？

别急，我们一个个地说明。

## string的真实长度

要知道string的长度，首先要知道string里到底存了什么，我们看下官方的文档：

```text
type string string
   string is the set of all strings of 8-bit bytes, conventionally but not
   necessarily representing UTF-8-encoded text. A string may be empty, but not
   nil. Values of string type are immutable.
```

是的，没看错，在string里存储的是字符按照utf8编码后的“8-bit bytes”二进制数据，再说得明确点，就是我们熟悉的byte类型：

```text
type byte = uint8
    byte is an alias for uint8 and is equivalent to uint8 in all ways. It is
    used, by convention, to distinguish byte values from 8-bit unsigned integer
    values.
```

我们都知道，utf8在表示中文时需要2个字节以上的空间，这里我们一个汉字是3字节，所以总长度就是我们直接用len得到的6。

## 从string中索引到的值

从string里使用索引值得到的数据也是byte类型的，所以才会输出数字，最好的证据在于此（最后还会有证明代码），还记得byte的文档吗：`type byte = uint8`

如果看不懂，没关系，这是golang的type alias语法，相当于给某个类型起了个别名，而不是创建了新类型，所以byte就是uint8。

所以，输出uint8类型的数据，那么自然会看到数字。

## range string时发生了什么？

那么range的情况呢，长度是，为什么只循环两次？

首先我们可以排除byte了，uint8怎么可能会有20000的值。

然后我们来看一下[官方文档](https://go.dev/doc/effective_go)，其中有这么一段：

> For strings, the range does more work for you, breaking out individual Unicode code points by parsing the UTF-8. Erroneous encodings consume one byte and produce the replacement rune U+FFFD.

有点长，大致意思就是range会把string里的byte重新转换成utf8字符，对于错误的编码就用一字节的占位符替代，这下清楚了，range实际上和如下代码基本等价：

```golang
for _, v := range []rune(s)
```

我们是字符串正好是2个utf8字符，所以循环输出两次。我们再看看看看rune的文档：

```text
type rune = int32
    rune is an alias for int32 and is equivalent to int32 in all ways. It is
    used, by convention, to distinguish character values from integer values.
```

`rune`是`int32`的别名，它的值是Unicode码点，所以当我们`println`时就看到了数字。

## 代码验证

虽然没什么必要，但我们还是可以通过代码不算太严谨地验证一下我们得到的结论，想获取变量的类型，使用reflect.TypeOf即可（无法获取别名，所以“不严谨”）：

```golang
package main

import (
    "fmt"
    "reflect"
)

func main() {
    s := "测试"
    fmt.Println("s type:", reflect.TypeOf(s))
    fmt.Println("s[index] type:", reflect.TypeOf(s[0]))
    for _, v := range s {
        fmt.Println("range value type:", reflect.TypeOf(v))
    }
}
```

输出：

```text
s type: string
s[index] type: uint8
range value type: int32
range value type: int32
```

与我们预想的一样，uint8是byte，int32是rune，虽然TypeOf无法输出类型别名，但我们还是可以粗略判断出它的类型名称。

通过这篇文章，我们已经对string类型有了全面的认知。
