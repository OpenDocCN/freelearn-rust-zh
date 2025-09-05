# 第九章：网络编程

在本章中，我们将介绍以下食谱：

+   设置基本 HTTP 服务器

+   配置 HTTP 服务器以执行回声和路由

+   配置 HTTP 服务器以执行文件服务

+   向 API 发送请求

+   设置基本的 UDP 套接字

+   配置 UDP 套接字以执行回声

+   通过 TLS 设置安全连接

# 简介

通过互联网，世界每天都在变得更小。网络以惊人的方式连接着人们。无数服务免费供您使用。数百万的人可以免费使用您的应用程序，甚至不需要安装它。

作为想要利用这一点的开发者，如果您以干净的方式设置了您的架构，将您的应用程序移植到互联网上可以非常简单。您唯一需要更改的是与外界交互的层。

本章将向您展示如何通过允许您的应用程序接受请求、响应它们，并展示如何创建对其他 Web 服务的请求来创建这一层。

# 设置基本 HTTP 服务器

让我们从将著名的 Hello World 程序带入 21 世纪，通过在服务器上托管它来开始我们的章节。我们将使用`hyper` crate 来完成这个任务，它是一个围绕所有 HTTP 事物的强类型包装器。除了是世界上速度最快的 HTTP 实现之一（[`www.techempower.com/benchmarks/#section=data-r15&hw=ph&test=plaintext`](https://www.techempower.com/benchmarks/#section=data-r15&hw=ph&test=plaintext)），它还被几乎所有主要的高级框架使用（[`github.com/flosse/rust-web-framework-comparison#high-level-frameworks`](https://github.com/flosse/rust-web-framework-comparison#high-level-frameworks)），唯一的例外是那些在 Rust 提供的`std::net::TcpStream`极其基本的*字符串类型*库上重新实现了它的框架。

# 准备工作

所有`hyper`食谱都与`futures`兼容，因此在继续之前，您应该阅读所有第八章，*与 Futures 一起工作*。

在撰写本文时，`hyper`尚未升级到`futures v0.2`（跟踪问题：[`github.com/hyperium/hyper/issues/1448`](https://github.com/hyperium/hyper/issues/1448)），因此我们将使用`futures v0.1`。这应该在未来没有问题（没有打趣的意思），因为所有相关代码都是按照应该与`0.2`兼容的方式编写的。

如果某些意外的 API 更改破坏了食谱，您可以在本书的 GitHub 仓库中找到它们的修复版本（[`github.com/jnferner/rust-standard-library-cookbook/tree/master/chapter-nine/src/bin`](https://github.com/jnferner/rust-standard-library-cookbook/tree/master/chapter-nine/src/bin)），它将始终更新以与所有库的最新版本兼容。

# 如何操作...

1.  使用`cargo new chapter-nine`创建一个 Rust 项目，在本章中对其进行工作。

1.  导航到新创建的 `chapter-nine` 文件夹。在本章的其余部分，我们将假设您的命令行当前位于此目录

1.  打开为您生成的 `Cargo.toml` 文件

1.  在 `[dependencies]` 下，添加以下行：

```rs
futures = "0.1.18"
hyper = "0.11.21"
```

1.  如果您愿意，可以访问 futures' ([`crates.io/crates/futures`](https://crates.io/crates/futures)) 和 hyper's ([`crates.io/crates/hyper`](https://crates.io/crates/hyper)) 的 *crates.io* 页面，检查最新版本并使用它

1.  在文件夹 `src` 内，创建一个名为 `bin` 的新文件夹

1.  删除生成的 `lib.rs` 文件，因为我们没有创建库

1.  在文件夹 `src/bin` 中，创建一个名为 `hello_world_server.rs` 的文件

1.  添加以下代码，并用 `cargo run --bin hello_world_server` 运行它：

```rs
1   extern crate futures;
2   extern crate hyper;
3   
4   use futures::future::Future;
5   use hyper::header::{ContentLength, ContentType};
6   use hyper::server::{const_service, service_fn, Http, Request, Response, Service};
7   use std::net::SocketAddr;
8   
9   const MESSAGE: &str = "Hello World!";
10  
11  fn main() {
12    // [::1] is the loopback address for IPv6, 3000 is a port
13    let addr = "[::1]:3000".parse().expect("Failed to parse address");
14    run_with_service_function(&addr).expect("Failed to run web server");
15  }
```

通过使用 `service_fn` 创建服务来运行服务器：

```rs
17  fn run_with_service_function(addr: &SocketAddr) -> Result<(), 
     hyper::Error> {
18    // Hyper is based on Services, which are construct that
19    // handle how to respond to requests.
20    // const_service and service_fn are convenience functions
21    // that build a service out of a closure
22    let hello_world = const_service(service_fn(|_| {
23      println!("Got a connection!");
24      // Return a Response with a body of type hyper::Body
25      Ok(Response::::new()
26        // Add header specifying content type as plain text
27        .with_header(ContentType::plaintext())
28        // Add header specifying the length of the message in 
          bytes
29        .with_header(ContentLength(MESSAGE.len() as u64))
30        // Add body with our message
31        .with_body(MESSAGE))
32    }));
33  
34    let server = Http::new().bind(addr, hello_world)?;
35    server.run()
36  }
```

通过手动创建一个实现 `Service` 的 `struct` 来运行服务器：

```rs
38  // The following function does the same, but uses an explicitely 
    created
39  // struct HelloWorld that implements the Service trait
40  fn run_with_service_struct(addr: &SocketAddr) -> Result<(), 
     hyper::Error> {
41    let server = Http::new().bind(addr, || Ok(HelloWorld))?;
42    server.run()
43  }
44  
45  struct HelloWorld;
46  impl Service for HelloWorld {
47    // Implementing a server requires specifying all involved 
      types
48    type Request = Request;
49    type Response = Response;
50    type Error = hyper::Error;
51    // The future that wraps your eventual Response
52    type Future = Box<Future>;
53  
54    fn call(&self, _: Request) -> Self::Future {
55      // In contrast to service_fn, we need to explicitely return 
        a future
56      Box::new(futures::future::ok(
57        Response::new()
58          .with_header(ContentType::plaintext())
59          .with_header(ContentLength(MESSAGE.len() as u64))
60          .with_body(MESSAGE),
61      ))
62    }
63  }
```

# 它是如何工作的...

在 `main` 中，我们首先将表示我们的 IPv6 回环地址（例如 `localhost`）的字符串解析为 `std::net::SocketAddr`，这是一个包含 IP 地址和端口的类型 [13]。当然，我们可以使用一个常量作为我们的地址，但我们展示的是如何从字符串中解析它，因为在实际应用中，您可能需要从环境变量中获取地址，正如在 第一章 *学习基础知识*；*与环境变量交互* 中所示。

然后，我们运行在 `run_with_service_function` [17] 中创建的 `hyper` 服务器。让我们通过了解一下 `hyper` 来看看这个函数。

`hyper` 中最基础的特质是 `Service`。它被定义为如下：

```rs
pub trait Service where
    <Self::Future as Future>::Item == Self::Response,
    <Self::Future as Future>::Error == Self::Error, {
    type Request;
    type Response;
    type Error;
    type Future: Future;
    fn call(&self, req: Self::Request) -> Self::Future;
}
```

应该很容易阅读 `call` 的签名：它接受一个 `Request` 并返回一个 `Response` 的 `Future`。`hyper` 使用这个特质来响应传入的请求。我们通常有两种方式来定义一个 `Service`：

+   手动创建一个实现 `Service` 的 `struct`，显式设置其关联类型为 `call` 返回的类型

+   通过传递一个返回 `Result` 的闭包给 `service_fn`，并使用 `const_service` 包装它，让 `Service` 为您构建

这两种变体都会产生完全相同的结果，所以这个例子包含了两种版本，以便您尝尝它们的味道。

`run_with_service_function`使用第二种风格[22]。它返回一个`Response`的`Result`，`service_fn`将其转换为`Response`的`Future`，因为`Result`实现了`Future`。`service_fn`然后为我们进行一些类型推导并创建一种`Service`。但我们的任务还没有完成。您可以看到，当`hyper`接收到一个新的连接时，它不会直接用`Request`调用我们的`Service`，而是首先复制它，以便为每个连接使用其自己的`Service`。这意味着我们的`Service`必须具有创建自身新实例的能力，这由`NewService`特质表示。幸运的是，我们也不需要自己实现它。我们`Service`核心的闭包不管理任何状态，所以我们可以称它为一个常量函数。常量非常容易复制，因为所有副本都保证是相同的。我们可以通过调用`const_service`来标记我们的`Service`为常量，这基本上只是将`Service`包装在一个`Arc`中，然后通过简单地返回其副本来实现`NewService`。但我们的`Service`究竟返回了什么呢？

`Response<hyper::Body>`创建一个新的 HTTP 响应[25]，并管理其体作为`hyper::Body`，这是一个未来的`Stream<Chunk>`。`Chunk`只是 HTTP 消息的一部分。这个`Response`是一个构建器，所以我们可以通过调用各种方法来更改其内容。在我们的代码中，我们将其`Content-Type`标题设置为`plaintext`，这是`hyper`对 MIME 类型`text/plain`的快捷方式[27]。

MIME 类型是用于通过 HTTP 服务的数据的标签。它告诉客户端如何处理它接收到的数据。例如，大多数浏览器不会将消息`<p>Hello World!</p>`作为 HTML 渲染，除非它带有标题`Content-Type: text/html`。

我们还将其`Content-Length`标题设置为消息的长度（以字节为单位），以便客户端知道他们应该期望多少数据[29]。最后，我们将消息体设置为消息，然后将其作为`"Hello World!"`发送到客户端[31]。

我们的服务现在可以绑定到一个新的`hyper::server::Http`实例上，然后我们运行它[34 和 35]。现在您可以使用您选择的浏览器，将其指向`http://localhost:3000`。如果一切顺利，您应该会看到一个`Hello World!`消息。

如果我们调用`run_with_service_struct`而不是这个，它使用手动创建的`Service`，也会发生同样的事情[40]。快速检查其实现显示给我们与最后一种方法[45 到 63]的关键区别：

```rs
struct HelloWorld;
impl Service for HelloWorld {
    type Request = Request;
    type Response = Response;
    type Error = hyper::Error;
    type Future = Box<Future<Item = Self::Response, Error = 
    Self::Error>>;

    fn call(&self, _: Request) -> Self::Future {
        Box::new(futures::future::ok(
            Response::new()
                .with_header(ContentType::plaintext())
                .with_header(ContentLength(MESSAGE.len() as u64))
                .with_body(MESSAGE),
        ))
    }
}
```

如您所见，我们需要明确指定基本上所有东西的具体类型[48 到 52]。我们也不能简单地在我们的`call`方法中返回一个`Result`，而需要返回实际的`Future`，并用`Box`[56]包装它，这样我们就不需要考虑我们正在使用哪种具体的`Future`类型了。

另一方面，这种方法与另一种方法相比有一个很大的优点：它可以以成员的形式管理状态。因为本章中所有的 `hyper` 菜谱都使用常量服务，即对等请求将返回相同的 `Response` 的服务，我们将使用第一个变体来创建服务。这仅仅是一个基于简单性的风格选择，因为它们都足够小，不值得提取到自己的 `struct` 中。在你的项目中，使用最适合当前用例的形式。

# 参见

+   *使用构建器模式* 和 第一章 中的 *与环境变量交互* 菜单，*学习基础知识*

# 配置一个执行回声和路由的 HTTP 服务器

我们学习了如何永久地提供相同的响应，但过一段时间后这会变得相当无聊。在这个菜谱中，你将学习如何读取请求并单独对它们进行响应。为此，我们将使用路由来区分对不同端点的请求。

# 准备工作

要测试这个菜谱，你需要一种轻松发送 HTTP 请求的方法。一个出色的免费工具是 Postman ([`www.getpostman.com/`](https://www.getpostman.com/))，它具有一个友好且易于理解的界面。如果你不想下载任何东西，你可以使用终端来做这个。如果你在 Windows 上，你可以打开 PowerShell 并输入以下命令来进行 HTTP 请求：

```rs
Invoke-WebRequest -UseBasicParsing <Your URL> -Method <Your method in CAPSLOCK> -Body <Your message as a string>
```

因此，如果你想将消息 `hello there, my echoing friend` POST 到 `http://localhost:3000/echo`，正如菜谱中稍后所要求的那样，你需要输入以下命令：

```rs
Invoke-WebRequest -UseBasicParsing http://localhost:3000/echo -Method POST -Body "Hello there, my echoing friend"
```

在 Unix 系统上，你可以使用 cURL 来做这个 ([`curl.haxx.se/`](https://curl.haxx.se/))。相应的命令如下：

```rs
curl -X <Your method> --data <Your message> -g <Your URL>
```

cURL 将将 `localhost` 解析为其 `/etc/hosts` 中的条目。在某些配置中，这将是 IPv4 环回地址 (`127.0.0.1`)。在其他一些配置中，你必须使用 `ip6-localhost`。检查你的 `/etc/hosts` 以确定使用什么。在任何情况下，显式的 `[::1]` 总是有效的。例如，以下命令将再次将消息 `hello there, my echoing friend` POST 到 `http://localhost:3000/echo`：

```rs
curl -X POST --data "Hello there my echoing friend" -g "http://[::1]:3000/echo"
```

# 如何操作...

1.  打开为你生成的 `Cargo.toml` 文件

1.  在 `[dependencies]` 下，如果你在上一个菜谱中没有这样做，请添加以下行：

```rs
futures = "0.1.18"
hyper = "0.11.21"
```

1.  如果你愿意，你可以访问 futures' ([`crates.io/crates/futures`](https://crates.io/crates/futures)) 和 hyper's ([`crates.io/crates/hyper`](https://crates.io/crates/hyper)) 的 *crates.io* 页面，检查最新版本并使用它。

1.  在 `src/bin` 文件夹中创建一个名为 `echo_server_with_routing.rs` 的文件

1.  添加以下代码，并使用 `cargo run --bin echo_server_with_routing` 运行它：

```rs
1   extern crate hyper;
2
3   use hyper::{Method, StatusCode};
4   use hyper::server::{const_service, service_fn, Http, Request, 
    Response};
5   use hyper::header::{ContentLength, ContentType};
6   use std::net::SocketAddr;
7
8   fn main() {
9     let addr = "[::1]:3000".parse().expect("Failed to parse 
      address");
10    run_echo_server(&addr).expect("Failed to run web server");
11  }
12
13  fn run_echo_server(addr: &SocketAddr) -> Result<(),  
    hyper::Error> {
14    let echo_service = const_service(service_fn(|req: Request| {
15      // An easy way to implement routing is
16      // to simply match the request's path
17      match (req.method(), req.path()) {
18        (&Method::Get, "/") => handle_root(),
19        (&Method::Post, "/echo") => handle_echo(req),
20        _ => handle_not_found(),
21      }
22    }));
23
24    let server = Http::new().bind(addr, echo_service)?;
25    server.run()
26  }
```

函数处理路由：

```rs
28  type ResponseResult = Result<Response, hyper::Error>;
29  fn handle_root() -> ResponseResult {
30    const MSG: &str = "Try doing a POST at /echo";
31    Ok(Response::new()
32      .with_header(ContentType::plaintext())
33      .with_header(ContentLength(MSG.len() as u64))
34      .with_body(MSG))
35  }
36
37  fn handle_echo(req: Request) -> ResponseResult {
38    // The echoing is implemented by setting the response's
39    // body to the request's body
40    Ok(Response::new().with_body(req.body()))
41  }
42
43  fn handle_not_found() -> ResponseResult {
44    // Return a 404 for every unsupported route
45    Ok(Response::new().with_status(StatusCode::NotFound))
46  }
```

# 它是如何工作的...

这个菜谱与上一个菜谱类似，所以让我们直接跳到 `Service` 的定义[14 到 22]：

```rs
|req: Request| {
    match (req.method(), req.path()) {
        (&Method::Get, "/") => handle_root(),
        (&Method::Post, "/echo") => handle_echo(req),
        _ => handle_not_found(),
    }
}
```

我们现在使用的是 `Request` 参数，而上一个菜谱只是简单地忽略了它。

因为 Rust 允许我们在元组上使用模式匹配，所以我们可以直接区分 HTTP 方法与路径组合。然后我们将程序的控制流传递给专门的路线处理程序，它们反过来负责返回响应。

在具有大量路由的大程序中，我们不会在一个函数中指定所有这些，而是将它们分散在命名空间中，并将它们分成子路由器。

`handle_root` [29] 的代码看起来几乎与上一章的 hello world `Service` 相同，但指示调用者向 `/post` 路由发送 POST 请求。

对于所说的 POST，我们的匹配会导致 `handle_echo` [37]，它简单地返回请求体作为响应体 [40]。你可以通过在 *准备就绪* 部分描述的方式将消息 POST 到 `http://localhost:3000/echo` 来尝试它。如果一切顺利，你的消息会直接返回给你。

最后，但同样重要的是，当没有路由匹配时，会调用 `handle_not_found` [43]。这次，我们不发送消息，而是返回世界上可能最著名的状态码：`404 Not Found` [45]。

# 配置 HTTP 服务器以执行文件服务

上一系列的菜谱对于构建网络服务非常有用，但让我们看看如何做 HTTP 最初创建的事情：向网络提供 HTML 文件。

# 如何做到这一点...

1.  打开为你生成的 `Cargo.toml` 文件。

1.  在 `[dependencies]` 下，如果你在上一个菜谱中没有这样做，请添加以下行：

```rs
futures = "0.1.18"
hyper = "0.11.21"
```

1.  如果你想的话，你可以访问 futures' ([`crates.io/crates/futures`](https://crates.io/crates/futures)) 和 hyper's ([`crates.io/crates/hyper`](https://crates.io/crates/hyper)) 的 *crates.io* 页面，检查最新版本并使用它。

1.  在 `chapter-nine` 文件夹中创建一个名为 `files` 的文件夹。

1.  在 `files` 文件夹中创建一个名为 `index.html` 的文件，并将以下代码添加到其中：

```rs
<!doctype html>
<html>

<head>
    <link rel="stylesheet" type="text/css" href="/style.css">
    <title>Home</title>
</head>

<body>
    <h1>Home</h1>
    <p>Welcome. You can access other files on this web server  
    aswell! Available links:</p>
    <ul>
        <li>
            <a href="/foo.html">Foo!</a>
        </li>
        <li>
            <a href="/bar.html">Bar!</a>
        </li>
    </ul>
</body>

</html>
```

1.  在 `files` 文件夹中创建一个名为 `foo.html` 的文件，并将以下代码添加到其中：

```rs
<!doctype html>
<html>

<head>
    <link rel="stylesheet" type="text/css" href="/style.css">
    <title>Foo</title>
</head>

<body>
    <p>Foo!</p>
</body>

</html>
```

1.  在 `files` 文件夹中创建一个名为 `bar.html` 的文件，并将以下代码添加到其中：

```rs
<!doctype html>
<html>

<head>
    <link rel="stylesheet" type="text/css" href="/style.css">
    <title>Bar</title>
</head>

<body>
    <p>Bar!</p>
</body>

</html>
```

1.  在 `files` 文件夹中创建一个名为 `not_found.html` 的文件，并将以下代码添加到其中：

```rs
<!doctype html>
<html>

<head>
    <link rel="stylesheet" type="text/css" href="/style.css">
    <title>Page Not Found</title>
</head>

<body>
    <h1>Page Not Found</h1>
    <p>We're sorry, we couldn't find the page you requested.</p>
    <p>Maybe it was renamed or moved?</p>
    <p>Try searching at the
        <a href="/index.html">start page</a>
    </p>
</body>

</html>
```

1.  在 `files` 文件夹中创建一个名为 `invalid_method.html` 的文件，并将以下代码添加到其中：

```rs
<!doctype html>
<html>

<head>
    <link rel="stylesheet" type="text/css" href="/style.css">
    <title>Error 405 (Method Not Allowed)</title>
</head>

<body>
    <h1>Error 405</h1>
    <p>The method used is not allowed for this URL</p>
</body>

</html>
```

1.  在 `src/bin` 文件夹中创建一个名为 `file_server.rs` 的文件。

1.  添加以下代码并使用 `cargo run --bin echo_server_with_routing` 运行它：

```rs
1   extern crate futures;
2   extern crate hyper;
3
4   use hyper::{Method, StatusCode};
5   use hyper::server::{const_service, service_fn, Http, Request,  
    Response};
6   use hyper::header::{ContentLength, ContentType};
7   use hyper::mime;
8   use futures::Future;
9   use futures::sync::oneshot;
10  use std::net::SocketAddr;
11  use std::thread;
12  use std::fs::File;
13  use std::io::{self, copy};
14
15  fn main() {
16    let addr = "[::1]:3000".parse().expect("Failed to parse 
      address");
17    run_file_server(&addr).expect("Failed to run web server");
18  }
19
20  fn run_file_server(addr: &SocketAddr) -> Result<(), 
    hyper::Error> {
21    let file_service = const_service(service_fn(|req: Request| {
22      // Setting up our routes
23      match (req.method(), req.path()) {
24        (&Method::Get, "/") => handle_root(),
25        (&Method::Get, path) => handle_get_file(path),
26        _ => handle_invalid_method(),
27      }
28    }));
29
30    let server = Http::new().bind(addr, file_service)?;
31    server.run()
32  }
```

以下是一些路由处理程序：

```rs
34  // Because we don't want the entire server to block when serving 
     a file,
35  // we are going to return a response wrapped in a future
36  type ResponseFuture = Box<Future>;
37  fn handle_root() -> ResponseFuture {
38    // Send the landing page
39    send_file_or_404("index.html")
40  }
41
42  fn handle_get_file(file: &str) -> ResponseFuture {
43    // Send whatever page was requested or fall back to a 404 page
44    send_file_or_404(file)
45  }
46
47  fn handle_invalid_method() -> ResponseFuture {
48    // Send a page telling the user that the method he used is not   
      supported
49    let response_future = send_file_or_404("invalid_method.html")
50      // Set the correct status code
51      .and_then(|response| 
        Ok(response.with_status(StatusCode::MethodNotAllowed)));
52    Box::new(response_future)
53  }
```

以下是为返回我们的文件的函数编写的代码：

```rs
55  // Send a future containing a response with the requested file  
    or a 404 page
56  fn send_file_or_404(path: &str) -> ResponseFuture {
57    // Sanitize the input to prevent unwanted data access
58    let path = sanitize_path(path);
59
60    let response_future = try_to_send_file(&path)
61      // try_to_send_file returns a future of Result<Response, 
        io::Error>
62      // turn it into a future of a future of Response with an 
        error of hyper::Error
63      .and_then(|response_result| response_result.map_err(|error| 
         error.into()))
64      // If something went wrong, send the 404 page instead
65      .or_else(|_| send_404());
66    Box::new(response_future)
67  }
68
69  // Return a requested file in a future of Result<Response, 
    io::Error>
70  // to indicate whether it exists or not
71  type ResponseResultFuture = Box<Future, Error = hyper::Error>>;
72  fn try_to_send_file(file: &str) -> ResponseResultFuture {
73    // Prepend "files/" to the file
74    let path = path_on_disk(file);
75    // Load the file in a separate thread into memory.
76    // As soon as it's done, send it back through a channel
77    let (tx, rx) = oneshot::channel();
78    thread::spawn(move || {
79      let mut file = match File::open(&path) {
80        Ok(file) => file,
81        Err(err) => {
82          println!("Failed to find file: {}", path);
83          // Send error through channel
84          tx.send(Err(err)).expect("Send error on file not 
            found");
85          return;
86        }
87      };
88
89      // buf is our in-memory representation of the file
90      let mut buf: Vec = Vec::new();
91      match copy(&mut file, &mut buf) {
92        Ok(_) => {
93          println!("Sending file: {}", path);
94          // Detect the content type by checking the file 
            extension
95          // or fall back to plaintext
96          let content_type = 
            get_content_type(&path).unwrap_or_else
            (ContentType::plaintext);
97          let res = Response::new()
98            .with_header(ContentLength(buf.len() as u64))
99            .with_header(content_type)
100           .with_body(buf);
101         // Send file through channel
102         tx.send(Ok(res))
103           .expect("Send error on successful file read");
104       }
105       Err(err) => {
106         // Send error through channel
107         tx.send(Err(err)).expect("Send error on error reading 
            file");
108       }
109     };
110   });
111   // Convert all encountered errors to hyper::Error
112   Box::new(rx.map_err(|error|  
      io::Error::new(io::ErrorKind::Other, 
      error).into()))
113 }
114
115 fn send_404() -> ResponseFuture {
116   // Try to send our 404 page
117   let response_future = 
      try_to_send_file("not_found.html").and_then(|response_result|   
      {
118     Ok(response_result.unwrap_or_else(|_| {
119       // If the 404 page doesn't exist, sent fallback text  
          instead
120       const ERROR_MSG: &str = "Failed to find \"File not found\" 
          page. How ironic\n";
121       Response::new()
122         .with_status(StatusCode::NotFound)
123         .with_header(ContentLength(ERROR_MSG.len() as u64))
124         .with_body(ERROR_MSG)
125     }))
126   });
127   Box::new(response_future)
128 }
```

以下是一些辅助函数：

```rs
130  fn sanitize_path(path: &str) -> String {
131    // Normalize the separators for the next steps
132    path.replace("\\", "/")
133      // Prevent the user from going up the filesystem
134      .replace("../", "")
135      // If the path comes straigh from the router,
136      // it will begin with a slash
137      .trim_left_matches(|c| c == '/')
138      // Remove slashes at the end as we only serve files
139      .trim_right_matches(|c| c == '/')
140      .to_string()
141  }
142
143  fn path_on_disk(path_to_file: &str) -> String {
144    "files/".to_string() + path_to_file
145  }
146
147  fn get_content_type(file: &str) -> Option {
148    // Check the file extension and return the respective MIME type
149   let pos = file.rfind('.')? + 1;
150   let mime_type = match &file[pos..] {
151     "txt" => mime::TEXT_PLAIN_UTF_8,
152     "html" => mime::TEXT_HTML_UTF_8,
153     "css" => mime::TEXT_CSS,
154     // This list can be extended for all types your server 
        should support
155     _ => return None,
156   };
157   Some(ContentType(mime_type))
158 }
```

# 它是如何工作的...

哇！文件太多了。当然，对于这个菜谱来说，HTML 和 CSS 的确切内容并不重要，因为我们将要专注于 Rust。我们已经把它们都放在了 `files` 文件夹中，因为我们打算通过名称使任何客户端都可以公开访问其内容。

服务器设置的基本原理与回声食谱相同：使用 `const_service` 和 `service_fn` [21] 创建一个 `Service`，匹配请求的方法和路径，然后在不同的函数中处理路由。然而，当我们查看返回类型时，我们可以注意到一个差异 [36]：

```rs
type ResponseFuture = Box<Future<Item = Response, Error = hyper::Error>>;
```

我们不再直接返回一个 `Response`，而是将其封装在一个 `Future` 中。这允许我们在将文件加载到内存时不会阻塞服务器；我们可以在主线程中继续处理请求，同时文件服务的 `Future` 在后台运行。

当查看我们的路由处理程序时，你可以看到它们都使用了 `send_file_or_404` 函数。让我们看看它 [56]：

```rs
fn send_file_or_404(path: &str) -> ResponseFuture {
    let path = sanitize_path(path);

    let response_future = try_to_send_file(&path)
        .and_then(|response_result| response_result.map_err(|error| 
         error.into()))
        .or_else(|_| send_404());
    Box::new(response_future)
}
```

首先，该函数清理我们的输入。`sanitize_path` [130 到 141] 的实现应该是相当直接的。它过滤掉潜在的麻烦制造者，这样恶意客户端就不能进行任何恶作剧，例如请求文件 `localhost:3000/../../../../home/admin/.ssh/id_rsa`。

然后我们在清理后的路径上调用 `try_to_send_file` [72]。我们将在下一分钟查看该函数，但就现在而言，查看其签名就足够了。它告诉我们它返回一个 `Result` 的 `Future`，这个 `Result` 可以是一个 `Response` 或一个 `io::Error`，因为这是在无效文件系统访问中遇到的错误。我们不能直接返回这个 `Future`，因为我们已经告诉 `hyper` 我们将返回一个 `Response` 的 `Future`，所以我们需要转换类型。如果从 `try_to_send_file` 生成的文件检索 `Future` 成功，我们就对其项目进行操作，该项目是一个 `Result<Response, io::Error>`。

因为 `hyper::Error` 实现了 `From<io::Error>`，我们可以通过调用 `.into()` [63]（参见 第五章，*高级数据结构*；*类型转换*，了解 `From` 特质的介绍）。这将返回一个 `Result<Response, hyper::Error>`。因为 `Future` 可以从 `Result` 构造，所以它将隐式地转换为 `Future<Response, hyper::Error>`，这正是我们想要的。额外的一点是，我们处理 `try_to_send_file` 返回错误的情况，在这种情况下，我们可以安全地假设文件不存在，因此我们通过调用 `send_404()` [65] 返回一个自定义的 `404 Not Found` 页面。在查看其实现之前，我们先看看 `try_to_send_file` [72]。

首先，我们使用 `path_on_disk` [74] 将请求的路径转换为本地文件系统路径，其实现如下 [144]：

```rs
"files/".to_string() + path_to_file
```

我们为此创建了一个自定义函数，这样你就可以轻松扩展文件系统逻辑。例如，对于 Unix 系统，通常会将所有静态 HTML 放在 `/var/www/`，而 Windows 网络服务器通常将它们的所有数据放在它们自己的安装文件夹中。或者你可能想读取用户提供的配置文件，并将其值存储在 `lazy_static` 中，如 第五章，*高级数据结构*；*使用懒静态变量*，并使用该路径。你可以在该函数中实现所有这些规则。

在 `try_to_send_file` 函数中，我们创建了一个 `oneshot::channel` 来以 `Future` 的形式发送数据 [77]。这个概念在 第八章，*使用 Future*；*使用 oneshot 通道* 中有详细的解释。

函数的其余部分现在创建了一个新线程，在后台将文件加载到内存中 [78]。我们首先打开文件 [79]，如果文件不存在，则通过通道返回错误。然后我们将整个文件复制到一个本地的字节数组中 [91]，并再次传播可能发生的任何错误 [107]。如果将数据复制到 RAM 的过程成功，我们返回一个包含文件内容的 `Response` [100]。在这个过程中，我们必须确定文件的适当 MIME 类型 [96]，正如在 *设置基本 HTTP 服务器* 的配方中所承诺的。为此，我们只需匹配文件的扩展名 [147 到 158]。

```rs
fn get_content_type(file: &str) -> Option<ContentType> {
    let pos = file.rfind('.')? + 1;
    let mime_type = match &file[pos..] {
        "txt" => mime::TEXT_PLAIN_UTF_8,
        "html" => mime::TEXT_HTML_UTF_8,
        "css" => mime::TEXT_CSS,
        _ => return None,
    };
    Some(ContentType(mime_type))
}
```

你可能会认为这种实现相当懒惰，应该有更好的方法，但请相信我，这正是所有大型网络服务器所做的方式。以 `nginx` 为例，([`nginx.org/en/`](https://nginx.org/en/))，你可以在这里找到 mime 检测算法：[`github.com/nginx/nginx/blob/master/conf/mime.types`](https://github.com/nginx/nginx/blob/master/conf/mime.types)。如果你计划提供新的文件类型，你可以扩展它们的扩展名的 `match`。`nginx` 源代码是这方面的良好资源。

`get_content_type` 如果没有匹配项 [155] 则返回 `None` 而不是默认内容类型，这样每个调用者都可以为自己决定一个默认值。在 `try_to_send_file` 中，我们使用 `.unwrap_or_else(ContentType::plaintext);` [96] 将回退 MIME 类型设置为 `text/plain`。

在我们的示例中，最后剩下的未解释的函数是 `send_404`，我们经常将其用作回退。你可以看到它实际上只是对 404 页面 [117] 调用 `try_to_send_file`，并在出错时发送一个静态消息 [124]。

在 `send_404` 中的回退实际上展示了 Rust 错误处理概念的美丽。因为强类型错误是函数签名的一部分，与像 C++ 这样的语言不同，在 C++ 中你永远不知道谁可能会抛出异常，你必须有意识地处理错误情况。尝试移除 `and_then` 及其关联的闭包，你就会看到编译器不会让你编译程序，因为你没有以任何方式处理 `try_to_send_file` 的 `Result`。

现在就打开您的浏览器，通过指向`http://localhost:3000/`来亲眼看看我们的文件服务器的结果。

# 还有更多...

尽管相对容易理解，但我们的`try_to_send_file`实现并不是无限可扩展的。想象一下同时为成百万的客户端提供和加载大量文件到内存中。这会很快让你的 RAM 达到极限。一个更可扩展的解决方案是分块发送文件，即部分一部分，这样你只需要在任何给定时间保持文件的一小部分在内存中。要实现这一点，你需要将文件内容复制到一个固定大小的有限`[u8]`缓冲区中，并通过一个额外的通道作为`hyper::Chunk`的实例发送，它实现了`From<Vec<T>>`。

# 参见

+   *将类型相互转换*和*创建* *懒静态变量*的配方在第五章，*高级数据结构*中。

+   *使用第八章中的 oneshot 通道*配方第八章，*与 Futures 一起工作*。

# 向 API 发送请求

本章的最后一个目的地将我们带离服务器，转向参与互联网通信的另一方：客户端。我们将使用`reqwest`，它是围绕`hyper`构建的，来创建对 Web 服务的 HTTPS 请求并将它们的数据解析成可用的 Rust 结构。您也可以使用这个配方的内容来编写您自己的 Web 服务的集成测试。

# 如何做到...

1.  打开为您生成的`Cargo.toml`文件。

1.  在`[dependencies]`下，如果您在上一个配方中没有这样做，请添加以下行：

```rs
reqwest = "0.8.5"
serde = "1.0.30"
serde_derive = "1.0.30"
```

1.  如果您愿意，您可以去`request`的([`crates.io/crates/reqwest`](https://crates.io/crates/reqwest))，`serde`的([`crates.io/crates/serde`](https://crates.io/crates/serde))，和`serde_derive`的([`crates.io/crates/serde_derive`](https://crates.io/crates/serde_derive)) *crates.io*页面检查最新版本，并使用这些版本代替。

1.  在`src/bin`文件夹中，创建一个名为`making_requests.rs`的文件。

1.  添加以下代码，并使用`cargo run --bin making_requests`运行它：

```rs
1   extern crate reqwest;
2   #[macro_use]
3   extern crate serde_derive;
4   
5   use std::fmt;
6   
7   #[derive(Serialize, Deserialize, Debug)]
8   // The JSON returned by the web service that hands posts out
9   // it written in camelCase, so we need to tell serde about that
10  #[serde(rename_all = "camelCase")]
11  struct Post {
12    user_id: u32,
13    id: u32,
14    title: String,
15    body: String,
16  }
17  
18  #[derive(Serialize, Deserialize, Debug)]
19  #[serde(rename_all = "camelCase")]
20  struct NewPost {
21    user_id: u32,
22    title: String,
23    body: String,
24  }
25  
26  #[derive(Serialize, Deserialize, Debug)]
27  #[serde(rename_all = "camelCase")]
28  // The following struct could be rewritten with a builder
29  struct UpdatedPost {
30    #[serde(skip_serializing_if = "Option::is_none")]
31    user_id: Option,
32    #[serde(skip_serializing_if = "Option::is_none")]
33    title: Option,
34    #[serde(skip_serializing_if = "Option::is_none")]
35    body: Option,
36  }
37  
38  struct PostCrud {
39    client: reqwest::Client,
40    endpoint: String,
41  }
42  
43  impl fmt::Display for Post {
44    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
45      write!(
46        f,
47        "User ID: {}\nID: {}\nTitle: {}\nBody: {}\n",
48        self.user_id, self.id, self.title, self.body
49      )
50    }
51  }
```

以下代码展示了正在实现的请求：

```rs
53  impl PostCrud {
54    fn new() -> Self {
55      PostCrud {
56        // Build an HTTP client. It's reusable!
57        client: reqwest::Client::new(),
58        // This is a link to a fake REST API service
59        endpoint: 
          "https://jsonplaceholder.typicode.com/posts".to_string(),
60      }
61    }
62  
63    fn create(&self, post: &NewPost) -> Result<Post, 
      reqwest::Error> {
64      let response = 
        self.client.post(&self.endpoint).json(post).send()?.json()?;
65      Ok(response)
66    }
67  
68    fn read(&self, id: u32) -> Result<Post, reqwest::Error> {
69      let url = format!("{}/{}", self.endpoint, id);
70      let response = self.client.get(&url).send()?.json()?;
71      Ok(response)
72    }
73  
74    fn update(&self, id: u32, post: &UpdatedPost) -> Result<Post, 
      reqwest::Error> {
75      let url = format!("{}/{}", self.endpoint, id);
76      let response = 
        self.client.patch(&url).json(post).send()?.json()?;
77      Ok(response)
78    }
79  
80    fn delete(&self, id: u32) -> Result<(), reqwest::Error> {
81      let url = format!("{}/{}", self.endpoint, id);
82      self.client.delete(&url).send()?;
83      Ok(())
84    }
85  }
```

以下代码展示了我们如何使用我们的 CRUD 客户端：

```rs
87  fn main() {
88    let post_crud = PostCrud::new();
89    let post = post_crud.read(1).expect("Failed to read post");
90    println!("Read a post:\n{}", post);
91  
92    let new_post = NewPost {
93      user_id: 2,
94      title: "Hello World!".to_string(),
95      body: "This is a new post, sent to a fake JSON API 
        server.\n".to_string(),
96    };
97    let post = post_crud.create(&new_post).expect("Failed to  
      create post");
98    println!("Created a post:\n{}", post);
99  
100   let updated_post = UpdatedPost {
101     user_id: None,
102     title: Some("New title".to_string()),
103     body: None,
104   };
105   let post = post_crud
106     .update(4, &updated_post)
107     .expect("Failed to update post");
108   println!("Updated a post:\n{}", post);
109 
110   post_crud.delete(51).expect("Failed to delete post");
111 }
```

# 它是如何工作的...

在顶部，我们定义了我们的结构。`Post`[11]，`NewPost` [20]，和`UpdatedPost` [29]都只是处理 API 不同需求的便捷方式。我们正在与之交互的特定 JSON API 使用 camelCase 变量，因此我们需要在每一个`struct`上指定这一点，否则`serde`将无法正确解析它们[10，19 和 27]。

因为与我们通信的`PATCH`方法不接受未更改变量的 null 值，所以我们把所有在`UpdatedPost`中等于`None`的标记为未序列化[30，32 和 34]：

```rs
#[serde(skip_serializing_if = "Option::is_none")]
```

此外，我们在`Post`上实现了`fmt::Display`特性，这样我们就可以很好地打印它了[43 到 51]。

但关于我们的模型就先说到这里；让我们来看看`PostCrud` [53]。它的目的是抽象化一个 CRUD（创建、读取、更新、删除）服务。为此，它通过`reqwest::Client` [57]提供了一个可重用的 HTTP 客户端，以及来自[`jsonplaceholder.typicode.com/`](https://jsonplaceholder.typicode.com/)的模拟 JSON API 端点。

它的方法展示了`reqwest`的使用是多么简单：你只需直接将所需的 HTTP 方法作为客户端上的一个函数使用，向它传递可选的数据，它将自动使用`.json()`为你反序列化，然后使用`.send()`发送请求，并通过第二次调用`.json()`再次将响应解析为 JSON [64]。

# 还有更多...

当然，`reqwest`也能够与非基于 JSON 的网络服务一起工作。为此它有多种方法，例如`query`，它将键值查询数组添加到 URL 中，或者`form`，它将`url-encoded`表单体添加到请求中。在使用所有这些方法时，`reqwest`会为你管理头部信息，但你也可以使用`headers`方法以任何你想要的方式显式地管理它们。

# 参见

+   第一章中的“使用构建器模式”配方，*学习基础知识*

+   第四章中的“使用 Serde 的序列化基础知识”配方，*序列化*
