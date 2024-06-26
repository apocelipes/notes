`[]byte(string1)` 只读的情况下1.6之后不会额外分配内存。（cmd/compile/internal/walk/convert.go:317），满足的情况比较少。

`string(bytes)` 只读情况下也不会额外分配内存，直接使用bytes作为数据源（`map[string(b)]`, `"a" + string(b) + "b"`）。由`runtime.slicebytetostringtmp`实现。

其他的转换是都要复制底层数据的。

`[]byte(str1 + str2)` 到1.22目前会先分配新的string存放加号的结果，然后再分配一个`[]byte`把string的结果复制进去。

最后一个模式的解决办法：builder+resize 或者 append，都只会有一次内存分配。

或者用`slices.Concat([]byte(string1), []byte(string2))`，也只有一次内存分配（1.21, 1.22实测）。

以后可能会有编译器层面的优化，这样产生的二进制代码会小一点：<https://github.com/golang/go/issues/62407>

string和`[]byte`的比较也有优化：

```go
var b1,b2 []byte
var s1 string
string(b1) == s1
string(b1) == string(b2)
```

上面的都不会额外分配内存，也就是说和字符串比较的时候`[]byte -> string`这样的转换会被优化，两者直接比较底层的数据，不需要进行类型转换。注意需要形式上满足比较运算符的调用，其他逻辑上等价的形式没有优化。
