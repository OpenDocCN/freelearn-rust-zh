# 前言

本书的目标是简要介绍一些 Rust 的基础（与 GUI 玩耍）和高级（异步编程）功能。因为有趣的项目总是语言学习过程中的巨大加分项，所以我们以这个焦点编写了这本书。我们认为这门语言很棒，并希望给你提供动力和知识，以便未来能有更多的 Rustaceans！

# 本书面向的对象

如果读者想最大限度地享受本书，他们只需要对 Rust 语言有基本了解，尽管建议始终打开文档以回答本书可能未提供的问题（我们，作者，并非无所不能，这很遗憾，我们知道）。对于完全不了解 Rust 的读者，我们建议他们首先阅读这里可以找到的 Rust 书籍[`doc.rust-lang.org/stable/book/`](https://doc.rust-lang.org/stable/book/)，然后再回来阅读这本书！

# 本书涵盖的内容

第一章, *Rust 基础*，涵盖 Rust 的安装并教授语言的语法和基本原理，以便你准备好使用它进行项目编码。

第二章, *从 SDL 开始*，展示了如何开始使用 SDL 及其主要功能，如事件和绘图。一旦项目创建，我们将创建一个显示图像的窗口。

第三章, *事件和基本游戏机制*，深入讲解如何处理事件。我们将编写 tetrimino 对象，并使它们根据接收的事件进行变化。

第四章, *添加所有游戏机制*，完成游戏机制。在本章结束时，我们将拥有一个完全运行的俄罗斯方块游戏。

第五章, *创建音乐播放器*，帮助你开始构建图形化音乐播放器。本章将仅涵盖用户界面。

第六章, *实现音乐播放器的引擎*，将音乐播放器引擎添加到图形应用程序中。

第七章, *使用 Relm 以更 Rust 的方式实现音乐播放器*，改进音乐播放器以添加播放功能，允许处理列表中的音乐以去除人声。

第八章, *理解 FTP*，通过实现同步 FTP 服务器来介绍 FTP 协议，为你在下一章编写异步版本做准备。

第九章, *实现异步 FTP 服务器*，使用 Tokio 实现了 FTP 协议。

第十章，*实现异步文件传输*，实现了 FTP 服务本身。这是应用程序能够上传和下载文件的地方。

附录，*Rust 最佳实践*，展示了如何编写优秀的 Rust API 以及如何使它们尽可能容易和愉快地使用。

# 要充分利用本书

您需要的并不多。此外，Rust 在所有操作系统上都得到了很好的支持。在这里，Linux 是支持最好的操作系统。您也可以在 Windows 和 macOS 上使用 Rust，您需要一个相当新的计算机；对于本书的目的，1GB 的 RAM 应该足够了。

# 下载示例代码文件

您可以从 [www.packtpub.com](http://www.packtpub.com) 的账户中下载本书的示例代码文件。如果您在其他地方购买了这本书，您可以访问 [www.packtpub.com/support](http://www.packtpub.com/support) 并注册，以便将文件直接通过电子邮件发送给您。

您可以通过以下步骤下载代码文件：

1.  在 [www.packtpub.com](http://www.packtpub.com/support) 登录或注册。

1.  选择“支持”选项卡。

1.  点击“代码下载与勘误”。

1.  在搜索框中输入书籍名称，并遵循屏幕上的说明。

下载完文件后，请确保使用最新版本的以下软件解压缩或提取文件夹：

+   适用于 Windows 的 WinRAR/7-Zip

+   适用于 Mac 的 Zipeg/iZip/UnRarX

+   适用于 Linux 的 7-Zip/PeaZip

本书代码包也托管在 GitHub 上，网址为[`github.com/PacktPublishing/Rust-Programming-By-Example`](https://github.com/PacktPublishing/Rust-Programming-By-Example)。我们还有其他来自我们丰富图书和视频目录的代码包，可在**[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)**找到。查看它们！

# 下载彩色图像

我们还提供了一份包含本书中使用的截图/图表彩色图像的 PDF 文件。您可以从这里下载：[`www.packtpub.com/sites/default/files/downloads/RustProgrammingByExample_ColorImages.pdf`](https://www.packtpub.com/sites/default/files/downloads/RustProgrammingByExample_ColorImages.pdf)

# 使用的约定

本书使用了多种文本约定。

`CodeInText`：表示文本中的代码单词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 昵称。以下是一个示例：“将下载的`WebStorm-10*.dmg`磁盘映像文件挂载为系统中的另一个磁盘。”

代码块设置如下：

```rs
html, body, #map {
 height: 100%; 
 margin: 0;
 padding: 0
}
```

当我们希望您注意代码块中的特定部分时，相关的行或项目将以粗体显示：

```rs
[default]
exten => s,1,Dial(Zap/1|30)
exten => s,2,Voicemail(u100)
exten => s,102,Voicemail(b100)
exten => i,1,Voicemail(s0)
```

任何命令行输入或输出都按以下方式编写：

```rs
$ mkdir css
$ cd css
```

**粗体**: 表示新术语、重要单词或您在屏幕上看到的单词。例如，菜单或对话框中的单词在文本中显示如下。以下是一个示例：“从管理面板中选择系统信息。”

警告或重要提示看起来像这样。

小贴士和技巧看起来像这样。

# 联系我们

我们始终欢迎读者的反馈。

**一般反馈**: 请通过 `feedback@packtpub.com` 发送电子邮件，并在邮件主题中提及书籍标题。如果您对本书的任何方面有疑问，请通过 `questions@packtpub.com` 发送电子邮件给我们。

**勘误**: 尽管我们已经尽最大努力确保内容的准确性，但错误仍然可能发生。如果您在这本书中发现了错误，如果您能向我们报告，我们将不胜感激。请访问 [www.packtpub.com/submit-errata](http://www.packtpub.com/submit-errata)，选择您的书籍，点击勘误提交表单链接，并输入详细信息。

**盗版**: 如果您在互联网上以任何形式发现了我们作品的非法副本，如果您能提供位置地址或网站名称，我们将不胜感激。请通过 `copyright@packtpub.com` 联系我们，并附上材料的链接。

**如果您有兴趣成为作者**：如果您在某个领域有专业知识，并且您有兴趣撰写或为书籍做出贡献，请访问 [authors.packtpub.com](http://authors.packtpub.com/)。

# 评论

请留下评论。一旦您阅读并使用了这本书，为何不在您购买它的网站上留下评论呢？潜在读者可以查看并使用您的客观意见来做出购买决定，Packt 公司可以了解您对我们产品的看法，我们的作者也可以看到他们对书籍的反馈。谢谢！

有关 Packt 的更多信息，请访问 [packtpub.com](https://www.packtpub.com/)。
