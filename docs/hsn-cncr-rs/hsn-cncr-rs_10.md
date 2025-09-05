# 第十章：未来主义——近期的 Rust

本书我们已经涵盖了大量的内容。

如果你已经从另一种系统编程语言中很好地了解了并行编程，我希望你现在对 Rust 的操作方式有了自信的掌握。它的所有权模型是严格的，一开始可能会感觉陌生。如果我的个人经验可以作为一个普遍的指南，那么不安全的 Rust 感觉更熟悉。我希望你也开始欣赏 Rust 的一般安全性，以及它在需要时在内存中执行繁琐操作的能力。我发现能够实现不安全代码，然后将其封装在安全接口中，这对我来说是不可思议的。

如果你已经从高级、垃圾回收的编程语言中相当熟悉并行编程，那么我希望这本书已经作为系统级并行编程的入门为你提供了帮助。正如你所知，内存安全并不能保证正确运行，因此这本书持续关注测试——生成性和模糊测试，以及性能检查——并对模型进行仔细推理。我主张，Rust 由于语言对所有权管理和生命周期的关注以及强大的静态类型系统，是易于进行良好并行编程的系统语言之一。Rust 并非对错误免疫——事实上，它容易受到广泛的错误影响——但这些在现代硬件上的所有语言都是共通的，而且 Rust 确实设法解决了一系列与内存相关的问题。

如果你已经了解 Rust，但不太了解并行编程，我非常希望这本书已经设法让你相信了一件事——并发不是魔法。并行编程是一个广泛的领域，需要大量的研究，但它可以理解和掌握。此外，它不必一次性掌握。这本书的形状为多条路径中的一条提供了论据。希望这本书能成为你旅程中的第一本书。

无论你的背景如何，感谢你阅读到最后一页。我知道，有时覆盖的材料并不容易，完成这一切肯定需要真正的意志力。我非常感激。我写这本书是为了满足我十年前的愿望。有机会把它写出来给你看，这是一件真正令人愉快的事情。

到本章结束时，我们将：

+   讨论了 Rust 中 SIMD 的未来

+   讨论了 Rust 中异步 I/O 的未来

+   讨论了专业化的稳定性和其可能的形式

+   讨论了 Rust 的更广泛的测试方法

+   介绍了参与 Rust 社区的各种途径

# 技术要求

本章没有软件要求。

你可以在 [`github.com/PacktPublishing/Hands-On-Concurrency-with-Rust`](https://github.com/PacktPublishing/Hands-On-Concurrency-with-Rust) 找到这本书项目的源代码。第十章的源代码位于 `Chapter10/`。

# 近期改进

2018 年 Rust 开发的重点是稳定化。自 2015 年的 1.0 版本以来，许多重要的 crate 仅限于夜间使用，无论是由于编译器的修改还是因为担心过于快速地标准化令人尴尬的 API。这种担忧是合理的。当时，Rust 是一种新的语言，它在 1.0 版本前的变化非常剧烈。一种新的语言需要时间来形成其自然风格。到 2017 年底，社区普遍认为需要一个稳定周期，即语言的某些自然表达已经或多或少地确立，而在这些方面没有确立的地方，通过一些工作可以确立。

让我们讨论一下与书中所讨论的主题相关的这个稳定化工作。

# SIMD

在这本书中，我们讨论了基于线程的并发。在第八章高级并行性 – Threadpools, Parallel Iterators, and Processes 中，我们通过 Rayon 的并行迭代器达到了更高的抽象层次。正如我们所看到的，rayon 使用了一个复杂的工作窃取线程池。线程只是现代 CPU 上实现并发的一种方法。从某种意义上说，具有深度预取流水线和复杂的分支预测器的 CPU 上的串行程序是并行的，正如我们在第二章顺序 Rust 性能和测试中看到的。利用这种并行性是一个精心设计数据结构并管理对其访问的问题，以及其他我们讨论的细节。在这本书中，我们没有探讨如何利用现代 CPU 并行执行连续内存中相同操作的能力，而不需要求助于线程——向量化。我们没有探讨这一点，因为在我写这篇文章的时候，该语言的标准库中只为向量化指令提供了最小限度的支持。

向量化有两种变体——MIMD 和 SIMD。MIMD 代表多个指令，多个数据。SIMD 代表单指令，单数据。基本思路是这样的：现代 CPU 可以并行地将一条指令应用于连续的、特定大小的数据块。仅此而已。现在，考虑一下在这本书中我们编写了多少个循环，其中我们对循环的每个元素都执行了完全相同的操作。不止几个！如果我们调查了字符串算法或者考虑进行数值计算——加密、物理模拟等——那么在这本书中就会有更多这样的循环。

当我写下这些文字时，SIMD 向量化合并到标准库的进程被 RFC 2366（[`github.com/rust-lang/rfcs/pull/2366`](https://github.com/rust-lang/rfcs/pull/2366)）所阻碍，目标是到年底前将 x86 SIMD 纳入稳定渠道，其他架构随后跟进。读者可能还记得第一章，“预备知识——机器架构和 Rust 入门”，其中提到 x86 和 ARM 的内存系统相当不同。嗯，这种差异也扩展到了它们的 SIMD 系统。确实可以通过稳定的 Rust 通过 stdsimd（[`crates.io/crates/stdsimd`](https://crates.io/crates/stdsimd)）和 simd（[`crates.io/crates/simd`](https://crates.io/crates/simd)）crate 来利用 SIMD 指令。对于程序员来说，stdsimd crate 是最好的目标，因为它最终将成为标准库的实现。

由于剩余空间有限，关于 SIMD 的讨论超出了本书的范围。简单来说，正确处理 SIMD 的细节与正确处理原子编程的细节难度相当。我预计，在稳定化之后，社区将构建高级库来利用 SIMD 方法，而无需进行大量的预先学习，尽管可能优化空间较小。例如，faster（[`crates.io/crates/faster`](https://crates.io/crates/faster)）crate 以 rayon 的精神暴露了 SIMD 并行迭代器。从最终用户的角度来看，那里的编程模型极其方便。

# 十六进制编码

让我们看看 stdsimd crate 的一个例子。我们在这里不会深入挖掘；我们只是想对这种编程的结构有一个大致的了解。我们已经拉取了 stdsimd 在`2f86c75a2479cf051b92fc98273daaf7f151e7a1`。我们将要检查的文件是`examples/hex.rs`。程序的目的是对 stdin 进行`hex`编码。例如：

```rs
> echo hope | cargo +nightly run --release --example hex -p stdsimd
    Finished release [optimized + debuginfo] target(s) in 0.52s
     Running `target/release/examples/hex`
686f70650a
```

注意，这需要 nightly 渠道，而不是我们在这本书中迄今为止所使用的稳定渠道。我在 nightly，2018-05-10 上进行了测试。实现细节变化非常快，依赖于如此不稳定的编译器特性，因此您应该确保使用列出的特定 nightly 渠道，否则示例代码可能无法编译。

`examples/hex.rs`的序言部分包含了许多我们在这本书中尚未见过的信息。让我们分两块来看，并揭示未知的内容：

```rs
#![feature(stdsimd)]
#![cfg_attr(test, feature(test))]
#![cfg_attr(feature = "cargo-clippy",
            allow(result_unwrap_used, print_stdout, option_unwrap_used,
                  shadow_reuse, cast_possible_wrap, cast_sign_loss,
                  missing_docs_in_private_items))]

#[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
#[macro_use]
extern crate stdsimd;

#[cfg(not(any(target_arch = "x86", target_arch = "x86_64")))]
extern crate stdsimd;
```

因为我们坚持使用稳定的 Rust，所以我们之前没有见过 `#![feature(stdsimd)]` 这样的东西。这是什么？Rust 允许有冒险精神的用户启用 rustc 中存在但尚未在语言的夜间版本中公开的功能。我说冒险是因为一个正在进行中的功能可能在其开发过程中保持稳定的 API，或者它可能每隔几天就改变一次，需要消费者端的更新。那么，`cfg_attr` 是什么？这是一个条件编译属性，可以选择性启用功能。你可以在 *The Rust Reference* 中了解所有关于它的信息，该信息链接在 *进一步阅读* 部分。有很多细节，但想法很简单——根据用户指令（如测试或运行环境）打开或关闭代码片段。

这后面的内容是 `#[cfg(any(target_arch = "x86", target_arch = "x86_64"))]` 的意思。在 x86 或 x86_64 上，stdsimd 将通过宏启用并外置，在其他平台上则不启用。现在，希望下面的前言部分是自解释的：

```rs
#[cfg(test)]
#[macro_use]
extern crate quickcheck;

use std::io::{self, Read};
use std::str;

#[cfg(target_arch = "x86")]
use stdsimd::arch::x86::*;
#[cfg(target_arch = "x86_64")]
use stdsimd::arch::x86_64::*;
```

这个程序的 `main` 函数相当简短：

```rs
fn main() {
    let mut input = Vec::new();
    io::stdin().read_to_end(&mut input).unwrap();
    let mut dst = vec![0; 2 * input.len()];
    let s = hex_encode(&input, &mut dst).unwrap();
    println!("{}", s);
}
```

整个 stdin 被读入输入，并创建了一个简短的向量来存储哈希值，称为 `dst`。还使用了 `hex_encode`。第一部分执行输入验证：

```rs
fn hex_encode<'a>(src: &[u8], dst: &'a mut [u8]) -> Result<&'a str, usize> {
    let len = src.len().checked_mul(2).unwrap();
    if dst.len() < len {
        return Err(len);
    }
```

注意返回值——成功时返回 `&str`，失败时返回 `usize`——错误的长度。函数的其余部分稍微复杂一些：

```rs
    #[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
    {
        if is_x86_feature_detected!("avx2") {
            return unsafe { hex_encode_avx2(src, dst) };
        }
        if is_x86_feature_detected!("sse4.1") {
            return unsafe { hex_encode_sse41(src, dst) };
        }
    }

    hex_encode_fallback(src, dst)
}
```

这引入了条件编译，我们之前已经见过，还包含了一个新内容——`is_x86_feature_detected!`。这个宏是 SIMD 功能的新特性，它在运行时通过 libcpuid 测试给定的 CPU 功能是否可用。如果是，则使用该功能。例如，在 ARM 处理器上，将执行 `hex_encode_fallback`。在 x86 处理器上，将根据芯片的确切能力执行 avx2 或 sse4.1 之一的实现。哪个实现被调用是在运行时确定的。让我们首先看看 `hex_encode_fallback` 的实现：

```rs
fn hex_encode_fallback<'a>(
    src: &[u8], dst: &'a mut [u8]
) -> Result<&'a str, usize> {
    fn hex(byte: u8) -> u8 {
        static TABLE: &[u8] = b"0123456789abcdef";
        TABLE[byte as usize]
    }

    for (byte, slots) in src.iter().zip(dst.chunks_mut(2)) {
        slots[0] = hex((*byte >> 4) & 0xf);
        slots[1] = hex(*byte & 0xf);
    }

    unsafe { Ok(str::from_utf8_unchecked(&dst[..src.len() * 2])) }
}
```

内部函数 `hex` 作为一个静态查找表，for 循环将 `src` 中的字节和从 `dst` 中拉取的两个块组合在一起，在循环过程中更新 `dst`。最后，`dst` 被转换为 `&str` 并返回。在这里使用 `from_utf8_unchecked` 是安全的，因为我们能验证 `dst` 中不可能存在非 utf8 字符，从而节省了检查。

好吧，现在这是后备方案。SIMD 变体是如何读取的？我们将查看 `hex_encode_avx2`，将 sse4.1 变体留给有抱负的读者。首先，我们看到条件编译和熟悉的功能类型再次出现：

```rs
#[target_feature(enable = "avx2")]
#[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
unsafe fn hex_encode_avx2<'a>(
    mut src: &[u8], dst: &'a mut [u8]
) -> Result<&'a str, usize> {
```

到目前为止，一切顺利。现在，看看以下内容：

```rs
    let ascii_zero = _mm256_set1_epi8(b'0' as i8);
    let nines = _mm256_set1_epi8(9);
    let ascii_a = _mm256_set1_epi8((b'a' - 9 - 1) as i8);
    let and4bits = _mm256_set1_epi8(0xf);
```

好吧，函数 `_mm256_set1_epi8` 确实是新的！文档（[`rust-lang-nursery.github.io/stdsimd/x86_64/stdsimd/arch/x86_64/fn._mm256_set1_epi8.html`](https://rust-lang-nursery.github.io/stdsimd/x86_64/stdsimd/arch/x86_64/fn._mm256_set1_epi8.html)）中说明了以下内容：

"将 8 位整数 a 广播到返回向量的所有元素。"

好的。哪个向量？许多 SIMD 指令都是以隐式大小的向量的形式执行的，大小由 CPU 架构定义。例如，我们一直在查看的函数返回一个 `stdsimd::arch::x86_64::__m256i`，一个 256 位宽的向量。所以，`ascii_zero` 是 256 位的零，`ascii_a` 是 256 位的 `a`，以此类推。顺便说一句，这些函数和类型都遵循英特尔命名方案。我理解 LLVM 使用自己的命名方案，但 Rust 为了减少开发者混淆，维护了一个从架构名称到 LLVM 名称的映射。通过保留架构名称，Rust 使得查找每个内嵌函数的详细信息变得非常容易。您可以在 `mm256_set1_epi8` 的 *英特尔内嵌函数指南* 中看到这一点（[`software.intel.com/sites/landingpage/IntrinsicsGuide/#text=_mm256_set1_epi8&expand=4676`](https://software.intel.com/sites/landingpage/IntrinsicsGuide/#text=_mm256_set1_epi8&expand=4676))。

实现然后进入一个循环，以 256 位宽的块处理 `src`，直到只剩下 32 位或更少：

```rs
    let mut i = 0_isize;
    while src.len() >= 32 {
        let invec = _mm256_loadu_si256(src.as_ptr() as *const _);
```

变量 `invec` 现在是 `src` 的 256 位，`_mm256_loadu_si256` 负责执行加载。SIMD 指令必须在其原生类型上操作，您明白的。在机器的实际现实中，没有类型，但有专用寄存器。接下来要讨论的内容很复杂，但我们可以通过推理来理解：

```rs
        let masked1 = _mm256_and_si256(invec, and4bits);
        let masked2 = _mm256_and_si256(
            _mm256_srli_epi64(invec, 4),
            and4bits
        );
```

好的，`_mm256_and_si256` 执行两个 `__m256i` 的按位布尔与操作，返回结果。`masked1` 是 `invec` 和 `and4bits` 的按位布尔与操作。对于 `masked2` 也是如此，但我们需要确定 `_mm256_srli_epi64` 做了什么。英特尔文档中说明了以下内容：

"将打包的 64 位整数在 dst 中右移 imm8 位，同时移入零，并将结果存储在 dst 中。"

`invec` 被细分为四个 64 位整数，然后这些整数右移四位。然后与 `and4bits` 进行布尔与操作，`masked2` 就出现了。如果您回顾 `hex_encode_fallback` 并稍微眯一下眼睛，您就可以开始看到它与 `(*byte >> 4) & 0xf` 的关系。接下来，两个比较掩码被组合在一起：

```rs
        let cmpmask1 = _mm256_cmpgt_epi8(masked1, nines);
        let cmpmask2 = _mm256_cmpgt_epi8(masked2, nines);
```

`_mm256_cmpgt_epi8` 对 256 位向量中的 8 位块进行大于比较，返回较大的字节。然后使用比较掩码来控制 `ascii_zero` 和 `ascii_a` 的混合，并将该结果添加到 `masked1`：

```rs
        let masked1 = _mm256_add_epi8(
            masked1,
            _mm256_blendv_epi8(ascii_zero, ascii_a, cmpmask1),
        );
        let masked2 = _mm256_add_epi8(
            masked2,
            _mm256_blendv_epi8(ascii_zero, ascii_a, cmpmask2),
        );
```

混合是按字节进行的，每个字节的最高位决定是否将字节从 `arg` 列表中的第一个向量或第二个向量移动过来。然后计算出的掩码交错选择高半部和低半部：

```rs
        let res1 = _mm256_unpacklo_epi8(masked2, masked1);
        let res2 = _mm256_unpackhi_epi8(masked2, masked1);
```

在交错掩码的高半部分，目标地址的前 128 位被填充自源地址的最高 128 位，其余位用零填充。在交错掩码的低半部分，目标地址的前 128 位被填充自源地址的最低 128 位，其余位用零填充。最后，计算出一个指向`dst`的指针，以 64 位步长进行计算，并进一步细分为 16 位块，然后在这里存储`res1`和`res2`：

```rs
        let base = dst.as_mut_ptr().offset(i * 2);
        let base1 = base.offset(0) as *mut _;
        let base2 = base.offset(16) as *mut _;
        let base3 = base.offset(32) as *mut _;
        let base4 = base.offset(48) as *mut _;
        _mm256_storeu2_m128i(base3, base1, res1);
        _mm256_storeu2_m128i(base4, base2, res2);
        src = &src[32..];
        i += 32;
    }
```

当`src`小于 32 字节时，会对其调用`hex_encode_sse4`，但我们将细节留给有冒险精神的读者：

```rs
    let i = i as usize;
    let _ = hex_encode_sse41(src, &mut dst[i * 2..]);

    Ok(str::from_utf8_unchecked(
        &dst[..src.len() * 2 + i * 2],
    ))
}
```

这很困难！如果我们在这里的目标不仅仅是了解 SIMD 编程风格，那么我们可能需要手动分析这些指令在位级别上的工作方式。这值得吗？这个例子来自 RFC 2325 ([`github.com/rust-lang/rfcs/blob/master/text/2325-stable-simd.md`](https://github.com/rust-lang/rfcs/blob/master/text/2325-stable-simd.md))，*稳定的 SIMD*。他们的基准测试表明，我们最初查看的回退实现每秒大约可以散列 600 兆字节。avx2 实现每秒散列 15,000 MB，性能提升了 60%。这并不算差。而且要记住，每个 CPU 核心都配备了 SIMD 能力。你还可以将这些计算分配给线程。

就像原子编程一样，SIMD 既不是万能的，也不是特别容易。不过，如果你需要速度，那么这些努力是值得的。

# Futures 和 async/await

异步 I/O 现在是 Rust 生态系统中的热门话题。我们在第八章“高级并行性 – 线程池、并行迭代器和进程”中简要讨论的 C10K 挑战，在操作系统引入可伸缩的 I/O 通知系统调用（如 BSD 中的 kqueue 和 Linux 中的 epoll）时得到了解决。在 epoll、kqueue 及其朋友出现之前，I/O 通知的处理时间随着文件描述符数量的增加而呈线性增长——这是一个真正的问题。本书中我们采取的积极而天真的网络方法也遭受了线性增长。我们为监听而打开的每个 TCP 套接字都需要在循环中进行轮询，这导致高流量连接得不到充分服务，而其他连接则得到了过多的服务。

Rust 的可扩展网络 I/O 抽象是 mio ([`crates.io/crates/mio`](https://crates.io/crates/mio))。如果你在 mio 内部探索，你会发现它是一些常见平台（Windows、Unixen）的 I/O 通知子系统的干净适配器：这就是全部了。mio 的用户不会得到很多额外的支持。你的 mio 套接字存储在哪里？这取决于程序员。你能包含多线程吗？由你决定。现在，这对非常具体的用例来说很棒——例如，cernan 使用 mio 来驱动其入口 I/O，因为我们希望对这些细节有非常精细的控制——但除非你有非常具体的需求，否则这可能会对你来说是一个非常繁琐的情况。社区已经在 tokio ([`crates.io/crates/tokio`](https://crates.io/crates/tokio))上投入了大量的精力，这是一个用于执行可扩展 I/O 的框架，它具有如背压、事务取消和高级抽象等基本功能，以简化请求/响应协议。Tokio 是一个快速发展的项目，但几乎肯定将成为未来几年中的*关键*项目之一。Tokio 在本质上是一个基于反应器（[`en.wikipedia.org/wiki/Reactor_pattern`](https://en.wikipedia.org/wiki/Reactor_pattern)）的架构；一个 I/O 事件变得可用，然后调用你已注册的处理程序。

如果你只需要响应一个事件，基于反应器的系统编程足够简单。但是，比你想象的更常见的是，在真实系统中，I/O 事件之间存在依赖关系——一个进入的 UDP 数据包会导致对网络内存缓存的 TCP 往返，这又导致对原始进入系统的出口 UDP 数据包，以及另一个。协调这一点是困难的。Tokio——以及 Rust 生态系统在某种程度上——完全投入了一个名为 futures ([`crates.io/crates/futures`](https://crates.io/crates/futures))的承诺库。承诺是对异步 I/O 的一个相当整洁的解决方案：底层的反应器调用一个已建立的接口，你在其中实现一个闭包（或一个特质，然后将其放入该接口），这样每个人的代码都保持松散耦合。Rust 的 futures 是可轮询的，这意味着你可以在一个 future 上调用 poll 函数，它可能解析为一个值或一个通知，表明该值尚未准备好。

这与其他语言的承诺系统类似。但是，正如任何记得 NodeJS 早期引入 JavaScript 生态系统的人都可以证明的那样，没有语言支持的承诺会变得很奇怪，并且很快就会变得非常嵌套。Rust 的借用检查器使得情况变得更糟，需要装箱或完全的弧，而在直线代码中，这本来是不必要的。

将承诺作为一等关注点的语言通常会演变 async/await 语法。异步函数是 Rust 中返回未来而几乎没有其他内容的函数。Rust 的未来是一个简单的特质，它从 futures 项目中提取出来。实际的实现将存在于语言的标准库之外，并允许有替代实现。异步函数的内部不会立即执行，而只有在未来被轮询时才会执行。函数的内部通过轮询执行，直到遇到 await 点，导致一个通知，表明值尚未准备好，并且正在轮询未来的任何内容都应该继续进行。

正如其他语言一样，Rust 提出的 async/await 语法实际上是在显式回调链结构之上的糖衣。但是，由于 async/await 语法和 futures 特质正在进入编译器，可以向借用检查系统添加规则以消除当前的装箱关注点。我预计，一旦它们变得更容易交互，你将看到更多稳定版 Rust 中的 futures。确实，正如我们在第八章《高级并行性——池、迭代器和进程》中看到的那样，rayon 正在投资时间开发基于 futures 的接口。

# 特定化

在第二章《顺序 Rust 性能和测试》中，我们提到，如果数据结构实现者打算针对稳定版 Rust 进行目标开发，*特定化*技术将被切断。特定化是一种方法，通过这种方法，一个泛型结构——比如一个`HashMap`——可以针对一个特定类型进行具体实现。通常，这样做是为了提高性能——比如将具有`u8`键的`HashMap`转换为数组——或者在可能的情况下提供一个更简单的实现。特定化在 Rust 中已经存在了一段时间，但由于与生命周期交互时的稳定性问题，它仅限于夜间项目，如编译器。经典的例子一直是以下内容：

```rs
trait Bad1 {
    fn bad1(&self);
}

impl<T> Bad1 for T {
    default fn bad1(&self) {
        println!("generic");
    }
}

// Specialization cannot work: trans doesn't know if T: 'static
impl<T: 'static> Bad1 for T {
    fn bad1(&self) {
        println!("specialized");
    }
}

fn main() {
    "test".bad1()
}
```

生命周期在编译过程中的某个点被消除。字符串字面量是静态的，但在调用特定类型之前，编译器中信息不足，无法做出决定。在特定化中的生命周期等价性也带来了困难。

有很长一段时间，人们认为特定化将错过 2018 年语言中包含的机会。现在，*也许*将包含一种更有限的特定化形式。只有当实现是关于生命周期的泛型时，特定化才会适用，它不会在特质生命周期中引入重复，不会重复泛型类型参数，并且所有特质界限本身都适用于特定化。具体的细节在我写这篇文章时还在变化，但请务必留意。

# 有趣的项目

目前 Rust 社区中有很多有趣的项目。在本节中，我想探讨两个我在书中没有详细讨论的项目。

# 模糊测试

在前面的章节中，我们使用 AFL 来验证我们的程序没有出现崩溃行为。虽然 AFL 非常常用，但它并不是 Rust 可用的唯一 fuzzer。LLVM 有一个本地的库——libfuzzer ([`llvm.org/docs/LibFuzzer.html`](https://llvm.org/docs/LibFuzzer.html))——覆盖了相同的空间，而 cargo-fuzz ([`crates.io/crates/cargo-fuzz`](https://crates.io/crates/cargo-fuzz))项目则充当执行器。你可能也对 honggfuzz-rs ([`crates.io/crates/honggfuzz`](https://crates.io/crates/honggfuzz))感兴趣，这是一个由 Google 开发的 fuzzer，用于搜索与安全相关的违规行为。它是原生多线程的——无需手动启动多个进程——并且可以进行网络 fuzzing。我传统上更喜欢使用 AFL 进行 fuzzing。honggfuzz 项目势头强劲，读者应该在他们的项目中尝试一下。

# Seer，Rust 的符号执行引擎

如我们之前讨论的，fuzzing 通过随机输入和二进制仪器探索程序的状态空间。这可能会很慢。符号执行 ([`en.wikipedia.org/wiki/Symbolic_execution`](https://en.wikipedia.org/wiki/Symbolic_execution)) 的雄心是允许对状态空间进行相同的探索，但无需随机探测。搜索程序崩溃是应用的一个领域，但它也可以与证明工具一起使用。在精心编写的程序中，符号执行可以让你证明你的程序永远不会达到错误状态。Rust 有一个部分实现的符号执行工具 seer ([`github.com/dwrensha/seer`](https://github.com/dwrensha/seer))。该项目使用 z3，一个约束求解器，在程序分支处生成分支输入。Seer 的 README，截至 SHA `91f12b2291fa52431f4ac73f28d3b18a0a56ff32`，有趣地解码了一个哈希值。这是通过定义哈希数据的解码为可崩溃条件来完成的。Seer 运行了一段时间后解码了哈希值，导致程序崩溃。它将解码的值报告在错误报告中。

对于 seer 来说，现在还处于早期阶段，但潜力是存在的。

# 社区

Rust 社区庞大且多元化。实际上，它如此之大，以至于很难知道在有问题或想法时该去哪里。更重要的是，中级到高级水平的书籍通常会假设读者对语言社区的了解与作者一样多。虽然我一直处于作者和读者关系的另一边，但我一直觉得这种假设令人沮丧。然而，作为一个作者，我现在理解了这种犹豫——社区信息不会保持更新。

唉，有些信息在你看到的时候可能已经过时了。读者请注意。

在整本书中，我们提到了 crates 生态系统；crates.io（[`crates.io/`](https://crates.io/））是 Rust 源项目的**唯一**位置。docs.rs（[https://docs.rs/](https://docs.rs/））是理解 crates 的重要资源，由 Rust 团队运营。您还可以使用`cargo docs`来获取您项目依赖文档的本地副本。我经常没有 Wi-Fi，我发现这非常有用。

与 Rust 中的几乎所有事物一样，社区本身在这网页上有文档（[https://www.rust-lang.org/en-US/community.html](https://www.rust-lang.org/en-US/community.html)）。IRC 对于实时、松散的对话来说相当常见，但 Rust 的沟通确实侧重于面向网络的方面。用户论坛（[`users.rust-lang.org/`](https://users.rust-lang.org/））是一个面向使用该语言并有问题或公告的人的网页论坛。编译器人员以及标准库实现者等都在内部论坛（[https://internals.rust-lang.org/](https://internals.rust-lang.org/））上闲逛。正如您可能预期的那样，这两个社区有很大的重叠。

在整本书中，我们引用了各种 RFC。Rust 通过一个请求评论系统进行演变，所有这些都被集中管理（[https://github.com/rust-lang/rfcs](https://github.com/rust-lang/rfcs)）。这仅针对 Rust 编译器和标准库。许多社区项目遵循类似的系统——例如，crossbeam RFCs（[`github.com/crossbeam-rs/rfcs`](https://github.com/crossbeam-rs/rfcs)）。顺便说一句，这些值得阅读。

# 我应该使用`unsafe`吗？

听到以下观点的变体并不罕见——*我不会使用任何包含`unsafe`块的库*。这种观点背后的理由是`unsafe`，嗯，表明该库可能存在潜在的不安全性，并可能导致您精心设计的程序崩溃。这是真的——某种程度上。正如我们在本书中看到的，使用`unsafe`构建一个在运行时完全安全的项目的可能性是存在的。我们也看到，不使用`unsafe`块构建的项目在运行时也可能失败。`unsafe`块的存在与否不应减少原始程序员对尽职调查的责任——编写测试、使用模糊测试工具探测实现等。此外，`unsafe`块的存在与否也不免除 crates 用户承担同样的责任。任何软件，在某种程度上，如果没有被证明是安全的，都应该被视为可疑。

勇敢地使用`unsafe`关键字。在必要时这样做。记住，编译器将无法像通常那样为您提供帮助——仅此而已。

# 摘要

在本书的最后一章中，我们讨论了即将到来的 Rust 语言的近中期改进——在野外一段时间后稳定的功能，大型项目的基础集成，可能或可能不成功的研发，以及其他主题。其中一些主题在本书的第二版中肯定会有自己的章节，如果本书有新版本的话（告诉你的朋友去买一本吧！）。我们还谈到了社区，这是寻找持续讨论、提问和参与的地方。然后，我们确认了，是的，当你有理由这样做时，你应该在 unsafe Rust 中编写代码。

好了，这就结束了。这本书的内容到此结束。

# 进一步阅读

+   *交付专长：一个关于稳健性的故事*，可在 [`aturon.github.io/blog/2017/07/08/lifetime-dispatch/`](https://aturon.github.io/blog/2017/07/08/lifetime-dispatch/) 查阅。长期以来，人们一直渴望看到专长在稳定版 Rust 中可用，而且并非没有尝试过，但至今仍未实现。在这篇博客文章中，Aaron Turon 讨论了 2017 年专长的困难，并在讨论中介绍了 Chalk 逻辑解释器。Chalk 本身就很有趣，尤其是如果你对编译器内部或逻辑编程感兴趣。

+   *最大最小化专长：始终适用的 impl*，可在 [`smallcultfollowing.com/babysteps/blog/2018/02/09/maximally-minimal-specialization-always-applicable-impls/`](http://smallcultfollowing.com/babysteps/blog/2018/02/09/maximally-minimal-specialization-always-applicable-impls/) 查阅。Niko Matsakis 在这篇文章中扩展了 Turon 的 *交付专长* 文章中讨论的主题，讨论了专长稳健性的最小-最大解决方案。这种方法似乎是最有可能最终实现的方法，但发现了缺陷。幸运的是，这些缺陷是可以解决的。这篇帖子是 Rust 社区中预 RFC 对话的一个很好的例子。

+   *针对 Rust 的声音和人体工程学专长*，可在 [`aturon.github.io/2018/04/05/sound-specialization/`](http://aturon.github.io/2018/04/05/sound-specialization/) 查阅。这篇博客文章讨论了最小-最大文章中的问题，并提出了本章讨论的实现方案。截至撰写本书时，这可能是最有可能实现的方法，除非有人发现了其中的缺陷。

+   *Chalk*，可在 [`github.com/rust-lang-nursery/chalk`](https://github.com/rust-lang-nursery/chalk) 查阅。Chalk 真的是一个非常有意思的 Rust 项目。根据项目描述，它是一个用 Rust 编写的 *PROLOG 风格的解释器*。到目前为止，Chalk 正被用来在 Rust 语言中推理专长，但计划有一天将其合并到 `rustc` 本身。截至 `SHA 94a1941a021842a5fcb35cd043145c8faae59f08` 的项目 README 中列出了关于 chalk 应用的优秀文章。

+   *《不稳定的书籍》*，可在[`doc.rust-lang.org/nightly/unstable-book/`](https://doc.rust-lang.org/nightly/unstable-book/)找到。`rustc`拥有大量正在开发中的特性。*《不稳定的书籍》*旨在收集这些特性、它们背后的理由以及任何相关的跟踪问题，值得一读，尤其是如果你打算为编译器项目做出贡献的话。

+   *Rust 中的零成本 future*，可在[`aturon.github.io/blog/2016/08/11/futures/`](http://aturon.github.io/blog/2016/08/11/futures/)找到。这篇文章向 Rust 社区介绍了 future 项目，并很好地解释了引入它的动机。随着时间的推移，future 的实际实现细节已经发生了变化，但这篇文章仍然值得一读。

+   *异步/等待*，可在[`github.com/rust-lang/rfcs/blob/master/text/2394-async_await.md`](https://github.com/rust-lang/rfcs/blob/master/text/2394-async_await.md)找到。RFC 2394 介绍了简化 Rust 中 future 的易用性的动机，并概述了其实施方法。该 RFC 本身是 Rust 如何演化的一个很好的例子——社区的需求转化为实验，然后转化为编译器的支持。
