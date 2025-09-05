# 前言

Mozilla 的 Rust 正在逐渐获得关注，它拥有惊人的特性和强大的库。本书将带你通过各种食谱，教你如何利用标准库来实现高效的解决方案。

本书首先简要介绍了标准库和集合的基本模块。从那里开始，食谱将涵盖支持文件/目录处理和（反）序列化的常见数据格式的 crate。你将了解与高级数据结构、错误处理和网络相关的 crate。你还将学习如何使用 futures 和实验性的夜间功能。本书还涵盖了 Rust 中最相关的外部 crate。你将能够使用库的标准模块来编写自己的算法。

到本书结束时，你将能够熟练地使用*《Rust 标准库食谱》*。

# 本书面向的对象

这本书是为那些想要探索 Rust 的力量并学习如何使用标准库来实现各种功能的开发者而写的。假设你具备基本的 Rust 编程知识和终端技能。

# 本书涵盖的内容

第一章，*学习基础知识*，构建了一个强大的基础，包括在所有各种情况下都很有用的基本原理和技术。它还将向你展示如何在命令行上接受用户输入，这样你就可以以交互式的方式编写所有后续章节，如果你愿意的话。

第二章，*使用集合*，为你提供了 Rust 中所有主要数据集合的全面概述，其中包括了如何在你的 RAM 中存储数据，以便你准备好为正确的任务选择正确的集合。

第三章，*处理文件和文件系统*，展示了如何将你的现有工具与文件系统连接起来，让你能够在计算机上存储、读取和搜索文件。我们还将学习如何压缩数据，以便高效地通过互联网发送。

第四章，*序列化*，介绍了当今最常见的数据格式以及如何在 Rust 中（反）序列化它们，使你能够通过跨工具通信将你的程序连接到许多服务。

第五章，*高级数据结构*，通过提供有关通用模块和 Rust 智能指针的有用信息，为后续章节构建了基础。你还将学习如何通过使用自定义#[derive()]语句在编译时为注解结构生成代码。

第六章，*处理错误*，向您介绍 Rust 的错误处理概念，以及如何通过为您的用例创建自定义错误和记录器来无缝与之交互。我们还将探讨如何设计结构，以便在用户未察觉的情况下清理其资源。

第七章，*并行和 Rayon*，带您进入多线程的世界，并证明 Rust 确实为您提供了*无畏的并发性*。您将学习如何轻松地调整您的算法，以充分利用您 CPU 的全部功能。

第八章，*使用未来*，向您介绍程序中的异步概念，并为您准备所有与 *futures*（Rust 中在程序后台运行的任务的版本）一起工作的库。

第九章，*网络*，通过教授您如何设置低级服务器、响应不同协议中的请求以及与世界万维网进行通信，帮助您连接到互联网。

第十章，*使用实验性夜间功能*，确保您在 Rust 知识上保持领先。它将通过向您展示最期待的功能（这些功能仍被视为不稳定）来展示编程的明天之路。

# 要充分利用本书

本书是为 rustc 1.24.1 和 rustc 1.26.0-nightly 版本的 Rust 编写并测试的；然而，Rust 强大的向后兼容性应使您能够使用除最后一章之外的所有章节的新版本。第十章*使用实验性夜间功能*正在与前沿技术合作，预计将通过突破性的变化得到改进。

要下载最新的 Rust 版本，请访问 [`rustup.rs/`](https://rustup.rs/)，在那里您可以下载适用于您操作系统的 Rust 安装程序。将其保留在标准设置上是可以的。在开始第十章，*使用实验性夜间功能*之前，请确保调用 rustup default nightly。不用担心，当时候您还会再次被提醒。

许多配方需要活跃的互联网连接，因为我们将与 crates 进行密集型工作。这些是 Rust 在互联网上分发库的方式，它们托管在 [`crates.io/`](https://crates.io/)。

您可能会想知道，一本关于 Rust 标准库（简称 std）的书为什么使用了这么多来自 std 之外的语言。那是因为与大多数其他系统语言不同，Rust 从一开始就被设计为具有强大的依赖管理。将 crate 拉入代码中非常容易，因此很多特定功能已经外包给了官方推荐的 crate。这有助于与 Rust 一起分发的核心标准库保持简单和非常稳定。

在 std 之后，最官方的 crate 组是*nursery* ([`github.com/rust-lang-nursery?language=rust`](https://github.com/rust-lang-nursery?language=rust))。这些 crate 是许多操作的标准，它们几乎足够稳定或通用，可以包含在 std 中。

如果我们在 nursery 中找不到某个菜谱的 crate，我们会查看 Rust 核心团队成员的 crate ([`github.com/orgs/rust-lang/people`](https://github.com/orgs/rust-lang/people))，他们投入了大量精力提供标准库中缺失的功能。这些 crate 不在 nursery 中，因为它们通常足够具体，不值得投入太多资源来积极维护它们。

本书中的所有代码都已使用最新的 rustfmt（rustfmt-nightly v0.4.0）格式化，您可以选择使用 rustup component add rustfmt-preview 进行下载，并使用 cargo fmt 运行。GitHub ([`github.com/jnferner/rust-standard-library-cookbook`](https://github.com/jnferner/rust-standard-library-cookbook))上的代码将积极维护，并使用 rustfmt 的新版本进行格式化，如果可用。在某些情况下，这意味着源代码行标记可能会过时。然而，由于这种变化通常不超过两到三行，因此应该不难找到代码。

所有代码都已通过 Rust 官方的代码检查工具 clippy ([`github.com/rust-lang-nursery/rust-clippy`](https://github.com/rust-lang-nursery/rust-clippy))进行过检查，使用版本 0.0.187。如果您愿意，可以使用 cargo +nightly install clippy 进行安装，并使用 cargo +nightly clippy 运行它。不过，最新版本经常会出现问题，所以如果它不能直接运行，请不要感到惊讶。

我们故意在代码中留下了某些 clippy 和 rustc 警告。其中大部分要么是死代码，这种情况发生在我们为了说明一个概念而给一个变量赋值，然后不再需要使用这个变量时；要么是使用占位符名称，如 foo、bar 或 baz，当变量的确切目的与配方无关时使用。

# 下载示例代码文件

您可以从[www.packtpub.com](http://www.packtpub.com)的账户下载本书的示例代码文件。如果您在其他地方购买了这本书，您可以访问[www.packtpub.com/support](http://www.packtpub.com/support)并注册，以便将文件直接通过电子邮件发送给您。

您可以通过以下步骤下载代码文件：

1.  在[www.packtpub.com](http://www.packtpub.com/support)登录或注册。

1.  选择 SUPPORT 标签。

1.  点击代码下载与勘误表。

1.  在搜索框中输入书籍名称，并遵循屏幕上的说明。

1.  下载文件后，请确保您使用最新版本解压缩或提取文件夹：

+   Windows 版的 WinRAR/7-Zip

+   Mac 版的 Zipeg/iZip/UnRarX

+   Linux 版的 7-Zip/PeaZip

书籍的代码包也托管在 GitHub 上，网址为[`github.com/PacktPublishing/Rust-Standard-Library-Cookbook/`](https://github.com/PacktPublishing/Rust-Standard-Library-Cookbook/)。如果代码有更新，它将在现有的 GitHub 仓库中更新。

我们还有其他来自我们丰富的书籍和视频目录的代码包，可在[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)找到。查看它们吧！

# 使用的约定

本书使用了多种文本约定。

`CodeInText`：表示文本中的代码单词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 昵称。以下是一个示例：“在`bin`文件夹中，创建一个名为`dynamic_json.rs`的文件。”

代码块设置如下：

```rs
let s = "Hello".to_string();
println!("s: {}", s);
let s = String::from("Hello");
println!("s: {}", s);
```

当我们希望您注意代码块中的特定部分时，相关的行或项目将以粗体显示：

```rs
let alphabet: Vec<_> = (b'A' .. b'z' + 1) // Start as u8
.map(|c| c as char)            // Convert all to chars
.filter(|c| c.is_alphabetic()) // Filter only alphabetic chars
.collect(); // Collect as Vec<char>
```

任何命令行输入或输出应如下编写：

```rs
name abraham
age 49
fav_colour red
hello world
(press 'Ctrl Z' on Windows or 'Ctrl D' on Unix)
```

**粗体**：表示新术语、重要单词或您在屏幕上看到的单词。例如，菜单或对话框中的单词在文本中显示如下。

警告或重要注意事项如下所示。

小贴士和技巧如下所示。

# 部分

在本书中，您将找到一些频繁出现的标题（*准备工作*、*如何做…*、*它是如何工作的…*、*更多内容…*和*参见*）。

为了清楚地说明如何完成食谱，请按照以下方式使用这些部分：

# 准备工作

本节将告诉您在食谱中可以期待什么，并描述如何设置任何软件或食谱所需的任何初步设置。

# 如何做…

本节包含遵循食谱所需的步骤。

# 它是如何工作的…

本节通常包含对上一节发生情况的详细解释。

# 更多内容…

本节包含有关食谱的附加信息，以便您对食谱有更多的了解。

# 参见

本节提供了对食谱其他有用信息的链接。

# 联系我们

我们读者的反馈总是受欢迎的。

**一般反馈**：请发送电子邮件至`feedback@packtpub.com`，并在邮件主题中提及书籍标题。如果您对本书的任何方面有疑问，请通过`questions@packtpub.com`给我们发送电子邮件。

**勘误表**：尽管我们已经尽一切努力确保内容的准确性，但错误仍然可能发生。如果您在此书中发现错误，我们将不胜感激，如果您能向我们报告此错误。请访问 [www.packtpub.com/submit-errata](http://www.packtpub.com/submit-errata)，选择您的书籍，点击勘误表提交表单链接，并输入详细信息。

**盗版**：如果您在互联网上遇到任何形式的我们作品的非法副本，我们将不胜感激，如果您能向我们提供位置地址或网站名称。请通过 `copyright@packtpub.com` 联系我们，并提供材料的链接。

**如果您有兴趣成为作者**：如果您在某个领域有专业知识，并且您有兴趣撰写或为书籍做出贡献，请访问 [authors.packtpub.com](http://authors.packtpub.com/)。

# 评论

请留下评论。一旦您阅读并使用过这本书，为何不在您购买书籍的网站上留下评论？潜在读者可以查看并使用您的客观意见来做出购买决定，Packt 可以了解您对我们产品的看法，我们的作者也可以看到他们对书籍的反馈。谢谢！

如需了解更多关于 Packt 的信息，请访问 [packtpub.com](https://www.packtpub.com/)。

# 免责声明

在明确声明的情况下，本书包含 Rust 编程语言（[https://doc.rust-lang.org/stable/book/](https://doc.rust-lang.org/stable/book/）），第一版和第二版中的代码和摘录，这些代码和摘录均在以下条款下以 MIT 许可证分发：

"版权所有 © 2010 Rust 项目开发者 允许任何获得此软件及其相关文档文件（“软件”）副本的人（“用户”）免费使用该软件，不受限制地处理该软件，包括但不限于使用、复制、修改、合并、发布、分发、再许可和/或销售软件副本，并允许向用户提供软件的人这样做，前提是遵守以下条件：上述版权声明和本许可声明应包含在软件的所有副本或主要部分中。软件按“原样”提供，不提供任何形式的保证，无论是明示的还是暗示的，包括但不限于适销性、特定用途的适用性和非侵权性。在任何情况下，作者或版权所有者均不对任何索赔、损害或其他责任负责，无论该责任是基于合同、侵权或其他原因，无论该责任是否源于、源于或与软件或其使用或其他方式有关。"
