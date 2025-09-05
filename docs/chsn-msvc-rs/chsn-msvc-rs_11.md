# 使用 Actix Crate 和演员进行并发

本章将展示一种基于演员模型（如 Erlang 或 Akka）创建微服务的替代方法。这种方法允许您通过将微服务拆分为小型独立任务，并通过消息传递相互交互来编写清晰和有效的代码。

到本章结束时，您将能够做到以下事情：

+   使用 Actix 框架和`actix-web`包创建微服务

+   为 Actix Web 框架创建中间件

# 技术要求

要实现和运行本章的所有示例，您至少需要一个版本为 1.31 的 Rust 编译器。

您可以在 GitHub 上找到本章代码示例的来源：[`github.com/PacktPublishing/Hands-On-Microservices-with-Rust/tree/master/Chapter11`](https://github.com/PacktPublishing/Hands-On-Microservices-with-Rust/tree/master/Chapter11)

# 微服务中的演员并发

如果您熟悉 Erlang 或 Akka，您可能已经知道演员是什么以及如何使用它们。但无论如何，我们将在本节中回顾演员模型的知识。

# 理解演员

我们已经在第十章中熟悉了演员，*微服务中的后台任务和线程池*，但让我们谈谈使用演员进行微服务。

演员是进行并发计算的模式。我们应该了解以下模型：

+   **线程**：在这个模型中，每个任务都在单独的线程中工作

+   **纤程或绿色线程**：在这个模型中，每个任务都有由特殊运行时安排的工作

+   **异步代码**：在这个模型中，每个任务都由一个反应器（实际上，这类似于纤程）运行

演员将这些方法结合成一个优雅的解决方案。要完成任何工作的一部分，您可以实现执行自己部分工作的演员，并通过消息与其他演员交互，以通知彼此整体进度。每个演员都有一个用于接收消息的邮箱，并且可以使用此地址向其他演员发送消息。

# 微服务中的演员

要使用演员开发微服务，您应该将您的服务拆分为解决不同类型工作的任务。例如，您可以为每个传入的连接或数据库交互使用单独的演员，甚至作为管理员来控制其他演员。每个演员都是一个在反应器中执行的非同步任务。

这种方法的优点如下：

+   将单独的演员编写比编写大量函数要简单

+   演员可能会失败并重新启动

+   您可以重用演员

使用演员的一个重要好处是可靠性，因为每个演员都可以失败并重新启动，因此您不需要编写冗长的恢复代码来处理故障。这并不意味着您的代码可以在任何地方调用`panic!`宏，但这确实意味着您可以将演员视为短生命周期任务，它们在小型任务上并发工作。

如果你设计演员得当，你也会获得很好的性能，因为与消息的交互可以帮助你将工作分成短反应，这不会长时间阻塞反应器。此外，你的源代码结构也会更加清晰。

# Actix 框架

Actix 框架为 Rust 提供了一个基于 `futures` crate 和一些异步代码的演员模型，允许演员以最小资源需求并发工作。

我认为这是用 Rust 创建网络应用和微服务的最佳工具之一。该框架包括两个优秀的 crate——包含核心结构的 `actix` crate 和实现 HTTP 和 WebSocket 协议的 `actix-web` crate。让我们创建一个将请求路由到其他微服务的微服务。

# 使用 actix-web 创建微服务

在本节中，我们将创建一个微服务，它看起来与我们之前在 第九章 中创建的其他微服务类似，即 *使用框架进行简单的 REST 定义和请求路由*，但内部使用演员模型来实现资源充分利用。

要使用 `actix-web` 创建一个微服务，你需要添加 `actix` 和 `actix-web` 两个 crate。首先，我们需要启动管理其他演员运行时的 `System` 演员实例。让我们创建一个 `System` 实例，并使用必要的路由启动一个 `actix-web` 服务器。

# 启动 actix-web 服务器

启动一个 `actix-web` 服务器实例看起来与其他 Rust 网络框架类似，但需要 `System` 演员实例。我们不需要直接使用 `System`，但需要在一切准备就绪时通过调用 `run` 方法来运行它。这个调用启动了 `System` 演员并阻塞了当前线程。内部，它使用了我们在前几章讨论过的 `block_on` 方法。

# 启动服务器

考虑以下代码：

```rs
fn main() {
    env_logger::init();
    let sys = actix::System::new("router");
    server::new(|| {
      // Insert `App` declaration here
    }).workers(1)
        .bind("127.0.0.1:8080")
        .unwrap()
        .start();
    let _ = sys.run();
}
```

我们通过调用 `server::new` 方法创建一个新的服务器，该方法期望一个闭包来返回 `App` 实例。在我们创建 `App` 实例之前，我们必须完成服务器并运行它。`workers` 方法设置运行演员的线程数。

你可以选择不显式设置此值，默认情况下，它将等于系统上的 CPU 数量。在许多情况下，这是性能的最佳可能值。

下一个 `bind` 方法的调用将服务器的套接字绑定到一个地址。如果无法绑定到地址，该方法返回 `Err`，如果我们不能在期望的端口上启动服务器，我们将 `unwrap` 结果以停止服务器。最后，我们调用 `start` 方法来启动 `Server` 演员。它返回一个包含地址的 `Addr` 结构体，你可以使用该地址向 `Server` 演员实例发送消息。

实际上，`Server` 演员不会运行，直到我们调用 `System` 实例的 `run` 方法。添加这个方法调用，然后我们将详细查看创建 `App` 实例。

# 创建应用

将以下代码插入到 `server::new` 函数调用的闭包中：

```rs
let app = App::with_state(State::default())
    .middleware(middleware::Logger::default())
    .middleware(IdentityService::new(
            CookieIdentityPolicy::new(&[0; 32])
            .name("auth-example")
            .secure(false),
            ))
    .middleware(Counter);
```

`App` 结构体包含有关状态、中间件和路由作用域的信息。为了将共享状态设置到我们的应用程序中，我们使用 `with_state` 方法来构建 `App` 实例。我们创建 `State` 结构体的默认实例，其声明如下：

```rs
#[derive(Default)]
struct State(RefCell<i64>);
```

`State` 包含一个带有 `i64` 值的 cell，用于计算所有请求。默认情况下，它使用 `0` 值创建。

在此之后，我们使用 `App` 的中间件方法来设置以下三个中间件：

+   `actix_web::middleware::Logger` 是一个使用 `log` crate 记录请求和响应的日志器

+   `actix_web::middleware::identity::IdentityService` 通过实现 `IdentityPolicy` 特性的身份后端来帮助识别请求

+   `Counter` 是一个中间件，我们将在接下来的 *中间件* 部分中创建它，并使用 `State` 来计算请求的总数量

对于我们的 `IdentityPolicy` 后端，我们使用与 `IdentityService` 同一身份子模块中的 `CookieIdentityPolicy`。`CookieIdentityPolicy` 期望一个至少有 32 字节的密钥。当创建了一个身份策略的 cookie 实例后，我们可以使用 `path`、`name` 和 `domain` 等方法来设置特定的 cookie 参数。我们还通过使用 `secure` 方法并设置 `false` 值来允许通过不安全的连接发送 cookie。

你应该了解 cookie 的两个特殊参数：`Secure` 和 `HttpOnly`。第一个要求使用安全的 HTTPS 连接来发送 cookie。如果你运行一个用于测试的服务并使用纯 HTTP 连接到它，那么 `CookieIdentityPolicy` 将不会工作。`HttpOnly` 参数不允许从 JavaScript 使用 cookie。`CookieIdentityPolicy` 将此参数设置为 true，并且你不能覆盖这种行为。

# 作用域和路由

我们必须添加到我们的 `App` 实例中的下一件事是路由。有一个 `route` 函数可以让你为任何路由设置处理器。但使用作用域来构建嵌套路径的结构会更周到。看看下面的代码：

```rs
app.scope("/api", |scope| {
    scope
        .route("/signup", http::Method::POST, signup)
        .route("/signin", http::Method::POST, signin)
        .route("/new_comment", http::Method::POST, new_comment)
        .route("/comments", http::Method::GET, comments)
})
```

我们 `App` 结构体的 `scope` 方法期望一个路径的前缀和一个以 `scope` 作为单一参数的闭包，并创建一个可以包含子路由的作用域。我们为 `/api` 路径前缀创建一个 `scope`，并使用 `route` 方法添加了四个路由：`/signup`、`/signin`、`/new_comment` 和 `/comments`。`route` 方法期望一个包括路径、方法和处理器的后缀。例如，如果服务器现在接收一个使用 `POST` 方法的 `/api/singup` 请求，它将调用 `signup` 函数。让我们为其他路径添加一个默认处理器。

我们的微服务还使用 `Counter` 中间件，我们将在稍后实现它，来计算请求的总数量。我们需要添加一个路由来渲染微服务的统计信息，如下所示：

```rs
.route("/counter", http::Method::GET, counter)
```

如您所见，我们这里不需要 `scope`，因为我们只有一个处理器，可以直接为 `App` 实例（而不是 `scope`）调用 `route` 方法。

# 静态文件处理器

对于之前 `scope` 中未列出的其他路径，我们将使用一个 `handler`，它将从文件夹中返回文件的 内容以提供静态资源。`handler` 方法期望一个路径的前缀，以及一个实现了 `Handler` 特质的类型。在我们的情况下，我们将使用一个现成的静态文件处理程序，`actix_web::fs::StaticFiles`。它需要一个指向本地文件夹的路径，我们还可以通过调用 `index_file` 方法设置索引文件：

```rs
app.handler(
    "/",
    fs::StaticFiles::new("./static/").unwrap().index_file("index.html")
)
```

现在，如果客户端向路径例如 `/index.html` 或 `/css/styles.css` 发送 `GET` 请求，那么 `StaticFiles` 处理程序将发送来自 `./static/` 本地文件夹中相应文件的 内容。

# HTTP 客户端

该微服务的处理程序作为代理工作，并将传入的请求重新发送到其他微服务，这些微服务将不会直接对用户可用。要向其他微服务发送请求，我们需要一个 HTTP 客户端。`actix_web` 包含一个。要使用客户端，我们需要添加两个函数：一个用于代理 `GET` 请求，另一个用于发送 `POST` 请求。

# GET 请求

要发送 `GET` 请求，我们创建一个 `get_request` 函数，它期望一个 `url` 参数并返回一个包含二进制数据的 `Future` 实例：

```rs
fn get_request(url: &str) -> impl Future<Item = Vec<u8>, Error = Error> {
    client::ClientRequest::get(url)
        .finish().into_future()
        .and_then(|req| {
            req.send()
                .map_err(Error::from)
                .and_then(|resp| resp.body().from_err())
                .map(|bytes| bytes.to_vec())
        })
}
```

我们使用 `ClientRequestBuilder` 来创建 `ClientRequest` 实例。`ClientRequest` 结构体已经具有创建具有预设 HTTP 方法的构建器的快捷方式。我们调用 `get` 方法，该方法仅将 `Method::GET` 值设置到作为 `ClientRequestBuilder` 实例调用方法的请求中。您还可以使用构建器设置额外的头或 cookie。当您完成这些值后，您必须通过调用以下方法之一从构建器创建 `ClientRequest` 实例：

+   `body` 将体值设置为可以转换为 `Into<Body>` 的二进制数据

+   `json` 将体值设置为一个可以序列化为 JSON 值的任何类型

+   `form` 将体值设置为一个可以使用 `serde_urlencoded: serializer` 序列化的类型

+   `streaming` 从 `Stream` 实例中消耗体值

+   `finish` 创建一个不带体值的请求

我们使用 `finish`，因为 `GET` 请求不包含体值。所有这些方法都返回一个包含 `ClientRequest` 实例的成功值的 `Result`。我们不展开 `Result`，而是通过调用 `into_future` 方法将其转换为 `Future` 值，以便在处理程序甚至无法构建请求时向客户端返回一个错误值。

由于我们有一个 `Future` 值，我们可以使用 `and_then` 方法添加下一个处理步骤。我们调用 `ClientRequest` 的 `send` 方法来创建一个 `SendRequest` 实例，该实例实现了 `Future` 特质并向服务器发送请求。由于 `send` 调用可以返回 `SendRequestError` 错误类型，我们将其包装在 `failure::Error` 中。

如果请求成功发送，我们可以使用 `body` 方法调用获取 `MessageBody` 值。此方法是 `HttpMessage` 特质的一部分。`MessageBody` 还实现了具有 `Bytes` 值的 `Future` 特质，我们使用 `and_then` 方法扩展未来链并从 `SendRequest` 转换值。

最后，我们使用 `Bytes` 的 `to_vec` 方法将其转换为 `Vec<u8>` 并将此值作为对客户端的响应。我们已经完成了将 `GET` 请求代理到另一个微服务的函数。让我们为 `POST` 请求创建一个类似的方法。

# `POST` 请求

对于 `POST` 请求，我们需要输入参数，这些参数将被序列化到请求体中，以及输出参数，这些参数将从请求的响应体中反序列化。看看下面的函数：

```rs
fn post_request<T, O>(url: &str, params: T) -> impl Future<Item = O, Error = Error>
where
    T: Serialize,
    O: for <'de> Deserialize<'de> + 'static,
{
    client::ClientRequest::post(url)
        .form(params).into_future().and_then(|req| {
            req.send()
                .map_err(Error::from).and_then(|resp| {
                    if resp.status().is_success() {
                        let fut = resp.json::<O>().from_err();
                        boxed(fut)
                    } else {
                        error!("Microservice error: {}", resp.status());
                        let fut = Err(format_err!("microservice error"))
                            .into_future().from_err();
                        boxed(fut)
                    }
                })
        })
}
```

`post_request` 函数使用 `ClientRequest` 的 `post` 方法创建 `ClientRequestBuilder`，并用 `params` 变量的值填充一个表单。我们将 `Result` 转换为 `Future` 并向服务器发送一个请求。同样，在 `GET` 版本中，我们处理响应，但采用不同的方式。我们通过 `HttpResponse` 的 `status` 方法调用获取响应的状态，并使用 `is_success` 方法调用检查它是否成功。

对于成功的响应，我们使用 `HttpResponse` 的 `json` 方法获取一个收集体并从 JSON 中反序列化的 `Future`。如果响应不成功，我们向客户端返回一个错误。现在，我们有了向其他微服务发送请求的方法，并且可以为每个路由实现处理器。

# 处理器

我们添加了代理传入请求并将它们重新发送到其他微服务的方法。现在，我们可以为我们的微服务支持的每个路径实现处理器。我们将为客户端提供一个整体 API，但实际上，我们将使用一组微服务向客户端提供所有必要的服务。让我们从实现 `/signup` 路径的处理器开始。

# 注册

路由微服务使用 `/signup` 路由将注册请求重新发送到绑定在 `127.0.0.1:8001` 地址的用户微服务。此请求使用 `UserForm` 填充并带有 `Form` 类型参数创建一个新的用户。看看下面的代码：

```rs
fn signup(params: Form<UserForm>) -> FutureResponse<HttpResponse> {
    let fut = post_request("http://127.0.0.1:8001/signup", params.into_inner())
        .map(|_: ()| {
            HttpResponse::Found()
            .header(header::LOCATION, "/login.html")
            .finish()
        });
    Box::new(fut)
}
```

我们调用之前声明的 `post_request` 函数，向用户微服务发送一个 `POST` 请求，如果它返回一个成功的响应，我们返回一个带有 `302` 状态码的响应。我们通过 `HttpResponse::Found` 函数调用创建一个带有相应状态码的 `HttpResponseBuilder`。之后，我们还通过 `HttpResponseBuilder` 的 `header` 方法调用设置 `LOCATION` 标头，将用户重定向到登录表单。最后，我们调用 `finish()` 从构建器创建一个 `HttpResponse` 并将其作为 boxed `Future` 对象返回。

函数具有 `FutureResponse` 返回类型，其实现如下：

```rs
type FutureResponse<I, E = Error> = Box<dyn Future<Item = I, Error = E>>;
```

如您所见，它是一个实现了 `Future` 特质的类型的 `Box`。

此外，函数还期望`Form<UserForm>`作为参数。`UserForm`结构体声明如下：

```rs
#[derive(Deserialize, Serialize)]
pub struct UserForm {
    email: String,
    password: String,
}
```

如您所见，它期望两个参数：`email`和`password`。两者都将从请求的查询字符串中提取，格式为`email=user@example.com&password=<secret>`。`Form`包装器有助于从响应体中提取数据。

`actix_web` 包通过大小限制请求和响应。如果您想发送或接收大量负载，您必须覆盖默认设置，这些设置通常不允许请求超过 256 KB。例如，如果您想增加限制，您可以使用`Route`的`with_config`方法调用提供的`FormConfig`结构体，并使用所需字节数调用配置的`limit`方法。HTTP 客户端也受到响应大小的限制。例如，如果您尝试从`JsonBody`实例中读取大型 JSON 对象，您可能需要在将其用作`Future`对象之前使用`limit`方法调用对其进行限制。

# 登录

其他方法允许用户使用提供的凭据登录到微服务。看看以下发送到`/signin`路径的`signin`函数，它处理发送的请求：

```rs
fn signin((req, params): (HttpRequest<State>, Form<UserForm>))
    -> FutureResponse<HttpResponse>
{
    let fut = post_request("http://127.0.0.1:8001/signin", params.into_inner())
        .map(move |id: UserId| {
            req.remember(id.id);
            HttpResponse::build_from(&req)
            .status(StatusCode::FOUND)
            .header(header::LOCATION, "/comments.html")
            .finish()
        });
    Box::new(fut)
}
```

函数有两个参数：`HttpRequest`和`Form`。第一个我们需要获取对共享`State`对象的访问权限。第二个我们需要从请求体中提取`UserForm`结构体。我们也可以在这里使用`post_request`函数，但期望它在其响应中返回一个`UserId`值。`UserId`结构体声明如下：

```rs
#[derive(Deserialize)]
pub struct UserId {
    id: String,
}
```

由于`HttpRequest`实现了`RequestIdentity`特质，并且我们将`IdentityService`连接到了`App`，我们可以使用用户的 ID 调用`remember`方法，将当前会话与用户关联。

然后，我们创建一个带有`302`状态码的响应，就像我们在前面的处理器中所做的那样，并将用户重定向到`/comments.html`页面。但我们必须从`HttpRequest`构建一个`HttpResponse`实例，以保持`remember`函数调用时的更改。

# 新评论

创建新评论的处理程序使用用户的身份来检查是否有凭据添加新评论：

```rs
fn new_comment((req, params): (HttpRequest<State>, Form<AddComment>))
    -> FutureResponse<HttpResponse>
{
    let fut = req.identity()
        .ok_or(format_err!("not authorized").into())
        .into_future()
        .and_then(move |uid| {
            let params = NewComment {
                uid,
                text: params.into_inner().text,
            };
            post_request::<_, ()>("http://127.0.0.1:8003/new_comment", params)
        })
        .then(move |_| {
            let res = HttpResponse::build_from(&req)
                .status(StatusCode::FOUND)
                .header(header::LOCATION, "/comments.html")
                .finish();
            Ok(res)
        });
    Box::new(fut)
}
```

此处理器允许每个已签名的用户留下评论。让我们看看这个处理器是如何工作的。

首先，它调用`RequestIdentity`特质的`identity`方法，该方法返回用户的 ID。我们将它转换为`Result`，以便将其转换为`Future`，并在用户未识别时返回错误。

我们使用返回的用户 ID 值来准备对评论微服务的请求。我们从`AddComment`表单中提取`text`字段，并使用用户的 ID 和评论创建一个`NewComment`结构体。结构体声明如下：

```rs
#[derive(Deserialize)]
pub struct AddComment {
    pub text: String,
}

#[derive(Serialize)]
pub struct NewComment {
    pub uid: String,
    pub text: String,
}
```

我们也可以使用一个带有可选`uid`的单个结构体，但为了安全起见，最好为不同的需求使用单独的结构体，因为如果我们使用相同的结构体并且在没有验证的情况下将其重新发送到另一个微服务，我们可能会创建一个漏洞，允许任何用户以另一个用户的身份添加评论。通过使用精确、严格的类型而不是通用、灵活的类型来尽量避免这类错误。

最后，我们像之前一样创建一个重定向客户端，并将用户发送到`/comments.html`页面。

# 评论

要查看之前处理程序创建的所有评论，我们必须向评论微服务发送一个`GET`请求，使用我们之前创建的`get_request`函数，并将响应数据重新发送给客户端：

```rs
fn comments(_req: HttpRequest<State>) -> FutureResponse<HttpResponse> {
    let fut = get_request("http://127.0.0.1:8003/list")
        .map(|data| {
            HttpResponse::Ok().body(data)
        });
    Box::new(fut)
}
```

# 计数器

打印请求总量的处理程序也有相当简单的实现，但在这个案例中，我们能够访问共享状态：

```rs
fn counter(req: HttpRequest<State>) -> String {
    format!("{}", req.state().0.borrow())
}
```

我们使用`HttpRequest`的`state`方法来获取对`State`实例的引用。由于计数器值存储在`RefCell`中，我们使用`borrow`方法从单元格中获取值。我们实现了所有处理程序，现在我们必须添加一些中间件来计算对微服务的每个请求。

# 中间件

`actix-web`包支持可以附加到`App`实例上的中间件，以处理每个请求和响应。中间件有助于记录请求、转换它们，甚至可以使用正则表达式控制对一组路径的访问。将中间件视为所有传入请求和传出响应的处理程序。要创建中间件，我们首先必须为其实现`Middleware`特质。看看以下代码：

```rs
pub struct Counter;

impl Middleware<State> for Counter {
    fn start(&self, req: &HttpRequest<State>) -> Result<Started> {
        let value = *req.state().0.borrow();
        *req.state().0.borrow_mut() = value + 1;
        Ok(Started::Done)
    }

    fn response(&self, _req: &HttpRequest<State>, resp: HttpResponse) -> Result<Response> {
        Ok(Response::Done(resp))
    }

    fn finish(&self, _req: &HttpRequest<State>, _resp: &HttpResponse) -> Finished {
        Finished::Done
    }
}
```

我们声明一个空的`Counter`结构体，并为其实现`Middleware`特质。

`Middleware`特质有一个带有状态的类型参数。由于我们想要使用`State`结构体的计数器，我们将其设置为类型参数，但如果你想要创建与不同状态兼容的中间件，你需要添加类型参数到你的实现中，并添加必要的特质的实现，以便你可以将其导出到你的模块或包中。

`Middleware`特质包含三个方法。我们实现了所有这些方法：

+   当请求准备就绪并将发送到处理程序时调用`start`

+   在处理程序返回响应后调用`response`

+   当数据已发送到客户端时调用`finish`

我们为`response`和`finish`方法使用默认实现。

对于第一种方法，我们在`Response::Done`包装器中返回没有任何更改的响应。如果想要返回生成`HttpResponse`的`Future`，`Response`还有一个`Future`变体。

对于第二种方法，我们返回`Finished`枚举的`Done`变体。它还有一个可以包含 boxed `Future`对象的`Future`变体，该对象将在`finish`方法结束后运行。让我们来探讨一下在我们的实现中`start`方法是如何工作的。

在`Counter`中间件的`start`方法实现中，我们将计算所有传入的请求。为此，我们从`RefCell`获取当前计数器的值，加`1`，并用新值替换单元格。最后，该方法返回一个`Started::Done`值，通知您当前请求将在处理链的下一个处理器/中间件中重用。`Started`枚举还有其他变体：

+   如果你想立即返回响应，则应使用`Response`。

+   如果你想运行一个将返回响应的`Future`，则应使用`Future`。

现在，微服务已经准备好构建和运行。

# 构建和运行

要运行微服务，请使用`cargo run`命令。由于我们没有其他微服务用于处理器，我们可以使用`counter`方法来检查服务器和`Counter`中间件是否正常工作。尝试在浏览器中打开`http://127.0.0.1:8080/stats/counter`。它将在空白页面上显示一个`1`值。如果你刷新页面，你会看到一个`3`值。那是因为浏览器在主请求之后也会发送一个请求来获取`favicon.ico`文件。

# 使用数据库

`actix-web`与`actix` crate 结合的另一个好功能是使用数据库的能力。你还记得我们如何使用`SyncArbyter`来执行后台任务吗？这是一个实现数据库交互的好方法，因为异步数据库连接器不足，我们必须使用同步的。让我们为我们的上一个示例添加将响应缓存到 Redis 数据库的功能。

# 数据库交互 actor

我们首先实现一个与数据库交互的 actor。复制上一个示例并添加`redis` crate 到依赖项中：

```rs
redis = "0.9"
```

我们使用 Redis，因为它非常适合缓存，但我们也可以在内存中存储缓存的值。

创建一个`src/cache.rs`模块并添加以下依赖项：

```rs
use actix::prelude::*;
use failure::Error;
use futures::Future;
use redis::{Commands, Client, RedisError};
```

它添加了来自`redis` crate 的类型，我们已经在第七章与数据库的可靠集成中使用过，用于与 Redis 存储交互。

# Actors

我们的 actor 必须保持一个`Client`实例。我们不使用连接池，因为我们将为处理数据库的并行请求使用多个 actor。看看下面的结构：

```rs
pub struct CacheActor {
    client: Client,
    expiration: usize,
}
```

结构中还包含一个`expiration`字段，该字段持有**生存时间**（**TTL**）周期。这定义了 Redis 将保持值的时间。

在实现中添加一个`new`方法，该方法使用提供的地址字符串创建一个`Client`实例，并将`client`和`expiration`值添加到`CacheActor`结构中，如下所示：

```rs
impl CacheActor {
    pub fn new(addr: &str, expiration: usize) -> Self {
        let client = Client::open(addr).unwrap();
        Self { client, expiration }
    }
}
```

此外，我们还需要为`SyncContext`实现一个`Actor` trait，就像我们在第十章背景任务和线程池在微服务中中调整工作者时做的那样：

```rs
impl Actor for CacheActor {
    type Context = SyncContext<Self>;
}
```

现在，我们可以添加支持设置和获取缓存值的消息。

# Messages

要与`CacheActor`交互，我们必须添加两种类型的消息：设置值和获取值。

# 设置值消息

我们添加的第一个消息类型是`SetValue`，它提供了一对用于缓存的键和新的值。结构体有两个字段——`path`用作键，`content`持有值：

```rs
struct SetValue {
    pub path: String,
    pub content: Vec<u8>,
}
```

让我们为`SetValue`结构体实现一个`Message`特质，如果设置了值，则使用空单元类型，如果数据库连接有问题，则返回`RedisError`：

```rs
impl Message for SetValue {
    type Result = Result<(), RedisError>;
}
```

`CacheActor`支持接收`SetValue`消息。让我们使用`Handler`特质来实现这一点：

```rs
impl Handler<SetValue> for CacheActor {
    type Result = Result<(), RedisError>;

    fn handle(&mut self, msg: SetValue, _: &mut Self::Context) -> Self::Result {
        self.client.set_ex(msg.path, msg.content, self.expiration)
    }
}
```

我们使用存储在`CacheActor`中的`Client`实例通过`set_ex`方法调用执行 Redis 的`SETEX`命令。这个命令以秒为单位设置一个带有过期时间的值。如您所见，实现接近第七章可靠与数据库集成的数据库交互函数，但作为特定消息的`Handler`实现。这种代码结构更简单、更直观。

# 获取值消息

`GetValue`结构体代表一个通过键（或路径，在我们的情况下）从 Redis 提取值的消息。它只包含一个带有`path`值的字段：

```rs
struct GetValue {
    pub path: String,
}
```

我们还必须实现`Message`特质，但我们希望它返回一个可选的`Vec<u8>`值，如果 Redis 包含提供的键的值：

```rs
impl Message for GetValue {
    type Result = Result<Option<Vec<u8>>, RedisError>;
}
```

`CacheActor`还实现了`GetValue`消息类型的`Handler`特质，并调用`Client`的`get`方法通过 Redis 存储的`GET`命令提取存储中的值：

```rs
impl Handler<GetValue> for CacheActor {
    type Result = Result<Option<Vec<u8>>, RedisError>;

    fn handle(&mut self, msg: GetValue, _: &mut Self::Context) -> Self::Result {
        self.client.get(&msg.path)
    }
}
```

如您所见，演员和消息足够简单，但我们必须使用`Addr`值来与他们交互。这不是一个简洁的方法。我们将添加一个特殊类型，允许方法与`CacheActor`实例交互。

# 链接到演员

下面的结构体包装了`CacheActor`的地址：

```rs
#[derive(Clone)]
pub struct CacheLink {
    addr: Addr<CacheActor>,
}
```

构造函数只填充这个`addr`字段以一个`Addr`值：

```rs
impl CacheLink {
    pub fn new(addr: Addr<CacheActor>) -> Self {
        Self { addr }
    }
}
```

我们需要一个`CacheLink`包装结构体来添加获取缓存功能的方法，但需要隐藏实现细节和消息交互。首先，我们需要一个获取缓存值的方法：

```rs
pub fn get_value(&self, path: &str) -> Box<Future<Item = Option<Vec<u8>>, Error = Error>> {
    let msg = GetValue {
        path: path.to_owned(),
    };
    let fut = self.addr.send(msg)
        .from_err::<Error>()
        .and_then(|x| x.map_err(Error::from));
    Box::new(fut)
}
```

前面的函数创建了一个带有`path`的新`GetValue`消息，并将此消息发送到`CacheLink`中包含的`Addr`。之后，它等待结果。函数返回这个交互序列作为一个 boxed `Future`。

下一个方法以类似的方式实现——`set_value`方法通过向`CacheActor`发送`SetValue`消息来设置缓存中的新值：

```rs
pub fn set_value(&self, path: &str, value: &[u8]) -> Box<Future<Item = (), Error = Error>> {
    let msg = SetValue {
        path: path.to_owned(),
        content: value.to_owned(),
    };
    let fut = self.addr.send(msg)
        .from_err::<Error>()
        .and_then(|x| x.map_err(Error::from));
    Box::new(fut)
}
```

要组合一个消息，我们使用一个`path`和一个字节数组引用转换为`Vec<u8>`值。现在，我们可以在服务器实现中使用`CacheActor`和`CacheLink`。

# 使用数据库演员

在本章前面的示例中，我们使用了共享的`State`来提供对存储为`i64`的计数器的访问，并用`RefCell`包装。我们重用这个结构体，但添加一个`CacheLink`字段来使用与`CacheActor`的连接来获取或设置缓存值。添加此字段：

```rs
struct State {
    counter: RefCell<i64>,
    cache: CacheLink,
}
```

我们之前为`State`结构体推导了一个`Default`特质，但现在我们需要一个新的构造函数，因为我们必须提供一个带有缓存演员实际地址的`CacheLink`实例：

```rs
impl State {
    fn new(cache: CacheLink) -> Self {
        Self {
            counter: RefCell::default(),
            cache,
        }
    }
}
```

在大多数情况下，缓存是这样工作的——它试图从一个缓存中提取一个值；如果它存在且未过期，则将值返回给客户端。如果没有有效的值，我们需要获取一个新的值。在我们获取它之后，我们必须将其存储在缓存中以供将来使用。

在前面的例子中，我们经常使用一个接收来自另一个微服务的`Response`的`Future`实例。为了简化我们对缓存的用法，让我们向我们的`State`实现添加`cache`方法。此方法将任何提供的`future`包装在一个路径中，并尝试提取缓存的值。如果值不可用，它将获取一个新的值，之后，它接收存储复制的值以缓存，并将值返回给客户端。此方法将提供的`Future`值包装在另一个`Future`特质实现中。看看以下实现：

```rs
fn cache<F>(&self, path: &str, fut: F)
    -> impl Future<Item = Vec<u8>, Error = Error>
where
    F: Future<Item = Vec<u8>, Error = Error> + 'static,
{
    let link = self.cache.clone();
    let path = path.to_owned();
    link.get_value(&path)
        .from_err::<Error>()
        .and_then(move |opt| {
            if let Some(cached) = opt {
                debug!("Cached value used");
                boxed(future::ok(cached))
            } else {
                let res = fut.and_then(move |data| {
                    link.set_value(&path, &data)
                        .then(move |_| {
                            debug!("Cache updated");
                            future::ok::<_, Error>(data)
                        })
                        .from_err::<Error>()
                });
                boxed(res)
            }
        })
}
```

实现使用`State`实例来克隆`CacheLink`。我们必须使用克隆的`link`，因为我们必须将其移动到使用它的闭包中，以存储一个新值，如果我们需要获取它的话。

首先，我们调用`CacheLink`的`get_value`方法，并获取一个请求从缓存中获取值的`Future`。由于该方法返回`Option`，我们将使用`and_then`方法检查该值是否存在于缓存中，并将该值返回给客户端。如果值已过期或不可用，我们将通过执行提供的`Future`来获取它，如果返回新值，则使用链接调用`set_value`方法。

现在，我们可以使用`cache`方法来缓存前一个例子中`comments`处理程序返回的评论列表：

```rs
fn comments(req: HttpRequest<State>) -> FutureResponse<HttpResponse> {
    let fut = get_request("http://127.0.0.1:8003/list");
    let fut = req.state().cache("/list", fut)
        .map(|data| {
            HttpResponse::Ok().body(data)
        });
    Box::new(fut)
}
```

首先，我们创建一个`Future`，使用我们之前实现的`get_request`方法从另一个微服务获取值。之后，我们使用请求的`state`方法获取`State`的引用，通过传递`/list`路径调用`cache`方法，然后创建一个`Future`实例以获取新值。

我们已经实现了我们数据库演员的所有部分。我们仍然需要使用`SyncArbiter`启动一组缓存演员，并将返回的`Addr`值包装在`CacheLink`中：

```rs
let addr = SyncArbiter::start(3, || {
    CacheActor::new("redis://127.0.0.1:6379/", 10)
});
let cache = CacheLink::new(addr);
server::new(move || {
    let state = State::new(cache.clone());
    App::with_state(state)
    // remains part App building
})
```

现在，你可以构建服务器。它将每 10 秒返回`/api/list`请求的缓存值。

使用演员的另一个好处是 WebSocket。有了这个，我们可以通过作为演员实现的状态机添加有状态交互到我们的微服务中。让我们在下一节中看看这个。

# WebSocket

WebSocket 是一种全双工通信协议，通过 HTTP 工作。WebSockets 通常作为主要 HTTP 交互的扩展使用，可用于实时更新或通知。

演员模型非常适合实现 WebSocket 处理器，因为你可以将代码组合和隔离在一个地方：在演员的实现中。`actix-web`支持 WebSocket 协议，在本节中，我们将向我们的微服务添加通知功能。也许我们用`actix-web`实现的所有功能使我们的示例变得有些复杂，但为了演示目的，保持所有功能以展示如何将服务器与多个演员和任务结合在一起是非常重要的。

# 重复演员

我们必须向所有连接的客户端发送关于新评论的通知。为此，我们必须保留所有连接客户端的列表以向他们发送通知。我们可以在每次连接时更新`State`实例以将每个新客户端添加到其中，但我们将创建一个更优雅的解决方案，使用一个路由器将消息重新发送给多个订阅者。在这种情况下，订阅者或监听器将是处理传入 WebSocket 连接的演员。

# 演员

我们将添加一个演员来重新发送消息给其他演员。我们需要从`actix`crate 中获取一些基本类型，以及一个`HashSet`来保存演员的地址。导入`NewComment`结构体，我们将克隆并重新发送：

```rs
use actix::{Actor, Context, Handler, Message, Recipient};
use std::collections::HashSet;
use super::NewComment;
```

添加一个包含`Recipient`实例的`HashSet`类型的`listeners`字段的`RepeaterActor`结构体：

```rs
pub struct RepeaterActor {
    listeners: HashSet<Recipient<RepeaterUpdate>>,
}
```

你熟悉`Addr`类型，但我们之前还没有使用过`Recipient`。实际上，你可以使用`recipient`方法调用将任何`Addr`实例转换为`Recipient`。`Recipient`类型是一个只支持一种`Message`类型的地址。

添加一个构造函数来创建一个空的`HashSet`：

```rs
impl RepeaterActor {
    pub fn new() -> Self {
        Self {
            listeners: HashSet::new(),
        }
    }
}
```

接下来，为这个结构体实现`Actor`特质：

```rs
impl Actor for RepeaterActor {
    type Context = Context<Self>;
}
```

只需要一个标准的`Context`类型作为`Actor`的关联上下文类型就足够了，因为它可以异步工作。

现在，我们必须向这个演员类型添加消息。

# 消息

我们将支持两种类型的消息。第一种是更新消息，它将一个新评论从一个演员传输到另一个演员。第二种是控制消息，它向演员添加或删除监听器。

# 更新消息

我们将从更新消息开始。添加一个包装`NewComment`类型的`RepeaterUpdate`结构体：

```rs
#[derive(Clone)]
pub struct RepeaterUpdate(pub NewComment);
```

如你所见，我们还推导了`Clone`特质，因为我们需要克隆这条消息以将其重新发送给多个订阅者。`NewComment`现在也必须是可克隆的。

让我们为`RepeaterUpdate`结构体实现`Message`特质。我们将为`Result`关联类型使用一个空类型，因为我们不关心这些消息的投递：

```rs
impl Message for RepeaterUpdate {
    type Result = ();
}
```

现在，我们可以为`RepeaterUpdate`消息类型实现一个`Handler`：

```rs
impl Handler<RepeaterUpdate> for RepeaterActor {
    type Result = ();

    fn handle(&mut self, msg: RepeaterUpdate, _: &mut Self::Context) -> Self::Result {
        for listener in &self.listeners {
            listener.do_send(msg.clone()).ok();
        }
    }
}
```

处理器的算法很简单：它遍历所有监听器（实际上，存储为`Recipient`实例的监听器地址）并向它们发送克隆的消息。换句话说，这个演员接收一条消息，然后立即将其发送给所有已知的监听器。

# 控制消息

以下消息类型是订阅或取消订阅`RepeaterUpdate`消息所必需的。添加以下枚举：

```rs
pub enum RepeaterControl {
    Subscribe(Recipient<RepeaterUpdate>),
    Unsubscribe(Recipient<RepeaterUpdate>),
}
```

它有两个变体，内部具有相同的`Recipient<RepeaterUpdate>`类型。actor 将发送它们自己的`Recipient`地址以开始监听更新或停止关于新评论的任何通知。

为`RepeaterControl`结构实现`Message`特质，将其转换为`message`类型并使用一个空的`Result`关联类型：

```rs
impl Message for RepeaterControl {
    type Result = ();
}
```

现在，我们可以为`RepeaterControl`消息实现一个`Handler`特质：

```rs
impl Handler<RepeaterControl> for RepeaterActor {
    type Result = ();

    fn handle(&mut self, msg: RepeaterControl, _: &mut Self::Context) -> Self::Result {
        match msg {
            RepeaterControl::Subscribe(listener) => {
                self.listeners.insert(listener);
            }
            RepeaterControl::Unsubscribe(listener) => {
                self.listeners.remove(&listener);
            }
        }
    }
}
```

前一个处理器的实现也很简单：它在`Subscribe`消息变体上添加一个新的`Recipient`到监听器集合中，并在`Unsubscribe`消息中移除`Recipient`。

重发`NewComment`值的 actor 已经准备好了，现在我们可以开始实现一个处理 WebSocket 连接的 actor。

# 通知 actor

通知 actor 实际上是 WebSocket 连接的处理程序，但它只执行一个功能——将`NewComment`值序列化为 JSON 并发送给客户端。

由于我们需要一个 JSON 序列化器，将`serde_json`crate 添加到依赖项中：

```rs
serde_json = "1.0"
```

然后，添加`src/notify.rs`模块并开始实现 actor。

# 演员

通知 actor 更复杂，我们需要更多类型来实现它。让我们来看看它们：

```rs
use actix::{Actor, ActorContext, AsyncContext, Handler, Recipient, StreamHandler};
use actix_web::ws::{Message, ProtocolError, WebsocketContext};
use crate::repeater::{RepeaterControl, RepeaterUpdate};
use std::time::{Duration, Instant};
use super::State;
```

首先，我们开始使用`actix_web`crate 的`ws`模块。它包含必要的`WebsocketContext`，我们将将其用作`Actor`特质实现中的上下文值。我们还需要`Message`和`ProtocolError`类型来实现 WebSocket 流处理。我们还导入了`ActorContext`以停止`Context`实例的方法来断开与客户端的连接。我们导入了`AsyncContext`特质以获取上下文的地址并运行在时间间隔上执行的任务。我们还没有使用的一个新类型是`StreamHandler`。它是实现从`Stream`发送到`Actor`的值的处理所必需的。

你可以使用`Handler`或`StreamHandler`来处理相同类型的消息。哪个更可取？规则很简单：如果你的 actor 将处理大量消息，最好使用`StreamHandler`并将消息流连接为一个`Stream`到`Actor`。`actix`运行时会进行检查，如果它调用相同的`Handler`，你可能会收到警告。

添加我们将用于向客户端发送`ping`消息的常量：

```rs
const PING_INTERVAL: Duration = Duration::from_secs(20);
const PING_TIMEOUT: Duration = Duration::from_secs(60);
```

常量包含间隔和超时值。

我们将向客户端发送 ping，因为我们必须保持连接活跃，因为服务器通常为 WebSocket 连接设置默认的超时时间。例如，如果没有任何活动，`nginx`将在 60 秒后关闭连接。如果你使用默认配置的`nginx`作为 WebSocket 连接的代理，那么你的连接可能会被中断。浏览器不会发送 ping，只会对传入的 ping 发送 pong。服务器负责向通过浏览器连接的客户端发送 ping，以防止因超时而断开连接。

将以下`NotifyActor`结构体添加到代码中：

```rs
pub struct NotifyActor {
     last_ping: Instant,
     repeater: Recipient<RepeaterControl>,
}
```

此 actor 有一个`last_ping`字段，用于保存最新 ping 的时间戳。此外，actor 还持有`Recipient`地址以发送`RepeaterControl`消息。我们将使用构造函数提供`RepeaterActor`的地址：

```rs
impl NotifyActor {
    pub fn new(repeater: Recipient<RepeaterControl>) -> Self {
        Self {
            last_ping: Instant::now(),
            repeater,
        }
    }
}
```

现在，我们必须为`NotifyActor`结构体实现`Actor`特质：

```rs
impl Actor for NotifyActor {
    type Context = WebsocketContext<Self, State>;

    fn started(&mut self, ctx: &mut Self::Context) {
        let msg = RepeaterControl::Subscribe(ctx.address().recipient());
        self.repeater.do_send(msg).ok();
        ctx.run_interval(PING_INTERVAL, |act, ctx| {
            if Instant::now().duration_since(act.last_ping) > PING_TIMEOUT {
                ctx.stop();
                return;
            }
            ctx.ping("ping");
        });
    }

    fn stopped(&mut self, ctx: &mut Self::Context) {
        let msg = RepeaterControl::Unsubscribe(ctx.address().recipient());
        self.repeater.do_send(msg).ok();
    }
}
```

这是我们第一次需要重写空的`started`和`stopped`方法。在`started`方法实现中，我们将创建一个`Subscribe`消息并通过`Repeater`发送它。此外，我们添加一个将在`PING_INTERVAL`上执行的任务，并使用`WebsocketContext`的`ping`方法发送 ping 消息。如果客户端从未向我们响应，则`last_ping`字段不会更新。如果间隔大于我们的`PING_TIMEOUT`值，我们将使用上下文的`stop`方法中断连接。

`stopped`方法实现要简单得多：它准备一个与 actor 相同地址的`Unsubscribe`事件，并将其发送给`RepeaterActor`。

我们已经准备好了 actor 实现，现在我们必须添加消息和流的处理器。

# 处理器

首先，我们将实现`ws::Message`消息的`StreamHandler`实例：

```rs
impl StreamHandler<Message, ProtocolError> for NotifyActor {
    fn handle(&mut self, msg: Message, ctx: &mut Self::Context) {
        match msg {
            Message::Ping(msg) => {
                self.last_ping = Instant::now();
                ctx.pong(&msg);
            }
            Message::Pong(_) => {
                self.last_ping = Instant::now();
            }
            Message::Text(_) => { },
            Message::Binary(_) => { },
            Message::Close(_) => {
                ctx.stop();
            }
        }
    }
}
```

这是使用`actix-web`实现 WebSocket 协议交互的基本方法。我们将在稍后使用`ws::start`方法将 WebSocket 消息的`Stream`附加到此 actor。

`Message`类型有多个变体，反映了 RFC 6455（官方协议规范）中的 WebSocket 消息类型。我们使用`Ping`和`Pong`来更新 actor 结构体的`last_ping`字段，并使用`Close`来根据用户的要求停止连接。

我们必须实现的最后一个`Handler`允许我们接收`RepeaterUpdate`消息并将`NewComment`值发送给客户端：

```rs
impl Handler<RepeaterUpdate> for NotifyActor {
    type Result = ();

    fn handle(&mut self, msg: RepeaterUpdate, ctx: &mut Self::Context) -> Self::Result {
        let RepeaterUpdate(comment) = msg;
        if let Ok(data) = serde_json::to_string(&comment) {
            ctx.text(data);
        }
    }
}
```

实现会解构一个`RepeaterUpdate`消息以获取`NewComment`值，使用`serde_json`crate 将其序列化为 JSON，并通过`WebsocketContext`的`text`方法将其发送给客户端。

我们已经有了所有必要的 actor，所以让我们将它们与服务器连接起来。

# 为服务器添加 WebSocket 支持

由于我们将扩展上一节中的示例，我们将重用`State`结构体，但为稍后在`main`函数中创建的`Repeater`actor 添加一个`Addr`：

```rs
pub struct State {
     counter: RefCell<i64>,
     cache: CacheLink,
     repeater: Addr<RepeaterActor>,
 }
```

更新构造函数以填充`repeater`字段：

```rs
fn new(cache: CacheLink, repeater: Addr<RepeaterActor>) -> Self {
     Self {
         counter: RefCell::default(),
         cache,
         repeater,
     }
 }
```

现在，我们可以创建一个 `RepeaterActor`，将演员的地址设置为 `State`，并将其用作我们 `App` 的状态：

```rs
let repeater = RepeaterActor::new().start();

 server::new(move || {
     let state = State::new(cache.clone(), repeater.clone());
     App::with_state(state)
         .resource("/ws", |r| r.method(http::Method::GET).f(ws_connect))
         // other
 })
```

此外，我们还添加了一个处理 HTTP 请求的处理程序，该处理程序具有 `App` 的资源方法调用，并将 `ws_connect` 函数传递给它。让我们看看这个函数的实现：

```rs
fn ws_connect(req: &HttpRequest<State>) -> Result<HttpResponse, Error> {
     let repeater = req.state().repeater.clone().recipient();
     ws::start(req, NotifyActor::new(repeater))
 }
```

这将 `RepeaterActor` 的地址克隆成一个 `Recipient`，然后用于创建一个 `NotifyActor` 实例。要启动该演员实例，你必须使用 `ws::start` 方法，该方法使用当前的 `Request` 并为该演员启动 `WebsocketContext`。

剩下的工作是将一个 `NewComment` 发送到 `RepeaterActor`，它将重新发送到任何连接客户端的 `NotifyActor` 实例：

```rs
fn new_comment((req, params): (HttpRequest<State>, Form<AddComment>)) -> FutureResponse<HttpResponse> {
     let repeater = req.state().repeater.clone();
     let fut = req.identity()
         .ok_or(format_err!("not authorized").into())
         .into_future()
         .and_then(move |uid| {
             let new_comment = NewComment {
                 uid,
                 text: params.into_inner().text,
             };
             let update = RepeaterUpdate(new_comment.clone());
             repeater
                 .send(update)
                 .then(move |_| Ok(new_comment))
         })
         .and_then(move |params| {
             post_request::<_, ()>("http://127.0.0.1:8003/new_comment", params)
         })
         .then(move |_| {
             let res = HttpResponse::build_from(&req)
                 .status(StatusCode::FOUND)
                 .header(header::LOCATION, "/comments.html")
                 .finish();
             Ok(res)
         });
     Box::new(fut)
 }
```

我们扩展了当用户添加新评论时被调用的 `new_comment` 处理程序，并添加了一个额外的步骤来向重复器发送 `NewComment` 值。在任何情况下，我们都会忽略将此消息发送给演员的结果，并向另一个微服务发送 POST 请求。值得注意的是，即使没有发送到其他微服务，客户端也会收到关于新评论的通知，但你可以通过改变链中相应 `Future` 的顺序来改进它。

# 摘要

在本章中，我们介绍了使用 Actix 框架创建微服务。我们发现了如何创建和配置一个 `App` 实例，该实例描述了所有要使用的路由和中间件。之后，我们实现了所有返回 `Future` 实例的处理程序。在所有处理程序中，我们也使用 `ClientRequest` 向另一个微服务发送请求，并使用异步方法通过 futures 将响应返回给客户端。最后，我们探讨了如何为 `actix-web` 包创建自己的 `Middleware`。

在下一章中，我们将检查可扩展的微服务架构，并探讨如何实现微服务之间的松耦合。我们还将考虑使用消息代理以灵活和可管理的方式在大型应用程序的部分之间交换消息。
