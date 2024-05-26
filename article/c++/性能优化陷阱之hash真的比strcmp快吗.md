最近网上冲浪的时候看到有人分享了自己最近一次性能优化的经验。我向来对性能是比较敏感的，所以就点进去看了。

然而我越看越觉得蹊跷，但本着“性能问题和性能优化要靠性能测试做依据”，我不能凭空怀疑别人吧，所以我做了完整的测试并写下了这篇文章。

## 可疑的优化方案

分享者遇到的问题很简单：他发现程序中超过一半的时间花在strcmp上，并且在抓取调用栈等深入分析之后，他发现大部分strcmp调用都是在拿各种字符串和某个固定的字符串（比如表示成功状态的“success”）进行比较，因此他想能不能针对这进行优化。

这里稍微给不熟悉c/c++的人解释下，strcmp函数用来比较两个字符串的大小，当两个字符串相同的时候，函数会返回0，c语言里想要判断两个字符串的内容是否相同只能使用这个函数。

好了，背景交代完，我们来看看他的优化方案：

> 既然经常和固定的字符串比较，那是不是能节约掉这样的strcmp调用呢？
> 我们可以利用hash函数计算出的hash值，先计算出那个固定的字符串的hash值存下来，然后再计算与它比较的字符串的hash值；
> 现在strcmp就转换成两个hash值的比较了，不相同的hash值一定意味着字符串不同，可以直接返回false；
> 相同hash值的字符串有可能是因为hash冲突，所以还需要回退到调用strcmp；
> hash值的比较本身就是一次整数的比较，非常快，比函数调用快得多，因此字符串不相等的情况可以得到优化；
> 系统里大多数时候和固定字符串比较的结果都是不相等，因此能被优化覆盖的情况占大多数。

代码写出来大致是这样的：

```c++
// 如果觉得inline里放static不舒服，可以把它们移出去变成文件作用域变量，当然不管怎么样对测试结果几乎没有影响
inline bool optimize_cmp(const char *str) {
    static auto hasher = std::hash<std::string_view>{};
    // 保存固定字符串的hash值，以后不会重复计算
    static const uint32_t target_hash = hasher(std::string_view{target});
    // 计算和固定字符串进行比较的字符串的hash
    const uint32_t h = hasher(str);
    // hash不相等，则一定不相等
    if (h != target_hash) {
        return false;
    }
    // hash相等，得回退到strcmp进行进一步检查
    return std::strcmp(target, str) == 0;
}
```

看起来好像有几分道理？但经验丰富或者对性能优化研究较深的人可能会产生违和感了：

本质上这是把一次strcmp的调用转换成了一次hash计算加一次整数值比较，但hash计算是比较吃cpu的，它真的能更快吗？

## 性能测试

口说无凭，性能优化讲究用数据说话。因此我们来设计一个性能测试。

首先当然是测试对象，就是上面的`optimize_cmp(待比较字符串)`和`strcmp(固定字符串，待比较字符串)`。

有人会说hash函数对`optimize_cmp`性能起决定性作用，没错是这样的，所以我选了目前我测试出的在X86-64机器上最快的字符串hash，正好是标准库（libstdc++）提供的`std::hash<std::string_view>`。另外我还追踪了下为什么libstdc++的hash这么快，因为它用的是优化版的`Murmur Hash`算法。虽说现在需要把字符串字面量转换成string_view，但`std::hash<std::string_view>`依然是最快的，而且转换本身不会花多少时间，对性能的影响几乎可以忽略不计。

strcmp当然也有优化，x86平台上它会尽量利用avx2指令。不过c++里还有个杀手锏constexpr，它能在编译阶段就比较两个字符串编译期常量，这相当于是作弊，运行期直接都拿到现成的结果了还能比啥？所以为了解决这个问题需要用点小手段。

所以整体上方案是分成两组测试，一组测试字符串不相等的也就是上面说的优化的场景，另一组测试字符串相等的情况看看性能损失有多少。

因为计算字符串hash需要遍历整个字符串，因此为了避免`strcmp("abcdefg", "a")`这种不公平情况（strcmp在这时通常检查完前几个字符就知道谁大谁小是否相等了），比较用的字符串除了一个空字符，其他都尽可能和固定字符串一样长且只修改最后一个字符来制造不相等，这样大家的计算量至少从理论上来说是差不多的。

代码如下：

```c++
const char *target = "abcdefgh";

// 用数组遍历避免strcmp被编译期计算
const char *not_match_data_set[] = {
    "abcdefgj",
    "abcdefgi",
    "abcdefgg",
    "abcdefgk",
    "",
};

const char *match_data_set[] = {
    target,
    target,
    target,
    target,
    target,
};

// 这里开始是测试函数
void bench_strcmp_not_match(benchmark::State &stat)
{
    for (auto _ : stat) {
        for (const char *s : not_match_data_set) {
            // 为了避免调用被优化掉，同时也兼顾测试了函数行为是否正确
            if (std::strcmp(target, s) == 0) {
                std::abort();
            }
        }
    }
}
BENCHMARK(bench_strcmp_not_match);

void bench_optimized_not_match(benchmark::State &stat)
{
    for (auto _ : stat) {
        for (const char *s : not_match_data_set) {
            if (optimize_cmp(s)) {
                std::abort();
            }
        }
    }
}
BENCHMARK(bench_optimized_not_match);

void bench_strcmp(benchmark::State &stat)
{
    for (auto _ : stat) {
        for (const char *s : match_data_set) {
            if (std::strcmp(target, s) != 0) {
                std::abort();
            }
        }
    }
}
BENCHMARK(bench_strcmp);

void bench_optimized(benchmark::State &stat)
{
    for (auto _ : stat) {
        for (const char *s : match_data_set) {
            if (!optimize_cmp(s)) {
                std::abort();
            }
        }
    }
}
BENCHMARK(bench_optimized);

BENCHMARK_MAIN();
```

测试使用了google benchmark，不熟悉的可以先看我以前写的几篇文章：

- [c++性能测试工具：google%20benchmark入门（一）](./c++性能测试工具：google%20benchmark入门（一）.md)
- [c++性能测试工具：google%20benchmark入门（二）](./c++性能测试工具：google%20benchmark入门（二）.md)
- [c++性能测试工具：google%20benchmark进阶（一）](./c++性能测试工具：google%20benchmark进阶（一）.md)

测试结果非常喜感：

```text
Running ./a.out
Run on (8 X 2400.01 MHz CPU s)
CPU Caches:
  L1 Data 32 KiB (x4)
  L1 Instruction 32 KiB (x4)
  L2 Unified 256 KiB (x4)
  L3 Unified 8192 KiB (x1)
Load Average: 0.11, 0.04, 0.01
--------------------------------------------------------------------
Benchmark                          Time             CPU   Iterations
--------------------------------------------------------------------
bench_strcmp_not_match          10.6 ns         10.5 ns     65624918
bench_optimized_not_match       23.2 ns         23.2 ns     29860768
bench_strcmp                    11.1 ns         11.1 ns     65170113
bench_optimized                 32.5 ns         32.5 ns     21280968
```

可以看到所谓的优化方案几乎**慢了整整一倍**。对了，这还是开了`-O2`优化选项后的结果，不开优化差距更加的大。

会不会是字符串太短了，体现不了优势？这就是为什么前面我要啰嗦那么多告诉你性能测试是怎么设计的，现在我们只需要改一下数据集，其他原理还是一样的：

```c++
const char *target = "aaabcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890";

const char *not_match_data_set[] = {
    "aaabcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567891",
    "aaabcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567892",
    "aaabcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567899",
    "aaabcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567898",
    "",
};
```

现在改成64个字符了，对于常见的字符串相等比较来说已经有点长了，然后我们再看测试结果：

```text
Running ./a.out
Run on (8 X 2400.01 MHz CPU s)
CPU Caches:
  L1 Data 32 KiB (x4)
  L1 Instruction 32 KiB (x4)
  L2 Unified 256 KiB (x4)
  L3 Unified 8192 KiB (x1)
Load Average: 0.04, 0.06, 0.01
--------------------------------------------------------------------
Benchmark                          Time             CPU   Iterations
--------------------------------------------------------------------
bench_strcmp_not_match          13.4 ns         13.2 ns     53983729
bench_optimized_not_match       52.0 ns         51.3 ns     10676509
bench_strcmp                    13.0 ns         12.9 ns     53231858
bench_optimized                 77.8 ns         77.6 ns      9317779
```

“优化方案”的性能更差了，竟然慢了四倍！而我们早有预期会下降的相等时的性能更是慢了高达6倍。。。

趋势也很明显，字符串越长“优化方案”性能越差。

嗯，一般优化都是正向的，逆向的优化确实不多见。

## 为什么优化无效

这个分享的“优化方案”在它的优化场景上性能不增反降，在选择trade-off的场景上产生了无法接受的性能倒退，更感人的是方案的代码本身也比直接调用strcmp复杂的多。

所以，这个方案既没有必要性也无法被生产环境接受。

但，这是为什么？

答案前面就说了：字符串的hash计算是很吃cpu的。计算时不仅要遍历字符串，为了尽量减少冲突还要和预先定义的素数等做运算，最重要的是还需要利用每个字符的相对位置来计算hash，否则你会看到“abcd”和“dcba”有一样的hash值。

这些意味着下面几点：

1. 字符串hash计算时很容易产生数据依赖，这会导致无法有效利用cpu的流水线预执行，相对来说strcmp只是简单遍历两个字符串，数据依赖非常少，这一点导致了一部分性能差异；
2. 因为数据依赖，hash计算很难被有效得向量化，而strcmp可以大量利用avx2甚至在新的CPU上已经能利用avx512了。strcmp三四条条指令就能处理完64个字符，但hash计算时三四条指令甚至有可能一个字符都没处理完；
3. 为了减少hash冲突，hash计算需要更多的步骤，和字符的大小比较相比需要花更多时间。

感兴趣的可以看看`murmur hash`是怎么实现的，上面每条都是它的痛点。

如果是嵌入式环境呢，那里的cpu功能受限，通常也没什么SIMD指令给你用。答案还是一样的，计算hash比循环比较字符大小要花费更多的指令，因此不可能更快最多之后两者性能差不多，这时候明显谁的代码更简单谁的方案就更好，因此这个“优化方案”又落选了。

还有没有更快的hash算法？有是有，但要么通用性不高要么功耗太大但无法显著超越strcmp，总的来说想要在性能上碾压strcmp，现阶段是没啥希望的。

另外还得给strcmp说句公道话，这个函数是热点中的热点，被人优化了几十年了，大多数常见场景都考虑到了，与其觉得strcmp本身是瓶颈，不如重新考虑考虑自己的设计和算法是否妥当比较好。

## 正确的优化

正确的优化该怎么做呢？由于我没有分享者的具体使用场景，所以不可能直接给出最优解，因此就泛泛而谈几点吧。

1. 不做优化。程序本身的逻辑/算法就是如此，再怎么优化这些比较也是省略不掉的。这时候不如考虑考虑升级硬件。
2. 如果要比较的字符串的长度都相等，可以试试memcmp，当然收益可能非常小，需要做测试来决定是否采用这个方案，因为本方案很不利于扩展，一不小心就会导致bug甚至高危漏洞。
3. 如果在用c++，那么尽量利用strcmp的编译期求值的能力，减少运行时的耗时。不过前提是要保证代码的可读性不能因为要利用编译期求值而产生代码异味。
4. 对于某些固定的数据集，需要在数据集上反复调用strcmp进行等值比较的，可以考虑用二分查找处理数据集或者把这些数据存放进关联容器里。虽然单次strcmp很快，但大量不必要的重复计算是会成为性能问题的，这时候hashmap之类关联性容器的优势就能体现出来了，一次hash虽然比单次strcmp慢了五倍，但比十次连续的strcmp调用快了一倍。
5. 最终极的办法，像上一节最后说的，重新思考自己的设计或算法是否合理，重新思考一个更高效的设计/算法。

还有，别忘了对自己的方案进行测试，免得出现了“负优化”。

## 总结

那篇分享文的作者最后没说是否有把这个方案实际应用到生产环境，也没有说具体带来了多少提升。但愿有人拦住了他吧。

从这里我们可以得出一个重要的经验：凡是讲优化的，既没给出性能测试，又没给出优化应用后的效果，那就得留个心眼了，这多半是有问题的无效优化甚至会是负优化。

最后的最后，还是那句话：性能问题有关的任何事都需要有可靠的性能测试做依据。
