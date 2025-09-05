# 第二章：*第二章*: 构建我们的第一个 Rocket Web 应用程序

在本章中，我们将探讨 Rocket，这是一个使用 Rust 编程语言创建 Web 应用程序的框架。在我们使用 Rocket 框架创建第一个 Web 应用程序之前，我们将对 Rocket 有一些了解。之后，我们将学习如何配置我们的 Rocket Web 应用程序。最后，在本章末尾，我们将探讨如何获取这个 Web 框架的帮助。

在本章中，我们将涵盖以下主要主题：

+   介绍 Rocket – 使用 Rust 语言编写的 Web 框架

+   创建我们的第一个 Rocket Web 应用程序

+   配置我们的 Rocket Web 应用程序

+   获取帮助

# 技术要求

对于本章和随后的章节，你需要安装在 *第一章* 中提到的要求，即 *介绍 Rust 语言* 和 Rust 工具链。如果你还没有安装 Rust 编译器工具链，请遵循 *第一章* 中提到的安装指南，即 *介绍 Rust 语言*。此外，安装一个带有 Rust 扩展的文本编辑器以及 Rust 工具如 `rustfmt` 或 `clippy` 会很有帮助。如果你还没有安装文本编辑器，可以使用带有 `rust-analyzer` 扩展的 Visual Studio Code 等开源软件。由于我们将向我们要创建的应用程序发出 HTTP 请求，你应该安装一个网络浏览器或其他 HTTP 客户端。

最后，Rocket 框架有几个版本发布，它们都略有不同。我们只会讨论 Rocket *0.5.0* 版本。如果你计划使用 Rocket 的其他版本，请不要担心，因为术语和概念几乎相同。你可以使用本章 *获取帮助* 部分中提到的 API 文档来查看你使用的 Rocket 框架版本的正确文档。

本章的代码可以在 [`github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter02`](https://github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter02) 找到。

# 介绍 Rocket – 使用 Rust 语言编写的 Web 框架

Rocket Web 框架的开发始于 2016 年 Sergio Benitez 的一个项目。长期以来，它一直使用大量的 Rust 宏系统来简化开发过程；因此，直到最近 2021 年，才可以使用稳定的 Rust 编译器。在开发过程中，Rust 添加了 async/await 功能。Rocket 开始整合 async/await，直到 2021 年关闭了相关的问题跟踪器。

Rocket 是一个相当简单的 Web 框架，没有很多花哨的功能，例如数据库 **对象关系映射**（ORM）或邮件系统。程序员可以使用其他 Rust 包扩展 Rocket 的功能，例如，通过添加第三方日志记录或连接到内存存储应用程序。

## Rocket 中的 HTTP 请求生命周期

处理 HTTP 请求是网络应用的一个基本组成部分。Rocket 网络框架将传入的 HTTP 请求视为一个生命周期。网络应用首先要做的是检查并确定哪个或哪些函数能够处理传入的请求。这部分称为**路由**。例如，如果有一个 *GET/something* 传入请求，Rocket 将检查所有已注册的路由以查找匹配项。

在路由之后，Rocket 将对传入请求进行**验证**，以检查与第一个函数中声明的类型和守卫是否匹配。如果结果不匹配且下一个处理传入请求的函数可用，Rocket 将继续对下一个函数进行验证，直到没有更多函数可以处理该传入请求。

验证后，Rocket 将根据程序员在函数体中编写的代码进行**处理**。例如，程序员使用请求中的数据创建一个 SQL 查询，将查询发送到数据库，从数据库检索结果，并使用结果创建 HTML。

Rocket 最后将返回一个**响应**，其中包含 HTTP 状态、头和主体。请求生命周期随后完成。

总结一下，Rocket 请求的生命周期是**路由 → 验证 → 处理 → 响应**。接下来，让我们讨论 Rocket 应用程序的启动方式。

## Rocket 发射序列

就像现实生活中的火箭一样，我们首先构建 Rocket 应用程序。在构建过程中，我们将**路由**（处理传入请求的函数）挂载到 Rocket 应用程序上。

在构建过程中，Rocket 应用程序还管理各种**状态**。状态是一个可以在路由处理程序中访问的 Rust 对象。例如，假设我们想要一个将事件发送到日志服务器的记录器。我们在构建 Rocket 应用程序时初始化记录器对象，当有传入请求时，我们可以在请求处理程序中使用已管理的记录器对象。

在构建过程中，我们还可以附加**整流罩**。在许多网络框架中，通常有一个中间件组件，用于过滤、检查、授权或修改传入的 HTTP 请求或响应。在 Rocket 框架中，提供中间件功能的函数被称为整流罩。例如，如果我们想为每个 HTTP 请求创建一个用于审计的**通用唯一标识符**（**UUID**），我们首先创建一个生成随机 UUID 并将其附加到请求 HTTP 头部的整流罩。我们还将整流罩附加到生成的 HTTP 响应中。接下来，我们将它附加到 Rocket 上。这个整流罩将拦截传入的请求和响应并进行修改。

在 Rocket 应用程序构建并准备就绪后，下一步当然是启动它。太好了，发射成功！现在，我们的 Rocket 应用程序已处于运行状态，准备好处理传入的请求。

现在我们对 Rocket 应用程序有了总体了解，让我们尝试创建一个简单的 Rocket 应用程序。

# 创建我们的第一个 Rocket Web 应用程序

在本节中，我们将创建一个非常简单的 Web 应用程序，它只处理一个 HTTP 路径。按照以下步骤创建我们的第一个 Rocket Web 应用程序：

1.  使用 Cargo 创建一个新的 Rust 应用程序：

    ```rs
    cargo new 01hello_rocket --name hello_rocket
    ```

我们在一个名为 `01hello_rocket` 的文件夹中创建了一个名为 `hello_rocket` 的应用程序。

1.  之后，让我们修改 `Cargo.toml` 文件。在 `[dependencies]` 之后添加以下行：

    ```rs
    rocket = "0.5.0-rc.1"
    ```

1.  在 `src/main.rs` 文件顶部追加以下行：

    ```rs
    #[macro_use]
    extern crate rocket;
    ```

这里，我们正在告诉 Rust 编译器通过使用 `#[macro_use]` 属性来使用 Rocket crate 中的宏。我们可以省略使用该属性，但那样就意味着我们必须为将要使用的每个宏指定 `use`。

1.  添加以下行以告知编译器我们正在使用来自 Rocket crate 的定义：

    ```rs
    use rocket::{Build, Rocket};
    ```

1.  之后，让我们创建我们的第一个 HTTP 处理器。在前面几行之后添加以下行：

    ```rs
    #[get("/")]
    fn index() -> &'static str {
        "Hello, Rocket!"
    }
    ```

这里，我们定义了一个返回 `str` 引用的函数。Rust 语言中的 `'a` 表示一个变量具有 `'a` 生命周期。引用的生存期取决于许多因素。我们将在 *第九章* *显示用户帖子* 中讨论这些内容，当我们更深入地讨论对象作用域和生命周期时。但是，`'static` 符号是特殊的，因为它意味着它将持续到应用程序仍然存活。我们还可以看到返回值是 `"Hello, Rocket"`，因为它是在最后一行，我们没有在末尾放置分号。

但是，`#[get("/")]` 属性是什么？记得之前我们使用过 `#[macro_use]` 属性吗？`rocket::get` 属性是一个宏属性，它指定了函数处理的 HTTP 方法、路由、HTTP 路径和参数。我们可以使用七个特定方法的路由属性：`get`、`put`、`post`、`delete`、`head`、`options` 和 `patch`。所有这些都与它们各自的 HTTP 方法名称相对应。

我们还可以使用替代宏来指定路由处理器，通过用以下内容替换属性宏：

```rs
#[route(GET, path = "/")]
```

1.  接下来，删除 `fn main()` 函数并添加以下行：

    ```rs
    #[launch]
    fn rocket() -> Rocket<Build> {
        rocket::build().mount("/", routes![index])
    }
    ```

我们创建了一个函数来生成 `main` 函数，因为我们使用了 `#[launch]` 属性。在函数内部，我们构建了 Rocket 并将具有 `index` 函数的路由挂载到 `"/"` 路径上。

1.  让我们尝试运行 `hello_rocket` 应用程序：

    ```rs
    > cargo run
       Updating crates.io index
       Compiling proc-macro2 v1.0.28
    ...
       Compiling hello_rocket v0.1.0 (/Users/karuna/
       Chapter02/01hello_rocket)
    Finished dev [unoptimized + debuginfo] target(s) 
        in 1m 39s
         Running `target/debug/hello_rocket`
    ![](img/01.png) Configured for debug.
       >> address: 127.0.0.1
       >> port: 8000
       >> workers: 8
       >> ident: Rocket
       >> keep-alive: 5s
    >> limits: bytes = 8KiB, data-form = 2MiB, file = 
    1MiB, form = 32KiB, json = 1MiB, msgpack = 1MiB, 
       string = 8KiB
       >> tls: disabled
       >> temp dir: /var/folders/gh/
       kgsn28fn3hvflpcfq70x6f1w0000gp/T/
       >> log level: normal
       >> cli colors: true
    >> shutdown: ctrlc = true, force = true, signals = 
       [SIGTERM], grace = 2s, mercy = 3s
    ![](img/04.png)  Routes:
       >> (index) GET /
    ![](img/02.png) Fairings:
       >> Shield (liftoff, response, singleton)
    ![](img/05.png) Shield:
       >> X-Content-Type-Options: nosniff
       >> X-Frame-Options: SAMEORIGIN
       >> Permissions-Policy: interest-cohort=()
    ![](img/03.png) Rocket has launched from http://127.0.0.1:8000
    ```

您可以看到应用程序打印了应用程序配置，例如保持活动超时时间、请求大小限制、日志级别、路由等，这些通常在 HTTP 服务器中看到，到终端。之后，应用程序打印了 Rocket 的各个部分。我们创建了一个单路由 `index` 函数，它处理 `GET /`。

然后，还有默认的内置公平性（Fairing），**Shield**。Shield 通过默认向所有响应注入 HTTP 安全性和隐私头来实现。

我们还看到应用程序已成功启动，现在正在`127.0.0.1`地址和`8000`端口上接受请求。

1.  现在，让我们测试应用程序是否真的在接收请求。您可以使用 Web 浏览器或任何 HTTP 客户端，因为这是一个非常简单的请求，但如果您使用命令行，请不要停止运行的应用程序；打开另一个终端：

    ```rs
    > curl http://127.0.0.1:8000
    Hello, Rocket!
    ```

您可以看到应用程序响应得非常完美。您还可以看到应用程序的日志：

```rs
GET /:
   >> Matched: (index) GET /
   >> Outcome: Success
   >> Response succeeded.
```

1.  现在，让我们尝试请求我们应用程序中不存在的东西：

    ```rs
    > curl http://127.0.0.1:8000/somepath
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="utf-8">
        <title>404 Not Found</title>
    </head>
    <body align="center">
        …
    </body>
    </html>
    ```

现在，让我们看看应用程序终端输出中发生了什么：

```rs
GET /somepath:
   >> No matching routes for GET /somepath.
   >> No 404 catcher registered. Using Rocket default.
   >> Response succeeded.
```

我们现在知道 Rocket 已经为`404`状态情况提供了默认的处理程序。

让我们回顾一下 Rocket 的生命周期，*路由 → 验证 → 处理 → 响应*。对 http://127.0.0.1:8000/的第一次请求是成功的，因为应用程序找到了`GET /`路由的处理程序。由于我们没有在应用程序中创建任何验证，该函数随后执行了一些非常简单的处理，返回了一个字符串。Rocket 框架已经为`&str`实现了`Responder`特质，因此它创建并返回了适当的 HTTP 响应。对`/somepath`的另一个请求没有通过路由部分，我们没有创建任何错误处理程序，因此 Rocket 应用程序为这个请求返回了一个默认的错误处理程序。

尝试在浏览器中打开它，并使用开发者工具检查响应，或者再次以详细模式运行`curl`命令以查看完整的 HTTP 响应，`curl -v http://127.0.0.1:8000/`和`curl -v http://127.0.0.1:8000/somepath`：

```rs
$ curl -v http://127.0.0.1:8000/somepath
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to 127.0.0.1 (127.0.0.1) port 8000 (#0)
> GET /somepath HTTP/1.1
> Host: 127.0.0.1:8000
> User-Agent: curl/7.64.1
> Accept: */*
> 
< HTTP/1.1 404 Not Found
< content-type: text/html; charset=utf-8
< server: Rocket
< x-frame-options: SAMEORIGIN
< x-content-type-options: nosniff
< permissions-policy: interest-cohort=()
< content-length: 383
< date: Fri, 17 Aug 1945 03:00:00 GMT
< 
<!DOCTYPE html>
...
* Connection #0 to host 127.0.0.1 left intact
</html>* Closing connection 0
```

您可以看到对`/`的访问完美无缺，而对`/somepath`的访问返回了一个带有`404`状态和某些 HTML 内容的 HTTP 响应。还有一些默认的隐私和安全 HTTP 头，这些是由 Shield fairing 注入的。

恭喜！您刚刚创建了一个由 Rocket 驱动的第一个 Web 应用程序。您刚刚构建的是一个常规的 Rocket Web 应用程序。接下来，让我们将其修改为异步 Web 应用程序。

## 异步应用程序

什么是异步应用程序？让我们假设我们的 Web 应用程序正在向数据库发送查询。在等待响应的几毫秒内，我们的应用程序线程只是在做无用功。对于单个用户和单个请求来说，这并不是问题。异步应用程序是一种允许处理器在存在阻塞任务（如等待数据库响应）时执行其他任务的应用程序。我们将在稍后详细讨论这个问题；现在，我们只想将我们的应用程序转换为异步应用程序。

让我们修改之前创建的应用程序，使其变为异步。您可以在`02hello_rocket_async`文件夹中找到示例：

1.  删除`use rocket::{Build, Rocket};`这一行，因为我们不会使用它。

1.  然后，让我们在`fn index()`之前添加`async`关键字。

1.  将`#[launch]`属性替换为`#[rocket::main]`。这是为了表示这个函数将成为我们应用程序中的主函数。

1.  添加`async`关键字，并将`fn launch()`重命名为`fn main()`。

1.  我们也不希望主函数返回任何内容，所以使用`remove -> Rocket<build>`。

1.  在调用`mount`之后添加`.launch().await;`。

最终的代码应该看起来像这样：

```rs
#[macro_use]
```

```rs
extern crate rocket;
```

```rs
#[get("/")]
```

```rs
async fn index() -> &'static str {
```

```rs
    "Hello, Rocket!"
```

```rs
}
```

```rs
#[rocket::main]
```

```rs
async fn main() {
```

```rs
    rocket::build().mount("/", routes![
```

```rs
    index]).launch().await;
```

```rs
}
```

使用*Ctrl* + *C*命令停止旧版本在服务器上的运行。你应该会看到类似以下的内容：

```rs
Warning: Received SIGINT. Requesting shutdown.
Received shutdown request. Waiting for pending I/O...
```

我们目前在`"/"`处理程序中没有任何阻塞任务，所以我们不会看到任何明显的收益。现在我们已经创建了我们的应用程序，让我们在下一节中对其进行配置。

# 配置我们的 Rocket 网络应用程序

让我们学习如何配置我们的 Rocket 网络应用程序，从为不同情况设置不同的配置文件开始。然后，我们将使用`Rocket.toml`文件进行配置。最后，我们将学习如何使用环境变量来配置我们的应用程序。

## 以不同配置文件启动 Rocket 应用程序

让我们运行我们的同步应用程序服务器，不使用发布标志，然后在另一个终端中，让我们看看我们是否可以对其进行基准测试：

1.  首先，让我们使用`cargo install benchrs`安装应用程序。没错，你也可以使用 Cargo 来安装应用程序！在你的终端中，你可以使用非常优秀的 Rust 程序，例如`ripgrep`，这是在代码中搜索字符串最快的应用程序之一。

如果你想调用 Cargo 安装的应用程序，你可以使用完整路径，或者如果你使用的是基于 Unix 的终端，将其添加到你的终端路径中。将以下行追加到你的`~/.profile`或任何将被终端加载的其他配置文件中：

```rs
export PATH="$HOME/.cargo/bin:$PATH"
```

如果你使用 Windows，Rustup 应该已经将 Cargo 的`bin`文件夹添加到你的路径中了。

1.  对运行中的应用程序进行基准测试：

    ```rs
    benchrs -c 30 -n 3000 -k http://127.0.0.1:8000/
    07:59:04.934 [INFO] benchrs:0.1.8
    07:59:04.934 [INFO] Spawning 8 threads
    07:59:05.699 [INFO] Ran in 0.7199552s 30 connections, 3000 requests with avg request time: 6.5126667ms, median: 6ms, 95th percentile: 11ms and 99th percentile: 14ms
    ```

你可以看到我们的应用程序在`0.7199552`秒内处理了大约 3,000 个请求。如果我们与其他重型框架进行比较，这是一个简单的应用程序的不错值。之后，现在暂时停止应用程序。

1.  现在，让我们再次运行应用程序，但这次是在发布模式下。你还记得如何在前一章中这样做吗？

    ```rs
    > cargo run –release
    ```

然后，它应该编译我们的应用程序以发布并运行它。

1.  当应用程序准备好再次接受请求后，在另一个终端中再次运行基准测试：

    ```rs
    $ benchrs -c 30 -n 3000 -k http://127.0.0.1:8000/
    08:12:51.388 [INFO] benchrs:0.1.8
    08:12:51.388 [INFO] Spawning 8 threads
    08:12:51.513 [INFO] Ran in 0.07942524s 30 connections, 3000 requests with avg request time: 0.021333333ms, median: 0ms, 
    5th percentile: 0ms and 99th percentile: 1ms
    ```

再次，结果非常令人印象深刻，但这里发生了什么？总的基准时间现在大约是 0.08 秒，比之前的总基准时间快了几乎 10 倍。

要了解速度增加的原因，我们需要了解 Rocket **配置文件**。配置文件是为一组配置所赋予的名称。

Rocket 应用程序有两个元配置文件：**default**和**global**。默认配置文件包含所有默认配置值。如果我们创建一个配置文件而没有设置其配置值，则将使用默认配置的值。至于全局配置文件，如果我们设置了配置值，则它将覆盖配置文件中设置的值。

除了这两个元配置文件之外，Rocket 框架还提供了两个配置。当以发布模式运行或编译应用程序时，Rocket 将使用**release**配置文件，而在调试模式运行或编译时，Rocket 将选择**debug**配置文件。以发布模式运行应用程序将显然生成一个优化的可执行二进制文件，但应用程序本身还有其他优化。例如，您将在应用程序输出中看到差异。默认情况下，调试配置文件会在终端上显示输出，但默认情况下，发布配置文件不会在终端输出中显示任何请求。

我们可以为配置文件创建任何名称，例如，`development`、`test`、`staging`、`sandbox`或`production`。使用对您的开发过程有意义的任何名称。例如，在一个用于 QA 测试的机器上，您可能希望将配置文件命名为`testing`。

要选择我们想要使用的配置文件，我们可以在环境变量中指定它。在命令行中使用`ROCKET_PROFILE=profile_name cargo run`。例如，您可以编写`ROCKET_PROFILE=profile_name cargo run –release`。

现在我们知道了如何以特定配置启动应用程序，让我们学习如何创建配置文件并配置 Rocket 应用程序。

## 配置 Rocket Web 应用程序

Rocket 有一种配置 Web 应用程序的方法。Web 框架使用`Provider`特质。有人可以创建一个从文件中读取 JSON 的类型，并为该类型实现`Provider`特质。该类型然后可以被使用 figment crate 作为配置源的应用程序消费。

在 Rust 中，有一个约定，如果结构实现了标准库特质`std::default::Default`，则使用默认值初始化结构。该特质被编写如下：

```rs
pub trait Default {
```

```rs
    fn default() -> Self;
```

```rs
}
```

例如，一个名为`StructName`的结构，实现了`Default`特质，将被调用为`StructName::default()`。Rocket 有一个实现`Default`特质的`rocket::Config`结构。然后使用默认值来配置应用程序。

如果您查看`rocket::Config`结构的源代码，它被编写如下：

```rs
pub struct Config {
```

```rs
    pub profile: Profile,
```

```rs
    pub address: IpAddr,
```

```rs
    pub port: u16,
```

```rs
    pub workers: usize,
```

```rs
    pub keep_alive: u32,
```

```rs
    pub limits: Limits,
```

```rs
    pub tls: Option<TlsConfig>,
```

```rs
    pub ident: Ident,
```

```rs
    pub secret_key: SecretKey,
```

```rs
    pub temp_dir: PathBuf,
```

```rs
    pub log_level: LogLevel,
```

```rs
    pub shutdown: Shutdown,
```

```rs
    pub cli_colors: bool,
```

```rs
    // some fields omitted
```

```rs
}
```

如您所见，有诸如`address`和`port`之类的字段，这些字段显然将决定应用程序的行为。如果您进一步检查源代码，您可以看到`Config`结构的`Default`特质实现。

Rocket 还有一些 figment 提供者，当我们在应用程序中使用`rocket::build()`方法时，会覆盖默认的`rocket::Config`值。

第一个片段提供者从 `Rocket.toml` 文件中读取，或者如果我们使用 `ROCKET_CONFIG` 环境变量运行应用程序，则是我们指定的文件。如果我们指定 `ROCKET_CONFIG`（例如，`ROCKET_CONFIG=our_config.toml`），它将在应用程序根目录中搜索 `our_config.toml`。如果应用程序找不到该配置文件，则应用程序将在父文件夹中查找，直到达到文件系统的根。如果我们指定绝对路径，例如，`ROCKET_CONFIG=/some/directory/our_config.toml`，则应用程序将仅在指定位置搜索该文件。

第二个片段提供者从环境变量中读取值。我们将在稍后看到如何做到这一点，但首先，让我们尝试使用 `Rocket.toml` 文件来配置 Rocket 应用程序。

## 使用 Rocket.toml 配置 Rocket 应用程序

我们首先需要了解的是可以在配置文件中使用的键列表。这些是我们可以在配置文件中使用的键：

+   `address`: 应用程序将在该地址提供服务。

+   `port`: 应用程序将在该端口上提供服务。

+   `workers`: 应用程序将使用此数量的线程。

+   `ident`: 如果我们指定 `false`，则应用程序不会在服务器 HTTP 标头中放置身份；如果我们指定 `string`，则应用程序将使用它作为服务器 HTTP 标头的身份。

+   `keep_alive`: 以秒为单位保持连接的超时时间。使用 `0` 来禁用 `keep_alive`。

+   `log_level`: 记录的最大级别（关闭/正常/调试/关键）。

+   `temp_dir`: 存储临时文件的目录路径。

+   `cli_colors`: 在日志中使用颜色和表情符号，或者不使用。这在禁用发布环境中的铃声和哨声很有用。

+   `secret_key`: Rocket 应用程序有一个类型来存储应用程序中的私有 cookies。私有 cookies 由此密钥加密。密钥长度为 256 位，您可以使用如 `openssl rand -base64 32` 这样的工具生成它。由于这是一个重要的密钥，您可能希望将其保存在安全的地方。

+   `tls`: 使用 `tls.key` 和 `tls.certs` 进入您的 TLS（传输层安全性）密钥和证书文件的路径。

+   `limits`: 此配置是嵌套的，用于限制服务器读取大小。您可以将值写入多字节单位，例如 1 MB（兆字节）或 1 MiB（梅吉字节）。有几个默认选项：

    +   `limits.form` – 32 KiB

    +   `limits.data-form` – 2 MiB

    +   `limits.file` – 1 MiB

    +   `limits.string` – 8 KiB

    +   `limits.bytes` – 8 KiB

    +   `limits.json` – 1 MiB

    +   `limits.msgpack` – 1 MiB

+   `shutdown`: 如果一个网络应用程序在处理某些内容时突然终止，正在处理的数据可能会意外损坏。例如，假设一个 Rocket 应用程序正在向数据库服务器发送更新数据的途中，但随后进程被突然终止。结果，数据不一致。此选项配置 Rocket 的平稳关闭行为。与 `limits` 类似，它有几个子配置：

    +   `shutdown.ctrlc` – 应用程序是否忽略 *Ctrl* + *C* 键盘按键？

    +   `shutdown.signals` – 触发关闭的 Unix 信号数组。仅在 Unix 或类 Unix 操作系统上有效。

    +   `shutdown.grace` – 在停止服务器之前完成未完成的服务器 I/O 的秒数。

    +   `shutdown.mercy` – 在停止之前完成未完成的连接 I/O 的秒数。

    +   `shutdown.force` – 指定是否要杀死拒绝合作的进程。

现在我们知道了我们可以使用哪些密钥，让我们尝试配置我们的应用程序。记得我们的应用程序运行在哪个端口上吗？假设现在我们想在端口 `3000` 上运行应用程序。让我们在我们的应用程序根目录中创建一个 `Rocket.toml` 文件：

```rs
[default]
```

```rs
port = 3000
```

现在，再次尝试运行应用程序：

```rs
$ cargo run
...
 Rocket has launched from http://127.0.0.1:3000
```

你可以看到它在工作；我们正在端口 `3000` 上运行应用程序。但是，如果我们想为不同的配置文件运行不同的应用程序配置呢？让我们尝试在 `Rocket.toml` 文件中添加这些行，并以发布模式运行应用程序：

```rs
[release]
```

```rs
port = 9999
$ cargo run --release
...
 Rocket has launched from http://127.0.0.1:9999
```

没错，我们可以为不同的配置文件指定配置。如果我们的选项是嵌套的，我们该怎么办？因为这个文件是一个 `.toml` 文件，我们可以这样写：

```rs
[default.tls]
```

```rs
certs = "/some/directory/cert-chain.pem"
```

```rs
key = "/some/directory/key.pem"
```

或者，我们可以这样写：

```rs
[default]
```

```rs
tls = { certs = "/some/directory/cert-chain.pem", key = "/some/directory/key.pem" }
```

现在，让我们查看具有默认配置的整个文件：

```rs
[default]
```

```rs
address = "127.0.0.1"
```

```rs
port = 8000
```

```rs
workers = 16
```

```rs
keep_alive = 5
```

```rs
ident = "Rocket"
```

```rs
log_level = "normal"
```

```rs
temp_dir = "/tmp"
```

```rs
cli_colors = true
```

```rs
## Please do not use this key, but generate your own with `openssl rand -base64 32`
```

```rs
secret_key = " BCbkLMhRRtYMerGKCcboyD4Mhf6/XefvhW0Wr8Q0s1Q="
```

```rs
[default.limits]
```

```rs
form = "32KiB"
```

```rs
data-form = "2MiB"
```

```rs
file = "1MiB"
```

```rs
string = "8KiB"
```

```rs
bytes = "8KiB"
```

```rs
json = "1MiB"
```

```rs
msgpack = "1MiB"
```

```rs
[default.tls]
```

```rs
certs = "/some/directory/cert-chain.pem
```

```rs
key = "/some/directory/key.pem
```

```rs
[default.shutdown]
```

```rs
ctrlc = true
```

```rs
signals = ["term"]
```

```rs
grace = 5
```

```rs
mercy = 5
```

```rs
force = true
```

尽管我们可以创建包含完整配置的文件，但使用 `Rocket.toml` 的最佳实践是依赖默认值，并且只写入我们真正需要覆盖的内容。

## 用环境变量覆盖配置

在检查 `Rocket.toml` 后，应用程序再次使用环境变量覆盖 `rocket::Config` 值。应用程序将检查 `ROCKET_*` 环境变量的可用性。例如，我们可能定义 `ROCKET_IDENT="Merpay"` 或 `ROCKET_TLS={certs="abc.pem",key="def.pem"}`。如果我们正在进行开发并且有多个团队成员，或者我们不希望在配置文件中存在某些内容并依赖于环境变量，例如，当我们在 Kubernetes Secrets 中存储 `secret_key` 时，这非常有用。在这种情况下，从环境变量中获取 `secret` 值比将其写入 `Rocket.toml` 并提交到源代码版本控制系统更安全。

让我们尝试通过运行应用程序并设置 `ROCKET_PORT=4000` 来覆盖配置：

```rs
$ ROCKET_PORT=4000 cargo run
...
 Rocket has launched from http://127.0.0.1:4000
```

环境变量覆盖是有效的；尽管我们在 `Rocket.toml` 文件中指定了端口 `3000`，但我们正在端口 `4000` 上运行应用程序。当我们配置应用程序连接到数据库时，我们将在 *第四章* *构建、点燃和发射火箭* 中学习如何扩展默认的 `rocket::Config` 以使用自定义配置。现在我们已经学会了如何配置 Rocket 应用程序，让我们找出我们可以在哪里获得 Rocket 网络框架的文档和帮助。

# 获取帮助

使用网络框架获得帮助是至关重要的。在本部分中，我们将了解我们可以在哪里获得 Rocket 框架的帮助和文档。

您可以从 Rocket 框架本身的网站上获得帮助：[`rocket.rs/`](https://rocket.rs/). 在该网站上，有一个如下指南：[`rocket.rs/v0.5-rc/guide/`](https://rocket.rs/v0.5-rc/guide/). 在该页面的左上角，有一个下拉菜单，您可以通过它选择 Rocket 网络框架的先前版本的文档。

在[`api.rocket.rs`](https://api.rocket.rs)上，您可以查看 API 的文档，但遗憾的是，这份文档是针对 Rocket 网络框架的 master 分支的。如果您想查看您框架版本的 API 文档，您必须手动搜索，例如`https://api.rocket.rs/v0.3/rocket/`或`https://api.rocket.rs/v0.4/rocket/`。

为 Rocket 框架生成离线文档的另一种方法是。从官方仓库[`github.com/SergioBenitez/Rocket`](https://github.com/SergioBenitez/Rocket)下载 Rocket 的源代码。然后，在文件夹内输入`./scripts/mk-docs.sh`来运行 shell 脚本。生成的文档很有用，因为有时其中的一些内容与[`api.rocket.rs`](https://api.rocket.rs)上的内容不同。例如，`rocket::Config`的定义及其在代码中的默认值与 API 文档中的略有不同。

# 摘要

在本章中，我们简要了解了 Rocket 的启动序列和请求生命周期。我们还创建了一个非常简单的应用程序，并将其转换为异步应用程序。之后，我们学习了 Rocket 的配置，使用`Rocket.toml`编写配置，并使用环境变量覆盖它。最后，我们学习了在哪里可以找到 Rocket 框架的文档。

现在我们已经使用 Rocket 网络框架创建了一个简单的应用程序，让我们在下一章中进一步讨论请求和响应。
