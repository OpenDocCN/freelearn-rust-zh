# 10

# 创建您的自己的运行时

在最后几章中，我们涵盖了与 Rust 的异步编程相关的许多方面，但我们是通过实现比 Rust 当前的抽象更替代和更简单的抽象来做到这一点的。

这最后一章将专注于通过改变我们的运行时，使其与 Rust 的 futures 和 async/await 一起工作，而不是我们自己的 futures 和 coroutine/wait 来弥合这个差距。由于我们已经几乎涵盖了关于协程、状态机、futures、wakers、运行时和 pinning 的所有知识，因此调整我们现在所拥有的将是一个相对简单的任务。

当一切正常工作时，我们将对我们的运行时进行一些实验，以展示和讨论一些使异步 Rust 对新用户来说有些困难的方面。

在总结本书中我们所做和所学的内容之前，我们将花一些时间讨论在异步 Rust 中我们可能期待的未来。

我们将涵盖以下主要主题：

+   使用 futures 和 async/await 创建我们的运行时

+   对我们的运行时进行实验

+   异步 Rust 的挑战

+   异步 Rust 的未来

# 技术要求

本章中的示例将基于上一章的代码，因此要求相同。示例是跨平台的，将在所有 Rust ([`doc.rust-lang.org/beta/rustc/platform-support.html#tier-1-with-host-tools`](https://doc.rust-lang.org/beta/rustc/platform-support.html#tier-1-with-host-tools)) 和 `mio` (https://github.com/tokio-rs/mio#platforms) 支持的平台上运行。

您唯一需要的是安装 Rust 并下载本书的存储库。本章中的所有代码都可以在 `ch10` 文件夹中找到。

在这个例子中，我们也将使用 `delayserver`，因此您需要打开一个单独的终端，进入存储库根目录下的 `delayserver` 文件夹，并输入 `cargo run` 以确保它已准备好并可供后续示例使用。

如果出于某种原因您需要更改 `delayserver` 监听的端口，请记住在代码中更改端口。

```rs
Creating our own runtime with futures and async/await
```

好的，所以我们已经到了最后阶段；我们最后要做的就是改变我们的运行时，使其使用 Rust 的 `Future` 特性、`Waker` 和 `async/await`。现在我们已经通过自己构建一切来覆盖了 Rust 异步编程的几乎所有复杂方面，这将对我们来说是一个相对简单的任务。我们甚至对 Rust 在此过程中不得不做出的设计决策进行了相当详细的探讨。

Rust 当前的异步编程模型是经过演化过程的结果。Rust 在其早期阶段开始使用绿色线程，但那时它还没有达到 1.0 版本。在达到 1.0 版本时，Rust 的标准库中根本就没有 futures 或异步操作的概念。这个空间在 futures-rs crate ([`github.com/rust-lang/futures-rs`](https://github.com/rust-lang/futures-rs)) 中得到了探索，这个 crate 仍然是异步抽象的摇篮。然而，不久 Rust 就围绕一个类似于我们今天的 `Future` 特征的版本稳定下来，通常被称为 *futures 0.1*。支持由 async/await 创建的协程已经在进行中，但设计达到最终阶段并进入标准库的稳定版本还需要几年时间。

因此，我们在异步实现中做出的许多选择都是 Rust 在实现过程中必须做出的真实选择。然而，这一切都把我们带到了这个点，所以让我们开始适应我们的运行时，使其与 Rust 的 futures 兼容。

在我们进入示例之前，让我们了解一下与我们的当前实现不同的地方：

+   Rust 使用的 `Future` 特征与我们现在的有所不同。最大的不同是它使用 `Context` 而不是 `Waker`。另一个不同之处在于它返回一个名为 `Poll` 的枚举，而不是 `PollState`。

+   `Context` 是 Rust 的 `Waker` 类型的包装器。它的唯一目的是为了使 API 兼容未来，以便将来可以持有额外的数据，而无需更改与 `Waker` 相关的任何内容。

+   `Poll` 枚举返回两种状态之一，`Ready(T)` 或 `Pending`。这与我们当前的 `PollState` 枚举略有不同，但这两个状态在我们的当前实现中与 `Ready(T)/NotReady` 意义相同。

+   在 Rust 中创建 `Wakers` 比我们当前的 `Waker` 要复杂一些。我们将在本章后面讨论如何以及为什么。

除了上述的不同之处，其他所有内容都可以保持原样。大部分情况下，我们这次只是进行了重命名和重构。

既然我们已经有了需要做什么的想法，现在是时候设置一切，以便我们可以启动我们的新示例。

注意

尽管我们在 Rust 中创建了一个运行时来正确运行 futures，但我们仍然通过避免错误处理和不专注于使运行时更加灵活来保持其简单性。改进我们的运行时当然是有可能的，而且虽然有时正确使用类型系统并取悦借用检查器可能有点棘手，但这与 *异步* Rust 的关系相对较小，而更多的是与 Rust 本身有关。

## 设置示例

提示

你可以在书的仓库中找到这个示例，在 `ch1``0``/a-rust-futures` 文件夹中。

我们将继续在上一个章节中留下的地方继续，所以让我们将我们拥有的所有内容复制到一个新项目中：

1.  创建一个名为 `a-rust-futures` 的新文件夹。

1.  将前一章中的示例中的所有内容复制过来。如果你遵循了我建议的命名规则，它将存储在 `e-coroutines-pin` 文件夹中。

1.  现在你应该有一个包含我们之前示例副本的文件夹，所以最后要做的就是将 `Cargo.toml` 中的项目名称更改为 `a-rust-futures`。

好的，那么让我们从我们想要运行的程序开始。打开 `main.rs`。

## main.rs

在尝试任何更复杂的事情之前，我们先回到程序最简单的版本，并让它运行起来。打开 `main.rs` 并用以下代码替换该文件中的所有代码：

ch10/a-rust-futures/src/main.rs

```rs
mod http;
mod runtime;
use crate::http::Http;
fn main() {
    let mut executor = runtime::init();
    executor.block_on(async_main());
}
async fn async_main() {
    println!("Program starting");
    let txt = Http::get("/600/HelloAsyncAwait").await;
    println!("{txt}");
    let txt = Http::get("/400/HelloAsyncAwait").await;
    println!("{txt}");
}
```

这次不需要 `corofy` 或任何特殊的东西。编译器会为我们重写这些。

注意

注意，我们已经移除了 `future` 模块的声明。这是因为我们根本不再需要它了。唯一的例外是，如果你想要保留并使用我们创建的用于将多个 future 连接起来的 `join_all` 函数。你可以尝试自己重写它，或者查看仓库并定位到 `ch1``0``/a-rust-futures-bonus/src/future.rs` 文件，在那里你可以找到我们示例的相同版本，只是这个版本保留了带有与 Rust futures 一起工作的 `join_all` 函数的 future 模块。

## future.rs

你可以完全删除这个文件，因为我们不再需要我们自己的 `Future` trait 了。

让我们直接进入 `http.rs` 并看看我们需要在那里做哪些更改。

## http.rs

我们需要更改的第一件事是我们的依赖项。我们将不再依赖于我们自己的 `Future`、`Waker` 和 `PollState`；相反，我们将依赖于标准库中的 `Future`、`Context` 和 `Poll`。我们的依赖项现在应该看起来像这样：

ch10/a-rust-futures/src/http.rs

```rs
use crate::runtime::{self, reactor};
use mio::Interest;
use std::{
    future::Future,
    io::{ErrorKind, Read, Write},
    pin::Pin,
    task::{Context, Poll},
};
```

我们必须在 `HttpGetFuture` 的 `poll` 实现中进行一些小的重构。

首先，我们需要更改 `poll` 函数的签名，使其符合新的 `Future` trait：

ch10/a-rust-futures/src/http.rs

```rs
fn poll(mut self: Pin<&mut Self>, cx, we have to change what we pass in to set_waker with the following:
			ch10/a-rust-futures/src/http.rs

```

runtime::reactor().set_waker(Poll instead of PollState. 要做到这一点，找到 poll 方法，并首先更改签名，使其与标准库中的 Future trait 匹配：

            ch10/a-rust-futures/src/http.rs

```rs
fn poll(mut self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>
```

            接下来，我们需要更改函数返回类型的地方（我在这里只展示了函数体的相关部分）：

            ch10/a-rust-futures/src/http.rs

```rs
loop {
            match self.stream.as_mut().unwrap().read(&mut buff) {
                Ok(0) => {
                    let s = String::from_utf8_lossy(&self.buffer).to_string();
                    runtime::reactor().deregister(self.stream.as_mut().unwrap(), id);
                    break Poll::Ready(s.to_string());
                }
                Ok(n) => {
                    self.buffer.extend(&buff[0..n]);
                    continue;
                }
                Err(e) if e.kind() == ErrorKind::WouldBlock => {
                    // always store the last given Waker
                    runtime::reactor().set_waker(cx, self.id);
                    break Poll::Pending;
                }
                Err(e) => panic!("{e:?}"),
            }
        }
```

            这个文件就到这里。不错吧？让我们看看在 executor 中我们需要做哪些更改，并打开 `executor.rs`。

            executor.rs

            在 `executor.rs` 中，我们需要更改的第一件事是我们的依赖项。这次，我们只依赖于标准库中的类型，并且我们的 `dependencies` 部分现在应该看起来像这样：

            ch10/a-rust-futures/src/runtime/executor.rs

```rs
use std::{
    cell::{Cell, RefCell},
    collections::HashMap,
    future::Future,
    pin::Pin,
    sync::{Arc, Mutex},
    task::{Poll, Context, Wake, Waker},
    thread::{self, Thread},
};
```

            我们的任务不再局限于仅输出 String，因此我们可以安全地为我们顶级 future 使用更合理的 `Output` 类型：

            ch10/a-rust-futures/src/runtime/executor.rs

```rs
type Task = Pin<Box<dyn Future<Output = Waker since the changes we make here will result in several other changes to this file.
			Creating a waker in Rust can be quite a complex task since Rust wants to give us maximum flexibility on how we choose to implement wakers. The reason for this is twofold:

				*   Wakers must work just as well on a server as it does on a microcontroller
				*   A waker must be a zero-cost abstraction

			Realizing that most programmers never need to create their own wakers, the cost that the lack of ergonomics has was deemed acceptable.
			Until quite recently, the only way to construct a waker in Rust was to create something very similar to a trait object without being a trait object. To do so, you had to go through quite a complex process of constructing a *v-table* (a set of function pointers), combining that with a pointer to the data that the waker stored, and creating  `RawWaker`.
			Fortunately, we don’t actually have to go through this process anymore as Rust now has the `Wake` trait. The `Wake` trait works if the `Waker` type we create is placed in `Arc`.
			Wrapping `Waker` in an `Arc` results in a heap allocation, but for most `Waker` implementations on the kind of systems we’re talking about in this book, that’s perfectly fine and what most production runtimes do. This simplifies things for us quite a bit.
			Info
			This is an example of Rust adopting what turns out to be best practices from the ecosystem. For a long time, a popular way to construct wakers was by implementing a trait called `ArcWake` provided by the `futures` crate ([`github.com/rust-lang/futures-rs`](https://github.com/rust-lang/futures-rs)). The `futures` crate is not a part of the language but it’s in the `rust-lang` repository and can be viewed much like a toolbox and nursery for abstractions that might end up in the language at some point in the future.
			To avoid confusion by having multiple things with the same name, let’s rename our concrete `Waker` type to `MyWaker`:
			ch10/a-rust-futures/src/runtime/executor.rs

```

#[derive(Clone)]

pub struct MyWaker {

thread: Thread,

id: usize,

ready_queue: Arc<Mutex<Vec<usize>>>,

}

```rs

			We can keep the implementation of `wake` pretty much the same, but we put it in the implementation of the `Wake` trait instead of just having a `wake` function on `MyWaker`:
			ch10/a-rust-futures/src/runtime/executor.rs

```

impl Wake for MyWaker {

fn wake(self: Arc<Self>) {

self.ready_queue

.lock()

.map(|mut q| q.push(self.id))

.unwrap();

self.thread.unpark();

}

}

```rs

			You’ll notice that the `wake` function takes a `self: Arc<Self>` argument, much like we saw when working with the `Pin` type. Writing the function signature this way means that `wake` is only callable on `MyWaker` instances that are wrapped in `Arc`.
			Since our `waker` has changed slightly, there are a few places we need to make some minor corrections. The first is in the `get_waker` function:
			ch10/a-rust-futures/src/runtime/executor.rs

```

fn get_waker(&self, id: usize) -> Arc<MyWaker> {

Arc::new(MyWaker {

id,

thread: thread::current(),

ready_queue: CURRENT_EXEC.with(|q| q.ready_queue.clone()),

})

}

```rs

			So, not a big change here. The only difference is that we heap-allocate the waker by placing it in `Arc`.
			The next place we need to make a change is in the `block_on` function.
			First, we need to change its signature so that it matches our new definition of a top-level future:
			ch10/a-rust-futures/src/runtime/executor.rs

```

pub fn block_on<F>(&mut self, future: F)

where

F: Future<Output = ()> + 'static,

{

```rs

			The next step is to change how we create a waker and wrap it in a `Context` struct in the `block_on` function:
			ch10/a-rust-futures/src/runtime/executor.rs

```

...

// 防止假唤醒

None => continue,

};

let waker: Waker = self.get_waker(id).into();

let mut cx = Context::from_waker(&waker);

match future.as_mut().poll(&mut cx) {

...

```rs

			This change is a little bit complex, so we’ll go through it step by step:

				1.  First, we get `Arc<MyWaker>` by calling the `get_waker` function just like we did before.
				2.  We convert `MyWaker` into a simple `Waker` by specifying the type we expect with `let waker: Waker` and calling `into()` on `MyWaker`. Since every instance of `MyWaker` is also a kind of `Waker`, this will convert it into the `Waker` type that’s defined in the standard library, which is just what we need.
				3.  Since `Future::poll` expects  `Context` and not `Waker`, we create a new `Context` struct with a reference to the waker we just created.

			The last place we need to make changes is to the signature of our `spawn` function so that it takes the new definition of top-level futures as well:
			ch10/a-rust-futures/src/runtime/executor.rs

```

pub fn spawn<F>(future: F)

where

F: Future<Output = reactor.rs.

            reactor.rs

            我们首先确保我们的依赖项是正确的。我们必须删除对旧 `Waker` 实现的依赖，而是从标准库中引入这些类型。`dependencies` 部分应该看起来像这样：

            ch10/a-rust-futures/src/runtime/reactor.rs

```rs
use mio::{net::TcpStream, Events, Interest, Poll, Registry, Token};
use std::{
    collections::HashMap,
    sync::{
        atomic::{AtomicUsize, Ordering},
        Arc, Mutex, OnceLock,
    },
    thread, task::{Context, Waker},
};
```

            我们需要做出两个小的更改。第一个更改是我们的 `set_waker` 函数现在接受 `Context`，它需要从中获取一个 `Waker` 对象：

            ch10/a-rust-futures/src/runtime/reactor.rs

```rs
pub fn set_waker(&self, cx: &Context, id: usize) {
        let _ = self
            .wakers
            .lock()
            .map(|mut w| w.insert(id, cx.waker().clone()).is_none())
            .unwrap();
    }
```

            最后的更改是在调用 `event_loop` 函数中的 `wake` 时需要调用一个稍微不同的方法：

            ch10/a-rust-futures/src/runtime/reactor.rs

```rs
if let Some(waker) = wakers.get(&id) {
    waker.wake_by_ref();
}
```

            由于现在调用 `wake` 会消耗 `self`，因此我们调用接受 `&self` 的版本，因为我们想保留这个 `waker` 以供以后使用。

            就这样。我们的运行时现在可以运行并充分利用异步 Rust 的全部功能。让我们在终端中输入 `cargo run` 来试试。

            我们应该得到之前看到过的相同输出：

```rs
Program starting
FIRST POLL - START OPERATION
main: 1 pending tasks. Sleep until notified.
HTTP/1.1 200 OK
content-length: 15
[==== ABBREVIATED ====]
HelloAsyncAwait
main: All tasks are finished
```

            这非常不错，不是吗？

            因此，现在我们已经创建了自己的异步运行时，它使用 Rust 的 `Future`、`Waker`、`Context` 和 `async/await`。

            现在我们可以自豪地称自己是运行时实现者，是时候做一些实验了。我会选择几个实验，这些实验也会让我们了解 Rust 中的运行时和 futures。我们还没有学完。

            在我们的运行时中进行实验

            注意

            你可以在书的存储库中找到这个示例，在 `ch1``0``/b-rust-futures-experiments` 文件夹中。不同的实验将作为 `async_main` 函数的不同版本按时间顺序实现。我将在代码片段的标题中指示哪个函数对应于存储库示例中的哪个函数。

            在我们开始实验之前，让我们把我们现在拥有的所有内容复制到一个新的文件夹中：

                1.  创建一个名为 `b-rust-futures-experiments` 的新文件夹。

                1.  将 `a-rust-futures` 文件夹中的所有内容复制到新文件夹中。

                1.  打开 `Cargo.toml` 并将 `name` 属性更改为 `b-rust-futures-experiments`。

            第一个实验将是用合适的 HTTP 客户端替换我们非常有限的 HTTP 客户端。

            做这件事最简单的方法是简单地选择另一个支持异步 Rust 的生产级 HTTP 客户端库，并使用它代替。

            因此，当我们试图找到我们 HTTP 客户端的合适替代品时，我们检查了最受欢迎的高级 HTTP 客户端库列表，并发现 `reqwest` 排在首位。这可能适用于我们的目的，所以让我们先尝试一下。

            我们首先做的事情是在 `Cargo.toml` 中将 `reqwest` 添加为依赖项，通过输入以下内容：

```rs
cargo add reqwest@0.11
```

            接下来，让我们更改我们的 `async_main` 函数，以便我们使用 `reqwest` 而不是我们自己的 HTTP 客户端：

            ch10/b-rust-futures-examples/src/main.rs (async_main2)

```rs
async fn async_main() {
    println!("Program starting");
    let url = "http://127.0.0.1:8080/600/HelloAsyncAwait1";
    let res = reqwest::get(url).await.unwrap();
    let txt = res.text().await.unwrap();
    println!("{txt}");
    let url = "http://127.0.0.1:8080/400/HelloAsyncAwait2";
    let res = reqwest::get(url).await.unwrap();
    let txt = res.text().await.unwrap();
    println!("{txt}");
}
```

            除了使用 `reqwest` API 之外，我还更改了我们发送的消息。大多数 HTTP 客户端不会返回原始 HTTP 响应给我们，通常只提供一种方便的方式来获取响应的 *body*，直到现在，我们的请求对此都是类似的。

            这应该就是我们需要更改的所有内容，所以让我们尝试通过编写 `cargo run` 来运行我们的程序：

```rs
     Running `target\debug\a-rust-futures.exe`
Program starting
thread 'main' panicked at C:\Users\cf\.cargo\registry\src\index.crates.io-6f17d22bba15001f\tokio-1.35.0\src\net\tcp\stream.rs:160:18:
there is no reactor running, must be called from the context of a Tokio 1.x runtime
```

            好吧，所以错误告诉我们没有运行反应器，并且它必须从 Tokio 1.x 运行时的上下文中调用。我们知道有一个反应器正在运行，只是不是 `reqwest` 期望的那个，所以让我们看看我们如何解决这个问题。

            我们显然需要将 Tokio 添加到我们的程序中，由于 Tokio 严重功能受限（这意味着默认情况下只有很少的功能被启用），我们将简化操作，并启用所有功能：

```rs
cargo add tokio@1 --features full
```

            根据文档，我们需要启动一个 Tokio 运行时，并显式进入它以启用反应器。`enter` 函数将返回 `EnterGuard` 给我们，只要我们需要反应器运行，我们就可以持有它。

            将此添加到我们的 `async_main` 函数顶部应该可以工作：

            ch10/b-rust-futures-examples/src/main.rs (async_main2)

```rs
use tokio::runtime::Runtime;
async fn async_main
    let rt = Runtime::new().unwrap();
    let _guard = rt.enter();
    println!("Program starting");
    let url = "http://127.0.0.1:8080/600/HelloAsyncAwait1";
    ...
```

            注意

            调用 `Runtime::new` 创建一个多线程的 Tokio 运行时，但 Tokio 也有一个单线程的运行时，你可以通过使用运行时构建器来创建，如下所示：`Builder::new_current_thread().enable_all().build().unwrap()`。如果你这样做，你最终会遇到一个奇特的问题：死锁。这个原因很有趣，你应该知道。

            Tokio 的单线程运行时只使用它被调用的线程来执行执行器和反应器。这与我们在运行时的第一个版本中做的事情非常相似。第八章。我们使用了 `Poll` 实例直接挂起我们的执行器。当我们的反应器和执行器在同一个线程上执行时，它们必须具有相同的机制来自动挂起并等待新事件，这意味着它们之间将存在紧密的耦合。

            在处理事件时，反应器必须首先唤醒以调用`Waker::wake`，但执行器是最后一个调用`thread::park`（就像我们做的那样）的。如果执行器通过调用`thread::park`（就像我们做的那样）自己挂起，那么反应器也会挂起，并且由于它们在同一个线程上运行，将永远不会唤醒。使这一切正常工作的唯一方法就是执行器挂起与反应器共享的东西（就像我们用`Poll`做的那样）。由于我们与 Tokio 没有紧密集成，我们得到的只是一个死锁。

            现在，如果我们再次尝试运行我们的程序，我们会得到以下输出：

```rs
Program starting
main: 1 pending tasks. Sleep until notified.
main: 1 pending tasks. Sleep until notified.
main: 1 pending tasks. Sleep until notified.
HelloAsyncAwait1
main: 1 pending tasks. Sleep until notified.
main: 1 pending tasks. Sleep until notified.
main: 1 pending tasks. Sleep until notified.
HelloAsyncAwait2
main: All tasks are finished
```

            好吧，所以现在一切如预期工作。唯一的区别是我们被唤醒了几次，但程序完成了并产生了预期的结果。

            在我们讨论我们刚才看到的情况之前，让我们再做一个实验。

            **Isahc**是一个承诺为*执行器无关*的 HTTP 客户端库，这意味着它不依赖于任何特定的执行器。让我们来测试一下。

            首先，我们通过输入以下内容添加对`isahc`的依赖：

```rs
cargo add isahc@1.7
```

            然后，我们重写我们的`main`函数，使其看起来像这样：

            ch10/b-rust-futures-examples/src/main.rs (async_main3)

```rs
use isahc::prelude::*;
async fn async_main() {
    println!("Program starting");
    let url = "http://127.0.0.1:8080/600/HelloAsyncAwait1";
    let mut res = isahc::get_async(url).await.unwrap();
    let txt = res.text().await.unwrap();
    println!("{txt}");
    let url = "http://127.0.0.1:8080/400/HelloAsyncAwait2";
    let mut res = isahc::get_async(url).await.unwrap();
    let txt = res.text().await.unwrap();
    println!("{txt}");
}
```

            现在，如果我们通过编写`cargo run`来运行我们的程序，我们会得到以下输出：

```rs
Program starting
main: 1 pending tasks. Sleep until notified.
main: 1 pending tasks. Sleep until notified.
main: 1 pending tasks. Sleep until notified.
HelloAsyncAwait1
main: 1 pending tasks. Sleep until notified.
main: 1 pending tasks. Sleep until notified.
main: 1 pending tasks. Sleep until notified.
HelloAsyncAwait2
main: All tasks are finished
```

            因此，我们得到了预期的输出，而无需跳过任何障碍。

            *为什么这一切都必须如此不直观？*

            那个问题的答案把我们带到了我们所有人编程时都会遇到的一些常见挑战的话题，所以让我们来谈谈其中一些最明显的挑战，并解释它们存在的原因，这样我们就可以找出如何最好地处理它们。

            异步 Rust 的挑战

            所以，虽然我们亲眼看到执行器和反应器可以被松散耦合，这反过来意味着理论上你可以混合匹配反应器和执行器，但问题是为什么我们在尝试这样做时会遇到如此多的摩擦？

            大多数使用过异步 Rust 的程序员都经历过由不兼容的异步库引起的问题，我们之前看到了你可能会得到的错误消息的例子。

            要理解这一点，我们必须稍微深入了解一下 Rust 中现有的异步运行时，特别是那些我们通常用于桌面和服务器应用程序的。

            显式与隐式反应器实例化

            信息

            我们接下来要讨论的未来类型是叶子未来，这种类型实际上代表了一个 I/O 操作（例如，`HttpGetFuture`）。

            当你在 Rust 中创建运行时，你还需要创建 Rust 标准库的非阻塞原语。互斥锁、通道、计时器、TcpStreams 等等都是需要异步等价物的东西。

            这些大多数都可以实现为不同类型的反应器，但随之而来的问题是：那个反应器是如何启动的？

            在我们自己的运行时和 Tokio 中，反应器作为运行时初始化的一部分启动。我们有一个`runtime::init()`函数，它调用`reactor::start()`，而 Tokio 有一个`Runtime::new()`和`Runtime::enter()`函数。

            如果我们尝试在没有启动反应器的情况下创建一个叶子 future（我们创建的唯一一个是`HttpGetFuture`），那么我们的运行时和 Tokio 都会崩溃。反应器必须**显式**实例化。

            相反，Isahc 带来了它自己的一种反应器。Isahc 建立在`libcurl`之上，这是一个高度可移植的 C 库，`libcurl`接受一个在操作准备就绪时被调用的回调。因此，Isahc 将接收到的唤醒器传递给这个回调，并确保在回调执行时调用`Waker::wake`。这有点过于简化，但基本上就是这样发生的。

            实际上，这意味着 Isahc 带来了它自己的反应器，因为它带有存储唤醒器并在操作准备就绪时对它们调用`wake`的机制。反应器是**隐式**启动的。

            偶然的是，这也是`async_std`和 Tokio 之间的一大主要区别。Tokio 需要**显式**实例化，而`async_std`则依赖于**隐式**实例化。

            我并不是为了好玩而深入探讨这个问题；虽然这似乎是一个微小的差异，但它对 Rust 中异步编程的直观性有着相当大的影响。

            这个问题主要在你开始使用不同于 Tokio 的其他运行时编程时出现，然后必须使用内部依赖于 Tokio 反应器存在的库。

            由于你无法在同一个线程上运行两个 Tokio 实例，因此库不能隐式启动一个 Tokio 反应器。相反，通常会发生的情况是，你尝试使用那个库，并得到一个像我们在前面的例子中遇到的那种错误。

            现在，你必须自己启动一个 Tokio 反应器，使用其他人创建的某种兼容包装器，或者查看你使用的运行时是否有内置机制来运行依赖于 Tokio 反应器的 future。

            对于大多数不了解反应器、执行器和不同类型的叶子 future 的人来说，这可能会相当不直观，并造成相当多的挫败感。

            注意

            我们在这里描述的问题相当常见，而且由于异步库很少很好地解释这一点，甚至很少尝试明确说明它们使用的运行时类型，这并没有得到帮助。一些库可能在`README`文件中某处提到它们是建立在 Tokio 之上的，而一些库可能只是简单地声明它们是建立在 Hyper 之上的，例如，假设你知道 Hyper 是建立在 Tokio 之上的（至少默认情况下是这样）。

            但是现在，你知道你应该检查这一点以避免任何惊喜，如果你遇到这个问题，你就知道确切的问题是什么。

            人体工程学 versus 效率和灵活性

            Rust 擅长于易用性和效率，这几乎让人难以忘记，当 Rust 面临在效率*或*易用性之间做出选择时，它将选择效率。生态系统中最受欢迎的许多 crate 都反映了这些价值观，这包括异步运行时。

            如果任务与执行者紧密集成，它们可以更有效率，因此，如果你在库中使用它们，你将依赖于那个特定的运行时。

            以**计时器**为例，但任务通知，其中*任务 A*通知*任务 B*它可以继续，这也是另一个具有一些相同权衡的例子。

            任务

            我们在未明确区分任务和未来的情况下使用了这些术语，所以让我们在这里澄清一下。我们首先在*第一章*中介绍了任务，它们仍然保留着相同的一般含义，但当我们谈论 Rust 中的运行时，它们有一个更具体的定义。任务是一个*顶级未来*，是我们向执行者播种的那个。执行者在不同任务之间进行调度。在运行时中，任务在很多方面代表了与操作系统中的线程相同的抽象。每个任务在 Rust 中都是一个未来，但根据这个定义，每个未来并不都是一个任务。

            你可以将`thread::sleep`视为一个计时器，在异步环境中我们经常需要这样的东西，因此我们的异步运行时将需要有一个`sleep`等效功能，告诉执行者将这个任务休眠指定的时间。

            我们可以将其实现为一个反应器，并为指定的持续时间让单独的操作系统线程休眠，然后唤醒正确的`Waker`。这将很简单，并且与执行者无关，因为执行者对发生的事情一无所知，它只关心在`Waker::wake`被调用时调度任务。然而，对于所有工作负载来说，这也不是最优效的（即使我们为所有计时器使用相同的线程）。

            另一种，且更为常见的方法是委托这个任务给执行者。在我们的运行时中，这可以通过让执行者存储一个有序的瞬间列表和相应的`Waker`来实现，这个`Waker`用于确定在它调用`thread::park`之前是否有任何计时器已过期。如果没有过期，我们可以计算出下一个计时器过期前的持续时间，并使用类似`thread::park_timeout`的东西来确保我们至少醒来处理那个计时器。

            用于存储计时器的算法可以高度优化，并且你可以避免仅为了计时器而需要额外线程的需求，以及这些线程之间同步的额外开销，仅为了通知一个计时器已过期。在多线程运行时中，当多个执行者频繁地向同一反应器添加计时器时，甚至可能会出现竞争。

            一些计时器以反应器风格作为独立的库实现，对于许多任务来说，这已经足够了。这里的关键点是，通过使用默认设置，你最终会绑定到一个特定的运行时，如果你想要避免你的库与特定的运行时紧密耦合，就必须进行仔细的考虑。

            每个人都同意的常见特质

            异步 Rust 中引起摩擦的最后一个话题是缺乏普遍认可的特质和接口，用于典型的异步操作。

            我想通过指出这一点来为这一部分内容做开场白：这是一个每天都在不断改进的领域，在`futures-rs`包中有一个异步 Rust 的特性和抽象的苗圃（[`github.com/rust-lang/futures-rs`](https://github.com/rust-lang/futures-rs)）。然而，由于异步 Rust 还处于早期阶段，在这样一本书中提及这一点是值得的。

            让我们以创建任务为例。当你用 Rust 编写一个高级异步库，比如一个网络服务器时，你可能会希望能够创建新的任务（顶级未来）。例如，服务器上的每个连接很可能是你想要在执行器上创建的新任务。

            现在，创建任务对每个执行器都是特定的，Rust 没有定义如何创建任务的特质。在`future-rs`包中建议了一个用于创建任务的特质，但创建一个既无成本又足够灵活以支持所有运行时的创建特质非常困难。

            有一些方法可以绕过这个问题。例如，流行的 HTTP 库 Hyper ([`hyper.rs/`](https://hyper.rs/))使用一个特质来表示执行器，并在内部使用它来创建新任务。这使得用户能够为不同的执行器实现这个特质并将其返回给 Hyper。通过为不同的执行器实现这个特质，Hyper 将使用一个与默认选项不同的创建者（默认选项是 Tokio 的执行器）。以下是如何使用 Hyper 与`async_std`一起使用的一个例子：[`github.com/async-rs/async-std-hyper`](https://github.com/async-rs/async-std-hyper)。

            然而，由于没有一种通用的方法来实现这一点，大多数依赖于特定执行器功能的库会做以下两件事之一：

                1.  选择一个运行时并坚持下去。

                1.  实现两个版本的库，支持用户通过启用正确的功能选择的不同流行运行时。

            异步释放

            异步释放，或者说异步析构函数，是异步 Rust 在撰写本书时尚未完全解决的问题。Rust 使用一种称为 RAII 的模式，这意味着当类型被创建时，它的资源也会被创建，当类型被销毁时，资源也会被释放。编译器会自动在对象超出作用域时插入一个释放调用的调用。

            以我们的运行时为例，当资源被释放时，它们以阻塞的方式释放。这通常不是一个大问题，因为释放不太可能长时间阻塞执行器，但并不总是这样。

            如果我们有一个需要很长时间才能完成的释放实现（例如，如果释放需要管理 I/O，或者需要对操作系统内核进行阻塞调用，这在 Rust 中是合法的，有时甚至不可避免），它可能会阻塞执行器。所以，在这种情况下，异步释放应该能够以某种方式向调度器让步，但目前这是不可能的。

            现在，这并不是作为异步库用户你可能会遇到的异步 Rust 的粗糙边缘，但了解这一点是值得的，因为目前，确保这不会引起问题的唯一方法是在类型中使用异步上下文时，小心地将内容放入释放实现中。

            因此，虽然这不是异步 Rust 中所有导致摩擦因素的详尽列表，但它是我认为最明显且值得了解的一些点。

            在我们结束这一章之前，让我们花一点时间谈谈在 Rust 中进行异步编程时，我们应该期待未来有什么。

            异步 Rust 的未来

            使异步 Rust 与其他语言不同的某些特性是不可避免的。异步 Rust 非常高效，延迟低，由于语言的设计和其核心价值，它背后有一个非常强大的类型系统。

            然而，今天感知到的许多复杂性更多与生态系统有关，以及大量程序员在没有正式结构的情况下必须就解决不同问题的最佳方式达成一致所导致的问题。生态系统在一段时间内变得碎片化，再加上异步编程对许多程序员来说是一个难以理解的话题，这最终增加了与异步 Rust 相关的认知负荷。

            我在本章中提到的所有问题和痛点都在不断改进。一些几年前可能出现在这个列表上的点现在甚至不值得提及。

            越来越多的常见特性和抽象将最终出现在标准库中，这使得异步 Rust 更加易用，因为使用它们的任何东西都将“直接工作”。

            随着不同的实验和设计比其他设计获得更多的关注，它们成为了事实上的标准，尽管在编写异步 Rust 时，你仍然有很多选择，但将会有一些路径可供选择，这些路径对那些想要“直接工作”的人来说摩擦最小。

            在对异步 Rust 和异步编程有足够的了解之后，我提到的这些问题毕竟相对较小，而且由于你对异步 Rust 的了解比大多数程序员都要多，我很难想象这些问题会给你带来很多麻烦。

            这并不意味着它不是一些值得了解的事情，因为你的同行程序员可能会在某个时刻遇到这些问题。

            摘要

            因此，在本章中，我们做了两件事。首先，我们对我们的运行时进行了一些相当小的修改，使其能够作为 Rust futures 的实际运行时。我们使用两个外部 HTTP 客户端库测试了运行时，以了解 Rust 中的 reactors、运行时和异步库。

            我们接下来讨论了一些使异步 Rust 对许多来自其他语言的程序员来说困难的事情。最后，我们也讨论了未来可以期待什么。

            根据你如何跟随学习以及你对所创建的示例进行了多少实验，如果你想要学习更多，你可以自己承担任何项目。

            学习的一个重要方面只在你自己进行实验时才会发生。拆解一切，看看什么会出错，以及如何修复它。改进我们创建的简单运行时，以学习新知识。

            有足够有趣的项目可以选择，但这里有一些建议：

                +   将我们使用`thread::park`实现的 parker 替换为合适的 parker。你可以从库中选择一个，或者自己创建一个 parker（我在`ch1``0`文件夹的末尾添加了一个名为`parker-bonus`的小奖励，其中包含一个简单的 parker 实现）。

                +   使用你自己创建的运行时实现一个简单的`delayserver`。为此，你必须能够编写一些原始的 HTTP 响应并创建一个简单的服务器。如果你阅读了名为《Rust 编程语言》的免费入门书籍，你就在最后一章中创建了一个简单的服务器（[`doc.rust-lang.org/book/ch20-02-multithreaded.html`](https://doc.rust-lang.org/book/ch20-02-multithreaded.html)），这为你提供了所需的基础。你还需要创建一个计时器，就像我们上面讨论的那样，或者使用现有的异步计时器 crate。

                +   你可以创建一个“正确”的多线程运行时，探索拥有全局任务队列的可能性，或者作为替代方案，实现一个可以当其他执行器完成自己的任务后从它们本地队列中窃取任务的 work-stealing 调度器。

            只有你的想象力才能设定你能做到的界限。重要的是要注意，仅仅因为你可以并且只是为了乐趣而做某件事情，这本身就是一种乐趣，我希望你能从中获得和我一样的乐趣。

            我将以几句话结束本章，谈谈如何使异步程序员的日常生活尽可能轻松。

            第一件事是意识到异步运行时不仅仅是一个你使用的库。它非常侵入性，几乎影响你程序中的每一件事。它是一个层，它重写、调度任务，并重新排序你习惯的程序流程。

            如果你不是特别想学习运行时，或者有非常具体的需求，我的明确建议是选择一个运行时，并坚持使用一段时间。了解它的所有内容——不一定一开始就了解所有内容，但随着你需要越来越多的功能，你最终会了解所有内容。这几乎就像在 Rust 的标准库中熟悉所有内容一样。

            你开始使用哪个运行时取决于你使用最多的 crates。Smol 和`async-std`共享许多实现细节，并且行为相似。它们的主要卖点在于它们的 API 力求尽可能接近标准库。结合反应器隐式实例化的事实，这可以带来略微更直观的体验和更平缓的学习曲线。两者都是生产级运行时，并且得到了广泛的应用。Smol 最初创建的目的是为了拥有一个程序员易于理解和学习的代码库，我认为这一点在今天仍然适用。

            话虽如此，截至写作时，对于寻找通用运行时的用户来说，最受欢迎的替代方案是**Tokio**([`tokio.rs/`](https://tokio.rs/))。Tokio 是 Rust 中最古老的异步运行时之一。它正在积极开发中，拥有一个友好且活跃的社区。文档非常出色。作为最受欢迎的运行时之一，这也意味着你很可能找到一个库，它正好满足你的需求，并且默认支持 Tokio。就我个人而言，我倾向于选择 Tokio，原因如上所述，但除非你有非常具体的需求，否则你不会选择这两个运行时中的任何一个出错。

            最后，我们不要忘记提到`futures-rs`crate([`github.com/rust-lang/futures-rs`](https://github.com/rust-lang/futures-rs))。我之前提到过这个 crate，但了解它非常有用，因为它包含了一些特质、抽象和异步 Rust 的执行器([`docs.rs/futures/latest/futures/executor/index.html`](https://docs.rs/futures/latest/futures/executor/index.html))。它充当一个异步工具箱，在许多情况下都很有用。

            后记

            因此，你已经走到了尽头。首先，恭喜！你已经完成了一段相当漫长的旅程！

            我们首先在*第一章*中讨论了并发和并行。我们甚至简要地介绍了历史、CPU 和操作系统、硬件以及中断。在*第二章*中，我们讨论了编程语言如何模拟异步程序流程。我们介绍了协程以及堆栈式和堆栈无关协程的区别。我们还讨论了操作系统线程、纤程/绿色线程和回调及其优缺点。

            然后，在 *第三章*（B20892_03.xhtml#_idTextAnchor063）中，我们研究了基于操作系统的事件队列，如 `epoll`、`kqueue` 和 IOCP。我们甚至深入研究了系统调用和跨平台抽象。

            在 *第四章*（B20892_04.xhtml#_idTextAnchor081）中，当我们使用 epoll 实现自己的类似 mio 的事件队列时，遇到了一些相当困难的地形。我们甚至不得不学习边缘触发和电平触发事件之间的区别。

            如果 *第四章*（B20892_04.xhtml#_idTextAnchor081）有些艰难，那么 *第五章*（B20892_05.xhtml#_idTextAnchor092）就像是攀登珠穆朗玛峰。没有人期望你记住那里涵盖的所有内容，但你阅读了它，并有一个可以用来实验的工作示例。我们实现了自己的纤程/绿色线程，在这个过程中，我们了解了一些关于处理器架构、ISAs、ABIs 和调用约定。我们甚至在 Rust 中学习了内联汇编。如果你曾经对栈与堆的区别感到不安全，那么在你创建了我们让 CPU 跳转到的栈之后，你现在肯定理解了。

            在 *第六章* 中，我们在深入探讨 *第七章*（B20892_07.xhtml#_idTextAnchor122）及以后的内容之前，对异步 Rust 进行了高级介绍，其中包括创建我们自己的协程和 `coroutine/wait` 语法。在 *第八章*（B20892_08.xhtml#_idTextAnchor138）中，我们在讨论基本运行时设计的同时创建了我们自己的运行时版本。我们还深入研究了反应器、执行器和唤醒器。

            在 *第九章*（B20892_09.xhtml#_idTextAnchor156）中，我们改进了我们的运行时，并发现了 Rust 中自引用结构体的危险。然后我们彻底研究了 Rust 中的 pinning 以及它如何帮助我们解决遇到的问题。

            最后，在 *第十章*（B20892_10.xhtml#_idTextAnchor178）中，我们看到了通过进行一些相当小的改动，我们的运行时变成了一个完整的 Rust futures 运行时。我们通过讨论异步 Rust 的一些已知挑战和对未来的期望来结束一切。

            Rust 社区非常包容和欢迎，如果你对这个主题感兴趣并想了解更多，我们很乐意欢迎你参与和贡献。异步 Rust 变得更好的一个方式是通过不同经验水平的人的贡献。如果你想参与其中，那么异步工作组（[`rust-lang.github.io/wg-async/welcome.html`](https://rust-lang.github.io/wg-async/welcome.html)）是一个不错的起点。还有一个围绕 Tokio 项目（[`github.com/tokio-rs/tokio/blob/master/CONTRIBUTING.md`](https://github.com/tokio-rs/tokio/blob/master/CONTRIBUTING.md)）的非常活跃的社区，以及许多其他社区，具体取决于你想深入研究哪个特定领域。不要害怕加入不同的频道并提问。

            现在我们已经到了最后，我想感谢你一直读到最后一页。我希望这本书能感觉像是我们共同经历的一次旅行，而不是一场讲座。我希望你是焦点，而不是我。

            我希望我做到了这一点，并且我真诚地希望你学到了一些你认为有用并且可以继续前进的知识。如果你做到了，那么我真心地为我所做的对你有价值而感到高兴。我祝愿你在异步编程的道路上一切顺利。

            下次再会！

            卡尔·弗雷德里克

```rs

```

```rs

```
