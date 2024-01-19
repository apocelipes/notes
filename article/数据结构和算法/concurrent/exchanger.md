用来在**两个**线程之间交换数据的数据结构。

Java里是支持超过两个线程交换的，但实现会引入额外复杂性，且会发生抢数据，最后需要的场景也不多，所以默认不支持>2的线程。超过两个要么undefined behavior要么panic。

行为：交换完成前exchange方法不会返回，或者没交换完成但出错了（系统错误，线程取消，超时等），只交换，绝不访问或修改待交换数据的内部（包括指针解引用）

交换数据流程：

1. 线程A将要交换给线程B的数据写入（CAS或者加锁也许），然后等待，这个过程叫offer
2. 线程B先获取A设置的数据，没有就等，拿到后再把自己要交换给A的数据写入，唤醒A，这个过程叫release
3. A被唤醒后拿走B写入的数据

java的实现略有不同：

- 两个线程交换的数据被组合成pair，两个field分为item（线程自己写入）和match（另一个线程写入后这边自己获取）
- CAS的时候直接把pair写进去
- java里全是引用，所以线程B把pair重新换成null之后修改pair的match，线程A也能看到
- 因为线程A不在执行，所以修改pair里的内容没有并发问题（只有B和A持有pair，且现在只有B能运行）
- A被唤醒后B不会再修改pair，所以A读取pair里的match是安全的
- 就算A再次进行交换，此时上一次交换的B还没退出，A只是重新赋值了item，只要B不直接返回node.item，提前复制一份引用，就没有race
- 好处是只需要一个原子变量，且match的写入和读取不需要同步措施

java里的pair看起来像这样：

```java
final class Node {
    ...
    int collides;           // 在当前bound下CAS失败的次数
    int hash;               // 伪随机数，用于自旋
    Object item;            // 这个线程的当前项，也就是需要交换的数据
    volatile Object match;  // 做releasing操作的线程传递的项
    volatile Thread parked; // 挂起时设置线程值，其他情况下为null
}
```

java实现交换流程伪代码：

```java
while (true) {
    if (slot is empty) { // offer
        node.item = 要交换的数据;
        if (can CAS slot from null to node) {
            写入成功，让出执行等待唤醒;
            // 被唤醒后返回线程B交换过来的数据
            return node.match;
        }
    } else if (can CAS slot from node to null) { // release
         // 将slot设置为空
        get the item in node;
        node.match = 要交换的数据;
        唤醒等待的线程;
    }
    // CAS retry
}
```

golang的实现。golang没有方便的唤醒特定协程的办法，只有：chan、条件变量。条件变量比较啰嗦，先用chan实现一版有等待和唤醒且有pair的。

```golang
type Node[T any] struct {
    item, match T // or *T, *T may be better
}

type Exchanger[T any] struct {
    // don't copy this
    slot atomic.Pointer[Node[T]]
    notify chan struct{}
}

func NewExchanger[T any]() *Exchanger[T] {
    // return an interface is better
    e := &Exchanger[T]{}
    // don't block the sender
    // and sender should done notify before waiter be notified
    // so use buffered chan will give us this happen-before promise
    e.notify = make(chan struct{}, 1)
}

func (e *Exchanger[T]) Exchange(value T) T {
    for {
        node := e.slot.Load()
        if node == nil {
            // 只在offer的时候创建node，因为release的时候用不着
            // 几乎不会多次创建，下次重试的时候多半是要走release的流程的
            // 真的运气不好重试后又到这也没办法，所以item和match最后是指针这样开销会很小
            n := &Node{}
            n.item = value
            if e.slot.CompareAndSwap(nil, n) {
                <- e.notify
                return n.match
            }
            // 大循环重试：
            // 因为可能slot已经被设置了，这时线程A从offer转变成release
            // 流程互相转换了也没关系，只要两边处理流程按顺序来就不会死锁或者race
        } else if e.slot.CompareAndSwap(node, nil) {
            ret := node.item
            node.match = value
            e.notify <- struct{}{}
            return ret
        }
    }
}
```

有一定危险性，只有一个线程调用exchange的时候会卡死自己，用select加超时可解决，但通常用不着。其次超过两个协程exchange的时候行为不可控。

不把两个交换数据放在同一个并发结构里，选用chan来存储，chan的内存模型符合需求：

```golang
type Exchanger[T any] struct {
    offer, release chan T
    offerID, releaseID atomic.Int64 // 标记自己是offer还是release
}

func NewExchanger[T any]() *Exchanger[T] {
    return &Exchanger[T]{
        offerID:   -1,
        releaseID: -1,
        offer:     make(chan T, 1), // 发送需要马上完成，尽量不等待
        release:   make(chan T, 1),
    }
}

func (e *Exchanger[T]) Exchange(value T) T {
    goid := getGoroutineID()

    // offer goroutine
    isOffer := e.offerID.CompareAndSwap(-1, goid)
    if !isOffer {
        isOffer = e.offerID.Load() == goid
    }
    if isOffer {
        e.offer <- value
        return <-e.release
    }

    // release goroutine
    isRelease := e.releaseID.CompareAndSwap(-1, goid)
    if !isRelease {
        isRelease = e.releaseID.Load() == goid
    }
    if isRelease {
        e.release <- value
        return <-e.offer // chan本身是加锁的队列，所以就算offer再次运行，本次结果也不会被影响
    }

    // other goroutine
    panic("sync: exchange called from neither offer nor release goroutine")
}
```

很简单，且绑定死在两个固定的线程上，但需要两个锁。做offer还是release不影响，反正发送和接收流程是一样的。风险和上一个一样，不过因为都是chan操作，所以可以简单得加上select做超时或者context取消。

同样的语义不用锁用cas自旋也许，缺点是浪费cpu，如果是c++倒是有atomic_wait之类的可以在自旋一段时间后挂起：

```golang
type Exchanger[T any] struct {
    offer, release     atomic.Value[T] // 不存在伪共享
    offerID, releaseID atomic.Int64 // 标记自己是offer还是release
}

func (e *Exchanger[T]) Exchange(value *T) T {
    goid := getGoroutineID()

    // offer goroutine
    isOffer := e.offerID.CompareAndSwap(-1, goid)
    if !isOffer {
        isOffer = e.offerID.Load() == goid
    }
    if isOffer {
        for !offer.CompareAndSwap(nil, value) {}
        var ret *T
        for ret = release.Load(); ret == nil {}
        release.Store(nil)
        return ret
    }

    // release goroutine
    isRelease := e.releaseID.CompareAndSwap(-1, goid)
    if !isRelease {
        isRelease = e.releaseID.Load() == goid
    }
    if isRelease {
        for !release.CompareAndSwap(nil, value) {}
        var ret *T
        for ret = offer.Load(); ret == nil {}
        offer.Store(nil) // 如果offer再次运行，会卡在CAS上（因为要是nil才交换），直到这句运行，所以安全
        // 更简单的作法：
        // for ret = offer.Swap(nil); ret == nil {}
        // swap会不停写原子变量，哪个性能更好得测试，我更倾向于swap，因为他把获取offer和重新置空原子化了
        return ret
    }

    // other goroutine
    panic("sync: exchange called from neither offer nor release goroutine")
}
```

风险一样，如果只有一个goroutine在交换，会卡住，尤其在另一边出错退出处理流程的时候。没什么干净的解决办法，加超时或者塞context进去，每次cas前检查一下是否被取消。

不想存goroutine id，那要么用slot的方法，要么把offer和release拆成单独的接口，但这样破坏了语义，使用上也不方便。

使用场景：两个线程交换buffer，一个只读数据然后处理读完后buffer交给另一个去重新填满，另一个只写数据写完交给处理的线程然后从那个线程里那空buffer重新填充数据。很典型的双缓冲。

```golang
func Handler(ex *Exchanger, buf []byte) {
    for {
        // first loop the buf is empty
        buf = ex.Exchange(buf)
        doSomethingUtilBufferAllCosumed(buf)
        // 或者初始就喂一个有数据的buf，这样处理顺序得变成
        // doSomethingUtilBufferAllCosumed(buf)
        // buf = ex.Exchange(buf)
        // 后者可以少交换一次，但都需要等待初始数据
    }
}

func DataPicker(ex *Exchanger, buf []byte) {
    for {
        // fill the buf first
        FillTheBuffer(buf)
        buf = ex.Exchange(buf)
    }
}
```
