# 前言

Rust 是一种系统级编程语言，越来越受欢迎。这种受欢迎程度是由其语义驱动的，它鼓励创建快速、可靠的软件。在这本书中，我们将学习语言的基础，最终达到可以开始编写可用的程序的程度。

# 本书面向对象

本书面向对学习日益流行的 Rust 编程语言感兴趣的人。您不必已经是程序员，但如果您是，那将有所帮助。

# 为了充分利用本书

至少对另一种编程语言有一些了解将有助于比较和对比。您需要互联网连接来下载和安装编译器工具链。假设您能够使用命令行工具。

# 下载示例代码文件

您可以从 [www.packt.com](http://www.packt.com) 的账户下载本书的示例代码文件。如果您在其他地方购买了本书，您可以访问 [www.packt.com/support](http://www.packt.com/support) 并注册，以便将文件直接通过电子邮件发送给您。

您可以通过以下步骤下载代码文件：

1.  在 [www.packt.com](http://www.packt.com) 登录或注册。

1.  选择“支持”标签。

1.  点击“代码下载与勘误”。

1.  在搜索框中输入书籍名称，并遵循屏幕上的说明。

文件下载后，请确保使用最新版本的以下软件解压或提取文件夹：

+   WinRAR/7-Zip for Windows

+   Zipeg/iZip/UnRarX for Mac

+   7-Zip/PeaZip for Linux

本书代码包托管在 GitHub 上，地址为 **[`github.com/PacktPublishing/Rust-Quick-Start-Guide`](https://github.com/PacktPublishing)**。如果代码有更新，它将在现有的 GitHub 仓库中更新。

我们还有来自我们丰富的图书和视频目录的其他代码包可供下载，地址为 **[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)**。查看它们！

# 下载彩色图像

我们还提供了一份包含本书中使用的截图/图表彩色图像的 PDF 文件。您可以从这里下载：[`www.packtpub.com/sites/default/files/downloads/9781789616705_ColorImages.pdf`](https://www.packtpub.com/sites/default/files/downloads/9781789616705_ColorImages.pdf)。

# 使用的约定

本书使用了多种文本约定。

`CodeInText`：表示文本中的代码单词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 昵称。以下是一个示例：“将下载的 `WebStorm-10*.dmg` 磁盘镜像文件作为系统中的另一个磁盘挂载。”

代码块设置如下：

```rs
fn main() {
     println!("Hello, world!");
 }

```

当我们希望您注意代码块中的特定部分时，相关的行或项目将以粗体显示：

```rs
if 3 > 4 {
    println!("Uh-oh. Three is greater than four.");
}
else if 3 == 4 {
    println!("There seems to be something wrong with math.");
}
```

任何命令行输入或输出都按以下方式编写：

```rs
$ mkdir css
$ cd css
```

**粗体**: 表示新术语、重要单词或屏幕上看到的单词。例如，菜单或对话框中的单词在文本中显示如下。以下是一个示例：“从管理面板中选择系统信息。”

警告或重要提示看起来像这样。

小贴士和技巧看起来像这样。

# 联系我们

我们欢迎读者的反馈。

**一般反馈**: 如果您对本书的任何方面有疑问，请在邮件主题中提及书名，并通过`customercare@packtpub.com`给我们发送邮件。

**勘误**: 尽管我们已经尽一切努力确保内容的准确性，但错误仍然可能发生。如果您在这本书中发现了错误，我们将不胜感激，如果您能向我们报告这一点。请访问[www.packt.com/submit-errata](http://www.packt.com/submit-errata)，选择您的书籍，点击勘误提交表单链接，并输入详细信息。

**盗版**: 如果您在互联网上以任何形式发现我们作品的非法副本，如果您能提供位置地址或网站名称，我们将不胜感激。请通过`copyright@packt.com`与我们联系，并提供材料的链接。

**如果您有兴趣成为作者**: 如果您在某个领域有专业知识，并且您有兴趣撰写或为书籍做出贡献，请访问[authors.packtpub.com](http://authors.packtpub.com/)。

# 评论

请留下评论。一旦您阅读并使用了这本书，为什么不在此处购买它的网站上留下评论呢？潜在读者可以查看并使用您的客观意见来做出购买决定，Packt 公司可以了解您对我们产品的看法，我们的作者也可以看到他们对书籍的反馈。谢谢！

如需了解 Packt 的更多信息，请访问[packt.com](http://www.packt.com/)。
