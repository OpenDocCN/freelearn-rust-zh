# *第三章*：Rocket 请求和响应

我们将在本章中更详细地讨论 Rocket **请求** 和 **响应**。第一部分将讨论 Rocket 如何以路由的形式处理传入请求。我们将了解路由的各个部分，包括 HTTP 方法、URI 和路径。然后，我们将创建一个使用路由各个部分的应用程序。我们还将讨论 Rust **特质**并实现一个 Rust 特质来创建请求处理器。

我们还将讨论 Rocket 路由处理器中的响应，并实现返回响应。之后，我们将更多地讨论各种内置响应实现，并学习如何创建错误处理器以在路由处理器失败时创建自定义错误。最后，我们将实现一个泛型错误处理器来处理常见的 HTTP 状态代码，如`404`和`500`。

到本章结束时，您将能够创建 Rocket 框架最重要的部分：处理传入请求并返回响应的函数。

在本章中，我们将涵盖以下主要主题：

+   理解 Rocket 路由

+   实现路由处理器

+   创建响应

+   创建默认错误处理器

# 技术要求

我们对于本章仍然有与*第二章*中*构建我们的第一个火箭 Web 应用程序*相同的技术要求。我们需要安装 Rust 编译器、文本编辑器和 HTTP 客户端。

您可以在此处找到本章的源代码：[`github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter03`](https://github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter03)。

# 理解 Rocket 路由

我们本章从讨论 Rocket 如何以路由的形式处理传入请求开始。我们编写可以用来处理传入请求的函数，将这些函数的上方放置路由属性，并将路由处理函数附加到 Rocket 上。一个路由包含一个 HTTP 方法和一个`src/main.rs`文件：

```rs
#[macro_use]
```

```rs
extern crate rocket;
```

```rs
use rocket::{Build, Rocket};
```

```rs
#[derive(FromForm)]
```

```rs
struct Filters {
```

```rs
    age: u8,
```

```rs
    active: bool,
```

```rs
}
```

```rs
#[route(GET, uri = "/user/<uuid>", rank = 1, format = "text/plain")]
```

```rs
fn user(uuid: &str) { /* ... */ }
```

```rs
#[route(GET, uri = "/users/<grade>?<filters..>")]
```

```rs
fn users(grade: u8, filters: Filters) { /* ... */ }
```

```rs
#[launch]
```

```rs
fn rocket() -> Rocket<Build> {
```

```rs
    rocket::build().mount("/", routes![user, users])
```

```rs
}
```

突出的行是`route`属性。您只能在自由函数中放置`route`属性，而不能在`Struct`的`impl`方法内部放置。现在，让我们详细讨论路由的各个部分。

## HTTP 方法

在路由定义内部看到的第一个参数是 HTTP 方法。HTTP 方法在`rocket::http::Method`枚举中定义。该枚举有`GET`、`PUT`、`POST`、`DELETE`、`OPTIONS`、`HEAD`、`TRACE`、`CONNECT`和`PATCH`成员，它们都对应于在 RFCs（请求评论）中定义的有效 HTTP 方法。

除了使用`#[route...]`宏之外，我们还可以使用其他属性来表示路由。我们可以直接使用特定方法的路由属性，如`#[get...]`。有七个特定方法的路由属性：`get`、`put`、`post`、`delete`、`head`、`options`和`patch`。我们可以将之前的路由属性重写为以下行：

```rs
#[get("/user/<uuid>", rank = 1, format = "text/plain")]
```

```rs
#[get("/users/<grade>?<filters..>")]
```

看起来很简单，对吧？不幸的是，如果我们想要处理`HTTP CONNECT`或`TRACE`，我们仍然必须使用`#[route...]`属性，因为这些方法没有特定于方法的路由属性。

## URI

在路由属性内部，我们可以看到`?`)是路径，问号之后的部分是查询。

路径和查询都可以被划分为`/`)，例如`/segment1/segment2`。查询通过`&`符号进行分段，例如`?segment1&segment2`。

一个段可以是`/static`或`?static`。动态段定义在尖括号`<>`内，例如`/<dynamic>`或`?<dynamic>`。

如果你声明一个动态段，你必须使用该段作为路由属性之后的函数的参数。以下是我们如何通过编写一个新的应用程序并添加此路由和函数处理程序来使用动态段的示例：

```rs
#[get("/<id>")]
```

```rs
fn process(id: u8) {/* ... */}
```

## 路径

路径在处理函数中的参数类型必须实现`rocket::request::FromParam`特性。你可能想知道为什么在前面的例子中我们使用了`u8`作为函数参数。答案是 Rocket 已经为重要类型，如`u8`，实现了`FromParam`特性。以下是一个已经实现`FromParam`特性的所有类型的列表：

+   原始类型，如`f32`、`f64`、`isize`、`i8`、`i16`、`i32`、`i64`、`i128`、`usize`、`u8`、`u16`、`u32`、`u64`、`u128`和`bool`。

+   Rust 标准库中的`std::num`模块的数值类型，例如`NonZeroI8`、`NonZeroI16`、`NonZeroI32`、`NonZeroI64`、`NonZeroI128`、`NonZeroIsize`、`NonZeroU8`、`NonZeroU16`、`NonZeroU32`、`NonZeroU64`、`NonZeroU128`和`NonZeroUsize`。

+   Rust 标准库中的`std::net`模块中的`net`类型，例如`IpAddr`、`Ipv4Addr`、`Ipv6Addr`、`SocketAddrV4`、`SocketAddrV6`和`SocketAddr`。

+   `&str`和`String`。

+   `Option<T>`和`Result<T, T::Error>`，其中`T:FromParam`。如果你是 Rust 的新手，这个语法是泛型类型的。`T:FromParam`意味着我们可以使用任何类型`T`，只要该类型实现了`FromParam`。例如，我们可以创建一个`User`结构体，为`User`实现`FromParam`，并将`Option<User>`用作函数处理器的参数。

如果你编写一个动态路径段，你必须使用处理函数中的参数，否则代码将无法编译。如果参数类型没有实现`FromParam`特性，代码同样会失败。

让我们看看如果我们从代码中移除`id: u8`参数，不使用处理函数中的参数会出现什么错误：

```rs
> cargo build
   ...
  Compiling route v0.1.0 (/Users/karuna/Chapter03/
  04UnusedParameter)
error: unused parameter
 --> src/main.rs:6:7
  |
6 | #[get("/<id>")]
  |       ^^^^^^^
error: [note] expected argument named `id` here
 --> src/main.rs:7:15
  |
7 | fn process_abc() { /* ... */ }
  |               ^^
```

然后，让我们编写一个动态段，它没有实现`FromParam`。定义一个空的`struct`，并将其用作处理函数中的参数：

```rs
struct S;
```

```rs
#[get("/<id>")]
```

```rs
fn process(id: S) { /* ... */ }
```

再次，代码将无法编译：

```rs
> cargo build
...
  Compiling route v0.1.0 (/home/karuna/workspace/
  rocketbook/Chapter03/05NotFromParam)
error[E0277]: the trait bound `S: FromParam<'_>` is not satisfied
--> src/main.rs:9:16
  | 
9 | fn process(id: S) { /* ... */ }
  |                ^ the trait `FromParam<'_>` is not implemented for `S` 
  | 
  = note: required by `from_param`
error: aborting due to previous error
```

我们可以从编译器的输出中看到，类型`S`必须实现`FromParam`特性。

在尖括号中还有一个动态形式，但后面跟着两个句点(`..`)，例如`/<dynamic..>`。这种动态形式被称为**多段**。

如果一个常规动态段必须实现`FromParam`特质，多个段必须实现`rocket::request::FromSegments`特质。Rocket 只为 Rust 标准库`std::path::PathBuf`类型提供`FromSegments`实现。`PathBuf`是一种在操作系统中表示文件路径的类型。这种实现对于从 Rocket 应用程序中提供静态文件非常有用。

你可能会认为从特定路径提供服务是危险的，因为任何人都可以尝试路径遍历，例如`"../../../password.txt"`。幸运的是，`PathBuf`的`FromSegments`实现已经考虑了安全问题。因此，对敏感路径的访问已被禁用，例如`".."`、`"."`或`"*"`。

另一种段类型是`<_>`或`<_..>`。如果你声明一个忽略段，它将不会显示在函数参数列表中。你必须将忽略的多个段作为路径中的最后一个参数声明，就像常规的多个段一样。

如果你想构建一个匹配很多内容但不想处理的 HTTP 路径，忽略段非常有用。例如，如果你有以下代码行，你可以拥有一个处理任何路径的网站。它将处理`/`、`/some`、`/1/2/3`或任何其他路径：

```rs
#[get("/<_>")]
```

```rs
fn index() {}
```

```rs
#[launch]
```

```rs
fn rocket() -> Rocket<Build> {
```

```rs
    rocket::build().mount("/", routes![index])
```

```rs
}
```

## 查询

就像路径一样，查询段可以是静态段或动态段（例如`"?<query1>&<query2>"`）或可以以多个`"?<query..>"`形式存在。多个查询形式称为`"?<_>"`，或者忽略尾随参数，如`"?<_..>"`。

动态查询和尾随参数都不应该实现`FromParam`，但两者都必须实现`rocket::form::FromForm`。我们将在*第八章*中更详细地讨论实现`FromForm`，*提供静态资源和模板*。

## 排名

URI 中的路径和查询段可以分为三种**颜色**：**静态**、**部分**或**通配符**。如果路径的所有段都是静态的，则该路径称为静态路径。如果查询的所有段都是静态的，我们说查询具有静态颜色。如果路径或查询的所有段都是动态的，我们称路径或查询为通配符。部分颜色是当路径或查询既有静态段又有动态段时。

我们为什么需要这些颜色？它们是确定路由的下一个参数，即**排名**所必需的。如果我们有多个处理相同路径的路由，那么 Rocket 将根据排名对函数进行排序，并从排名最低的函数开始检查。让我们看一个例子：

```rs
#[get("/<rank>", rank = 1)]
```

```rs
fn first(rank: u8) -> String {
```

```rs
    let result = rank + 10;
```

```rs
    format!("Your rank is, {}!", result)
```

```rs
}
```

```rs
#[get("/<name>", rank = 2)]
```

```rs
fn second(name: &str) -> String {
```

```rs
    format!("Hello, {}!", name)
```

```rs
}
```

```rs
#[launch]
```

```rs
fn rocket() -> Rocket<Build> {
```

```rs
    rocket::build().mount("/", routes![first, second])
```

```rs
}
```

在这里，我们看到我们有两个函数处理相同的路径，但具有两个不同的函数签名。由于 Rust 不支持函数重载，我们创建了两个不同名称的函数。让我们尝试调用每个路由：

```rs
> curl http://127.0.0.1:8000/1
Your rank is, 11!
> curl http://127.0.0.1:8000/jane
Hello, jane!
```

当我们在另一个终端查看应用程序日志时，我们可以看到 Rocket 是如何选择路由的：

```rs
GET /1:
   >> Matched: (first) GET /<rank>
   >> Outcome: Success
   >> Response succeeded.
GET /jane:
   >> Matched: (first) GET /<rank>
   >> `rank: u8` param guard parsed forwarding with error 
      "jane"
   >> Outcome: Forward
   >> Matched: (second) GET /<name> [2]
   >> Outcome: Success
   >> Response succeeded.
```

尝试在源代码中反转优先级，并思考如果用`u8`作为参数调用会发生什么。之后，尝试请求端点以查看您的猜测是否正确。

让我们回顾一下 Rocket 的 URI 颜色。Rocket 将路径和查询的颜色优先级排序如下：

+   静态路径，静态查询 = -12

+   静态路径，部分查询 = -11

+   静态路径，通配查询 = -10

+   静态路径，无查询 = -9

+   部分路径，静态查询 = -8

+   部分路径，部分查询 = -7

+   部分路径，通配查询 = -6

+   部分路径，无查询 = -5

+   通配路径，静态查询 = -4

+   通配路径，部分查询 = -3

+   通配路径，通配查询 = -2

+   通配路径，无查询 = -1

您可以看到路径的优先级较低，静态优先级低于部分，最后，部分优先级低于通配颜色。在创建多个路由时请记住这一点，因为输出可能不是您预期的，因为您的路由可能具有较低的或较高的优先级。

## 格式

在路由中我们可以使用的另一个参数是`format`。在具有有效载荷的 HTTP 方法请求中，例如`POST`、`PUT`、`PATCH`和`DELETE`，HTTP 请求的`Content-Type`将与该参数的值进行比较。当处理没有有效载荷的 HTTP 请求，例如`GET`、`HEAD`和`OPTIONS`时，Rocket 会检查并匹配路由的格式与 HTTP 请求的`Accept`头。

让我们为`format`参数创建一个示例。创建一个新的应用程序并添加以下行：

```rs
#[get("/get", format = "text/plain")]
```

```rs
fn get() -> &'static str {
```

```rs
    "GET Request"
```

```rs
}
```

```rs
#[post("/post", format = "form")]
```

```rs
fn post() -> &'static str {
```

```rs
    "POST Request"
```

```rs
}
```

```rs
#[launch]
```

```rs
fn rocket() -> Rocket<Build> {
```

```rs
    rocket::build().mount("/", routes![get, post])
```

```rs
}
```

如果您仔细观察，`/get`端点的格式使用的是`"text/plain"` IANA（互联网分配号码权威机构）媒体类型，但`/post`端点的格式不是正确的 IANA 媒体类型。这是因为 Rocket 接受以下缩写并将其转换为正确的 IANA 媒体类型：

+   `"any"` → `"*/*"`

+   `"binary"` → `"application/octet-stream"`

+   `"bytes"` → `"application/octet-stream"`

+   `"html"` → `"text/html; charset=utf-8"`

+   `"plain"` → `"text/html; charset=utf-8"`

+   `"text"` → `"text/html; charset=utf-8"`

+   `"json"` → `"application/json"`

+   `"msgpack"` → `"application/msgpack"`

+   `"form"` → `"application/x-www-form-urlencoded"`

+   `"js"` → `"application/javascript"`

+   `"css"` → `"text/css; charset=utf-8"`

+   `"multipart"` → `"multipart/form-data"`

+   `"xml"` → `"text/xml; charset=utf-8"`

+   `"pdf"` → `"application/pdf"`

现在，运行应用程序并调用这两个端点以查看它们的行为。首先，使用正确的和错误的`Accept`头调用`/get`端点：

```rs
> curl -H "Accept: text/plain" http://127.0.0.1:8000/get
GET Request
> curl -H "Accept: application/json" http://127.0.0.1:8000/get
{
  "error": {
    "code": 404,
    "reason": "Not Found",
    "description": "The requested resource could not be 
    found."
  }
}
```

具有正确`Accept`头的请求返回正确的响应，而具有不正确`Accept`头的请求返回`404`，但带有`"Content-Type: application/json"`响应头。现在，向`/post`端点发送`POST`请求以查看响应：

```rs
> curl -X POST -H "Content-Type: application/x-www-form-urlencoded" http://127.0.0.1:8000/post
POST Request
> curl -X POST -H "Content-Type: text/plain" http://127.0.0.1:8000/post
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>404 Not Found</title>
</head>
...
</html>
```

我们的应用程序输出了预期的响应，但响应的`Content-Type`不是我们预期的。我们将在本章后面学习如何创建默认的错误处理器。

## 数据

路由中的`data`参数用于处理请求体。数据必须以动态形式存在，例如动态的`<something>` URI 段。之后，声明的属性必须作为参数包含在路由属性之后的函数中。例如，看看以下几行：

```rs
#[derive(FromForm)]
```

```rs
struct Filters {
```

```rs
    age: u8,
```

```rs
    active: bool,
```

```rs
}
```

```rs
#[post("/post", data = "<data>")]
```

```rs
fn post(data: Form<Filters>) -> &'static str {
```

```rs
    "POST Request"
```

```rs
}
```

```rs
#[launch]
```

```rs
fn rocket() -> Rocket<Build> {
```

```rs
    rocket::build().mount("/", routes![post])
```

```rs
}
```

如果你没有在路由之后的函数中将`data`作为参数包含，Rust 将在编译时对此提出抱怨。尝试从函数签名中移除`data`参数，并尝试编译以查看编译器错误输出的实际效果。

当我们实现表单和上传文件到服务器时，我们将在后面了解更多关于数据的内容。现在我们已经学习了关于 Rocket 路由的知识，让我们创建一个应用程序来实现处理请求的路由。

# 实现路由处理器

在这里，我们将创建一个处理路由的应用程序。我们正在重用本章中编写的第一段代码。想法是我们有多个用户数据，我们希望发送请求以根据请求中发送的 ID 选择并返回选定的用户数据。在本部分中，我们将实现请求和选择路由处理器的部分。在下一节中，我们将学习如何创建自定义响应类型。在随后的章节中，我们将创建一个处理请求不匹配任何我们拥有的用户数据的处理器。最后，在最后一节中，我们将创建一个默认的错误处理器来处理无效请求。

让我们先复制第一段代码到一个新的文件夹中。之后，在`src/main.rs`中，在`Filter`定义之后添加一个`User`结构体：

```rs
struct Filters {
```

```rs
    ...
```

```rs
}
```

```rs
#[derive(Debug)]
```

```rs
struct User {
```

```rs
    uuid: String,
```

```rs
    name: String,
```

```rs
    age: u8,
```

```rs
    grade: u8,
```

```rs
    active: bool,
```

```rs
}
```

对于`User`结构体，我们使用`uuid`作为对象标识。原因是如果我们使用`usize`或其他数值类型作为 ID 而不进行任何认证，我们可能会陷入**不安全的直接对象引用**（**IDOR**）安全漏洞，其中未经授权的用户可以轻易猜测任何数字作为 ID。作为标识符的 UUID 更难以猜测。

此外，在实际应用中，我们可能需要为*姓名*和*年龄*创建透明加密，因为这些信息可能被视为个人可识别信息，但为了学习的目的，我们在这本书中将其省略。

我们还在结构体顶部添加了`#[derive(Debug)]`属性。该属性会自动为结构体创建一个用于打印的`fmt::Debug`实现。然后我们可以在代码中使用它，例如`format!("{:?}", User)`。`Debug`属性的一个要求是所有类型成员都必须实现`Debug`；然而，在我们的情况下这并不是问题，因为所有 Rust 标准库类型已经实现了`Debug`特质。

关于下一步，我们希望在集合数据结构中存储几个 `User` 数据。我们可以将它们存储在一个 `[User; 5]` 数组或一个可增长的 `std::vec::Vec` 数组类型中。要找到数组中的用户数据，我们可以逐个迭代数组或 Vec，直到结束或找到匹配项，但这不是最佳方案，因为对于大型数组来说，这很耗时。

在计算机科学中，有一些更好的数据结构可以用来存储数据，并且可以通过索引轻松找到对象，例如哈希表。Rust 有许多库实现了各种数据结构，哈希表就是其中之一。在标准库中，我们可以在 `std::collections::HashMap` 中找到它。

除了使用标准库之外，我们还可以使用其他替代方案，因为 Rust 社区已经创建了许多与数据结构相关的库。尝试在 [`crates.io`](https://crates.io) 或 [`lib.rs`](https://lib.rs) 中搜索。例如，如果我们不使用标准库，我们可以使用替代 crate，如 `hashbrown`。

让我们在 `User` 结构体声明之后在 `src/main.rs` 文件中实现它。不幸的是，`HashMap` 的创建需要堆分配，所以我们不能将 `HashMap` 赋值给静态变量。添加以下代码将不起作用：

```rs
use std::collections::HashMap;
```

```rs
...
```

```rs
static USERS: HashMap<&str, User> = {
```

```rs
    let map = HashMap::new();
```

```rs
    map.insert(
```

```rs
        "3e3dd4ae-3c37-40c6-aa64-7061f284ce28",
```

```rs
        User {
```

```rs
            uuid: String::from("3e3dd4ae-3c37-40c6-aa64-
```

```rs
            7061f284ce28"),
```

```rs
            name: String::from("John Doe"),
```

```rs
            age: 18,
```

```rs
            grade: 1,
```

```rs
            active: true,
```

```rs
        },
```

```rs
    );
```

```rs
    map
```

```rs
};
```

有几种方法可以将 `HashMap` 赋值给静态变量，但最好的建议是使用来自 `lazy_static` crate 的 `lazy_static!` 宏，它在运行时执行代码并执行堆分配。让我们将其添加到我们的代码中。首先，在 `Cargo.toml` 依赖项中添加 `lazy_static`：

```rs
[dependencies]
```

```rs
lazy_static = "1.4.0"
```

```rs
rocket = "0.5.0-rc.1"
```

然后，按照以下方式在代码中使用和实现它。如果你想稍后测试，可以随意添加额外的用户：

```rs
use lazy_static::lazy_static;
```

```rs
use std::collections::HashMap;
```

```rs
...
```

```rs
lazy_static! {
```

```rs
    static ref USERS: HashMap<&'static str, User> = {
```

```rs
        let mut map = HashMap::new();
```

```rs
        map.insert(
```

```rs
            "3e3dd4ae-3c37-40c6-aa64-7061f284ce28",
```

```rs
            User {
```

```rs
                ...
```

```rs
            },
```

```rs
        );
```

```rs
        map
```

```rs
    };
```

```rs
}
```

让我们按照以下方式修改 `fn user(...)`：

```rs
#[get("/user/<uuid>", rank = 1, format = "text/plain")]
```

```rs
fn user(uuid: &str) -> String {
```

```rs
    let user = USERS.get(uuid);
```

```rs
    match user {
```

```rs
        Some(u) => format!("Found user: {:?}", u),
```

```rs
        None => String::from("User not found"),
```

```rs
    }
```

```rs
}
```

我们希望函数在调用时返回一些内容，因此我们在函数签名中添加 `-> String`。

`HashMap` 有许多方法，例如用于插入新键值对的 `insert()` 方法，或者 `keys()` 方法，它返回 `HashMap` 中键的迭代器。我们只是使用了 `get()` 方法，它返回 `std::option::Option`。记住，`Option` 只是一个枚举，可以是 `None`，或者如果它包含值，则是 `Some(T)`。最后，`match` 控制流运算符根据值是 `None` 还是 `Some(u)` 返回适当的字符串。

现在，如果我们尝试向 `http://127.0.0.1:8000/user/3e3dd4ae-3c37-40c6-aa64-7061f284ce28` 发送一个 `GET` 请求，我们可以看到它将返回正确的响应，如果我们向 `http://127.0.0.1:8000/user/other` 发送一个 `GET` 请求，它将返回 `"User not found"`。

现在，让我们实现 `users()` 函数。让我们回顾一下原始签名：

```rs
#[route(GET, uri = "/users/<grade>?<filters..>")]
```

```rs
fn users(grade: u8, filters: Filters) {}
```

因为 `u8` 已经实现了 `FromParam`，所以我们可以直接使用它。但是，我们想看看如何为自定义类型实现 `FromParam`。让我们改变我们的用例，使其路径类似于 `"/users/<name_grade>?<filters...>"`。

首先，创建一个自定义的 `NameGrade` 结构体。`'r` 注解意味着这个结构体应该只在其名称字段中引用的字符串存在期间存在：

```rs
struct NameGrade<'r> {
```

```rs
    name: &'r str,
```

```rs
    grade: u8,
```

```rs
}
```

如果我们要实现一个特性，我们必须查看该特性的签名。Rust 编译器要求类型实现特性中的所有方法和类型占位符。我们可以从 Rocket API 文档中找到 `FromParam` 的特性定义：

```rs
pub trait FromParam<'a>: Sized {
```

```rs
    type Error: Debug;
```

```rs
    fn from_param(param: &'a str) -> Result<Self, Self::Error>;
```

```rs
}
```

`type Error: Debug;` 被称为类型占位符。某些特性要求实现必须具有某种类型。任何实现了此特性的类型都应该使用一个具体的类型，该类型也具有调试特性。因为我们只想显示错误信息，所以我们可以使用 `&'static str` 作为此实现的 `Error` 类型。然后，按照如下方式编写 `NameGrade` 的特性实现签名：

```rs
use rocket::{request::FromParam, Build, Rocket};
```

```rs
...
```

```rs
impl<'r> FromParam<'r> for NameGrade<'r> {
```

```rs
    type Error = &'static str;
```

```rs
    fn from_param(param: &'r str) -> Result<Self, Self::
```

```rs
    Error> {}
```

```rs
}
```

在函数内部，将我们想要显示给应用程序用户的消息添加进去：

```rs
const ERROR_MESSAGE: Result<NameGrade, &'static str> = Err("Error parsing user parameter");
```

然后，让我们在 `'_'` 字符处拆分输入参数：

```rs
let name_grade_vec: Vec<&'r str> = param.split('_').collect();
```

`name_grade_vec` 的长度将是 `2` 或 `other`，因此我们可以对它使用 `match`。由于 `name_grade_vec[0]` 是一个字符串，我们可以直接使用它，但对于第二个成员，我们必须对其进行解析。而且，由于结果可以是任何东西，我们必须使用一种特殊的语法，其形式如下 `::<Type>`。这种语法被 Rust 社区亲切地称为 **turbofish**。

就像 `Option` 一样，`Result` 只是一个枚举，可以是 `Ok(T)` 或 `Err(E)`。如果程序成功解析 `u8`，该方法可以返回 `Ok(NameGrade{...})`，否则函数可以返回 `Err("...")`：

```rs
match name_grade_vec.len() {
```

```rs
    2 => match name_grade_vec[1].parse::<u8>() {
```

```rs
        Ok(n) => Ok(Self {
```

```rs
            name: name_grade_vec[0],
```

```rs
            grade: n,
```

```rs
        }),
```

```rs
        Err(_) => ERROR_MESSAGE,
```

```rs
    },
```

```rs
    _ => ERROR_MESSAGE,
```

```rs
}
```

现在我们已经为 `NameGrade` 实现了 `FromParam`，我们可以在 `users()` 函数中使用 `NameGrade` 作为参数。我们还想将 `String` 作为函数的返回类型：

```rs
#[get("/users/<name_grade>?<filters..>")]
```

```rs
fn users(name_grade: NameGrade, filters: Filters) -> String {}
```

在函数内部，编写用 `name_grade` 和 `filters` 过滤 `USERS` 哈希表的例程：

```rs
let users: Vec<&User> = USERS
```

```rs
    .values()
```

```rs
    .filter(|user| user.name.contains(&name_grade.name) && 
```

```rs
    user.grade == name_grade.grade)
```

```rs
    .filter(|user| user.age == filters.age && user.active 
```

```rs
    == filters.active)
```

```rs
    .collect();
```

`HashMap` 有一个 `values()` 方法，它返回 `std::collections::hash_map::Values`。`Values` 实现了 `std::iter::Iterator`，因此我们可以使用 `filter()` 方法对其进行过滤。`filter()` 方法接受一个 *闭包*，它返回 Rust 的 bool 类型。`filter()` 方法本身返回 `std::iter::Filter`，它实现了 `Iterator` 特性。`Iterator` 特性有一个 `collect()` 方法，可以用来将项目收集到集合中。有时，如果结果类型不能被编译器推断出来，你必须使用 `::<Type>` turbofish 在 `collect::<Type>()` 中。

然后，我们可以将收集到的用户转换为 `String`：

```rs
if users.len() > 0 {
```

```rs
    users
```

```rs
        .iter()
```

```rs
        .map(|u| u.name.to_string())
```

```rs
        .collect::<Vec<String>>()
```

```rs
        .join(",")
```

```rs
} else {
```

```rs
    String::from("No user found")
```

```rs
}
```

完成此操作后，运行应用程序并尝试调用 `users()` 函数：

```rs
curl -G -d age=18 -d active=true http://127.0.0.1:8000/users/John_1
```

它可以工作，但问题是查询参数很麻烦；我们希望 `Filters` 是可选的。让我们稍微修改一下代码。将 `fn users` 的签名更改为以下内容：

```rs
fn users(name_grade: NameGrade, filters: Option<Filters>) -> String {
```

```rs
...
```

```rs
        .filter(|user| {
```

```rs
            if let Some(fts) = &filters {
```

```rs
                user.age == fts.age && user.active == 
```

```rs
                fts.active
```

```rs
            } else {
```

```rs
                true
```

```rs
            }
```

```rs
        })
```

```rs
...
```

你可能会被这段代码弄糊涂：`if let Some(fts) = &filters`。这是 Rust 中的解构语法之一，就像这段代码一样：

```rs
match something {
```

```rs
    Ok(i) => /* use i here */ "",
```

```rs
    Err(err) => /* use err here */ "",
```

```rs
}
```

我们已经实现了这两个端点的请求部分，`user()`和`users()`，但那些端点的返回类型是 Rust 标准库类型`String`。我们想要使用我们自己的自定义类型。所以，让我们在下一节中看看如何直接从`User`结构体创建响应。

# 创建响应

让我们为`User`类型实现一个自定义响应。在 Rocket 中，所有实现了`rocket::response::Responder`的类型都可以用作处理路由的函数的返回类型。

让我们看看`Responder`特质的签名。这个特质需要两个生命周期，`'r`和`'o'`。结果生命周期`'o'`必须至少等于`'r'`生命周期：

```rs
pub trait Responder<'r, 'o: 'r> {
```

```rs
    fn respond_to(self, request: &'r Request<'_>) -> 
```

```rs
    Result<'o>;
```

```rs
}
```

首先，我们可以包含实现`User`结构`Responder`特质的所需模块：

```rs
use rocket::http::ContentType;
```

```rs
use rocket::response::{self, Responder, Response};
```

```rs
use std::io::Cursor;
```

然后，添加`User`结构的实现签名：

```rs
impl<'r> Responder<'r, 'r> for &'r User {
```

```rs
    fn respond_to(self, _: &'r Request<'_>) -> 
```

```rs
    response::Result<'r> {    }
```

```rs
}
```

为什么我们使用`rocket::response::{self...}`而不是`rocket::response::{Result...}`？如果我们返回`-> Result`，我们就不能使用在 Rust 中相当普遍的`std::result::Result`类型。在`respond_to()`方法体中写下以下行：

```rs
let user = format!("Found user: {:?}", self);
```

```rs
Response::build()
```

```rs
    .sized_body(user.len(), Cursor::new(user))
```

```rs
    .raw_header("X-USER-ID", self.uuid.to_string())
```

```rs
    .header(ContentType::Plain)
```

```rs
    .ok()
```

应用程序从`User`对象生成一个用户`String`，然后通过调用`Response::build()`生成`rocket::response::Builder`。我们可以为`Builder`实例设置各种有效载荷；例如，`sized_body()`方法添加响应体，`raw_header()`和`header()`添加 HTTP 头，最后，我们使用`finalize()`方法生成`response::Result()`。

`sized_body()`方法的第一个参数是`Option`，参数可以是`None`。因此，`sized_body()`方法需要第二个参数实现`tokio::io::AsyncRead + tokio::io::AsyncSeek`特质以自动确定大小。幸运的是，我们可以将主体包装在`std::io::Cursor`中，因为 Tokio 已经为`Cursor`实现了这些特质。

当我们实现`std::iter::Iterator`特质和`rocket::response::Builder`时，可以观察到的一个常见模式是通过链式命令调用`Something`实例，例如`Something.new().func1().func2()`：

```rs
struct Something {}
```

```rs
impl Something {
```

```rs
    fn new() -> Something { ... }
```

```rs
    fn func1(&mut self) -> &mut Something { ... }
```

```rs
    fn func2(&mut self) -> &mut Something { ... }
```

```rs
}
```

让我们也修改`users()`函数以返回一个新的`Responder`。我们正在定义一个新的类型，这通常被称为**newtype**习语。如果我们想包装一个集合或绕过**孤儿规则**，这个习语很有用。

孤儿规则意味着`type`或`impl`都不在我们的应用程序或 crate 中。例如，我们无法在我们的应用程序中实现`impl Responder for Iterator`。原因是`Iterator`是在标准库中定义的，而`Responder`是在 Rocket crate 中定义的。

我们可以使用以下行中的 newtype 习语：

```rs
struct NewUser<'a>(Vec<&'a User>);
```

注意，结构体有一个`struct NewType(type1, type2, ...)`。

我们也可以调用一个无名称字段的`struct`为`(type1, type2, type3)`。然后我们可以通过索引访问结构体的字段，例如`self.0`、`self.1`等等。

在新类型定义之后，添加以下实现：

```rs
impl<'r> Responder<'r, 'r> for NewUser<'r> {
```

```rs
    fn respond_to(self, _: &'r Request<'_>) -> 
```

```rs
    response::Result<'r> {
```

```rs
        let user = self
```

```rs
            .0
```

```rs
            .iter()
```

```rs
            .map(|u| format!("{:?}", u))
```

```rs
            .collect::<Vec<String>>()
```

```rs
            .join(",");
```

```rs
        Response::build()
```

```rs
            .sized_body(user.len(), Cursor::new(user))
```

```rs
            .header(ContentType::Plain)
```

```rs
            .ok()
```

```rs
    }
```

```rs
}
```

与为`User`类型实现的`Responder`类似，在`NewUser`的`Responder`实现中，我们基本上再次遍历用户集合，将它们收集为一个字符串，并再次构建`response::Result`。

最后，让我们在`user()`和`users()`函数中使用`User`和`NewUser`结构体作为响应类型：

```rs
#[get("/user/<uuid>", rank = 1, format = "text/plain")]
```

```rs
fn user(uuid: &str) -> Option<&User> {
```

```rs
    let user = USERS.get(uuid);
```

```rs
    match user {
```

```rs
        Some(u) => Some(u),
```

```rs
        None => None,
```

```rs
    }
```

```rs
}
```

```rs
#[get("/users/<name_grade>?<filters..>")]
```

```rs
fn users(name_grade: NameGrade, filters: Option<Filters>) -> Option<NewUser> {
```

```rs
    ...
```

```rs
    if users.len() > 0 {
```

```rs
        Some(NewUser(users))
```

```rs
    } else {
```

```rs
        None
```

```rs
    }
```

```rs
}
```

现在我们已经学会了如何为类型实现`Responder`特质，让我们在下一节中了解更多 Rocket 提供的包装器。

## 包装响应器

Rocket 有两个模块可以用来包装返回的`Responder`。

第一个模块是`rocket::response::status`，它有以下结构体：`Accepted`、`BadRequest`、`Conflict`、`Created`、`Custom`、`Forbidden`、`NoContent`、`NotFound`和`Unauthorized`。除了`Custom`之外的所有响应器都设置状态，就像它们对应的 HTTP 响应代码一样。例如，我们可以将之前的`user()`函数修改如下：

```rs
use rocket::response::status;
```

```rs
...
```

```rs
fn user(uuid: &str) -> status::Accepted<&User>  {
```

```rs
    ...
```

```rs
    status::Accepted(user)
```

```rs
}
```

`Custom`类型可以用来包装带有其他不在其他结构体中可用的 HTTP 代码的响应。例如，看看以下行：

```rs
use rocket::http::Status;
```

```rs
use rocket::response::status;
```

```rs
...
```

```rs
fn user(uuid: &str) -> status::Custom<&User>  {
```

```rs
    ...
```

```rs
    status::Custom(Status::PreconditionFailed, user)
```

```rs
}
```

另一个模块`rocket::response::content`有以下结构体：`Css`、`Custom`、`Html`、`JavaScript`、`Json`、`MsgPack`、`Plain`和`Xml`。与`status`模块类似，`content`模块用于设置响应的`Content-Type`。例如，我们可以将我们的代码修改为以下行：

```rs
use rocket::response::content;
```

```rs
use rocket::http::ContentType;
```

```rs
...
```

```rs
fn user(uuid: &str) -> content::Plain<&User> {
```

```rs
    ...
```

```rs
    content::Plain(user)
```

```rs
}
```

```rs
...
```

```rs
fn users(name_grade: NameGrade, filters: Option<Filters>) -> content::Custom<NewUser> {
```

```rs
    ...
```

```rs
    status::Custom(ContentType::Plain, NewUser(users));
```

```rs
}
```

我们也可以像以下示例那样结合这两个模块：

```rs
fn user(uuid: &str) -> status::Accepted<content::Plain<&User>> {
```

```rs
    ...
```

```rs
    status::Accepted(content::Plain(user))
```

```rs
}
```

我们可以使用`rocket::http::Status`和`rocket::http::ContentType`重写这一点：

```rs
fn user(uuid: &str) -> (Status, (ContentType, &User)) {
```

```rs
    ...
```

```rs
    (Status::Accepted, (ContentType::Plain, user))
```

```rs
}
```

现在，你可能想知道这些结构体如何创建 HTTP `Status`和`Content-Type`以及使用另一个`Responder`实现体体。答案是`Response`结构体有两个方法：`join()`和`merge()`。

假设有两个`Response`实例：`original`和`override`。`original.join(override)`方法如果`original`中不存在，则合并`override`的体和状态。`join()`方法还会附加来自`override`的相同头。

同时，`merge()`方法用`override`的原始体和状态替换`original`，如果`override`中存在，则替换`original`的原始头。

让我们重写我们的应用程序以使用默认响应。这次我们想要添加一个新的 HTTP 头`"X-CUSTOM-ID"`。为此，实现以下函数：

```rs
fn default_response<'r>() -> response::Response<'r> {
```

```rs
    Response::build()
```

```rs
        .header(ContentType::Plain)
```

```rs
        .raw_header("X-CUSTOM-ID", "CUSTOM")
```

```rs
        .finalize()
```

```rs
}
```

然后，修改`User`结构的`Responder`实现：

```rs
fn respond_to(self, _: &'r Request<'_>) -> response::Result<'r> {
```

```rs
    let base_response = default_response();
```

```rs
    let user = format!("Found user: {:?}", self);
```

```rs
    Response::build()
```

```rs
        .sized_body(user.len(), Cursor::new(user))
```

```rs
        .raw_header("X-USER-ID", self.uuid.to_string())
```

```rs
        .merge(base_response)
```

```rs
        .ok()
```

```rs
}
```

最后，修改`NewUser`的`Responder`实现。但这次，我们想要添加额外的值：`"X-CUSTOM-ID"`头。我们可以使用`join()`方法做到这一点：

```rs
fn respond_to(self, _: &'r Request<'_>) -> response::Result<'r> {
```

```rs
    let base_response = default_response();
```

```rs
    ...
```

```rs
    Response::build()
```

```rs
        .sized_body(user.len(), Cursor::new(user))
```

```rs
        .raw_header("X-CUSTOM-ID", "USERS")
```

```rs
        .join(base_response)
```

```rs
        .ok()
```

```rs
}
```

再次尝试打开`user`和`users`的 URL；你应该看到正确的`Content-Type`和`X-CUSTOM-ID`：

```rs
< x-custom-id: CUSTOM 
< content-type: text/plain; charset=utf-8
< x-custom-id: USERS 
< x-custom-id: CUSTOM 
< content-type: text/plain; charset=utf-8
```

## 内置实现

除了`content`和`status`包装器外，Rocket 已经为几种类型实现了`Responder`特质，以使开发者更容易使用。以下是一个已经实现`Responder`特质的类型列表：

+   `std::option::Option` – 对于任何已经实现了 `Responder` 的 `T` 类型，我们都可以返回 `Option<T>`。如果返回的变体是 `Some(T)`，则将 `T` 返回给客户端。我们已经在 `user()` 和 `users()` 函数中看到了这种返回类型的示例。

+   `std::result::Result` – `Result<T, E>` 中的两种变体 `T` 和 `E` 都应该实现 `Responder`。例如，我们可以将我们的 `user()` 实现更改为返回 `status::NotFound`，如下所示：

    ```rs
    use rocket::response::status::NotFound;
    ...
    fn user(uuid: &str) -> Result<&User, NotFound<&str>> {
        let user = USERS.get(uuid);
        user.ok_or(NotFound("User not found"))
    }
    ```

+   `&str` 和 `String` – 这些类型以文本内容作为响应体返回，并且 `Content-Type` 为 "text/plain"。

+   `rocket::fs::NamedFile` – 这个 `Responder` 特性会根据文件内容自动返回一个指定了 `Content-Type` 的文件。例如，我们有 `"static/favicon.png"` 文件，我们想在应用程序中提供它。请看以下示例：

    ```rs
    use rocket::fs::{NamedFile, relative};
    use std::path::Path;
    #[get("/favicon.png")]
    async fn favicon() -> NamedFile {
        NamedFile::open(Path::new(relative!(
        "static")).join("favicon.png")).await.unwrap()
    }
    ```

+   `rocket::response::Redirect` – `Redirect` 用于向客户端返回一个重定向响应。我们将在 *第八章*，*提供静态资源和模板* 中更详细地讨论 `Redirect`。

+   `rocket_dyn_templates::Template` – 这个响应者返回一个动态模板。我们将在 *第八章*，*提供静态资源和模板* 中更详细地讨论模板。

+   `rocket::serde::json::Json` – 这个类型使得返回 JSON 类型变得容易。要使用这个响应者实现，你必须在 `Cargo.toml` 中启用 `"json"` 功能，如下所示：`rocket = {version = "0.5.0-rc.1", features = ["json"]}`。我们将在 *第十一章*，*安全和添加 API 及 JSON* 中更详细地讨论 JSON。

+   `rocket::response::Flash` – `Flash` 是一种在客户端访问后会被删除的 cookie 类型。我们将在 *第十一章*，*安全和添加 API 及 JSON* 中学习如何使用这种类型。

+   `rocket::serde::msgpack::MsgPack` – `Cargo.toml` 中的 `"msgpack"` 功能。

+   `rocket::response::stream` 模块中的各种 `stream` 响应者 – 我们将在 *第九章*，*显示用户帖子* 和 *第十章*，*上传和处理帖子* 中了解更多关于这些响应者的信息。

我们已经实现了一些路由，派生自 `FromParam`，并创建了实现了 `Responder` 特性的类型。在下一节中，我们将学习如何为相同的 HTTP 状态码创建默认的错误处理器。

# 创建默认错误处理器

应用程序应该能够在处理过程中处理可能随时发生的错误。在 Web 应用程序中，将错误返回给客户端的标准方式是使用 HTTP 状态码。Rocket 提供了一种以 `rocket::Catcher` 的形式处理向客户端返回错误的方法。

捕获处理器的工作方式与路由处理器类似，但有几点例外。让我们修改我们的最后一个应用程序来看看它是如何工作的。让我们回顾一下我们是如何实现 `user()` 函数的：

```rs
fn user(uuid: &str) -> Result<&User, NotFound<&str>> {
```

```rs
    let user = USERS.get(uuid);
```

```rs
    user.ok_or(NotFound("User not found"))
```

```rs
}
```

如果我们请求 `GET /user/wrongid`，应用程序将返回一个带有代码 `404`、`"text/plain"` 内容类型和 `"User not found"` 体的 HTTP 响应。让我们将函数改回返回 `Option`：

```rs
fn user(uuid: &str) -> Option<&User> {
```

```rs
    USERS.get(uuid) 
```

```rs
}
```

返回 `Option` 的函数，其中变体为 `None`，将使用默认的 `404` 错误处理器。之后，我们可以按以下方式实现默认的 `404` 处理器：

```rs
#[catch(404)]
```

```rs
fn not_found(req: &Request) -> String {
```

```rs
    format!("We cannot find this page {}.", req.uri())
```

```rs
}
```

注意函数上方的 `#[catch(404)]` 属性。它看起来像是一个路由指令。我们可以使用 `200` 到 `599` 或 `default` 之间的任何有效 HTTP 状态码。如果我们放置 `default`，它将用于代码中未声明的任何 HTTP 状态码。

与 `route` 类似，`catch` 属性必须放在一个自由函数上方。我们无法在 `impl` 块内的方法上方放置 `catch` 属性。同样，与路由处理函数一样，捕获函数必须返回一个实现了 `Responder` 的类型。

处理错误的函数可以有零个、一个或两个参数。如果函数有一个参数，参数类型必须是 `&rocket::Request`。如果函数有两个参数，第一个参数类型必须是 `rocket::http::Status`，第二个参数必须是 `&Request`。

捕获函数连接到火箭的方式略有不同。在我们使用 `mount()` 和 `routes!` 宏处理路由函数时，我们使用 `register()` 和 `catchers!` 宏处理捕获函数：

```rs
#[launch]
```

```rs
fn rocket() -> Rocket<Build> {
```

```rs
    rocket::build().mount("/", routes![user, users, 
```

```rs
    favicon]).register("/", catchers![not_found])
```

```rs
}
```

我们如何告诉路由处理函数使用捕获器？假设一个捕获器已经被定义并注册，如下所示：

```rs
#[catch(403)]
```

```rs
fn forbidden(req: &Request) -> String {
```

```rs
    format!("Access forbidden {}.", req.uri())
```

```rs
}
```

```rs
fn rocket() -> Rocket<Build> {
```

```rs
    rocket::build().mount("/", routes![user, users, 
```

```rs
    favicon]).register("/", catchers![not_found, 
```

```rs
    forbidden])
```

```rs
}
```

然后，我们可以在路由处理函数中直接返回 `rocket::http::Status`。状态随后将被转发到任何已注册的捕获器或火箭内置的捕获器：

```rs
use rocket::http::{Status, ContentType};
```

```rs
...
```

```rs
fn users(name_grade: NameGrade, filters: Option<Filters>) -> Result<NewUser, Status> {
```

```rs
    ...
```

```rs
    if users.is_empty() {
```

```rs
        Err(Status::Forbidden)
```

```rs
    } else {
```

```rs
        Ok(NewUser(users))
```

```rs
    }
```

```rs
}
```

尝试调用此端点的 `GET` 请求，看看会发生什么：

```rs
curl -v http://127.0.0.1:8000/users/John_2
...
< HTTP/1.1 403 Forbidden
< content-type: text/plain; charset=utf-8
...
< 
* Connection #0 to host 127.0.0.1 left intact
Access Forbidden /users/John_2.* Closing connection 0
```

应用程序返回 `403` 默认处理器中的字符串，并且也返回正确的 HTTP 状态。

# 摘要

本章探讨了 Rocket 框架最重要的部分之一。我们学习了路由及其组成部分，如 HTTP 方法、URI、路径、查询、排名和数据。我们还实现了一些路由以及与应用程序相关的各种路由类型。之后，我们探讨了创建响应类型的方法，并学习了 `Responder` 特性中已实现的各个包装器和类型。最后，我们学习了如何创建捕获器并将其连接到 Rocket 应用程序。

在下一章中，我们将学习其他火箭组件，例如状态和整流罩。我们将学习火箭应用程序的初始化过程，以及我们如何使用这些状态和整流罩来创建更现代和复杂的应用程序。
