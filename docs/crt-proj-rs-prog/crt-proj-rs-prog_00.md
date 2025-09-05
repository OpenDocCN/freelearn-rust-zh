前言

本书展示了 Rust 程序员可以免费使用的最有趣和最有用的库和框架，用于构建有趣和有用的项目，例如前端和后端 Web 应用程序、游戏、解释器、编译器、计算机模拟器和 Linux 可加载模块。

# 本书面向对象

本书面向已经学习过 Rust 编程语言并渴望将其应用于构建有用软件的开发者，无论是商业项目还是个人爱好项目。本书涵盖了多样化的需求，如构建 Web 应用程序、计算机游戏、解释器、编译器、模拟器或设备驱动程序。

阅读数据库章节需要一些 SQL 知识，而阅读 Linux 模块章节则需要 C 编程语言和 Linux 工具的知识。

# 本书涵盖内容

第一章，*Rust 2018 – 生产力*，介绍了 Rust 语言及其工具和库生态系统中的最新创新。特别是，它展示了如何使用一些广泛使用的实用库。

第二章，*存储和检索数据*，介绍了如何在 Rust 世界中读取和写入一些最流行的文本文件格式：TOML、JSON 和 XML。它还描述了如何访问 Rust 世界中一些最流行的数据库引擎，如 SQLite、PostgreSQL 和 Redis。

第三章，*创建 REST Web 服务*，介绍了如何使用 Actix 框架开发 REST 服务，该服务可以用作任何类型客户端应用程序的后端，尤其是 Web 应用程序。

第四章，*创建完整的后端 Web 应用程序*，介绍了如何使用 Tera 模板引擎替换文本文件中的占位符，以及如何使用 Actix 框架创建完整的后端 Web 应用程序。

第五章，*使用 Yew 创建客户端 WebAssembly 应用程序*，介绍了如何使用利用 WebAssembly 技术的 Yew 框架来创建 Web 应用程序的前端。

第六章，*使用 Quicksilver 创建 WebAssembly 游戏*，介绍了如何使用 Quicksilver 框架创建可在网页浏览器中运行的图形 2D 游戏，利用 WebAssembly 技术，或者作为桌面应用程序。

第七章，*使用 ggez 创建桌面二维游戏*，介绍了如何使用 ggez 框架创建桌面图形 2D 游戏，包括小部件的覆盖范围。

第八章，*使用解析器组合进行解释和编译*，描述了如何使用 Nom 解析器组合创建形式语言的解析器，然后构建语法检查器、解释器和编译器。

第九章，*使用 Nom 创建计算机模拟器*，描述了如何使用 Nom 库解析二进制数据并解释机器语言程序，这是构建计算机模拟器的第一步。

第十章，*创建 Linux 内核模块*，描述了如何使用 Rust 构建 Linux 可加载模块，重点关注 Mint 发行版；具体来说，将构建一个字符设备驱动程序。

第十一章，*Rust 的未来*，描述了未来几年 Rust 生态系统可能出现的创新。特别是，简要展示了新的异步编程技术。

# 要充分利用本书

| **本书涵盖的软件/硬件** | **操作系统要求** |
| --- | --- |
| 您需要在计算机上安装 Rust 1.31 版（自 2018 年 12 月起）或更高版本。本书的内容在 64 位 Linux Mint 和 32 位 Windows 10 系统上进行了测试。大多数示例应该适用于任何支持 Rust 的系统。第五章，*使用 Yew 创建客户端 WebAssembly 应用程序*，和第六章，*使用 Quicksilver 创建 WebAssembly 游戏*，需要支持 WebAssembly 的网络浏览器，如 Chrome 或 Firefox。第六章，*使用 Quicksilver 创建 WebAssembly 游戏*，和第七章，*使用 ggez 创建桌面二维游戏*，需要支持 OpenGL。第十章，*创建 Linux 内核模块*，仅在 Linux Mint 上运行。 |

**如果您正在使用本书的数字版，我们建议您亲自输入代码或通过下一节中提供的 GitHub 仓库访问代码。这样做将帮助您避免与代码复制和粘贴相关的任何潜在错误。**

## 下载示例代码文件

您可以从[www.packt.com](http://www.packt.com)的账户下载本书的示例代码文件。如果您在其他地方购买了此书，您可以访问[www.packtpub.com/support](https://www.packtpub.com/support)并注册，以便将文件直接通过电子邮件发送给您。

您可以通过以下步骤下载代码文件：

1.  在[www.packt.com](http://www.packt.com)登录或注册。

1.  选择“支持”标签。

1.  点击“代码下载”。

1.  在“搜索”框中输入书籍名称，并遵循屏幕上的说明。

文件下载完成后，请确保使用最新版本的软件解压缩或提取文件夹：

+   Windows 版的 WinRAR/7-Zip

+   Mac 版的 Zipeg/iZip/UnRarX

+   Linux 版的 7-Zip/PeaZip

本书代码包也托管在 GitHub 上，地址为**[`github.com/PacktPublishing/Creative-Projects-for-Rust-Programmers`](https://github.com/PacktPublishing/Creative-Projects-for-Rust-Programmers)**。如果代码有更新，它将在现有的 GitHub 仓库中更新。

我们还有其他来自我们丰富图书和视频目录的代码包，可在**[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)**上找到。查看它们吧！

## 下载彩色图像

我们还提供了一份包含本书中使用的截图/图表彩色图像的 PDF 文件。您可以从这里下载：[`static.packt-cdn.com/downloads/9781789346220_ColorImages.pdf`](https://static.packt-cdn.com/downloads/9781789346220_ColorImages.pdf)。

## 使用约定

本书使用了许多文本约定。

`CodeInText`：表示文本中的代码单词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 昵称。以下是一个示例：“`pos`变量是`digits`数组中当前数字的位置。”

代码块设置如下：

```rs
{
    for pos in pos..5 {
        print!("{}", digits[pos] as u8 as char);
}
```

任何命令行输入或输出都按如下方式编写：

```rs
curl -X GET http://localhost:8080/datafile.txt
```

**粗体**：表示新术语、重要单词或屏幕上看到的单词。例如，菜单或对话框中的单词在文本中显示如下。以下是一个示例：“名称部分编辑框和其右侧的过滤器按钮用于过滤下方的表格，方式类似于`list`项目。”

警告或重要提示如下所示。

技巧和窍门如下所示。

# 联系我们

读者反馈始终欢迎。

**一般反馈**：如果您对本书的任何方面有疑问，请在邮件主题中提及书名，并通过`customercare@packtpub.com`给我们发送邮件。

**勘误表**：尽管我们已经尽最大努力确保内容的准确性，但错误仍然可能发生。如果您在这本书中发现了错误，如果您能向我们报告，我们将不胜感激。请访问[www.packtpub.com/support/errata](https://www.packtpub.com/support/errata)，选择您的书籍，点击勘误提交表单链接，并输入详细信息。

**盗版**：如果您在互联网上以任何形式发现我们作品的非法副本，如果您能提供位置地址或网站名称，我们将不胜感激。请通过`copyright@packt.com`与我们联系，并提供材料的链接。

**如果您有兴趣成为作者**：如果您在某个主题上具有专业知识，并且您有兴趣撰写或为书籍做出贡献，请访问[authors.packtpub.com](http://authors.packtpub.com/)。

## 评论

请留下您的评价。一旦您阅读并使用了这本书，为何不在购买它的网站上留下评价呢？潜在读者可以查看并使用您的客观意见来做出购买决定，我们 Packt 公司可以了解您对我们产品的看法，并且我们的作者可以查看他们对书籍的反馈。谢谢！

如需了解更多关于 Packt 的信息，请访问 [packt.com](http://www.packt.com/)。
