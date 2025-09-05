# 第十四章：微服务优化

优化是微服务开发过程中的一个重要部分。通过优化，你可以提高微服务的性能，并降低基础设施和硬件的成本。在本章中，我们将探讨基准测试，以及一些优化技术，例如缓存、使用结构体重用共享数据而不获取所有权，以及如何使用编译器的选项来优化微服务。本章将帮助你通过优化技术来提高你的微服务的性能。

在本章中，我们将涵盖以下主题：

+   性能测量工具

+   优化技术

# 技术要求

要运行本章中的示例，你需要一个 Rust 编译器（版本 1.31 及以上）来构建用于测试的示例，以及构建你需要来测量性能的工具。

本章的示例源代码可以在 GitHub 上找到：[`github.com/PacktPublishing/Hands-On-Microservices-with-Rust/tree/master/Chapter14`](https://github.com/PacktPublishing/Hands-On-Microservices-with-Rust/tree/master/Chapter14)。

# 性能测量工具

在你决定在微服务中优化某些内容之前，你必须测量其性能。你不应该一开始就编写最优和快速的微服务，因为并非每个微服务都需要良好的性能，如果你的微服务内部存在瓶颈，它将在高负载下遇到困难。

让我们探索一对基准测试工具。我们将探讨用 Rust 编写的工具，因为如果你需要测试具有极高负载的特殊情况，它们可以简单地用来构建你自己的测量工具。

# Welle

Welle 是流行的**Apache 基准测试工具**（**ab**）的替代品，该工具用于基准测试 HTTP 服务器。它可以生成一批请求到指定的 URL，并测量每个请求的响应时间。最后，它收集关于平均响应时间和失败请求数量的统计数据。

要安装此工具，请使用以下命令：

```rs
cargo install welle
```

使用此工具很简单：设置要测试的 URL 和要发送到服务器的请求数量：

```rs
welle --num-requests 10000 http://localhost:8080
```

默认情况下，该工具使用单个线程发送请求并等待响应以发送下一个请求。但你可以通过设置带有所需线程数的`--concurrent-requests`命令行参数来在更多线程之间分割请求。

当测量完成后，它将打印出类似以下报告：

```rs
Total Requests: 10000
Concurrency Count: 1
Total Completed Requests: 10000
Total Errored Requests: 0
Total 5XX Requests: 0

Total Time Taken: 6.170019816s
Avg Time Taken: 617.001µs
Total Time In Flight: 5.47647967s
Avg Time In Flight: 547.647µs

Percentage of the requests served within a certain time:
50%: 786.541µs
66%: 891.163µs
75%: 947.87µs
80%: 982.323µs
90%: 1.052751ms
95%: 1.107814ms
99%: 1.210104ms
100%: 2.676919ms
```

如果你想要使用除`GET`之外的 HTTP 方法，你可以使用`--method`命令行参数来设置它。

# Drill

Drill 更为复杂，允许你对微服务进行负载测试。它不仅发送请求批次，还使用测试脚本来生成一系列活动。它帮助你执行一个可以用来测量整个应用程序性能的负载测试。

要安装`drill`，请使用以下命令：

```rs
cargo install drill
```

安装完成后，你必须配置将要执行的负载测试。创建一个`benchmark.yml`文件并添加以下负载测试脚本：

```rs
---

threads: 4
base: 'http://localhost:8080'
iterations: 5
rampup: 2

plan:
  - name: Index Page
    request:
      url: /
```

要使用此脚本开始测试，请运行以下命令：

```rs
drill --benchmark benchmark.yml --stats
```

它将通过发送使用脚本中的规则构建的 HTTP 请求来对你的微服务进行负载测试，并打印出如下报告：

```rs
Threads 4
Iterations 5
Rampup 2
Base URL http://localhost:8080

Index Page                http://localhost:8080/ 200 OK 7ms
Index Page                http://localhost:8080/ 200 OK 8ms
...
Index Page                http://localhost:8080/ 200 OK 1ms

Concurrency Level 4
Time taken for tests 0.2 seconds
Total requests 20
Successful requests 20
Failed requests 0
Requests per second 126.01 [#/sec]
Median time per request 1ms
Average time per request 3ms
Sample standard deviation 3ms
```

这两个工具都适合测试你的微服务的性能。如果你想要优化指定的处理器，Welle 适合测量单个请求类型的性能。Drill 适合生成复杂的负载，以测量应用程序可以服务多少用户。

让我们看看一个例子，我们添加一些优化，并使用 Welle 测试差异。

# 测量和优化性能

在本节中，我们将测量一个使用两种选项编译的示例微服务的性能：没有优化，以及由编译器进行的优化。微服务将向客户端发送渲染后的索引页面。我们将使用 Welle 工具来测量这个微服务的性能，以查看我们是否可以改进它。

# 基本示例

让我们在基于`actix-web`crate 的新 crate 中创建一个微服务。

将以下依赖项添加到`Cargo.toml`：

```rs
[dependencies]
actix = "0.7"
actix-web = "0.7"
askama = "0.6"
chrono = "0.4"
env_logger = "0.5"
futures = "0.1"

[build-dependencies]
askama = "0.6"
```

我们将构建一个微型的服务器，该服务器将以一分钟精度异步渲染当前时间索引页面。事实上，有一个快捷函数可以做到这一点：

```rs
fn now() -> String {
    Utc::now().to_string()
}
```

我们使用`askama`crate 来渲染索引页面的模板，并从共享状态中插入当前时间。对于时间值，我们使用`String`而不是直接从`chrono`crate 中的类型，以便获得使用内存堆的值：

```rs
#[derive(Template)]
#[template(path = "index.html")]
struct IndexTemplate {
    time: String,
}
```

对于共享状态，我们将使用一个包含`last_minute`值的结构体，该值以`Mutex`包裹的`String`形式表示：

```rs
#[derive(Clone)]
struct State {
    last_minute: Arc<Mutex<String>>,
}
```

如你所记得，`Mutex`为多个线程提供了对任何类型的并发访问。它锁定用于读取和写入值。

对于索引页面，我们将使用以下处理器：

```rs
fn index(req: &HttpRequest<State>) -> HttpResponse {
    let last_minute = req.state().last_minute.lock().unwrap();
    let template = IndexTemplate { time: last_minute.to_owned() };
    let body = template.render().unwrap();
    HttpResponse::Ok().body(body)
}
```

上一段代码块中显示的处理程序锁定了一个在`HttpRequest`中可用的`State`实例的`last_minute`字段。然后我们使用这个值来填充`IndexTemplate`结构体并调用`render`方法来渲染模板，并将生成的值用作新的`HttpResponse`的正文。

我们使用`main`函数开始应用程序，该函数准备`Server`实例并启动一个单独的线程来更新共享的`State`：

```rs
fn main() {
    let sys = actix::System::new("fast-service");

    let value = now();
    let last_minute = Arc::new(Mutex::new(value));

    let last_minute_ref = last_minute.clone();
    thread::spawn(move || {
        loop {
            {
                let mut last_minute = last_minute_ref.lock().unwrap();
                *last_minute = now();
            }
            thread::sleep(Duration::from_secs(3));
        }
    });

    let state = State {
        last_minute,
    };
    server::new(move || {
        App::with_state(state.clone())
            .middleware(middleware::Logger::default())
            .resource("/", |r| r.f(index))
    })
    .bind("127.0.0.1:8080")
    .unwrap()
    .start();
    let _ = sys.run();
}
```

我们在这个示例中使用了 Actix Web 框架。你可以在第十一章中了解更多关于这个框架的信息，*使用 Actors 和 Actix Crate 处理并发*。这个简单的示例已经准备好编译和启动。我们还将使用 Welle 工具检查这段代码的性能。

# 性能

首先，我们将使用标准命令不带任何标志来构建和运行代码：

```rs
cargo run
```

我们构建了一个包含大量调试信息的二进制文件，它可以像我们在第十三章中做的那样，与 LLDB 一起使用，进行*测试和调试 Rust 微服务*。调试符号会降低性能，但我们将检查它，以便与没有这些符号的版本进行比较。

让我们用`100000`个请求从`10`个并发活动加载运行中的服务器。由于我们的服务器绑定到了`localhost`的`8080`端口，我们可以使用以下参数的`welle`命令来测量性能：

```rs
welle --concurrent-requests 10 --num-requests 100000 http://localhost:8080
```

这需要大约 30 秒（取决于你的系统）并且工具将打印报告：

```rs
Total Requests: 100000
Concurrency Count: 10
Total Completed Requests: 100000
Total Errored Requests: 0
Total 5XX Requests: 0

Total Time Taken: 29.883248121s
Avg Time Taken: 298.832µs
Total Time In Flight: 287.14008722s
Avg Time In Flight: 2.8714ms

Percentage of the requests served within a certain time:
50%: 3.347297ms
66%: 4.487828ms
75%: 5.456439ms
80%: 6.15643ms
90%: 8.40495ms
95%: 10.27307ms
99%: 14.99426ms
100%: 144.630208ms
```

在报告中，你可以看到平均响应时间为 300 毫秒。这是一个负担了调试的服务。让我们重新编译这个示例，并应用优化。在`cargo run`命令上设置`--release`标志：

```rs
cargo run --release
```

此命令将`-C opt-level=3`优化标志传递给`rustc`编译器。如果你使用不带`--release`标志的`cargo`，它将`opt-level`设置为`2`。

在服务器重新编译并启动后，我们再次使用带有相同参数的 Welle 工具。它报告了其他值：

```rs
Total Requests: 100000
Concurrency Count: 10
Total Completed Requests: 100000
Total Errored Requests: 0
Total 5XX Requests: 0

Total Time Taken: 8.010280915s
Avg Time Taken: 80.102µs
Total Time In Flight: 63.961189338s
Avg Time In Flight: 639.611µs

Percentage of the requests served within a certain time:
50%: 806.717µs
66%: 983.35µs
75%: 1.118933ms
80%: 1.215726ms
90%: 1.557405ms
95%: 1.972497ms
99%: 3.500056ms
100%: 37.844721ms
```

如我们所见，请求的平均处理时间已经降低到超过 70%。结果已经相当不错。但我们能否再降低一些？让我们尝试通过一些优化来实现这一点。

# 优化

我们将尝试将三种优化应用到我们在上一节中创建的代码中：

+   我们将尝试减少共享状态阻塞。

+   我们将通过引用在状态中重用值。

+   我们将添加响应的缓存。

在我们实现它们之后，我们将检查第一次两个改进后的性能，然后再次使用缓存进行检查。

在本节的代码中，我们将逐步实现所有优化，但如果你从本书的 GitHub 仓库中下载了示例，你将在本章项目的`Cargo.toml`文件中找到以下特性：

```rs
[features]
default = []
cache = []
rwlock = []
borrow = []
fast = ["cache", "rwlock", "borrow"]
```

这里的代码使用特性为你提供了一个单独激活或禁用任何优化的能力。我们看到以下内容：

+   `cache`：激活请求的缓存

+   `rwlock`：使用`RwLock`代替`Mutex`进行`State`

+   `borrow`：通过引用重用值

让我们实现所有这些优化，并将它们全部应用到性能差异的测量中。

# 无阻塞状态共享

在第一次优化中，我们将用`RwLock`替换`Mutex`，因为`Mutex`在读取和写入时都会锁定，但`RwLock`允许我们有一个单独的写入者或多个读取者。它允许我们在没有更新值的情况下避免读取值的阻塞。这适用于我们的示例，因为我们很少更新共享值，但必须从多个处理程序实例中读取它。

`RwLock`是`Mutex`的一个替代品，它将读取者和写入者分开，但`RwLock`的使用与`Mutex`一样简单。在`State`结构中将`Mutex`替换为`RwLock`：

```rs
#[derive(Clone)]
struct State {
    // last_minute: Arc<Mutex<String>>,
    last_minute: Arc<RwLock<String>>,
}
```

此外，我们必须将创建 `last_minute` 引用计数器的类型替换为相应的类型：

```rs
// let last_minute = Arc::new(Mutex::new(value));
let last_minute = Arc::new(RwLock::new(value));
```

在工作者的代码中，我们将使用 `RwLock` 的 `write` 方法锁定值以写入并设置新的时间值。它的独占锁将阻塞所有潜在的读取者和写入者，只有一个写入者可以更改值：

```rs
// let mut last_minute = last_minute_ref.lock().unwrap();
let mut last_minute = last_minute_ref.write().unwrap();
```

由于每个 `worker` 每三秒会获取一次独占锁，因此增加同时读取的数量是一个小的代价。

在处理器中，我们将使用 `read` 方法锁定 `RwLock` 以进行读取：

```rs
// let last_minute = req.state().last_minute.lock().unwrap();
let last_minute = req.state().last_minute.read().unwrap();
```

这段代码不会被其他处理器阻塞，除非 `worker` 更新值。它允许所有处理器同时工作。

现在我们可以实现第二个改进——避免值的克隆并通过引用使用它们。

# 通过引用重用值

为了渲染索引页面的模板，我们使用一个带有 `String` 字段的结构体，并且我们必须填充 `IndexTemplate` 结构体来调用其上的 `render` 方法。但是模板需要值的所有权，我们必须克隆它。克隆会消耗时间。为了避免这种 CPU 成本，我们可以使用值的引用，因为如果我们克隆一个使用内存堆的值，我们必须分配新的内存空间并将值的字节复制到新的位置。

这就是我们可以向一个值添加引用的方法：

```rs
struct IndexTemplate<'a> {
    // time: String,
    time: &'a str,
}
```

我们给一个结构体添加了 `'a` 生命周期，因为我们内部使用了一个引用，而这个结构体不能比我们引用的字符串值活得久。

对于 `Future` 实例的组合，使用引用并不总是可能的，因为我们必须构建一个生成 `HttpResponse` 的 `Future`，但它的生命周期比调用处理器要长。在这种情况下，如果你拥有它的所有权并使用如 fold 这样的方法将值传递到组合器链的所有步骤中，你可以重用这个值。对于可能消耗大量 CPU 时间的较大值来说，这是很有价值的。

现在我们可以使用对借用 `last_minute` 值的引用：

```rs
// let template = IndexTemplate { time: last_minute.to_owned() };
let template = IndexTemplate { time: &last_minute };
```

我们之前使用的 `to_owned` 方法克隆了我们放入 `IndexTemplate` 的值，但现在我们可以使用引用并完全避免克隆。

我们现在需要做的就是实现缓存，这有助于避免模板渲染。

# 缓存

我们将使用缓存来存储渲染的模板，并将其作为对后续请求的响应。理想情况下，缓存应该有一个生命周期，因为如果缓存没有更新，那么客户端将看不到页面上的任何更新。但为了我们的演示应用程序，我们不会重置缓存以确保它工作，因为我们的小型微服务渲染时间，我们可以看到它是否冻结。现在我们将向 `State` 结构体添加一个新字段以保留用于未来响应的渲染模板：

```rs
cached: Arc<RwLock<Option<String>>>
```

我们将使用`RwLock`，因为我们必须至少更新这个值一次，但对于那些不会更新且可以初始化的值，我们可以使用不带任何包装器的`String`类型来保护其免受并发访问，例如`RwLock`或`Mutex`。换句话说，如果你只读取它，可以直接使用`String`类型。

我们还必须使用`None`初始化值，因为我们需要渲染一次模板以获取缓存值。

向`State`实例添加一个空值：

```rs
let cached = Arc::new(RwLock::new(None));
let state = State {
    last_minute,
    cached,
};
```

现在我们可以使用一个`cached`值来构建对用户请求的快速响应。但必须考虑到并非所有信息都可以展示给每个用户。缓存可以根据一些关于用户的信息来分隔值，例如，它可以使用位置信息为来自同一国家的用户获取相同的缓存值。以下代码改进了`index`处理器并接受一个`cached`值，如果缓存值存在，则使用它来生成一个新的`HttpResponse`：

```rs
let cached = req.state().cached.read().unwrap();
if let Some(ref body) = *cached {
    return HttpResponse::Ok().body(body.to_owned());
}
```

我们立即返回一个`cached`值，因为已经存储了一个渲染的模板，我们不需要花费时间在渲染上。但如果不存在值，我们可以使用以下代码生成它，并设置`cached`值：

```rs
let mut cached = req.state().cached.write().unwrap();
*cached = Some(body.clone());
```

之后，我们将返回`HttpResponse`给客户端的原始代码保留下来。

现在我们已经实现了所有优化并编译了代码，可以测量优化后新版本的性能。

# 带优化编译

包含了三个优化。我们可以使用其中一些来检查性能差异。首先，我们将使用`RwLock`编译代码并借用状态值的特性。如果你使用书籍的 GitHub 仓库中的代码，你可以使用带有相应功能名称的`--features`参数运行必要的优化：

```rs
cargo run --release --features rwlock,borrow
```

一旦服务器准备就绪，我们可以使用 Welle 运行之前用来测量这个微服务性能的相同测试，以测量优化后的服务器版本可以处理多少个传入请求。

测试完成后，工具将打印出如下报告：

```rs
Total Requests: 100000
Concurrency Count: 10
Total Completed Requests: 100000
Total Errored Requests: 0
Total 5XX Requests: 0

Total Time Taken: 7.94342667s
Avg Time Taken: 79.434µs
Total Time In Flight: 64.120106299s
Avg Time In Flight: 641.201µs

Percentage of the requests served within a certain time:
50%: 791.554µs
66%: 976.074µs
75%: 1.120545ms
80%: 1.225029ms
90%: 1.585564ms
95%: 2.049917ms
99%: 3.749288ms
100%: 13.867011ms
```

如您所见，应用程序运行得更快——它只需要`79.434`微秒，而不是`80.10`微秒。差异不到 1%，但对于已经运行得很快的处理程序来说是个好成绩。

让我们尝试激活我们实现的全部优化，包括缓存。要使用 GitHub 上的示例这样做，请使用以下参数：

```rs
cargo run --release --features fast
```

服务器准备就绪后，让我们再次开始测试。使用相同的测试参数，我们得到了更好的报告：

```rs
Total Requests: 100000
Concurrency Count: 10
Total Completed Requests: 100000
Total Errored Requests: 0
Total 5XX Requests: 0

Total Time Taken: 7.820692644s
Avg Time Taken: 78.206µs
Total Time In Flight: 62.359549787s
Avg Time In Flight: 623.595µs

Percentage of the requests served within a certain time:
50%: 787.329µs
66%: 963.956µs
75%: 1.099572ms
80%: 1.199914ms
90%: 1.530326ms
95%: 1.939557ms
99%: 3.410659ms
100%: 10.272402ms
```

服务器对请求的响应时间为`78.206`微秒。这比没有优化的原始版本快了 2%以上，原始版本平均每个请求需要 80.10 微秒。

你可能认为差异不大，但事实上，差异很大。这是一个微小的例子，但试着想象一下优化一个处理程序，该处理程序向数据库发出三个请求，并渲染一个包含要插入的值数组的 200 KB 模板。对于重型处理程序，你可以通过 20%甚至更多的提升来提高性能。但请记住，你应该记住过度优化是一种极端措施，因为它使得代码更难开发，并添加更多功能而不影响已达到的性能。

最好不要将任何优化视为日常任务，因为你可能会花费大量时间优化短小的代码，以获得客户不需要的特性的 2%的性能提升。

# 优化技术

在上一节中，我们优化了源代码，但还有通过使用特殊的编译标志和第三方工具进行优化的替代技术。在本节中，我们将介绍一些这些优化技术。我们将简要讨论减少大小、基准测试和剖析 Rust 代码。

优化是一个富有创造性的主题。没有特殊的优化秘方，但在这个章节中，我们将创建一个小的微服务，该服务生成一个包含当前时间的索引页面，然后我们将尝试对其进行优化。通过这个例子，我希望向你展示一些适合你项目的优化思路。

# 链接时间优化

编译器在编译过程中自动执行许多优化，但我们也可以在源代码编译后激活一些优化。这种技术称为**链接时间优化**（**LTO**），在代码链接和整个程序可用后应用。你可以通过在你的项目中的`Cargo.toml`文件中添加一个额外的部分来为 Rust 程序激活此优化：

```rs
[profile.release]
lto = true
```

如果你已经用这个优化编译了项目，强制重新构建你的项目。但这个选项也需要更多的时间进行编译。这种优化并不一定能提高性能，但可以帮助减小二进制文件的大小。

如果你激活了所有优化选项，并不意味着你已经制作了应用最快的版本。过多的优化可能会降低程序的性能，你应该将结果与原始版本进行比较。通常，只使用标准的`--release`选项就能帮助编译器生成一个在编译速度和性能之间达到最佳平衡的二进制文件。

正常的 Rust 程序使用 panic 宏来处理未处理的错误并打印回溯。对于优化，你可以考虑将其关闭。让我们在下一节中看看这个技术。

# 放弃恐慌

错误处理代码也需要空间，并可能影响性能。如果你尝试编写不会恐慌、会尝试解决问题并且只有在遇到无法解决的问题时才会失败的微服务，你可以考虑使用 abort（立即终止程序而不回绕堆栈），而不是 Rust 的`panic`。

要激活它，请将以下内容添加到你的`Cargo.toml`文件中：

```rs
[profile.release]
panic = "abort"
```

现在，如果你的程序失败，它不会引发`panic`，并且会立即停止而不会打印回溯信息。

中断是危险的。如果你的程序被中断，它写日志或向分布式跟踪传递 span 的机会更小。对于微服务，你可以为跟踪创建一个单独的线程，即使主线程失败，也要等待所有可用的跟踪记录被存储。

有时候，你不仅需要提高性能，还必须减小二进制文件的大小。让我们看看如何做到这一点。

# 减小二进制文件的大小

你可能想要减小二进制文件的大小。通常这并不是必要的，但如果你的分布式应用程序使用了一些空间有限且需要小型二进制文件的硬件，那么这可能会很有用。要减小你编译的应用程序的大小，你可以使用`strip`命令，这是`binutils`包的一部分：

```rs
strip <path_to_your_binary>
```

例如，我尝试从本章*基本示例*部分创建的微服务的编译二进制文件中删除调试符号。使用`cargo build`命令编译的带有调试符号的二进制文件从 79 MB 减少到 12 MB。使用`--release`标志编译的版本从 8.5 MB 减少到 4.7 MB。

但请记住，你不能调试经过`strip`处理的二进制文件，因为工具会移除所有必要的调试信息。

有时候，你可能想要比较一些优化的想法，并测量哪一个更好。你可以使用基准测试来做到这一点。让我们看看`cargo`提供的基准测试功能。

# 独立的基准测试

Rust 支持开箱即用的基准测试。你可以用它来比较同一问题的不同解决方案的性能，或者了解应用程序某些部分的执行时间。

要使用基准测试，你必须添加一个带有`#[bench]`属性的函数。该函数期望有一个对`Bencher`实例的可变引用。例如，让我们比较克隆一个`String`与对其取引用：

```rs
#![feature(test)]
extern crate test;
use test::Bencher;

#[bench]
fn bench_clone(b: &mut Bencher) {
    let data = "data".to_string();
    b.iter(move || {
        let _data = data.clone();
    });
}

#[bench]
fn bench_ref(b: &mut Bencher) {
    let data = "data".to_string();
    b.iter(move || {
        let _data = &data;
    });
}
```

要进行基准测试，你必须向`Bencher`实例的`iter`方法提供一个你想要测量的代码闭包。你还需要在测试模块中添加带有`#![feature(test)]`的`test`功能，并使用`extern crate test`来导入`test`包，从而从这个模块中导入`Bencher`类型。

`bench_clone`函数有一个`String`值，并且由`Bencher`在每次测量时克隆它。在`bench_ref`中，我们取一个`String`值的引用。

现在，你可以使用`cargo`启动基准测试：

```rs
cargo bench
```

它会为测试编译代码（带有`#[cfg(test)]`属性的代码项将被激活），然后运行基准测试。对于我们的示例，我们得到了以下结果：

```rs
running 2 tests
test bench_clone ... bench:          32 ns/iter (+/- 9)
test bench_ref   ... bench:           0 ns/iter (+/- 0)

test result: ok. 0 passed; 0 failed; 0 ignored; 2 measured; 0 filtered out
```

如我们所预期，取`String`的引用不耗时，但`String`的克隆每次调用`clone`方法需要`32`纳秒。

记住，您可以对 CPU 密集型任务进行良好的基准测试，但不能对 I/O 密集型任务进行基准测试，因为 I/O 任务更多地依赖于硬件质量和操作系统性能。

如果您想基准测试运行应用程序中某些函数的操作，那么您必须使用分析器。让我们尝试使用分析器分析一些代码。

# 分析

基准测试对于检查代码的一部分很有用，但它们不适合检查运行应用程序的性能。如果您需要探索代码中某些函数的性能，您必须使用分析器。

分析器会导出有关代码和函数执行的信息，并记录代码工作的时间跨度。Rust 生态系统中有一种名为 **flame** 的分析器。让我们探索如何使用它。

分析需要时间，您应该将其作为一个功能来避免在生产安装中影响性能。将 `flame` 包添加到您的项目中，并作为可选使用。添加一个功能（例如来自 `flamer` 包仓库的官方示例；我将其命名为 `flame_it`），并将 `flame` 依赖项添加到其中：

```rs
[dependencies]
flame = { version = "0.2", optional = true }

[features]
default = []
flame_it = ["flame"]
```

现在，如果您想激活分析，您必须使用 `flame_it` 功能编译项目。

使用 `flame` 包非常简单，包括三个场景：

+   直接使用 `start` 和 `end` 方法。

+   使用 `start_guard` 方法，它创建一个用于测量执行时间的 `Span`。当 `Span` 实例被丢弃时，自动结束测量。

+   使用 `span_of` 来测量在闭包中隔离的代码。

我们将像在 第十三章 的 `OpenTracing` 示例中所做的那样使用跨度：*测试和调试 Rust 微服务*：

```rs
use std::fs::File;

pub fn main() {
    {
        let _req_span = flame::start_guard("incoming request");
        {
            let _db_span = flame::start_guard("database query");
            let _resp_span = flame::start_guard("generating response");
        }
    }

    flame::dump_html(&mut File::create("out.html").unwrap()).unwrap();
    flame::dump_json(&mut File::create("out.json").unwrap()).unwrap();
    flame::dump_stdout();
}
```

您不需要收集跨度或将它们发送到 `Receiver`，就像我们为 Jaeger 所做的那样，但使用 `flame` 进行分析看起来像是跟踪。

在执行结束时，您必须以适当的格式导出报告，例如 HTML 或 JSON，将其打印到控制台，或写入 `Writer` 实例。我们使用了前三个。我们实现了主函数，并使用 `start_quard` 方法创建 `Span` 实例来测量一些代码片段的执行时间。之后，我们将编写报告。

使用已激活的分析功能的示例编译并运行此示例：

```rs
cargo run --features flame_it
```

前面的命令编译并打印报告到控制台：

```rs
THREAD: 140431102022912
| incoming request: 0.033606ms
  | database query: 0.016583ms
    | generating response: 0.008326ms
    + 0.008257ms
  + 0.017023ms
```

如您所见，我们创建了三个跨度。您也可以在文件中找到两个报告，`out.json` 和 `out.html`。如果您在浏览器中打开 HTML 报告，它将呈现如下：

![图片](img/1ec7f264-3641-4908-80e4-9f0bc1cf6c18.png)

在前面的屏幕截图中，您可以看到我们程序每个活动的相对执行时间。颜色较深的块表示执行时间较长。如您所见，分析对于找到可以优化其他技术的慢速代码部分很有用。

# 摘要

在本章中，我们讨论了优化。首先，我们探索了用于测量性能的工具——Welle，它是经典**Apache 基准测试工具**的替代品，以及 Drill，它使用脚本执行负载测试。

然后我们创建了一个微小的微服务并测量了其性能。专注于结果，我们对那个微服务进行了一些优化——我们避免了在读取共享状态时阻塞，我们通过引用重用了一个值而不是克隆它，我们还增加了渲染模板的缓存。然后我们测量了优化后的微服务的性能，并将其与原始版本进行了比较。

在本章的最后部分，我们了解了优化的一些替代技术——使用 LTO，在无需回溯的情况下终止执行而不是恐慌，减小编译二进制文件的大小，对代码的小片段进行基准测试，以及使用性能分析为你的项目服务。

在下一章中，我们将探讨使用 Docker 创建带有 Rust 微服务的镜像，以在预配置的环境中运行微服务，从而加快产品交付给客户的速度。
