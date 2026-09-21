在/sys/fs/cgroup，一般是systemd自动挂载的。

与v1不同。可以控制各个资源的max，min和权重，memory.oom.group控制oom是kill还是强制睡眠进程直到有可用内存。

创建/sys/fs/cgroup/example，这个是一个subgroup。v2中所有控制在同一目录中完成。

```bash
# 自己的cgroup.controllers里没有的不能加入
echo "+cpu" >> /sys/fs/cgroup/example/cgroup.subtree_control
echo "+memory" >> /sys/fs/cgroup/example/cgroup.subtree_control
echo "+pids" >> /sys/fs/cgroup/example/cgroup.subtree_control
mkdir /sys/fs/cgroup/example/tasks 创建子组，只有父group里开放给子group的控制器
# example自身的控制器需要在更上层的目录里修改
```
添加相应的资源控制器进subgroup。控制器可以按目录树继承，子组的控制可以覆盖父组的，但必须以满足父组的限制为前提。新建目录后自动创建目录名的子组，目录里包含控制文件

cgroup v2 严格遵循“非根节点如果开启了 subtree_control（即含有子 group，"No Internal Processes"（无内部进程）约束），则其自身不能包含任何进程”的原则。因此我们必须创建子group并把进程加入子group。

```bash
echo "200000 1000000" > /sys/fs/cgroup/example/tasks/cpu.max
```
限制cpu的使用上限，单位微秒，第一个表示所有进程加起来可运行的时间，第二个表示周期。
意思是1秒里只允许运行200毫秒。剩下的时间进程会休眠。cpu监视器上会显示出周期性的cpu上升和下降。

```bash
# 设为 2G 硬上限，超过会oom kill
echo "2G" > /sys/fs/cgroup/example/tasks/memory.max

# 设置 600M 的软保护，防止被频繁回收
echo "600M" > /sys/fs/cgroup/example/tasks/memory.low

# 设为 1G 软上限，超过时不oom，系统会先尝试回收内存页
echo "1G" > /sys/fs/cgroup/example/tasks/memory.high

# 设置 500M 的硬下限，低于时系统尽可能不回收内存页
echo "500M" > /sys/fs/cgroup/example/tasks/memory.min

# 设置 OOM 时整体 kill 掉整个 subgroup，0则按score选单个进程杀死
echo "1" > /sys/fs/cgroup/example/tasks/memory.oom.group
```

```bash
# 控制group里最多同时存在100个进程，超出则会报错，比如fork返回EAGAIN
echo 100 > /sys/fs/cgroup/example/tasks/pids.max

# 去除限制
echo max > /sys/fs/cgroup/example/tasks/pids.max

# 只读选项，查看当前group里有多少进程（包含所有状态）
cat /sys/fs/cgroup/example/tasks/pids.current
```

```bash
echo 483947 > /sys/fs/cgroup/example/tasks/cgroup.procs
```
把要控制的进程加入group。注意是用`>`，成功后进程移入新的group，不再受老group控制。v2支持用过将线程TID写入`cgroup.threads`使得不同线程进入不同group，但需要group的类型是`domain threaded`或`threaded`。

```bash
cat /proc/483947/cgroup
0::/example/tasks
```
显示已经加入。

把进程从subgroup移除：只能转移到另一个cgroup或者返回root cgroup，把pid加入同一层级的另一个cgroup即可。

进程退出会自动从cgroups里删除。group需要rmdir或者cgroup.kill才能删除。

冻结整个group及其subgroup中的所有进程：

```bash
# 冻结
echo 1 > /sys/fs/cgroup/example/cgroup.freeze

# 解冻
echo 0 > /sys/fs/cgroup/example/cgroup.freeze

# 彻底杀死example和子group。发送SIGKILL
echo 1 > /sys/fs/cgroup/example/cgroup.kill
```

内核会直接修改进程的状态为冻结，不需要信号或者其他交互方式。

`.peak`文件表示指标对应的运行期间历史峰值，比如`pids.peak`表示本组及其子组中历史最多同时有多少个进程。

所有控制器：

| 控制器 | 主要功能 | 常用文件及说明 |
| --- | --- | --- |
| **cpu** | 限制 CPU 时间、权重 | `cpu.max`, `cpu.weight` |
| **memory** | 限制内存、OOM 策略 | `memory.max`, `memory.high`, `memory.oom.group` |
| **pids** | 限制进程数 | `pids.max`, `pids.current`, `pids.events` |
| **io** | 限制磁盘 I/O | `io.max`, `io.weight` |
| **hugetlb** | 限制大页内存，很多系统默认不起用 | `hugetlb.<size>.max`, `hugetlb.<hugepagesize>.current` |
| **cpuset** | 指定 CPU/NUMA 节点 | `cpuset.cpus`, `cpuset.mems`, `cpuset.cpus.effective / cpuset.mems.effective` |
| **rdma** | 限制 RDMA 资源 | `rdma.max`, `rdma.current` |
| **devices** | 控制设备访问权限 | 已彻底废弃，现在使用ebpf控制 |
| **dmem** | 控制外部设备内存，比如显存 | 内核6.8+才支持 |
| **misc** | 管理杂项硬件的设置，比如npu、加密卡 | 值一般为`<name> value`，不同name代表了不同硬件的不同配置 |

四种cgroup类型：

```text
      [domain]               <-- 普通进程级 cgroup
          │
  [domain threaded]          <-- 线程树的根节点 (负责 memory 等全局控制)
     ┌────┴────┐
 [threaded] [threaded]       <-- 真正挂载 Thread 的节点 (负责 cpu 等线程级控制)
```

| Cgroup Type | 核心含义 | 适用对象 / 作用范围 | 支持的控制器类型 |
| --- | --- | --- | --- |
| **`domain`** | 标准进程级 cgroup（默认） | 整个进程（PID/TGID） | 支持所有资源控制器（`memory`, `cpu`, `io` 等） |
| **`threaded`** | 纯线程级 cgroup | 单个线程（TID） | 仅支持线程级控制器（如 `cpu`, `cpuset`, `pids`） |
| **`domain threaded`** | 线程化子树的根节点 | 进程与线程混合边界/子树根 | 作为全局资源（如 `memory`）的控制边界 |
| **`domain invalid`** | 非法或未就绪的过渡状态 | 状态出错或配置冲突的节点 | 不允许加入任何进程或线程 |
