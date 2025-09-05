# FFI 和嵌入——结合 Rust 和其他语言

在本书的这一部分，我们基本上是独立地讨论了 Rust。Rust 被有意设计成可以通过其**外部函数接口**（**FFI**）调用外部编程语言，并且自身也可以嵌入。许多现代编程语言都提供了 FFI、易于嵌入或两者兼而有之。例如，Python 可以非常方便地调用具有 C 调用约定的库，并且只需稍加考虑就可以嵌入。Lua 是一种类似于 Python 的高级和垃圾回收语言，它有一个方便的 FFI，并且可以轻松嵌入。Erlang 有几个 FFI 接口，但 Erlang 本身并不容易嵌入到用户空间环境中。有趣的是，将 Erlang 编译成 RTOS 映像相当直接。

在本章中，我们将讨论在 Rust 中调用外部代码以及将 Rust 嵌入到外部编程语言中。我们将从扩展我们在第八章的末尾介绍的 corewars evolver 程序——feruscore——开始。您被鼓励在开始本章之前阅读该材料。在我们通过在内部嵌入 C MARS 模拟器来扩展 feruscore 之后，我们将转向从 C 程序中调用 Rust 函数。我们将展示如何将 Lua 嵌入到 Rust 系统中以方便脚本编写，并以将 Rust 嵌入到高级垃圾回收语言（Python 和 Erlang）中结束本章。

到本章结束时，我们将：

+   将上一节中的 feruscore 项目调整为包含 C 模拟器

+   展示了将 Lua 代码包含到 Rust 项目中的方法

+   展示了将 Rust 代码包含到 C 项目中的方法

+   展示了将 Rust 代码包含到 Python 项目中的方法

+   展示了将 Rust 代码包含到 Erlang/Elixir 项目中的方法

# 将 C 嵌入到 Rust 中——feruscore 不使用进程

当我们在上一章结束对 feruscore 的讨论时，我们已经构建了一个程序，可以通过模拟自然选择来发现 corewars 战士。这是通过将进化的战士写入磁盘，使用 Rust 的操作系统进程接口调用 pmars（事实上的标准 MARS）来竞争，以发现它们的相对适应性来完成的。我们使用了 Rayon（Rust 的非常方便的数据并行库）来在可用的 CPU 之间分配竞争的工作负载。不幸的是，实现相当慢。构建锦标赛选择标准可能比我们希望的更难表达——尽管我确信一定有一个聪明的读者会大幅改进这一点并让我感到惊讶。真正的痛点是将每个战士多次序列化到磁盘，重复分配相似的结构来建立每一轮，然后消耗 pmars 的分配和解析开销。正是这一步，调用外部程序来运行竞争，我们将在本章中解决。我们还将解决其他两个问题，因为，嗯，为什么不全力以赴呢？

并非 feruscore 的所有源代码都出现在本章中。其中一些在上一章中进行了深入讨论，一些——例如基准测试代码——将是书中已覆盖材料的重复。C 代码没有全部打印出来，因为它非常密集且非常长。你可以在本书的源代码库中找到完整的列表。

# MARS C 接口

Corewars 是一款古老的比赛，现在有很多 MARS 的实现。pMARS，如我提到的，是事实上的标准，实现了 '94 ICWS 草案，并添加了后来的补充，例如新的指令和称为 *p-space* 的东西，我们在这里不会深入讨论。Corewars 从未是严肃的业务。现在在线上可用的许多 MARS 在过去几年中相互交换代码，编写时对正确性的关注程度不同，有时是为旧机器编写的，在研究实验室中，多处理是一个问题。大多数没有在其内部库接口和可执行消费者之间做出区分，就像 Rust 程序那样。一些幸运的少数被设计为嵌入式，或者至少是旧 MARS 代码库的提取，可以扩展。其中一些幸运的少数在其实现中不使用静态全局变量，并且可以嵌入到多处理环境中，例如 feruscore。

在 MARS 实现的传统大背景下，我们将从一个现有的、公有领域的 MARS 开始，将其精简为一个可管理的核心，并给它带来新的变化。具体来说，本章中引入的 feruscore 嵌入了一个 1.9.2 版本的 exhaust ([`corewar.co.uk/pihlaja/exhaust/`](http://corewar.co.uk/pihlaja/exhaust/))，这是 M. Joonas Pihlaja 在 2000 年代初编写的。Pihlaja 似乎从 pMARS 中提取了部分代码，尤其是在 exhaust 的主函数和解析函数周围。我们并不需要这些代码。我们需要的是模拟器。这意味着我们可以丢弃与解析相关的任何内容，以及为 exhaust 的`main`函数所需的所有支持代码。我们需要的提取代码位于 feruscore 项目的根目录下，在`c_src/`中。我们将嵌入的函数都实现在`c_src/sim.c`中。这些函数来自`c_src/sim.h`：

```rs
void sim_free_bufs(mars_t* mars);
void sim_clear_core(mars_t* mars);

int sim_alloc_bufs(mars_t* mars);
int sim_load_warrior(mars_t* mars, uint32_t pos, 
                     const insn_t* const code, uint16_t len);
int sim_mw(mars_t* mars, const uint16_t * const war_pos_tab,
           uint32_t *death_tab );
```

`sim_alloc_bufs`函数负责分配空白`mars_t`的内部存储，这是模拟所执行的结构体。`sim_clear_core`在回合之间清除`mars_t`，将核心内存设置为`DAT`，并重置任何战士队列。`sim_load_warrior`将战士加载到核心内存中，战士实际上只是一个指向包含指令的数组的指针—`insn_t`—并传递`len`长度。`sim_mw`运行模拟，从`war_pos_tab`读取战士位置，并在战士完全死亡时写入`death_tab`。

# 从 Rust 创建 C-structs

现在，我们如何从 Rust 调用这些函数？此外，我们如何创建`mars_t`或`insn_t`的实例？回想一下第三章，*Rust 内存模型 – 所有权、引用和操作*，Rust 允许对结构体的内存布局进行控制。具体来说，所有 Rust 结构体都是隐式地`repr(Rust)`，类型对齐到字节边界，结构体字段按编译器认为合适的方式进行排序，以及其他细节。我们还提到了结构体的`repr(C)`的存在，其中 Rust 将以与 C 相同的方式布局结构体的内存表示。现在，这些知识将得到应用。

我们将这样做。首先，我们将 C 代码编译成一个库，并将其链接到 feruscore。这是通过在项目根目录放置一个`build.rs`文件并使用 cc ([`crates.io/crates/cc`](https://crates.io/crates/cc)) crate 来生成一个静态归档来完成的：

```rs
extern crate cc;

fn main() {
    cc::Build::new()
        .file("c_src/sim.c")
        .flag("-std=c11")
        .flag("-O3")
        .flag("-Wall")
        .flag("-Werror")
        .flag("-Wunused")
        .flag("-Wpedantic")
        .flag("-Wunreachable-code")
        .compile("mars");
}
```

当项目构建时，Cargo 会将`libmars.a`生成到`target/`目录下。但是，我们如何创建`insn_t`呢？我们复制了表示。本项目的 C 部分定义`insn_t`如下：

```rs
typedef struct insn_st {
  uint16_t a, b;
  uint16_t in;
} insn_t;
```

`uint16_t a`和`uint16_t b`是指令的*a*-字段和*b*-字段，其中`uint16_t`是指令中的`OpCode`、`Modifier`和`Modes`的压缩表示。项目的 Rust 部分定义指令如下：

```rs
#[derive(PartialEq, Eq, Copy, Clone, Debug, Default)]
#[repr(C)]
pub struct Instruction {
    a: u16,
    b: u16,
    ins: u16,
}
```

这是`inst_t` C 的确切布局。读者会注意到这与我们在上一章中看到的`Instruction`定义有很大不同。此外，请注意字段名并不重要，只有位表示。C 结构体在结构体的最后一个字段中调用，但在 Rust 中这是一个保留关键字，所以在 Rust 这边是`ins`。现在，`ins`字段发生了什么？回想一下，`Mode`枚举只有五个字段。我们实际上只需要三个位来编码一个模式，将枚举转换为数值表示。对于指令的其他组件也有类似的想法。`ins`字段的布局如下：

```rs
bit         15 14 13 12 11 10 09 08 07 06 05 04 03 02 01 00
field       |-flags-| |–-opcode–-| |–mod-| |b-mode| |a-mode|
```

a 字段的`Mode`编码在位 0、1 和 2。b 字段的`Mode`在位 3、4 和 5，其他指令组件以此类推。最后两个位，14 和 15，编码了一个几乎总是为零的`flag`。非零标志是向模拟器指示非零指令是`START`指令——战士的第 0 条指令不一定是 MARS 首先执行的指令。这种紧凑的结构需要程序员做更多的工作来支持它。例如，`Instruction`不能再由程序员直接创建，而必须通过构建器构建。`InstructionBuilder`定义在`src/instruction.rs`中，如下所示：

```rs
pub struct InstructionBuilder {
    core_size: u16,
    ins: u16,
    a: u16,
    b: u16,
}
```

和往常一样，我们必须跟踪核心大小。构建构建器这个过程足够直接，到这个阶段为止：

```rs
impl InstructionBuilder {
    pub fn new(core_size: u16) -> Self {
        InstructionBuilder {
            core_size,
            ins: 0_u16,
            a: 0,
            b: 0,
        }
    }
```

将字段写入指令需要一点位操作。以下是如何写入一个`Modifier`：

```rs
    pub fn modifier(mut self, modifier: Modifier) -> Self {
        let modifier_no = modifier as u16;
        self.ins &= !MODIFIER_MASK;
        self.ins |= modifier_no << MODIFIER_MASK.trailing_zeros();
        self
    }
```

常量`MODIFIER_MASK`在源文件顶部的某个代码块中定义，与其他字段掩码一起：

```rs
const AMODE_MASK: u16 = 0b0000_0000_0000_0111;
const BMODE_MASK: u16 = 0b0000_0000_0011_1000;
const MODIFIER_MASK: u16 = 0b0000_0001_1100_0000;
const OP_CODE_MASK: u16 = 0b0011_1110_0000_0000;
const FLAG_MASK: u16 = 0b1100_0000_0000_0000;
```

注意到掩码中的相关位是 1 位。在`InstructionBuilder::modifier`中，我们使用`&=`操作符对掩码取反，然后将`ins`与掩码取反后的结果进行布尔与操作，将之前存在的`Modifier`清零。完成这些后，将编码为 u16 的`Modifier`左移，并布尔或操作到正确的位置。`trailing_zeros()`函数返回一个单词低端的连续零的总数，这是我们每个掩码需要左移的确切数量。那些在其他语言中做过位操作工作的读者可能会觉得这个过程非常简洁。我也这么认为。Rust 的整数显式二进制形式使得编写和后来理解掩码变得非常容易。常见的位操作和查询在每种基本整数类型上都有实现。非常有用。

`OpCode`布局有所变化。我们不使用`repr(C)`枚举，因为位表示并不重要。重要的是，由于这个枚举没有字段，所以它转换到的整数。在源代码中第一个映射到 0，第二个映射到 1，以此类推。C 代码在`c_src/insn.h`中定义操作码如下：

```rs
enum ex_op {
    EX_DAT,             /* must be 0 */
    EX_SPL,
    EX_MOV,
    EX_DJN,
    EX_ADD,
    EX_JMZ,
    EX_SUB,
    EX_SEQ,
    EX_SNE,
    EX_SLT,
    EX_JMN,
    EX_JMP,
    EX_NOP,
    EX_MUL,
    EX_MODM,
    EX_DIV,             /* 16 */
};
```

Rust 版本如下：

```rs
#[derive(PartialEq, Eq, Copy, Clone, Debug, Rand)]
pub enum OpCode {
    Dat,  // 0
    Spl,  // 1
    Mov,  // 2
    Djn,  // 3
    Add,  // 4
    Jmz,  // 5
    Sub,  // 6
    Seq,  // 7
    Sne,  // 8
    Slt,  // 9
    Jmn,  // 10
    Jmp,  // 11
    Nop,  // 12
    Mul,  // 13
    Modm, // 14
    Div,  // 15
}
```

其他指令组件已经稍微调整了一下，以应对 C 代码所需的变更。好消息是，这种表示形式比上一章中的表示形式更紧凑，并且应该保留，即使所有 C 代码都移植到 Rust 中，这也是我们稍后将要讨论的主题。但是——我将省略`InstructionBuilder`的完整定义，因为一旦你看过一组设置函数，你就已经看过了它们——所有这些位操作确实使得实现更难以看到，并且难以一眼纠正。指令模块现在有 QuickCheck 测试来验证所有字段是否正确设置，这意味着无论字段设置和重置多少次，它们都可以立即准备好。鼓励你自己检查 QuickCheck 测试。

高级理念是这样的——创建一个空的`指令`，然后在该指令上运行一系列变更订单——暂时将其移入`指令构建器`以允许修改——然后读取并确认变更的字段已变为所更改的值。这项技术与我们在其他地方之前看到的技术是一致的。

现在，关于那个`mars_t`呢？在`c_src/sim.h`中的 C 定义是：

```rs
typedef struct mars_st {
  uint32_t nWarriors;

  uint32_t cycles;
  uint16_t coresize;
  uint32_t processes;

  uint16_t maxWarriorLength;

  w_t* warTab;
  insn_t* coreMem;
  insn_t** queueMem;
} mars_t;
```

`nWarriors`字段设置模拟中将有多少战士，对于 feruscore 来说，总是两个周期，控制了如果两个战士都还活着，一个回合将需要多少周期才能结束，处理可用的最大进程数，`maxWarriorLength`显示战士可能的最大指令数。所有这些在上一章中或多或少都是熟悉的，只是在新编程语言中，并且有不同的名称。最后三个字段是指向数组的指针，并且对于模拟函数来说是私有的。这些是通过`sim_alloc_bufs`和`sim_free_bufs`分别分配和释放的。这个结构的 Rust 版本看起来是这样的，来自`src/mars.rs`：

```rs
#[repr(C)]
pub struct Mars {
    n_warriors: u32,
    cycles: u32,
    core_size: u16,
    processes: u32,
    max_warrior_length: u16,
    war_tab: *mut WarTable,
    core_mem: *mut Instruction,
    queue_mem: *const *mut Instruction,
}
```

这里唯一的新类型是`WarTable`。尽管我们的代码永远不会显式地操作战士表，但我们仍然必须与 C 兼容。`WarTable`的定义如下：

```rs
#[repr(C)]
struct WarTable {
    tail: *mut *mut Instruction,
    head: *mut *mut Instruction,
    nprocs: u32,
    succ: *mut WarTable,
    pred: *mut WarTable,
    id: u32,
}
```

我们可能可以只通过在`Mars`中将这些私有字段设置为`mars_st`中的`void`指针来解决这个问题，但这会减少项目 C 端的信息类型，并且这种方法可能会阻碍未来的移植工作。由于项目 Rust 端类型明确，因此考虑在 Rust 中重写 C 函数要容易得多。

# 调用 C 函数

现在我们可以创建具有 C 位布局的 Rust 结构，我们可以开始将内存传递到 C 中。重要的是要理解，这是一个固有的不安全活动。C 代码可能会以破坏 Rust 不变性的方式操作内存，而我们唯一能确保这不是真的方法是在事先审计 C 代码。这与模糊测试一样，我们也会进行。我们如何链接到`libmars.a`？一个小的`extern`块就可以做到，在`src/mars.rs`中：

```rs
#[link(name = "mars")]
extern "C" {
    fn sim_alloc_bufs(mars: *mut Mars) -> isize;
    fn sim_free_bufs(mars: *mut Mars) -> ();
    fn sim_clear_core(mars: *mut Mars) -> ();
    fn sim_load_warrior(mars: *mut Mars, pos: u32, 
                        code: *const Instruction, 
                        len: u16) -> isize;
    fn sim_mw(mars: *mut Mars, war_pos_tab: *const u16, 
              death_tab: *mut u32) -> isize;
}
```

使用 `#[link(name = "mars")]`，我们指示编译器链接到构建过程中生成的 `libmars.a`。如果我们正在链接到系统库，方法将是相同的。Rust Nomicon 部分关于 FFI 的内容——在本章末尾的 *进一步阅读* 部分中引用——链接到 libsnappy，例如。extern 块通知编译器，这些函数将需要使用 C ABI 调用，而不是 Rust 的。让我们将 `sim_load_warrior` 进行并排比较。以下是 C 版本：

```rs
int sim_load_warrior(mars_t* mars, uint32_t pos, 
                     const insn_t* const code, uint16_t len);
```

这是 Rust 版本：

```rs
fn sim_load_warrior(mars: *mut Mars, pos: u32, 
                    code: *const Instruction, 
                    len: u16) -> isize;
```

这些是相似的。尽管 Rust 可以免费获取代码的 const 注释，但 C 没有与 Rust 相同的可变性概念。我们通过使所有指令的查询函数都接受 `&self` 来增强了这一点。经验法则——除非 C 是 const-correct，否则你可能需要假设 Rust 需要 `*mut`。这并不总是正确的，但它可以节省很多麻烦。现在，关于这个概念，细心的读者可能会注意到，当我将 exhaust 的代码移植到 feruscore 时，我将其修改为使用 `stdint.h`。在未修改的代码中，`pos` 有无符号整型或 `usize` 类型。模拟器 C 代码假设 `pos` 至少有 32 位宽，因此显式转换为 32 位整数。这并不完全必要，但使用固定宽度类型是一种良好的做法，因为它们最小化了在 CPU 架构之间移动时的惊喜。

# 管理跨语言所有权

Rust 现在知道 C 函数了，并且我们已经将内部结构重新设计为使用 C 布局：现在是时候将操作 `mars_t` 的 C 函数与我们的 `Mars` 链接起来。因为我们正在 Rust 分配器和 C 分配器之间共享 `Mars` 的所有权，所以我们必须小心地将 `Mars` 结构体中的字段初始化为 null 指针，这些字段将由 C 拥有，如下所示：

```rs
impl MarsBuilder {
    pub fn freeze(self) -> Mars {
        let mut mars = Mars {
            n_warriors: 2,
            cycles: u32::from(self.cycles.unwrap_or(10_000)),
            core_size: self.core_size.unwrap_or(8_000),
            processes: self.processes.unwrap_or(10_000),
            max_warrior_length: self.max_warrior_length.unwrap_or(100),
            war_tab: ptr::null_mut(),
            core_mem: ptr::null_mut(),
            queue_mem: ptr::null_mut(),
        };
        unsafe {
            sim_alloc_bufs(&mut mars);
        }
        mars
    }
```

这个 `MarsBuilder` 是一个用于生成 `Mars` 的构建器模式。实现的其他部分按预期工作：

```rs
    pub fn core_size(mut self, core_size: u16) -> Self {
        self.core_size = Some(core_size);
        self
    }

    pub fn cycles(mut self, cycles: u16) -> Self {
        self.cycles = Some(cycles);
        self
    }

    pub fn processes(mut self, processes: u32) -> Self {
        self.processes = Some(processes);
        self
    }

    pub fn max_warrior_length(mut self, max_warrior_length: u16) -> Self {
        self.max_warrior_length = Some(max_warrior_length);
        self
    }
}
```

回到 `MarsBuilder::freeze(self) -> Mars`。这个函数创建一个新的 `Mars`，然后立即将其传递给 `sim_alloc_bufs`：

```rs
unsafe {
    sim_alloc_bufs(&mut mars);
}
```

`&mut mars` 自动转换为 `*mut mars`，正如我们从前几章所知，解引用原始指针是不安全的。它是 *C* 解引用原始指针的事实更是锦上添花。现在，让我们看一下 `sim_alloc_bufs` 并了解正在发生的事情。在 `c_src/sim.c` 中：

```rs
int sim_alloc_bufs(mars_t* mars) {
    mars->coreMem = (insn_t*)malloc(sizeof(insn_t) * mars->coresize);
    mars->queueMem = (insn_t**)malloc(sizeof(insn_t*) * 
                       (mars->nWarriors * mars->processes + 1));
    mars->warTab = (w_t*)malloc(sizeof(w_t)*mars->nWarriors);

    return (mars->coreMem
            && mars->queueMem
            && mars->warTab);
}
```

C 拥有的三个字段——`coreMem`、`queueMem` 和 `warTab`——是根据存储的结构的尺寸分配的，返回值是 `malloc` 返回值的布尔与，这是一种确定 `malloc` 是否曾经返回 `NULL` 的简便方法，这表示系统上没有更多的内存可用。如果我们决定向 Rust 结构体中添加一个新字段，而没有更新 C 结构体以反映这一变化，这些存储空间就会太小。最终，某个地方的代码会超出数组边界并崩溃。这可不是什么好事。

但是！我们刚刚从 Rust 调用了 C 代码。

让我们谈谈所有权问题。`Mars` 是一个结构，它既不完全由 Rust 拥有，也不完全由 C 拥有。这是可以接受的，并且也不少见，尤其是如果你正在部分（或完全）将 C 代码库移植到 Rust。这意味着我们必须小心处理 `Drop`：

```rs
impl Drop for Mars {
    fn drop(&mut self) {
        unsafe { sim_free_bufs(self) }
    }
}
```

正如我们在前面的章节中看到的，当使用原始内存时，我们必须显式地进行 `Drop` 操作。`Mars` 也不例外。这里的 `sim_free_bufs` 调用清理了 C 所拥有的内存，而 Rust 负责处理其余部分。如果清理更加困难——这种情况有时会发生——你必须小心避免在释放后解引用 C 所拥有的指针。`sim_free_bufs` 的实现如下：

```rs
void sim_free_bufs(mars_t* mars)
{
    free(mars->coreMem);
    free(mars->queueMem);
    free(mars->warTab);
}
```

# 运行模拟

剩下的工作就是运行模拟中的战士。在上一章中，`Individual` 负责管理自己的 pmars 进程。现在，`Mars` 将负责托管竞争对手，你可能已经从 C API 中了解到这一点。我们简单地没有在 `Individual` 中存储任何东西的空间，通过将竞争推入 `Mars`，我们可以避免在临时的 `Mars` 结构上发生分配波动。

之前绑定到 `Individual` 的完整函数现在已移动到 `Mars`：

```rs
impl Mars {
    /// Competes two Individuals at random locations
    ///
    /// The return of this function indicates the winner. 
    /// `Winner::Right(12)` will mean that the 'right' player 
    /// won 12 more rounds than did 'left'. This may mean they
    /// tied 40_000 times or maybe they only played 12 rounds 
    /// and right won each time.
    pub fn compete(&mut self, rounds: u16, left: &Individual, 
                   right: &Individual) -> Winner {
        let mut wins = Winner::Tie;
        for _ in 0..rounds {
            let core_size = self.core_size;
            let half_core = (core_size / 2) - self.max_warrior_length;
            let upper_core = core_size - self.max_warrior_length;
            let left_pos = thread_rng().gen_range(0, upper_core);
            let right_pos = if (left_pos + self.max_warrior_length) 
                               < half_core {
                thread_rng().gen_range(half_core + 1, upper_core)
            } else {
                thread_rng().gen_range(0, half_core)
            };
            wins = wins + self.compete_inner(left, left_pos, 
                                             right, right_pos);
        }
        tally_fitness(wins);
        BATTLES.fetch_add(1, Ordering::Relaxed);
        wins
    }
```

C API 要求我们计算战士的偏移量，并注意不要重叠它们。这里采取的方法是随机放置左边的 `Individual`，确定它是在核心的上部还是下部，然后放置右边的 `Individual`，同时注意不要将它们放置在核心的 `end` 之后。竞争的实际实现是 `compete_inner`：

```rs
    pub fn compete_inner(
        &mut self,
        left: &Individual,
        left_pos: u16,
        right: &Individual,
        right_pos: u16,
    ) -> Winner {
        let (left_len, left_code) = left.as_ptr();
        let (right_len, right_code) = right.as_ptr();

        let warrior_position_table: Vec<u16> = vec![left_pos, right_pos];
        let mut deaths: Vec<u32> = vec![u32::max_value(), 
                                        u32::max_value()];
```

我们调用 `Individual::as_ptr() -> (u16, *const Instruction)` 来获取个体的染色体及其长度的原始视图。没有这个，我们就无法将任何内容传递给 C 函数。`warrior_position_table` 通知 MARS 哪些指令是其竞争对手的起始点。我们可以在战士中搜索 `START` 标志，并将其放置在 `warrior_position_table` 中。这是一个留给读者的改进。死亡表将由模拟器代码填充。如果两个战士在竞争中同时死亡，数组将是 `[0, 1]`。死亡表用 `u32::max_value()` 填充，以便于区分无结果和结果。在开始竞争之前，我们必须清除模拟器——它可能充满了来自上次比赛的指令：

```rs
        unsafe {
            sim_clear_core(self);
```

如果你拉取 `sim_clear_core` 的实现，你会找到一个将 `core_mem` 设置为 0 的 memset 操作。回想一下，`DAT` 是 `Instruction` 枚举的第 0 个变体。与 `pmars` 不同，这个模拟器必须使用 `DAT` 作为默认指令，但它确实使得重置字段非常快速。`sim_clear_core` 还清理了进程队列和其他 C 所拥有的存储。加载战士的信息是一个将我们已计算的信息插入的过程：

```rs
            assert_eq!(
                0,
                sim_load_warrior(self, left_pos.into(), 
                                 left_code, left_len)
            );
            assert_eq!(
                0,
                sim_load_warrior(self, right_pos.into(), 
                                 right_code, right_len)
            );
```

如果结果是非零的，这表明发生了某些严重且无法恢复的错误。`sim_load_warrior` 是一个数组操作，将战士指令写入 `core_mem` 中的定义偏移量。如果我们想的话，完全可以使用 Rust 重新编写 `sim_clear_core` 和 `sim_load_warrior` 函数。最后，字段已清除且战士已加载，我们能够进行模拟：

```rs
            let alive = sim_mw(self, 
                               warrior_position_table.as_ptr(),     
                               deaths.as_mut_ptr());
            assert_ne!(-1, alive);
        }
```

`sim_mw` 函数返回模拟运行结束时剩余的战士总数。如果这个值是 `-1`，则表示发生了某些灾难性的、无法恢复的错误。

现在，因为我们有一个很好的类型系统可以与之玩耍，我们并不想用整数向量向用户返回结果。我们保留了看到的 `Winner` 类型

在上一章中，在返回之前进行快速转换：

```rs
        let left_dead = deaths[0] != u32::max_value();
        let right_dead = deaths[1] != u32::max_value();
        match (left_dead, right_dead) {
            (false, false) | (true, true) => Winner::Tie,
            (true, false) => Winner::Right(1),
            (false, true) => Winner::Left(1),
        }
    }
}
```

你可能已经注意到 `Winner` 实现了 `Add`，允许竞争信号战士赢得了多少轮。`impl Add for Winner` 也在 `src/mars.rs` 中，如果你好奇想看看它。作为对 `Mars` 的最终合理性测试，我们确认 Imp 会比其他结果更经常地输给 Dwarf：

```rs
#[cfg(test)]
mod tests {
    use super::*;
    use individual::*;

    #[test]
    fn imp_vs_dwarf() {
        let core_size = 8_000;
        let rounds = 100;
        let mut mars = MarsBuilder::default()
                           .core_size(core_size).freeze();
        let imp = ringers::imp(core_size);
        let dwarf = ringers::dwarf(core_size);
        let res = mars.compete(rounds, &imp, &dwarf);
        println!("RES: {:?}", res);
        match res {
            Winner::Left(_) | Winner::Tie => { 
                panic!("imp should lose to dwarf")
            },
            Winner::Right(_) => {}
        }
    }
}
```

回想一下，Imp 不能自杀，只能传播。Dwarf 以规律、递增的偏移量轰炸核心内存。因此，Imp 与 Dwarf 的竞争是一场 Imp 寻找 Dwarf 指令和 Dwarf 在 Imp 的路径上放下 `DAT` 的赛跑。

# 模糊测试模拟

在本章中，Feruscore 已被修改以包括 Rust 和通过 FFI 的原始内存访问。应负责的事情是模糊测试 `Mars` 并确保我们没有造成段错误。我们将使用 AFL，我们之前在 第二章 和 第五章 中讨论过，分别是 *Sequential Rust Performance and Testing* 和 *Locks – Mutex, Condvar, Barriers, and RWLock*。模糊测试目标是 `src/bin/fuzz_target.rs`。模糊测试的技巧在于确保稳定性。也就是说，如果某些输入被多次应用，导致多个路径出现，那么 AFL 真的无法完成其工作。在确定性系统中，模糊测试更有效率。我们小心地使 `Mars::compete_inner` 确定性，而 `Mars::compete` 使用随机性来确定战士的位置。因此，模糊测试将只通过 `compete_inner`。`fuzz_target` 的前言不包含任何新的 crate：

```rs
extern crate byteorder;
extern crate feruscore;

use byteorder::{BigEndian, ReadBytesExt};
use feruscore::individual::*;
use feruscore::instruction::*;
use feruscore::mars::*;
use std::io::{self, Cursor, Read};
```

记住，AFL 通过 stdin 传递一个字节切片，模糊测试目标是负责将该数组反序列化为对自己有意义的某种形式。我们将构建一个 `Config` 结构体：

```rs
#[derive(Debug)]
struct Config {
    pub rounds: u16,
    pub core_size: u16,
    pub cycles: u16,
    pub processes: u16,
    pub max_warrior_length: u16,
    pub left_chromosome_size: u16,
    pub right_chromosome_size: u16,
    pub left: Individual,
    pub right: Individual,
    pub left_pos: u16,
    pub right_pos: u16,
}
```

希望这些领域对您来说都很熟悉。`rounds`、`core_size`、`cycles` 和 `processes` 每个都会影响 MARS 环境。`max_warrior_length`、`left_chromosome_size` 和 `right_chromosome_size` 影响将要相互竞争的两个个体。`left` 和 `right` 是那些 `Individual` 实例。`left_pos` 和 `right_pos` 设置战士将在 MARS 核心内存中的位置。我们从字节切片反序列化出的数字不一定总是完全合理，因此需要一些清理工作，如下所示：

```rs
impl Config {
    pub fn new(rdr: &mut Cursor<Vec<u8>>) -> io::Result<Config> {
        let rounds = (rdr.read_u16::<BigEndian>()? % 1000).max(1);
        let core_size = (rdr.read_u16::<BigEndian>()? % 24_000).max(256);
        let cycles = (rdr.read_u16::<BigEndian>()? % 10_000).max(100);
        let processes = (rdr.read_u16::<BigEndian>()? % 1024).max(2);
        let max_warrior_length = (rdr.read_u16::<BigEndian>()? % 256).max(4);
        let left_chromosome_size = (rdr.read_u16::<BigEndian>()? 
                                       % max_warrior_length).max(2);
        let right_chromosome_size = (rdr.read_u16::<BigEndian>()? 
                                        % max_warrior_length).max(2);
```

例如，0 轮次没有任何意义，所以我们说至少必须有一轮。同样，我们需要两个进程，至少希望有四个战士指令，等等。创建左右战士的问题在于将字节读取器传递给 `Config::mk_individual`：

```rs
        let left = Config::mk_individual(rdr, 
                                         max_warrior_length,
                                         left_chromosome_size, 
                                         core_size)?;
        let right =
            Config::mk_individual(rdr, 
                                  max_warrior_length, 
                                  right_chromosome_size, 
                                  core_size)?;
```

`Config::mk_individual` 将反序列化到 `InstructionBuilder`。整个事情有点尴尬。虽然我们可以将无字段的枚举转换为整数，但无法在没有一些复杂的匹配语句的情况下，从整数转换到无字段的枚举：

```rs
    fn mk_individual(
        rdr: &mut Cursor<Vec<u8>>,
        max_chromosome_size: u16,
        chromosome_size: u16,
        core_size: u16,
    ) -> io::Result<Individual> {
        assert!(chromosome_size <= max_chromosome_size);
        let mut indv = IndividualBuilder::new();
        for _ in 0..(chromosome_size as usize) {
            let builder = InstructionBuilder::new(core_size);
            let a_field = rdr.read_i8()?;
            let b_field = rdr.read_i8()?;
            let a_mode = match rdr.read_u8()? % 5 {
                0 => Mode::Direct,
                1 => Mode::Immediate,
                2 => Mode::Indirect,
                3 => Mode::Decrement,
                _ => Mode::Increment,
            };
            let b_mode = match rdr.read_u8()? % 5 {
                0 => Mode::Direct,
                1 => Mode::Immediate,
                2 => Mode::Indirect,
                3 => Mode::Decrement,
                _ => Mode::Increment,
            };
```

在这里，我们已经建立了 `InstructionBuilder` 并从字节切片中读取了 a 场和 b 场的 `Mode`。如果添加了字段，我们就必须通过这里来更新模糊测试代码。这真是个麻烦事。以相同的方式读取 `Modifier`：

```rs
            let modifier = match rdr.read_u8()? % 7 {
                0 => Modifier::F,
                1 => Modifier::A,
                2 => Modifier::B,
                3 => Modifier::AB,
                4 => Modifier::BA,
                5 => Modifier::X,
                _ => Modifier::I,
            };
```

读取 `OpCode` 也是如此：

```rs
            let opcode = match rdr.read_u8()? % 16 {
                0 => OpCode::Dat,   // 0
                1 => OpCode::Spl,   // 1
                2 => OpCode::Mov,   // 2
                3 => OpCode::Djn,   // 3
                4 => OpCode::Add,   // 4
                5 => OpCode::Jmz,   // 5
                6 => OpCode::Sub,   // 6
                7 => OpCode::Seq,   // 7
                8 => OpCode::Sne,   // 8
                9 => OpCode::Slt,   // 9
                10 => OpCode::Jmn,  // 10
                11 => OpCode::Jmp,  // 11
                12 => OpCode::Nop,  // 12
                13 => OpCode::Mul,  // 13
                14 => OpCode::Modm, // 14
                _ => OpCode::Div,   // 15
            };
```

生成指令很简单，多亏了这里使用的构建器模式：

```rs
            let inst = builder
                .a_field(a_field)
                .b_field(b_field)
                .a_mode(a_mode)
                .b_mode(b_mode)
                .modifier(modifier)
                .opcode(opcode)
                .freeze();
            indv = indv.push(inst);
        }
        Ok(indv.freeze())
    }
```

回到 `Config::new`，我们创建左右位置：

```rs
        let left_pos =
            Config::adjust_pos(core_size, 
                               rdr.read_u16::<BigEndian>()?, 
                               max_warrior_length);
        let right_pos =
            Config::adjust_pos(core_size, 
                               rdr.read_u16::<BigEndian>()?, 
                               max_warrior_length);
```

`adjust_pos` 函数是一件小事，目的是确保战士的位置在合理的范围内：

```rs
    fn adjust_pos(core_size: u16, mut pos: u16, space: u16) -> u16 {
        pos %= core_size;
        if (pos + space) > core_size {
            let past = (pos + space) - core_size;
            pos - past
        } else {
            pos
        }
    }
```

战士与这个计算重叠是完全可能的。这是可以的。我们进行模糊测试的雄心壮志不是检查程序的逻辑，而是寻找崩溃。实际上，如果两个战士重叠导致崩溃，这是我们必须要知道的事实。`Config::new` 的结束相当直接：

```rs
        Ok(Config {
            rounds,
            core_size,
            cycles,
            processes,
            max_warrior_length,
            left_chromosome_size,
            right_chromosome_size,
            left,
            right,
            left_pos,
            right_pos,
        })
    }
```

在所有这些之后，`fuzz_target` 的主函数是最小的：

```rs
fn main() {
    let mut input: Vec<u8> = Vec::with_capacity(1024);
    let result = io::stdin().read_to_end(&mut input);
    if result.is_err() {
        return;
    }
    let mut rdr = Cursor::new(input);
    if let Ok(config) = Config::new(&mut rdr) {
        let mut mars = MarsBuilder::default()
            .core_size(config.core_size)
            .cycles(config.cycles)
            .processes(u32::from(config.processes))
            .max_warrior_length(config.max_warrior_length as u16)
            .freeze();
        mars.compete_inner(
            &config.left,
            config.left_pos,
            &config.right,
            config.right_pos,
        );
    }
}
```

`Stdin` 被捕获并从它构建了一个 `Cursor`，我们将其传递给 `Config::new`，如前所述。生成的 `Config` 用于驱动 `MarsBuilder`，然后 `Mars` 就是两个可能重叠或可能不重叠的模糊测试 `Individual` 实例竞争的舞台。记住，在运行 AFL 之前，一定要运行 `cargo afl build`——发布版，而不是 cargo build ——发布版。两者都可以工作，但第一个在发现崩溃方面要快得多，因为 AFL 的仪器将内联在可执行文件中。我发现，即使是一个单独的实例 `cargo afl fuzz -i /tmp/in -o /tmp/out target/release/fuzz_target` 也能以不错的速度完成 AFL 循环。代码中分支不多，因此 AFL 要探索的路径也少。

# `feruscore` 可执行文件

最后要介绍的是`src/bin/feruscore.rs`。仔细的读者会注意到，到目前为止，实现中几乎没有使用 rayon。实际上，在这个版本中并没有使用 rayon。以下是项目的完整`Cargo.toml`：

```rs
[package]
name = "feruscore"
version = "0.2.0"
authors = ["Brian L. Troutwine <brian@troutwine.us>"]

[dependencies]
rand = "0.4"
rand_derive = "0.3"
libc = "0.2.0"
byteorder = "1.0"
num_cpus = "1.0"

[build-dependencies]
cc = "1.0"

[dev-dependencies]
quickcheck = "0.6"
criterion = "0.2"

[[bench]]
name = "mars_bench"
harness = false

[[bin]]
name = "feruscore"

[[bin]]
name = "fuzz_target"
```

如本章开头所述，feruscore 除了调用操作系统进程外，还存在两个问题：相似结构的重复分配和对锦标赛的较差控制。从某种角度看，比较`Individual`的适应性是一个排序函数。Rayon 确实具有并行排序能力，`ParallelSliceMut<T: Send>::par_sort(&mut self)`，其中`T: Ord`。我们可以利用这一点，为`Individual`定义一个`Ord`，每次比较都会分配一个新的`Mars`。然而，许多微小的分配会严重影响速度。一个线程本地的`Mars`可以将这个数量减少到每个线程一个分配，但这样我们仍然放弃了一些控制。例如，如果不检查 rayon 的源代码，我们能否确定种群块的大小大致相等？通常情况下，这并不是一个问题，但对我们来说却是。Rayon 要求我们执行折叠和减少步骤，这也是我们不需要做的工作，如果我们调整我们的目标的话。

处理并行遗传算法的一种常见方法，也是我们现在要采用的方法，是创建岛屿，这些岛屿并行进行进化。用户设置一个全局种群，这个种群被分配到岛屿之间，线程被分配到岛屿以模拟一定数量的代。在达到这一代数的限制后，岛屿种群被合并、打乱并重新分配到岛屿。这有助于减少跨线程通信，这可能会带来缓存局部性问题。

`src/bin/feruscore.rs`的序言部分很简单：

```rs
extern crate feruscore;
extern crate num_cpus;
extern crate rand;

use feruscore::individual::*;
use feruscore::mars::*;
use rand::{thread_rng, Rng};
use std::fs::{self, DirBuilder, File};
use std::io::{self, BufWriter};
use std::path::{Path, PathBuf};
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::mpsc;
use std::{thread, time};
```

与上一章相比，配置和报告的全局变量有所减少：

```rs
// configuration
const POPULATION_SIZE: usize = 1_048_576 / 8; 
const CHROMOSOME_SIZE: u16 = 100;
const CORE_SIZE: u16 = 8000;
const GENE_MUTATION_CHANCE: u32 = 100;
const ROUNDS: u16 = 100;

// reporting
static GENERATIONS: AtomicUsize = AtomicUsize::new(0);
```

`report`函数几乎与上一章相同，只是要读取的原子变量少了一些。检查点也几乎完全相同。为了节省空间，我们将跳过重新打印这两个函数。新的内容是`sort_by_tournament`：

```rs
fn sort_by_tournament(mars: &mut Mars, 
                      mut population: Vec<Individual>) 
    -> Vec<Individual> 
{
    let mut i = population.len() - 1;
    while i >= (population.len() / 2) {
        let left_idx = thread_rng().gen_range(0, i);
        let mut right_idx = left_idx;
        while left_idx == right_idx {
            right_idx = thread_rng().gen_range(0, i);
        }
        match mars.compete(ROUNDS, 
                           &population[left_idx], 
                           &population[right_idx]) 
        {
            Winner::Right(_) => {
                population.swap(i, right_idx);
            }
            Winner::Left(_) => {
                population.swap(i, left_idx);
            }
            Winner::Tie => {}
        }
        i -= 1;
    }
    population
}
```

这个函数根据传递的`Mars`内部竞争的结果对种群进行排序。读者会注意到，这并不是一个真正的排序，因为最后一个元素是最适应的，但最后一个元素是第一个锦标赛的获胜者，倒数第二个是第二个锦标赛的获胜者，以此类推。只有`population.len() / 2`个竞争进行，产生的冠军与精确按适应性排序的种群相比具有优势。鼓励读者尝试自己实现`sort_by_tournament`。现在，让我们看看`main`函数：

```rs
fn main() {
    let mut out_recvs = Vec::with_capacity(num_cpus::get());
    let mut in_sends = Vec::with_capacity(num_cpus::get());

    let total_children = 128;

    let thr_portion = POPULATION_SIZE / num_cpus::get();
    for _ in 0..num_cpus::get() {
        let (in_snd, in_rcv) = mpsc::sync_channel(1);
        let (out_snd, out_rcv) = mpsc::sync_channel(1);
        let mut population: Vec<Individual> = (0..thr_portion)
            .into_iter()
            .map(|_| Individual::new(CHROMOSOME_SIZE))
            .collect();
        population.pop();
        population.pop();
        population.push(ringers::imp(CORE_SIZE));
        population.push(ringers::dwarf(CORE_SIZE));
        let _ = thread::spawn(move || island(in_rcv, 
                                             out_snd, 
                                             total_children));
        in_snd.send(population).unwrap();
        in_sends.push(in_snd);
        out_recvs.push(out_rcv);
    }
```

在 feruscore 运行中岛屿的总数将根据系统上可用的 CPU 数量而变化。每个岛屿都会创建两个同步的 MPSC 通道，一个用于主线程将种群推入工作线程，另一个用于工作线程将种群推回主线程。实现中称这些为`in_*`和`out_*`发送者和接收者。你可以再次看到，我们正在构建一个由随机`Individual`战士组成的种群并将他们推入其中，尽管岛屿种群不是`POPULATION_SIZE`，而是由可用 CPU 数量均匀分割的`POPULATION_SIZE`。在岛屿线程拥有种群后，启动报告线程，主要是为了避免 UI 垃圾信息：

```rs
    let _ = thread::spawn(report);
```

报告函数与上一章中关于 feruscore 的讨论大致相同，为了简洁起见，这里不再列出。

`main`函数的最后一段是重组循环。当岛屿线程完成竞赛后，它们将写入它们的输出发送者，该发送者会被重组循环拾取：

```rs
    let mut mars = MarsBuilder::default().core_size(CORE_SIZE).freeze();
    let mut global_population: Vec<Individual> = 
        Vec::with_capacity(POPULATION_SIZE);
    loop {
        for out_rcv in &mut out_recvs {
            let mut pop = out_rcv.recv().unwrap();
            global_population.append(&mut pop);
        }
```

一旦所有岛屿合并在一起，整个种群就会被打乱：

```rs
        assert_eq!(global_population.len(), POPULATION_SIZE);
        thread_rng().shuffle(&mut global_population);
```

这样，一代就完成了。我们`checkpoint`，这次进行了一次额外的锦标赛，从全局种群中拉取保存的优胜者：

```rs
        let generation = GENERATIONS.fetch_add(1, Ordering::Relaxed);
        if generation % 100 == 0 {
            global_population = sort_by_tournament(&mut mars, 
                                                   global_population);
            checkpoint(generation, &global_population)
                .expect("could not checkpoint");
        }
```

最后，种群被重新分割并发送到岛屿线程：

```rs
        let split_idx = global_population.len() / num_cpus::get();

        for in_snd in &mut in_sends {
            let idx = global_population.len() - split_idx;
            let pop = global_population.split_off(idx);
            in_snd.send(pop).unwrap();
        }
    }
}
```

现在，岛屿在做什么？嗯，它是一个无限循环，从种群分配接收器中提取：

```rs
fn island(
    recv: mpsc::Receiver<Vec<Individual>>,
    snd: mpsc::SyncSender<Vec<Individual>>,
    total_children: usize,
) -> () {
    let mut mars = MarsBuilder::default().core_size(CORE_SIZE).freeze();

    while let Ok(mut population) = recv.recv() {
```

现在在循环上方分配了一个`Mars`，这意味着我们将在每个 feruscore 运行中只进行`num_cpu::get()`次这种结构的分配。也请记住，当通道中没有数据时，`Receiver::recv`会阻塞，因此岛屿线程在没有工作可做时不会消耗 CPU。循环的内部应该很熟悉。首先，将`Individual`战士放入竞赛：

```rs
        // tournament, fitness and selection of parents
        population = sort_by_tournament(&mut mars, population);
```

通过从种群中选取高排名成员并产生两个子代到种群的低端，直到达到每代所需子代总数，来完成繁殖：

```rs
        // reproduce
        let mut child_idx = 0;
        let mut parent_idx = population.len() - 1;
        while child_idx < total_children {
            let left_idx = parent_idx;
            let right_idx = parent_idx - 1;
            parent_idx -= 2;

            population[child_idx] = population[left_idx]
                                    .reproduce(&population[right_idx]);
            child_idx += 1;
            population[child_idx] = population[left_idx]
                                    .reproduce(&population[right_idx]);
            child_idx += 1;
        }
```

在这个实现中，我们还在新引入的子代之前引入了随机新的种群成员：

```rs
        for i in 0..32 {
            population[child_idx + i] = Individual::new(CHROMOSOME_SIZE)
        }
```

随机注入可以帮助抑制种群中的过早收敛。记住，模拟进化是一种状态空间搜索。还有一点变化是，种群中的所有成员都有突变的机会：

```rs
        // mutation
        for indv in population.iter_mut() {
            indv.mutate(GENE_MUTATION_CHANCE);
        }
```

最后，我们将种群推回重组线程，对种群进行了彻底的处理：

```rs
        snd.send(population).expect("could not send");
    }
}
```

在这里进行的所有更改——减少小分配、每代减少竞赛总数、移除 pmars 解析及 spawn 开销——使得实现速度大幅提升。回想一下，上一章中的实现难以达到每秒 500 场竞赛——或者如 UI 所示为`BATTLES`——的峰值。以下是我在最近一次八小时运行中得到的报告诊断：

```rs
GENERATION(5459):
    RUNTIME (sec):  31248
    BATTLES:        41181
       BATTLES/s:   14340
    FITNESS:
        00...10:    151
        11...20:    2
        21...30:    0
        31...40:    1
        41...50:    0
        51...60:    0
        61...60:    2
        71...70:    1
        81...80:    0
        91...100:   41024
```

这是在一个 8,000 大小核心内存上的每秒 14,000 次战斗，或者大约每小时 650 代。仍然不是特别快，但足够你在一段时间后开始产生相当平庸的玩家。减少核心内存大小将提高运行时间，限制战士的最大长度也是如此。对构建更好的进化器感兴趣的人应该调查不同的评估适应度的方式，引入更多的铃声，并看看将 `sim_mw` 移植到 Rust 中是否会提高运行时间。这个模拟器不支持 pMARS 所有的指令范围，所以这也是一个改进的领域。

我很乐意听听你的想法。

# 将 Lua 集成到 Rust 中

本书讨论的许多程序都使用一个 *报告* 线程来通知用户程序的运行行为。这些报告函数都使用 Rust 编码，是可执行文件中不变的部分。但是，如果我们想让最终用户能够提供他们自己的报告例程呢？或者，考虑一下本书之前讨论过的 cernan 项目 ([`crates.io/crates/cernan`](https://crates.io/crates/cernan))，它支持一个 *可编程过滤器*，这是一个在线数据流过滤器，最终用户可以在不更改 cernan 二进制文件的情况下对其进行编程。你是如何完成这个技巧的？

一个常见的答案，不仅在 Rust 中，在许多编译语言中也是如此，就是嵌入一个 Lua 解释器 ([`www.lua.org/`](https://www.lua.org/)) 并在启动时读取用户程序。事实上，这是一个如此常见的答案，以至于在 crates 生态系统中有许多 Lua 集成可供选择。在这里我们将使用 rlua ([`crates.io/crates/rlua`](https://crates.io/crates/rlua))，因为它是一个安全的选择，并且项目文档非常好。其他 Lua 集成适合不同的目标。例如，Cernan 使用不同的、不一定安全的嵌入，因为我们需要允许最终用户定义他们自己的函数。

在上一章中，我们编写了一个名为 `sniffer` 的项目，其目的是三方面的——在接口上收集以太网数据包，报告它们，并将以太网数据包回显。让我们将这个程序改编一下，让用户能够通过自定义脚本来决定如何报告。项目的 `Cargo.toml` 文件略有不同，包括包含 rlua 依赖项并删除了线程密集型替代可执行文件：

```rs
[package]
name = "sniffer"
version = "0.2.0"
authors = ["Brian L. Troutwine <brian@troutwine.us>"]

[dependencies]
pnet = "0.21"
rlua = "0.13"

[[bin]]
name = "sniffer"
```

在 `src/bin/sniffer.rs` 中定义的 `sniffer` 程序的前置部分对我们来说并没有惊喜：

```rs
extern crate pnet;
extern crate rlua;

use pnet::datalink::Channel::Ethernet;
use pnet::datalink::{self, DataLinkReceiver, MacAddr, 
                     NetworkInterface};
use pnet::packet::ethernet::{EtherType, EthernetPacket};
use rlua::{prelude, Function, Lua};
use std::fs::File;
use std::io::{prelude::*, BufReader};
use std::path::Path;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::mpsc;
use std::{env, thread};
```

我们从 rlua 库中导入 `Function` 和 `Lua`，但这基本上就是所有新内容。`SKIPPED_PACKETS` 和 `Payload` 的细节没有改变：

```rs
static SKIPPED_PACKETS: AtomicUsize = AtomicUsize::new(0);

#[derive(Debug)]
enum Payload {
    Packet {
        source: MacAddr,
        destination: MacAddr,
        kind: EtherType,
    },
    Pulse(u64),
}
```

我已经在这个例子中移除了以太网数据包的回显，因为在繁忙的以太网网络上这样做并不一定是一件好事。因此，`watch_interface` 比以前更加简洁：

```rs
fn watch_interface(mut rx: Box<DataLinkReceiver>, snd: mpsc::SyncSender<Payload>) {
    loop {
        match rx.next() {
            Ok(packet) => {
                let packet = EthernetPacket::new(packet).unwrap();

                let payload: Payload = Payload::Packet {
                    source: packet.get_source(),
                    destination: packet.get_destination(),
                    kind: packet.get_ethertype(),
                };
                if snd.try_send(payload).is_err() {
                    SKIPPED_PACKETS.fetch_add(1, Ordering::Relaxed);
                }
            }
            Err(e) => {
                panic!("An error occurred while reading: {}", e);
            }
        }
    }
}
```

`timer` 函数也没有改变：

```rs
fn timer(snd: mpsc::SyncSender<Payload>) -> () {
    use std::{thread, time};
    let one_second = time::Duration::from_millis(1000);

    let mut pulses = 0;
    loop {
        thread::sleep(one_second);
        snd.send(Payload::Pulse(pulses)).unwrap();
        pulses += 1;
    }
}
```

`main`函数现在接收一个额外的参数，目的是要传递给 Lua 函数的磁盘路径：

```rs
fn main() {
    let interface_name = env::args().nth(1).unwrap();
    let pulse_fn_file_arg = env::args().nth(2).unwrap();
    let pulse_fn_file = Path::new(&pulse_fn_file_arg);
    assert!(pulse_fn_file.exists());
    let interface_names_match = |iface: &NetworkInterface| {
        iface.name == interface_name
    };

    // Find the network interface with the provided name
    let interfaces = datalink::interfaces();
    let interface = interfaces
        .into_iter()
        .filter(interface_names_match)
        .next()
        .unwrap();
```

该函数旨在处理从计时器线程传入的`Payload::Pulse`。上一章中的收集函数不再存在。为了实际使用用户提供的函数，我们必须通过`rlua::Lua::new()`创建一个新的 Lua 虚拟机，然后将函数加载到其中，如下所示：

```rs
    let lua = Lua::new();
    let pulse_fn = eval(&lua, pulse_fn_file, Some("pulse")).unwrap();
```

`eval`函数简短，主要与从磁盘读取有关：

```rs
fn eval<'a>(lua: &'a Lua, path: &Path, name: Option<&str>) 
    -> prelude::LuaResult<Function<'a>> 
{
    let f = File::open(path).unwrap();
    let mut reader = BufReader::new(f);

    let mut buf = Vec::new();
    reader.read_to_end(&mut buf).unwrap();
    let s = ::std::str::from_utf8(&buf).unwrap();

    lua.eval(s, name)
}
```

那里的特定于 Lua 的部分是`lua.eval(s, name)`。它评估从磁盘读取的 Lua 代码，并返回一个函数，这是一个 Rust 可调用的 Lua 片段。Lua 源中提供的任何用户名称都被忽略，名称的存在是为了在 Rust 端添加错误消息的上下文。rlua 在其 Rust API 中不公开`loadfile`，尽管其他 Lua 嵌入确实如此。

`main`函数的下一部分基本上没有变化，尽管包缓冲通道已从 10 个元素增加到`100`：

```rs
    let (snd, rcv) = mpsc::sync_channel(100);

    let timer_snd = snd.clone();
    let _ = thread::spawn(move || timer(timer_snd));

    let _iface_handler = match datalink::channel(&interface, 
                                                 Default::default()) 
    {
        Ok(Ethernet(_tx, rx)) => {
            let snd = snd.clone();
            thread::spawn(|| watch_interface(rx, snd))
        }
        Ok(_) => panic!("Unhandled channel type"),
        Err(e) => panic!(
            "An error occurred when creating the datalink channel: {}",
            e
        ),
    };
```

现在，事情将要发生变化。首先，我们创建三个 Lua 表格来存储`Payload::Packet`的组件：

```rs
    let destinations = lua.create_table().unwrap();
    let sources = lua.create_table().unwrap();
    let kinds = lua.create_table().unwrap();
```

在上一章中，我们使用了三个`HashMap`，但现在，我们需要能够轻松传递给 Lua 的东西。主线程负责收集`Payload`实例的角色，这是收集函数曾经扮演的角色：收集`Payload`实例。这节省了一个线程，意味着我们不需要仔细安排发送 Lua 虚拟机跨线程边界：

```rs
    while let Ok(payload) = rcv.recv() {
        match payload {
            Payload::Packet {
                source,
                destination,
                kind,
            } => {
                let d_cnt = destinations
                                .get(destination.to_string())
                                .unwrap_or(0);
                destinations
                    .set(destination.to_string(), d_cnt + 1)
                    .unwrap();

                let s_cnt = sources.get(source.to_string()).unwrap_or(0);
                sources.set(source.to_string(), s_cnt + 1).unwrap();

                let k_cnt = kinds.get(kind.to_string()).unwrap_or(0);
                kinds.set(kind.to_string(), k_cnt + 1).unwrap();
            }
```

在这里，我们已经开始从接收器中提取有效载荷，并处理跨过来的`Payload::Packets`。注意我们在这个循环之前创建的表格现在正在被填充。同样的基本聚合操作正在发挥作用；持续计算进入的`Packet`组件。一个更有冒险精神的读者可能会扩展这个程序，允许 Lua 端构建自己的聚合。现在剩下的只是处理`Payload::Pulse`：

```rs
            Payload::Pulse(id) => {
                let skipped_packets = SKIPPED_PACKETS
                                      .swap(0, Ordering::Relaxed);
                pulse_fn
                    .call::<_, ()>((
                        id,
                        skipped_packets,
                        destinations.clone(),
                        sources.clone(),
                        kinds.clone(),
                    ))
                    .unwrap()
            }
        }
    }
}
```

之前创建的`pulse_fn`函数使用四个参数调用——脉冲 ID、`SKIPPED_PACKETS`和聚合表格。我们预计`pulse_fn`不会有任何返回值，因此有`<_, ()>`部分。这个版本的嗅探器包括一个示例包函数，定义在`examples/pulse.lua`中：

```rs
function (id, skipped_packets, dest_tbl, src_tbl, kind_tbl)
   print("ID: ", id)
   print("SKIPPED PACKETS: ", skipped_packets)
   print("DESTINATIONS:")
   for k,v in pairs(dest_tbl) do
      print(k, v)
   end
   print("SOURCES:")
   for k,v in pairs(src_tbl) do
      print(k, v)
   end
   print("KINDS:")
   for k,v in pairs(kind_tbl) do
      print(k, v)
   end
end
```

读者会注意到它基本上与之前收集函数所做的工作相同。运行嗅探器的工作方式与之前相同。确保执行`cargo build --release`然后：

```rs
> ./target/release/sniffer en0 examples/pulse.lua
ID:     0
SKIPPED PACKETS:    0
DESTINATIONS:
5c:f9:38:8b:4a:b6   11
8c:59:73:1d:c8:73   16
SOURCES:
5c:f9:38:8b:4a:b6   16
8c:59:73:1d:c8:73   11
KINDS:
Ipv4    27
ID:     1
SKIPPED PACKETS:    0
DESTINATIONS:
5c:f9:38:8b:4a:b6   14
8c:59:73:1d:c8:73   20
SOURCES:
5c:f9:38:8b:4a:b6   20
8c:59:73:1d:c8:73   14
KINDS:
Ipv4    34
```

注意分配的读者会注意到这里有很多克隆操作。这是真的。主要的编译是 Lua 的 GC 的互操作性。安全的 Rust 接口必须假设传递给 Lua 的数据将被垃圾回收。但是，Lua 支持用户数据的概念，程序员将不透明的数据块与 Lua 函数关联以操作它。Lua 还支持轻量级用户数据，它与用户数据非常相似，只是轻量级变体与内存的指针相关联。RLua 中的 UserData 类型做得相当不错，有雄心的读者可以构建一个实现`UserData`的`PacketAggregation`类型，以避免所有克隆操作。

将高级语言与系统语言结合通常是一场在内存管理复杂性、最终用户负担和初始编程难度之间的权衡游戏。rlua 在这些权衡中做得非常出色。像 mond 这样的东西在 cernan 中使用，在这方面做得不那么好，但使用上更加灵活。

# 嵌入 Rust

到目前为止，我们已经看到了如何将 C 和 Lua 嵌入 Rust。但是，如果你想要将 Rust 与其他编程语言结合使用呢？这样做是一种非常实用的技术，可以提高解释型语言的运行时性能，实现内存安全的扩展，这在以前你可能需要使用 C 或 C++。如果你的目标高级语言在并发嵌入方面有困难，Rust 则是一个更大的优势。Python 程序在这方面表现不佳——至少是那些在 CPython 或 PyPy 上实现的——因为全局解释器锁（一个内部互斥锁，锁定对象的字节码）。例如，将大量数据的计算任务卸载到 Rust + Rayon 扩展中，既可以简化编程，又可以提高计算速度。

好吧，太棒了。我们如何实现这类事情呢？Rust 的方法很简单：如果你可以嵌入 C，你就可以嵌入 Rust。

# 进入 C

让我们在 C 中嵌入一些 Rust。quantiles 库（[`crates.io/crates/quantiles`](https://crates.io/crates/quantiles)）——在第四章中讨论过，*Sync 和 Send——Rust 并发的基石*——实现了在线汇总算法。这些算法在产生错误结果方面很容易出错，但更常见的是存储比严格必要更多的点。在 quantiles 中投入了大量工作以确保实现的算法接近理论上的最小存储需求，因此对于 C 中的在线汇总重用这个库而不是重新做所有这些工作是有意义的。

具体来说，让我们将`quantiles::ckms::CKMS<f32>`暴露给 C ([`docs.rs/quantiles/0.7.1/quantiles/ckms/struct.CKMS.html`](https://docs.rs/quantiles/0.7.1/quantiles/ckms/struct.CKMS.html))。由于 C 在类型中缺乏任何泛型方式，我们必须使类型具体化，但这没关系。

# Rust 方面

在`Cargo.toml`文件中有一些新内容需要讨论：

```rs
[package]
name = "embed_quantiles"
version = "0.1.0"
authors = ["Brian L. Troutwine <brian@troutwine.us>"]

[dependencies]
quantiles = "0.7"

[lib]
name = "embed_quantiles"
crate-type = ["staticlib"]
```

特别注意，我们正在构建`embed_quantiles`库，使用`crate-type = ["staticlib"]`。这意味着什么？Rust 编译器能够进行许多种链接，通常它们都是隐式的。例如，二进制文件是`crate-type = ["bin"]`。有一个长长的不同链接类型的列表，我已经在*进一步阅读*部分中包含了它们。鼓励感兴趣的读者查阅。我们希望在这里生成一个静态链接库，否则每个试图使用`embed_quantiles`的 C 程序员都需要在他们的系统上安装 Rust 的共享库。这……并不理想。一个`staticlib`crate 将存档 Rust 代码需要的 Rust 标准库的所有部分。然后，这个存档可以像平常一样链接到 C。

好的，现在，如果我们打算生成一个 C 可以调用的静态存档，我们必须导出 Rust 函数以 C ABI。换句话说，我们必须在 Rust 之上套上一层 C 的皮肤。C++程序员会熟悉这种策略。这个项目中的唯一一个 Rust 文件是`src/lib.rs`，其前导部分可能正如你所预期的那样：

```rs
extern crate quantiles;

use quantiles::ckms::CKMS;
```

我们已经引入了分位数库并导入了`CKMS`。没什么大不了的。不过，看看这个：

```rs
#[no_mangle]
pub extern "C" fn alloc_ckms(error: f64) -> *mut CKMS<f32> {
    let ckms = Box::new(CKMS::new(error));
    Box::into_raw(ckms)
}
```

嘿！有一大堆新东西。首先，`#[no_mangle]`？静态库必须导出一个符号供链接器使用，以便进行链接。这些符号是函数、静态变量等等。通常，Rust 可以自由地篡改符号的名称以包含信息，比如模块位置，或者实际上，编译器想要做的任何事情。至于名称篡改的确切语义，截至本文写作时是未定义的。如果我们打算从 C 调用一个函数，我们必须有确切的符号名称来引用。`no_mangle`关闭了名称篡改，保留了我们书写的名称。这意味着我们必须小心不要造成符号冲突。类似于导入函数，这里的`extern C`意味着这个函数应该按照 C ABI 编写。技术上，我们也可以写成`extern fn`，省略 C，因为 C ABI 是隐式的默认值。

`alloc_ckms`分配一个新的`CKMS`，返回对其的可变指针。与 C 的互操作性需要原始指针，这是有道理的。我们在嵌入 Rust 时必须非常注意内存所有权——Rust 拥有内存，这意味着我们需要提供一个释放函数？或者，其他语言拥有内存？大多数情况下，保持 Rust 的所有权更容易，因为为了释放内存，编译器需要知道类型的内存大小。通过传递指针出来，就像我们在这里做的那样，我们让 C 对`CKMS`的大小一无所知。这个项目的 C 部分只知道它有一个*不透明结构体*要处理。这是 C 库中常见的策略，有很好的理由。以下是释放`CKMS`的示例：

```rs
#[no_mangle]
pub extern "C" fn free_ckms(ckms: *mut CKMS<f32>) -> () {
    unsafe {
        let ckms = Box::from_raw(ckms);
        drop(ckms);
    }
}
```

注意，在`alloc_ckms`中，我们正在装箱`CKMS`——强制将其放入堆中——而在`free_ckms`中，我们正在从其指针构建一个装箱的`CKMS`。我们在第三章“Rust 内存模型——所有权、引用和操作”的上下文中广泛讨论了装箱和释放内存。将值插入`CKMS`是足够直接的：

```rs
#[no_mangle]
pub extern "C" fn ckms_insert(ckms: &mut CKMS<f32>, value: f32) -> () {
    ckms.insert(value)
}
```

查询需要一点解释：

```rs
#[no_mangle]
pub extern "C" fn query(ckms: &mut CKMS<f32>, q: f64, 
                        quant: *mut f32) 
    -> i8 
{
    unsafe {
        if let Some((_, res)) = ckms.query(q) {
            *quant = res;
            0
        } else {
            -1
        }
    }
}
```

在 C API 中发出错误条件是棘手的。在 Rust 中，我们返回某种复合类型，例如`Option`。在 C 中，没有为你的 API 构建错误信号结构体就没有这样的事情。除了错误结构体方法之外，通常会在写入答案的指针中写入已知的无意义内容，或者返回一个负值。C 期望将答案写入由 quant 指向的 32 位浮点数中，但没有简单的方法可以将无意义的内容写入数值。因此，查询返回一个`i8`；成功时为零，失败时为负。更复杂的 API 会通过返回不同的负值来区分失败。

就这些了！当你运行`cargo build --release`时，一个遵循 C ABI 的静态库将被踢出到`target/release`目录。我们现在可以将其链接到一个 C 程序中。

# C 方面

我们的 C 程序是`c_src/main.c`。在我们定义`embed_quantiles`接口之前，我们需要一些系统头文件：

```rs
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
```

唯一稍微不寻常的是`time.h`。我们引入它是因为我们将随机浮点数推入`CKMS`结构体。请注意，这不是加密安全的随机数。以下是 C 对刚刚创建的 Rust API 的视图：

```rs
struct ckms;

struct ckms* alloc_ckms(double error);
void free_ckms(struct ckms* ckms);
void ckms_insert(struct ckms* ckms, float value);
int8_t query(struct ckms* ckms, double q, float* quant);
```

注意到`CKMS`结构体。这是一个不透明的结构体。C 现在知道有一个结构体，但它不知道它有多大。这没关系。我们所有的函数都只对这个结构体的指针进行操作，而 C 知道这个指针的大小。`main`函数很简单：

```rs
int main(void) {
  srand(time(NULL));

  struct ckms* ckms = alloc_ckms(0.001);

  for (int i=1; i <= 1000000; i++) {
    ckms_insert(ckms, (float)rand()/(float)(RAND_MAX/10000));
  }

  float quant = 0.0;
  if (query(ckms, 0.75, &quant) < 0) {
    printf("ERROR IN QUERY");
    return 1;
  }
  printf("75th percentile: %f\n", quant);

  free_ckms(ckms);
  return 0;
}
```

我们分配一个`CKMS`，其误差界限为 0.001，将 100 万个随机浮点数加载到`CKMS`中，然后查询第 75 百分位数，应该大约是 7,500。最后，函数释放`CKMS`并退出。简短而直接。

现在，使用 clang 编译器构建和链接`libembed_quantiles.a`相当容易，而使用 GCC 则稍微麻烦一些。我已经包括了一个 Makefile，它已经在 OS X 和 Linux 上进行了测试：

```rs
CC ?= $(shell which cc)
CARGO ?= $(shell which cargo)
OS := $(shell uname)

run: clean build
        ./target/release/embed_quantiles

clean:
        $(CARGO) clean
        rm -f ./target/release/embed_quantiles

build:
        $(CARGO) build --release
ifeq ($(OS),Darwin) # Assumption: CC == Clang
        $(CC) -std=c11 -Wall -Werror -Wpedantic -O3 c_src/main.c \
                -L target/release/ -l embed_quantiles \
                -o target/release/embed_quantiles
else # Assumption: CC == GCC
        $(CC) -std=c11 -Wall -Werror -Wpedantic -O3 c_src/main.c \
                -L target/release/ -lpthread -Wl,–no-as-needed -ldl \
                -lm -lembed_quantiles \
                -o target/release/embed_quantiles
endif

.PHONY: clean run
```

我没有 Windows 机器来测试，所以，嗯，当这不可避免地不起作用时，我真的很抱歉。希望这里的信息足够你快速解决问题。一旦你放好了 Makefile，你应该能够运行 make run 并看到以下类似的内容：

```rs
> make run
/home/blt/.cargo/bin/cargo clean
rm -f ./target/release/embed_quantiles
/home/blt/.cargo/bin/cargo build --release
   Compiling quantiles v0.7.1
   Compiling embed_quantiles v0.1.0
    Finished release [optimized] target(s) in 0.91 secs
cc -std=c11 -Wall -Werror -Wpedantic -O3 c_src/main.c \
        -L target/release/ -lpthread -Wl,–no-as-needed -ldl \
        -lm -lembed_quantiles \
        -o target/release/embed_quantiles
./target/release/embed_quantiles
75th percentile: 7499.636230
```

百分位数将会有所波动，但应该接近 7,500，就像现在这样。

# 进入 Python

让我们看看如何将 Rust 集成到 Python 中。我们将编写一个函数来计算由 Python 构建的 cytpes 数组中尾随零的数量。由于解释器已经编译并链接，我们无法将 Rust 静态链接到 Python，因此我们需要创建一个动态库。`Cargo.toml` 项目反映了这一点：

```rs
[package]
name = "zero_count"
version = "0.1.0"
authors = ["Brian L. Troutwine <brian@troutwine.us>"]

[dependencies]

[lib]
name = "zero_count"
crate-type = ["dylib"]
```

唯一的 Rust 文件，`src/lib.rs`，其中只有一个函数：

```rs
#[no_mangle]
pub extern "C" fn tail_zero_count(arr: *const u16, len: usize) -> u64 {
    let mut zeros: u64 = 0;
    unsafe {
        for i in 0..len {
            zeros += (*arr.offset(i as isize)).trailing_zeros() as u64
        }
    }
    zeros
}
```

`tail_zero_count` 函数接受一个指向 `u16` 数组的指针以及该数组的长度。它遍历数组，对每个 `u16` 调用 `trailing_zeros()` 并将值累加到全局变量 `zeros` 中：然后返回这个全局和。在项目的顶部运行 `cargo build --release`，你将在 `target/release` 目录下找到项目的动态库——可能是 `libzero_count.dylib`、`libzero_count.dll` 或 `libzero_count.so`，具体取决于你的主机。到目前为止，一切顺利。

现在调用此函数由 Python 负责。以下是一个小型示例，位于项目根目录下的 `zero_count.py`：

```rs
import ctypes
import random

length = 1000000
lzc = ctypes.cdll.LoadLibrary("target/release/libzero_count.dylib")
arr = (ctypes.c_uint16 * length)(*[random.randint(0, 65000) for _ in range(0, length)])
print(lzc.tail_zero_count(ctypes.pointer(arr), length))
```

我们导入 cytpes 和随机库，然后加载共享库——在这里，遵循 OS X 的命名约定——并将其绑定到 `lzc`。如果你在除 OS X 以外的操作系统上运行此代码，请编辑此代码以读取 `[so|dll]`。一旦 `lzc` 被绑定，我们就使用 ctypes API 创建一个随机的 `u16` 值数组，并调用该数组上的 `tail_zero_count`。使用这种方法，Python 被迫在将数组传递给 Rust 之前分配整个数组，所以不要增加太多长度。运行程序只需调用 Python 的 `zero_count.py`，如下所示：

```rs
> python zero_count.py
1000457
>  python zero_count.py
999401
> python zero_count.py
1002295
```

处理不透明结构指针——就像我们在 C 示例中所做的那样——在 ctypes 文档中有很好的记录。Rust 将不知道区别；它只是输出符合 C ABI 的对象。

# 进入 Erlang/Elixir

在本章的最后部分，我们将研究将 Rust 集成到 BEAM 中，这是支撑 Erlang 和 Elixir 的虚拟机。BEAM 是一个复杂的工作窃取调度系统，碰巧是可编程的。Erlang 进程是小的 C 结构体，在调度线程之间弹跳，并携带足够的信息，允许对固定数量的指令进行解释。在 Erlang/Elixir 中没有共享内存的概念：系统中的不同并发演员之间的通信*必须*通过消息传递来实现。这样做有许多好处，并且虚拟机在可能的情况下会做大量工作以避免复制。Erlang/Elixir 进程将消息接收到一个消息队列中，这是一个双端线程安全的队列，我们在整本书中讨论过。

Erlang 和 Elixir 因其高效处理实时高规模 IO 而闻名。毕竟，Erlang 是在爱立信发明的，作为电话控制软件。这些语言不为人所知的是串行性能。Erlang 中的顺序计算相对较慢，只是因为它很容易启动并发计算，这某种程度上弥补了这一点。某种程度上。有时，你需要原始的串行性能。

Erlang 对此有几个解决方案。一个*端口*通过双向字节流打开一个外部操作系统进程。我们在上一章中看到了一个类似的方法，即 feruscore 嵌入 pmars。一个*端口驱动程序*是一个链接的共享对象，通常用 C 编写。端口驱动程序接口提供了向 BEAM 的调度器提供中断提示的功能，并且有很多优点。端口驱动程序*是一个进程，必须能够处理 Erlang 进程通常需要处理的事情：服务其消息队列，处理特殊的中断信号，协作地调度自己，等等。这些都是非平凡的编写。最后，Erlang 支持原生实现函数（NIF）的概念。这些比端口驱动程序简单，并且是同步的，它们是简单地可调用的函数，碰巧是用除 Erlang 以外的语言编写的。NIF 是共享库，通常用 C 实现。端口驱动程序和 NIF 都有一个严重的缺点：内存问题会破坏 BEAM 并使你的应用程序离线。Erlang 系统通常部署在容错是一个重要因素的地方，而破坏虚拟机是一个大忌。

因此，Erlang/Elixir 社区对 Rust 有很大的兴趣。Rustler 项目（[`crates.io/crates/rustler`](https://crates.io/crates/rustler)）旨在使将 Rust 结合到 Elixir 项目中作为 NIF 变得简单。让我们看看一个简短的示例项目，由 Sonny Scroggin 在 2018 年旧金山的 Code BEAM 上展示——beamcoin（[`github.com/blt/beamcoin`](https://github.com/blt/beamcoin)）。我们将在 SHA `3f510076990588c51e4feb1df990ce54ff921a06`处讨论该项目。

beamcoin 项目并未全部列出。我们主要删除了构建配置。你可以在本书的源代码库中找到完整的列表。

Elixir 的构建系统——Rustler 原生目标语言的 BEAM 语言——被称为 Mix。其配置文件位于项目的根目录下的`mix.exs`：

```rs
defmodule Beamcoin.Mixfile do
  use Mix.Project

  def project do
    [app: :beamcoin,
     version: "0.1.0",
     elixir: "~> 1.5",
     start_permanent: Mix.env == :prod,
     compilers: [:rustler] ++ Mix.compilers(),
     deps: deps(),
     rustler_crates: rustler_crates()]
  end

  def application do
    [extra_applications: [:logger]]
  end

  defp deps do
    [{:rustler, github: "hansihe/rustler", sparse: "rustler_mix"}]
  end

  defp rustler_crates do
    [beamcoin: [
      path: "native/beamcoin",
      mode: mode(Mix.env)
    ]]
  end

  defp mode(:prod), do: :release
  defp mode(_), do: :debug
end
```

这里有很多我们不会涉及的内容。然而，请注意这个部分：

```rs
  defp rustler_crates do
    [beamcoin: [
      path: "native/beamcoin",
      mode: mode(Mix.env)
    ]]
  end
```

在`native/beamcoin`目录下嵌入的项目中，是我们将要探索的 Rust 库。其 cargo 配置文件位于`native/beamcoin/Cargo.toml`：

```rs
[package]
name = "beamcoin"
version = "0.1.0"
authors = []

[lib]
name = "beamcoin"
path = "src/lib.rs"
crate-type = ["dylib"]

[dependencies]
rustler = { git = "https://github.com/hansihe/rustler", branch = "master" }
rustler_codegen = { git = "https://github.com/hansihe/rustler", branch = "master" }
lazy_static = "0.2"
num_cpus = "1.0"
scoped-pool = "1.0.0"
sha2 = "0.7"
```

这里没有什么太令人惊讶的。当 Rust 编译这个动态库（libbeamcoin）时，我们已经看到了几乎所有的依赖项。`rustler` 和 `rustler_codegen` 分别是 Rustler 的接口和编译器生成器。`rustler_codegen` 去除了我们可能需要做的很多样板 C 外部工作。`sha2` 是来自 RustCrypto 项目的 crate，它实现了 sha2 哈希算法。Beamcoin 是一个有点玩笑性质的项目。其想法是在线程之间分配数字，并计算 sha2-256 哈希的后面的零位数，用预设的零位数进行挖掘。在 Erlang 中做这件事会非常慢，但正如我们将看到的，在 Rust 中这是一个相对快速的运算。`scoped-pool` crate 是一个线程池库，它是可发送的，这意味着它可以放在 `lazy_static!` 中。

`src/lib.rs` 的前言足够简单：

```rs
#[macro_use]
extern crate lazy_static;
extern crate num_cpus;
extern crate scoped_pool;
extern crate sha2;
#[macro_use]
extern crate rustler;

use rustler::{thread, Encoder, Env, Error, Term};
use scoped_pool::Pool;
use sha2::Digest;
use std::mem;
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::{mpsc, Arc};
```

我们之前已经看到了很多这样的内容。Rustler 的导入是 Erlang C NIF API 的类似物。挖掘的难度由一个顶级 `usize` 控制，称为 `DIFFICULTY`：

```rs
const DIFFICULTY: usize = 6;
```

这个值控制了在声明可挖掘之前，需要在哈希的后面找到的总零位数。这是我们的线程池：

```rs
lazy_static! {
    static ref POOL: Pool = Pool::new(num_cpus::get());
}
```

我之前提到过，BEAM 维护自己的调度线程。这是真的。beamcoin NIF 也维护了自己的线程池，以分配挖掘任务。现在，Rustler 减少了样板代码，但无法完全去除。例如，我们必须告诉 BEAM 与哪个符号关联的解析函数，并预先定义 *原子* 以供使用：

```rs
rustler_atoms! {
    atom ok;
    atom error;
}

rustler_export_nifs! {
    "Elixir.Beamcoin",
    [("native_mine", 0, mine)],
    None
}
```

Erlang 原子是一个命名常量。这些是在 Erlang/Elixir 程序中极其常见的数据类型。池中的每个线程都会被分配到一段正整数以供搜索，根据它们的线程号进行剥离。池中的工作者使用 MPSC 通道与挖掘线程通信，告知已找到结果：

```rs
#[derive(Debug)]
struct Solution(u64, String);

fn mine<'a>(caller: Env<'a>, _args: &[Term<'a>]) 
    -> Result<Term<'a>, Error> 
{
    thread::spawn::<thread::ThreadSpawner, _>(caller, move |env| {
        let is_solved = Arc::new(AtomicBool::new(false));
        let (sender, receiver) = mpsc::channel();

        for i in 0..num_cpus::get() {
            let sender_n = sender.clone();
            let is_solved = is_solved.clone();

            POOL.spawn(move || {
                search_for_solution(i as u64, num_cpus::get() as u64, 
                                    sender_n, is_solved);
            });
        }

        match receiver.recv() {
            Ok(Solution(i, hash)) => (atoms::ok(), i, hash).encode(env),
            Err(_) => (
                atoms::error(),
                "Worker threads disconnected before the \
                 solution was found".to_owned(),
            ).encode(env),
        }
    });

    Ok(atoms::ok().encode(caller))
}
```

`search_for_solution` 函数是一个小循环：

```rs
fn search_for_solution(
    mut number: u64,
    step: u64,
    sender: mpsc::Sender<Solution>,
    is_solved: Arc<AtomicBool>,
) -> () {
    let id = number;
    while !is_solved.load(Ordering::Relaxed) {
        if let Some(solution) = verify_number(number) {
            if let Ok(_) = sender.send(solution) {
                is_solved.store(true, Ordering::Relaxed);
                break;
            } else {
                println!("Worker {} has shut down without \
                         finding a solution.", id);
            }
        }
        number += step;
    }
}
```

在循环的顶部，池中的每个线程都会检查是否有其他线程挖掘了 beamcoin。如果有，函数退出，线程在池中可用于新的工作。否则，调用 `verify_number` 函数。该函数是：

```rs
fn verify_number(number: u64) -> Option<Solution> {
    let number_bytes: [u8; 8] = unsafe { 
        mem::transmute::<u64, [u8; 8]>(number) 
    };
    let hash = sha2::Sha256::digest(&number_bytes);

    let top_idx = hash.len() - 1;
    // Hex chars are 16 bits, we have 8 bits. /2 is conversion.
    let trailing_zero_bytes = DIFFICULTY / 2;

    let mut jackpot = true;
    for i in 0..trailing_zero_bytes {
        jackpot &= hash[top_idx - i] == 0;
    }

    if jackpot {
        Some(Solution(number, format!("{:X}", hash)))
    } else {
        None
    }
}
```

数字被传递进来，转换成一个包含 8 个成员的字节数组，并进行哈希。如果哈希具有适当数量的尾随零字节，函数结束时 jackpot 为真，并返回数字及其哈希。仔细的读者会注意到，Rust 模块导出了一个名为 `native_mine` 的 NIF。一般来说，Erlang NIF 有一个系统语言组件，作为后备有一个 BEAM 原生实现。系统语言 NIF 实现传统上被称为 `native_*`。

最后的部分是包装原生 NIF 位点的 Elixir 模块。这个模块被称为 `Beamcoin`，位于 `lib/beamcoin.ex`：

```rs
defmodule Beamcoin do
  use Rustler, otp_app: :beamcoin

  require Logger

  def mine do
    :ok = native_mine()

    start = System.system_time(:seconds)
    receive do
      {:ok, number, hash} ->
        done = System.system_time(:seconds)
        total = done - start
        Logger.info("Solution found in #{total} seconds")
        Logger.info("The number is: #{number}")
        Logger.info("The hash is: #{hash}")
      {:error, reason} ->
        Logger.error(inspect reason)
    end
  end

  def native_mine, do: :erlang.nif_error(:nif_not_loaded)
end
```

安装完 Elixir ([`elixir-lang.org/install.html`](https://elixir-lang.org/install.html)) 后，您可以移动到项目的根目录，执行 `mix deps.get` 以获取项目的依赖项，然后使用 `MIX_ENV=prod iex -S mix` 来启动 Elixir repl。后者命令看起来可能像这样：

```rs
> MIX_ENV=prod iex -S mix
Erlang/OTP 20 [erts-9.3] [source] [64-bit] [smp:4:4] [ds:4:4:10] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

Compiling NIF crate :beamcoin (native/beamcoin)...
    Finished release [optimized] target(s) in 0.70 secs
Interactive Elixir (1.6.4) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)>
```

在提示符下，输入 `Beamcoin.mine` 并按 *Enter*。几秒钟后，您应该会看到：

```rs
iex(1)> Beamcoin.mine

23:30:45.243 [info]  Solution found in 3 seconds

23:30:45.243 [info]  The number is: 10097471

23:30:45.243 [info]  The hash is: 2354BB63E90673A357F53EBC96141D5E95FD26B3058AFAD7B1F7BACC9D000000
:ok
```

或者，类似的东西。

在 BEAM 中构建 NIFs 是一个复杂的话题，我们在这里几乎没有涉及。Sonny Scroggin 的演讲，包含在 *进一步阅读* 部分，详细介绍了这个细微差别。如果您对 BEAM 的工作原理感兴趣，我在 *进一步阅读* 部分也包含了我关于这个主题的演讲。数十年的精心努力都投入到了 BEAM 中，这确实有所体现。

# 摘要

在本章中，我们讨论了在 Rust 中嵌入语言以及反之亦然。Rust 是一种非常有用的编程语言，它本身就是一个强大的工具，但经过精心设计，可以与现有的语言生态系统互操作。将困难的并发工作委托给 Rust 在一个内存不安全的环境中是一个强大的模型。Mozilla 在 Firefox 上的工作已经证明了这条道路是富有成效的。同样，有数十年的经过良好测试的库和特定领域——天气建模、物理学、20 世纪 80 年代的有趣编程游戏——理论上可以用 Rust 重新编写，但可能更好地通过安全接口来整合。

本章是最后一章，旨在教会您一项新的、广泛的技能。如果您已经读到这本书的这一部分，感谢您。为您写作是一段真正的乐趣。希望您现在对在 Rust 中进行底层并发有了坚实的基础，并且有信心阅读您遇到的绝大多数 Rust 代码库。Rust 中有很多内容，希望现在看起来更熟悉了。在本书的下一章，也是最后一章中，我们将讨论 Rust 的未来，哪些语言特性与本书相关即将到来，以及它们如何为我们带来新的能力。

# 进一步阅读

+   *Rust 编写的 FFI 示例*，可在 [`github.com/alexcrichton/rust-ffi-examples`](https://github.com/alexcrichton/rust-ffi-examples) 找到。Alex Crichton 的这个仓库是一个 Rust 中 FFI 示例的良好集合。这本书中关于这个主题的文档相当不错，但仔细阅读工作代码总不会有害。

+   *《黑客的乐趣》*，亨利·沃伦（小）。如果您喜欢本章对 feruscore 的处理中包含的位操作，您会喜欢《黑客的乐趣》。这本书现在已经过时了，其中一些算法在 64 位字上不再工作，但它仍然值得一读，尤其是如果您像我一样，努力将固定宽度类型保持得尽可能小。

+   *外部函数接口*，可在[`doc.rust-lang.org/nomicon/ffi.html`](https://doc.rust-lang.org/nomicon/ffi.html)找到。Nomicon 为压缩库 snappy 构建了一个高级包装器。这个包装器在以下方面进行了扩展，具体来说是与 C 回调和 vardic 函数调用相关。

+   *全局解释器锁*，可在[`wiki.python.org/moin/GlobalInterpreterLock`](https://wiki.python.org/moin/GlobalInterpreterLock)找到。GIL 长期以来一直是用 Python 编写的多进程软件的痛处。这篇维基百科条目讨论了强制执行 GIL 的技术细节以及历史上试图解决这个问题所做的尝试。

+   *Rust 之旅*，可在[`github.com/chucklefish/rlua/blob/master/examples/guided_tour.rs`](https://github.com/chucklefish/rlua/blob/master/examples/guided_tour.rs)找到。rlua crate 包含一个导游模块，该模块有很好的文档和可运行性。我在其他项目中没有看到过这种文档方法，我强烈建议你检查一下。首先，它对学习 rlua 很有帮助。其次，它写得很好，对读者有同情心：这是技术写作的一个很好的例子。

+   *链接*，可在[`doc.rust-lang.org/reference/linkage.html`](https://doc.rust-lang.org/reference/linkage.html)找到。这是 Rust 参考手册中的链接章节。这里的细节非常具体，但在明确链接时通常有必要。普通读者可能会或多或少地使用我们在这章中涵盖的信息，但总有一些新的领域需要特定的知识。

+   *Rust 在其他语言中*，可在[`doc.rust-lang.org/1.2.0/book/rust-inside-other-languages.html`](https://doc.rust-lang.org/1.2.0/book/rust-inside-other-languages.html)找到。这本书的这一章与本章类似——嵌入 Rust，但速度更快，并且使用了不同的高级语言。具体来说，本书涵盖了将 Rust 嵌入 Ruby 和 NodeJS，而我们没有涉及。

+   *Rust 中的 FFI - 为 libcpuid 编写绑定*，可在[`siciarz.net/ffi-rust-writing-bindings-libcpuid/`](http://siciarz.net/ffi-rust-writing-bindings-libcpuid/)找到。Zbigniew Siciarz 已经撰写了关于 Rust 的文章，并用 Rust 进行写作有一段时间了。你可能知道他的 *24 天 Rust* 系列。在这篇文章中，Sicarz 记录了为 libcpuid 构建安全包装器的过程，该库的职责是轮询操作系统以获取有关用户 CPU 的信息。

+   *用 Rust 将 Elixir 带到金属上*，Sonny Scroggin，可在[`www.youtube.com/watch?v=lSLTwWqTbKQ`](https://www.youtube.com/watch?v=lSLTwWqTbKQ)找到。在这一章中，我们展示了 Beamcoin，这是在同一项目中结合了 Elixir 和 Rust。将 NIFs 集成到 BEAM 系统是一个复杂的话题。这个在 2017 年 NDC London 上发表的演讲强烈推荐作为该主题的介绍。

+   *零散拼凑至太空：可靠性、安全性与 Erlang 原则*，作者：Brian L. Troutwine，可在[`www.youtube.com/watch?v=pwoaJvrJE_U`](https://www.youtube.com/watch?v=pwoaJvrJE_U)找到。几十年来，在 BEAM 上投入了大量的工作，使得这些语言在容错软件部署中占据了关键位置。没有检查的情况下，BEAM 是如何工作的，这本身就是一个谜。在这场演讲中，我介绍了 BEAM 的语义模型，然后从高层次上讨论了它的实现。
