# 使用 Tokio 进行异步网络编程

在顺序编程模型中，代码总是按照编程语言的语义顺序执行。因此，如果某个操作由于某种原因（等待资源等）而阻塞，整个执行就会阻塞，并且只有在该操作完成后才能继续进行。这通常会导致资源利用率低下，因为主线程会忙于等待某个操作。在 GUI 应用程序中，这也导致了较差的用户交互性，因为负责管理 GUI 的主线程正忙于等待其他事情。在我们特定的网络编程案例中，这是一个主要问题，因为我们经常需要等待套接字上的数据可用。在过去，我们通过使用多个线程来解决这个问题。在那个模型中，我们将一个昂贵的操作委托给后台线程，使主线程可以用于用户交互或其他任务。相比之下，异步编程模型规定，任何操作都不应该阻塞。相反，应该有一种机制来检查它们是否已经完成，从主线程中进行检查。但我们如何实现这一点呢？一种简单的方法是让每个操作在自己的线程中运行，然后对所有这些线程进行连接。实际上，由于潜在的线程数量众多以及它们之间的协调，这种方法是麻烦的。

Rust 提供了一些 crate，支持使用基于 futures 的事件循环驱动模型进行异步编程。我们将在本章中详细研究这一点。以下是本章我们将涉及的主题：

+   Rust 中的未来抽象

+   使用 tokio 堆栈进行异步编程

# 展望未来

Rust 异步编程故事的核心是 futures crate。这个 crate 提供了一个名为*future*的结构。这本质上是一个操作结果的占位符。正如你所期望的，操作的结果可以是两种状态之一——要么操作仍在进行中，结果尚未可用，要么操作已完成，结果可用。请注意，在第二种情况下，可能发生错误，使得结果变得无关紧要。

该库提供了一个名为`Future`的特质（以及其他一些东西），任何类型都可以实现这个特质来能够像未来一样行动。这个特质看起来是这样的：

```rs
trait Future {
    type Item;
    type Error;
    fn poll(&mut self) -> Poll<Self::Item, Self::Error>;
    ...
}
```

在这里，`Item`指的是操作成功完成时返回的结果类型，而`Error`是操作失败时返回的类型。实现必须指定这些类型，并实现获取计算当前状态的`poll`方法。如果它已经完成，将返回结果。如果没有，未来将注册当前任务对给定操作的结果感兴趣。这个函数返回一个`Poll`，其外观如下：

```rs
type Poll<T, E> = Result<Async<T>, E>;
```

`Poll`被类型化为另一个名为`Async`（以及给定的错误类型）的类型的结果，该类型将在下面定义。

```rs
pub enum Async<T> {
    Ready(T),
    NotReady,
}
```

`Async`是一个枚举，可以是`Ready(T)`或`NotReady`。这两个最后的状态对应于操作的状态。因此，轮询函数可以返回三种可能的状态：

+   当操作成功完成且结果在内部变量`result`中时，会返回`Ok(Async::Ready(result))`。

+   当操作尚未完成且结果不可用时，会返回`Ok(Async::NotReady)`。请注意，这并不表示错误条件。

+   当操作遇到错误时，会返回`Err(e)`。在这种情况下，没有结果可用。

很容易注意到，`Future`本质上是一个`Result`，它可能还在运行以实际产生那个`Result`。如果移除`Result`可能在任何时间点未准备好的情况，我们只剩下两个选项：`Ok`和`Err`，它们正好对应于`Result`。

因此，一个`Future`可以代表任何需要非平凡时间才能完成的事情。这可以是一个网络事件、磁盘读取等等。现在，在这个阶段最常见的疑问是：我们如何从一个给定的函数中返回一个`Future`？有几种方法可以做到这一点。让我们在这里看一个例子。项目设置和以往一样。

```rs
$ cargo new --bin futures-example
```

我们需要在我们的 Cargo 配置中添加一些库，它看起来像这样：

```rs
[package]
name = "futures-example"
version = "0.1.0"
authors = ["Foo<foo@bar.com>"]

[dependencies]
futures = "0.1.17"
futures-cpupool = "0.1.7"
```

在我们的主文件中，我们像往常一样设置一切。我们感兴趣的是找出一个给定的整数是否为素数，这将代表我们操作中需要一些时间才能完成的部分。我们有两个函数，正是为了做到这一点。这两个函数使用两种不同的风格返回`Future`，我们稍后会看到。实际上，原始的素性测试方法并没有慢到足以成为一个好的例子。因此，我们不得不随机休眠一段时间来模拟缓慢。

```rs
// ch7/futures-example/src/main.rs

#![feature(conservative_impl_trait)]
extern crate futures;
extern crate futures_cpupool;

use std::io;
use futures::Future;
use futures_cpupool::CpuPool;

// This implementation returns a boxed future
fn check_prime_boxed(n: u64) -> Box<Future<Item = bool, Error = io::Error>> {
    for i in 2..n {
        if n % i == 0 { return Box::new(futures::future::ok(false)); }
    }
    Box::new(futures::future::ok(true))
}

// This returns a future using impl trait
fn check_prime_impl_trait(n: u64) -> impl Future<Item = bool, Error = io::Error> {
        for i in 2..n {
        if n % i == 0 { return futures::future::ok(false); }
    }
    futures::future::ok(true)
}

// This does not return a future
fn check_prime(n: u64) -> bool {
    for i in 2..n {
        if n % i == 0 { return false }
    }
    true
}

fn main() {
    let input: u64 = 58466453;
    println!("Right before first call");
    let res_one = check_prime_boxed(input);
    println!("Called check_prime_boxed");
    let res_two = check_prime_impl_trait(input);
    println!("Called check_prime_impl_trait");
    println!("Results are {} and {}", res_one.wait().unwrap(),
    res_two.wait().unwrap());

    let thread_pool = CpuPool::new(4);
    let res_three = thread_pool.spawn_fn(move || {
        let temp = check_prime(input);
        let result: Result<bool, ()> = Ok(temp);
        result
    });
    println!("Called check_prime in another thread");
    println!("Result from the last call: {}", res_three.wait().unwrap());
}
```

返回未来的主要方法有几种。第一种是使用特质对象，就像在 `check_prime_boxed` 中做的那样。现在，`Box` 是一个指向堆上对象的指针类型。它是一个受管理的指针，意味着当对象超出作用域时，对象将被自动清理。函数的返回类型是一个特质对象，它可以代表任何将 `Item` 设置为 bool 且将 `Error` 设置为 `io::Error` 的未来。因此，这代表了动态分派。返回未来的第二种方法是使用 `impl` 特质特性。在 `check_prime_impl_trait` 的情况下，我们就是这样做的。我们说该函数返回一个实现了 `Future<Item=bool, Error=io::Error>` 的类型，并且由于任何实现了 `Future` 特质的类型都是未来，我们的函数正在返回一个未来。请注意，在这种情况下，我们不需要在返回结果之前装箱。因此，这种方法的一个优点是返回未来不需要进行分配。我们的两个函数都使用 `future::ok` 函数来表示我们的计算已经成功完成，并给出了给定的结果。另一种选择是实际上不返回一个未来，而是使用基于未来的线程池 crate 来执行创建未来和管理未来的繁重工作。这就是 `check_prime` 的情况，它只返回一个 `bool`。在我们的主函数中，我们使用 futures-`cpupool` crate 设置了一个线程池，并在该池中运行最后一个函数。我们得到一个可以调用 `wait` 来获取结果的未来。为了达到相同的目标，还有一个完全不同的选项，即返回一个实现了 `Future` 特质的自定义类型。这是最不便捷的，因为它需要编写一些额外的代码，但它是最灵活的方法。

`impl` 特性目前还不是稳定特性。因此，`check_prime_impl_trait` 只能在 nightly Rust 上工作。

构建了一个未来之后，下一个目标是执行它。有三种方法可以做到这一点：

+   在当前线程中：这将导致当前线程被阻塞，直到未来完成执行。在我们之前的例子中，`res_one` 和 `res_two` 在主线程上执行，阻止了用户交互。

+   在线程池中：对于 `res_three` 来说，这是这种情况，它在名为 `thread_pool` 的线程池中执行。因此，在这种情况下，调用线程可以自由地继续自己的处理。

+   在事件循环中：在某些情况下，上述两种方法都不可能。那时唯一的选择是在事件循环中执行未来。方便的是，tokio-core crate 提供了面向未来的 API 来使用事件循环。我们将在下一节更深入地探讨这个模型。

在我们的主函数中，我们在主线程中调用了前两个函数。因此，它们将阻塞主线程的执行。然而，最后一个函数是在不同的线程上运行的。在这种情况下，主线程可以立即自由地打印出`check_prime`已被调用的信息。当在`future`上调用`wait`时，它再次被阻塞。请注意，在所有情况下，`future`都是惰性评估的。当我们运行这个程序时，我们应该看到以下内容：

```rs
$ cargo run
   Compiling futures-example v0.1.0 (file:///src/ch7/futures-example)
    Finished dev [unoptimized + debuginfo] target(s) in 0.77 secs
     Running `target/debug/futures-example`
Right before first call
Called check_prime_boxed
Called check_prime_impl_trait
Results are true and true
Called check_prime in another thread
Result from the last call: true
```

将`future`与常规线程区分开来的是，它们可以以符合人体工程学的方式链式连接。这就像说，*下载网页，然后解析 html，然后提取一个特定的单词*。这些串联步骤中的每一个都是一个`future`，下一个步骤不能开始，除非第一个步骤已经完成。整个操作也是一个`Future`，由多个组成部分`future`组成。当这个更大的`future`正在执行时，它被称为任务。该 crate 在`futures::task`命名空间中提供了一些用于与任务交互的 API。该库提供了一些函数来以这种方式处理`future`。当一个给定的类型实现了`Future`特质（实现了`poll`方法）时，编译器可以提供所有这些组合器的实现。让我们看看使用链式连接实现超时功能的示例。我们将使用`tokio-timer` crate 作为超时`future`，在我们的代码中，我们有两个相互竞争的函数，它们会随机休眠一段时间，然后向调用者返回一个固定的字符串。我们将同时调度所有这些函数，如果我们收到第一个函数对应的字符串，我们就宣布它获胜。同样，这也适用于第二个函数。如果我们都没有收到，我们知道超时`future`已经触发。让我们从项目设置开始：

```rs
$ cargo new --bin futures-chaining
```

然后，我们在`Cargo.toml`中添加我们的依赖项

```rs
[package]
name = "futures-chaining"
version = "0.1.0"
authors = ["Foo <foo@bar.com>"]

[dependencies]
tokio-timer = "0.1.2"
futures = "0.1.17"
futures-cpupool = "0.1.7"
rand = "0.3.18"
```

和上次一样，我们使用线程池来执行我们的`future`，使用`futures-cpupool` crate。让我们看看代码：

```rs
// ch7/futures-chaining/src/main.rs

extern crate futures;
extern crate futures_cpupool;
extern crate tokio_timer;
extern crate rand;

use futures::future::select_ok;
use std::time::Duration;

use futures::Future;
use futures_cpupool::CpuPool;
use tokio_timer::Timer;
use std::thread;
use rand::{thread_rng, Rng};

// First player, identified by the string "player_one"
fn player_one() -> &'static str {
    let d = thread_rng().gen_range::<u64>(1, 5);
    thread::sleep(Duration::from_secs(d));
    "player_one"
}

// Second player, identified by the string "player_two"
fn player_two() -> &'static str {
    let d = thread_rng().gen_range::<u64>(1, 5);
    thread::sleep(Duration::from_secs(d));
    "player_two"
}

fn main() {
    let pool = CpuPool::new_num_cpus();
    let timer = Timer::default();

    // Defining the timeout future
    let timeout = timer.sleep(Duration::from_secs(3))
        .then(|_| Err(()));

    // Running the first player in the pool
    let one = pool.spawn_fn(|| {
        Ok(player_one())
    });

    // Running second player in the pool
    let two = pool.spawn_fn(|| {
        Ok(player_two())
    });

    let tasks = vec![one, two];
    // Combining the players with the timeout future
    // and filtering out result
    let winner = select_ok(tasks).select(timeout).map(|(result, _)|
    result);
    let result = winner.wait().ok();
    match result {
        Some(("player_one", _)) => println!("Player one won"),
        Some(("player_two", _)) => println!("Player two won"),
        Some((_, _)) | None => println!("Timed out"),
    }
}
```

我们的两个玩家非常相似；它们都生成一个介于 `1` 和 `5` 之间的随机数，并睡眠相应的时间。之后，它们返回一个与它们名称对应的固定字符串。我们稍后会使用这些字符串来唯一标识它们。在我们的主函数中，我们初始化线程池和计时器。我们使用计时器上的组合器来返回一个在 `3` 秒后出错的未来。然后我们在线程池中启动两个玩家，并从这些玩家返回 `Result` 作为未来。请注意，这些函数现在实际上并没有真正运行，因为 futures 是懒加载的。然后我们将这些未来放入一个列表中，并使用 `select_ok` 组合器并行运行这些。这个函数接受一个未来的可迭代集合，并选择第一个成功的未来；这里的唯一限制是传递给此函数的所有未来都应该属于同一类型。因此，我们不能将超时未来传递到这里。我们使用接受两个未来的 `select` 组合器将 `select_ok` 的结果与超时未来链接起来，该组合器等待任一完成执行。结果未来将包含已经完成和尚未完成的部分。然后我们使用 `map` 组合器丢弃第二部分。最后，我们阻塞在未来的上，并使用 `ok()` 信号结束链。然后我们可以将结果与已知的字符串进行比较，以确定哪个未来获胜，并相应地打印出消息。

这是一些运行的结果。由于我们的超时时间小于两个函数中任一的最大睡眠时间，我们应该看到一些超时。每当一个函数选择的时间小于超时时间时，它就有机会获胜。

```rs
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/futures-chaining`
Player two won
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/futures-chaining`
Player one won
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/futures-chaining`
Timed out
```

# 与流和汇一起工作

futures crate 提供了另一个用于懒加载一系列事件的有用抽象，称为 `Stream`。如果 `Future` 对应于 `Result`，那么 `Stream` 就对应于 `Iterator`。从语义上看，它们与 futures 非常相似，看起来是这样的：

```rs
trait Stream {
    type Item;
    type Error;
    fn poll(& mut self) -> Poll<Option<Self::Item>, Self::Error>;
    ...
}
```

这里的唯一区别是返回类型被包裹在一个 `Option` 中，就像 `Iterator` 特性一样。因此，这里的 `None` 会表示流已终止。此外，所有流都是未来，可以使用 `into_future` 转换。让我们看看使用这个构造的一个例子。我们将部分重用之前章节中的 collatz 示例。第一步是设置项目：

```rs
$ cargo new --bin streams
```

添加了所有依赖项后，我们的 Cargo 配置看起来如下：

```rs
[package]
name = "streams"
version = "0.1.0"
authors = ["Foo<foo@bar.com>"]

[dependencies]
futures = "0.1.17"
rand = "0.3.18"
```

在设置好一切之后，我们的主文件将如下所示。在这种情况下，我们有一个名为 `CollatzStream` 的结构体，它有两个字段用于当前状态和结束状态（始终为 `1`）。我们将在这个结构体上实现 `Stream` 特性，使其表现得像一个流：

```rs
// ch7/streams/src/main.rs

extern crate futures;
extern crate rand;

use std::{io, thread};
use std::time::Duration;
use futures::stream::Stream;
use futures::{Poll, Async};
use rand::{thread_rng, Rng};
use futures::Future;

// This struct holds the current state and the end condition
// for the stream
#[derive(Debug)]
struct CollatzStream {
    current: u64,
    end: u64,
}

// A constructor to initialize the struct with defaults
impl CollatzStream {
    fn new(start: u64) -> CollatzStream {
        CollatzStream {
            current: start,
            end: 1
        }
    }
}

// Implementation of the Stream trait for our struct
impl Stream for CollatzStream {
    type Item = u64;
    type Error = io::Error;
    fn poll(&mut self) -> Poll<Option<Self::Item>, io::Error> {
        let d = thread_rng().gen_range::<u64>(1, 5);
        thread::sleep(Duration::from_secs(d));
        if self.current % 2 == 0 {
            self.current = self.current / 2;
        } else {
            self.current = 3 * self.current + 1;
        }
        if self.current == self.end {
            Ok(Async::Ready(None))
        } else {
            Ok(Async::Ready(Some(self.current)))
        }
    }
}

fn main() {
    let stream = CollatzStream::new(10);
    let f = stream.for_each(|num| {
        println!("{}", num);
        Ok(())
    });
    f.wait().ok();
}
```

我们通过在 `1` 到 `5` 秒之间随机暂停来模拟返回结果的延迟。在我们的轮询实现中，当达到 `1` 时，我们返回 `Ok(Async::Ready(None))` 来表示流已结束。否则，我们返回当前状态作为 `Ok(Async::Ready(Some(self.current)))`。很容易注意到，除了流语义外，这个实现与迭代器的实现相同。在我们的主函数中，我们初始化结构体并使用 `for_each` 组合子来打印流中的每个项目。这个组合子返回一个未来，我们在其上调用 `wait` 和 `ok` 来阻塞并获取所有结果。以下是运行最后一个示例时我们看到的内容：

```rs
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/streams`
5
16
8
4
2
```

就像 `Future` 特性一样，`Stream` 特性也支持许多其他组合子，这些组合子适用于不同的目的。`Stream` 的对偶是 `Sink`，它是异步事件的接收者。这在模拟 Rust 通道的发送端、网络套接字、文件描述符等时非常有用。

在任何异步系统中，一个常见的模式是同步。这变得很重要，因为组件通常需要相互通信以传递数据或协调任务。我们过去使用通道解决了这个确切的问题。但是，这些构造在这里不适用，因为标准库中的通道实现不是异步的。因此，futures 有自己的通道实现，它提供了您从异步系统期望的所有保证。让我们看看一个例子；我们的项目设置应该如下所示：

```rs
$ cargo new --bin futures-ping-pong
```

Cargo 配置应该如下所示：

```rs
[package]
name = "futures-ping-pong"
version = "0.1.0"
authors = ["Foo<foo@bar.com>"]

[dependencies]
futures = "0.1"
tokio-core = "0.1"
rand = "0.3.18"
```

现在我们有两个函数。一个函数会等待随机的时间，然后随机返回 `"ping"` 或 `"pong"`。这个函数将是我们的发送者。以下是它的样子：

```rs
// ch7/futures-ping-pong/src/main

extern crate futures;
extern crate rand;
extern crate tokio_core;

use std::thread;
use std::fmt::Debug;
use std::time::Duration;
use futures::Future;
use rand::{thread_rng, Rng};

use futures::sync::mpsc;
use futures::{Sink, Stream};
use futures::sync::mpsc::Receiver;

// Randomly selects a sleep duration between 1 and 5 seconds. Then
// randomly returns either "ping" or "pong"
fn sender() -> &'static str {
    let mut d = thread_rng();
    thread::sleep(Duration::from_secs(d.gen_range::<u64>(1, 5)));
    d.choose(&["ping", "pong"]).unwrap()
}

// Receives input on the given channel and prints each item
fn receiver<T: Debug>(recv: Receiver<T>) {
    let f = recv.for_each(|item| {
        println!("{:?}", item);
        Ok(())
    });
    f.wait().ok();
}

fn main() {
    let (tx, rx) = mpsc::channel(100);
    let h1 = thread::spawn(|| {
        tx.send(sender()).wait().ok();
    });
    let h2 = thread::spawn(|| {
        receiver::<&str>(rx);
    });
    h1.join().unwrap();
    h2.join().unwrap();
}
```

futures 库提供了两种类型的通道：一种是一次性使用的 *oneshot* 通道，可以用来发送和接收任何消息，还有一种是可以多次使用的常规 *mpsc* 通道。在我们的主函数中，我们获取通道的两端，并在另一个线程中将发送者作为未来启动。接收者也在另一个线程中启动。在这两种情况下，我们记录处理程序以便稍后等待它们完成（使用 `join`）。请注意，我们的接收者将通道的接收端作为参数。因为 `Receiver` 实现了 `Stream`，我们可以使用 `and_then` 组合子在它上面来打印值。最后，我们在退出接收函数之前在未来的 `wait()` 和 `ok()` 上调用。在主函数中，我们通过两个线程处理程序来连接它们，以驱动它们完成。

运行最后一个示例将随机打印 `"ping"` 或 `"pong"`，具体取决于通过通道发送的内容。请注意，实际的打印操作发生在接收端。

```rs
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/futures-ping-pong`
"ping"
```

`futures` crate 还在 `futures::sync::BiLock` 中提供了一个锁定机制，这与 `std::sync::Mutex` 非常相似。这是一个面向未来的互斥锁，用于在两个所有者之间仲裁资源共享。请注意，`BiLock` 只适用于两个未来，这是一个令人烦恼的限制。以下是它是如何工作的：我们感兴趣的是修改我们的最后一个示例，以在发送函数被调用时显示一个计数器。现在我们的计数器需要是线程安全的，以便可以在消费者之间共享。使用 Cargo 设置项目：

```rs
$ cargo new --bin future-bilock
```

我们的 `Cargo.toml` 文件应该完全相同，以下是主文件的外观：

```rs
// ch7/future-bilock/src/main.rs

extern crate futures;
extern crate rand;

use std::thread;
use std::fmt::Debug;
use std::time::Duration;
use futures::{Future, Async};
use rand::{thread_rng, Rng};

use futures::sync::{mpsc, BiLock};
use futures::{Sink, Stream};
use futures::sync::mpsc::Receiver;

// Increments the shared counter if it can acquire a lock, then
// sleeps for a random duration between 1 and 5 seconds, then
// randomly returns either "ping" or "pong"
fn sender(send: &BiLock<u64>) -> &'static str {
    match send.poll_lock() {
        Async::Ready(mut lock) => *lock += 1,
        Async::NotReady => ()
    }
    let mut d = thread_rng();
    thread::sleep(Duration::from_secs(d.gen_range::<u64>(1, 5)));
    d.choose(&["ping", "pong"]).unwrap()
}

// Tries to acquire a lock on the shared variable and prints it's
// value if it got the lock. Then prints each item in the given
// stream
fn receiver<T: Debug>(recv: Receiver<T>, recv_lock: BiLock<u64>) {
    match recv_lock.poll_lock() {
        Async::Ready(lock) => println!("Value of lock {}", *lock),
        Async::NotReady => ()
    }
    let f = recv.for_each(|item| {
        println!("{:?}", item);
        Ok(())
    });
    f.wait().ok();
}

fn main() {
    let counter = 0;
    let (send, recv) = BiLock::new(counter);
    let (tx, rx) = mpsc::channel(100);
    let h1 = thread::spawn(move || {
        tx.send(sender(&send)).wait().ok();
    });
    let h2 = thread::spawn(|| {
        receiver::<&str>(rx, recv);
    });
    h1.join().unwrap();
    h2.join().unwrap();
}
```

虽然这基本上与上一个示例相同，但有一些区别。在我们的主函数中，我们将计数器设置为零。然后我们在计数器上创建一个 `BiLock`。构造函数返回两个类似于通道的句柄，然后我们可以将它们传递出去。然后我们创建我们的通道并启动发送者。现在，如果我们查看发送者，它已经被修改为接受一个 `BiLock` 的引用。在该函数中，我们尝试使用 `poll_lock` 获取锁，如果成功，我们增加计数器。否则，我们不做任何事情。然后我们继续我们通常的业务，返回 `"ping"` 或 `"pong"`。接收者也被修改为接受一个 `BiLock`。在那里，我们尝试获取锁，如果成功，我们打印出被锁定数据的值。在我们的主函数中，我们使用线程启动这些 futures，并在它们上等待它们完成。

这是一个在尝试获取锁失败时发生的情况，当双方都未能获取锁。在真实示例中，我们希望优雅地处理错误并重试。我们为了简洁起见省略了这部分内容：

```rs
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/futures-bilock`
thread '<unnamed>' panicked at 'no Task is currently running', libcore/option.rs:917:5
note: Run with `RUST_BACKTRACE=1` for a backtrace.
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: Any', libcore/result.rs:945:5
```

这是一个良好的运行示例：

```rs
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/futures-bilock`
Value of lock 1
"pong"
```

# 正在前往 tokio

tokio 生态系统是 Rust 中网络堆栈的一个实现。它具有标准库的所有主要功能，主要区别在于它是非阻塞的（大多数常见调用不会阻塞当前线程）。这是通过使用 mio 来完成所有底层繁重工作，并使用 futures 来抽象长运行操作来实现的。该生态系统有两个基本 crate，其他所有内容都是围绕这些构建的：

+   `tokio-proto` 提供了构建异步服务器和客户端的原语。这严重依赖于 mio 进行底层网络和 futures 进行抽象。

+   `tokio-core` 提供了一个事件循环来运行 futures，以及一些相关的 API。当应用程序需要精细控制 IO 时，这很有用。

正如我们在上一节中提到的，运行 futures 的一种方法是在事件循环上。事件循环（在 tokio 中称为 `reactor`）是一个无限循环，它监听定义的事件，并在接收到一个事件时采取适当的行动。以下是它是如何工作的：我们将借用我们之前的示例，该示例确定给定的输入是否为素数。这返回一个包含结果的 future，然后我们将其打印出来。项目设置与以往相同：

```rs
$ cargo new --bin futures-loop
```

这里是`Cargo.toml`应该看起来像什么：

```rs
[package]
name = "futures-loop"
version = "0.1.0"
authors = ["Foo<foo@bar.com>"]

[dependencies]
futures = "0.1"
tokio-core = "0.1"
```

对于这个例子，我们将在一个无限循环中接收输入。对于每个输入，我们将去除换行符和空格，并尝试将其解析为`u64`。看起来是这样的：

```rs
// ch7/futures-loop/src/main.rs

extern crate futures;
extern crate tokio_core;

use std::io;
use std::io::BufRead;
use futures::Future;
use tokio_core::reactor::Core;

fn check_prime_boxed(n: u64) -> Box<Future<Item = bool, Error = io::Error>> {
    for i in 2..n {
        if n % i == 0 {
            return Box::new(futures::future::ok(false));
        }
    }
    Box::new(futures::future::ok(true))
}

fn main() {
    let mut core = Core::new().expect("Could not create event loop");
    let stdin = io::stdin();

    loop {
        let mut line = String::new();
        stdin
            .lock()
            .read_line(&mut line)
            .expect("Could not read from stdin");
        let input = line.trim()
            .parse::<u64>()
            .expect("Could not parse input as u64");
        let result = core.run(check_prime_boxed(input))
            .expect("Could not run future");
        println!("{}", result);
    }
}
```

在我们的主函数中，我们创建核心并启动我们的无限循环。我们使用核心的`run`方法来启动一个任务以异步执行未来。结果被收集并在标准输出上打印。一个会话应该看起来像这样：

```rs
$ cargo run
12
false
13
true
991
true
```

`tokio-proto`是一个异步服务器（和客户端）构建工具包。任何使用此工具包的服务器都有以下三个不同的层：

+   一个编解码器，它决定了如何从底层套接字读取和写入数据，形成我们协议的传输层。随后，这一层是最低的（最接近物理介质）。在实践中，编写编解码器相当于实现库中的一些给定特质，这些特质处理字节流。

+   一个协议位于编解码器之上和实际运行协议的事件循环之下。这充当粘合剂，将它们绑定在一起。tokio 支持多种协议类型，具体取决于应用程序：一个简单的请求-响应类型协议、一个多路复用协议和一个流协议。我们将很快深入探讨这些。

+   一个实际运行所有这些操作作为未来的服务。由于这只是一个未来，一个简单的方式来思考它是一个将输入转换为最终响应（可能是错误）的异步函数。在实践中，大部分计算都在这一层完成。

由于层是可互换的，实现可以完全自由地交换协议类型、服务或编解码器。让我们看看一个使用`tokio-proto`的简单服务的例子。这是一个传统的请求-响应服务，提供基于文本的接口。它接收一个数字并返回其 Collatz 序列作为数组。如果输入不是一个有效的整数，它将返回一条消息指出这一点。我们的项目设置相当简单：

```rs
$ cargo new --bin collatz-proto
```

Cargo 配置看起来像以下示例：

```rs
[package]
name = "collatz-proto"
version = "0.1.0"
authors = ["Foo<foo@bar.com>"]

[dependencies]
bytes = "0.4"
futures = "0.1"
tokio-io = "0.1"
tokio-core = "0.1"
tokio-proto = "0.1"
tokio-service = "0.1"
```

如前所述，我们需要实现不同的层。在我们的当前情况下，我们的每一层都不需要保留太多状态。因此，它们可以用单元结构体来表示。如果不是这种情况，我们就需要在这些层中放入一些数据。

```rs
// ch7/collatz-proto/src/main.rs

extern crate bytes;
extern crate futures;
extern crate tokio_io;
extern crate tokio_proto;
extern crate tokio_service;

use std::io;
use std::str;
use bytes::BytesMut;
use tokio_io::codec::{Encoder, Decoder};
use tokio_io::{AsyncRead, AsyncWrite};
use tokio_io::codec::Framed;
use tokio_proto::pipeline::ServerProto;
use tokio_service::Service;
use futures::{future, Future};
use tokio_proto::TcpServer;

// Codec implementation, our codec is a simple unit struct
pub struct CollatzCodec;

// Decoding a byte stream from the underlying socket
impl Decoder for CollatzCodec {
    type Item = String;
    type Error = io::Error;

    fn decode(&mut self, buf: &mut BytesMut) -> io::Result<Option<String>> {
        // Since a newline denotes end of input, read till a newline
        if let Some(i) = buf.iter().position(|&b| b == b'\n') {
            let line = buf.split_to(i);
            // and remove the newline
            buf.split_to(1);
            // try to decode into an UTF8 string before passing
            // to the protocol
            match str::from_utf8(&line) {
                Ok(s) => Ok(Some(s.to_string())),
                Err(_) => Err(io::Error::new(io::ErrorKind::Other, 
                "invalid UTF-8")),
            }
        } else {
            Ok(None)
        }
    }
}

// Encoding a string to a newline terminated byte stream
impl Encoder for CollatzCodec {
    type Item = String;
    type Error = io::Error;

    fn encode(&mut self, msg: String, buf: &mut BytesMut) -> 
    io::Result<()> {
        buf.extend(msg.as_bytes());
        buf.extend(b"\n");
        Ok(())
    }
}

// Protocol implementation as an unit struct
pub struct CollatzProto;

impl<T: AsyncRead + AsyncWrite + 'static> ServerProto<T> for CollatzProto {
    type Request = String;
    type Response = String;
    type Transport = Framed<T, CollatzCodec>;
    type BindTransport = Result<Self::Transport, io::Error>;
    fn bind_transport(&self, io: T) -> Self::BindTransport {
        Ok(io.framed(CollatzCodec))
    }
}

// Service implementation
pub struct CollatzService;

fn get_sequence(n: u64) -> Vec<u64> {
    let mut n = n.clone();
    let mut result = vec![];
    result.push(n);
    while n > 1 {
        if n % 2 == 0 {
            n /= 2;
        } else {
            n = 3 * n + 1;
        }
        result.push(n);
    }
    result
}

impl Service for CollatzService {
    type Request = String;
    type Response = String;
    type Error = io::Error;
    type Future = Box<Future<Item = Self::Response, Error = Self::Error>>;

    fn call(&self, req: Self::Request) -> Self::Future {
        match req.trim().parse::<u64>() {
            Ok(num) => {
                let res = get_sequence(num);
                Box::new(future::ok(format!("{:?}", res)))
            }
            Err(_) => Box::new(future::ok("Could not parse input as an
            u64".to_owned())),
        }
    }
}

fn main() {
    let addr = "0.0.0.0:9999".parse().unwrap();
    let server = TcpServer::new(CollatzProto, addr);
    server.serve(|| Ok(CollatzService));
}
```

如前所述，第一步是告诉编解码器如何从套接字读取数据。这是通过从`tokio_io::codec`实现`Encoder`和`Decoder`来完成的。请注意，我们在这里不需要处理原始套接字；我们得到一个字节流作为输入，我们可以自由地处理它。根据我们之前定义的协议，换行符表示输入的结束。因此，在我们的解码器中，我们读取到换行符，然后返回去除该换行符后的数据作为一个 UTF-8 编码的字符串。在发生错误的情况下，我们返回`None`。

`Encoder`实现正好相反：它将字符串转换成字节流。下一步是协议定义，这个协议非常简单，因为它既不做多路复用也不做流处理。我们实现`bind_transport`将编解码器绑定到我们的原始套接字，我们稍后会得到它。这里的唯一问题是这里的`Request`和`Response`类型应该与编解码器匹配。设置好这些之后，下一步是实现服务，通过声明一个单元结构体并在其上实现`Service`特质。我们的辅助函数`get_sequence`根据输入的`u64`返回柯朗序列。`Service`中的`call`方法实现了计算响应的逻辑。我们将输入解析为`u64`（记住，我们的编解码器将输入作为字符串返回）。如果没有出错，我们调用我们的辅助函数并返回一个静态字符串作为结果，否则返回一个错误。我们的主函数看起来与使用标准网络类型的函数类似，但我们使用 tokio 的`TcpServer`类型，它接受我们的套接字（将其绑定到编解码器）和我们的协议定义。最后，我们调用`serve`方法，并将我们的服务作为闭包传递。这个方法负责管理事件循环并在退出时清理事物。

让我们使用`telnet`与之交互。下面是一个会话的示例：

```rs
$ telnet localhost 9999
Connected to localhost.
Escape character is '^]'.
12
[12, 6, 3, 10, 5, 16, 8, 4, 2, 1]
30
[30, 15, 46, 23, 70, 35, 106, 53, 160, 80, 40, 20, 10, 5, 16, 8, 4, 2, 1]
foobar
Could not parse input as an u64
```

和往常一样，编写一个客户端来与我们的服务器交互将更有用。我们将从在事件循环中运行 future 的示例中借用很多内容。我们首先设置我们的项目：

```rs
$ cargo new --bin collatz-client
```

我们的货物设置将看起来像这样：

```rs
[package]
name = "collatz-client"
version = "0.1.0"
authors = ["Abhishek Chanda <abhishek.becs@gmail.com>"]

[dependencies]
futures = "0.1"
tokio-core = "0.1"
tokio-io = "0.1"
```

下面是我们的主文件：

```rs
// ch7/collatz-client/src/main.rs

extern crate futures;
extern crate tokio_core;
extern crate tokio_io;

use std::net::SocketAddr;
use std::io::BufReader;
use futures::Future;
use tokio_core::reactor::Core;
use tokio_core::net::TcpStream;

fn main() {
    let mut core = Core::new().expect("Could not create event loop");
    let handle = core.handle();
    let addr: SocketAddr = "127.0.0.1:9999".parse().expect("Could not parse as SocketAddr");
    let socket = TcpStream::connect(&addr, &handle);
    let request = socket.and_then(|socket| {
        tokio_io::io::write_all(socket, b"110\n")
    });
    let response = request.and_then(|(socket, _request)| {
        let sock = BufReader::new(socket);
        tokio_io::io::read_until(sock, b'\n', Vec::new())
    });
    let (_socket, data) = core.run(response).unwrap();
    println!("{}", String::from_utf8_lossy(&data));
}
```

当然，这个设置会将单个整数发送到服务器（十进制中的 110），但将这个操作放入循环以读取输入并发送这些数据是微不足道的。我们将这个任务留给读者去练习。在这里，我们创建了一个事件循环并获取其句柄。然后，我们使用异步的`TcpStream`实现连接到给定地址的服务器。这会返回一个 future，我们使用`and_then`将其与闭包结合以写入给定的套接字。整个结构返回一个新的 future，称为`request`，它与一个读取 future 链在一起。最终的 future 称为`response`，并在事件循环上运行。最后，我们读取响应并将其打印出来。在每一步，我们必须遵守我们的协议，即换行符表示服务器和客户端的输入结束。下面是一个会话的示例：

```rs
$ cargo run
   Compiling futures-loop v0.1.0 (file:///src/ch7/collatz-client)
    Finished dev [unoptimized + debuginfo] target(s) in 0.94 secs
     Running `target/debug/futures-loop`
[110, 55, 166, 83, 250, 125, 376, 188, 94, 47, 142, 71, 214, 107, 322, 161, 484, 242, 121, 364, 182, 91, 274, 137, 412, 206, 103, 310, 155, 466, 233, 700, 350, 175, 526, 263, 790, 395, 1186, 593, 1780, 890, 445, 1336, 668, 334, 167, 502, 251, 754, 377, 1132, 566, 283, 850, 425, 1276, 638, 319, 958, 479, 1438, 719, 2158, 1079, 3238, 1619, 4858, 2429, 7288, 3644, 1822, 911, 2734, 1367, 4102, 2051, 6154, 3077, 9232, 4616, 2308, 1154, 577, 1732, 866, 433, 1300, 650, 325, 976, 488, 244, 122, 61, 184, 92, 46, 23, 70, 35, 106, 53, 160, 80, 40, 20, 10, 5, 16, 8, 4, 2, 1]
```

# tokio 中的套接字多路复用

服务器中异步请求处理的一种模式是通过多路复用传入的连接。在这种情况下，每个连接被分配一个某种类型的唯一 ID，并且每当有一个准备好时就会发出回复，而不考虑它接收的顺序。因此，这允许更高的吞吐量，因为最短的任务隐含地获得了最高的优先级。这种模型也使得服务器在面对大量不同复杂度的传入请求时具有高度的响应性。传统的类 Unix 系统使用 select 和 poll 系统调用来支持套接字多路复用。

在 tokio 生态系统中，这通过一系列特性来实现，这些特性使得实现多路复用协议成为可能。服务器的基本结构与简单服务器相同：我们有编解码器、使用编解码器的协议，以及实际运行协议的服务。这里唯一的区别是我们将给每个传入的请求分配一个请求 ID。这将用于稍后发送响应时的去歧义。我们还需要实现`tokio_proto::multiplex`命名空间中的一些特性。作为一个例子，我们将修改我们的 collatz 服务器并为其添加多路复用功能。在这个情况下，我们的项目设置略有不同，因为我们计划使用 Cargo 运行二进制文件，并且我们的项目将是一个库。我们是这样设置的：

```rs
$ cargo new collatz-multiplexed
```

Cargo 配置类似：

```rs
[package]
name = "collatz-multiplexed"
version = "0.1.0"
authors = ["Foo <foo@bar.com>"]

[dependencies]
bytes = "0.4"
futures = "0.1"
tokio-io = "0.1"
tokio-core = "0.1"
tokio-proto = "0.1"
tokio-service = "0.1"
```

下面是`lib.rs`文件的样子：

```rs
// ch7/collatz-multiplexed/src/lib.rs

extern crate bytes;
extern crate futures;
extern crate tokio_core;
extern crate tokio_io;
extern crate tokio_proto;
extern crate tokio_service;

use futures::{future, Future};

use tokio_io::{AsyncRead, AsyncWrite};
use tokio_io::codec::{Decoder, Encoder, Framed};
use tokio_core::net::TcpStream;
use tokio_core::reactor::Handle;
use tokio_proto::TcpClient;
use tokio_proto::multiplex::{ClientProto, ClientService, RequestId, ServerProto};
use tokio_service::Service;

use bytes::{BigEndian, Buf, BufMut, BytesMut};

use std::{io, str};
use std::net::SocketAddr;

// Everything client side
// Represents a client connecting to our server
pub struct Client {
    inner: ClientService<TcpStream, CollatzProto>,
}

impl Client {
    pub fn connect(
        addr: &SocketAddr,
        handle: &Handle,
    ) -> Box<Future<Item = Client, Error = io::Error>> {
        let ret = TcpClient::new(CollatzProto)
            .connect(addr, handle)
            .map(|service| Client {
                inner: service,
            });

        Box::new(ret)
    }
}

impl Service for Client {
    type Request = String;
    type Response = String;
    type Error = io::Error;
    type Future = Box<Future<Item = String, Error = io::Error>>;

    fn call(&self, req: String) -> Self::Future {
        Box::new(self.inner.call(req).and_then(move |resp| Ok(resp)))
    }
}

// Everything server side
pub struct CollatzCodec;
pub struct CollatzProto;

// Represents a frame that has a RequestId and the actual data (String)
type CollatzFrame = (RequestId, String);

impl Decoder for CollatzCodec {
    type Item = CollatzFrame;
    type Error = io::Error;

    fn decode(&mut self, buf: &mut BytesMut) -> 
    Result<Option<CollatzFrame>, io::Error> {
        // Do not proceed if we haven't received at least 6 bytes yet
        // 4 bytes for the RequestId + data + 1 byte for newline
        if buf.len() < 5 {
            return Ok(None);
        }
        let newline = buf[4..].iter().position(|b| *b == b'\n');
        if let Some(n) = newline {
            let line = buf.split_to(n + 4);
            buf.split_to(1);
            let request_id = io::Cursor::new(&line[0..4]).get_u32:
            :<BigEndian>();
            return match str::from_utf8(&line.as_ref()[4..]) {
                Ok(s) => Ok(Some((u64::from(request_id),
                s.to_string()))),
                Err(_) => Err(io::Error::new(io::ErrorKind::Other, 
                "invalid string")),
            };
        }
        // Frame is not complete if it does not have a newline at the
        end
        Ok(None)
    }
}

impl Encoder for CollatzCodec {
    type Item = CollatzFrame;
    type Error = io::Error;

    fn encode(&mut self, msg: CollatzFrame, buf: &mut BytesMut) ->
    io::Result<()> {
        // Calculate final message length first
        let len = 4 + msg.1.len() + 1;
        buf.reserve(len);

        let (request_id, msg) = msg;

        buf.put_u32::<BigEndian>(request_id as u32);
        buf.put_slice(msg.as_bytes());
        buf.put_u8(b'\n');

        Ok(())
    }
}

impl<T: AsyncRead + AsyncWrite + 'static> ClientProto<T> for CollatzProto {
    type Request = String;
    type Response = String;

    type Transport = Framed<T, CollatzCodec>;
    type BindTransport = Result<Self::Transport, io::Error>;

    fn bind_transport(&self, io: T) -> Self::BindTransport {
        Ok(io.framed(CollatzCodec))
    }
}

impl<T: AsyncRead + AsyncWrite + 'static> ServerProto<T> for CollatzProto {
    type Request = String;
    type Response = String;

    type Transport = Framed<T, CollatzCodec>;
    type BindTransport = Result<Self::Transport, io::Error>;

    fn bind_transport(&self, io: T) -> Self::BindTransport {
        Ok(io.framed(CollatzCodec))
    }
}

pub struct CollatzService;

fn get_sequence(mut n: u64) -> Vec<u64> {
    let mut result = vec![];
    result.push(n);
    while n > 1 {
        if n % 2 == 0 {
            n /= 2;
        } else {
            n = 3 * n + 1;
        }
        result.push(n);
    }
    result
}

impl Service for CollatzService {
    type Request = String;
    type Response = String;
    type Error = io::Error;
    type Future = Box<Future<Item = Self::Response, Error = Self::Error>>;

    fn call(&self, req: Self::Request) -> Self::Future {
        match req.trim().parse::<u64>() {
            Ok(num) => {
                let res = get_sequence(num);
                Box::new(future::ok(format!("{:?}", res)))
            }
            Err(_) => Box::new(future::ok("Could not parse input as an
            u64".to_owned())),
        }
    }
}
```

Tokio 提供了一个内置类型`RequestId`来表示传入请求的唯一 ID，所有与其相关的状态都由 tokio 内部管理。我们定义了一个名为`CollatzFrame`的自定义数据类型来表示我们的帧；它包含`RequestId`和用于我们数据的`String`。我们继续实现`Decoder`和`Encoder`，就像上次一样，为`CollatzCodec`。但是，在这两种情况下，我们必须考虑到头部中的请求 ID 和尾随的换行符。因为`RequestId`类型在底层是`u64`，它总是占用四个字节，再加上一个换行符的字节。因此，如果我们收到了少于 5 个字节，我们知道整个帧还没有收到。请注意，这并不是一个错误情况，帧仍在传输中，所以我们返回`Ok(None)`。然后我们检查缓冲区是否有换行符（符合我们的协议）。如果一切看起来都很好，我们从前 4 个字节解析请求 ID（请注意，这将是在网络字节序）。然后我们构造一个`CollatzFrame`实例并返回它。编码器的实现是相反的；我们只需要将请求 ID 放回，然后是实际的数据，最后以换行符结束。

下一步是实现`ServerProto`和`ClientProto`的`CollatzProto`；这两个都是绑定编解码器与传输的样板代码。像上次一样，最后一步是实现服务。这一步没有任何变化。请注意，在实现编解码器后，我们不需要关心处理请求 ID，因为后续阶段根本看不到它。编解码器处理并管理它，同时将实际数据传递到后续层。

这是我们帧的外观：

![图片](img/00016.jpeg)

我们的请求帧，其中包含作为头部的 RequestId 和一个尾随换行符

这次，我们的客户端也将基于 tokio。我们的`Client`结构体封装了一个`ClientService`实例，它接受底层的 TCP 流和要使用的协议实现。我们有一个名为`connect`的便利函数，用于`Client`类型，它连接到指定的服务器并返回一个 future。最后，我们在`Client`中实现`Service`，其中`call`方法返回一个 future。我们将服务器和客户端作为示例放入名为`examples`的目录中。这样，cargo 就知道这些应该作为与此 crate 关联的示例运行。服务器看起来是这样的：

```rs
// ch7/collatz-multiplexed/examples/server.rs

extern crate collatz_multiplexed as collatz;
extern crate tokio_proto;

use tokio_proto::TcpServer;
use collatz::{CollatzService, CollatzProto};

fn main() {
    let addr = "0.0.0.0:9999".parse().unwrap();
    TcpServer::new(CollatzProto, addr).serve(|| Ok(CollatzService));
}
```

这基本上和上次一样，只是在一个不同的文件中。我们必须将父 crate 声明为外部依赖，这样 Cargo 才能正确地链接一切。这是客户端的外观：

```rs
// ch7/collatz-multiplexed/examples/client.rs

extern crate collatz_multiplexed as collatz;

extern crate futures;
extern crate tokio_core;
extern crate tokio_service;

use futures::Future;
use tokio_core::reactor::Core;
use tokio_service::Service;

pub fn main() {
    let addr = "127.0.0.1:9999".parse().unwrap();
    let mut core = Core::new().unwrap();
    let handle = core.handle();

    core.run(
        collatz::Client::connect(&addr, &handle)
            .and_then(|client| {
                client.call("110".to_string())
                    .and_then(move |response| {
                        println!("We got back: {:?}", response);
                        Ok(())
                    })
            })
    ).unwrap();
}
```

我们在事件循环中运行我们的客户端，使用 tokio-core。我们使用客户端上定义的`connect`方法获取一个封装连接的 future。我们使用`and_then`组合器，并使用`call`方法向服务器发送一个字符串。由于此方法也返回一个 future，我们可以在内部 future 上使用`and_then`组合器来提取响应，然后通过返回`Ok(())`来解析它。这也解析了外部 future。

现在，如果我们打开两个终端，在一个终端中运行服务器，在另一个终端中运行客户端，我们应该在客户端看到以下内容。请注意，由于我们没有复杂的重试和错误处理，服务器应该在客户端之前运行：

```rs
$ cargo run --example client
   Compiling collatz-multiplexed v0.1.0 (file:///src/ch7/collatz-multiplexed)
    Finished dev [unoptimized + debuginfo] target(s) in 0.93 secs
     Running `target/debug/examples/client`
We got back: "[110, 55, 166, 83, 250, 125, 376, 188, 94, 47, 142, 71, 214, 107, 322, 161, 484, 242, 121, 364, 182, 91, 274, 137, 412, 206, 103, 310, 155, 466, 233, 700, 350, 175, 526, 263, 790, 395, 1186, 593, 1780, 890, 445, 1336, 668, 334, 167, 502, 251, 754, 377, 1132, 566, 283, 850, 425, 1276, 638, 319, 958, 479, 1438, 719, 2158, 1079, 3238, 1619, 4858, 2429, 7288, 3644, 1822, 911, 2734, 1367, 4102, 2051, 6154, 3077, 9232, 4616, 2308, 1154, 577, 1732, 866, 433, 1300, 650, 325, 976, 488, 244, 122, 61, 184, 92, 46, 23, 70, 35, 106, 53, 160, 80, 40, 20, 10, 5, 16, 8, 4, 2, 1]"
```

如预期的那样，这个输出与之前的结果匹配。

# 编写流式协议

在许多情况下，一个协议有一个数据单元，其中包含一个附加的头部。服务器通常首先读取头部，然后根据这个头部决定如何处理数据。在某些情况下，服务器可能能够根据头部进行一些处理。一个这样的例子是 IP，它有一个包含目标地址的头部。服务器可能开始根据该信息运行最长前缀匹配，然后再读取主体。在本节中，我们将探讨如何使用 tokio 编写这样的服务器。我们将扩展我们的玩具 collatz 协议，包括一个头部和一些数据主体，并从这里开始。让我们从一个例子开始，我们的项目设置将与上次完全相同，使用 Cargo 进行设置：

```rs
$ cargo new collatz-streaming
```

Cargo 配置没有太大变化：

```rs
[package]
name = "collatz-streaming"
version = "0.1.0"
authors = ["Foo <foo@bar.com>"]

[dependencies]
bytes = "0.4"
futures = "0.1"
tokio-io = "0.1"
tokio-core = "0.1"
tokio-proto = "0.1"
tokio-service = "0.1"
```

由于这个示例较大，我们将其分解为几个组成部分，如下所示。第一部分展示了设置客户端的过程：

```rs
// ch7/collatz-streaming/src/lib.rs

extern crate bytes;
extern crate futures;
extern crate tokio_core;
extern crate tokio_io;
extern crate tokio_proto;
extern crate tokio_service;

use futures::{future, Future, Poll, Stream};
use futures::sync::mpsc;
use tokio_io::{AsyncRead, AsyncWrite};
use tokio_io::codec::{Decoder, Encoder, Framed};
use tokio_core::reactor::Handle;
use tokio_proto::TcpClient;
use tokio_proto::streaming::{Body, Message};
use tokio_proto::streaming::pipeline::{ClientProto, Frame, ServerProto};
use tokio_proto::util::client_proxy::ClientProxy;
use tokio_service::Service;
use std::str::FromStr;
use bytes::{BufMut, BytesMut};
use std::{io, str};
use std::net::SocketAddr;

// Everything about clients
type CollatzMessage = Message<String, Body<String, io::Error>>;

#[derive(Debug)]
pub enum CollatzInput {
    Once(String),
    Stream(CollatzStream),
}

pub struct CollatzProto;

pub struct Client {
    inner: ClientProxy<CollatzMessage, CollatzMessage, io::Error>,
}

impl Client {
    pub fn connect(
        addr: &SocketAddr,
        handle: &Handle,
    ) -> Box<Future<Item = Client, Error = io::Error>> {
        let ret = TcpClient::new(CollatzProto)
            .connect(addr, handle)
            .map(|cp| Client { inner: cp });

        Box::new(ret)
    }
}

impl Service for Client {
    type Request = CollatzInput;
    type Response = CollatzInput;
    type Error = io::Error;
    type Future = Box<Future<Item = Self::Response, Error = 
    io::Error>>;

    fn call(&self, req: CollatzInput) -> Self::Future {
        Box::new(self.inner.call(req.into()).map(CollatzInput::from))
    }
}
```

总是首先设置外部 crate 并包含所有必需的内容。我们定义了一些我们将要使用的类型。`CollatzMessage` 是我们的协议接收到的消息；它有一个头部和一个主体，都是 `String` 类型。`CollatzInput` 是协议的输入流，它是一个枚举，有两个变体：`Once` 代表我们以非流式方式接收数据的情况，而 `Stream` 是第二种情况。协议实现是一个名为 `CollatzProto` 的单元结构。然后我们定义了一个客户端结构，它有一个内部的 `ClientProxy` 实例，这是实际客户端的实现。它接受三种类型，前两种是整个服务器的请求和响应，最后一种是错误。然后我们为 `Client` 结构实现一个连接方法，使用 `CollatzProto` 进行连接，并返回一个包含连接的 future。最后一步是实现 `Client` 的 `Service`，输入和输出都是 `CollatzInput` 类型，因此我们必须使用 future 上的 map 将输出转换为该类型。让我们继续到服务器；它看起来是这样的：

```rs
//ch7/collatz-streaming/src/lib.rs

// Everything about server
#[derive(Debug)]
pub struct CollatzStream {
    inner: Body<String, io::Error>,
}

impl CollatzStream {
    pub fn pair() -> (mpsc::Sender<Result<String, io::Error>>,
    CollatzStream) {
        let (tx, rx) = Body::pair();
        (tx, CollatzStream { inner: rx })
    }
}

impl Stream for CollatzStream {
    type Item = String;
    type Error = io::Error;

    fn poll(&mut self) -> Poll<Option<String>, io::Error> {
        self.inner.poll()
    }
}

pub struct CollatzCodec {
    decoding_head: bool,
}

// Decodes a frame to a byte slice
impl Decoder for CollatzCodec {
    type Item = Frame<String, String, io::Error>;
    type Error = io::Error;

    fn decode(&mut self, buf: &mut BytesMut) -> Result<Option<Self::Item>, io::Error> {
        if let Some(n) = buf.as_ref().iter().position(|b| *b == b'\n') {
            let line = buf.split_to(n);

            buf.split_to(1);
            return match str::from_utf8(line.as_ref()) {
                Ok(s) => {
                    if s == "" {
                        let decoding_head = self.decoding_head;
                        self.decoding_head = !decoding_head;

                        if decoding_head {
                            Ok(Some(Frame::Message {
                                message: s.to_string(),
                                body: true,
                            }))
                        } else {
                            Ok(Some(Frame::Body { chunk: None }))
                        }
                    } else {
                        if self.decoding_head {
                            Ok(Some(Frame::Message {
                                message: s.to_string(),
                                body: false,
                            }))
                        } else {
                            Ok(Some(Frame::Body {
                                chunk: Some(s.to_string()),
                            }))
                        }
                    }
                }
                Err(_) => Err(io::Error::new(io::ErrorKind::Other, 
                "invalid string")),
            };
        }

        Ok(None)
    }
}

// Encodes a given byte slice to a frame
impl Encoder for CollatzCodec {
    type Item = Frame<String, String, io::Error>;
    type Error = io::Error;

    fn encode(&mut self, msg: Self::Item, buf: &mut BytesMut) ->
    io::Result<()> {
        match msg {
            Frame::Message { message, body } => {
                buf.reserve(message.len());
                buf.extend(message.as_bytes());
            }
            Frame::Body { chunk } => {
                if let Some(chunk) = chunk {
                    buf.reserve(chunk.len());
                    buf.extend(chunk.as_bytes());
                }
            }
            Frame::Error { error } => {
                return Err(error);
            }
        }
        buf.put_u8(b'\n');
        Ok(())
    }
}

impl<T: AsyncRead + AsyncWrite + 'static> ClientProto<T> for CollatzProto {
    type Request = String;
    type RequestBody = String;
    type Response = String;
    type ResponseBody = String;
    type Error = io::Error;

    type Transport = Framed<T, CollatzCodec>;
    type BindTransport = Result<Self::Transport, io::Error>;

    fn bind_transport(&self, io: T) -> Self::BindTransport {
        let codec = CollatzCodec {
            decoding_head: true,
        };

        Ok(io.framed(codec))
    }
}

impl<T: AsyncRead + AsyncWrite + 'static> ServerProto<T> for CollatzProto {
    type Request = String;
    type RequestBody = String;
    type Response = String;
    type ResponseBody = String;
    type Error = io::Error;

    type Transport = Framed<T, CollatzCodec>;
    type BindTransport = Result<Self::Transport, io::Error>;

    fn bind_transport(&self, io: T) -> Self::BindTransport {
        let codec = CollatzCodec {
            decoding_head: true,
        };

        Ok(io.framed(codec))
    }
}
```

如预期，`CollatzStream` 的主体要么是一个字符串，要么是发生了错误。现在，对于流式协议，我们需要提供一个函数的实现，该函数返回流的一半发送者；我们在 `CollatzStream` 的 `pair` 函数中这样做。接下来，我们为我们的自定义流实现 `Stream` 特性；在这种情况下，`poll` 方法简单地轮询内部 `Body` 以获取更多数据。在设置好流之后，我们可以实现编解码器。在这里，我们需要维护一种方式来知道我们此刻正在处理帧的哪一部分。这是通过一个名为 `decoding_head` 的布尔值来完成的，我们需要根据需要翻转它。我们需要为我们的编解码器实现 `Decoder`，这基本上和上次一样；只需注意我们需要跟踪流和非流的情况以及之前定义的布尔值。`Encoder` 的实现是相反的。我们还需要将协议实现绑定到编解码器；这是通过实现 `ClientProto` 和 `ServerProto` 为 `CollatzProto` 来完成的。在两种情况下，我们都将布尔值设置为 true，因为接收消息后要读取的第一个东西是头部。

在堆栈中的最后一步是通过实现 `Service` 特性为 `CollatzService` 实现服务。在那里，我们读取头部并尝试将其解析为 `u64`。如果这做得很好，我们就继续计算该 `u64` 的 collatz 序列，并以 `CollatzInput::Once` 的形式在 leaf future 中返回结果。在另一种情况下，我们遍历主体并在控制台上打印它。最后，我们向客户端返回一个固定的字符串。这就是它的样子：

```rs
//ch7/collatz-streaming/src/lib.rs

pub struct CollatzService;

// Given an u64, returns it's collatz sequence
fn get_sequence(mut n: u64) -> Vec<u64> {
    let mut result = vec![];
    result.push(n);
    while n > 1 {
        if n % 2 == 0 {
            n /= 2;
        } else {
            n = 3 * n + 1;
        }
        result.push(n);
    }
    result
}

// Removes leading and trailing whitespaces from a given line
// and tries to parse it as a u64
fn clean_line(line: &str) -> Result<u64, <u64 as FromStr>::Err> {
    line.trim().parse::<u64>()
}

impl Service for CollatzService {
    type Request = CollatzInput;
    type Response = CollatzInput;
    type Error = io::Error;
    type Future = Box<Future<Item = Self::Response, Error = Self::Error>>;

    fn call(&self, req: Self::Request) -> Self::Future {
        match req {
            CollatzInput::Once(line) => {
                println!("Server got: {}", line);
                let res = get_sequence(clean_line(&line).unwrap());
                Box::new(future::done(Ok(CollatzInput::Once
                (format!("{:?}", res)))))
            }
            CollatzInput::Stream(body) => {
                let resp = body.for_each(|line| {
                    println!("{}", line);
                    Ok(())
                }).map(|_| CollatzInput::Once("Foo".to_string()));

                Box::new(resp) as Box<Future<Item = Self::
                Response, Error = io::Error>>
            }
        }
    }
}
```

我们还编写了两个从`CollatzMessage`到`CollatzInput`以及相反的转换辅助器，通过相应地实现`From`特质。像其他所有内容一样，我们不得不面对两种情况：当消息有主体时，以及当它没有主体时（换句话说，头部已经到达，但消息的其他部分还没有到达）。以下是这些：

```rs
// ch7/collatz-streaming/src/lib.rs

// Converts a CollatzMessage to a CollatzInput
impl From<CollatzMessage> for CollatzInput {
    fn from(src: CollatzMessage) -> CollatzInput {
        match src {
            Message::WithoutBody(line) => CollatzInput::Once(line),
            Message::WithBody(_, body) => CollatzInput::Stream(CollatzStream { inner: body }),
        }
    }
}

// Converts a CollatzInput to a Message<String, Body>
impl From<CollatzInput> for Message<String, Body<String, io::Error>> {
    fn from(src: CollatzInput) -> Self {
        match src {
            CollatzInput::Once(line) => Message::WithoutBody(line),
            CollatzInput::Stream(body) => {
                let CollatzStream { inner } = body;
                Message::WithBody("".to_string(), inner)
            }
        }
    }
}
```

在设置好服务器和客户端之后，我们将像上次一样以示例的形式实现我们的测试。它们看起来是这样的：

```rs
// ch7/collatz-streaming/examples/server.rs

extern crate collatz_streaming as collatz;
extern crate futures;

extern crate tokio_proto;

use tokio_proto::TcpServer;
use collatz::{CollatzProto, CollatzService};

fn main() {
    let addr = "0.0.0.0:9999".parse().unwrap();
```

```rs
    TcpServer::new(CollatzProto, addr).serve(|| Ok(CollatzService));
}
```

客户端看起来是这样的。与服务器相比，这里有一些内容需要消化：

```rs
// ch7/collatz-streaming/examples/client.rs

extern crate collatz_streaming as collatz;

extern crate futures;
extern crate tokio_core;
extern crate tokio_service;

use collatz::{CollatzInput, CollatzStream};
use std::thread;
use futures::Sink;
use futures::Future;
use tokio_core::reactor::Core;
use tokio_service::Service;

pub fn main() {
    let addr = "127.0.0.1:9999".parse().unwrap();
    let mut core = Core::new().unwrap();
    let handle = core.handle();

    // Run the client in the event loop
    core.run(
        collatz::Client::connect(&addr, &handle)
            .and_then(|client| {
                client.call(CollatzInput::Once("10".to_string()))
                    .and_then(move |response| {
                        println!("Response: {:?}", response);

                        let (mut tx, rx) = CollatzStream::pair();

                        thread::spawn(move || {
                            for msg in &["Hello", "world", "!"] {
                                tx =
                                tx.send(Ok(msg.to_string()))
                                .wait().unwrap();
                            }
                        });

                        client.call(CollatzInput::Stream(rx))
                    })
                    .and_then(|response| {
                        println!("Response: {:?}", response);
                        Ok(())
                    })
            })
    ).unwrap();
}
```

我们使用之前定义的`connect`方法在已知地址和端口上设置到服务器的连接。我们使用`and_then`组合器向服务器发送一个固定字符串，并打印响应。此时，我们已经发送了头部，接下来我们将发送主体。这是通过将流分成两半并使用发送者发送多个字符串来完成的。一个最终的组合器打印响应并解决未来。所有之前的内容都在事件循环中运行。

这是服务器会话的样子：

```rs
$ cargo run --example server
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/examples/server`
Server got: 10
Hello
world
!
```

而客户端看起来是这样的：

```rs
$ cargo run --example client
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/examples/client`
Response: Once("[10, 5, 16, 8, 4, 2, 1]")
Response: Once("Foo")
```

如预期，服务器在收到请求的实际主体之前处理了头部（我们知道这一点是因为我们在发送头部之后发送了主体）。

除了我们在这里讨论的内容之外，tokio 还支持许多其他功能。例如，你可以通过更改`ServerProto`和`ClientProto`实现来交换设置消息，在构建`BindTransport`未来之前实现协议握手。这一点非常重要，因为许多网络协议需要某种形式的握手来设置共享状态以协同工作。请注意，一个协议可以是流式和管道化的，或者流式和复用的。对于这些，实现需要分别替换`streaming::pipeline`或`streaming::multiplex`命名空间中的特质。

# 更大的 tokio 生态系统

让我们看看 tokio 生态系统当前的状态。以下是写作时在`tokio-rs` GitHub 组织中的常用 crate：

| **Crate** | **函数** |
| --- | --- |
| `tokio-minihttp` | 在 tokio 中的简单 HTTP 服务器实现；不应在生产中使用。 |
| `tokio-core` | 具有未来感知的网络实现；核心事件循环。 |
| `tokio-io` | tokio 的 IO 原语。 |
| `tokio-curl` | 使用 tokio 的基于`libcurl`的 HTTP 客户端实现。 |
| `tokio-uds` | 使用 tokio 的非阻塞 Unix 域套接字。 |
| `tokio-tls` | 基于 tokio 的 TLS 和 SSL 实现。 |
| `tokio-service` | 提供了我们广泛使用的`Service`特质。 |
| `tokio-proto` | 提供了一个使用 tokio 构建网络协议的框架；我们广泛使用了这个。 |
| `tokio-socks5` | 使用 tokio 的 SOCKS5 服务器，尚未准备好用于生产。 |
| `tokio-middleware` | 用于 tokio 服务的中间件集合；目前缺少基本服务。 |
| `tokio-times` | 基于 tokio 的定时器相关功能。 |
| `tokio-line` | 用于演示 tokio 的样本行协议。 |
| `tokio-redis` | 基于 tokio 的证明概念 Redis 客户端；不应在生产中使用。 |
| `service-fn` | 为给定的闭包提供实现 Service 特性的函数。 |

注意，其中许多已经很长时间没有更新，或者是一些应该不用于任何有用目的的证明概念实现。但这不是问题。自它推出以来，大量独立的实用工具已经采用了 tokio，从而形成了一个充满活力的生态系统。而且，据我们观察，这是任何开源项目成功的真正标志。

让我们看看前面列表中一些常用的库，首先是 `tokio-curl`。在我们的示例中，我们将简单地从已知位置下载单个文件，将其写入本地磁盘，并打印出我们从服务器获取的头部信息。由于这是一个二进制文件，我们将按照以下方式设置项目：

```rs
$ cargo new --bin tokio-curl
```

下面是 Cargo 设置的示例：

```rs
[package]
name = "tokio-curl"
version = "0.1.0"
authors = ["Foo <foo@bar.com>"]

[dependencies]
tokio-curl = "0.1"
tokio-core = "0.1"
curl = "0.4.8"
```

由于 `tokio-curl` 库是 Rust `curl` 库的包装器，因此我们还需要包含它。以下是主文件：

```rs
// ch7/toki-curl/src/main.rs

extern crate curl;
extern crate tokio_core;
extern crate tokio_curl;

use curl::easy::Easy;
use tokio_core::reactor::Core;
use tokio_curl::Session;
use std::io::Write;
use std::fs::File;

fn main() {
    let mut core = Core::new().unwrap();
    let session = Session::new(core.handle());

    let mut handle = Easy::new();
    let mut file = File::create("foo.zip").unwrap();
    handle.get(true).unwrap();
    handle.url("http://ipv4.download.thinkbroadband.com/5MB.zip").unwrap();
    handle.header_function(|header| {
        print!("{}", std::str::from_utf8(header).unwrap());
        true
    }).unwrap();
    handle.write_function(move |data| {
        file.write_all(data).unwrap();
        Ok(data.len())
    }).unwrap();

    let request = session.perform(handle);

    let mut response = core.run(request).unwrap();
    println!("{:?}", response.response_code());
}
```

我们将使用来自 `curl` crate 的 `Easy` API。我们首先创建我们的事件循环和 HTTP 会话。然后创建一个 `libcurl` 将用于处理我们请求的句柄。我们使用布尔值调用 `get` 方法来表示我们感兴趣进行 HTTP GET 操作。然后我们将 URL 传递给句柄。接下来，我们设置两个作为闭包传递的回调函数。第一个被称为 `header_function`；这个函数显示客户端的每个头部信息。第二个被称为 `write_function`，它将我们获取的数据写入我们的文件。最后，我们通过调用会话的 `perform` 函数创建一个请求。最后，我们在事件循环中运行请求并打印出我们得到的状态码。

运行此操作将执行以下操作：

```rs
$ cargo run
   Compiling tokio-curl v0.1.0 (file:///src/ch7/tokio-curl)
    Finished dev [unoptimized + debuginfo] target(s) in 0.97 secs
     Running `target/debug/tokio-curl`
HTTP/1.1 200 OK
Server: nginx
Date: Mon, 25 Dec 2017 20:16:12 GMT
Content-Type: application/zip
Content-Length: 5242880
Last-Modified: Mon, 02 Jun 2008 15:30:42 GMT
Connection: keep-alive
ETag: "48441222-500000"
Access-Control-Allow-Origin: *
Accept-Ranges: bytes

Ok(200)
```

这将在当前目录中生成一个名为 `foo.zip` 的文件。您可以使用常规文件下载该文件，并比较两个文件的 SHA 哈希值，以验证它们确实相同。

# 结论

本章是 Rust 更大生态系统中最激动人心的组件之一的一个介绍。Futures 和 Tokio 生态系统提供了强大的原语，这些原语可以广泛应用于应用程序中，包括网络软件。本身，Futures 可以用来模拟任何其他情况下缓慢的或依赖于外部资源的计算。与 tokio 结合使用，它可以用来模拟复杂的流水线或复用等协议行为。

使用这些工具的主要缺点集中在缺乏适当的文档和示例。此外，这些应用程序的错误信息通常非常模板化，因此往往冗长。因为 Rust 编译器本身并不了解这些抽象，它经常会抱怨类型不匹配，用户必须深入推理嵌套的类型。在某个阶段，实现一个能够将这些错误转换为更直观形式的编译器插件可能是有意义的。

在下一章中，我们将探讨在 Rust 中实现常见的与安全相关的原语。
