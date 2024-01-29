[errgroup](https://pkg.go.dev/golang.org/x/sync/errgroup)想必稍有经验的golang程序员都应该听说过，实际项目中用过的也应该不在少数。它和`sync.WaitGroup`类似，都可以发起执行并等待一组协程直到所有协程运行结束。除此之外errgroup还可以在协程出错时取消当前的context，以及它还能控制可运行的协程的数量。

但在日常的代码review时我注意到了几个比较常见的问题，这些问题有的无伤大雅最多只会造成一些性能损失，有的则会导致资源泄露甚至是死锁崩溃。

这里对这些比较典型的误用做下记录。

## 多余的context嵌套

先说个不是很常见但我还是遇到过两三次的不太妥当的用法。

我们知道errgroup在协程返回错误的时候会取消掉创建时传入的context，这是为了能让同组的其他协程知道有错误发生应该尽快退出执行。

所以errgroup使用的context应该是派生于当前上下文的新的context，这样才不会让可能的取消操作影响到errgroup之外的范围。

因此第一个常见误用出现了：

```golang
func DoWork(ctx context.Context) {
    errCtx, cancel := context.WithCancel(ctx)
    defer cancel()
    group, errCtx := errgroup.WithContext(ctx)
    ...
}
```

误用在哪呢？答案是context会自动帮我们派生出新的context，除了需要设置超时一般不需要再次额外封装，看源代码：

```golang
// https://github.com/golang/sync/blob/master/errgroup/errgroup.go

// WithContext returns a new Group and an associated Context derived from ctx.
//
// The derived Context is canceled the first time a function passed to Go
// returns a non-nil error or the first time Wait returns, whichever occurs
// first.
func WithContext(ctx context.Context) (*Group, context.Context) {
	ctx, cancel := withCancelCause(ctx)
	return &Group{cancel: cancel}, ctx
}

// https://github.com/golang/sync/blob/master/errgroup/go120.go

func withCancelCause(parent context.Context) (context.Context, func(error)) {
	return context.WithCancelCause(parent)
}
```

多于的嵌套会浪费内存，以及会对性能带来负面影响，尤其是需要从context里取出某些value的时候，因为取value是对一层层嵌套的context递归查找的，嵌套层数越多查找就有可能越慢。

不过前面也说到了，有一种情况是允许的，那就是对整个errgroup所有的协程设置超时：

```golang
func DoWork(ctx context.Context) {
    errCtx, cancel := context.WithTimeout(ctx, 10 * time.Second)
    defer cancel()
    group, errCtx := errgroup.WithContext(ctx)
    ...
}
```

目前想设置超时只能这样做，所以这种算是特例。

## Wait返回的时机

第二种误用比第一种要常见些。主要是对errgroup的行为理解上有误解。

这种误解经常表现为：如果协程返回错误或者ctx的超时被触发，`Wait`方法就会立即返回。

**这并不是事实。**

先来看看`Wait`的文档怎么说的：

> Wait blocks until all function calls from the Go method have returned, then returns the first non-nil error (if any) from them.

`Wait`需要等到所有goroutine返回后它才会返回。哪怕超时了，context取消了也一样，需要先等所有协程退出。再来看代码：

```golang
// https://github.com/golang/sync/blob/master/errgroup/errgroup.go

func (g *Group) Wait() error {
	g.wg.Wait()
	if g.cancel != nil {
		g.cancel(g.err)
	}
	return g.err
}
```

可以看到确实需要先等所有协程返回。如果你观察比较敏锐的话，其实能发现errgroup会对协程做包装，会不会包装的代码里有什么办法提前中止协程的执行呢？还是来看代码：

```golang
// https://github.com/golang/sync/blob/master/errgroup/errgroup.go

func (g *Group) Go(f func() error) {
	// 检查当前协程是否可运行的代码，先忽略

	g.wg.Add(1)
	go func() {
		defer g.done()  // 重点在这

		if err := f(); err != nil {
			g.errOnce.Do(func() {
				g.err = err
				if g.cancel != nil {
					g.cancel(g.err)
				}
			})
		}
	}()
}
```

注意那个defer，这意味着done只有在包装的函数运行结束（在你自己的函数f运行完并设置了error以及取消了ctx之后）时才会执行。

如果你自己的函数里不检查超时和上下文是否被取消，那leak和卡死问题就要找上门来了，比如下面这样的：

```golang
func main() {
    errCtx, cancel := context.WithTimeout(context.Background(), 1 * time.Second)
    defer cancel()
    group, errCtx := errgroup.WithContext(errCtx)
    group.Go(func () error {
        time.Sleep(10 * time.Second)
        fmt.Println("running")
        return nil
    })
    group.Go(func () error {
        return errors.New("error")
    })
    fmt.Println(group.Wait())
}
```

猜猜运行结果和执行时间。答案是`running\nerror\n`，运行需要10秒以上。

这种误用也很好识别，只要传给`Go`方法的函数里没有好好处理`errCtx`，那多半是有问题的。

不过要说句公道话，`Go`的参数形式不符合一般使用context的惯例，`Wait`的行为和其他能自主取消线程执行的语言也不一样造成了误用，语言和接口设计得背一半锅不能全赖用它的程序员。

## SetLimit和死锁

这种就更常见了，尤其发生在把errgroup当成普通协程池用的时候。

先来我最爱的猜谜游戏，下面的代码运行结果是什么？

```golang
func main() {
    group, _ := errgroup.WithContext(context.Background())
    group.SetLimit(2) // 想法：只允许2个协程同时运行，但多个任务提交到“协程池”
    group.Go(func () error {
        fmt.Println("running 1")
        // 运行子任务
        group.Go(func () error {
            fmt.Println("sub running 1")
            return nil
        })
        group.Go(func () error {
            fmt.Println("sub running 2")
            return nil
        })
        return nil
    })
    group.Go(func () error {
        fmt.Println("running 2")
        // 运行子任务
        group.Go(func () error {
            fmt.Println("sub running 3")
            return nil
        })
        group.Go(func () error {
            fmt.Println("sub running 4")
            return nil
        })
        return nil
    })
    fmt.Println(group.Wait())
}
```

答案是会死锁panic。而且是100%触发。

我会详细的解释这是为什么，但在之前我要说一个重要的知识点：

**`SetLimit`设置的不是同时在运行的协程数量，而是设置errgroup内最多同时能持有多少个协程，errgroup持有的协程可以在运行也可以在等待运行。**

如果每个running的sub running只有一个，那么有小概率不会死锁，所以我特地每组创建了两个，原因没那么复杂，看来后面的解释之后可以自行推理。

下面来解释，首先看`SetLimit`的代码，一切是从这开始的：

```golang
// https://github.com/golang/sync/blob/master/errgroup/errgroup.go

// SetLimit limits the number of active goroutines in this group to at most n.
// A negative value indicates no limit.
//
// Any subsequent call to the Go method will block until it can add an active
// goroutine without exceeding the configured limit.
//
// The limit must not be modified while any goroutines in the group are active.
func (g *Group) SetLimit(n int) {
	if n < 0 {
		g.sem = nil
		return
	}
	if len(g.sem) != 0 {
		panic(fmt.Errorf("errgroup: modify limit while %v goroutines in the group are still active", len(g.sem)))
	}
	g.sem = make(chan token, n)
}
```

`g.sem`是`chan strcut{}`。做的事很简单，如果参数大于0就按参数初始化一个长度为n的chan给`g.sem`，小于0就清空`g.sem`。如果你经验比较丰富的话，已经可以看出来这是一个简单的`ticket pool`模式了，这个模式在grpc里也有应用。

`ticket pool`模式的原理是设置一个固定大小为n的空chan，然后协程要运行的时候向这个chan写入数据，协程运行结束的时候从chan里把写入的数据读出（可能会读到别人写进去的，但只要遵循这个写入读出的顺序就没问题）。如果chan的写入阻塞了，就说明已经有n个协程在运行了，新的协程需要等到有协程执行完并读出数据后才能继续执行；正常情况下读出操作不会被阻塞。这个是限制goroutine数量的最常见的手段之一。根据写入操作实在协程内部还是发起协程的调用者那里进行，这个模式还能分别控制“最大同时运行的goroutine数量”或“goroutine总数量”。其中`goroutine的总数量 = 在运行的goroutine数量 + 其他等待运行goroutine的数量`。

而errgroup属于后者。还记得`Go`的代码里我注释掉的那部分吧，现在可以看了：

```golang
// https://github.com/golang/sync/blob/master/errgroup/errgroup.go

func (g *Group) Go(f func() error) {
	if g.sem != nil {
		g.sem <- token{} // token是struct{}
	}

	g.wg.Add(1)
	go func() {
		defer g.done()

		if err := f(); err != nil {
			// 设置错误值
		}
	}()
}

func (g *Group) done() {
	if g.sem != nil {
		<-g.sem // 从ticket pool里读出
	}
	g.wg.Done()
}
```

进入`Go`的时候并没有启动协程，而是先检查`sem`，如果有设置limit，就需要按操作ticket pool的流程先写入数据。写入成功才会创建协程，协程运行结束后把数据读出。这样限制了errgroup最大可以持有的协程数量，因为超过数量限制会阻塞住不创建新的协程。

在`Go`完成sem的写入并执行go语句之前，errgroup并没有“持有”go语句创建的这个协程。协程运行结束并把sem的数据读出后，group将不会继续“持有”这个协程。

问题就出在写入那里。假设调度器是这样运行我们的猜谜代码的：

1. 先启动running 1的协程，sem空位有2个，正常运行，running 1运行结束后它写入的数据才会被读出
2. 接着启动running 2，sem还剩一个空位，没问题，running 2运行结束后它写入的数据才会被读出
3. running 2先被执行，于是准备创建sub running 3的协程
4. 这时sem没空位了，创建sub running 3的`Go`阻塞
5. 调度器发现running 2被阻塞了，于是让running 1执行（假设而已，多核处理器上很可能是同时运行的）
6. running 1输出后准备创建sub running 1的协程
7. sem还是满的，`Go`又阻塞了
8. 调度器发现running 1和running 2都阻塞了，于是只能让main goroutine执行（这里忽略runtime自己的协程，因为不影响死锁检测结果）
9. main阻塞在`Wait`上，所有其他协程执行完才能继续执行
10. 没有能继续运行下去的协程，全都阻塞了（注意是阻塞不是sleep），死锁检测发现这种情况，panic

我知道实际执行顺序肯定不一样，但死锁的原因一样的：因为之前的协程没有让出ticket pool，后面的子任务需要向pool写入，而前面占有pool的协程需要等子任务执行完才会让出pool。**这是一个典型的循环依赖导致的死锁，诱因是同一个errgroup的嵌套使用**。

是什么导致了你踩坑呢？最大的可能是文档里那个“active”。这个词太模糊了，你可以发现它即能代指running又能代指runnable，还能两个同时代指。这里因为下面还有一段话，所以可以根据上下文估摸着猜出active想代指的是所有被创建出来的协程不管它们在不在运行。但如果你只看了第一段话就先入为主放心大胆用的话，坑就来了。这样的词缺少足够的上下文时连母语者都会觉得有二义性，更何况我们这些作为第二语言甚至第三语言的人。

而errgroup选择限制goroutine总数量也是有原因的：只限制同时运行的goroutine的数量就没法限制协程的总数量，协程虽然很轻量，但还是要占用内存以及花费cpu资源来调度的，不受控制很可能会产生灾难性后果，比如一个不当心在循环里创建了数百万个协程导致严重的内存占用和调度压力，控制了总数量这类问题就可以避免。

幸运的是，这个误用也很好识别，**但凡有嵌套使用同一个errgroup的时候，就要警报大作了**。

更幸运的是，如果你没有嵌套调用，那么这个`SetLimit`不管设置成哪个数字，都能正常限制顶层的goroutine的数量（或者不做限制），它不能限制的是从顶层协程里嵌套调用派生出的子协程，只要不嵌套调用同一个group，什么问题的不会有。

前面两种误用都是该避免的，然而嵌套的errgroup虽然不多见但确实有用处，所以我也会提供写简单的解决方案以供参考。

第一种是设置一个足够的limit数值，聪明人应该发现了，如果把limit设置成希望group里同时存在的协程的总数量（顶层+所有嵌套派生的），问题就能避免。这没错，但我不推荐，两点原因：

1. 设置成总数后起不到限制同时运行的协程的数量，在go里控制同时运行的协程数量是个很麻烦的事，limit通常只能起到“上限”的作用，但如果上限设置大了就容易出现问题。比如你的系统只能同时运行3个协程，你还有别的任务占用了一个协程在运行，为了避免死锁你设置了limit为4，这时候资源抢占和协程调度延迟都会明显上升，出现这类情况你的系统就离崩溃只有一步之遥了。
2. 算这个数量很麻烦，上面的例子你可以很简单算出是4，如果我再套一层或者加上几个可以跳过`Go`调用的条件分支呢？而且limit设置多了是起不到限制goroutine数量的作用的，设少了会死锁。
3. limit多半是个写死的常量或者干脆是魔数，那么下次协程的逻辑改了这个数字多半得跟着改，如果你算错了或者忘记改了，那么你就惨了，死锁就像个地雷一样埋下了。

综上，你应该用第二种方法：永远不要嵌套使用同一个errgroup，真有嵌套需求也应该使用新的errgroup实例，这样可以避免死锁，也最符合当前需求的语义：

```golang
func main() {
    group, errCtx := errgroup.WithContext(context.Background())
    group.SetLimit(1) // 想法：只允许2个协程同时运行，但多个任务提交到“协程池”
    group.Go(func () error {
        fmt.Println("running 1")
        // 运行子任务
        // 新建一个errgroup，上下文使用外层group的
        subGroup, _ := errgroup.WithContext(errCtx)
        subGroup.SetLimit(1)
        subGroup.Go(func () error {
            fmt.Println("sub running 1")
            return nil
        })
        subGroup.Go(func () error {
            fmt.Println("sub running 2")
            return nil
        })
        fmt.Println(subGroup.Wait())
        return nil
    })
    group.Go(func () error {
        fmt.Println("running 2")
        // 运行子任务
        subGroup, _ := errgroup.WithContext(errCtx)
        subGroup.SetLimit(1)
        subGroup.Go(func () error {
            fmt.Println("sub running 3")
            return nil
        })
        subGroup.Go(func () error {
            fmt.Println("sub running 4")
            return nil
        })
        fmt.Println(subGroup.Wait())
        return nil
    })
    fmt.Println(group.Wait())
}
```

是的，现在所有limit设置成1也不会死锁。因为没有嵌套调用，因此也没有资源间的循环依赖了。

当然还有终极方案：别把errgroup当成协程池，如果你有复杂功能依赖于协程池找个功能全面的真正的协程池比如ants之类的用。

对了。你问`SetLimit`传0进去会发生什么，那当然是直接死锁了。这也符合语义，因为你的group里不能有任何协程，这时候再调`Go`当然是不对的，死锁panic也是应该的。所以传0进去导致死锁这不算坑，也算不上误用。

## 总结

总结下上面三个误用：

1. 传递有多余嵌套的context给errgroup
2. 在加入errgroup的协程里没有正确处理context取消和超时
3. 嵌套使用同一个errgroup

已有的静态分析工具不是很能识别这类问题，要么自己写个能识别的，要么只能靠review把关了。

比较大众的观点认为go简单易用，但实际上并不总是如此，有句话叫“Simple is not Easy”，go的使用者需要时刻为“大道至简”付出相应的代价。
