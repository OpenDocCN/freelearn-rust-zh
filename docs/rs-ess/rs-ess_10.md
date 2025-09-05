# 附录 A.进一步探索

Rust 是一种非常丰富的语言。在这本书中，我们没有讨论 Rust 的每一个概念，也没有在每一个细节上讨论。在这里，我们将讨论我们省略了什么，以及读者可以在哪里找到更多关于这些主题的信息或细节。

# Rust 和标准库的稳定性

Rust 1.0 生产版本承诺提供稳定性；如果您的代码在 Rust 稳定版 1.0 上编译，它将在 Rust 稳定版 1.x 上编译，无需或仅需最小更改。

Rust 的开发遵循一个具有三个发布渠道（夜间、beta 和稳定）的列车模型，并且每六周将发布一个新的稳定版本。生产用户将更倾向于坚持使用稳定分支。每六周，将发布一个新的 beta 版本；这排除了所有不稳定代码，因此您知道，如果您正在使用 beta 或稳定版本，您的代码将继续编译。同时，现有的 beta 分支将被提升为稳定发布。如果您想要最新的更改和新增功能，则使用夜间渠道；它包括可能以向后不兼容的方式更改的不稳定特性和库。

标准库中的绝大多数功能现在都已稳定。如需深入了解，请参阅[`doc.rust-lang.org/std/`](http://doc.rust-lang.org/std/)上的文档。

# Crates 的生态系统

通常有一个趋势是将较少使用或更实验性的**应用程序编程接口**（**APIs**）从语言和标准库中移出，放入它们自己的 crates 中。Rust 的 crates 生态系统不断增长，您可以在[`crates.io/`](https://crates.io/)找到，截至撰写本文时（2015 年 5 月）已有超过 2,000 个 crates。

在*Awesome Rust* ([`github.com/kud1ing/awesome-rust`](https://github.com/kud1ing/awesome-rust))，您可以找到经过精选的 Rust 项目列表。该网站仅包含有用且稳定的项目，并指出它们是否在最新的 Rust 版本中编译。此外，值得搜索*Rust Kit* ([`rustkit.io/`](http://rustkit.io/))，以及位于[`www.rust-ci.org/projects/`](http://www.rust-ci.org/projects/)的 Rust-CI 仓库。

通常，当您开始一个需要特定功能的项目时，建议您搜索已经可用的 crates。很可能已经存在一个符合您需求的 crates，或者您可能可以找到一些可用的起始代码，您可以在其基础上构建您确切需要的东西。

# 其他学习 Rust 的资源

本书几乎涵盖了所谓的“书” ([`doc.rust-lang.org/book/`](http://doc.rust-lang.org/book/)) 中的所有主题，有时甚至超越了它。尽管如此，Rust 网站上的“书”仍然是一个寻找最新信息的良好资源，同时还有 Rust 代码示例的优秀集合，可以在 Rust 主页上的“更多示例”链接找到，也可以通过 [`rustbyexample.com/`](http://rustbyexample.com/) 访问。对于最完整、最深入的信息，请参考 [`doc.rust-lang.org/reference.html`](http://doc.rust-lang.org/reference.html) 上的参考。

在 Reddit ([`www.reddit.com/r/rust`](https://www.reddit.com/r/rust)) 和 Stack Overflow ([`stackoverflow.com/questions/tagged/rust`](https://stackoverflow.com/questions/tagged/rust)) 上提问或关注并评论讨论也可以帮助你。最后但同样重要的是，当你有紧急的 Rust 问题需要解决时，你可以在 [`client01.chat.mibbit.com/?server=irc.mozilla.org&channel=%23rust`](https://client01.chat.mibbit.com/?server=irc.mozilla.org&channel=%23rust) 的 IRC 频道与友好的专家聊天。

你可以在 [`doc.rust-lang.org/nightly/style/`](http://doc.rust-lang.org/nightly/style/) 找到关于 Rust 的编码指南资源。

*24 天 Rust 学习* 是 Zbigniew Siciarz 推荐的一系列关于 Rust 高级主题的精彩文章；你可以在 [`siciarz.net/24-days-of-rust-conclusion/`](https://siciarz.net/24-days-of-rust-conclusion/) 查看索引。

## 文件和数据库

标准库提供了 `std::io::fs` 模块用于文件系统操作：

+   如果你必须处理 **逗号分隔值**（**CSV**）文件，可以使用可用的 crate，例如 `simple_csv`、`csv` 或 `xsv`。在 [`siciarz.net/24-days-of-rust-csv/`](https://siciarz.net/24-days-of-rust-csv/) 的文章可以帮你入门。

+   对于处理 **JSON** 文件，可以使用 `rustc-serialize` 或 `json_macros` 这样的 crate；开始时可以阅读 [`siciarz.net/24-days-of-rust-working-json/`](https://siciarz.net/24-days-of-rust-working-json/) 上的信息。

+   对于 **XML** 格式，有许多可能性，例如 `rust-xml` 和 `xml-rs` crate。

对于数据库，有可用于以下技术的 crate：

+   SQLite3（`rust-sqlite` crate）

+   **PostgreSQL**（`postgres` 和 `r2d2_postgres` crate）；你可以通过 [`siciarz.net/24-days-of-rust-postgres/`](https://siciarz.net/24-days-of-rust-postgres/) 开始使用它。

+   **MySQL**（`mysql` crate）

+   对于 **MongoDB**，有由 MongoDB 开发者构建的 `mongo` crate；更多关于这个的信息，请访问 [`blog.mongodb.org/post/56426792420/introducing-the-mongodb-driver-for-the-rust`](http://blog.mongodb.org/post/56426792420/introducing-the-mongodb-driver-for-the-rust)。

+   对于**Redis**，有`redis`、`redis-rs`或`rust-redis`的 crate；查看[`siciarz.net/24-days-of-rust-redis/`](https://siciarz.net/24-days-of-rust-redis/)以获取快速入门信息。

+   如果你感兴趣的是**对象关系映射器**（**ORM**）框架，请查看 deuterium crate。

## 图形和游戏

它的高性能和底层能力使 Rust 成为图形和游戏领域的理想选择。搜索图形会揭示 OpenGL（带有`gl-rs`、`glfw-sys`包）、Core Graphics（带有`gfx`、`gdk`包）和其他的绑定。

在游戏方面，有 Piston 和 chipmunk 2D 的游戏引擎以及 SDL1、SDL2 和 Allegro5 的绑定。一个简单 3D 游戏引擎的 crate 是`kiss3d`。存在许多物理（`ncollide`）和数学（`nalgebra`和`cgmath-rs`）crate，这些 crate 在这里可能很有用。

## Web 开发

可以在[`arewewebyet.com/`](http://arewewebyet.com/)找到该领域的总体状况概述。目前，用于开发 HTTP 应用程序的最先进和稳定的 crate 是 hyper。它速度快，包含 HTTP 客户端和服务器，可以构建复杂的 Web 应用程序。要开始使用它，请阅读入门文章[`siciarz.net/24-days-of-rust-hyper/`](https://siciarz.net/24-days-of-rust-hyper/)。

基于 hyper 构建的 HTTP 客户端库有`rust-request`和`rest_client`。一个新的名为 teepee 的 Rust HTTP Toolkit 项目正在兴起（[http://teepee.rs/](http://teepee.rs/））。它看起来很有前途，但在撰写本书时还处于起步阶段。

对于 Web 框架，最佳可用的项目是 iron。如果你只需要一个轻量级的微 Web 框架，rustful 可能是你的选择。如果你需要一个**表示状态转移**（**REST**）框架，选择 rustless。另一个有用的 Web 框架，目前仍在积极开发中，是 nickel（[http://nickel.rs/](http://nickel.rs/））。

当然，你绝对不能忽视新兴的新 servo 浏览器！

此外，还存在许多其他类别的 crate，如函数式编程和嵌入式编程（[http://spin.atomicobject.com/2015/02/20/rust-language-c-embedded/](http://spin.atomicobject.com/2015/02/20/rust-language-c-embedded/））、数据结构、图像处理（`image` crate）、音频、压缩、编码和加密（`rust-crypto`和`crypto`）、正则表达式、解析、哈希、工具、测试、模板引擎等等。你可以查看 Rust-CI 仓库或 Awesome Rust 汇编；你可以参考*crate 生态系统*部分中的链接，以了解可用的内容。Zinc（[http://zinc.rs/](http://zinc.rs/））是一个使用 Rust 编写处理器代码栈的项目示例（目前为 ARM）。

这本书中关于 Rust 的旅程就此结束。我们希望您享受阅读它如同我们享受撰写它一样。您现在有了坚实的基础，可以开始使用 Rust 进行开发了。我们也希望这个简要概述向您展示了为什么 Rust 是软件开发界的一颗新星，并且您会在您的项目中使用它。加入 Rust 社区，开始发挥您的编程才能。也许我们会在 Rust（非）宇宙中再次相遇。
