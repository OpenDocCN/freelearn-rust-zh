# 第二章：Rust 及其生态系统的介绍

Rust 编程语言由 Mozilla 赞助，并得到全球开发者社区的支持。Rust 被推广为一种支持自动内存管理（无需运行时或垃圾收集器开销）、无数据竞争的并发（由编译器强制执行）以及零成本抽象和泛型的系统编程语言。在随后的章节中，我们将更详细地讨论这些特性。Rust 是静态类型，并借鉴了许多函数式编程思想。Rust 的一个迷人之处在于，它使用类型系统来保证内存安全，而不使用运行时。这使得 Rust 特别适合低资源嵌入式设备和实时系统，这些系统需要对代码正确性提供强有力的保证。另一方面，这通常意味着编译器必须做更多的工作来确保语法正确性，然后翻译源代码，从而导致构建时间更长。尽管社区正在努力尽可能减少编译时间，但这仍然是许多开发者遇到的一个重要问题。

**低级虚拟机**（**LLVM**）项目最初是一个大学研究项目，旨在开发一套构建编译器的工具，这些编译器可以为一系列 CPU 架构生成机器代码。这是通过使用**LLVM 中间表示**（**LLVM IR**）实现的。工具链可以将任何高级语言编译成 LLVM IR，然后针对特定的 CPU 进行编译。Rust 编译器严重依赖于 LLVM 项目以实现互操作性，将其用作后端。它实际上将 Rust 代码翻译成 LLVM 的中间表示，并根据需要进行优化。然后 LLVM 将其转换为特定平台的机器代码，该代码在 CPU 上运行。

在本章中，我们将涵盖以下主题：

+   生态系统介绍和 Rust 的工作原理

+   安装 Rust 和设置工具链

+   从借用检查器和所有权的工作原理开始，介绍其主要特性

+   泛型和如何与所有权模型一起工作的特质系统

+   错误处理和宏系统

+   并发原语

+   测试原语

注意，本章是对语言及其一些最显著特性的非常高级概述，而不是深入探讨。

# Rust 生态系统

开源项目的成功或失败往往取决于其周围社区的强度。拥有一个连贯的生态系统有助于构建一个强大的社区。由于 Rust 主要由 Mozilla 推动，因此他们能够围绕它构建一个强大的生态系统，主要组成部分包括：

+   源代码：Rust 在 GitHub 上托管所有源代码。开发者被鼓励在 GitHub 上报告错误和提交拉取请求。在撰写本书时，GitHub 上的 Rust 仓库有 1,868 个独特的贡献者，超过 2,700 个开放的错误报告和 90 个开放的拉取请求。Rust 的核心团队由 Mozilla 员工和其他组织（如 Google、百度等）的贡献者组成。团队使用 GitHub 进行所有协作；即使是任何组件的重大更改，也必须首先通过撰写 **请求评论**（**RFC**）来提出。这样，每个人都有机会查看它并协作改进它。一旦获得批准，实际更改就可以实施。

+   编译器：Rust 编译器的名称为 *rustc*。由于 Rust 对编译器版本采用语义版本控制，因此在小版本之间不可能有任何向后不兼容的破坏性更改。在撰写本书时，编译器已经达到 1.0 版本，因此可以假设在 2.0 版本之前不会有任何破坏性更改。请注意，偶尔确实会有破坏性更改发生。但在所有这些情况下，它们都被视为错误，并尽快修复。

为了在不破坏现有依赖库的情况下方便添加新的编译器功能，Rust 以阶段的方式发布新的编译器版本。在任何时候，都维护着三个不同的编译器版本（以及标准库）。

+   第一个版本被称为 **nightly**。正如其名所示，它每晚从源代码树的顶端构建。由于这个版本只经过单元和集成测试，因此在现实世界中通常会有更多错误。

+   第二阶段是 **beta**，这是一个计划中的发布版本。当一个夜间版本达到这个阶段时，它已经经过了多次单元、集成和回归测试。此外，社区也有时间在实际项目中使用它并与 Rust 团队分享反馈。

+   一旦每个人都对发布版本有信心，它就会被标记为 **稳定** 版本并发布。由于编译器支持各种平台（从 Windows 到 Redox）和各种架构（amd64），每个版本都为所有平台和架构的组合提供了预构建的二进制文件。

+   安装机制：社区支持的安装机制是通过一个名为 *rustup* 的工具。这个工具可以安装指定版本的 Rust 以及使用它所需的所有内容（包括编译器、标准库、包管理器等）。

+   包管理器：Rust 的包管理器称为 *Cargo*，而单个包称为 *crates*。所有外部库和应用程序都可以打包成一个 crate，并使用 Cargo CLI 工具发布。用户可以使用它来搜索和安装包。所有 crate 都可以使用以下网站进行搜索：[`crates.io/`](https://crates.io/)。对于所有托管在 *crates.io* 上的包，相应的文档可在：[`docs.rs/`](https://docs.rs/) 上找到。

# Rust 入门

Rust 工具链安装程序可在[`www.rustup.rs/`](https://www.rustup.rs/)找到。以下命令将在系统上安装工具链的所有三个版本。对于本书中的示例，我们将使用运行 Ubuntu 16.04 的 Linux 机器。虽然 Rust 的大部分内容不应依赖于操作系统，但可能会有一些细微的差异。

我们将指出对操作系统的任何严格依赖：

```rs
# curl https://sh.rustup.rs -sSf | sh
# source $HOME/.cargo/env
# rustup install nightly beta
```

我们需要通过编辑**.bashrc**将 Cargo 的 bin 目录添加到我们的**PATH**中。运行以下命令来完成此操作：

```rs
$ echo "export PATH=$HOME/.cargo/bin:$PATH" >> ~/.bashrc
```

Rust 的安装自带了大量内置文档；可以通过运行以下命令来访问它们。这应该在浏览器窗口中打开文档：

```rs
# rustup doc
```

下一步是设置 Rust 项目并运行它，全部使用 Cargo：

```rs
# cargo new --bin hello-rust
```

这告诉 Cargo 在当前目录下设置一个名为`hello-rust`的新项目。Cargo 将创建一个同名目录并设置基本结构。由于此项目的类型被设置为二进制，Cargo 将生成一个名为`main.rs`的文件，该文件将包含一个空的`main`函数，这是应用程序的入口点。这里的另一个（默认）选项是库，在这种情况下，将生成一个名为`lib.rs`的文件。名为`Cargo.toml`的文件包含当前项目的元数据，并由 Cargo 使用。所有源代码都位于`src`目录中：

```rs
# tree hello-rust/
hello-rust/
├── Cargo.toml
└── src
    └── main.rs

1 directory, 2 files
```

然后，可以使用以下命令构建和运行项目。请注意，此命令应在 Cargo 之前创建的`hello-rust`目录中运行：

```rs
# cargo run
```

有趣的是，此命令对目录进行了相当大的修改。`target`目录包含编译工件。其结构高度依赖于平台，但始终包括在给定构建模式下运行应用程序所需的所有内容。默认构建模式是`debug`，它包括用于调试器的调试信息和符号：

```rs
# tree hello-rust/
hello-rust/
├── Cargo.lock
├── Cargo.toml
├── src
│   └── main.rs
└── target
    └── debug
        ├── build
        ├── deps
        │   └── hello_rust-392ba379262c5523
        ├── examples
        ├── hello-rust
        ├── hello-rust.d
        ├── incremental
        └── native

8 directories, 6 files
crates.io:
```

```rs
# cargo search term
    Updating registry `https://github.com/rust-lang/crates.io-index`
term = "0.4.6" # A terminal formatting library
ansi_term = "0.9.0" # Library for ANSI terminal colours and styles (bold, underline)
term-painter = "0.2.4" # Coloring and formatting terminal output
term_size = "0.3.0" # functions for determining terminal sizes and dimensions
rust_erl_ext = "0.2.1" # Erlang external term format codec.
slog-term = "2.2.0" # Unix terminal drain and formatter for slog-rs
colored = "1.5.2" # The most simple way to add colors in your terminal
term_grid = "0.1.6" # Library for formatting strings into a grid layout
rust-tfidf = "1.0.4" # Library to calculate TF-IDF (Term Frequency - Inverse Document Frequency) for generic documents
aterm = "0.20.0" # Implementation of the Annotated Terms data structure
... and 1147 crates more (use --limit N to see more)
hello and world in green and red, respectively:
```

```rs
[package]
name = "hello-rust"
version = "0.1.0"
authors = ["Foo Bar <foo.bar@foobar.com>"]

[dependencies]
term = "0.4.6"
hello world! on the screen. Further, it prints hello in green and world! in red:
```

```rs
// chapter2/hello-rust/src/main.rs

extern crate term;

fn main() {
    let mut t = term::stdout().unwrap();
    t.fg(term::color::GREEN).unwrap();
    write!(t, "hello, ").unwrap();

    t.fg(term::color::RED).unwrap();
    writeln!(t, "world!").unwrap();

    t.reset().unwrap();
}
```

在 Rust 中，每个应用程序都必须有一个名为`main`的单个入口点，该入口点应定义为不带参数的函数。函数使用`fn`关键字定义。短语`extern crate term`告诉工具链我们想要将外部 crate 作为当前应用程序的依赖项。

现在，我们可以使用 Cargo 运行它。它将自动下载和构建所需的库及其所有依赖项。最后，它调用 Rust 编译器，以便我们的应用程序与库链接并运行可执行文件。Cargo 还生成一个名为`Cargo.lock`的文件，该文件包含以一致方式运行应用程序所需的所有内容的快照。此文件不应手动编辑。由于 cargo 在本地缓存所有依赖项，后续调用不需要互联网访问：

```rs
$ cargo run
    Updating registry `https://github.com/rust-lang/crates.io-index`
   Compiling term v0.4.6
   Compiling hello-rust v0.1.0 (file:///Users/Abhishek/Desktop/rust-book/src/chapter2/hello-rust)
    Finished dev [unoptimized + debuginfo] target(s) in 3.20 secs
     Running `target/debug/hello-rust`
hello, world!
```

# 借用检查器简介

Rust 最重要的方面是所有权和借用模型。基于对借用规则的严格执行，编译器可以在没有外部垃圾收集器的情况下保证内存安全。这是通过借用检查器完成的，它是编译器的一个子系统。根据定义，每个创建的资源都有一个生命周期和与之关联的所有者，它遵循以下规则：

+   在任何时间点，每个资源都恰好有一个所有者。默认情况下，所有者是创建该资源的变量，其生命周期与包含的作用域相同。其他人如果需要可以借用或复制该资源。请注意，资源可以是任何东西，从变量或函数。函数从其调用者那里接管资源；从函数返回时，所有权会转移回去。

+   当所有者的作用域执行完毕后，它拥有的所有资源都将被丢弃。这是由编译器静态计算的，然后根据此产生机器代码。

以下代码片段展示了这些规则的一些示例：

```rs
// chapter2/ownership-heap.rs

fn main() {
    let s = String::from("Test");
    heap_example(s);
}

fn heap_example(input: String) {
    let mystr = input;
    let _otherstr = mystr;
    println!("{}", mystr);
}
```

在 Rust 中，使用`let`关键字声明变量。所有变量默认是不可变的，可以通过使用`mut`关键字使其可变。`::`语法指的是给定命名空间中的对象，在这种情况下是`from`函数。`println!`是编译器提供的一个内置宏，用于写入标准输出并带有换行符。函数使用`fn`关键字定义。当我们尝试构建它时，我们得到以下错误：

```rs
# rustc ownership-heap.rs
error[E0382]: use of moved value: `mystr`
 --> ownership-heap.rs:9:20
  |
8 | let _otherstr = mystr;
  | --------- value moved here
9 | println!("{}", mystr);
  | ^^^^^ value used here after move
  |
  = note: move occurs because `mystr` has type `std::string::String`, which does not implement the `Copy` trait

error: aborting due to previous error
```

在这个例子中，创建了一个字符串资源，由函数`heap_example`中的变量`mystr`拥有。因此，其生命周期与作用域相同。在底层，由于编译器在编译时不知道字符串的长度，它必须将其放置在堆上。所有者变量在栈上创建，并指向堆上的资源。当我们将资源分配给新变量时，资源现在由新变量拥有。Rust 会在此时将`mystr`标记为无效，以防止与资源相关的内存可能被多次释放的情况。因此，编译失败以确保内存安全。我们可以强制编译器复制资源，并让第二个所有者指向新创建的资源。为此，我们需要`.clone()`名为`mystr`的资源。以下是它的样子：

```rs
// chapter2/ownership-heap-fixed.rs

fn main() {
    let s = String::from("Test");
    heap_example(s);
}

fn heap_example(input: String) {
    let mystr = input;
    let _otherstr = mystr.clone();
    println!("{}", mystr);
}
```

如预期的那样，在编译时没有抛出任何错误，并且在运行时打印了给定的字符串`"Test"`。注意，到目前为止，我们一直在使用 Cargo 来运行我们的代码。由于在这种情况下，我们只有一个简单的文件，没有外部依赖，我们将直接使用 Rust 编译器来编译我们的代码，并且我们将手动运行它：

```rs
$ rustc ownership-heap-fixed.rs && ./ownership-heap-fixed
Test
```

考虑以下代码示例，它展示了资源存储在栈上的情况：

```rs
// chapter2/ownership-stack.rs

fn main() {
    let i = 42;
    stack_example(i);
}

fn stack_example(input: i32) {
    let x = input;
    let _y = x;
    println!("{}", x);
}
```

有趣的是，尽管它与之前的代码块看起来完全相同，但这不会抛出编译错误。我们直接使用 Rust 编译器从命令行构建和运行这个程序：

```rs
# rustc ownership-stack.rs && ./ownership-stack
42
```

差别在于变量的类型。在这里，原始所有者和资源都是在栈上创建的。当资源被重新分配时，它会被复制到新的所有者那里。这是可能的，因为编译器知道整数的尺寸总是固定的（因此可以放在栈上）。Rust 提供了一种特殊的方式来表示一个类型可以通过 `Copy` 特性放在栈上。我们的例子之所以可行，仅仅是因为内置整数（以及一些其他类型）被标记了此特性。我们将在后续章节中更详细地解释特性系统。

有些人可能已经注意到，将未知长度的资源复制到一个函数中可能会导致内存膨胀。在许多语言中，调用者会传递一个指向内存位置的指针，然后传递给函数。Rust 通过使用引用来实现这一点。这些引用允许你引用一个资源，而不必真正拥有它。当一个函数接收一个资源的引用时，我们说它借用了那个资源。在下面的例子中，函数 `heap_example` 借用了变量 `s` 所拥有的资源。由于借用不是绝对的所有权，借用变量的作用域不会影响与资源相关的内存释放方式。这也意味着在函数中不可能多次释放借用的资源，因为函数作用域内没有人真正拥有那个资源。因此，之前失败的代码在这个情况下是可行的：

```rs
// chapter2/ownership-borrow.rs

fn main() {
    let s = String::from("Test");
    heap_example(&s);
}

fn heap_example(input: &String) {
    let mystr = input;
    let _otherstr = mystr;
    println!("{}", mystr);
}
```

借用规则还意味着借用是不可变的。然而，可能会有需要修改借用的借用的情况。为了处理这些情况，Rust 允许可变引用（或借用）。正如预期的那样，这让我们回到了第一个例子中的问题，并且编译失败，以下代码：

```rs
// chapter2/ownership-mut-borrow.rs

fn main() {
    let mut s = String::from("Test");
    heap_example(&mut s);
}

fn heap_example(input: &mut String) {
    let mystr = input;
    let _otherstr = mystr;
    println!("{}", mystr);
}
```

注意，一个资源在作用域内只能被可变借用一次。编译器将拒绝编译尝试做其他事情的代码。虽然这看起来可能像是一个令人烦恼的错误，但你需要记住，在一个工作应用程序中，这些函数通常会从竞争的线程中被调用。如果由于编程错误导致同步错误，我们最终会得到一个数据竞争，其中多个未同步的线程竞相修改相同的资源。这个特性有助于防止这种情况。

另一个与引用密切相关的高级语言特性是生存期的概念。引用在作用域内存在时存活，因此其生存期是封装作用域的生存期。在 Rust 中声明的所有变量都可以有一个显式的生存期省略，这给它赋予了一个名称。这对于借用检查器推理变量的相对生存期很有用。一般来说，不需要为每个变量都指定显式的生存期名称，因为编译器会管理这一点。在某些场景下，这很有必要，尤其是在自动生存期确定无法工作的情况下。让我们看看一个发生这种情况的例子：

```rs
// chapter2/lifetime.rs

fn main() {
    let v1 = vec![1, 2, 3, 4, 5];
    let v2 = vec![1, 2];

    println!("{:?}", longer_vector(&v1, &v2));
}

fn longer_vector(x: &[i32], y: &[i32]) -> &[i32] {
    if x.len() > y.len() { x } else { y }
}
```

`vec!` 宏从给定的一组对象列表中构建一个向量。请注意，与之前的例子不同，我们这里的函数需要返回一个值给调用者。我们需要使用箭头语法来指定返回类型。这里，我们给出了两个向量，我们想打印出两个中最长的。我们的 `longer_vector` 函数正是这样做的。它接收两个向量的引用，计算它们的长度，并返回长度较大的那个向量的引用。这会导致以下错误而无法编译：

```rs
# rustc lifetime.rs
error[E0106]: missing lifetime specifier
 --> lifetime.rs:8:43
  |
8 | fn longer_vector(x: &[i32], y: &[i32]) -> &[i32] {
  | ^ expected lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `x` or `y`

error: aborting due to previous error
```

这告诉我们编译器无法确定返回的引用应该指向第一个参数还是第二个参数，因此它无法确定其应该存活多长时间。由于我们没有控制输入，所以在编译时无法确定这一点。这里的一个关键洞察是，我们不需要在编译时知道所有引用的生存期。我们需要确保以下事项成立：

+   两个输入应该具有相同的生存期，因为我们想在函数中比较它们的长度

+   返回值应该具有与输入相同的生存期，即两者中较长的那个

给定这两个公理，可以得出结论，两个输入和返回值应该具有相同的生存期。我们可以像以下代码片段所示那样进行注释：

```rs
fn longer_vector<'a>(x: &'a[i32], y: &'a[i32]) -> &'a[i32] {
    if x.len() > y.len() { x } else { y }
}
```

这正如预期的那样工作，因为编译器可以愉快地保证代码的正确性。生存期参数也可以附加到结构体和方法定义上。有一个特殊的生存期称为 `'static`，它指的是整个程序的生命周期。

Rust 最近接受了一个提议，要添加一个新的指定生存期 `'fn`，它将与最内层函数或闭包的作用域相等。

# 泛型和特质系统

Rust 支持编写在编译时或运行时绑定到更具体类型的泛型代码。熟悉 C++ 模板的开发者可能会注意到，Rust 中的泛型在语法上与模板非常相似。以下示例说明了如何使用泛型编程。我们还将介绍一些之前未讨论过的新结构，我们将在继续进行时进行解释。

与 C 和 C++类似，Rust 的`struct`定义了一个用户自定义的类型，它将多个逻辑上连接的资源聚合在一个单元中。我们这里的`struct`定义了一个包含两个变量的元组。我们定义了一个泛型`struct`，并使用了一个泛型`type parameter`，在这里写作`<T>`。`struct`中的每个成员都被定义为该类型。我们后来定义了一个泛型函数，用于计算元组的两个元素的和。让我们看看这个简单的实现：

```rs
// chapter2/generic-function.rs

struct Tuple<T> {
    first: T,
    second: T,
}

fn main() {
    let tuple_u32: Tuple<u32> = Tuple {first: 4u32, second: 2u32 };
    let tuple_u64: Tuple<u64> = Tuple {first: 5u64, second: 6u64 };
    println!("{}", sum(tuple_u32));
    println!("{}", sum(tuple_u64));

    let tuple: Tuple<String> = Tuple {first: "One".to_owned(), second: "Two".to_owned() };
    println!("{}", sum(tuple));
}

fn sum<T>(tuple: Tuple<T>) -> T
{
    tuple.first + tuple.second
}
```

这将无法编译，编译器抛出了以下错误：

```rs
$ rustc generic-function.rs
error[E0369]: binary operation `+` cannot be applied to type `T`
  --> generic-function-error.rs:18:5
   |
18 | tuple.first + tuple.second
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: `T` might need a bound for `std::ops::Add`

error: aborting due to previous error
```

这个错误很重要。编译器告诉我们它不知道如何将两个类型为`T`的操作数相加。它还（正确地）猜测`T`类型需要通过`Add`特质进行绑定。这意味着`T`可能的实际类型列表应该只包含实现了`Add`特质的类型，这些类型的实际引用可以进行相加。让我们继续在`sum`函数中添加这个特质绑定。现在我们的代码应该看起来像这样：

```rs
// chapter2/generic-function-fixed.rs

use std::ops::Add;

struct Tuple<T> {
    first: T,
    second: T,
}

fn main() {
    let tuple_u32: Tuple<u32> = Tuple {first: 4u32, second: 2u32 };
    let tuple_u64: Tuple<u64> = Tuple {first: 5u64, second: 6u64 };
    println!("{}", sum(tuple_u32));
    println!("{}", sum(tuple_u64));

    // These lines fail to compile
    let tuple: Tuple<String> = Tuple {first: "One".to_owned(), second: "Two".to_owned() };
    println!("{}", sum(tuple));
}

// We constrain the possible types of T to those which implement the Add trait
fn sum<T: Add<Output = T>>(tuple: Tuple<T>) -> T
{
    tuple.first + tuple.second
}
```

为了使这可行，元素必须是可求和的；它们相加应该有一个逻辑意义。因此，我们将`T`参数可能的类型限制为实现了`Add`特质的那些类型。我们还需要让编译器知道这个函数的输出应该是`T`类型。有了这些信息，我们可以构造元组并调用求和函数，它们将按预期行为。此外，请注意，字符串的元组将无法编译，因为字符串没有实现`Add`特质。

从最后一个例子中，人们可能会注意到特质对于正确实现泛型是必不可少的。它们帮助编译器推理泛型类型的属性。本质上，一个特质定义了一个类型的属性。库定义了一组常用特质及其内置类型的实现。对于任何用户自定义类型，用户需要通过定义和实现特质来定义这些类型应该具有的属性。

```rs
// chapter2/traits.rs

trait Max<T> {
    fn max(&self) -> T;
}

struct ThreeTuple<T> {
    first: T,
    second: T,
    third: T,
}

// PartialOrd enables comparing
impl<T: PartialOrd + Copy> Max<T> for ThreeTuple<T> {
    fn max(&self) -> T {
        if self.first >= self.second && self.first >= self.third {
            self.first
        }
        else if self.second >= self.first && self.second >= self.third {
            self.second
        }
        else {
            self.third
        }
    }
}

struct TwoTuple<T> {
    first: T,
    second: T,
}

impl<T: PartialOrd + Copy> Max<T> for TwoTuple<T> {
    fn max(&self) -> T {
        if self.first >= self.second {
            self.first
        } else { self.second }
    }
}

fn main() {
    let two_tuple: TwoTuple<u32> = TwoTuple {first: 4u32, second: 2u32 };
    let three_tuple: ThreeTuple<u64> = ThreeTuple {first: 6u64,
                                       second: 5u64, third: 10u64 };

    println!("{}", two_tuple.max());
    println!("{}", three_tuple.max());
}
```

我们首先定义一个泛型特性类型`T`。我们的特性有一个单一的功能，它将返回实现它的给定类型的最大值。什么算是最大值是一个实现细节，在这个阶段并不重要。然后我们定义一个包含三个元素的元组，每个元素都是相同的泛型类型。稍后，我们为定义的类型实现我们的特性。在 Rust 中，如果没有显式的返回语句，函数会返回最后一个表达式。使用这种风格在社区中被认为是惯用的。我们的`max`函数在`if...else`块中使用了这个特性。为了使实现工作，泛型类型必须在它们之间定义一个排序关系，这样我们就可以比较它们。在 Rust 中，这是通过将可能的类型限制为实现了`PartialOrd`特性的类型来实现的。我们还需要对`Copy`特性施加约束，以便编译器可以在从函数返回之前对 self 参数进行复制。我们继续定义另一个包含两个元素的元组。我们以类似的方式在这里实现相同的特性。当我们实际上在`main`函数中使用这些特性时，它们按预期工作。

特性也可以用来扩展内置类型并添加新的功能。让我们看看以下示例：

```rs
// chapter2/extending-types.rs

// Trait for our behavior
trait Sawtooth {
    fn sawtooth(&self) -> Self;
}

// Extending the builtin f64 type
impl Sawtooth for f64 {
    fn sawtooth(&self) -> f64 {
        self - self.floor()
    }
}

fn main() {
    println!("{}", 2.34f64.sawtooth());
}
```

在这里，我们想要在内置的`f64`类型上实现锯齿波([`en.wikipedia.org/wiki/Sawtooth_wave`](https://en.wikipedia.org/wiki/Sawtooth_wave))函数。这个函数在标准库中不可用，因此我们需要编写一些代码来通过扩展标准库使其工作。为了无缝集成到类型系统中，我们需要定义一个特性和为`f64`类型实现它。这使我们能够使用新的函数，就像使用`f64`类型上的任何其他内置函数一样，通过点符号使用。

标准库提供了一些内置特性；其中最常见的是`Display`和`Debug`。这两个特性用于在打印时格式化类型。`Display`对应于空格式化器`{}`，而`Debug`对应于格式化调试输出。所有的数学运算都定义为特性，例如`Add`、`Div`等。如果用户定义的类型带有`#[derive]`属性，编译器会尝试提供默认实现。然而，如果需要，实现可以选择覆盖这些中的任何一个。以下代码片段展示了这种情况：

```rs
// chapter2/derive.rs

use std::fmt;
use std::fmt::Display;

#[derive(Debug, Hash)]
struct Point<T> {
    x: T,
    y: T,
}

impl<T> fmt::Display for Point<T> where T: Display {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

fn main() {
    let p: Point<u32> = Point { x: 4u32, y: 2u32 };

    // uses Display
    println!("{}", p);

    // uses Debug
    println!("{:?}", p);
}
```

我们定义了一个具有两个字段的泛型点结构。我们让编译器生成一些常见特质的实现。然而，我们必须手动实现 `Display` 特质，因为编译器无法确定显示用户定义类型的最优方式。我们必须使用 `where` 子句将泛型类型限制为实现了 `Display` 特质的类型。这个例子还展示了基于特质约束类型的另一种方法。在设置好所有这些之后，我们可以使用默认格式化程序显示我们的点。这会产生以下输出：

```rs
# rustc derive.rs && ./derive
(4, 2)
Point { x: 4, y: 2 }
```

# 错误处理

Rust 的一个主要目标就是让开发者能够编写健壮的软件。这其中的一个关键组成部分就是高级错误处理。在本节中，我们将更深入地探讨 Rust 是如何处理错误的。但在那之前，让我们先偏离一下，看看一些类型理论。具体来说，我们感兴趣的是**代数数据类型**（**ADT**），这是通过组合其他类型形成的类型。最常见的两种 ADT 是求和类型和乘积类型。Rust 中的 `struct` 是乘积类型的一个例子。这个名字来源于这样一个事实：给定一个结构体，其类型的范围本质上是其各个组成部分的范围的笛卡尔积，因为类型的实例具有其所有组成部分类型的值。相反，当 ADT 只能假设其组成部分之一的数据类型时，就是求和类型。Rust 中的 `enum` 就是这样一个例子。虽然 Rust 中的枚举与 C 语言和其他语言中的枚举类似，但 Rust 枚举提供了一些增强功能：它们允许变体携带数据。

现在，回到错误处理。Rust 强制要求可能产生错误的操作必须返回一个特殊的 `enum`，该 `enum` 携带结果。方便的是，这个 `enum` 看起来是这样的：

```rs
enum Result<T, E> {
    Ok(T),     
    Err(E), 
}
```

两种可能的选择被称为变体。在这种情况下，它们分别代表非错误情况和错误情况。请注意，这是通用的定义，因此实现可以自由地在两种情况下定义类型。这在需要扩展标准错误类型并实现自定义错误的程序中非常有用。让我们看看一个实际应用的例子：

```rs
// chapter2/custom-errors.rs

use std::fmt;
use std::error::Error;

#[derive(Debug)]
enum OperationsError {
    DivideByZeroError,
}

// Useful for displaying the error nicely
impl fmt::Display for OperationsError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match *self {
        OperationsError::DivideByZeroError => f.write_str("Cannot divide by zero"),
        }
    }
}

// Registers the custom error as an error
impl Error for OperationsError {
    fn description(&self) -> &str {
        match *self {
            OperationsError::DivideByZeroError => "Cannot divide by zero",
        }
    }
}

// Divides the dividend by the divisor and returns the result. Returns
// an error if the divisor is zero
fn divide(dividend: u32, divisor: u32) -> Result<u32, OperationsError> {
    if divisor == 0u32 {
        Err(OperationsError::DivideByZeroError)
    } else {
        Ok(dividend / divisor)
    }
}

fn main() {
    let result1 = divide(100, 0);
    println!("{:?}", result1);

    let result2 = divide(100, 2);
    println!("{:?}", result2.unwrap());
}
```

对于这个示例，我们定义了一个函数，该函数简单地返回第一个操作数除以第二个操作数时的商。这个函数必须处理除数为零的错误情况。我们还想让它在其调用者遇到这种情况时发出错误信号。此外，让我们假设这是一个将扩展以包含更多此类操作的库的一部分。为了使代码易于管理，我们为我们的库创建了一个错误类，其中包含一个表示除以零错误的元素。为了让 Rust 编译器知道枚举是一个错误类型，我们的枚举必须实现标准库中的`Error`特质。它还需要手动实现`Display`特质。在设置好这些样板代码之后，我们可以定义我们的除法方法。我们将利用泛型`Result`特质来注解，在成功的情况下，它应该返回一个`u32`类型的值，与操作数相同。在失败的情况下，它应该返回类型为`OperationsError`的错误。在函数中，如果我们的除数为零，我们将引发错误。否则，我们执行除法，将结果包装在`Ok`中，使其成为`Result`枚举的一个变体，并返回它。在我们的`main`函数中，我们用零除数调用这个函数。结果将是一个错误，如第一个打印宏所示。在第二次调用中，我们知道除数不是零。因此，我们可以安全地解包结果，将其从`Ok(50)`转换为`50`。标准库提供了一些实用函数来处理`Result`类型，安全地向调用者报告错误。

这里是上一个示例的运行样本：

```rs
$ rustc custom-errors.rs && ./custom-errors
Err(DivideByZeroError)
50
None variant. The Some variant handles the case where it holds the actual value of the type T:
```

```rs
pub enum Option<T> {
    None,
    Some(T),
}
```

给定这种类型，我们可以这样编写我们的除法函数：

```rs
// chapter2/options.rs

fn divide(dividend: u32, divisor: u32) -> Option<u32> {
    if divisor == 0u32 {
        None
    } else {
        Some(dividend / divisor)
    }
}

fn main() {
    let result1 = divide(100, 0);

    match result1 {
        None => println!("Error occurred"),
        Some(result) => println!("The result is {}", result),
    }

    let result2 = divide(100, 2);
    println!("{:?}", result2.unwrap());
}
```

我们修改我们的函数以返回一个类型为`u32`的`Option`。在我们的`main`函数中，我们调用我们的函数。在这种情况下，我们可以根据返回类型进行匹配。如果它恰好是`None`，我们知道函数没有成功。在这种情况下，我们可以打印一个错误。如果它返回`Some`，我们提取其底层值并打印它。第二次调用工作正常，因为我们知道它没有收到零除数。使用`Option`进行错误处理可能更容易管理，因为它涉及更少的样板代码。然而，在具有自定义错误类型的库中，这可能有点难以管理，因为错误不是由类型系统处理的。

注意，`Option`可以表示为给定类型和单位类型的`Result`。

`type Option<T> = Result<T, ()>;`。

我们之前描述的错误处理已经使用了可恢复的错误。然而，在某些情况下，如果发生错误，中止执行可能是明智的。标准库提供了`panic!`宏来处理这种情况。调用此宏将停止当前线程的执行，在屏幕上打印一条消息，并回滚调用栈。然而，需要谨慎使用，因为在很多情况下，更好的选择是正确处理错误并将错误向上传递给调用者。

许多内置方法和函数在出错时会调用这个宏。让我们看看以下示例：

```rs
// chapter2/panic.rs

fn parse_int(s: String) -> u64 {
    return s.parse::<u64>().expect("Could not parse as integer")
}

fn main() {
    // works fine
    let _ = parse_int("1".to_owned());

    // panics
    let _ = parse_int("abcd".to_owned());
}
```

这会引发以下错误：

```rs
# ./panic
thread 'main' panicked at 'Could not parse as integer: ParseIntError { kind: InvalidDigit }', src/libcore/result.rs:906:4
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

调用 panic 的一些方法是`expect()`和`unwrap()`。

# 宏系统

Rust 支持一个经过多年演变的宏系统。Rust 宏的一个显著特点是它们保证不会意外地引用其作用域之外的标识符，因此 Rust 中的宏实现是“卫生”的。正如预期的那样，Rust 宏在编译前被展开为源代码，并与翻译单元一起编译。编译器对展开的宏执行作用域规则以使它们卫生。Rust 宏与其他构造不同，因为它们总是以感叹号`!`结尾。

现代 Rust 有两种处理宏的方法；较老的，基于语法的宏方法，以及较新的，过程宏方法。让我们看看这些方法中的每一个：

# 语法宏

这个宏系统自 Rust 1.0 之前的版本以来一直是 Rust 的一部分。这些宏是通过一个名为`macro_rules!`的宏定义的。让我们看看一个示例：

```rs
// chapter2/syntactic-macro.rs

macro_rules! factorial {
    ($x:expr) => {
        {
            let mut result = 1;
            for i in 1..($x+1) {
                result = result * i;
            }
            result
        }
    };
}

fn main() {
    let arg = std::env::args().nth(1).expect("Please provide only one argument");
    println!("{:?}", factorial!(arg.parse::<u64>().expect("Could not parse to an integer")));
}
```

我们从定义阶乘宏开始。由于我们不希望编译器拒绝编译我们的代码，因为它可能会溢出宏栈，我们将使用非递归实现。Rust 中的语法宏是一组规则，其中左侧指定规则应该如何与输入匹配，右侧指定它应该展开成什么。规则通过`=>`运算符映射到右侧的表达式。规则局部变量使用`$`符号声明。匹配规则使用一种特殊的宏语言表达，它有一套自己的保留关键字。我们的声明表明我们希望接受任何有效的 Rust 表达式；在这种情况下，它应该评估为整数。我们将把它留给调用者来确保这是真的。然后我们从这个范围的最后一个整数开始循环到 1，同时累积结果。一旦完成，我们使用隐式返回语法返回结果。

我们的调用者是主函数，因为我们使用`std::env`模块从用户那里获取输入。我们获取列表中的第一个输入，如果没有输入则抛出错误。然后我们打印出宏的结果，并在传递之前尝试将输入解析为`u64`。我们还处理解析可能失败的情况。这按预期工作：

```rs
# rustc syntactic-macro.rs && ./syntactic-macro 5
120
```

Rust 还提供了一些调试宏的工具。有人可能会对查看展开的宏是什么样子感兴趣。`trace_macros!` 宏正是这样做的。为了使其工作，我们需要启用一个功能门，如下面的代码片段所示（由于在 Rust 中尚不稳定，此代码只能在 Rust nightly 版本中工作）：

```rs
#![feature(trace_macros)]
trace_macros!(true);
```

注意，展开还包括`println!`，因为它是在标准库中定义的宏。

同样的展开也可以使用以下命令来调用编译器进行检查：

`rustc -Z unstable-options --pretty expanded syntactic-macro.rs`。

# **过程宏**

虽然常规的语法宏在许多场景下很有用，但某些应用需要更高级的代码生成功能，这些功能最好使用编译器操作的抽象语法树（AST）来实现。因此，有必要扩展宏系统以包括这一功能。后来决定，旧的宏系统和这个新系统，即所谓的**过程宏**，将共存。随着时间的推移，这个新系统预计将取代语法宏系统。编译器支持从外部 crate 加载插件；这些插件可以在编译器生成 AST 后接收 AST。有 API 可以修改 AST 以添加所需的新代码。关于这个系统的详细讨论超出了本书的范围。

# Rust 中的函数式特性

Rust 受到了像 Haskell 和 OCaml 这样的函数式语言的影响。不出所料，Rust 在语言和标准库中都对函数式编程提供了丰富的支持。在本节中，我们将探讨其中的一些。

# 高阶函数

我们之前已经看到，Rust 函数定义了一个独立的范围，其中所有局部变量都存在。因此，除非它们被明确地作为参数传递，否则范围外的变量永远不会泄漏到其中。可能会有这种情况，这不是期望的行为；闭包提供了一个类似匿名函数的机制，它能够访问其定义范围内定义的所有资源。这使得编译器能够强制执行相同的借用检查规则，同时使代码的重用更容易。在 Rust 术语中，一个典型的闭包会借用其周围作用域的所有绑定。可以通过使用`move`关键字标记来强制闭包拥有这些绑定。让我们看看一些例子：

```rs
// chapter2/closure-borrow.rs

fn main() {
    // closure with two parameters
    let add = |a, b| a + b;
    assert_eq!(add(2, 3), 5);

    // common use cases are on iterators
    println!("{:?}", (1..10).filter(|x| x % 2 == 0).collect::<Vec<u32>>());

    // using a variable from enclosing scope
    let times = 2;
    println!("{:?}", (1..10).map(|x| x * times).collect::<Vec<i32>>());
}
```

第一个例子是一个简单的闭包，它接受两个数字并对其求和。第二个例子更为复杂；它展示了闭包在函数式编程中的实际应用。我们感兴趣的是过滤一个整数列表，只收集其中的偶数。因此，我们从 1 到 10 的范围开始，它返回内置类型`Range`的一个实例。由于该类型实现了`IntoIterator`特质，该类型表现得像一个迭代器。因此，我们可以通过传递一个只返回输入可以被 2 整除的闭包来过滤它。最后，我们将结果迭代器收集到一个`u32`类型的向量中，并打印出来。最后一个例子在结构上相似。它从闭包的包围作用域中借用变量 times，并使用它来映射到范围的项目。 

让我们看看在闭包中使用`move`关键字的一个例子：

```rs
// chapter2/closure-move.rs

fn main() {
    let mut times = 2;
    {
        // This is in a new scope
        let mut borrow = |x| times += x;
        borrow(5);
    }
    assert_eq!(times, 7);

    let mut own = move |x| times += x;
    own(5);
    assert_eq!(times, 7);

}
```

第一个闭包（`borrow`）与我们之前讨论的闭包之间的区别是，它会修改从封装作用域继承的变量。我们必须声明变量和闭包为`mut`。我们还需要将闭包放在不同的作用域中，这样编译器就不会在我们尝试断言其值时抱怨双重借用。如断言，名为`borrow`的闭包从其父作用域借用变量，这就是为什么它的原始值变为`7`。第二个名为`own`的闭包是一个移动闭包，因此它获得了变量`times`的一个副本。为了使这可行，变量必须实现`Copy`特质，这样编译器才能将其复制到闭包中，所有内置类型都这样做。由于闭包获取的变量和原始变量不是同一个，编译器不会抱怨双重借用。此外，变量的原始值不会改变。这类闭包在实现线程时非常重要，我们将在后面的章节中看到。标准库还支持使用多个内置特质在用户定义的函数或方法中接受和返回闭包，如下表所示：

| **特质名称** | **函数** |
| --- | --- |
| `std::ops::Fn` | 由不接受可变捕获变量的闭包实现。 |
| `std::ops::FnMut` | 由需要修改捕获变量的闭包实现。 |
| `std::ops::FnOnce` | 由所有闭包实现。表示闭包可以被调用正好一次。 |

# 迭代器

另一个重要的功能性方面是惰性迭代。给定一组类型，应该能够以任何给定的顺序遍历这些类型或其子集。在 Rust 中，一个常见的迭代器是一个范围，它有一个起始点和结束点。让我们看看它们是如何工作的：

```rs
// chapter2/range.rs

#![feature(inclusive_range_syntax)]

fn main() {
    let numbers = 1..5;
    for number in numbers {
        println!("{}", number);
    }
    println!("------------------");
    let inclusive = 1..=5;
    for number in inclusive {
        println!("{}", number);
    }
}
```

第一个范围是一个排他性范围，从第一个元素到倒数第二个元素。第二个范围是一个包含性范围，一直延伸到最后一个元素。请注意，包含性范围是一个实验性特性，未来可能会发生变化。

如预期的那样，Rust 确实提供了接口，允许用户定义的类型可以迭代。类型只需要实现`std::iterator::Iterator`特质。让我们看看一个例子。我们感兴趣的是生成给定整数的柯勒茨序列([`en.wikipedia.org/wiki/Collatz_conjecture`](https://en.wikipedia.org/wiki/Collatz_conjecture))，这是由以下递归关系给出的，给定一个整数：

+   如果它是偶数，就除以二

+   如果它是奇数，就乘以 3 然后加 1

根据猜想，这个序列将始终终止于 1。我们将假设这是真的，并定义我们的代码以尊重这一点：

```rs
// chapter2/collatz.rs

// This struct holds state while iterating
struct Collatz {
    current: u64,
    end: u64,
}

// Iterator implementation
impl Iterator for Collatz {
    type Item = u64;

    fn next(&mut self) -> Option<u64> {
        if self.current % 2 == 0 {
            self.current = self.current / 2;
        } else {
            self.current = 3 * self.current + 1;
        }

        if self.current == self.end {
            None
        } else {
            Some(self.current)
        }
    }
}

// Utility function to start iteration
fn collatz(start: u64) -> Collatz {
    Collatz { current: start, end: 1u64 }
}

fn main() {
    let input = 10;

    // First 2 items
    for n in collatz(input).take(2) {
        println!("{}", n);
    }

    // Dropping first 2 items
    for n in collatz(input).skip(2) {
        println!("{}", n);
    }
}
```

在我们的代码中，当前迭代的态由名为 `Collatz` 的结构体表示。我们在其上实现了 `Iterator` 协议。为此，我们需要实现 `next` 函数，该函数接收当前态并生成下一个态。当它达到结束态时，它必须返回一个 `None`，以便调用者知道迭代器已经耗尽。这由函数的可空返回值表示。鉴于递归关系，实现是直接的。在我们的主函数中，我们实例化初始态，我们可以使用常规的 `for` 循环进行迭代。`Iterator` 特质自动实现了许多有用的函数；`take` 函数从迭代器中取出给定数量的元素，而 `skip` 函数则跳过给定数量的元素。所有这些对于处理可迭代集合都非常重要。

以下是我们示例运行的结果：

```rs
$ rustc collatz.rs && ./collatz
5
16
8
4
2
```

# 并发原语

Rust 的一个承诺是能够实现 *无畏的并发*。很自然地，Rust 通过多种机制支持编写并发代码。在本章中，我们将讨论其中的一些。我们已经看到 Rust 编译器如何使用借用检查来确保程序在编译时的正确性。事实证明，这些原语在验证并发代码的正确性时也非常有用。现在，有多种方式在语言中实现线程。最简单的方式是为平台中创建的每个线程创建一个新的操作系统线程。这通常被称为 1:1 线程。另一方面，多个应用程序线程可以映射到一个操作系统线程。这被称为 N:1 线程。虽然这种方法资源消耗较少，因为我们最终得到的实际线程较少，但上下文切换的开销更高。一种中间方案称为 M:N 线程，其中多个应用程序线程映射到多个操作系统级别的线程。这种方法需要最大的保护措施，并且使用运行时实现，而 Rust 避免使用运行时。因此，Rust 使用 1:1 模型。在 Rust 中，一个线程对应一个操作系统线程，这与 Go 等语言不同。让我们先看看 Rust 是如何使编写多线程应用程序成为可能的：

```rs
// chapter2/threads.rs

use std::thread;

fn main() {
    for i in 1..10 {
        let handle = thread::spawn(move || {
            println!("Hello from thread number {}", i);
        });
        let _ = handle.join();
    }
}
```

我们首先导入线程库。在我们的主函数中，我们创建一个空向量，我们将使用它来存储我们创建的线程的引用，以便我们可以等待它们退出。线程实际上是通过 `thread::spawn` 创建的，我们必须传递一个闭包，该闭包将在每个线程中执行。由于我们必须从封装作用域（循环索引 `i`）借用变量，因此闭包本身必须是一个移动闭包。在退出闭包之前，我们调用当前线程句柄的 `join`，以便所有线程相互等待。这产生了以下输出：

```rs
# rustc threads.rs && ./threads
Hello from thread number 1
Hello from thread number 2
Hello from thread number 3
Hello from thread number 4
Hello from thread number 5
Hello from thread number 6
Hello from thread number 7
Hello from thread number 8
Hello from thread number 9
```

多线程应用程序的真正力量在于线程可以合作完成有意义的工作。为此，需要两个重要的事情。线程需要能够相互传递数据，并且应该有方法来协调线程的调度，以便它们不会相互干扰。对于第一个问题，Rust 通过通道提供了消息传递机制。让我们看看以下示例：

```rs
// chapter2/channels.rs

use std::thread;
use std::sync::mpsc;

fn main() {
    let rhs = vec![10, 20, 30, 40, 50, 60, 70];
    let lhs = vec![1, 2, 3, 4, 5, 6, 7];
    let (tx, rx) = mpsc::channel();

    assert_eq!(rhs.len(), lhs.len());
    for i in 1..rhs.len() {
        let rhs = rhs.clone();
        let lhs = lhs.clone();
        let tx = tx.clone();
        let handle = thread::spawn(move || {
            let s = format!("Thread {} added {} and {}, result {}", i,
            rhs[i], lhs[i], rhs[i] + lhs[i]);
            tx.clone().send(s).unwrap();
        });
        let _ = handle.join().unwrap();
    }

    drop(tx);
    for result in rx {
        println!("{}", result);
    }
}
```

这个例子与上一个例子非常相似。我们导入必要的模块以便能够处理通道。我们定义了两个向量，并将为两个向量中的每一对元素创建一个线程，以便我们可以将它们相加并返回结果。我们创建了通道，它返回发送端和接收端的句柄。作为一个安全检查，我们确保两个向量确实具有相同的长度。然后，我们继续创建我们的线程。由于我们在这里需要访问外部变量，线程需要接受一个类似于上一个例子的`move`闭包。此外，编译器将尝试使用`Copy`特质来复制这些变量到线程中。在这种情况下，这将会失败，因为向量类型没有实现`Copy`。我们需要显式地`clone`资源，这样它们就不需要被复制。我们运行计算，并将结果发送到管道的发送端。稍后，我们连接所有线程。在我们遍历接收端并打印结果之前，我们需要显式地丢弃对原始发送端句柄的引用，这样在我们开始接收之前，所有发送者都会被销毁（复制的发送者将在线程退出时自动销毁）。这会打印出以下内容，正如预期的那样：

```rs
# rustc channels.rs && ./channels
Thread 1 added 20 and 2, result 22
Thread 2 added 30 and 3, result 33
Thread 3 added 40 and 4, result 44
Thread 4 added 50 and 5, result 55
Thread 5 added 60 and 6, result 66
Thread 6 added 70 and 7, result 77
```

还要注意，mpsc 代表多个生产者单消费者。

在与多个线程一起工作时，另一个常见的习惯用法是在所有这些线程之间共享一个公共状态。然而，在许多情况下，这可能会成为一个问题。调用者需要仔细设置排除机制，以确保状态在无竞争的情况下共享。幸运的是，借用检查器可以帮助确保这一点更容易。Rust 有几个智能指针用于处理共享状态。库还提供了一个通用的互斥锁类型，可以在多线程工作时用作锁。但可能最重要的是`Send`和`Sync`特质。任何实现了`Send`特质的数据类型都可以在多个线程之间安全共享。`Sync`特质表示对给定数据的访问在多个线程中是安全的。这些特质有一些规则：

+   所有内置类型都实现了`Send`和`Sync`特质，除了任何`unsafe`类型，一些智能指针类型如`Rc<T>`和`UnsafeCell<T>`。

+   如果复合类型没有包含任何没有实现`Send`和`Sync`的类型，它将自动实现这两个特质。

`std::sync`包提供了许多类型和辅助工具，用于处理并行代码。

在上一段中，我们提到了 unsafe Rust。让我们稍微详细地看看这一点。Rust 编译器通过使用健壮的类型系统提供了一些关于安全编程的强保证。然而，在某些情况下，这些可能更多地成为负担。为了处理这些情况，语言提供了一种退出这些保证的方法。标记为 `unsafe` 关键字的代码块可以做 Rust 可以做的任何事情，以及以下：

+   解引用原始指针类型 (*mut T 或 `*const T`)

+   调用 unsafe 函数或方法

+   实现一个标记为 `unsafe` 的特质

+   修改静态变量

让我们看看一个使用 `unsafe` 代码块来解引用指针的示例：

```rs
// chapter2/unsafe.rs

fn main() {
    let num: u32 = 42;
    let p: *const u32 = &num;

    unsafe {
        assert_eq!(*p, num);
    }
}
```

这里，我们创建了一个变量及其指针；如果我们尝试在不使用 `unsafe` 块的情况下解引用指针，编译器将拒绝编译。在 `unsafe` 块内部，我们在解引用时获得了原始值。虽然与 unsafe 代码一起工作可能很危险，但它对于底层编程（如内核（RedoxOS）和嵌入式系统）非常有用。

# 测试

Rust 将测试视为一等构造；生态系统中的所有工具都支持测试。编译器提供了一个内置的配置属性，用于指定测试模块。还有一个测试属性，用于指定函数为测试。当 Cargo 从头生成项目时，它会设置这个样板代码。让我们看看一个示例项目；我们将称之为阶乘。它将导出一个宏，用于计算给定整数的阶乘。由于我们之前已经方便地编写了这样一个宏，所以我们在这里将重用那段代码。请注意，由于这个 crate 将用作库，它没有主函数：

```rs
# cargo new factorial --lib
# cargo test
   Compiling factorial v0.1.0 (file:///Users/Abhishek/Desktop/rust-book/src/ch2/factorial)
    Finished dev [unoptimized + debuginfo] target(s) in 1.6 secs
     Running target/debug/deps/factorial-364286f171614349

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests factorial

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

运行 `cargo test` 会运行 Cargo 为我们生成的存根测试。我们将复制阶乘宏的代码到 `lib.rs`，它看起来像这样：

```rs
// chapter2/factorial/src/lib.rs

#[allow(unused_macros)]
#[macro_export]
macro_rules! factorial {
    ($x:expr) => {
        {
            let mut result = 1;
            for i in 1..($x+1) {
                result = result * i;
            }
            result
        }
    };
}

#[cfg(test)]
mod tests {
    #[test]
    fn test_factorial() {
        assert_eq!(factorial!(5), 120);
    }
}
```

我们还添加了一个测试，以确保阶乘确实按预期工作。`#[macro_export]` 属性告诉编译器这个宏将在 crate 外部使用。编译器内置的 `assert_eq!` 宏检查两个参数是否确实相等。我们还需要放置 `#[allow(unused_macros)]` 属性，因为没有它，编译器会抱怨这个宏在非测试代码中没有使用。如果我们添加一个类似的测试：

```rs
    #[test]
    fn test_factorial_fail() {
        assert_eq!(factorial!(5), 121);
    }
```

这显然是错误的，并且如预期的那样失败，并给出了描述性的错误。编译器还支持一个名为 `#[should_panic]` 的属性，用于标记应该引发恐慌的测试。在这种情况下，只有当发生恐慌时，测试才会通过。另一种编写测试的方法是在文档中，这也将在 Cargo 调用中运行。

这是一个在文档中带有工作示例的重要工具，这些示例在代码库演变过程中保证能够正常工作。让我们继续添加一些关于阶乘宏的 doctest：

```rs
// chapter2/factorial/src/lib.rs

/// The factorial crate provides a macro to compute factorial of a given
/// integer
/// # Examples
///
/// ```

/// # #[macro_use] extern crate factorial;

/// # fn main() {

/// assert_eq!(factorial!(0), 1);

/// assert_eq!(factorial!(6), 720);

/// # }

/// ```rs
///

#[macro_export]
macro_rules! factorial {
    ($x:expr) => {
        {
            let mut result = 1;
            for i in 1..($x+1) {
                result = result * i;
            }
            result
        }
    };
}
```

宏的 doctests 与其他内容的 doctests 略有不同：

+   他们必须使用`#[macro_use]`属性来标记这里正在使用宏。请注意，依赖于导出宏的 crate 的外部 crate 也必须使用该属性。

+   他们必须定义主函数并在 doctests 中包含一个`extern crate`指令。对于其他所有内容，编译器会根据需要生成主函数。额外的`#`标记隐藏了这些内容，使其不会出现在生成的文档中。

通常，测试模块、doctests 和`#[test]`属性仅应用于单元测试。集成测试应放置在顶级测试目录中。

Rust 团队正在努力在测试系统中添加运行基准测试的支持。目前这仅限于 nightly 版本。

# 摘要

本章是对 Rust 语言及其生态系统的非常简短的介绍。鉴于在 Rust 方面的这一背景，让我们来看一个经常被问到的问题：公司是否应该采用 Rust？像工程中的许多事情一样，正确的答案取决于许多因素。采用 Rust 的一个主要原因是能够以尽可能小的足迹编写健壮的代码。因此，Rust 适合针对嵌入式设备的项目。这个领域传统上使用汇编、C 和 C++。Rust 可以提供相同的表现力保证，同时确保代码的正确性。Rust 也很好地用于从 Python 或 Ruby 卸载性能密集型计算。Rust 的主要痛点是学习曲线可能很陡峭。因此，试图采用 Rust 的团队可能会花很多时间与编译器斗争，试图运行代码。然而，这会随着时间的推移而缓解。幸运的是，编译器错误信息通常非常有帮助。2017 年，Rust 团队决定将人体工程学作为优先事项。这一推动使得新开发者的入职变得更加容易。对于大型 Rust 项目，编译时间可能比 C、C++或 Go 更长。这可能会成为某些团队的问题。有几种方法可以解决这个问题，其中之一是增量编译。因此，很难找到一个适合所有情况的解决方案。希望这篇简短的介绍能帮助决定是否在新项目中选择 Rust。

在下一章中，我们将通过研究 Rust 如何在网络中处理两个主机之间的 TCP 和 UDP 连接，来在此基础上构建我们在这里学到的内容。
