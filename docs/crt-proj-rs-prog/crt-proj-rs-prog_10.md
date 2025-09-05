创建 Linux 内核模块

任何合理的操作系统都可以通过可加载模块进行扩展。这是为了支持操作系统创建组织未特别支持的硬件，因此这些可加载模块通常被称为**设备驱动程序**。

然而，操作系统的这种可扩展性也可以被用于其他目的。例如，可以通过可加载模块在内核本身中支持特定的文件系统或网络协议，而无需更改和重新编译实际的内核。

在本章中，我们将探讨如何构建内核可加载模块，特别是针对 Linux 操作系统和 x86_64 CPU 架构。这里描述的概念和命令也适用于其他 CPU 架构。

本章将涵盖以下主题：

+   准备环境

+   创建模板模块

+   使用全局变量

+   分配内存

+   为字符设备创建驱动程序

到本章结束时，你将学习一些关于操作系统扩展模块的一般概念，特别是如何创建、管理和调试 Linux 内核模块。

# 第十一章：技术要求

要理解本章内容，应了解一些 Linux 操作系统的概念。特别是，你需要了解以下内容：

+   如何使用 Linux 命令解释器（即，**shell**）

+   如何理解 C 语言源代码

+   如何使用 GCC 编译器或 Clang 编译器

如果你没有这方面的知识，可以参考以下网络资源：

+   有许多教程教你如何使用 Linux 命令解释器。其中一个适合 Ubuntu Linux 分发初学者的教程可以在[`ubuntu.com/tutorials/command-line-for-beginners#1-overview`](https://ubuntu.com/tutorials/command-line-for-beginners#1-overview)找到。一个更高级和完整的免费书籍可以在[`wiki.lib.sun.ac.za/images/c/ca/TLCL-13.07.pdf`](https://wiki.lib.sun.ac.za/images/c/ca/TLCL-13.07.pdf)找到。

+   有许多教程教你关于 C 编程语言。其中之一是[`www.tutorialspoint.com/cprogramming/index.htm`](https://www.tutorialspoint.com/cprogramming/index.htm)。

+   Clang 编译器的参考可以在[`clang.llvm.org/docs/ClangCommandLineReference.html`](https://clang.llvm.org/docs/ClangCommandLineReference.html)找到。

本章中的代码示例仅在特定的 Linux 版本上开发和测试过——一个使用 4.15.0-72-generic 内核版本的 Linux Mint 分发——因此它们仅保证与这个版本兼容。Mint 分发源自 Debian 分发，因此它共享了 Debian 大多数命令。桌面环境无关紧要。

要运行本章中的示例，你应该有超级用户（root）权限访问基于 x86_64 架构的先前分布的系统。

要构建一个内核模块，需要编写大量的模板代码。这项工作已经在 GitHub 上的开源项目中为你完成了，GitHub 地址为[`github.com/lizhuohua/linux-kernel-module-rust`](https://github.com/lizhuohua/linux-kernel-module-rust)。该 GitHub 项目的一部分已经被复制到一个框架中，用于编写 Linux 内核模块，这些模块将在本章中使用。这可以在与本章相关的存储库的`linux-fw`文件夹中找到。

此外，为了简化，不会进行交叉编译——也就是说，内核模块将在它将被使用的相同操作系统中构建。这有点不寻常，因为通常，可加载模块是为不适合软件开发的操作系统的或架构开发的；在某些情况下，目标系统过于受限，无法运行方便的开发环境，例如微控制器。

在其他情况下，情况正好相反——目标系统对于单个开发者来说成本过高，例如超级计算机。

本章的完整源代码可以在存储库[`github.com/PacktPublishing/Creative-Projects-for-Rust-Programmers`](https://github.com/PacktPublishing/Creative-Projects-for-Rust-Programmers)的`Chapter10`文件夹中找到。

# 项目概述

在本章中，我们将探讨四个项目，展示如何构建越来越复杂的 Linux 内核模块：

+   `boilerplate`: 一个非常简单的内核模块，展示了构建你自己的模块所需的最小要求

+   `state`: 一个保持一些全局静态变量的模块——即一个**静态**状态

+   `allocating`: 一个分配堆内存的模块——即一个**动态**状态

+   `dots`: 一个实现只读字符设备的模块，它可以与文件系统路径名关联，然后可以像文件一样读取

# 理解内核模块

内核模块必须满足操作系统强加的某些要求，因此尝试用面向应用的编程语言（如 Java 或 JavaScript）编写内核模块是非常不合理的。通常，内核模块只使用汇编语言或 C 编写，有时也使用 C++。然而，Rust 被设计成一种系统编程语言，因此实际上可以使用 Rust 编写可加载的内核模块。

虽然 Rust 通常是一种可移植的编程语言——相同的源代码可以重新编译为不同的 CPU 架构和不同的操作系统——但对于内核模块来说并非如此。特定的内核模块必须为特定的操作系统设计和实现。此外，通常还需要针对特定的机器架构进行目标定位，尽管核心逻辑可以是架构无关的。因此，本章中的示例将仅针对 Linux 操作系统和 x86_64 CPU 架构。

## 准备环境

一些安装工作必须以超级用户权限执行。因此，您应该在安装系统范围包或更改内核中的任何命令之前，在命令前加上`sudo`。或者，您应该定期以超级用户身份工作。不用说，这是危险的，因为一个错误的命令可能会危及整个系统。要作为超级用户工作，请在终端中输入以下命令：

```rs
su root
```

然后，输入您的超级用户密码。

Linux 操作系统期望其模块只使用 C 语言编写。如果您想用 Rust 编写内核模块，必须使用粘合软件将您的 Rust 代码与 Linux 的 C 语言接口。

因此，必须使用 C 编译器来构建这个粘合软件。这里将使用`clang`编译器。这是**低级虚拟机**（**LLVM**）项目的一部分。

Rust 编译器还使用 LLVM 项目的库来生成机器代码。

您可以通过输入以下命令在您的 Linux 系统中安装`clang`编译器：

```rs
sudo apt update
sudo apt install llvm clang
```

注意，`apt`命令是 Debian 派生的发行版的典型命令，在许多 Linux 发行版和其他操作系统中不可用。

然后，您需要确保您的当前操作系统的 C 语言头文件已安装。您可以通过输入`uname -r`命令来发现您当前操作系统的版本。这将打印出类似`4.15.0-72-generic`的内容。您可以通过使用类似以下命令来安装特定版本的内核的头文件：

```rs
sudo apt install linux-headers-4.15.0-72-generic
```

您可以通过输入以下命令来组合这两个命令：

```rs
sudo apt install linux-headers-"$(uname -r)"
```

这将为您的系统生成正确的命令。

在撰写本文时，Linux 内核模块只能使用 Rust 编译器的`nightly`版本创建。要安装此编译器的最新版本，请输入以下命令：

```rs
rustup toolchain install nightly
```

此外，还需要 Rust 编译器的源代码和格式化 Rust 源代码的工具。您可以通过输入以下命令来确保它们已安装：

```rs
rustup component add --toolchain=nightly rust-src rustfmt 
```

为了确保默认使用 Rust 的 x86_64 架构和 Linux 的`nightly`工具链，运行以下命令：

```rs
rustup default nightly-x86_64-unknown-linux-gnu
```

如果您的系统上没有安装其他目标平台，这可以缩短为`rustup default nightly`。

我们知道`cargo`实用程序有几个子命令，例如`new`、`build`和`run`。对于这个项目，还需要一个额外的`cargo`子命令——`xbuild`子命令。这个名字代表**交叉构建**，意味着为另一个平台编译。实际上，它用于为不同于编译器运行的平台生成机器代码。在这种情况下，这意味着虽然我们正在运行的编译器是一个在用户空间运行的标准可执行文件，但我们正在生成的代码将在内核空间运行，因此它需要一个不同的标准库。您可以通过输入以下行来安装该子命令：

```rs
cargo install cargo-xbuild
```

然后，在你从 GitHub 下载了与本章相关的源代码之后，你就可以运行示例了。

注意，在下载的源代码中，每个项目都有一个文件夹，还有一个名为`linux-fw`的文件夹。这个文件夹包含开发 Linux 内核模块的框架，示例假设它位于这个位置。

# 一个模板模块

第一个项目是最小、可加载的内核模块，因此被称为**模板**。当模块加载时，它将打印一条消息，当模块卸载时，它将打印另一条消息。

在`boilerplate`文件夹中，有以下源文件：

+   `Cargo.toml`: Rust 项目的构建指令

+   `src/lib.rs`: Rust 源代码

+   `Makefile`: 生成和编译 C 语言粘合代码以及将生成的目标代码链接到内核模块的构建指令

+   `bd`: 一个用于构建内核模块调试配置的 shell 脚本

+   `br`: 一个用于构建内核模块发布配置的 shell 脚本

让我们从构建内核模块开始。

## 构建和运行内核模块

要为调试目的构建内核模块，打开`boilerplate`文件夹并输入以下命令：

```rs
./bd
```

当然，这个文件必须具有可执行权限。然而，当它从 GitHub 仓库安装时，它应该已经具有这些权限。

第一次运行此脚本时，它将构建框架本身，因此需要相当长的时间。之后，它将在几分钟内构建`boilerplate`项目。

在`build`命令完成后，当前文件夹中应该会出现几个文件。其中一个是名为`boilerplate.ko`的文件，其中`ko`（代表**内核对象**）是我们想要安装的内核模块。它的大小非常大，因为它包含大量的调试信息。

一个提供 Linux 模块文件信息的 Linux 命令是`modinfo`。你可以通过输入以下命令来使用它：

```rs
modinfo boilerplate.ko
```

这应该会打印出有关指定文件的一些信息。要将模块加载到内核中，请输入以下命令：

```rs
sudo insmod boilerplate.ko
```

`insmod`（插入模块）命令从指定的文件加载 Linux 模块并将其添加到正在运行的内核中。当然，这是一个特权操作，可能会危及整个计算机系统的安全性和安全性，因此只有超级用户才能运行它。这就解释了为什么需要使用`sudo`命令。如果命令成功；终端上不会打印任何内容。

`lsmod`（列出模块）命令打印出所有当前已加载的模块的列表。要选择你感兴趣的模块，你可以使用`grep`实用程序过滤输出。因此，你可以输入以下命令：

```rs
lsmod | grep -w boilerplate
```

如果`boilerplate`已加载，你将得到类似以下的一行：

![图片](img/d6593d77-acd5-453f-ac0c-05c578940982.png)

这行包含模块的名称、它使用的内存字节数以及这些模块的当前使用次数。

要卸载已加载的模块，你可以输入以下命令：

```rs
sudo rmmod boilerplate
```

`rmmod`（移除模块）命令从正在运行的 Linux 内核中卸载指定的模块。如果模块当前未加载，则此命令将打印错误消息并执行无操作。

现在，让我们看看这个模块的行为。Linux 有一个仅内存的日志区域，称为**内核缓冲区**。内核模块可以向该缓冲区追加文本行。当`boilerplate`模块被加载时，它将`boilerplate: Loaded`文本追加到内核缓冲区。当`boilerplate`模块被卸载时，它将`boilerplate: Unloaded`文本追加。只有内核及其模块可以写入它，但每个人都可以使用`dmesg`（即**显示消息**）实用程序来读取它。

如果你将`dmesg`输入到终端中，内核缓冲区的全部内容将被打印到终端。通常，内核缓冲区中有数千条消息，自系统上次重启以来由几个模块写入，但最后两行应该是`boilerplate`模块附加的。要查看仅最后 10 行并保持其颜色，请输入以下命令：

```rs
dmesg --color=always | tail
```

最后两行看起来应该如下所示：

![图片](img/98cc4ca1-f1c6-4db5-9ff7-0c97923ff550.png)

任何一行的第一部分，括号内的内容，是内核写入的时间戳。这是自内核开始以来的秒和微秒数。该行的其余部分由模块代码写入。

现在，我们可以看到`bd`脚本是如何构建这个内核模块的。

## 构建命令

`bd`脚本的内容如下：

```rs
#!/bin/sh
cur_dir=$(pwd)
cd ../linux-fw
cargo build
cd $cur_dir
RUST_TARGET_PATH=$(pwd)/../linux-fw cargo xbuild --target x86_64-linux-kernel-module && make
```

让我们看看代码中发生了什么：

+   第一行声明这是一个 shell 脚本，因此将使用 Bourne shell 程序来运行它。

+   第二行将当前文件夹的路径保存到一个临时变量中。

+   第三、第四和第五行进入框架文件夹，为调试配置构建框架，然后返回到原始文件夹。

+   最后一行构建了该模块本身。注意，它以`&& make`结束。这意味着在成功运行该行第一部分的命令后，必须运行该行第二部分的命令（`make`命令）。相反，如果第一部分的命令失败，则不会运行第二命令。该行以`RUST_TARGET_PATH=$(pwd)/../linux-fw`子句开始。它创建了一个名为`RUST_TARGET_PATH`的环境变量，该变量仅对命令行其余部分有效。它包含`framework`文件夹的绝对路径名。然后，调用`cargo`工具，并带有`xbuild --target x86_64-linux-kernel-module`参数。这是一个`xbuild`子命令，用于编译不同于当前平台的程序，其余的命令指定目标为`x86_64-linux-kernel-module`。这个目标是特定于我们使用的框架。为了解释如何使用此目标，有必要检查`Cargo.toml`文件，该文件包含以下代码：

```rs
[package]
name = "boilerplate"
version = "0.1.0"
authors = []
edition = "2018"

[lib]
crate-type = ["staticlib"]

[dependencies]
linux-kernel-module = { path = "../linux-fw" }

[profile.release]
panic = "abort"
lto = true

[profile.dev]
panic = "abort"
```

`package` 部分是常规的。`lib` 部分的 `crate-type` 项指定编译目标是静态链接库。

`dependencies` 部分的 `linux-kernel-module` 模块指定了包含框架的文件夹的相对路径。如果你希望将 `framework` 文件夹安装在这个项目的另一个相对位置或使用另一个名称，你应该更改此路径，以及 `RUST_TARGET_PATH` 环境变量。

多亏了这个指令，才可能使用 `cargo` 命令行中指定的目标。

剩余的部分指定，在发生恐慌时，应立即执行立即中止（无输出）操作，并且在发布配置中，**链接时优化**（**LTO**）应被激活。

完成这个 `cargo` 命令后，应该已经创建了 `target/x86_64-linux-kernel-module/debug/libboilerplate.a` 静态链接库。与其他任何 Linux 静态链接库一样，它的名称以 `lib` 开头，以 `.a` 结尾。

命令行的最后一部分运行 `make` 工具，这是一个主要用于 C 语言开发的 `build` 工具。正如 `cargo` 工具使用 `Cargo.toml` 文件来知道要做什么一样，`make` 工具使用 `Makefile` 文件来完成同样的目的。

在这里，我们不检查 `Makefile`，但只是说它读取由 `cargo` 生成的静态库，并用一些 C 语言粘合代码封装它，以生成 `boilerplate.ko` 文件，这是内核模块。

除了 `bd` 文件外，还有一个 `br` 文件，它与 `bd` 类似，但使用 `release` 选项同时运行 `cargo` 和 `make`，因此生成一个优化的内核模块。你可以通过输入以下命令来运行它：

```rs
./br
```

生成的模块将覆盖由 `bd` 创建的 `boilerplate.ko` 文件。你可以看到新文件在磁盘上要小得多，并且使用 `lsmod` 工具可以看到它在内存中也要小得多。

## 模板模块的源代码

现在，让我们来检查这个项目的 Rust 源代码。它包含在 `src/lib.rs` 文件中。第一行如下：

```rs
#![no_std]
```

这是一个指令，用于避免在这个项目中加载 Rust 标准库。实际上，标准库中的许多例程都假设作为应用程序代码运行——在用户空间，而不是内核内部——因此它们不能在这个项目中使用。当然，在这个指令之后，我们习惯使用的许多 Rust 函数将不再自动可用。

特别地，默认情况下不包含堆内存分配器，因此默认情况下不允许向量或字符串进行堆内存分配。如果你尝试使用 `Vec` 或 `String` 类型，你会得到一个 `use of undeclared type or module` 错误信息。

接下来的几行如下：

```rs
use linux_kernel_module::c_types;
use linux_kernel_module::println;
```

这些行将一些名称导入当前源文件。这些名称在框架中定义。

第一行导入了与 C 语言数据类型相对应的一些数据类型的声明。它们是必需的，以便与内核接口，内核期望模块是用 C 语言编写的。在此声明之后，您可以使用例如`c_types::c_int`表达式，它对应于 C 语言的`int`数据类型。

第二行导入了一个名为`println`的宏，就像标准库中的那样，现在已经不再可用。实际上，它可以以相同的方式使用，但不是打印到终端，而是将一行追加到内核缓冲区，前面带有时间戳。

然后，模块有两个入口点——`init_module`函数，当模块被加载时由内核调用，以及`cleanup_module`函数，当模块被卸载时由内核调用。它们由以下代码定义：

```rs
#[no_mangle]
pub extern "C" fn init_module() -> c_types::c_int {
    println!("boilerplate: Loaded");
    0
}

#[no_mangle]
pub extern "C" fn cleanup_module() {
    println!("boilerplate: Unloaded");
}
```

它们的`no_mangle`属性是向链接器发出的指令，以保留这个确切函数名，以便内核可以通过其名称找到这个函数。它的`extern "C"`子句指定函数调用约定必须是 C 语言通常使用的约定。

这些函数不接受任何参数，但第一个函数返回一个值，表示初始化的结果。`0`表示成功，而`1`表示失败。Linux 规定这个值的类型是 C 语言的`int`变量，而框架中的`c_types::c_int`类型正好代表这种二进制类型。

这两个函数将我们在上一节中看到的消息打印到内核缓冲区。此外，这两个函数都是可选的，但如果`init_module`函数缺失，链接器将发出警告。

文件的最后两行如下：

```rs
#[link_section = ".modinfo"]
pub static MODINFO: [u8; 12] = *b"license=GPL\0";
```

他们为链接器定义了一个字符串资源，以便将其插入到生成的可执行文件中。该字符串资源的名称是`.modinfo`，其值是`licence=GPL`。该值必须是一个以空字符终止的 ASCII 字符串，因为这是 C 语言中通常使用的字符串类型。本节不是必需的，但如果它缺失，链接器将发出警告。

# 使用全局变量

之前项目的模块模板只是打印了一些静态文本。然而，对于模块来说，通常有一些变量必须在模块的生命周期内访问。通常，Rust 不使用可变的全局变量，因为它们不安全，只是在`main`函数中定义它们，并将它们作为参数传递给由`main`调用的函数。然而，内核模块没有`main`函数。它们有内核调用的入口点，因此，为了保持共享的可变变量，必须使用一些不安全的代码。

`State`项目展示了如何定义和使用共享的可变变量。要运行它，进入`state`文件夹并输入`./bd`。然后，输入以下四个命令：

```rs
sudo insmod state.ko
lsmod | grep -w state
sudo rmmod state
dmesg --color=always | tail
```

让我们看看我们做了什么：

+   最后一个命令将模块加载到内核中，不会向控制台输出任何内容。

+   第二个命令将显示通过检索所有已加载的模块并过滤名为`state`的模块来加载模块。

+   第三个命令将从内核卸载模块，不会在控制台输出任何内容。

+   最后一个命令将显示此模块添加到内核缓冲区的两行。它们看起来像这样：

```rs
[123456.789012] state: Loaded
[123463.987654] state: Unloaded 1001
```

除了时间戳之外，它们与`boilerplate`示例的不同之处在于模块的名称以及第二行添加了数字`1001`。

让我们看看这个项目的源代码，展示它与`boilerplate`源代码相比的不同之处。`lib.rs`文件包含以下附加行：

```rs
struct GlobalData { n: u16 }

static mut GLOBAL: GlobalData = GlobalData { n: 1000 };
```

第一行定义了一个名为`GlobalData`的数据结构类型，它只包含一个 16 位的无符号数。第二行定义并初始化了这个类型的静态可变变量，命名为`GLOBAL`。

然后，`init_module`函数包含以下附加语句：

```rs
unsafe { GLOBAL.n += 1; }
```

这增加了全局变量。由于它被初始化为`1000`，在模块加载后，这个变量的值变为`1001`。

最后，`cleanup_module`函数中的语句被以下内容替换：

```rs
println!("state: Unloaded {}", unsafe { GLOBAL.n });
```

这将格式化和打印全局变量的值。请注意，读取和写入全局变量是一个*不安全操作*，因为它提供了对可变静态对象的访问。

`bd`和`br`文件与`boilerplate`项目中的文件相同。`Cargo.toml`和`Makefile`文件与`boilerplate`项目中的文件不同，因为将`boilerplate`字符串替换为`state`字符串。

# 分配内存

前一个项目定义了一个全局变量，但没有执行内存分配。即使在内核模块中，也可以分配内存，如`allocating`项目所示。

要运行此项目，打开`allocating`文件夹，输入`./bd`。然后，输入以下四个命令：

```rs
sudo insmod allocating.ko
lsmod | grep -w allocating
sudo rmmod allocating
dmesg --color=always | tail
```

这些命令的行为与上一个项目的相应命令非常相似，但最后一个命令会在时间戳之后打印一行文本：

```rs
allocating: Unloaded 1001 abcd 500000
```

让我们检查这个项目的源代码，看看它与`boilerplate`源代码相比有哪些不同。`lib.rs`文件包含以下附加行：

```rs
extern crate alloc;
use crate::alloc::string::String;
use crate::alloc::vec::Vec;
```

第一行明确声明需要一个内存分配器。否则，由于没有使用标准库，将不会将内存分配器链接到可执行模块。

第二行和第三行需要分别在源代码中包含`String`和`Vec`类型。否则，它们将不可用于源代码。然后，还有以下全局声明：

```rs
struct GlobalData {
    n: u16,
    msg: String,
    values: Vec<i32>,
}

static mut GLOBAL: GlobalData = GlobalData {
    n: 1000,
    msg: String::new(),
    values: Vec::new(),
};
```

现在，数据结构包含三个字段。其中两个字段`msg`和`values`在它们不为空时使用堆内存，而`GLOBAL`变量初始化了所有这些。在这里不允许内存分配，因此这些动态字段必须是空的。

在 `init_module` 函数中，与其他入口点一样，允许分配，因此以下代码是有效的：

```rs
unsafe {
    GLOBAL.n += 1;
    GLOBAL.msg += "abcd";
    GLOBAL.values.push(500_000);
}
```

这将更改全局变量的所有字段，为 `msg` 字符串和 `values` 向量分配内存。最后，在 `cleanup_module` 函数中使用以下语句访问全局变量以打印其值：

```rs
unsafe {
    println!("allocating: Unloaded {} {} {}",
        GLOBAL.n,
        GLOBAL.msg,
        GLOBAL.values[0]
    );
}
```

代码的其他部分保持不变。

# 字符设备

类 Unix 系统以其将 I/O 设备映射到文件系统的功能而闻名。除了预定义的 I/O 设备外，还可以将自定义设备定义为内核模块。内核设备可以连接到真实硬件，也可以是**虚拟的**。在本项目中，我们将构建一个虚拟设备。

在类 Unix 系统中，有两种类型的 I/O 设备：**块设备**和**字符设备**。前者在一次操作中处理字节数组（即它们是缓冲的），而后者一次只能处理一个字节，没有缓冲。

通常，设备可以读取、写入或两者兼而有之。我们的设备将是一个只读设备。因此，我们将构建一个文件系统映射的、虚拟的、只读字符设备。

## 构建字符设备

在这里，我们将构建一个字符设备驱动程序（或简称为**字符设备**）。字符设备是一种一次只能处理一个字节且没有缓冲的设备驱动程序。我们设备的操作将非常简单——对于从它读取的每个字节，它将返回一个点字符，但对于每 10 个字符，将返回一个星号而不是点。

要构建它，打开 `dots` 文件夹，并输入 `./bd`。当前文件夹中将创建几个文件，包括我们的内核模块 `dots.ko` 文件。

要安装它并检查是否已加载，请输入以下命令：

```rs
sudo insmod dots.ko
lsmod | grep -w dots
```

现在，内核模块已作为字符设备加载，但尚未映射到特殊文件。然而，您可以使用以下命令在已加载的设备中找到它：

```rs
grep -w dots /proc/devices
```

`/proc/devices` 虚拟文件包含所有已加载设备模块的列表。其中，在 `Character devices` 部分，应该有一行如下所示：

```rs
236 dots
```

这意味着存在一个名为 `dots` 的加载字符设备驱动程序，其内部标识符为 `236`。这个内部标识符也被称为**主设备号**，因为它是一对数字中的第一个数字，实际上用于识别设备。另一个数字，称为**次设备号**，未使用，但可以设置为 `0`。

主设备号可能因系统而异，也可能因加载而异，因为它是内核在模块加载时分配的。无论如何，它是一个小的正整数。

现在，我们必须将这些设备驱动程序与一个特殊文件相关联，这是一个文件系统中的入口点，可以用作文件，但实际上是一个指向设备驱动程序的句柄。这个操作是通过以下命令完成的，你应该用你在 `/proc/devices` 文件中找到的主设备号替换 `236`：

```rs
sudo mknod /dev/dots1 c 236 0
```

`mknod` Linux 命令创建一个特殊设备文件。前面的命令在 `dev` 文件夹中创建了一个名为 `dots1` 的特殊文件。

这是一个具有特权的命令，原因有两个：

+   只有超级用户可以创建特殊文件。

+   只有超级用户可以在 `dev` 文件夹中创建文件。

`c` 字符表示创建的设备将是一个字符设备。接下来的两个数字——`236` 和 `0`——是新虚拟设备的主设备号和次设备号。

注意，特殊文件（`dots1`）的名称可以与设备（`dots`）的名称不同，因为特殊文件与设备驱动程序之间的关联是通过主设备号来完成的。

创建特殊文件后，你可以从中读取一些字节。`head` 命令读取文本文件的第一行或字节。所以，输入以下命令：

```rs
head -c42 /dev/dots1
```

这将在控制台打印以下文本：

```rs
.........*.........*.........*.........*..
```

这个命令从指定的文件中读取前 42 个字节。

当询问第一个字节时，模块返回一个点。当询问第二个字节时，模块返回另一个点，以此类推，直到前九个字节。然而，当询问第 10 个字节时，模块返回一个星号。然后，这种行为会重复——在九个点之后，会不断地返回星号。实际上，只返回了 42 个字符，因为 `head` 命令从我们的设备请求了 42 个字符。

换句话说，如果模块生成的字符的序号是 10 的倍数，那么它就是一个星号；否则，它是一个点。

你可以根据 `dots` 模块创建其他特殊文件。例如，输入以下内容：

```rs
sudo mknod /dev/dots2 c 236 0
```

然后，输入以下命令：

```rs
head -c12 /dev/dots2
```

这将在控制台打印以下文本：

```rs
.......*....
```

注意，打印了 12 个字符，这是 `head` 命令要求的，但这次，星号在第 8 个字符处，而不是第 10 个。这是因为 `dots1` 和 `dots2` 这两个特殊文件都与同一个内核模块相关联，标识符为 (`236, 0`)，名称为 `dots`。该模块记得它已经生成了 42 个字符，因此，在生成七个点之后，它必须生成它的第 50 个字符，这个字符必须是星号，因为它是一个 10 的倍数。

你可以尝试输入整个文件，但这些操作永远不会自发结束，因为模块会继续生成字符，就像它是一个无限文件。尝试输入以下命令，然后通过按下 *Ctrl* +* C* 来停止它：

```rs
cat /dev/dots1
```

将会打印出一串快速流动的字符，直到你停止它。

你可以通过输入以下命令删除特殊文件：

```rs
sudo rm /dev/dots1 /dev/dots2
```

你可以通过输入以下命令卸载模块：

```rs
sudo rmmod dots
```

如果你卸载模块而不删除特殊文件，它们将无效。如果你然后尝试使用其中一个，例如通过输入`head -c4 /dev/dots1`，你会得到以下错误信息：

```rs
head: cannot open '/dev/dots1' for reading: No such device or address
```

现在，让我们通过输入以下内容来查看附加到内核缓冲区的信息：

```rs
dmesg --color=always | tail
```

你会看到打印出的最后两行将与以下内容相似：

```rs
[123456.789012] dots: Loaded with major device number 236
[123463.987654] dots: Unloaded 54
```

第一行，在模块加载时打印，也显示了模块的主要编号。最后一行，在模块卸载时打印，也显示了模块生成的总字节数（如果你没有运行`cat`命令，则为*42 + 12 = 54*）。现在，让我们看看这个模块的实现。

## dots 模块的源代码

你将在其他项目中找到的唯一相关差异是在`src/lib.rs`文件中。

首先，`src/lib.rs`文件声明了使用`Box`泛型类型，这与前一个项目中的`String`和`Vec`类似，默认情况下不包括。然后，它声明了一些其他与内核的绑定：

```rs
use linux_kernel_module::bindings::{
    __register_chrdev, __unregister_chrdev, _copy_to_user, file, file_operations, loff_t,
};
```

它们的含义如下：

+   `__register_chrdev`：在内核中注册字符设备的函数。

+   `__unregister_chrdev`：从内核中注销字符设备的函数。

+   `_copy_to_user`：从内核空间到用户空间复制一系列字节的函数。

+   `file`：表示文件的数据类型。这个项目实际上并没有使用它。

+   `file_operations`：包含在文件上实现的操作的数据类型。只有此模块实现了`read`操作。这可以看作是用户代码的视角。当用户代码*读取*时，内核模块*写入*。

+   `loff_t`：表示长内存偏移量的数据类型，如内核所使用。这个项目实际上并没有使用它。

### 全局信息

全局信息保存在以下数据类型中：

```rs
struct CharDeviceGlobalData {
    major: c_types::c_uint,
    name: &'static str,
    fops: Option<Box<file_operations>>,
    count: u64,
}
```

让我们理解前面的代码：

+   第一个字段（`major`）是设备的主要编号。

+   第二个字段（`name`）是模块的名称。

+   第三个字段（`fops`，简称**文件操作**）是实现所需文件操作的函数引用集合。这个引用集合将被分配到堆上，因此它被封装在一个`Box`对象中。任何`Box`对象必须从其创建时起封装一个有效的值，但`fops`字段引用的文件操作集合只能在内核初始化模块时创建；因此，这个字段被封装在一个`Option`对象中，它将被 Rust 初始化为`None`，并在内核初始化模块时接收一个`Box`对象。

+   最后一个字段（`count`）是生成的字节数计数器。

如预期的那样，以下是全球对象的声明和初始化：

```rs
static mut GLOBAL: CharDeviceGlobalData = CharDeviceGlobalData {
    major: 0,
    name: "dots\0",
    fops: None,
    count: 0,
};
```

该模块只包含三个函数：`init_module`、`cleanup_module` 和 `read_dot`。前两个函数分别在模块加载和卸载时由内核调用。第三个函数在每次有用户代码尝试从这个模块读取字节时由内核调用。

虽然 `init_module` 和 `cleanup_module` 函数通过它们的名称（因此它们必须具有确切的这些名称）链接，并且必须先于 `#[no_mangle]` 指令以避免 Rust 改变它们的名称，但 `read_dot` 函数将通过其地址而不是其名称传递给内核。因此，它可以有您喜欢的任何名称，并且对于它不需要 `#[no_mangle]` 指令。

### 初始化调用

让我们看看 `init_module` 函数体的一部分：

```rs
let mut fops = Box::new(file_operations::default());
fops.read = Some(read_dot);
let major = unsafe {
    __register_chrdev(
        0,
        0,
        256,
        GLOBAL.name.as_bytes().as_ptr() as *const i8,
        &*fops,
    )
};
```

在第一个语句中，创建了一个包含文件操作引用的 `file_operations` 结构体，并使用默认值放入一个 `Box` 对象中。

任何文件操作的默认值是 `None`，这意味着当需要此类操作时不会执行任何操作。我们将仅使用 `read` 文件操作，并且我们需要调用 `read_dot` 函数进行此操作。因此，在第二个语句中，此函数被分配给新创建的结构体的 `read` 字段。

第三个语句调用了 `__register_chrdev` 内核函数，该函数用于注册字符设备。此函数在网页上正式文档化，可在[`www.kernel.org/doc/html/latest/core-api/kernel-api.html?highlight=__register_chrdev#c.__register_chrdev`](https://www.kernel.org/doc/html/latest/core-api/kernel-api.html?highlight=__register_chrdev#c.__register_chrdev)找到。此函数的五个参数具有以下用途：

+   第一个参数是设备所需的主设备号。然而，如果它是 `0`，就像我们的情况一样，内核将生成主设备号并由函数返回。

+   第二个参数是从中生成次设备号的起始值。我们将从 `0` 开始。

+   第三个参数是我们请求分配的次设备号的数量。我们将分配 256 个次设备号，从 `0` 到 `255`。

+   第四个参数是我们正在注册的设备范围的名称。内核期望一个以空字符终止的 ASCII 字符串。因此，`name` 字段已声明为以二进制 `0` 结尾，在这里，一个相当复杂的表达式只是改变了这个名称的数据类型。`as_bytes()` 调用将字符串切片转换为字节切片。`as_ptr()` 调用获取此切片的第一个字节的地址。`as *const i8` 子句将此 Rust 指针转换为字节的原始指针。

+   第五个参数是文件操作结构体的地址。当执行读取操作时，内核将仅使用其 `read` 字段。

现在，让我们看看 `init_module` 函数的其余部分：

```rs
if major < 0 {
    return 1;
}
unsafe {
    GLOBAL.major = major as c_types::c_uint;
}
println!("dots: Loaded with major device number {}", major);
unsafe {
    GLOBAL.fops = Some(fops);
}
0
```

调用 `__register_chrdev` 返回的主要数字应该是由内核生成的一个非负数。只有在出错的情况下才会返回负数。由于我们希望在注册失败时失败模块的加载，因此我们返回 `1`——在这种情况下，表示模块加载失败。

在成功的情况下，主要数字存储在我们全局结构的 `major` 字段中。然后，将成功消息添加到内核缓冲区，包含生成的次要数字。

最后，将 `fops` 文件操作结构存储在全局结构中。

注意，在注册调用之后，内核保留 `fops` 结构的地址，因此在此函数注册期间不应更改此地址。然而，这成立，因为此结构是由 `Box::new` 调分配的，并且 `fops` 的赋值只是移动 `Box` 对象，即堆对象的指针，而不是堆对象本身。这解释了为什么使用 `Box` 对象。

### 清理调用

现在，让我们看看 `cleanup_module` 函数的主体：

```rs
unsafe {
    println!("dots: Unloaded {}", GLOBAL.count);
    __unregister_chrdev(
        GLOBAL.major,
        0,
        256,
        GLOBAL.name.as_bytes().as_ptr() as *const i8,
    )
}
```

第一条语句将卸载消息打印到内核缓冲区，包括自模块加载以来从该模块读取的总字节数。

第二条语句调用 `__unregister_chrdev` 内核函数，该函数注销先前注册的字符设备。此函数在网页上正式文档化，可在[`www.kernel.org/doc/html/latest/core-api/kernel-api.html?highlight=__unregister_chrdev#c.__unregister_chrdev`](https://www.kernel.org/doc/html/latest/core-api/kernel-api.html?highlight=__unregister_chrdev#c.__unregister_chrdev)找到。

其参数与用于注册设备的函数的前四个参数相当相似。它们必须与相应的注册值相同。然而，在注册函数中，我们指定 `0` 作为主要数字，而在这里我们必须指定实际的主要数字。

### 读取函数

最后，让我们看看内核每次某些用户代码尝试从该模块读取一个字节时将调用的函数的定义：

```rs
extern "C" fn read_dot(
    _arg1: *mut file,
    arg2: *mut c_types::c_char,
    _arg3: usize,
    _arg4: *mut loff_t,
) -> isize {
    unsafe {
        GLOBAL.count += 1;
        _copy_to_user(
            arg2 as *mut c_types::c_void,
            if GLOBAL.count % 10 == 0 { "*" } else { "." }.as_ptr() as *const c_types::c_void,
            1,
        );
        1
    }
}
```

此外，此函数必须由 `extern "C"` 子句装饰，以确保其调用约定与内核使用的相同，即系统 C 语言编译器使用的约定。

此函数有四个参数，但我们只会使用第二个。此参数是指向用户空间中结构的指针，其中必须写入生成的字符。函数的主体只包含三条语句。

第一条语句增加用户代码（由内核模块编写）读取的总字节数。

第二个语句是对内核函数`_copy_to_user`的调用。当你想要从一个由内核代码控制的内存区域复制一个或多个字节到由用户代码控制的内存区域时，可以使用此函数，因为此操作不允许简单的赋值。此函数在[`www.kernel.org/doc/htmldocs/kernel-api/API---copy-to-user.html`](https://www.kernel.org/doc/htmldocs/kernel-api/API---copy-to-user.html)有官方文档说明。

它的第一个参数是目标地址，这是我们想要写入字节的内存位置。在我们的例子中，这仅仅是`read_dot`函数的第二个参数，转换成适当的数据类型。

第二个参数是源地址，这是我们想要返回给用户的字节的内存位置。在我们的例子中，我们希望在九个点之后返回一个星号。因此，我们检查读取字符的总数是否是`10`的倍数。在这种情况下，我们使用只包含一个星号的静态字符串切片：否则，我们有一个包含点的字符串切片。对`as_ptr()`的调用获取字符串切片的第一个字节的地址，而`as *const c_types::c_void`子句将其转换为与 C 语言的`const void *`数据类型相对应的预期数据类型。

第三个参数是要复制的字节数。当然，在我们的例子中，这是`1`。

这就是生成点和星号所需的所有内容。

# 摘要

在本章中，我们探讨了可以使用 Rust 语言而不是典型的 C 编程语言来创建 Linux 操作系统内核的可加载模块的工具和技术。

特别是，我们看到了在 Mint 发行版和 x86_64 架构上可以使用的一系列命令，用于配置构建和测试可加载内核模块的适当环境。我们还研究了`modinfo`、`lsmod`、`insmod`、`rmmod`、`dmesg`和`mknod`命令行工具。

我们看到，为了创建内核模块，拥有一个针对 Rust 编译器的目标框架的代码框架是有用的。使用这个目标，Rust 源代码被编译成 Linux 静态库。然后，这个库与一些 C 语言的粘合代码链接，形成一个可加载的内核模块。

我们创建了四个逐渐增加复杂性的项目——`boilerplate`、`state`、`allocating`和`dots`。特别是，`dots`项目创建了一个模块，可以使用`mknod`命令将其映射到特殊文件；在此映射之后，当读取这个特殊文件时，会生成一系列点和星号。

在下一章和最后一章中，我们将考虑 Rust 生态系统在未来几年内的进步——语言、标准库、标准工具以及免费提供的库和工具。还包括对新增的异步编程的支持的描述。

# 问题

1.  什么是 Linux 可加载内核模块？

1.  Linux 内核期望使用哪种编程语言为其模块编写？

1.  什么是内核缓冲区以及它每一行的第一部分是什么？

1.  `modinfo`、`lsmod`、`insmod`和`rmmod`Linux 命令的目的是什么？

1.  为什么默认情况下，`String`、`Vec`和`Box`数据类型在 Rust 代码构建内核模块时不可用？

1.  `#[no_mangle]`Rust 指令的目的是什么？

1.  `extern "C"`Rust 子句的目的是什么？

1.  `init_module`和`cleanup_module`函数的目的是什么？

1.  `__register_chrdev`和`__unregister_chrdev`函数的目的是什么？

1.  应该使用哪个函数将字节序列从内核空间内存复制到用户空间内存？

# 进一步阅读

本章项目中使用的框架是对开源仓库的修改，该仓库可以在[`github.com/lizhuohua/linux-kernel-module-rust`](https://github.com/lizhuohua/linux-kernel-module-rust)找到。此仓库包含有关此主题的更多示例和文档。

Linux 内核的文档可以在[`www.kernel.org/doc/html/latest/`](https://www.kernel.org/doc/html/latest/)找到。
