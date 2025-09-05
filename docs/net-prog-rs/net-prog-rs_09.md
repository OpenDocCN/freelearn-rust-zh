# 附录

Rust 是一个开源项目，拥有来自世界各地的众多贡献者。与任何此类项目一样，通常存在多种解决同一问题的方法。crate 生态系统使得这变得更容易，因为人们可以发布多个 crate，提出解决同一组问题的多种方法。这种方法在真正的开源精神中培养了健康的竞争感。在本书中，我们已经涵盖了生态系统中的许多不同主题。本章将是对这些主题的补充。我们将讨论：

+   基于协程和生成器的并发方法

+   async/await 抽象

+   数据并行

+   使用 Pest 进行解析

+   生态系统中的各种实用工具

# 协程和生成器简介

我们之前探讨了 Tokio 生态系统。我们看到了在 Tokio 中链式连接 future 是非常常见的，这产生了一个更大的任务，然后可以按需调度。在实践中，以下通常看起来像伪代码：

```rs
fn download_parse(url: &str) {
    let task = download(url)
               .and_then(|data| parse_html(data))
               .and_then(|link| process_link(link))
               .map_err(|e| OurErr(e));
    Core.run(task);
}
```

在这里，我们的函数接收一个 URL 并递归地下载原始 HTML。然后它解析并收集文档中的链接。我们的任务在事件循环中运行。由于所有回调及其交互，这里的控制流可能更难以跟踪。随着任务规模增大和条件分支增多，这变得更加复杂。

协程的概念帮助我们更好地理解这些例子中的非线性。协程（也称为**生成器**）是函数的一种推广，它可以随意挂起或恢复自身，并将控制权交给另一个实体。由于它们不需要被超实体抢占，因此通常更容易处理。请注意，我们始终假设一个协作模型，其中协程在等待计算完成或 I/O 时会挂起，并且我们忽略了协程可能以其他方式恶意行为的案例。当协程再次开始执行时，它会从上次离开的地方继续，提供一种连续性。它们通常有多个入口和出口点。

在设置好之后，一个子程序（函数）就变成了协程的一个特殊情况，它有且只有一个入口和出口点，并且不能被外部挂起。计算机科学文献也区分了生成器和协程，认为前者在挂起后无法控制执行继续的位置，而后者可以。然而，在我们当前的环境中，我们将互换使用生成器和协程。

协程可以分为两大类：无栈协程和有栈协程。无栈协程在挂起时不会维护栈。因此，它们不能在任意位置恢复。相反，有栈协程始终维护一个小栈。这使得它们可以从执行中的任意点挂起和恢复。它们在挂起时总是保留完整的状态。因此，从调用者的角度来看，它们的行为就像任何可以独立于当前执行运行的常规函数。在实践中，有栈协程通常更占用资源，但更容易与调用者一起工作。请注意，与线程和进程相比，所有协程的资源消耗都较小。

在最近过去，Rust 在标准库中实验了一种生成器实现。这些位于 `std::ops` 中，并且像所有新功能一样，这背后有多个功能门：`generators` 和 `generator_trait`。这个实现有几个部分。首先，有一个新的 `yield` 关键字用于从生成器中 `yield`。生成器通过重载闭包语法来定义。其次，有一些项目如下定义：

```rs
pub trait Generator {
    type Yield;
    type Return;
    fn resume(&mut self) -> GeneratorState<Self::Yield, Self::Return>; 
}
```

由于这些是生成器（而不是协程），它们只能有一个 `yield` 和一个 `return`。

`Generator` 特性有两种类型：一种用于 `yield` 的情况，另一种用于 `return` 的情况。`resume` 函数从最后一个点恢复执行。`resume` 的返回值是 `GeneratorState`，这是一个枚举类型，看起来如下：

```rs
pub enum GeneratorState<Y, R> {
    Yielded(Y),
    Complete(R)
}
```

有两个变体；`Yielded` 代表 `yield` 语句的变体，而 `Complete` 代表 `return` 语句的变体。此外，`Yielded` 变体表示生成器可以从最后一个 `yield` 语句继续执行。`Complete` 表示生成器已经完成执行。

让我们看看一个使用这个生成 Collatz 序列的例子：

```rs
// appendix/collatz-generator/src/main.rs

#![feature(generators, generator_trait)]

use std::ops::{Generator, GeneratorState};
use std::env;

fn main() {
    let input = env::args()
        .nth(1)
        .expect("Please provide only one argument")
        .parse::<u64>()
        .expect("Could not convert input to integer");

    // The given expression will evaluate to our generator
    let mut generator = || {
        let end = 1u64;
        let mut current: u64 = input;
        while current != end {
            yield current;
            if current % 2 == 0 {
                current /= 2;
            } else {
                current = 3 * current + 1;
            }
        }
        return end;
    };

    loop {
        // The resume call can have two results. If we have an
        // yielded value, we print it. If we have a completed
        // value, we print it and break from the loop (this is
        // the return case)
        match generator.resume() {
            GeneratorState::Yielded(el) => println!("{}", el),
            GeneratorState::Complete(el) => {
                println!("{}", el);
                break;
            }
        }
    }
}
```

由于这些在标准库中，我们不需要外部依赖。我们从功能门和导入开始。在我们的主函数中，我们使用闭包语法定义一个生成器。注意它如何从封装作用域捕获名为 `input` 的变量。我们不是返回序列的当前位置，而是 `yield` 它。当我们完成时，我们从生成器返回。现在我们需要在生成器上调用 `resume` 来实际运行它。我们在一个无限循环中这样做，因为我们事先不知道需要迭代多少次。在这个循环中，我们对 `resume` 调用进行 `match`；在两个分支中，我们打印出我们拥有的值。此外，在 `Complete` 分支中，我们需要从循环中 `break` 出来。

注意，我们在这里没有使用隐式返回语法；而是执行了显式的 `return end;`。这不是必需的，但在这个情况下这使得代码更容易阅读。

这就是它产生的结果：

```rs
$ cargo run 10
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/collatz-generator 10`
10
5
16
8
4
2
1
```

目前，生成器仅在 nightly Rust 中可用。它们的语法预计会随着时间的推移而改变，可能变化很大。

# May 处理协程的方式

Rust 生态系统有几个协程实现，而核心团队正在努力进行语言内实现。在这些实现中，一个广泛使用的是名为*May*的实现，它是一个基于生成器的独立堆栈式协程库。May 力求足够用户友好，以便可以使用一个简单的宏异步调用函数，该宏接受函数作为参数。与 Go 编程语言的功能相匹配，这个宏被称为`go!`。让我们看看使用 May 的一个小例子。

我们将使用我们熟悉的老朋友，Collatz 序列，来举例；这将展示我们实现相同目标的多种方式。让我们从使用 Cargo 设置我们的项目开始：

```rs
[package]
name = "collatz-may"
version = "0.1.0"
authors = ["Foo <foo@bar.com>"]

[dependencies]
may = "0.2.0"
generator = "0.6"
```

以下为主要文件。这里有两个示例；一个使用生成器 crate 在 Collatz 序列中产生数字，充当协程。另一个是一个常规函数，它使用`go!`宏作为协程运行：

```rs
// appendix/collatz-may/src/main.rs

#![feature(conservative_impl_trait)]

#[macro_use]
extern crate generator;
#[macro_use]
extern crate may;

use std::env;
use generator::Gn;

// Returns a generator as an iterator
fn collatz_generator(start: u64) -> impl Iterator<Item = u64> {
    Gn::new_scoped(move |mut s| {
        let end = 1u64;
        let mut current: u64 = start;
        while current != end {
            s.yield_(current);
            if current % 2 == 0 {
                current /= 2;
            } else {
                current = 3 * current + 1;
            }
        }
        s.yield_(end);
        done!();
    })
}

// A regular function returning result in a vector
fn collatz(start: u64) -> Vec<u64> {
    let end = 1u64;
    let mut current: u64 = start;
    let mut result = Vec::new();
    while current != end {
        result.push(current);
        if current % 2 == 0 {
            current /= 2;
        } else {
            current = 3 * current + 1;
        }
    }
    result.push(end);
    result
}

fn main() {
    let input = env::args()
        .nth(1)
        .expect("Please provide only one argument")
        .parse::<u64>()
        .expect("Could not convert input to integer");

    // Using the go! macro
    go!(move || {
        println!("{:?}", collatz(input));
    }).join()
        .unwrap();

    // results is a generator expression that we can
    // iterate over
    let results = collatz_generator(input);
    for result in results {
        println!("{}", result);
    }
}
```

让我们从`collatz_generator`函数开始，它接受一个起始输入并返回一个类型为`u64`的迭代器。为了能够指定这一点，我们需要激活`conservative_impl_trait`功能。我们使用生成器 crate 中的`Gn::new_scoped`创建一个作用域生成器。它接受一个闭包，实际上执行计算。我们使用`yield_`函数产生当前值，并使用`done!`宏表示计算结束。

我们的第二个示例是一个返回 Collatz 序列中数字向量的常规函数。它在向量中收集中间结果，并在序列达到`1`时最终返回它。在我们的主函数中，我们像往常一样解析和清理输入。然后我们使用`go!`宏异步地在协程中调用我们的非生成器函数。然而，`collatz_generator`返回一个迭代器，我们可以在循环中迭代它，同时打印出数字。

如预期的那样，输出看起来是这样的：

```rs
$ cargo run 10
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/collatz-may 10`
[10, 5, 16, 8, 4, 2, 1]
10
5
16
8
4
2
1
```

May 还包括一个异步网络堆栈实现（如 Tokio），在过去的几个月里，它已经聚集了一个由几个依赖 crate 组成的迷你生态系统。除了生成器和协程实现之外，还有一个基于这些的 HTTP 库，一个 RPC 库，以及一个支持基于 actor 编程的 crate。让我们看看一个使用 May 编写 hyper HTTP 的示例。以下是 Cargo 配置的样子：

```rs
[package]
name = "may-http"
version = "0.1.0"
authors = ["Foo <foo@bar.com>"]

[dependencies]
may_minihttp = { git = "https://github.com/Xudong-Huang/may_minihttp.git" }
may = "0.2.0"
num_cpus = "1.7.0"
```

在撰写本文时，`may_minihttp`尚未在`crates.io`上发布，因此我们需要使用仓库来构建。以下是主要文件：

```rs
// appendix/may-http/src/main.rs

extern crate may;
extern crate may_minihttp;
extern crate num_cpus;

use std::io;
use may_minihttp::{HttpServer, HttpService, Request, Response};
use std::{thread, time};

fn heavy_work() -> String {
    let duration = time::Duration::from_millis(100);
    thread::sleep(duration);
    "done".to_string()
}

#[derive(Clone, Copy)]
struct Echo;

// Implementation of HttpService for our service struct Echo
// This implementation defines how content should be served
impl HttpService for Echo {
    fn call(&self, req: Request) -> io::Result<Response> {
        let mut resp = Response::new();
        match (req.method(), req.path()) {
            ("GET", "/data") => {
                let b = heavy_work();
                resp.body(&b).status_code(200, "OK");
            }
            (&_, _) => {
                resp.status_code(404, "Not found");
            }
        }
        Ok(resp)
    }
}

fn main() {
    // Set the number of IO workers to the number of cores
    may::config().set_io_workers(num_cpus::get());
    let server = HttpServer(Echo).start("0.0.0.0:3000").unwrap();
    server.join().unwrap();
}
```

这次，我们的代码比 Hyper 要短得多。这是因为 May 很好地抽象了很多东西，同时让我们拥有类似的功能集。就像之前的`Service` trait 一样，`HttpService` trait 定义了服务器。功能由`call`函数定义。在函数调用和响应构建方面有一些细微的差异。这里的另一个优点是它不暴露 future，并且与常规`Result`一起工作。可以说，这个模型更容易理解和遵循。在`main`函数中，我们将 I/O 工作者的数量设置为我们的核心数。然后我们在端口`3000`上启动服务器并等待其退出。根据 GitHub 页面上的某些粗略基准测试，基于 May 的 HTTP 服务器比 Tokio 实现略快。

运行服务器后，我们看到的是这个。在这个特定的运行中，它收到了两个`GET`请求：

```rs
$ cargo run
   Compiling may-http v0.1.0 (file:///Users/Abhishek/Desktop/rust-book/src/appendix/may-http)
    Finished dev [unoptimized + debuginfo] target(s) in 1.57 secs
     Running `target/debug/may-http`
Incoming request <HTTP Request GET /data>
Incoming request <HTTP Request GET /data>
^C
```

我们客户端只是`curl`，并且我们看到每个请求都会打印出`done`。请注意，由于我们的服务器和客户端在同一台物理机器上，我们可以使用`127.0.0.1`作为服务器的地址。如果不是这种情况，应该使用实际的地址：

```rs
$ curl http://127.0.0.1:3000/data
done
```

# 等待 future

在上一节中，我们看到了由多个 future 组成的任务通常难以编写和调试。一个试图解决这个问题的方法是使用一个封装 future 和相关错误处理的 crate，从而产生更线性的代码流。这个 crate 被称为*futures-await*，并且正在积极开发中。

这个 crate 提供了处理 future 的两种主要机制：

+   可以应用于函数的`#[async]`属性，将其标记为异步。这些函数必须返回它们的计算`Result`作为 future。

+   可以与异步函数一起使用的`await!`宏，用于消耗 future，返回一个`Result`。

给定这些构造，我们之前的示例下载将看起来像这样：

```rs
#[async]
fn download(url: &str) -> io::Result<Data> {
    ...
}

#[async]
fn parse_html(data: &Data) -> io::Result<Links> {
    ...
}

#[async]
fn process_links(links: &Links) -> io::Result<bool> {
    ...
}

#[async]
fn download_parse(url: &str) -> io::Result<bool> {
    let data = await!(download(url));
    let links = await!(parse_html(data));
    let result = await!(process_links(links));
    Ok(result)
}

fn main() {
    let task = download_parse("foo.bar");
    Core.run(task).unwrap();
}
```

这可能比使用 future 的例子更容易阅读。内部，编译器将此转换为类似于我们之前的示例的代码。此外，由于每个步骤都返回一个`Result`，我们可以使用`?`运算符优雅地冒泡错误。最终的任务可以像往常一样在事件循环中运行。

让我们来看一个更具体的例子，使用这个 crate 重写我们的超服务器项目。在这种情况下，我们的 Cargo 设置看起来是这样的：

```rs
[package]
name = "hyper-async-await"
version = "0.1.0"
authors = ["Foo <foo@bar.com>"]

[dependencies]
hyper = "0.11.7"
futures = "0.1.17"
net2 = "0.2.31"
tokio-core = "0.1.10"
num_cpus = "1.0"
futures-await = "0.1.0"
```

下面是我们的代码，如下面的代码片段所示。请注意，我们没有使用像上次那样的 futures crate 中的类型。相反，我们使用了从 futures-await 重新导出的类型，这些类型是这些类型的封装版本：

```rs
// appendix/hyper-async-await/src/main.rs

#![feature(proc_macro, conservative_impl_trait, generators)]

extern crate futures_await as futures;
extern crate hyper;
extern crate net2;
extern crate tokio_core;
extern crate num_cpus;

use futures::prelude::*;
use net2::unix::UnixTcpBuilderExt;
use tokio_core::reactor::Core;
use tokio_core::net::TcpListener;
use std::{thread, time};
use std::net::SocketAddr;
use std::sync::Arc;
use hyper::{Get, StatusCode};
use hyper::header::ContentLength;
use hyper::server::{Http, Service, Request, Response};
use futures::future::FutureResult;
use std::io;

// Our blocking function that waits for a random
// amount of time and then returns a fixed string
fn heavy_work() -> String {
    let duration = time::Duration::from_millis(100);
    thread::sleep(duration);
    "done".to_string()
}

#[derive(Clone, Copy)]
struct Echo;

// Service implementation for the Echo struct
impl Service for Echo {
    type Request = Request;
    type Response = Response;
    type Error = hyper::Error;
    type Future = FutureResult<Response, hyper::Error>;

    fn call(&self, req: Request) -> Self::Future {
        futures::future::ok(match (req.method(), req.path()) {
            (&Get, "/data") => {
                let b = heavy_work().into_bytes();
                Response::new()
                    .with_header(ContentLength(b.len() as u64))
                    .with_body(b)
            }
            _ => Response::new().with_status(StatusCode::NotFound),
        })
    }
}

// Sets up everything and runs the event loop
fn serve(addr: &SocketAddr, protocol: &Http) {
    let mut core = Core::new().unwrap();
    let handle = core.handle();
    let listener = net2::TcpBuilder::new_v4()
        .unwrap()
        .reuse_port(true)
        .unwrap()
        .bind(addr)
        .unwrap()
        .listen(128)
        .unwrap();
    let listener = TcpListener::from_listener
    (listener, addr, &handle).unwrap();
    let server = async_block! {
        #[async]
        for (socket, addr) in listener.incoming() {
            protocol.bind_connection(&handle, socket, addr, Echo);
        }
        Ok::<(), io::Error>(())
    };
    core.run(server).unwrap();
}

fn start_server(num: usize, addr: &str) {
    let addr = addr.parse().unwrap();

    let protocol = Arc::new(Http::new());
    {
        for _ in 0..num - 1 {
            let protocol = Arc::clone(&protocol);
            thread::spawn(move || serve(&addr, &protocol));
        }
    }
    serve(&addr, &protocol);
}

fn main() {
    start_server(num_cpus::get(), "0.0.0.0:3000");
}
```

`async_block!`宏接受一个闭包并将其转换为`async`函数。因此，我们这里的服务器是一个`async`函数。我们还使用异步的`for`循环（一个标记为`#[async]`的`for`循环）异步地遍历连接流。其余的代码与上次完全相同。运行服务器很简单；我们将使用 Cargo：

```rs
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/hyper-server-faster`
```

在客户端，我们可以使用 curl：

```rs
$ curl http://127.0.0.1:3000/data
done$ curl http://127.0.0.1:3000/data
done$
```

在撰写本文时，运行此示例将产生关于使用 *bind_connection* 的警告。由于没有明确的弃用时间表，我们将暂时忽略此警告。

# 数据并行

数据并行是一种通过使数据成为中心实体来加速计算的方法。这与我们迄今为止看到的基于协程和线程的并行处理形成对比。在那些情况下，我们首先确定可以独立运行的任务。然后根据需要将可用数据分配给那些任务。这种方法通常被称为 **任务并行**。本节讨论的主题是数据并行。在这种情况下，我们需要找出哪些输入数据部分可以独立使用；然后可以将多个任务分配给各个部分。这也符合分而治之的方法，一个强有力的例子是 `mergesort`。

Rust 生态系统有一个名为 **Rayon** 的库，它提供了编写数据并行代码的简单 API。让我们看看使用 Rayon 对给定切片进行二分查找的简单示例。我们首先使用 `cargo` 设置我们的项目：

```rs
$ cargo new --bin rayon-search
```

让我们看看 Cargo 配置文件：

```rs
[package]
name = "rayon-search"
version = "0.1.0"
authors = ["Foo <foo@bar.com>"]

[dependencies]
rayon = "0.9.0"
```

在我们的代码中，我们实现了两种二分查找函数，两者都是递归的。原始实现称为 `binary_search_recursive`，并且不执行任何数据并行处理。另一种版本，称为 `binary_search_rayon`，并行计算两种情况。这两个函数都接受一个类型为 `T` 的切片，该切片实现了一些特质。它们还接受相同类型的元素。这些函数将在切片中查找指定的元素，如果存在则返回 `true`，否则返回 `false`。现在让我们看看代码：

```rs
// appendix/rayon-search/src/main.rs

extern crate rayon;

use std::fmt::Debug;
use rayon::scope;

// Parallel binary search, searches the two halves in parallel
fn binary_search_rayon<T: Ord + Send + Copy + Sync + Debug>(src: &mut [T], el: T) -> bool {
    src.sort();
    let mid = src.len() / 2;
    let srcmid = src[mid];
    if src.len() == 1 && src[0] != el {
        return false;
    }
    if el == srcmid {
        true
    } else {
        let mut left_result = false;
        let mut right_result = false;
        let (left, right) = src.split_at_mut(mid);
        scope(|s| if el < srcmid {
            s.spawn(|_| left_result = binary_search_rayon(left, el))
        } else {
            s.spawn(|_| right_result = binary_search_rayon(right, el))
        });
        left_result || right_result
    }
}

// Non-parallel binary search, goes like this:
// 1\. Sort input
// 2\. Find middle point, return if middle point is target
// 3\. If not, recursively search left and right halves
fn binary_search_recursive<T: Ord + Send + Copy>(src: &mut [T], el: T) -> bool {
    src.sort();
    let mid = src.len() / 2;
    let srcmid = src[mid];
    if src.len() == 1 && src[0] != el {
        return false;
    }
    if el == srcmid {
        true
    } else {
        let (left, right) = src.split_at_mut(mid);
        if el < srcmid {
            binary_search_recursive(left, el)
        } else {
            binary_search_recursive(right, el)
        }
    }
}

fn main() {
    let mut v = vec![100, 12, 121, 1, 23, 35];
    println!("{}", binary_search_recursive(&mut v, 5));
    println!("{}", binary_search_rayon(&mut v, 5));
    println!("{}", binary_search_rayon(&mut v, 100));
}
```

在这两种情况下，首先要做的事情是对输入切片进行排序，因为二分查找需要排序的输入。`binary_search_recursive` 是直接的；我们计算中间点，如果那里的元素是我们想要的，我们返回 `true`。我们还包括一个检查切片中是否只剩下一个元素，并且如果那个元素是我们想要的。相应地返回 `true` 或 `false`。这个情况形成了我们递归的基础情况。然后我们可以检查我们想要的元素是否小于或大于当前的中点。根据这个检查，我们递归地搜索两边。

Rayon 的情况基本上相同，唯一的区别在于我们如何递归。我们使用 `scope` 并行生成两种情况并收集它们的结果。作用域接受一个闭包，并在这个命名作用域 `s` 中调用它。作用域还确保在退出之前每个任务都已完成。我们从每个半部分收集结果到两个变量中，最后，我们返回这些结果的逻辑或，因为我们关心的是在两个半部分中的任何一个找到元素。以下是一个示例运行情况：

```rs
$ cargo run
   Compiling rayon-search v0.1.0 (file:///Users/Abhishek/Desktop/rust-book/src/appendix/rayon-search)
    Finished dev [unoptimized + debuginfo] target(s) in 0.88 secs
     Running `target/debug/rayon-search`
false
false
true
```

Rayon 还提供了一个并行迭代器，这是一个与标准库中的迭代器具有相同语义的迭代器，但元素可能以并行方式访问。这种结构在每种数据单元都可以完全独立处理，且每个处理任务之间没有任何同步的情况下非常有用。让我们看看如何使用这些迭代器的例子，从使用 Cargo 的项目设置开始：

```rs
$ cargo new --bin rayon-parallel
```

在这种情况下，我们将使用 Rayon 比较常规迭代器和并行迭代器的性能。为此，我们需要使用 `rustc-test` 包。以下是 Cargo 设置的看起来：

```rs
[package]
name = "rayon-parallel"
version = "0.1.0"
authors = ["Foo <foo@bar.com>"]

[dependencies]
rayon = "0.9.0"
rustc-test = "0.3.0"
```

下面是代码，如以下代码片段所示。我们有两个执行完全相同操作的函数。它们都接收一个整数向量，然后遍历该向量并过滤出偶数整数。最后，它们返回奇数整数的平方：

```rs
// appendix/rayon-parallel/src/main.rs

#![feature(test)]

extern crate rayon;
extern crate test;

use rayon::prelude::*;

// This function uses a parallel iterator to
// iterate over the input vector, filtering
// elements that are even and then squares the remaining
fn filter_parallel(src: Vec<u64>) -> Vec<u64> {
    src.par_iter().filter(|x| *x % 2 != 0)
    .map(|x| x * x)
    .collect()
}

// This function does exactly the same operation
// but uses a regular, sequential iterator
fn filter_sequential(src: Vec<u64>) -> Vec<u64> {
    src.iter().filter(|x| *x % 2 != 0)
    .map(|x| x * x)
    .collect()
}

fn main() {
    let nums_one = (1..10).collect();
    println!("{:?}", filter_sequential(nums_one));

    let nums_two = (1..10).collect();
    println!("{:?}", filter_parallel(nums_two));
}

#[cfg(test)]
mod tests {
    use super::*;
    use test::Bencher;

    #[bench]
    fn bench_filter_sequential(b: &mut Bencher) {
        b.iter(|| filter_sequential((1..1000).collect::<Vec<u64>>()));
    }

    #[bench]
    fn bench_filter_parallel(b: &mut Bencher) {
        b.iter(|| filter_parallel((1..1000).collect::<Vec<u64>>()));
    }
}
```

我们首先从 Rayon 导入所有内容。在 `filter_parallel` 中，我们使用 `par_iter` 来获取一个并行迭代器。`filter_sequential` 与之相同，唯一的区别在于它使用 `iter` 函数来获取一个常规迭代器。在我们的主函数中，我们创建两个序列并将它们传递给我们的函数，同时打印输出。以下是我们应该看到的内容：

```rs
$ cargo run
   Compiling rayon-parallel v0.1.0 (file:///Users/Abhishek/Desktop/rust-book/src/appendix/rayon-parallel)
    Finished dev [unoptimized + debuginfo] target(s) in 1.65 secs
     Running `target/debug/rayon-parallel`
[1, 9, 25, 49, 81]
[1, 9, 25, 49, 81]
```

并不令人意外，两者返回相同的结果。这个例子中最重要的一部分是基准测试。为了使其工作，我们需要使用 `#![feature(test)]` 激活测试功能并声明一个新的测试模块。在那里，我们从顶层模块导入所有内容，在这个案例中，顶层模块是主文件。我们还导入了 `test::Bencher`，它将被用于运行基准测试。基准测试由 `#[bench]` 属性定义，这些属性应用于接受一个对象作为 `Bencher` 类型可变引用的函数。我们将需要基准测试的函数传递给基准测试器，基准测试器负责运行这些函数并打印结果。

可以使用 Cargo 运行基准测试：

```rs
$ cargo bench
    Finished release [optimized] target(s) in 0.0 secs
     Running target/release/deps/rayon_parallel-333850e4b1422ead

running 2 tests
test tests::bench_filter_parallel ... bench: 92,630 ns/iter (+/- 4,942)
test tests::bench_filter_sequential ... bench: 1,587 ns/iter (+/- 269)

test result: ok. 0 passed; 0 failed; 0 ignored; 2 measured; 0 filtered out
```

这个输出显示了两个函数以及执行每个迭代所需的时间。括号中的数字表示给定测量的置信区间。虽然并行版本的置信区间比非并行版本大，但它确实执行了比非并行版本多 58 倍的迭代。因此，并行版本要快得多。

# 使用 Pest 进行解析

我们在第四章*数据序列化、反序列化和解析*中研究了不同的解析技术。我们探讨了使用 Nom 的解析器组合器，从较小的部分构建一个大型解析器。解决解析文本数据相同问题的另一种完全不同的方法是使用**解析表达式语法（PEG）**。PEG 是一种形式语法，它定义了解析器应该如何行为。因此，它包括一组有限的规则，从基本标记到更复杂的结构。可以接受此类语法的库是 Pest。让我们看看使用 Pest 重写我们的 HTTP 解析示例第四章*数据序列化、反序列化和解析*的例子。从 Cargo 项目设置开始：

```rs
$ cargo new --bin pest-example
```

和往常一样，我们需要声明对 Pest 组件的依赖，如下所示：

```rs
[package]
name = "pest-example"
version = "0.1.0"
authors = ["Foo <foo@bar.com>"]

[dependencies]
pest = "¹.0"
pest_derive = "¹.0"
```

下一步是定义我们的语法，它是一系列解析规则的线性集合。像之前一样，我们对解析`HTTP GET`或`POST`请求感兴趣。以下是语法的样子：

```rs
// src/appendix/pest-example/src/grammar.pest

newline = { "\n" }
carriage_return = { "\r" }
space = { " " }
get = { "GET" }
post = { "POST" }
sep = { "/" }
version = { "HTTP/1.1" }
chars = { 'a'..'z' | 'A'..'Z' }
request = { get | post }

ident_list = _{ request ~ space ~ sep ~ chars+ ~ sep ~ space ~ version ~ carriage_return ~ newline }
```

第一步是定义要逐字匹配的文本规则。这些对应于 Nom 中的叶解析器。我们定义了换行符、回车符、空格、两个请求字符串、分隔符以及 HTTP 版本固定字符串的文本。我们还定义了`request`作为两个请求文本的逻辑或。字符列表是所有小写字母和所有大写字母的逻辑或。到此为止，我们已经有了定义最终规则所需的一切。这由`ident_list`给出，由请求、一个空格、然后是分隔符组成；然后我们指出我们的解析器应使用`*`接受一个或多个字符。下一个有效输入又是分隔符，后面跟一个空格、版本字符串、回车符，最后是换行符。请注意，连续输入由`~`字符分隔。前面的下划线`_`表示这是一个静默规则，并且应该仅在顶层使用，正如我们很快就会看到的。

主文件看起来是这样的：

```rs
// src/appendix/pest-example/src/main.rs

extern crate pest;
#[macro_use]
extern crate pest_derive;

use pest::Parser;

#[derive(Parser)]
#[grammar = "grammar.pest"]
// Unit struct that will be used as the parser
struct RequestParser;

fn main() {
    let get = RequestParser::parse(Rule::ident_list, "GET /foobar/ HTTP/1.1\r\n")
        .unwrap_or_else(|e| panic!("{}", e));
    for pair in get {
        println!("Rule: {:?}", pair.as_rule());
        println!("Span: {:?}", pair.clone().into_span());
        println!("Text: {}", pair.clone().into_span().as_str());
    }

    let _ = RequestParser::parse(Rule::ident_list, "WRONG /foobar/
    HTTP/1.1\r\n")
        .unwrap_or_else(|e| panic!("{}", e));
}
```

代码很简单；库提供了一个基本特性，称为`Parser`。这可以通过使用名为`grammar`的属性为单元结构自定义派生，以基于语法文件生成一个功能解析器。值得注意的是，这个库非常高效地使用自定义派生和自定义属性来提供更好的用户体验。在我们的例子中，单元结构被称为`RequestParser`，它实现了`parse`方法。在我们的主函数中，我们调用这个方法，传入解析应该开始的规则（在我们的情况下，那恰好是最终的顶级规则，称为`ident_list`）以及要解析的字符串。由于解析失败后继续没有太多意义，错误通过终止来处理。

在设置好这个结构后，我们尝试解析两个字符串。第一个是一个正常的 HTTP 请求。`parse`方法返回一个解析令牌流上的迭代器。我们遍历它们并打印出与令牌匹配的规则名称，输入中包含令牌的范围，以及该令牌中的文本。稍后，我们尝试解析一个没有有效 HTTP 请求的字符串。以下是输出：

```rs
$ cargo run
   Compiling pest-example v0.1.0 (file:///Users/Abhishek/Desktop/rust-book/src/chapter4/pest-example)
    Finished dev [unoptimized + debuginfo] target(s) in 1.0 secs
     Running `target/debug/pest-example`
Rule: request
Span: Span { start: 0, end: 3 }
Text: GET
Rule: space
Span: Span { start: 3, end: 4 }
Text:
Rule: sep
Span: Span { start: 4, end: 5 }
Text: /
Rule: chars
Span: Span { start: 5, end: 6 }
Text: f
Rule: chars
Span: Span { start: 6, end: 7 }
Text: o
Rule: chars
Span: Span { start: 7, end: 8 }
Text: o
Rule: chars
Span: Span { start: 8, end: 9 }
Text: b
Rule: chars
Span: Span { start: 9, end: 10 }
Text: a
Rule: chars
Span: Span { start: 10, end: 11 }
Text: r
Rule: sep
Span: Span { start: 11, end: 12 }
Text: /
Rule: space
Span: Span { start: 12, end: 13 }
Text:
Rule: version
Span: Span { start: 13, end: 21 }
Text: HTTP/1.1
Rule: carriage_return
Span: Span { start: 21, end: 22 }
Text:
Rule: newline
Span: Span { start: 22, end: 23 }
Text:

thread 'main' panicked at ' --> 1:1
  |
1 | WRONG /foobar/ HTTP/1.1
  | ^---
  |
  = expected request', src/main.rs:21:29
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

首先要注意的是解析错误的 HTTP 请求失败了。错误信息很友好且清晰，解释了解析失败的确切位置。正确的请求解析成功并打印出了所有必要的令牌和详细信息，以便进一步处理。

# 杂项实用工具

在 C 和 C++中，一个常见的流程是定义一组位作为标志。它们通常定义为 2 的幂，因此第一个标志的十进制值为 1，第二个标志的值为 2，依此类推。这有助于执行这些标志的逻辑组合。Rust 生态系统有一个 crate 来简化相同的流程。让我们看看使用 bitflags crate 处理标志的示例。让我们从使用 Cargo 初始化一个空项目开始：

```rs
$ cargo new --bin bitflags-example
```

我们将设置项目清单以添加`bitflags`作为依赖项：

```rs
$ cat Cargo.toml
[package]
name = "bitflags-example"
version = "0.1.0"
authors = ["Foo <foo@bar.com>"]

[dependencies]
bitflags = "1.0"
```

当所有这些都准备好了，我们的主文件将看起来像这样：

```rs
// appendix/bitflags-example/src/main.rs

#[macro_use]
extern crate bitflags;

// This macro defines a struct that holds our
// flags. This also defines a number of convinience
// methods on the struct.
bitflags! {
    struct Flags: u32 {
        const X = 0b00000001;
        const Y = 0b00000010;
    }
}

// We define a custom trait to print a
// given bitflag in decimal
pub trait Format {
    fn decimal(&self);
}

// We implement our trait for the Flags struct
// which is defined by the bitflags! macro
impl Format for Flags {
    fn decimal(&self) {
        println!("Decimal: {}", self.bits());
    }
}

// Main driver function
fn main() {
    // A logical OR of two given bitflags
    let flags = Flags::X | Flags::Y;

    // Prints the decimal representation of
    // the logical OR
    flags.decimal();

    // Same as before
    (Flags::X | Flags::Y).decimal();

    // Prints one individual flag in decimal
    (Flags::Y).decimal();

    // Examples of the convenience methods mentioned
    // earlier. The all method gets the current state
    // as a human readable string. The contain method
    // returns a bool indicating if the given bitflag
    // has the other flag.
    println!("Current state: {:?}", Flags::all());
    println!("Contains X? {:?}", flags.contains(Flags::X));
}
```

我们导入我们的依赖项，然后使用`bitflags!`宏定义一系列标志，如前所述，我们将它们的值设置为 2 的幂。我们还展示了使用特性系统附加到`bitflags`上的额外属性。为此，我们有一个自定义特性`Format`，它将给定输入打印为十进制。转换是通过使用返回给定输入中所有位的`bits()`方法来实现的。下一步是实现我们的特性为`Flags`结构。

在完成这些后，我们继续到`main`函数；在那里，我们构造两个给定标志的逻辑或。我们使用`decimal`方法打印出位标志的表示，并确保它们相等。最后，我们使用`all`函数显示标志的人类可读形式。在这里，`contains`函数返回`true`，因为标志`X`确实在`X`和`Y`的逻辑或中。

运行此代码后我们应该看到以下内容：

```rs
$ cargo run
   Compiling bitflags-example v0.1.0 (file:///Users/Abhishek/Desktop/rust-book/src/appendix/bitflags-example)
    Finished dev [unoptimized + debuginfo] target(s) in 0.48 secs
     Running `target/debug/bitflags-example`
Decimal: 3
Decimal: 3
Decimal: 2
Current state: X | Y
Contains X? true
```

单个标志的值始终应该是整数类型。

网络编程中另一个有用的实用工具是`url` crate。这个 crate 提供了解析 URL 部分的功能，从网页链接到相对地址。让我们看一个非常简单的例子，从项目设置开始：

```rs
$ cargo new --bin url-example
```

货物清单应该看起来像这样：

```rs
$ cat Cargo.toml
[package]
name = "url-example"
version = "0.1.0"
authors = ["Foo <foo@bar.com>"]

[dependencies]
url = "1.6.0"
```

让我们看看主文件。在这个相对简短的示例中，我们正在解析 GitLab URL 以提取一些重要的信息：

```rs
// appendix/url-example/src/main.rs

extern crate url;

use url::Url;

fn main() {
    // We are parsing a gitlab URL. This one happens to be using
    // git and https, a given username/password and a fragment
    // pointing to one line in the source
    let url = Url::parse("git+https://foo:bar@gitlab.com/gitlab-org/gitlab-ce/blob/master/config/routes/development.rb#L8").unwrap();

    // Prints the scheme
    println!("Scheme: {}", url.scheme());

    // Prints the username
    println!("Username: {}", url.username());

    // Prints the password
    println!("Password: {}", url.password().unwrap());

    // Prints the fragment (everything after the #)
    println!("Fragment: {}", url.fragment().unwrap());

    // Prints the host
    println!("Host: {:?}", url.host().unwrap());
}
```

这个示例 URL 包含一个片段，指向文件中的一行数字。方案设置为 git，并且为基于 HTTP 的认证设置了用户名和密码。URL crate 提供了一个名为`parse`的方法，它接受一个字符串并返回一个包含所有所需信息的结构体。我们随后可以调用该变量的单个方法来打印出相关信息。

下面是这段代码的输出结果，符合我们的预期：

```rs
$ cargo run
   Compiling url-example v0.1.0 (file:///Users/Abhishek/Desktop/rust-book/src/appendix/url-example)
    Finished dev [unoptimized + debuginfo] target(s) in 0.58 secs
     Running `target/debug/url-example`
Scheme: git+https
Username: foo
Password: bar
Fragment: L8
Host: Domain("gitlab.com")
```

# 摘要

这最后一章包含了多个主题，我们认为它们不够主流，不适合放在其他章节中。但我们应该记住，在一个像 Rust 这样的庞大生态系统中，事物发展非常迅速。所以，今天可能不是主流的一些想法，明天可能会被社区采纳。

总体来说，Rust 是一种非常棒的语言，具有巨大的潜力。我们真诚地希望这本书能帮助读者了解如何利用其力量进行网络编程。
