# 第八章：与未来一起工作

在本章中，我们将介绍以下食谱：

+   提供带有 CPU 池的未来并等待它们

+   为未来实现错误处理

+   组合未来

+   使用流

+   使用汇（Sinks）

+   使用单次通道

+   返回未来

+   使用 BiLocks 锁定资源

# 简介

未来提供了异步计算的零成本抽象构建块。异步通信对于处理超时、跨线程池计算、网络响应以及任何不立即返回值的函数非常有用。

在同步块中，计算机会在等待每个命令返回一个值后，依次执行每个命令。如果您在发送电子邮件时应用同步模型，您将发送消息，盯着您的收件箱，直到收到收件人的回复。

幸运的是，生活并不总是同步的。在我们发送电子邮件后，我们可以切换到另一个应用程序或离开椅子。我们可以开始执行其他任务，如购买杂货、做晚饭或读书。我们的注意力可以同时集中在执行其他任务上。定期地，我们会检查收件箱以查看收件人的回复。定期检查新消息的过程说明了异步模型。与人类不同，计算机可以在我们的收件箱中检查新消息，并同时执行其他任务。

Rust 的未来通过实现轮询模型来工作，该模型利用一个中心组件（例如，软件、硬件设备和网络主机）来处理来自其他组件的状态报告。中心或主组件会重复地向其他组件发送信号，直到主组件收到更新、中断信号或轮询事件超时。

要更好地理解 Rust 模型中并发的工作原理，您可以查看 Alex Crichton 的并发演示文稿，链接为 [`github.com/alexcrichton/talks`](https://github.com/alexcrichton/talks)。在我们的食谱中，我们将在主线程中使用 `futures::executor::block_on` 函数来返回值。这是为了演示目的而故意这样做的。在实际应用中，您将在另一个单独的线程中使用 `block_on`，并且您的函数将返回某种 `futures::Future` 实现例如 `futures::future::FutureResult`。

在撰写本文时，未来在其代码库中进行了大量的开发性更改。您可以在他们的官方仓库 [`github.com/rust-lang-nursery/futures-rfcs`](https://github.com/rust-lang-nursery/futures-rfcs) 上查看未来的 RFC（请求评论）。

# 提供带有 CPU 池的未来并等待它们

未来通常被分配给一个`Task`，然后被分配给一个`Executor`。当一个任务*唤醒*时，执行器将任务放入队列，并将在任务上调用`poll()`直到进程完成。未来为我们提供了执行任务的一些方便的方法：

+   使用`futures::executor::block_on()`手动生成未来任务。

+   使用`futures::executor::LocalPool`，这对于在单个线程上执行许多小任务非常有用。在我们的未来返回中，我们不需要实现`Send`，因为我们只在一个线程上涉及任务。然而，如果你省略了`Send`特性，你需要在`Executor`上使用`futures::executor::spawn_local()`。

+   使用`futures::executor::ThreadPool`，它允许我们将任务卸载到其他线程。

# 如何做到这一点...

1.  使用`cargo new futures`创建一个 Rust 项目，在本章中工作。

1.  导航到新创建的`futures`文件夹。在本章的其余部分，我们将假设你的命令行在这个目录内。

1.  在`src`文件夹内，创建一个名为`bin`的新文件夹。

1.  删除生成的`lib.rs`文件，因为我们没有创建库。

1.  打开生成的`Cargo.toml`文件。

1.  在`[dependencies]`下添加以下行：

```rs
futures = "0.2.0-beta"
futures-util = "0.2.0-beta"
```

1.  在`src/bin`文件夹中，创建一个名为`pool.rs`的文件。

1.  添加以下代码，并使用`cargo run —bin pool`运行它：

```rs
1   extern crate futures;
2   
3   use futures::prelude::*;
4   use futures::task::Context;
5   use futures::channel::oneshot;
6   use futures::future::{FutureResult, lazy, ok};
7   use futures::executor::{block_on, Executor, LocalPool, 
    ThreadPoolBuilder};
8   
9   use std::cell::Cell;
10  use std::rc::Rc;
11  use std::sync::mpsc;
12  use std::thread;
13  use std::time::Duration;
```

让我们添加我们的常量、枚举、结构和特性实现：

```rs
15  #[derive(Clone, Copy, Debug)]
16  enum Status {
17    Loading,
18    FetchingData,
19    Loaded,
20  }
21  
22  #[derive(Clone, Copy, Debug)]
23  struct Container {
24    name: &'static str,
25    status: Status,
26    ticks: u64,
27  }
28  
29  impl Container {
30    fn new(name: &'static str) -> Self {
31      Container {
32        name: name,
33        status: Status::Loading,
34        ticks: 3,
35      }
36    }
37  
38    // simulate ourselves retreiving a score from a remote  
      database
39    fn pull_score(&mut self) -> FutureResult<u32, Never> {
40      self.status = Status::Loaded;
41      thread::sleep(Duration::from_secs(self.ticks));
42      ok(100)
43    }
44  }
45  
46  impl Future for Container {
47    type Item = ();
48    type Error = Never;
49  
50    fn poll(&mut self, _cx: &mut Context) -> Poll<Self::Item,  
      Self::Error> {
51      Ok(Async::Ready(()))
52    }
53  }
55  const FINISHED: Result<(), Never> = Ok(());
56  
57  fn new_status(unit: &'static str, status: Status) {
58    println!("{}: new status: {:?}", unit, status);
59  }
```

让我们添加我们的第一个本地线程函数：

```rs
61  fn local_until() {
62    let mut container = Container::new("acme");
63  
64    // setup our green thread pool
65    let mut pool = LocalPool::new();
66    let mut exec = pool.executor();
67  
68    // lazy will only execute the closure once the future has
      been polled
69    // we will simulate the poll by returning using the 
      future::ok method
70  
71    // typically, we perform some heavy computational process  
      within this closure
72    // such as loading graphic assets, sound, other parts of our  
      framework/library/etc.
73    let f = lazy(move |_| -> FutureResult<Container, Never> {
74      container.status = Status::FetchingData;
75      ok(container)
76    });
77  
78    println!("container's current status: {:?}",  
      container.status);
79  
80    container = pool.run_until(f, &mut exec).unwrap();
81    new_status("local_until", container.status);
82  
83    // just to demonstrate a simulation of "fetching data over a  
      network"
84    println!("Fetching our container's score...");
85    let score = block_on(container.pull_score()).unwrap();
86    println!("Our container's score is: {:?}", score);
87  
88    // see if our status has changed since we fetched our score
89    new_status("local_until", container.status);
90  }
```

现在是我们的本地生成的线程示例：

```rs
92  fn local_spawns_completed() {
93    let (tx, rx) = oneshot::channel();
94    let mut container = Container::new("acme");
95  
96    let mut pool = LocalPool::new();
97    let mut exec = pool.executor();
98  
99    // change our container's status and then send it to our  
      oneshot channel
100   exec.spawn_local(lazy(move |_| {
101       container.status = Status::Loaded;
102       tx.send(container).unwrap();
103       FINISHED
104     }))
105     .unwrap();
106 
107   container = pool.run_until(rx, &mut exec).unwrap();
108   new_status("local_spanws_completed", container.status);
109 }
110 
111 fn local_nested() {
112   let mut container = Container::new("acme");
114   // we will need Rc (reference counts) since 
      we are referencing multiple owners
115   // and we are not using Arc (atomic reference counts) 
      since we are only using
116   // a local pool which is on the same thread technically
117   let cnt = Rc::new(Cell::new(container));
118   let cnt_2 = cnt.clone();
119 
120   let mut pool = LocalPool::new();
121   let mut exec = pool.executor();
122   let mut exec_2 = pool.executor();
123 
124   let _ = exec.spawn_local(lazy(move |_| {
125     exec_2.spawn_local(lazy(move |_| {
126         let mut container = cnt_2.get();
127         container.status = Status::Loaded;
128 
129         cnt_2.set(container);
130         FINISHED
131       }))
132       .unwrap();
133     FINISHED
134   }));
135 
136   let _ = pool.run(&mut exec);
137 
138   container = cnt.get();
139   new_status("local_nested", container.status);
140 }
```

现在是我们的线程池示例：

```rs
142 fn thread_pool() {
143   let (tx, rx) = mpsc::sync_channel(2);
144   let tx_2 = tx.clone();
145 
146   // there are various thread builder options which are 
      referenced at
147   // https://docs.rs/futures/0.2.0- 
      beta/futures/executor/struct.ThreadPoolBuilder.html
148   let mut cpu_pool = ThreadPoolBuilder::new()
149     .pool_size(2) // default is the number of cpus
150     .create();
151 
152   // We need to box this part since we need the Send +'static trait
153   // in order to safely send information across threads
154   let _ = cpu_pool.spawn(Box::new(lazy(move |_| {
155     tx.send(1).unwrap();
156     FINISHED
157   })));
158 
159   let f = lazy(move |_| {
160     tx_2.send(1).unwrap();
161     FINISHED
162   });
163 
164   let _ = cpu_pool.run(f);
165 
166   let cnt = rx.into_iter().count();
167   println!("Count should be 2: {:?}", cnt);
168 }
```

最后，我们的`main`函数：

```rs
170 fn main() {
171   println!("local_until():");
172   local_until();
173 
174   println!("\nlocal_spawns_completed():");
175   local_spawns_completed();
176 
177   println!("\nlocal_nested():");
178   local_nested();
179 
180   println!("\nthread_pool():");
181   thread_pool();
182 }
```

# 它是如何工作的...

让我们先介绍一下`Future`特性：

+   实现`Future`特性只需要三个约束：一个`Item`类型，一个`Error`类型，以及一个`poll()`函数。实际的特性看起来如下：

```rs
pub trait Future {
    type Item;
    type Error;
    fn poll(
        &mut self, 
        cx: &mut Context
    ) -> Result<Async<Self::Item>, Self::Error>;
}
```

+   `Poll<Self::Item, Self::Error>`是一个类型，它转换为`Result<Async<T>, E>`，其中`T = Item`和`E = Error`。这就是我们在第 50 行使用的示例。

+   当使用位于`futures::executor`的执行器执行`futures::task::Waker`（也可以称为*Task*）时，或者通过构建`futures::task::Context`并使用未来包装器（如`futures::future::poll_fn`）手动唤醒时，会调用`poll()`。

现在，让我们转到我们的`local_until()`函数：

+   `LocalPool`为我们提供了使用单个线程并发运行任务的能力。这对于具有最小复杂性的函数非常有用，例如传统的 I/O 绑定函数。`LocalPools`可以有多个`LocalExecutors`（正如我们在第 65 行创建的那样），它们可以生成我们的任务。由于我们的任务是单线程的，我们不需要`Box`或添加`Send`特性到我们的未来。

+   `futures::future::lazy`函数将创建一个新的未来，从一个`FnOnce`闭包，这个闭包返回的将是与闭包返回的相同的未来（任何`futures::future::IntoFuture`特性），在我们的情况下，这个未来是`FutureResult<Container, Never>`。

+   从`LocalPool`执行`run_until(F: Future)`函数将执行所有 future 任务，直到`Future`（表示为`F`）被标记为完成。该函数在完成时会返回`Result<<F as Future>::Item, <F as Future>::Error>`。在示例中，我们在第 75 行返回`futures::future::ok(Container)`，所以我们的`F::Item`将是我们的`Container`。

对于我们的`local_spawns_completed()`函数：

+   首先，我们设置了我们的`futures::channel::oneshot`通道（稍后将在*使用 oneshot 通道*部分进行解释）。

+   我们将使用`oneshot`通道的`futures::channel::oneshot::Receiver`作为在`run_until()`函数中运行直到完成的 future。这允许我们演示在从另一个线程或任务接收到信号之前（在我们的示例中，这发生在第 102 行的`tx.send(...)`命令）轮询将如何工作。

+   `LocalExecutor`的`spawn_local()`是一个特殊的`spawn`函数，它赋予我们执行 future 函数而不需要实现`Send`特质的权限。

接下来，我们的`local_nested()`函数：

+   我们设置了我们的常规`Container`，然后声明了一个引用计数器，这将允许我们在多个 executors 或线程之间保持一个值（这将是我们`Container`）。由于我们使用`spawn_local()`，它在一个绿色线程（由虚拟机或运行时库调度的线程）上执行 future，所以我们不需要使用原子引用计数器。

+   `LocalPool`的`run(exec: &mut Executor)`函数将运行池中生成的任何 future，直到所有 future 都完成。这也包括可能在其他任务中`spawn`额外任务的任何 executors，如我们的示例所示。

关于我们的`thread_pool()`函数：

+   创建了一个`std::sync::mspc::sync_channel`，目的是为了演示阻塞线程。

+   接下来，我们使用默认设置创建了一个`ThreadPool`，并调用了它的`spawn(F: Box<Future<Item = (), Error = Never> + 'static + Send>)`函数，无论何时我们决定执行池，它都会轮询任务直到完成。

+   在设置好我们的任务后，我们执行`ThreadPool`的`run(F: Future)`函数，这将阻塞调用`run()`的线程，直到`F: Future`完成。即使池中还有其他任务被生成和运行，该函数在 future 完成时也会返回一个值。使用`mspc::sync_channel`之前有助于减轻这个问题，但会在被调用时阻塞线程。

+   使用`ThreadPoolBuilder`，你可以：

    +   设置工作线程的数量

    +   调整栈大小

    +   为池设置一个前缀名称

    +   在每个工作线程启动后，在运行任何任务之前运行一个函数（函数签名为`Fn(usize) + Send + Sync + 'static`）

    +   在每个工作线程关闭之前执行一个函数（函数签名为`Fn(usize) + Send + Sync + 'static`）

# 处理 future 中的错误

在实际应用中，我们不会从直接返回 `Async::Ready<T>` 或 `FutureResult<T, E>` 的异步函数中立即返回一个值。网络请求超时，缓冲区满，由于错误或中断服务变得不可用，以及许多其他问题每天都在出现。尽管我们喜欢从混乱中建立秩序，但由于自然发生的熵（程序员可能称之为 *范围蔓延*）和衰减（软件更新、新的计算机科学范式等），通常混乱获胜。幸运的是，futures 库为我们提供了一个简单的方法来实现错误处理。

# 如何做...

1.  在 `bin` 文件夹内，创建一个名为 `errors.rs` 的新文件。

1.  添加以下代码并使用 `cargo run --bin errors` 运行它：

```rs
1   extern crate futures;
2   
3   use futures::prelude::*;
4   use futures::executor::block_on;
5   use futures::stream;
6   use futures::task::Context;
7   use futures::future::{FutureResult, err};
```

1.  然后，让我们添加我们的结构和实现：

```rs
9   struct MyFuture {}
10  impl MyFuture {
11    fn new() -> Self {
12      MyFuture {}
13    }
14  }
15  
16  fn map_error_example() -> FutureResult<(), &'static str> {
17    err::<(), &'static str>("map_error has occurred")
18  }
19  
20  fn err_into_example() -> FutureResult<(), u8> {
21    err::<(), u8>(1)
22  }
23  
24  fn or_else_example() -> FutureResult<(), &'static str> {
25    err::<(), &'static str>("or_else error has occurred")
26  }
27  
28  impl Future for MyFuture {
29    type Item = ();
30    type Error = &'static str;
31  
32    fn poll(&mut self, _cx: &mut Context) -> Poll<Self::Item, Self::Error> {
33      Err("A generic error goes here")
34    }
35  }
36  
37  struct FuturePanic {}
38  
39  impl Future for FuturePanic {
40    type Item = ();
41    type Error = ();
42  
43    fn poll(&mut self, _cx: &mut Context) -> Poll<Self::Item, 
      Self::Error> {
44      panic!("It seems like there was a major issue with  
        catch_unwind_example")
45    }
46  }
```

1.  然后，让我们添加我们的泛型错误处理函数/示例：

```rs
48  fn using_recover() {
49    let f = MyFuture::new();
50  
51    let f_recover = f.recover::<Never, _>(|err| {
52      println!("An error has occurred: {}", err);
53      ()
54    });
55  
56    block_on(f_recover).unwrap();
57  }
58  
59  fn map_error() {
60    let map_fn = |err| format!("map_error_example: {}", err);
61  
62    if let Err(e) = block_on(map_error_example().map_err(map_fn)) 
     {
63      println!("block_on error: {}", e)
64    }
65  }
66  
67  fn err_into() {
68    if let Err(e) = block_on(err_into_example().err_into::()) {
69      println!("block_on error code: {:?}", e)
70    }
71  }
72  
73  fn or_else() {
74    if let Err(e) = block_on(or_else_example()
75      .or_else(|_| Err("changed or_else's error message"))) {
76      println!("block_on error: {}", e)
77    }
78  }
```

1.  现在是我们的 `panic` 函数：

```rs
80  fn catch_unwind() {
81    let f = FuturePanic {};
82  
83    if let Err(e) = block_on(f.catch_unwind()) {
84      let err = e.downcast::<&'static str>().unwrap();
85      println!("block_on error: {:?}", err)
86    }
87  }
88  
89  fn stream_panics() {
90    let stream_ok = stream::iter_ok::<_, bool>(vec![Some(1),  
      Some(7), None, Some(20)]);
91    // We panic on "None" values in order to simulate a stream  
      that panics
92    let stream_map = stream_ok.map(|o| o.unwrap());
93  
94    // We can use catch_unwind() for catching panics
95    let stream = stream_map.catch_unwind().then(|r| Ok::<_, ()> 
      (r));
96    let stream_results: Vec<_> =  
       block_on(stream.collect()).unwrap();
97  
98    // Here we can use the partition() function to separate the Ok  
      and Err values
99    let (oks, errs): (Vec<_>, Vec<_>) =  
      stream_results.into_iter().partition(Result::is_ok);
100   let ok_values: Vec<_> =   
      oks.into_iter().map(Result::unwrap).collect();
101   let err_values: Vec<_> =  
      errs.into_iter().map(Result::unwrap_err).collect();
102 
103   println!("Panic's Ok values: {:?}", ok_values);
104   println!("Panic's Err values: {:?}", err_values);
105 }
```

1.  最后，我们的 `main` 函数：

```rs
107 fn main() {
108   println!("using_recover():");
109   using_recover();
110 
111   println!("\nmap_error():");
112   map_error();
113 
114   println!("\nerr_into():");
115   err_into();
116 
117   println!("\nor_else():");
118   or_else();
119 
120   println!("\ncatch_unwind():");
121   catch_unwind();
122 
123   println!("\nstream_panics():");
124   stream_panics();
125 }
```

# 它是如何工作的...

让我们从 `using_recover()` 函数开始：

+   在 future 中发生的任何错误都将被转换成 `<Self as Future>::Item`。任何 `<Self as Future>::Error` 类型都可以传递，因为我们永远不会产生实际的错误。

+   `futures::executor::block_on(F: Future)` 函数将在调用线程中运行一个 future，直到完成。任何在 futures 的 `default executor` 中的任务也将在这个调用线程上运行，但由于 `F` 可能会在任务完成之前完成，所以任务可能永远不会完成。如果这种情况发生，那么产生的任务将被丢弃。`LocalPool` 常常被推荐用于缓解这个问题，但对我们来说 `block_on()` 将足够。

所有这些错误处理函数都可以在 `futures::FutureExt` 特性中找到。

现在，让我们看看我们的 `map_error()` 函数：

+   `<Self as Future>::map_err<E, F>(F: FnOnce(Self::Error) -> E)` 函数将 future 的 (`Self`) 错误映射到另一个错误，同时返回一个新的 future。这个函数通常与组合器（如 select 或 join）一起使用，因为我们可以保证 futures 将具有相同的错误类型以完成组合。

接下来，是 `err_into()` 函数：

+   使用 `std::convert::Into` 特性将 `Self::Error` 转换为另一种 `Error` 类型

+   与 `futures::FutureExt::map_err` 类似，这个函数对于组合组合器很有用

`or_else()` 函数：

+   如果 `<Self as Future>` 返回一个错误，`futures::FutureExt::or_else` 将执行一个具有以下签名的闭包：`FnOnce(Self::Error) -> futures::future::IntoFuture<Item = Self::Item>`

+   对于将失败的组合器链接在一起很有用

+   如果 future 成功完成、panic 或其 future 被丢弃，闭包将不会执行

然后是 `catch_unwind()` 函数：

+   这个函数通常不推荐作为处理错误的方式，并且仅通过 Rust 的 `std` 选项启用（默认启用）

+   Future 特性实现了 `AssertUnwindSafe` 特性作为 `AssertUnwindSafe<F: Future>` 特性

最后，`stream_panics()` 函数：

+   在第 95 行，此 `futures::StreamExt::catch_unwind` 函数类似于 `futures::FutureExt::catch_unwind`

+   如果发生 panic，它将是流中的最后一个元素

+   此功能仅在 Rust 的 `std` 选项启用时才可用

+   `AssertUnwindSafe` 特性也实现了流作为 `AssertUnwindSafe<S: Stream>`

流的组合器位于 `futures::StreamExt` 特性中，它具有与 `futures::FutureExt` 相同的函数，还有一些额外的流特定组合器，如 `split()` 和 `skip_while()`，这些可能对您的项目很有用。

# 参见

+   第六章，*处理错误*

# 结合 futures

结合和链式调用我们的 futures 允许我们按顺序执行多个操作，并有助于更好地组织我们的代码。它们可以用来转换、拼接、过滤等 `<Self as Future>::Item`s。

# 如何做到...

1.  在 `bin` 文件夹中，创建一个名为 `combinators.rs` 的新文件。

1.  添加以下代码，并使用 `cargo run --bin combinators` 运行它：

```rs
1   extern crate futures;
2   extern crate futures_util;
3   
4   use futures::prelude::*;
5   use futures::channel::{mpsc, oneshot};
6   use futures::executor::block_on;
7   use futures::future::{ok, err, join_all, select_all, poll_fn};
8   use futures::stream::iter_result;
9   use futures_util::stream::select_all as select_all_stream;
10  
11  use std::thread;
12  
13  const FINISHED: Result<Async<()>, Never> = Ok(Async::Ready(()));
```

1.  让我们添加我们的 `join_all` 示例函数：

```rs
15  fn join_all_example() {
16    let future1 = Ok::<_, ()>(vec![1, 2, 3]);
17    let future2 = Ok(vec![10, 20, 30]);
18    let future3 = Ok(vec![100, 200, 300]);
19  
20    let results = block_on(join_all(vec![future1, future2,  
      future3])).unwrap();
21    println!("Results of joining 3 futures: {:?}", results);
22  
23    // For parameters with a lifetime
24    fn sum_vecs<'a>(vecs: Vec<&'a [i32]>) -> Box<Future, Error = 
      ()> + 'static> {
25      Box::new(join_all(vecs.into_iter().map(|x| Ok::<i32, ()> 
        (x.iter().sum()))))
26    }
27  
28    let sum_results = block_on(sum_vecs(vec![&[1, 3, 5], &[6, 7, 
      8], &[0]])).unwrap();
29    println!("sum_results: {:?}", sum_results);
30  }
31  
```

接下来，我们将编写我们的 `shared` 函数：

```rs
32  fn shared() {
33    let thread_number = 2;
34    let (tx, rx) = oneshot::channel::();
35    let f = rx.shared();
36    let threads = (0..thread_number)
37      .map(|thread_index| {
38        let cloned_f = f.clone();
39        thread::spawn(move || {
40          let value = block_on(cloned_f).unwrap();
41          println!("Thread #{}: {:?}", thread_index, *value);
42        })
43      })
44      .collect::<Vec<_>>();
45    tx.send(42).unwrap();
46  
47    let shared_return = block_on(f).unwrap();
48    println!("shared_return: {:?}", shared_return);
49  
50    for f in threads {
51      f.join().unwrap();
52    }
53  }
```

现在我们来看一下我们的 `select_all` 示例：

```rs
55  fn select_all_example() {
56    let vec = vec![ok(3), err(24), ok(7), ok(9)];
57  
58    let (value, _, vec) = block_on(select_all(vec)).unwrap();
59    println!("Value of vec: = {}", value);
60  
61    let (value, _, vec) = 
      block_on(select_all(vec)).err().unwrap();
62    println!("Value of vec: = {}", value);
63  
64    let (value, _, vec) = block_on(select_all(vec)).unwrap();
65    println!("Value of vec: = {}", value);
66  
67    let (value, _, _) = block_on(select_all(vec)).unwrap();
68    println!("Value of vec: = {}", value);
69  
70    let (tx_1, rx_1) = mpsc::unbounded::();
71    let (tx_2, rx_2) = mpsc::unbounded::();
72    let (tx_3, rx_3) = mpsc::unbounded::();
73  
74    let streams = vec![rx_1, rx_2, rx_3];
75    let stream = select_all_stream(streams);
76  
77    tx_1.unbounded_send(3).unwrap();
78    tx_2.unbounded_send(6).unwrap();
79    tx_3.unbounded_send(9).unwrap();
80  
81    let (value, details) = block_on(stream.next()).unwrap();
82  
83    println!("value for select_all on streams: {:?}", value);
84    println!("stream details: {:?}", details);
85  }
```

现在我们可以添加我们的 `flatten`、`fuse` 和 `inspect` 函数：

```rs
87  fn flatten() {
88    let f = ok::<_, _>(ok::<u32, Never>(100));
89    let f = f.flatten();
90    let results = block_on(f).unwrap();
91    println!("results: {}", results);
92  }
93  
94  fn fuse() {
95    let mut f = ok::<u32, Never>(123).fuse();
96  
97    block_on(poll_fn(move |mut cx| {
98        let first_result = f.poll(&mut cx);
99        let second_result = f.poll(&mut cx);
100       let third_result = f.poll(&mut cx);
101 
102       println!("first result: {:?}", first_result);
103       println!("second result: {:?}", second_result);
104       println!("third result: {:?}", third_result);
105 
106       FINISHED
107     }))
108     .unwrap();
109 }
110 
111 fn inspect() {
112   let f = ok::<u32, Never>(111);
113   let f = f.inspect(|&val| println!("inspecting: {}", val));
114   let results = block_on(f).unwrap();
115   println!("results: {}", results);
116 }
```

然后我们可以添加我们的 `chaining` 示例：

```rs
118 fn chaining() {
119   let (tx, rx) = mpsc::channel(3);
120   let f = tx.send(1)
121     .and_then(|tx| tx.send(2))
122     .and_then(|tx| tx.send(3));
123 
124   let t = thread::spawn(move || {
125     block_on(f.into_future()).unwrap();
126   });
127 
128   t.join().unwrap();
129 
130   let result: Vec<_> = block_on(rx.collect()).unwrap();
131   println!("Result from chaining and_then: {:?}", result);
132 
133   // Chaining streams together
134   let stream1 = iter_result(vec![Ok(10), Err(false)]);
135   let stream2 = iter_result(vec![Err(true), Ok(20)]);
136 
137   let stream = stream1.chain(stream2)
138     .then(|result| Ok::<_, ()>(result));
139 
140   let result: Vec<_> = block_on(stream.collect()).unwrap();
141   println!("Result from chaining our streams together: {:?}",   
      result);
142 }
```

现在我们来看一下 `main` 函数：

```rs
144 fn main() {
145   println!("join_all_example():");
146   join_all_example();
147 
148   println!("\nshared():");
149   shared();
150 
151   println!("\nselect_all_example():");
152   select_all_example();
153 
154   println!("\nflatten():");
155   flatten();
156 
157   println!("\nfuse():");
158   fuse();
159 
160   println!("\ninspect():");
161   inspect();
162 
163   println!("\nchaining():");
164   chaining();
165 }
```

# 它是如何工作的...

`join_all()` 函数：

+   从几个 futures 收集结果并返回一个新的具有 `futures::future::JoinAll<F: Future>` 特性的 future

+   新的 future 将在 `futures::future::join_all` 调用中执行所有聚合 future 的命令，以 FIFO 顺序返回一个 `Vec<T: Future::Item>` 向量

+   一个错误将立即返回并取消其他相关 future

以及 `shared()` 函数：

+   `futures::FutureExt::shared` 将创建一个可以克隆的句柄，它解析为 `<T as futures::future::SharedItem>` 的返回值，它可以延迟到 `T`。

+   适用于在多个线程上轮询 future

+   此方法仅在 Rust 的 `std` 选项启用时才可用（默认情况下是启用的）

+   基础结果是 `futures::future::Shared<Future::Item>`，它实现了 `Send` 和 `Sync` 特性

+   使用 `futures::future::Shared::peek(&self)` 如果任何单个共享句柄已经完成，将返回一个值而不阻塞

接下来，`select_all_example()` 函数：

+   `futures::FutureExt::select_all` 返回一个新的 future，它从一系列向量中选择

+   返回值是 `futures::future::SelectAll`，它允许我们遍历结果

+   一旦某个 future 完成其执行，此函数将返回 future 的项目、执行索引以及还需要处理的 future 列表

然后是 `flatten()` 函数：

+   `futures::FutureExt::flatten` 将将未来组合在一起，其返回的结果是它们的项被展平

+   结果项必须实现 `futures::future::IntoFuture` 特性

接下来是 `fuse()` 函数：

+   当轮询已经返回 `futures::Async::Ready` 或 `Err` 值的未来时，存在 `undefined behavior` 的小概率，例如恐慌或永远阻塞。`futures::FutureExt::fuse` 函数允许我们再次 `poll` 未来，而不用担心 `undefined behavior`，并且总是返回 `futures::Async::Pending`。

+   被融合的未来在完成时将被丢弃，以回收资源。

`inspect()` 函数：

+   `futures::FutureExt::inspect` 允许我们窥视未来的一个项，这在我们在链式组合器时非常有用。

然后是 `chaining()` 函数：

+   我们首先创建一个包含三个值的通道，并使用 `futures::FutureExt::and_then` 组合器 `spawn` 一个线程将这三个值发送到通道的接收者。我们在第 130 行从通道中收集结果。

+   然后，我们在第 134 行和第 135 行将两个流链在一起，在第 140 行发生收集。两个流的结果应该在第 137 行和第 138 行链在一起。

# 参见

+   *使用向量* 和 *将集合作为迭代器访问* 的配方在 第二章，**处理集合**

# 使用流

流是一个事件管道，它异步地向调用者返回一个值。`Streams` 对于需要 `Iterator` 特性的项更有用，而 `Futures` 对于 `Result` 值更合适。当流中发生错误时，错误不会停止流，并且对流的轮询仍然会返回其他结果，直到返回 `None` 值。

`Streams` 和 `Channels` 对于一些人来说可能有点令人困惑。`Streams` 用于连续、缓冲的数据，而 `Channels` 更适合端点之间的完成消息。

# 如何做到这一点...

1.  在 `bin` 文件夹中，创建一个名为 `streams.rs` 的新文件。

1.  添加以下代码，并使用 `cargo run --bin streams` 运行它：

```rs
1   extern crate futures;
2   
3   use std::thread;
4   
5   use futures::prelude::*;
6   use futures::executor::block_on;
7   use futures::future::poll_fn;
8   use futures::stream::{iter_ok, iter_result};
9   use futures::channel::mpsc;
```

1.  现在，让我们添加我们的常量、实现等：

```rs
11  #[derive(Debug)]
12  struct QuickStream {
13    ticks: usize,
14  }
15  
16  impl Stream for QuickStream {
17    type Item = usize;
18    type Error = Never;
19  
20    fn poll_next(&mut self, _cx: &mut task::Context) -> 
      Poll<Option, Self::Error> {
21      match self.ticks {
22        ref mut ticks if *ticks > 0 => {
23          *ticks -= 1;
24          println!("Ticks left on QuickStream: {}", *ticks);
25          Ok(Async::Ready(Some(*ticks)))
26        }
27        _ => {
28          println!("QuickStream is closing!");
29          Ok(Async::Ready(None))
30        }
31      }
32    }
33  }
34  
35  const FINISHED: Result<Async<()>, Never> = Ok(Async::Ready(()));
```

1.  我们的 `quick_streams` 示例将是：

```rs
37  fn quick_streams() {
38    let mut quick_stream = QuickStream { ticks: 10 };
39  
40    // Collect the first poll() call
41    block_on(poll_fn(|cx| {
42        let res = quick_stream.poll_next(cx).unwrap();
43        println!("Quick stream's value: {:?}", res);
44        FINISHED
45      }))
46      .unwrap();
47  
48    // Collect the second poll() call
49    block_on(poll_fn(|cx| {
50        let res = quick_stream.poll_next(cx).unwrap();
51        println!("Quick stream's next svalue: {:?}", res);
52        FINISHED
53      }))
54      .unwrap();
55  
56    // And now we should be starting from 7 when collecting the 
      rest of the stream
57    let result: Vec<_> =  
      block_on(quick_stream.collect()).unwrap();
58    println!("quick_streams final result: {:?}", result);
59  }
```

1.  有几种方法可以遍历流；让我们将它们添加到我们的代码库中：

```rs
61  fn iterate_streams() {
62    use std::borrow::BorrowMut;
63  
64    let stream_response = vec![Ok(5), Ok(7), Err(false), Ok(3)];
65    let stream_response2 = vec![Ok(5), Ok(7), Err(false), Ok(3)];
66  
67    // Useful for converting any of the `Iterator` traits into a 
      `Stream` trait.
68    let ok_stream = iter_ok::<_, ()>(vec![1, 5, 23, 12]);
69    let ok_stream2 = iter_ok::<_, ()>(vec![7, 2, 14, 19]);
70  
71    let mut result_stream = iter_result(stream_response);
72    let result_stream2 = iter_result(stream_response2);
73  
74    let ok_stream_response: Vec<_> = 
      block_on(ok_stream.collect()).unwrap();
75    println!("ok_stream_response: {:?}", ok_stream_response);
76  
77    let mut count = 1;
78    loop {
79      match block_on(result_stream.borrow_mut().next()) {
80        Ok((res, _)) => {
81          match res {
82            Some(r) => println!("iter_result_stream result #{}: 
              {}", count, r),
83            None => { break }
84          }
85        },
86        Err((err, _)) => println!("iter_result_stream had an 
          error #{}: {:?}", count, err),
87      }
88      count += 1;
89    }
90  
91    // Alternative way of iterating through an ok stream
92    let ok_res: Vec<_> = block_on(ok_stream2.collect()).unwrap();
93    for ok_val in ok_res.into_iter() {
94      println!("ok_stream2 value: {}", ok_val);
95    }
96  
97    let (_, stream) = block_on(result_stream2.next()).unwrap();
98    let (_, stream) = block_on(stream.next()).unwrap();
99    let (err, _) = block_on(stream.next()).unwrap_err();
100 
101   println!("The error for our result_stream2 was: {:?}", err);
102 
103   println!("All done.");
104 }
```

1.  现在我们来看我们的通道示例：

```rs
106 fn channel_threads() {
107   const MAX: usize = 10;
108   let (mut tx, rx) = mpsc::channel(0);
109 
110   let t = thread::spawn(move || {
111     for i in 0..MAX {
112       loop {
113         if tx.try_send(i).is_ok() {
114           break;
115         } else {
116           println!("Thread transaction #{} is still pending!", i);
117         }
118       }
119     }
120   });
121 
122   let result: Vec<_> = block_on(rx.collect()).unwrap();
123   for (index, res) in result.into_iter().enumerate() {
124     println!("Channel #{} result: {}", index, res);
125   }
126 
127   t.join().unwrap();
128 }
```

1.  处理错误和通道可以这样做：

```rs
130 fn channel_error() {
131   let (mut tx, rx) = mpsc::channel(0);
132 
133   tx.try_send("hola").unwrap();
134 
135   // This should fail
136   match tx.try_send("fail") {
137     Ok(_) => println!("This should not have been successful"),
138     Err(err) => println!("Send failed! {:?}", err),
139   }
140 
141   let (result, rx) = block_on(rx.next()).ok().unwrap();
142   println!("The result of the channel transaction is: {}",
143        result.unwrap());
144 
145   // Now we should be able send to the transaction since we 
      poll'ed a result already
146   tx.try_send("hasta la vista").unwrap();
147   drop(tx);
148 
149   let (result, rx) = block_on(rx.next()).ok().unwrap();
150   println!("The next result of the channel transaction is: {}",
151        result.unwrap());
152 
153   // Pulling more should result in None
154   let (result, _) = block_on(rx.next()).ok().unwrap();
155   println!("The last result of the channel transaction is:  
      {:?}",
156        result);
157 }
```

1.  我们甚至可以一起处理缓冲区和通道。让我们添加我们的 `channel_buffer` 函数：

```rs
159 fn channel_buffer() {
160   let (mut tx, mut rx) = mpsc::channel::(0);
161 
162   let f = poll_fn(move |cx| {
163     if !tx.poll_ready(cx).unwrap().is_ready() {
164       panic!("transactions should be ready right away!");
165     }
166 
167     tx.start_send(20).unwrap();
168     if tx.poll_ready(cx).unwrap().is_pending() {
169       println!("transaction is pending...");
170     }
171 
172     // When we're still in "Pending mode" we should not be able
173     // to send more messages/values to the receiver
174     if tx.start_send(10).unwrap_err().is_full() {
175       println!("transaction could not have been sent to the 
          receiver due \
176             to being full...");
177     }
178 
179     let result = rx.poll_next(cx).unwrap();
180     println!("the first result is: {:?}", result);
181     println!("is transaction ready? {:?}",
182          tx.poll_ready(cx).unwrap().is_ready());
183 
184     // We should now be able to send another message 
        since we've pulled
185     // the first message into a result/value/variable.
186     if !tx.poll_ready(cx).unwrap().is_ready() {
187       panic!("transaction should be ready!");
188     }
189 
190     tx.start_send(22).unwrap();
191     let result = rx.poll_next(cx).unwrap();
192     println!("new result for transaction is: {:?}", result);
193 
194     FINISHED
195   });
196 
197   block_on(f).unwrap();
198 }
```

1.  虽然我们正在使用 futures crate，但这并不意味着一切都必须是并发的。添加以下示例以演示如何使用通道进行阻塞：

```rs
200 fn channel_threads_blocking() {
201   let (tx, rx) = mpsc::channel::(0);
202   let (tx_2, rx_2) = mpsc::channel::<()>(2);
203 
204   let t = thread::spawn(move || {
205     let tx_2 = tx_2.sink_map_err(|_| panic!());
206     let (a, b) = 
        block_on(tx.send(10).join(tx_2.send(()))).unwrap();
207 
208     block_on(a.send(30).join(b.send(()))).unwrap();
209   });
210 
211   let (_, rx_2) = block_on(rx_2.next()).ok().unwrap();
212   let (result, rx) = block_on(rx.next()).ok().unwrap();
213   println!("The first number that we sent was: {}", 
      result.unwrap());
214 
215   drop(block_on(rx_2.next()).ok().unwrap());
216   let (result, _) = block_on(rx.next()).ok().unwrap();
217   println!("The second number that we sent was: {}", 
      result.unwrap());
218 
219   t.join().unwrap();
220 }
```

1.  有时候我们需要像无界通道这样的概念；让我们添加我们的 `channel_unbounded` 函数：

```rs
222 fn channel_unbounded() {
223   const MAX_SENDS: u32 = 5;
224   const MAX_THREADS: u32 = 4;
225   let (tx, rx) = mpsc::unbounded::();
226 
227   let t = thread::spawn(move || {
228     let result: Vec<_> = block_on(rx.collect()).unwrap();
229     for item in result.iter() {
230       println!("channel_unbounded: results on rx: {:?}", item);
231     }
232   });
233 
234   for _ in 0..MAX_THREADS {
235     let tx = tx.clone();
236 
237     thread::spawn(move || {
238       for _ in 0..MAX_SENDS {
239         tx.unbounded_send(1).unwrap();
240       }
241     });
242   }
243 
244   drop(tx);
245 
246   t.join().ok().unwrap();
247 }
```

1.  现在我们可以添加我们的 `main` 函数：

```rs
249 fn main() {
250   println!("quick_streams():");
251   quick_streams();
252 
253   println!("\niterate_streams():");
254   iterate_streams();
255 
256   println!("\nchannel_threads():");
257   channel_threads();
258 
259   println!("\nchannel_error():");
260   channel_error();
261 
262   println!("\nchannel_buffer():");
263   channel_buffer();
264 
265   println!("\nchannel_threads_blocking():");
266   channel_threads_blocking();
267 
268   println!("\nchannel_unbounded():");
269   channel_unbounded();
270 }
```

# 它是如何工作的...

首先，让我们谈谈 `QuickStream` 结构：

+   `poll_next()`函数将不断被调用，并且随着每次迭代的进行，`i`的 ticks 属性将递减`1`。

+   当 ticks 属性达到`0`时，轮询将停止，并返回`futures::Async::Ready<None>`。

在`quick_streams()`函数内部：

+   我们通过使用`futures::future::poll_on(f: FnMut(|cx: Context|))`构建一个`futures::task::Context`，这样我们就可以在 42 和 50 行显式调用`QuickStream`的`poll_next()`函数。

+   由于我们在 38 行声明了`10`个 ticks，我们的前两个`block_on`的`poll_next()`调用应该产生`9`和`8`。

+   下一个`block_on`调用，在 57 行，将不断轮询`QuickStream`，直到 ticks 属性等于零时返回`futures::Async::Ready<None>`。

在`iterate_streams()`内部：

+   `futures::stream::iter_ok`将一个`Iterator`转换为一个`Stream`，它将始终准备好返回下一个值。

+   `futures::stream::iter_result`与`iter_ok`做同样的事情，只是我们使用`Result`值而不是`Ok`值。

+   在 78 到 89 行，我们遍历流的输出并打印出一些信息，这取决于值是`Ok`还是`Error`类型。如果我们的流返回了`None`类型，那么我们将退出循环。

+   92 到 95 行显示了使用`into_iter()`调用迭代流`Ok`结果的另一种方法。

+   97 到 99 行显示了迭代流`Result`返回类型的另一种方法。

循环、迭代结果和`collect()`调用是同步的。我们只使用这些函数进行演示/教育目的。在实际应用中，将使用如`map()`、`filter()`、`and_then()`等组合子。

`channel_threads()`函数：

+   在 107 行，我们定义了我们想要尝试的最大发送次数。

+   在 108 行，我们声明了一个通道以发送消息。通道容量是`buffer size（futures::channel::mpsc::channel 的参数）+ 发送者数量`（每个发送者都保证在通道中有一个槽位）。通道将返回一个`futures::channel::mpsc::Receiver<T>`，它实现了`Stream`特质，以及一个`futures::channel::mpsc::Sender<T>`，它实现了`Sink`特质。

+   110 到 120 行是我们`spawn`一个线程并尝试发送 10 个信号的地方，循环直到每个发送都成功发送。

+   我们在 122 到 125 行收集并显示我们的结果，并在 127 行合并我们的线程。

`channel_error()`部分：

+   在 131 行，我们使用`0 usize`缓冲区作为参数声明我们的通道，这给我们提供了一个初始发送者的槽位。

+   我们在 133 行成功发送了第一条消息。

+   136 到 139 行应该失败，因为我们正在尝试向一个被认为是满的通道发送消息（因为我们没有收到值，删除初始发送者，刷新流等）。

+   在第 146 行，我们使用发送者的`futures::channel::mpsc::Sender::try_send(&mut self, msg: T)`函数，除非我们不使用第 147 行的`drop(T)`调用发送者的销毁方法，否则它不会阻塞我们的线程。

+   在接收到最后一个值之后，对流的任何额外轮询都将始终返回`None`。

接下来是`channel_buffer()`函数：

+   我们在第 162 行使用`poll_fn()`设置了一个 future 闭包。

+   我们在第 163 行到第 165 行使用其`futures::sink::poll_ready(&mut self, cx: &mut futures::task::Context)`方法检查我们的发送者是否准备好被轮询。

+   接收器有一个名为`futures::sink::start_send(&mut self, item: <Self as Sink>::SinkItem) -> Result<(), <Self as Sink>::SinkError>`的方法，它准备要发送的消息，但不会发送，直到我们刷新或关闭接收器。`poll_flush()`通常用于确保从接收器发送了每条消息。

+   使用`futures::stream::poll_next(&mut self, cx: &mut futures::task::Context)`方法轮询流以获取下一个值，也将减轻接收器/发送者中的空间，就像我们在第 179 行所做的那样。

+   我们可以检查我们的发送者是否准备好，就像我们在第 182 行使用`futures::Async::is_ready(&self) -> bool`方法所做的那样。

+   我们最终的值应该是`22`，并从第 192 行显示到控制台。

然后是`channel_threads_blocking()`函数：

+   首先，我们在第 201 行和第 202 行设置了我们的通道。

+   然后我们`spawn`了一个线程，该线程将`tx_2`的所有错误映射到`panic!`（第 205 行），然后我们向第一个通道发送`10`的值，同时将第二个发送者与`()`值连接起来（第 206 行）。在第 208 行，我们向第二个通道发送`30`的值和另一个空值`()`。

+   在第 211 行我们轮询第二个通道，它将包含一个值为`()`的值。

+   在第 212 行我们轮询第一个通道，它将包含一个值为`10`的值。

+   我们在第 215 行丢弃了第二个通道的接收者，因为我们需要在第 208 行的`tx_2.send()`调用上关闭或刷新（`tx_2`在这一行被称为变量`b`）。

+   执行丢弃操作后，我们最终可以从第一个通道的发送者返回第二个值，这应该是`30`。

以及`channel_unbounded()`函数：

+   在第 225 行我们声明了一个`unbounded channel`，这意味着只要接收者没有关闭，向这个通道发送消息总是会成功。消息将根据需要缓冲，由于这个通道是无界的，因此我们的应用程序可能会耗尽我们的可用内存。

+   从第 227 行到第 232 行，`spawn`了一个线程来收集接收者的所有消息（第 228 行），我们在第 229 行遍历它们。第 230 行的项目是一个元组，包含接收消息的索引和消息的值（在我们的例子中，这始终是 1）。

+   从第 237 行到第 241 行将`spawn`出线程的数量（使用`MAX_THREADS`常量）以及我们想要每个线程发送的次数（使用`MAX_THREADS`常量）。

+   在第 244 行我们将丢弃（关闭）通道的发送者，以便我们可以收集第 228 行的所有消息。

+   在第 246 行，我们将生成的线程与当前线程连接，这将执行收集和迭代命令（第 228 行至第 231 行）。

# 使用 Sinks

输出端是通道、套接字、管道等 *发送端*，其中可以异步发送消息。输出端通过启动发送信号进行通信，然后进行轮询。在使用输出端时需要注意的一点是，它们可能会耗尽发送空间，这将阻止发送更多消息。

# 如何做到...

1.  在 `bin` 文件夹中，创建一个名为 `sinks.rs` 的新文件。

1.  添加以下代码并使用 `cargo run --bin sinks` 运行它：

```rs
1   extern crate futures;
2   
3   use futures::prelude::*;
4   use futures::future::poll_fn;
5   use futures::executor::block_on;
6   use futures::sink::flush;
7   use futures::stream::iter_ok;
8   use futures::task::{Waker, Context};
9   
10  use std::mem;
```

1.  让我们添加使用向量作为 `sinks` 的示例：

```rs
12  fn vector_sinks() {
13    let mut vector = Vec::new();
14    let result = vector.start_send(0);
15    let result2 = vector.start_send(7);
16  
17    println!("vector_sink: results of sending should both be 
      Ok(()): {:?} and {:?}",
18         result,
19         result2);
20    println!("The entire vector is now {:?}", vector);
21  
22    // Now we need to flush our vector sink.
23    let flush = flush(vector);
24    println!("Our flush value: {:?}", flush);
25    println!("Our vector value: {:?}",  
      flush.into_inner().unwrap());
26  
27    let vector = Vec::new();
28    let mut result = vector.send(2);
29    // safe to unwrap since we know that we have not flushed the 
      sink yet
30    let result = result.get_mut().unwrap().send(4);
31  
32    println!("Result of send(): {:?}", result);
33    println!("Our vector after send(): {:?}", 
      result.get_ref().unwrap());
34  
35    let vector = block_on(result).unwrap();
36    println!("Our vector should already have one element: {:?}", 
      vector);
37  
38    let result = block_on(vector.send(2)).unwrap();
39    println!("We can still send to our stick to ammend values: 
      {:?}",
40         result);
41  
42    let vector = Vec::new();
43    let send_all = vector.send_all(iter_ok(vec![1, 2, 3]));
44    println!("The value of vector's send_all: {:?}", send_all);
45  
46    // Add some more elements to our vector...
47    let (vector, _) = block_on(send_all).unwrap();
48    let (result, _) = block_on(vector.send_all(iter_ok(vec![0, 6, 
      7]))).unwrap();
49    println!("send_all's return value: {:?}", result);
50  }
```

我们可以映射/转换我们的 `sinks` 值。让我们添加我们的 `mapping_sinks` 示例：

```rs
52  fn mapping_sinks() {
53    let sink = Vec::new().with(|elem: i32| Ok::<i32, Never>(elem 
      * elem));
54  
55    let sink = block_on(sink.send(0)).unwrap();
56    let sink = block_on(sink.send(3)).unwrap();
57    let sink = block_on(sink.send(5)).unwrap();
58    println!("sink with() value: {:?}", sink.into_inner());
59  
60    let sink = Vec::new().with_flat_map(|elem| iter_ok(vec![elem; 
      elem].into_iter().map(|y| y * y)));
61  
62    let sink = block_on(sink.send(0)).unwrap();
63    let sink = block_on(sink.send(3)).unwrap();
64    let sink = block_on(sink.send(5)).unwrap();
65    let sink = block_on(sink.send(7)).unwrap();
66    println!("sink with_flat_map() value: {:?}", 
      sink.into_inner());
67  }
```

我们甚至可以向多个 `sinks` 发送消息。让我们添加我们的 `fanout` 函数：

```rs
69  fn fanout() {
70    let sink1 = vec![];
71    let sink2 = vec![];
72    let sink = sink1.fanout(sink2);
73    let stream = iter_ok(vec![1, 2, 3]);
74    let (sink, _) = block_on(sink.send_all(stream)).unwrap();
75    let (sink1, sink2) = sink.into_inner();
76  
77    println!("sink1 values: {:?}", sink1);
78    println!("sink2 values: {:?}", sink2);
79  }
```

接下来，我们将想要实现一个自定义输出端的结构。有时我们的应用程序将需要我们手动刷新 `sinks` 而不是自动执行。让我们添加我们的 `ManualSink` 结构：

```rs
81  #[derive(Debug)]
82  struct ManualSink {
83    data: Vec,
84    waiting_tasks: Vec,
85  }
86  
87  impl Sink for ManualSink {
88    type SinkItem = Option; // Pass None to flush
89    type SinkError = ();
90  
91    fn start_send(&mut self, op: Option) -> Result<(), 
      Self::SinkError> {
92      if let Some(item) = op {
93        self.data.push(item);
94      } else {
95        self.force_flush();
96      }
97  
98      Ok(())
99    }
100 
101   fn poll_ready(&mut self, _cx: &mut Context) -> Poll<(), ()> {
102     Ok(Async::Ready(()))
103   }
104 
105   fn poll_flush(&mut self, cx: &mut Context) -> Poll<(), ()> {
106     if self.data.is_empty() {
107       Ok(Async::Ready(()))
108     } else {
109       self.waiting_tasks.push(cx.waker().clone());
110       Ok(Async::Pending)
111     }
112   }
113 
114   fn poll_close(&mut self, _cx: &mut Context) -> Poll<(), ()> {
115     Ok(().into())
116   }
117 }
118 
119 impl ManualSink {
120   fn new() -> ManualSink {
121     ManualSink {
122       data: Vec::new(),
123       waiting_tasks: Vec::new(),
124     }
125   }
126 
127   fn force_flush(&mut self) -> Vec {
128     for task in self.waiting_tasks.clone() {
129       println!("Executing a task before replacing our values");
130       task.wake();
131     }
132 
133     mem::replace(&mut self.data, vec![])
134   }
135 }
```

现在是我们的 `manual flush` 函数：

```rs
137 fn manual_flush() {
138   let mut sink = ManualSink::new().with(|x| Ok::<Option, ()> 
      (x));
139   let _ = sink.get_mut().start_send(Some(3));
140   let _ = sink.get_mut().start_send(Some(7));
141 
142   let f = poll_fn(move |cx| -> Poll<Option<_>, Never> {
143     // Try to flush our ManualSink
144     let _ = sink.get_mut().poll_flush(cx);
145     let _ = flush(sink.get_mut());
146 
147     println!("Our sink after trying to flush: {:?}", 
        sink.get_ref());
148 
149     let results = sink.get_mut().force_flush();
150     println!("Sink data after manually flushing: {:?}",
151          sink.get_ref().data);
152     println!("Final results of sink: {:?}", results);
153 
154     Ok(Async::Ready(Some(())))
155   });
156 
157   block_on(f).unwrap();
158 }
```

最后，我们可以添加我们的 `main` 函数：

```rs
160 fn main() {
161   println!("vector_sinks():");
162   vector_sinks();
163 
164   println!("\nmapping_sinks():");
165   mapping_sinks();
166 
167   println!("\nfanout():");
168   fanout();
169 
170   println!("\nmanual_flush():");
171   manual_flush();
172 }
```

# 它是如何工作的...

首先，让我们看看 `futures::Sink` 特性本身：

```rs
pub trait Sink {
    type SinkItem;
    type SinkError;

    fn poll_ready(
        &mut self, 
        cx: &mut Context
    ) -> Result<Async<()>, Self::SinkError>;
    fn start_send(
        &mut self, 
        item: Self::SinkItem
    ) -> Result<(), Self::SinkError>;
    fn poll_flush(
        &mut self, 
        cx: &mut Context
    ) -> Result<Async<()>, Self::SinkError>;
    fn poll_close(
        &mut self, 
        cx: &mut Context
    ) -> Result<Async<()>, Self::SinkError>;
}
```

我们已经熟悉了来自 futures 和 streams 的 `Item` 和 `Error` 概念，因此我们将继续到所需的函数：

+   在每次尝试使用 `start_send` 之前，必须使用返回值 `Ok(futures::Async::Ready(()))` 调用 `poll_ready`。如果输出端收到错误，输出端将无法再接收项。

+   如前所述，`start_send` 准备要发送的消息，但只有在刷新或关闭输出端之前，它才会发送。如果输出端使用缓冲区，则 `Sink::SinkItem` 不会在缓冲区完全完成后被处理。

+   `poll_flush` 将刷新输出端，这将允许我们收集正在处理的项。如果输出端在缓冲区中没有更多项，则将返回 `futures::Async::Ready`，否则输出端将返回 `futures::Async::Pending`。

+   `poll_close` 将刷新并关闭输出端，遵循与 `poll_flush` 相同的返回规则。

现在，让我们看看我们的 `vector_sinks()` 函数：

+   输出端是为 `Vec<T>` 类型实现的，因此我们可以声明一个可变向量并使用 `start_send()` 函数，该函数将立即将我们的值轮询到第 13 行至第 15 行的向量中。

+   在第 28 行，我们使用 `futures::SinkExt::send(self, item: Self::SinkItem)`，这将完成在项被处理并通过输出端刷新后。建议使用 `futures::SinkExt::send_all` 来批量发送多个项，而不是在每次发送调用之间手动刷新（如第 43 行所示）。

我们的 `mapping_sinks()` 函数：

+   第 51 行展示了如何使用 `futures::SinkExt::with` 函数在接收器内映射/操作元素。这个函数产生一个新的接收器，它遍历每个项目，并将最终值 *作为一个 future* 发送到 *父* 接收器。

+   第 60 行说明了 `futures::SinkExt::flat_with_map` 函数，它基本上与 `futures::SinkExt::with` 函数具有相同的功能，除了每个迭代的项被作为流值发送到 *父* 接收器，并且将返回一个 `Iterator::flat_map` 值而不是 `Iterator::map`。

接下来是 `fanout()` 函数：

+   `futures::SinkExt::fanout` 函数允许我们一次向多个接收器发送消息，就像我们在第 72 行所做的那样。

然后执行 `manual_flush()`：

+   我们首先使用 `ManualSink<T>` 构造函数实现我们自己的 `Sink` 特性（第 81 到 135 行）。我们的 `ManualSink` 的 `poll_flush` 方法只有在我们的数据向量为空时才会返回 `Async::Ready()`，否则，我们将任务（`futures::task::Waker`）推入通过 `waiting_tasks` 属性的队列中。我们在 `force_flush()` 函数（第 128 行）中使用 `waiting_tasks` 属性来手动 *唤醒* 我们的任务（第 130 行）。

+   在第 138 到 140 行，我们构建了我们的 `ManualSink<Option<i32>>` 并开始发送一些值。

+   我们在第 142 行使用 `poll_fn` 来快速构建一个 `futures::task::Context`，以便我们可以将此值传递给底层的轮询调用。

+   在第 144 行，我们手动调用我们的 `poll_flush()` 函数，由于任务被放置在 `waiting_tasks` 属性中，所以它不会执行实际的任务。

+   直到我们调用 `force_flush()`，我们的接收器将不会返回任何值（如第 150-151 行所示）。一旦这个函数被调用，并且底层的 `Waker` 任务执行完毕，我们就可以看到我们之前发送的消息（第 152 行，第 139 和 140 行）。

# 使用单次发送通道

单次发送通道在你只需要向通道发送一条消息时很有用。单次发送通道适用于那些实际上只需要更新/通知一次的任务，例如，接收者是否阅读了你的消息，或者作为任务管道中的最终目的地，通知最终用户任务已完成。

# 如何实现...

1.  在 `bin` 文件夹内，创建一个名为 `oneshot.rs` 的新文件。

1.  添加以下代码并使用 `cargo run --bin oneshot` 运行它：

```rs
1   extern crate futures;
2   
3   use futures::prelude::*;
4   use futures::channel::oneshot::*;
5   use futures::executor::block_on;
6   use futures::future::poll_fn;
7   use futures::stream::futures_ordered;
8   
9   const FINISHED: Result<Async<()>, Never> =
    Ok(Async::Ready(()));
10  
11  fn send_example() {
12    // First, we'll need to initiate some oneshot channels like 
      so:
13    let (tx_1, rx_1) = channel::();
14    let (tx_2, rx_2) = channel::();
15    let (tx_3, rx_3) = channel::();
16  
17    // We can decide if we want to sort our futures by FIFO 
      (futures_ordered)
18    // or if the order doesn't matter (futures_unordered)
19    // Note: All futured_ordered()'ed futures must be set as a 
      Box type
20    let mut ordered_stream = futures_ordered(vec![
21      Box::new(rx_1) as Box<Future>,
22      Box::new(rx_2) as Box<Future>,
23    ]);
24  
25    ordered_stream.push(Box::new(rx_3) as Box<Future>);
26  
27    // unordered example:
28    // let unordered_stream = futures_unordered(vec![rx_1, rx_2, 
      rx_3]);
29  
30    // Call an API, database, etc. and return the values (in our  
      case we're typecasting to u32)
31    tx_1.send(7).unwrap();
32    tx_2.send(12).unwrap();
33    tx_3.send(3).unwrap();
34  
35    let ordered_results: Vec<_> = 
      block_on(ordered_stream.collect()).unwrap();
36    println!("Ordered stream results: {:?}", ordered_results);
37  }
38  
39  fn check_if_closed() {
40    let (tx, rx) = channel::();
41  
42    println!("Is our channel canceled? {:?}", tx.is_canceled());
43    drop(rx);
44  
45    println!("Is our channel canceled now? {:?}", 
      tx.is_canceled());
46  }
47  
48  fn check_if_ready() {
49    let (mut tx, rx) = channel::();
50    let mut rx = Some(rx);
51  
52    block_on(poll_fn(|cx| {
53        println!("Is the transaction pending? {:?}",
54             tx.poll_cancel(cx).unwrap().is_pending());
55        drop(rx.take());
56  
57        let is_ready = tx.poll_cancel(cx).unwrap().is_ready();
58        let is_pending = 
          tx.poll_cancel(cx).unwrap().is_pending();
59  
60        println!("Are we ready? {:?} This means that the pending 
          should be false: {:?}",
61             is_ready,
62             is_pending);
63        FINISHED
64      }))
65      .unwrap();
66  }
67  
68  fn main() {
69    println!("send_example():");
70    send_example();
71  
72    println!("\ncheck_if_closed():");
73    check_if_closed();
74  
75    println!("\ncheck_if_ready():");
76    check_if_ready();
77  }
```

# 它是如何工作的...

在我们的 `send_example()` 函数中：

+   在第 13 到 15 行，我们设置了三个 `oneshot` 通道。

+   在第 20 到 23 行，我们使用 `futures::stream::futures_ordered`，它将 future 的列表（任何 `IntoIterator` 值）转换为在先进先出（FIFO）基础上产生结果的 `Stream`。如果任何底层 future 在下一个 future 被调用之前没有完成，这个函数将等待直到长时间运行的 future 完成，然后将其内部重新排序到正确的顺序。

+   第 25 行显示我们可以将额外的 futures 推入 `futures_ordered` 迭代器中。

+   第 28 行演示了另一个不依赖于基于 FIFO 排序的排序函数，称为 `futures::stream::futures_unordered`。这个函数的性能将优于其对应物 `futures_ordered`，但对我们这个示例来说，我们发送的值不足以产生差异。

+   在第 31 到 33 行，我们向我们的通道发送值，模拟从 API、数据库等返回值的过程。如果发送成功，则返回 `Ok(())`，否则返回 `Err` 类型。

+   在我们最后两行（35 和 36）中，我们收集 `futures_ordered` 的值并将它们显示到控制台。

接下来，是 `check_if_closed()` 函数：

+   我们的通道应该保持打开状态，直到我们显式地丢弃/销毁接收器（或向通道发送一个值）。我们可以通过调用 `futures::channel::oneshot::Sender::is_canceled(&self) -> bool` 函数来检查我们接收器的状态，我们在第 42 和 45 行已经这样做过了。

然后是 `check_if_ready()` 函数：

+   在第 50 行，我们显式地为一个 oneshot 的接收器分配一个值，这将使我们的接收器处于挂起状态（因为它已经有一个值了）。

+   我们在第 55 行丢弃了我们的接收器，我们可以通过使用我们的发送器的 `futures::channel::oneshot::Sender::poll_cancel` 函数来检查我们的接收器是否就绪，我们在第 57 和 58 行使用它。`poll_cancel` 将在接收器被丢弃时返回 `Ok(Async::Ready)`，如果接收器没有被丢弃，则返回 `Ok(Async::Pending)`。

# 返回 futures

`Future` 特性依赖于三个主要成分：一个类型、一个错误和一个返回 `Result<Async<T>, E>` 结构的 `poll()` 函数。`poll()` 方法永远不会阻塞主线程，而 `Async<T>` 是一个具有两个变体的枚举器：`Ready(T)` 和 `Pending`。定期地，`poll()` 方法将由任务上下文的 `waker()` 特性调用，位于 `futures::task::context::waker` 中，直到有值可以返回。

# 如何做到这一点...

1.  在 `src/bin` 文件夹中，创建一个名为 `returning.rs` 的文件。

1.  添加以下代码并使用 `cargo run —bin returning` 运行它：

```rs
1   extern crate futures;
2   
3   use futures::executor::block_on;
4   use futures::future::{join_all, Future, FutureResult, ok};
5   use futures::prelude::*;
6   
7   #[derive(Clone, Copy, Debug, PartialEq)]
8   enum PlayerStatus {
9     Loading,
10    Default,
11    Jumping,
12  }
13  
14  #[derive(Clone, Copy, Debug)]
15  struct Player {
16    name: &'static str,
17    status: PlayerStatus,
18    score: u32,
19    ticks: usize,
20  }
```

1.  现在是结构体的实现：

```rs
22  impl Player {
23    fn new(name: &'static str) -> Self {
24      let mut ticks = 1;
25      // Give Bob more ticks explicitly
26      if name == "Bob" {
27        ticks = 5;
28      }
29  
30      Player {
31        name: name,
32        status: PlayerStatus::Loading,
33        score: 0,
34        ticks: ticks,
35      }
36    }
37  
38    fn set_status(&mut self, status: PlayerStatus) ->  
      FutureResult<&mut Self, Never> {
39      self.status = status;
40      ok(self)
41    }
42  
43    fn can_add_points(&mut self) -> bool {
44      if self.status == PlayerStatus::Default {
45        return true;
46      }
47  
48      println!("We couldn't add any points for {}!", self.name);
49      return false;
50    }
51  
52    fn add_points(&mut self, points: u32) -> Async<&mut Self> {
53      if !self.can_add_points() {
54        Async::Ready(self)
55      } else {
56        let new_score = self.score + points;
57        // Here we would send the new score to a remote server
58        // but for now we will manaully increment the player's 
          score.
59  
60        self.score = new_score;
61  
62        Async::Ready(self)
63      }
64    }
65  }
66  
67  impl Future for Player {
68    type Item = Player;
69    type Error = ();
70  
71    fn poll(&mut self, cx: &mut task::Context) -> 
      Poll<Self::Item, Self::Error> {
72      // Presuming we fetch our player's score from a
73      // server upon initial load.
74      // After we perform the fetch send the Result value.
75  
76      println!("Player {} has been poll'ed!", self.name);
77  
78      if self.ticks == 0 {
79        self.status = PlayerStatus::Default;
80        Ok(Async::Ready(*self))
81      } else {
82        self.ticks -= 1;
83        cx.waker().wake();
84        Ok(Async::Pending)
85      }
86    }
87  }
```

1.  接下来，我们将添加我们的 `helper` 函数和用于给玩家添加分数的 `Async` 函数：

```rs
89  fn async_add_points(player: &mut Player,
90            points: u32)
91            -> Box<Future + Send> {
92    // Presuming that player.add_points() will send the points to a
93    // database/server over a network and returns an updated
94    // player score from the server/database.
95    let _ = player.add_points(points);
96  
97    // Additionally, we may want to add logging mechanisms,
98    // friend notifications, etc. here.
99  
100   return Box::new(ok(*player));
101 }
102 
103 fn display_scoreboard(players: Vec<&Player>) {
104   for player in players {
105     println!("{}'s Score: {}", player.name, player.score);
106   }
107 }
```

1.  最后，实际的用法：

```rs
109 fn main() {
110   let mut player1 = Player::new("Bob");
111   let mut player2 = Player::new("Alice");
112 
113   let tasks = join_all(vec![player1, player2]);
114 
115   let f = join_all(vec![
116     async_add_points(&mut player1, 5),
117     async_add_points(&mut player2, 2),
118   ])
119     .then(|x| {
120       println!("First batch of adding points is done.");
121       x
122     });
123 
124   block_on(f).unwrap();
125 
126   let players = block_on(tasks).unwrap();
127   player1 = players[0];
128   player2 = players[1];
129 
130   println!("Scores should be zero since no players were  
      loaded");
131   display_scoreboard(vec![&player1, &player2]);
132 
133   // In our minigame, a player cannot score if they are  
      currently
134   // in the air or "jumping."
135   // Let's make one of our players' status set to the jumping 
      status.
136 
137   let f = 
       player2.set_status(PlayerStatus::Jumping).and_then(move |mut 
       new_player2| {
138     async_add_points(&mut player1, 10)
139       .and_then(move |_| {
140         println!("Finished trying to give Player 1 points.");
141         async_add_points(&mut new_player2, 2)
142       })
143       .then(move |new_player2| {
144         println!("Finished trying to give Player 2 points.");
145         println!("Player 1 (Bob) should have a score of 10 and 
            Player 2 (Alice) should \
146               have a score of 0");
147 
148         // unwrap is used here to since
149         display_scoreboard(vec![&player1, 
            &new_player2.unwrap()]);
150         new_player2
151       })
152   });
153 
154   block_on(f).unwrap();
155 
156   println!("All done!");
157 }
```

# 它是如何工作的...

让我们先介绍参与这个示例的结构：

+   `PlayerStatus` 是一个枚举器，用于在玩家的实例上维护一个 *全局* 状态。变体有：

    +   “加载”，这是初始状态

    +   “默认”，在我们完成加载玩家的统计数据后应用

    +   “跳跃”是一种特殊状态，由于游戏规则，它不会允许我们向玩家的计分板上添加分数

+   `Player` 包含玩家主要属性，以及一个名为 `ticks` 的特殊属性，它存储了我们想要在将玩家的状态从 `Loading` 转换为 `Default` 之前通过 `poll()` 运行的周期数。

现在，让我们来看一下我们的实现：

+   跳转到 `Player` 结构中的 `fn set_status(&mut self, status: PlayerStatus) -> FutureResult<&mut Self, Never>` 函数，我们会注意到返回值是 `FutureResult`，这告诉 futures 这个函数将立即从 `futures::futures` 的 `result()`、`ok()` 或 `err()` 函数返回一个计算值。这对于快速原型设计我们的应用程序非常有用，同时能够利用我们的 `executors` 和未来组合器。

+   在 `fn add_points(&mut self, points: u32) -> Async<&mut Self>` 函数中，我们立即返回我们的 `Async` 值，因为我们目前没有服务器可以使用，但我们会为需要异步计算的函数实现 `Async<T>` 值，基于 `FutureResult`。

+   我们使用玩家的 `ticks` 属性来模拟网络请求所需的时间。`Poll<I, E>` 将会一直执行，只要我们返回 `Async::Pending`（行 [x]）。执行器需要知道是否需要再次轮询任务。任务的 `Waker` 特性负责处理这些通知，我们可以在第 83 行手动调用它，使用 `cx.waker().wake()`。一旦玩家的 `ticks` 属性达到零，我们发送一个 `Async::Ready(self)` 信号，这告诉执行器不再轮询此函数。

对于我们的 `async_add_points()` 辅助方法：

+   我们返回 `Box<Future<Item = Player, Error = Never> + Send>`，这告诉 futures 这个函数最终将返回一个 `Player` 类型的值（因为我们 `Never` 返回错误）。

+   返回值中的 `+ Send` 部分对于我们的当前代码库不是必需的，但将来，我们可能希望将这些任务卸载到其他线程上，因为执行器需要这样做。跨线程生成需要我们返回 `futures::prelude::Never` 类型作为错误，以及一个 `'static` 变量。

+   当使用组合器（如 `then` 和 `and_then`）调用未来函数时，我们需要返回 `Never` 错误类型或与同一组合器流中调用的其他每个未来函数相同的错误类型。

最后，让我们来看一下我们的主块：

+   我们使用 `futures::future::join_all` 函数，它接受任何包含所有 `InfoFuture` 特性元素的 `IntoIterator`（这应该是所有未来函数）。这要么收集并返回排序为 FIFO 的 `Vec<T>`，要么在集合中的任何未来函数返回第一个错误时取消执行，这成为 `join_all()` 调用的返回值。

+   `then()` 和 `and_then()` 是内部使用 `future::Chain` 并返回 `Future` 特性值的组合器，这允许我们添加更多组合器。有关组合器的更多信息，请参阅 *使用组合器和工具* 部分。

+   `block_on()`是一个执行器方法，它处理任何未来函数或值作为其输入，并返回`Result<Future::Item, Future::Error>`。当运行此方法时，包含该方法的函数将阻塞，直到未来（s）完成。派生的任务将在默认执行器上执行，但它们可能不会在`block_on`完成其任务（s）之前完成。如果`block_on()`在派生任务之前完成，则那些派生的任务将被丢弃。

+   我们还可以使用`block_on()`作为快速运行我们的周期/滴答和执行任务（s）的方法，这会调用我们的`poll()`函数。我们在第 124 行使用此方法来*最初加载玩家*进入游戏。

# 还有更多...

返回未来的`box()`方法会导致堆上的额外分配。另一种返回未来的方法是使用 Rust 的`nightly`版本，或者等待此问题[`github.com/rust-lang/rust/issues/34511`](https://github.com/rust-lang/rust/issues/34511)得到解决。新的`async_add_points()`方法将返回一个隐含的`Future`特质，其外观如下：

```rs
fn async_add_points<F>(f: F, player: &mut Player, points: u32) -> impl Future<Item = Player, Error = F::Error>
where F: Future<Item = Player>,
{
    // Presuming that player.add_points() will send the points to a  
    // database/server over a network and returns
    // an updated player score from the server/database.
    let _ = player.add_points(points).flatten();

    // Additionally, we may want to add logging mechanisms, friend 
    notifications, etc. here.

    return f.map(player.clone());
}
```

如果我们对一个未来调用`poll()`超过一次，Rust 可能会引起`undefined behavior`。这个问题可以通过使用`into_stream()`方法将未来转换为流或使用添加了微小运行时开销的`fuse()`适配器来缓解。

任务通常是通过使用`executor`（例如`block_on()`辅助函数）来执行/轮询的。你可以通过创建`task::Context`并直接从任务中调用`poll()`来手动执行任务。作为一般规则，建议不要手动调用`poll()`，而应该由执行器自动管理轮询。

# 参见

+   第五章中的*数据结构高级*章节中的*数据装箱*配方

+   第七章，*并行性与 Rayon*

# 使用 BiLocks 锁定资源

当我们需要在多个线程之间存储一个值，并且该值最多有两个所有者时，会使用 BiLocks。BiLock 类型的适用用途包括分割 TCP/UDP 数据用于读写，或者在接收器和流之间添加一层（用于日志记录、监控等），或者它可以是同时作为接收器和流。

当使用带有额外 crate（例如 tokio 或 hyper）的未来时，了解 BiLocks 可以帮助我们将数据包装在其他 crate 的常用方法周围。这将使我们能够在不等待 crate 的维护者明确支持并发的情况下，在现有 crate 之上构建未来和并发。BiLocks 是一个非常底层的实用工具，但了解它们的工作原理可以帮助我们在未来的（Web）应用中走得更远。

在下一章中，我们将主要关注 Rust 的网络编程，但我们也会练习将 futures 与其他 crate 集成。如果需要将 TCP/UDP 流分割成互斥状态，可以使用 BiLocks，尽管使用我们将要使用的 crate 时这样做不是必需的。

# 如何实现...

1.  在`src/bin`文件夹中，创建一个名为`bilocks.rs`的文件。

1.  添加以下代码并使用`cargo run —bin bilocks`运行它：

```rs
1   extern crate futures;
2   extern crate futures_util;
3   
4   use futures::prelude::*;
5   use futures::executor::LocalPool;
6   use futures::task::{Context, LocalMap, Wake, Waker};
7   use futures_util::lock::BiLock;
8   
9   use std::sync::Arc;
10  
11  struct FakeWaker;
12  impl Wake for FakeWaker {
13    fn wake(_: &Arc) {}
14  }
15  
16  struct Reader {
17    lock: BiLock,
18  }
19  
20  struct Writer {
21    lock: BiLock,
22  }
23  
24  fn split() -> (Reader, Writer) {
25    let (a, b) = BiLock::new(0);
26    (Reader { lock: a }, Writer { lock: b })
27  }
29  fn main() {
30    let pool = LocalPool::new();
31    let mut exec = pool.executor();
32    let waker = Waker::from(Arc::new(FakeWaker));
33    let mut map = LocalMap::new();
34    let mut cx = Context::new(&mut map, &waker, &mut exec);
35  
36    let (reader, writer) = split();
37    println!("Lock should be ready for writer: {}",
38         writer.lock.poll_lock(&mut cx).is_ready());
39    println!("Lock should be ready for reader: {}",
40         reader.lock.poll_lock(&mut cx).is_ready());
41  
42    let mut writer_lock = match writer.lock.lock().poll(&mut 
      cx).unwrap() {
43      Async::Ready(t) => t,
44      _ => panic!("We should be able to lock with writer"),
45    };
46  
47    println!("Lock should now be pending for reader: {}",
48         reader.lock.poll_lock(&mut cx).is_pending());
49    *writer_lock = 123;
50  
51    let mut lock = reader.lock.lock();
52    match lock.poll(&mut cx).unwrap() {
53      Async::Ready(_) => {
54        panic!("The lock should not be lockable since writer has 
          already locked it!")
55      }
56      _ => println!("Couldn't lock with reader since writer has 
        already initiated the lock"),
57    };
58  
59    let writer = writer_lock.unlock();
60  
61    let reader_lock = match lock.poll(&mut cx).unwrap() {
62      Async::Ready(t) => t,
63      _ => panic!("We should be able to lock with reader"),
64    };
65  
66    println!("The new value for the lock is: {}", *reader_lock);
67  
68    let reader = reader_lock.unlock();
69    let reunited_value = reader.reunite(writer).unwrap();
70  
71    println!("After reuniting our locks, the final value is 
      still: {}",
72         reunited_value);
73  }
```

# 它是如何工作的...

+   首先，我们需要实现一个假的`futures::task::Waker`，以便在我们创建一个新的上下文时使用（这就是第 11 行至第 14 行的`FakeWaker`结构的作用）。

+   由于 BiLocks 需要两个所有者，我们将所有权分为两个不同的结构，称为`Reader<T>`（第 16 行至第 18 行）和`Writer<T>`（第 20 行至第 22 行）。

+   我们的`split() -> (Reader<u32>, Writer<u32>)`函数只是为了更好地结构化/组织我们的代码，并且在调用`BiLock::new(t: T)`时，返回类型是两个`futures_util::lock::BiLock`元素的元组。

现在初步代码已经解释完毕，让我们深入到我们的`main()`函数：

+   在第 30 行至第 34 行，我们设置了一个新的`LocalPool`、`LocalExecutor`、`Waker`（`FakeWaker`）和一个`LocalMap`（任务内的本地数据映射存储），用于创建一个新的`Context`，因为我们将会手动轮询锁以进行演示。

+   第 38 行和第 40 行使用了`futures_util::lock::BiLock::poll_lock`函数，如果锁可用，则返回一个`Async<futures_util::lock::BiLockGuard<T>>`值。如果锁不可用，则函数将返回`Async::Pending`。当引用被丢弃时，锁（`BiLockGuard<T>`）将解锁。

+   在第 42 行，我们执行`writer.lock.lock()`，这将阻塞锁并返回一个`BiLockAcquire<T>`，这是一个可以被轮询的未来。当轮询`BiLockAcquire`时，返回一个`Poll<BiLockAcquired<T>, ()>`值，并且这个值可以被可变地解引用。

+   在第 48 行，我们现在可以看到锁目前处于`Async::Pending`状态，这不会允许我们再次锁定 BiLock，正如第 51 行至第 57 行所示。

+   在修改我们的锁的值（第 49 行）之后，我们现在应该解锁它（第 59 行），以便其他所有者可以引用它（第 61 行至第 64 行）。

+   当我们调用`BiLockAcquired::unlock()`（第 68 行）时，返回原始的`BiLock<T>`，并且锁正式解锁。

+   在第 69 行，我们执行`futures_util::lock::BiLock::reunite(other: T)`，这恢复了锁的值并销毁了 BiLock 引用的*两个部分*（假设`T`是`BiLock::new()`调用中 BiLock 的另一部分）。
