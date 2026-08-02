ntsync是Linux内核在6.14引入的新功能，它可以在Linux模拟Windows的可等待对象操作。

这个功能主要由wine团队开发和维护，借此可以在Linux上提供性能更高的Windows接口模拟。

这次我们来简单了解一下ntsync，并编写几个简单的示例。

## 启用ntsync

ntsync被实现为内核模块，因此在使用前需要先确保它被内核加载并启用。

首先内核版本必须在6.14以上。

接着检查内核是否支持ntsync：

```console
zgrep CONFIG_NTSYNC /proc/config.gz || grep CONFIG_NTSYNC /boot/config-$(uname -r)
```

如果看到`CONFIG_NTSYNC=m`说明内核支持，`CONFIG_NTSYNC=y`说明不仅支持而且默认启用了ntsync。

下一步是看ntsync是否启用，`ls -l /dev/ntsync`应该看到一个权限是`crw-rw-rw-`的字符设备，如果没看到则需要运行下面的命令：

```console
sudo modprobe ntsync
```

执行后没有报错的话，相应的设备就已经创建好并挂载到/dev目录下了。

## ntsync的使用流程

程序通过打开`/dev/ntsync`来创建ntsync实例，每打开一次都会创建不同的实例（NT virtual machine），这些实例之间是完全隔离的。和其他的设备文件一样，当不再使用时需要用close关闭它。

ntsync目前支持三种对象：

1. semaphore信号量。可以设置信号量的最大值，和普通的信号量用法一样。支持原子地操作多个。
2. mutex互斥锁，和普通的互斥锁一样，支持同一线程递归加锁，支持原子地操作多个。
3. event，固定最大值为1的semaphore，和eventfd很像，也支持原子地操作多个。

程序和ntsync交互完全通过`ioctl`系统调用，ntsync实例收到相应请求后会在内核空间里处理各种同步操作，这些操作保证是原子的。

所以ntsync的使用并不复杂：

1. 先打开ntsync设备创建实例
2. 通过ntsync实例创建三种对象
3. 接着用`ioctl`请求实例或对象完成操作
4. 在实例和对象不再被需要时使用`close`关闭它。

复杂的同步原语处理和线程安全都是ntsync内部帮我们处理的，如果不准备扩展/修改ntsync，我们完全没必要深入关心这种细节。

因为不同实例是完全隔离的，所以从不同实例产生的信号量、锁之类的对象也不可以混用，想要得到并发安全，不同线程需要使用从相同实例中创建出的对象。

还有两个重要的概念需要了解：

1. 触发，每种对象都有很多种状态，当对象的状态变为某个指定的状态时，这个对象就被“触发”了
2. wait，等待对象被触发的操作，对象被触发后wait操作也可以原子地改变该对象的状态

当然这只是粗略的工作流程，下面我们看看每种对象具体怎么使用，先从最简单的event开始。

## event对象的使用

在代码中使用ntsync时需要包含`linux/ntsync.h`头文件，里面定义了`ioctl`使用的数据结构和宏。

每种对象都需要从ntsync实例创建，创建event对象需要给`ioctl`传递下面这个数据：

```c
struct ntsync_event_args {
     __u32 signaled;
     __u32 manual;
};
```

其中`signaled`字段控制创建出来的event是否已经触发，任何非零值都代表已触发；manual则控制event是否会自动重置，0表示event触发后自动重置，其他值则表示需要开发者手动重置。

我们通过ioctl来创建event对象：

```c
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/ntsync.h>

#include <stdlib.h>
#include <stdio.h>

int main()
{
    // 先创建ntsync实例
    int ntsync_fd = open("/dev/ntsync", O_RDWR|O_CLOEXEC);
    if (ntsync_fd < 0) {
        perror("open ntsync");
        exit(1);
    }

    // 创建event对象，默认未触发、自动重置
    // ioctl会返回event对象的文件描述符
    struct ntsync_event_args e_args = {
        .signaled = 0,
        .manual = 0,
    };
    int ntsync_event = ioctl(
        ntsync_fd,
        NTSYNC_IOC_CREATE_EVENT,
        &e_args
    );
    if (ntsync_event < 0) {
        perror("open event");
        exit(1);
    }

    // 不再使用后要关闭
    close(ntsync_event);
    close(ntsync_fd);
}
```

event对象的用法和eventfd很像但更灵活。

如果一个event已经触发，那么任何在它身上的wait操作都会立即返回，否则wait会持续阻塞到超时为止。

自动重置和手动重置时event的行为会有不同：自动重置时每次event被触发都只能唤醒一个等待的线程，后续的wait将阻塞到event重新被触发；手动重置则会唤醒所有等待线程并让后续任意次wait操作也立即返回，直到主动调用reset操作，就像一个开关。

因此自动触发更像单点通知，手动重置则像群发通知或者线程安全的状态开关。

触发event的代码比较简单，触发有两种：

```c
// 触发信号，第三个参数类型是int32_t*，用来获取event对象之前的触发状态，不能传NULL
ioctl(ntsync_event, NTSYNC_IOC_EVENT_SET, &prev);

// 唤醒所有等待线程，然后把event设置为未触发状态，第三个参数和上一种操作的一样
ioctl(ntsync_event, NTSYNC_IOC_EVENT_PULSE, &prev);
```

wait操作会稍微复杂一些，因为wait是三种对象共用的，wait需要额外的数据结构：

```c
struct ntsync_wait_args {
    __u64 timeout; // 超时，U64_MAX表示无限阻塞，单位是ns，如果有设置NTSYNC_WAIT_REALTIME，则使用挂钟时间判断是否超时，否则使用开机到现在的纳秒数，注意这个参数填的应该是超时时间点的时间戳，而不是时间长度
    __u64 objs;    // 指向ntsync创建的对象数组的指针，对象都必须是有效的
    __u32 count;   // objs里有多少个对象
    __u32 owner;   // mutex的所有者，objs里有mutex的时候必须传，其他情况下可以忽略
    __u32 index;   // index，因为wait可以等一组对象里的任意一个，因此这个参数是告诉你objs里哪个对象被触发的，如果是alert导致的中断返回，index会设置为count
    __u32 alert;   // 必须是0或者一个event对象的描述符，传event时如果该event被触发则wait会立即中断
    __u32 flags;   // 目前只支持0或者NTSYNC_WAIT_REALTIME，后者控制超时判断的依据
    __u32 pad;     // 为了内存对齐的填充，无意义
};
```

wait操作有两种，等待一组对象中的某个被触发，或者等待整组所有对象都处于触发状态：

```c
// 等待一组中的某一个
ioctl(ntsync_fd, NTSYNC_IOC_WAIT_ANY, &wait_args);
// 等待一组中的所有
ioctl(ntsync_fd, NTSYNC_IOC_WAIT_ALL, &wait_args);
```

学会基础操作后我们看看例子。首先是自动重置。例子中会有三个子线程wait同一个event，每次只有一个线程会被唤醒，线程会休眠10秒再次触发event：

```c
// 传递给子线程的结构体
typedef struct {
    const char *name;
    int dev_fd;       // /dev/ntsync 设备 fd
    int event_fd;     // NTSYNC Event 对象的 fd
} ThreadArgs;

void print_current_time(const char *thread_name, const char *msg) {
    time_t rawtime;
    struct tm *timeinfo;
    char buffer[80];

    time(&rawtime);
    timeinfo = localtime(&rawtime);
    strftime(buffer, sizeof(buffer), "%H:%M:%S", timeinfo);

    printf("[%s] 线程 %s %s\n", buffer, thread_name, msg);
}

// 子线程函数
void* thread_func(void *arg) {
    ThreadArgs *t_args = (ThreadArgs*)arg;

    // 配置 WAIT 参数
    uint32_t objs[1] = { (uint32_t)t_args->event_fd };
    struct ntsync_wait_args wait_args = {
        .objs = (uint64_t)(uintptr_t)objs,
        .count = 1,
        .owner = 0,
        .index = 0,
        .flags = NTSYNC_WAIT_REALTIME,
        .timeout = UINT64_MAX // 无限等待，直到获得事件
    };

    // 等待触发
    int ret = ioctl(t_args->dev_fd, NTSYNC_IOC_WAIT_ANY, &wait_args);
    if (ret != 0) {
        perror("NTSYNC_IOC_WAIT_ANY failed");
        return NULL;
    }

    // event 触发后，输出线程名称和当前时间
    print_current_time(t_args->name, "被触发");

    sleep(10);

    // 再次触发事件
    uint32_t prev_state = 0;
    if (ioctl(t_args->event_fd, NTSYNC_IOC_EVENT_SET, &prev_state) < 0) {
        perror("NTSYNC_IOC_EVENT_SET failed");
    }

    return NULL;
}

int main()
{
    struct ntsync_event_args e_args = {
        .signaled = 0,
        .manual = 0, // 自动重置
    };
    ... // 创建event的过程不再重复
    pthread_t threads[3];
    ThreadArgs t_args[3] = {
        { .name = "A", .dev_fd = ntsync_fd, .event_fd = ntsync_event },
        { .name = "B", .dev_fd = ntsync_fd, .event_fd = ntsync_event },
        { .name = "C", .dev_fd = ntsync_fd, .event_fd = ntsync_event }
    };

    for (int i = 0; i < 3; i++) {
        pthread_create(&threads[i], NULL, thread_func, &t_args[i]);
    }
    print_current_time("main", "开始执行");
    // 不应该用sleep来控制同步，这里不sleep也没问题，但sleep方便展示效果
    sleep(1);

    printf("--- [main] 首次触发 Event 信号 ---\n\n");
    uint32_t prev_state = 0;
    if (ioctl(ntsync_event, NTSYNC_IOC_EVENT_SET, &prev_state) < 0) {
        perror("main thread SET Event failed");
    }
    
    for (int i = 0; i < 3; i++) {
        pthread_join(threads[i], NULL);
    }

    printf("\n");
    print_current_time("main", "结束执行");
    printf("--- [main] 所有子线程运行完毕，准备退出 ---\n");

    ... // 关闭资源的部分也忽略了
}
```

编译运行会输出如下的内容：

```console
[09:31:33] 线程 main 开始执行
--- [main] 首次触发 Event 信号 ---

[09:31:34] 线程 A 被触发
[09:31:44] 线程 B 被触发
[09:31:54] 线程 C 被触发

[09:32:04] 线程 main 结束执行
--- [main] 所有子线程运行完毕，准备退出 ---
```

可以看到一次只有一个线程被唤醒。

如果我们改为手动重置模式，设置`e_args.manual = 1`，则效果完全不同：

```console
[09:37:45] 线程 main 开始执行
--- [main] 首次触发 Event 信号 ---

[09:37:46] 线程 C 被触发
[09:37:46] 线程 A 被触发
[09:37:46] 线程 B 被触发

[09:37:56] 线程 main 结束执行
--- [main] 所有子线程运行完毕，准备退出 ---
```

可以看到所有等待线程被同时唤醒了。

event对象还支持reset操作，可以把已触发且需要手动重置的event对象设置为未触发状态：

```c
// 第三个参数是出参，返回上一个状态，不可以传NULL
ioctl(ntsync_event, NTSYNC_IOC_EVENT_RESET, &prev);
```

event对象的另一个作用是用来中止wait操作，后面会详细介绍。

## semaphore对象的使用

信号量的创建和event很像：

```c
struct ntsync_sem_args {
     __u32 count;
     __u32 max;
};

struct ntsync_sem_args s_args = {
    .count = 0,
    .max = 2,
};
int ntsync_sem = ioctl(
    ntsync_fd,
    NTSYNC_IOC_CREATE_SEM,
    &s_args
);
```

信号量只要count大于0就认为是被触发的，wait操作会立即返回。

信号量最经典的就是PV操作，wait在这里是V，会将count减一，而`NTSYNC_IOC_SEM_RELEASE`则是P，将count加上特定的值：

```c
// delta是count需要加上的值，超过max限制会报错
// delta传0不会报错，但也不会有任何效果，不可以传NULL
ioctl(ntsync_sem, NTSYNC_IOC_SEM_RELEASE, &delta)
```

如果P操作时count是0，则RELEASE执行成功后delta个在等待该信号量的线程会被唤醒。

信号量的操作很简单，它的例子也很简单。例子会创建一个max为2的信号量，三个子线程会wait这个信号量，获得信号量的线程打印时间并睡眠10秒，醒来后执行release：

```c
typedef struct {
    const char *name;
    int dev_fd;       // /dev/ntsync 设备 fd
    int obj_fd;     // NTSYNC SEM 对象的 fd
} ThreadArgs;

void* thread_func(void *arg) {
    ThreadArgs *t_args = (ThreadArgs*)arg;

    // 配置 WAIT 参数
    uint32_t objs[1] = { (uint32_t)t_args->obj_fd };
    struct ntsync_wait_args wait_args = {
        .objs = (uint64_t)(uintptr_t)objs,
        .count = 1,
        .owner = 0,
        .index = 0,
        .flags = NTSYNC_WAIT_REALTIME,
        .timeout = UINT64_MAX // 无限等待，直到获得事件
    };

    // 等待触发
    int ret = ioctl(t_args->dev_fd, NTSYNC_IOC_WAIT_ANY, &wait_args);
    if (ret != 0) {
        perror("NTSYNC_IOC_WAIT_ANY failed");
        return NULL;
    }

    print_current_time(t_args->name, "被触发");

    sleep(10);

    // RELEASE
    uint32_t delta = 1;
    if (ioctl(t_args->obj_fd, NTSYNC_IOC_SEM_RELEASE, &delta) < 0) {
        perror("NTSYNC_IOC_SEM_RELEASE failed");
    }

    return NULL;
}

int main()
{
    // 先创建ntsync实例
    int ntsync_fd = open("/dev/ntsync", O_RDWR|O_CLOEXEC);
    if (ntsync_fd < 0) {
        perror("open ntsync");
        exit(1);
    }

    // 创建信号量，最多只允许两个线程触发
    struct ntsync_sem_args s_args = {
        .count = 2,
        .max = 2,
    };

    int ntsync_sem = ioctl(
        ntsync_fd,
        NTSYNC_IOC_CREATE_SEM,
        &s_args
    );
    if (ntsync_sem < 0) {
        perror("open sem");
        exit(1);
    }

    pthread_t threads[3];
    ThreadArgs t_args[3] = {
        { .name = "A", .dev_fd = ntsync_fd, .obj_fd = ntsync_sem },
        { .name = "B", .dev_fd = ntsync_fd, .obj_fd = ntsync_sem },
        { .name = "C", .dev_fd = ntsync_fd, .obj_fd = ntsync_sem }
    };

    for (int i = 0; i < 3; i++) {
        pthread_create(&threads[i], NULL, thread_func, &t_args[i]);
    }
    // 这句输出的顺序是不确定的
    print_current_time("main", "开始执行");
    
    for (int i = 0; i < 3; i++) {
        pthread_join(threads[i], NULL);
    }

    printf("\n");
    print_current_time("main", "结束执行");

    // close
}
```

编译执行：

```console
[09:19:11] 线程 A 被触发
[09:19:11] 线程 main 开始执行
[09:19:11] 线程 C 被触发
[09:19:21] 线程 B 被触发

[09:19:31] 线程 main 结束执行
```

执行可以看到A和C先执行，B在等了10秒之后才执行。

## mutex对象的使用

到现在为止我们都只等待了一个对象，而ntsync的真正威力在于可以等待多个对象被触发。

这个特性在互斥锁这里尤其有用，因为不管是pthread和futex，想要加多把锁都得严格遵守特定的加锁顺序，否则就会造成死锁，而ntsync直接在内核层面通过一个ioctl调用处理掉了，没有死锁的风险。

创建mutex和创建其他对象一样：

```c
struct ntsync_mutex_args {
     __u32 owner; // 锁的持有者，0表示没有持有者
     __u32 count; // 递归计数，超过1表示持有者递归加锁
};

struct ntsync_mutex_args m_args = {
    .owner = 0, // 通常是gettid
    .count = 0,
};
int ntsync_mutex = ioctl(
    ntsync_fd,
    NTSYNC_IOC_CREATE_MUTEX,
    &m_args
);
```

mutex有所有者（owner）的概念，只有所有者才能执行解锁操作。owner的id不需要是真实的线程tid，任何正整数都可以。

mutex的owner为0的时候视为被触发，wait操作会返回，并把args中的owner参数设置给mutex。mutex被触发是所有等待线程中只有一个会被触发，这点和我们常用的互斥锁是一样的。

如果mutex已经有owner，且同一个owner再次执行wait，这会递归加锁，wait会立即返回，count会加1。

mutex支持两种解锁操作：

```c
struct ntsync_mutex_args m_args = {
    .owner = gettid(), // mutex的持有者
    .count = 0, // 出参，内核会把解锁操作前的count值写进去
};
// 普通解锁，递归存在时只解锁一层，count减一
ioctl(ntsync_mutex, NTSYNC_IOC_MUTEX_UNLOCK, &m_args);

// 不管是不是递归了，立即解锁
uint32_t owner_id = gettid();
ioctl(ntsync_mutex, NTSYNC_IOC_MUTEX_KILL, &owner_id);
```

只有owner才能解锁，传其他值进去都会报错。`NTSYNC_IOC_MUTEX_KILL`可以立即解锁递归的mutex并触发，但会导致wait操作会返回错误并把errno设置成`EOWNERDEAD`，这种情况下ioctl调用是算做成功的，新owner也会被正常设置，所以判断操作是否出错的时候一定要注意。

最后我们看个一次加多个锁的例子。A、B、C三个线程分别解锁三个不同的锁并睡眠10秒、20秒、30秒，主线程则在创建子线程后立即对三个锁加锁，主线程只有在拿到所有的锁之后才能继续运行：

```c
typedef struct {
    const char *name;
    int obj_fd;
    int sleep_time;
    uint32_t owner_id;
} ThreadArgs;

void* thread_func(void *arg) {
    ThreadArgs *t_args = (ThreadArgs*)arg;

    print_current_time(t_args->name, "开始运行");

    sleep(t_args->sleep_time);

    // 解锁
    struct ntsync_mutex_args m_args = {
        .owner = t_args->owner_id,
        .count = 0,
    };
    if (ioctl(t_args->obj_fd, NTSYNC_IOC_MUTEX_UNLOCK, &m_args) < 0) {
        perror("NTSYNC_IOC_MUTEX_UNLOCK failed");
    }

    return NULL;
}

int main()
{
    int ntsync_fd = open("/dev/ntsync", O_RDWR|O_CLOEXEC);
    if (ntsync_fd < 0) {
        perror("open ntsync");
        exit(1);
    }

    struct ntsync_mutex_args m_args_A = {
        .owner = 123,
        .count = 1, // 在设置了owner时count必须为非0值
    };
    int ntsync_mutex_A = ioctl(
        ntsync_fd,
        NTSYNC_IOC_CREATE_MUTEX,
        &m_args_A
    );
    if (ntsync_mutex_A < 0) {
        perror("open mutex_A");
        exit(1);
    }

    struct ntsync_mutex_args m_args_B = {
        .owner = 234,
        .count = 1,
    };
    int ntsync_mutex_B = ioctl(
        ntsync_fd,
        NTSYNC_IOC_CREATE_MUTEX,
        &m_args_B
    );
    if (ntsync_mutex_B < 0) {
        perror("open mutex_B");
        exit(1);
    }

    struct ntsync_mutex_args m_args_C = {
        .owner = 345,
        .count = 1,
    };
    int ntsync_mutex_C = ioctl(
        ntsync_fd,
        NTSYNC_IOC_CREATE_MUTEX,
        &m_args_C
    );
    if (ntsync_mutex_C < 0) {
        perror("open mutex_C");
        exit(1);
    }

    pthread_t threads[3];
    ThreadArgs t_args[3] = {
        { .name = "A", .obj_fd = ntsync_mutex_A, .sleep_time = 10, .owner_id = 123 },
        { .name = "B", .obj_fd = ntsync_mutex_B, .sleep_time = 20, .owner_id = 234 },
        { .name = "C", .obj_fd = ntsync_mutex_C, .sleep_time = 30, .owner_id = 345 }
    };

    for (int i = 0; i < 3; i++) {
        pthread_create(&threads[i], NULL, thread_func, &t_args[i]);
    }

    uint32_t objs[] = { (uint32_t)ntsync_mutex_A, (uint32_t)ntsync_mutex_B, (uint32_t)ntsync_mutex_C };
    struct ntsync_wait_args wait_args = {
        .objs = (uint64_t)(uintptr_t)objs,
        .count = 3,
        .owner = gettid(), // 设置主线程自己的id
        .index = 0,
        .flags = NTSYNC_WAIT_REALTIME,
        .timeout = UINT64_MAX
    };

    // 等待所有锁都拿到，这边需要等至少30秒
    int ret = ioctl(ntsync_fd, NTSYNC_IOC_WAIT_ALL, &wait_args);
    if (ret != 0) {
        perror("NTSYNC_IOC_WAIT_ALL failed");
        exit(1);
    }

    print_current_time("main", "开始执行");
    
    for (int i = 0; i < 3; i++) {
        pthread_join(threads[i], NULL);
    }

    print_current_time("main", "结束执行");

    // close 释放资源
}
```

`NTSYNC_IOC_WAIT_ALL`和any差不多，但会等待objs里所有对象都触发才返回，另一点不同是args里的index在成功返回时永远设置为0。

我们利用了owner可以自定义的特点，先创建了几个已经处于锁定状态的mutex对象，然后让子线程解锁，主线程需要等所有子线程都解锁后才能运行。编译执行会看到类似下面的输出：

```console
[10:22:23] 线程 B 开始运行
[10:22:23] 线程 A 开始运行
[10:22:23] 线程 C 开始运行
[10:22:53] 线程 main 开始执行
[10:22:53] 线程 main 结束执行
```

可以看到主线程要拿到所有mutex才会继续执行。

一次可以等待的最大对象数由宏`NTSYNC_MAX_WAIT_COUNT`定义，objs的元素数量超过这个值也会导致报错。

mutex没有批量解锁的功能。

值得注意的是，ALL操作需要所有对象全部处于触发状态才会返回，打个比方如果在等待B和C的锁时，A又重新被锁定且不释放，那么ALL操作会一直阻塞下去，直到A、B、C都处于被触发状态。

## 超时和取消

前面的例子里我们都没有使用超时，只要将`timeout`参数设置为`UINT64_MAX`以外的正整数就可以实现超时返回。

注意`timeout`参数传递的是超时发生的时间点的时间戳，比如要实现超时等待5秒，就需要用现在的时间戳加上5秒传递给timeout：

```c
// 获取当前绝对纳秒时间戳
static uint64_t get_abstime_ns(clockid_t clock_id, uint64_t relative_ns) {
    struct timespec ts;
    clock_gettime(clock_id, &ts);
    uint64_t now_ns = (uint64_t)ts.tv_sec * 1000000000ULL + ts.tv_nsec;
    return now_ns + relative_ns;
}

struct ntsync_event_args e_args_A = {
    .signaled = 0,
    .manual = 0,
};
int ntsync_event_A = ioctl(
    ntsync_fd,
    NTSYNC_IOC_CREATE_EVENT,
    &e_args_A
);
if (ntsync_event_A < 0) {
    perror("open event_A");
    exit(1);
}

uint32_t objs[] = { (uint32_t)ntsync_event_A };
struct ntsync_wait_args wait_args = {
    .objs = (uint64_t)(uintptr_t)objs,
    .count = 1,
    .owner = 0,
    .index = 0,
    .flags = NTSYNC_WAIT_REALTIME,
    .timeout = get_abstime_ns(CLOCK_REALTIME, 5000000000ULL), // 5秒
};

// 等待触发，因为event永远不会被触发，所以等待5秒后超时
int ret = ioctl(ntsync_fd, NTSYNC_IOC_WAIT_ANY, &wait_args);
if (ret != 0) {
    perror("NTSYNC_IOC_WAIT_ANY failed");
    exit(1);
}
```

编译运行，等待5秒后会输出：`NTSYNC_IOC_WAIT_ANY failed: Connection timed out`。这是因为超时后errno会被设置为`ETIMEDOUT`，这个错误码以前只有网络相关的系统调用在设置，因此错误信息变成了连接超时。

使用`NTSYNC_WAIT_REALTIME`判断超时的缺点是会受系统时间影响，例如系统时钟漂移或者润秒都会导致出问题。如果要严格控制超时的时间，应该选用其他时钟类型：

```c
struct ntsync_wait_args wait_args = {
    .objs = (uint64_t)(uintptr_t)objs,
    .count = 1,
    .owner = 0,
    .index = 0,
    .flags = 0, // 注意是0
    .timeout = get_abstime_ns(CLOCK_MONOTONIC, 5000000000ULL), // 5秒
};
```

`CLOCK_MONOTONIC`表示从系统启动后到现在的时间，这个时间不会受系统时钟的影响。

讲完超时，我们再来看看如何取消wait操作。

在触发超时之前，wait操作可以被取消。被取消时wait操作会正常返回不会报错，区分是否是取消导致的返回需要判断wait_args里的`index == count`。我们看个例子：

```c
typedef struct {
    int obj_fd;
    int sleep_time;
} ThreadArgs;

void* thread_func(void *arg) {
    ThreadArgs *t_args = (ThreadArgs*)arg;

    sleep(t_args->sleep_time);

    uint32_t prev_state = 0;
    if (ioctl(t_args->obj_fd, NTSYNC_IOC_EVENT_SET, &prev_state) < 0) {
        perror("NTSYNC_IOC_EVENT_SET failed");
    }

    return NULL;
}

int main()
{
    int ntsync_fd = open("/dev/ntsync", O_RDWR|O_CLOEXEC);
    if (ntsync_fd < 0) {
        perror("open ntsync");
        exit(1);
    }
    struct ntsync_event_args e_args = {
        .signaled = 0,
        .manual = 0,
    };
    int ntsync_event_A = ioctl(
        ntsync_fd,
        NTSYNC_IOC_CREATE_EVENT,
        &e_args
    );
    if (ntsync_event_A < 0) {
        perror("open event_A");
        exit(1);
    }
    struct ntsync_event_args e_args_B = {
        .signaled = 0,
        .manual = 0,
    };
    int ntsync_event_B = ioctl(
        ntsync_fd,
        NTSYNC_IOC_CREATE_EVENT,
        &e_args
    );
    if (ntsync_event_B < 0) {
        perror("open event_B");
        exit(1);
    }
    int ntsync_event_C = ioctl(
        ntsync_fd,
        NTSYNC_IOC_CREATE_EVENT,
        &e_args
    );
    if (ntsync_event_C < 0) {
        perror("open event_C");
        exit(1);
    }
    int ntsync_event_cancel = ioctl(
        ntsync_fd,
        NTSYNC_IOC_CREATE_EVENT,
        &e_args
    );
    if (ntsync_event_cancel < 0) {
        perror("open event_cancel");
        exit(1);
    }

    pthread_t threads[1];
    ThreadArgs t_args[1] = {
        { .obj_fd = ntsync_event_cancel, .sleep_time = 10 },
    };

    pthread_create(&threads[0], NULL, thread_func, &t_args[0]);

    uint32_t objs[] = { (uint32_t)ntsync_event_A, (uint32_t)ntsync_event_B, (uint32_t)ntsync_event_C };
    struct ntsync_wait_args wait_args = {
        .objs = (uint64_t)(uintptr_t)objs,
        .count = 3,
        .owner = 0,
        .index = 0,
        .flags = 0,
        .timeout = UINT64_MAX,
        .alert = ntsync_event_cancel, // 子线程中触发取消，只能传递event对象的描述符
    };

    int ret = ioctl(ntsync_fd, NTSYNC_IOC_WAIT_ANY, &wait_args);
    if (ret != 0) {
        perror("NTSYNC_IOC_WAIT_ANY failed");
        exit(1);
    }

    
    pthread_join(threads[0], NULL);

    printf("index = %d, count = %d\n", wait_args.index, wait_args.count);

    close(ntsync_fd);
}
```

通过触发传给alert参数的event对象，wait被取消了，index的值也被设置为count：

```console
index = 3, count = 3
```

`NTSYNC_IOC_WAIT_ALL`的取消操作是一样的。

不过对于any，还一个坑——any允许event同时出现在objs和alert里，该条件下即使取消用的event对象被触发，`NTSYNC_IOC_WAIT_ANY`也会正常返回，index被设置为取消event在objs中的索引，换而言之wait_any正常返回而不是被中止：

```diff
- uint32_t objs[] = { (uint32_t)ntsync_event_A, (uint32_t)ntsync_event_B, (uint32_t)ntsync_event_C };
+ uint32_t objs[] = { (uint32_t)ntsync_event_A, ()ntsync_event_cancel, (uint32_t)ntsync_event_B, (uint32_t)ntsync_event_C };
 struct ntsync_wait_args wait_args = {
     .objs = (uint64_t)(uintptr_t)objs,
-    .count = 3,
+    .count = 4
     .owner = 0,
     .index = 0,
     .flags = 0,
     .timeout = UINT64_MAX,
     .alert = ntsync_event_cancel, // 子线程中触发取消
 };
```

我们把`ntsync_event_cancel`也加入进objs，下标是1。现在输出会变成：

```console
index = 1, count = 4
```

可见wait_any没有被取消而是正常返回了。这一行为是API文档中明确声明的，从语义上来说取消和就绪是同时到来的，取哪种状态都有其道理。

`NTSYNC_IOC_WAIT_ALL`不允许alert指定的event对象出现在objs里，因此没有类似的问题。作为良好的开发实践，any也应该遵守同样的规则——不要将alert中的对象放入objs。

## 读取对象的状态信息

所有对象都可以通过ioctl取得它们当前的状态信息：

```c
ioctl(ntsync_event, NTSYNC_IOC_EVENT_READ, &args); // args是ntsync_event_args
ioctl(ntsync_sem, NTSYNC_IOC_SEM_READ, &args); // args是ntsync_sem_args
ioctl(ntsync_mutex, NTSYNC_IOC_MUTEX_READ, &args); // args是ntsync_mutex_args
```

不过并不推荐使用这些接口，因为读取状态和后续操作是分隔开的，不是原子操作，因此会引入竞态条件，导致并发问题。

## 使用场景

除了用来模拟Windows的接口以方便移植Windows程序，ntsync的使用场景还是比较有限的。

另一个比较有用的场景是需要混合等待信号量和锁、需要等待多个对象同时就绪的场景，但这样的场景并不多，而且需要把同步手段完全替换成ntsync支持的三种对象上，开发复杂度会增加很多。

而且event之外剩下两个对象在Linux上都有轻量级替代，这些替代的性能甚至更好，比如pthread和Go标准库实现的mutex在竞争不多的情况下基本都是纯原子操作，而ntsync需要走ioctl系统调用。

如上面所说，ntsync的性能是不如其他原生的同步原语的，之所以在wine里能带来显著的提升，是因为wine在之前的版本中没有一种简单高效的办法来模拟Windows的API，ntsync对wine有提升不代表对其他原生Linux程序有类似的性能提升。

尽管使用场景很少，但作为知识储备了解一下ntsync的使用还是值得的，这有助于拓宽自己的眼界，并且丰富了自己的并发武器库。

## 总结

ntsync近似模拟了Windows接口在可等待对象上的行为，为移植Windows程序提供了便利。

虽然wine已经可以启用ntsync，但我还是不推荐自己使用这个功能，因为ntsync的接口还未稳定，而且内核默认并不编译也不启用这个模块，激进地使用这一功能会导致不能向后兼容，同时支持的平台和内核版本也会受限。

当然，ntsync的代码已经存在于很多个Linux内核版本里了，包括一个LTS版本，即使接口再变，也不会和这个教程相差太多。期待后续版本的ntsync可以提供更多的功能和更好的性能。
