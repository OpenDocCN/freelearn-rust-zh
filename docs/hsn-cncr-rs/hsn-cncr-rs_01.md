# 预备知识 – 机器架构和 Rust 入门

在本章中，我们将处理本书的预备知识，确保涵盖构建本书其余部分所需的基本内容。本书是关于 Rust 编程语言中的并行编程。因此，了解现代计算硬件的高级概念是至关重要的。我们需要了解现代 CPU 的工作方式，内存总线如何影响 CPU 执行工作的能力，以及对于计算机能够同时做很多事情意味着什么。此外，我们将讨论验证您的 Rust 安装，并涵盖为本书关注的两种 CPU 架构生成可执行文件。

到本章结束时，我们将：

+   讨论了 CPU 操作的概要模型

+   讨论了计算机内存的概要模型

+   对 Rust 内存模型进行了初步讨论

+   调查了为 x86 和 ARM 生成可运行的 Rust 程序

+   调查了这些程序的调试

# 技术要求

本章需要任何可以安装 Rust 的现代计算机。具体细节将在下面详细说明。感兴趣的读者可以选择投资 ARM 开发板。我在写作时使用了 Raspberry Pi 3 Model B。下面使用了 Valgrind 工具套件。许多操作系统捆绑了 Valgrind 包，但您可以在[`valgrind.org/`](http://valgrind.org/)找到您系统的进一步安装说明。使用了`gdb`和`lldb`工具，这些工具通常与 gcc 和 llvm 编译器工具链一起安装。

您可以在 GitHub 上找到本书项目的源代码：[`github.com/PacktPublishing/Hands-On-Concurrency-with-Rust`](https://github.com/PacktPublishing/Hands-On-Concurrency-with-Rust)。本章没有相关的源代码。

# 机器

在这本书中，无论具体是哪种 Rust 技术，我们都会尝试教授一种与现代并行计算机的**机械亲和力**。我们将涉及两种模型类型的并行性——并发内存操作和数据并行性。在这本书的大部分时间里，我们将专注于并发内存操作，这种并行性中，多个 CPU 争夺操作共享的、可寻址的内存。数据并行性，即 CPU 能够同时使用单一或多个指令对多个字进行操作，也会涉及到，但具体细节是 CPU 特定的，并且随着这本书的出版，必要的内建函数才刚刚在基础语言中变得可用。幸运的是，作为具有现代库管理的系统语言，Rust 将使我们能够轻松地引入适当的库并发出正确的指令，或者我们可以自己内联汇编。

关于并行机器算法的抽象构造的文献必须选择一个机器模型来操作。在文献中，并行随机访问机（PRAM）很常见。在这本书中，我们将关注两种具体的机器架构：

+   x86

+   ARM

这些机器被选择是因为它们很常见，并且它们各自具有在第六章，“原子——同步的基本原理”中非常重要的特定属性。实际机器在重要方面偏离了 PRAM 模型。最明显的是，实际机器具有有限的 CPU 数量和有限的 RAM 量。内存位置不是从每个 CPU 均匀可访问的；实际上，缓存层次对计算机程序的性能有重大影响。这并不是说 PRAM 是一种荒谬的简化，任何你在文献中找到的其他模型也是如此。应该理解的是，随着我们的工作，我们将需要出于必要而绘制抽象的线条，因为进一步的细节并不能提高我们解决问题的能力。我们还必须理解我们的抽象如何适合我们的工作，以及它们如何与其他人的抽象相关联，以便我们可以学习和分享。在这本书中，我们将关注理解我们机器的经验方法，包括仔细的测量、汇编代码的检查和替代实现的实验。这将被结合我们机器的抽象模型，比 PRAM 更具体，但在细节上仍然关注总缓存层、缓存大小、总线速度、微代码版本等等。鼓励读者在需要时添加更多具体性，如果他们感到有信心这么做。

# CPU

CPU 是一种解释指令流并在这个过程中操作与其连接的存储和其他设备的设备。CPU 的最简单模型，也是通常在本科计算机科学中首先介绍的是，CPU 从一个模糊的地方接收一条指令，执行解释，接收下一条指令，解释它，以此类推。CPU 通过振荡电路维持内部节奏，所有指令都需要一定数量的振荡脉冲——或者称为*时钟周期*——来执行。在某些 CPU 模型中，每条指令将执行相同数量的时钟周期，而在其他模型中，指令的周期数将有所不同。一些 CPU 指令修改寄存器，具有非常低的延迟，并且 CPU 中内置了专门但极其有限的内存位置。其他 CPU 指令修改主存储器，RAM。其他指令在寄存器和 RAM 之间或反之移动或复制信息。RAM——其读写延迟远高于寄存器，但数量更多——以及其他存储设备，如 SSD，通过专用总线连接到 CPU。

这些总线的确切性质、它们的带宽和传输延迟，在不同机器架构之间是不同的。在一些系统中，RAM 中的每个位置都是可寻址的——这意味着它可以在恒定时间内从可用的 CPU 中读取或写入。在其他系统中，情况并非如此——一些 RAM 是 CPU 本地的，而一些是 CPU 远程的。一些指令控制特殊的硬件中断，导致内存范围被写入其他总线连接的存储设备。大多数情况下，与 RAM 相比，这些设备极其缓慢，而与寄存器相比，RAM 本身也较慢。

所有这些都是为了解释在 CPU 的最简单模型中，即指令按顺序执行的情况下，指令可能会在等待读取或写入执行的过程中停滞许多时钟周期。为此，重要的是要理解几乎所有的 CPU——特别是我们将在本书中关注的 CPU——都会执行指令的乱序执行。只要 CPU 能够证明两条指令序列对内存的访问是明显不同的——也就是说，它们不会相互干扰——那么 CPU 就是自由的，并且很可能会重新排序指令。这正是 C 语言关于未初始化内存的未定义行为如此有趣的原因之一。也许你的程序已经填充了内存，也许没有。乱序执行使得推理处理器的行为变得困难，但它的好处是 CPU 可以通过延迟在某种内存访问上停滞的指令序列来更快地执行程序。

同样地，大多数现代 CPU——特别是我们将在本书中关注的 CPU——执行*分支预测*。假设我们的程序中的分支在执行时倾向于以相同的方式分支——例如，假设我们有一个配置为在程序生命周期内启用的功能标志测试。执行分支预测的 CPU 将在等待其他停滞的指令流水线指令时，*推测性地执行*分支的一侧，该分支倾向于以某种方式分支。当分支指令序列赶上分支测试时，如果测试结果与预测一致，那么已经完成了很多工作，指令序列可以跳得很远。不幸的是，如果分支预测错误，那么所有这些先前的工作都必须被拆解并丢弃，并且必须计算正确的分支，这相当昂贵。这就是为什么你会发现，那些担心他们程序性能细节的程序员往往会担心分支，并试图移除它们。

所有这些都非常耗电，需要重新排序指令以避免停顿，或者提前执行可能会被丢弃的计算。耗电意味着发热，意味着需要冷却，这意味着更多的电力消耗。所有这些对于技术文明的可持续性来说并不一定好，这取决于所有这些电力的产生方式和废热的排放地点。为此，许多现代 CPU 集成了某种类型的电源调节功能。一些 CPU 会延长其时钟脉冲之间的时间，这意味着在一段时间内它们执行的指令比它们可能执行的要少。其他 CPU 会像平时一样快速前进，然后关闭一段时间，同时消耗最少的电力和冷却。构建和运行高效能 CPU 的确切方法远远超出了本书的范围。重要的是要理解，由于所有这些，在其他条件相同的情况下，你的程序执行速度会因运行而异，因为 CPU 会决定是否需要节省电力。我们将在下一章中看到这一点，当我们手动设置省电设置时。

# 内存和缓存

CPU 的内存存储，在上一个章节中提到，相当有限。它仅限于通用寄存器中的一小部分单词，以及在有限情况下的专用寄存器。寄存器由于构造和在芯片上的位置而非常快，但它们并不适合存储。现代机器通过总线或总线将 CPU 连接到*主内存*，这是一个非常大的随机可寻址字节的块。这种随机可寻址很重要，因为它意味着，与其他类型的存储不同，从 RAM 中检索第 0 个字节的成本与检索第 1,000,000,000 个字节的成本是相同的。我们程序员不需要做任何愚蠢的技巧来确保我们的结构出现在 RAM 的前面，以便更快地检索或修改，而物理位置在存储中对于旋转硬盘和磁带驱动器来说是一个紧迫的问题。我们的 CPU 如何与内存交互在不同的平台上有所不同，接下来的讨论在很大程度上得益于 Mark Batty 在 2014 年出版的《C11 和 C++11 并发模型》一书中对描述的借鉴。

在一个暴露顺序一致内存访问模型的机器中，每个内存加载或存储操作都必须与其他操作同步进行，包括在具有多个 CPU 或每个 CPU 有多个执行线程的系统中的情况。这限制了重要的优化——考虑在顺序一致模型中绕过内存停顿的挑战——因此，本书中考虑的处理器都不提供这种模型。它*会*以名称出现在文献中，因为这种模型易于推理，值得了解，尤其是在研究原子文献时。

x86 平台表现得像顺序一致，除了每个执行线程在写入被刷新到主内存之前都维护一个 FIFO 缓冲区。此外，还有一个全局锁用于协调执行线程之间的原子读和写操作。这里有很多东西需要解释。首先，我使用*load*和*store*与*read*和*write*互换使用，正如大多数文献所做的那样。还有平载荷/存储和*原子*载荷/存储之间的变体区别。原子载荷/存储是特殊的，因为它们的效果可以在执行线程之间进行协调，从而允许以不同级别的保证进行协调。x86 处理器平台提供了强制刷新写入缓冲区的栅栏指令，这会阻止其他尝试访问已写入主内存范围的线程执行，直到刷新完成。这就是全局锁的作用。如果没有原子性，写入将会随意刷新。在 x86 平台上，对内存中可寻址位置的写入是一致的，这意味着它们是全局有序的，并且读取将按照写入的顺序看到该位置上的值。与 ARM 相比，x86 上实现这一功能的方式非常简单——写入直接写入主内存。

让我们来看一个例子。从 Batty 的*优秀*论文中汲取灵感，考虑一个场景，其中父线程将两个变量*x*和*y*设置为 0，然后创建了两个线程，比如叫做`A`和`*B*`。线程`A`负责将*x*设置为，然后*y*设置为 1，而线程`B`负责将`*x*`的值加载到一个名为`thr_x`的线程局部变量中，将`*y*`的值加载到一个名为`thr_y`的线程局部变量中。这看起来可能如下所示：

```rs
WRITE x := 0
WRITE y := 0
[A] WRITE x := 1
[A] WRITE y := 1
FLUSH A
[B] READ x
[B] WRITE thr_x := x
[B] READ y
[B] WRITE thr_y := y
```

在这个特定的例子中，`thr_x == 1` 和 `thr_y == 1`。如果 CPU 按照不同的顺序执行刷新操作，这个结果就会不同。例如，看看以下内容：

```rs
WRITE x := 0
WRITE y := 0
[A] WRITE x := 1
[A] WRITE y := 1
[B] READ x
[B] WRITE thr_x := x
FLUSH A
[B] READ y
[B] WRITE thr_y := y
```

这种情况的结果是`thr_x == 0`和`thr_y == 1`。在没有其他协调的情况下，唯一的其他有效交错是`thr_x == 0`和`thr_y == 0`。也就是说，由于 x86 上的写缓冲区，线程 A 中`*x*`的写入永远不会重新排序到在`y: thr_x == 1`和`thr_y == 0`的写入之后发生。除非你喜欢将这个小程序作为客厅魔术，否则这真的很糟糕。我们希望程序具有确定性。为此，x86 提供了不同的栅栏和锁指令，这些指令控制线程何时以及如何刷新它们的本地写缓冲区，以及何时以及如何从主内存中读取字节数据。这里的精确交互是复杂的。我们将在第三章《Rust 内存模型——所有权、引用和操作》和第六章《原子操作——同步的原始操作》中详细讨论这个问题。简单来说，现在有一个`SFENCE`指令可用，它可以强制实现顺序一致性。我们可以如下使用这个指令：

```rs
WRITE x := 0
WRITE y := 0
[A] WRITE x := 1
[A] WRITE y := 1
[A] SFENCE        [B] SFENCE
                  [B] READ x
                  [B] WRITE thr_x := x
                  [B] READ y
                  [B] WRITE thr_y := y
```

由此，我们得到`thr_x == 1`和`thr_y == 1`。ARM 处理器略有不同——1/0 是一个有效的交错。ARM 没有全局锁，没有栅栏，也没有写缓冲区。在 ARM 中，一系列指令——称为传播列表，原因将在后面阐明——处于未提交状态，这意味着发起指令的线程会将其视为正在执行，但它们尚未传播到其他线程。这些未提交的指令可能已经执行——导致内存中的副作用，或者没有执行，允许进行推测执行和在前一节中讨论的其他性能技巧。具体来说，读取可能从线程的本地传播列表中满足，但写入则不行。分支指令会导致导致分支指令的传播列表被提交，可能不是程序员指定的顺序。内存子系统会跟踪哪些传播列表被发送到哪些线程，这意味着线程的私有加载和存储可能是不按顺序的，而提交的加载和存储在线程之间可能看起来是不按顺序的。在 ARM 上，通过内存子系统的更积极参与来维护一致性。

在 x86 架构中，主内存是执行操作的对象，而在 ARM 架构中，内存子系统会对请求做出响应，并且可能会使之前的未提交请求失效。写请求涉及一个读响应事件，通过这个事件可以唯一识别，而读请求必须引用一个写操作。

这设置了一个*数据依赖链*。读响应可能在稍后失效，但写操作则不行。每个位置都会收到一个*一致性提交*排序，它记录了全局的写操作顺序，这个顺序是按照线程将传播列表传播到线程时构建的。

这非常复杂。最终结果是，对线程视图中的内存的写入可能不是按照程序员顺序进行的，提交到主内存的写入也可能不是按顺序进行的，并且读取请求也可能不是按程序员顺序进行的。因此，以下代码是完美的，因为我们示例中没有分支：

```rs
WRITE x := 0
WRITE y := 0
[A] WRITE x := 1
[B] READ x
[A] WRITE y := 1
[B] WRITE thr_y := y
[B] WRITE thr_x := x
[B] READ y
```

ARM 提供了控制依赖关系的指令，称为屏障。ARM 中有三种依赖类型。地址依赖意味着使用加载操作来计算访问内存的地址。控制依赖意味着导致内存访问的程序流程依赖于一个加载操作。我们已经熟悉数据依赖。

虽然有大量的主内存可用，但它并不特别快。好吧，让我们澄清一下。这里的“快”是一个相对术语。在真空中，光子每 1 纳秒大约可以传播 30.5 厘米，这意味着在理想情况下，我位于美国西海岸，应该能够向美国东海岸发送一条消息，并在大约 80 毫秒内收到回复。当然，我在忽略请求处理时间、互联网的现实以及其他因素；我们只是在处理大致的数字。考虑到 80 毫秒是 8,000,000 纳秒。从 CPU 到主内存的读取操作大约需要 100 纳秒，具体取决于你的特定计算机架构、芯片的细节以及其他因素。所有这些都是为了澄清，当我们说“并不特别快”时，我们是在正常人类时间尺度之外工作的。

有时很难避免误判事物是否足够快。比如说，我们有一个 4 GHz 的处理器。每纳秒我们有多少个时钟周期？结果是 4 个。比如说，我们有一个需要访问主内存的指令——记住，这需要 100 纳秒——并且恰好能在 4 个周期内，或者 1 纳秒内完成其工作。那么这条指令将在等待时停滞 99 纳秒，这意味着我们可能失去了 99 条本可以由 CPU 执行的指令。CPU 将通过其优化技巧弥补一些损失。这些技巧只能走这么远，除非我们的计算非常、非常幸运。

为了避免主内存对计算性能的影响，处理器制造商在处理器和主内存之间引入了*缓存*。如今，许多机器都有三层*数据缓存*，以及一个*指令缓存*，称为 dCACHE 和 iCACHE，这些工具我们将在后面使用。您会发现 dCACHE 现在通常由三层组成，每一层都比上一层大，但出于成本或功耗的考虑也较慢。最低、最小的层被称为 L1，接下来是 L2，以此类推。CPU 从主内存中以工作块——或者简单地说是块——的形式读取到缓存中，这些块的大小与被读取的缓存大小相同。内存访问将优先在 L1 缓存上进行，然后是 L2，然后是 L3，以此类推，每个级别的缺失和块读取都会带来时间上的惩罚。然而，缓存命中却比直接访问主内存快得多。L1 dCACHE 引用为 0.5 纳秒，足够快，以至于我们假设的 4 周期指令比所需的内存访问慢 8 倍。L2 dCACHE 引用为 7 纳秒，仍然比主内存好得多。当然，具体的数字会因系统而异，我们将在下一章中做很多工作来直接测量它们。缓存一致性——保持高命中率与缺失率的比例——是构建现代机器上快速软件的一个重要组成部分。在这本书中，我们永远无法摆脱 CPU 缓存。

# 内存模型

处理器处理内存的细节既复杂又非常特定于该处理器。编程语言——在这里 Rust 也不例外——发明了一种*内存模型*来掩盖所有支持的处理器的细节，同时，理想情况下，给程序员足够的自由来利用每个处理器的特定功能。系统语言也倾向于以绝对自由的形式允许从语言内存模型中逃脱，这正是 Rust 在*不安全*方面的特点。

在其内存模型方面，Rust 深受 C++的启发。Rust 中暴露的原子排序是 LLVM 的，也就是 C++11 的。这是一件好事——任何与 C++或 LLVM 相关的文献都将立即适用于 Rust。内存顺序是一个复杂的话题，当学习时，依赖为 C++编写的材料通常非常有帮助。这在研究无锁/等待自由结构——我们将在本书后面看到——时尤为重要，因为关于这些主题的文献通常用 C++来讨论。考虑到 C11 编写的文献也适用，尽管可能不太直接。

现在并不是说 C++程序员会立即在并发 Rust 中感到舒适。这些概念会熟悉，但并不完全正确。这是因为 Rust 的内存模型还包括一个关于引用消耗的概念。在 Rust 中，内存中的某个对象必须使用零次或一次，但不能更多。顺便提一下，这是线性类型理论中一种称为*仿射类型*的版本的应用，如果您想了解更多关于这个主题的内容。现在，以这种方式限制内存访问的后果是，Rust 能够在编译时保证安全的内存访问——线程不能在不协调的情况下同时引用内存中的同一位置；同一线程中的无序访问是不允许的，等等。Rust 代码*内存安全*，无需依赖垃圾回收器。根据本书的评估，内存安全是一个很好的胜利，尽管引入的限制确实使得某些在 C++或类似语言中更易于构建的结构实现变得更加复杂。

这是一个我们将在第三章中更详细讨论的主题，*Rust 内存模型——所有权、引用和操作*。

# 准备工作

现在我们已经掌握了机器，这本书将处理我们需要在 Rust 编译器上同步的内容。Rust 有三种频道可供选择——稳定版、测试版和夜间版。Rust 遵循六周发布周期。每天夜间频道都会更新，包含自前一天以来 master 分支上所有新提交的补丁。与测试版和稳定版相比，夜间频道是特殊的，因为它是唯一一个能够编译夜间功能的编译器版本。Rust 对向后兼容性非常认真。语言变更建议在夜间版本中实施，由社区讨论，并在一段时间内使用后，才会从夜间版本状态提升到语言本身。测试版每六周从当前夜间版滚动更新。稳定版也同时从测试版滚动更新，每六周一次。

这意味着编译器的新稳定版本最多只有六周的历史。您选择哪个频道来工作取决于您和组织。我所了解的大多数团队都使用 nightly 版本并发布稳定版本，因为生态系统中的许多重要工具——如 clippy 和 rustfmt——仅通过 nightly 功能可用，但稳定频道提供了稳定的开发目标。您会发现，出于这个原因，许多库在 crate 生态系统中都努力保持稳定。

除非另有说明，否则本书将专注于稳定渠道。我们需要安装两个目标——一个用于我们的 x86 机器，另一个用于我们的 ARMv7。您的操作系统可能已经为您打包了 Rust——恭喜！——但社区倾向于推荐使用`rustup`，尤其是在管理多个、版本固定的目标时。如果您不熟悉`rustup`，可以在[rustup.rs](https://www.rustup.rs/)找到对其使用和说明的令人信服的解释。如果您正在为在运行`rustup`的机器上使用的 Rust 目标进行安装，它将执行必要的三元组检测。假设您的机器中有一个 x86 芯片并且正在运行 Linux，那么以下两个命令将产生相同的结果：

```rs
> rustup install stable

> rustup target add x86_64-unknown-linux-gnu
```

这两个命令都将指示`rustup`跟踪目标`x86_64-unknown-linux-gnu`并安装 Rust 的稳定渠道版本。如果您正在运行 OS X 或 Windows，第一个变体将安装一个稍微不同的三元组，但芯片才是真正重要的。第二个目标，我们需要更精确一些：

```rs
> rustup target add stable-armv7-unknown-linux-gnueabihf
```

现在您可以为您的开发机器、x86（这可能是同一件事）以及运行 Linux 的 ARMv7（如现成的 RaspberryPi 3）生成 x86 二进制文件。如果您打算为 ARMv7 生成可执行文件——如果您有或可以获取芯片，则建议这样做——那么您还需要安装适当的交叉编译器来链接。在基于 Debian 的开发系统上，您可以运行以下命令：

```rs
> apt-get install gcc-arm-linux-gnueabihf
```

指令将因操作系统而异，老实说，这可能是设置交叉编译项目最棘手的部分。不要绝望。如果迫不得已，您总是可以在主机上编译。编译将会很慢。在我们进入有趣的部分之前的最后一步设置是告诉 cargo 如何链接我们的 ARMv7 目标。现在，请注意，在本书写作时，交叉编译是 Rust 社区的一个活跃的研究领域。以下配置文件的调整可能在这本书出版和您阅读它之间有所变化。抱歉。Rust 社区文档肯定会帮助弥补一些差异。无论如何，根据您的操作系统，将以下内容——或类似内容——添加到`~/.cargo/config`：

```rs
[target.armv7-unknown-linux-gnueabihf]
linker = "arm-linux-gnueabihf-gcc"
```

如果不存在`~/.cargo/config`，请使用前面的内容创建它。

# 有趣的部分

让我们创建一个默认的 cargo 项目，并确认我们能否为我们的机器生成适当的汇编代码。这样做将是本书的一个重要支柱。现在，选择磁盘上的一个目录来放置默认项目，并导航到那里。这个例子将使用`~/projects`，但确切路径无关紧要。然后，生成默认项目：

```rs
~projects > cargo new --bin hello_world
     Created binary (application) `hello_world` project
```

好吧，用一些 x86 汇编代码来奖励自己吧：

```rs
hello_world > RUSTFLAGS="--emit asm" cargo build --target=x86_64-unknown-linux-gnu
   Compiling hello_world v0.1.0 (file:///home/blt/projects/hello_world)
       Finished dev [unoptimized + debuginfo] target(s) in 0.33 secs
hello_world > file target/x86_64-unknown-linux-gnu/debug/deps/hello_world-6000abe15b385411.s
target/x86_64-unknown-linux-gnu/debug/deps/hello_world-6000abe15b385411.s: assembler source, ASCII text
```

请注意，如果您在 OS X x86 上编译 Rust 二进制文件，则它可能无法在 Linux x86 上运行，反之亦然。这是由于操作系统自身接口的差异造成的。您最好在 x86 Linux 机器或 x86 OS X 机器上编译，并在那里运行二进制文件。这正是本书中展示的材料所采取的方法。

话虽如此，用一些 ARMv7 汇编代码奖励一下自己吧：

```rs
hello_world > RUSTFLAGS="--emit asm" cargo build --target=armv7-unknown-linux-gnueabihf
   Compiling hello_world v0.1.0 (file:///home/blt/projects/hello_world)
       Finished dev [unoptimized + debuginfo] target(s) in 0.45 secs
hello_world > file target/armv7-unknown-linux-gnueabihf/debug/deps/hello_world-6000abe15b385411.s
target/armv7-unknown-linux-gnueabihf/debug/deps/hello_world-6000abe15b385411.s: assembler source, ASCII text
```

当然，如果您想构建发布版本，只需给 `cargo` 传递 `--release` 标志即可：

```rs
hello_world > RUSTFLAGS="--emit asm" cargo build --target=armv7-unknown-linux-gnueabihf --release
Compiling hello_world v0.1.0 (file:///home/blt/projects/hello_world)
Finished release [optimized] target(s) in 0.45 secs
hello_world > wc -l target/armv7-unknown-linux-gnueabihf/
debug/   release/
hello_world > wc -l target/armv7-unknown-linux-gnueabihf/*/deps/hello_world*.s
 1448 target/armv7-unknown-linux-gnueabihf/debug/deps/hello_world-6000abe15b385411.s
  101 target/armv7-unknown-linux-gnueabihf/release/deps/hello_world-dd65a12bd347f015.s
 1549 total
```

比较生成的代码差异很有趣——并且富有教育意义！值得注意的是，发布编译将完全删除调试信息。说到调试器，让我们来谈谈调试器。

# 调试 Rust 程序

根据您的语言背景，Rust 中的调试情况可能会非常熟悉和舒适，或者可能会让您觉得有点简陋。Rust 依赖于其他编程语言常用的调试工具——`gdb` 或 `lldb`。两者都可以工作，尽管从历史上看，`lldb` 一直存在一些问题，并且直到大约 2016 年中旬，这两个工具才支持未混淆的 Rust。让我们尝试在上一节中的 `hello_world` 上使用 `gdb`：

```rs
hello_world > gdb target/x86_64-unknown-linux-gnu/debug/hello_world
GNU gdb (Debian 7.12-6) 7.12.0.20161007-git
Copyright (C) 2016 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from target/x86_64-unknown-linux-gnu/debug/hello_world...done.
warning: Missing auto-load script at offset 0 in section .debug_gdb_scripts
of file /home/blt/projects/hello_world/target/x86_64-unknown-linux-gnu/debug/hello_world.
Use `info auto-load python-scripts [REGEXP]' to list them.
(gdb) run
Starting program: /home/blt/projects/hello_world/target/x86_64-unknown-linux-gnu/debug/hello_world
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
Hello, world!
[Inferior 1 (process 15973) exited normally]
```

让我们也尝试一下 `lldb`:

```rs
hello_world > lldb --version
lldb version 3.9.1 ( revision )
hello_world > lldb target/x86_64-unknown-linux-gnu/debug/hello_world
(lldb) target create "target/x86_64-unknown-linux-gnu/debug/hello_world"
Current executable set to 'target/x86_64-unknown-linux-gnu/debug/hello_world' (x86_64).
(lldb) process launch
Process 16000 launched: '/home/blt/projects/hello_world/target/x86_64-unknown-linux-gnu/debug/hello_world' (x86_64)
Hello, world!
Process 16000 exited with status = 0 (0x00000000)
```

任何一个调试器都是可行的，并且强烈建议您选择适合您调试风格的那个。这本书将倾向于使用 `lldb`，因为作者有模糊的个人偏好。

在 Rust 开发（以及本书的其他部分）中，您通常会看到的另一个工具集是 `valgrind`。由于 Rust 是内存安全的，您可能会想知道 `valgrind` 何时会被使用。好吧，每当您使用 `unsafe` 时。Rust 中的 `unsafe` 关键字在日常代码中并不常见，但有时会出现在从热点代码路径中挤出额外百分点的场景中。请注意，`unsafe` 块将绝对会出现在这本书中。如果我们对 `hello_world` 运行 `valgrind`，我们会得到预期的无泄漏结果：

```rs
hello_world > valgrind --tool=memcheck target/x86_64-unknown-linux-gnu/debug/hello_world
==16462== Memcheck, a memory error detector
==16462== Copyright (C) 2002-2015, and GNU GPL'd, by Julian Seward et al.
==16462== Using Valgrind-3.12.0.SVN and LibVEX; rerun with -h for copyright info
==16462== Command: target/x86_64-unknown-linux-gnu/debug/hello_world
==16462==
Hello, world!
==16462==
==16462== HEAP SUMMARY:
==16462==     in use at exit: 0 bytes in 0 blocks
==16462==   total heap usage: 7 allocs, 7 frees, 2,032 bytes allocated
==16462==
==16462== All heap blocks were freed -- no leaks are possible
==16462==
==16462== For counts of detected and suppressed errors, rerun with: -v
==16462== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```

在 Rust 程序中分析内存使用是编写性能关键型项目时的一项重要日常任务。为此，我们使用 Massif，堆分析器：

```rs
hello_world > valgrind --tool=massif target/x86_64-unknown-linux-gnu/debug/hello_world
==16471== Massif, a heap profiler
==16471== Copyright (C) 2003-2015, and GNU GPL'd, by Nicholas Nethercote
==16471== Using Valgrind-3.12.0.SVN and LibVEX; rerun with -h for copyright info
==16471== Command: target/x86_64-unknown-linux-gnu/debug/hello_world
==16471==
Hello, world!
==16471==
```

分析缓存也是一个重要的常规任务。为此，我们使用 `cachegrind`，缓存和分支预测分析器：

```rs
hello_world > valgrind --tool=cachegrind target/x86_64-unknown-linux-gnu/debug/hello_world
==16495== Cachegrind, a cache and branch-prediction profiler
==16495== Copyright (C) 2002-2015, and GNU GPL'd, by Nicholas Nethercote et al.
==16495== Using Valgrind-3.12.0.SVN and LibVEX; rerun with -h for copyright info
==16495== Command: target/x86_64-unknown-linux-gnu/debug/hello_world
==16495==
--16495-- warning: L3 cache found, using its data for the LL simulation.
Hello, world!
==16495==
==16495== I   refs:      533,954
==16495== I1  misses:      2,064
==16495== LLi misses:      1,907
==16495== I1  miss rate:    0.39%
==16495== LLi miss rate:    0.36%
==16495==
==16495== D   refs:      190,313  (131,906 rd   + 58,407 wr)
==16495== D1  misses:      4,665  (  3,209 rd   +  1,456 wr)
==16495== LLd misses:      3,480  (  2,104 rd   +  1,376 wr)
==16495== D1  miss rate:     2.5% (    2.4%     +    2.5%  )
==16495== LLd miss rate:     1.8% (    1.6%     +    2.4%  )
==16495==
==16495== LL refs:         6,729  (  5,273 rd   +  1,456 wr)
==16495== LL misses:       5,387  (  4,011 rd   +  1,376 wr)
==16495== LL miss rate:      0.7% (    0.6%     +    2.4%  )
```

这些工具将在整本书中以及许多比 `hello_world` 更有趣的项目中使用。但 `hello_world` 是在这篇文章中实现的第一个交叉编译，这可不是一件小事。

# 摘要

在本章中，我们讨论了本书的预备知识、现代 CPU 的机器架构以及内存的加载和存储可能如何交织出令人惊讶的结果，说明了我们需要某种同步来处理计算的需求。这种同步的需求将推动本书的大部分讨论。我们还讨论了安装 Rust、编译器的通道以及我们专注于*稳定*通道的意图。我们还为 x86 和 ARM 系统编译了一个简单的程序。我们还讨论了我们简单程序的调试和性能分析，这主要是作为工具的证明概念。下一章，第二章，*顺序 Rust 性能和测试*，将更深入地探讨这个领域。

# 进一步阅读

在每一章的结尾，我们将包括一个参考文献列表，这些参考文献强烈推荐给希望进一步深入研究章节中讨论的主题的读者，相关 Rust 社区讨论的链接，或重要工具的文档链接。参考文献可能出现在多个章节中，因为如果某件事很重要，它值得重复。

+   *并行算法导论*，1992 年，约瑟夫·贾贾。这是一本优秀的教科书，介绍了重要的抽象模型。这本书显著关注算法的抽象实现，那时缓存一致性和指令流水线不太重要，所以如果你要找一本，请注意这一点。

+   *程序员应了解的内存知识*，2006 年，乌尔里希·德雷珀。这是一本经典书籍，详细描述了现代计算机中内存的工作原理，尽管这本书是在写作本书时十二年前的作品。

+   *计算机架构：定量方法*，2011 年，约翰·亨尼西和戴维·帕特森。这是一本经典书籍，更多地面向计算机架构师而不是软件工程师。尽管如此，即使你跳过其中的电路图，深入研究这本书也是值得的。

+   *C11 和 C++11 并发模型*，2014 年，马克·巴蒂。巴蒂形式化了 C++11/C11 内存模型，如果你能掌握他的逻辑语言，这是一种学习内存模型及其后果的极好方式。

+   *LLVM 原子指令和并发指南*，可在[`llvm.org/docs/Atomics.html`](https://llvm.org/docs/Atomics.html)找到。[ ](https://llvm.org/docs/Atomics.html)Rust 特别记录了其并发内存模型为 LLVM 模型。本指南及其链接的文档将是阅读本书的任何 Rust 程序员的熟悉领域。

+   *缓存推测侧信道*，2018 年，ARM。分支推测执行导致令人惊讶的信息泄露。ARM 的这篇论文对 ARM 上的推测执行进行了非常清晰的讨论，以及整洁的示例。

+   *std::memory_order* 可在 [`en.cppreference.com/w/cpp/atomic/memory_order`](http://en.cppreference.com/w/cpp/atomic/memory_order) 查找。虽然这份文档涵盖了 C++ 中定义的内存顺序，但其关于 C++ 内存顺序保证后果的示例既直接又具有说明性。

+   *Valgrind 用户手册* 可在 [`valgrind.org/docs/manual/manual.html`](http://valgrind.org/docs/manual/manual.html) 查找。我们将广泛使用 `Valgrind`，对于任何系统程序员来说，熟悉这些工具都是非常值得的。文档不可避免地触及了与本书相同的一些材料，并可能有助于从不同的角度阐明问题。

+   *编译器探索器* 可在 [`rust.godbolt.org/`](https://rust.godbolt.org/) 查找。*编译器探索器* 更多的是一个设计精良的在线工具，而不是一份论文。它允许轻松进行交叉编译，并提供指令的简化解释。
