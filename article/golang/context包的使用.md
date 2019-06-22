<blockquote id="bookmark">
  <h4>本文索引</h4>
  <ul>
    <li><a href="#problem">问题引入</a></li>
    <li><a href="#introduce">context包简介</a></li>
    <li><a href="#example">示例</a></li>
  </ul>
</blockquote>

<h2 id="problem">问题引入</h2>
goroutine为我们提供了轻量级的并发实现，作为golang最大的亮点之一更是备受推崇。

goroutine的简单固然有利于我们的开发，但简单总是有代价的，考虑如下例子：

```golang
func httpDo(req *http.Request, resp *http.Response) {
  for {
    select {
    case <-time.After(5 * time.Second):
      // 从req读取数据然后发送给resp

    // 其他的一些逻辑（如果有的话）
    }
  }
}

func startListener() {
  // start http listener
  for {
    req, resp := HTTPListener.Accept()
    go httpDo(req, resp)
  }
}
```

上面的例子中，goroutine`httpDo`每隔5秒读取一次请求数据并发送给响应链接，`startListener`则每收到一个请求就启动一个goroutine去处理，虽然是伪代码，不过你已经发现了这是golang处理请求等并发任务时的惯用模型。

看着不是很简单吗，简单而又强大。确实如此，但有一个小问题。假如我的`startListener`崩溃了或者需要重新启动，这时前面那些链接都需要断开重连，那么我们应该怎么停止那些goroutine呢？

答案是做不到。原因很简单，当我们使用`go func()`启动一个goroutine后，除了`channel`和`sync`包中的同步手段之外，我们没有任何可以控制goroutine的方法。简单的说，除非goroutine在函数体内return或者主goroutine终止运行，否则我们是不能通过外部手段干扰goroutine使其终止的。因此在上述例子中那些goroutine无法终止，这会造成goroutine leak。开头已经说过，goroutine足够轻量，通常对于一个函数体不是死循环的goroutine来说我们大可不必关心它的退出操作，然而对于例子中的goroutine来说它会持续运行下去，虽然每个goroutine只占用很少的资源，但如果数量足够大的话被浪费的资源是相当惊人的，而一个长时间运行的程序必然因为得不到释放的资源而出问题。更为致命的是这种leak的goroutine可能还会造成逻辑上的错误从而引发更严重的问题。

当然，一点简单的改造就可以避免问题，这也是goroutine的强大之处。前面我们提到`channel`等同步手段可以间接地控制goroutine，所以我们可以利用一个空`chan`来达到终止所有goroutine的目的：

```golang
func httpDo(req *http.Request, resp *http.Response, done <-chan struct{}) {
  for {
    select {
    case <-done:
      // 避免goroutine leak
      return
    case <-time.After(5 * time.Second):
      // 从req读取数据然后发送给resp

    // 其他的一些逻辑（如果有的话）
    }
  }
}

func startListener() {
  // start http listener

  done := make(chan struct{})
  defer close(done)
  for {
    req, resp := HTTPListener.Accept()
    go httpDo(req, resp, done)
  }
}
```

修改过的程序我们使用一个`chan struct{}`变量进行控制，当`startListener`退出时（无论正常结束还是panic）done都会关闭，关闭后的`chan`会返回对应类型0值，于是goroutine的select会收到done关闭的信号，随后跟着退出，goroutine leak被避免。

当然，这么做不够优雅，毕竟当`startListener`这样的函数增多后我们不得不每次都写大量重复的代码，这样会让开发变得乏味。

所以golang1.7引入了`context`包用来优雅地退出goroutine。

<h2 id="introduce">context包简介</h2>
golang为了实现优雅地退出goroutine，在1.7引入了`context`。虽然名字叫“上下文”(context)不过其实只是我们在上一节例子的包装。

`context.Context`是一个接口：

```golang
type Context interface {
    // 返回超时时间（duration加上创建context对象时的时间），如果已经超时ok为true
    // 返回的时间也可以是自己设置的time.Time
    Deadline() (deadline time.Time, ok bool)

    // done信号，和上一节的做法一样，这里进行了一些包装
    Done() <-chan struct{}

    // 如果Done未被关闭就返回nil。
    // 否则返回相应的错误，比如调用了cancel()会返回Canceled；超时会返回DeadlineExceeded
    Err() error

    // 可以给context设置一些值，使用方法和map类似，key需要支持==比较操作，value需要是并发安全的
    Value(key interface{}) interface{}
}
```

实现了Context接口的对象都是并发安全的（如果你自己实现了这个接口也必须确保并发安全）。

context的使用很简单，首先在需要产生goroutine的函数中创建一个context对象，然后将其作为goroutine的第一个参数传入，例如`go func(ctx context.Context) {} (ctx)`，如果在goroutine里还会运行新的goroutine，那么就继续传递这个context对象。

如此一来最初的那个context对象就被称为parent， 其余goroutine中的被称为关联context，通过这种关系我们就可以把相关的goroutine联系在一起。

对于一个作为parent的context对象来说它也必须基于一个parent来创建，所以context提供了两个创建空context的函数：

```golang
func Background() Context
func TODO() Context
```

两者都返回一个空context，一个context不会被取消（cancel），也不会超时。它们唯一的区别是`TODO`表示你的代码正在准备使用context但仍然需要一些调整，这回告诉静态代码分析工具`go vet`不汇报某些context的使用错误，而通常我们应该使用`Background`产生的context来创建我们自己的context对象。

有了parent之后就可以创建我们需要的context对象了，context包提供了三种context，分别是是普通context，超时context以及带值的context：

```golang
// 普通context，通常这样调用ctx, cancel := context.WithCancel(context.Background())
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

// 带超时的context，超时之后会自动close对象的Done，与调用CancelFunc的效果一样
// WithDeadline 明确地设置一个d指定的系统时钟时间，如果超过就触发超时
// WithTimeout 设置一个相对的超时时间，也就是deadline设为timeout加上当前的系统时间
// 因为两者事实上都依赖于系统时钟，所以可能存在微小的误差，所以官方不推荐把超时间隔设置得太小
// 通常这样调用ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)

// 带有值的context，没有CancelFunc，所以它只用于值的多goroutine传递和共享
// 通常这样调用ctx := context.WithValue(context.Background(), "key", myValue)
func WithValue(parent Context, key, val interface{}) Context
```

对于会返回`CancelFunc`的函数，我们必须要使用`defer cancel()`，否则静态检查例如`go vet`会报错，理由是因为如果不用defer来终止context的话不能避免goroutine leak，对于带有超时的context来说cancel还可以停止计时器释放对应的资源。另外多次调用cancel是无害的，所以及时一个context因为超时而被取消，你依然可以对其使用cancel。所以我们应该把cancel的调用放在defer语句中。

上面是在主goroutine中的处理，对于传入context的goroutine来说需要做一些结构上的改变：

```golang
func coroutine(ctx context.Context, data <-chan int) {
  // setup something
  for {
    select {
    case <-ctx.Done():
      // 一些清理操作
      return
    case i := <-data:
      go handle(ctx, i)
    }
  }
}
```

可以看见goroutine的主要逻辑结构需要由select包裹，首先检查本次任务有没有取消，没有取消或者超时就从chan里读取数据进行处理，如果需要启动其他goroutine就把ctx传递下去。

golang的初学者可能会对这段代码产生不少疑惑，但是等熟悉了goroutine+chan的使用后就会发现这只是对既有模型的微调，十分便于迁移和修改。

<h2 id="example">示例</h2>
虽然说了这么多，实际上还都是些很抽象的概念，所以这一节举几个例子辅助理解。

首先是使用超时context的例子，每个goroutine运行5秒，每隔一秒打印一段信息，5秒后终止运行：

```golang
func coroutine(ctx context.Context, duration time.Duration, id int, wg *sync.WaitGroup) {
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("goroutine %d finish\n", id)
            wg.Done()
            return
        case <-time.After(duration):
            fmt.Printf("message from goroutine %d\n", id)
        }
    }
}

func main() {
    wg := &sync.WaitGroup{}
    ctx, cancel := context.WithTimeout(context.Background(), 5 * time.Second)
    defer cancel()

    for i := 0; i < 5; i++ {
        wg.Add(1)
        go coroutine(ctx, 1 * time.Second, i, wg)
    }

    wg.Wait()
}
```

我们使用`WaitGroup`等待所有的goroutine执行完毕，在收到`<-ctx.Done()`的终止信号后使wg中需要等待的goroutine数量减一。因为context只负责取消goroutine，不负责等待goroutine运行，所以需要配合一点辅助手段。如果运行程序你会得到类似如下结果（不同环境运行结果可能不同）：

```text
message from goroutine 0
message from goroutine 2
message from goroutine 4
message from goroutine 3
message from goroutine 1
message from goroutine 2
message from goroutine 4
message from goroutine 0
message from goroutine 1
message from goroutine 3
message from goroutine 3
message from goroutine 0
message from goroutine 4
message from goroutine 2
message from goroutine 1
message from goroutine 0
message from goroutine 2
message from goroutine 4
message from goroutine 3
message from goroutine 1
goroutine 0 finish
goroutine 3 finish
goroutine 1 finish
goroutine 2 finish
goroutine 4 finish
```

上一个例子中示范了超时控制，下一个例子将会演示如何用普通context取消一个goroutine：

```golang
func main() {
    // gen是一个生成器，返回从1开始的递增数字直到自身被取消
    gen := func(ctx context.Context) <-chan int {
        dst := make(chan int)
        n := 1
        go func() {
            for {
                select {
                case <-ctx.Done():
                    return
                case dst <- n:
                    n++
                }
            }
        }()
        return dst
    }

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    for n := range gen(ctx) {
        fmt.Println(n)
        // 生成到5时终止生成器运行
        if n == 5 {
            break
        }
    }
}
```

运行结果将会输出1-5的数字，当生成5之后for循环终止，main退出前defer语句生效，终止goroutine的运行。

最后一个例子是如何在goroutine间共享变量的。

因为可能会被多个goroutine同时修改，所以我们的value必须保证并发安全，不过也可以换种思路，只要保证对value的操作是并发安全的就可以了：

```golang
func main() {
    var v int64
    wg := sync.WaitGroup{}
    ctx := context.WithValue(context.Background(), "myKey", &v)

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(ctx context.Context, key string) {
            // 取出来的是interface{}，需要先断言成我们需要的类型
            value := ctx.Value(key).(*int64)
            // 原子操作，并发安全
            atomic.AddInt64(value, 1)
            wg.Done()
        }(ctx, "myKey")
    }

    wg.Wait()
    // 类型断言成*int64然后解引用
    fmt.Println(*(ctx.Value("myKey").(*int64)))
}
```

运行结果会打印出10，因为有10个goroutine分别对v原子地加了一。

当然，引入类型断言后代码复杂度有所提升，但数据的共享却方便了，你可以基于带值的context为parent继续构建可以取消或超时的context，同时可以在其中分发数据而无需将其作为参数传递。

context包的使用就是这么简单，还有更多对于context的应用，这里就不一一列举了，希望各位读者在以后的开发中能够多加利用context包，写出健壮的更优雅的代码。

##### 参考

[context包官方文档](https://golang.org/pkg/context/)
[官方博客的介绍](https://blog.golang.org/context)
