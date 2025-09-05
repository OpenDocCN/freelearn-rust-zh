# 网络安全编程

自从 Ruby 编程语言的流行框架 Rails 以来，创建后端网络服务似乎成了动态类型语言的领域。这一趋势随着 Python 和 JavaScript 作为这些任务的主要语言的兴起而得到加强。毕竟，这些技术的本质使得创建这些服务特别快速，对服务的更改（例如，JSON 响应中的新字段）也简单易行。对我们许多人来说，回归静态类型进行网络服务感觉有些奇怪；毕竟，要启动“某种东西”需要更长的时间。

然而，这也有代价：如今，许多服务部署在云端，这意味着采用了按需付费的模式，并且（实际上）具有无限的扩展性。由于——最值得注意的是——Python 以其执行速度而闻名，我们现在可以在云服务提供商的账单上看到这种开销的成本。10% 的更快执行时间意味着在相同硬件和相同质量水平上服务 10% 的更多客户（例如，响应时间）。同样，较小的设备从较低的资源使用中受益，这转化为更快的软件，因此消耗更少的能源。Rust 作为一种系统编程语言，在设计时考虑了零开销，在速度或效率等方面与 C 语言非常接近。因此，对于大量使用的网络服务来说，在 Rust 中编写关键部分是合理的，Dropbox 就曾著名地采取这一举措来提高其服务质量并节省成本。

Rust 是一种非常适合网络的语言，在本章中，我们将探讨使用在许多不同应用程序中使用的框架来创建常规 RESTful API。您可以期待学习以下内容：

+   设置网络服务器

+   设计 RESTful API

+   处理 JSON 有效载荷

+   网络错误处理

+   渲染 HTML 模板

+   使用 ORM 将数据保存到数据库

+   使用 ORM 运行高级查询

+   网络上的身份验证

# 设置网络服务器

在过去几年中，网络服务器已经发生了变化。早期网络应用程序通常部署在某种类型的网络服务器应用程序之后，如 Apache Tomcat ([`tomcat.apache.org/`](http://tomcat.apache.org/))、IIS ([`www.iis.net/`](https://www.iis.net/)) 和 nginx ([`www.nginx.com/`](https://www.nginx.com/))，但现在更常见的是将服务部分嵌入到应用程序中。这不仅对运维人员来说更容易，还允许开发者对整个应用程序有更紧密的控制。让我们看看如何开始并设置一个基本的静态网络服务器。

# 准备工作

让我们使用`cargo new static-web`设置一个 Rust 二进制项目。由于我们将在本地端口`8081`上提供服务，请确保该端口可访问。在新建的项目文件夹中，我们需要一个额外的文件夹`static/`，你可以在这里放置一个有趣的`.jpg`图像并提供服务。我们将假设这个图像叫做`foxes.jpg`。

最后，使用 VS Code 打开整个目录。

# 如何做到这一点...

我们将在几个步骤中设置和运行我们自己的 Web 服务器：

1.  首先打开`src/main.rs`，让我们添加一些代码。我们将从导入和简单的索引处理器开始，逐步向下到`main`函数：

```rs
#[macro_use]
extern crate actix_web;

use actix_web::{web, App, middleware, HttpServer, Responder, Result};
use std::{env};
use actix_files as fs;

fn index() -> Result<fs::NamedFile> {
    Ok(fs::NamedFile::open("static/index.html")?)
}
```

1.  然而，这不会是唯一的请求处理器，所以让我们添加一些更多，看看请求处理是如何工作的：

```rs
fn path_parser(info: web::Path<(String, i32)>) -> impl Responder {
    format!("You tried to reach '{}/{}'", info.0, info.1)
}

fn rust_cookbook() -> impl Responder {
    format!("Welcome to the Rust Cookbook")
}

#[get("/foxes")]
fn foxes() -> Result<fs::NamedFile> {
    Ok(fs::NamedFile::open("static/foxes.jpg")?)
}
```

1.  缺少的是`main`函数。这个`main`函数启动服务器并附加我们在上一步中创建的服务：

```rs
fn main() -> std::io::Result<()> {
    env::set_var("RUST_LOG", "actix_web=debug");
    env_logger::init();
    HttpServer::new(
        || App::new()
            .wrap(middleware::Logger::default())
            .service(foxes)
            .service(web::resource("/").to(index))
            .service(web::resource("/welcome").to(rust_cookbook))  
            .service(web::resource("/{path}/{id}").to(path_parser)))
        .bind("127.0.0.1:8081")?
        .run()
}
```

1.  在第一个处理器中，我们提到了一个静态的`index.html`处理器，但我们还没有创建它。向一个新文件添加一个简单的`marquee`输出，并将其保存为`static/index.html`：

```rs
<html>
    <body>
        <marquee><h1>Hello World</h1></marquee>
    </body>
</html>
```

1.  我们还需要做的一件重要的事情是调整`Cargo.toml`。按照以下方式在`Cargo.toml`中声明依赖项：

```rs
[dependencies]
actix-web = "1"
env_logger = "0.6"
actix-files = "0"
```

1.  使用终端执行`cargo run`并运行代码，然后在浏览器窗口中打开`http://localhost:8081/`、`http://localhost:8081/welcome`、`http://localhost:8081/foxes`和`http://localhost:8081/somethingarbitrary/10`：

```rs
$ cargo run
   Compiling autocfg v0.1.4
   Compiling semver-parser v0.7.0
   Compiling libc v0.2.59
[...]
   Compiling static-web v0.1.0 (Rust-Cookbook/Chapter08/static-web)
    Finished dev [unoptimized + debuginfo] target(s) in 1m 51s
     Running `target/debug/static-web`
```

这是`http://localhost:8081`的输出，由`index`函数处理：

![图片](img/a7d7d636-1c6d-4374-9972-f8b2bcb88743.png)

您也可以在`http://localhost:8081/welcome`调用`welcome`处理器：

![图片](img/fd5b036f-2750-4554-a671-803761ba3eb0.png)

我们的静态处理器在`http://localhost:8081/foxes`返回柏林 Mozilla 办公室的照片：

![图片](img/a78dedb7-20a5-4320-ba71-134a7ac1947c.png)

最后，我们添加了一个路径处理器，它解析路径中的字符串和整数，只是为了返回这些值：

![图片](img/dd845229-2c6e-468d-a19a-3ec92867e03d.png)

为了验证请求确实被我们的 Web 服务器处理，你应该查看在运行`cargo run`的终端中的日志输出中的单个请求：

```rs
[...]
    Finished dev [unoptimized + debuginfo] target(s) in 1m 51s
     Running `target/debug/static-web`
[2019-07-17T06:20:27Z INFO actix_web::middleware::logger] 127.0.0.1:35358 "GET / HTTP/1.1" 200 89 "-" "Mozilla/5.0 (X11; Fedora; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86 Safari/537.36" 0.004907
[2019-07-17T06:21:58Z INFO actix_web::middleware::logger] 127.0.0.1:36154 "GET /welcome HTTP/1.1" 200 28 "-" "Mozilla/5.0 (X11; Fedora; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86 Safari/537.36" 0.000844
^[[B[2019-07-17T06:22:34Z INFO actix_web::middleware::logger] 127.0.0.1:36476 "GET /somethingarbitrary/10 HTTP/1.1" 200 42 "-" "Mozilla/5.0 (X11; Fedora; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86 Safari/537.36" 0.000804
[2019-07-17T06:24:22Z INFO actix_web::middleware::logger] 127.0.0.1:37424 "GET /foxes HTTP/1.1" 200 1416043 "-" "Mozilla/5.0 (X11; Fedora; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86 Safari/537.36" 0.010263
```

现在，让我们幕后了解代码，以便更好地理解它。

# 它是如何工作的...

`actix-web` ([`actix.rs`](https://actix.rs))是一个多功能的 Web 框架，它——在许多其他事情中——高效地服务静态文件。在本教程中，我们介绍了如何声明和注册请求处理器，以及一些提供响应的方法。在典型的 Web 框架中，有几种方法可以实现这些任务（声明处理器、创建响应），*步骤 1*到*3*展示了使用`actix-web`实现这两种方法的方式：

+   使用属性（`#[get("/foxes")]`）

+   通过服务注册调用（`.service(web::resource("/welcome").to(rust_cookbook))`）

无论我们以何种方式将处理器与路由关联，每个处理器都被包装在一个工厂中，该工厂按需创建新的处理器实例，这在编译器错误指向`#[get(...)]`属性而不是实际函数时非常明显。路径包括用于从路径传递数据到处理器函数的带类型的占位符——但更多内容将在下一个菜谱（*设计 RESTful API*）中介绍。

在*步骤 3*中，我们还添加了记录中间件，记录用户代理、时间和 IP 地址，这样我们也可以在服务器端看到请求。所有这些操作都是使用`actix-web`方法链来完成的，它很好地结构化了调用。对`run()`的调用会阻塞应用程序并启动 actix 主循环。

*步骤 6*中的照片是在 2019 年 Rust All Hands 活动期间在 Mozilla 的柏林办公室拍摄的。是的，那些是 Firefox 枕头。

*步骤 4*添加了一个非常基本的`index.html`文件以供服务，而*步骤 5*则声明了`Cargo.toml`中的依赖项，就像我们之前做的那样。

在最后一步，我们运行代码并显示输出——在浏览器和日志中。

我们已经成功学习了设置 Web 服务器的基础知识。掌握了服务静态文件和图片以及解析路径参数的知识，我们可以继续学习下一个菜谱。

# 设计 RESTful API

几乎所有东西都依赖于 Web 资源——从通过 JavaScript 动态获取数据并在 HTML 中显示的单页应用程序，到特定服务的应用程序集成。Web 服务上的资源可以是任何东西，但它通常使用可读的 URI 来表示，这样信息就已经通过使用特定的路径来传输，然后只接受它需要处理的信息。这允许在内部和全局结构化代码，并利用所有`HTTP`方法创建一个开发者可以使用的表达性接口。RESTful API([`www.codecademy.com/articles/what-is-rest`](https://www.codecademy.com/articles/what-is-rest))理想地捕捉了所有这些好处。

# 准备工作

使用`cargo new api`创建一个 Rust 二进制项目。由于我们将在本地端口`8081`上提供服务，请确保该端口可访问。在新建的项目文件夹中，我们需要一个额外的`static/`文件夹，你可以在这里放置一个有趣的`.jpg`图片以供服务。此外，请确保命令行上有一个像`curl`([`curl.haxx.se/`](https://curl.haxx.se/))这样的程序。或者，Postman([`www.getpostman.com/`](https://www.getpostman.com/))是一个具有图形界面的工具，可以完成相同的功能。

最后，使用 VS Code 打开整个目录。

# 如何操作...

让我们分几个步骤来构建一个 API：

1.  打开`src/main.rs`以添加服务器和请求处理的主要代码。让我们一步一步来，从导入开始：

```rs
#[macro_use]
extern crate actix_web;

use actix_files as fs;
use actix_web::{ guard,
    http::header, http::Method, middleware, web, App, HttpRequest, HttpResponse, HttpServer,
    Responder, Result,
};
use std::env;
use std::path::PathBuf;
```

1.  接下来，我们定义了一些处理器，它们接收请求并使用这些数据。将这些行添加到`main.rs`中：

```rs
#[get("by-id/{id}")]
fn bookmark_by_id(id: web::Path<(i32)>) -> impl Responder {
    format!("{{ \"id\": {}, \"url\": \"https://blog.x5ff.xyz\" }}", id)
}

fn echo_bookmark(req: HttpRequest) -> impl Responder {
    let id: i32 = req.match_info().query("id").parse().unwrap();
    format!("{:?}", id)
}

#[get("/captures/{tail:.*}")]
fn captures(req: HttpRequest) -> Result<fs::NamedFile> {
    let mut root = PathBuf::from("static/");
    let tail: PathBuf = req.match_info().query("tail").parse().unwrap();
    root.push(tail);

    Ok(fs::NamedFile::open(root)?)
}

#[get("from-bitly/{bitlyid}")]
fn bit_ly(req: HttpRequest) -> HttpResponse {
    let bitly_id = req.match_info().get("bitlyid").unwrap();
    let url = req.url_for("bitly", &[bitly_id]).unwrap();
    HttpResponse::Found()
        .header(header::LOCATION, url.into_string())
        .finish()
        .into_body()
}

#[get("/")]
fn bookmarks_index() -> impl Responder {
    format!("Welcome to your quick and easy bookmarking service!")
}
```

1.  接下来，我们还应该将处理器注册到 web 服务器上。以下是 `main.rs` 的 `main` 函数：

```rs
fn main() -> std::io::Result<()> {
    env::set_var("RUST_LOG", "actix_web=debug");
    env_logger::init();
    HttpServer::new(|| {
        App::new()
            .wrap(middleware::Logger::default())
            .service(
                web::scope("/api")
                .service(
                    web::scope("/bookmarks")
                    .service(captures)
                    .service(bookmark_by_id)
                    .service(bit_ly)
                    .service(web::resource("add/{id}")
                        .name("add") 

                        .guard(guard::Any(guard::Put())
                        .or(guard::Post()))
                        .to(echo_bookmark))
            ))
            .service(
                web::scope("/bookmarks")
                    .service(bookmarks_index)
            )
            .external_resource("bitly", "https://bit.ly/{bitly}")

    })
    .bind("127.0.0.1:8081")?
    .run()
}
```

1.  最后，我们应该将 `Cargo.toml` 适配以包含这些新的依赖项：

```rs
[dependencies]
actix-web = "1"
env_logger = "0.6"
actix-files = "0"
```

1.  现在，我们可以使用 `cargo run` 构建并运行应用程序。然后，让我们看看是否可以使用 `curl` 或 Postman 来访问 API，这应该会产生以下类似的日志输出：

```rs
$ cargo run
 Finished dev [unoptimized + debuginfo] target(s) in 0.09s
 Running `target/debug/api`
[2019-07-17T15:38:14Z INFO actix_web::middleware::logger] 127.0.0.1:50426 "GET /bookmarks/ HTTP/1.1" 200 51 "-" "curl/7.64.0" 0.000655
[2019-07-17T15:40:07Z INFO actix_web::middleware::logger] 127.0.0.1:51386 "GET /api/bookmarks/by-id/10 HTTP/1.1" 200 44 "-" "curl/7.64.0" 0.001103
[2019-07-17T15:40:41Z INFO actix_web::middleware::logger] 127.0.0.1:51676 "GET /api/bookmarks/from-bitly/2NOMT6Q HTTP/1.1" 302 0 "-" "curl/7.64.0" 0.007269
[2019-07-17T15:42:26Z INFO actix_web::middleware::logger] 127.0.0.1:52566 "PUT /api/bookmarks/add/10 HTTP/1.1" 200 2 "-" "curl/7.64.0" 0.000704
[2019-07-17T15:42:33Z INFO actix_web::middleware::logger] 127.0.0.1:52626 "POST /api/bookmarks/add/10 HTTP/1.1" 200 2 "-" "curl/7.64.0" 0.001098
[2019-07-17T15:42:39Z INFO actix_web::middleware::logger] 127.0.0.1:52678 "DELETE /api/bookmarks/add/10 HTTP/1.1" 404 0 "-" "curl/7.64.0" 0.000630
[2019-07-17T15:43:30Z INFO actix_web::middleware::logger] 127.0.0.1:53094 "GET /api/bookmarks/captures/does-not/exist HTTP/1.1" 404 38 "-" "curl/7.64.0" 0.003554
[2019-07-17T15:43:39Z INFO actix_web::middleware::logger] 127.0.0.1:53170 "GET /api/bookmarks/captures/foxes.jpg HTTP/1.1" 200 59072 "-" "curl/7.64.0" 0.013600
```

以下是一些 `curl` 请求——它们应该很容易用 Postman 来复制：

```rs
$ curl localhost:8081/bookmarks/
Welcome to your quick and easy bookmarking service!⏎ 
$ curl localhost:8081/api/bookmarks/by-id/10
{ "id": 10, "url": "https://blog.x5ff.xyz" }⏎
$ curl -v localhost:8081/api/bookmarks/from-bitly/2NOMT6Q
* Trying ::1...
* TCP_NODELAY set
* connect to ::1 port 8081 failed: Connection refused
* Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8081 (#0)
> GET /api/bookmarks/from-bitly/2NOMT6Q HTTP/1.1
> Host: localhost:8081
> User-Agent: curl/7.64.0
> Accept: */*
> 
< HTTP/1.1 302 Found
< content-length: 0
< location: https://bit.ly/2NOMT6Q
< date: Wed, 17 Jul 2019 15:40:45 GMT
<
$ curl -X PUT localhost:8081/api/bookmarks/add/10
10⏎ 
$ curl -X POST localhost:8081/api/bookmarks/add/10
10⏎
$ curl -v -X DELETE localhost:8081/api/bookmarks/add/10
* Trying ::1...
* TCP_NODELAY set
* connect to ::1 port 8081 failed: Connection refused
* Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8081 (#0)
> DELETE /api/bookmarks/add/10 HTTP/1.1
> Host: localhost:8081
> User-Agent: curl/7.64.0
> Accept: */*
> 
< HTTP/1.1 404 Not Found
< content-length: 0
< date: Wed, 17 Jul 2019 15:42:51 GMT
< 
* Connection #0 to host localhost left intact
$ curl localhost:8081/api/bookmarks/captures/does-not/exist
No such file or directory (os error 2)⏎ 17:43:31
$ curl localhost:8081/api/bookmarks/captures/foxes.jpg
Warning: Binary output can mess up your terminal. Use "--output -" to tell 
Warning: curl to output it to your terminal anyway, or consider "--output 
Warning: <FILE>" to save to a file.
```

就这样了，但还有很多东西要解释，让我们看看为什么它会以这种方式工作。

# 它是如何工作的...

设计 *良好* 的 API 是困难的，并且需要很好地掌握可能发生的事情——特别是对于新的框架和语言。`actix-web` 已经证明了自己是一个多才多艺的工具，它有效地使用类型来取得很好的效果。*步骤 1* 通过导入一些类型和特性来设置这一点。

只有在 *步骤 2* 和 *步骤 3* 中，事情才会变得更有趣。在这里，我们通过使用将函数包装成工厂的属性（在其下方都是异步演员；查看 第四章，*无畏并发* 中的 *使用演员处理异步消息*）或者让 `web::resource()` 类型来完成，几乎以 `actix-web` 允许的所有方式定义了各种处理器。无论哪种方式，每个处理器函数都有一个与之关联的路由，并且将并行调用。路由还包含可以使用 `{}` 语法指定的参数，该语法还允许使用正则表达式（例如，包含 `"{tail:.*}"` 的路由 - 这是一个简写，它接收 `tail` 键下的路径剩余部分）。

不要让用户直接访问你的文件系统中的文件，就像我们在这里做的那样。这在很多方面都是个坏主意，但最重要的是它提供了一种执行文件系统中任何文件的方法。更好的方法是提供一个抽象文件的白名单——例如，Base64 ([`developer.mozilla.org/en-US/docs/Web/API/WindowBase64/Base64_encoding_and_decoding`](https://developer.mozilla.org/en-US/docs/Web/API/WindowBase64/Base64_encoding_and_decoding)) 编码的——使用一个独立的关键字：例如，UUID ([`tools.ietf.org/html/rfc4122`](https://tools.ietf.org/html/rfc4122))。

如果一个函数提供了一个 `Path<T>` 类型的输入参数，那么 `T` 就是相应路径变量中检查的内容。因此，如果函数头期望 `i32`，那么尝试传递字符串的人的请求将会失败。你可以通过 `bookmarks/by-id/{id}` 路径来验证这一点。作为 `Path<T>` 的替代方案，你也可以接收整个 `HttpRequest` ([`docs.rs/actix-web/1.0.3/actix_web/struct.HttpRequest.html`](https://docs.rs/actix-web/1.0.3/actix_web/struct.HttpRequest.html)) 作为参数，并通过 `.query()` 函数提取所需信息。`echo_bookmark` 和 `bit_ly` 函数展示了如何使用这些方法。

响应的行为类似。`actix-web` 提供了一个 `Responder` 特性，该特性为标准类型（如 `String`，以及我们看到的正确响应内容类型）实现了，这使得处理程序更易于阅读。同样，返回 `HttpResponse` 类型提供了更精细的控制。此外，还有一些结果和类似类型会自动转换为适当的响应，但展示所有这些内容将超出本书的范围。请查阅 `actix-web` 文档以获取更多信息。

属性的一个缺点是只有一个可以放在函数的顶部——那么我们如何重用函数以处理两种不同的 `HTTP` 方法？`echo_bookmark` 仅在 `PUT` 和 `POST` 上注册以响应输入 ID，而不是在 `DELETE`、`HEAD`、`GET` 以及更多方法上。这是通过守卫实现的，只有当条件满足时才会转发请求。查看文档 ([`docs.rs/actix-web/1.0.3/actix_web/guard/index.html`](https://docs.rs/actix-web/1.0.3/actix_web/guard/index.html)) 以获取更多信息。

*第 4 步* 展示了如何修改 `Cargo.toml` 以使其正常工作，而在 *第 5 步* 中，我们可以尝试使用 Web 服务。如果你花点时间观察 `curl` 的响应，我们会收到预期的结果。默认情况下，`curl` 不会跟随重定向，因此，带有位置头并指向我将去的地方的 `HTTP` 响应代码 `302`。这种重定向是由 `actix-web` 提供的外部资源提供的，这对于这些情况很有用。

现在我们已经了解了如何在 `actix-web` 中设计 API，让我们继续到下一个配方。

# 处理 JSON 负载

在学习如何创建 API 之后，我们需要传递数据。虽然路径提供了一种方法来做这件事，但任何稍微复杂一些的东西（例如，一个很长的项目列表）很快就会显示出这些方法的局限性。这就是为什么通常使用其他格式来结构化数据——JSON ([`json.org/`](http://json.org/)) 是 Web 服务中最受欢迎的。在本章中，我们将使用之前的 API 并通过处理和返回 JSON 来增强它。

# 准备工作

让我们使用 `cargo new json-handling` 设置一个 Rust 二进制项目。由于我们将在本地端口 `8081` 上提供服务，请确保该端口可访问。此外，还需要一个如 `curl` 或 Postman 这样的程序来测试 Web 服务。

最后，使用 VS Code 打开整个目录。

# 如何做到这一点...

执行以下步骤以实现此配方：

1.  在 `src/main.rs` 中，我们首先添加导入：

```rs
#[macro_use]
extern crate actix_web;

use actix_web::{
    guard, http::Method, middleware, web, App, HttpResponse, HttpServer,
};
use serde_derive::{Deserialize, Serialize};
use std::env;
```

1.  接下来，让我们创建一些处理函数以及一个可序列化的 JSON 类型。将以下代码添加到 `src/main.rs` 中：

```rs
#[derive(Debug, Clone, Serialize, Deserialize)]
struct Bookmark {
    id: i32,
    url: String,
}

#[get("by-id/{id}")]
fn bookmarks_by_id(id: web::Path<(i32)>) -> HttpResponse {
    let bookmark = Bookmark {
        id: *id,
        url: "https://blog.x5ff.xyz".into(),
    };
    HttpResponse::Ok().json(bookmark)
}

fn echo_bookmark(bookmark: web::Json<Bookmark>) -> HttpResponse {
    HttpResponse::Ok().json(bookmark.clone())
}
```

1.  最后，我们在 `main` 函数中将处理程序注册到 Web 服务器上：

```rs
fn main() -> std::io::Result<()> {
    env::set_var("RUST_LOG", "actix_web=debug");
    env_logger::init();
    HttpServer::new(|| {
        App::new().wrap(middleware::Logger::default()).service(
            web::scope("/api").service(
                web::scope("/bookmarks")
                    .service(bookmarks_by_id)
                    .service(
                        web::resource("add/{id}")
                            .name("add")
                            .guard(guard::Any(guard::Put()).
                             or(guard::Post()))
                            .to(echo_bookmark),
                    )
                    .default_service(web::route().method
                     (Method::GET)),
            ),
        )
    })
    .bind("127.0.0.1:8081")?
    .run()
}
```

1.  我们还需要在 `Cargo.toml` 中指定依赖项。用以下依赖项替换现有的依赖项：

```rs
[dependencies]
actix-web = "1"
serde = "1"
serde_derive = "1"
env_logger = "0.6"
```

1.  然后，我们可以通过运行 `cargo run` 并从不同的终端使用 `curl` 发送请求来测试它是否工作：命令及其响应应该如下所示：

```rs
$ curl -d "{\"id\":10,\"url\":\"https://blog.x5ff.xyz\"}" localhost:8081/api/bookmarks/add/10
Content type error⏎
$ curl -d "{\"id\":10,\"url\":\"https://blog.x5ff.xyz\"}" -H "Content-Type: application/json" localhost:8081/api/bookmarks/add/10
{"id":10,"url":"https://blog.x5ff.xyz"}⏎
$ curl localhost:8081/api/bookmarks/by-id/1
{"id":1,"url":"https://blog.x5ff.xyz"}⏎ 
```

同时，`cargo run` 的日志输出显示了来自服务器端的请求：

```rs
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.08s
     Running `target/debug/json-handling`
[2019-07-13T17:06:22Z INFO actix_web::middleware::logger] 127.0.0.1:48880 "POST /api/bookmarks/add/10 HTTP/1.1" 400 63 "-" "curl/7.64.0" 0.001955
[2019-07-13T17:06:51Z INFO actix_web::middleware::logger] 127.0.0.1:49124 "POST /api/bookmarks/add/10 HTTP/1.1" 200 39 "-" "curl/7.64.0" 0.001290
[2019-07-18T06:34:18Z INFO actix_web::middleware::logger] 127.0.0.1:54900 "GET /api/bookmarks/by-id/1 HTTP/1.1" 200 39 "-" "curl/7.64.0" 0.001636
```

这很快也很简单，对吧？让我们看看它是如何工作的。

# 它是如何工作的...

将 JSON 处理添加到 `actix-web` 网络服务中很容易——归功于流行的 `serde` 包（[`crates.io/crates/serde`](https://crates.io/crates/serde)）的深度集成。在 *步骤 1* 的一些导入之后，我们在 *步骤 2* 中声明一个 `Bookmark` 结构体作为 `Serialize` 和 `Deserialize`，这使得 `serde` 能够为该数据类型生成和解析 JSON。

由于返回和消费 JSON 是一个非常常见的任务，处理函数的变化也是最小的。返回 JSON 有效载荷所需的函数附加到 `HttpResponse` 工厂方法上，该方法执行所有操作，包括设置适当的内容类型。在消费部分，有一个 `web::Json<T>` 类型来处理将转发到请求处理器的任何内容进行反序列化和检查。我们也可以依赖框架在这里做大部分的重活。

在 *步骤 3* 中注册处理器的操作与之前的菜谱没有不同；JSON 输入仅在处理函数中声明。`actix-web` 文档中有更多变体（[`actix.rs/docs/request/#json-request`](https://actix.rs/docs/request/#json-request)）及其示例（[`github.com/actix/examples/tree/master/json`](https://github.com/actix/examples/tree/master/json)）。同样，*步骤 4* 包含了我们也在其他菜谱中使用过的必需依赖项。

在 *步骤 5* 中，我们运行整个项目并查看其工作情况：如果我们传递 JSON，输入 `content-type` 标头必须设置为适当的 MIME 类型（`application/json`）；返回值也有这个头设置（以及 `content-length` 头），这样浏览器或其他程序可以轻松地处理结果。

让我们继续，看看另一个菜谱。

# 网络错误处理

网络服务的各个层次使得错误处理变得复杂，即使没有安全要求：应该传达什么以及何时传达？错误是否应该只在最后一刻才冒泡出来处理，还是更早一些？关于级联处理呢？在这个菜谱中，我们将揭示一些在 `actix-web` 中优雅处理这些问题的方法。

# 准备工作

让我们使用 `cargo new web-errors` 设置一个 Rust 二进制项目。由于我们将在本地端口 `8081` 上提供服务，请确保该端口可访问。此外，还需要一个程序，如 `curl` 或 Postman，来测试网络服务。

最后，使用 VS Code 打开整个目录。

# 如何操作...

你只需几个步骤就能理解 `actix-web` 的错误处理：

1.  在 `src/main.rs` 中，我们将添加基本的导入：

```rs
#[macro_use]
extern crate actix_web;
use failure::Fail;

use actix_web::{ http, middleware, web, App, HttpResponse, HttpServer, error
};
use serde_derive::{Deserialize, Serialize};
use std::env;
```

1.  作为下一步，我们将定义我们的错误类型，并使用属性增强它们，以便框架知道这些类型：

```rs
#[derive(Fail, Debug)]
enum WebError {
    #[fail(display = "Invalid id '{}'", id)]
    InvalidIdError { id: i32 },
    #[fail(display = "Invalid request, please try again later")]
    RandomInternalError,
}

impl error::ResponseError for WebError {
    fn error_response(&self) -> HttpResponse {
        match *self {
            WebError::InvalidIdError { .. } => HttpResponse::new(http::StatusCode::BAD_REQUEST),
            WebError::RandomInternalError => HttpResponse::new(http::StatusCode::INTERNAL_SERVER_ERROR)
        }
    }
}
```

1.  然后，我们将处理函数添加到 `src/main.rs` 中，并在 `main()` 中注册它：

```rs
#[derive(Debug, Clone, Serialize, Deserialize)]
struct Bookmark {
    id: i32,
    url: String,
}

#[get("by-id/{id}")]
fn bookmarks_by_id(id: web::Path<(i32)>) -> Result<HttpResponse, WebError> {
    if *id < 10 {
        Ok(HttpResponse::Ok().json(Bookmark {
            id: *id,
            url: "https://blog.x5ff.xyz".into(),
        }))
    }
    else {
        Err(WebError::InvalidIdError { id: *id })
    }
}

fn main() -> std::io::Result<()> {
    env::set_var("RUST_LOG", "actix_web=debug");
    env_logger::init();
    HttpServer::new(|| {
        App::new()
            .wrap(middleware::Logger::default())
            .service(
                web::scope("/bookmarks")
                    .service(bookmarks_by_id)
            )
            .route(
                "/underconstruction",
                web::get().to(|| Result::<HttpResponse, 
                WebError>::Err(WebError::RandomInternalError)),
            )
    })
    .bind("127.0.0.1:8081")?
    .run()
}
```

1.  为了导入依赖项，我们还需要调整 `Cargo.toml`：

```rs
[dependencies]
actix-web = "1"
serde = "1"
serde_derive = "1"
env_logger = "0.6"
failure = "0"
```

1.  为了完成这个配方，让我们看看如何使用 `cargo run` 和 `curl` 一起工作。以下是请求处理后的服务器输出：

```rs
$ cargo run
  Compiling web-errors v0.1.0 (Rust-Cookbook/Chapter08/web-errors)
    Finished dev [unoptimized + debuginfo] target(s) in 7.74s
     Running `target/debug/web-errors`
[2019-07-19T17:33:43Z INFO actix_web::middleware::logger] 127.0.0.1:46316 "GET /bookmarks/by-id/1 HTTP/1.1" 200 38 "-" "curl/7.64.0" 0.001529
[2019-07-19T17:33:47Z INFO actix_web::middleware::logger] 127.0.0.1:46352 "GET /bookmarks/by-id/100 HTTP/1.1" 400 16 "-" "curl/7.64.0" 0.000952
[2019-07-19T17:33:54Z INFO actix_web::middleware::logger] 127.0.0.1:46412 "GET /underconstruction HTTP/1.1" 500 39 "-" "curl/7.64.0" 0.000275
```

以下是用 `curl` 的详细模式显示的请求示例：

```rs
$ curl -v localhost:8081/bookmarks/by-id/1
* Trying ::1...
* TCP_NODELAY set
* connect to ::1 port 8081 failed: Connection refused
* Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8081 (#0)
> GET /bookmarks/by-id/1 HTTP/1.1
> Host: localhost:8081
> User-Agent: curl/7.64.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< content-length: 38
< content-type: application/json
< date: Fri, 19 Jul 2019 17:33:43 GMT
< 
* Connection #0 to host localhost left intact
{"id":1,"url":"https://blog.x5ff.xyz"}⏎ 
```

请求错误的 ID 会返回适当的 HTTP 状态代码：

```rs
$ curl -v localhost:8081/bookmarks/by-id/100
* Trying ::1...
* TCP_NODELAY set
* connect to ::1 port 8081 failed: Connection refused
* Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8081 (#0)
> GET /bookmarks/by-id/100 HTTP/1.1
> Host: localhost:8081
> User-Agent: curl/7.64.0
> Accept: */*
> 
< HTTP/1.1 400 Bad Request
< content-length: 16
< content-type: text/plain
< date: Fri, 19 Jul 2019 17:33:47 GMT
< 
* Connection #0 to host localhost left intact
Invalid id '100'⏎
```

正如预期的那样，对 `/underconstruction` 的请求返回 HTTP 500 错误（内部服务器错误）：

```rs
$ curl -v localhost:8081/underconstruction
* Trying ::1...
* TCP_NODELAY set
* connect to ::1 port 8081 failed: Connection refused
* Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8081 (#0)
> GET /underconstruction HTTP/1.1
> Host: localhost:8081
> User-Agent: curl/7.64.0
> Accept: */*
> 
< HTTP/1.1 500 Internal Server Error
< content-length: 39
< content-type: text/plain
< date: Fri, 19 Jul 2019 17:33:54 GMT
< 
* Connection #0 to host localhost left intact
Invalid request, please try again later⏎ 
```

由于这工作得很好，让我们看看它是如何工作的。

# 它是如何工作的...

`actix-web` 使用一个错误特性和将 Rust 错误转换为 `HttpResponses`。这个特性和默认错误自动实现，但只响应默认的 *内部服务器错误* 消息。

在 *步骤 1* 和 *步骤 2* 中，我们正在设置自定义错误，以便我们可以返回与用户当前正在执行的操作（或尝试执行的操作）相关的消息。与其他错误一样（见第五章，*处理错误和其他结果*），我们使用枚举来提供一个伞状结构，以便将错误变体匹配起来。每个变体都增加了一个属性，它提供了一个带有格式字符串的相应错误消息——这是由 `failure` crate（[`crates.io/crates/failure`](https://crates.io/crates/failure)）提供的功能。这里的消息是针对响应代码 500（默认值）的最后一招类型消息。这个 HTTP 响应代码，连同错误体（如 HTML 页面）一起，可以通过实现 `actix_web::error::ResponseError` 特性进行自定义。使用 `error_response()` 函数提供的任何 `HttpResponse` 都将返回给客户端。

如果您自己调用该函数，则不会附加 `#[fail(display="...")]` 消息。始终使用 Rust 的 `Result` 枚举将错误传达给 `actix_web`。

*第 3 步* 定义了 web 服务的处理函数，并且由于它使用 JSON 响应，因此需要一个用于序列化信息的结构体。在这个例子中，我们还在使用任意数字 10 作为返回错误的截止点——使用 Rust 的 `Result` 枚举。这提供了一个框架无关的方式来处理不良结果，就像我们使用纯 Rust 一样。第二个路由 `/underconstruction` 展示了 `actix-web` 路由的实现方式：作为一个闭包。由于它立即返回一个错误，我们必须明确告诉编译器返回类型以及它是一个可能返回 `HttpResponse` 或 `WebError` 的 `Result` 枚举。然后我们直接返回后者。*第 4 步* 展示了所需的依赖项，并告诉我们必须包含 failure crate。在最后一步，我们运行代码并通过发出 `curl` 请求来测试它，并检查服务器端的日志。这并不复杂，对吧？如果您想深入了解，也可以查看 `actix-web` 文档 ([`actix.rs/docs/errors/`](https://actix.rs/docs/errors/))。

让我们继续下一个步骤。

# 渲染 HTML 模板

虽然 JSON 是一个非常易于阅读且易于处理的格式，但许多人仍然更喜欢更互动的体验——例如网站。虽然这并不是 `actix-web` 的原生功能，但一些模板引擎提供了无缝集成，以最小化组装和输出 HTML 所需的调用。与简单地提供静态网站相比，主要区别在于模板引擎将变量输出和 Rust 代码渲染到增强的 HTML 页面中，以产生适应应用程序状态的任何内容。在本步骤中，我们将查看 **另一个 Rust 模板引擎**（**Yarte**）([`crates.io/crates/yarte`](https://crates.io/crates/yarte)) 与 `actix-web` 的集成。

# 准备工作

使用 `cargo new html-templates` 创建一个 Rust 二进制项目，并确保本地主机可以访问端口 `8081`。在创建项目目录后，您还需要创建一些额外的文件夹和文件。静态目录内的图像文件可以是任何图像，只要有一个 Base64 编码的版本作为文本文件即可。您可以使用在线服务或 Linux 上的 Base64 二进制文件（[`linux.die.net/man/1/base64`](https://linux.die.net/man/1/base64)）来创建自己的（您需要相应地更改代码中的名称）或使用我们仓库中的版本。`.hbs` 文件将在本步骤中填充（创建）：

```rs
html-templates/
├── Cargo.toml
├── src
│   └── main.rs
├── static
│   ├── packtpub.com.b64
│   ├── packtpub.com.png
│   ├── placeholder.b64
│   ├── placeholder.png
│   ├── x5ff.xyz.b64
│   └── x5ff.xyz.png
└── templates
    ├── index.hbs
    └── partials
        └──bookmark.hbs
```

最后，使用 VS Code 打开整个目录。

# 如何操作...

只需几个步骤就可以创建模板化的网页：

1.  首先，让我们向 `src/main.rs` 添加一些代码。用以下内容替换默认片段（注意：`PLACEHOLDER_IMG` 中的 Base64 编码字符串在此处进行了缩写。获取完整的 Base64 编码图像请访问 [`blog.x5ff.xyz/other/placeholder.b64`](https://blog.x5ff.xyz/other/placeholder.b64)）：

```rs
#[macro_use]
extern crate actix_web;

use actix_web::{middleware, web, App, HttpServer, Responder};
use chrono::prelude::*;
use std::env;
use yarte::Template;

const PLACEHOLDER_IMG: &str =
    "iVBORw0KGgoAAAANS[...]s1NR+4AAAAASUVORK5CYII=";

#[derive(Template)]
#[template(path = "index.hbs")]
struct IndexViewModel {
    user: String,
    bookmarks: Vec<BookmarkViewModel>,
}

#[derive(Debug, Clone)]
struct BookmarkViewModel {
    timestamp: Date<Utc>,
    url: String,
    mime: String,
    base64_image: String,
}
```

在调整 `src/main.rs` 之后，将所需的依赖项添加到 `Cargo.toml`：

```rs
[dependencies] 
actix-web = "1"
serde = "1"
serde_derive = "1"
env_logger = "0.6"
base64 = "0.10.1"
yarte = {version = "0", features=["with-actix-web"]}
chrono = "0.4"⏎ 

```

1.  在声明模板之后，我们需要注册一个处理程序来提供服务：

```rs
#[get("/{name}")]
pub fn index(name: web::Path<(String)>) -> impl Responder {
    let user_name = name.as_str().into();
```

首先，让我们为已识别的用户添加书签数据：

```rs
    if &user_name == "Claus" {
        IndexViewModel {
            user: user_name,
            bookmarks: vec![
                BookmarkViewModel {
                    timestamp: Utc.ymd(2019, 7, 20),
                    url: "https://blog.x5ff.xyz".into(),
                    mime: "image/png".into(),
                    base64_image: std::fs::read_to_string
                    ("static/x5ff.xyz.b64")
                        .unwrap_or(PLACEHOLDER_IMG.into()),
                },
                BookmarkViewModel {
                    timestamp: Utc.ymd(2017, 9, 1),
                    url: "https://microsoft.com".into(),
                    mime: "image/png".into(),
                    base64_image: std::fs::read_to_string
                    ("static/microsoft.com.b64")
                        .unwrap_or(PLACEHOLDER_IMG.into()),
                },
                BookmarkViewModel {
                    timestamp: Utc.ymd(2019, 2, 2),
                    url: "https://www.packtpub.com/".into(),
                    mime: "image/png".into(),
                    base64_image: std::fs::read_to_string
                    ("static/packtpub.com.b64")
                        .unwrap_or(PLACEHOLDER_IMG.into()),
                },
            ],
        }
```

对于其他所有人（未识别的用户），我们可以简单地返回一个空向量：

```rs
    } else {
        IndexViewModel {
            user: user_name,
            bookmarks: vec![],
        }
    }
}
```

最后，让我们在 `main` 函数中启动服务器：

```rs
fn main() -> std::io::Result<()> {
    env::set_var("RUST_LOG", "actix_web=debug");
    env_logger::init();
    HttpServer::new(|| {
        App::new()
            .wrap(middleware::Logger::default())
            .service(web::scope("/bookmarks").service(index))
    })
    .bind("127.0.0.1:8081")?
    .run()
}
```

1.  代码已经准备好了，但我们仍然缺少模板。这就是我们在 `.hbs` 文件中添加一些内容的地方。首先，让我们向 `templates/index.hbs` 添加代码：

```rs
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <link href="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-ggOyR0iXCbMQv3Xipma34MD+dH/1fQ784/j6cY/iJTQUOhcWr7x9JvoRxT2MZw1T" crossorigin="anonymous">
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
</head>
```

在头部之后，我们需要一个 HTML 主体来标记数据：

```rs
<body>
    <div class="container">
        <div class="row">
            <div class="col-lg-12 pb-3">
                <h1>Welcome {{ user }}.</h1>
                <h2 class="text-muted">Your bookmarks:</h2>
            </div>
        </div>

        {{#if bookmarks.is_empty() }}
        <div class="row">
            <div class="col-lg-12">
            No bookmarks :(
            </div>
        </div>
        {{~/if}}

        {{#each bookmarks}}
            <div class="row {{# if index % 2 == 1 }} bg-light text-
            dark {{/if }} mt-2 mb-2">
            {{> partials/bookmark }}
            </div>
        {{~/each}}
    </div>
</body>
</html>
```

1.  我们在这个最后的模板中调用了一个部分，所以让我们也向它添加一些代码。打开 `templates/partials/bookmark.hbs` 并插入以下内容：

```rs
<div class="col-lg-2">
    <img class="rounded img-fluid p-1" src="img/>    {{ base64_image }}"> </div>
<div class="col-lg-10">
    <a href="{{ url }}">
        <h3>{{ url.replace("https://", "") }}</h3>
    </a>
    <i class="text-muted">Added {{ timestamp.format("%Y-%m-
    %d").to_string() }}</i>
</div>
```

1.  是时候尝试一下了！使用 `cargo run` 启动服务器并记录输出，同时打开浏览器窗口访问 `localhost:8081/bookmarks/Hans` 以及 `localhost:8081/bookmarks/Claus`，以查看是否正常工作。以下是浏览器窗口在打开这些 URL 后 `cargo run` 显示的内容：

```rs
$ cargo run
   Compiling html-templates v0.1.0 (Rust-Cookbook/Chapter08/html-templates)
 Finished dev [unoptimized + debuginfo] target(s) in 2m 38s
     Running `target/debug/html-templates`
[2019-07-20T16:36:06Z INFO actix_web::middleware::logger] 127.0.0.1:50060 "GET /bookmarks/Claus HTTP/1.1" 200 425706 "-" "Mozilla/5.0 (X11; Fedora; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86 Safari/537.36" 0.013246
[2019-07-20T16:37:34Z INFO actix_web::middleware::logger] 127.0.0.1:50798 "GET /bookmarks/Hans HTTP/1.1" 200 821 "-" "Mozilla/5.0 (X11; Fedora; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86 Safari/537.36" 0.000730
```

以下是一个未识别用户的查询结果：

![](img/b92368b6-0067-49e0-863f-17be693eb176.png)

对于已识别的用户，系统返回适当的内容：

![](img/ce12f078-f0c7-4980-ae72-957f06fd6698.png)

让我们找出这是为什么。

# 它是如何工作的...

在许多语言中，创建模板引擎有点像入门教程——这可能是 Yarte 这个名字的由来。虽然选择很多，但 `actix-web` 提供了三个其他引擎的示例；我们建议您在他们的 GitHub 仓库中查看它们（[`github.com/actix/examples`](https://github.com/actix/examples)）。这个食谱的 *第一步* 已经涵盖了某些重要的工作：导入内容并声明视图模型（如 MVVM 模式：[`blogs.msdn.microsoft.com/msgulfcommunity/2013/03/13/understanding-the-basics-of-mvvm-design-pattern/`](https://blogs.msdn.microsoft.com/msgulfcommunity/2013/03/13/understanding-the-basics-of-mvvm-design-pattern/)）。Yarte 提供了宏属性，可以将特定的模型与模板文件关联起来——并且它会自动找到 `templates` 文件夹。如果这不适合您的项目，它们允许您相应地配置框架。更多详情请访问他们的网站（[`yarte.netlify.com/`](https://yarte.netlify.com/)）。我们使用的是嵌套模型，其中内部结构不需要自己的关联模板。

在*第 2 步*中，我们在`/bookmarks`作用域下注册了处理函数，并指定了`/{name}`路径，这导致 URL：`/bookmarks/{name}`。`actix-web`在检查路由方面非常严格，所以`/bookmarks/{name}**/**`将会返回一个错误（404）。处理函数返回一个包含 Claus 名字的小书签列表，但不为其他人返回，这在更现实的场景中会从数据库中获取。无论如何，我们使用这个硬编码的版本，并添加了日志中间件，以便我们可以看到发生了什么。我们还使用了一个常量作为占位符图片，您可以在[`blog.x5ff.xyz/other/placeholder.b64`](https://blog.x5ff.xyz/other/placeholder.b64)下载它。

我们在第 3 步中定义的模板是引擎之间主要的不同之处。使用众所周知的`{{ rust-code }}`符号，我们可以增强常规 HTML 以生成更复杂的输出。有各种类型的循环、条件、变量和部分。部分之所以重要，是因为它们允许你将视图部分拆分为可重用的组件，这些组件甚至不一定是 HTML/Yarte 模板，可以是任何文本。

编译过程将这些模板拉入，将它们与我们之前声明的类型结合起来——有一个重要的后果。目前，更改模板需要重新编译`main.rs`文件，以反映更改，因此建议使用 touch 或类似的命令来设置`src/main.rs`的修改日期。之后，`cargo`的行为就像`src/main.rs`发生了变化一样。

*第 4 步* 实现了显示每个书签的部分，类似于第 3 步中的索引模板。只有在*第 5 步*中，我们才会运行并查看结果：一个简单的网站，显示了当用户被识别（即：有与该名称相关联的数据）和当用户未被识别时的相关书签列表。通过使用流行的 Bootstrap CSS 框架（[`getbootstrap.com`](https://getbootstrap.com)），实现了最小化设计。

现在，让我们继续下一个菜谱。

# 使用 ORM 将数据保存到数据库

对于对象关系映射器的看法差异很大：当 SQL 数据库存储了世界上所有的数据时，它们的用途被强烈鼓励，但随后当实际上整个世界的数据都如此时，它们很快就失去了青睐。通常这些框架在易用性、语言集成和可扩展性之间提供了一种权衡。虽然查询 TB 级的数据确实需要一种根本不同的方法，但简单的 CRUD 型业务应用程序与为你做繁重工作的框架配合得很好，而且——最重要的是——它们与它们连接的实际数据库多少是独立的。Rust 的宏在这里非常有用——它们允许 ORM 框架在编译时做这些事情，因此它是内存安全的、类型安全的，并且速度快。让我们看看它是如何实现的。

# 准备工作

使用 `cargo new orm` 创建一个 Rust 二进制项目，并确保本地主机可以访问端口 `8081`。要访问服务，获取一个程序，例如 `curl` 或 Postman，来执行 `POST`、`GET` 以及更多类型的网络请求，以及一个用于创建和管理 SQLite ([`www.sqlite.org/index.html`](https://www.sqlite.org/index.html)) 数据库的程序（例如，sqlitebrowser: [`github.com/sqlitebrowser/sqlitebrowser`](https://github.com/sqlitebrowser/sqlitebrowser))。

使用 SQLite 数据库管理器，在 `db` 文件夹中创建一个新的数据库，`bookmarks.sqlite`。然后，添加一个遵循此模式的表：

```rs
CREATE TABLE bookmarks(id TEXT PRIMARY KEY, url TEXT);
```

接下来，我们将在项目中使用 `libsqlite3` 库和头文件。在 Linux、WSL 和 macOS 上，从软件包仓库安装适当的软件包。在 Ubuntu 和 WSL 上，您可以使用类似 `apt-get install libsqlite3-dev` 的命令。对于其他发行版和 macOS，请使用您首选的包管理器安装 `libsqlite3` 及其头文件。

原生 Windows 10 用户可能需要从 [`www.sqlite.org/download.html`](https://www.sqlite.org/download.html) 下载 `dll` 二进制文件并将它们放入项目目录中。然而，强烈推荐使用 Linux/macOS。

最后，使用 VS Code 打开整个目录。

# 如何做到这一点...

只需几个步骤就可以运行数据库查询：

1.  在 `src/main.rs` 中，我们将添加基本的导入：

```rs
#[macro_use]
extern crate diesel;
mod models;
mod schema;

use actix_web::{middleware, web, App, Error, HttpResponse, HttpServer};

use std::env;

use diesel::prelude::*;
use diesel::sqlite::SqliteConnection;
use futures::Future;
use models::{Bookmark, NewBookmark};
use serde_derive::{Deserialize, Serialize};
```

1.  让我们在 `main.rs` 中设置一些辅助类型和一个用于连接字符串的常量：

```rs
// Helpers
const SQLITE_DB_URL: &str = "db/bookmarks.sqlite";

#[derive(Debug, Serialize, Deserialize)]
struct WebBookmark {
    url: String,
}

fn connect(db_url: &str) -> SqliteConnection {
    SqliteConnection::establish(&SQLITE_DB_URL)
     .expect(&format!("Error connecting to {}", db_url))
}
```

1.  我们还需要一些处理器，所以让我们将它们添加到文件中，从通过 ID 获取书签开始：

```rs
// Handlers
fn bookmarks_by_id(req_id: web::Path<(String)>) -> impl 
 Future<Item = HttpResponse, Error = Error> {
    web::block(move || {
        use self::schema::bookmarks::dsl::*;

        let conn = connect(&SQLITE_DB_URL);
        bookmarks
            .filter(id.eq(req_id.as_str()))
            .limit(1)
            .load::<Bookmark>(&conn)
    })
    .then(|res| match res {
        Ok(obj) => Ok(HttpResponse::Ok().json(obj)),
        Err(_) => Ok(HttpResponse::InternalServerError().into()),
    })
}
```

为了找出所有 ID，我们还想有一个处理器，它返回所有书签：

```rs
fn all_bookmarks() -> impl Future<Item = HttpResponse, Error = Error> {
    web::block(move || {
        use self::schema::bookmarks::dsl::*;

        let conn = connect(&SQLITE_DB_URL);
        bookmarks.load::<Bookmark>(&conn)
    })
    .then(|res| match res {
        Ok(obj) => Ok(HttpResponse::Ok().json(obj)),
        Err(_) => Ok(HttpResponse::InternalServerError().into()),
    })
}
```

接下来，让我们看看是否可以添加一些书签：

```rs
fn bookmarks_add(
    bookmark: web::Json<WebBookmark>,
     ) -> impl Future<Item = HttpResponse, Error = Error> {
    web::block(move || {
        use self::schema::bookmarks::dsl::*;

        let conn = connect(&SQLITE_DB_URL);
        let new_id = format!("{}", uuid::Uuid::new_v4());
        let new_bookmark = NewBookmark {
            id: &new_id,
            url: &bookmark.url,
        };
        diesel::insert_into(bookmarks)
            .values(&new_bookmark)
            .execute(&conn)
            .map(|_| new_id)
    })
    .then(|res| match res {
        Ok(obj) => Ok(HttpResponse::Ok().json(obj)),
        Err(_) => Ok(HttpResponse::InternalServerError().into()),
    })
}
```

几乎完整的 CRUD（创建、读取、更新、删除）操作还缺少 `delete` 函数：

```rs
fn bookmarks_delete(
    req_id: web::Path<(String)>,
     ) -> impl Future<Item = HttpResponse, Error = Error> {
    web::block(move || {
        use self::schema::bookmarks::dsl::*;

        let conn = connect(&SQLITE_DB_URL);
        diesel::delete(bookmarks.filter(id.eq(req_id.as_str())))
         .execute(&conn)
    })
    .then(|res| match res {
        Ok(obj) => Ok(HttpResponse::Ok().json(obj)),
        Err(_) => Ok(HttpResponse::InternalServerError().into()),
    })
}
```

最后，我们在 `main` 函数中将所有这些整合起来，该函数启动服务器并附加这些处理器：

```rs
fn main() -> std::io::Result<()> {
    env::set_var("RUST_LOG", "actix_web=debug");
    env_logger::init();
    HttpServer::new(move || {
        App::new().wrap(middleware::Logger::default()).service(
            web::scope("/api").service(
                web::scope("/bookmarks")
                    .service(web::resource("/all").route(web::get()
                    .to_async(all_bookmarks)))
                    .service(
                        web::resource("by-id/{id}").route(web
                        ::get().to_async(bookmarks_by_id)),
                    )
                    .service(
                        web::resource("/")
                            .data(web::JsonConfig::default())      
                            .route(web::post().to_async
                            (bookmarks_add)),
                    )
                    .service(
                        web::resource("by-
                        id/{id}").route(web::delete()
                        .to_async(bookmarks_delete)),
                    ),
            ),
        )
    })
    .bind("127.0.0.1:8081")?
    .run()
}
```

1.  那么，模型在哪里？它们在自己的文件 `src/models.rs` 中。创建它并添加以下内容：

```rs
use crate::schema::bookmarks;
use serde_derive::Serialize;

#[derive(Debug, Clone, Insertable)]
#[table_name = "bookmarks"]
pub struct NewBookmark<'a> {
    pub id: &'a str,
    pub url: &'a str,
}

#[derive(Serialize, Queryable)]
pub struct Bookmark {
    pub id: String,
    pub url: String,
}
```

1.  还有另一个尚未创建的导入：`src/schema.rs`。同样创建该文件，并添加以下代码：

```rs
table! {
    bookmarks (id) {
        id -> Text,
        url -> Text,
    }
}
```

1.  如同往常，我们需要调整 `Cargo.toml` 以下载依赖项：

```rs
[dependencies]
actix-web = "1"
serde = "1"
serde_derive = "1"
env_logger = "0.6"
diesel = {version = "1.4", features = ["sqlite"] }
uuid = { version = "0.7", features = ["serde", "v4"] }
futures = "0.1"
```

1.  这应该设置好一切，以便使用 `cargo run` 运行网络服务并观察日志输出（在请求之后）：

```rs
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.16s
     Running `target/debug/orm`
[2019-07-20T19:33:33Z INFO actix_web::middleware::logger] 127.0.0.1:54560 "GET /api/bookmarks/all HTTP/1.1" 200 2 "-" "curl/7.64.0" 0.004737
[2019-07-20T19:33:52Z INFO actix_web::middleware::logger] 127.0.0.1:54722 "POST /api/bookmarks/ HTTP/1.1" 200 1 "-" "curl/7.64.0" 0.017087
[2019-07-20T19:33:55Z INFO actix_web::middleware::logger] 127.0.0.1:54750 "GET /api/bookmarks/all HTTP/1.1" 200 77 "-" "curl/7.64.0" 0.002248
[2019-07-20T19:34:11Z INFO actix_web::middleware::logger] 127.0.0.1:54890 "GET /api/bookmarks/by-id/9b2a4264-3db6-4c50-88f1-807b20b5841e HTTP/1.1" 200 77 "-" "curl/7.64.0" 0.003298
[2019-07-20T19:34:23Z INFO actix_web::middleware::logger] 127.0.0.1:54992 "DELETE /api/bookmarks/by-id/9b2a4264-3db6-4c50-88f1-807b20b5841e HTTP/1.1" 200 1 "-" "curl/7.64.0" 0.017980
[2019-07-20T19:34:27Z INFO actix_web::middleware::logger] 127.0.0.1:55030 "GET /api/bookmarks/all HTTP/1.1" 200 2 "-" "curl/7.64.0" 0.000972
```

我们可以使用 `curl` 与网络服务交互，以下是一些预期的调用和输出：

```rs
$ curl localhost:8081/api/bookmarks/all
[]⏎
$ curl -d "{\"url\":\"https://blog.x5ff.xyz\"}" -H "Content-Type: application/json" localhost:8081/api/bookmarks/
"9b2a4264-3db6-4c50-88f1-807b20b5841e"⏎
$ curl localhost:8081/api/bookmarks/all
[{"id":"9b2a4264-3db6-4c50-88f1-807b20b5841e","url":"https://blog.x5ff.xyz"}]⏎
$ curl localhost:8081/api/bookmarks/by-id/9b2a4264-3db6-4c50-88f1-807b20b5841e
[{"id":"9b2a4264-3db6-4c50-88f1-807b20b5841e","url":"https://blog.x5ff.xyz"}]⏎
$ curl -X "DELETE" localhost:8081/api/bookmarks/by-id/9b2a4264-3db6-4c50-88f1-807b20b5841e
1⏎
$ curl localhost:8081/api/bookmarks/all
[]⏎
```

让我们看看这是如何工作的。

# 它是如何工作的...

`diesel-rs` 是 Rust 最著名的数据库连接框架，它提供了快速、类型安全和易于使用的体验，用于映射数据库表。这再次得益于宏的强大功能，它能够在编译时创建零成本的抽象。然而，为了实现这一点，我们需要权衡一些事情，并且了解如何使用这个框架是很重要的。

SQLite 没有非常严格的类型系统。这就是为什么我们可以使用一个名为 text 的泛型类型来使用字符串。其他数据库可能有更细微的类型。查看 SQLite3 类型（[`www.sqlite.org/datatype3.html`](https://www.sqlite.org/datatype3.html)）以获取更多信息。

在*步骤 1*中，我们正在准备导入——没有什么特别有趣的内容，但你会注意到`models.rs`和`schema.rs`的声明。再进一步，在*步骤 2*中，我们看到一个连接字符串（实际上只是一个文件路径）常量，我们将使用它在连接函数中连接到数据库。此外，我们正在创建一个 JSON 网络服务，因此我们创建了传输对象类型`WebBookmark`。我们在*步骤 3*中创建这些处理器，一个用于添加、检索（所有和按 ID）以及删除书签实体。

所有这些处理器都返回一个`Future`对象并异步运行。虽然处理器总是异步运行的（它们是演员），但它们明确返回类型，因为它们使用同步部分连接到数据库——`diesel-rs`目前不是线程安全的。这个同步部分是通过使用`web::block`语句实现的，它返回一个映射到`Future`和适当的`HttpResponse`类型的结果。在`bookmarks_add`处理器的例子中，它返回新创建的 ID 作为 JSON 字符串，而`bookmarks_delete`返回删除影响的行数。所有处理器在发生错误时都返回 500。

如果你想了解如何使用连接池并正确管理它们，请查看`actix-web`示例中的 diesel（[`github.com/actix/examples/tree/master/diesel`](https://github.com/actix/examples/tree/master/diesel)）。它使用了 Rust 的`r2d2`包（[`github.com/sfackler/r2d2`](https://github.com/sfackler/r2d2)）。

*步骤 3*还注册了这些函数及其相应的路由。`by-id`路由接受两种不同的方法（`GET`和`DELETE`），由于`bookmarks_add`函数的异步特性，数据必须显式声明`JsonConfig`以自动解析 JSON 输入。所有注册都使用`to_async`方法完成，这使得属性方法无法使用。

只有在 *步骤 4* 和 *步骤 5* 中，我们才创建 `diesel-rs` 特定的代码。`models.rs` 是一个包含所有我们的模型文件的文件，它们都是表格中行的抽象，但 `NewBookmark` 类型负责插入新对象（`table_name` 和 `Insertable` 属性将其附加到 DSL），而 `Bookmark` 则返回给用户（diesel 的 `Queryable` 和 Serde 的 `Serialize` 使之成为可能）。`schema.rs` 包含一个宏调用，声明表名（`bookmarks`）、其主键（`id`）和列（`id` 和 `url`）及其由 diesel 理解的数据类型。还有许多其他类型；查看 diesel 对 `table!` 的深入解释 ([`diesel.rs/guides/schema-in-depth/`](https://diesel.rs/guides/schema-in-depth/))。

*步骤 6* 展示了 `diesel-rs` 与不同数据库的工作方式；所有这些都是必须声明的功能。此外，diesel 还有一个用于数据库迁移和其他有趣功能的 CLI，因此请查看其入门指南 ([`diesel.rs/guides/getting-started/`](https://diesel.rs/guides/getting-started/)) 以获取更多信息。在 *步骤 7* 中，我们最终可以运行网络服务并插入/查询一些数据。

然而，让我们继续使用 ORM 框架进行更高级的操作。

# 使用 ORM 运行高级查询

ORM 的一个主要缺点通常是执行非常规路径操作时的复杂性。SQL——关系数据库使用的语言——是标准化的，但其类型并不总是与应用程序所做的工作兼容。在本食谱中，我们将探讨在 Rust 的 `diesel-rs` 中运行更高级查询的几种方法。

# 准备工作

使用 `cargo new advanced-orm` 创建一个 Rust 二进制项目，并确保本地主机可以访问端口 `8081`。要访问服务，可以使用 `curl` 或 Postman 等程序执行 `POST`、`GET` 等类型的网络请求，以及用于创建和管理 SQLite ([`www.sqlite.org/index.html`](https://www.sqlite.org/index.html)) 数据库的程序（例如，sqlitebrowser: [`github.com/sqlitebrowser/sqlitebrowser`](https://github.com/sqlitebrowser/sqlitebrowser)）。

如果您确保更新数据库表，则可以重用并扩展前一个食谱（*使用 ORM 将数据保存到数据库中*）中的代码。

使用 SQLite 数据库管理器，在文件夹 `db` 中创建一个新的数据库，名为 `bookmarks.sqlite`。然后，添加符合以下模式的表：

```rs
CREATE TABLE bookmarks(id TEXT PRIMARY KEY, url TEXT, added TEXT);
CREATE TABLE comments(id TEXT PRIMARY KEY, bookmark_id TEXT, comment TEXT);
```

接下来，我们将在项目中使用 `libsqlite3` 库和头文件。在 Linux、WSL 和 macOS 上，从软件包仓库安装适当的软件包。在 Ubuntu 和 WSL 上，可以使用类似 `apt-get install libsqlite3-dev` 的命令。

原生的 Windows 10 用户可能需要从 [`www.sqlite.org/download.html`](https://www.sqlite.org/download.html) 下载 `dll` 二进制文件并将它们放置到项目目录中。然而，使用 Linux/macOS 被高度推荐。

最后，使用 VS Code 打开整个目录。

# 如何操作...

只需几个步骤就可以使用模板：

1.  `src/main.rs`将包含处理程序和主函数。让我们先添加一些辅助类型和函数：

```rs
#[macro_use]
extern crate diesel;
mod models;
mod schema;

use actix_web::{middleware, web, App, Error, HttpResponse, HttpServer};

use std::env;

use crate::schema::{date, julianday};
use chrono::prelude::*;
use diesel::prelude::*;
use diesel::sqlite::SqliteConnection;
use futures::Future;
use serde_derive::{Deserialize, Serialize};
```

在一些导入之后，让我们设置辅助工具：

```rs
// Helpers
const SQLITE_DB_URL: &str = "db/bookmarks.sqlite";

#[derive(Debug, Serialize, Deserialize)]
struct WebBookmark {
    url: String,
    comment: Option<String>,
}

#[derive(Debug, Serialize, Deserialize)]
struct WebBookmarkResponse {
    id: String,
    added: String,
    url: String,
    comment: Option<String>,
}

fn connect(db_url: &str) -> SqliteConnection {
    SqliteConnection::establish(&SQLITE_DB_URL).expect(&format!("Error connecting to {}", db_url))
}
```

1.  一个新的处理程序将检索具有儒略日期的书签。让我们添加它以及一些其他众所周知的处理程序：

```rs
fn bookmarks_as_julian_by_date(
    at: web::Path<(String)>,
    ) -> impl Future<Item = HttpResponse, Error = Error> {
    web::block(move || {
        use self::schema::bookmarks::dsl::*;
        let conn = connect(&SQLITE_DB_URL);
        bookmarks
            .select((id, url, julianday(added)))
            .filter(date(added).eq(at.as_str()))
            .load::<models::JulianBookmark>(&conn)
    })
    .then(|res| match res {
        Ok(obj) => Ok(HttpResponse::Ok().json(obj)),
        Err(_) => Ok(HttpResponse::InternalServerError().into()),
    })
}
```

添加书签是这些众所周知的处理程序之一：

```rs
fn bookmarks_add(
    bookmark: web::Json<WebBookmark>,
    ) -> impl Future<Item = HttpResponse, 
    Error = Error> {
    web::block(move || {
        use self::schema::bookmarks::dsl::*;
        use self::schema::comments::dsl::*;

        let conn = connect(&SQLITE_DB_URL);
        let new_id = format!("{}", uuid::Uuid::new_v4());
        let now = Utc::now().to_rfc3339();
        let new_bookmark = models::NewBookmark {
            id: &new_id,
            url: &bookmark.url,
            added: &now,
        };

        if let Some(comment_) = &bookmark.comment {
            let new_comment_id = format!("{}", 
            uuid::Uuid::new_v4());
            let new_comment = models::NewComment {
                comment_id: &new_comment_id,
                bookmark_id: &new_id,
                comment: &comment_,
            };
            let _ = diesel::insert_into(comments)
                .values(&new_comment)
                .execute(&conn);
        }

        diesel::insert_into(bookmarks)
            .values(&new_bookmark)
            .execute(&conn)
            .map(|_| new_id)
    })
    .then(|res| match res {
        Ok(obj) => Ok(HttpResponse::Ok().json(obj)),
        Err(_) => Ok(HttpResponse::InternalServerError().into()),
    })
}
```

接下来，删除书签是一个重要的处理程序：

```rs
fn bookmarks_delete(
    req_id: web::Path<(String)>,
    ) -> impl Future<Item = HttpResponse, Error = Error> {
    web::block(move || {
        use self::schema::bookmarks::dsl::*;
        use self::schema::comments::dsl::*;

        let conn = connect(&SQLITE_DB_URL);
        diesel::delete(bookmarks.filter(id.eq(req_id.as_str())))
            .execute(&conn)
            .and_then(|_| {
                diesel::delete(comments.filter(bookmark_id.eq
                (req_id.as_str()))).execute(&conn)
            })
    })
    .then(|res| match res {
        Ok(obj) => Ok(HttpResponse::Ok().json(obj)),
        Err(_) => Ok(HttpResponse::InternalServerError().into()),
    })
}
```

1.  现在我们能够添加和删除评论和书签，我们只需要一次性检索它们：

```rs
fn all_bookmarks() -> impl Future<Item = HttpResponse, Error = Error> {
    web::block(move || {
        use self::schema::bookmarks::dsl::*;
        use self::schema::comments::dsl::*;

        let conn = connect(&SQLITE_DB_URL);
        bookmarks
            .left_outer_join(comments)
            .load::<(models::Bookmark, Option<models::Comment>)>
            (&conn)
            .map(
                |bookmarks_: Vec<(models::Bookmark, 
                Option<models::Comment>)>| {
                    let responses: Vec<WebBookmarkResponse> = 
                    bookmarks_
                        .into_iter()
                        .map(|(b, c)| WebBookmarkResponse {
                            id: b.id,
                            url: b.url,
                            added: b.added,
                            comment: c.map(|c| c.comment),
                        })
                        .collect();
                    responses
                },
            )
    })
    .then(|res| match res {
        Ok(obj) => Ok(HttpResponse::Ok().json(obj)),
        Err(_) => Ok(HttpResponse::InternalServerError().into()),
    })
}
```

最后，我们在`main()`中连接一切：

```rs
fn main() -> std::io::Result<()> {
    env::set_var("RUST_LOG", "actix_web=debug");
    env_logger::init();
    HttpServer::new(move || {
        App::new().wrap(middleware::Logger::default()).service(
            web::scope("/api").service(
                web::scope("/bookmarks")
                    .service(web::resource("/all").route
                    (web::get().to_async(all_bookmarks)))
                    .service(
                        web::resource("added_on/{at}/julian")
                            .route(web::get().to_async
                            (bookmarks_as_julian_by_date)),
                    )
                    .service(
                        web::resource("/")
                            .data(web::JsonConfig::default())
                            .route(web::post().to_async
                            (bookmarks_add)),
                    )
                    .service(
                        web::resource("by-                    
                        id/{id}").route(web::delete().
                        to_async(bookmarks_delete)),
                    ),
            ),
        )
    })
    .bind("127.0.0.1:8081")?
    .run()
}
```

1.  为了将评论与书签一起保存，我们不得不扩展模式和模型。使用以下内容创建（或编辑）`src/schema.rs`：

```rs
use diesel::sql_types::Text;
joinable!(comments -> bookmarks (bookmark_id));
allow_tables_to_appear_in_same_query!(comments, bookmarks);

sql_function! {
    fn julianday(t: Text) -> Float;
}
sql_function! {
    fn date(t: Text) -> Text;
}

table! {
    bookmarks (id) {
        id -> Text,
        url -> Text,
        added -> Text,
    }
}

table! {
    comments (comment_id) {
        comment_id -> Text,
        bookmark_id -> Text,
        comment -> Text,
    }
}
```

1.  接下来，创建或更新`src/models.rs`以创建这些类型的 Rust 表示：

```rs
use crate::schema::{bookmarks, comments};
use serde_derive::Serialize;

#[derive(Debug, Clone, Insertable)]
#[table_name = "bookmarks"]
pub struct NewBookmark<'a> {
    pub id: &'a str,
    pub url: &'a str,
    pub added: &'a str,
}

#[derive(Debug, Serialize, Queryable)]
pub struct Bookmark {
    pub id: String,
    pub url: String,
    pub added: String,
}

#[derive(Serialize, Queryable)]
pub struct JulianBookmark {
    pub id: String,
    pub url: String,
    pub julian: f32,
}

#[derive(Debug, Serialize, Queryable)]
pub struct Comment {
    pub bookmark_id: String,
    pub comment_id: String,
    pub comment: String,
}

#[derive(Debug, Clone, Insertable)]
#[table_name = "comments"]
pub struct NewComment<'a> {
    pub bookmark_id: &'a str,
    pub comment_id: &'a str,
    pub comment: &'a str,
}
```

1.  为了导入依赖项，我们还需要调整`Cargo.toml`：

```rs
[dependencies]
actix-web = "1"
serde = "1"
serde_derive = "1"
env_logger = "0.6"
diesel = {version = "1.4", features = ["sqlite"] }
uuid = { version = "0.7", features = ["serde", "v4"] }
futures = "0.1"
chrono = "0.4"
```

1.  为了完成这个食谱，让我们看看如何使用`cargo run`和`curl`一起工作。请求应该按照以下日志输出做出响应：

```rs
$ curl http://localhost:8081/api/bookmarks/all
[]⏎
$ curl -d "{\"url\":\"https://blog.x5ff.xyz\"}" -H "Content-Type: application/json" localhost:8081/api/bookmarks/
"db5538f4-e2f9-4170-bc38-02af42e6ef59"⏎
$ curl -d "{\"url\":\"https://www.packtpub.com\", \"comment\": \"Great books\"}" -H "Content-Type:       
  application/json" localhost:8081/api/bookmarks/
"5648b8c3-635e-4d55-9592-d6dfab59b32d"⏎
$ curl http://localhost:8081/api/bookmarks/all
[{
    "id": "db5538f4-e2f9-4170-bc38-02af42e6ef59",
    "added": "2019-07-23T10:32:51.020749289+00:00",
    "url": "https://blog.x5ff.xyz",
    "comment": null
 },
 {
    "id": "5648b8c3-635e-4d55-9592-d6dfab59b32d",
    "added": "2019-07-23T10:32:59.899292263+00:00",
    "url": "https://www.packtpub.com",
    "comment": "Great books"
 }]⏎
$ curl http://localhost:8081/api/bookmarks/added_on/2019-07-23/julian
[{
    "id": "db5538f4-e2f9-4170-bc38-02af42e6ef59",
    "url": "https://blog.x5ff.xyz",
    "julian": 2458688.0
},
{
    "id": "5648b8c3-635e-4d55-9592-d6dfab59b32d",
    "url": "https://www.packtpub.com",
    "julian": 2458688.0
}]⏎ 
```

这里是请求生成的服务器日志，打印到`cargo run`运行的终端中：

```rs
$ cargo run
   Compiling advanced-orm v0.1.0 (Rust-Cookbook/Chapter08/advanced-orm)
 Finished dev [unoptimized + debuginfo] target(s) in 4.75s
 Running `target/debug/advanced-orm`
[2019-07-23T10:32:36Z INFO actix_web::middleware::logger] 127.0.0.1:39962 "GET /api/bookmarks/all HTTP/1.1" 200 2 "-" "curl/7.64.0" 0.004323
[2019-07-23T10:32:51Z INFO actix_web::middleware::logger] 127.0.0.1:40094 "POST /api/bookmarks/ HTTP/1.1" 200 38 "-" "curl/7.64.0" 0.018222
[2019-07-23T10:32:59Z INFO actix_web::middleware::logger] 127.0.0.1:40172 "POST /api/bookmarks/ HTTP/1.1" 200 38 "-" "curl/7.64.0" 0.025890
[2019-07-23T10:33:06Z INFO actix_web::middleware::logger] 127.0.0.1:40226 "GET /api/bookmarks/all HTTP/1.1" 200 287 "-" "curl/7.64.0" 0.001803
[2019-07-23T10:34:18Z INFO actix_web::middleware::logger] 127.0.0.1:40844 "GET /api/bookmarks/added_on/2019-07-23/julian HTTP/1.1" 200 194 "-" "curl/7.64.0" 0.001653
```

在幕后，有很多事情在进行。让我们找出发生了什么。

# 它是如何工作的...

使用`diesel-rs`需要很好地理解其内部工作原理，以实现预期的结果。查看之前的食谱（*使用 ORM 将数据保存到数据库中*）以了解基础知识的一些细节。在这个食谱中，我们直接深入更高级的内容。

在*步骤 1*的一些基本设置之后，*步骤 2*创建了一个新的处理程序，该处理程序检索特定日期添加的所有书签，并返回儒略日期（[`en.wikipedia.org/wiki/Julian_day`](https://en.wikipedia.org/wiki/Julian_day)）。计算是使用 SQLite 的几个标量函数之一：`juliandate()`（[`www.sqlite.org/lang_datefunc.html`](https://www.sqlite.org/lang_datefunc.html)）。那么，我们是如何将这个函数引入 Rust 的呢？*步骤 4*展示了`diesel-rs`的方式：通过使用`sql_function!`宏（[`docs.diesel.rs/diesel/macro.sql_function.html`](https://docs.diesel.rs/diesel/macro.sql_function.html)）来适当地映射数据类型和输出。由于我们在这里映射的是一个预存在的函数，因此不需要进一步的操作（这应该与存储过程相同）。

**步骤 2**涵盖的另一个方面是向多个表插入和从多个表删除，这得益于 SQLite 禁用的引用完整性约束（[`www.w3resource.com/sql/joins/joining-tables-through-referential-integrity.php`](https://www.w3resource.com/sql/joins/joining-tables-through-referential-integrity.php)）。如果此约束被强制执行，请查看`diesel-rs`事务（[`docs.diesel.rs/diesel/connection/trait.Connection.html#method.transaction`](https://docs.diesel.rs/diesel/connection/trait.Connection.html#method.transaction)）。**步骤 3**继续展示如何检索这些数据——使用左外连接。左连接从左侧（如果连接看起来如下：`bookmarks LEFT JOIN comments`则为`bookmarks`）的每一行开始，并尝试与右侧表中的行匹配，这意味着无论是否有评论，我们都会得到每个书签。为了映射这个结果集，我们必须提供一个相应的数据类型来解析，`diesel-rs`期望这个类型是`(Bookmark, Option<Comment>)`。由于`left_join()`调用没有提到要连接哪些列，框架是如何知道的？再次，在**步骤 4**中，我们通过两个宏声明这两个表为`joinable`：`joinable`（[`docs.diesel.rs/diesel/macro.joinable.html`](https://docs.diesel.rs/diesel/macro.joinable.html)）和`allow_tables_to_appear_in_same_query`（[`docs.diesel.rs/diesel/macro.allow_tables_to_appear_in_same_query.html`](https://docs.diesel.rs/diesel/macro.allow_tables_to_appear_in_same_query.html)）。结果检索后，我们将它们映射到一个`Serializable`组合类型，以隐藏从用户那里隐藏这个实现细节。

只有在**步骤 4**和**步骤 5**中，我们才负责绘制柴油数据库表和行的映射——这里没有什么令人惊讶的。`Queryable`属性对于`diesel-rs`将元组映射到类型非常重要——无论实际表是什么。对于更具体的查询，我们也可以直接与元组一起工作。**步骤 6**负责处理依赖关系。

**步骤 7**运行服务器，细心的读者会注意到一件事：编译时间比平时长。我们怀疑`diesel-rs`在幕后做了很多工作，创建了类型安全的代码，以保持动态运行时开销低。然而，这可能会显著影响更大的项目，但一旦编译，类型有助于避免错误并使服务顺利运行。

我们格式化了`curl`输出，使其更易于阅读，并且输出工作正如预期。`serde`提供了 JSON 对象的持续序列化和反序列化；因此，输入时`comment`字段是可选的，但在输出时呈现为`null`。

虽然 `diesel-rs` 尝试抽象许多数据库操作，但它使用 `sql_query` 接口（[`docs.diesel.rs/diesel/fn.sql_query.html`](https://docs.diesel.rs/diesel/fn.sql_query.html)）与其他 SQL 语句一起工作。然而，更复杂的分组聚合操作尚不支持——即使在原始 SQL 接口中——这是令人遗憾的。您可以在 GitHub 上跟踪进度（[`github.com/diesel-rs/diesel/issues/210`](https://github.com/diesel-rs/diesel/issues/210)）。

现在我们对使用 `diesel-rs` 运行查询有了更多了解，让我们继续下一个菜谱。

# 网络上的身份验证

在公共接口上安全地运行网络服务本身就是一项挑战，需要考虑很多事情。虽然许多细节属于安全工程师的工作描述，但开发人员应至少遵守一组最佳实践，以便他们能够赢得用户的信任。一开始，是传输加密（TLS），由于反向代理和负载均衡器提供了惊人的简单集成（以及 let's encrypt：[`letsencrypt.org/`](https://letsencrypt.org/) 提供免费证书），我们在这个章节中没有包含任何菜谱。本章重点介绍使用 `actix-web` 中间件基础设施在应用层通过 JWT ([`jwt.io/`](https://jwt.io/)) 验证请求。

# 准备工作

使用 `cargo new authentication` 创建一个 Rust 二进制项目，并确保端口 `8081` 可以从本地主机访问。要访问服务，获取一个程序，如 `curl` 或 Postman，以执行 `POST`、`GET` 和更多类型的网络请求。

最后，使用 VS Code 打开整个目录。

# 如何做到这一点...

只需几个步骤即可验证用户身份：

1.  在 `src/main.rs` 中，我们首先声明所需的导入：

```rs
#[macro_use]
extern crate actix_web;
mod middlewares;
use actix_web::{http, middleware, web, App, HttpResponse, HttpServer, Responder};
use jsonwebtoken::{encode, Header};
use middlewares::Claims;
use serde_derive::{Deserialize, Serialize};
use std::env;
```

1.  在处理完这些之后，我们可以关注更相关的部分。让我们声明一些用于身份验证和处理程序的基本内容，我们想要访问：

```rs
const PASSWORD: &str = "swordfish";
pub const TOKEN_SECRET: &str = "0fd2af6f";

#[derive(Debug, Serialize, Deserialize)]
struct Login {
    password: String,
}

#[get("/secret")]
fn authed() -> impl Responder {
    format!("Congrats, you are authenticated")
}
```

1.  接下来，我们需要一个处理程序来登录用户并在他们提供预期的密码时创建令牌，以及设置一切的 `main()` 函数：

```rs
fn login(login: web::Json<Login>) -> HttpResponse {
    // TODO: have a proper security concept
    if &login.password == PASSWORD {
        let claims = Claims {
            user_id: "1".into(),
        };
        encode(&Header::default(), &claims, TOKEN_SECRET.as_ref())
            .map(|token| {
                HttpResponse::Ok()
                    .header(http::header::AUTHORIZATION, format!
                    ("Bearer {}", token))
                    .finish()
            })
            .unwrap_or(HttpResponse::InternalServerError().into())
    } else {
        HttpResponse::Unauthorized().into()
    }
}

fn main() -> std::io::Result<()> {
    env::set_var("RUST_LOG", "actix_web=debug");
    env_logger::init();
    HttpServer::new(|| {
        App::new()
            .wrap(middleware::Logger::default())
            .wrap(middlewares::JwtLogin)
            .service(authed)
            .service(web::resource("/login").route(web::post().to(login)))
    })
    .bind("127.0.0.1:8081")?
    .run()
}
```

1.  `main()` 函数中的 `wrap()` 调用已经透露了一些细节——我们需要中间件来处理身份验证。让我们创建一个新文件，`src/middlewares.rs`，并包含以下代码：

```rs
use actix_service::{Service, Transform};
use actix_web::dev::{ServiceRequest, ServiceResponse};
use actix_web::{http, Error, HttpResponse};
use futures::future::{ok, Either, FutureResult};
use futures::Poll;
use jsonwebtoken::{decode, Validation};
use serde_derive::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
pub struct Claims {
    pub user_id: String,
}

pub struct JwtLogin;

impl<S, B> Transform<S> for JwtLogin
where
    S: Service<Request = ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
{
    type Request = ServiceRequest;
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = JwtLoginMiddleware<S>;
    type Future = FutureResult<Self::Transform, Self::InitError>;

    fn new_transform(&self, service: S) -> Self::Future {
        ok(JwtLoginMiddleware { service })
    }
}
```

1.  在 *步骤 4* 的代码中，我们看到另一个需要实现的 struct：`JwtLoginMiddleware`。让我们将其添加到 `src/middlewares.rs`：

```rs
pub struct JwtLoginMiddleware<S> {
    service: S,
}

impl<S, B> Service for JwtLoginMiddleware<S>
where
    S: Service<Request = ServiceRequest, Response = 
    ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
{
    type Request = ServiceRequest;
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = Either<S::Future, FutureResult<Self::Response, 
    Self::Error>>;

    fn poll_ready(&mut self) -> Poll<(), Self::Error> {
        self.service.poll_ready()
    }
```

最重要的代码可以在调用函数实现中找到，其中请求通过应用中间件（并验证令牌）传递：

```rs
    fn call(&mut self, req: ServiceRequest) -> Self::Future {
        if req.path() == "/login" {
            Either::A(self.service.call(req))
        } else {
            if let Some(header_value) = 
            req.headers().get(http::header::AUTHORIZATION) {
                let token = header_value.to_str().unwrap().
                replace("Bearer", "");
                let mut validation = Validation::default();
                validation.validate_exp = false; // our logins don't 
                // expire
                if let Ok(_) =
                    decode::<Claims>(&token.trim(), 
                    crate::TOKEN_SECRET.as_ref(), &validation)
                {
                    Either::A(self.service.call(req))
                } else {
                    Either::B(ok(
                        req.into_response(HttpResponse::Unauthorized()
                        .finish().into_body())
                    ))
                }
            } else {
                Either::B(ok(
                    req.into_response(HttpResponse::Unauthorized().
                    finish().into_body())
                ))
            }
        }
    }
}
```

1.  在我们可以运行服务器之前，我们还需要更新 `Cargo.toml` 以包含当前依赖项：

```rs
[dependencies]
actix-web = "1"
serde = "1"
serde_derive = "1"
env_logger = "0.6"
jsonwebtoken = "6"
futures = "0.1"
actix-service = "0.4"
```

1.  激动人心的——让我们试试吧！使用 `cargo run` 启动服务器并发出一些 `curl` 请求：

```rs
$ cargo run
   Compiling authentication v0.1.0 (Rust-Cookbook/Chapter08/authentication)
    Finished dev [unoptimized + debuginfo] target(s) in 6.07s
     Running `target/debug/authentication`
[2019-07-22T21:28:07Z INFO actix_web::middleware::logger] 127.0.0.1:33280 "POST /login HTTP/1.1" 401 0 "-" "curl/7.64.0" 0.009627
[2019-07-22T21:28:13Z INFO actix_web::middleware::logger] 127.0.0.1:33334 "POST /login HTTP/1.1" 200 0 "-" "curl/7.64.0" 0.009191
[2019-07-22T21:28:21Z INFO actix_web::middleware::logger] 127.0.0.1:33404 "GET /secret HTTP/1.1" 200 31 "-" "curl/7.64.0" 0.000784

```

以下是对每个请求的 `curl` 输出。首先，未授权的请求：

```rs
$ curl -v localhost:8081/secret
* Trying ::1...
* TCP_NODELAY set
* connect to ::1 port 8081 failed: Connection refused
* Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8081 (#0)
> GET /secret HTTP/1.1
> Host: localhost:8081
> User-Agent: curl/7.64.0
> Accept: */*
> 
< HTTP/1.1 401 Unauthorized
< content-length: 0
< date: Mon, 22 Jul 2019 21:27:48 GMT
< 
* Connection #0 to host localhost left intact
```

接下来，我们尝试使用无效的密码登录：

```rs
$ curl -d "{\"password\":\"a-good-guess\"}" -H "Content-Type: application/json" 
  http://localhost:8081/login -v
* Trying ::1...
* TCP_NODELAY set
* connect to ::1 port 8081 failed: Connection refused
* Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8081 (#0)
> POST /login HTTP/1.1
> Host: localhost:8081
> User-Agent: curl/7.64.0
> Accept: */*
> Content-Type: application/json
> Content-Length: 27
> 
* upload completely sent off: 27 out of 27 bytes
< HTTP/1.1 401 Unauthorized
< content-length: 0
< date: Mon, 22 Jul 2019 21:28:07 GMT
< 
* Connection #0 to host localhost left intact
```

然后，我们使用真实密码并接收一个令牌回来：

```rs
$ curl -d "{\"password\":\"swordfish\"}" -H "Content-Type: application/json" 
  http://localhost:8081/login -v
* Trying ::1...
* TCP_NODELAY set
* connect to ::1 port 8081 failed: Connection refused
* Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8081 (#0)
> POST /login HTTP/1.1
> Host: localhost:8081
> User-Agent: curl/7.64.0
> Accept: */*
> Content-Type: application/json
> Content-Length: 24
> 
* upload completely sent off: 24 out of 24 bytes
< HTTP/1.1 200 OK
< content-length: 0
< authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoiMSJ9.V_Po0UCGZqNmbXw0hYozeFLsNpjTZeSh8wcyELavx-c
< date: Mon, 22 Jul 2019 21:28:13 GMT
< 
* Connection #0 to host localhost left intact
```

在 `authorization` 标头 ([`developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization)) 中包含此令牌后，我们就可以访问秘密资源：

```rs

$ curl -H "authorization: Bearer 
  eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoiMSJ9.V_Po0UCGZqNmbXw0hYozeFLsNpjTZeSh8wcyELavx-  
  c" http://localhost:8081/secret -v
* Trying ::1...
* TCP_NODELAY set
* connect to ::1 port 8081 failed: Connection refused
* Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8081 (#0)
> GET /secret HTTP/1.1
> Host: localhost:8081
> User-Agent: curl/7.64.0
> Accept: */*
> authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoiMSJ9.V_Po0UCGZqNmbXw0hYozeFLsNpjTZeSh8wcyELavx-c
> 
< HTTP/1.1 200 OK
< content-length: 31
< content-type: text/plain; charset=utf-8
< date: Mon, 22 Jul 2019 21:28:21 GMT
< 
* Connection #0 to host localhost left intact
Congrats, you are authenticated⏎
```

让我们揭开面纱，看看它是如何工作的。

# 它是如何工作的...

JWT 是在 Web 应用程序中提供身份验证和授权的绝佳方式。正如官方网站所示，JWT 由三部分组成：

+   标题，提供关于标记的元信息

+   其负载，即信息被发送的地方（JSON 序列化）

+   一个签名来保证令牌在传输过程中未被更改

这些部分是 Base64 编码的，并用 `.` 连接成一个字符串。这个字符串被放入 HTTP 请求的 `authorization` 标头 ([`developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization)) 中。一个重要的注意事项是，对于这种身份验证，TLS 是强制性的，因为标头以及所有其他内容都是以明文发送的——每个人都能看到令牌。

负载数据可以包含您希望携带来回的用户信息。然而，也有一些特殊字段：`iss`、`sub` 和 `exp`。`iss` 提供了发行者的凭证（以任何方式），`sub` 是主题，而 `exp` 是过期时间戳。这是因为 JWT 可以用于通过联盟进行身份验证，即第三方服务。对于此实现，我们使用一个名为 `jsonwebtoken` 的 crate ([`github.com/Keats/jsonwebtoken`](https://github.com/Keats/jsonwebtoken))。

在 *步骤 1* 中，我们只是设置导入——这里没有什么特别之处。只有 *步骤 2* 提供了一些有趣的内容：一个硬编码的密码（**不良**的安全实践，但足以用于演示）以及一个硬编码的秘密（也是 **不良**）。真实的应用程序可以使用秘密存储库来存储秘密（例如，Azure Key Vault：[`azure.microsoft.com/en-in/services/key-vault/`](https://azure.microsoft.com/en-in/services/key-vault/))，并在数据库中存储密码的散列。在相同的步骤中，我们还声明了登录的输入数据结构——我们只关心密码——以及路径/秘密的处理程序，它应该在登录后才能工作。

以下步骤创建登录处理程序：如果密码匹配，处理程序将创建一个新的包含有效载荷数据（一个名为 `Claims` 的结构体）和用于签名令牌的 HMAC（默认为 HS256）算法的令牌，并将其返回。然后，处理程序与 `App` 实例一起注册，包括在以下步骤中实现的新的 JWT 认证中间件。

*第 4 步* 和 *第 5 步* 负责创建用于验证 JWT 令牌的中间件。*第 4 步* 包含之前提到的 `Claims` 类型；然而，如果请求和响应类型保持默认，则其余代码主要是必需的样板代码。如果我们想检索用户信息传递给处理程序，我们将考虑定义自定义请求。只有在 *第 5 步* 中，我们才实现重要的部分：`call()` 函数。这个函数在每次请求处理之前被调用，并决定是否继续或停止传播它。显然，`/login` 路由是例外，总是会传递给处理程序。

每个其他路由都必须包含一个名为 `authorization` 的头字段和一个名为 `Bearer` 的类型，以及令牌，例如，（截断）`authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJ[...]8wcyELavx-c`。`call()` 函数提取令牌并尝试使用其密钥对其进行解码。如果成功，则将调用传递给处理程序；如果不成功，则用户显然无权访问资源——如果没有授权头也会发生这种情况。`jsonwebtoken` 默认验证 `exp` 字段（我们的 `Claims` 类型没有这个），这就是我们在这个示例中关闭的原因。为了简洁起见，我们在解析头字节的字符串时使用了 `unwrap()`。然而，如果遇到未知字节，这可能会使线程崩溃。

这里返回的类型是从 `futures` 库（[`docs.rs/futures/`](https://docs.rs/futures/)）导入的，并提供 `Either` 类型（[`docs.rs/futures/0.1.28/futures/future/enum.Either.html`](https://docs.rs/futures/0.1.28/futures/future/enum.Either.html)）以及 `ok()` 函数（[`docs.rs/futures/0.1.28/futures/future/fn.ok.html`](https://docs.rs/futures/0.1.28/futures/future/fn.ok.html)）。查看它们的文档以了解更多信息。

*第 6 步* 仅声明额外的依赖项，而在 *第 7 步* 中，我们可以运行服务器！首先检查 `curl` 请求——你能看到什么问题吗？在记录发生之前，阻止未经授权的请求。此外，我们已用粗体标记了重要部分。

这就结束了这一章。我们希望您喜欢网络编程食谱。下一章将涵盖更接近底层的内容：系统编程。
