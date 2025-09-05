

# 第七章：协程和 async/await

既然你已经对 Rust 的异步模型有了简要的了解，是时候看看它如何与我们在这本书中迄今为止所涵盖的其他内容相契合了。

Rust 的未来是一个基于无栈协程的异步模型示例，在本章中，我们将探讨这究竟意味着什么，以及它与有栈协程（纤程/绿色线程）有何不同。

我们将以基于简化模型的未来和`async/await`的示例为中心，看看我们如何使用它来创建可挂起和可恢复的任务，就像我们创建自己的纤程时做的那样。

好消息是，这比实现我们自己的纤程/绿色线程容易得多，因为我们可以留在 Rust 中，这更安全。不利的一面是，它稍微抽象一些，并且与编程语言理论以及计算机科学紧密相关。

在本章中，我们将涵盖以下内容：

+   无栈协程简介

+   手写协程的例子

+   `async/await`

# 技术要求

本章中的所有示例都将跨平台，所以你唯一需要的是安装 Rust 以及下载属于本书的本地存储库。本章中的所有代码都将位于`ch07`文件夹中。

在这个例子中，我们也将使用`delayserver`，所以你需要打开一个终端，进入存储库根目录下的`delayserver`文件夹，并运行`cargo run`，以便它为后续的示例准备好并可用。

如果因为某种原因你需要更改`delayserver`监听的端口号，请记住更改代码中的端口号。

# 无栈协程简介

因此，我们终于到达了介绍本书中建模异步操作最后一种方法的点。你可能还记得，我们在*第二章*中给出了关于有栈和无栈协程的高级概述。在*第五章*中，我们在编写自己的纤程/绿色线程时实现了一个有栈协程的例子，所以现在我们该更深入地看看无栈协程是如何实现和使用的。

如果你还记得，在*第一章*中我们提到，如果我们想让任务并发运行（同时进行），但不一定并行，我们需要能够**暂停和恢复**任务。

在其最简单的形式中，协程只是一个可以通过将控制权交还给其调用者、另一个协程或调度器来暂停和恢复的任务。

许多语言都将有一个协程实现，它还提供了一个运行时，为你处理调度和非阻塞 I/O，但区分协程是什么以及创建异步系统所涉及的其余机制是有帮助的。

这在 Rust 中尤其如此，因为 Rust 没有提供运行时，只提供了创建具有语言原生支持的协程所需的基础设施。Rust 确保所有使用 Rust 编程的人使用相同的抽象来处理可以暂停和恢复的任务，但它将所有其他使异步系统启动和运行的具体细节留给程序员。

无栈协程或仅仅是协程？

最常见的情况是，你会看到*无栈协程*简单地被称为*协程*。为了保持一致性（你还记得我不喜欢根据上下文引入具有不同含义的术语），我一直将协程称为*无栈*或*有栈*，但今后，我只需简单地称无栈协程为**协程**。这也是你在其他来源阅读关于它们时可以期待的内容。

纤维/绿色线程以与操作系统非常相似的方式表示这种可恢复的任务。一个任务有一个栈，其中存储/恢复其当前执行状态，使其能够暂停和恢复任务。

状态机在其最简单的形式中是一个具有预定义状态集的数据结构。在协程的情况下，每个状态代表一个可能的暂停/恢复点。我们不将暂停/恢复任务的所需状态存储在单独的栈中，而是将其保存在数据结构中。

这有一些优点，我之前已经介绍过，但最突出的是它们非常高效和灵活。缺点是，你永远不会想手动编写这些状态机（你将在本章中看到原因），因此你需要来自编译器或其他机制的支持来重写你的代码，使其成为状态机而不是正常函数调用。

结果是，你得到的是一个看起来非常简单的东西。它看起来像是一个函数/子程序，你可以很容易地将其映射到可以使用简单的汇编`call`指令运行的东西，但实际上你得到的是一个相当复杂且与预期不同的东西，它看起来也不像你期望的那样。

生成器与协程的比较

生成器也是状态机，正是我们将在本章中介绍的那种。它们通常在一种语言中实现，以创建向调用函数产生值的州机。

从理论上讲，你可以根据它们产生的结果来区分协程和生成器。生成器通常仅限于向调用函数产生结果。协程可以产生结果给另一个协程、调度器或简单地给调用者，在这种情况下，它们就像生成器一样。

在我看来，在它们之间做出区分实际上没有意义。它们代表了创建可以暂停和恢复执行的任务的相同底层机制，因此在这本书中，我们将它们视为基本上是同一件事。

现在我们已经用文字描述了什么是协程，我们可以开始看看它们在代码中的样子。

# 手写协程的例子

我们接下来要使用的例子是 Rust 异步模型的简化版本。我们将创建和实现以下内容：

+   我们自己的简化版 `Future` trait

+   一个只能执行 GET 请求的简单 HTTP 客户端

+   我们可以暂停和恢复的任务，实现为一个状态机

+   我们自己的简化版 `async/await` 语法称为 `coroutine/wait`

+   一个自制的预处理器，将我们的 `coroutine/wait` 函数转换成状态机，就像 `async/await` 被转换一样

为了真正揭开协程、未来和 `async/await` 的神秘面纱，我们不得不做一些妥协。如果我们不这样做，我们最终会重新实现今天 Rust 中所有的 `async/await` 和未来，这对于仅仅理解底层技术和概念来说太多了。

因此，我们的例子将做以下事情：

+   避免错误处理。如果发生任何失败，我们将引发恐慌。

+   要具体，而不是泛化。创建泛型解决方案会引入很多复杂性，并使得底层概念更难推理，因为我们随后不得不创建额外的抽象层。尽管如此，我们的解决方案在需要的地方将有一些泛型方面。

+   在它能做什么方面有限制。你当然可以自由地扩展、更改和玩转所有这些例子（我鼓励你这样做），但在例子中，我们只涵盖我们需要的内容，而不是更多。

+   避免宏。

因此，在解决完这些问题后，让我们开始我们的例子。

你需要做的第一件事是创建一个新的文件夹。这个第一个例子可以在仓库中的 `ch07/a-coroutine` 目录下找到，所以我建议你也将其命名为 `a-coroutine`。

然后，进入文件夹并运行 `cargo init` 来初始化一个新的 crate。

现在我们有一个新项目正在运行，我们可以创建我们需要的模块和文件夹：

首先，在 `main.rs` 中，声明两个模块如下：

ch07/a-coroutine/src/main.rs

```rs
mod http;
mod future;
```

接下来，在 `src` 文件夹中创建两个新文件：

+   `future.rs`，将包含我们的未来相关代码

+   `http.rs`，它将包含我们 HTTP 客户端相关的代码

我们最后需要做的一件事是添加对 `mio` 的依赖。我们将使用 `mio` 中的 `TcpStream`，因为我们将在接下来的章节中构建这个例子，并使用 `mio` 作为我们的非阻塞 I/O 库，因为我们已经熟悉它：

ch07/a-coroutine/Cargo.toml

```rs
[dependencies]
mio = { version = "0.8", features = ["net", "os-poll"] }
```

让我们从 `future.rs` 开始，并首先实现我们的未来相关代码。

## Futures 模块

在 `futures.rs` 中，我们首先将定义一个 `Future` trait。它看起来如下：

ch07/a-coroutine/src/future.rs

```rs
pub trait Future {
    type Output;
    fn poll(&mut self) -> PollState<Self::Output>;
}
```

如果我们将它与 Rust 标准库中的 `Future` trait 进行对比，你会发现它们非常相似，除了我们不取 `cx: &mut Context<'_>` 作为参数，并且我们返回一个具有不同名称的 `enum`，只是为了区分它们，以免混淆：

```rs
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

我们接下来要做的是定义一个 `PollState<T>` `enum`：

ch07/a-coroutine/src/future.rs

```rs
pub enum PollState<T> {
    Ready(T),
    NotReady,
}
```

再次，如果我们将其与 Rust 标准库中的 `Poll` 枚举进行比较，我们会发现它们实际上是一样的：

```rs
pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

现在，为了使我们的示例的第一个迭代能够运行，我们只需要这些。让我们继续到下一个文件：`http.rs`。

## HTTP 模块

在这个模块中，我们将实现一个非常简单的 HTTP 客户端。这个客户端只能向我们的 `delayserver` 发送 GET 请求，因为我们只是用它来表示典型的 I/O 操作，并不关心能否做更多我们不需要的事情。

我们首先将一些类型和特质从标准库以及我们的 `Futures` 模块导入：

ch07/a-coroutine/src/http.rs

```rs
use crate::future::{Future, PollState};
use std::io::{ErrorKind, Read, Write};
```

接下来，我们创建了一个小的辅助函数来编写我们的 HTTP 请求。我们之前在这本书中已经使用过这段代码，所以在这里我不会再次解释它：

ch07/a-coroutine/src/http.rs

```rs
fn get_req(path: &str) -> String {
    format!(
        "GET {path} HTTP/1.1\r\n\
             Host: localhost\r\n\
             Connection: close\r\n\
             \r\n"
    )
}
```

因此，现在我们可以开始编写我们的 HTTP 客户端了。实现非常简短且简单：

```rs
pub struct Http;
impl Http {
    pub fn get(path: &str) -> impl Future<Output = String> {
        HttpGetFuture::new(path)
    }
}
```

我们在这里实际上并不需要一个结构体，但我们添加了一个，因为我们可能在以后某个时刻想要添加一些状态。这也是将属于 HTTP 客户端的功能分组在一起的好方法。

我们的 HTTP 客户端只有一个函数，即 `get`，它最终会向我们的 `delayserver` 发送一个带有指定路径的 GET 请求（记住，在这个示例 URL 中，路径是所有加粗的内容：`http://127.0.0.1:8080`**/1000/HelloWorld**），

在函数体中，你首先会注意到这里并没有发生太多事情。我们只返回 `HttpGetFuture`，就这么多。

在函数签名中，你可以看到它返回一个实现 `Future` 特质的对象，当它解析时输出一个 `String`。从这个函数返回的字符串将是来自服务器的响应。

现在，我们本可以直接在 `Http` 结构体上实现 future 特质，但我认为更好的设计是允许一个 `Http` 实例提供多个 `Futures`，而不是让 `Http` 本身实现 `Future`。

让我们更仔细地看看 `HttpGetFuture`，因为那里发生的事情更多。

只是为了指出，以免将来有疑问，`HttpGetFuture` 是一个**叶子未来**的例子，并且它将是我们在本例中使用的唯一叶子未来。

让我们在文件中添加结构体声明：

ch07/a-coroutine/src/http.rs

```rs
struct HttpGetFuture {
    stream: Option<mio::net::TcpStream>,
    buffer: Vec<u8>,
    path: String,
}
```

这个数据结构将为我们保存一些数据：

+   `stream`：这保存了一个 `Option<mio::net::TcpStream>`。这将是 `Option`，因为我们不会在创建此结构的同时连接到流。

+   `buffer`：我们将从 `TcpStream` 读取数据并将其全部放入这个缓冲区，直到我们读取了服务器返回的所有数据。

+   `path`：这个简单地存储了我们的 GET 请求的路径，以便我们以后可以使用它。

我们接下来要查看的是 `HttpGetFuture` 的 `impl` 块：

ch07/a-coroutine/src/http.rs

```rs
impl HttpGetFuture {
    fn new(path: &'static str) -> Self {
        Self {
            stream: None,
            buffer: vec![],
            Path: path.to_string(),
        }
    }
    fn write_request(&mut self) {
        let stream = std::net::TcpStream::connect("127.0.0.1:8080").unwrap();
        stream.set_nonblocking(true).unwrap();
        let mut stream = mio::net::TcpStream::from_std(stream);
        stream.write_all(get_req(&self.path).as_bytes()).unwrap();
        self.stream = Some(stream);
    }
}
```

`impl` 块定义了两个函数。第一个是 `new`，它只是设置初始状态。

下一个函数是`write_requst`，它将 GET 请求发送到服务器。您在*第四章*的示例中已经看到过这段代码，所以这应该看起来很熟悉。

注意

当*创建* `HttpGetFuture`时，我们实际上并没有做任何与 GET 请求相关的事情，这意味着对`Http::get`的调用会立即返回，只带有一个简单的数据结构。

与早期示例相比，我们传递了`localhost`的*IP 地址*而不是 DNS 名称。我们采取与之前相同的捷径，让`connect`是阻塞的，而其他一切都是非阻塞的。

下一步是向服务器发送 GET 请求。这将是非阻塞的，我们不需要等待它完成，因为我们无论如何都会等待响应。

文件的最后一部分是最重要的——我们定义的`Future`特质的实现：

ch07/a-coroutine/src/http.rs

```rs
impl Future for HttpGetFuture {
    type Output = String;
    fn poll(&mut self) -> PollState<Self::Output> {
        if self.stream.is_none() {
            println!("FIRST POLL - START OPERATION");
            self.write_request();
            return PollState::NotReady;
        }
        let mut buff = vec![0u8; 4096];
        loop {
            match self.stream.as_mut().unwrap().read(&mut buff) {
                Ok(0) => {
                    let s = String::from_utf8_lossy(&self.buffer);
                    break PollState::Ready(s.to_string());
                }
                Ok(n) => {
                    self.buffer.extend(&buff[0..n]);
                    continue;
                }
                Err(e) if e.kind() == ErrorKind::WouldBlock => {
                    break PollState::NotReady;
                }
                Err(e) if e.kind() == ErrorKind::Interrupted => {
                    continue;
                }
                Err(e) => panic!("{e:?}"),
            }
        }
    }
}
```

好吧，所以这里就是一切发生的地方。我们首先做的事情是将关联类型`Output`设置为`String`。

我们接下来要做的就是检查这是否是第一次调用`poll`。我们通过检查`self.stream`是否为`None`来完成这个操作。

如果这是我们第一次调用`poll`，我们会打印一条消息（只是为了看到第一次这个 future 被轮询的情况），然后我们将 GET 请求写入服务器。

在第一次轮询时，我们返回`PollState::NotReady`，因此`HttpGetFuture`至少还需要被轮询一次才能返回任何结果。

函数的下一部分尝试从我们的`TcpStream`读取数据。

我们之前已经讨论过这个问题，所以我会简要说明，但基本上有五件事情可能发生：

1.  调用成功返回，读取了`0`个字节。我们已经从流中读取了所有数据，并收到了整个 GET 响应。我们在返回之前，从读取的数据中创建一个`String`并将其包装在`PollState::Ready`中。

1.  调用成功返回，读取了`n > 0`个字节。如果是这种情况，我们将数据读取到我们的缓冲区中，将数据追加到`self.buffer`中，并立即尝试从流中读取更多数据。

1.  我们得到一个`WouldBlock`类型的错误。如果是这种情况，我们知道由于我们将流设置为非阻塞，数据尚未准备好或者有更多数据但我们尚未收到。在这种情况下，我们返回`PollState::NotReady`以表明需要更多轮询调用来完成操作。

1.  我们得到一个`Interrupted`类型的错误。这是一个特殊情况，因为读取可以被信号中断。如果发生这种情况，处理错误的通常方式是简单地再次尝试读取。

1.  我们得到一个我们无法处理的错误，并且由于我们的示例没有进行错误处理，我们简单地`panic!`

有一个微妙的地方我想指出。我们可以将其视为一个非常简单的具有三个状态的状态机：

+   未开始，由`self.stream`为`None`表示

+   等待中，由`self.stream`为`Some`且对`stream.read`的读取返回`WouldBlock`表示

+   已解决，通过`self.stream`为`Some`以及调用`stream.read`返回`0`字节来指示

如你所见，这个模型很好地映射到了操作系统在尝试读取我们的`TcpStream`时报告的状态。

大多数像这样的叶子未来将会非常简单，尽管我们没有在这里明确状态，但它仍然适合我们基于协程构建的状态机模型。

## 所有未来都必须是懒加载的吗？

懒加载的未来是在它第一次被轮询之前不执行任何工作。

如果你阅读关于 Rust 中的未来的内容，这会经常出现，并且由于我们的`Future`特例正是基于这个模型，同样的问题也会在这里出现。对这个问题的简单回答是：不！

没有什么强制叶子未来，比如我们在这里写的，必须是懒加载的。如果我们想在调用 `Http::get` 函数时发送 HTTP 请求，我们可以这样做。如果你这么想，如果我们只是这样做，这可能会引起一个可能很大的变化，从而影响我们在程序中实现并发的方式。

现在的工作方式是，必须有人至少调用一次 `poll` 来实际发送请求。结果是，调用这个未来的 `poll` 的人将不得不对许多未来调用 `poll`，如果他们想让它们并发运行的话。

如果我们在创建未来时立即启动操作，你可以创建许多未来，即使你逐个轮询它们以完成，它们也会并发运行。如果你在当前设计中逐个轮询它们以完成，未来将不会并发地前进。请稍作思考。

类似 JavaScript 这样的语言在协程创建时就开始执行操作，因此没有“一种方式”来做这件事。每次遇到协程实现时，你应该找出它们是懒加载的还是急加载的，因为这将影响你如何使用它们编程。

尽管在这种情况下我们可以使我们的未来变得急加载，但我们实际上不应该这样做。由于 Rust 中的程序员期望未来是懒加载的，他们可能会依赖于在你对它们调用 `poll` 之前不发生任何事情，如果你写的未来行为不同，可能会有意外的副作用。

现在，当你读到 Rust 的未来总是懒加载的，这是一个我经常看到的说法，它指的是使用 `async/await` 生成的编译器生成的状态机。正如我们稍后将会看到的，当你的异步函数被编译器重写时，它们是以一种方式构建的，这样你在一个 `async` 函数体中写的任何内容都不会在第一次调用 `Future::poll` 之前执行。

好的，所以我们已经涵盖了`Future`特性和我们命名为`HttpGetFuture`的叶子未来。下一步是创建一个可以在预定义点停止和恢复的任务。

## 创建协程

我们将从零开始构建我们的知识和理解。我们首先要做的是创建一个可以通过将其建模为手动状态机来停止和恢复的任务。

一旦我们完成，我们将看看这种建模暂停任务的方式如何使我们能够编写类似于`async/await`的语法，并依赖于代码转换来创建这些状态机，而不是手动编写它们。

我们将创建一个简单的程序，它将执行以下操作：

1.  当我们的暂停任务开始时打印一条消息。

1.  向我们的`delayserver`发起 GET 请求。

1.  等待 GET 请求。

1.  打印来自服务器的响应。

1.  向我们的`delayserver`发起第二次 GET 请求。

1.  等待来自服务器的第二次响应。

1.  打印来自服务器的响应。

1.  退出程序。

此外，我们将通过在自定义协程上多次调用`Future::poll`来执行我们的程序，直到运行完成。目前还没有运行时、反应器或执行器，因为我们将这些内容留到下一章介绍。

如果我们将程序编写为一个`async`函数，它将如下所示：

```rs
async fn async_main() {
    println!("Program starting")
    let txt = Http::get("/1000/HelloWorld").await;
    println!("{txt}");
    let txt2 = Http::("500/HelloWorld2").await;
    println!("{txt2}");
}
```

在`main.rs`中，首先进行必要的导入和模块声明：

ch07/a-coroutine/src/main.rs

```rs
use std::time::Instant;
mod future;
mod http;
use crate::http::Http;
use future::{Future, PollState};
```

我们接下来要写的是我们的可停止/可恢复任务，称为`Coroutine`：

ch07/a-coroutine/src/main.rs

```rs
struct Coroutine {
    state: State,
}
```

一旦完成，我们将编写这个任务可能处于的不同状态：

ch07/a-coroutine/src/main.rs

```rs
enum State {
    Start,
    Wait1(Box<dyn Future<Output = String>>),
    Wait2(Box<dyn Future<Output = String>>),
    Resolved,
}
```

这个特定的协程可以处于四种状态：

+   `Coroutine`已创建，但尚未被轮询。

+   `Http::get`，我们得到一个存储在`State` `enum`中的`HttpGetFuture`返回值。在此点，我们将控制权交回调用函数，以便它可以在需要时执行其他操作。我们选择使其对所有输出`String`的`Future`函数是通用的，但由于我们目前只有一种类型的未来，我们也可以简单地使其仅持有`HttpGetFuture`，它将以相同的方式工作。

+   `Http::get`是我们将控制权交回调用函数的第二个地方。

+   **已解决**：未来已解决，没有更多的工作要做。

注意

我们本可以直接将`Coroutine`定义为`enum`，因为它只持有表示其状态的`enum`。但我们将设置这个示例，以便我们可以在本书的后面部分添加一些状态到`Coroutine`。

接下来是`Coroutine`的实现：

ch07/a-coroutine/src/main.rs

```rs
impl Coroutine {
    fn new() -> Self {
        Self {
            state: State::Start,
        }
    }
}
```

到目前为止，这相当简单。当创建一个新的`Coroutine`时，我们只需将其设置为`State::Start`即可。

现在我们来到了实际工作在`Coroutine`的`Future`实现部分。我将带您浏览代码：

ch07/a-coroutine/src/main.rs

```rs
impl Future for Coroutine {
    type Output = ();
    fn poll(&mut self) -> PollState<Self::Output> {
        loop {
            match self.state {
                State::Start => {
                    println!("Program starting");
                    let fut = Box::new(Http::get("/600/HelloWorld1"));
                    self.state = State::Wait1(fut);
                }
                State::Wait1(ref mut fut) => match fut.poll() {
                    PollState::Ready(txt) => {
                        println!("{txt}");
                        let fut2 = Box::new(Http::get("/400/HelloWorld2"));
                        self.state = State::Wait2(fut2);
                    }
                    PollState::NotReady => break PollState::NotReady,
                },
                State::Wait2(ref mut fut2) => match fut2.poll() {
                    PollState::Ready(txt2) => {
                        println!("{txt2}");
                        self.state = State::Resolved;
                        break PollState::Ready(());
                    }
                    PollState::NotReady => break PollState::NotReady,
                },
                State::Resolved => panic!("Polled a resolved future"),
            }
        }
    }
}
```

让我们从顶部开始：

1.  我们首先将`Output`类型设置为`()`。由于我们不会返回任何内容，这仅仅使我们的示例更简单。

1.  接下来是`poll`方法的实现。首先您会注意到我们写了一个匹配`self.state`的`loop`实例。我们这样做是为了推动状态机向前发展，直到我们达到一个点，没有从我们的子未来中获得`PollState::NotReady`我们就无法进一步进展。

1.  如果状态是`State::Start`，我们知道这是第一次被轮询，所以我们运行我们需要运行的任何指令，直到我们到达需要解决的新未来的点。

1.  当我们调用`Http::get`时，我们返回一个需要在我们进一步进展之前完成轮询的未来。

1.  在这一点上，我们将状态更改为`State::Wait1`，并存储我们想要解析的未来，以便在下一个状态中访问它。

1.  我们的状态机现在已从`Start`变为`Wait1`。由于我们在`match`语句上循环，我们立即进入下一个状态，并在下一次迭代中到达`State::Wait1`的匹配分支。

1.  在`Wait1`中，我们首先对等待的`Future`实例调用`poll`。

1.  如果未来返回`PollState::NotReady`，我们只需将其冒泡到调用者处，通过跳出循环并返回`NotReady`。

1.  如果未来返回`PollState::Ready`并附带我们的数据，我们知道我们可以执行依赖于第一个未来数据的指令，并进入下一个状态。在我们的情况下，我们只打印出返回的数据，所以这只有一行代码。

1.  接下来，我们通过调用`Http::get`获得一个新的未来。我们将状态设置为`Wait2`，就像我们从`State::Start`到`State::Wait1`所做的那样。

1.  就像第一次我们得到一个需要在继续之前解决的未来一样，我们将其保存起来，以便在`State::Wait2`中访问它。

1.  由于我们处于循环中，接下来发生的事情是我们到达`Wait2`的匹配分支，在这里，我们重复与`State::Wait1`相同的步骤，但针对不同的未来。

1.  如果它返回带有我们的数据的`Ready`，我们就采取行动，并将我们的`Coroutine`的最终状态设置为`State::Resolved`。还有一个重要的变化：这次，我们希望通知调用者这个未来已经完成，所以我们跳出循环并返回`PollState::Ready`。

如果有人试图再次在我们的`Coroutine`上调用`poll`，我们将引发恐慌，因此调用者必须确保跟踪未来何时返回`PollState::Ready`，并确保永远不再调用它。在我们到达`main`函数之前做的最后一件事是在我们称为`async_main`的函数中创建一个新的`Coroutine`。这样，当我们在本章的最后部分讨论`async/await`时，我们可以将更改保持在最小：

ch07/a-coroutine/src/main.rs

```rs
fn async_main() -> impl Future<Output = ()> {
    Coroutine::new()
}
```

因此，在这个点上，我们已经完成了协程的编写，剩下要做的就是编写一些逻辑来通过`main`函数的不同阶段驱动状态机。

这里要注意的一点是，我们的主函数只是一个普通的主函数。我们主函数中的循环驱动异步操作完成：

ch07/a-coroutine/src/main.rs

```rs
fn main() {
    let mut future = async_main();
    loop {
        match future.poll() {
            PollState::NotReady => {
                println!("Schedule other tasks");
            },
            PollState::Ready(_) => break,
        }
        thread::sleep(Duration::from_millis(100));
    }
}
```

这个函数非常简单。我们首先获取`async_main`返回的未来，然后在一个循环中对其调用`poll`，直到它返回`PollState::Ready`。

每当我们收到 `PollState::NotReady` 的返回值时，控制权就交回到了我们这里。如果我们想的话，我们在这里可以做一些其他的工作，比如安排另一个任务，但在这个例子中，我们只是打印 `安排` `其他任务`。

我们还通过在每次调用时暂停 100 毫秒来限制循环的运行频率。这样我们不会因为打印输出而感到不知所措，并且我们可以假设每次我们在控制台看到 `"安排其他任务"` 打印出来之间大约有 100 毫秒的时间间隔。

如果我们运行这个例子，我们会得到以下输出：

```rs
Program starting
FIRST POLL - START OPERATION
Schedule other tasks
Schedule other tasks
Schedule other tasks
Schedule other tasks
Schedule other tasks
Schedule other tasks
HTTP/1.1 200 OK
content-length: 11
connection: close
content-type: text/plain; charset=utf-8
date: Tue, 24 Oct 2023 20:39:13 GMT
HelloWorld1
FIRST POLL - START OPERATION
Schedule other tasks
Schedule other tasks
Schedule other tasks
Schedule other tasks
HTTP/1.1 200 OK
content-length: 11
connection: close
content-type: text/plain; charset=utf-8
date: Tue, 24 Oct 2023 20:39:13 GMT
HelloWorld2
```

通过查看打印输出，你可以了解程序流程。

1.  首先，我们看到 `程序开始`，这是在协程开始时执行的。

1.  我们接着看到，我们立即跳转到 `第一次轮询 – 开始操作` 的消息，我们只在从我们的 HTTP 客户端返回的未来对象第一次轮询时打印这个消息。

1.  接下来，我们可以看到我们又回到了 `main` 函数中，在这个时候，如果我们有其他任务，理论上我们可以继续运行其他任务

1.  每 100 毫秒，我们检查任务是否完成，并得到同样的消息，告诉我们可以安排其他任务

1.  然后，大约 600 毫秒后，我们收到一个打印出来的响应

1.  我们重复这个过程，直到我们收到并打印出从服务器返回的第二响应

恭喜你，你现在创建了一个可以在不同点暂停和恢复的任务，允许它在进行中。

### 谁会想写这样的代码来完成一个简单的任务呢？

答案是没有一个人！

是的，这听起来有点夸张，但我敢猜测，与编写 55 行状态机相比，很少有程序员更喜欢编写 7 行正常的顺序代码来完成相同的事情。

如果我们回顾一下大多数用户空间并发操作的抽象目标，我们会发现这种做法只检查了我们想要达到的三个目标中的一个：

+   高效

+   表达性

+   易于使用且难以误用

我们的状态机将会是高效的，但这基本上就是全部了。

然而，你也可能注意到，这种疯狂中其实有一定的规律。这可能不会让你感到惊讶，但如果我们在每个函数的开始和每个我们想要将控制权交还给调用者的点使用几个关键字进行标记，并且由系统为我们生成状态机，那么我们编写的代码可能会简单得多。这正是 `async/await` 的基本理念。

让我们去看看在我们的例子中这会如何工作。

# async/await

之前的例子可以简单地用 `async/await` 关键字写成以下形式：

```rs
async fn async_main() {
    println!("Program starting")
    let txt = Http::get("/1000/HelloWorld").await;
    println!("{txt}");
    let txt2 = Http::("500/HelloWorld2").await;
    println!("{txt2}");
}
```

这只有七行代码，看起来非常熟悉，就像你在一个普通的子例程/函数中编写的代码一样。

结果表明，我们可以让编译器为我们编写这些状态机，而不是自己编写。不仅如此，我们只需使用简单的宏来帮助我们，这正是当前的 `async/await` 语法在成为语言一部分之前的原型设计方式。您可以在[`github.com/alexcrichton/futures-await`](https://github.com/alexcrichton/futures-await)中看到一个例子。

当然，缺点是这些函数看起来像普通的子程序，但实际上在本质上非常不同。在像 Rust 这样的强类型语言中，它使用借用语义而不是垃圾回收器，不可能隐藏这些函数是不同的这一事实。这可能会让程序员感到困惑，因为他们期望一切都能以相同的方式表现。

Coroutine bonus example

为了展示我们的例子与使用 Rust 中的 `std::future:::Future` 特性和 `async/await` 所获得的行为有多接近，我创建了一个与我们在 `a-coroutines` 中所做的完全相同的例子，使用“正确”的未来和 `async/await` 语法。您首先会注意到，这只需要对代码进行非常小的修改。其次，您可以亲自看到输出显示了与我们在自己编写状态机的例子中完全相同的程序流程。您将在存储库的 `ch07/a-coroutines-bonus` 文件夹中找到这个例子。

因此，让我们更进一步。为了避免混淆，并且由于我们的协程目前只向调用函数让步（还没有调度器、事件循环或类似的东西），我们使用一种稍微不同的语法，称为 `coroutine/wait`，并创建一种让我们自己生成这些状态机的方法。

## coroutine/wait

`coroutine/wait` 语法将与 `async/await` 语法有明显的相似之处，尽管它要有限得多。

基本规则如下：

+   每个以 `coroutine` 前缀的函数都将被重写为类似于我们编写的状态机。

+   函数上标记为 `coroutine` 的返回类型将被重写，以便它们返回 `-> impl Future<Output = String>`（是的，我们的语法将仅处理输出为 `String` 的未来）。

+   只有实现了 `Future` 特性的对象才能后缀 `.wait`。这些点将在我们的状态机中表示为单独的阶段。

+   以 `coroutine` 前缀的函数可以调用普通函数，但普通函数不能调用 `coroutine` 函数并期望有任何动作发生，除非它们反复调用 `poll` 直到返回 `PollState::Ready`。

我们的实现将确保如果我们编写以下代码，它将编译为我们在本章开头编写的相同状态机（除了所有协程都将返回一个 String）：

```rs
coroutine fn async_main() {
    println!("Program starting")
    let txt = Http::get("/1000/HelloWorld").wait;
    println!("{txt}");
    let txt2 = Http::("500/HelloWorld2").wait;
    println!("{txt2}");
}
```

但等等。`coroutine/wait` 在 Rust 中不是有效的关键字。如果我那样写，我会得到编译错误！

你是对的。所以，我创建了一个名为`corofy`的小程序，它将`coroutine/wait`函数重写为我们这些状态机。让我们快速解释一下。

## corofy——协程预处理器

在 Rust 中重写代码的最佳方式是使用宏系统。缺点是它并不清楚它最终编译成什么样子，而且对于我们的使用场景来说，展开宏并不是最优的，因为我们的主要目标之一是查看我们编写的代码与它转换成的内容之间的差异。此外，除非你经常使用宏，否则宏可能会变得相当复杂，难以阅读和理解。

相反，corofy 是你可以从仓库中的`ch07/corofy`下找到的正常 Rust 程序。

如果你进入那个文件夹，你可以通过写下以下内容来全局安装该工具：

```rs
cargo install --path .
```

现在你可以从任何地方使用这个工具了。它通过提供一个包含`coroutine/wait`语法的输入文件来工作，例如`corofy ./src/main.rs [可选输出文件]`。如果你没有指定输出文件，它将在同一文件夹中创建一个以`_corofied`后缀命名的文件。

注意

工具的功能极其有限。诚实的理由是我想在我们到达 2300 年之前完成这个示例，而且我重新从头开始重写了整个 Rust 编译器，只是为了提供一个使用`coroutine/wait`关键字时的稳健体验。

结果表明，在没有访问 Rust 的类型系统的情况下编写这样的转换是非常困难的。这个工具的主要用途将是转换我们在这里编写的示例，但它可能也适用于相同示例的微小变化（比如添加更多的等待点或在每个等待点之间执行更有趣的任务）。有关`corofy`的限制，请参阅 README。

还有一件事：我假设你指定了没有明确的输出文件，所以输出文件将与输入文件同名，后缀为`_corofied`。

程序读取你给出的文件，并搜索`coroutine`关键字的用法。它将这些函数注释掉（这样它们仍然在文件中），将它们放在文件末尾，并在`wait`点下面直接写出状态机实现，指出状态机的哪些部分是你实际上在`wait`点之间编写的代码。

现在我已经介绍了我们的新工具，是时候开始使用了。

## b-async-await——一个协程/wait 转换的示例

让我们先稍微扩展一下我们的示例。现在我们有一个程序可以输出我们的状态机，这样我们更容易创建一些示例并涵盖我们协程实现的一些更复杂的部分。

我们下面的示例将基于与第一个示例完全相同的代码。在仓库中，你可以在`ch07/b-async-await`下找到这个示例。

如果你从书中编写每个示例并且不依赖于仓库中的现有代码，你可以做两件事之一：

+   不断更改第一个示例中的代码

+   创建一个新的 cargo 项目，命名为`b-async-await`，并将上一个示例中的`src`文件夹和`Cargo.toml`中的`dependencies`部分的所有内容复制到新的项目中。

无论你选择什么，你都应该在你面前有相同的代码。

让我们简单地将`main.rs`中的代码更改为以下内容：

ch07/b-async-await/src/main.rs

```rs
use std::time::Instant;
mod http;
mod future;
use future::*;
use crate::http::Http;
fn get_path(i: usize) -> String {
    format!("/{}/HelloWorld{i}", i * 1000)
}
coroutine fn async_main() {
    println!("Program starting");
    let txt = Http::get(&get_path(0)).wait;
    println!("{txt}");
    let txt = Http::get(&get_path(1)).wait;
    println!("{txt}");
    let txt = Http::get(&get_path(2)).wait;
    println!("{txt}");
    let txt = Http::get(&get_path(3)).wait;
    println!("{txt}");
    let txt = Http::get(&get_path(4)).wait;
    println!("{txt}");
}
fn main() {
    let start = Instant::now();
    let mut future = async_main();
    loop {
        match future.poll() {
            PollState::NotReady => (),
            PollState::Ready(_) => break,
        }
    }
    println!("\nELAPSED TIME: {}", start.elapsed().as_secs_f32());
}
```

这段代码包含了一些更改。首先，我们添加了一个方便的函数`get_path`，用于创建新的路径，以便我们可以在 GET 请求中使用它，并基于我们传递的整数添加延迟和消息。

接下来，在我们的`async_main`函数中，我们创建了五个具有从`0`到`4`秒不同延迟的请求。

我们所做的最后一个更改是在我们的`main`函数中。我们不再在每次调用`poll`时打印消息，因此，我们不再使用`thread::sleep`来限制调用次数。相反，我们测量从我们进入`main`函数到退出它的时间，因为我们可以用这个作为证明我们的代码是否并发运行的方法。

现在由于我们的`main.rs`看起来与前面的示例相同，我们可以使用`corofy`将其重写为一个状态机，所以假设我们处于`ch07/b-async-await`的根目录中，我们可以编写以下内容：

```rs
corofy ./src/main.rs
```

这应该在`src`文件夹中输出一个名为`main_corofied.rs`的文件，你可以打开并检查它。

现在，你可以复制这个文件中`main_corofied.rs`的所有内容，并将其粘贴到`main.rs`中。

注意

为了方便，项目根目录中有一个名为`original_main.rs`的文件，其中包含我们之前展示的`main.rs`的代码，所以你不需要保存`main.rs`的原始内容。如果你通过从书中的项目复制它来自己编写每个示例，在你覆盖它之前将`main.rs`的原始内容存储在某个地方是明智的。

我不会在这里展示整个状态机，因为使用`coroutine/wait`编写的 39 行代码在作为状态机编写时变成了 170 行代码，但我们的`State` `enum`现在看起来像这样：

```rs
enum State0 {
    Start,
    Wait1(Box<dyn Future<Output = String>>),
    Wait2(Box<dyn Future<Output = String>>),
    Wait3(Box<dyn Future<Output = String>>),
    Wait4(Box<dyn Future<Output = String>>),
    Wait5(Box<dyn Future<Output = String>>),
    Resolved,
}
```

如果你使用`cargo run`运行程序，你现在会得到以下输出：

```rs
Program starting
FIRST POLL - START OPERATION
HTTP/1.1 200 OK
content-length: 11
connection: close
content-type: text/plain; charset=utf-8
date: Tue, xx xxx xxxx 21:05:55 GMT
HelloWorld0
FIRST POLL - START OPERATION
HTTP/1.1 200 OK
content-length: 11
connection: close
content-type: text/plain; charset=utf-8
date: Tue, xx xxx xxxx 21:05:56 GMT
HelloWorld1
FIRST POLL - START OPERATION
HTTP/1.1 200 OK
content-length: 11
connection: close
content-type: text/plain; charset=utf-8
date: Tue, xx xxx xxxx 21:05:58 GMT
HelloWorld2
FIRST POLL - START OPERATION
HTTP/1.1 200 OK
content-length: 11
connection: close
content-type: text/plain; charset=utf-8
date: Tue, xx xxx xxxx 21:06:01 GMT
HelloWorld3
FIRST POLL - START OPERATION
HTTP/1.1 200 OK
content-length: 11
connection: close
content-type: text/plain; charset=utf-8
date: Tue, xx xxx xxxx 21:06:05 GMT
HelloWorld4
ELAPSED TIME: 10.043025
```

所以，你看，我们的代码按预期运行。

由于我们在每次调用`Http::get`时都调用了`wait`，代码是顺序执行的，当我们查看 10 秒的经过时间时，这一点很明显。

这是有意义的，因为我们请求的延迟是`0 + 1 + 2 + 3 + 4`，等于 10 秒。

如果我们想让我们的未来运行并发呢？

你还记得我们讨论过这些未来是*懒的*吗？很好。所以，你知道仅仅创建一个未来并不能获得并发性。我们需要轮询它们以启动操作。

为了解决这个问题，我们借鉴了`join_all`的一些灵感。它接受一组未来，并将它们并发地驱动到完成。

让我们为这一章创建最后一个示例，其中我们只做这件事。

# c-async-await—并发未来

好的，我们将基于上一个示例继续进行，并做同样的事情。创建一个名为 `c-async-await` 的新项目，并将 `Cargo.toml` 和 `src` 文件夹中的所有内容复制过来。

我们首先要做的事情是去 `future.rs` 并在我们的现有代码下方添加一个 `join_all` 函数：

ch07/c-async-await/src/future.rs

```rs
pub fn join_all<F: Future>(futures: Vec<F>) -> JoinAll<F> {
    let futures = futures.into_iter().map(|f| (false, f)).collect();
    JoinAll {
        futures,
        finished_count: 0,
    }
}
```

这个函数接受一个未来集合作为参数，并返回一个 `JoinAll<F>` 未来。

这个函数只是创建一个新的集合。在这个集合中，我们将有由我们收到的原始未来和一个表示未来是否解决的 `bool` 值组成的元组。

接下来，我们有我们 `JoinAll` 结构体的定义：

ch07/c-async-await/src/future.rs

```rs
pub struct JoinAll<F: Future> {
    futures: Vec<(bool, F)>,
    finished_count: usize,
}
```

这个结构体将简单地存储我们创建的集合和一个 `finished_count`。最后一个字段将使跟踪有多少未来被解决变得稍微容易一些。

如我们所习惯的，大多数有趣的部分都发生在 `JoinAll` 的 `Future` 实现中：

```rs
impl<F: Future> Future for JoinAll<F> {
    type Output = String;
    fn poll(&mut self) -> PollState<Self::Output> {
        for (finished, fut) in self.futures.iter_mut() {
            if *finished {
                continue;
            }
            match fut.poll() {
                PollState::Ready(_) => {
                    *finished = true;
                    self.finished_count += 1;
                }
                PollState::NotReady => continue,
            }
        }
        if self.finished_count == self.futures.len() {
            PollState::Ready(String::new())
        } else {
            PollState::NotReady
        }
    }
}
```

我们将 `Output` 设置为 `String`。这可能会让你感到奇怪，因为我们实际上并没有从这个实现中返回任何东西。原因是 `corofy` 只与返回 `String` 的未来一起工作（这是它许多缺点之一），所以我们只是接受这一点，并在完成时返回一个空字符串。

接下来是 `poll` 实现的下一步。我们首先对每个（标志，未来）元组进行循环：

```rs
for (finished, fut) in self.futures.iter_mut()
```

在循环内部，我们首先检查这个未来的标志是否设置为 `finished`。如果是，我们只需转到集合中的下一个项目。

如果它还没有完成，我们 `poll` 这个未来。

如果我们得到 `PollState::Ready`，我们将这个未来的标志设置为 `true`，这样我们就不会再次轮询它，并增加完成计数。

注意

值得注意的是，我们在这里创建的 `join_all` 实现不会以任何有意义的方式与返回值的未来一起工作。在我们的例子中，我们只是扔掉了这个值，但请记住，我们现在试图尽可能保持简单，我们只想展示调用 `join_all` 的并发方面。

Tokio 的 `join_all` 实现将所有返回的值放入一个 `Vec<T>` 中，并在 `JoinAll` 未来解决时返回它们。

如果我们得到 `PollState::NotReady`，我们只需继续到集合中的下一个未来。

在遍历整个集合之后，我们检查我们是否已经解决了最初收到的所有未来，在 `if self.finished_count == self.futures.len()`。

如果我们所有的未来都已经被解决，我们将返回 `PollState::Ready` 并带一个空字符串（为了使 `corofy` 满意）。如果还有未解决的未来，我们将返回 `PollState::NotReady`。

重要

这里有一个需要注意的微妙之处。第一次调用`JoinAll::poll`时，它将对集合中的每个 future 调用`poll`。对每个 future 进行轮询将启动它们所代表的任何操作，并允许它们*并发地进步*。这是通过懒协程实现并发的一种方式，就像我们在这里处理的那样。

接下来，我们将对`main.rs`进行的更改。

`main`函数将保持不变，以及文件开头的导入和声明，所以我只会展示我们更改的`coroutine/await`函数：

```rs
coroutine fn request(i: usize) {
    let path = format!("/{}/HelloWorld{i}", i * 1000);
    let txt = Http::get(&path).wait;
    println!("{txt}");
}
coroutine fn async_main() {
    println!("Program starting");
    let mut futures = vec![];
    for i in 0..5 {
        futures.push(request(i));
    }
    future::join_all(futures).wait;
}
```

注意

在仓库中，如果你在所有复制粘贴的过程中丢失了跟踪，你可以在`ch07/c-async-await/original_main.rs`中找到放入`main.rs`的正确代码。

现在我们有两个`coroutine/wait`函数。`async_main`将`read_request`创建的一组协程存储在`Vec<T: Future>`中。

然后它创建了一个`JoinAll` future，并对其调用`wait`。

下一个`coroutine/wait`函数是`read_requests`，它接受一个整数作为输入，并使用该整数创建 GET 请求。这个协程将等待响应，并在响应到达时打印出结果。

由于我们创建的请求延迟为`0, 1, 2, 3, 4`秒，因此我们预计整个程序将在四秒多一点的时间内完成，因为所有任务都将*并发地进行*。那些延迟短的将在四秒延迟的任务完成时完成。

现在我们可以通过确保我们位于`ch07/c-async-await`文件夹中，并编写`corofy ./src/main.rs`，将我们的`coroutine/await`函数转换为状态机。

你现在应该在`src`文件夹中看到一个名为`main_corofied.rs`的文件。复制其内容，并用它替换`main.rs`中的内容。

如果你通过编写`cargo run`来运行程序，你应该会得到以下输出：

```rs
Program starting
FIRST POLL - START OPERATION
FIRST POLL - START OPERATION
FIRST POLL - START OPERATION
FIRST POLL - START OPERATION
FIRST POLL - START OPERATION
HTTP/1.1 200 OK
content-length: 11
connection: close
content-type: text/plain; charset=utf-8
date: Tue, xx xxx xxxx 21:11:36 GMT
HelloWorld0
HTTP/1.1 200 OK
content-length: 11
connection: close
content-type: text/plain; charset=utf-8
date: Tue, xx xxx xxxx 21:11:37 GMT
HelloWorld1
HTTP/1.1 200 OK
content-length: 11
connection: close
content-type: text/plain; charset=utf-8
date: Tue, xx xxx xxxx 21:11:38 GMT
HelloWorld2
HTTP/1.1 200 OK
content-length: 11
connection: close
content-type: text/plain; charset=utf-8
date: Tue, xx xxx xxxx 21:11:39 GMT
HelloWorld3
HTTP/1.1 200 OK
content-length: 11
connection: close
content-type: text/plain; charset=utf-8
date: Tue, xx xxx xxxx 21:11:40 GMT
HelloWorld4
ELAPSED TIME: 4.0084987
```

这里要注意的是经过的时间。现在正好超过四秒，就像我们预期的那样，当我们的 future 并发运行时。

如果我们看看`coroutine/await`是如何从程序员的视角改变编写协程的体验的，我们会看到我们现在离目标更近了：

+   **高效**：状态机不需要上下文切换，只需保存/恢复与该特定任务相关的数据。我们没有增长与分段栈的问题，因为它们都使用相同的操作系统提供的栈。

+   **表达性**：我们可以像在“正常”Rust 中一样编写代码，并且有了编译器的支持，我们可以得到相同的错误消息并使用相同的工具。

+   从一个普通函数到`async`函数，并期望发生任何有意义的事情；你必须以某种方式主动轮询它以完成，随着我们开始添加运行时，这会变得更加复杂。然而，就大部分而言，我们可以像我们习惯的那样编写程序。

# 最后的想法

在我们结束本章之前，我想指出，现在应该对我们来说已经很清楚为什么协程实际上并不是**可抢占的**。如果你还记得在*第二章*中，我们提到一个*堆栈型*协程（例如我们纤维/绿色线程的例子）可以被*抢占*，并且可以在任何点上暂停其执行。这是因为它们有一个堆栈，暂停一个任务就像将当前执行状态存储到堆栈中并跳转到另一个任务一样简单。

这在这里是不可能的。我们能停止和恢复执行的地方只有预先定义的挂起点，这些挂起点是我们手动用`wait`标记的。

理论上，如果你有一个紧密集成的系统，你控制编译器、协程定义、调度器和 I/O 原语，你可以向状态机添加额外的状态，并创建额外的挂起/恢复点。这些挂起点对用户来说是透明的，并且与正常的等待/挂起点不同对待。

例如，每次你遇到一个正常的函数调用时，你可以在我们的状态机中添加一个挂起点（一个新的状态），在那里你检查当前任务是否已经用完了其时间预算或类似的事情。如果是这样，你可以安排另一个任务运行，并在稍后某个时间点恢复任务，尽管这并没有以协作的方式进行。

然而，尽管这对用户来说是不可见的，但这并不等同于能够在代码的任何点停止/恢复执行。这也会违反协程通常隐含的协作性质。

# 摘要

干得好！在本章中，我们介绍了很多代码，并设置了一个示例，我们将在接下来的章节中继续使用。

到目前为止，我们一直专注于使用`futures`和`async/await`来模拟和创建可以在特定点暂停和恢复的任务。我们知道这是同时拥有正在执行的任务的先决条件。我们通过引入我们自己的简化版`Future`特性和我们自己的`coroutine/wait`语法来实现这一点，这些语法比 Rust 的`futures`和`async/await`语法要有限得多，但更容易理解，并且更容易在心理上理解这与纤维/绿色线程（至少我希望是这样）是如何工作的。

我们还讨论了急切协程和懒协程之间的区别以及它们如何影响你实现并发的方式。我们从 Tokio 的`join_all`函数中汲取了灵感，并实现了我们自己的版本。

在本章中，我们只是创建了可以暂停和恢复的任务。目前还没有事件循环、调度或其他类似的东西，但不用担心。它们正是我们将在下一章中探讨的内容。好消息是，像本章这样清晰地理解协程是非常困难的事情之一。
