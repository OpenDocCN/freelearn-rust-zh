# 8

# 构建 RESTful 服务

我们用 Rust 编写的待办事项应用在技术上是可以工作的。然而，我们还需要做一些改进。在本章中，我们将随着对**RESTful** **API**设计概念的探索来应用这些改进。

在本章中，我们将在请求击中视图之前最终拒绝未经授权的用户，通过评估我们系统的层次结构并重构我们处理请求的方式在整个请求生命周期中。然后，我们将使用这种身份验证来允许单个用户拥有他们自己的待办事项列表。最后，我们将记录我们的请求，以便我们可以调试我们的应用程序并深入了解我们的应用程序是如何运行的，在前端缓存数据以减少 API 调用。我们还将探索一些有用的概念，例如按命令执行代码和创建统一的接口来分割前端 URL 和后端 URL。

在本章中，我们将涵盖以下主题：

+   什么是 RESTful 服务？

+   映射我们的分层系统

+   构建统一接口

+   实现无状态

+   记录我们的服务器流量

+   缓存

+   按需代码

到本章结束时，我们将重构我们的 Rust 应用程序以支持 RESTful API 的原则。这意味着我们将绘制出 Rust 应用程序的层次结构，创建统一的 API 端点，记录应用程序中的请求，并在前端缓存结果。

# 技术要求

本章的代码可以在[`github.com/PacktPublishing/Rust-Web-Programming-2nd-Edition/tree/main/chapter08`](https://github.com/PacktPublishing/Rust-Web-Programming-2nd-Edition/tree/main/chapter08)找到

# 什么是 RESTful 服务？

`GET`（获取）、`PUT`（更新）、`POST`（创建）和`DELETE`（删除）我们的用户和待办事项。RESTful 方法的目的是通过重用可以管理和更新的组件来提高速度/性能、可靠性和可扩展性，这些组件不会影响系统。

你可能已经注意到，在 Rust 之前，慢速的高级语言似乎在 Web 开发中是一个明智的选择。这是因为它们编写起来更快、更安全。这是由于 Web 开发中数据处理速度瓶颈主要是网络连接速度。RESTful 设计旨在通过优化系统来提高速度，例如减少 API 调用，而不是仅仅关注算法速度。考虑到这一点，在本节中，我们将涵盖以下 RESTful 概念：

+   **分层系统**：这使我们能够添加额外的功能，例如授权，而无需更改接口。例如，如果我们必须在每个视图中检查**JSON Web Token**（JWT），那么这将是一大堆重复的代码，难以维护且容易出错。

+   **统一系统**：这简化并解耦了架构，使得应用的部分可以独立进化而不会相互冲突。

+   **无状态性**：这确保了我们的应用程序不会直接在服务器上保存任何东西。这对微服务和云计算有影响。

+   **日志记录**：这使我们能够窥视我们的应用程序并查看其运行情况，即使没有显示错误，也能暴露出不良行为。

+   **缓存**：这使我们能够在前端存储数据，以减少对后端 API 的 API 调用次数。

+   **按需代码**：这是我们的后端服务器在前端直接运行代码的地方。

我们将在下一节中探讨分层系统概念。

# 映射我们的分层系统

分层系统由具有不同功能单元的层组成。可以认为这些层是不同的服务器。这在微服务和大型系统中可能是正确的。当涉及到数据的不同层时，这也可能是正确的。在大型系统中，拥有经常被访问和更新的*热数据*和很少被访问的*冷数据*是有意义的。然而，虽然将层视为位于不同的服务器上很容易，但它们可以位于同一服务器上。我们可以用以下图表来映射我们的层：

![图 8.1 – 我们应用程序的层](img/Figure_8.1_B18722.jpg)

图 8.1 – 我们应用程序的层

如您所见，我们的应用程序遵循以下流程：

1.  首先，我们的*HTTP 处理器*通过监听我们在创建服务器时定义的端口来接受调用。

1.  然后，它通过*中间件*，这是通过在我们的应用程序上使用`wrap_fn`函数定义的。

1.  一旦完成，请求的 URL 就会被映射到正确的视图以及我们在`src/json_serialization/`目录中定义的模式。这些模式会被传递到在`src/views`目录中定义的资源（我们的视图）。

如果我们想更新或从数据库获取数据，我们使用 Diesel ORM 将这些请求映射。在这个阶段，我们已经定义了所有层来有效地管理数据流，除了我们的中间件。正如前一章所指出的，*第七章*，*管理用户会话*，我们已经通过实现`FromRequest`特质来为`JwToken`结构体实现了我们的中间件以进行身份验证。有了这个，我们可以看到我们可以使用`wrap_fn`或实现`FromRequest`特质来实现我们的中间件。你认为我们应该在什么时候使用`wrap_fn`或`FromRequest`特质？两者都有优点和缺点。如果我们想为特定的单个视图实现中间件，那么实现`FromRequest`特质是最好的选择。这是因为我们可以将实现`FromRequest`特质的结构体插入我们想要的视图中。身份验证是实现`FromRequest`特质的良好用例，因为我们想挑选和选择哪些端点需要身份验证。然而，如果我们想实施一个一揽子规则，我们最好在`wrap_fn`函数中实现视图的认证选择。在`wrap_fn`函数中实现我们的中间件意味着它适用于每个请求。

这的一个例子可能是我们不再支持所有端点的版本一。如果我们打算这样做，我们必须警告第三方用户我们不再支持 API 的版本一的决定。一旦我们的日期过去，我们必须提供一个有用的消息，说明我们不再支持版本一。在我们开始工作于我们的中间件层之前，我们必须在`main.rs`文件顶部定义以下导入：

```rs
use actix_web::{App, HttpServer, HttpResponse};
use actix_service::Service;
use futures::future::{ok, Either};
use actix_cors::Cors;
```

为了确保我们知道进入的请求是针对一个`v1`端点的，我们必须定义一个标志，我们可以在稍后决定是否处理请求或拒绝它时进行检查。我们可以通过在我们的`main.rs`文件中使用以下代码来实现这一点：

```rs
.wrap_fn(|req, srv|{
    let passed: bool;
    if req.path().contains("/v1/") {
        passed = false;
    } else {
        passed = true;
    }
. . .
```

从前面的代码中，我们可以看到我们声明有一个名为`passed`的布尔值。如果`v1`不在 URL 中，则将其设置为`true`。如果`v1`存在于 URL 中，则`passed`被设置为`false`。

现在我们已经定义了一个标志，我们可以用它来决定请求会发生什么。在我们这样做之前，我们必须注意`wrap_fn`的最后几行，如以下代码块所示：

```rs
let future = srv.call(req);
async {
    let result = fut.await?;
    Ok(result)
}
```

我们正在等待调用完成，然后返回名为`result`的变量作为结果。在我们的`v1` API 调用阻塞之后，我们必须检查请求是否通过。如果它通过了，我们就运行前面的代码。然而，如果请求失败，我们必须绕过这一点并定义另一个 future，它只是响应。

从字面上看，这似乎很简单。两者都将返回相同的结果，即响应。然而，Rust 不会编译。它将基于不兼容的类型抛出一个错误。这是因为`async`块的行为类似于闭包。这意味着每个`async`块都是其自己的类型。这可能会令人沮丧，并且由于这个细微的细节，它可能导致开发者花费数小时试图让两个未来相互配合。

幸运的是，在 futures crate 中有一个枚举可以为我们解决这个问题。`Either`枚举将具有相同关联类型的两个不同的未来、流或汇合并入一个单一类型。这使得我们能够匹配`passed`标志，并使用以下代码触发并返回适当的过程：

```rs
    let end_result;
    if passed == true {
        end_result = Either::Left(srv.call(req))
    }
    else {
        let resp = HttpResponse::NotImplemented().body(
            "v1 API is no longer supported"
            );
        end_result = Either::Right(
            ok(req.into_response(resp)
                  .map_into_boxed_body())
        )
    }
    async move {
        let result = end_result.await?;
        Ok(result)
    }
}).configure(views::views_factory).wrap(cors);
```

从前面的代码中，我们可以看到我们根据`passed`标志将`end_result`分配为视图调用，或者直接返回一个未授权的响应。然后我们在`wrap_fn`的末尾返回这个结果。了解如何使用`Either`枚举是一个很有用的技巧，当你需要代码在两个不同的未来之间进行选择时，这将为你节省数小时。

要检查我们是否正在阻止`v1`，我们可以调用一个简单的`get`请求，如下面的图所示：

![图 8.2 – 阻塞 v1 的响应](img/Figure_8.2_B18722.jpg)

图 8.2 – 阻塞 v1 的响应

我们可以看到我们的 API 调用被一个有用的消息阻止。如果我们通过 Postman 进行 API 调用，我们将看到我们得到一个`501 Not Implemented`错误，如下面的图所示：

![图 8.3 – 阻塞 v1 的 Postman 响应](img/Figure_8.3_B18722.jpg)

图 8.3 – 阻塞 v1 的 Postman 响应

我们可能希望在将来添加更多获取项的资源。这可能会引起潜在的问题，因为一些视图可能会与应用程序视图发生冲突。例如，我们的待办事项项 API 视图只有前缀`item`。获取所有项需要`v1/item/get`端点。

可以合理地开发一个视图，用于以后通过`v1/item/get/{id}`端点详细查看待办事项并进行编辑。然而，这增加了前端应用程序视图和后端 API 调用之间发生冲突的风险。为了防止这种情况，我们必须确保我们的 API 具有统一的接口。

# 构建统一的接口

具有统一的接口意味着我们的资源可以通过 URL 唯一识别。这解耦了后端端点和前端视图，使得我们的应用可以扩展而不会在前端视图和后端端点之间发生冲突。我们使用版本标签解耦了后端和前端。当一个 URL 端点包含版本标签，如`v1`或`v2`，我们知道这个调用正在击中后端 Rust 服务器。当我们开发我们的 Rust 服务器时，我们可能想要在 API 调用的新版本上工作。然而，我们不想允许用户访问正在开发中的版本。为了使实时用户能够访问一个版本，同时我们在测试服务器上部署另一个版本，我们需要动态地为服务器定义 API 版本。根据你在本书中迄今为止获得的知识，你可以在定义服务器之前，在`main.rs`文件的`main`函数中简单地定义版本号并加载它。然而，我们必须为每个请求读取`config.yml`配置文件。记住，当我们设置数据库连接池时，我们只从`config.yml`文件中读取一次连接字符串，这意味着它在程序的整个生命周期中都是存在的。我们希望定义版本一次，然后在程序的整个生命周期中引用它。直观上，你可能会想在定义服务器之前，在`main.rs`文件的`main`函数中定义版本，然后在`wrap_fn`内部访问版本的定义，如下面的示例所示：

```rs
let outcome = "test".to_owned();
HttpServer::new(|| {
    . . .
    let app = App::new()
        .wrap_fn(|req, srv|{
            println!("{}", outcome);
            . . .
        }
    . . .
```

然而，如果我们尝试编译前面的代码，它将失败，因为`outcome`变量的生命周期不够长。我们可以通过以下代码将`outcome`变量转换为常量：

```rs
const OUTCOME: &str = "test";
HttpServer::new(|| {
    . . .
    let app = App::new()
        .wrap_fn(|req, srv|{
            println!("{}", outcome);
            . . .
        }
    . . .
```

上述代码将无任何生命周期问题地运行。然而，如果我们加载我们的版本，我们必须从文件中读取它。在 Rust 中，如果我们从文件中读取，我们不知道从文件中读取的变量的大小。因此，我们从文件中读取的变量将是一个字符串。这里的问题是分配字符串不是在编译时可以计算的事情。因此，我们将不得不直接将版本写入我们的`main.rs`文件。我们可以通过使用`build`文件来实现这一点。

注意

我们在这个问题中利用`build`文件来教授`build`文件的概念，这样你如果需要的话就可以使用它们。没有任何阻止你在代码中硬编码常量的因素。

这是在 Rust 应用程序运行之前单个 Rust 文件运行的阶段。当我们在编译主 Rust 应用程序时，这个`build`文件将自动运行。我们可以在`Cargo.toml`文件的`build dependencies`部分定义运行我们的`build`文件所需的依赖项，以下是一段代码：

```rs
[package]
name = "web_app"
version = "0.1.0"
edition = "2021"
build = "build.rs"
[build-dependencies]
serde_yaml = "0.8.23"
serde = { version = "1.0.136", features = ["derive"] }
```

这意味着我们的 `build` Rust 文件定义在应用程序的根目录中的 `build.rs` 文件中。然后，我们在 `[build-dependencies]` 部分中定义构建阶段所需的依赖项。现在，我们的依赖项已经定义，我们的 `build.rs` 文件可以采取以下形式：

```rs
use std::fs::File;
use std::io::Write;
use std::collections::HashMap;
use serde_yaml;
fn main() {
  let file =
      std::fs::File::open("./build_config.yml").unwrap();
  let map: HashMap<String, serde_yaml::Value> =
      serde_yaml::from_reader(file).unwrap();
  let version =
      map.get("ALLOWED_VERSION").unwrap().as_str()
          .unwrap();
  let mut f =
      File::create("./src/output_data.txt").unwrap();
  write!(f, "{}", version).unwrap();
}
```

在这里，我们可以看到我们需要从 YAML 文件中导入我们需要读取的内容，并将其写入标准文本文件。然后，我们将打开一个 `build_config.yml` 文件，该文件位于 Web 应用程序的根目录中，与 `config.yml` 文件相邻。然后，我们将从 `build_config.yml` 文件中提取 `ALLOWED_VERSION` 并将其写入文本文件。现在我们已经定义了构建过程以及从 `build_config.yml` 文件中需要的内容，我们的 `build_config.yml` 文件将必须采取以下形式：

```rs
ALLOWED_VERSION: v1
```

现在我们已经为我们的构建定义了一切，我们可以通过我们在 `build.rs` 文件中写入的文件引入一个 `const` 实例来表示我们的版本。为此，我们的 `main.rs` 文件需要一些更改。首先，我们使用以下代码定义 `const`：

```rs
const ALLOWED_VERSION: &'static str = include_str!(
    "./output_data.txt");
HttpServer::new(|| {
. . .
```

然后，我们使用以下代码认为请求通过，如果版本是被允许的：

```rs
HttpServer::new(|| {
. . .
let app = App::new()
.wrap_fn(|req, srv|{
    let passed: bool;
    if *&req.path().contains(&format!("/{}/",
                             ALLOWED_VERSION)) {
        passed = true;
    } else {
        passed = false;
    }
```

然后，我们使用以下代码定义错误响应和服务调用：

```rs
. . .
let end_result = match passed {
    true => {
        Either::Left(srv.call(req))
    },
    false => {
        let resp = HttpResponse::NotImplemented()
            .body(format!("only {} API is supported",
                ALLOWED_VERSION));
        Either::Right(
            ok(req.into_response(resp).map_into_boxed_body())
        )
    }
};
. . .
```

我们现在可以构建并运行我们的应用程序，并支持特定版本。如果我们运行我们的应用程序并发出一个 `v2` 请求，我们会得到以下响应：

![图 8.4 – 邮递员对被阻止 v2 的响应](img/Figure_8.4_B18722.jpg)

图 8.4 – 邮递员对被阻止 v2 的响应

我们可以看到我们的版本保护器现在正在工作。这也意味着我们必须使用 React 应用程序来访问前端视图，或者你可以在前端 API 端点中添加一个 `v1`。

现在，如果我们运行我们的应用程序，我们可以看到我们的前端与新的端点一起工作。有了这个，我们离为我们的应用程序开发 RESTful API 又近了一步。然而，我们仍然有一些明显的不足。目前，我们可以创建另一个用户并使用该用户登录。在下一节中，我们将探讨如何以无状态的方式管理我们的用户状态。

# 实现无状态

无状态是指服务器不存储任何关于客户端会话的信息。这里的优势是显而易见的。它使我们的应用程序能够更容易地扩展，因为我们通过在客户端而不是服务器端存储会话信息来释放服务器端的资源。

它还使我们能够更灵活地选择计算方法。例如，假设我们的应用程序在受欢迎程度上爆炸式增长。结果，我们可能希望在我们的两个计算实例或服务器上启动我们的应用程序，并让负载均衡器以平衡的方式将流量引导到这两个实例。如果信息存储在服务器上，用户将会有不一致的体验。

他们可能在某个计算实例上更新会话状态，但当他们再次请求时，他们可能会遇到另一个具有过时数据的计算实例。考虑到这一点，仅仅通过在客户端存储所有内容来实现无状态是不够的。如果我们的数据库不依赖于我们应用程序的计算实例，我们也可以在这个数据库上存储我们的数据，如下面的图所示：

![图 8.5 – 我们的无状态方法](img/Figure_8.5_B18722.jpg)

图 8.5 – 我们的无状态方法

如您所见，我们的应用程序已经是无状态的。我们在前端将用户 ID 存储在 JWT 中，并将我们的用户数据模型和待办事项存储在我们的 PostgreSQL 数据库中。然而，我们可能想在应用程序中存储 Rust 结构体。例如，我们可以构建一个结构体来计算击中服务器的请求数量。根据*图 8**.5*，我们不能仅仅在服务器上本地保存我们的结构体。相反，我们将我们的结构体存储在 Redis 中，按照以下图中的过程执行：

![图 8.6 – 在 Redis 中保存结构体的步骤](img/Figure_8.6_B18722.jpg)

图 8.6 – 在 Redis 中保存结构体的步骤

PostgreSQL 和 Redis 之间的区别

Redis 是一个数据库，但它与 PostgreSQL 数据库不同。Redis 更接近于键值存储。由于数据存储在内存中，Redis 也很快。虽然 Redis 在管理表及其相互关系方面不如 PostgreSQL 完整，但它确实具有优势。Redis 支持有用的数据结构，如列表、集合、散列、队列和通道。您还可以为插入 Redis 的数据设置过期时间。您也不需要使用 Redis 处理数据迁移。这使得 Redis 成为缓存您需要快速访问但不太关心持久性的数据的理想数据库。对于通道和队列，Redis 也是促进订阅者和发布者之间通信的理想选择。

我们可以通过执行以下步骤来实现*图 8**.6*中的过程：

1.  定义 Redis 服务的 Docker。

1.  更新 Rust 依赖项。

1.  更新 Redis 连接的配置文件。

1.  构建一个可以保存和加载到 Redis 数据库中的计数器结构体。

1.  为每个请求实现计数器。

让我们详细回顾每个步骤：

1.  当涉及到启动 Redis Docker 服务时，我们需要使用标准 Redis 容器和标准端口。在我们实现了 Redis 服务之后，我们的`docker-compose.yml`文件应该处于以下状态：

    ```rs
    version: "3.7"
    ```

    ```rs
    services:
    ```

    ```rs
      postgres:
    ```

    ```rs
        container_name: 'to-do-postgres'
    ```

    ```rs
        image: 'postgres:11.2'
    ```

    ```rs
        restart: always
    ```

    ```rs
        ports:
    ```

    ```rs
          - '5433:5432'
    ```

    ```rs
        environment:
    ```

    ```rs
          - 'POSTGRES_USER=username'
    ```

    ```rs
          - 'POSTGRES_DB=to_do'
    ```

    ```rs
          - 'POSTGRES_PASSWORD=password'
    ```

    ```rs
      redis:
    ```

    ```rs
          container_name: 'to-do-redis'
    ```

    ```rs
          image: 'redis:5.0.5'
    ```

    ```rs
          ports:
    ```

    ```rs
            - '6379:6379'
    ```

我们可以看到，我们现在已经在本地机器上运行了 Redis 服务和数据库服务。现在 Redis 可以运行，我们需要在下一步更新我们的依赖项。

1.  回顾*图 8**.6*，在将数据插入 Redis 之前，我们需要将 Rust 结构体序列化为字节。考虑到这些步骤，我们需要在`Cargo.toml`文件中添加以下依赖项：

    ```rs
    [dependencies]
    ```

    ```rs
    . . .
    ```

    ```rs
    redis = "0.21.5"
    ```

我们正在使用`redis`crate 连接到 Redis 数据库。现在我们的依赖项已经定义，我们可以开始定义我们的配置文件。

1.  当涉及到我们的`config.yml`文件时，我们必须添加 Redis 数据库连接的 URL。在本书的这个时间点，我们的`config.yml`文件应该具有以下形式：

    ```rs
    DB_URL: postgres://username:password@localhost:5433/to_do
    ```

    ```rs
    SECRET_KEY: secret
    ```

    ```rs
    EXPIRE_MINUTES: 120
    ```

    ```rs
    REDIS_URL: redis://127.0.0.1/
    ```

我们还没有为`REDIS_URL`参数添加端口号。这是因为我们正在使用 Redis 服务的标准端口，即`6379`，所以我们不需要定义端口。我们现在有了所有数据，可以定义一个可以连接到 Redis 的结构体，我们将在下一步中这样做。

1.  我们将在`src/counter.rs`文件中定义我们的`Counter`结构体。首先，我们必须导入以下内容：

    ```rs
    use serde::{Deserialize, Serialize};
    ```

    ```rs
    use crate::config::Config;
    ```

1.  我们将使用`Config`实例来获取 Redis URL，并使用`Deserialize`和`Serialize`特质来启用转换为字节。我们的`Counter`结构体具有以下形式：

    ```rs
    #[derive(Serialize, Deserialize, Debug)]
    ```

    ```rs
    pub struct Counter {
    ```

    ```rs
        pub count: i32
    ```

    ```rs
    }
    ```

1.  现在我们已经定义了具有所有特质的`Counter`结构体，我们需要定义以下代码中所需的函数：

    ```rs
    impl Counter {
    ```

    ```rs
        fn get_redis_url() -> String {
    ```

    ```rs
            . . .
    ```

    ```rs
        }
    ```

    ```rs
        pub fn save(self) {
    ```

    ```rs
            . . .
    ```

    ```rs
        }
    ```

    ```rs
        pub fn load() -> Counter {
    ```

    ```rs
            . . .
    ```

    ```rs
        }
    ```

    ```rs
    }
    ```

1.  在定义了前面的函数之后，我们可以将我们的`Counter`结构体加载和保存到 Redis 数据库中。当涉及到构建我们的`get_redis_url`函数时，它应该具有以下形式：

    ```rs
    fn get_redis_url() -> String {
    ```

    ```rs
        let config = Config::new();
    ```

    ```rs
        config.map.get("REDIS_URL")
    ```

    ```rs
                  .unwrap().as_str()
    ```

    ```rs
                  .unwrap().to_owned()
    ```

    ```rs
    }
    ```

1.  现在我们有了 Redis URL，我们可以使用以下代码保存我们的`Counter`结构体：

    ```rs
    pub fn save(self) -> Result<(), redis::RedisError> {
    ```

    ```rs
        let serialized = serde_yaml::to_vec(&self).unwrap();
    ```

    ```rs
        let client = match redis::Client::open(
    ```

    ```rs
                         Counter::get_redis_url()) {
    ```

    ```rs
            Ok(client) => client,
    ```

    ```rs
            Err(error) => return Err(error)
    ```

    ```rs
        };
    ```

    ```rs
        let mut con = match client.get_connection() {
    ```

    ```rs
            Ok(con) => con,
    ```

    ```rs
            Err(error) => return Err(error)
    ```

    ```rs
        };
    ```

    ```rs
        match redis::cmd("SET").arg("COUNTER")
    ```

    ```rs
                               .arg(serialized)
    ```

    ```rs
                               .query::<Vec<u8>>(&mut con) {
    ```

    ```rs
            Ok(_) => Ok(()),
    ```

    ```rs
            Err(error) => Err(error)
    ```

    ```rs
        }
    ```

    ```rs
    }
    ```

1.  在这里，我们可以看到我们可以将我们的`Counter`结构体序列化为`Vec<u8>`。然后我们将定义 Redis 客户端，并在键`"COUNTER"`下插入我们的序列化`Counter`结构体。然而，Redis 还有更多功能，但你可以通过将 Redis 视为一个大型的可扩展内存哈希表来利用 Redis 本章。带着哈希表的概念，你认为我们如何从 Redis 数据库中获取`Counter`结构体？你可能已经猜到了；我们使用带有`"COUNTER"`键的`GET`命令，然后使用以下代码反序列化：

    ```rs
    pub fn load() -> Result<Counter, redis::RedisError> {
    ```

    ```rs
        let client = match redis::Client::open(
    ```

    ```rs
                         Counter::get_redis_url()){
    ```

    ```rs
            Ok(client) => client,
    ```

    ```rs
            Err(error) => return Err(error)
    ```

    ```rs
        };
    ```

    ```rs
        let mut con = match client.get_connection() {
    ```

    ```rs
            Ok(con) => con,
    ```

    ```rs
            Err(error) => return Err(error)
    ```

    ```rs
        };
    ```

    ```rs
        let byte_data: Vec<u8> = match redis::cmd("GET")
    ```

    ```rs
                                     .arg("COUNTER")
    ```

    ```rs
                                     .query(&mut con) {
    ```

    ```rs
            Ok(data) => data,
    ```

    ```rs
            Err(error) => return Err(error)
    ```

    ```rs
        };
    ```

    ```rs
        Ok(serde_yaml::from_slice(&byte_data).unwrap())
    ```

    ```rs
    }
    ```

我们现在已经定义了`Counter`结构体。我们已经在`main.rs`文件中实现了所有内容。

1.  当每次接收到请求时增加计数，我们需要在`main.rs`文件中执行以下代码：

    ```rs
    . . .
    ```

    ```rs
    mod counter;
    ```

    ```rs
    . . .
    ```

    ```rs
    #[actix_web::main]
    ```

    ```rs
    async fn main() -> std::io::Result<()> {
    ```

    ```rs
        . . .
    ```

    ```rs
        let site_counter = counter::Counter{count: 0};
    ```

    ```rs
        site_counter.save();
    ```

    ```rs
        HttpServer::new(|| {
    ```

    ```rs
            . . .
    ```

    ```rs
            let app = App::new()
    ```

    ```rs
                .wrap_fn(|req, srv|{
    ```

    ```rs
                    let passed: bool;
    ```

    ```rs
                    let mut site_counter = counter::
    ```

    ```rs
                                           Counter::load()
    ```

    ```rs
                                           .unwrap();
    ```

    ```rs
                    site_counter.count += 1;
    ```

    ```rs
                    println!("{:?}", &site_counter);
    ```

    ```rs
                    site_counter.save();
    ```

    ```rs
                    . . .
    ```

1.  在这里，我们可以看到我们定义了`counter`模块。在我们启动服务器之前，我们需要创建新的`Counter`结构体并将其插入 Redis。然后我们从 Redis 获取`Counter`，增加计数，然后为每个请求保存它。

现在我们运行服务器时，我们可以看到每次我们用请求击中我们的服务器时，计数器都在增加。我们的打印输出应该如下所示：

```rs
Counter { count: 1 }
Counter { count: 2 }
Counter { count: 3 }
Counter { count: 4 }
```

现在我们已经集成了另一个存储选项，我们的应用程序本质上按我们想要的方式运行。如果我们现在想要发布我们的应用程序，实际上没有什么能阻止我们使用 Docker 配置构建，并在带有数据库和**NGINX**的服务器上部署它。

并发问题

如果两个服务器同时请求计数器，可能会错过请求。计数器示例被探索以展示如何将序列化结构存储在 Redis 中。如果您需要在 Redis 数据库中实现一个简单的计数器并且并发是一个关注点，建议您使用`INCR`命令。`INCR`命令在 Redis 数据库中增加所选键下的数字，并返回新的增加后的数字。鉴于计数器在 Redis 数据库中增加，我们已经降低了并发问题的风险。

然而，我们总是可以添加一些内容。在下一节中，我们将研究日志请求。

# 记录我们的服务器流量

到目前为止，我们的应用程序没有记录任何内容。这不会直接影响应用程序的运行。然而，日志记录有一些优点。日志记录使我们能够调试我们的应用程序。

目前，由于我们正在本地开发，可能感觉日志记录并不是真的必要。然而，在生产环境中，有许多原因可能导致应用程序失败，包括 Docker 容器编排问题。记录已发生的过程的日志可以帮助我们定位错误。我们还可以使用日志来查看边缘情况和错误何时出现，以便监控应用程序的整体健康状况。在日志记录方面，我们可以构建四种类型的日志：

+   **信息性（info）**: 这是一种通用日志记录。如果我们想跟踪一般过程及其进展，我们使用这种类型的日志。使用示例包括启动和停止服务器，以及记录我们想要监控的某些检查点，例如 HTTP 请求。

+   **冗长**: 这类信息类似于前一点中定义的类型。然而，它提供了更细粒度的信息，以告知我们过程的更详细流程。这种类型的日志主要用于调试目的，并且在生产设置中通常应避免使用。

+   **警告**: 我们在记录一个失败的过程且不应被忽略时使用此类型。然而，我们可以使用此代替引发错误，因为我们不希望服务中断或用户意识到具体的错误。日志本身是为了让我们意识到问题，以便我们可以采取行动。如调用另一个服务器失败等问题适合此类。

+   **错误**: 这是指由于错误而中断过程的地方，我们需要尽快解决这个问题。我们还需要通知用户交易没有完成。一个很好的例子是连接或向数据库插入数据失败。如果发生这种情况，交易发生的记录将不存在，并且无法事后解决。然而，应该注意的是，过程可以继续运行。

如果出现关于服务器无法发送电子邮件、连接到另一个服务器以调度产品进行发货等问题警告。一旦我们解决了问题，我们可以在这一时间段内回溯性地进行数据库调用，并使用正确的信息调用服务器。

在最坏的情况下，会有延迟。由于服务器在订单甚至进入数据库之前就被错误中断，我们将无法进行数据库调用。考虑到这一点，很明显为什么错误日志记录非常重要，因为用户需要被告知存在问题，并且他们的交易没有完成，这会促使他们稍后再次尝试。

我们可以考虑在错误日志中包含足够的信息，以便在问题解决后回溯性地更新数据库并完成剩余的过程，从而消除通知用户的需求。虽然这很有吸引力，但我们必须考虑两件事。日志数据通常是未结构化的。

进入日志的内容没有质量控制。因此，一旦我们最终成功将日志数据转换成正确的格式，仍然有可能损坏的数据会进入数据库。

第二个问题是日志不被认为是安全的。它们被复制并发送给其他开发人员在危机中，并且可以被插入到其他管道和网站上，例如 *Bugsnag*，以监控日志。考虑到日志的性质，在日志中包含任何可识别的信息都不是一个好的做法。

现在我们已经了解了日志记录的用途，我们可以开始配置我们自己的记录器。当涉及到日志记录时，我们将使用 Actix-web 记录器。这让我们在配置和良好地与我们的 Actix 服务器一起工作的情况下，有灵活性来记录什么。为了构建我们的记录器，我们必须在`Cargo.toml`文件中定义一个新的 crate，以下代码：

```rs
[dependencies]
. . .
env_logger = "0.9.0"
```

这使我们能够使用环境变量来配置记录器。现在我们可以专注于`main.rs`，因为这是我们的记录器被配置和使用的地方。首先，我们将使用以下代码导入我们的记录器：

```rs
use actix_web::{. . ., middleware::Logger};
```

使用这个导入，我们可以在`main`函数中定义我们的记录器，以下代码：

```rs
. . .
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    . . .
    env_logger::init_from_env(env_logger::Env::new()
                              .default_filter_or("info"));
. . .
```

在这里，我们声明我们的记录器将信息记录到 info 流中。记录器配置完成后，我们可以使用以下代码将记录器包装到我们的服务器上：

```rs
. . .
        async move {
            let result = end_result.await?;
            Ok(result)
        }
}).configure(views::views_factory).wrap(cors)
    .wrap(Logger::new("%a %{User-Agent}i %r %s %D"));
return app
. . .
```

我们可以在我们的记录器中看到我们传递了`"&a %{User-Agent}I %r %s %D"`字符串。这个字符串被记录器解释，告诉它们要记录什么。Actix 记录器可以接受以下输入：

+   `%%`: 百分号

+   `%a`: 远程 IP 地址（如果使用反向代理，则为代理的 IP 地址）

+   `%t`: 请求开始处理的时间

+   `%P`: 服务请求的子进程的进程 ID

+   `%r`: 请求的第一行

+   `%s`: 响应状态码

+   `%b`: 响应的大小（包括 HTTP 头部的字节数）

+   `%T`：处理请求所需的时间，以秒为单位，带有小数点后六位的浮点格式

+   `%D`：处理请求所需的时间，以毫秒为单位

+   `%{``FOO}i`：`request.headers['FOO']`

+   `%{``FOO}o`：`response.headers['FOO']`

+   `%{``FOO}e`：`os.environ['FOO']`

基于这些输入，我们可以计算出我们将记录远程 IP 地址、用户代理、请求端点和处理请求所需的时间。我们将为我们的 Rust 服务器上的每个请求都这样做。以日志记录启动我们的 Rust 服务器会给我们以下输出：

```rs
[2022-05-25T17:22:32Z INFO  actix_server::builder] Starting 8 workers
[2022-05-25T17:22:32Z INFO  actix_server::server] Actix runtime found; starting in Actix runtime
```

在这里，我们可以看到服务器启动时自动记录了工作进程的数量。然后，如果我们启动我们的前端，我们应该被提示登录，因为令牌现在应该已经过期了。一个完整的标准请求日志应该看起来像以下输出：

```rs
[2022-05-25T17:14:56Z INFO  actix_web::middleware::logger]
127.0.0.1 Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)
AppleWebKit/537.36 (KHTML, like Gecko)
Chrome/101.0.4951.64 Safari/537.36
GET /v1/item/get HTTP/1.1 401 9.466000
```

我们可以看到时间、这是一个`INFO`级别的日志以及哪个记录器记录了它。我们还可以看到我的 IP 地址，因为我是在本地运行我的应用程序，我的电脑/浏览器详情以及带有`401`响应代码的 API 调用。如果我们从请求日志中删除除方法、API 端点、响应代码和响应时间之外的所有内容，我们的登录提示将看起来像以下：

```rs
GET /v1/item/get HTTP/1.1 401 9.466000
OPTIONS /v1/auth/login HTTP/1.1 200 0.254000
POST /v1/auth/login HTTP/1.1 200 1405.585000
OPTIONS /v1/item/get HTTP/1.1 200 0.082000
GET /v1/item/get HTTP/1.1 200 15.470000
```

我们可以看到我在获取项目时失败了，得到了一个未授权的响应。然后我登录并从登录中得到了一个`OK`响应。在这里，我们可以看到一个`OPTIONS`和`POST`方法。`OPTIONS`方法是为了我们的 CORS，这就是为什么`OPTIONS`调用处理时间只是其他 API 请求的一小部分。我们可以看到我们随后得到了我们的项目，然后渲染到页面上。然而，我们可以看到在刷新页面时以下日志中发生的事情：

```rs
OPTIONS /v1/item/get HTTP/1.1 200 0.251000
OPTIONS /v1/item/get HTTP/1.1 200 0.367000
GET /v1/item/get HTTP/1.1 200 88.317000
GET /v1/item/get HTTP/1.1 200 9.523000
```

我们可以看到有两个针对项目的`GET`请求。然而，我们没有更改数据库中的待办事项。这不是错误，但这是浪费的。为了优化这一点，我们可以在下一节利用缓存的 REST 约束。

# 缓存

缓存是我们将数据存储在前端以便重复使用的地方。这使我们能够减少对后端 API 调用的次数并降低延迟。因为好处如此明显，所以可能会诱使我们缓存一切。然而，还有一些事情需要考虑。

并发是一个明显的问题。数据可能会过时，导致发送错误信息到后端时产生混淆和数据损坏。也存在安全方面的担忧。如果一个用户注销，另一个用户在同一台电脑上登录，那么第二个用户可能会访问第一个用户的物品。因此，必须有一些检查措施。正确的用户需要登录，数据需要带时间戳，这样如果缓存的数据在某个时间段后被访问，就会发起一个`GET`请求来刷新数据。

我们的应用程序相当安全。除非我们登录，否则我们无法访问任何内容。我们可以在应用程序中缓存的主体过程是`GET`项目调用。所有其他编辑后端项目列表状态的调用都会返回更新后的项目。考虑到这一点，我们的缓存机制看起来如下：

![图 8.7 – 我们的缓存方法](img/Figure_8.7_B18722.jpg)

图 8.7 – 我们的缓存方法

图表中的循环可以在刷新页面时执行任意多次。然而，这可能不是一个好主意。如果一个用户在厨房使用手机登录我们的应用程序来更新列表，那么当用户回到电脑上工作并刷新电脑页面时，用户会遇到问题。这个缓存系统会暴露用户给过时的数据，这些数据将被发送到后端。我们可以通过引用时间戳来降低这种情况发生的风险。当时间戳超过截止时间时，我们将在用户刷新页面时发起另一个 API 调用以刷新数据。

当涉及到我们的缓存逻辑时，它将全部在`front_end/src/App.js`文件下的`getItems`函数中实现。我们的`getItems`函数将具有以下布局：

```rs
getItems() {
  . . .
  if (difference <= 120) {
      . . .
  }
  else {
      axios.get("http://127.0.0.1:8000/v1/item/get",
          {headers: {"token": localStorage
                      .getItem("token")}}).then(response => {
              . . .
              })
          }).catch(error => {
          if (error.response.status === 401) {
              this.logout();
          }
      });
  }
}
```

在这里，我们声明最后缓存的项和当前时间之间的时间差必须小于 120 秒，即 2 分钟。如果时间差低于 2 分钟，我们将从缓存中获取我们的数据。然而，如果时间差超过 2 分钟，我们将向我们的 API 后端发起请求。如果我们收到未授权的响应，我们将注销。首先，在这个`getItems`函数中，我们获取项目缓存的日期，并使用以下代码计算那时和现在之间的差异：

```rs
let cachedData = Date.parse(localStorage
                            .getItem("item-cache-date"));
let now = new Date();
let difference = Math.round((now - cachedData) / (1000));
```

如果我们的时间差是 2 分钟，我们将从本地存储中获取我们的数据，并使用以下代码用这些数据更新我们的状态：

```rs
let pendingItems =
    JSON.parse(localStorage.getItem("item-cache-data-pending"));
let doneItems =
    JSON.parse(localStorage.getItem("item-cache-data-
                                     done"));
let pendingItemsCount = pendingItems.length;
let doneItemsCount = doneItems.length;
this.setState({
  "pending_items": this.processItemValues(pendingItems),
  "done_items": this.processItemValues(doneItems),
  "pending_items_count": pendingItemsCount,
  "done_items_count": doneItemsCount
})
```

在这里，我们必须解析来自本地存储的数据，因为本地存储只处理字符串数据。由于本地存储只处理字符串，我们必须在用以下代码发起 API 请求时将我们插入本地存储的数据进行字符串化：

```rs
let pending_items = response.data["pending_items"]
let done_items = response.data["done_items"]
localStorage.setItem("item-cache-date", new Date());
localStorage.setItem("item-cache-data-pending",
                      JSON.stringify(pending_items));
localStorage.setItem("item-cache-data-done",
                      JSON.stringify(done_items));
this.setState({
  "pending_items": this.processItemValues(pending_items),
  "done_items": this.processItemValues(done_items),
  "pending_items_count":
      response.data["pending_item_count"],
  "done_items_count": response.data["done_item_count"]
})
```

如果我们运行我们的应用程序，我们只会发起一次 API 调用。如果我们在大约 2 分钟内刷新我们的应用程序，我们会看到尽管我们的前端从缓存中渲染了所有项目，但并没有新的 API 调用。然而，如果我们创建、编辑或删除一个项目，然后在 2 分钟内刷新页面，我们会看到我们的视图会恢复到之前过时的状态。这是因为创建、编辑和删除的项目也会回到它们之前的状态，但它们并没有存储在本地存储中。这可以通过更新我们的`handleReturnedState`函数以及以下代码来处理：

```rs
handleReturnedState = (response) => {
  let pending_items = response.data["pending_items"]
  let done_items = response.data["done_items"]
  localStorage.setItem("item-cache-date", new Date());
  localStorage.setItem("item-cache-data-pending",
                        JSON.stringify(pending_items));
  localStorage.setItem("item-cache-data-done",
                        JSON.stringify(done_items));
  this.setState({
     "pending_items":this.processItemValues(pending_items),
     "done_items": this.processItemValues(done_items),
     "pending_items_count":response
         .data["pending_item_count"],
      "done_items_count": response.data["done_item_count"]
  })
}
```

在这里，我们做到了。我们已经成功缓存了我们的数据，并重新使用它以防止我们的后端 API 被过度调用。这也可以应用于其他前端过程。例如，当用户结账时，可以缓存并使用客户购物车。

这使我们的简单网站更接近成为一个网络应用程序。然而，我们必须承认，随着我们越来越多地使用缓存，前端复杂性也在增加。对于我们的应用程序，这就是缓存停止的地方。目前，我们不需要在接下来的一个小时里对我们的应用程序进行任何更多的修改。然而，还有一个概念我们应该简要介绍，那就是按需代码。

# 按需代码

按需代码是在后端服务器直接在前端执行代码。这个限制是可选的，并且并不广泛使用。然而，它可能很有用，因为它赋予后端服务器决定何时以及如何在前端执行代码的权利。我们已经在做这件事了；在我们的登出视图中，我们通过简单地以字符串的形式返回它，直接在前端执行 JavaScript。这是在`src/views/auth/logout.rs`文件中完成的。我们必须记住，我们现在已经将待办事项添加到了我们的本地存储中。如果我们不在登出时从本地存储中删除这些项目，那么如果有人设法在 2 分钟内登录到同一台电脑上的自己的账户，他们就能访问我们的待办事项。虽然这种情况非常不可能发生，但我们还是应该确保安全。记住，我们的登出视图在`src/views/auth/logout.rs`文件中具有以下形式：

```rs
pub async fn logout() -> HttpResponse {
    HttpResponse::Ok()
        .content_type("text/html; charset=utf-8")
        .body(. . .)
}
```

在我们的响应体中，我们有以下内容：

```rs
"<html>\
<script>\
    localStorage.removeItem('user-token'); \
    localStorage.removeItem('item-cache-date'); \
    localStorage.removeItem('item-cache-data-pending'); \
    localStorage.removeItem('item-cache-data-done'); \
    window.location.replace(
        document.location.origin);\
</script>\
</html>"
```

通过这种方式，我们不仅删除了用户令牌，还删除了所有项目和日期。这样，一旦我们登出，我们的数据就安全了。

# 摘要

在本章中，我们已经探讨了 RESTful 设计的各个方面，并将它们应用到我们的应用程序中。我们已经评估了应用程序的层，使我们能够重构中间件，以便根据结果处理两种不同的未来。这不仅仅停止在授权请求。根据请求的参数，我们可以实现中间件将请求重定向到其他服务器，或者直接以按需代码响应的形式做出一些更改，然后进行另一个 API 调用。这种方法为我们提供了另一个工具，在视图之前，在中间件中具有多个未来结果的定制逻辑。

然后，我们重构了我们的路径结构，以使接口统一，防止前端和后端视图之间的冲突。

然后，我们探讨了不同的日志级别，并记录了所有请求以突出显示沉默但不受欢迎的行为。在重构我们的前端以纠正这个问题后，我们随后使用日志来评估当将待办事项缓存到前端以防止过多的 API 调用时，我们的缓存机制是否工作正常。现在，我们的应用程序可以接受了。我们总是可以做出改进；然而，我们还没有达到如果我们将应用程序部署到服务器上，我们就能监控它、在出现问题时代码日志、管理具有各自待办事项的多用户以及在他们到达视图之前拒绝未经授权的请求的阶段。我们还有缓存，我们的应用程序是无状态的，在 PostgreSQL 和 Redis 数据库上访问和写入数据。

在下一章中，我们将为我们的 Rust 结构编写单元测试，为我们的 API 端点编写功能测试，以及清理代码以准备部署。

# 问题

1.  为什么我们不能简单地将多个未来状态编码到中间件中，仅仅根据请求参数和授权结果调用并返回正确的一个，而必须用枚举来包装它们呢？

1.  我们如何添加新版本的视图，同时仍然支持旧视图，以防我们的 API 为可能不会立即更新的移动应用和第三方提供服务？

1.  为什么在弹性云计算时代，无状态约束变得越来越重要？

1.  我们如何利用 JWT 的特性来集成另一个服务？

1.  警告日志消息隐藏了错误已经发生的事实，但仍然提醒我们修复它。我们为什么要麻烦告诉用户发生了错误并尝试使用错误日志再次尝试？

1.  记录所有请求的优点是什么？

1.  为什么我们有时必须使用`async move`？

# 答案

1.  Rust 的强类型系统会抱怨。这是因为`async`块的行为类似于闭包，意味着每个`async`块都是其自己的类型。指向多个未来就像指向多个类型一样，因此它看起来像我们正在返回多个不同的类型。

1.  我们在视图目录中添加了一个新模块，其中包含新的视图。这些视图具有相同的端点和视图，但需要新的参数。然后我们可以在工厂函数中添加一个版本参数。这些新视图将具有包含`v2`的相同端点。这使用户能够使用新旧 API 端点。然后我们通知用户旧版本将不再被支持，给他们时间来更新。在特定时间，我们将构建中的版本移动到`v2`，切断所有执行`v1`调用的请求，并返回一个有用的消息，指出`v1`不再被支持。为了使这个过渡工作，我们必须更新`build`配置中允许的版本列表以支持版本。

1.  随着编排工具、微服务和按需弹性计算实例的普及，根据需求启动和关闭弹性计算实例已成为更常见的做法。如果我们将数据存储在实例本身上，当用户进行另一个 API 调用时，无法保证用户会访问到相同的实例，从而导致数据读取和写入的不一致性。

1.  JWT 令牌使我们能够存储用户 ID。如果第二个服务有相同的密钥，我们只需将带有 JWT 的请求传递给其他服务即可。其他服务不需要有登录视图或访问用户数据库的权限，但仍能正常工作。

1.  当发生错误，使我们无法事后回溯并解决问题时，我们必须引发错误而不是警告。一个经典的错误例子是无法写入数据库。一个良好的警告例子是另一个服务没有响应。当其他服务运行时，我们可以进行数据库调用并调用该服务以完成整个过程。

1.  在生产环境中，当进行故障排除时，需要评估服务器的状态。例如，如果用户没有体验到更新，我们可以快速检查日志以查看服务器是否实际上接收到了请求，或者前端缓存是否存在错误。我们还可以用它来查看我们的应用程序是否按预期运行。

1.  可能存在一种可能性，即我们在`async`块中引用的变量的生命周期可能不足以看到`async`块的结束。为了解决这个问题，我们可以将变量的所有权转移到带有`async` `move`块的块中。

# 第四部分：测试和部署

当一个应用程序构建完成后，我们需要将其部署到服务器上以便其他人可以使用它。我们还需要测试它以确保它符合我们的预期，然后再部署它。在本部分中，我们将介绍使用 Postman 等工具进行的单元测试和端到端测试。我们将构建自己的构建和测试管道来自动化测试、构建和部署过程。我们将介绍 HTTP 请求如何路由到服务器以及 HTTPS 协议是什么，以便我们可以在 AWS 上实现它。我们还将使用 NGINX 将流量路由到我们的前端和后端，在 AWS 上的两个独立服务器之间平衡流量，并使用 AWS 安全组锁定这些服务器和负载均衡器的流量。我们将使用 Terraform 自动化 AWS 基础设施。

本部分包括以下章节：

+   *第九章*, *测试我们的应用程序端点和组件*

+   *第十章*, *在 AWS 上部署我们的应用程序*

+   *第十一章*, *在 AWS 上使用 NGINX 配置 HTTPS*
