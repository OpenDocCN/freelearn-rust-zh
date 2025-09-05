# 第八章：实现并发

并发是指同时做两件事的行为。在单核处理器上，这意味着 **多任务处理**。在多任务处理时，操作系统将在运行进程之间切换，以便每个进程都能获得处理器使用的时间份额。在多核处理器上，并发进程可以同时运行。

在本章中，我们将探讨不同的并发模型。其中一些工具与实际应用相关，而其他工具则更多用于教育目的。在这里，我们推荐并解释了并发中的线程模型。此外，我们还将解释如何使用函数式设计模式使开发有效使用并发的程序变得更加容易。

学习成果将包括以下内容：

+   适当地识别并应用子进程并发

+   理解 nix 分叉并发模型及其优势

+   适当地识别并应用线程并发

+   理解 Rust 原始 `Send` 和 `Sync` 特性

+   识别并应用演员设计模式

# 技术要求

运行提供的示例需要 Rust 的最新版本：

[`www.rust-lang.org/en-US/install.html`](https://www.rust-lang.org/en-US/install.html)

本章的代码也可在 GitHub 上找到：

[`github.com/PacktPublishing/Hands-On-Functional-Programming-in-RUST`](https://github.com/PacktPublishing/Hands-On-Functional-Programming-in-RUST)

每个章节的 `README.md` 文件中也包含了具体的安装和构建说明。

# 使用子进程并发

子进程是从另一个进程中启动的命令。作为一个简单的例子，让我们创建一个有三个子进程的父进程。`process_a` 将是父进程。考虑以下代码片段：

```rs
use std::process::Command;
use std::env::current_exe;

fn main() {
   let path = current_exe()
             .expect("could not find current executable");
   let path = path.with_file_name("process_b");

   let mut children = Vec::new();
   for _ in 0..3 {
      children.push(
         Command::new(path.as_os_str())
                 .spawn()
                 .expect("failed to execute process")
      );
   }
   for mut c in children {
      c.wait()
       .expect("failed to wait on child");
   }
}
```

子进程 `process_b` 运行一个循环并打印其自己的进程 ID。如下所示：

```rs
use std::{thread,time};
use std::process;
fn main() {
   let t = time::Duration::from_millis(1000);
   loop {
      println!("process b #{}", process::id());
      thread::sleep(t);
   }
}
```

如果你运行 `process_a`，那么你将看到来自三个 `process_b` 进程的输出：

```rs
process b #54061
process b #54060
process b #54059
process b #54061
process b #54059
process b #54060
```

如果你从 `process_a` 开始检查进程树，那么你将发现三个 `process_b` 进程作为子进程附加，如下面的代码所示：

```rs
$ ps -a | grep process_a
54058 ttys001    0:00.00 process_a
55093 ttys004    0:00.00 grep process_a
$ pstree 54058
54058 process_a
>   54059 process_b
>   54060 process_b
>   54061 process_b
```

检查进程树的先前命令需要 Unix-like 命令提示符。然而，子进程模块本身基本上是平台无关的。

子进程并发对于想要运行和管理其他项目或实用程序非常有用。子进程并发做得很好的一个例子是 `cron` 实用程序。`cron` 接受一个配置文件，该文件指定了要运行的不同命令以及运行它们的时间表。`cron` 会持续在后台运行，并在适当的时间根据时间表启动每个配置的进程。

子进程并发通常不适合并行计算。使用 `subprocess::Command` 接口时，父进程和子进程之间不会共享任何资源。此外，这些进程之间难以轻松共享信息。

# 理解 nix 分叉并发

在 1995 年将线程作为 POSIX 操作系统的标准之前，可用的最佳并发选项是`fork`。在这些操作系统中，`fork`是一个相当原始的命令，允许程序作为子进程创建自己的副本。`fork`这个名字来源于将一个进程分成两个的想法。

`fork`不是平台无关的，具体来说，它在 Windows 上不可用，我们建议使用线程。然而，出于教育目的，介绍一些`fork`的概念是有帮助的，因为这些概念也与线程编程相关。

以下代码是将先前的`process_a`、`process_b`示例翻译成使用`fork`的代码：

```rs
extern crate nix;
use nix::unistd::{fork,ForkResult};
use std::{thread,time};
use std::process;

fn main() {
   let mut children = Vec::new();
   for _ in 0..3 {
      match fork().expect("fork failed") {
         ForkResult::Parent{ child: pid } => { children.push(pid); }
         ForkResult::Child => {
            let t = time::Duration::from_millis(1000);
            loop {
               println!("child process #{}", process::id());
               thread::sleep(t);
            }
         }
      }
   }
   let t = time::Duration::from_millis(1000);
   loop {
      println!("parent process #{}", process::id());
      thread::sleep(t);
   }
}
```

在这个例子中，父-子关系与我们的第一个例子非常相似。我们有三个子进程在运行，一个父进程在管理它们。

应该注意的是，初始时，被`fork`的进程共享内存。只有当任一进程修改其内存时，操作系统才会执行一个称为**写时复制**的操作，复制内存。这种行为是运行进程之间共享内存的第一步。

为了演示写时复制，让我们分配 200 MB 的内存并`fork` 500 个进程。如果没有写时复制，这将需要 100 GB 的内存，并且会崩溃大多数个人电脑。考虑以下代码：

```rs
extern crate nix;
use nix::unistd::{fork};
use std::{thread,time};

fn main() {
   let mut big_data: Vec<u8> = Vec::with_capacity(200000000);
   big_data.push(1);
   big_data.push(2);
   big_data.push(3);
   //Both sides of the fork, will continue to fork
   //This is called a fork bomb
   for _ in 0..9 {
      fork().expect("fork failed");
   }
   //2⁹ = 512

   let t = time::Duration::from_millis(1000);
   loop {
      //copy on write, not on read
      big_data[2];
      thread::sleep(t);
   }
}
```

父进程的许多资源也仍然可用，并且可以从子进程中安全地使用。这对于在父进程中监听套接字并在子进程中轮询传入连接的服务器应用程序非常有用。这个简单的技巧允许服务器应用程序在工作进程之间分配工作：

```rs
extern crate nix;
use nix::unistd::{fork,ForkResult};
use std::{thread,time};
use std::process;
use std::io::prelude::*;
use std::net::TcpListener;

fn serve(listener: TcpListener) -> ! {
   for stream in listener.incoming() {
      let mut buffer = [0; 2048];
      let mut tcp = stream.unwrap();
      tcp.read(&mut buffer).expect("tcp read failed");
      let response = format!("respond from #{}\n", process::id());
      tcp.write(response.as_bytes()).expect("tcp write failed");
   }
   panic!("unreachable");
}

fn main() {
   let listener = TcpListener::bind("127.0.0.1:8888").unwrap();
   let mut children = Vec::new();
   for _ in 0..3 {
      match fork().expect("fork failed") {
         ForkResult::Parent{ child: pid } => { children.push(pid); }
         ForkResult::Child => { serve(listener) }
      }
   }

   let t = time::Duration::from_millis(1000);
   loop {
      thread::sleep(t);
   }
}
```

在这个例子中，我们开始监听端口号`8888`的连接。然后，在三次`fork`之后，我们开始用我们的工作进程提供响应。向服务器发送请求，我们可以确认确实有多个进程在竞争提供服务。考虑以下代码：

```rs
$ curl 'http://localhost:8888/'
respond from #59485
$ curl 'http://localhost:8888/'
respond from #59486
$ curl 'http://localhost:8888/'
respond from #59487
$ curl 'http://localhost:8888/'
respond from #59485
$ curl 'http://localhost:8888/'
respond from #59486
```

所有三个工作进程至少提供了一次响应。结合内存共享的第一种策略和这种新的内置负载均衡概念，`fork`进程有效地解决了许多需要并发性的常见问题。

然而，`fork`并发模型非常僵化。这两个技巧都需要在分配资源后战略性地`fork`应用程序。一旦进程被分割，`fork`就完全无助于解决问题。在 POSIX 中，已经创建了额外的标准来解决这一问题。通过通道发送信息或共享内存是一种常见的模式，就像在 Rust 中一样。然而，这些解决方案没有一个像线程那样实用。

线程隐式地允许进程间消息传递和内存共享。线程的风险是，共享消息或内存可能不是线程安全的，可能会导致内存损坏。Rust 从头开始构建，以使线程编程更安全。

# 使用线程并发

Rust 线程具有以下特性：

+   共享内存

+   共享资源，例如文件或套接字

+   通常具有线程安全性

+   支持线程间消息传递

+   具有平台无关性

由于上述原因，我们建议 Rust 线程比子进程更适合大多数并发用例。如果您想分发计算、绕过阻塞操作或以其他方式利用并发来为您的应用程序提供服务——请使用线程。

为了展示线程模式，我们可以重新实现前面的示例。以下是三个子线程：

```rs
use std::{thread,time};
use std::process;
extern crate thread_id;

fn main() {
   for _ in 0..3 {
      thread::spawn(|| {
         let t = time::Duration::from_millis(1000);
         loop {
            println!("child thread #{}:{}", process::id(), 
       thread_id::get());
            thread::sleep(t);
         }
      });
   }
   let t = time::Duration::from_millis(1000);
   loop {
      println!("parent thread #{}:{}", process::id(), 
      thread_id::get());
      thread::sleep(t);
   }
}
```

在这里，我们启动了三个线程，并让它们运行。我们打印进程 ID，但我们必须也打印线程 ID，因为线程共享相同的进程 ID。以下是演示这一点的输出：

```rs
parent thread #59804:140735902303104
child thread #59804:123145412530176
child thread #59804:123145410420736
child thread #59804:123145408311296
parent thread #59804:140735902303104
child thread #59804:123145410420736
child thread #59804:123145408311296
```

下一个要移植的例子是 500 个进程和共享内存。在一个线程程序中，共享可能看起来像以下代码片段：

```rs
use std::{thread,time};
use std::sync::{Mutex, Arc};

fn main() {
   let mut big_data: Vec<u8> = Vec::with_capacity(200000000);
   big_data.push(1);
   big_data.push(2);
   big_data.push(3);
   let big_data = Arc::new(Mutex::new(big_data));
   for _ in 0..512 {
      let big_data = Arc::clone(&big_data);
      thread::spawn(move || {
         let t = time::Duration::from_millis(1000);
         loop {
            let d = big_data.lock().unwrap();
            (*d)[2];
            thread::sleep(t);
         }
      });
   }
   let t = time::Duration::from_millis(1000);
   loop {
      thread::sleep(t);
   }
}
```

进程启动了 500 个线程，它们共享相同的内存。此外，多亏了锁，如果我们想修改这个内存，我们可以安全地这样做。

让我们尝试以下代码所示的服务器示例：

```rs
use std::{thread,time};
use std::process;
extern crate thread_id;
use std::io::prelude::*;
use std::net::{TcpListener,TcpStream};
use std::sync::{Arc,Mutex};

fn serve(incoming: Arc<Mutex<Vec<TcpStream>>>) {
   let t = time::Duration::from_millis(10);
   loop {
      {
         let mut incoming = incoming.lock().unwrap();
         for stream in incoming.iter() {
            let mut buffer = [0; 2048];
            let mut tcp = stream;
            tcp.read(&mut buffer).expect("tcp read failed");
            let response = format!("respond from #{}:{}\n", 
              process::id(), thread_id::get());
            tcp.write(response.as_bytes()).expect("tcp write failed");
         }
         incoming.clear();
      }
      thread::sleep(t);
   }
}

fn main() {
   let listener = TcpListener::bind("127.0.0.1:8888").unwrap();
   let incoming = Vec::new();
   let incoming = Arc::new(Mutex::new(incoming));
   for _ in 0..3 {
      let incoming = Arc::clone(&incoming);
      thread::spawn(move || {
         serve(incoming);
      });
   }

   for stream in listener.incoming() {
      let mut incoming = incoming.lock().unwrap();
      (*incoming).push(stream.unwrap());
   }
}
```

在这里，三个工作进程从父进程那里抓取请求队列，这些请求由父进程提供服务。所有三个子进程和父进程都需要读取和修改请求队列。为了修改请求队列，每个线程都必须锁定数据。这里有一个子进程和父进程之间的舞蹈，以避免长时间持有锁。如果一个线程垄断了锁定的资源，那么所有其他想要使用数据的进程都必须等待。

锁定和等待的权衡被称为**竞争**。在最坏的情况下，两个线程可以各自持有锁，同时等待另一个线程释放它持有的锁。这被称为**死锁**。

竞争是与可变共享状态相关的一个难题。对于前面的服务器案例，向子线程发送消息会更好。消息传递不会创建锁。

下面是一个无锁服务器的示例：

```rs
use std::{thread,time};
use std::process;
use std::io::prelude::*;
extern crate thread_id;
use std::net::{TcpListener,TcpStream};
use std::sync::mpsc::{channel,Receiver};
use std::collections::VecDeque;

fn serve(receiver: Receiver<TcpStream>) {
   let t = time::Duration::from_millis(10);
   loop {
      let mut tcp = receiver.recv().unwrap();
      let mut buffer = [0; 2048];
      tcp.read(&mut buffer).expect("tcp read failed");
      let response = format!("respond from #{}:{}\n", process::id(), 
             thread_id::get());
      tcp.write(response.as_bytes()).expect("tcp write failed");
      thread::sleep(t);
   }
}

fn main() {
   let listener = TcpListener::bind("127.0.0.1:8888").unwrap();
   let mut channels = VecDeque::new();
   for _ in 0..3 {
      let (sender, receiver) = channel();
      channels.push_back(sender);
      thread::spawn(move || {
         serve(receiver);
      });
   }
   for stream in listener.incoming() {
      let round_robin = channels.pop_front().unwrap();
      round_robin.send(stream.unwrap()).unwrap();
      channels.push_back(round_robin);
   }
}
```

在这种情况下，通道工作得更好。这个多线程服务器由父进程控制负载均衡，并且不受锁竞争的影响。

通道并不严格优于共享状态。例如，合法的竞争性资源用锁来处理是很好的。考虑以下代码片段：

```rs
use std::{thread,time};
extern crate rand;
use std::sync::{Arc,Mutex};
#[macro_use] extern crate lazy_static;
lazy_static! {
   static ref NEURAL_NET_WEIGHTS: Vec<Arc<Mutex<Vec<f64>>>> = {
      let mut nn = Vec::with_capacity(10000);
      for _ in 0..10000 {
         let mut mm = Vec::with_capacity(100);
         for _ in 0..100 {
            mm.push(rand::random::<f64>());
         }
         let mm = Arc::new(Mutex::new(mm));
         nn.push(mm);
      }
      nn
   };
}

fn train() {
   let t = time::Duration::from_millis(100);
   loop {
      for _ in 0..100 {
         let update_position = rand::random::<u64>() % 1000000;
         let update_column = update_position / 10000;
         let update_row = update_position % 100;
         let update_value = rand::random::<f64>();
         let mut update_column = NEURAL_NET_WEIGHTS[update_column as usize].lock().unwrap();
         update_column[update_row as usize] = update_value;
      }
      thread::sleep(t);
   }
}

fn main() {
   let t = time::Duration::from_millis(1000);
   for _ in 0..500 {
      thread::spawn(train);
   }
   loop {
      thread::sleep(t);
   }
}
```

在这里，我们有一个大的可变数据结构（一个神经网络），它被分解成行和列。每一列都有一个线程安全的锁。行数据都与同一个锁相关联。这种模式对于数据密集型和计算密集型程序非常有用。神经网络训练是这种技术可能相关的良好例子。不幸的是，代码并没有实现一个实际的神经网络，但它确实展示了如何使用锁并发来实现这一点。

# 理解 Send 和 Sync 特性

在之前的神经网络示例中，我们使用了一个静态数据结构，它在没有包裹在计数器或锁中时在多个线程间共享。它包含锁，但为什么外部的数据结构被允许共享？

为了回答这个问题，让我们首先回顾一下所有权规则：

+   Rust 中的每个值都有一个称为其 **所有者** 的变量

+   每次只能有一个所有者

+   当所有者超出作用域时，其值将被丢弃

在这些规则的基础上，让我们尝试在多个线程间共享一个变量，如下所示：

```rs
use std::thread;

fn main() {
   let a = vec![1, 2, 3];

   thread::spawn(|| {
      println!("a = {:?}", a);
   });
}
```

如果我们尝试编译这个，那么我们会得到一个错误，抱怨以下内容：

```rs
closure may outlive the current function, but it borrows `a`, which is owned by the current function
```

这个错误指示以下内容：

+   在闭包内部引用变量 `a` 是可以的

+   变量 `a` 的生命周期比闭包长

发送到线程的闭包必须具有静态生命周期。变量 `a` 是局部变量，因此它将在静态闭包之前超出作用域。

为了修复这个错误，通常会将变量 `a` 移动到闭包中。因此，`a` 将继承闭包相同的生命周期：

```rs
use std::thread;

fn main() {
   let a = vec![1, 2, 3];

   thread::spawn(move || {
      println!("a = {:?}", a);
   });
}
```

这个程序可以编译并运行。变量 `a` 的所有权被转移到闭包中，因此避免了生命周期问题。需要注意的是，转移变量的所有权意味着原始变量不再有效。这是由所有权规则编号 `2`——一次只能有一个所有者所导致的。

如果我们再次尝试共享变量，我们会得到一个错误：

```rs
use std::thread;

fn main() {
   let a = vec![1, 2, 3];

   thread::spawn(move || {
      println!("a = {:?}", a);
   });

   thread::spawn(move || {
      println!("a = {:?}", a);
   });
}
```

编译这个会给我们这个错误信息：

```rs
$ rustc t.rs
error[E0382]: capture of moved value: `a`
  --> t.rs:11:28
   |
6  |    thread::spawn(move || {
   |                  ------- value moved (into closure) here
...
11 |       println!("a = {:?}", a);
   |                            ^ value captured here after move
   |
   = note: move occurs because `a` has type `std::vec::Vec<i32>`, which does not implement the `Copy` trait

error: aborting due to previous error

For more information about this error, try `rustc --explain E0382`.
```

这个编译器错误有点复杂。它说以下内容：

+   **移动值的捕获**：`a`

+   这里移动的值（进入闭包）

+   在这里捕获的移动后的值

+   注意——移动发生是因为 `a` 没有实现 `Copy` 特性

错误信息中的第四部分告诉我们，如果 `a` 实现了 `Copy` 特性，那么我们就不会有这个错误。然而，那将隐式地为我们复制变量，这意味着我们不会共享数据。所以，这个建议对我们来说没有用。

主要问题是第一部分——移动值的捕获 `a`：

1.  首先，我们将变量 `a` 移动到第一个闭包中。我们需要这样做以避免生命周期问题并使用该变量。在闭包中使用变量称为 **捕获**。

1.  接下来我们在第二个闭包中使用变量 `a`。这是 `移动后的值捕获`。

所以我们的问题是移动变量 `a` 使其对于进一步使用无效。这个问题的一个更简单的例子如下：

```rs
fn main() {
   let a = vec![1, 2, 3];
   let b = a;
}
```

通过将 `a` 中的值的所有权移动到 `b`，我们使原始变量无效。

那我们该怎么办？我们卡住了吗？

在神经网络示例中，我们使用了一个共享的数据结构，所以显然必须有一种方法。如果有方法，希望也有规则来解释这个问题。要完全理解 Rust 中的线程安全规则，你必须理解三个概念——作用域、`Send` 和 `Sync`。

首先，让我们解决作用域问题。线程的作用域意味着使用的变量必须允许捕获它们所使用的变量。变量可以通过值、引用或可变引用来捕获。

我们的第一个例子，没有使用 `move`，几乎成功了。唯一的问题是，我们使用的变量的生命周期过早地超出了作用域。所有线程闭包都必须具有静态生命周期，因此它们捕获的变量也必须具有静态生命周期。对此进行调整，我们可以创建一个简单的双线程程序，通过引用捕获我们的变量 `A`，因此不需要移动变量：

```rs
use std::thread;

fn main() {
   static A: [u8; 100] = [22; 100];

   thread::spawn(|| {
      A[3];
   });

   thread::spawn(|| {
      A[3]
   });
}
```

从静态变量中读取是安全的。修改静态变量是不安全的。静态变量也不允许直接分配堆内存，因此它们可能难以处理。

使用 `lazy_static` 包是一个创建具有内存分配和初始化需求的静态变量的好方法：

```rs
use std::thread;
#[macro_use] extern crate lazy_static;

lazy_static! {
   static ref A: Vec<u32> = {
      vec![1, 2, 3]
   };
}

fn main() {
   thread::spawn(|| {
      A[1];
   });

   thread::spawn(|| {
      A[2];
   });
}
```

解决作用域问题的第二种方法是使用引用计数器，例如 `Arc`。在这里，我们使用 `Arc` 而不是 `Rc`，因为 `Arc` 是线程安全的，而 `Rc` 不是。考虑以下代码：

```rs
use std::thread;
use std::sync::{Arc};

fn main() {
   let a = Arc::new(vec![1, 2, 3]);
   {
      let a = Arc::clone(&a);
      thread::spawn(move || {
         a[1];
      });
   }

   {
      let a = Arc::clone(&a);
      thread::spawn(move || {
         a[1];
      });
   }
}
```

引用计数器将引用移动到闭包中。然而，内部数据是共享的，因此可以引用公共数据。

如果共享数据应该被修改，那么一个 `Mutex` 锁可以允许线程安全的锁定。另一个有用的锁是 `std::sync::RwLock`。如下所示：

```rs
use std::thread;
use std::sync::{Arc,Mutex};

fn main() {
   let a = Arc::new(Mutex::new(vec![1, 2, 3]));
   {
      let a = Arc::clone(&a);
      thread::spawn(move || {
         let mut a = a.lock().unwrap();
         (*a)[1] = 2;
      });
   }
   {
      let a = Arc::clone(&a);
      thread::spawn(move || {
         let mut a = a.lock().unwrap();
         (*a)[1] = 3;
      });
   }
}
```

那么为什么在锁定之后允许修改，而在锁定之前不允许呢？答案是 `Send` 和 `Sync`。

`Send` 和 `Sync` 是标记特性。标记特性不实现任何功能；然而，它表明一个类型具有某些属性。这两个属性告诉编译器在数据线程间共享时应允许哪些行为。

这些是关于线程数据共享的规则：

+   如果将一个类型安全地发送到另一个线程，则该类型是 `Send`。

+   如果一个类型可以在多个线程间安全地共享，则该类型是 `Sync`。

要创建可以在多个线程间共享的可变数据，无论数据类型如何，你必须实现 `Sync`。标准 Rust 库有一些线程安全的并发原语，如 `Mutex`，用于此目的。如果你不喜欢可用的选项，那么你可以搜索另一个包或自己创建一个。

要为类型实现 `Sync`，只需实现没有主体的特性：

```rs
use std::thread;

struct MyBox(u8);
unsafe impl Send for MyBox {}
unsafe impl Sync for MyBox {}

static A: MyBox = MyBox(22);

fn main() {
   thread::spawn(move || {
      A.0
   });
   thread::spawn(move || {
      A.0
   });
}
```

警告——错误地实现 `Send` 或 `Sync` 可能会导致未定义的行为。这些特性总是不安全的实现。幸运的是，这两个标记特性通常由编译器自动推导，所以你很少需要手动推导它们。

在心中牢记这些各种规则，我们可以看到 Rust 如何防止许多常见的线程错误。首先，所有权系统防止了许多问题。然后，为了允许一些线程间的通信，我们发现通道和锁可以帮助安全地实现大多数并发模型。

这需要进行大量的试错，但总的来说，我们了解到`thread`、`move`、`channel`、`Arc`和`Mutex`将帮助我们解决大多数问题。

# 使用函数式设计进行并发

并发迫使程序员更加注意信息共享。这种困难偶然地鼓励了良好的函数式编程实践，如不可变数据和纯函数；当计算不是上下文相关时，它往往也是线程安全的。

函数式编程听起来非常适合并发，但有没有缺点？

在一个意图良好但效果不佳的例子中，在开发名为**Haskell**的函数式语言期间，开发团队（[`www.infoq.com/interviews/armstrong-peyton-jones-erlang-haskell`](https://www.infoq.com/interviews/armstrong-peyton-jones-erlang-haskell)）希望通过并发来使程序运行得更快。由于 Haskell 语言的独特特性，可以在新线程中运行所有表达式和子表达式。开发团队认为这听起来很棒，并进行了测试。

结果是，花费在创建新线程上的时间比进行任何计算的时间还要多。这个想法本身还是有价值的，但最终证明实现自动并发是困难的。并发编程中有许多权衡。让程序员就这些权衡做出决策是当前最先进的状态。

因此，从函数式编程来看，哪些模式被证明是有用的？

并发编程有许多模式，但在这里我们将介绍一些基本模式：

+   **actor**：线程和行为模式

+   **监督者**：监控和管理 actor

+   **路由器**：在 actor 之间发送消息

+   **monad**：可组合的行为单元

首先，让我们看看以下代码中的 actor：

```rs
use std::thread;
use std::sync::mpsc::{channel};
use std::time;

fn main() {
   let (pinginsend,pinginrecv) = channel();
   let (pingoutsend,pingoutrecv) = channel();
   let mut ping = 1;
   thread::spawn(move || {
      let t = time::Duration::from_millis(1000);
      loop {
         let n = pinginrecv.recv().unwrap();
         ping += n;
         println!("ping {}", ping);
         thread::sleep(t);
         pingoutsend.send(ping).unwrap();
      }
   });

   let (ponginsend,ponginrecv) = channel();
   let (pongoutsend,pongoutrecv) = channel();
   let mut pong = 2;
   thread::spawn(move || {
      let t = time::Duration::from_millis(1000);
      loop {
         let n = ponginrecv.recv().unwrap();
         pong += n;
         println!("pong {}", pong);
         thread::sleep(t);
         pongoutsend.send(pong).unwrap();
      }
   });

   let mut d = 3;
   loop {
      pinginsend.send(d).unwrap();
      d = pingoutrecv.recv().unwrap();
      ponginsend.send(d).unwrap();
      d = pongoutrecv.recv().unwrap();
   }
}
```

这里我们有两个线程在相互发送消息。这真的和之前的任何例子有很大不同吗？

在函数式编程中有一个相当常见的说法：“*闭包是穷人的对象，而对象是穷人的闭包*”。

根据面向对象编程，对象有类型、字段和方法。我们定义的闭包持有它们自己的可变状态，就像对象上的字段一样。ping 和 pong 闭包有略微不同的类型。闭包内的行为可以被视为闭包对象上的一个无名称方法。在这里，对象和闭包之间有相似之处。

然而，使用普通对象会更好一些。尝试这样做的问题在于线程边界会阻碍操作。线程不暴露方法，只进行消息传递。作为一个折衷方案，我们可以将消息传递封装成方法的形式。这将隐藏所有的通道管理，使得使用并发对象进行编程更加方便。我们将这种模式称为 actor 模型。

演员与 OOP 对象非常相似，额外的一个属性是它生活在自己的线程中。消息被发送到演员，演员处理消息，并可能发送出自己的一些消息。演员模型就像一个繁忙的城市，人们生活在其中，从事不同的工作，但根据他们自己的时间表相互交流和交换。

有一些箱子试图提供优雅的并发演员行为，但我们不会特别推荐任何一种。目前，请只是眯起眼睛，继续假装闭包与对象相似。

在下一个例子中，让我们将这些演员包装成函数，以便更容易地创建它们：

```rs
use std::thread;
use std::sync::mpsc::{channel,Sender,Receiver};
use std::time;
extern crate rand;

fn new_ping() -> (Sender<u64>, Receiver<u64>) {
   let (pinginsend,pinginrecv) = channel();
   let (pingoutsend,pingoutrecv) = channel();
   let mut ping = 1;
   thread::spawn(move || {
      let t = time::Duration::from_millis(1000);
      loop {
         let n = pinginrecv.recv().unwrap();
         ping += n;
         println!("ping {}", ping);
         thread::sleep(t);
         pingoutsend.send(ping).unwrap();
      }
   });
   (pinginsend, pingoutrecv)
}

fn new_pong() -> (Sender<u64>, Receiver<u64>) {
   let (ponginsend,ponginrecv) = channel();
   let (pongoutsend,pongoutrecv) = channel();
   let mut pong = 2;
   thread::spawn(move || {
      let t = time::Duration::from_millis(1000);
      loop {
         let n = ponginrecv.recv().unwrap();
         pong += n;
         println!("pong {}", pong);
         thread::sleep(t);
         pongoutsend.send(pong).unwrap();
      }
   });
   (ponginsend, pongoutrecv)
}
```

要运行示例，我们将创建每种类型的三个演员，并将通道存储在一个向量中，如下面的代码所示：

```rs
fn main() {
   let pings = vec![new_ping(), new_ping(), new_ping()];
   let pongs = vec![new_pong(), new_pong(), new_pong()];
   loop {
      let mut d = 3;

      let (ref pingin,ref pingout) = pings[(rand::random::<u64>() % 3) as usize];
      pingin.send(d).unwrap();
      d = pingout.recv().unwrap();

      let (ref pongin,ref pongout) = pongs[(rand::random::<u64>() % 3) as usize];
      pongin.send(d).unwrap();
      pongout.recv().unwrap();
   }
}
```

现在，我们为每个演员组有了演员和一个非常基本的监督者。这里的监督者只是一个向量，用于跟踪每个演员的通信通道。一个好的监督者应该定期检查每个演员的健康状况，杀死不良演员，并补充良好演员的库存。

我们将要提到的最后一个基于角色的原始方法是路由。路由是面向对象编程的方法等价物。OOP 方法调用最初被称为 **消息传递**。演员模型非常面向对象，因此我们仍然通过实际传递消息来调用方法。我们仍在使用穷人的对象（闭包），所以我们的路由可能看起来像是一个美化过的 `if` 语句。

要启动我们的演员路由器，我们将定义两种数据类型——地址和消息。地址应该定义消息的所有可能目的地和路由行为。消息应该对应于所有演员的所有可能方法调用。以下是我们的扩展乒乓应用：

```rs
use std::thread;
use std::sync::mpsc::{channel,Sender,Receiver};
use std::time;
extern crate rand;

enum Address {
   Ping,
   Pong
}

enum Message {
   PingPlus(u64),
   PongPlus(u64),
}
```

然后，我们定义我们的演员。现在，它们需要与新的 `Message` 类型匹配，并且发出的消息应该有一个 `Address`，除了 `Message`。尽管有所变化，代码仍然非常相似：

```rs
fn new_ping() -> (Sender<Message>, Receiver<(Address,Message)>) {
   let (pinginsend,pinginrecv) = channel();
   let (pingoutsend,pingoutrecv) = channel();
   let mut ping = 1;
   thread::spawn(move || {
      let t = time::Duration::from_millis(1000);
      loop {
         let msg = pinginrecv.recv().unwrap();
         match msg {
            Message::PingPlus(n) => { ping += n; },
            _ => panic!("Unexpected message")
         }
         println!("ping {}", ping);
         thread::sleep(t);
         pingoutsend.send((
            Address::Pong,
            Message::PongPlus(ping)
         )).unwrap();
         pingoutsend.send((
            Address::Pong,
            Message::PongPlus(ping)
         )).unwrap();
      }
   });
   (pinginsend, pingoutrecv)
}

fn new_pong() -> (Sender<Message>, Receiver<(Address,Message)>) {
   let (ponginsend,ponginrecv) = channel();
   let (pongoutsend,pongoutrecv) = channel();
   let mut pong = 1;
   thread::spawn(move || {
      let t = time::Duration::from_millis(1000);
      loop {
         let msg = ponginrecv.recv().unwrap();
         match msg {
            Message::PongPlus(n) => { pong += n; },
            _ => panic!("Unexpected message")
         }
         println!("pong {}", pong);
         thread::sleep(t);
         pongoutsend.send((
            Address::Ping,
            Message::PingPlus(pong)
         )).unwrap();
         pongoutsend.send((
            Address::Ping,
            Message::PingPlus(pong)
         )).unwrap();
      }
   });
   (ponginsend, pongoutrecv)
}
```

每个乒乓进程循环消费一条消息，并发送两条消息。程序的最后一个组件是初始化和路由：

```rs
fn main() {
   let pings = vec![new_ping(), new_ping(), new_ping()];
   let pongs = vec![new_pong(), new_pong(), new_pong()];

   //Start the action
   pings[0].0.send(Message::PingPlus(1)).unwrap();

   //This thread will be the router
   //This is a busy wait and otherwise bad code
   //select! would be much better, but it is still experimental
   //https://doc.rust-lang.org/std/macro.select.html
   let t = time::Duration::from_millis(10);
   loop {
      let mut mail = Vec::new();

      for (_,r) in pings.iter() {
         for (addr,msg) in r.try_iter() {
            mail.push((addr,msg));
         }
      }
      for (_,r) in pongs.iter() {
         for (addr,msg) in r.try_iter() {
            mail.push((addr,msg));
         }
      }

      for (addr,msg) in mail.into_iter() {
         match addr {
            Address::Ping => {
               let (ref s,_) = pings[(rand::random::<u32>() as usize) % pings.len()];
               s.send(msg).unwrap();
            },
            Address::Pong => {
               let (ref s,_) = pongs[(rand::random::<u32>() as usize) % pongs.len()];
               s.send(msg).unwrap();
            }
         }
      }
      thread::sleep(t);
   }
}
```

在初始化了不同的演员之后，主线程开始充当路由器。路由器是一个单线程，唯一的责任是找到目的地，然后移动、复制、克隆以及其他方式将消息分发给接收线程。这不是一个复杂的解决方案，但它是有效的，并且只使用了我们迄今为止引入的类型安全、线程安全、平台无关的原始类型。

在一个更复杂的例子中，路由 `Address` 通常具有以下特点：

+   演员角色

+   方法名称

+   参数类型签名

那么消息将是根据前面的类型签名提供的参数。从演员发送消息就像发送你的`(Address,Message)`到路由器一样简单。此时，路由器应该定期检查每个通道是否有新的路由请求。当它看到新消息时，它将选择满足`Address`条件的演员，并将消息发送到该演员的收件箱。

观察输出，每次乒乓动作都会将接收到的消息数量翻倍。如果每个线程不做那么多睡眠，那么程序可能会迅速失控。消息噪声是过度使用演员模型的一个风险。

# 摘要

在本章中，我们介绍了并发计算的基本原理。子进程、分叉进程和线程是所有并发应用程序的基本构建块。在 Rust 的线程中，语言引入了额外的关注点，以鼓励类型和线程安全。

在几个示例中，我们使用分叉或线程构建了并发网络服务器。后来，在探索线程行为时，我们仔细观察了线程之间可以共享哪些数据以及如何安全地在线程之间发送信息。

在设计模式部分，我们介绍了演员设计模式。这种流行的技术结合了面向对象编程的一些元素和函数式编程的其他概念。结果是专为复杂健壮的并发设计的一种编程工具。

在下一章中，我们将探讨性能、调试和元编程。性能可能难以衡量或比较，但我们将尝试介绍对性能严格有益的习惯。为了帮助调试，我们将探讨主动和被动技术来解决这些问题。主动调试是一套技术，如适当的错误处理，它要么防止错误，要么使错误更容易记录和解决。被动技术对于没有明显原因的困难错误很有用。最后，元编程可以在幕后完成大量复杂的工作，使丑陋的代码看起来更美观。

# 问题

1.  什么是子进程？

1.  为什么分叉被称为分叉？

1.  分叉（fork）是否仍然有用？

1.  线程何时被标准化？

1.  为什么有时需要`move`来处理线程闭包？

1.  `Send`和`Sync`特质的区别是什么？

1.  为什么我们可以不对`Mutex`加锁就进行修改，而不需要使用不安全的代码块？
