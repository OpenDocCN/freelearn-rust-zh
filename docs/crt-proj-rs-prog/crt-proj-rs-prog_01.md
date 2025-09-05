Rust 2018：生产力

Rust 标准库和工具在过去几年中得到了很大的改进。自 2018 年 2 月以来，Rust 生态系统已经变得非常广泛和多样化。已经创建了四个领域工作小组，每个小组覆盖一个主要的应用领域。这些领域已经相当成熟，但这一发展使它们能够进一步改进。在未来几年，我们还将看到其他领域工作小组的引入。

即使作为一个开发者学习了语言，开发一个高质量且成本效益高的应用程序也不是一件容易的事情。为了避免重复造（可能是低质量的）轮子，作为开发者，你应该使用一个高质量的框架或一些高质量的库，这些库涵盖了你要开发的类型的应用程序。

这本书的目的是指导你作为开发者选择最适合开发软件的开源 Rust 库。本书涵盖了几个典型领域，每个领域使用不同的库。由于一些非标准库在许多不同领域都有用，如果将它们限制在单一领域，将会非常局限。

在本章中，你将学习以下主题：

+   理解 Rust 的不同版本

+   理解最近对 Rust 做出的最重要的改进

+   理解领域工作小组

+   理解本书中将涵盖的项目类型

+   一些有用的 Rust 库简介

# 第二章：技术要求

要跟随这本书，你需要访问一台安装了最新 Rust 系统的计算机。任何自 1.31 版本以来的版本都是可以的。稍后将为一些特定项目列出一些可选库。

任何引用的源代码和附加示例都可以（并且应该）从以下存储库下载：[`github.com/PacktPublishing/Creative-Projects-for-Rust-Programmers`](https://github.com/PacktPublishing/Creative-Projects-for-Rust-Programmers).

# 理解 Rust 的不同版本

在 2018 年 12 月 6 日，Rust 语言、其编译器和其标准库的一个非常重要的版本被发布：稳定版本 **1.31**。这个版本被定义为 **2018 版本**，这意味着它是一个里程碑，将作为未来年份的参考。

在此之前，还有一个版本，1.0，被定义为 **2015 版本**。这个版本的特点是 *稳定性*。直到版本 1.0，编译器的每个版本都对语言或标准库进行了破坏性更改，迫使开发者对其代码库进行大规模的更改。从版本 1.0 开始，已经做出了努力，以确保任何未来的编译器版本都可以正确编译为版本 1.0 或后续版本编写的任何代码。这被称为 **向后兼容性**。

然而，在 2018 版本发布之前，许多功能已经应用于语言和标准库。许多新库使用了这些新功能，这意味着这些库不能由较旧的编译器使用。因此，有必要将 Rust 的一个特定版本标记为旨在与较新库一起使用。这是 2018 版本的主要原因。

添加到语言的一些功能被标记为 2015 版本，而其他功能被标记为 2018 版本。2015 版本的功能只是小改进，而 2018 版本的功能是更深入的变化。开发者必须将他们的 crate 标记为 2018 版本，才能使用特定于 2018 版本的功能。

此外，尽管 2015 版本标志着语言和标准库的稳定里程碑，但命令行工具实际上并没有稳定；它们仍然相当不成熟。从 2015 年 5 月到 2018 年 12 月的三年半时间里，主要的官方命令行工具已经成熟，语言也得到了改进，以允许更高效的编码。2018 版本可以用“生产力”这个词来描述。

以下表格显示了语言、标准库和工具中稳定的功能的时间线：

| **2015** | **五月**: 2015 版本 | **八月**: 多核 CPU 上的并行编译 |  |  |  |  |  |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **2016** | **四月**: 支持微软 C 编译器格式 | **五月**: 捕获 panic 的能力 | **九月**: 改进的编译器错误信息 | **十一月**: `?` 操作符 | **十二月**: `rustup` 命令 |  |  |
| **2017** | **二月**: 自定义 derive 属性 | **三月**: cargo check 命令 | **七月**: `union` 关键字 | **八月**: 关联常量 | **十一月**: 与 Option 一起的 `?` 操作符 |  |  |

| **2018** | **二月**:

+   四个领域工作组的形成。

+   `rustfmt` 程序

| **五月**:

+   Rust 编程语言第二版。

+   `impl` 特性语言功能。

+   `main` 可以返回一个 Result。

+   使用 `..=` 的包含范围

+   本地类型 *i128* 和 *u128*。

+   `match` 的改进模式

| **六月**:

+   SIMD 库功能

+   `dyn` 特性语言功能

| **八月**: 自定义全局分配器 | **九月**:

+   `cargo fix` 命令

+   `cargo clippy` 命令

| **十月**:

+   过程宏

+   模块系统和 `use` 语句的更改

+   原始标识符

+   `no_std` 应用程序

| **十二月**:

+   2018 版本

+   非词法生命周期

+   `const fn` 语言功能

+   新的 [`www.rust-lang.org/`](https://www.rust-lang.org/) 网站

+   `try`、`async` 和 `await` 是保留字

|

自 2015 年版以来，已经应用了许多改进。更多信息可以在官方文档中找到（[`blog.rust-lang.org/2018/12/06/Rust-1.31-and-rust-2018.html`](https://blog.rust-lang.org/2018/12/06/Rust-1.31-and-rust-2018.html))。以下列出了最重要的改进：

+   一本新的官方教程书籍，免费在线提供（[`doc.rust-lang.org/book/`](https://doc.rust-lang.org/book/))，或打印在纸上（由 Steve Klabnik 和 Carol Nichols 编写的 *Rust 编程语言*）。

+   一个全新的官方网站。

+   成立了四个领域工作组，这些是开放委员会，旨在设计四个关键领域的生态系统未来：

    +   **网络编程**：围绕延迟计算的概念设计新的异步范式，称为 *future*，就像在其他语言中已经做的那样，例如 C++、C# 和 JavaScript（使用 *promises*）。

    +   **命令行应用程序**：设计一些标准库来支持任何非图形、非嵌入式应用程序。

    +   **WebAssembly**：设计工具和库来构建可在网页浏览器中运行的应用程序。

    +   **嵌入式软件**：设计工具和库来构建在裸机系统或严格受限的硬件上运行的应用程序。

+   我们见证了语言的一些良好改进：

    +   非词法生命周期；任何不再使用的绑定都被认为是 *已死亡*。例如，现在这个程序是被允许的：

```rs
fn main() {
    let mut _a = 7;
    let _ref_to_a = &_a;
    _a = 9;
}
```

在此代码中，变量 `_a` 绑定的对象在第二个语句中被变量 `_ref_to_a` 借用。在非词法生命周期的引入之前，此类绑定会持续到作用域的末尾，因此最后一个语句将是非法的，因为它试图在 `_ref_to_a` 仍然借用该对象时通过绑定 `_a` 来更改该对象。现在，因为变量 `_ref_to_a` 不再使用，其生命周期在声明它的同一行结束，因此，在最后一个语句中，变量 `_a` 再次可以自由地更改其对象。

+   +   `Impl Trait` 功能，允许函数返回未指定的类型，例如 **闭包**。

    +   `i128` 和 `u128` 本地类型。

    +   一些其他保留关键字，例如 `try`、`async` 和 `await`。

    +   `?` 操作符，即使在 `main` 函数中也可以使用，因为它现在可以返回 `Result`。以下是一个 `main` 函数返回 `Result` 的程序示例：

```rs
fn main() -> Result<(), String> {
    Err("Hi".to_string())
}
```

它可以通过返回通常的空元组或通过返回你指定的类型来成功，在这种情况下是 `String`。以下是一个使用 `?` 操作符的示例，该操作符在 `main` 函数中使用：

```rs
fn main() -> Result<(), usize> {
    let array = [12, 19, 27];
    let found = array.binary_search(&19)?;
    println!("Found {}", found);
    let found = array.binary_search(&20)?;
    println!("Found {}", found);
    Ok(())
}
```

这个程序将在标准输出流上打印 `Found 1`，这意味着数字 `19` 在位置 `1` 被找到，并且它将在标准错误流上打印 `Error: 2`，这意味着数字 `20` 没有被找到，但它应该被插入到位置 `2`。

+   +   过程宏，允许一种元编程，在编译时通过操作源代码生成 Rust 代码。

    +   在 `match` 表达式中实现更强大和更直观的模式匹配。

+   以及对标准工具的一些改进：

    +   `rustup` 程序，它允许用户轻松选择默认的编译器目标或更新工具链。

    +   `rustfix` 程序，它将 2015 版本的项目转换为 2018 版本的项目。

    +   Clippy 程序，它检查非惯用语法，并为提高代码可维护性提出代码更改建议。

    +   更快的编译速度，特别是如果只需要进行语法检查的话。

    +   **Rust 语言服务器**（**RLS**）程序，目前仍然不稳定，但它允许 IDE 和可编程编辑器发现语法错误，并提出允许的操作。

Rust 语言像任何其他编程语言一样仍在不断发展。以下领域仍需改进：

+   IDE 工具，包括语言解释器（REPL）和图形调试器

+   支持裸机实时软件开发库和工具

+   主要应用程序领域的应用程序级框架和库

本书将主要关注列表上的第三点。

# 项目

当我们编写现实世界应用程序时，Rust 语言及其标准库是不够的。需要特定类型的应用程序框架，例如 GUI 应用程序、网络应用程序或游戏。

当然，如果你使用高质量和全面的库，你可以减少需要编写的代码行数。使用库还有以下两个优点：

+   整体设计得到改进，尤其是如果您使用框架（因为它强加了一个架构到您的应用程序上），它将由知识渊博的工程师创建，并由众多用户经过时间考验。

+   错误数量将减少，因为它们将经过比您可能能够应用的更彻底的测试。

实际上有很多 Rust 库，也称为 **crates**，但大多数质量较低或应用范围相当狭窄。本书将探讨 Rust 语言一些典型应用领域的最佳质量和最完整的库。

应用程序领域如下：

+   **Web 应用程序**：有各种流行的技术，包括以下：

    +   REST 网络服务（仅后端）

    +   一个事件驱动的网络客户端（仅前端）

    +   一个完整的网络应用程序（全栈）

    +   仅前端的一个网络游戏

+   **游戏**：当我说“游戏”时，我并不是指任何娱乐性的东西。我指的是一个显示连续动画的图形应用程序，与事件驱动的图形应用程序相反，后者在事件发生之前不做任何事情，例如用户按下键、移动鼠标或从连接中接收到一些数据。除了网页浏览器中的游戏，还有桌面和笔记本电脑游戏、游戏机游戏以及移动设备游戏。然而，游戏机和移动设备目前还没有得到 Rust 的良好支持，所以本书中我们将只探讨桌面和笔记本电脑游戏。

+   **语言解释器**：有两种可以解释的语言。这两种语言都在本书中进行了介绍：

    +   **文本**：类似于编程语言、标记语言或机器命令语言

    +   **二进制**：类似于要模拟的计算机的机器语言，或编程语言的中间字节码。

+   **C 语言可调用的库**：这是 Rust 的一个重要用例：开发一个可以被其他应用程序调用的库，通常是用高级语言编写的。Rust 不能假设其他语言可以调用 Rust 代码，但它可以假设它们可以调用 C 语言代码。我们将探讨如何构建一个可以像 C 语言编写的库一样调用的库。一个特别具有挑战性的案例是构建 Linux 操作系统的模块，它臭名昭著地必须用 C 语言编写。

大多数应用程序都会从文件、通信通道或数据库中读取和写入数据。在下一章中，我们将探讨各种不同的技术，这些技术将对所有其他项目都有用。

其他应用领域尚未在此列出，因为它们在 Rust 中要么使用不多，要么还不够成熟，或者它们仍然处于变动之中。这些不成熟领域的库在几年后将完全不同。这些领域包括微控制器软件、或其他实时或低资源系统软件，以及移动或可穿戴系统软件。

# 在本书中通过示例进行实践

要跟随书中的示例，您应该从在线仓库下载所有示例：[`github.com/PacktPublishing/Creative-Projects-for-Rust-Programmers`](https://github.com/PacktPublishing/Creative-Projects-for-Rust-Programmers)。此仓库包含每个章节的子文件夹，以及章节中任何项目的子子文件夹。

例如，要运行本章中的`use_rand`项目，您应该前往`Chapter01/use_rand`文件夹，并输入`cargo run`。请注意，任何项目的最重要文件是`cargo.toml`和`src/main.rs`，因此您应该首先查看它们。

# 探索一些实用工具包

在继续查看如何使用最复杂的包之前，让我们先看看一些基本的 Rust 包。这些不是标准库的一部分，但它们在许多不同类型的项目中都很实用。所有 Rust 开发者都应该了解它们，因为它们具有通用适用性。

## 伪随机数生成器 – rand 包

生成伪随机数的能力对于多种应用都是必需的，特别是对于游戏。`rand` 包相当复杂，但它的基本用法在以下示例（命名为 `use_rand`）中展示：

```rs
// Declare basic functions for pseudo-random number generators.
use rand::prelude::*;

fn main() {
    // Create a pseudo-Random Number Generator for the current thread
    let mut rng = thread_rng();

    // Print an integer number
    // between 0 (included) and 20 (excluded).
    println!("{}", rng.gen_range(0, 20));

    // Print a floating-point number
    // between 0 (included) and 1 (excluded).
    println!("{}", rng.gen::<f64>());

    // Generate a Boolean.
    println!("{}", if rng.gen() { "Heads" } else { "Tails" });
}
```

首先，你创建一个伪随机数生成器对象。然后，你在这个对象上调用几个方法。任何生成器都必须是 **可变的**，因为任何生成都会修改生成器的状态。

`gen_range` 方法在右开区间内生成一个整数。`gen` 泛型方法生成指定类型的数字。有时，这种类型可以推断出来，就像在最后一个语句中，期望的是一个布尔值。如果生成的类型是浮点数，它在 0 和 1 之间，但不包括 1。

## 日志记录 – log 包

对于任何类型的软件，特别是对于服务器，发出日志消息的能力是必不可少的。日志架构有两个组件：

+   **API**：由 `log` 包定义

+   **实现**：由几个可能的包定义

这里，展示了使用流行的 `env_logger` 包的示例。如果你想要从库中发出日志消息，你应该只添加 API 包作为依赖项，因为定义日志实现包是应用程序的责任。

在以下示例（命名为 `use_env_logger`）中，我们展示了一个应用程序（而不是库），因此我们需要这两个包：

```rs
#[macro_use]
extern crate log;

fn main() {
    env_logger::init();
    error!("Error message");
    warn!("Warning message");
    info!("Information message");
    debug!("Debugging message");
}
```

在 Unix 类似控制台中，在运行 `cargo build` 之后，执行以下命令：

```rs
RUST_LOG=debug ./target/debug/use_env_logger
```

它将打印类似以下内容：

```rs
[2020-01-11T15:43:44Z ERROR logging] Error message
[2020-01-11T15:43:44Z WARN logging] Warning message
[2020-01-11T15:43:44Z INFO logging] Information message
[2020-01-11T15:43:44Z DEBUG logging] Debugging message
```

在命令开始处输入 `RUST_LOG=debug`，你定义了临时环境变量 `RUST_LOG`，其值为 `debug`。`debug` 级别是最高的，因此所有日志语句都会执行。相反，如果你执行以下命令，只有前三条线将被打印，因为 `info` 级别不够详细，无法打印调试信息：

```rs
RUST_LOG=info ./target/debug/use_env_logger
```

同样，如果你执行以下命令，只有前两行将被打印，因为 `warn` 级别不够详细，无法打印 `debug` 或 `info` 消息：

```rs
RUST_LOG=warn ./target/debug/use_env_logger
```

如果你执行以下命令之一，只有第一行将被打印，因为默认的日志级别是 `error`：

+   `RUST_LOG=error ./target/debug/use_env_logger`

+   `./target/debug/use_env_logger`

## 在运行时初始化静态变量 – lazy_static 包

众所周知，Rust 不允许在安全代码中存在**可变静态变量**。在安全代码中允许存在不可变静态变量，但它们必须由常量表达式初始化，可能通过调用`const fn`函数来实现。然而，编译器必须能够评估任何静态变量的初始化表达式。

有时，然而，需要在使用时初始化静态变量，因为初始值取决于输入，如命令行参数或配置选项。此外，如果变量的初始化需要很长时间，而不是在程序开始时初始化它，那么可能最好只在变量第一次使用时初始化它。这种技术被称为**延迟初始化**。

有一个小 crate，名为`lazy_static`，它只包含一个与 crate 同名宏。这可以用来解决之前提到的问题。其使用方法如下（项目名为`use_lazy_static`）：

```rs
use lazy_static::lazy_static;
use std::collections::HashMap;

lazy_static! {
    static ref DICTIONARY: HashMap<u32, &'static str> = {
        let mut m = HashMap::new();
        m.insert(11, "foo");
        m.insert(12, "bar");
        println!("Initialized");
        m
    };
}

fn main() {
    println!("Started");
    println!("DICTIONARY contains {:?}", *DICTIONARY);
    println!("DICTIONARY contains {:?}", *DICTIONARY);
}
```

这将打印以下输出：

```rs
Started
Initialized
DICTIONARY contains {12: "bar", 11: "foo"}
DICTIONARY contains {12: "bar", 11: "foo"}
```

如您所见，`main`函数首先开始执行。然后，它尝试访问`DICTIONARY`静态变量，这次访问导致变量的初始化。初始化的值，即一个引用，随后被解引用并打印出来。

最后一条语句，与之前的语句相同，不会再次执行初始化，正如您所看到的，`Initialized`文本没有再次打印出来。

## 解析命令行 – structopt crate

任何程序的命令行参数都很容易通过`std::env::args()`迭代器访问。然而，解析这些参数的代码实际上相当繁琐。为了获得更易于维护的代码，可以使用`structopt` crate，如下面的项目所示（项目名为`use_structopt`）：

```rs
use std::path::PathBuf;
use structopt::StructOpt;

#[derive(StructOpt, Debug)]
struct Opt {
    /// Activate verbose mode
    #[structopt(short = "v", long = "verbose")]
    verbose: bool,

    /// File to generate
    #[structopt(short = "r", long = "result", parse(from_os_str))]
    result_file: PathBuf,

    /// Files to process
    #[structopt(name = "FILE", parse(from_os_str))]
    files: Vec<PathBuf>,
}

fn main() {
    println!("{:#?}", Opt::from_args());
}
```

如果你执行`cargo run input1.txt input2.txt -v --result res.xyz`命令，你应该得到以下输出：

```rs
Opt {
    verbose: true,
    result_file: "res.txt",
    files: [
        "input1.tx",
        "input2.txt"
    ]
}
```

如您所见，文件名`input1.txt`和`input2.txt`已经被加载到结构的`files`字段中。`--result res.xyz`参数导致`result_file`字段被填充，而`-v`参数导致`verbose`字段被设置为`true`，而不是默认的`false`。

# 摘要

在本章中，我们介绍了新的 Rust 2018 版。我们学习了本书将要描述的项目类型。然后，我们快速浏览了四个有用的 crate，你可以在你的 Rust 代码中应用它们。

在下一章中，我们将学习如何将数据存储或检索到文件、数据库或其他应用程序中。

# 问题

1.  有没有官方的 Rust 语言学习书籍？

1.  2015 年最长的原始 Rust 整数有多长，到 2018 年底又是多长？

1.  2018 年底有哪四个领域工作组？

1.  Clippy 工具的目的何在？

1.  `rustfix`工具的目的是什么？

1.  编写一个程序，生成 10 个介于 100 和 400 之间的伪随机`f32`数字。

1.  编写一个程序，生成 10 个介于 100 到 400 之间的伪随机`i32`数字（不截断或四舍五入前一个练习中生成的数字）。

1.  编写一个程序，创建一个包含 1 到 200 之间所有平方整数的静态向量。

1.  编写一个程序，输出警告信息和信息消息，然后运行它，以便只显示警告信息。

1.  尝试解析一个包含 1 到 20 之间值的命令行参数，如果值超出范围，则输出错误信息。短选项应为`-l`，长选项应为`--level`。
