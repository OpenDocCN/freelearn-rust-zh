# 前言

欢迎您。本书的目的是教初学者和中级 Rust 程序员如何利用 Rust 编程语言中的现代并行机器。本书将包含与 Rust 编程语言特别相关的各种信息，特别是关于其标准库的信息，但它也将包含更普遍适用的信息，但恰好是用 Rust 表达。从我个人的角度来看，Rust 本身*并不是*一个非常具有创造性的语言。它的主要贡献，在我看来，是将仿射类型主流化并应用于内存分配跟踪。在大多数其他方面，它是一个熟悉的系统编程语言，对于那些有 GC（垃圾收集）无编程语言背景的人来说，应该会感到熟悉——经过一些调整。这是好事，因为我们的目标是研究并发——关于这个主题的论文和书籍中有很多信息，我们理解和应用了这些概念。本书将参考许多这样的作品，其上下文是 C++、ATS、ADA 和类似的语言。

# 这本书面向的对象是谁

如果这是您读的第一本 Rust 编程书籍，我衷心感谢您的热情，但鼓励您寻找适合的编程语言入门介绍。这本书将直接进入正题，如果您对基础知识有疑问，可能不太合适。Rust 社区已经制作了*优秀*的文档，包括入门文本。《Rust 编程语言入门》（[`doc.rust-lang.org/book/first-edition/`](https://doc.rust-lang.org/book/first-edition/）），第一版，是许多已经在社区中的人学习这门语言的方式。这本书的第二版，在写作时仍在进行中，看起来比原始文本有所改进，也值得推荐。还有许多其他优秀的入门介绍可供购买。

# 要充分利用这本书

本书在相对较短的空间内涵盖了深入的主题。在整个文本中，预期读者对 Rust 编程语言感到舒适，能够访问 Rust 编译器和用于编译的计算机，并执行 Rust 程序。本书中出现的其他附加软件在第一章“预备知识 – 计算机架构和 Rust 入门”中有所介绍，并建议使用但不是强制性的。

本书的基本前提如下——并行编程很难但并非不可能。当关注计算环境并对产生的程序进行验证时，可以很好地进行并行编程。为此，每一章都是针对向读者传授章节主题的坚实基础而编写的。一旦消化了章节内容，读者将有望在该主题的现有文献中找到一条坚实的路径。在这方面，这是一本关于开始而不是结束的书。

我强烈建议读者在阅读本书时积极发挥作用，下载源代码，在阅读过程中独立调查项目，而不依赖于本书的观点，并使用您操作系统上的工具检查运行中的程序。像任何其他事情一样，并行编程能力是通过实践获得和磨练的技能。

最后一点——让学习过程按照自己的节奏进行。如果某一章节的主题没有立即在您心中扎根，那没关系。对我来说，写书的过程是灵感闪现和知识逐渐展开的过程，就像一朵花慢慢绽放。我想象阅读本书的过程也将类似。

# 下载示例代码文件

本书代码包托管在 GitHub 上，网址为[`github.com/PacktPublishing/Hands-On-Concurrency-with-Rust`](https://github.com/PacktPublishing/Hands-On-Concurrency-with-Rust)。如果代码有更新，它将在现有的 GitHub 仓库中更新。

您也可以从[www.packtpub.com](http://www.packtpub.com)的账户中下载本书的示例代码文件。如果您在其他地方购买了本书，您可以访问[www.packtpub.com/support](http://www.packtpub.com/support)并注册，以便将文件直接通过电子邮件发送给您。

您可以通过以下步骤下载代码文件：

1.  在[www.packtpub.com](http://www.packtpub.com/support)登录或注册。

1.  选择“SUPPORT”标签。

1.  点击“代码下载与勘误”。

1.  在搜索框中输入书名，并遵循屏幕上的说明。

文件下载完成后，请确保您使用最新版本的软件解压缩或提取文件夹：

+   Windows 下的 WinRAR/7-Zip

+   Mac 下的 Zipeg/iZip/UnRarX

+   Linux 下的 7-Zip/PeaZip

我们还有来自我们丰富的图书和视频目录中的其他代码包可供选择，请访问**[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)**。查看它们！

# 使用的约定

在本书中使用了多种文本约定。

`CodeInText`: 表示文本中的代码单词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 昵称。以下是一个示例：“种子是用于`XorShiftRng`，移动到上一行的`max_in_memory_bytes`和`max_disk_bytes`是用于 hopper。”

代码块设置如下：

```rs
fn main() {
    println!("Apollo is the name of a space program but also my dog.");
}
```

任何命令行输入或输出都应如下所示：

```rs
> cat hello.rs
fn main() {
 println!("Apollo is the name of a space program but also my dog.");
}
> rustc -C opt-level=2 hello.rs
> ./hello
Apollo is the name of a space program but also my dog.
```

**粗体**: 表示新术语、重要单词或屏幕上看到的单词。

警告或重要注意事项如下所示。

小贴士和技巧看起来像这样。

# 联系我们

欢迎提供反馈。

**一般反馈**: 请通过电子邮件`feedback@packtpub.com`发送反馈，并在邮件主题中提及书籍标题。如果您对本书的任何方面有疑问，请通过电子邮件`questions@packtpub.com`联系我们。

**勘误表**：尽管我们已经尽一切努力确保内容的准确性，但错误仍然可能发生。如果您在这本书中发现了错误，我们将不胜感激，如果您能向我们报告这一点。请访问[www.packtpub.com/submit-errata](http://www.packtpub.com/submit-errata)，选择您的书籍，点击勘误表提交表单链接，并输入详细信息。

**盗版**: 如果您在互联网上发现我们作品的任何非法副本，如果您能提供位置地址或网站名称，我们将不胜感激。请通过电子邮件`copyright@packtpub.com`联系我们，并附上材料的链接。

**如果您有兴趣成为作者**：如果您在某个领域有专业知识，并且您有兴趣撰写或为书籍做出贡献，请访问[authors.packtpub.com](http://authors.packtpub.com/)。

# 评论

请在您购买书籍的网站上为本书留下评论。评论帮助我们 Packt 了解您对我们的产品的看法，并且我们的作者可以看到他们对本书的反馈。谢谢！

如需了解 Packt 的更多信息，请访问 [packtpub.com](https://www.packtpub.com/).
