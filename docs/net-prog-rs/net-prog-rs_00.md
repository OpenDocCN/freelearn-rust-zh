# 前言

Rust 近年来稳步成为最重要的新编程语言之一。就像 C 或 C++一样，Rust 允许开发者编写足够低级的代码，使 Rust 代码运行得更快。由于 Rust 在设计上就是内存安全的，它不允许代码在空指针异常上崩溃。这些特性使其成为编写低级网络应用的天然选择。这本书将帮助开发者开始使用 Rust 编写网络应用。

# 本书面向的对象

本书的目标读者是那些对使用 Rust 编写网络软件感兴趣的软件工程师。

# 本书涵盖的内容

第一章，*客户端/服务器网络简介*，从零开始温和地介绍计算机网络。这包括 IP 地址、TCP/UDP 和 DNS。这构成了我们在后续章节讨论的基础。

第二章，*Rust 及其生态系统简介*，包含了对 Rust 的介绍。这是一个全面的介绍，应该足以让读者开始。我们假设读者对编程有一定的了解。

第三章，*使用 Rust 进行 TCP 和 UDP*，深入探讨了使用 Rust 进行网络编程。我们首先使用标准库进行基本的套接字编程。然后我们看看生态系统中可用于网络编程的一些 crate。

第四章，*数据序列化、反序列化和解析*，解释了网络计算的一个重要方面是处理数据。本章是关于使用 Serde 进行序列化和反序列化的介绍。我们还探讨了使用 nom 和其他框架进行解析。

第五章，*应用层协议*，上升一层来查看在 TCP/IP 之上运行的协议。我们查看了一些可以与之合作的 crate，如 RPC、SMTP、FTP 和 TFTP。

第六章，*在互联网中谈论 HTTP*，解释了互联网最常见应用之一是 HTTP。我们探讨了 Hyper 和 Rocket 等用于编写 HTTP 服务器和客户端的 crate。

第七章，*使用 Tokio 进行异步网络编程*，探讨了使用 futures、streams 和事件循环进行异步编程的 Tokio 堆栈。

第八章，*安全性*，深入探讨了保护我们之前描述的服务。这是使用证书和密钥。

第九章，*附录*讨论了已经出现的一些 crate，它们提出了本书已涵盖的做事方式的替代方法。这包括 async/await 语法、使用 Pest 进行解析等。我们将在附录中讨论其中的一些。

# 要充分利用本书

1.  他们要么已经熟悉 Rust，要么计划开始学习这门语言。

1.  他们有使用其他编程语言进行软件工程商业背景，并且了解使用不同编程语言开发软件的权衡。

1.  他们对网络概念有基本的了解。

1.  他们可以理解为什么分布式系统在现代计算中很重要。

# 下载示例代码文件

您可以从[www.packtpub.com](http://www.packtpub.com)的账户下载本书的示例代码文件。如果您在其他地方购买了本书，您可以访问[www.packtpub.com/support](http://www.packtpub.com/support)并注册，以便将文件直接通过电子邮件发送给您。

您可以通过以下步骤下载代码文件：

1.  在[www.packtpub.com](http://www.packtpub.com/support)登录或注册。

1.  选择 SUPPORT 标签页。

1.  点击代码下载与勘误。

1.  在搜索框中输入书名，并遵循屏幕上的说明。

文件下载后，请确保使用最新版本解压缩或提取文件夹：

+   Windows 上的 WinRAR/7-Zip

+   Mac 上的 Zipeg/iZip/UnRarX

+   Linux 上的 7-Zip/PeaZip

本书代码包也托管在 GitHub 上，网址为[`github.com/PacktPublishing/Network-Programming-with-Rust`](https://github.com/PacktPublishing/Network-Programming-with-Rust)。我们还有其他来自我们丰富图书和视频目录的代码包可用，网址为**[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)**。请查看它们！

# 使用的约定

本书使用了多种文本约定。

`CodeInText`: 表示文本中的代码单词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 昵称。以下是一个示例：“`target` 目录包含编译工件。”

代码块设置如下：

```rs
[package]
name = "hello-rust"
version = "0.1.0"
authors = ["Foo Bar <foo.bar@foobar.com>"]
```

任何命令行输入或输出都应如下编写：

```rs
# cargo new --bin hello-rust
```

**粗体**: 表示新术语、重要单词或屏幕上看到的单词。例如，菜单或对话框中的单词在文本中显示如下。以下是一个示例：“它不需要为相同的连接调用 connect:”

警告或重要提示看起来是这样的。

小贴士和技巧看起来是这样的。

# 联系我们

我们始终欢迎读者的反馈。

**一般反馈**: 发送电子邮件至 `feedback@packtpub.com`，并在邮件主题中提及书名。如果您对本书的任何方面有疑问，请发送电子邮件至 `questions@packtpub.com`。

**勘误**: 尽管我们已经尽最大努力确保内容的准确性，但错误仍然可能发生。如果你在这本书中发现了错误，我们将不胜感激，如果你能向我们报告这个错误。请访问[www.packtpub.com/submit-errata](http://www.packtpub.com/submit-errata)，选择你的书籍，点击勘误提交表单链接，并输入详细信息。

**盗版**: 如果你在互联网上以任何形式遇到我们作品的非法副本，如果你能提供位置地址或网站名称，我们将不胜感激。请通过`copyright@packtpub.com`与我们联系，并附上材料的链接。

**如果你有兴趣成为作者**: 如果你有一个你擅长的主题，并且你对撰写或为书籍做出贡献感兴趣，请访问[authors.packtpub.com](http://authors.packtpub.com/)。

# 评论

请留下评论。一旦你阅读并使用了这本书，为什么不在你购买它的网站上留下评论呢？潜在的读者可以看到并使用你的无偏见意见来做出购买决定，我们 Packt 可以了解你对我们的产品有何看法，我们的作者也可以看到他们对书籍的反馈。谢谢！

如需了解更多关于 Packt 的信息，请访问[packtpub.com](https://www.packtpub.com/)。
