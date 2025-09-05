# 微服务中的后台任务和线程池

在本章中，你将学习如何在微服务中使用后台任务。在第五章，*使用 Futures Crate 理解异步操作*中，我们创建了一个提供用户上传图像功能的微服务。现在，我们将创建这个服务的另一个版本，该版本加载一个图像并返回该图像的缩放版本。

为了充分利用可用资源，微服务必须使用异步代码实现，但微服务的每个部分并不都可以异步。例如，需要大量 CPU 负载的部分或必须使用共享资源的部分应该在一个单独的线程中实现，或者使用线程池来避免阻塞用于处理异步代码事件循环的主线程。

在本章中，我们将介绍以下主题：

+   如何与生成的线程交互

+   如何使用`futures-cpupool`和`tokio-threadpool`crate

# 技术要求

在本章中，我们将通过添加图像缩放功能来改进第五章，*使用 Futures Crate 理解异步操作*中的图像微服务。要编译示例，你需要 Rust 编译器，版本 1.31 或更高。

你可以从 GitHub 上的项目获取本章示例的源代码：[`github.com/PacktPublishing/Hands-On-Microservices-with-Rust/tree/master/Chapter10`](https://github.com/PacktPublishing/Hands-On-Microservices-with-Rust/tree/master/Chapter10)。

# 与线程交互

我们首先将使用一个单独的线程来实现这个功能，该线程将缩放所有传入的图像。之后，我们将使用线程池改进微服务。在本节中，我们将开始使用线程在后台执行任务。

# 同步还是异步？

在这本书中，我们更喜欢创建异步微服务，因为这些可以处理大量的并发请求。然而，并不是每个任务都可以通过异步方式处理。我们是否可以使用异步微服务取决于任务的类型和它需要的资源。让我们进一步探讨这种差异。

# I/O 密集型任务与 CPU 密集型任务

有两种类型的任务。如果一个任务不进行很多计算，但进行大量的输入/输出操作，那么它被称为 I/O 密集型任务。由于 CPU 的速度远快于输入/输出总线，我们不得不等待很长时间才能使总线或设备可用进行读写。I/O 密集型任务可以通过异步方式很好地处理。

如果一个任务使用 CPU 进行大量操作，那么它被称为 CPU 密集型任务。例如，图像缩放是一种 CPU 密集型任务，因为它从原始图像重新计算像素，但只有在准备好时才保存结果。

I/O 密集型任务和 CPU 密集型任务之间的区别并不明显，并不是每个任务都可以严格归类为 I/O 或 CPU 领域。为了调整图像大小，你必须将整个图像保持在内存中，但如果你提供的服务转码视频流，它可能需要大量的 I/O 和 CPU 资源。

# 异步上下文中的同步任务

假设你知道任务属于哪个类别，无论是 I/O 还是 CPU。IO 任务可以在单个线程中处理，因为它们必须等待大量的 I/O 数据。然而，如果你的硬件具有多核 CPU 和大量的 I/O 设备，那么单个线程就不够了。你可能会决定使用单个异步上下文中的多个线程，但有一个问题——并不是每个异步任务都可以在线程之间传递。例如，SQLite-嵌入式数据库将服务数据存储在线程局部存储中，你不能在多个线程中使用相同的数据库句柄。

SQLite 不能异步地与数据库一起工作；它有异步方法可以与在单独线程中运行的实例交互，但你必须记住，并不是每个任务都可以在多线程上下文中运行。

如果我们拥有多核硬件，一个好的解决方案是使用线程池来处理连接。你可以将连接上下文从池中的任何线程传递，这样就可以异步地处理连接。

Rust 和编写良好的 crate 可以防止你犯错；在我看来，Rust 是编写快速和安全的软件的最佳工具。然而，重要的是要意识到某些难以通过编译器检测到的特定情况，这发生在你在异步上下文中调用阻塞操作时。异步应用程序使用一个反应器，当数据准备好读取或写入时，它会调用必要的代码，但如果你已经调用了阻塞方法，反应器就无法被调用，并且所有由被阻塞的线程处理的连接都将被阻塞。更糟糕的是，如果你调用与反应器相关的同步方法，应用程序可能会完全阻塞。例如，如果你尝试向由反应器处理的接收器发送消息，但通道已满，程序将被阻塞，因为反应器必须被调用以排空通道，但由于线程已被发送的消息阻塞，这无法完成。看看以下示例：

```rs
fn do_send(tx: mpsc::Sender<Msg>) -> impl Future<Item = (), Error = ()> {
    future::lazy(|| {
      tx.send(Msg::Event).wait(); // The program will be blocked here
    })
}
```

结论很简单——只应在异步上下文中使用异步操作。

# 在文件上使用 IO 操作的限制

如前所述，一些库，例如 SQLite，使用阻塞操作来对数据库进行查询并获取结果，但这取决于它们使用的 I/O 类型。在现代操作系统中，网络堆栈是完全异步的，但文件的输入/输出更难异步使用。操作系统包含执行异步读取或写入的函数，但很难实现跨平台的兼容性。使用单独的线程来处理硬盘 I/O 交互会更简单。`tokio` 包使用单独的线程来处理文件的 I/O。其他平台，如 Go 或 Erlang，也做同样的事情。你可以为特定操作系统使用异步 I/O 来处理文件，但这不是一个非常灵活的方法。

现在你已经了解了同步和异步任务之间的区别，我们准备创建一个使用单独线程来处理调整图像大小的 CPU 密集型任务的异步服务。

# 为图像处理启动一个线程

在我们的第一个例子中，我们将创建一个微服务，该服务期望接收一个包含图像的请求，将其完全加载到内存中，将其发送到线程进行调整大小，并等待结果。让我们首先创建一个期望图像数据和响应的线程。为了接收请求，我们将使用 `mpsc::channel` 模块和 `oneshot::channel` 用于响应，因为多个客户端不能发送请求，我们只期望每个图像有一个响应。对于请求，我们将使用以下结构体：

```rs
struct WorkerRequest {
    buffer: Vec<u8>,
    width: u16,
    height: u16,
    tx: oneshot::Sender<WorkerResponse>,
}
```

`WorkerRequest` 包含用于二进制图像数据的 `buffer` 字段，所需调整大小的图像的 `width` 和 `height`，以及一个 `oneshot::Sender` 类型的 `tx` 发送器，用于发送 `WorkerReponse` 响应。

响应通过类型别名呈现为 `Result` 类型，该类型包含成功的结果，其中包含调整大小图像的二进制数据或错误：

```rs
type WorkerResponse = Result<Vec<u8>, Error>;
```

我们现在可以创建一个支持这些消息并执行调整大小的线程：

```rs
fn start_worker() -> mpsc::Sender<WorkerRequest> {
    let (tx, rx) = mpsc::channel::<WorkerRequest>(1);
    thread::spawn(move || {
        let requests = rx.wait();
        for req in requests {
            if let Ok(req) = req {
                let resp = convert(req.buffer, req.width, req.height).map_err(other);
                req.tx.send(resp).ok();
            }
        }
    });
    tx
}
```

由于我们为所有调整大小请求使用单个线程，我们可以使用 `Sender` 和 `Receiver` 的 `wait` 方法与客户端交互。前面的代码从 `mpsc` 模块创建了一个 `channel`，该 `channel` 可以在缓冲区中保持一条消息。我们不需要在缓冲区中为消息腾出更多空间，因为调整大小需要很长时间，而我们只需要在我们处理图像的同时将下一条消息发送给接收器。

我们使用`thread::spawn`方法来创建一个新的线程，并使用处理函数。`Receiver::wait`方法将`Receiver`转换为接收消息的阻塞迭代器。我们使用一个简单的循环来遍历所有请求。在这里不需要反应器。如果成功接收到消息，我们将处理请求。为了转换图像，我们使用以下代码片段中描述的`convert`方法。我们将结果发送到`oneshot::Sender`，它没有`wait`方法；我们只需要调用`send`方法，它返回一个`Result`。这个操作不会阻塞，也不需要反应器，因为它内部使用`UnsafeCell`来为实现了`Future`特质的`Receiver`提供一个值。

要调整图像大小，我们使用一个`image`包。这个包包含了一组丰富的图像转换方法，并支持多种图像格式。看看`convert`函数的实现：

```rs
fn convert(data: Vec<u8>, width: u16, height: u16) -> ImageResult<Vec<u8>> {
    let format = image::guess_format(&data)?;
    let img = image::load_from_memory(&data)?;
    let scaled = img.resize(width as u32, height as u32, FilterType::Lanczos3);
    let mut result = Vec::new();
    scaled.write_to(&mut result, format)?;
    Ok(result)
}
```

该函数期望接收图像的二进制数据，以及与其宽度和高度相关的数据。`convert`函数返回一个`ImageResult`，这是`Result`类型的一个别名，其错误类型为`ImageError`。我们使用这个错误类型，因为`convert`函数实现内部的一些方法可能会返回这种类型的错误。

实现的第一行尝试使用`guess_format`函数猜测传入数据的格式。我们可以在之后使用这个格式值来为输出图像使用相同的格式。之后，我们使用`load_from_memory`函数从数据向量中读取图像。这个调用读取数据并实际上将图像消耗的内存量加倍——如果你想要同时处理多个图像，请注意这一点。调整大小后，我们将缩放后的图像写入向量，并作为`Result`返回。缩放后的图像也消耗了一些内存，这意味着我们几乎将内存消耗量翻了两番。最好为传入消息的大小、宽度和高度添加限制，以防止内存溢出。

现在，我们可以实现`main`函数，它创建一个工作线程并启动服务器实例：

```rs
fn main() {
    let addr = ([127, 0, 0, 1], 8080).into();
    let builder = Server::bind(&addr);
    let tx = start_worker();
    let server = builder.serve(move || {
        let tx = tx.clone();
        service_fn(move |req| microservice_handler(tx.clone(), req))
    });
    let server = server.map_err(drop);
    hyper::rt::run(server);
}
```

与上一章的`main`函数相比，这里唯一的区别是我们调用`start_worker`函数，并使用返回的`Sender`作为处理函数的参数，同时附带一个请求。

让我们看看`microservice_handler`的实现，并了解它是如何与工作线程交互的。

# 以异步方式与线程交互

图像调整微服务的处理函数包含两个分支：一个用于索引页面，一个用于调整请求。看看以下代码：

```rs
fn microservice_handler(tx: mpsc::Sender<WorkerRequest>, req: Request<Body>)
    -> Box<Future<Item=Response<Body>, Error=Error> + Send>
{
    match (req.method(), req.uri().path().to_owned().as_ref()) {
        (&Method::GET, "/") => {
            Box::new(future::ok(Response::new(INDEX.into())))
        },
        (&Method::POST, "/resize") => {
            let (width, height) = {
                // Extracting parameters here
            };
            let body = ; // It's a Future for generating a body of the Response
            Box::new(body)
        },
        _ => {
            response_with_code(StatusCode::NOT_FOUND)
        },
    }
}
```

在处理函数的`resize`分支部分，我们必须执行各种操作：提取参数、从流中收集正文、向工作线程发送任务以及生成正文。由于我们使用异步代码，我们将创建一系列方法调用，以构建必要的`Future`对象。

要提取参数，我们使用以下代码：

```rs
let (width, height) = {
    let uri = req.uri().query().unwrap_or("");
    let query = queryst::parse(uri).unwrap_or(Value::Null);
    let w = to_number(&query["width"], 180);
    let h = to_number(&query["height"], 180);
    (w, h)
};
```

我们使用`uri`的`query`部分和`queryst` crate 的`parse`函数将参数解析为`Json::Value`。之后，我们可以通过索引提取必要的值，因为`Value`类型实现了`std::ops::Index`特质。通过索引获取值返回一个`Value`，如果值未设置，则返回`Value::Null`。`to_number`函数尝试将值表示为字符串并将其解析为`u16`值。或者，它返回一个默认值，这是作为第二个参数设置的：

```rs
fn to_number(value: &Value, default: u16) -> u16 {
    value.as_str()
        .and_then(|x| x.parse::<u16>().ok())
        .unwrap_or(default)
}
```

默认情况下，我们将使用 180 × 180 像素的图像大小。

处理分支的下一部分使用我们从查询字符串中提取的大小参数创建响应体。以下代码收集请求流到一个向量中，并使用工作者实例调整图像大小：

```rs
let body = req.into_body()
    .map_err(other)
    .concat2()
    .map(|chunk| chunk.to_vec())
    .and_then(move |buffer| {
        let (resp_tx, resp_rx) = oneshot::channel();
        let resp_rx = resp_rx.map_err(other);
        let request = WorkerRequest { buffer, width, height, tx: resp_tx };
        tx.send(request)
            .map_err(other)
            .and_then(move |_| resp_rx)
            .and_then(|x| x)
    })
    .map(|resp|  Response::new(resp.into()));
```

要与工作者交互，我们创建一个`oneshot::channel`实例和一个带有必要参数的`WorkerRequest`。之后，我们使用`tx`变量向工作者发送请求，`tx`是一个连接到工作者并提供了`microservice_handler`函数调用的`Sender`实例。`send`方法创建一个如果消息成功发送则成功的 future。我们使用`and_then`方法添加一个步骤到这个 future 中，该方法从实现了`Future`特质的`oneshot::Recevier`中读取一个值。

当缩放的消息准备好时，我们将其作为`Future`的结果，并将其`map`到响应。

使用`curl`发送图像来测试示例：

```rs
curl --request POST \
 --data-binary "@../../media/image.jpg" \
 --output "files/resized.jpg" \
 "http://localhost:8080/resize?width=100&height=100"
```

我们已从媒体文件夹发送了`image.jpg`并将其结果保存到`files/resized.jpg`文件中。

这个微服务的最大缺点是它只使用单个线程，这很快就会成为瓶颈。为了防止这种情况，我们可以使用多个线程来共享 CPU 资源以处理更多请求。现在让我们看看如何使用线程池。

# 使用线程池

要使用线程池，你不需要特殊的库。相反，你可以实现一个调度器，将请求发送到一组线程。你甚至可以检查工作者的响应来决定选择哪个线程进行处理，但有一些现成的 crate 可以帮助更优雅地解决这个问题。在本节中，我们将探讨`futures-cpupool`和`tokio-threadpool`crate。

# CpuPool

在这里，我们将重用现有的微服务并移除`start_worker`函数，以及`WorkerRequest`和`WorkerResult`类型。保留`convert`函数并添加一个新的依赖到`Cargo.toml`：

```rs
futures-cpupool = "0.1"
```

从该 crate 导入`CpuPool`类型：

```rs
use futures_cpupool::CpuPool;
```

现在这个池已经准备好在请求处理程序中使用。我们可以像在先前的例子中传递工作者线程的`Sender`一样传递它作为参数：

```rs
fn main() {
    let addr = ([127, 0, 0, 1], 8080).into();
    let pool = CpuPool::new(4);
    let builder = Server::bind(&addr);
    let server = builder.serve(move || {
        let pool = pool.clone();
        service_fn(move |req| microservice_handler(pool.clone(), req))
    });
    let server = server.map_err(drop);
    hyper::rt::run(server);
}
```

在前面的代码中，我们创建了一个包含四个线程的线程池，并将其传递给`serve`函数以`clone`它用于处理程序。处理程序函数将池作为第一个参数：

```rs
fn microservice_handler(pool: CpuPool, req: Request<Body>)
     -> Box<Future<Item=Response<Body>, Error=Error> + Send>
```

我们使用相同的分支和代码来提取宽度和高度参数。然而，我们改变了对图像的转换方式：

```rs
let body = req.into_body()
    .map_err(other)
    .concat2()
    .map(|chunk| chunk.to_vec())
    .and_then(move |buffer| {
        let task = future::lazy(move || convert(&buffer, width, height));
        pool.spawn(task).map_err(other)
    })
    .map(|resp| Response::new(resp.into()));
```

这个实现的代码变得更加紧凑和准确。在这个实现中，我们也将一个体收集到一个`Vec`二进制中，但是为了转换图像，我们使用一个在`CpuPool`中使用`spawn`方法产生的懒`Future`。

我们使用`future::lazy`调用来推迟`convert`函数的执行。如果没有`lazy`调用，`convert`函数将立即被调用并阻塞所有 IO 活动。你还可以使用`Builder`为`CpuPool`设置特定的参数。这有助于设置池中线程的数量、堆栈大小以及在新线程启动后和停止前将被调用的钩子。

`CpuPool`不是使用池的唯一方法。让我们看看另一个例子。

# 阻塞部分

`tokio-threadpool`存储库包含一个声明如下所示的`blocking`函数：

```rs
pub fn blocking<F, T>(f: F) -> Poll<T, BlockingError> where F: FnOnce() -> T 
```

这个函数期望任何执行阻塞操作的功能，并在一个单独的线程中运行它，提供一个可以由反应器使用的`Poll`结果。这是一个稍微低级的方法，但它被`tokio`和其他存储库（在文件上执行 IO 操作）积极使用。

这种方法的优点是我们不需要手动创建线程池。我们可以使用简单的`main`函数，就像我们之前做的那样：

```rs
fn main() {
    let addr = ([127, 0, 0, 1], 8080).into();
    let builder = Server::bind(&addr);
    let server = builder.serve(|| service_fn(|req| microservice_handler(req)));
    let server = server.map_err(drop);
    hyper::rt::run(server);
}
```

要启动一个调用`convert`函数的任务，我们可以使用以下代码：

```rs
let body = req.into_body()
    .map_err(other)
    .concat2()
    .map(|chunk| chunk.to_vec())
        .and_then(move |buffer| {
            future::poll_fn(move || {
                let buffer = &buffer;
                blocking(move || {
                    convert(buffer, width, height).unwrap()
                })
            })
            .map_err(other)
        })
        .map(|resp| Response::new(resp.into()));
```

`blocking`函数调用将任务执行委托给另一个线程，并在每次调用时返回一个`Poll`，直到执行结果准备好。要调用返回`Poll`结果的原始函数，我们可以使用`future::poll_fn`函数调用将该函数包装起来，将任何轮询函数转换为`Future`实例。看起来很简单，不是吗？我们甚至没有手动创建线程池。

例如，`tokio-fs`存储库使用这种方法在文件上实现 IO 操作：

```rs
impl Write for File {
    fn write(&mut self, buf: &[u8]) -> io::Result<usize> {
        ::would_block(|| self.std().write(buf))
    }
    fn flush(&mut self) -> io::Result<()> {
        ::would_block(|| self.std().flush())
    }
}
```

`would_block`是阻塞函数的一个包装器：

```rs
fn would_block<F, T>(f: F) -> io::Result<T>
where F: FnOnce() -> io::Result<T>,
{
    match tokio_threadpool::blocking(f) {
        Ok(Ready(Ok(v))) => Ok(v),
        Ok(Ready(Err(err))) => {
            debug_assert_ne!(err.kind(), WouldBlock);
            Err(err)
        }
        Ok(NotReady) => Err(WouldBlock.into()),
        Err(_) => Err(blocking_err()),
    }
}
```

现在，你知道任何阻塞操作都可以与异步反应器结合使用。这种方法不仅用于与文件系统交互，还用于数据库和其他不支持`futures`存储库或需要大量 CPU 计算的存储库。

# 演员

线程和线程池是利用服务器更多资源的好方法，但编程风格很繁琐。你必须考虑很多细节：发送和接收消息、负载分配以及重新启动失败的线程。

另一种并行运行任务的方法是演员。演员模型是一种计算模型，它使用称为**演员**的计算原语。它们并行工作并通过传递消息相互交互。与使用线程或池相比，这是一种更灵活的方法，因为你可以将每个复杂任务委托给一个单独的演员，该演员接收消息并将结果返回给任何向演员发送请求的实体。你的代码结构良好，你甚至可以重用演员来处理不同的项目。

我们已经学习了`futures`和`tokio`库，它们直接使用起来比较复杂，但它们是构建异步计算框架的良好基础，尤其是实现演员模型非常好。`actix`库已经实现了这一点：它基于这两个库将演员模型引入 Rust。让我们研究一下我们如何使用演员来执行后台任务。

我们将重新实现缩放微服务，但添加三个演员：一个缩放演员，一个计数演员，它计算请求数量，以及一个日志演员，它将计数值写入`syslog`。

# actix 框架的基础

`actix`库提供了一个组织良好的演员模型，使用起来很简单。有一些主要概念你应该记住。首先，`actix`库有`System`类型，这是维护演员系统的主要类型。在创建和启动任何演员之前，你必须创建`System`实例。实际上，`System`是一个控制整个过程的`Actor`，可以用来停止应用程序。

`Actor`是框架中最常用的特质。实现了`Actor`特质的每个类型都可以被启动。我们将在本章后面实现这个特质。此外，`actix`库还包含`Arbiter`类型，这是一个事件循环控制器，每个线程必须有一个。有`SyncArbiter`来运行 CPU 密集型任务，这个仲裁者使用线程池来执行演员。

每个`Actor`都必须在一个`Context`中工作，这是一个运行时环境，可以用来启动其他任务。此外，每个`Actor`实例都带有一个`Address`，你可以用它向演员发送消息并接收响应。在我们的示例中，我们将所有必要的演员的地址存储在共享状态中，以便从不同的处理程序并行使用它们。

`Address`提供了一个期望实现`Message`特质的类型的`send`方法。为了为`Actor`实现消息处理，你必须为演员的类型实现`Handler`特质。

让我们为我们的缩放微服务创建三个演员。

# 实现演员

首先，我们必须导入所有必要的依赖项。在本章中，我们将使用与之前示例相同的公共依赖项，但你还需要将以下依赖项添加到`Cargo.toml`中：

```rs
actix = "0.7"
failure = "0.1"
syslog = "4.0"
```

我们添加了`actix`crate。它是当前示例的主要 crate。此外，我们还导入了`failure`crate，因为我们将使用`Fail`特质来获取对`compat`方法的访问权限，该方法将实现`Fail`特质的任何错误类型转换为实现`std::error::Error`特质的`Compat`类型。

此外，我们将使用`syslog`，并添加了`syslog`crate 来访问系统 API。`syslog`是系统日志的标准。我们将用它来演示 actor 如何执行整个过程中的单独任务。现在，我们可以将`actors`模块添加到我们的示例中，并添加三个 actor。

# 计数 actor

我们要实现的第一个 actor 是一个计数器。它接收一个包含字符串的消息，并计算相同字符串的数量。我们将用它来计算指定路径的请求数量。

# 类型

创建`src/actors/count.rs`模块并导入以下类型：

```rs
use actix::{Actor, Context, Handler, Message};
use std::collections::HashMap;

type Value = u64;
```

我们将使用`Actor`特质来实现 actor 的行为，该行为在`Context`中工作，接收`Message`并通过`Handler`特质的实现来处理它。此外，我们需要`HashMap`来存储所有计数。我们还添加了`Value`类型别名，并将其用作计数的类型。

# `Actor`

`Actor`是一个实现了`Actor`特质的`struct`。我们将使用一个包含`HashMap`的`struct`来计数传入字符串的数量：

```rs
pub struct CountActor {
    counter: HashMap<String, Value>,
}

impl CountActor {
    pub fn new() -> Self {
        Self {
            counter: HashMap::new(),
        }
    }
}
```

我们添加了一个新方法来创建一个空的`CountActor`实例。

现在，我们可以为我们的`struct`实现`Actor`特质。实现很简单：

```rs
impl Actor for CountActor {
    type Context = Context<Self>;
}
```

我们指定一个上下文并将其设置为`Context`类型。actor 特质包含不同方法的默认实现，这些方法可以帮助你响应 actor 的生命周期事件：

+   `started`: 当`Actor`实例启动时调用此方法。

+   `stopping`: 当`Actor`实例切换到停止状态（例如，当调用`Context::stop`时）时调用此方法。

+   `stopped`: 当`Actor`实例停止时调用此方法。

现在，让我们添加一个 actor 将处理的消息类型。

# `Message`

计数 actor 期望一个包含字符串的消息，我们将添加以下`struct`：

```rs
pub struct Count(pub String);

impl Message for Count {
    type Result = Value;
}
```

`Count``struct`有一个字段，类型为`String`，并实现了 Actix 框架的`Message`特质。此实现允许我们使用 actor 的地址发送`Count`实例。

`Message`特质需要关联类型`Result`。在消息处理完毕后，将返回此值。我们将为提供的字符串返回一个计数器值。

# `Handler`

为了支持传入的消息类型，我们必须为我们的 actor 实现`Handler`特质。让我们为我们的`CountActor`实现`Count`消息的`Handler`：

```rs
impl Handler<Count> for CountActor {
    type Result = Value;

    fn handle(&mut self, Count(path): Count, _: &mut Context<Self>) -> Self::Result {
        let value = self.counter.entry(path).or_default();
        *value = *value + 1;
        *value
    }
}
```

我们还必须设置与结果相同类型的关联类型。

处理发生在`Handler`特质的`handle`方法体中。我们将使用`Count`消息获取提供的字符串条目，并提取`HashMap`中的条目。如果没有找到条目，我们将获取一个默认值，该值对于`u64`类型（`Value`别名）等于 0，并将`1`添加到该值中。

现在`ConnActor`已经准备好使用。我们可以实例化它，并使用演员的地址来计数 HTTP 请求的路径。让我们再添加两个演员。

# 日志演员

日志记录演员将向`syslog`添加记录。

# 类型

我们需要`actix`crate 的基本类型，并从`syslog`crate 导入一些类型：

```rs
use actix::{Actor, Context, Handler, Message};
use syslog::{Facility, Formatter3164, Logger, LoggerBackend};
```

我们不需要详细研究`syslog`crate，但让我们讨论一些基本类型。

你也可以使用`use actix::prelude::*;`来导入`actix`crate 中几乎所有常用的类型。

`Logger`是一个主要结构体，允许将方法写入`syslog`。它包括按级别从高到低记录日志的方法：`emerg`、`alert`、`crit`、`err`、`warning`、`notice`、`info`、`debug`。`LoggerBackend`枚举指定了连接到日志记录器的类型。它可以是套接字或 UNIX 套接字。`Facility`枚举指定了写入日志的应用程序类型。`Formatter3164`指定了日志记录的格式。

在两个 RFC 中描述了两种`Syslog`协议：`3164`和`5424`。这就是为什么格式化程序有如此奇怪的名字。

现在，我们可以实现日志记录演员。

# 演员

主要类型是`LogActor`结构体，它包含一个位于`writer`字段的`Logger`实例：

```rs
pub struct LogActor {
    writer: Logger<LoggerBackend, String, Formatter3164>,
}
```

我们将在`Handler`特质实现中使用这个记录器来写入消息，但现在我们需要为我们结构体提供一个构造函数，因为我们必须在启动时配置`Logger`：

```rs
impl LogActor {
     pub fn new() -> Self {
         let formatter = Formatter3164 {
             facility: Facility::LOG_USER,
             hostname: None,
             process: "rust-microservice".into(),
             pid: 0,
         };
         let writer = syslog::unix(formatter).unwrap();
         Self {
             writer,
         }
     }
 }
```

我们添加了一个新方法，该方法使用`Facility`值和进程名称填充`Formatter3164`结构体，并将其他字段设置为空白值。我们通过调用`syslog::unix`方法并提供一个格式化程序来创建一个`Logger`实例。我们将`Logger`存储在`writer`字段中，并返回一个`LogActor`结构体的实例。

要添加演员的行为，我们将为`LogActor`结构体实现`Actor`特质：

```rs
impl Actor for LogActor {
    type Context = Context<Self>;
}
```

由于这个演员将与服务器实例和计数演员在同一个线程中工作，我们将使用基本的`Context`类型。

# 消息

我们需要一个消息来发送消息以便将它们写入`syslog`。有一个简单的结构体，带有一个公共`String`字段就足够了：

```rs
pub struct Log(pub String);

impl Message for Log {
    type Result = ();
}
```

我们添加了`Log`结构体并为它实现了`Message`特质。对于这个消息，我们不需要返回值，因为日志记录将是一个单向过程，并且所有错误都将被忽略，因为它们对于微服务应用来说不是关键的。但是，如果你的微服务必须在一个严格的安全环境中工作，你也必须通知管理员有关日志记录的问题。

# 处理器

`Log`消息的`Handler`非常简单。我们使用提供的信息调用`Logger`的 info 方法，并通过将`Result`转换为`Option`来忽略错误：

```rs
impl Handler<Log> for LogActor {
     type Result = ();

     fn handle(&mut self, Log(mesg): Log, _: &mut Context<Self>) -> Self::Result {
         self.writer.info(mesg).ok();
     }
 }
```

我们必须实现的最后一个演员是调整大小的演员。

# 调整大小的演员

调整大小的演员调整传入的消息，并将调整大小后的消息返回给客户端。

# 类型

我们不需要任何特殊类型，将使用`actix`crate 的基本类型，并从之前使用的`image`crate 导入类型：

```rs
use actix::{Actor, Handler, Message, SyncContext};
use image::{ImageResult, FilterType};

type Buffer = Vec<u8>;
```

我们将在处理器的实现中将之前示例中的函数主体转换为函数，这就是为什么我们导入了`image`包中的类型。我们为`Vec<u8>`类型添加了`Buffer`别名以方便使用。

# 演员

我们需要一个没有字段的`struct`，因为我们将在`SyncArbiter`中使用它，它将在多个线程中运行多个演员。添加`ResizeActor`结构体：

```rs
pub struct ResizeActor;

impl Actor for ResizeActor {
    type Context = SyncContext<Self>;
}
```

我们不需要特殊的构造函数，并且我们实现了带有`SyncContext`类型的`Actor`特质，用于关联的`Context`类型。我们将使用此上下文类型使此演员适合`SyncArbiter`的同步环境：

# 消息

在本例中我们不使用转换函数，但我们需要相同的参数，我们将从`Resize`结构体中获取它们：

```rs
pub struct Resize {
    pub buffer: Buffer,
    pub width: u16,
    pub height: u16,
}

impl Message for Resize {
    type Result = ImageResult<Buffer>;
}
```

我们提供了一个包含图像数据的`buffer`以及所需的`width`和`height`。在`Resize`结构体的`Message`特质实现中，我们使用`ImageResult<Buffer>`类型。与`convert`函数返回的相同结果类型。我们将在 HTTP 处理器实现中稍后从演员获取此值。

# 处理器

我们为`ResizeActor`实现了`Resize`消息的`Handler`，但使用了传递的消息的字段来使用`convert`函数的主体：

```rs
impl Handler<Resize> for ResizeActor {
     type Result = ImageResult<Buffer>;

     fn handle(&mut self, data: Resize, _: &mut SyncContext<Self>) -> Self::Result {
         let format = image::guess_format(&data.buffer)?;
         let img = image::load_from_memory(&data.buffer)?;
         let scaled = img.resize(data.width as u32, data.height as u32, FilterType::Lanczos3);
         let mut result = Vec::new();
         scaled.write_to(&mut result, format)?;
         Ok(result)
     }
 }
```

我们还使用`SyncContext`而不是`Context`，就像我们之前为演员所做的那样。

所有演员都已准备就绪，您需要将所有模块添加到`src/actors/mod.rs`文件中：

```rs
pub mod count;
pub mod log;
pub mod resize;
```

现在，我们可以实现一个服务器，该服务器使用演员为每个请求执行调整大小和其他任务。

# 使用带有演员的服务器

导入服务器所需的全部必要类型。值得注意的是，只有那些你不熟悉的部分：

```rs
use actix::{Actor, Addr};
use actix::sync::SyncArbiter;
```

`Addr`是演员的地址。`SyncArbiter`是一个同步事件循环控制器，它以同步方式处理每条消息。我们需要它来处理调整大小的演员。还要添加`actors`模块并导入我们在子模块中声明的所有类型：

```rs
mod actors;

use self::actors::{
    count::{Count, CountActor},
    log::{Log, LogActor},
    resize::{Resize, ResizeActor},
};
```

我们需要一个共享状态来保存我们将用于处理请求的所有演员的地址：

```rs
#[derive(Clone)]
struct State {
    resize: Addr<ResizeActor>,
    count: Addr<CountActor>,
    log: Addr<LogActor>,
}
```

`Addr`类型是可克隆的，我们可以为我们的`State`结构体推导出`Clone`特质，因为我们必须为`hyper`的每个服务函数进行克隆。让我们实现一个带有新共享`State`的`main`函数：

```rs
fn main() {
    actix::run(|| {
        let resize = SyncArbiter::start(2, || ResizeActor);
        let count = CountActor::new().start();
        let log = LogActor::new().start();

        let state = State { resize, count, log };

        let addr = ([127, 0, 0, 1], 8080).into();
        let builder = Server::bind(&addr);
        let server = builder.serve(move || {
            let state = state.clone();
            service_fn(move |req| microservice_handler(&state, req))
        });
        server.map_err(drop)
    });
}
```

首先，我们必须启动事件循环。这是通过调用`actix::run`方法来完成的。我们传递一个闭包来准备所有演员，并返回一个`Future`来运行。我们将使用**`hyper`**的`Server`类型。

在闭包中，我们使用一个产生`ResizeActor`实例的函数来启动`SyncArbiter`。通过第一个参数，我们设置`SyncArbiter`将用于处理请求的线程数量。`start`方法返回一个仲裁器的地址，该仲裁器将消息路由到调整大小的演员。

要启动其他 actor，我们可以使用`Actor`特质的`start`方法，因为`actix::run`方法为我们创建了一个`System`实例和一个默认的`Arbiter`。我们就是这样创建了`CountActor`和`LogActor`的。`Actor`特质的`start`方法还返回 actor 的地址。我们将它们全部放入一个新的`State`结构体中。

之后，我们创建了一个`Server`实例，就像之前的例子一样，但还传递了一个克隆的`State`引用。

# 请求处理器

在我们实现 HTTP 请求处理器之前，让我们添加一个函数，该函数使用`State`向`CountActor`发送消息，并使用返回的值通过`LogActor`打印它。看看下面的函数：

```rs
fn count_up(state: &State, path: &str) -> impl Future<Item=(), Error=Error> {
    let path = path.to_string();
    let log = state.log.clone();
    state.count.send(Count(path.clone()))
        .and_then(move |value| {
            let message = format!("total requests for '{}' is {}", path, value);
            log.send(Log(message))
        })
        .map_err(|err| other(err.compat()))
}
```

我们将路径转换成了`String`类型，因为我们需要这种类型来发送`Count`消息，并将其移动到发送`Log`消息给`LogActor`的`Future`中。此外，我们还需要将`Addr`克隆到`LogActor`中，因为当计数器的值可用后，我们还需要在闭包中使用它。现在让我们创建一个`Future`，依次发送`Count`消息和`Log`消息。

`Addr`结构体有一个`send`方法，该方法返回一个实现了`Future`特质的`Request`实例。`Request`将在可用时返回一个计数器值。我们使用`Future`的`and_then`方法将额外的`Future`添加到链中。我们需要准备一个消息用于`syslog`，并使用克隆的`Addr`将其发送到`LogActor`。

我们还将错误转换为`io::Error`，但发送方法返回的`MaiboxError`是一个实现了`Fail`特质的错误类型，但没有实现标准库中的`Error`特质，因此我们必须使用`compat`方法将错误转换为`failure`crate 中实现的`Compat`类型。

我们将在两个路径`/`和`/resize`上使用`count_up`方法，看看`microservice_handler`的实现：

```rs
fn microservice_handler(state: &State, req: Request<Body>)
    -> Box<Future<Item=Response<Body>, Error=Error> + Send>
{
    match (req.method(), req.uri().path().to_owned().as_ref()) {
        (&Method::GET, "/") => {
            let fut = count_up(state, "/").map(|_| Response::new(INDEX.into()));
            Box::new(fut)
        },
        (&Method::POST, "/resize") => {
            let (width, height) = {
                let uri = req.uri().query().unwrap_or("");
                let query = queryst::parse(uri).unwrap_or(Value::Null);
                let w = to_number(&query["width"], 180);
                let h = to_number(&query["height"], 180);
                (w, h)
            };
            // Add an implementation here
            Box::new(fut)
        },
        _ => {
            response_with_code(StatusCode::NOT_FOUND)
        },
    }
}
```

在某些部分保持不变，但现在它将`State`的引用作为第一个参数。由于这个处理函数必须返回一个`Future`实现，我们可以使用`count_up`函数调用的返回值，但将值替换为`Response`。我们已经在根路径上做了这件事。让我们使用`ResizeActor`的`Addr`添加调整大小的功能。

要将图像缓冲区发送给 actor，我们必须使用`collect2`方法从`Request`的`Body`中收集它，就像我们之前做的那样：

```rs
let resize = state.resize.clone();
let body = req.into_body()
    .map_err(other)
    .concat2()
    .map(|chunk| {
        chunk.to_vec()
    })
    .and_then(move |buffer| {
        let msg = Resize {
            buffer,
            width,
            height,
        };
        resize.send(msg)
            .map_err(|err| other(err.compat()))
            .and_then(|x| x.map_err(other))
    })
    .map(|resp| {
        Response::new(resp.into())
    });
let fut = count_up(state, "/resize").and_then(move |_| body);
```

之后，我们创建了`Resize`消息，并使用该 actor 克隆的`Addr`将其发送给`ResizeActor`。我们将所有错误转换为`io::Error`。但是等等，我们还没有添加请求计数和日志记录。在创建用于调整图像大小的`Future`之前，使用`and_then`方法调用`count_up`函数，并将其放在前面。

就这些了！现在每个请求都会发送到`CountActor`，然后向`LogActor`发送一个信息消息，调整大小的请求也会连接所有数据并发送到`ResizeActor`进行调整。是时候测试它了。

# 构建和运行

使用 `cargo run` 子命令来构建和运行代码。当服务器启动时，使用 `curl` 命令发送带有图像的 `POST` 请求。你可以找到此前置命令的参数示例。

例如，我用浏览器请求了根路径五次，并发送了一次调整大小的请求。它将调整大小的消息存储到了 `files` 文件夹中。是的，它工作了！现在我们可以检查日志演员是否向 `syslog` 添加记录。使用以下命令来打印日志：

```rs
journalctl
```

你可以找到以下记录：

```rs
Jan 11 19:48:53 localhost.localdomain rust-microservice[21466]: total requests for '/' = 1
Jan 11 19:48:55 localhost.localdomain rust-microservice[21466]: total requests for '/' = 2
Jan 11 19:48:55 localhost.localdomain rust-microservice[21466]: total requests for '/' = 3
Jan 11 19:48:56 localhost.localdomain rust-microservice[21466]: total requests for '/' = 4
Jan 11 19:48:56 localhost.localdomain rust-microservice[21466]: total requests for '/' = 5
Jan 11 19:49:16 localhost.localdomain rust-microservice[21466]: total requests for '/resize' = 1
```

如你所见，我们对根路径有五个请求，对 `/resize` 路径有一个请求。

如果你没有 `jounrnalctl` 命令，你可以尝试使用 `less /var/log/syslog` 命令来打印日志。

此示例使用了演员来运行并发活动。实际上，只有 `ResizeActor` 使用了与 `SyncArbiter` 一起的单独线程。`CountActor` 和 `LogActor` 使用与 `hyper` 服务器相同的线程。但这没关系，因为演员本身不会占用很多 CPU。

# 摘要

在本章中，我们探讨了如何在微服务中使用线程池。我们研究了三种方法：使用普通线程、使用 `futures-cpupool` 库和使用 `tokio-threadpool` 库。我们使用来自 `futures` 库的通道与异步代码中的线程进行交互。特殊的库会自动完成所有交互；你所需要做的只是调用一个将在单独线程中执行的功能。

此外，我们熟悉了 `actix` 库和演员模型，这有助于将任务分割成由智能运行时管理的独立单元来运行。

在下一章中，我们将学习如何使用 Rust 与不同的数据库交互，包括 PostgreSQL、MySQL、Redis、MongoDB 和 DynamoDB。
