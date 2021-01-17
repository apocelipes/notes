很久没写博客了，不得不说go语言爱好者周刊是个宝贝，本来想随便看看打发时间的，没想到一下子给了我久违的灵感。

[go语言爱好者周刊78期](https://zhuanlan.zhihu.com/p/344888294)出了一道非常有意思的题目。

我们来看看题目。先给出如下的代码：

```golang
package main

import (
    "fmt"
    "time"
)

func main() {
    ch1 := make(chan int)
    go fmt.Println(<-ch1)
    ch1 <- 5
    time.Sleep(1 * time.Second)
}
```

请问这串代码的输出是什么。

我最先想到的是5，毕竟代码很简单，反应比较快的话代码看完结果也就推断出来了。

然而题目给出的其中一个选项是输出死锁报错，这个选项引起了我的好奇，于是我运行了一下：

```bash
$ go run a.go

fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive]:
main.main()
        /tmp/a.go:10 +0x65
exit status 2
```

啊这。真的死锁了。那么我猜会不会和执行顺序有关呢？于是我写了个脚本运行1000次看看：

```bash
#!/bin/bash

for i in {0..1000}
do
    go run a.go &> /dev/null
    if [ $? -eq 0 ]
    then
        echo 'success!'
        break
    fi
done
```

结果自然是一次也没成功，即使你改成10000哪怕是1000000也是一样的。执行顺序带来的影响我们可以排除了。

如果你仔细观察的话，所有的报错也都是一样的：`goroutine 1 [chan receive]:`，在这里死锁了。

那么会不会是因为使用了无缓冲chan的原因呢？golang的内存模型规定了无缓冲chan的接受happens before发送操作，这会不会带来影响呢（其实仔细想想就很快排除了，happens before确定的是内存的可见性，而不是指令执行的时间顺序），所以我改了下代码：

```golang
func main() {
    ch1 := make(chan int, 100)
    go fmt.Println(<-ch1)
    ch1 <- 5
    time.Sleep(1 * time.Second)
}
```

这次我们使用了一个有容纳100个元素的buff的channel，然而结果还是没有一点改变。

到这里我的思路中断了。

不过我还有google啊，所以我用“golang channel deadlock”为关键词搜索了一下，然后发现了一些有意思的结果。

那就是所有的chan的死锁的代码基本都能抽象成下面的形式：

```golang
func main() {
    ch1 := make(chan int) // 是否有buff无影响
    _ = <-chan
    ch1 <- 5
}
```

这个代码毫无疑问是会死锁的，因为从chan接收值而chan里是空的会导致当前goroutine进入等待，而当前goroutine不能继续运行的话就永远没办法向chan里写入值，死锁就在这里产生了。

在仔细观察一下，你就会发现题目的代码和这很像：

```golang
func main() {
    ch1 := make(chan int)
    go fmt.Println(<-ch1)
    ch1 <- 5
    // sleep是为了main routine不会过早退出
}
```

答案只有一个，`<-ch1`发生在main goroutine里了。

为了佐证这一观点，我有查阅了golang language spec，关于[go语句](https://golang.org/ref/spec#Go_statements)有如下的描述：

> The function value and parameters are evaluated as usual in the calling goroutine, but unlike with a regular call, program execution does not wait for the invoked function to complete.

> 函数和它的参数会像通常那样在使用go语句的那个goroutine里被执行，但不像常规的函数调用，程序不会同步等待这个函数执行完毕。

如果在看看有关[求值](https://golang.org/ref/spec#Calls)的部分：

> calls f with arguments a1, a2, … an. Except for one special case, arguments must be single-valued expressions assignable to the parameter types of F and are evaluated before the function is called.

> 用参数a1, a2等调用函数f，出了一个特例之外他们都必须是单值表达式，并且在函数运行前被求值。

上面说的特例是方法调用，方法的receiver会用特定的位置传给method。

这样事情的来龙去脉就清晰明了了，我们来梳理一下。

假设我们在main goroutine里启动一个子goroutine叫b，那么实际上在main goroutine里发生的事情是这样的：

1. main goroutine执行到go语句
2. go语句发现后面的函数表达式需要传递参数
3. 于是被传递的参数在main goroutine里求值
4. 新的goroutine b被创建，刚求值的参数传递给需要执行的函数（假设叫f），f在goroutine b中开始执行
5. go语句结束，控制流程回到main goroutine

所以`go fmt.Println(<-ch1)`里的chan接收操作是在main goroutine里执行的，因此死锁是板上钉钉的事情。

如果改成下面这样，死锁就不会发生：

```golang
package main

import (
    "fmt"
    "time"
)

func main() {
    ch1 := make(chan int)
    go func() {
        fmt.Println(<-ch1)
    }()
    ch1 <- 5
    time.Sleep(1 * time.Second)
}
```

这是因为`<-ch1`这回货真价实地发生在了不同的goroutine里，死锁自然也不存在了。

这题很坏，坏就坏在`fmt.Println(...)`这样的形式容易让人迷惑，以为这个调用本身在新的goroutine里执行，然而真正在新goroutine里执行的却是`fmt.Println`内部的函数实现代码，而不是`fmt.Println(...)`这句，参数会在这之前就被求值。

那么这能让我们学到什么呢？答案是永远也不要写出题目里那样的代码，对于chan的操作应该确保是在和执行go语句的goroutine不同的routine中运行的。

不过万事不绝对，带buff的chan会有些例外，当然这些以后有机会再说吧:P
