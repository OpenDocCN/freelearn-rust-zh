# 第四章：无畏并发

并发和并行是现代编程的重要组成部分，Rust 完美地配备了处理这些挑战的能力。借用和所有权模型对于防止数据竞争（在数据库世界中被称为**异常**）非常有效，因为变量默认是不可变的，如果需要可变性，则不能有其他任何对数据的引用。这使得 Rust 中的任何类型的并发都更加安全且复杂度更低（与其他许多语言相比）。

在本章中，我们将介绍几种使用并发来解决问题的方法，甚至还会探讨未来，这些在未来写作时还不是语言的一部分。如果你在将来阅读这篇文章（不是字面意思），这可能已经是核心语言的一部分了，你可以查看 *使用 futures 进行异步编程* 食谱以供历史参考。

在本章中，我们将介绍以下食谱：

+   将数据移动到新线程中

+   管理多个线程

+   线程间的消息传递

+   共享可变状态

+   多进程

+   将顺序代码并行化

+   向量中的并发数据处理

+   共享不可变状态

+   演员 和 异步消息

+   使用 futures 进行异步编程

# 将数据移动到新线程中

Rust 线程的工作方式与其他任何语言一样——在作用域中。任何其他作用域（如闭包）都可以轻松地从父作用域借用变量，因为很容易确定变量何时被丢弃。然而，在启动线程时，与父作用域的生命周期相比，其生命周期是无法知道的，因此引用可能在任何时候变得无效。

为了解决这个问题，线程作用域可以拥有其变量——内存被**移动**到线程的作用域中。让我们看看这是如何实现的！

# 如何做到这一点...

按照以下步骤查看如何在线程之间移动内存：

1.  使用 `cargo new simple-threads` 创建一个新的应用程序项目，并在 Visual Studio Code 中打开该目录。

1.  编辑 `src/main.rs` 并启动一个简单的线程，该线程不会将数据移动到其作用域中。由于这是线程的最简单形式，让我们打印一些内容到命令行并等待：

```rs
use std::thread;
use std::time::Duration;

fn start_no_shared_data_thread() -> thread::JoinHandle<()> {
    thread::spawn(|| {
        // since we are not using a parent scope variable in here
        // no move is required
        println!("Waiting for three seconds.");
        thread::sleep(Duration::from_secs(3)); 
        println!("Done")
    })
}
```

1.  现在，让我们在 `fn main()` 中调用新函数。将 `hello world` 碎片替换为以下内容：

```rs
    let no_move_thread = start_no_shared_data_thread();

    for _ in 0..10 {
        print!(":");
    }

    println!("Waiting for the thread to finish ... {:?}", 
    no_move_thread.join());
```

1.  让我们运行代码看看它是否工作：

```rs
$ cargo run
 Compiling simple-threads v0.1.0 (Rust-Cookbook/Chapter05/simple-
 threads)
    Finished dev [unoptimized + debuginfo] target(s) in 0.35s
     Running `target/debug/simple-threads`
::::::::::Waiting for three seconds.
Done
Waiting for the thread to finish ... Ok(())
```

1.  现在，让我们将一些外部数据移动到线程中。向 `src/main.rs` 中添加另一个函数：

```rs
fn start_shared_data_thread(a_number: i32, a_vec: Vec<i32>) -> thread::JoinHandle<Vec<i32>> {
    // thread::spawn(move || {
    thread::spawn(|| {
        print!(" a_vec ---> [");
        for i in a_vec.iter() {
            print!(" {} ", i);
        }
        println!("]");
        println!(" A number from inside the thread: {}", a_number);
        a_vec // let's return ownership
    })
}
```

1.  为了演示底层发生的事情，我们目前省略了 `move` 关键字。使用以下代码扩展 `main` 函数：

```rs
    let a_number = 42;
    let a_vec = vec![1,2,3,4,5];

    let move_thread = start_shared_data_thread(a_number, a_vec);

    println!("We can still use a Copy-enabled type: {}", a_number); 
    println!("Waiting for the thread to finish ... {:?}", 
    move_thread.join());
```

1.  它是否工作？让我们尝试 `cargo run`：

```rs
$ cargo run
Compiling simple-threads v0.1.0 (Rust-Cookbook/Chapter04/simple-threads)
error[E0373]: closure may outlive the current function, but it borrows `a_number`, which is owned by the current function
  --> src/main.rs:22:20
   |
22 | thread::spawn(|| {
   |               ^^ may outlive borrowed value `a_number`
...
29 |    println!(" A number from inside the thread: {}", a_number);
   |                                                     -------- `a_number` is borrowed here
   |
note: function requires argument type to outlive `'static`
  --> src/main.rs:22:6
   |
23 | /     thread::spawn(|| {
24 | |        print!(" a_vec ---> [");
25 | |        for i in a_vec.iter() {
...  |
30 | |        a_vec // let's return ownership
31 | |     })
   | |______^
help: to force the closure to take ownership of `a_number` (and any other referenced variables), use the `move` keyword
   |
23 | thread::spawn(move || {
   |               ^^^^^^^

error: aborting due to previous error

For more information about this error, try `rustc --explain E0373`.
error: Could not compile `simple-threads`.

To learn more, run the command again with --verbose.
```

1.  我们已经看到了：要将任何类型的数据移动到线程作用域中，我们需要通过使用 `move` 关键字将值移动到作用域中来转移所有权。让我们遵循编译器的指示：

```rs
///
/// Starts a thread moving the function's input parameters
/// 
fn start_shared_data_thread(a_number: i32, a_vec: Vec<i32>) -> thread::JoinHandle<Vec<i32>> {
    thread::spawn(move || {
    // thread::spawn(|| {
        print!(" a_vec ---> [");
        for i in a_vec.iter() {
            print!(" {} ", i);
        }
        println!("]");
        println!(" A number from inside the thread: {}", a_number);
        a_vec // let's return ownership
    })
}
```

1.  让我们再次尝试使用 `cargo run`：

```rs
$ cargo run
   Compiling simple-threads v0.1.0 (Rust-Cookbook/Chapter04/simple-
   threads)
    Finished dev [unoptimized + debuginfo] target(s) in 0.38s
     Running `target/debug/simple-threads`
::::::::::Waiting for three seconds.
Done
Waiting for the thread to finish ... Ok(())
We can still use a Copy-enabled type: 42
   a_vec ---> [ 1 2 3 4 5 ]
   A number from inside the thread: 42
Waiting for the thread to finish ... Ok([1, 2, 3, 4, 5])
```

现在，让我们深入了解代码，以更好地理解它。

# 它是如何工作的...

Rust 中的线程行为与常规函数非常相似：它们可以拥有所有权，并且可以使用与闭包相同的语法（`|| {}` 是一个不带参数的空/`noop` 函数）。因此，我们必须像对待函数一样对待它们，并从所有权和借用（或更具体地说：生命周期）的角度来考虑。将引用（默认行为）传递到这个线程函数中，使得编译器无法跟踪引用的有效性，这对代码安全是一个问题。Rust 通过引入 `move` 关键字来解决此问题。

使用 `move` 关键字改变了默认的借用行为，将每个变量的所有权移动到作用域中。因此，除非这些值实现了 `Copy` 特性（如 `i32`），或者借用时具有比线程更长的生命周期（如 `str` 文本的 `'static' 生命周期），否则它们将无法在线程的父作用域中使用。

返回所有权也像在函数中一样工作——通过 `return` 语句。等待其他线程（使用 `join()`）的线程可以解包 `join()` 的结果来检索返回值。

Rust 中的线程是操作系统的原生线程，每个线程都有自己的局部状态和执行栈。当它们崩溃时，只有线程停止，而不是整个程序。

我们已经成功地将数据移动到新的线程中。现在让我们继续到下一个菜谱。

# 管理多个线程

单线程很棒，但在现实中，许多用例需要大量的线程来并行执行大规模数据集。这已经被几年前发布的 map/reduce 模式所普及，并且仍然是一种处理多个文件、数据库结果中的行等不同事物的并行的好方法。无论来源如何，只要处理不是相互依赖的，就可以将其分块并 **映射**，这两者 Rust 都可以使其变得简单且无数据竞争条件。

# 如何实现...

在这个菜谱中，我们将添加更多线程来进行映射风格的数据处理。按照以下步骤操作：

1.  运行 `cargo new multiple-threads` 来创建一个新的应用程序项目，并在 Visual Studio Code 中打开该目录。

1.  在 `src/main.rs` 中，在 `main()` 之上添加以下函数：

```rs
use std::thread;

///
/// Doubles each element in the provided chunks in parallel and returns the results.
/// 
fn parallel_map(data: Vec<Vec<i32>>) -> Vec<thread::JoinHandle<Vec<i32>>> {
    data.into_iter()
        .map(|chunk| thread::spawn(move ||
         chunk.into_iter().map(|c| 
         c * 2).collect()))
        .collect()
}
```

1.  在这个函数中，我们为每个传入的块启动一个线程。这个线程只将数字加倍，因此函数为每个包含此转换结果的块返回 `Vec<i32>`。现在我们需要创建输入数据并调用该函数。让我们扩展 `main` 来实现这一点：

```rs
fn main() {

    // Prepare chunked data
    let data = vec![vec![1, 2, 3], vec![4, 4, 5], vec![6, 7, 7]];

    // work on the data in parallel
    let results: Vec<i32> = parallel_map(data.clone())
        .into_iter() // an owned iterator over the results
        .flat_map(|thread| thread.join().unwrap()) // join each 
         thread
        .collect(); // collect the results into a Vec

    // flatten the original data structure
    let data: Vec<i32> = data.into_iter().flat_map(|e| e)
     .collect();

    // print the results
    println!("{:?} -> {:?}", data, results);
}
```

1.  使用 `cargo run` 我们现在可以看到结果：

```rs
$ cargo run
   Compiling multiple-threads v0.1.0 (Rust-
    Cookbook/Chapter04/multiple-threads)
    Finished dev [unoptimized + debuginfo] target(s) in 0.45s
     Running `target/debug/multiple-threads`
    [1, 2, 3, 4, 4, 5, 6, 7, 7] -> [2, 4, 6, 8, 8, 10, 12, 14, 14]
```

现在，让我们深入了解代码，以更好地理解它。

# 它是如何工作的...

承认，在 Rust 中处理多个线程就像我们在处理单线程一样，因为没有方便的方法来连接线程列表或类似的东西。相反，我们可以使用 Rust 的迭代器的力量以表达的方式做到这一点。有了这些功能结构，我们可以用一系列函数来替换`for`循环，这些函数可以懒惰地处理集合，这使得代码更容易处理且更高效。

在完成*步骤 1*的项目设置后，我们实现了一个多线程函数，用于对每个块应用一个操作。这些块仅仅是向量的部分，在这个例子中，一个操作——一个简单的将输入变量加倍的功能——可以对任何类型的任务进行。*步骤 3*展示了如何调用多线程的`mapping`函数，以及如何通过在未来的/承诺([`dist-prog-book.com/chapter/2/futures.html`](http://dist-prog-book.com/chapter/2/futures.html))方式中使用`JoinHandle`来获取结果。*步骤 4*简单地展示了它按预期工作，通过输出加倍后的块作为扁平列表。

另一个有趣的是，我们不得不克隆数据的次数。由于将数据传递到线程中只能通过将值移动到每个线程的内存空间中，因此克隆通常是解决这些共享问题的唯一方法。然而，我们将在本章后面的食谱（*共享不可变状态*）中介绍一个类似多个`Rc`的方法，所以让我们继续下一个食谱。

# 使用通道在线程之间进行通信

在许多标准库和编程语言中，线程间的消息传递一直是一个问题，因为许多依赖于用户应用锁定。这导致了死锁，对于新手来说有些令人生畏，这也是为什么许多开发者对 Go 普及通道概念感到兴奋，我们也可以在 Rust 中找到这一点。Rust 的通道非常适合用几行代码设计一个安全、事件驱动的应用程序，而不需要任何显式的锁定。

# 如何做到这一点...

让我们创建一个简单的应用程序，该程序在命令行中可视化传入的值：

1.  运行`cargo new channels`以创建一个新的应用程序项目，并在 Visual Studio Code 中打开该目录。

1.  首先，让我们先处理基础知识。打开`src/main.rs`文件，并添加导入和一个`enum`结构：

```rs
use std::sync::mpsc::{Sender, Receiver};
use std::sync::mpsc;
use std::thread;

use rand::prelude::*;
use std::time::Duration;

enum ChartValue {
    Star(usize),
    Pipe(usize)
}
```

1.  然后，在`main`函数内部，我们使用`mpsc::channel()`函数创建了一个通道，以及两个负责发送的线程。之后，我们将使用两个线程以可变延迟向主线程发送消息。以下是代码：

```rs
fn main() {
    let (tx, rx): (Sender<ChartValue>, Receiver<ChartValue>) = 
     mpsc::channel();

    let pipe_sender = tx.clone();

    thread::spawn(move || {
        loop {
            pipe_sender.send(ChartValue::Pipe(random::<usize>() % 
             80)).unwrap();
            thread::sleep(Duration::from_millis(random::<u64>() % 
             800));
        }
    });

    let star_sender = tx.clone();
    thread::spawn(move || {
        loop {
            star_sender.send(ChartValue::Star(random::<usize>() % 
             80)).unwrap();
            thread::sleep(Duration::from_millis(random::<u64>() % 
             800));
        }
    });
```

1.  两个线程都在向通道发送数据，所以缺少的是通道的接收端来处理输入数据。接收器提供了两个函数，`recv()`和`recv_timeout()`，这两个函数都会阻塞调用线程，直到接收到一个项目（或达到超时）。我们只是将要打印字符乘以传入的值：

```rs
    while let Ok(val) = rx.recv_timeout(Duration::from_secs(3)) {

        println!("{}", match val {
            ChartValue::Pipe(v) => "|".repeat(v + 1),
            ChartValue::Star(v) => "*".repeat(v + 1)
        });
    }
}
```

1.  为了在最终运行程序时使用 `rand`，我们仍然需要将其添加到 `Cargo.toml` 中，如下所示：

```rs
[dependencies]
rand = "⁰.5"
```

1.  最后，让我们看看程序是如何运行的——它将无限期地运行。要停止它，请按 *Ctrl* + *C*。用 `cargo run` 运行它：

```rs
$ cargo run
   Compiling channels v0.1.0 (Rust-Cookbook/Chapter04/channels)
    Finished dev [unoptimized + debuginfo] target(s) in 1.38s
     Running `target/debug/channels`
||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||
****************************
|||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||
|||||||||||||||||||||||||||||||||
*********************************************************
||||||||||||||||||||||||||||
************************************************************
*****************************
||||||||||||||||
***********
||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||
*******************************
|||||||||||||||||||||||||||||||||||||||||
*************************************************************
|||||||||||||||||||||||||||||||||||||||||||||||
*******************************
************************************************************************
*******************
******************************************************
|||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||
||||||||||||||||||||||||||||||||
************************************************
*
||||||||||||||||||||||||||||||||||||||||
***********************************************
||||||
*************************
|||||||||||||||||||
|||||||||||||||||||||||||||||||||||
^C⏎ 
```

这是如何工作的？让我们深入了解代码，以更好地理解它。

# 它是如何工作的...

通道是 **多生产者单消费者** 数据结构，由许多发送者（带有轻量级克隆）组成，但只有一个接收者。在底层，通道不锁定，而是依赖于一个 `unsafe` 数据结构，该结构允许检测和管理流的状态。通道很好地处理了跨线程发送数据，并且可以用来创建演员风格的框架或反应式 map-reduce 风格的数据处理引擎。

这是 Rust 如何做到 **无畏并发** 的一个例子：进入的数据由通道拥有，直到接收者检索它，这时新的所有者接管。通道还充当队列，并持有元素直到它们被检索。这不仅使开发者免于实现交换，而且还免费为常规队列添加了并发性。

我们在这个菜谱的 *第 3 步* 中创建通道，并将发送者传递到不同的线程中，这些线程开始发送之前定义的（在 *第 2 步*）`enum` 类型，以便接收者打印。这种打印是在 *第 4 步* 通过循环带有三秒超时的阻塞迭代器来完成的。*第 5 步* 展示了如何将依赖项添加到 `Cargo.toml`，而在 *第 6 步* 我们看到了输出：多行，包含随机数量的元素，这些元素可以是星号 (`*`) 或管道 (`|`)。

我们已经成功介绍了如何使用通道在线程之间轻松通信。现在让我们继续下一个菜谱。

# 共享可变状态

Rust 的所有权和借用模型大大简化了不可变数据的访问和传输——那么共享状态怎么办？有许多应用程序需要从多个线程对共享资源进行可变访问。让我们看看这是如何完成的！

# 如何去做...

在这个菜谱中，我们将创建一个非常简单的模拟：

1.  运行 `cargo new black-white` 来创建一个新的应用程序项目，并在 Visual Studio Code 中打开该目录。

1.  打开 `src/main.rs` 以添加一些代码。首先，我们需要一些导入和一个 `enum` 来使我们的模拟更有趣：

```rs
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

///
/// A simple enum with only two variations: black and white
/// 
#[derive(Debug)]
enum Shade {
    Black,
    White,
}
```

1.  为了展示两个线程之间的共享状态，显然我们需要一个在某个东西上工作的线程。这将是一个着色任务，其中每个线程只有在黑色是前一个元素时才向向量添加白色，反之亦然。因此，每个线程都需要读取并根据输出写入共享向量。让我们看看执行此操作的代码：

```rs
fn new_painter_thread(data: Arc<Mutex<Vec<Shade>>>) -> thread::JoinHandle<()> {
    thread::spawn(move || loop {
        {
            // create a scope to release the mutex as quickly as    
            // possible
            let mut d = data.lock().unwrap();
            if d.len() > 0 {
                match d[d.len() - 1] {
                    Shade::Black => d.push(Shade::White),
                    Shade::White => d.push(Shade::Black),
                }
            } else {
                d.push(Shade::Black)
            }
            if d.len() > 5 {
                break;
            }
        }
        // slow things down a little
        thread::sleep(Duration::from_secs(1));
    })
}
```

1.  在这个阶段，我们剩下的就是创建多个线程，并将数据的 `Arc` 实例传递给它们来处理：

```rs
fn main() {
    let data = Arc::new(Mutex::new(vec![]));
    let threads: Vec<thread::JoinHandle<()>> =
        (0..2)
        .map(|_| new_painter_thread(data.clone()))
        .collect();

    let _: Vec<()> = threads
        .into_iter()
        .map(|t| t.join().unwrap())
        .collect();

    println!("Result: {:?}", data);
}
```

1.  让我们用 `cargo run` 运行代码：

```rs
$ cargo run
   Compiling black-white v0.1.0 (Rust-Cookbook/Chapter04/black-
   white)
    Finished dev [unoptimized + debuginfo] target(s) in 0.35s
     Running `target/debug/black-white`
     Result: Mutex { data: [Black, White, Black, White, Black, 
    White, Black] }
```

现在，让我们深入了解代码，以更好地理解它。

# 它是如何工作的...

Rust 的所有权原则是一把双刃剑：一方面，它保护免受意外后果的影响，并使编译时内存管理成为可能；另一方面，可变访问的获取要困难得多。虽然管理起来更复杂，但共享可变访问对于性能来说可以非常出色。

`Arc`代表**原子引用计数器**。这使得它们与常规引用计数器（`Rc`）非常相似，唯一的区别是`Arc`通过**原子递增**来完成其工作，这是线程安全的。因此，它们是跨线程引用计数的唯一选择。

在 Rust 中，这是以类似内部可变性的方式进行的（[`doc.rust-lang.org/book/ch15-05-interior-mutability.html`](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html)），但使用`Arc`和`Mutex`类型（而不是`Rc`和`RefCell`），其中`Mutex`拥有它限制访问的实际内存部分（在步骤 3 的片段中，我们就是这样创建`Vec`的）。正如*步骤 2*所示，要获取值的可变引用，必须严格锁定`Mutex`实例，并且只有在返回的数据实例被丢弃（例如，当作用域结束时）之后，它才会返回。因此，保持`Mutex`的作用域尽可能小是很重要的（注意*步骤 2*中的附加`{ ... }`）！

在许多用例中，基于通道的方法可以达到相同的目标，而无需处理`Mutex`和死锁（当多个`Mutex`锁相互等待解锁）的恐惧。

我们已经成功学习了如何使用通道来共享可变状态。现在让我们继续下一个菜谱。

# Rust 中的多进程

线程对于进程内并发非常出色，当然也是将工作负载分散到多个核心的首选方法。每当需要调用其他程序，或者需要独立、重量级任务时，子进程就是最佳选择。随着最近编排型应用程序（如 Kubernetes、Docker Swarm、Mesos 等）的兴起，管理子进程也变得更为重要。在这个菜谱中，我们将与子进程进行通信和管理。

# 如何做到这一点...

按照以下步骤创建一个简单的应用程序，用于搜索文件系统：

1.  使用`cargo new child-processes`创建一个新的项目，并在 Visual Studio Code 中打开它。

1.  在 Windows 上，从 PowerShell 窗口中执行`cargo run`（最后一步），因为它包含所有必需的二进制文件。

1.  在导入几个（标准库）依赖项之后，让我们编写基本的`struct`来保存结果数据。在`main`函数顶部添加以下内容：

```rs
use std::io::Write;
use std::process::{Command, Stdio};

#[derive(Debug)]
struct SearchResult {
    query: String,
    results: Vec<String>,
}
```

1.  调用`find`二进制文件（实际执行搜索）的函数将结果转换为*步骤 1*中的`struct`。这个函数看起来是这样的：

```rs
fn search_file(name: String) -> SearchResult {
    let ps_child = Command::new("find")
        .args(&[".", "-iname", &format!("{}", name)])
        .stdout(Stdio::piped())
        .output()
        .expect("Could not spawn process");

    let results = String::from_utf8_lossy(&ps_child.stdout);
    let result_rows: Vec<String> = results
        .split("\n")
        .map(|e| e.to_string())
        .filter(|s| s.len() > 1)
        .collect();

    SearchResult {
        query: name,
        results: result_rows,
    }
}
```

1.  太棒了！现在我们知道了如何调用外部二进制文件，传递参数，并将任何 `stdout` 输出转发到 Rust 程序。那么，如何写入外部程序的 `stdin` 呢？我们将添加以下函数来完成这个任务：

```rs
fn process_roundtrip() -> String {
    let mut cat_child = Command::new("cat")
        .stdin(Stdio::piped())
        .stdout(Stdio::piped())
        .spawn()
        .expect("Could not spawn process");

    let stdin = cat_child.stdin.as_mut().expect("Could 
     not attach to stdin");

    stdin
        .write_all(b"datadatadata")
        .expect("Could not write to child process");
    String::from_utf8(
        cat_child
            .wait_with_output()
            .expect("Something went wrong")
            .stdout
            .as_slice()
            .iter()
            .cloned()
            .collect(),
    )
    .unwrap()
}
```

1.  为了看到它的实际效果，我们还需要在程序的 `main()` 部分调用这些函数。用以下内容替换默认的 `main()` 函数的内容：

```rs
fn main() {
    println!("Reading from /bin/cat > {:?}", process_roundtrip());
    println!(
        "Using 'find' to search for '*.rs': {:?}",
        search_file("*.rs".to_owned())
    )
}
```

1.  现在我们可以通过发出 `cargo run` 来检查它是否工作：

```rs
$ cargo run
   Compiling child-processes v0.1.0 (Rust-Cookbook/Chapter04/child-
    processes)
    Finished dev [unoptimized + debuginfo] target(s) in 0.59s
    Running `target/debug/child-processes`
    Reading from /bin/cat > "datadatadata"
    Using 'find' to search for '*.rs': SearchResult { query: "
    *.rs", results: ["./src/main.rs"] }
```

现在，让我们深入了解代码，以更好地理解它。

# 它是如何工作的...

通过使用 Rust 的运行子进程和操作其输入输出的能力，将现有应用程序集成到新程序的流程中变得非常容易。在 *步骤 1* 中，我们正是通过使用带有参数的 `find` 程序并将输出解析到我们自己的数据结构中来实现这一点。

在 *步骤 3* 中，我们进一步发送数据到子进程，并恢复相同的文本（使用类似 `cat` 的 `echo` 风格）。你会注意到每个函数中都有字符串解析，这是必需的，因为 Windows 和 Linux/macOS 使用不同的字节大小来编码它们的字符（**UTF-16** 和 **UTF-8** 分别）。同样，`b"string"` 将字面量转换为适合当前平台的字节数组字面量。

这些操作的关键成分是 **管道**，这是一个在命令行中使用 `|` （**管道**）符号可用的操作。我们鼓励您也尝试其他 `Stdio` 结构的变体，看看它们能引导您到何处！

我们已经成功地学习了 Rust 中的多进程。现在让我们继续下一个配方。

# 将顺序代码并行化

从头开始创建高度并发的应用程序在许多技术和语言中相对简单。然而，当多个开发者必须基于某种预存在的工作（无论是遗留的还是其他类型）进行构建时，创建这些高度并发的应用程序就会变得复杂。多亏了不同语言之间的 API 差异、最佳实践或技术限制，现有的序列操作不能在没有深入分析的情况下并行运行。如果潜在的好处不显著，谁会去做呢？借助 Rust 的强大迭代器，我们能否在不进行重大代码更改的情况下并行运行操作？我们的答案是肯定的！

# 如何做到这一点...

这个配方展示了如何仅通过几个步骤使用 `rayon-rs` 简单地使应用程序并行运行，而不需要大量努力：

1.  使用 `cargo new use-rayon --lib` 创建一个新的项目，并在 Visual Studio Code 中打开它。

1.  打开 `Cargo.toml` 以向项目添加所需的依赖项。我们将基于 `rayon` 并使用 `criterion` 的基准测试功能：

```rs
# replace the default [dependencies] section...
[dependencies]
rayon = "1.0.3"

[dev-dependencies]
criterion = "0.2.11"
rand = "⁰.5"

[[bench]]
name = "seq_vs_par"
harness = false
```

1.  作为示例算法，我们将使用归并排序，这是一种类似于快速排序的复杂、分而治之的算法（[`www.geeksforgeeks.org/quick-sort-vs-merge-sort/`](https://www.geeksforgeeks.org/quick-sort-vs-merge-sort/)）。让我们从添加 `merge_sort_seq()` 函数到 `src/lib.rs` 的顺序版本开始：

```rs
///
/// Regular, sequential merge sort implementation
/// 
pub fn merge_sort_seq<T: PartialOrd + Clone + Default>(collection: &[T]) -> Vec<T> {
    if collection.len() > 1 {
        let (l, r) = collection.split_at(collection.len() / 2);
        let (sorted_l, sorted_r) = (merge_sort_seq(l), 
         merge_sort_seq(r));
        sorted_merge(sorted_l, sorted_r)
    } else {
        collection.to_vec()
    }
}
```

1.  归并排序的高级视图很简单：将集合分成两半，直到不能再分，然后按顺序合并这些半部分。分割部分已经完成；缺少的是合并部分。将以下片段插入`lib.rs`：

```rs
///
/// Merges two collections into one. 
/// 
fn sorted_merge<T: Default + Clone + PartialOrd>(sorted_l: Vec<T>, sorted_r: Vec<T>) -> Vec<T> {
    let mut result: Vec<T> = vec![Default::default(); sorted_l.len() 
     + sorted_r.len()];

    let (mut i, mut j) = (0, 0);
    let mut k = 0;
    while i < sorted_l.len() && j < sorted_r.len() {
        if sorted_l[i] <= sorted_r[j] {
            result[k] = sorted_l[i].clone();
            i += 1;
        } else {
            result[k] = sorted_r[j].clone();
            j += 1;
        }
        k += 1;
    }
    while i < sorted_l.len() {
        result[k] = sorted_l[i].clone();
        k += 1;
        i += 1;
    }

    while j < sorted_r.len() {
        result[k] = sorted_r[j].clone();
        k += 1;
        j += 1;
    }
    result
}
```

1.  最后，我们还需要导入`rayon`，这是一个用于轻松创建并行应用程序的 crate，然后添加一个修改过的、并行化的归并排序版本：

```rs
use rayon;
```

1.  接下来，我们添加一个修改过的归并排序版本：

```rs
///
/// Merge sort implementation using parallelism.
/// 
pub fn merge_sort_par<T>(collection: &[T]) -> Vec<T>
where
    T: PartialOrd + Clone + Default + Send + Sync,
{
    if collection.len() > 1 {
        let (l, r) = collection.split_at(collection.len() / 2);
        let (sorted_l, sorted_r) = rayon::join(|| merge_sort_par(l), 
        || merge_sort_par(r));
        sorted_merge(sorted_l, sorted_r)
    } else {
        collection.to_vec()
    }
}
```

1.  很好——你能找到变化吗？为了确保两个版本都能提供相同的结果，让我们添加一些测试：

```rs
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_merge_sort_seq() {
        assert_eq!(merge_sort_seq(&vec![9, 8, 7, 6]), vec![6, 7, 8, 
         9]);
        assert_eq!(merge_sort_seq(&vec![6, 8, 7, 9]), vec![6, 7, 8, 
         9]);
        assert_eq!(merge_sort_seq(&vec![2, 1, 1, 1, 1]), vec![1, 1, 
         1, 1, 2]);
    }

    #[test]
    fn test_merge_sort_par() {
        assert_eq!(merge_sort_par(&vec![9, 8, 7, 6]), vec![6, 7, 8, 
         9]);
        assert_eq!(merge_sort_par(&vec![6, 8, 7, 9]), vec![6, 7, 8, 
         9]);
        assert_eq!(merge_sort_par(&vec![2, 1, 1, 1, 1]), vec![1, 1, 
         1, 1, 2]);
    }
}
```

1.  运行`cargo test`，你应该看到成功的测试：

```rs
$ cargo test
   Compiling use-rayon v0.1.0 (Rust-Cookbook/Chapter04/use-rayon)
    Finished dev [unoptimized + debuginfo] target(s) in 0.67s
     Running target/debug/deps/use_rayon-1fb58536866a2b92

running 2 tests
test tests::test_merge_sort_seq ... ok
test tests::test_merge_sort_par ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests use-rayon

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

1.  然而，我们真正感兴趣的是基准测试——它是否会更快？为此，创建一个包含`seq_vs_par.rs`文件的`benches`文件夹。打开文件并添加以下代码：

```rs
#[macro_use]
extern crate criterion;
use criterion::black_box;
use criterion::Criterion;
use rand::prelude::*;
use std::cell::RefCell;
use use_rayon::{merge_sort_par, merge_sort_seq};

fn random_number_vec(size: usize) -> Vec<i64> {
    let mut v: Vec<i64> = (0..size as i64).collect();
    let mut rng = thread_rng();
    rng.shuffle(&mut v);
    v
}

thread_local!(static ITEMS: RefCell<Vec<i64>> = RefCell::new(random_number_vec(100_000)));

fn bench_seq(c: &mut Criterion) {
    c.bench_function("10k merge sort (sequential)", |b| {
        ITEMS.with(|item| b.iter(||         
        black_box(merge_sort_seq(&item.borrow()))));
    });
}

fn bench_par(c: &mut Criterion) {
    c.bench_function("10k merge sort (parallel)", |b| {
        ITEMS.with(|item| b.iter(|| 
        black_box(merge_sort_par(&item.borrow()))));
    });
}
criterion_group!(benches, bench_seq, bench_par);

criterion_main!(benches);
```

1.  当我们运行`cargo bench`时，我们得到实际的数字来比较并行和顺序实现（变化指的是之前相同基准测试的运行）：

```rs
$ cargo bench
   Compiling use-rayon v0.1.0 (Rust-Cookbook/Chapter04/use-rayon)
    Finished release [optimized] target(s) in 1.84s
     Running target/release/deps/use_rayon-eb085695289744ef

running 2 tests
test tests::test_merge_sort_par ... ignored
test tests::test_merge_sort_seq ... ignored

test result: ok. 0 passed; 0 failed; 2 ignored; 0 measured; 0 filtered out

Running target/release/deps/seq_vs_par-6383ba0d412acb2b
Gnuplot not found, disabling plotting
10k merge sort (sequential) 
                        time: [13.815 ms 13.860 ms 13.906 ms]
                        change: [-6.7401% -5.1611% -3.6593%] (p = 
                         0.00 < 0.05)
                        Performance has improved.
Found 5 outliers among 100 measurements (5.00%)
  3 (3.00%) high mild
  2 (2.00%) high severe

10k merge sort (parallel) 
                        time: [10.037 ms 10.067 ms 10.096 ms]
                        change: [-15.322% -13.276% -11.510%] (p = 
                        0.00 < 0.05)
                        Performance has improved.
Found 6 outliers among 100 measurements (6.00%)
  1 (1.00%) low severe
  1 (1.00%) high mild
  4 (4.00%) high severe

Gnuplot not found, disabling plotting
```

现在，让我们检查所有这些意味着什么，揭开代码的神秘面纱。

# 它是如何工作的...

`rayon-rs` ([`github.com/rayon-rs/rayon`](https://github.com/rayon-rs/rayon)) 是一个流行的数据并行 crate，它只需要少量修改就可以将自动并发引入代码。在我们的例子中，我们使用`rayon::join`操作来创建流行的归并排序算法的并行版本。

在*步骤 1*中，我们添加了基准测试的依赖项（`[dev-dependencies]`）以及实际构建库的依赖项（`[dependencies]`）。但在*步骤 2*和*步骤 3*中，我们实现了一个常规的归并排序变体。一旦我们在*步骤 4*中添加了`rayon`依赖项，我们就可以在*步骤 5*中添加`rayon::join`来并行运行每个分支（到左右部分的排序），如果可能的话，在每个自己的闭包（`|/*no params*/| {/* do work */}`，或简写为`|/*no params*/| /*do work*/`）中并行执行。`join`的文档可以在[`docs.rs/rayon/1.2.0/rayon/fn.join.html`](https://docs.rs/rayon/1.2.0/rayon/fn.join.html)找到，其中详细说明了何时它能加速事情。

在*步骤 8*中，我们正在创建一个符合 criterion 要求的基准测试。库在`src/`目录外编译一个文件，在基准测试框架中运行并输出数字（如*步骤 9*所示）——在这些数字中，我们可以看到仅通过添加一行代码就实现了轻微但一致的性能提升。在基准测试文件中，我们正在对相同随机向量（`thread_local!()`类似于`static`）的 10 万个随机数进行排序。

我们已经成功学习了如何将顺序代码并行化。现在让我们继续到下一个菜谱。

# 向量中的并发数据处理

Rust 的 `Vec` 是一个伟大的数据结构，它不仅用于存储数据，还作为某种管理工具。在本章早期的一个配方（*管理多个线程*）中，我们看到了当我们捕获 `Vec` 中多个线程的句柄，然后使用 `map()` 函数将它们连接起来时的情况。这次，我们将专注于并行处理常规 `Vec` 实例，而不增加额外的开销。在前一个配方中，我们看到了 `rayon-rs` 的强大功能，现在我们将使用它来并行化数据处理。

# 如何做到这一点...

在以下步骤中，让我们更多地使用 `rayon-rs`：

1.  使用 `cargo new concurrent-processing --lib` 创建一个新的项目，并在 Visual Studio Code 中打开它。

1.  首先，我们必须通过在 `Cargo.toml` 中添加几行来添加 `rayon` 作为依赖项。此外，`rand` 包和用于基准测试的 criterion 将在稍后有用，因此让我们也将它们添加并适当配置：

```rs
[dependencies]
rayon = "1.0.3"

[dev-dependencies]
criterion = "0.2.11"
rand = "⁰.5"

[[bench]]
name = "seq_vs_par"
harness = false
```

1.  由于我们将添加一个重要的统计误差度量，即平方误差之和，请打开 `src/lib.rs`。在其顺序版本中，我们简单地遍历预测及其原始值以找出差异，然后平方它，并汇总结果。让我们将此添加到文件中：

```rs
pub fn ssqe_sequential(y: &[f32], y_predicted: &[f32]) -> Option<f32> {
    if y.len() == y_predicted.len() {
        let y_iter = y.iter();
        let y_pred_iter = y_predicted.iter();

        Some(
            y_iter
                .zip(y_pred_iter)
                .map(|(y, y_pred)| (y - y_pred).powi(2))
                .sum()
        ) 
    } else {
        None
    }
}
```

1.  这看起来很容易并行化，而 `rayon` 正好为我们提供了所需的工具。让我们使用并发来创建几乎相同的代码：

```rs
use rayon::prelude::*;

pub fn ssqe(y: &[f32], y_predicted: &[f32]) -> Option<f32> {
    if y.len() == y_predicted.len() {
        let y_iter = y.par_iter();
        let y_pred_iter = y_predicted.par_iter();

        Some(
            y_iter
                .zip(y_pred_iter)
                .map(|(y, y_pred)| (y - y_pred).powi(2))
                .reduce(|| 0.0, |a, b| a + b),
        ) // or sum()
    } else {
        None
    }
}
```

1.  虽然与顺序代码的差异非常微妙，但这些变化对执行速度有重大影响！在我们继续之前，我们应该添加一些测试来查看实际调用函数的结果。让我们先从并行版本开始：

```rs
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_sum_of_sq_errors() {
        assert_eq!(
            ssqe(&[1.0, 1.0, 1.0, 1.0], &[2.0, 2.0, 2.0, 2.0]),
            Some(4.0)
        );
        assert_eq!(
            ssqe(&[-1.0, -1.0, -1.0, -1.0], &[-2.0, -2.0, -2.0, 
             -2.0]),
            Some(4.0)
        );
        assert_eq!(
            ssqe(&[-1.0, -1.0, -1.0, -1.0], &[2.0, 2.0, 2.0, 2.0]),
            Some(36.0)
        );
        assert_eq!(
            ssqe(&[1.0, 1.0, 1.0, 1.0], &[2.0, 2.0, 2.0, 2.0]),
            Some(4.0)
        );
        assert_eq!(
            ssqe(&[1.0, 1.0, 1.0, 1.0], &[2.0, 2.0, 2.0, 2.0]),
            Some(4.0)
        );
    }
```

1.  顺序代码应该有相同的结果，因此让我们复制顺序代码版本的测试：

```rs
    #[test]
    fn test_sum_of_sq_errors_seq() {
        assert_eq!(
            ssqe_sequential(&[1.0, 1.0, 1.0, 1.0], &[2.0, 2.0, 2.0, 
             2.0]),
            Some(4.0)
        );
        assert_eq!(
            ssqe_sequential(&[-1.0, -1.0, -1.0, -1.0], &[-2.0,
             -2.0, -2.0, -2.0]),
            Some(4.0)
        );
        assert_eq!(
            ssqe_sequential(&[-1.0, -1.0, -1.0, -1.0], &[2.0, 2.0, 
             2.0, 2.0]),
            Some(36.0)
        );
        assert_eq!(
            ssqe_sequential(&[1.0, 1.0, 1.0, 1.0], &[2.0, 2.0, 2.0, 
             2.0]),
            Some(4.0)
        );
        assert_eq!(
            ssqe_sequential(&[1.0, 1.0, 1.0, 1.0], &[2.0, 2.0, 2.0, 
             2.0]),
            Some(4.0)
        );
    }
}
```

1.  为了检查一切是否按预期工作，请在之间运行 `cargo test`：

```rs
$ cargo test
   Compiling concurrent-processing v0.1.0 (Rust-
    Cookbook/Chapter04/concurrent-processing)
    Finished dev [unoptimized + debuginfo] target(s) in 0.84s
     Running target/debug/deps/concurrent_processing-
      250eef41459fd2af

running 2 tests
test tests::test_sum_of_sq_errors_seq ... ok
test tests::test_sum_of_sq_errors ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests concurrent-processing

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

1.  作为 `rayon` 的一个额外功能，让我们也在 `src/lib.rs` 中添加一些更多函数。这次，它们与在 `str` 中计数字母数字字符相关：

```rs
pub fn seq_count_alpha_nums(corpus: &str) -> usize {
    corpus.chars().filter(|c| c.is_alphanumeric()).count()
}

pub fn par_count_alpha_nums(corpus: &str) -> usize {
    corpus.par_chars().filter(|c| c.is_alphanumeric()).count()
}
```

1.  现在，让我们看看哪个性能更好，并添加一个基准测试。为此，在 `src/` 旁边创建一个 `benches/` 目录，并添加一个 `seq_vs_par.rs` 文件。添加以下基准测试和辅助函数以查看速度提升。让我们从定义基准测试处理的基本数据的几个辅助函数开始：

```rs
#[macro_use]
extern crate criterion;
use concurrent_processing::{ssqe, ssqe_sequential, seq_count_alpha_nums, par_count_alpha_nums};
use criterion::{black_box, Criterion};
use std::cell::RefCell;
use rand::prelude::*;

const SEQ_LEN: usize = 1_000_000;
thread_local!(static ITEMS: RefCell<(Vec<f32>, Vec<f32>)> = {
    let y_values: (Vec<f32>, Vec<f32>) = (0..SEQ_LEN).map(|_| 
     (random::<f32>(), random::<f32>()) )
    .unzip();
    RefCell::new(y_values)
});

const MAX_CHARS: usize = 100_000;
thread_local!(static CHARS: RefCell<String> = {
    let items: String = (0..MAX_CHARS).map(|_| random::<char>
     ()).collect();
    RefCell::new(items)
});
```

1.  接下来，我们将创建基准测试本身：

```rs
fn bench_count_seq(c: &mut Criterion) {
    c.bench_function("Counting in sequence", |b| {
        CHARS.with(|item| b.iter(|| 
         black_box(seq_count_alpha_nums(&item.borrow()))))
    });
}

fn bench_count_par(c: &mut Criterion) {
    c.bench_function("Counting in parallel", |b| {
        CHARS.with(|item| b.iter(|| 
         black_box(par_count_alpha_nums(&item.borrow()))))
    });
}
```

1.  让我们创建另一个基准测试：

```rs
fn bench_seq(c: &mut Criterion) {
    c.bench_function("Sequential vector operation", |b| {
        ITEMS.with(|y_values| {
            let y_borrowed = y_values.borrow();
            b.iter(|| black_box(ssqe_sequential(&y_borrowed.0, 
             &y_borrowed.1)))
        })
    });
}

fn bench_par(c: &mut Criterion) {
    c.bench_function("Parallel vector operation", |b| {
        ITEMS.with(|y_values| {
            let y_borrowed = y_values.borrow();
            b.iter(|| black_box(ssqe(&y_borrowed.0, 
            &y_borrowed.1)))
        })
    });
}

criterion_group!(benches, bench_seq, bench_par,bench_count_par, bench_count_seq);

criterion_main!(benches);
```

1.  有此可用，运行 `cargo bench` 并（经过一段时间后）检查输出以查看改进和计时（变化的部分指的是与上次运行相同基准测试的变化）：

```rs
$ cargo bench
   Compiling concurrent-processing v0.1.0 (Rust-
    Cookbook/Chapter04/concurrent-processing)
    Finished release [optimized] target(s) in 2.37s
     Running target/release/deps/concurrent_processing-
      eedf0fd3b1e51fe0

running 2 tests
test tests::test_sum_of_sq_errors ... ignored
test tests::test_sum_of_sq_errors_seq ... ignored

test result: ok. 0 passed; 0 failed; 2 ignored; 0 measured; 0 filtered out

Running target/release/deps/seq_vs_par-ddd71082d4bd9dd6
Gnuplot not found, disabling plotting
Sequential vector operation 
                        time: [1.0631 ms 1.0681 ms 1.0756 ms]
                        change: [-4.8191% -3.4333% -2.3243%] (p = 
                        0.00 < 0.05)
                        Performance has improved.
Found 4 outliers among 100 measurements (4.00%)
  2 (2.00%) high mild
  2 (2.00%) high severe

Parallel vector operation 
                        time: [408.93 us 417.14 us 425.82 us]
                        change: [-9.5623% -6.0044% -2.2126%] (p = 
                        0.00 < 0.05)
                        Performance has improved.
Found 15 outliers among 100 measurements (15.00%)
  2 (2.00%) low mild
  7 (7.00%) high mild
  6 (6.00%) high severe

Counting in parallel time: [552.01 us 564.97 us 580.51 us] 
                        change: [+2.3072% +6.9101% +11.580%] (p = 
                        0.00 < 0.05)
                        Performance has regressed.
Found 4 outliers among 100 measurements (4.00%)
  3 (3.00%) high mild
  1 (1.00%) high severe

Counting in sequence time: [992.84 us 1.0137 ms 1.0396 ms] 
                        change: [+9.3014% +12.494% +15.338%] (p = 
                        0.00 < 0.05)
                        Performance has regressed.
Found 4 outliers among 100 measurements (4.00%)
  4 (4.00%) high mild

Gnuplot not found, disabling plotting
```

现在，让我们深入了解代码以更好地理解它。

# 它是如何工作的...

再次强调，`rayon-rs`——一个出色的库——通过更改**单行代码**实现了基准性能的大约 50% 的提升（并行与顺序）。这对于许多应用程序来说都是重要的，尤其是在机器学习中，算法的损失函数在训练周期中需要运行数百或数千次。将此时间减半将立即对生产力产生重大影响。

在设置好一切后的第一步（*步骤 3*、*步骤 4* 和 *步骤 5*），我们正在创建平方误差和的顺序和并行实现（[`hlab.stanford.edu/brian/error_sum_of_squares.html`](https://hlab.stanford.edu/brian/error_sum_of_squares.html)），唯一的区别是 `par_iter()` 与包括一些测试的 `iter()` 调用。然后我们添加一些——更常见的——计数函数到我们的基准测试套件中，我们将在 *步骤 7* 和 *步骤 8* 中创建和调用它们。再次强调，顺序和并行算法每次都在完全相同的数据集上工作，以避免任何不幸的事件。

我们已经成功学习了如何在向量中并发处理数据。现在让我们继续到下一个菜谱。

# 共享不可变状态

有时，当程序在多个线程上运行时，当前版本设置以及更多内容都作为单一的真实点提供给线程。在 Rust 中，在变量不可变且类型被标记为安全共享的情况下，线程之间的状态共享是直接的。为了标记类型为线程安全，实现必须确保访问信息时不会发生任何不一致。

Rust 使用两个标记特质——`Send` 和 `Sync`——来管理这些选项。让我们看看如何。

# 如何做到这一点...

在几个步骤中，我们将探索不可变状态：

1.  运行 `cargo new immutable-states` 以创建一个新的应用程序项目，并在 Visual Studio Code 中打开该目录。

1.  首先，我们将添加导入和一个 `noop` 函数到我们的 `src/main.rs` 文件中：

```rs
use std::thread;
use std::rc::Rc;
use std::sync::Arc;
use std::sync::mpsc::channel;

fn noop<T>(_: T) {}
```

1.  让我们探索不同类型如何在线程之间共享。`mpsc::channel` 类型提供了一个很好的现成示例，用于共享状态。让我们从一个按预期工作的基线开始：

```rs
fn main() {
    let (sender, receiver) = channel::<usize>();

    thread::spawn(move || {
        let thread_local_read_only_clone = sender.clone();
        noop(thread_local_read_only_clone);
    });
}
```

1.  要查看其工作情况，请执行 `cargo build`。任何关于非法状态共享的错误都将由编译器找到：

```rs
$ cargo build
   Compiling immutable-states v0.1.0 (Rust-Cookbook/Chapter04
   /immutable-states)
warning: unused import: `std::rc::Rc`
 --> src/main.rs:2:5
  |
2 | use std::rc::Rc;
  | ^^^^^^^^^^^
  |
  = note: #[warn(unused_imports)] on by default

warning: unused import: `std::sync::Arc`
 --> src/main.rs:3:5
  |
3 | use std::sync::Arc;
  | ^^^^^^^^^^^^^^

warning: unused variable: `receiver`
  --> src/main.rs:10:18
   |
10 | let (sender, receiver) = channel::<usize>();
   | ^^^^^^^^ help: consider prefixing with an underscore: `_receiver`
   |
   = note: #[warn(unused_variables)] on by default

    Finished dev [unoptimized + debuginfo] target(s) in 0.58s
```

1.  现在，我们将尝试用接收者做同样的事情。它会工作吗？将以下内容添加到 `main` 函数中：

```rs
    let c = Arc::new(receiver);
    thread::spawn(move || {
        noop(c.clone());
    });
```

1.  运行 `cargo build` 以获取更详细的消息：

```rs
$ cargo build
   Compiling immutable-states v0.1.0 (Rust-Cookbook/Chapter04
    /immutable-states)
warning: unused import: `std::rc::Rc`
 --> src/main.rs:2:5
  |
2 | use std::rc::Rc;
  | ^^^^^^^^^^^
  |
  = note: #[warn(unused_imports)] on by default

error[E0277]: `std::sync::mpsc::Receiver<usize>` cannot be shared between threads safely
  --> src/main.rs:26:5
   |
26 | thread::spawn(move || {
   | ^^^^^^^^^^^^^ `std::sync::mpsc::Receiver<usize>` cannot be shared between threads safely
   |
   = help: the trait `std::marker::Sync` is not implemented for `std::sync::mpsc::Receiver<usize>`
   = note: required because of the requirements on the impl of `std::marker::Send` for `std::sync::Arc<std::sync::mpsc::Receiver<usize>>`
   = note: required because it appears within the type `[closure@src/main.rs:26:19: 28:6 c:std::sync::Arc<std::sync::mpsc::Receiver<usize>>]`
   = note: required by `std::thread::spawn`

error: aborting due to previous error

For more information about this error, try `rustc --explain E0277`.
error: Could not compile `immutable-states`.

To learn more, run the command again with --verbose.
```

1.  由于接收者只为单个线程设计，用于从通道中获取数据，因此预期使用 `Arc` 无法避免这种情况。同样，也不可能简单地将 `Rc` 包装到 `Arc` 中，使其能够在线程之间可用。添加以下内容以查看错误：

```rs
   let b = Arc::new(Rc::new(vec![]));
    thread::spawn(move || {
        let thread_local_read_only_clone = b.clone();
        noop(thread_local_read_only_clone);
    });
```

1.  `cargo build` 再次揭示了后果——关于类型无法在线程之间发送的错误：

```rs
$ cargo build
   Compiling immutable-states v0.1.0 (Rust-Cookbook/Chapter04
   /immutable-states)
error[E0277]: `std::rc::Rc<std::vec::Vec<_>>` cannot be sent between threads safely
  --> src/main.rs:19:5
   |
19 | thread::spawn(move || {
   | ^^^^^^^^^^^^^ `std::rc::Rc<std::vec::Vec<_>>` cannot be sent between threads safely
   |
   = help: the trait `std::marker::Send` is not implemented for `std::rc::Rc<std::vec::Vec<_>>`
   = note: required because of the requirements on the impl of `std::marker::Send` for `std::sync::Arc<std::rc::Rc<std::vec::Vec<_>>>`
   = note: required because it appears within the type `[closure@src/main.rs:19:19: 22:6 b:std::sync::Arc<std::rc::Rc<std::vec::Vec<_>>>]`
   = note: required by `std::thread::spawn`

error[E0277]: `std::rc::Rc<std::vec::Vec<_>>` cannot be shared between threads safely
  --> src/main.rs:19:5
   |
19 | thread::spawn(move || {
   | ^^^^^^^^^^^^^ `std::rc::Rc<std::vec::Vec<_>>` cannot be shared between threads safely
   |
   = help: the trait `std::marker::Sync` is not implemented for `std::rc::Rc<std::vec::Vec<_>>`
   = note: required because of the requirements on the impl of `std::marker::Send` for `std::sync::Arc<std::rc::Rc<std::vec::Vec<_>>>`
   = note: required because it appears within the type `[closure@src/main.rs:19:19: 22:6 b:std::sync::Arc<std::rc::Rc<std::vec::Vec<_>>>]`
   = note: required by `std::thread::spawn`

error: aborting due to 2 previous errors

For more information about this error, try `rustc --explain E0277`.
error: Could not compile `immutable-states`.

To learn more, run the command again with --verbose. 
```

现在，让我们深入了解代码，以更好地理解它。

# 它是如何工作的...

由于这个菜谱实际上在最后一步未能构建，并指向了一个错误信息，发生了什么？我们学习了 `Send` 和 `Sync`。这些标记特性和错误类型将在最令人惊讶和关键的情况下出现在你的路径上。由于它们在存在时无缝工作，我们不得不创建一个失败的示例来向你展示它们所做的事情以及如何做到这一点。

在 Rust 中，标记特性([`doc.rust-lang.org/std/marker/index.html`](https://doc.rust-lang.org/std/marker/index.html))向编译器发出信号。在并发的情况下，这是跨线程共享的能力。`Sync`（多线程共享访问）和 `Send`（所有者可以安全地从一条线程转移到另一条线程）特性几乎为所有默认数据结构实现，但如果需要 `unsafe` 代码，则必须手动添加标记特性——这也是 `unsafe`。

因此，大多数数据结构都将能够从其属性中继承 `Send` 和 `Sync`，这就是在 *步骤 2* 和 *步骤 3* 中发生的事情。通常，你还会将实例包裹在 `Arc` 中以便更容易处理。然而，多个 `Arc` 实例需要其包含的类型实现 `Send` 和 `Sync`。在 *步骤 4* 和 *步骤 6* 中，我们尝试将可用的类型放入 `Arc` 中——而不实现 `Sync` 或 `Send`。*步骤 5* 和 *步骤 7* 展示了尝试时编译器的错误信息。如果你想了解更多，并了解如何将 `marker` 特性([`doc.rust-lang.org/std/marker/index.html`](https://doc.rust-lang.org/std/marker/index.html))添加到自定义类型中，请查看[`doc.rust-lang.org/nomicon/send-and-sync.html`](https://doc.rust-lang.org/nomicon/send-and-sync.html)上的文档。

现在我们对 `Send` 和 `Sync` 了解更多，并发程序中的状态共享就不再是谜了。让我们继续到下一个菜谱。

# 使用演员处理异步消息

可扩展架构和异步编程导致了演员和基于演员的设计([`mattferderer.com/what-is-the-actor-model-and-when-should-you-use-it`](https://mattferderer.com/what-is-the-actor-model-and-when-should-you-use-it))的兴起，这些设计得到了 Akka ([`akka.io/`](https://akka.io/)) 等框架的促进。尽管 Rust 拥有强大的并发功能，但 Rust 中的演员仍然难以正确使用，并且缺乏许多其他库所拥有的文档。在这个菜谱中，我们将探索 Rust 的演员框架 `actix` 的基础知识，该框架是在流行的 Akka 之后创建的。

# 如何做到这一点...

只需几个步骤即可实现基于演员的传感器数据读取器：

1.  使用 `cargo new actors` 创建一个新的二进制应用程序，并在 Visual Studio Code 中打开该目录。

1.  在 `Cargo.toml` 配置文件中包含所需的依赖项：

```rs
[package]
name = "actors"
version = "0.1.0"
authors = ["Claus Matzinger <claus.matzinger+kb@gmail.com>"]
edition = "2018"

[dependencies]
actix = "⁰.8"
rand = "0.5"
```

1.  打开 `src/main.rs` 以在 `main` 函数之前添加代码。让我们从导入开始：

```rs
use actix::prelude::*;
use std::thread;
use std::time::Duration;
use rand::prelude::*;
```

1.  为了创建一个演员系统，我们必须考虑应用程序的结构。演员可以被视为一个消息接收器，其中有一个邮箱，消息堆积在那里直到被处理。为了简单起见，让我们模拟一些传感器数据作为消息，每个消息由一个`u64`时间戳和一个`f32`值组成：

```rs
///
/// A mock sensor function
/// 
fn read_sensordata() -> f32 {
     random::<f32>() * 10.0
}

#[derive(Debug, Message)]
struct Sensordata(pub u64, pub f32);
```

1.  在一个典型系统中，我们会使用 I/O 循环以预定的时间间隔从传感器（s）中读取。由于`actix`([`github.com/actix/actix/`](https://github.com/actix/actix/))建立在 Tokio([`tokio.rs/`](https://tokio.rs/))之上，这部分内容可以在这个菜谱之外探索。为了模拟快速读取和慢速处理步骤，我们将它实现为一个`for`循环：

```rs
fn main() -> std::io::Result<()> {
    System::run(|| {
        println!(">> Press Ctrl-C to stop the program");
        // start multi threaded actor host (arbiter) with 2 threads
        let sender = SyncArbiter::start(N_THREADS, || 
        DBWriter);

        // send messages to the actor 
        for n in 0..10_000 {
            let my_timestamp = n as u64;
            let data = read_sensordata();
            sender.do_send(Sensordata(my_timestamp, data));
        }
    })
}
```

1.  让我们来处理最重要的部分：演员的消息处理。`actix`要求您实现`Handler<T>`特质。在`main`函数之前添加以下实现：

```rs
struct DBWriter;

impl Actor for DBWriter {
    type Context = SyncContext<Self>;
}

impl Handler<Sensordata> for DBWriter {
    type Result = ();

    fn handle(&mut self, msg: Sensordata, _: &mut Self::Context) -> 
     Self::Result {

        // send stuff somewhere and handle the results
        println!(" {:?}", msg);
        thread::sleep(Duration::from_millis(300));
    }
}
```

1.  使用`cargo run`来运行程序，看看它是如何生成人工传感器数据的（如果您不想等待它完成，请按*Ctrl + C*）：

```rs
$ cargo run
   Compiling actors v0.1.0 (Rust-Cookbook/Chapter04/actors)
    Finished dev [unoptimized + debuginfo] target(s) in 2.05s
     Running `target/debug/actors`
>> Press Ctrl-C to stop the program
  Sensordata(0, 2.2577233)
  Sensordata(1, 4.039347)
  Sensordata(2, 8.981095)
  Sensordata(3, 1.1506838)
  Sensordata(4, 7.5091066)
  Sensordata(5, 2.5614727)
  Sensordata(6, 3.6907816)
  Sensordata(7, 7.907603)
  ^C⏎    
```

现在，让我们幕后了解代码，以便更好地理解。

# 它是如何工作的...

演员模型通过面向对象的方法解决了在线程间传递数据的不足。通过利用演员之间消息的隐式队列，它可以防止昂贵的锁定和损坏状态。关于这个主题有大量的内容，例如，在 Akka 的文档中[`doc.akka.io/docs/akka/current/guide/actors-intro.html`](https://doc.akka.io/docs/akka/current/guide/actors-intro.html)。

在前两个步骤准备项目之后，*步骤 3*展示了使用宏（`[#derive()]`）实现`Message`特质的代码。有了这个，我们继续设置主*系统*——运行演员调度和幕后消息传递的主循环。

`actix`使用`Arbiters`来运行不同的演员和任务。一个常规的仲裁者基本上是一个单线程的事件循环，有助于在非并发环境中工作。另一方面，`SyncArbiter`是一个多线程版本，允许跨线程使用演员。在我们的例子中，我们使用了三个线程。

在*步骤 5*中，我们看到了处理器的最小实现要求。使用`SyncArbiter`不允许通过返回值发送消息，这就是为什么结果现在是一个空元组。处理器也针对特定的消息类型，处理函数通过发出`thread::sleep`来模拟长时间运行的操作——这之所以有效，是因为它是唯一在该特定线程中运行的演员。

我们只是触及了`actix`能做什么的表面（省略了全能的 Tokio 任务和流）。查看他们关于这个主题的书([`actix.rs/book/actix/`](https://actix.rs/book/actix/))以及他们在 GitHub 存储库中的示例[.](https://tokio.rs)

我们已经成功地学习了如何使用演员处理异步消息。现在让我们继续下一个菜谱。

# 使用 futures 进行异步编程

在 JavaScript、TypeScript、C# 和类似技术中，使用 futures 是一种常见的技巧——通过在它们的语法中添加 `async`/`await` 关键字而变得流行。简而言之，futures（或承诺）是一个函数的保证，在某个时刻，句柄将被解决，并将返回实际值。然而，并没有明确的时间表明这将会发生——但你可以安排一系列承诺，这些承诺依次解决。在 Rust 中这是如何工作的？让我们在这个食谱中找出答案。

在撰写本文时，`async`/`await` 正在经历重大开发。根据你阅读这本书的时间，示例可能已经停止工作。在这种情况下，我们要求你在配套的存储库中打开一个问题，这样我们就可以修复这些问题。有关更新，请查看 Rust `async` 工作组的存储库 [`github.com/rustasync/team`](https://github.com/rustasync/team)。

# 如何做到这一点...

在几个步骤中，我们将能够在 Rust 中使用 `async` 和 `await` 以实现无缝并发：

1.  使用 `cargo new async-await` 创建一个新的二进制应用程序，并在 Visual Studio Code 中打开该目录。

1.  如同往常，当我们集成库时，我们必须将依赖项添加到 `Cargo.toml`：

```rs
[package]
name = "async-await"
version = "0.1.0"
authors = ["Claus Matzinger <claus.matzinger+kb@gmail.com>"]
edition = "2018"

[dependencies]
runtime = "0.3.0-alpha.6"
surf = "1.0"
```

1.  在 `src/main.rs` 中，我们必须导入依赖项。在文件顶部添加以下行：

```rs
use surf::Exception;
use surf::http::StatusCode;
```

1.  经典的例子是等待一个网络请求完成。这通常很难判断，因为网络资源和/或中间的网络可能由其他人拥有，并且可能已经关闭。`surf` ([`github.com/rustasync/surf`](https://github.com/rustasync/surf)) 默认为 `async`，因此需要大量使用 `.await` 语法。让我们声明一个 `async` 函数来进行获取：

```rs
async fn response_code(url: &str) -> Result<StatusCode, Exception> {
    let res = surf::get(url).await?;
    Ok(res.status())
}
```

1.  现在我们需要一个 `async main` 函数来调用 `response_code() async` 函数：

```rs
#[runtime::main]
async fn main() -> Result<(), Exception> {
    let url = "https://www.rust-lang.org";
    let status = response_code(url).await?;
    println!("{} responded with HTTP {}", url, status);
    Ok(())
}
```

1.  让我们通过运行 `cargo run` 来看看代码是否工作（预期结果是 `200 OK`）：

```rs
$ cargo +nightly run
 Compiling async-await v0.1.0 (Rust-Cookbook/Chapter04/async-await)
    Finished dev [unoptimized + debuginfo] target(s) in 1.81s
     Running `target/debug/async-await`
     https://www.rust-lang.org responded with HTTP 200 OK
```

`async` 和 `await` 在 Rust 社区中已经讨论了很长时间。让我们看看这个食谱是如何工作的。

# 它是如何工作的...

Futures（通常称为承诺）通常完全集成到语言中，并带有内置的运行时。在 Rust 中，团队选择了一种更雄心勃勃的方法，并将运行时留给社区来实现（目前是这样）。目前，两个项目 Tokio 和 Romio ([`github.com/withoutboats/romio`](https://github.com/withoutboats/romio)) 以及 `juliex` ([`github.com/withoutboats/juliex`](https://github.com/withoutboats/juliex)) 为这些 futures 提供了最复杂的支持。随着 2018 版本中 Rust 语法中 `async`/`await` 的最近添加，各种实现成熟只是时间问题。

在 *第 1 步* 中设置好依赖项后，*第 2 步* 显示我们不需要启用 `async` 和 `await` 宏/语法就可以在代码中使用它们——这曾经是一个长期的要求。然后，我们导入所需的包。巧合的是，当我们在忙于这本书时，Rust 异步工作组构建了一个新的异步网络库——称为 `surf`。由于这个包是完全异步构建的，我们更倾向于使用它而不是更成熟的包，如 `hyper` ([`hyper.rs`](https://hyper.rs))。

在 *第 3 步* 中，我们声明了一个 `async` 函数，它自动返回一个 `Future` ([`doc.rust-lang.org/std/future/trait.Future.html`](https://doc.rust-lang.org/std/future/trait.Future.html)) 类型，并且只能在其他 `async` 范围内调用。*第 4 步* 展示了如何使用 `async main` 函数创建这样的范围。这就结束了吗？不——`#[runtime::main]` 属性揭示了这一点：运行时无缝启动并分配执行任何异步操作。

虽然 `runtime` 包 ([`docs.rs/runtime/0.3.0-alpha.7/runtime/`](https://docs.rs/runtime/0.3.0-alpha.7/runtime/)) 对实际实现不敏感，但默认情况下是基于 `romio` 和 `juliex` 的本地运行时（检查您的 `Cargo.lock` 文件），但您也可以启用功能更丰富的 [tokio](https://tokio.rs) 运行时，以启用流、定时器等功能，用于异步操作之上。

在 `async` 函数内部，我们可以使用附加到 `Future` 实现者的 `await` 关键字 ([`doc.rust-lang.org/std/future/trait.Future.html`](https://doc.rust-lang.org/std/future/trait.Future.html))，例如 `surf` 请求 ([`github.com/rustasync/surf/blob/master/src/request.rs#L563`](https://github.com/rustasync/surf/blob/master/src/request.rs#L563))，其中运行时会调用 `poll()` 直到结果可用。这也可能导致错误，这意味着我们必须处理错误，这通常使用 `?` 操作符来完成。`surf` 还提供了一个通用的 `Exception` 类型 ([`docs.rs/surf/1.0.2/surf/type.Exception.html`](https://docs.rs/surf/1.0.2/surf/type.Exception.html)) 别名来处理可能发生的一切。

尽管在 Rust 快速发展的生态系统中还有一些事情可能会发生变化，但使用 `async`/`await` 现在终于可以一起使用，而无需高度不稳定的包。拥有这一点对 Rust 的实用性是一个重大的提升。现在，让我们继续到另一个章节。
