# 前言

你是否希望将你的 Web 应用程序推向极限，以实现速度和低能耗，同时仍然保持内存安全？Rust 使你能够在没有垃圾回收和与 C 编程语言相似的能量消耗的情况下实现内存安全。这意味着你可以相对容易地创建高性能和安全的应用程序。

本书将带你经历 Web 开发的每个阶段，最终部署使用 Rust 构建并打包在 distroless Docker 中的高级 Web 应用程序，在 AWS 上使用我们创建的自动化构建和部署管道，生成的服务器镜像大小约为 50MB。

你将从 Rust 编程的介绍开始，这样你就可以避免在从传统动态编程语言迁移时遇到常见的问题。本书将向你展示如何为跨越多页和模块的项目结构化 Rust 代码。接下来，你将探索 Actix Web 框架，并搭建一个基本的 Web 服务器。随着你的进步，你将学习如何处理 JSON 请求，并通过 HTML、CSS 和 JavaScript 显示服务器数据，甚至为我们的数据构建一个基本的 React 应用程序。你还将学习如何在 Rust 中持久化数据并创建 RESTful 服务，其中我们在前端登录和验证用户，并缓存数据。稍后，你将在 AWS 上为应用程序构建自动化的构建和部署流程，使用两个 EC2 实例，我们将从注册域名平衡 HTTPS 流量到这些 EC2 实例上的应用程序，我们将使用 Terraform 构建这些实例。你还将通过在 Terraform 中配置安全组直接锁定到 EC2 实例的流量。然后，你将涵盖多层级构建以生成 distroless Rust 镜像。最后，你将涵盖高级 Web 概念，探索异步 Rust、Tokio、Hyper 和 TCP 帧。有了这些工具，你将实现 actor 模型，使你能够实现高级复杂的异步实时事件处理系统，通过构建一个基本的股票购买系统进行实践。你将通过在 Redis 上构建自己的队列机制来结束本书，其中你的 Rust 自建服务器和工作节点消费队列上的任务并处理这些任务。

# 这本书面向的对象

这本关于使用 Rust 进行 Web 编程的书籍是为那些在传统语言（如 Python、Ruby、JavaScript 和 Java）中编程的 Web 开发者所写，他们希望使用 Rust 开发高性能的 Web 应用程序。尽管不需要有 Rust 的先验经验，但如果你想要充分利用这本书，你需要对 Web 开发原则有扎实的理解，并对 HTML、CSS 和 JavaScript 有基本的知识。

# 本书涵盖的内容

*第一章*，*Rust 快速入门*，提供了 Rust 编程语言的基础知识。

*第二章*，*在 Rust 中设计你的 Web 应用程序*，涵盖了在 Rust 中构建和管理应用程序。

*第三章*, *处理 HTTP 请求*，介绍了使用 Actix Web 框架构建一个基本的 Rust 服务器，该服务器可以处理 HTTP 请求。

*第四章*, *处理 HTTP 请求*，介绍了从传入的 HTTP 请求中提取和处理数据。

*第五章*, *在浏览器中显示内容*，介绍了使用 HTML、CSS 和 JavaScript 以及 React 从服务器显示数据并向服务器发送请求。

*第六章*, *使用 PostgreSQL 进行数据持久性*，介绍了在 PostgreSQL 中管理和结构化数据，以及使用我们的 Rust Web 服务器与数据库交互。

*第七章*, *管理用户会话*，介绍了在向 Web 服务器发送请求时进行身份验证和管理用户会话。

*第八章*, *构建 RESTful 服务*，介绍了为 Rust Web 服务器实现 RESTful 概念。

*第九章*, *测试我们的应用程序端点和组件*，介绍了端到端测试管道和在使用 Postman 对 Rust Web 服务器进行单元测试。

*第十章*, *在 AWS 上部署我们的应用程序*，介绍了使用 Docker 构建自动化构建和部署管道，并在 AWS 上使用 Terraform 自动化基础设施构建。

*第十一章*, *在 AWS 上使用 NGINX 配置 HTTPS*，介绍了在 AWS 上通过 NGINX 配置 HTTPS 和路由，并通过负载均衡将流量路由到不同的应用程序，这取决于 URL 中的端点。

*第十二章*, *在 Rocket 中重构我们的应用程序*，介绍了将现有应用程序集成到 Rocket Web 框架中。

*第十三章*, *干净 Web 应用程序仓库的最佳实践*，介绍了使用多阶段 Docker 构建来清理 Web 应用程序仓库，以生成更小的镜像，并初始化 Docker 容器以在部署时自动化数据库迁移。

*第十四章*, *探索 Tokio 框架*，介绍了使用 Tokio 框架实现基本异步代码，以促进异步运行时。

*第十五章*, *使用 Tokio 接受 TCP 流量*，涵盖了发送、接收和处理 TCP 流量的内容。

*第十六章*, *在 TCP 之上构建协议*，介绍了使用结构和帧将 TCP 字节流处理成高级数据结构。

*第十七章*, *使用 Hyper 框架实现 Actors 和异步操作*，介绍了如何使用 actor 框架构建一个异步系统，并通过 Hyper 框架接受 HTTP 请求。

*第十八章*，*使用 Redis 排队任务*，涵盖了接受 HTTP 请求并将它们打包成任务放入 Redis 队列以供工作池处理。

# 为了充分利用本书

你需要了解一些关于 HTML 和 CSS 的基本概念。你还需要对 JavaScript 有一些基本理解。然而，HTML、CSS 和 JavaScript 只需要用于在浏览器中显示数据。如果你只是阅读本书以构建 Rust 后端 API 服务器，那么不需要了解 HTML、CSS 和 JavaScript。

还需要一些关于编程概念（如函数和循环）的基本理解，因为这些内容本书不会涉及。

| **本书涵盖的软件/硬件** | **操作系统要求** |
| --- | --- |
| Rust | Windows、macOS 或 Linux（任何） |
| 节点（JavaScript） | Windows、macOS 或 Linux（任何） |
| Python 3 | Windows、macOS 或 Linux（任何） |
| Docker | Windows、macOS 或 Linux（任何） |
| docker-compose | Windows、macOS 或 Linux（任何） |
| Postman | Windows、macOS 或 Linux（任何） |
| Terraform | Windows、macOS 或 Linux（任何） |

为了充分利用 AWS 部署章节，你需要一个 AWS 账户，如果你不符合免费层资格，这可能会产生费用。然而，构建是使用 Terraform 自动化的，因此启动和关闭构建将非常快速和简单，所以你在阅读本书时不需要保持基础设施运行，以将成本降至最低。

**如果你使用的是本书的数字版，我们建议你亲自输入代码或从本书的 GitHub 仓库（下一节中有一个链接）获取代码。这样做将帮助你避免与代码复制和粘贴相关的任何潜在错误。**

到这本书结束时，你将拥有构建和部署 Rust 服务器的基础知识。然而，必须注意的是，Rust 是一种强大的语言。由于本书侧重于网络编程和部署，在本书之后 Rust 编程还有改进的空间。建议在本书之后进一步阅读 Rust，以便你能够解决更复杂的问题。

# 下载示例代码文件

你可以从 GitHub 下载本书的示例代码文件：[`github.com/PacktPublishing/Rust-Web-Programming-2nd-Edition`](https://github.com/PacktPublishing/Rust-Web-Programming-2nd-Edition)。如果代码有更新，它将在 GitHub 仓库中更新。

我们还有其他来自我们丰富的图书和视频目录的代码包，可在 [`github.com/PacktPublishing/`](https://github.com/PacktPublishing/) 获取。查看它们吧！

# 下载彩色图像

我们还提供了一份包含本书中使用的截图和图表彩色图像的 PDF 文件。你可以从这里下载：[`packt.link/Z1lgk`](https://packt.link/Z1lgk)。

# 使用的约定

本书使用了多种文本约定。

`文本中的代码`：表示文本中的代码单词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 昵称。以下是一个示例：“这意味着我们必须修改待办事项表现有的模式，并在`src/schema.rs`文件中添加用户模式。”

代码块设置如下：

```rs
table! {
    to_do (id) {
        id -> Int4,
        title -> Varchar,
        status -> Varchar,
        date -> Timestamp,
        user_id -> Int4,
    }
}
table! {
    users (id) {
        id -> Int4,
        username -> Varchar,
        email -> Varchar,
        password -> Varchar,
        unique_id -> Varchar,
    }
}
```

当我们希望将您的注意力引向代码块中的特定部分时，相关的行或项目将以粗体显示：

```rs
[default]
exten => s,1,Dial(Zap/1|30)
exten => s,2,Voicemail(u100)
exten => s,102,Voicemail(b100)
exten => i,1,Voicemail(s0)
```

任何命令行输入或输出应如下所示：

```rs
docker-compose up
```

**粗体**：表示新术语、重要单词或您在屏幕上看到的单词。例如，菜单或对话框中的单词以粗体显示。以下是一个示例：“如果我们按下 Postman 中的**发送**按钮两次，在最初的 30 秒内，我们会得到以下输出：”

小贴士或重要注意事项

看起来是这样的。

# 联系我们

我们始终欢迎读者的反馈。

**一般反馈**：如果您对本书的任何方面有疑问，请通过电子邮件发送至 customercare@packtpub.com，并在邮件主题中提及书名。

**勘误**：尽管我们已经尽一切努力确保内容的准确性，但错误仍然可能发生。如果您在这本书中发现了错误，我们将不胜感激，如果您能向我们报告这一点。请访问[www.packtpub.com/support/errata](http://www.packtpub.com/support/errata)并填写表格。

**盗版**：如果您在互联网上以任何形式发现我们作品的非法副本，如果您能提供位置地址或网站名称，我们将不胜感激。请通过电子邮件发送至 copyright@packt.com 并提供材料的链接。

**如果您有兴趣成为作者**：如果您在某个主题上具有专业知识，并且您有兴趣撰写或为书籍做出贡献，请访问[authors.packtpub.com](http://authors.packtpub.com)。

# 分享您的想法

读完《Rust Web Programming - 第二版》后，我们很乐意听听您的想法！请点击此处直接进入此书的亚马逊评论页面并分享您的反馈。

您的评论对我们和科技社区都很重要，并将帮助我们确保我们提供高质量的内容。

# 下载本书的免费 PDF 副本

感谢您购买本书！

您喜欢在路上阅读，但无法携带您的印刷书籍到处走？您的电子书购买是否与您选择的设备不兼容？

别担心，现在，随着每本 Packt 书籍，您都可以免费获得该书的 DRM 免费 PDF 版本。

在任何地方、任何设备上阅读。将您最喜欢的技术书籍中的代码直接搜索、复制并粘贴到您的应用程序中。

优惠远不止于此，您还可以获得独家折扣、时事通讯和每日免费内容的每日电子邮件。

按照以下简单步骤获取好处：

1.  扫描下面的二维码或访问以下链接

![](img/B18722_QR_Free_PDF.jpg)

https://packt.link/free-ebook/9781803234694

1.  提交您的购买证明

1.  就这些！我们将直接将您的免费 PDF 和其他福利发送到您的电子邮件

# 第一部分：Rust Web 开发入门

在 Rust 中进行编码可能具有挑战性。在本部分中，我们涵盖了 Rust 编程的基础知识以及如何在项目中构建具有依赖关系的跨多个文件的程序。在本部分结束时，您将能够使用 Rust 构建应用程序，管理依赖项，导航 Rust 借用检查器，并管理数据集合、结构体、基本设计结构以及引用结构体。

本部分包括以下章节：

+   *第一章*, *Rust 快速入门*

+   *第二章*, *在 Rust 中设计您的 Web 应用程序*
