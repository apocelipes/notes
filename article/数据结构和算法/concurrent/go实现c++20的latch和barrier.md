latch就是一次性的waitgroup。

例子：

```c++
#include <iostream>
#include <thread>
#include <vector>
#include <latch>
#include <syncstream>

void worker(std::stop_token stopToken, std::latch& start_latch, std::latch& done_latch, int id) {
    // 等待所有线程准备好
    start_latch.wait();

    {
        // 使用 std::osyncstream 来同步输出
		if (stopToken.stop_requested()) {
            std::osyncstream(std::cout) << "Thread " << id << " is canceled.\n";
            goto out;
        }
        std::osyncstream(std::cout) << "Thread " << id << " is working...\n";
    }

    std::this_thread::sleep_for(std::chrono::milliseconds(100));

    {
		if (stopToken.stop_requested()) {
            std::osyncstream(std::cout) << "Thread " << id << " is canceled.\n";
            goto out;
        }
        std::osyncstream(std::cout) << "Thread " << id << " has finished.\n";
    }

out:
    // 通知任务完成
    done_latch.count_down(); // 相当于WaitGroup.Done
}

int main() {
    constexpr int num_threads = 5;
	// latch只能初始化时指定值
    std::latch start_latch(1); // 用于同步线程的启动，初始值设为1
    std::latch done_latch(num_threads);  // 用于同步线程的结束

    std::vector<std::jthread> threads;
    for (int i = 0; i < num_threads; ++i) {
        threads.emplace_back(worker, std::ref(start_latch), std::ref(done_latch), i + 1);
    }

    // 释放所有线程开始工作
    start_latch.count_down();

    // 主线程 sleep 102 毫秒
    std::this_thread::sleep_for(std::chrono::milliseconds(100));

    // 取消 id 为 1、2、3 的线程
    for (int i = 0; i < 3; ++i) {
        threads[i].request_stop();
    }
    
    // 等待所有线程完成工作
    done_latch.wait();
    
    std::cout << "All threads have finished.\n";
    return 0;
}

/* Possible output:

Thread 1 is working...
Thread 3 is working...
Thread 5 is working...
Thread 4 is working...
Thread 2 is working...
Thread 3 has finished.
Thread 4 has finished.
Thread 1 is canceled.
Thread 5 has finished.
Thread 2 is canceled.
All threads have finished.

*/
```

`std::latch`不能重复利用，重用是未定义行为，既然是未定义行为那么也没必要强求panic，放着不管也行。如果要禁止重用就得atomic+waitgroup了。


barrier类似waitgroup，但它有一个“轮次”的概念，上一轮的操作不应该影响到下一轮，所以不能直接用waitgroup，需要配合atomic做包装，waitgroup只是用来实现轻量级的批量唤醒，替换成chan也行。因为已经有atomic了，所以c++里规定的那些未定义行为都能识别到，因此全部panic。

```golang
func main() {
	const nG = 4
	b := NewBarrierWithCallback(nG, func() { fmt.Println("Barrier on") })
	wg := &sync.WaitGroup{}
	wg.Add(nG)
	fmt.Println("main start")
	for i := range nG {
		go func() {
			fmt.Printf("start: %d\n", i)
			b.ArriveAndWait()
			fmt.Printf("end: %d\n", i)
			b.ArriveAndWait()
			fmt.Printf("done: %d\n", i)
			wg.Done()
		}()
	}
	wg.Wait()
	fmt.Println("main end")
	fmt.Println("size of Barrier:", unsafe.Sizeof(Barrier{}))
}

type Barrier struct {
	arrived    atomic.Int64
	expect     atomic.Int64
	generation atomic.Pointer[sync.WaitGroup]
	callback   func()
}

func NewBarrier(n int64) *Barrier {
	return NewBarrierWithCallback(n, nil)
}

func NewBarrierWithCallback(n int64, callback func()) *Barrier {
	if n <= 0 {
		panic("wrong")
	}
	var ret Barrier
	ret.arrived.Store(n)
	ret.expect.Store(n)
	if callback == nil {
		ret.callback = func() {}
	} else {
		ret.callback = callback
	}
	ret.generation.Store(&sync.WaitGroup{})
	ret.generation.Load().Add(1)
	return &ret
}

type StopToken *sync.WaitGroup

func (b *Barrier) Arrive(n int64) StopToken {
	if n <= 0 {
		panic("wrong arrive")
	}
	wg := b.generation.Load()
	st := StopToken(wg)
	arrived := b.arrived.Add(-n)
	expect := b.expect.Load()
	if arrived < 0 || arrived > expect {
		panic("wrong number")
	}
	if arrived == 0 {
		// 这时本轮只有一个goroutine在操作，所以调用callback是安全的，多余的goroutine会触发panic
		b.callback()
		b.generation.Store(&sync.WaitGroup{})
		b.generation.Load().Add(1)
		b.arrived.Store(expect)
		wg.Done()
	}
	return st
}

func (b *Barrier) Wait(st StopToken) {
	wg := (*sync.WaitGroup)(st)
	wg.Wait()
}

func (b *Barrier) ArriveAndWait() {
	b.Wait(b.Arrive(1))
}

func (b *Barrier) ArriveAndDrop() {
	if b.expect.Add(-1) < 0 {
		panic("wrong expect")
	}
	b.Arrive(1)
}

// 可能的输出:
/*
main start
start: 3
start: 0
start: 1
start: 2
Barrier on
end: 2
end: 1
end: 3
end: 0
Barrier on
done: 0
done: 3
done: 2
done: 1
main end
size of Barrier: 32
*/
```

和waitgroup在用法上有很大区别，waitgroup主要用在一等多，而barrier主要是多等多（所有goroutine等待某个条件达成才继续执行）arrive_and_wait用的比较多。
