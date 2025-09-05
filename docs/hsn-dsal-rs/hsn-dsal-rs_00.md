# 前言

当我第一次努力每年学习一门编程语言时，我从 Ruby 开始，然后学习了一点点 Scala，直到 2015 年，我开始学习一门非常新的语言：Rust。我第一次尝试创建一个 Slack（一个团队聊天程序）机器人多少有些成功，但非常令人沮丧。习惯了 Python 对 JSON 数据的灵活性和宽容的编译器，Rust 陡峭的学习曲线很快让我感到压力山大。

接下来的项目更加成功。一个数据库驱动程序，以及我为树莓派定制的**物联网**（**IoT**）客户端和服务器应用程序，使我能够以非常稳定的方式收集温度数据。与 Python 不同，如果程序编译成功，它几乎肯定会按预期工作——我非常喜欢它。

从那时起，发生了许多变化。像微软和亚马逊这样的大公司开始采用 Rust 作为在嵌入式设备和云中创建安全且快速代码的方式。随着**WebAssembly**（**Wasm**）的兴起，Rust 在 Web 前端领域也开始受到关注，游戏公司也开始在 Rust 中构建游戏引擎。2018 年是技术和 Rust 社区的一个好年，两者都将在 2019 年（以及以后）继续增长。

因此，我希望提供一种从实用角度创建更复杂 Rust 代码的学习资源。无论你的旅程将你引向何方，了解 Rust 及其各种编程模型将使你对代码的看法变得更好。

# 本书面向的对象

Rust 有很好的教程，可以学习语言的基础知识。每个会议都有研讨会，许多城市都有定期的聚会，还有一个非常有帮助的在线社区。然而，许多开发者发现自己已经超出了这些资源，但仍感觉没有准备好更复杂的解决方案。特别是来自不同背景，有多年经验的人，过渡可能会很艰难：一方面有“Hello World！”程序的一些示例；另一方面，有数千行代码的巨大的 Rust 开源项目——很难快速学习。如果你有这样的感觉，那么这本书就是为你准备的。

# 本书涵盖的内容

第一章，*你好，Rust！*，简要回顾了 Rust 编程语言及其 2018 版的变化。

第二章，*货物与板条箱*，讨论了 Rust 的`cargo`构建工具。我们将探讨配置以及构建过程和模块化选项。

第三章，*高效存储*，探讨了在 Rust 中，知道值存储的位置不仅对性能很重要，而且对理解错误信息和语言本身也很重要。在本章中，我们思考了栈和堆内存。

第四章，*列表，列表，还有更多列表*，涵盖了第一个数据结构：列表。通过几个示例，这一章深入探讨了顺序数据结构的变体及其实现。

第五章，*健壮的树*，继续我们探索流行数据结构的旅程：树是下一个要讨论的。在几个详细的示例中，我们探讨了这些高效设计的内部工作原理以及它们如何显著提高应用程序的性能。

第六章，*探索映射和集合*，探讨了最流行的键值存储：映射。在这一章中，详细描述了围绕哈希映射；哈希；以及它们的近亲集合的技术。

第七章，*Rust 中的集合*，试图与 Rust 程序员的日常生活联系起来，深入探讨了 Rust `std::collections` 库的细节，该库包含了 Rust 标准库提供的各种数据结构。

第八章，*算法评估*，教你如何评估和比较算法。

第九章，*排序事物*，将探讨排序值，这是编程中的一个重要任务——这一章揭示了如何快速且安全地完成这项任务。

第十章，*寻找东西*，转向搜索，如果没有基本的数据结构来支持它，这一点尤为重要。在这些情况下，我们使用算法来快速找到我们想要的东西。

第十一章，*随机与组合*，我们将看到，除了排序和搜索之外，还有很多问题可以通过算法来解决。这一章全部关于这些：随机数生成、回溯以及提高计算复杂度。

第十二章，*标准库算法*，探讨了 Rust 标准库在排序和搜索等日常算法任务中的实现方式。

# 为了充分利用这本书

这本书附带了很多代码示例和实现。为了让你尽可能多地学习，建议安装 Rust（任何高于 1.33 的版本都应适用）并运行所有示例。以下是一些关于文本编辑器和其它工具的推荐：

+   微软的 Visual Studio Code ([`code.visualstudio.com/`](https://code.visualstudio.com/))，可以说是最好的 Rust 代码编辑器之一

+   通过插件支持 Visual Studio Code 的 Rust ([`github.com/rust-lang/rls-vscode`](https://github.com/rust-lang/rls-vscode))

+   **Rust 语言服务器**（**RLS**），位于 [`github.com/rust-lang/rls-vscode`](https://github.com/rust-lang/rls-vscode)，通过 `rustup`（[`rustup.rs/`](https://rustup.rs/)）安装。

+   使用 Visual Studio Code 的 LLDB 前端插件（[`github.com/vadimcn/vscode-lldb`](https://github.com/vadimcn/vscode-lldb)）提供的调试支持

在设置好此环境并熟悉它之后，对您的日常 Rust 编程大有裨益，并让您能够调试和检查本书提供的代码的工作原理。为了最大限度地利用本书，我们建议您执行以下操作：

+   检查存储库中的源代码以获得完整情况。代码片段只是展示具体内容的独立示例。

+   不要盲目相信我们的结果；运行每个子项目（章节）的测试和基准测试以自行重现结果。

# 下载彩色图像

我们还提供了一份包含本书中使用的截图/图表彩色图像的 PDF 文件。您可以从这里下载：[`www.packtpub.com/sites/default/files/downloads/9781788995528_ColorImages.pdf`](https://www.packtpub.com/sites/default/files/downloads/9781788995528_ColorImages.pdf)。

# 下载示例代码文件

您可以从 [www.packt.com](http://www.packt.com) 的账户下载本书的示例代码文件。如果您在其他地方购买了本书，您可以访问 [www.packt.com/support](http://www.packt.com/support) 并注册，以便将文件直接通过电子邮件发送给您。

您可以通过以下步骤下载代码文件：

1.  在 [www.packt.com](http://www.packt.com) 登录或注册。

1.  选择“支持”选项卡。

1.  点击“代码下载与勘误”。

1.  在搜索框中输入书籍名称，并遵循屏幕上的说明。

文件下载完成后，请确保使用最新版本的以下软件解压或提取文件夹：

+   Windows 上的 WinRAR/7-Zip

+   Mac 上的 Zipeg/iZip/UnRarX

+   Linux 上的 7-Zip/PeaZip

本书代码包也托管在 GitHub 上，地址为 [`github.com/PacktPublishing/Hands-On-Data-Structures-and-Algorithms-with-Rust`](https://github.com/PacktPublishing/Hands-On-Data-Structures-and-Algorithms-with-Rust)。如果代码有更新，它将在现有的 GitHub 仓库中更新。

我们还有来自我们丰富的图书和视频目录的其他代码包可供选择，位于 **[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)**。查看它们！

# 使用的约定

本书使用了多种文本约定。

`CodeInText`：表示文本中的代码单词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 账号。以下是一个示例：“原因是 `passing_through` 变量比 `x` 存活时间更长。”

代码块应如下设置：

```rs
fn my_function() {
    let x = 10;
    do_something(x); // ownership is moved here
    let y = x; // x is now invalid!
}
```

当我们希望您注意代码块中的特定部分时，相关的行或项目将以粗体显示：

```rs
fn main() { 
    let mut a = 42; 
    let b = &a; // borrow a 
    let c = &mut a; // borrow a again, mutably 
    // ... but don't ever use b
}
```

任何命令行输入或输出都应如下编写：

```rs
$ cargo test 
```

**粗体**: 表示新术语、重要单词或屏幕上看到的单词。例如，菜单或对话框中的单词在文本中显示如下。以下是一个示例：“从管理面板中选择系统信息。”

警告或重要注意事项看起来像这样。

小贴士和技巧看起来像这样。

# 联系我们

我们始终欢迎读者的反馈。

**一般反馈**: 如果你对此书的任何方面有疑问，请在邮件主题中提及书名，并通过`customercare@packtpub.com`发送邮件给我们。

**勘误**: 尽管我们已经尽一切努力确保内容的准确性，但错误仍然可能发生。如果你在这本书中发现了错误，我们将不胜感激，如果你能向我们报告这一点。请访问[www.packt.com/submit-errata](http://www.packt.com/submit-errata)，选择你的书籍，点击勘误提交表单链接，并输入详细信息。

**盗版**: 如果你在互联网上以任何形式发现我们作品的非法副本，如果你能提供位置地址或网站名称，我们将不胜感激。请通过`copyright@packt.com`与我们联系，并附上材料的链接。

**如果你有兴趣成为作者**: 如果你精通某个主题，并且你感兴趣的是撰写或为书籍做出贡献，请访问[authors.packtpub.com](http://authors.packtpub.com/).

# 评论

留下评论。一旦你阅读并使用了这本书，为什么不在你购买它的网站上留下评论呢？潜在读者可以看到并使用你的客观意见来做出购买决定，我们 Packt 可以了解你对我们的产品的看法，我们的作者也可以看到他们对书籍的反馈。谢谢！

如需了解 Packt 的更多信息，请访问 [packt.com](http://www.packt.com/).
