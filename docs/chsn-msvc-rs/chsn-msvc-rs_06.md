# 反应式微服务 - 提高容量和性能

如果您为您的应用程序采用微服务架构，您将获得松耦合的好处，这意味着每个微服务都足够独立，可以由不同的团队开发和维护。这是一种对业务任务的异步方法，但并非唯一的好处；还有其他好处。您可以通过仅扩展承担巨大负载的微服务来提高容量和性能。为了实现这一点，您的微服务必须是反应式的，并且必须是自维持的，通过消息传递与其他微服务进行交互。

在本章中，您将了解什么是反应式微服务，以及如何使用消息传递来确保微服务的连通性。此外，我们还将讨论反应式微服务是否可以是异步的。

在本章中，我们将涵盖以下主题：

+   什么是反应式微服务？

+   JSON-RPC

+   gRPC

# 技术要求

本章将介绍在 Rust 中使用**远程过程调用（RPCs**）。您需要一个工作的 Rust 编译器，因为我们将会使用`jsonrpc-http-server`和`grpc`包创建两个示例。

如果您想测试 TLS 连接，您需要 OpenSSL 版本 0.9，因为`grpc`包目前还不支持 1.0 或更高版本。大多数现代操作系统已经切换到 1.0，但您可以将示例构建到支持版本 0.9 的 Docker 镜像中，或者等待`grpc`包更新到最新的 OpenSSL 版本。我们将构建不带 TLS 的测试示例。

您可以从本章的 GitHub 仓库中找到示例的来源，网址为[`github.com/PacktPublishing/Hands-On-Microservices-with-Rust/tree/master/Chapter06`](https://github.com/PacktPublishing/Hands-On-Microservices-with-Rust/tree/master/Chapter06)。

# 什么是反应式微服务？

微服务架构意味着应用程序中存在多个部分。在过去，大多数应用程序都是单体，所有部分都包含在单个代码库中。微服务方法为我们提供了将代码库分割到多个团队和开发者的机会，为每个微服务提供一个单独的生命周期，以及部分通过一个通用协议进行交互。

这是否意味着您的应用程序将完全摆脱单体应用程序的所有缺陷？不。您可以编写相互之间如此紧密相关的微服务，以至于您甚至无法正确地更新它们。

这是如何实现的？想象一下，您有一个微服务需要等待另一个微服务的响应才能向客户端发送响应。反过来，另一个微服务也必须等待另一个微服务的响应，依此类推。如果您将应用程序的微服务紧密关联起来，您会发现与单体相同的缺点。

你必须编写微服务作为可以用于多个项目（而不仅仅是你的项目）的独立应用程序。你如何实现这一点？开发一个反应式微服务。让我们看看这是什么。

# 松散耦合

松散耦合是一种软件开发方法，意味着应用程序的每个部分应该对其他部分了解很少的信息。在传统应用程序中，如果你编写一个 GUI 组件，例如一个按钮，它必须可以在任何地方、任何应用程序中使用。作为另一个例子，如果你开发一个用于处理声音硬件的库，这也意味着你可以与每个应用程序一起使用；该库不限于在一种应用程序中使用。然而，在微服务中，松散耦合意味着一个微服务不知道其他微服务或它们的数量。

如果你开发一个应用程序，你应该将其部分——微服务——编写为独立的应用程序。例如，一个发送推送通知到移动平台的通告服务不会知道 CRM、会计或甚至用户。这是否可能？是的！你可以编写一个使用通用协议并通过消息传递与其他服务交互的微服务。这被称为**消息驱动应用程序**。

# 消息驱动应用程序

传统的微服务有一个立即返回结果的 API，并且一个工作应用程序的每个参与者都必须了解其他部分。这种方法使微服务之间的关系保持紧密。这样的应用程序难以维护、更新和扩展。

如果你的应用程序通过由其他微服务处理的消息进行交互，那就更加方便了。当你使用消息作为交互单元时，这种方法被称为**消息驱动**。消息帮助你减少微服务的耦合，因为你可以同时为多个服务处理一条消息，或者为特定消息类型添加额外的处理消息。

要有完全解耦的微服务，你应该使用消息队列或消息代理服务。我们将在第十二章，[可扩展微服务架构](https://cdp.packtpub.com/hands_on_microservices_with_rust_2018/wp-admin/post.php?post=405&action=edit)中详细了解这种方法，其中我们讨论了可扩展架构。

# 异步

由于反应式微服务使用消息传递，你必须异步处理消息。这并不意味着你必须使用异步 crate，如`tokio`或`futures`。但这意味着没有任何一条消息可以阻塞服务；每条消息都在短时间内被处理，如果服务必须执行长时间的任务，它应该将其作为后台任务执行，并通过发送包含该结果的消息来通知执行该任务的线程。为了实现这种行为，你可以使用多个线程而不需要异步代码。但对于反应式微服务来说，使用异步代码又是怎样的情况呢？

# 反应式微服务应该是异步的吗？

很常见的情况是，由于异步应用程序被错误地称为**反应式**，因为它们的事件循环等待外部事件，在等待时不会浪费服务器资源。反应式微服务不会为了保持传入连接以返回结果而浪费资源，因为当客户端短暂连接到反应式微服务时，它会提交任务并断开连接。在客户端等待微服务的异步响应后。反应式微服务不需要是异步的，但它们可以是。

当你选择消息传递进行交互时，你必须考虑到微服务必须是异步的，并且可以同时处理多个消息。传统的同步代码不能像异步代码那样处理那么多消息，因为同步代码必须等待 I/O 资源，而异步代码则对 I/O 事件做出反应，并尽可能利用资源。

更简单地说，如果你的微服务必须处理数十万条消息，你应该使用异步代码。如果你的微服务没有重负载，使用一个适度的同步算法就足够了。

# 基于未来和流的反应式微服务

如果你已经决定使用异步代码实现反应式微服务，你可以考虑使用未来库作为基础。这个库为你提供了构建反应式链的类型，以异步方式处理所有消息。但请记住，手动编写所有`Future`实例可能很困难。Rust 编译器即将推出一个新功能，提供了`async`/`await`运算符，通过编写具有`Result`返回类型的传统函数来简化`Future`特质的实现。这个特性是不稳定的，我们不会在本书中考虑它。

如果你不想编写低级代码，我推荐你使用`actix`库。这是一个非常好的框架，让你可以像编写同步代码一样编写异步代码。

如果你需要最低级别的控制，你可以使用作为`futures`库基础的`mio`库。它为你提供了对 I/O 事件的完全控制，你可以从服务器的资源中榨取最大速度。

# 消息代理

消息代理让你可以将所有消息发送到中央点，该点路由和将消息传递到必要的微服务。在某些情况下，这可能会成为瓶颈，因为全部负载将落在单个应用程序——消息代理上。但在大多数情况下，这是一个很好的方法，可以帮助你解耦微服务，并几乎无感知地更新任何微服务。

要使用消息代理，只需要支持 AMQP 协议即可。所有流行的消息代理都与该协议兼容。`lapin-futures`库提供了类型和方法，通过`futures`库的 API 使用 AMQP 协议。如果你想要使用`mio`库的低级控制，可以使用`lapin-async`库。

# 远程过程调用

如果你想要直接连接微服务，你可以使用 RPC 来允许一个服务的功能被另一个服务远程调用。有许多 RPC 框架，它们具有不同的格式和速度潜力。让我们看看一些流行的协议。

# JSON-RPC

JSON-RPC 协议使用序列化为 JSON 格式的消息。它使用一种特殊的请求和响应格式，这里进行了描述：[`www.JSON-RPC.org/specification`](https://www.jsonrpc.org/specification)。该协议可以使用不同的传输方式，例如 HTTP、Unix 套接字，甚至是 stdio。在本章的后面，你会找到一个使用此协议的示例。

# gRPC

gRPC 协议是由 Google 创建的，它使用 Protocol Buffer 序列化格式来处理消息。此外，该协议基于`HTTP/2`传输的优点，允许你实现卓越的性能。你可以在[`grpc.io/docs/`](https://grpc.io/docs/)找到更多关于该协议的信息。在本章的后面，你还会找到一个使用此协议的示例。

# Thrift

Apache Thrift 是由 Facebook 开发的一种二进制通信协议。尽管该协议是二进制的，但它支持许多语言，如 C++、Go 和 Java。支持的传输方式包括文件、内存和套接字。

# 其他 RPC 框架

还有其他 RPC 协议，如 Cap'n Proto、XML-RPC，甚至是古老的 SOAP。有些为 Rust 提供了实现，但我建议在 JSON-RPC、gRPC 和 Thrift 之间进行选择，因为它们是最常用于微服务的。

# RPC 和 REST

你可能会问，是否可以使用 REST API 或传统的 Web API 来实现反应式微服务。当然——可以！你可以有两种方式来做这件事：

+   有网关可以将 REST 请求转换为 JSON-RPC 或其他协议。例如，gRPC 有一个现成的：[`github.com/grpc-ecosystem/grpc-gateway`](https://github.com/grpc-ecosystem/grpc-gateway)。你甚至可以编写自己的网关——对于简单或特定的情况来说并不难。

+   你可以使用 Web API 从一个服务器向另一个服务器发送消息。一个微服务不必只有一个 API 路径，但你可以为 JSON 或其他格式的消息添加一个特殊处理程序。对于传输，你不仅可以使用 HTTP，还可以使用 WebSocket 协议。

# 反应式宣言

如果你将反应式架构视为一种标准化方法，你不会找到一个指南或规则来指导如何让你的微服务变得反应式，但有一个《反应式宣言》，你可以在这里找到：[`www.reactivemanifesto.org/`](https://www.reactivemanifesto.org/)。它包含了一系列原则，你可以使用这些原则来启发你改进应用程序的想法。

现在我们可以创建一个针对 JSON-RPC 协议的反应式微服务的示例。

# 理解 JSON-RPC

有一些 crate 提供了支持 JSON-RPC 协议的功能。大多数情况下，crate 只支持服务器或客户端的一侧，而不是两者都支持。有些 crate 甚至不支持异步计算。

# JSON-RPC 的工作原理

JSON-RPC 协议使用以下格式的 JSON 消息来请求：

```rs
{"jsonrpc": "2.0", "method": "substring", "params": [2, 6, \"string\"], "id": 1}
```

前面的 JSON 消息调用了一个可以返回如下结果的服务器的`substring`远程方法：

```rs
{"jsonrpc": "2.0", "result": "ring", "id": 1}
```

值得注意的是，客户端确定请求的标识符并必须跟踪这些值。服务器对 ID 不敏感，它们使用连接来跟踪请求。

该协议有两个版本——1.0 和 2.0。它们很相似，但在第二个版本中，客户端和服务器是分离的。此外，它是传输无关的，因为第一个版本使用连接事件来确定行为。错误和参数也有改进。你应该为新项目使用版本 2.0。

为了支持 JSON-RPC，你的服务器必须响应这类 JSON 请求。该协议实现起来非常简单，但我们将使用`jsonrpc-http-server` crate，它使用 HTTP 传输并提供启动服务器所需的类型。

# 创建微服务

在本节中，我们将创建一个支持 JSON-RPC 协议并具有两个方法的微服务示例。该微服务将支持作为微服务环的一部分工作。我们将向一个微服务发送消息，该微服务将向环形中的下一个微服务发送消息，然后该微服务将消息进一步传递。

我们将创建一个环形示例，因为如果实现不正确，你的微服务将被阻塞，因为它们不能像反应式服务那样并行处理请求。

# 依赖项

首先，我们需要导入必要的依赖项：

```rs
failure = "0.1"
JSON-RPC = { git = "https://github.com/apoelstra/rust-JSON-RPC" }
jsonrpc-http-server = { git = "https://github.com/paritytech/JSON-RPC" }
log = "0.4"
env_logger = "0.6"
serde = "1.0"
serde_derive = "1.0"
```

很可能，你对大多数 crate 都很熟悉，除了`jsonrpc`和`json-rpc-server`。第一个是基于`hyper` crate 的 JSON-RPC 客户端。第二个也使用了`hyper` crate，并提供了 JSON-RPC 的服务器功能。

让我们导入必要的类型，并简要介绍一下它们：

```rs
use failure::Error;
use JSON-RPC::client::Client;
use JSON-RPC::error::Error as ClientError;
use JSON-RPC_http_server::ServerBuilder;
use JSON-RPC_http_server::JSON-RPC_core::{IoHandler, Error as ServerError, Value};
use log::{debug, error, trace};
use serde::Deserialize;
use std::env;
use std::fmt;
use std::net::SocketAddr;
use std::sync::Mutex;
use std::sync::mpsc::{channel, Sender};
use std::thread;
```

JSON-RPC crate 有一个`Client`类型，我们将使用它来调用其他服务的远程方法。我们还从该 crate 中导入`Error`作为`ClientError`以避免与来自`failure` crate 的`Error`名称冲突。

对于服务器端，我们将使用`jsonrpc-http-server` crate 中的`ServerBuilder`。此外，我们需要将`Error`从该 crate 重命名为`ServerError`。为了实现函数处理器，我们需要导入`IoHandler`，它可以用来将函数附加为 RPC 方法。此外，我们还需要一个`Value`（实际上，这个类型是从`serde_json` crate 重新导入的），它用作 RPC 方法的返回类型。

为了避免方法名称的错误，因为我们将在服务器实现中使用它们两次，然后在客户端中再次使用，我们将名称声明为字符串常量：

```rs
const START_ROLL_CALL: &str = "start_roll_call";
const MARK_ITSELF: &str = "mark_itself";
```

第一个方法将从微服务开始发送消息到下一个微服务。第二个方法用于停止这个点名过程。

# 客户端

为了与其他微服务实例交互并调用它们的远程方法，我们将创建一个单独的结构体，因为它比直接使用 JSON-RPC 的`Client`更方便。但在任何情况下，我们都在我们的结构体内部使用此类型：

```rs
struct Remote {
    client: Client,
}
```

我们将使用`Remote`结构体来调用远程服务。为了创建这个结构体，我们将使用以下构造函数：

```rs
impl Remote {
    fn new(addr: SocketAddr) -> Self {
        let url = format!("http://{}", addr);
        let client = Client::new(url, None, None);
        Self {
            client
        }
    }
}
```

`Client`结构体期望一个`String` URL 作为参数，但我们将使用`SocketAddr`来创建 URL。

此外，我们还需要一个泛型函数，它将使用`Client`实例来调用远程方法。向`Remote`结构体的实现中添加`call_method`方法：

```rs
fn call_method<T>(&self, meth: &str, args: &[Value]) -> Result<T, ClientError>
where
    T: for <'de> Deserialize<'de>,
{
    let request = self.client.build_request(meth, args);
    self.client.send_request(&request).and_then(|res| res.into_result::<T>())
}
```

使用 JSON-RPC crate 调用 JSON-RPC 方法很简单。使用`Client`实例的`build_request`方法创建一个`Request`，然后使用同一`Client`的`send_request`方法发送它。有一个名为`do_rpc`的方法可以在单个调用中完成这个操作。我们将使用更详细的方法来展示你可以预先定义请求并使用它们来加速调用准备。此外，使用面向业务的 struct 方法而不是原始的`Client`更令人愉快。我们使用包装器隔离实现，以隐藏 RPC 调用的细节。如果你决定改为其他协议，比如 gRPC，会怎样呢？

向`Remote`结构体的实现中添加特殊方法，使用`call_method`方法进行调用。首先，我们需要`start_roll_call`函数：

```rs
fn start_roll_call(&self) -> Result<bool, ClientError> {
    self.call_method(START_ROLL_CALL, &[])
}
```

调用时不会传递任何参数，但它期望结果是`bool`类型。我们使用一个常量作为远程方法的名字。

向`Remote`结构体添加`mark_itself`方法：

```rs
fn mark_itself(&self) -> Result<bool, ClientError> {
    self.call_method("mark_itself", &[])
}
```

它也不发送任何参数，并返回`bool`值。

现在，我们可以添加一个工作者来将传入调用与传出调用分开。

# 工作者

由于我们有两种方法，我们将添加一个结构体以从工作线程执行这些方法的远程调用。向代码中添加`Action`枚举：

```rs
enum Action {
    StartRollCall,
    MarkItself,
}
```

它有两个变体：`StartRollCall`用于执行远程`start_roll_call`方法调用，以及`MarkItself`变体用于调用远程`mark_itself`方法。

现在，我们可以添加一个函数在单独的线程中创建一个工作者。如果我们将在传入方法处理程序中立即执行传出调用，我们可以阻塞执行，因为我们有一个微服务环，阻塞一个微服务将阻塞整个环的交互。

无阻塞是反应式微服务的一个重要属性。微服务必须并行或异步处理所有调用，但永远不应该长时间阻塞执行。它们应该像我们在讨论的演员模型中的演员一样工作。

看一下`spawn_worker`函数：

```rs
fn spawn_worker() -> Result<Sender<Action>, Error> {
    let (tx, rx) = channel();
    let next: SocketAddr = env::var("NEXT")?.parse()?;
    thread::spawn(move || {
        let remote = Remote::new(next);
        let mut in_roll_call = false;
        for action in rx.iter() {
            match action {
                Action::StartRollCall => {
                    if !in_roll_call {
                        if remote.start_roll_call().is_ok() {
                            debug!("ON");
                            in_roll_call = true;
                        }
                    } else {
                        if remote.mark_itself().is_ok() {
                            debug!("OFF");
                            in_roll_call = false;
                        }
                    }
                }
                Action::MarkItself => {
                    if in_roll_call {
                        if remote.mark_itself().is_ok() {
                            debug!("OFF");
                            in_roll_call = false;
                        }
                    } else {
                        debug!("SKIP");
                    }
                }
            }
        }
    });
    Ok(tx)
}
```

此函数创建一个通道并使用从 `NEXT` 环境变量中提取的地址启动一个新的线程，该线程使用一个例程处理从通道接收到的所有消息。我们使用从 `NEXT` 环境变量中提取的地址创建 `Remote` 实例。

有一个标志表示已调用 `start_roll_call` 方法。当接收到 `StartRollCall` 消息并调用远程服务器的 `start_roll_call` 方法时，我们将它设置为 `true`。如果标志已经设置为 `true`，并且常规收到 `StartRollCall` 消息，则线程将调用 `mark_itself` 远程方法。换句话说，我们将调用所有运行服务实例的 `start_roll_call` 方法。当所有服务都将标志设置为 `true` 时，我们将调用所有服务的 `mark_itself` 方法。

让我们启动一个服务器并运行一个服务环。

# 服务器

`main` 函数激活一个记录器并启动一个工作进程。然后，我们提取 `ADDRESS` 环境变量以使用此地址值来绑定服务器的套接字。看看以下代码：

```rs
fn main() -> Result<(), Error> {
    env_logger::init();
    let tx = spawn_worker()?;
    let addr: SocketAddr = env::var("ADDRESS")?.parse()?;
    let mut io = IoHandler::default();
    let sender = Mutex::new(tx.clone());
    io.add_method(START_ROLL_CALL, move |_| {
        trace!("START_ROLL_CALL");
        let tx = sender
            .lock()
            .map_err(to_internal)?;
        tx.send(Action::StartRollCall)
            .map_err(to_internal)
            .map(|_| Value::Bool(true))
    });
    let sender = Mutex::new(tx.clone());
    io.add_method(MARK_ITSELF, move |_| {
        trace!("MARK_ITSELF");
        let tx = sender
            .lock()
            .map_err(to_internal)?;
        tx.send(Action::MarkItself)
            .map_err(to_internal)
            .map(|_| Value::Bool(true))
    });
    let server = ServerBuilder::new(io).start_http(&addr)?;
    Ok(server.wait())
}
```

要实现 JSON-RPC 方法，我们使用 `IoHandler` 结构体。它有一个 `add_method` 方法，该方法期望方法名称，并需要一个包含此方法实现的闭包。

我们添加了两个方法，`start_roll_call` 和 `mark_itself`，使用常量作为这些方法的名称。这些方法的实现很简单：我们只准备相应的 `Action` 消息并将它们发送到工作线程。

JSON-RPC 方法实现必须返回 `Result<Value, ServerError>` 值。要将任何其他错误转换为 `ServerError`，我们使用以下函数：

```rs
fn to_internal<E: fmt::Display>(err: E) -> ServerError {
    error!("Error: {}", err);
    ServerError::internal_error()
}
```

此函数仅打印当前错误消息，并使用 `ServerError` 类型的 `internal_error` 方法创建一个带有 `InternalError` 代码的错误。

在主函数的末尾，我们使用创建的 `IoHandler` 创建一个新的 `ServerBuilder` 实例，并启动 HTTP 服务器以监听带有 `start_http` 服务器的 JSON-RPC 请求。

现在我们可以启动一个服务环来测试它。

# 编译和运行

使用 `cargo build` 子命令编译此示例，并使用以下命令启动三个服务实例（在每个单独的终端窗口中运行每个命令以查看日志）：

```rs
RUST_LOG=JSON-RPC_ring=trace ADDRESS=127.0.0.1:4444 NEXT=127.0.0.1:5555 target/debug/JSON-RPC-ring
RUST_LOG=JSON-RPC_ring=trace ADDRESS=127.0.0.1:5555 NEXT=127.0.0.1:6666 target/debug/JSON-RPC-ring
RUST_LOG=JSON-RPC_ring=trace ADDRESS=127.0.0.1:6666 NEXT=127.0.0.1:4444 target/debug/JSON-RPC-ring
```

当三个服务启动时，从另一个终端窗口使用 `curl` 准备并发送一个 JSON-RPC 调用请求：

```rs
curl -H "Content-Type: application/json" --data-binary '{"JSON-RPC":"2.0","id":"curl","method":"start_roll_call","params":[]}' http://127.0.0.1:4444
```

使用此命令，我们启动所有服务的交互，它们将在环中相互调用。您将看到每个服务的日志。第一个将打印如下内容：

```rs
[2019-01-14T10:45:29Z TRACE JSON-RPC_ring] START_ROLL_CALL
[2019-01-14T10:45:29Z DEBUG JSON-RPC_ring] ON
[2019-01-14T10:45:29Z TRACE JSON-RPC_ring] START_ROLL_CALL
[2019-01-14T10:45:29Z DEBUG JSON-RPC_ring] OFF
[2019-01-14T10:45:29Z TRACE JSON-RPC_ring] MARK_ITSELF
[2019-01-14T10:45:29Z DEBUG JSON-RPC_ring] SKIP
```

第二个将打印如下内容：

```rs
[2019-01-14T10:45:29Z TRACE JSON-RPC_ring] START_ROLL_CALL
[2019-01-14T10:45:29Z DEBUG JSON-RPC_ring] ON
[2019-01-14T10:45:29Z TRACE JSON-RPC_ring] MARK_ITSELF
[2019-01-14T10:45:29Z DEBUG JSON-RPC_ring] OFF
```

第三个将输出以下日志：

```rs
[2019-01-14T10:45:29Z TRACE JSON-RPC_ring] START_ROLL_CALL
[2019-01-14T10:45:29Z DEBUG JSON-RPC_ring] ON
[2019-01-14T10:45:29Z TRACE JSON-RPC_ring] MARK_ITSELF
[2019-01-14T10:45:29Z DEBUG JSON-RPC_ring] OFF
```

所有服务作为过程的独立参与者工作，对传入的消息做出反应，并在有消息要发送时向其他服务发送消息。

# 了解 gRPC

在本节中，我们将把 JSON-RPC 环形示例重写为 gRPC。这个协议与 JSON-RPC 不同，因为它需要一个协议声明——预定义的交互模式。这种限制对大型项目来说是有益的，因为你不能在消息布局中犯错误，但使用 Rust，JSON-RPC 也是可靠的，因为你必须精确地声明所有结构体，如果你接收到一个不正确的 JSON 消息，你会得到一个错误。使用 gRPC，你根本不必关心这一点。

# gRPC 的工作原理

与 JSON-RPC 相比，gRPC 的优点在于速度。gRPC 可以更快地工作，因为它使用了一种快速的序列化格式——Protocol Buffers。gRPC 和 Protocol Buffers 都最初由 Google 开发，并在高性能案例中得到了验证。

gRPC 使用 `HTTP/2` 进行传输。它是一个非常快且优秀的传输协议。首先，它是二进制的：所有请求和响应都被压缩成一个紧凑的字节部分并压缩。它是多路复用的：你可以同时发送很多请求，但 `HTTP/1` 要求请求的顺序。

gRPC 需要一个方案，并使用 Protocol Buffers 作为 **接口定义语言**（**IDL**）。在开始编写服务的实现之前，你必须编写包含所有类型和服务的声明的 `proto` 文件。之后，你需要将声明编译成源代码（在我们的案例中是 Rust 编程语言），并使用它们来编写实现。

`protobuf` crate 和常见的 gRPC crate 以该 crate 为基础。实际上，crate 并不多；只有两个：`grpcio` crate，它是对原始 gRPC 核心库的包装，以及 `grpc` crate，它是协议的纯 Rust 实现。

现在，我们可以将之前的示例从 JSON-RPC 协议重写为 gRPC。首先，我们必须添加所有必要的依赖项并编写我们服务的声明。

# 创建微服务

gRPC 示例非常复杂，因为我们必须声明一个交互协议。我们还需要添加 `build.rs` 文件，从协议描述生成 Rust 源代码。

由于从 curl 调用 gRPC 很困难，我们还将添加一个客户端，帮助我们测试服务。你也可以使用其他可用于调试 gRPC 应用程序的工具。

# 依赖项

创建一个新的 `binary` crate 并在编辑器中打开 `Cargo.toml` 文件。我们将探索这个文件的每一个部分，因为构建 gRPC 示例比使用灵活交互协议（如 JSON-RPC）的服务更复杂。我们将使用 Edition 2018，正如我们在本书中的大多数示例中所做的那样：

```rs
[package]
name = "grpc-ring"
version = "0.1.0"
authors = ["your email"]
edition = "2018"
```

在依赖项中，我们需要一组基本的 crate——`failure`、`log` 和 `env_logger`。此外，我们添加了 `protobuf` crate。我们不会直接使用它，但它在稍后本节中从协议描述生成的 Rust 源代码中会被使用。当前示例中最重要的 crate 是 grpc。我们将使用 GitHub 上的版本，因为该 crate 正在积极开发中：

```rs
[dependencies]
env_logger = "0.6"
failure = "0.1"
grpc = { git = "https://github.com/stepancheg/grpc-rust" }
log = "0.4"
protobuf = "2.2"
```

实际上，`grpc` crate 的 GitHub 仓库是一个工作区，并且还包含 `protoc-rust-grpc` crate，我们将使用它通过 `build.rs` 文件在 Rust 中生成协议声明。将此依赖项添加到 `Cargo.toml` 的 `[build-dependencies]` 部分：

```rs
[build-dependencies]
protoc-rust-grpc = { git = "https://github.com/stepancheg/grpc-rust" }
```

我们创建的 `example` crate 将生成两个二进制文件——服务器和客户端。正如我所说的，我们需要一个客户端，因为它比手动准备调用更简单，并且使用 curl 调用 gRPC 方法。

第一个二进制文件是由 `src/server.rs` 文件构建的服务器：

```rs
[[bin]]
name = "grpc-ring"
path = "src/server.rs"
test = false
```

第二个二进制文件使用 `src/client.rs` 文件构建客户端：

```rs
[[bin]]
name = "grpc-ring-client"
path = "src/client.rs"
test = false
```

我们还有 `src/lib.rs` 用于通用部分，但我们必须描述一个协议并创建 `build.rs` 文件。

# 协议

gRPC 使用一种特殊的语言进行协议声明。该语言有两种版本——`proto2` 和 `proto3`。我们将使用第二种，因为它更现代。创建一个 `ring.proto` 文件并添加以下声明：

```rs
syntax = "proto3";

option java_multiple_files = true;
option java_package = "rust.microservices.ring";
option java_outer_classname = "RingProto";
option objc_class_prefix = "RING";

package ringproto;

message Empty { }

service Ring {
  rpc StartRollCall (Empty) returns (Empty);
  rpc MarkItself (Empty) returns (Empty);
}
```

如您所见，我们指定了 `proto3` 语法。选项为您提供了设置不同语言源文件生成属性的能力，如果您将与其他应用程序或其他微服务交互服务，您可能需要设置这些选项。对于我们的示例，我们不需要设置这些选项，但如果您从其他开发者那里获取它，您可能需要在文件中包含这部分。

协议声明包含一个使用 `package` 修饰符设置的包名和我们将它设置为 `ringproto` 的包名。

此外，我们还添加了一个没有字段的 `Empty` 消息。我们将使用此类型作为所有方法的输入和输出参数，但对于实际的微服务来说，使用不同的类型会更好。首先，您不能没有输入和输出参数的方法。第二个原因是未来的服务改进。如果您想在以后添加额外的字段到协议中，您可以这样做。此外，协议可以轻松地处理协议的不同版本；通常，您可以同时使用新旧微服务，因为 Protocol Buffers 与额外字段配合良好，并且您可以在需要时扩展协议。

服务声明包含在 `service` 部分。您可以在协议声明文件中拥有多个服务的声明，并在实现中仅使用必要的声明服务。但我们的环形示例只需要一个服务声明。添加 `Ring` 服务并包含两个带有 `rpc` 修饰符的 RPC 方法。我们添加了 `StartRollCall` 方法和 `MakeItself`。与上一个示例相同。两者都接受 `Empty` 值作为输入参数，并返回 `Empty`。

服务名称很重要，因为它将被用作生成 Rust 源代码中多个类型的名称前缀。您可以使用 `protoc` 工具创建源代码，但创建一个在编译期间生成带有协议类型的源代码的构建脚本会更方便。

# 生成接口

Rust 构建脚本允许你实现一个函数，该函数将为项目编译做一些额外的准备。在我们的例子中，我们有一个`ring.proto`文件，其中包含协议定义，我们希望使用`protoc-rust-grpc`包将其转换为 Rust 源代码。

在项目中创建`build.rs`文件并添加以下内容：

```rs
extern crate protoc_rust_grpc;

fn main() {
    protoc_rust_grpc::run(protoc_rust_grpc::Args {
        out_dir: "src",
        includes: &[],
        input: &["ring.proto"],
        rust_protobuf: true,
        ..Default::default()
    }).expect("protoc-rust-grpc");
}
```

构建脚本使用`main`函数作为入口点，在其中你可以实现任何想要的活动。我们使用了`protoc-rust-grpc`包的 run 函数与`Args`——我们在`out_dir`字段中设置了输出目录，将`ring.proto`文件作为输入声明设置在`input`字段中，激活了`rust_protobuf`布尔标志以生成`rust**-**protobuf`包的源代码（如果你已经使用`protobuf`包并使用它生成类型，则不需要它），然后设置`includes`字段为一个空数组。

然后，当你运行`cargo build`时，它将在`src`文件夹中生成两个模块：`ring.rs`和`ring_grpc.rs`。我不会在这里放置其源代码，因为生成的文件很大，但我们将使用它来创建一个 gRPC 客户端的包装器，就像我们在前面的例子中所做的那样。

# 共享客户端

打开`lib.rs`源文件并添加两个生成的模块：

```rs
mod ring;
mod ring_grpc;
```

导入我们需要创建 gRPC 客户端包装器的类型：

```rs
use crate::ring::Empty;
use crate::ring_grpc::{Ring, RingClient};
use grpc::{ClientConf, ClientStubExt, Error as GrpcError, RequestOptions};
use std::net::SocketAddr;
```

如你所见，生成的模块包含我们在`ring.proto`文件中声明的类型。`ring`模块包含`Empty`结构体，而`ring_grpc`模块包含`Ring`特质，它表示远程服务的接口。此外，`protoc_rust_grpc`在生成的构建脚本中创建了`RingClient`类型。此类型是一个可以用来调用远程方法的客户端。我们用我们自己的结构体包装它，因为`RingClient`生成`Future`实例，我们将使用`Remote`包装器来执行它们并获取结果。

我们还使用来自`grpc`包的类型。`Error`类型被导入为`GrpcError`；

`RequestOptions`，它是准备方法调用请求所必需的；`ClientConf`，用于为`HTTP/2`连接添加额外的配置参数（我们将使用默认值）；以及`ClientStubExt`，它为客户端提供连接方法。

在内部添加持有`RingClient`实例的`Remote`结构体：

```rs
pub struct Remote {
    client: RingClient,
}
```

我们使用这个结构体来创建客户端和服务器。添加一个新的方法从提供的`SocketAddr`构建新的`Remote`实例：

```rs
impl Remote {
    pub fn new(addr: SocketAddr) -> Result<Self, GrpcError> {
        let host = addr.ip().to_string();
        let port = addr.port();
        let conf = ClientConf::default();
        let client = RingClient::new_plain(&host, port, conf)?;
        Ok(Self {
            client
        })
    }
}
```

由于生成的客户端期望有单独的主机和端口值，我们从`SocketAddr`值中提取它们。此外，我们创建默认的`ClientConf`配置，并使用所有这些值来创建`RingClient`实例，并将其放入新的`Remote`实例中。

我们创建`Remote`结构体以拥有调用远程方法的基本方法。向`Remote`实现中添加`start_roll_call`方法以调用`StartRollCall` gRPC 方法：

```rs
pub fn start_roll_call(&self) -> Result<Empty, GrpcError> {
    self.client.start_roll_call(RequestOptions::new(), Empty::new())
        .wait()
        .map(|(_, value, _)| value)
}
```

`RingClient`已经具有此方法，但它期望我们想要隐藏的参数，并返回一个我们想要立即使用`wait`方法调用的`Future`实例。`Future`返回一个包含三个项目的元组，但我们只需要一个值，因为其他值包含我们不需要的元数据。

以类似的方式实现`mark_itself`方法来调用`MarkItself` gRPC 方法：

```rs
pub fn mark_itself(&self) -> Result<Empty, GrpcError> {
    self.client.mark_itself(RequestOptions::new(), Empty::new())
        .wait()
        .map(|(_, value, _)| value)
}
```

现在我们可以实现客户端和服务器，因为两者都需要`Remote`结构体来执行 RPC 调用。

# 客户端

添加`src/client.rs`文件并导入一些类型：

```rs
use failure::Error;
use grpc_ring::Remote;
use std::env;
```

我们需要从`failure` crate 中获取一个通用的`Error`类型，因为它是最常用的错误处理类型，并且导入我们之前创建的`Remote`结构体。

客户端是一个非常简单的工具。它使用在`NEXT`环境变量中提供的地址调用服务的`StartRollCall`远程 gRPC 方法：

```rs
fn main() -> Result<(), Error> {
    let next = env::var("NEXT")?.parse()?;
    let remote = Remote::new(next)?;
    remote.start_roll_call()?;
    Ok(())
}
```

使用解析的`SocketAddr`值创建`Remote`实例并执行调用。就是这样。服务器非常复杂。让我们来实现它。

# 服务器实现

添加`src/server.rs`源文件并将其添加到其中：

```rs
mod ring;
mod ring_grpc;
```

我们需要这些模块，因为我们将为我们的 RPC 处理程序实现`Ring`特质。看看我们将使用的类型：

```rs
use crate::ring::Empty;
use crate::ring_grpc::{Ring, RingServer};
use failure::Error;
use grpc::{Error as GrpcError, ServerBuilder, SingleResponse, RequestOptions};
use grpc_ring::Remote;
use log::{debug, trace};
use std::env;
use std::net::SocketAddr;
use std::sync::Mutex;
use std::sync::mpsc::{channel, Receiver, Sender};
```

你可能还不熟悉的一些类型是`ServerBuilder`，它用于创建服务器实例并填充服务实现，以及`SingleResponse`是处理程序调用的结果。其他类型你已经知道了。

# 服务实现

我们需要一个自己的类型来实现`Ring`特质以实现服务的 RPC 接口。但我们还必须保留一个`Sender`以供工作进程发送动作。添加带有`Action`的`Sender`和`Mutex`包装的`RingImpl`结构体，因为`Ring`特质还要求实现`Sync`特质：

```rs
struct RingImpl {
    sender: Mutex<Sender<Action>>,
}
```

我们将从`Sender`实例构建一个实例：

```rs
impl RingImpl {
    fn new(sender: Sender<Action>) -> Self {
        Self {
            sender: Mutex::new(sender),
        }
    }
}
```

对于每个传入的方法调用，我们需要向工作进程发送`Action`，我们可以在`RingImpl`实现中添加一个方法来在所有处理程序中重用它：

```rs
fn send_action(&self, action: Action) -> SingleResponse<Empty> {
    let tx = try_or_response!(self.sender.lock());
    try_or_response!(tx.send(action));
    let result = Empty::new();
    SingleResponse::completed(result)
}
```

`send_action`函数接收一个`Action`值，并锁定一个`Mutex`来使用`Sender`。最后，它创建一个`Empty`值并将其作为`SingleResponse`实例返回。如果你注意到了，我们使用了我们定义的`try_or_response!`宏，因为`SingleResponse`是一个`Future`实例，我们必须在任何成功或失败的情况下返回此类型。

此宏类似于标准库中的`try!`宏。它使用 match 来提取值或在没有错误值的情况下返回结果：

```rs
macro_rules! try_or_response {
    ($x:expr) => {{
        match $x {
            Ok(value) => {
                value
            }
            Err(err) => {
                let error = GrpcError::Panic(err.to_string());
                return SingleResponse::err(error);
            }
        }
    }};
}
```

前面的宏使用`GrpcError`的`Panic`变体创建`SingleResponse`实例，但使用现有错误值的错误描述。

# 处理程序

现在，我们可以实现我们在`ring.proto`文件中声明的`Ring`服务的 gRPC 方法。我们有与方法相同名称的`Ring`特质。每个方法都期望一个`Empty`值并必须返回此类型，因为我们已经在声明中定义了它。此外，每个方法都必须返回作为结果的`SingleResponse`类型。我们已经定义了发送`Action`值到工作线程并返回带有`Empty`值的`SingleResponse`响应的`send_action`方法。让我们使用`send_action`方法来实现我们必须要实现的方法：

```rs
impl Ring for RingImpl {
    fn start_roll_call(&self, _: RequestOptions, _: Empty) -> SingleResponse<Empty> {
        trace!("START_ROLL_CALL");
        self.send_action(Action::StartRollCall)
    }

    fn mark_itself(&self, _: RequestOptions, _: Empty) -> SingleResponse<Empty> {
        trace!("MARK_ITSELF");
        self.send_action(Action::MarkItself)
    }
}
```

我们对 gRPC 方法处理器的实现相当简单，但你也可以添加更合理的实现，并从 Future 异步地生成`SingleResponse`。

# 主函数

一切都为`main`函数的实现准备好了：

```rs
fn main() -> Result<(), Error> {
    env_logger::init();
    let (tx, rx) = channel();
    let addr: SocketAddr = env::var("ADDRESS")?.parse()?;
    let mut server = ServerBuilder::new_plain();
    server.http.set_addr(addr)?;
    let ring = RingImpl::new(tx);
    server.add_service(RingServer::new_service_def(ring));
    server.http.set_cpu_pool_threads(4);
    let _server = server.build()?;

    worker_loop(rx)
}
```

我们初始化了一个日志记录器，并创建了一个通道，我们将使用它从`RingImpl`向工作线程发送动作。我们从`ADDRESS`环境变量中提取了`SocketAddr`，并使用这个值将服务器绑定到提供的地址。

我们使用`new_plain`方法创建了一个`ServerBuilder`实例。它创建了一个没有 TLS 的服务器，因为 gRPC 支持安全连接，我们必须为`ServerBuilder`提供一个实现`TlcAcceptor`特质的类型参数。但是使用`new_plain`，我们使用来自`tls_api_stub`包的`TlasAcceptor`存根。`ServerBuilder`结构体包含了`httpbis::ServerBuilder`类型的`http`字段。我们可以使用这个文件来设置服务器套接字的绑定地址。

之后，我们创建了`RingImpl`实例，并使用`ServiceBuilder`的`add_service`方法附加一个服务实现，但我们必须提供服务的泛型`grpc::rt::ServerServiceDefinition`定义，并使用`RingServer`类型的`new_service_def`来为`RingImpl`实例创建它。

最后，我们设置处理传入请求的线程池中线程的数量，并调用`ServiceBuilder`的`build`方法来启动服务器。但是等等——如果你不调用`build`方法，主线程将被终止，你必须添加一个循环或其他例程来保持主线程活跃。

幸运的是，我们需要一个工作线程，我们可以使用主线程来运行它。如果你只需要运行 gRPC 服务器，你可以使用一个带有`thread::park`方法调用的`loop`，这将阻塞线程，直到它被`unpark`方法调用解除阻塞。这种方法被异步运行时内部使用。

我们将使用`worker_loop`函数调用，但我们还没有实现这个函数。

# 工作线程

我们已经在 JSON-RPC 示例中实现了工作线程。在 gRPC 版本中，我们使用相同的代码，但期望一个`Receiver`值，并且不创建新的线程：

```rs
fn worker_loop(receiver: Receiver<Action>) -> Result<(), Error> {
    let next = env::var("NEXT")?.parse()?;
    let remote = Remote::new(next)?;
    let mut in_roll_call = false;
    for action in receiver.iter() {
        match action { /* Action variants here */ }
    }
    Ok(())
}
```

让我们编译并运行这个示例。

# 编译和运行

使用`cargo build`子命令构建服务器和客户端。

如果你想要指定二进制文件，请使用带有二进制文件名的--bin 参数。

此外，你可以使用`cargo watch`工具进行构建。

如果你使用`cargo watch`工具，那么`build.rs`脚本将生成带有 gRPC 类型的文件，并且`watch`将不断重启构建。为了防止这种情况，你可以设置命令的`--ignore`参数，使用文件名模式的忽略。在我们的例子中，我们必须运行`cargo watch --ignore 'src/ring*'`命令。

当两个二进制文件都构建完成后，在三个不同的终端中运行三个实例：

```rs
RUST_LOG=grpc_ring=trace ADDRESS=127.0.0.1:4444 NEXT=127.0.0.1:5555 target/debug/grpc-ring
RUST_LOG=grpc_ring=trace ADDRESS=127.0.0.1:5555 NEXT=127.0.0.1:6666 target/debug/grpc-ring
RUST_LOG=grpc_ring=trace ADDRESS=127.0.0.1:6666 NEXT=127.0.0.1:4444 target/debug/grpc-ring
```

当所有服务启动后，使用客户端向第一个服务发送请求：

```rs
NEXT=127.0.0.1:4444 target/debug/grpc-ring-client
```

此命令将调用远程方法`start_roll_call`，你将看到与前面 JSON-RPC 示例中类似的服务器日志。

# 摘要

本章介绍了创建响应式微服务架构的良好实践。我们从基本概念开始学习：什么是响应式方法，如何实现它，以及远程过程调用如何帮助实现消息驱动架构。此外，我们还讨论了你可以用 Rust 简单使用的现有 RPC 框架和 crate。

为了演示响应式应用程序的工作原理，我们创建了两个使用 RPC 方法相互交互的微服务示例。我们创建了一个应用程序，它使用一组正在运行的微服务，这些微服务在一个循环中相互发送请求，直到每个实例都了解一个事件。

我们还创建了一个示例，它使用 JSON-RPC 协议进行实例交互，并使用`jsonrpc-http-server`crate 作为服务器端，使用 JSON-RPCcrate 作为客户端。

之后，我们创建了一个示例，它使用 gRPC 协议进行微服务交互，并使用了`grpc`crate，该 crate 涵盖了客户端和服务器端。

在下一章中，我们将开始将微服务与数据库集成，并探索可用于与以下数据库交互的 crate：PostgreSQL、MySQL、Redis、MongoDB、DynamoDB。
