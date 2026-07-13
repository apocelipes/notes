| TLS 模型 | 工作原理 | `.o` 中典型 x86-64 relocation | 未放宽时性能 | 可放入 `.a` | 最终进入主程序/PIE | 最终进入 `.so` | 典型用途 | 主要限制 |
|---|---|---|---|---|---|---|---|---|
| local-exec | 链接主程序时确定 TLS 变量相对 thread pointer 的固定偏移，运行时直接通过 `%fs:offset` 一类指令访问 | `R_X86_64_TPOFF32` | 最高 | 可以，但归档中的对象只能用于合适的最终主程序 | 可以 | 不可以 | 主程序访问自身定义、不可抢占的 TLS | TLS 符号必须确定属于最终可执行文件；不能作为通用 `.so` 或插件代码 |
| initial-exec | 从 GOT 读取 TLS 变量相对 thread pointer 的固定偏移，再用 thread pointer 加该偏移访问 | `R_X86_64_GOTTPOFF`；最终 `.so` 的 GOT 常使用 `R_X86_64_TPOFF64` | 高 | 可以，最终用途决定是否合法 | 可以 | 有条件可以 | 启动时加载、性能敏感的共享库 TLS | 依赖 static TLS 空间；通用 `dlopen` 场景不能保证成功 |
| local-dynamic | 通过 `__tls_get_addr` 获得当前模块 TLS block 的基址，再加同模块内变量的固定偏移 | `R_X86_64_TLSLD` + `R_X86_64_DTPOFF32` | 中；多个本地 TLS 访问可复用模块基址 | 可以 | 可以，且常可放宽为 local-exec | 可以 | 共享库内部访问本模块不可抢占的 TLS | TLS 符号必须确定属于当前模块（不可抢占，需要是static或者hidden）；定位模块基址时有 `__tls_get_addr` 开销 |
| global-dynamic | 通过包含 module ID 和模块内偏移的 TLS index 调用 `__tls_get_addr`，动态定位完整地址 | `R_X86_64_TLSGD`；最终 GOT 常使用 `R_X86_64_DTPMOD64` + `R_X86_64_DTPOFF64` | 通常最低 | 可以 | 可以，且常可被放宽 | 可以 | 默认可见、可抢占、跨模块或最通用的 TLS 访问 | 未放宽时通常需要 `__tls_get_addr`，访问成本最高 |

`local-exec` 和 `initial-exec` 依赖 static TLS 布局；`local-dynamic` 和 `global-dynamic` 能通过 `__tls_get_addr` 支持更通用的模块 TLS 定位。

local-exec 不能用于共享库，因为共享库可能被加载到其他模块中，静态链接器无法把其中 TLS 变量当成最终主程序 TLS 布局的一部分来固定处理。包含 local-exec 对象的 .a 可以直接链接进主程序，但不能再把同一个对象嵌入通用 .so。

initial-exec 可以用于共享库，但它要求目标 TLS 变量获得 static TLS (PT_TLS) 偏移。主程序及其启动时加载的依赖通常可以满足这个条件。

local-dynamic 将“定位当前模块 TLS block”和“计算模块内变量偏移”分成两步：

```text
module_base = __tls_get_addr({current_module_id, 0})
address     = module_base + link_time_constant_dtpoff
```

global-dynamic 使用完整 TLS index：

```text
tls_index = {module_id(symbol), dtpoff(symbol)}
address   = __tls_get_addr(tls_index)
```

- `R_X86_64_TPOFF32`
  - 含义：TLS 变量相对 thread pointer 的有符号 32 位固定偏移。
  - 常见模型：`local-exec`。
  - 处理阶段：通常由静态链接器在生成最终主程序时解析，不需要保留为运行时动态重定位。
  - 特点：不需要 GOT，也不需要调用 __tls_get_addr，因此访问速度最快。

- `R_X86_64_GOTTPOFF`
  - 含义：引用保存 TP-relative TLS 偏移的 GOT 项。
  - 常见模型：`initial-exec`。
  - 处理阶段：静态链接器使用它构造或定位相应 GOT 项。
  - 特点：运行时需要一次 GOT 读取，但不需要调用 __tls_get_addr。

- `R_X86_64_TPOFF64`
  - 含义：TLS 变量相对 thread pointer 的 64 位偏移。
  - 常见模型：initial-exec 的最终动态重定位。
  - 处理阶段：在共享库中，动态加载器通常用它填写 `R_X86_64_GOTTPOFF` 所引用的 GOT 项。

- `R_X86_64_TLSLD`
  - 含义：标记 local-dynamic 的模块 TLS index 访问序列。
  - 常见模型：local-dynamic。
  - 特点：通过 `__tls_get_addr` 获得当前模块的 TLS block 基址。
  - 最终动态重定位：模块 TLS index 中的 module ID 通常由 `R_X86_64_DTPMOD64` 填写。

- `R_X86_64_DTPOFF32`
  - 含义：TLS 变量相对其所在模块 TLS block 起点的有符号 32 位偏移。
  - 常见模型：`local-dynamic`。
  - 处理阶段：通常由静态链接器解析为当前模块内部的固定偏移。
  - 特点：取得一次模块基址后，同一模块中的多个 TLS 变量可以分别加各自的 DTPOFF。

- `R_X86_64_TLSGD`
  - 含义：标记 global-dynamic TLS index 的访问序列。
  - 常见模型：`global-dynamic`。
  - 特点：通过 `__tls_get_addr` 按“模块 + 符号”动态解析 TLS 地址，需要保留必要的动态重定位

- `R_X86_64_DTPMOD64`
  - 含义：TLS 符号所在模块的 module ID。
  - 常见模型：`global-dynamic`；local-dynamic 的模块 index 也会使用它。
  - 特点：通常由动态加载器在加载 `.so` 时填充。

- `R_X86_64_DTPOFF64`
  - 含义：TLS 符号在其模块 TLS block 内的偏移。
  - 常见模型：`global-dynamic` / 动态 TLS。
  - 特点：通常和 `R_X86_64_DTPMOD64` 一起用于动态 TLS 地址解析。

1. `R_X86_64_TLSGD`、`R_X86_64_TLSLD`、`R_X86_64_GOTTPOFF`、`R_X86_64_TPOFF32`、`R_X86_64_DTPOFF32` 等通常首先出现在可重定位目标文件 `.o` 中，供静态链接器识别和转换。
2. `R_X86_64_DTPMOD64`、`R_X86_64_DTPOFF64`、`R_X86_64_TPOFF64` 等常见于最终 `.so` 的 GOT 或动态重定位表中，供动态加载器填写。
3. 链接器还可能进行 TLS relaxation，将较通用的访问序列转换为更快的模型，因此最终文件里不一定保留输入 `.o` 中的原始 relocation。

## GCC 的通用默认行为是：

- 使用 `-fPIC` 时，默认可见且位置未知的 TLS 通常采用 global-dynamic；本地、hidden 或不可抢占符号可能采用 local-dynamic，之后链接器还可继续放宽。
- 不使用 `-fPIC` 时，编译器可以采用面向最终可执行文件的 initial-exec 等模型。
- 使用 `-fPIE` 时，编译器可以利用“最终产物是主程序”的条件，链接器也更容易将 TLS 访问放宽为 local-exec。

这些只是默认选择。以下方式可以显式覆盖 TLS 模型：

```bash
gcc -ftls-model=global-dynamic ...
gcc -ftls-model=local-dynamic ...
gcc -ftls-model=initial-exec ...
gcc -ftls-model=local-exec ...
```

## 常见错误的判断

### `R_X86_64_TPOFF32` 无法用于共享库

典型错误：

```text
relocation R_X86_64_TPOFF32 against symbol `x' can not be used when making a shared object
```

这通常说明 local-exec TLS 对象正在被链接进 `.so`。应重新编译为适合共享库的模型，通常是使用 `-fPIC` 并移除显式 local-exec 设置。

### `R_X86_64_DTPOFF32` 无法用于共享库

`R_X86_64_DTPOFF32` 本身不是共享库禁用的 relocation，它是经典 local-dynamic 序列的一部分。若出现类似错误，更常见的原因是：

- relocation 指向默认可见、可能被抢占的 TLS 符号；
- TLS 符号实际定义在其他模块，静态链接器无法确定当前模块内偏移；
- 代码显式强制了 local-dynamic，但符号绑定语义不满足该模型；
- 目标文件、visibility 或链接选项之间存在不一致。

这时应根据语义改用 global-dynamic，或者在符号确实只能由本模块定义时将其设为 hidden/local。重新使用 `-fPIC` 编译可能通过改变默认模型而解决问题，但如果代码显式强制了 local-dynamic，它不一定足够。
