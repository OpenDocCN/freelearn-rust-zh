# Cargo 和 Crates

Rust 是一种相当年轻的语言，从头开始设计，旨在成为程序员的一个实用且有用的工具。这是一个非常好的情况：没有遗留应用程序需要关心，并且从其他语言中学到的许多经验教训都融入了 Rust——特别是在工具方面。

在过去，集成和管理第三方包一直是许多语言的问题，并且存在几种不同的方法：

+   **NPM**：Node 的包管理器，在 JavaScript 社区中非常受欢迎

+   **Maven**：基于 XML 格式的企业级 Java 包管理器

+   **NuGet**：.NET 的包管理器

+   **PyPI**：Python 包索引

这些方法各有不同的配置风格、命名指南、发布基础设施、功能、插件等。Rust 团队从所有这些方法中学习，并构建了自己的版本：`cargo`。本章将全部关于`cargo`的力量，以及如何和在哪里与大量的包（称为 crates）集成。无论你是在开发自己的小型库，还是在构建大型企业级系统，`cargo`都将是项目的一个核心部分。通过阅读本章，你可以期待以下内容：

+   了解更多关于`cargo`、其配置和插件的信息

+   了解更多关于不同类型的 crates

+   在`cargo`中完成的基准测试和测试集成

# Cargo

基本的 Rust 工具由三个程序组成：

+   `cargo`：Rust 包管理器

+   `rustc`：Rust 编译器

+   `rustup`：Rust 工具链管理器

大多数用户永远不会直接接触（甚至看到）`rustc`，而是通常使用`rustup`来安装它，然后让`cargo`来协调编译过程。

不带任何参数运行`cargo`会显示它提供的子命令：

```rs
$ cargo
Rust's package manager 

USAGE: 
 cargo [OPTIONS] [SUBCOMMAND] 

OPTIONS: 
 -V, --version           Print version info and exit 
 --list              List installed commands 
 --explain <CODE>    Run `rustc --explain CODE` 
 -v, --verbose           Use verbose output (-vv very verbose/build.rs output) 
 -q, --quiet             No output printed to stdout 
 --color <WHEN>      Coloring: auto, always, never 
 --frozen            Require Cargo.lock and cache are up to date 
 --locked            Require Cargo.lock is up to date 
 -Z <FLAG>...            Unstable (nightly-only) flags to Cargo, see 'cargo -Z help' for details 
 -h, --help              Prints help information 

Some common cargo commands are (see all commands with --list): 
 build       Compile the current project 
 check       Analyze the current project and report errors, but don't build object files 
 clean       Remove the target directory 
 doc         Build this project's and its dependencies' documentation 
 new         Create a new cargo project 
 init        Create a new cargo project in an existing directory 
 run         Build and execute src/main.rs 
 test        Run the tests 
 bench       Run the benchmarks 
 update      Update dependencies listed in Cargo.lock 
 search      Search registry for crates 
 publish     Package and upload this project to the registry 
 install     Install a Rust binary 
 uninstall   Uninstall a Rust binary 
```

这里有一些关于包管理器可以做什么的线索。除了为项目解决不同类型的依赖关系外，它还充当基准测试和单元/集成测试的测试运行器，并提供对`crates.io`（[`crates.io/`](https://crates.io/)）等注册表的访问。许多这些属性都可以在`.cargo/config`文件中进行配置，该文件使用 TOML（[`github.com/toml-lang/toml`](https://github.com/toml-lang/toml)）语法，可以在你的主目录、项目目录或两者之间的层次结构中。

可以配置的各个属性可以很容易地随时间演变，因此我们将关注一些核心部分。

**本地仓库**可以通过文件根部分的`paths`属性（一个路径数组）进行自定义，而`cargo new`命令的任何命令行参数都可以在文件的`[cargo-new]`部分找到。如果这些自定义仓库是远程的，可以在`[registry]`和`[http]`中配置`proxy`地址和端口、自定义证书颁发机构（`cainfo`）或高延迟（`timeout`）。

这些是针对具有私有仓库的企业系统或使用共享驱动作为缓存的 CI 构建的有用配置。然而，可以通过让用户在 `[target.$triple]` 部分提供一些配置来**自定义工具链**（例如，`[target.wasm32-unknown-unknown]` 用于自定义 Wasm 目标）。每个这些部分都包含以下属性：

+   针对所选三重组合的特定 `linker`

+   通过自定义 `ar` 的另一个归档器

+   用于运行程序及其相关测试的 `runner`

+   编译器的标志位于 `rustflags`

最后，**构建配置**是在 `[build]` 部分设置的，其中可以设置 `jobs` 的数量、二进制文件，如 `rustc` 或 `rustdoc`、`target` 三重组合、`rustflags` 或增量编译。要了解更多关于配置 `cargo` 和获取此配置的示例，请访问 [`doc.rust-lang.org/cargo/reference/config.html`](https://doc.rust-lang.org/cargo/reference/config.html)。

在接下来的章节中，我们将探讨 `cargo` 的核心：项目。

# 项目配置

为了识别 Rust 项目，`cargo` 需要其清单存在，其中大多数其他方面（元数据、源代码位置等）可以配置。一旦完成这些配置，构建项目将创建另一个文件：`Cargo.lock`。此文件包含项目的依赖项树，包括库版本和位置，以便加快未来的构建。这两个文件对于 Rust 项目都是必不可少的。

# 清单文件 – Cargo.toml

`Cargo.toml` 文件遵循——正如其名所示——TOML 结构。它是手写的，包含有关项目以及依赖项、指向其他资源的链接、构建配置文件、示例等元数据。其中大部分是可选的，并有合理的默认值。实际上，`cargo new` 命令生成清单的最小版本：

```rs
[package]
name = "ch2"
version = "0.1.0"
authors = ["Claus Matzinger"]
edition = "2018"

[dependencies]
```

还有更多章节和属性，我们将在下面介绍一些重要的内容。

# 软件包

此清单部分主要关于软件包的元数据，例如名称、版本和作者，但也包含指向默认为相应页面的文档的链接 ([`docs.rs/`](https://docs.rs/)). 虽然许多这些字段是为了支持 `crates.io` 并显示各种指标（类别、徽章、仓库、主页等），但某些字段无论是否发布在那里都应该填写，例如许可证（特别是开源项目）。

另一个有趣的章节是 `package.metadata` 中的元数据表，因为它被 `cargo` 忽略。这意味着项目可以在清单中存储自己的数据，用于项目或发布相关的属性——例如，用于在 Android 的 Google Play 商店发布，或生成 Linux 软件包的信息。

# 配置文件

当你运行`cargo build`、`cargo build --release`或`cargo test`时，`cargo`使用配置文件来确定每个阶段的单独设置。虽然这些有合理的默认值，但你可能想要自定义一些设置。清单提供了这些开关，包括`[profile.dev]`、`[profile.release]`、`[profile.test]`和`[profile.bench]`部分：

```rs
[profile.release] 
opt-level = 3 
debug = false 
rpath = false 
lto = false 
debug-assertions = false 
codegen-units = 16 
panic = 'unwind' 
incremental = false 
overflow-checks = false
```

这些值是默认值（截至撰写本书时），对大多数用户来说已经很有用了。

# 依赖项

这可能是对大多数开发者来说最重要的部分。依赖项部分包含一个值列表，代表`crates.io`（或你配置的私有注册表）上的 crate 名称，作为键，版本作为值。

与版本字符串一样，也可以提供内联表作为值，指定可选性或其他字段：

```rs
[dependencies]
hyper = "*"
rand = { version = "0.5", optional = true } 
```

有趣的是，由于这是一个对象，TOML 允许我们像使用部分一样使用它：

```rs
[dependencies]
hyper = "*"

[dependencies.rand]
version = "0.5"
features = ["stdweb"]  
```

由于在 2018 版中，`.rs`文件内的`extern` crate 声明是可选的，因此可以通过使用`package`属性在`Cargo.toml`规范内部重命名依赖项。然后，指定的键可以成为此`package`的别名，如下所示：

```rs
[dependencies]
# import in Rust with "use web::*"
web = { version = "*", package = "hyper" }

[dependencies.random] # import in Rust with "use random::*" 
version = "0.5"
package = "rand"
features = ["stdweb"] 
```

功能是特定于 crate 的字符串，包括或排除某些功能。在 rand（以及一些其他情况）中，`stdweb`是一个功能，允许我们在 Wasm 场景中使用 crate，通过排除那些否则无法编译的内容。请注意，这些功能可能在它们依赖于工具链时自动应用。

需要通过这些对象指定的是对远程 Git 仓库或本地路径的依赖。这对于在本地测试库的补丁版本非常有用，而无需将其发布到`crates.io` ([`crates.io/`](https://crates.io/))，并在父构建阶段由`cargo`构建：

```rs
[dependencies] 
hyper = "*"
rand = { git = "https://github.com/rust-lang-nursery/rand", branch = "0.4" }
```

使用`cargo`指定版本也遵循一个模式。由于任何 crate 都鼓励遵循语义版本控制方案（`<major>.<minor>.<patch>`），因此存在包括或排除某些版本（以及因此 API）的运算符。对于`cargo`，这些运算符如下：

+   **波浪号** (`~`): 只允许补丁增加。

+   **插入符** (`^`): 不会进行主要更新（从 2.0.1 到 2.1.0 是可以的，到 3.0.1 则不行！）。

+   **通配符** (`*`): 允许任何版本，但也可以用来替换一个位置。

这些运算符避免了未来的依赖问题，并引入了一个稳定的 API，同时不会错过所需的更新和修复。

无法发布带有通配符依赖的 crate。毕竟，目标计算机应该使用哪个版本？这就是为什么`cargo`在运行`cargo publish`时强制执行显式版本号。

有几种方式可以处理特定目的的依赖。它们可以通过平台（`[target.wasm32-unknown-unknown]`）声明，或者通过它们的意图：存在一个依赖类型，`[dev-dependencies]`，用于编译测试、示例和基准测试，但还有一个仅用于构建的依赖规范，`[build-dependencies]`，它将与其他依赖分开。

一旦指定了依赖，它们就会被解析并查找，以在项目内部生成依赖树。这就是`Cargo.lock`文件发挥作用的地方。

# 依赖 – Cargo.lock

这里有一句来自`cargo`常见问题解答（[`doc.rust-lang.org/cargo/faq.html`](https://doc.rust-lang.org/cargo/faq.html)）的精彩引言，关于此文件的目的以及它所做的工作：

Cargo.lock 的目的在于描述成功构建时的世界状态。然后，通过确保正在编译的依赖完全相同，它被用来确保在构建项目的任何机器上都能提供确定性的构建。

这种序列化状态可以轻松地在团队或计算机之间传输。因此，如果依赖引入了补丁更新中的错误，除非运行`cargo update`，否则你的构建应该不会受到太大影响。实际上，建议将`Cargo.lock`文件提交到版本控制中，以保留一个稳定且可工作的构建。出于调试目的，简化依赖树也非常方便。

# 命令

`cargo`支持大量易于扩展的命令。它与项目深度集成，允许添加额外的构建脚本、基准测试、测试等。

# 编译和运行命令

作为主要的构建工具，`cargo`通过创建并执行输出二进制文件（通常位于`target/<profile>/<target-triple>/`）来进行编译和运行。

如果需要使用不同语言编写的库来先于 Rust 构建，那该怎么办？这正是构建脚本发挥作用的地方。如*项目配置*部分所述，清单提供了一个名为`build`的字段，它接受一个指向`build`脚本的路径或名称。

脚本本身可以是一个普通的 Rust 二进制文件，在指定的文件夹中生成输出，甚至可以在`Cargo.toml`中指定依赖（`[build-dependencies]`，但不能是其他任何内容）。任何所需的信息（目标架构、输出等）都通过环境变量传递给程序，并且任何针对`cargo`的输出都需要遵循`cargo:key=value`格式。这些信息会被`cargo`捕获以配置后续步骤。虽然构建本地依赖是最受欢迎的，但也可以生成代码（如绑定、数据访问类等）。更多详情请参阅`cargo`参考文档：[`doc.rust-lang.org/cargo/reference/build-scripts.html`](https://doc.rust-lang.org/cargo/reference/build-scripts.html)。

较大的项目需要比简单的`src/`文件夹更复杂的结构来包含所有源代码，这就是为什么`cargo`提供了将项目拆分为子项目（称为工作空间）的选项。这对于像微服务（每个服务可以是项目）或松耦合组件（清洁架构）这样的架构模式很有用。要设置此环境，将每个子项目放置在子目录中，并在工作空间中创建一个`Cargo.toml`文件来声明其成员：

```rs
[workspace] 
members = [ "core", "web", "data"]
```

这将应用在顶层运行的任何命令到工作空间中的每个 crate。调用`cargo test`将运行所有类型的测试，这可能会花费很长时间。

# 测试

在命令方面，`cargo`支持`test`和`bench`来运行 crate 的测试。这些测试通过在模块内部创建一个“模块”并用`#[cfg(test)]`进行注释来指定。此外，每个测试还必须用`#[test]`或`#[bench]`进行注释，而后者需要传递一个参数给`Bencher`，这是一个基准运行器类，允许我们收集每次运行的统计数据：

```rs
#![feature(test)] 
extern crate test;

pub fn my_add(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)] 
mod tests { 
    use super::*;
    use test::Bencher;

    #[test] 
    fn this_works() { 
        assert_eq!(my_add(1, 1), 2);
    }

    #[test]
    #[should_panic(expected = "attempt to add with overflow")]
    fn this_does_not_work() {
        assert_eq!(my_add(std::i32::MAX, std::i32::MAX), 0);
    }

    #[bench]
    fn how_fast(b: &mut Bencher) {
        b.iter(|| my_add(42, 42))
    }
}
```

运行`cargo test`后，输出符合预期：

```rs
Finished dev [unoptimized + debuginfo] target(s) in 0.02s
 Running target/debug/deps/ch2-6372277a4cd95206

running 3 tests
test tests::how_fast ... ok
test tests::this_works ... ok
test tests::this_does_not_work ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

在这个例子中，测试正在导入并调用其父模块中的一个函数，称为`my_add`。其中一个测试甚至期望抛出一个`panic!`（由溢出引起），这就是为什么添加了`#[should_panic]`注释的原因。

此外，`cargo`支持 doctests，这是一种特殊的测试形式。重构时最繁琐的事情之一是更新文档中的示例，这就是为什么它们经常不起作用。从 Python 迁移过来，doctest 是解决这个困境的方案。通过在指定的示例中运行实际代码，doctests 确保文档中打印出的所有内容都可以执行——同时创建一个黑盒测试。 

每个 Rust 函数都可以使用特殊的文档字符串进行注释——该字符串用于在 DOCS.RS（[`docs.rs/`](https://docs.rs/)）生成文档。

doctests 仅适用于库类型的 crate。

这份文档有章节（由 Markdown 标题`#`指示），如果某个特定章节被命名为`Examples`，则其中包含的任何代码都将被编译并运行：

```rs
/// # A new Section
/// this [markdown](https://daringfireball.net/projects/markdown/) is picked up by `Rustdoc` 
```

我们现在可以通过创建几行文档来向先前的示例添加另一个测试：

```rs
/// # A Simple Addition
/// 
/// Adds two integers.
/// 
/// # Arguments
/// 
/// - *a* the first term, needs to be i32
/// - *b* the second term, also a i32
/// 
/// ## Returns
/// The sum of *a* and *b*.
/// 
/// # Panics
/// The addition is not done safely, overflows will panic!
/// 
/// # Examples
/// 
/// ```rust

/// assert_eq!(ch2::my_add(1, 1), 2);

/// ```rs
pub fn my_add(a: i32, b: i32) -> i32 {
    a + b
} 
```

`cargo test`命令现在将运行示例中的代码：

```rs
$ cargo test 
 Compiling ch2 v0.1.0 (file:///home/cm/workspace/Mine/rust.algorithms.data.structures/code/ch2) 
 Finished dev [unoptimized + debuginfo] target(s) in 0.58s 
 Running target/debug/deps/ch1-8ed0f81f04655fe4 

running 0 tests 

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out 

 Running target/debug/deps/ch2-3ddb7f7cbab6792d 

running 3 tests 
test tests::how_fast ... ok 
test tests::this_does_not_work ... ok 
test tests::this_works ... ok 

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out 

 Doc-tests ch2 

running 1 test 
test src/lib.rs - my_add (line 26) ... ok 

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

对于较大的测试或黑盒测试，也可以（并且推荐）将测试放入项目的子文件夹中，称为`tests`。`cargo`会自动检测并相应地运行测试。

除了测试之外，通常还需要其他命令（如代码度量、linting 等）和推荐。为此，`cargo`提供了一个第三方命令接口。

# 第三方子命令

`cargo` 允许通过子命令扩展其命令行界面。这些子命令是当调用 `cargo <command>`（例如，`cargo clippy` 用于流行的代码检查器）时被调用的二进制文件。

为了安装一个新的命令（针对特定的工具链），运行 `cargo +nightly install clippy`，这将下载、编译并安装一个名为 `cargo-clippy` 的 crate，并将其放入您家目录中的 `.cargo` 目录。实际上，这适用于任何被称作 `cargo-<something>` 且可以从任何命令行执行的二进制文件。`cargo` 项目在 [`github.com/rust-lang/cargo/wiki/Third-party-cargo-subcommands`](https://github.com/rust-lang/cargo/wiki/Third-party-cargo-subcommands) 的仓库中维护了一个有用的子命令更新列表。

# Crates

一旦完成所有编译和测试，Rust 的模块（crates）可以轻松打包和分发，无论它们是库还是可执行文件。首先，让我们看看 Rust 二进制文件的一般情况。

# Rust 库和二进制文件

Rust 中有可执行二进制文件和库。当这些 Rust 程序使用依赖项时，它们依赖于链接器来集成这些依赖项，以便在至少当前平台上运行。链接主要有两种类型：静态链接和动态链接——这两种类型都多少依赖于操作系统。

# 静态和动态库

通常，Rust 依赖项有两种类型的链接：

+   **静态链接**：通过 `rlib` 格式。

+   **动态链接**：通过共享库（`.so` 或 `.dll`）。

如果可以找到相应的 `rlib`，首选是静态链接，因此将所有依赖项包含到输出二进制文件中，使得文件变得很大（这让嵌入式程序员感到沮丧）。因此，如果多个 Rust 程序使用相同的依赖项，每个程序都会带有其内置的版本。尽管如此，这完全取决于上下文，因为，正如 Go 的成功所展示的，静态链接可以简化复杂的部署，因为只需要分发单个文件。

除了大小之外，静态链接方法还有其他缺点：对于静态库，所有依赖项都必须是 `rlib` 类型，这是 Rust 的原生包格式，并且不能包含动态库，因为格式（例如，ELF 系统上的 `.so`（动态）和 `.a`（静态））是不可转换的。

对于 Rust，动态链接通常用于本地依赖项，因为它们通常在操作系统中可用，不需要包含在包中。Rust 编译器可以通过 `-C prefer-dynamic` 标志优先使用动态链接，这将使编译器首先查找相应的动态库。

这就是编译器的当前策略：根据输出格式（`--crate-format= rlib`、`dylib`、`staticlib`、`library` 或 `bin`），它决定最佳的链接类型，并通过标志影响你的选择。然而，有一条规则是输出不能静态链接相同的库两次，因此它不会链接具有相同静态依赖的两个库。

关于这个主题的更多信息，我们建议查看[`doc.rust-lang.org/reference/linkage.html`](https://doc.rust-lang.org/reference/linkage.html)。话虽如此，编译器通常是可信的，除非有互操作性目标，否则 `rustc` 将决定最优方案。

# 链接和互操作性

Rust 编译成原生代码，就像许多其他语言一样，这很好，因为它扩展了可用的库，并允许你选择最佳技术来解决一个问题。“与其他人友好相处”一直是 Rust 的主要设计目标。

在那个层面上，互操作性就像声明你想要导入的函数，并动态链接导出此函数的库一样简单。这个过程在很大程度上是自动化的：唯一需要的是创建和声明一个构建脚本，该脚本编译依赖项，然后告诉链接器输出所在的位置。根据你构建的库类型，链接器执行必要的操作将其包含到 Rust 程序中：静态链接或动态链接（默认）。

如果只有一个要动态链接的原生库，清单文件提供了一个 `links` 属性来指定这一点。通过使用外部函数接口，程序化地与这些包含的库交互非常简单。

# FFI

**外部函数接口**（**FFI**）是 Rust 通过简单的关键字 `extern` 调用其他原生库（反之亦然）的方式。通过声明一个 `extern` 函数，编译器知道，要么需要通过链接器（导入）绑定外部接口，要么声明的函数将被导出，以便其他语言可以使用它（导出）。

除了关键字之外，编译器和链接器还需要了解预期的二进制布局类型。这就是为什么通常的 `extern` 声明看起来如下：

```rs
extern "C" {
    fn imported_function() -> i32;
}

#[no_mangle]
pub extern "C" fn exported_function() -> i32 {
    42
}
```

这允许从 Rust 中调用 `C` 库函数。然而，有一个注意事项：调用部分必须包裹在 `unsafe` 部分。Rust 编译器无法保证外部库的安全性，因此对它的内存管理持悲观态度是有道理的。导出的函数是安全的，通过添加 `#[no_mangle]` 属性，没有名称混淆，因此可以通过其名称找到它。

为了使用 C/C++库中可用的专用算法库，有一个名为`rust-bindgen`的工具，它可以生成合适的结构、`extern "C"`声明和数据类型。更多详情请访问[`github.com/rust-lang-nursery/rust-bindgen`](https://github.com/rust-lang-nursery/rust-bindgen)。这些互操作性能力使得 Rust 代码可用于遗留软件或用于非常不同的环境中，例如网页前端。

# Wasm

**Wasm**，现在通常称为**WebAssembly**，是一种二进制格式，旨在补充 JavaScript，Rust 可以编译成这种格式。该格式旨在在多个沙盒执行环境中（如网页浏览器或 Node.js 运行时）作为堆栈机器运行，用于性能关键的应用程序([`blog.x5ff.xyz/blog/azure-functions-wasm-rust/`](https://blog.x5ff.xyz/blog/azure-functions-wasm-rust/))。尽管目前仍处于早期阶段，但 Rust 和 Wasm 目标已被用于实时前端设置（如浏览器游戏），并且在 2018 年有一个专门的工作组寻求改进这种集成。

与其他目标，例如 ARM 类似，Wasm 目标是一个 LLVM（Rust 构建所依赖的编译器技术）后端，因此必须使用`rustup target add wasm32-unknown-unknown`进行安装。此外，没有必要声明二进制布局（`extern "C"`中的`"C"`）和不同的 bindgen 工具来完成剩余工作：`wasm-bindgen`，可在[`github.com/rustwasm/wasm-bindgen`](https://github.com/rustwasm/wasm-bindgen)找到。我们强烈建议阅读文档以获取更多信息。

# 主要仓库 - crates.io

`crates.io`网站([`crates.io/`](https://crates.io/))提供了一个巨大的 crates 仓库，可用于 Rust。除了可发现性功能，如`tags`和`search`，它还允许 Rust 程序员将他们的工作提供给他人。

仓库本身提供了与仓库交互的 API 以及大量关于`cargo`、crates 等文档的指针。源代码可在 GitHub 上找到——我们建议查看仓库以获取更多信息：[`github.com/rust-lang/crates.io`](https://github.com/rust-lang/crates.io)。

# 发布

为了让开发者的 crate 进入这个仓库，`cargo`提供了一个命令：`cargo publish`。实际上，这个命令在幕后做了更多的事情：首先运行`cargo`包来创建一个包含上传内容的`*.crate`文件。然后通过基本上运行`cargo test`来验证包的内容，并检查本地仓库中是否有未提交的文件。只有当这些检查通过时，`cargo`才会将`*.crate`文件的内容上传到仓库。这需要一个有效的`crates.io`账户（可以通过 GitHub 登录获得）来获取个人秘密 API 令牌，并且 crate 必须遵循某些规则。

在之前提到的 Wasm 目标下，甚至可以将 Rust 包发布到著名的 JavaScript 包仓库：`npm`。请注意，Wasm 的支持仍然非常新，但一旦库编译成 Wasm，就可以使用 Wasm-pack 打包成一个 `npm` 包：[`github.com/rustwasm/wasm-pack`](https://github.com/rustwasm/wasm-pack)。

`crates.io` 致力于成为 Rust 库的永久存储库，因此没有“取消发布”按钮。版本可以通过 `cargo yank` 被撤回，但这不会删除任何代码；它只会禁止更新这个特定版本。此外，你还可以在仓库网站上设置团队结构、多彩的 README、徽章等等，我们强烈建议你也查看相关的文档：[`doc.rust-lang.org/cargo/reference/publishing.html`](https://doc.rust-lang.org/cargo/reference/publishing.html)。

# 摘要

`cargo` 是 Rust 的包管理器和构建工具，可以通过名为 `Cargo.toml` 的清单进行配置。该文件由 `cargo` 使用，以构建具有指定依赖项、配置文件、工作空间和包元数据的所需二进制文件。在这个过程中，包状态被保存在一个名为 `Cargo.lock` 的文件中。得益于其 LLVM 前端，Rust 能够在各种平台上编译成原生代码，包括网络（使用 Wasm），从而保持高度的互操作性。成功构建的包可以发布到一个名为 `crates.io` 的仓库，这是一个 Rust 库和二进制文件的中央枢纽。

在我们深入数据结构（从列表开始）之前，下一章将介绍 Rust 在内存中存储变量和数据的方式，是复制还是克隆，以及什么是有大小和无大小的类型。

# 问题

+   `cargo` 都做了些什么？

+   `cargo` 提供了代码风格检查支持吗？

+   在哪些情况下，`Cargo.lock` 文件对于发布很重要？

+   发布到 `crates.io` 需要满足哪些要求？

+   Wasm 是什么，为什么你应该关心？

+   Rust 项目中的测试是如何组织的？

# 进一步阅读

你可以参考以下链接，了解更多关于本章涵盖主题的信息：

+   [`crates.io`](https://crates.io)

+   [`doc.rust-lang.org/cargo/`](https://doc.rust-lang.org/cargo/)

+   [`github.com/rustwasm/team`](https://github.com/rustwasm/team)

+   [`webassembly.org`](https://webassembly.org)

+   [`blog.x5ff.xyz/blog/azure-functions-wasm-rust/`](https://blog.x5ff.xyz/blog/azure-functions-wasm-rust/)
