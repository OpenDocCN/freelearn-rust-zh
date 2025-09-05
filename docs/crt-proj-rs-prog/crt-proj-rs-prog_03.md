创建 REST 网络服务

从历史上看，许多技术已被开发和用于创建客户端-服务器系统。然而，在最近几十年中，所有客户端-服务器架构都趋向于基于 Web——也就是说，基于 **超文本传输协议** (**HTTP**)。HTTP 基于 **传输控制协议** (**TCP**) 和 **互联网协议** (**IP**)。特别是，两种基于 Web 的架构已经变得流行——**简单对象访问协议** (**SOAP**) 和 **表征状态转移** (**REST**)。

虽然 SOAP 是一个实际协议，但 REST 只是一系列 *原则*。遵循 REST 原则的网络服务被称为 RESTful。在本章中，我们将看到如何使用流行的 Actix 网络框架构建 RESTful 服务。

任何网络服务（包括 REST 网络服务）都可以被任何网络客户端使用——也就是说，任何可以发送 TCP/IP 网络上的 HTTP 请求的程序都可以作为网络客户端。最典型的网络客户端是在网络浏览器中运行的网页，并包含 JavaScript 代码。任何用任何编程语言编写并在实现 TCP/IP 协议的任何操作系统上运行的程序都可以作为网络客户端。

网络服务器也被称为 **后端**，而网络客户端被称为 **前端**。

本章将涵盖以下主题：

+   REST 架构

+   使用 Actix 网络框架构建网络服务的存根并实现 REST 原则

+   构建一个完整的网络服务，能够根据客户端请求上传文件、下载文件和删除文件

+   将内部状态作为内存数据库或数据库连接池来处理

+   使用 **JavaScript 对象表示法** (**JSON**) 格式向客户端发送数据

# 技术要求

为了轻松理解本章内容，您应该具备 HTTP 的入门级知识。所需的概念如下：

+   **统一资源标识符** (**URIs**)

+   方法（如 `GET`）

+   标头

+   主体

+   内容类型（如 `plain/text`）

+   状态码（如 `Not Found=404`）

在开始本章的项目之前，应在您的计算机上安装一个通用的 HTTP 客户端。示例中使用的工具是命令行工具 **curl**，在许多操作系统上免费提供。官方下载页面是 [`curl.haxx.se/download.html`](https://curl.haxx.se/download.html)。特别是 Microsoft Windows 的页面是 [`curl.haxx.se/windows/`](https://curl.haxx.se/windows/)。

或者，您可以使用几个免费的网络浏览器实用工具之一，例如 Chrome 的 Advanced REST Client 或 Firefox 的 RESTED 和 RESTer。

本章的完整源代码位于存储库的 `Chapter03` 文件夹中，该文件夹位于 [`github.com/PacktPublishing/Creative-Projects-for-Rust-Programmers`](https://github.com/PacktPublishing/Creative-Projects-for-Rust-Programmers)。

# REST 架构

REST 架构在 HTTP 协议的基础上构建得非常牢固，但它不要求任何特定的数据格式，因此它可以以多种格式传输数据，如纯文本、JSON、**可扩展标记语言**（**XML**）或二进制（编码为 Base64）。

许多网络资源描述了 REST 架构范式是什么。其中一个可以在[`en.wikipedia.org/wiki/Representational_state_transfer`](https://en.wikipedia.org/wiki/Representational_state_transfer)找到。

然而，REST 架构的概念非常简单。它是**万维网**（**WWW**）项目背后的思想的纯粹扩展。

**万维网**项目于 1989 年诞生，作为一个全球性的**超文本**图书馆。超文本是一种包含指向其他文档链接的文档，通过反复点击链接，你可以仅使用鼠标查看许多文档。这样的文档散布在互联网上，并由一个唯一的描述符，即**统一资源定位符**（**URL**）进行标识。共享此类文档的协议是 HTTP，文档是用**超文本标记语言**（**HTML**）编写的。文档可以嵌入图像，这些图像也通过 URL 地址进行引用。

HTTP 协议允许你将页面下载到你的文档查看器（网页浏览器）中，也可以上传新文档与他人共享。你还可以用新版本替换现有文档，或删除现有文档。

如果将**文档**或**文件**的概念替换为**命名数据**或**资源**的概念，你就得到了 REST 的概念。与 RESTful 服务器的任何交互都是对数据片段的操作，通过其名称进行引用。当然，这样的数据可以是磁盘文件，也可以是数据库中的一组记录，这些记录通过查询进行标识，甚至可以是内存中保留的变量。

RESTful 服务器的一个独特之处在于服务器端没有客户端会话。与任何超文本服务器一样，RESTful 服务器不会存储客户端已登录的事实。如果有与会话相关的数据，例如当前用户或之前访问的页面，这些数据仅属于客户端。因此，每当客户端需要访问受保护的服务或特定用户的数据时，请求必须包含用户的凭据。

为了提高性能，服务器可以将会话信息存储在缓存中，但这应该是透明的。服务器（除了性能之外）应该表现得好像它没有保留任何会话信息。

# 项目概述

我们将构建几个项目，每个项目都引入了新的功能。让我们依次看看每个项目：

+   第一个项目将构建一个服务的雏形，该服务应允许任何客户端上传、下载或从服务器删除文件。这个项目展示了如何创建 REST **应用程序编程接口**（**API**），但它并不执行任何有用的操作。

+   第二个项目将实现前一个项目中描述的 API。它将构建一个服务，实际上允许任何客户端从服务器文件系统中上传、下载或删除文件。

+   第三个项目将构建一个服务，允许客户端向服务器进程中的内存数据库添加键值记录，并调用服务器中预定义的一些查询。这些查询的结果将以纯文本格式发送回客户端。

+   第四个项目将与第三个项目类似，但结果将以 JSON 格式编码。

我们的源代码很小，但它包括了 Actix web crate，而 Actix web crate 又包括了大约 200 个 crate，因此任何项目的第一次构建将需要大约 10 分钟。在应用代码的任何更改之后，构建将需要 12 到 30 秒。

选择 Actix web crate 是因为它是功能最全面、最可靠、高性能且文档良好的 Rust 后端 Web 应用程序框架。

这个框架不仅限于 RESTful 服务，因为它可以用来构建不同类型的后端 Web 软件。它是 Actix net 框架的扩展，这是一个旨在实现不同类型网络服务的框架。

# 重要的背景理论和上下文

之前我们提到，RESTful 服务基于 HTTP 协议。这是一个相当复杂的协议，但它的最重要的部分相当简单。下面是它的简化版本。

协议基于一对消息。首先，客户端向服务器发送请求，服务器在接收到这个请求后，通过向客户端发送响应来回复。这两个消息都是**美国信息交换标准代码**（**ASCII**）文本，因此它们很容易被操作。

HTTP 协议通常基于 TCP/IP 协议，这保证了这些消息到达指定的进程。

让我们看看一个典型的 HTTP 请求消息，如下所示：

```rs
GET /users/susan/index.html HTTP/1.1
Host: www.acme.com
Accept: image/png, image/jpeg, */*
Accept-Language: en-us
User-Agent: Mozilla/5.0

```

这条消息包含六行，因为结尾有一个空行。

第一行以单词`GET`开头。这个单词是*方法*，它指定了请求的操作。然后是一个 Unix 风格的*路径*，然后是协议的版本（这里，它是`1.1`）。

接下来是四行相对简单的属性。这些属性是*头信息*。有许多可能的可选头信息。

第一行空行之后的文本是*主体*。在这里，主体是空的。主体用于发送原始数据——甚至大量数据。

因此，任何 HTTP 协议的请求都会向特定的服务器发送一个命令名（方法），然后是一个资源标识符（路径）。然后是一系列属性（每行一个），然后是一个空行，最后是可能的原始数据（主体）。

最重要的方法如下详细说明：

+   `GET`：这请求从服务器下载资源（通常是 HTML 文件或图像文件，但也可能是任何数据）。路径指定了资源应读取的位置。

+   `POST`：这向服务器发送一些数据，服务器应将其视为新的。路径指定了添加这些数据的位置。如果路径标识了任何现有数据，服务器应返回错误代码。要发布的数据的内含在正文部分。

+   `PUT`：这与 `POST` 命令类似，但它的目的是替换现有数据。

+   `DELETE`：这请求根据路径指定的资源被移除。它有一个空体。

这里是一个典型的 HTTP 响应消息：

```rs
HTTP/1.1 200 OK
Date: Wed, 15 Apr 2020 14:03:39 GMT
Server: Apache/2.2.14
Accept-Ranges: bytes
Content-Length: 42
Connection: close
Content-Type: text/html

<html><body><p>Some text</p></body></html>
```

任何响应消息的第一行以协议版本开始，后跟文本格式和数字格式的状态码。成功表示为 `200 OK`。

然后，有几个标题——在这个例子中有六个——然后是一个空行，然后是正文，正文可能为空。在这种情况下，正文包含一些 HTML 代码。

您可以在以下位置找到有关 HTTP 协议的更多信息：[`en.wikipedia.org/wiki/Hypertext_Transfer_Protocol`](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol)。

# 构建 REST 服务的存根

REST 服务的典型示例是为上传和下载文本文件而设计的网络服务。由于这可能会过于复杂而难以理解，我们首先将查看一个更简单的项目，即 `file_transfer_stub` 项目，该项目模拟此服务而不在文件系统中实际执行任何操作。

您将看到无状态 RESTful 网络服务的 API 结构，而不会被有关命令实现的细节所淹没。

在下一节中，此示例将通过所需实现来完成，以获得一个工作的文件管理网络应用。

## 运行和测试服务

要运行此服务，只需在控制台中键入命令 `cargo run` 即可。构建程序后，它将打印 `Listening at address 127.0.0.1:8080 ...`，并且它将保持监听传入的请求。

要测试它，我们需要一个网络客户端。如果您更喜欢，可以使用浏览器扩展，但在此章节中，我们将使用 curl 命令行工具。

`file_transfer_stub` 服务和 `file_transfer` 服务（我们将在下一节中看到）具有相同的 API，包含以下四个命令：

1.  下载具有指定名称的文件。

1.  上传具有指定名称和指定内容的文件。

1.  上传具有指定名称前缀和指定内容的文件，作为响应获得完整名称。

1.  删除指定名称的文件。

## 使用 GET 方法获取资源

在 REST 架构中下载资源时，应使用 `GET` 方法。对于这些命令，URL 应指定要下载的文件名。不应传递任何附加数据，响应应包含文件内容和状态码，可以是 `200`、`404` 或 `500`：

1.  在控制台中输入以下命令：

```rs
curl -X GET http://localhost:8080/datafile.txt
```

1.  在那个控制台中，应该打印以下模拟行，然后立即出现提示符：

```rs
Contents of the file.
```

1.  同时，在另一个控制台中，应该打印以下行：

```rs
Downloading file "datafile.txt" ... Downloaded file "datafile.txt"
```

此命令模拟从服务器文件系统中下载 `datafile.txt` 文件的请求。

1.  `GET` 方法是 curl 的默认方法，因此你可以简单地输入以下命令：

```rs
curl http://localhost:8080/datafile.txt
```

1.  此外，你可以通过输入以下命令将输出重定向到任何文件：

```rs
curl http://localhost:8080/datafile.txt >localfile.txt
```

因此，我们现在已经看到我们的网络服务如何通过 curl 下载远程文件，将其打印到控制台，或者将其保存到本地文件。

## 使用 PUT 方法将命名资源发送到服务器

在 REST 架构中上传资源时，应使用 `PUT` 或 `POST` 方法。`PUT` 方法用于客户端知道资源应存储的位置时，本质上，它将是其 *标识键*。如果已存在具有该键的资源，则该资源将被新上传的资源替换：

1.  在控制台中输入以下命令：

```rs
curl -X PUT http://localhost:8080/datafile.txt -d "File contents."
```

1.  在那个控制台中，提示符应立即出现。同时，在另一个控制台中，应该打印以下行：

```rs
Uploading file "datafile.txt" ... Uploaded file "datafile.txt"
```

此命令模拟向服务器发送文件的请求，客户端指定该资源的名称，因此如果已存在同名资源，则该资源将被覆盖。

1.  你可以使用 *curl* 以以下方式发送指定本地文件中的数据：

```rs
curl -X PUT http://localhost:8080/datafile.txt -d @localfile.txt
```

在这里，`curl` 命令有一个额外的参数 `-d`，它允许我们指定要发送到服务器的数据。如果它后面跟着一个 `@` 符号，则该符号后面的文本用作上传文件的路径。

对于这些命令，URI 应指定要上传的文件名称和文件内容，并且响应应只包含状态码，可以是 `200`、`201`（已创建）或 `500`。`200` 和 `201` 之间的区别在于，在第一种情况下，现有文件被覆盖，在第二种情况下，创建了一个新文件。

因此，我们现在已经学会了如何使用 curl 通过我们的网络服务上传字符串到远程文件，同时指定文件名。

## 使用 POST 方法将新资源发送到服务器

在 REST 架构中，`POST` 方法是在服务负责为新资源生成标识键时使用的方法。因此，请求不需要指定它。客户端可以指定标识符的模式或前缀。由于键是自动生成的且唯一，因此不可能有另一个具有相同键的资源。但是，应该将生成的键返回给客户端，否则，之后无法引用该资源：

1.  要上传一个未知名称的文件，请在控制台中输入以下命令：

```rs
curl -X POST http://localhost:8080/data -d "File contents."
```

1.  在那个控制台中，应打印文本 `data17.txt`，然后出现提示符。这是从服务器接收到的模拟文件名。同时，在另一个控制台中，应打印以下行：

```rs
Uploading file "data*.txt" ... Uploaded file "data17.txt"
```

此命令表示向服务器发送文件请求，服务器为该资源指定一个新唯一名称，以确保不会覆盖其他资源。

对于此命令，URI 不应指定要上传文件的完整名称，而只需指定前缀；当然，请求也应包含文件内容。响应应包含新创建文件的完整名称和状态码。在这种情况下，状态码只能是 `201` 或 `500`，因为已排除文件已存在的可能性。

现在我们已经学会了如何使用 curl 将字符串上传到新的远程文件，并将为该文件命名的工作留给服务器。我们还看到生成的文件名作为响应被发送回来。

## 使用 DELETE 方法删除资源

在 REST 架构中，要删除资源，应使用 `DELETE` 方法：

1.  将以下命令输入到控制台（不用担心——不会删除任何文件！）：

```rs
curl -X DELETE http://localhost:8080/datafile.txt
```

1.  输入该命令后，提示符应立即出现。同时，在服务器控制台中，应打印以下行：

```rs
Deleting file "datafile.txt" ... Deleted file "datafile.txt"
```

此命令表示从服务器文件系统中删除文件请求。对于此类命令，URL 应指定要删除的文件名称。不需要传递其他数据，唯一的响应是状态码，可以是 `200`、`404` 或 `500`。因此，我们已经看到我们的网络服务如何使用 *curl* 删除远程文件。

作为总结，本服务的可能状态码如下：

+   `200`: OK

+   `201`: 已创建

+   `404`: 未找到

+   `500`: 内部服务器错误

此外，我们的 API 的四个命令如下：

| **方法** | **URI** | **请求数据格式** | **响应数据格式** | **状态码** |
| --- | --- | --- | --- | --- |
| `GET` | `/{filename}` | --- | text/plain | `200`, `404`, `500` |
| `PUT` | `/{filename}` | text/plain | --- | `200`, `201`, `500` |
| `POST` | `/{filename prefix}` | text/plain | text/plain | `201`, `500` |
| `DELETE` | `/{filename}` | --- | --- | `200`, `404`, `500` |

## 发送无效命令

让我们看看服务器接收到无效命令时的行为：

1.  将以下命令输入到控制台：

```rs
curl -X GET http://localhost:8080/a/b
```

1.  在那个控制台中，提示符应立即出现。同时，在另一个控制台中，应打印以下行：

```rs
Invalid URI: "/a/b"
```

此命令表示从服务器获取 `/a/b` 资源请求，但，由于我们的 API 不允许这种指定资源的方法，服务拒绝该请求。

## 检查代码

`main` 函数包含以下语句：

```rs
HttpServer::new(|| ... )
.bind(server_address)?
.run()
```

第一行创建了一个 HTTP 服务器的实例。在这里，闭包的主体被省略了。

第二行将服务器绑定到一个 IP 端点，这是一个由 IP 地址和 IP 端口组成的对，如果绑定失败则返回错误。

第三行将当前线程置于该端点的监听模式。它阻塞线程，等待传入的 TCP 连接请求。

`HttpServer::new`调用的参数是一个闭包，如下所示：

```rs
App::new()
    .service(
        web::resource("/{filename}")
            .route(web::delete().to(delete_file))
            .route(web::get().to(download_file))
            .route(web::put().to(upload_specified_file))
            .route(web::post().to(upload_new_file)),
     )
     .default_service(web::route().to(invalid_resource))
```

在这个闭包中，创建了一个新的 Web 应用，然后对其应用了一个对`service`函数的调用。这样一个函数包含了对`resource`函数的调用，该函数返回一个对象，对其应用了四个对`route`函数的调用。最后，对应用对象应用了`default_service`函数的调用。

这个复杂的语句实现了一个基于 HTTP 请求的路径和方法来决定调用哪个函数的机制。在 Web 编程术语中，这种机制被称为**路由**。

请求路由首先在地址 URI 和一或多个模式之间执行模式匹配。在这种情况下，只有一个模式，`/{filename}`，它描述了一个具有初始斜杠然后是一个单词的 URI。这个单词与`filename`名称相关联。

对`route`方法的四个调用基于 HTTP 方法（`DELETE`、`GET`、`PUT`、`POST`）进行路由。对于每个可能的 HTTP 方法都有一个特定的函数，然后调用一个`to`函数，该函数的参数是一个处理函数。

这样的`route`调用意味着以下内容：

+   如果当前 HTTP 命令的请求方法是`DELETE`，则应该通过转到`delete_file`函数来处理这样的请求。

+   如果当前 HTTP 命令的请求方法是`GET`，则应该通过转到`download_file`函数来处理这样的请求。

+   如果当前 HTTP 命令的请求方法是`PUT`，则应该通过转到`upload_specified_file`函数来处理这样的请求。

+   如果当前 HTTP 命令的请求方法是`POST`，则应该通过转到`upload_new_file`函数来处理这样的请求。

这四个名为**处理程序**的处理函数当然必须在当前作用域中实现。实际上，它们被定义了，尽管与`TODO`注释交织在一起，回忆起要有一个工作应用程序而不是存根所缺少的内容。尽管如此，这些处理程序包含了很多功能。

这样的路由机制可以用英语阅读，例如，对于一个`DELETE`命令：

创建一个`service`来管理名为`/{filename}`的`web::resource`，将`delete`命令路由到`delete_file`处理程序。

在所有模式之后，是对`default_service`函数的调用，它代表一个捕获所有模式，通常用于处理无效 URI，如前例中的`/a/b`。

捕获所有语句的参数——即`web::route().to(invalid_resource)`——导致路由到`invalid_resource`函数。你可以这样读：

对于这个 `web` 命令，将其路由到 `invalid_resource` 函数。

现在，让我们看看处理器，从最简单的一个开始，如下所示：

```rs
fn invalid_resource(req: HttpRequest) -> impl Responder {
    println!("Invalid URI: \"{}\"", req.uri());
    HttpResponse::NotFound()
}
```

此函数接收一个 `HttpRequest` 对象并返回实现了 `Responder` 特性的某个对象。这意味着它处理一个 HTTP 请求，并返回可以转换为 HTTP 响应的对象。

此函数相当简单，因为它所做的工作很少。它将 URI 打印到控制台，并返回一个 *未找到* HTTP 状态码。

其他四个处理器接收不同的参数。它是这样的：`info: Path<(String,)>`。这样的参数包含之前匹配的路径的描述，其中 `filename` 参数被放入一个单值元组中，该元组位于 `Path` 对象内部。这是因为这样的处理器不需要整个 HTTP 请求，但它们需要解析的路径参数。

注意，我们有一个处理器接收 `HttpRequest` 类型的参数，而其他处理器接收 `Path<(String,)>` 类型的参数。这种语法是可能的，因为 `main` 函数中调用的 `to` 函数期望一个泛型函数作为参数，其参数可以是几种不同类型。

所有四个处理器都以以下语句开始：

```rs
let filename = &info.0;
```

这样的语句从与路径模式匹配产生的参数元组的第一个（也是唯一一个）字段中提取一个引用。只要路径恰好包含一个参数，这个操作就会成功。`/a/b` 路径无法与模式匹配，因为它有两个参数。同样，`/` 路径也无法匹配，因为它没有参数。这些情况最终会落在 *通配符* 模式上。

现在，让我们专门检查 `delete_file` 函数。它继续以下几行：

```rs
print!("Deleting file \"{}\" ... ", filename);
flush_stdout();

// TODO: Delete the file.

println!("Deleted file \"{}\"", filename);
HttpResponse::Ok()
```

它有两个信息打印语句，并以返回成功值结束。在中间，实际删除文件的语句仍然缺失。调用 `flush_stdout` 函数是为了立即在控制台上输出文本。

`download_file` 函数与此类似，但它需要返回文件内容，因此响应更为复杂，如下面的代码片段所示：

```rs
HttpResponse::Ok().content_type("text/plain").body(contents)
```

`Ok()` 调用返回的对象首先通过调用 `content_type` 并将返回体的类型设置为 `text/plain` 来装饰，然后通过调用 `body` 并将文件内容设置为响应体。

`upload_specified_file` 函数相当简单，因为它的两个主要任务尚未完成：从请求体中获取要放入文件中的文本，并将该文本保存到文件中，如下面的代码块所示：

```rs
print!("Uploading file \"{}\" ... ", filename);
flush_stdout();

// TODO: Get from the client the contents to write into the file.
let _contents = "Contents of the file.\n".to_string();

// TODO: Create the file and write the contents into it.

println!("Uploaded file \"{}\"", filename);
HttpResponse::Ok()
```

`upload_new_file` 函数与此类似，但它还应有一个尚未实现的步骤：为要保存的文件生成一个唯一的文件名，如下面的代码块所示：

```rs
print!("Uploading file \"{}*.txt\" ... ", filename_prefix);
flush_stdout();

// TODO: Get from the client the contents to write into the file.
let _contents = "Contents of the file.\n".to_string();

// TODO: Generate new filename and create that file.
let file_id = 17;

let filename = format!("{}{}.txt", filename_prefix, file_id);

// TODO: Write the contents into the file.

println!("Uploaded file \"{}\"", filename);
HttpResponse::Ok().content_type("text/plain").body(filename)
```

因此，我们已经检查了网络服务存根的所有 Rust 代码。在下文中，我们将查看此服务的完整实现。

# 构建一个完整的网络服务

`file_transfer`项目通过填充缺失的功能来完成`file_transfer_stub`项目。

在上一个项目中省略了功能，原因如下：

+   要有一个非常简单的服务，实际上并不真正访问文件系统

+   只要有同步处理

+   忽略任何类型的失败，并保持代码简单

在这里，这些限制已经被移除。首先，让我们看看如果你编译并运行`file_transfer`项目会发生什么，然后使用与上一节相同的命令进行测试。

## 下载文件

让我们尝试以下步骤来下载文件：

1.  在控制台中输入以下命令：

```rs
curl -X GET http://localhost:8080/datafile.txt
```

1.  如果下载成功，服务器将在控制台打印以下行：

```rs
Downloading file "datafile.txt" ... Downloaded file "datafile.txt"
```

在客户端的控制台中，curl 打印出该文件的正文。

如果发生错误，服务将打印以下内容：

```rs
Downloading file "datafile.txt" ... Failed to read file "datafile.txt": No such file or directory (os error 2)
```

我们现在已经看到我们的网络服务如何通过 curl 下载文件。在下一节中，我们将学习我们的网络服务如何对远程文件执行其他操作。

## 将字符串上传到指定的文件

这是将字符串上传到具有指定名称的远程文件的命令：

```rs
curl -X PUT http://localhost:8080/datafile.txt -d "File contents."
```

如果上传成功，服务器将在控制台打印以下内容：

```rs
Uploading file "datafile.txt" ... Uploaded file "datafile.txt"
```

如果文件已经存在，它将被覆盖。如果它不存在，它将被创建。

如果发生错误，网络服务将打印以下行：

```rs
Uploading file "datafile.txt" ... Failed to create file "datafile.txt"
```

或者，它将打印以下行：

```rs
Uploading file "datafile.txt" ... Failed to write file "datafile.txt"
```

这就是我们的网络服务如何通过 curl 上传一个字符串到远程文件，同时指定文件名。

## 将字符串上传到新文件

这是将字符串上传到由服务器选择的名称的远程文件的命令：

```rs
curl -X POST http://localhost:8080/data -d "File contents."
```

如果上传成功，服务器将在控制台打印类似于以下的内容：

```rs
Uploading file "data*.txt" ... Uploaded file "data917.txt"
```

这个输出显示文件名包含一个伪随机数——在这个例子中，这是`917`，但你可能会看到其他一些数字。

在客户端的控制台中，curl 打印出该新文件的名称，因为服务器已经将其发送回客户端。

如果发生错误，服务器将打印以下行：

```rs
Uploading file "data*.txt" ... Failed to create new file with prefix "data", after 100 attempts.
```

或者，它将打印以下行：

```rs
Uploading file "data*.txt" ... Failed to write file "data917.txt"
```

这就是我们的网络服务如何通过 curl 上传一个字符串到新的远程文件，将创建新文件名的任务留给服务器。curl 工具将这个新名字作为响应接收。

## 删除文件

这是删除远程文件的命令：

```rs
curl -X DELETE http://localhost:8080/datafile.txt
```

如果删除成功，服务器将在控制台打印以下行：

```rs
Deleting file "datafile.txt" ... Deleted file "datafile.txt"
```

否则，它将打印这个：

```rs
Deleting file "datafile.txt" ... Failed to delete file "datafile.txt": No such file or directory (os error 2)
```

这就是我们的网络服务如何通过 curl 删除远程文件。

## 检查代码

让我们现在来检查这个程序和上一节中描述的程序之间的差异。`Cargo.toml`文件包含两个新的依赖项，如下面的代码片段所示：

```rs
futures = "0.1"
rand = "0.6"
```

`futures`crate 用于异步操作，而`rand`crate 用于随机生成上传文件的唯一名称。

许多新的数据类型已从外部 crate 导入，如下面的代码块所示：

```rs
use actix_web::Error;
use futures::{
    future::{ok, Future},
    Stream,
};
use rand::prelude::*;
use std::fs::{File, OpenOptions};
```

主函数只有两个更改，如下所示：

```rs
.route(web::put().to_async(upload_specified_file))
.route(web::post().to_async(upload_new_file)),
```

在这里，两个对`to`函数的调用已被替换为对`to_async`函数的调用。虽然`to`函数是**同步的**（即，它保持当前线程忙碌，直到该函数完成），但`to_async`函数是**异步的**（即，它可以推迟到预期事件发生）。

这种更改是由上传请求的本质所要求的。此类请求可以发送大文件（几个兆字节），而 TCP/IP 协议将此类文件分割成小数据包。如果服务器在接收到第一个数据包后只是等待所有数据包的到来，它可能会浪费很多时间。即使有多个线程，如果许多用户同时上传文件，系统也会尽可能多地分配线程来处理此类上传，这相当低效。一个更高效的解决方案是异步处理。

然而，`to_async`函数不能接收一个同步处理程序作为参数。它必须接收一个返回具有`impl Future<Item = HttpResponse, Error = Error>`类型的值的函数，而不是由同步处理程序返回的`impl Responder`类型。实际上，这是两个上传处理程序`upload_specified_file`和`upload_new_file`返回的类型。

返回的对象是抽象类型，但必须实现`Future`trait。自 2011 年以来，C++中也使用了*future*的概念，类似于 JavaScript 的*promises*。它表示将来可用的值，同时，当前线程可以处理其他事件。

Futures 被实现为异步闭包，这意味着这些闭包被放入内部 futures 列表的队列中，而不是立即运行。当当前线程没有其他任务运行时，队列顶部的 future 被从队列中移除并执行。

如果两个 future 被链式调用，第一个链的失败会导致第二个 future 被销毁。否则，如果链的第一个 future 成功，第二个 future 有机会运行。

回到两个上传函数，它们签名的一个更改是它们现在接收两个参数。除了包含文件名的`Path<(String,)>`类型的参数外，还有一个`Payload`类型的参数。记住，内容可以分块到达，因此这样的`Payload`参数不包含文件的文本，但它是一个对象，用于异步获取上传文件的正文。

其使用相对复杂。

首先，对于两个上传处理程序，有以下的代码：

```rs
payload
    .map_err(Error::from)
    .fold(web::BytesMut::new(), move |mut body, chunk| {
        body.extend_from_slice(&chunk);
        Ok::<_, Error>(body)
    })
    .and_then(move |contents| {
```

需要调用`map_err`来转换错误类型。

`fold` 的调用每次从网络接收一块数据，并使用它来扩展 `BytesMut` 类型的对象。这种类型实现了一种可扩展的缓冲区。

`and_then` 的调用将另一个 future 链接到当前的一个。它接收一个闭包，当 `fold` 的处理完成时将被调用。这个闭包接收所有上传的内容作为参数。这是链式调用两个 future 的方法——以这种方式调用的任何闭包都是在前一个闭包完成后异步执行的。

闭包的内容只是将接收到的内容写入指定名称的文件。这个操作是同步的。

闭包的最后一行是 `ok(HttpResponse::Ok().finish())`。这是从 future 返回的方式。注意小写的 `ok`。

`upload_new_file` 函数在网页编程概念上与之前的函数相似。它更复杂，仅仅是因为以下原因：

+   而不是提供一个完整的文件名，只提供了一个前缀，其余部分必须生成一个伪随机数。

+   结果文件名必须发送到客户端。

生成唯一文件名的算法如下：

1.  生成一个三位伪随机数，并将其连接到前缀。

1.  获得的名称用于创建一个文件；这避免了覆盖具有该名称的现有文件。

1.  如果发生冲突，将生成另一个数字，直到创建一个新文件，或者直到尝试了 100 次失败的尝试。

当然，这假设上传的文件数量始终远小于 1,000。

已经进行了其他更改，以考虑失败的可能性。

`delete_file` 函数的最后一部分现在看起来是这样的：

```rs
match std::fs::remove_file(&filename) {
    Ok(_) => {
        println!("Deleted file \"{}\"", filename);
        HttpResponse::Ok()
    }
    Err(error) => {
        println!("Failed to delete file \"{}\": {}", filename, error);
        HttpResponse::NotFound()
    }
}
```

此代码处理文件删除失败的情况。注意，在出现错误的情况下，不是返回表示数字 `200` 的成功状态码 `HttpResponse::Ok()`，而是返回表示数字 `404` 的 `HttpResponse::NotFound()` 失败代码。

`download_file` 函数现在包含一个局部函数，用于将整个文件内容读入一个字符串，如下所示：

```rs
fn read_file_contents(filename: &str) -> std::io::Result<String> {
    use std::io::Read;
    let mut contents = String::new();
    File::open(filename)?.read_to_string(&mut contents)?;
    Ok(contents)
}
```

函数以一些代码结束，以处理函数可能的失败，如下所示：

```rs
match read_file_contents(&filename) {
    Ok(contents) => {
        println!("Downloaded file \"{}\"", filename);
        HttpResponse::Ok().content_type("text/plain").body(contents)
    }
    Err(error) => {
        println!("Failed to read file \"{}\": {}", filename, error);
        HttpResponse::NotFound().finish()
    }
}
```

# 构建一个有状态的服务器

`file_transfer_stub` 项目的网页应用是完全无状态的，这意味着每个操作的行为独立于之前的操作。其他解释方式是，没有数据从一个命令保持到下一个命令，或者它只计算纯函数。

`file_transfer` 项目的网页应用有一个状态，但这个状态仅限于文件系统。这种状态是数据文件的内容。尽管如此，应用程序本身仍然是无状态的。没有变量从一个请求处理持续到另一个请求处理。

REST 原则通常被解释为规定任何 API *必须是无状态的*。这是一个误解，因为 REST 服务 *可以* 有状态，但它们 *必须表现得像无状态一样*。无状态意味着，除了文件系统和数据库外，没有信息在服务器中从一次请求处理持续到另一次请求处理。表现得像无状态意味着任何请求序列都应该获得相同的结果，即使服务器在两次请求之间被终止并重新启动。

显然，如果服务器被终止，其状态就会丢失。因此，作为无状态的行为意味着即使状态被重置，行为也应该保持一致。那么，可能的服务器状态有什么作用呢？它是为了存储可以通过任何请求再次获取的信息，但这样做可能会很昂贵。这就是缓存的概念。

通常，任何 REST web 服务器都有一个内部状态。在这个状态中存储的典型信息是数据库连接池。池最初是空的，当第一个处理器必须连接到数据库时，它会搜索池以查找可用的连接。如果找到了，它就会使用它。否则，会创建一个新的连接并将其添加到池中。池是一个共享状态，必须传递给任何请求处理器。

在前几节的项目中，请求处理器是纯函数；它们没有共享公共状态的可能性。在 `memory_db` 项目中，我们将看到如何在 Actix web 框架中实现共享状态，并将其传递给任何请求处理器。

这个 web 应用程序代表了对一个非常简单的数据库的访问。而不是执行对数据库的实际访问，这需要在您的计算机上进行进一步的安装，它只是调用了在 `src/data_access.rs` 文件中定义的 `data_access` 模块导出的某些函数，这些函数将数据库保持在内存中。

内存数据库是所有请求处理器共享的状态。在一个更现实的应用中，状态将只包含一个或多个与外部数据库的连接。

## 如何拥有一个有状态的服务器

要在 Actix 服务中拥有状态，必须声明一个结构体，并且任何应该作为状态一部分的数据都应该是该结构体的字段。

在 `main.rs` 文件的开始处，有以下的代码：

```rs
struct AppState {
    db: db_access::DbConnection,
}

```

在我们的 web 应用程序的状态中，我们只需要一个字段，但可以添加其他字段。

在 `db_access` 模块中声明的 `DbConnection` 类型代表了我们 web 应用程序的状态。在 `main` 函数中，在创建服务器之前，有以下的语句实例化了 `AppState`，然后适当地封装了它：

```rs
let db_conn = web::Data::new(Mutex::new(AppState {
    db: db_access::DbConnection::new(),
}));
```

状态被所有请求共享，Actix web 框架使用多个线程来处理请求，因此状态必须是线程安全的。在 Rust 中声明线程安全对象的一种典型方式是将它封装在一个 `Mutex` 对象中。然后，这个对象被封装在一个 `Data` 对象中。

为了确保这种状态传递给任何处理程序，必须在调用`service`函数之前添加以下行：

```rs
.register_data(db_conn.clone())
```

这里，`db_conn`对象被克隆（由于它是一个智能指针，所以成本较低），并注册到应用程序中。

这种注册的效果是，现在可以向请求处理程序（同步和异步）添加另一种类型的参数，如下所示：

```rs
state: web::Data<Mutex<AppState>>
```

这种参数可以在如下语句中使用：

```rs
let db_conn = &mut state.lock().unwrap().db
```

这里，状态被锁定以防止其他请求的并发访问，并访问其`db`字段。

## 该服务的 API

此应用程序中的其余代码并不特别令人惊讶。API 从`main`函数中使用的名称中很清楚，如下面的代码块所示：

```rs
.service(
    web::resource("/persons/ids")
        .route(web::get().to(get_all_persons_ids)))
.service(
    web::resource("/person/name_by_id/{id}")
        .route(web::get().to(get_person_name_by_id)),
)
.service(
    web::resource("/persons")
        .route(web::get().to(get_persons)))
.service(
    web::resource("/person/{name}")
        .route(web::post().to(insert_person)))
.default_service(
    web::route().to(invalid_resource))
```

注意，前三个模式使用`GET`方法，因此它们*查询*数据库。最后一个使用`POST`方法，因此它将新记录插入到数据库中。

注意以下词汇约定。

第一个和第三个模式的 URI 路径以复数名词`persons`开头，这意味着零个、一个或多个项目将由此请求管理，并且任何此类项目代表一个人。相反，第二个和第四个模式的 URI 路径以单数名词`person`开头，这意味着最多只能由一个项目管理此请求。

第一个模式以复数名词`ids`结尾，因此将处理与`id`相关的几个项目。它没有条件，因此请求所有 ID。第二个模式包含单词`name_by_id`，后面跟一个`id`参数，因此它是请求`name`数据库列的所有记录，其中`id`列的值指定。

即使在有任何疑问的情况下，处理函数或注释的名称也应该使服务的操作清晰，而无需阅读处理程序的代码。在查看处理程序的实现时，请注意它们要么根本不返回任何内容，要么只返回简单的文本。

## 测试服务

让我们通过一些 curl 操作来测试服务。

首先，我们应该填充最初为空的数据库。记住，由于它仅在内存中，每次启动服务时都是空的。

在启动程序后，输入以下命令：

```rs
curl -X POST http://localhost:8080/person/John
curl -X POST http://localhost:8080/person/Jonathan
curl -X POST http://localhost:8080/person/Mary%20Jane
```

在第一个命令之后，应在控制台打印数字`1`。在第二个命令之后，应打印`2`，在第三个命令之后，应打印`3`。它们是插入的人名的 ID。

现在，输入以下命令：

```rs
curl -X GET http://localhost:8080/persons/ids
```

应打印以下内容：`1, 2, 3`。这是数据库中所有 ID 的集合。

现在，输入以下命令：

```rs
curl -X GET http://localhost:8080/person/name_by_id/3
```

应打印以下内容：`Mary Jane`。这是`id`等于`3`的唯一人员的姓名。注意，输入序列`%20`已被解码为空格。

现在，输入以下命令：

```rs
curl -X GET http://localhost:8080/persons?partial_name=an
```

它应该打印以下内容：`2: Jonathan; 3: Mary Jane`。这是包含`name`列包含`an`子字符串的所有人员的集合。

## 数据库的实现

整个数据库实现都保存在`db_access.rs`源文件中。

数据库的实现相当简单。它是一个`DbConnection`类型，包含`Vec<Person>`，其中`Person`是一个包含两个字段的 struct——`id`和`name`。

`DbConnection`的方法描述如下：

+   `new`: 这将创建一个新的数据库。

+   `get_all_persons_ids(&self) -> impl Iterator<Item = u32> + '_`: 这返回一个迭代器，它提供了数据库中包含的所有 ID。此类迭代器的生命周期不能超过数据库本身的寿命。

+   `get_person_name_by_id(&self, id: u32) -> Option<String>`: 如果存在具有指定 ID 的唯一人员，则返回该人员的姓名，否则返回零。

+   `get_persons_id_and_name_by_partial_name<'a>(&'a self, subname: &'a str) -> impl Iterator<Item = (u32, String)> + 'a`: 这返回一个迭代器，它提供了所有姓名包含指定字符串的人员的 ID 和姓名。此类迭代器的生命周期不能超过数据库本身的寿命，也不能超过指定的字符串。

+   `insert_person(&mut self, name: &str) -> u32`: 这向数据库添加一条记录，包含一个生成的 ID 和指定的`name`。这返回生成的 ID。

## 处理查询

请求处理器，包含在`main.rs`文件中，获取几种类型的参数，如下所示：

+   `web::Data<Mutex<AppState>>`: 如前所述，这用于访问共享应用程序状态。

+   `Path<(String,)>`: 如前所述，这用于访问请求的路径。

+   `HttpRequest`: 如前所述，这用于访问一般请求信息。

但同时，请求处理器也获得了`web::Query<Filter>`参数来访问请求的可选参数。

`get_persons`处理器有一个查询参数——它是一个泛型参数，其参数是`Filter`类型。此类类型如下定义：

```rs
#[derive(Deserialize)]
pub struct Filter {
    partial_name: Option<String>,
}
```

此定义允许请求，如`http://localhost:8080/persons?partial_name=an`。在这个请求中，路径只是`/persons`，而`?partial_name=an`是所谓的查询。在这种情况下，它只包含一个参数，其键为`partial_name`，其值为`an`。它是一个字符串，它是可选的。这正是`Filter`结构体所描述的。

此外，此类类型是可序列化的，因为此类对象必须通过序列化被请求读取。

`get_persons`函数通过以下表达式访问查询：

```rs
&query.partial_name.clone().unwrap_or_else(|| "".to_string()),
```

`partial_name`字段被克隆以获取一个字符串。如果它不存在，则将其视为空字符串。

# 返回 JSON 数据

上一节返回了纯文本数据。在 Web 服务中这是不寻常的，并且很少令人满意。通常，Web 服务以 JSON、XML 或其他结构化格式返回数据。`json_db`项目与`memory_db`项目相同，除了它以 JSON 格式返回数据。

首先，让我们看看当在它上面执行上一节中的相同 curl 命令时会发生什么，如下所示：

+   插入的行为相同，因为它们只是打印了一个数字。

+   第一个查询应该打印以下内容：`[1,2,3]`。这三个数字在一个数组中，因此它们被括号包围。

+   第二个查询应该打印以下内容：`"Mary Jane"`。名字是一个字符串，因此它被引号包围。

+   第三个查询应该打印以下内容：`[[2,"Jonathan"],[3,"Mary Jane"]]`。人员序列是一个包含两个记录的数组，每个记录都是一个包含两个值的数组，一个是数字，一个是字符串。

现在，让我们看看这个项目与之前项目的代码差异。

在`Cargo.toml`文件中，增加了一个依赖项，如下所示：

```rs
serde_json = "1.0"
```

这是为了将数据序列化为 JSON 格式。

在`main.rs`文件中，`get_all_persons_ids`函数（而不是简单地返回一个字符串）有如下代码：

```rs
HttpResponse::Ok()
    .content_type("application/json")
    .body(
        json!(db_conn.get_all_persons_ids().collect::<Vec<_>>())
        .to_string())
```

首先，创建一个带有状态码`Ok`的响应；然后，将其内容类型设置为`application/json`，以便让客户端知道如何解释它将接收到的数据；最后，使用从`serde_json`crate 中取出的`json`宏设置其主体。这个宏接受一个表达式——在这种情况下，类型为`Vec<Person>`——并返回一个`serde_json::Value`值。现在，我们需要一个字符串，因此调用`to_string()`。注意，`json!`宏要求其参数实现`Serialize`特质或可转换为字符串。

`get_person_name_by_id`、`get_persons`和`insert_person`函数有类似的变化。`main`函数没有变化。`db_access.rs`文件是相同的。

# 摘要

我们已经了解了一些 Actix Web 框架的特性。这是一个非常复杂的框架，涵盖了后端 Web 开发者的大多数需求，并且仍在积极开发中。

尤其是在`file_transfer_stub`项目中，我们学习了如何创建一个 RESTful 服务的 API。在`file_transfer`项目中，我们讨论了如何实现我们网络服务的操作。在`memory_db`项目中，我们了解了如何管理内部状态，特别是包含数据库连接的状态。在`json_db`项目中，我们看到了如何以 JSON 格式发送响应。

在下一章中，我们将学习如何创建一个完整的后端 Web 应用程序。

# 问题

1.  根据 REST 原则，`GET`、`PUT`、`POST`和`DELETE`HTTP 方法分别代表什么意思？

1.  哪个命令行工具可以用来测试一个网络服务？

1.  请求处理器如何检索 URI 参数的值？

1.  如何指定 HTTP 响应的内容类型？

1.  如何生成一个唯一的文件名？

1.  为什么无状态的 API 服务需要管理状态？

1.  为什么服务的状态必须封装在`Data`和`Mutex`对象中？

1.  为什么异步处理在 Web 服务中可能是有用的？

1.  futures 的`and_then`函数的目的是什么？

1.  哪些 crate 对于以 JSON 格式组合 HTTP 响应是有用的？

# 进一步阅读

要了解更多关于 Actix 框架的信息，请查看官方文档[`actix.rs/docs/`](https://actix.rs/docs/)，并查看官方示例[`github.com/actix/examples/`](https://github.com/actix/examples/)。
