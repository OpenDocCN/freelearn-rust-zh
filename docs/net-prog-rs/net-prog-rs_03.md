# 第三章：使用 Rust 进行 TCP 和 UDP

作为一种系统编程语言，Rust 标准库支持与网络栈交互。所有与网络相关的功能都位于`std::net`命名空间中；读取和写入套接字也使用了来自`std::io`的`Read`和`Write`特质。这里一些最重要的结构包括`IpAddr`，它表示一个通用的 IP 地址，可以是 v4 或 v6，`SocketAddr`表示一个通用的套接字地址（主机上的 IP 和端口的组合），`TcpListener`和`TcpStream`用于 TCP 通信，`UdpSocket`用于 UDP，等等。目前，标准库不提供任何用于处理较低级别网络栈的 API。虽然这可能在将来改变，但许多 crate 填补了这一空白。其中最重要的是`libpnet`，它提供了一套用于低级别网络编程的 API。

一些其他重要的网络 crate 是`net2`和`socket2`。这些原本是为了孵化可能被移至标准库的 API 而设计的。当认为某些功能有用且足够稳定时，这些功能会被移植到 Rust 核心仓库。不幸的是，在所有情况下这并不按计划进行。总的来说，社区现在建议使用 tokio 生态系统中的 crate 来编写不需要精细控制套接字语义的高性能网络应用程序。请注意，tokio 不在本章的范围内，我们将在下一章中介绍它。

在本章中，我们将涵盖以下主题：

+   Rust 中一个简单的多线程 TCP 客户端和服务器看起来是什么样子

+   编写一个简单的多线程 UDP 客户端和服务器

+   `std::net`中的许多功能

+   学习如何使用`net2`、`ipnetwork`和`libpnet`

为了简化，本章中的所有代码都将仅处理 IPv4。将给定的示例扩展到 IPv6 应该是微不足道的。

# 一个简单的 TCP 服务器和客户端

大多数网络示例都是从 echo 服务器开始的。所以，让我们先写一个基本的 echo 服务器，看看所有部件是如何组合在一起的。我们将使用标准库中的线程模型来并行处理多个客户端。代码如下：

```rs
// chapter3/tcp-echo-server.rs

use std::net::{TcpListener, TcpStream};
use std::thread;

use std::io::{Read, Write, Error};

// Handles a single client
fn handle_client(mut stream: TcpStream) -> Result<(), Error> {
    println!("Incoming connection from: {}", stream.peer_addr()?);
    let mut buf = [0; 512];
    loop {
        let bytes_read = stream.read(&mut buf)?;
        if bytes_read == 0 { return Ok(()); }
        stream.write(&buf[..bytes_read])?;
    }
}

fn main() {
    let listener = TcpListener::bind("0.0.0.0:8888")
                               .expect("Could not bind");
    for stream in listener.incoming() {
        match stream {
            Err(e) => { eprintln!("failed: {}", e) }
            Ok(stream) => {
                thread::spawn(move || {
                    handle_client(stream)
                    .unwrap_or_else(|error| eprintln!("{:?}", error));
                });
            }
        }
    }
}
```

在我们的`main`函数中，我们创建一个新的`TcpListener`，在 Rust 中，它代表一个监听客户端传入连接的 TCP 套接字。在我们的示例中，我们硬编码了本地地址和端口；将本地地址设置为`0.0.0.0`告诉内核将此套接字绑定到该主机上所有可用的接口。在这里设置一个知名端口很重要，因为我们需要知道这个端口才能从客户端连接。在实际应用中，这应该是一个可配置的参数，从 CLI 或配置文件中获取。我们在本地 IP 和端口对上调用`bind`以创建一个本地监听套接字。如前所述，我们选择的 IP 将绑定此套接字到主机上的所有接口，端口为 8888。因此，任何可以连接到连接到该主机的网络的客户端都将能够与该主机通信。正如我们在上一章中看到的，`expect`函数在没有错误的情况下返回监听器。如果情况不是这样，它将使用给定的消息恐慌。在这里，在绑定端口失败时恐慌实际上是可行的，因为如果失败了，服务器将无法继续工作。`listener`上的`incoming`方法返回一个迭代器，遍历已连接到服务器的流。我们遍历它们并检查是否有任何遇到错误。在这种情况下，我们可以打印错误并继续下一个已连接的客户端。请注意，在这种情况下恐慌是不适当的，因为如果一些客户端由于某种原因遇到错误，服务器仍然可以正常工作。

现在，我们必须在一个无限循环中从每个客户端读取数据。但在主线程中运行无限循环将会阻塞它，其他客户端将无法连接。这种行为肯定是不希望的。因此，我们必须创建一个工作线程来处理每个客户端连接。从每个流中读取并写回的逻辑封装在名为`handle_client`的函数中。每个线程接收一个调用此函数的闭包。这个闭包必须是一个`move`闭包，因为必须从封装的作用域中读取一个变量（`stream`）。在函数中，我们打印远程端点的地址和端口，然后定义一个缓冲区来临时存储数据。我们还确保缓冲区被清零。然后我们运行一个无限循环，在其中读取流中的所有数据。流中的读取方法返回它已读取的数据长度。它可以在两种情况下返回零，如果它已到达流的末尾，或者如果给定的缓冲区长度为零。我们肯定第二种情况是不正确的。因此，当读取方法返回零时，我们跳出循环（和函数）。在这种情况下，我们返回一个`Ok()`。然后我们使用切片语法将相同的数据写回流。请注意，我们使用了`eprintln!`来输出错误。这个宏将给定的字符串写入标准错误，最近已经稳定下来。

有人在读取和写入流时可能注意到明显的错误处理缺失。但实际上并非如此。我们已经使用了`?`运算符来处理这些调用中的错误。如果一切顺利，此运算符将结果展开为`Ok`；否则，它将错误提前返回给调用函数。考虑到这种设置，函数的返回类型必须是空类型，以处理成功情况，或者`io::Error`类型，以处理错误情况。注意，在这种情况下实现自定义错误可能是一个好主意，并返回这些错误而不是内置错误。另外，请注意，目前`?`运算符不能在`main`函数中使用，因为`main`函数不返回`Result`。

Rust 最近接受了一个 RFC，该 RFC 提议在`main`函数中使用`?`运算符。

从终端与服务器交互很简单。当我们在 Linux 机器上运行服务器，并在另一个终端运行`nc`时，输入到`nc`的任何文本都应该被回显。注意，如果客户端和服务器在同一节点上运行，我们可以使用 127.0.0.1 作为服务器地址：

```rs
$ nc <server ip> 8888
test
test
foobar
foobar
foobarbaz
foobarbaz
^C
```

虽然使用`nc`与服务器交互是可以的，但从头开始编写客户端会更有趣。在本节中，我们将看到一个简单的 TCP 客户端可能的样子。这个客户端将从`stdin`读取输入作为字符串，并将其发送到服务器。当它收到回复时，它将在`stdout`中打印出来。在我们的示例中，客户端和服务器运行在同一物理主机上，因此我们可以使用 127.0.0.1 作为服务器地址：

```rs
// chapter3/tcp-client.rs

use std::net::TcpStream;
use std::str;
use std::io::{self, BufRead, BufReader, Write};

fn main() {
    let mut stream = TcpStream::connect("127.0.0.1:8888")
                               .expect("Could not connect to server");
    loop {
        let mut input = String::new();
        let mut buffer: Vec<u8> = Vec::new();
        io::stdin().read_line(&mut input)
                   .expect("Failed to read from stdin");
        stream.write(input.as_bytes())
              .expect("Failed to write to server");

        let mut reader = BufReader::new(&stream);

        reader.read_until(b'\n', &mut buffer)
              .expect("Could not read into buffer");
        print!("{}", str::from_utf8(&buffer)
               .expect("Could not write buffer as string"));
    }
}
```

在这种情况下，我们首先导入所有必需的库。然后，我们使用`TcpStream::connect`设置到服务器的连接，该函数接受远程端点地址作为字符串。像所有 TCP 连接一样，客户端需要知道远程 IP 和端口才能连接。如果设置连接失败，我们将使用错误消息终止我们的程序。然后我们启动一个无限循环，在这个循环中，我们将初始化一个空字符串以读取本地用户输入，以及一个`u8`向量以读取来自服务器的响应。由于 Rust 中的向量会根据需要增长，我们不需要在每次迭代中手动分块数据。`read_line`函数从标准输入读取一行并将其存储在名为`input`的变量中。然后，它作为字节流写入连接。此时，如果一切按预期进行，服务器应该已经发送了响应。我们将使用`BufReader`读取该响应，该`BufReader`负责内部分块数据。这也使得读取更高效，因为将不会进行比必要的更多系统调用。`read_until`方法读取缓冲区中的数据，该缓冲区根据需要增长。最后，我们可以使用`from_utf8`方法将缓冲区打印为字符串。

运行客户端很简单，并且正如预期的那样，其行为与`nc`完全相同：

```rs
$ rustc tcp-client.rs && ./tcp-client
test
test
foobar
foobar
foobarbaz
foobarbaz
^C
```

实际应用通常比这更复杂。服务器可能需要一些时间来处理输入然后再返回响应。让我们通过在 `handle_client` 函数中随机睡眠一段时间来模拟这种情况；`main` 函数将保持与上一个示例完全相同。第一步是使用 `cargo` 创建我们的项目：

```rs
$ cargo new --bin tcp-echo-random
```

注意，我们需要在 `Cargo.toml` 中添加 `rand` crate，如下代码片段所示：

```rs
[package]
name = "tcp-echo-random"
version = "0.1.0"
authors = ["Foo <foo@bar.com>"]

[dependencies]
rand = "0.3.17"
```

在设置好依赖项后，让我们修改 `handle_client` 函数，在发送响应之前先随机睡眠一段时间：

```rs
// chapter3/tcp-echo-random/src/main.rs

extern crate rand;

use std::net::{TcpListener, TcpStream};
use std::thread;
use rand::{thread_rng, Rng};
use std::time::Duration;
use std::io::{Read, Write, Error};

fn handle_client(mut stream: TcpStream) -> Result<(), Error> {
    let mut buf = [0; 512];
    loop {
        let bytes_read = stream.read(&mut buf)?;
        if bytes_read == 0 { return Ok(()) }
        let sleep = Duration::from_secs(*thread_rng()
                             .choose(&[0, 1, 2, 3, 4, 5])
                             .unwrap());
        println!("Sleeping for {:?} before replying", sleep);
        std::thread::sleep(sleep);
        stream.write(&buf[..bytes_read])?;
    }
}

fn main() {
    let listener = TcpListener::bind("127.0.0.1:8888").expect("Could
    not bind");
    for stream in listener.incoming() {
        match stream {
            Err(e) => eprintln!("failed: {}", e),
            Ok(stream) => {
                thread::spawn(move || {
                    handle_client(stream).unwrap_or_else(|error|
                    eprintln!("{:?}", error));
                });
            }
        }
    }
}
```

在我们的主文件中，我们必须声明对 `rand` crate 的依赖，并将其声明为 `extern crate`。我们使用 `thread_rng` 函数随机选择一个介于零和五之间的整数，然后使用 `std::thread::sleep` 睡眠相应的时间长度。在客户端，我们将设置读取和连接超时，因为来自服务器的回复不会立即出现：

```rs
// chapter3/tcp-client-timeout.rs

use std::net::TcpStream;
use std::str;
use std::io::{self, BufRead, BufReader, Write};
use std::time::Duration;
use std::net::SocketAddr;

fn main() {
    let remote: SocketAddr = "127.0.0.1:8888".parse().unwrap();
    let mut stream = TcpStream::connect_timeout(&remote,
    Duration::from_secs(1))
                               .expect("Could not connect to server");
    stream.set_read_timeout(Some(Duration::from_secs(3)))
          .expect("Could not set a read timeout");
    loop {
        let mut input = String::new();
        let mut buffer: Vec<u8> = Vec::new();
        io::stdin().read_line(&mut input).expect("Failed to read from
        stdin");
        stream.write(input.as_bytes()).expect("Failed to write to
        server");

        let mut reader = BufReader::new(&stream);

        reader.read_until(b'\n', &mut buffer)
              .expect("Could not read into buffer");
        print!("{}", str::from_utf8(&buffer)
                    .expect("Could not write buffer as string"));
    }
}
```

在这里，我们使用 `set_read_timeout` 将超时设置为三秒。因此，如果服务器睡眠超过三秒，客户端将终止连接。这个函数很奇怪，因为它接受 `Option<Duration>` 来能够指定一个 `None` 的 `Duration`。因此，我们需要在传递给这个函数之前将我们的 `Duration` 包装在 `Some` 中。现在，如果我们打开两个会话，一个使用 cargo 运行服务器，另一个运行客户端，我们会看到以下内容；服务器打印出它为每个接受的客户端睡眠了多长时间：

```rs
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/tcp-echo-random`
Sleeping for Duration { secs: 2, nanos: 0 } before replying
Sleeping for Duration { secs: 1, nanos: 0 } before replying
Sleeping for Duration { secs: 1, nanos: 0 } before replying
Sleeping for Duration { secs: 5, nanos: 0 } before replying
```

在客户端，我们有一个单独的文件（不是一个 cargo 项目），我们将使用 `rustc` 构建，并在编译后直接运行可执行文件：

```rs
$ rustc tcp-client-timeout.rs && ./tcp-client-timeout
test
test
foo
foo
bar
bar
baz
thread 'main' panicked at 'Could not read into buffer: Error { repr: Os { code: 35, message: "Resource temporarily unavailable" } }', src/libcore/result.rs:906:4
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

对于前三个输入，服务器选择的延迟小于三秒。客户端在三秒内收到响应，并且没有终止连接。对于最后一条消息，延迟为五秒，这导致客户端终止读取。

# 一个简单的 UDP 服务器和客户端

与我们之前编写的 UDP 服务器和 TCP 服务器之间有一些语义上的差异。与 TCP 不同，UDP 没有流结构。这源于两种协议之间的语义差异。让我们看看一个 UDP 服务器可能的样子：

```rs
// chapter3/udp-echo-server.rs

use std::thread;
use std::net::UdpSocket;

fn main() {
    let socket = UdpSocket::bind("0.0.0.0:8888")
                           .expect("Could not bind socket");

    loop {
        let mut buf = [0u8; 1500];
        let sock = socket.try_clone().expect("Failed to clone socket");
        match socket.recv_from(&mut buf) {
            Ok((_, src)) => {
                thread::spawn(move || {
                    println!("Handling connection from {}", src);
                    sock.send_to(&buf, &src)
                        .expect("Failed to send a response");
                });
            },
            Err(e) => {
                eprintln!("couldn't recieve a datagram: {}", e);
            }
        }
    }
}
```

与 TCP 一样，我们从绑定到给定端口上的本地地址开始，并处理绑定可能失败的可能性。由于 UDP 是无连接协议，我们不需要进行滑动窗口来读取所有数据。因此，我们可以分配一个给定大小的静态缓冲区。动态检测底层网络卡的 MTU 并将缓冲区大小设置为该值会更好，因为这是每个 UDP 数据包的最大大小。然而，由于常见局域网的 MTU 大约是 1,500，我们可以在这里分配一个同样大小的缓冲区。`try_clone` 方法克隆给定的套接字并返回一个新的套接字，该套接字被移动到闭包中。

我们然后从套接字读取，在`Ok()`情况下返回读取的数据长度和源。然后我们启动一个新线程，在该线程中我们将相同的缓冲区写回到给定的套接字。对于任何可能失败的操作，我们都需要像处理 TCP 服务器那样处理错误。

与上次一样，与这个服务器交互——使用`nc`。唯一的区别是，在这种情况下，我们需要传递`-u`标志来强制`nc`只使用 UDP。看看下面的例子：

```rs
$ nc -u 127.0.0.1 8888
test
test
test
test
^C
```

现在，让我们编写一个简单的 UDP 客户端以实现相同的结果。正如我们将看到的，TCP 服务器和这个之间有一些细微的差别：

```rs
// chapter3/udp-client.rs

use std::net::UdpSocket;
use std::{str,io};

fn main() {
    let socket = UdpSocket::bind("127.0.0.1:8000")
                           .expect("Could not bind client socket");
    socket.connect("127.0.0.1:8888")
          .expect("Could not connect to server");
    loop {
        let mut input = String::new();
        let mut buffer = [0u8; 1500];
        io::stdin().read_line(&mut input)
                   .expect("Failed to read from stdin");
        socket.send(input.as_bytes())
              .expect("Failed to write to server");

        socket.recv_from(&mut buffer)
              .expect("Could not read into buffer");
        print!("{}", str::from_utf8(&buffer)
                         .expect("Could not write buffer as string"));
    }
}
```

与上一节中看到的 TCP 客户端相比，这个基本客户端有一个主要的不同之处。在这种情况下，在连接到服务器之前，`bind`到客户端套接字是绝对必要的。一旦完成，其余的例子基本上是相同的。在客户端和服务器端运行它会产生与 TCP 案例相似的结果。以下是服务器端的一个会话：

```rs
$ rustc udp-echo-server.rs && ./udp-echo-server
Handling connection from 127.0.0.1:8000
Handling connection from 127.0.0.1:8000
Handling connection from 127.0.0.1:8000
^C
```

在客户端，我们看到的是：

```rs
$ rustc udp-client.rs && ./udp-client
test
test
foo
foo
bar
bar
^C
```

# UDP 多播

`UdpSocket`类型有几个方法，而相应的 TCP 类型没有。其中最有趣的是用于多播和广播的方法。让我们通过一个示例服务器和客户端来看看多播是如何工作的。对于这个例子，我们将客户端和服务器合并到一个文件中。在`main`函数中，我们将检查是否传递了 CLI 参数。如果有，我们将运行客户端；否则，我们将运行服务器。请注意，参数的值将不会被使用；它将被视为一个布尔值：

```rs
// chapter3/udp-multicast.rs

use std::{env, str};
use std::net::{UdpSocket, Ipv4Addr};

fn main() {
    let mcast_group: Ipv4Addr = "239.0.0.1".parse().unwrap();
    let port: u16 = 6000;
    let any = "0.0.0.0".parse().unwrap();
    let mut buffer = [0u8; 1600];
    if env::args().count() > 1 {
        // client case
        let socket = UdpSocket::bind((any, port))
                               .expect("Could not bind client socket");
        socket.join_multicast_v4(&mcast_group, &any)
              .expect("Could not join multicast group");
        socket.recv_from(&mut buffer)
              .expect("Failed to write to server");
        print!("{}", str::from_utf8(&buffer)
                         .expect("Could not write buffer as string"));
    } else {
        // server case
        let socket = UdpSocket::bind((any, 0))
                               .expect("Could not bind socket");
        socket.send_to("Hello world!".as_bytes(), &(mcast_group, port))
              .expect("Failed to write data");
    }
}
```

这里的客户端和服务器部分与我们之前讨论的大多数相似。一个不同之处在于`join_multicast_v4`调用使当前套接字加入一个带有传递地址的多播组。对于服务器和客户端，我们在绑定时没有指定单个地址。相反，我们使用特殊地址`0.0.0.0`，表示任何可用的地址。这相当于向底层的`setsockopt`调用传递`INADDR_ANY`。在服务器的情况下，我们将其发送到多播组。运行这个稍微有点棘手。由于标准库中没有方法可以设置`SO_REUSEADDR`和`SO_REUSEPORT`，我们需要在多个不同的机器上运行客户端，并在另一台机器上运行服务器。为了使其工作，所有这些都需要在同一个网络中，并且多播组的地址需要是一个有效的多播地址（前四个位应该是 1110）。`UdpSocket`类型还支持离开多播组、广播等。请注意，由于广播在定义上是由两个主机之间的连接，所以对于 TCP 来说没有意义。

运行前面的例子很简单；在一个主机上，我们将运行服务器，在另一个主机上运行客户端。在这种配置下，服务器端的输出应该如下所示：

```rs
$ rustc udp-multicast.rs && ./udp-multicast server
Hello world!
```

# std::net 中的杂项实用工具

标准库中另一个重要的类型是 `IpAddr`，它表示一个 IP 地址。不出所料，它是一个有两个变体的枚举，一个用于 v4 地址，另一个用于 v6 地址。所有这些类型都有方法根据它们的类型（全局、环回、多播等）对地址进行分类。请注意，其中许多方法尚未稳定，因此仅在夜间编译器中可用。它们位于名为 `ip` 的功能标志之后，必须在 crate 根目录中包含该标志，以便可以使用这些方法。一个与之密切相关的类型是 `SocketAddr`，它是 IP 地址和端口号的组合。因此，它也有两个变体，一个用于 v4，一个用于 v6。让我们看看一些例子：

```rs
// chapter3/ip-socket-addr.rs

#![feature(ip)]

use std::net::{IpAddr, SocketAddr};

fn main() {
    // construct an IpAddr from a string and check it
    // represents the loopback address
    let local: IpAddr = "127.0.0.1".parse().unwrap();
    assert!(local.is_loopback());

    // construct a globally routable IPv6 address from individual
    octets
    // and assert it is classified correctly
    let global: IpAddr = IpAddr::from([0, 0, 0x1c9, 0, 0, 0xafc8, 0,
    0x1]);
    assert!(global.is_global());

    // construct a SocketAddr from a string an assert that the
    underlying
    // IP is a IPv4 address
    let local_sa: SocketAddr = "127.0.0.1:80".parse().unwrap();
    assert!(local_sa.is_ipv4());

    // construct a SocketAddr from a IPv6 address and a port, assert
    that
    // the underlying address is indeed IPv6
    let global_sa = SocketAddr::new(global, 80u16);
    assert!(global_sa.is_ipv6());
}
```

`feature(ip)` 声明是必要的，因为 `is_global` 函数尚未稳定。这个例子没有产生任何输出，因为所有断言都应该评估为真。

一个常见的功能是 DNS 查询，给定一个主机名。Rust 使用 `lookup_host` 函数来完成这项工作，该函数返回 `LookupHost` 类型，实际上是一个 DNS 响应的迭代器。让我们看看如何使用它。这个函数由 `looup_host` 标志控制，并且必须包含在夜间编译器中使用此函数：

```rs
// chapter3/lookup-host.rs

#![feature(lookup_host)]

use std::env;
use std::net::lookup_host;

fn main() {
    let args: Vec<_> = env::args().collect();
    if args.len() != 2 {
        eprintln!("Please provide only one host name");
        std::process::exit(1);
    } else {
        let addresses = lookup_host(&args[1]).unwrap();
        for address in addresses {
            println!("{}", address.ip());
        }
    }
}
```

在这里，我们读取一个 CLI 参数，如果没有给出恰好一个名称来解析，则退出。否则，我们使用给定的主机名调用 `lookup_host`。我们遍历返回的结果并打印每个的 IP 地址。请注意，每个返回的结果都是 `SocketAddr` 类型；因为我们只对 IP 地址感兴趣，所以我们使用 `ip()` 方法提取它。这个函数对应于 libc 中的 `getaddrinfo` 调用，因此它只返回 `A` 和 `AAAA` 记录类型。运行结果符合预期：

```rs
$ rustc lookup-host.rs && ./lookup-host google.com
2a00:1450:4009:810::200e
216.58.206.110
```

目前，标准库中无法进行反向 DNS 查询。在下一节中，我们将讨论生态系统中可以用于高级网络功能的一些 crate。例如，`trust-dns` crate 支持与 DNS 服务器进行更详细的交互，并且它还支持查询所有记录类型以及反向 DNS。

# 一些相关的 crate

一个细心的读者可能会注意到，许多常见的网络相关功能都缺失在标准库中。例如，没有处理 IP 网络（CIDR）的方法。让我们看看 `ipnetwork` crate 如何帮助解决这个问题。由于我们将使用外部 crate，示例必须在 cargo 项目中。我们需要将其添加为 `Cargo.toml` 中的依赖项。让我们先设置一个项目：

```rs
$ cargo new --bin ipnetwork-example
```

这将生成一个 `Cargo.toml` 文件，我们需要修改它以声明我们的依赖项。一旦我们这样做，它应该看起来像这样：

```rs
[package]
name = "ipnetwork-example"
version = "0.1.0"
authors = ["Foo <foo@bar.com>"]

[dependencies]
ipnetwork = "0.12.7"
```

在设置好项目后，让我们看看我们的 `main` 函数：

```rs
// chapter3/ipnetwork-example/src/main.rs

extern crate ipnetwork;

use std::net::Ipv4Addr;
use ipnetwork::{IpNetwork, Ipv4Network, Ipv6Network};

fn main() {
    let net = IpNetwork::new("192.168.122.0".parse().unwrap(), 22)
                        .expect("Could not construct a network");
    let str_net: IpNetwork = "192.168.122.0/22".parse().unwrap();

    assert!(net == str_net);
    assert!(net.is_ipv4());

    let net4: Ipv4Network = "192.168.121.0/22".parse().unwrap();
    assert!(net4.size() == 2u64.pow(32 - 22));
    assert!(net4.contains(Ipv4Addr::new(192, 168, 121, 3)));

    let _net6: Ipv6Network = "2001:db8::0/96".parse().unwrap();
    for addr in net4.iter().take(10) {
        println!("{}", addr);
    }
}
```

前两行展示了构建 `IpNetwork` 实例的两种不同方式，要么使用构造函数，要么通过解析字符串。接下来的 `assert` 确保它们确实是相同的。之后的 `assert` 确保我们创建的网络是一个 v4 网络。接下来，我们特别创建了 `Ipv4Network` 对象，正如预期的那样，网络的大小与 *2^(32 - prefix)* 匹配。下一个 `assert` 确保在该网络中的 `contains` 方法对 IP 地址正确工作。然后我们创建了一个 `Ipv6Network`，由于所有这些类型都实现了迭代器协议，我们可以通过 `for` 循环遍历网络并打印出单个地址。以下是运行最后一个示例时应看到的输出：

```rs
$ cargo run
   Compiling ipnetwork v0.12.7
   Compiling ipnetwork-example v0.1.0 (file:///Users/Abhishek/Desktop/rust-book/src/chapter3/ipnetwork-example)
    Finished dev [unoptimized + debuginfo] target(s) in 1.18 secs
     Running `target/debug/ipnetwork-example`
192.168.120.0
192.168.120.1
192.168.120.2
192.168.120.3
192.168.120.4
192.168.120.5
192.168.120.6
192.168.120.7
192.168.120.8
192.168.120.9
```

标准库也缺乏对套接字和连接的精细控制，一个例子就是设置 `SO_REUSEADDR` 的能力，正如我们之前所描述的。主要原因在于社区尚未能够就如何最好地暴露这些功能同时保持一个干净的 API 达成强有力的共识。在这个上下文中，一个有用的库是 `mio`，它提供了基于线程的并发性的替代方案。`mio` 实质上运行一个事件循环，所有参与者都在其中注册。当有事件发生时，每个监听器都会被通知，并且他们有处理该事件的选项。让我们看看以下示例。像上次一样，我们需要使用 cargo 设置项目：

```rs
$ cargo new --bin mio-example
```

下一步是将 `mio` 添加为依赖项；`Cargo.toml` 文件应如下所示：

```rs
[package]
name = "mio-example"
version = "0.1.0"
authors = ["Foo <foo@bar.com>"]

[dependencies]
mio = "0.6.11"
```

就像所有其他的 cargo 项目一样，我们需要在 `Cargo.toml` 中声明 `mio` 作为依赖项，并将其固定到特定版本，这样 cargo 就可以下载它并将其链接到我们的应用程序：

```rs
// chapter3/mio-example/src/main.rs

extern crate mio;

use mio::*;
use mio::tcp::TcpListener;

use std::net::SocketAddr;
use std::env;

// This will be later used to identify the server on the event loop
const SERVER: Token = Token(0);

// Represents a simple TCP server using mio
struct TCPServer {
    address: SocketAddr,
}

// Implementation for the TCP server
impl TCPServer {
    fn new(port: u32) -> Self {
        let address = format!("0.0.0.0:{}", port)
            .parse::<SocketAddr>().unwrap();

        TCPServer {
            address,
        }
    }

    // Actually binds the server to a given address and runs it
    // This function also sets up the event loop that dispatches
    // events. Later, we use a match on the token on the event
    // to determine if the event is for the server.
    fn run(&mut self) {
        let server = TcpListener::bind(&self.address)
            .expect("Could not bind to port");
        let poll = Poll::new().unwrap();
        poll.register(&server, 
                       SERVER,
                       Ready::readable(),
                       PollOpt::edge()).unwrap();

        let mut events = Events::with_capacity(1024);
        loop {
            poll.poll(&mut events, None).unwrap();

            for event in events.iter() {
                match event.token() {
                    SERVER => {
                        let (_stream, remote) =
                        server.accept().unwrap();
                        println!("Connection from {}", remote);
                    }
                    _ => {
                        unreachable!();
                    }
                }
            }
        }
    }
}

fn main() {
    let args: Vec<String> = env::args().collect();
    if args.len() != 2 {
        eprintln!("Please provide only one port number as argument");
        std::process::exit(1);
    }
    let mut server = TCPServer::new(args[1].parse::<u32>()
                               .expect("Could not parse as u32"));
    server.run();
}
```

与我们之前的示例不同，这是一个仅打印客户端源 IP 和端口的 TCP 服务器。在 `mio` 中，事件循环上的每个监听器都会分配一个令牌，该令牌可以用来在事件传递时区分监听器。我们在构造函数中定义了一个用于我们的服务器（`TCPServer`）的结构体，并将其 `bind` 到所有本地地址，并返回该结构体的一个实例。该结构体的 `run` 方法将套接字绑定到给定的套接字地址；然后它使用 `Poll` 结构体来实例化事件循环。

然后它注册了服务器套接字，并在实例上设置了一个令牌。我们还指示当事件准备好读取或写入时应该发出警报。最后，我们指示我们只想接收边缘触发的事件，这意味着事件在接收时应完全消耗，否则在相同令牌上的后续调用可能会阻塞它。然后我们为我们的事件设置了一个空的容器。完成所有样板代码后，我们进入一个无限循环，并使用我们刚刚创建的事件容器开始轮询。我们遍历事件列表，如果任何事件令牌与服务器令牌匹配，我们知道它是为服务器准备的。然后我们可以接受连接并打印远程端的信息。然后我们回到下一个事件，依此类推。在我们的`main`函数中，我们首先处理 CLI 参数，确保我们传递了一个整数端口号。然后，我们实例化服务器并调用其上的`run`方法。

这里是一个示例会话，展示了当两个客户端连接到服务器时运行服务器的情况。请注意，可以使用`nc`或之前提到的 TCP 客户端来连接到这个服务器：

```rs
$ cargo run 4321
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/mio-example 4321`
Connection from 127.0.0.1:60955
Connection from 127.0.0.1:60956
^C
```

从标准库和这里讨论的 crates 中缺失的一些主要功能包括与物理网络设备工作的能力、一个更友好的 API 来构建和解析数据包等。有一个 crate 可以帮助处理`libpnet`中的底层网络相关事务。让我们使用它来编写一个小的数据包转储器：

```rs
$ cat Cargo.toml
[package]
name = "pnet-example"
version = "0.1.0"
authors = ["Foo Bar <foo@bar.com>"]

[dependencies]
pnet = "0.20.0"
```

我们初始化我们的 Cargo 项目如下：

```rs
$ cargo new --bin pnet-example
```

然后我们将`pnet`添加为依赖项，将其锁定到特定版本（目前可用的最新版本）。然后我们可以继续到我们的源代码，它应该看起来像这样：

```rs
// chapter3/pnet-example/src/main.rs

extern crate pnet;

use pnet::datalink::{self, NetworkInterface};
use pnet::datalink::Channel::Ethernet;
use pnet::packet::ethernet::{EtherTypes, EthernetPacket};
use pnet::packet::ipv4::Ipv4Packet;
use pnet::packet::tcp::TcpPacket;
use pnet::packet::ip::IpNextHeaderProtocols;
use pnet::packet::Packet;

use std::env;

// Handles a single ethernet packet
fn handle_packet(ethernet: &EthernetPacket) {
    match ethernet.get_ethertype() {
        EtherTypes::Ipv4 => {
            let header = Ipv4Packet::new(ethernet.payload());
            if let Some(header) = header {
                match header.get_next_level_protocol() {
                    IpNextHeaderProtocols::Tcp => {
                        let tcp = TcpPacket::new(header.payload());
                        if let Some(tcp) = tcp {
                            println!(
                                "Got a TCP packet {}:{} to {}:{}",
                                header.get_source(),
                                tcp.get_source(),
                                header.get_destination(),
                                tcp.get_destination()
                            );
                        }
                    }
                    _ => println!("Ignoring non TCP packet"),
                }
            }
        }
        _ => println!("Ignoring non IPv4 packet"),
    }
}

fn main() {
    let interface_name = env::args().nth(1).unwrap();

    // Get all interfaces
    let interfaces = datalink::interfaces();
    // Filter the list to find the given interface name
    let interface = interfaces
        .into_iter()
        .filter(|iface: &NetworkInterface| iface.name == interface_name)
        .next()
        .expect("Error getting interface");

    let (_tx, mut rx) = match datalink::channel(&interface, Default::default()) {
        Ok(Ethernet(tx, rx)) => (tx, rx),
        Ok(_) => panic!("Unhandled channel type"),
        Err(e) => {
            panic!(
                "An error occurred when creating the datalink channel:
                {}",e
            )
        }
    };

    // Loop over packets arriving on the given interface
    loop {
        match rx.next() {
            Ok(packet) => {
                let packet = EthernetPacket::new(packet).unwrap();
                handle_packet(&packet);
            }
            Err(e) => {
            panic!("An error occurred while reading: {}", e);
            }
        }
    }
}
```

如往常一样，我们首先声明`pnet`为一个外部 crate。然后我们导入我们将要使用的一堆东西。我们通过 CLI 参数接收要嗅探的接口名称。`datalink::interfaces()`给我们当前主机上所有可用接口的列表，我们通过提供的接口名称过滤这个列表。如果我们找不到匹配项，我们抛出一个错误并退出。`datalink::channel()`调用给我们一个发送和接收数据包的通道。在这种情况下，我们不需要关心发送端，因为我们只对嗅探数据包感兴趣。我们根据返回的通道类型进行匹配，以确保我们只处理以太网。通道的接收端`rx`给我们一个迭代器，它在每次`next()`调用时产生数据包。

数据包随后传递给`handle_packet`函数，该函数提取相关信息并打印出来。对于这个玩具示例，我们将只处理基于 IPv4 的 TCP 数据包。现实中的网络显然会得到带有 UDP 和 TCP 的 IPv6 和 ICMP 数据包。所有这些组合都将在此忽略。

在 `handle_packet` 函数中，我们根据数据包的 ethertype 匹配以确保我们只处理 IPv4 数据包。由于整个以太网数据包的有效负载是 IP 数据包（参看第一章，*客户端/服务器网络简介*)，我们从有效负载中构造一个 IP 数据包。`get_next_level_protocol()` 调用返回传输协议，如果它匹配 TCP，我们就从前一层的有效负载中构造一个 TCP 数据包。在这个时候，我们可以从 TCP 数据包中打印出源端口和目标端口。源 IP 和目标 IP 将在封装的 IP 数据包中。以下代码块显示了示例运行。我们需要将监听接口的名称作为命令行参数传递给我们的程序。以下是在 Linux 中获取接口名称的方法：

```rs
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether f4:4d:30:ac:88:ee brd ff:ff:ff:ff:ff:ff
    inet 192.168.5.15/22 brd 192.168.7.255 scope global enp1s0
       valid_lft forever preferred_lft forever
    inet6 fe80::58c6:9ccc:e78c:caa6/64 scope link 
       valid_lft forever preferred_lft forever
```

在这个示例中，我们将忽略回环接口 `l0`，因为它不接收很多流量，并使用其他接口 `enp1s0`。我们还将以 root 权限（使用 sudo）运行此示例，因为它需要直接访问网络设备。

第一步是使用 cargo 构建项目并运行可执行文件。请注意，这个示例的确切输出可能略有不同，这取决于到达的数据包：

```rs
$ cargo build
$ sudo ./target/debug/pnet-example enp1s0
Got a TCP packet 192.168.0.2:53041 to 104.82.249.116:443
Got a TCP packet 104.82.249.116:443 to 192.168.0.2:53041
Got a TCP packet 192.168.0.2:53064 to 17.142.169.200:443
Got a TCP packet 192.168.0.2:53064 to 17.142.169.200:443
Got a TCP packet 17.142.169.200:443 to 192.168.0.2:53064
Got a TCP packet 17.142.169.200:443 to 192.168.0.2:53064
Got a TCP packet 192.168.0.2:53064 to 17.142.169.200:443
Got a TCP packet 192.168.0.2:52086 to 52.174.153.60:443
Ignoring non IPv4 packet
Got a TCP packet 52.174.153.60:443 to 192.168.0.2:52086
Got a TCP packet 192.168.0.2:52086 to 52.174.153.60:443
Ignoring non IPv4 packet
Ignoring non IPv4 packet
Ignoring non IPv4 packet
Ignoring non IPv4 packet
```

在上一节中，我们看到了标准库中 DNS 相关功能的局限性。`trust-dns` 是一个在 DNS 相关事务中广泛流行的 crate。让我们看看如何使用它来查询给定名称的示例。让我们从一个空项目开始：

```rs
$ cargo new --bin trust-dns-example
```

然后，我们首先在 `Cargo.toml` 中添加所需 crate 的版本：

```rs
[package]
name = "trust-dns-example"
version = "0.1.0"
authors = ["Foo <foo@bar.com>"]

[dependencies]
trust-dns-resolver = "0.6.0"
trust-dns = "0.12.0"
```

我们的应用程序依赖于 `trust-dns` 来处理 DNS 相关的事务。像往常一样，在使用我们的应用程序之前，我们将它添加到 `Cargo.toml` 中：

```rs
// chapter3/trust-dns-example/src/main.rs

extern crate trust_dns_resolver;
extern crate trust_dns;

use std::env;

use trust_dns_resolver::Resolver;
use trust_dns_resolver::config::*;

use trust_dns::rr::record_type::RecordType;

fn main() {
    let args: Vec<String> = env::args().collect();
    if args.len() != 2 {
        eprintln!("Please provide a name to query");
        std::process::exit(1);
    }
    let resolver = Resolver::new(ResolverConfig::default(),
                                 ResolverOpts::default()).unwrap();

    // Add a dot to the given name
    let query = format!("{}.", args[1]);

    // Run the DNS query
    let response = resolver.lookup_ip(query.as_str());
    println!("Using the synchronous resolver");
    for ans in response.iter() {
        println!("{:?}", ans);
    }

    println!("Using the system resolver");
    let system_resolver = Resolver::from_system_conf().unwrap();
    let system_response = system_resolver.lookup_ip(query.as_str());
    for ans in system_response.iter() {
        println!("{:?}", ans);
    }

    let ns = resolver.lookup(query.as_str(), RecordType::NS);
    println!("NS records using the synchronous resolver");
    for ans in ns.iter() {
        println!("{:?}", ans);
    }
}
```

我们设置了所有必需的导入和 `extern` crate 声明。在这里，我们期望从 CLI 参数中获取要解析的名称，如果一切顺利，它应该在 `args[1]` 中。这个 crate 支持两种类型的同步 DNS 解析器。`Resolver::new` 创建一个同步解析器，默认情况下，它将使用 Google 的公共 DNS 作为上游服务器。`Resolver::from_system_conf` 创建一个使用系统 `resolv.conf` 配置的同步解析器。因此，这个第二个选项仅在 Unix 系统上可用。在我们将查询传递给 `resolver` 之前，我们使用 `format!` 宏将名称格式化为 FQDN，并在名称后附加一个 `.`。我们使用 `lookup_ip` 函数传递查询，该函数随后返回 DNS 问题的答案的迭代器。一旦我们得到它，我们就可以遍历它并打印出每个答案。正如其名所示，`lookup_ip` 函数只查找 `A` 和 `AAAA` 记录。还有一个更通用的 `lookup` 函数，可以接受要查询的记录类型。在最后一步，我们想要获取给定名称的所有 `NS` 记录。一旦我们得到答案，我们就遍历它并打印结果。

`trust-dns` 也支持基于 tokio 的异步 DNS 解析器。

一个示例会话将看起来像这样：

```rs
$ cargo run google.com
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/trust-dns-example google.com`
Using the synchronous resolver
LookupIp(Lookup { rdatas: [AAAA(2a00:1450:4009:811::200e), A(216.58.206.142)] })
Using the system resolver
LookupIp(Lookup { rdatas: [A(216.58.206.110), AAAA(2a00:1450:4009:810::200e)] })
NS records using the synchronous resolver
Lookup { rdatas: [NS(Name { is_fqdn: true, labels: ["ns3", "google", "com"] }), NS(Name { is_fqdn: true, labels: ["ns1", "google", "com"] }), NS(Name { is_fqdn: true, labels: ["ns4", "google", "com"] }), NS(Name { is_fqdn: true, labels: ["ns2", "google", "com"] })] }
```

在本例中，所有打印都使用结构的调试表示形式。实际应用可能需要按需格式化这些内容。

# 摘要

本章简要介绍了 Rust 中的基本网络功能。我们从 `std::net` 中给出的功能开始，并使用这些功能编写了一些 TCP 和 UDP 服务器。然后，我们查看了一些同一命名空间中的其他实用工具。最后，我们回顾了几个旨在扩展标准库网络功能功能的 crate 的示例。请注意，始终可以使用基于 POSIX 兼容网络代码并具有对套接字和网络设备细粒度控制的 libc crate 来编写网络代码，但这可能会导致代码不安全，违反 Rust 的安全性保证。另一个名为 nix 的 crate 旨在提供原生的 Rust libc 功能，以便它保留编译器提供的所有内存和类型安全性保证：这可能是一个需要非常细粒度控制网络的人的有用替代方案。

在下一章中，我们将探讨在服务器或客户端接收到数据后，如何使用 Rust 生态系统中的多种序列化/反序列化方法来处理数据。
