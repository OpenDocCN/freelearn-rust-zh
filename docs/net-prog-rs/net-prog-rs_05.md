# 第五章：应用层协议

正如我们在前几章中看到的，网络中的两个主机在流或离散的包中交换字节。通常，这些字节的处理工作由高级应用程序来完成，使其对应用程序有意义。这些应用程序在传输层之上定义了一个新的协议层，通常称为应用层协议。在本章中，我们将探讨这些协议中的一些。

设计应用层协议时有许多重要的考虑因素。实现需要至少了解以下详细信息：

+   通信是广播还是点对点？在前一种情况下，底层传输协议必须是 UDP。在后一种情况下，可以是 TCP 或 UDP。

+   协议需要可靠的传输吗？如果是，TCP 是唯一的选择。否则，UDP 也可能适用。

+   应用程序需要字节流（TCP）吗，还是可以按包逐个处理（UDP）？

+   双方之间如何表示输入的结束？

+   使用的数据格式和编码是什么？

一些非常常用的应用层协议包括 DNS（我们在前面的章节中学习过）和 HTTP（我们将在下一章学习）。除此之外，一个非常重要的应用层工具集，常用于基于微服务的架构，是 gRPC。另一个每个人至少使用过几次的应用层协议是 SMTP，这是电子邮件的协议。

在本章中，我们将研究以下主题：

+   RPC 是如何工作的。具体来说，我们将探讨 gRPC，并使用工具包编写一个小的服务器和客户端。

+   我们将查看一个可以用于发送电子邮件的 crate 调用`lettre`。

+   最后一个主题将是用 Rust 编写一个简单的 FTP 客户端和 TFTP 服务器。

# RPC 简介

在常规编程中，将常用逻辑封装在函数中通常很有用，这样就可以在多个地方重用。随着网络化和分布式系统的兴起，让一组公共操作可以通过网络访问变得必要，这样经过验证的客户端就可以调用它们。这通常被称为**远程过程调用**（RPC）。在第四章*，*数据序列化、反序列化和解析*中，我们看到了一个简单的例子，当服务器返回给定点与原点的距离时。现实世界的 RPC 定义了许多应用层协议，这些协议要复杂得多。最受欢迎的 RPC 实现之一是 gRPC，它最初由谷歌引入，后来转为开源模式。gRPC 在互联网规模网络上提供高性能 RPC，并被广泛应用于许多项目中，包括 Kubernetes。

在深入研究 gRPC 之前，让我们看看相关的工具——协议缓冲区。它是一套机制，用于在应用程序之间构建语言和平台中立的交换结构化数据。它定义了自己的**接口定义语言**（**IDL**）来描述数据格式，以及一个可以将该格式转换为代码并生成代码以进行转换的编译器。IDL 还允许定义抽象服务：编译器可以用来生成给定语言的存根的输入和输出消息格式。我们将在后续示例中看到一个数据格式定义的例子。编译器有插件可以生成大量语言的输出代码，包括 Rust。在我们的示例中，我们将在构建脚本中使用这样的插件来自动生成 Rust 模块。现在，gRPC 使用协议缓冲区来定义底层数据和消息。消息在 TCP/IP 之上通过 HTTP/2 进行交换。在实践中，这种通信模式通常更快，因为它可以更好地利用现有连接，并且 HTTP/2 支持双向异步连接。gRPC 作为一个有偏见的系统，代表我们做出了许多关于我们在上一节讨论的考虑的假设。大多数这些默认值（如 HTTP/2 通过 TCP）都是因为它们支持 gRPC 提供的先进功能（如双向流）。一些其他默认值，如使用`protobuf`，可以与其他消息格式实现交换。

对于我们的 gRPC 示例，我们将构建一个类似于 Uber 的服务。它有一个中央服务器，客户端（出租车）可以记录他们的名字和位置。然后，当用户请求出租车并给出他们的位置时，服务器会发送一个靠近该用户的出租车列表。理想情况下，这个服务器应该有两个客户端类别，一个用于出租车，一个用于用户。但为了简单起见，我们将假设我们只有一种类型的客户端。

让我们从设置项目开始。像往常一样，我们将使用 Cargo CLI 初始化项目：

```rs
$ cargo new --bin grpc_example
protoc-rust-grpc crate. Hence, we will need to add that as a build dependency:
```

```rs
$ cat Cargo.toml
[package]
name = "grpc_example"
version = "0.1.0"
authors = ["Foo<foo@bar.com>"]

[dependencies]
protobuf = "1.4.1"
grpc = "0.2.1"
tls-api = "0.1.8"

[build-dependencies]
protoc-rust-grpc = "0.2.1"
```

以下脚本是我们的构建脚本。它只是一个 Rust 可执行文件（具有主函数），Cargo 在调用给定项目的编译器之前构建并运行它。请注意，此脚本的默认名称为`build.rs`，并且它必须位于项目根目录中。然而，这些参数可以在 Cargo 配置文件中进行配置：

```rs
// ch5/grpc/build.rs

extern crate protoc_rust_grpc;

fn main() {
    protoc_rust_grpc::run(protoc_rust_grpc::Args {
        out_dir: "src",
        includes: &[],
        input: &["foobar.proto"],
```

```rs
        rust_protobuf: true,
    }).expect("Failed to generate Rust src");
}
```

构建脚本最常见的使用案例之一是代码生成（如我们当前的项目）。它们还可以用于在主机上查找和配置本地库等。

在脚本中，我们使用`protoc_rust_grpc`包来从我们的`proto`文件（称为`foobar.proto`）生成 Rust 模块。我们还设置了`rust_protobuf`标志以使其生成 protobuf 消息。请注意，`protoc`二进制文件必须在`$PATH`中可用才能正常工作。这是 protobuf 包的一部分。按照以下步骤从源代码安装它：

1.  从 GitHub 下载预构建的二进制文件：

```rs
$ curl -Lo https://github.com/google/protobuf/releases/download/v3.5.1/protoc-3.5.1-linux-x86_64.zip
```

1.  解压缩存档：

```rs
$ unzip protoc-3.5.1-linux-x86_64.zip -d protoc3
```

1.  将二进制文件复制到 `$PATH` 中的某个位置：

```rs
$ sudo mv protoc3/bin/* /usr/local/bin/
```

当前示例已在 Ubuntu 16.04 上使用 protoc 版本 3.5.1 进行了测试。

接下来，我们需要协议定义，如下代码片段所示：

```rs
// ch5/grpc/foobar.proto

syntax = "proto3";

package foobar;

// Top level gRPC service with two RPC calls
service FooBarService {
    rpc record_cab_location(CabLocationRequest) returns
    (CabLocationResponse);
    rpc get_cabs(GetCabRequest) returns (GetCabResponse);
}

// A request to record location of a cab
// Name: unique name for a cab
// Location: current location of the given cab
message CabLocationRequest {
    string name = 1;
    Location location = 2;
}

// A response for a CabLocationRequest
// Accepted: a boolean indicating if this
// request was accepted for processing
message CabLocationResponse {
    bool accepted = 1;
}

// A request to return cabs at a given location
// Location: a given location
message GetCabRequest {
    Location location = 1;
}

// A response for GetCabLocation
// Cabs: list of cabs around the given location
message GetCabResponse {
    repeated Cab cabs = 1;
}

// Message that the CabLocationRequest passes
// to the server
message Cab {
    string name = 1;
    Location location = 2;
}

// Message with the location of a cab
message Location {
    float latitude = 1;
    float longitude = 2;
}
```

proto 文件以 protobuf IDP 规范版本的声明开始；我们将使用版本 3。包声明表明所有生成的代码都将放置在一个名为 `foobar` 的 Rust 模块中，所有其他生成的代码都将放置在一个名为 `foobar_grpc` 的模块中。我们定义了一个名为 `FooBarService` 的服务，它有两个 RPC 函数；`record_cab_location` 记录出租车的位置，给定其名称和位置，而 `get_cabs` 返回一组出租车，给定一个位置。我们还需要为每个请求和响应定义所有相关的 `protobuf` 消息。规范还定义了多个与编程语言中的数据类型（字符串、浮点数等）紧密对应的自定义数据类型。

在设置好与 `protobuf` 消息格式和函数相关的一切之后，我们可以使用 Cargo 生成实际的 Rust 代码。生成的代码将位于 `src` 目录中，并命名为 `foobar.rs` 和 `foobar_grpc.rs`。这些名称由编译器自动分配。`lib.rs` 文件应使用 pub mod 语法重新导出这些模块。请注意，Cargo 构建不会为我们修改 `lib.rs` 文件；这需要手动完成。让我们继续我们的服务器和客户端。以下是服务器的外观：

```rs
// ch5/grpc/src/bin/server.rs

extern crate grpc_example;
extern crate grpc;
extern crate protobuf;

use std::thread;

use grpc_example::foobar_grpc::*;
use grpc_example::foobar::*;

struct FooBarServer;

// Implementation of RPC functions
    impl FooBarService for FooBarServer {
    fn record_cab_location(&self,
                       _m: grpc::RequestOptions,
                       req: CabLocationRequest)
                       ->
        grpc::SingleResponse<CabLocationResponse> {
        let mut r = CabLocationResponse::new();

        println!("Recorded cab {} at {}, {}", req.get_name(),
        req.get_location().latitude, req.get_location().longitude);

        r.set_accepted(true);
        grpc::SingleResponse::completed(r)
    }

    fn get_cabs(&self,
                      _m: grpc::RequestOptions,
                      _req: GetCabRequest)
                      -> grpc::SingleResponse<GetCabResponse> {
        let mut r = GetCabResponse::new();

        let mut location = Location::new();
        location.latitude = 40.7128;
        location.longitude = -74.0060;

        let mut one = Cab::new();
        one.set_name("Limo".to_owned());
        one.set_location(location.clone());

        let mut two = Cab::new();
        two.set_name("Merc".to_owned());
        two.set_location(location.clone());

        let vec = vec![one, two];
        let cabs = ::protobuf::RepeatedField::from_vec(vec);

        r.set_cabs(cabs);

        grpc::SingleResponse::completed(r)
    }
}

fn main() {
    let mut server = grpc::ServerBuilder::new_plain();
    server.http.set_port(9001);
    server.add_service(FooBarServiceServer::new_service_def(FooBarServer));
    server.http.set_cpu_pool_threads(4);
    let _server = server.build().expect("Could not start server");
    loop {
        thread::park();
    }
}
```

注意，这个服务器与我们之前章节中编写的服务器非常不同。这是因为 `grpc::ServerBuilder` 封装了编写服务器时的大部分复杂性。`FooBarService` 是为我们生成的 `protobuf` 编译器服务，在 `foobar_grpc.rs` 文件中定义为 trait。正如预期的那样，这个 trait 有两个方法：`record_cab_location` 和 `get_cabs`。因此，对于我们的服务器，我们需要在一个结构体上实现这个 trait 并将其传递给 `ServerBuilder` 以在指定的端口上运行。

在我们的玩具示例中，我们实际上不会记录出租车位置。一个真实世界的应用会希望将这些信息存储在数据库中以供以后查询。相反，我们只需打印一条消息，说明我们收到了一个新的位置。我们还需要处理一些样板代码，以确保所有 gRPC 语义都得到满足。在 `get_cabs` 函数中，我们总是为所有请求返回一个静态的出租车列表。请注意，由于所有 `protobuf` 消息都是为我们生成的，我们免费获得了一些实用函数，如 `get_name` 和 `get_location`。最后，在 `main` 函数中，我们将我们的服务器结构传递给 gRPC，在指定的端口上创建一个新的服务器并无限循环运行它。

我们的客户端实际上是在由 `protobuf` 编译器生成的源代码中定义为一个结构体。我们只需要确保客户端具有与我们运行服务器相同的端口号。我们使用客户端结构体的 `new_plain` 方法，并传递一个地址和端口号给它，以及一些默认选项。然后我们可以通过 RPC 调用 `record_cab_location` 和 `get_cabs` 方法并处理响应：

```rs
// ch5/grpc/src/bin/client.rs

extern crate grpc_example;
extern crate grpc;

use grpc_example::foobar::*;
use grpc_example::foobar_grpc::*;

fn main() {
    // Create a client to talk to a given server
    let client = FooBarServiceClient::new_plain("127.0.0.1", 9001,
    Default::default()).unwrap();

    let mut req = CabLocationRequest::new();
    req.set_name("foo".to_string());

    let mut location = Location::new();
    location.latitude = 40.730610;
    location.longitude = -73.935242;
    req.set_location(location);

    // First RPC call
    let resp = client.record_cab_location(grpc::RequestOptions::new(),
    req);
    match resp.wait() {
        Err(e) => panic!("{:?}", e),
        Ok((_, r, _)) => println!("{:?}", r),
    }

    let mut nearby_req = GetCabRequest::new();
    let mut location = Location::new();
    location.latitude = 40.730610;
    location.longitude = -73.935242;
    nearby_req.set_location(location);

    // Another RPC call
    let nearby_resp = client.get_cabs(grpc::RequestOptions::new(),
    nearby_req);
    match nearby_resp.wait() {
        Err(e) => panic!("{:?}", e),
        Ok((_, cabs, _)) => println!("{:?}", cabs),
    }
}
```

客户端运行的过程如下。如前所述，这并不像应有的那样动态，因为它只返回硬编码的值：

```rs
$ cargo run --bin client
   Blocking waiting for file lock on build directory
   Compiling grpc_example v0.1.0 (file:///rust-book/src/ch5/grpc)
   Finished dev [unoptimized + debuginfo] target(s) in 3.94 secs
     Running `/rust-book/src/ch5/grpc/target/debug/client`
accepted: true
cabs {name: "Limo" location {latitude: 40.7128 longitude: -74.006}} cabs {name: "Merc" location {latitude: 40.7128 longitude: -74.006}}
```

注意它在与服务器通信后立即退出。另一方面，服务器在一个无限循环中运行，直到接收到信号才退出：

```rs
$ cargo run --bin server
   Compiling grpc_example v0.1.0 (file:///rust-book/src/ch5/grpc)
    Finished dev [unoptimized + debuginfo] target(s) in 5.93 secs
     Running `/rust-book/src/ch5/grpc/target/debug/server`
Recorded cab foo at 40.73061, -73.93524
```

# SMTP 简介

互联网电子邮件使用一种称为 **简单邮件传输协议（SMTP**）的协议，这是一个 IETF 标准。与 HTTP 类似，它是一个简单的基于 TCP 的文本协议，默认使用端口 `25`。在本节中，我们将查看使用 `lettre` 发送电子邮件的小示例。为了使这成为可能，让我们首先设置我们的项目：

```rs
$ cargo new --bin lettre-example
```

现在，我们的 `Cargo.toml` 文件应该看起来像这样：

```rs
$ cat Cargo.toml
[package]
name = "lettre-example"
version = "0.1.0"
authors = ["Foo<foo@bar.com>"]

[dependencies]
lettre = "0.7"
uuid = "0.5.1"
native-tls = "0.1.4"
```

假设我们想要自动发送服务器的崩溃报告。为了实现这一点，我们需要有一个可访问的 SMTP 服务器在运行。我们还需要有一个用户，可以使用在该服务器上设置的密码进行身份验证。设置好这些后，我们的代码将看起来像这样：

```rs
// ch5/lettre-example/src/main.rs

extern crate uuid;
extern crate lettre;
extern crate native_tls;

use std::env;
use lettre::{SendableEmail, EmailAddress, EmailTransport};
use lettre::smtp::{SmtpTransportBuilder, SUBMISSION_PORT};
use lettre::smtp::authentication::Credentials;
use lettre::smtp::client::net::ClientTlsParameters;

use native_tls::TlsConnector;

// This struct represents our email with all the data
// we want to send.
struct CrashReport {
    to: Vec<EmailAddress>,
    from: EmailAddress,
    message_id: String,
    message: Vec<u8>,
}

// A simple constructor for our email.
impl CrashReport {
    pub fn new(from_address: EmailAddress,
        to_addresses: Vec<EmailAddress>,
        message_id: String,
        message: String) -> CrashReport {
            CrashReport { from: from_address,
            to: to_addresses,
            message_id: message_id,
            message: message.into_bytes()
            }
        }
}

impl<'a> SendableEmail<'a, &'a [u8]> for CrashReport {
    fn to(&self) -> Vec<EmailAddress> {
        self.to.clone()
    }

    fn from(&self) -> EmailAddress {
        self.from.clone()
    }

    fn message_id(&self) -> String {
        self.message_id.clone()
    }

    fn message(&'a self) -> Box<&[u8]> {
        Box::new(self.message.as_slice())
    }
}

fn main() {
    let server = "smtp.foo.bar";
    let connector = TlsConnector::builder().unwrap().build().unwrap();
    let mut transport = SmtpTransportBuilder::new((server, SUBMISSION_PORT), lettre::ClientSecurity::Opportunistic(<ClientTlsParameters>::new(server.to_string(), connector)))
.expect("Failed to create transport")
    .credentials(Credentials::new(env::var("USERNAME").unwrap_or_else(|_| "user".to_string()),
env::var("PASSWORD").unwrap_or_else
(|_| "password".to_string())))
    .build();
    let report = CrashReport::new(EmailAddress::
    new("foo@bar.com".to_string()), vec!   
    [EmailAddress::new("foo@bar.com".to_string())]
    , "foo".to_string(), "OOPS!".to_string());
    transport.send(&report).expect("Failed to send the report");
}
```

我们的电子邮件由 `CrashReport` 结构体表示；正如预期的那样，它有一个 `from` 发件人电子邮件地址。`to` 字段是一个电子邮件地址的向量，使我们能够向多个地址发送电子邮件。我们为该结构体实现了一个构造函数。`lettre` 包含一个名为 `SendableEmail` 的特质，它具有 SMTP 服务器发送电子邮件所需的一组属性。为了使用户定义的电子邮件可发送，它需要实现该特质。在我们的例子中，`CrashReport` 需要实现它。我们继续实现特质中所有必需的方法。此时，一个新的 `CrashReport` 实例应该可以作为电子邮件发送。

在我们的主函数中，我们需要对 SMTP 服务器进行 `auth` 以发送我们的电子邮件。我们创建一个包含与 SMTP 服务器通信所需所有信息的传输对象。用户名和密码可以作为环境变量（或默认值）传递。然后我们创建一个 `CrashReport` 实例，并使用传输对象的 `send` 方法发送它。运行此操作不会输出任何信息（如果它成功运行）。

可能有人已经注意到 `lettre` 暴露的 API 并不容易使用。这主要是因为在撰写本文时，该库在很大程度上还不成熟，处于 0.7 版本。因此，我们应该期待在达到 1.0 版本之前，API 会发生重大变化。

# FTP 和 TFTP 简介

另一个常见的应用层协议是 **文件传输协议**（**FTP**）。这是一个基于文本的协议，其中服务器和客户端通过交换文本命令来上传和下载文件。Rust 生态系统有一个名为 rust-ftp 的 crate，可以用来与 FTP 服务器进行程序性交互。让我们看看它的一个使用示例。我们使用 Cargo 设置我们的项目：

```rs
$ cargo new --bin ftp-example
```

我们的 `Cargo.toml` 应该看起来像这样：

```rs
[package]
name = "ftp-example"
version = "0.1.0"
authors = ["Foo<foo@bar.com>"]

[dependencies.ftp]
version = "2.2.1"
```

为了使这个例子工作，我们需要一个运行中的 FTP 服务器。一旦我们设置好并确保常规 FTP 客户端可以连接到它，我们就可以继续我们的主要代码：

```rs
// ch5/ftp-example/src/main.rs

extern crate ftp;

use std::str;
use std::io::Cursor;
use ftp::{FtpStream, FtpError};

fn run_ftp(addr: &str, user: &str, pass: &str) -> Result<(), FtpError> {
    let mut ftp_stream = FtpStream::connect((addr, 21))?;
    ftp_stream.login(user, pass)?;
    println!("current dir: {}", ftp_stream.pwd()?);

    let data = "A random string to write to a file";
    let mut reader = Cursor::new(data);
    ftp_stream.put("my_file.txt", &mut reader)?;

    ftp_stream.quit()
}

fn main() {
    run_ftp("ftp.dlptest.com", "dlpuser@dlptest.com", "eiTqR7EMZD5zy7M").unwrap();
}
```

在我们的例子中，我们将连接到一个位于 `ftp.dlptest.com` 的免费公共 FTP 服务器。使用该服务器的凭据位于此网站：[`dlptest.com/ftp-test/`](https://dlptest.com/ftp-test/)。我们称为 `run_ftp` 的辅助函数接收一个 FTP 服务器的地址，其中用户名和密码作为字符串。然后它连接到端口号 `21`（FTP 的默认端口）。接着使用提供的凭据登录，然后打印当前目录（应该是 `/`）。然后我们使用 `put` 函数在那里写入文件，最后关闭与服务器的连接。在我们的 `main` 函数中，我们只需调用辅助函数并传入所需的参数。

这里需要注意的一个问题是 `cursor` 的使用；它代表一个内存缓冲区，并提供了在该缓冲区上实现 `Read`、`Write` 和 `Seek` 的方法。`put` 函数期望一个实现了 `Read` 的输入；将我们的数据包装在 `Cursor` 中会自动为我们完成这一点。

运行此示例时，我们会看到以下内容：

```rs
$ cargo run
   Compiling ftp-example v0.1.0 (file:///Users/Abhishek/Desktop/rust-book/src/ch5/ftp-example)
    Finished dev [unoptimized + debuginfo] target(s) in 1.5 secs
     Running `target/debug/ftp-example`
current dir: /
```

在这个例子中，我们使用了一个公开的 FTP 服务器。由于这个服务器不在我们的控制之下，它可能会在没有通知的情况下离线。如果发生这种情况，示例需要修改以使用另一个服务器。

与 FTP 密切相关的一个协议称为 **简单文件传输协议**（**TFTP**）。TFTP 是基于文本的，就像 FTP 一样，但与 FTP 不同，它更容易实现和维护。它使用 UDP 进行传输，并且不提供任何认证原语。由于它更快、更轻量，它经常在嵌入式系统和引导协议（如 PXE 和 BOOTP）中实现。让我们看看使用名为 `tftp_server` 的 crate 的简单 TFTP 服务器。在这个例子中，我们将从 Cargo 开始，如下所示：

```rs
$ cargo new --bin tftp-example
```

我们的清单非常简单，看起来像这样：

```rs
[package]
name = "tftp-example"
version = "0.1.0"
authors = ["Foo <foo@bar.com>"]

[dependencies]
tftp_server = "0.0.2"
```

我们的主要文件将看起来像这样：

```rs
// ch5/tftp-example/src/main.rs

extern crate tftp_server;

use tftp_server::server::TftpServer;

use std::net::SocketAddr;
use std::str::FromStr;

fn main() {
    let addr = format!("0.0.0.0:{}", 69);
    let socket_addr = SocketAddr::from_str(addr.as_str()).expect("Error
    parsing address");
    let mut server =
    TftpServer::new_from_addr(&socket_addr).expect("Error creating
    server");
    match server.run() {
        Ok(_) => println!("Server completed successfully!"),
```

```rs
        Err(e) => println!("Error: {:?}", e),
    }
}
```

这可以在任何安装了 TFTP 客户端的 Unix 机器上轻松测试。如果我们在一个终端上运行此示例，并在另一个终端上运行客户端，我们需要将客户端连接到本地的端口号 `69`。然后我们应该能够从服务器下载文件。

运行此可能需要 root 权限。如果是这种情况，请使用 `sudo`。

**$ sudo ./target/debug/tftp-example**

一个示例会话如下：

```rs
$ tftp
tftp> connect 127.0.0.1 69
tftp> status
Connected to 127.0.0.1.
Mode: netascii Verbose: off Tracing: off
Rexmt-interval: 5 seconds, Max-timeout: 25 seconds
tftp> get Cargo.toml
Received 144 bytes in 0.0 seconds
tftp> ^D
```

# 摘要

在本章中，我们基于之前所学的内容进行了扩展。本质上，我们将网络栈提升到了应用层。我们研究了构建应用层协议的一些主要考虑因素。然后我们探讨了 RPC，特别是 gRPC，研究了它是如何帮助开发者构建大规模网络服务的。接着我们查看了一个可以用于通过 SMTP 服务器发送电子邮件的 Rust crate。在本书的其他部分也涵盖了其他应用层协议的情况下，我们应该对这些协议有很好的理解。

HTTP 是一种基于文本的应用层协议，它值得单独一章来介绍。在下一章中，我们将更深入地研究它，并编写一些代码使其工作。
