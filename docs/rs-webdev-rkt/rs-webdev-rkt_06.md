# *第五章*：设计用户生成型应用程序

我们将编写一个 Rocket 应用程序，以便更多地了解 Rocket 网络框架。在本章中，我们将设计应用程序并创建应用程序骨架。然后，我们将把应用程序骨架拆分成更小的可管理模块。

阅读本章后，您将能够设计和创建应用程序骨架，并将您的应用程序模块化到您喜欢的程度。

在本章中，我们将涵盖以下主要主题：

+   设计用户生成型网络应用程序

+   规划用户结构

+   创建应用程序路由

+   模块化 Rocket 应用程序

# 技术要求

对于本章，我们与上一章有相同的技术要求。我们需要一个 Rust 编译器、一个文本编辑器、一个 HTTP 客户端和一个 PostgreSQL 数据库服务器。

您可以在[`github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter05`](https://github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter05)找到本章的源代码。

# 设计用户生成型网络应用程序

到目前为止，我们已经获得了一些关于 Rocket 框架的基本知识，例如路由、请求、响应、状态和公平性。让我们在此基础上扩展知识，并通过创建一个完整的应用程序来学习 Rocket 框架的其他功能，如请求守卫、cookies 系统、表单、上传和模板。

我们应用程序的想法是处理用户的各种操作，并且每个用户都可以创建和删除用户生成的内容，例如文本、照片或视频。

我们可以先创建我们想要执行的要求。在各种开发方法中，有许多形式和名称用于定义要求，例如用户故事、用例、软件要求或软件需求规范。

指定要求后，我们通常可以创建应用程序骨架。然后，我们可以实现应用程序并测试实现。

在我们的情况下，因为我们希望实用并理解代码层面的情况，我们将指定要求并在同一步骤中创建应用程序骨架。

让我们从创建一个新的应用程序开始。然后，将应用程序命名为`"our_application"`，并在`Cargo.toml`中包含`rocket`和`rocket_db_pools`crate：

```rs
[package]
```

```rs
edition = "2018"
```

```rs
name = "our_application"
```

```rs
version = "0.1.0"
```

```rs
[dependencies]
```

```rs
rocket = {path = "../../../rocket/core/lib/", features = ["uuid"]}
```

```rs
rocket_db_pools = {path = "../../../rocket/contrib/db_pools/lib/", features =[ "sqlx_postgres"]}
```

修改`src/main.rs`文件以删除`main()`函数，并确保我们拥有最基本 Rocket 应用程序：

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
#[launch]
```

```rs
async fn rocket() -> Rocket<Build> {
```

```rs
    rocket::build()
```

```rs
}
```

让我们通过规划我们希望在应用程序中拥有的用户数据来进入下一步。

## 规划用户结构

让我们在应用程序中编写用户结构。在最基本层面上，我们希望有一个`uuid`字段，其类型为`Uuid`，作为唯一标识符，以及一个`username`字段，其类型为`String`，作为人类可记忆的标识符。然后，我们可以添加额外的列，例如`email`和`description`，其类型为`String`，以存储有关我们用户的一些更多信息。

我们还希望用户数据中包含 `password`，但是拥有明文 `password` 字段不是一个选择。有几个哈希选项，但是显然，我们不能使用不安全的旧哈希函数，如 `md5` 或 `sha1`。然而，我们可以使用更安全的哈希加密，如 `bcrypt`、`scrypt` 或 `argon2`。在这本书中，我们将使用 `argon2id` 函数，因为它对 `String` 作为 `password_hash` 类型具有更强的抵抗力。

我们还希望为我们的用户添加一个 `status` 列。状态可以是 `active` 或 `inactive`，因此我们可以使用 `bool` 类型。但是，在未来，我们可能希望它具有可扩展性，并具有其他状态，例如，如果要求用户在可以使用我们的应用程序之前包含电子邮件信息并确认他们的电子邮件，则可以使用 `confirmed`。我们必须使用另一种类型。

在 Rust 中，我们有 `enum`，这是一个具有多个变体的类型。我们可以有一个具有隐式区分符或显式区分符的枚举。

隐式区分符枚举是一个成员没有被赋予区分符的枚举；它自动从 `0` 开始，例如，`enum Status {Active, Inactive}`。使用隐式区分符枚举意味着我们必须在 PostgreSQL 中使用 `CREATE TYPE` SQL 语句添加一个新的数据类型，例如，`CREATE TYPE status AS ENUM ('active', 'inactive');`。

如果我们使用显式区分符枚举，即成员被赋予区分符的枚举，我们可以使用 PostgreSQL 的 `INTEGER` 类型并将其映射到 `rust i32`。一个显式区分符枚举看起来如下所示：

```rs
enum Status {
```

```rs
    Inactive = 0,
```

```rs
    Active = 1,
```

```rs
}
```

因为使用显式区分符枚举更简单，所以我们将选择这种类型作为用户状态列。

我们还希望跟踪用户数据何时创建以及何时更新。Rust 标准库提供了 `std::time` 用于时间量化类型，但这个模块非常原始，不适合日常操作。有几个尝试为 Rust 创建好的日期和时间库，例如 `time` 或 `chrono` 依赖项，幸运的是，`sqlx` 已经支持这两个依赖项。我们选择使用 `chrono` 作为这本书。

根据这些要求，让我们编写 `User` 类型的结构定义和 `sqlx` 迁移：

1.  在 `Cargo.toml` 中，添加 `sqlx`、`chrono` 和 `uuid` 依赖项：

    ```rs
    sqlx = {version = "0.5.9", features = ["postgres", "uuid", "runtime-tokio-rustls", "chrono"]}
    chrono = "0.4"
    uuid = {version = "0.8.2", features = ["v4"]}
    ```

1.  在 `src/main.rs` 中，添加 `UserStatus` 枚举和 `User` 结构体：

    ```rs
    use chrono::{offset::Utc, DateTime};
    use rocket_db_pools::sqlx::FromRow;
    use uuid::Uuid;
    #[derive(sqlx::Type, Debug)]
    #[repr(i32)]
    enum UserStatus {
        Inactive = 0,
        Active = 1,
    }
    #[derive(Debug, FromRow)]
    struct User {
        uuid: Uuid,
        username: String,
        email: String,
        password_hash: String,
        description: String,
        status: UserStatus,
        created_at: DateTime<Utc>,
        updated_at: DateTime<Utc>,
    }
    ```

注意，我们设置了具有显式区分符的 `UserStatus` 枚举，并在 `User` 结构体中使用了 `UserStatus` 作为状态类型。

1.  之后，让我们在 `Rocket.toml` 文件中设置数据库 URL 配置：

    ```rs
    [default]
    [default.databases.main_connection]
    url = "postgres://username:password@localhost/rocket"
    ```

1.  然后，再次使用 `sqlx migrate add` 命令创建数据库迁移，并按如下方式修改生成的迁移文件：

    ```rs
    CREATE TABLE IF NOT EXISTS users
    (
        uuid          UUID PRIMARY KEY,
        username      VARCHAR NOT NULL UNIQUE,
        email         VARCHAR NOT NULL UNIQUE,
        password_hash VARCHAR NOT NULL,
        description   TEXT,
        status        INTEGER NOT NULL DEFAULT 0,
        created_at    TIMESTAMPTZ NOT NULL DEFAULT 
        CURRENT_TIMESTAMP,
        updated_at    TIMESTAMPTZ NOT NULL DEFAULT 
        CURRENT_TIMESTAMP
    );
    ```

注意，我们设置了 `INTEGER`，它在 Rust 中对应于 `i32` 作为 `status` 列类型。还有一点需要注意，因为 PostgreSQL 中的 `UNIQUE` 约束已经自动为 `username` 和 `email` 创建了索引，所以我们不需要为这两个列添加自定义索引。

不要忘记再次运行 `sqlx migrate run` 命令行以运行此迁移。

1.  让我们在 `src/main.rs` 中添加这些行来初始化数据库连接池公平处理：

    ```rs
    use rocket_db_pools::{sqlx::{FromRow, PgPool}, Database};
    ...
    #[derive(Database)]
    #[database("main_connection")]
    struct DBConnection(PgPool);
    async fn rocket() -> Rocket<Build> {
        rocket::build().attach(DBConnection::init())
    }
    ```

在我们的 `User` 结构体准备好之后，接下来我们可以编写与用户相关的路由的代码框架，例如创建或删除用户。

## 创建用户路由

在先前的应用程序中，我们主要处理获取用户数据，但在现实世界的应用程序中，我们还想进行其他操作，如插入、更新和删除数据。我们可以将获取用户数据（用户和用户列表）的两个函数扩展为 **创建、读取、更新和删除** (**CRUD**) 函数。这四个基本函数可以被认为是持久数据存储的基本操作。

在一个 Web 应用程序中，存在一种基于 HTTP 方法的架构风格来执行操作。如果我们想获取一个实体或一组实体，我们使用 `HTTP GET` 方法。如果我们想创建一个实体，我们使用 `HTTP POST` 方法。如果我们想更新一个实体，我们使用 `HTTP PUT` 或 `PATCH` 方法。最后，如果我们想删除一个实体，我们使用 `HTTP DELETE` 方法。使用这些 HTTP 方法来统一传递数据被称为 **表征状态转移** (**REST**)，遵循该约束的应用程序被称为 **RESTful**。

在我们为应用程序创建 RESTful 用户路由之前，让我们考虑我们想要处理哪些传入参数以及我们想要返回哪些响应。在前面的章节中，我们创建了返回 `String` 的路由，但大多数 Web 都是由 HTML 组成的。

对于用户路由响应，我们希望得到 HTML，所以我们可以使用 `rocket::response::content::RawHtml`。我们可以将其包裹在 `Result` 中，以 `Status` 作为错误类型。让我们创建一个类型别名以避免每次使用它作为路由函数返回类型时都写 `Result<RawHtml<String>, Status>`。在 `src/main.rs` 中添加以下内容：

```rs
type HtmlResponse = Result<RawHtml<String>, Status>;
```

对于用户路由请求，请求的有效负载将根据请求的内容而有所不同。对于使用 GET 获取特定用户信息的函数，我们需要知道用户的标识符，在我们的例子中，它将是 `uuid` 类型中的 `&str`。我们只需要引用（`&str`），因为我们不会处理 `uuid`，所以不需要 `String` 类型：

```rs
#[get("/users/<_uuid>", format = "text/html")]
```

```rs
async fn get_user(mut _db: Connection<DBConnection>, _uuid: &str) -> HtmlResponse {
```

```rs
    todo!("will implement later")
```

```rs
}
```

如果我们定义了一个变量或传递了一个参数但没有使用它，编译器会发出警告，所以我们现在在函数参数的变量名前使用下划线（`_`）来抑制编译器警告。在我们稍后实现函数时，我们会将变量改为前面没有下划线的变量。

就像 `unimplemented!` 宏一样，`todo!` 宏对于原型设计很有用。语义上的区别是，如果我们使用 `todo!`，我们是在说代码将会被实现，而如果我们使用 `unimplemented!`，我们并没有做出任何承诺。

挂载路由并尝试现在运行应用程序，并对该端点进行 HTTP 请求。你可以看到应用程序将如何引发恐慌，但幸运的是，Rocket 使用`std::panic::catch_unwind`函数在服务器中处理捕获恐慌。

对于用户列表，我们必须考虑我们应用程序的可扩展性。如果我们有很多用户，如果我们尝试查询所有用户，这不会非常高效。我们需要在我们的应用程序中引入某种分页。

使用`Uuid`作为实体 ID 的一个弱点是我们不能按其 ID 对实体进行排序和排序。我们必须使用另一个有序字段。幸运的是，我们已经定义了具有 1 微秒分辨率的`created_at`字段，它可以进行排序。

但是，请注意，如果你的应用程序正在处理高流量或在分布式系统中，微秒级的分辨率可能不够。你可以使用一个公式来计算`TIMESTAMPZ`的碰撞概率，该公式用于计算*生日悖论*。你可以通过使用单调 ID 或支持纳秒级分辨率的硬件和数据库来解决这个问题，但高度可扩展的应用程序超出了本书关于 Rocket Web 框架的范围。

现在让我们先定义`Pagination`结构体，然后我们将在稍后实现这个结构体。因为我们想在用户列表中使用`Pagination`，并将其用作`request`参数，所以我们可以自动使用`#[derive(FromForm)]`来自动生成`rocket::form::FromForm`的实现。但是，我们必须创建一个新的类型，`OurDateTime`，因为孤儿规则意味着我们无法为`DateTime<Utc>`实现`rocket::form::FromForField`：

```rs
use rocket::form::{self, DataField, FromFormField, ValueField};
```

```rs
...
```

```rs
#[derive(Debug, FromRow)]
```

```rs
struct OurDateTime(DateTime<Utc>);
```

```rs
#[rocket::async_trait]
```

```rs
impl<'r> FromFormField<'r> for OurDateTime {
```

```rs
fn from_value(_: ValueField<'r>) -> form::Result<'r, 
```

```rs
    Self> {
```

```rs
        todo!("will implement later")
```

```rs
    }
```

```rs
    async fn from_data(_: DataField<'r, '_>) -> form::
```

```rs
    Result<'r, Self> {
```

```rs
        todo!("will implement later")
```

```rs
    }
```

```rs
}
```

```rs
#[derive(FromForm)]
```

```rs
struct Pagination {
```

```rs
    cursor: OurDateTime,
```

```rs
    limit: usize,
```

```rs
}
```

```rs
#[derive(sqlx::Type, Debug, FromFormField)]
```

```rs
#[repr(i32)]
```

```rs
enum UserStatus {
```

```rs
    ...
```

```rs
}
```

```rs
#[derive(Debug, FromRow, FromForm)]
```

```rs
struct User {
```

```rs
    ...
```

```rs
    created_at: OurDateTime,
```

```rs
    updated_at: OurDateTime,
```

```rs
}
```

现在，我们可以为用户列表创建一个未实现的功能：

```rs
#[get("/users?<_pagination>", format = "text/html")]
```

```rs
async fn get_users(mut _db: Connection<DBConnection>, _pagination: Option<Pagination>) -> HtmlResponse {
```

```rs
    todo!("will implement later")
```

```rs
}
```

我们需要一个页面来填写输入新用户数据的表单：

```rs
#[get("/users/new", format = "text/html")]
```

```rs
async fn new_user(mut _db: Connection<DBConnection>) -> HtmlResponse {
```

```rs
    todo!("will implement later")
```

```rs
}
```

之后，我们可以创建一个处理创建用户数据的函数：

```rs
use rocket::form::{self, DataField, Form, FromFormField, ValueField};
```

```rs
...
```

```rs
#[post("/users", format = "text/html", data = "<_user>")]
```

```rs
async fn create_user(mut _db: Connection<DBConnection>, _user: Form<User>) -> HtmlResponse {
```

```rs
    todo!("will implement later")
```

```rs
}
```

我们需要一个页面来修改现有的用户数据：

```rs
#[get("/users/edit/<_uuid>", format = "text/html")]
```

```rs
async fn edit_user(mut _db: Connection<DBConnection>, _uuid: &str) -> HtmlResponse {
```

```rs
    todo!("will implement later")
```

```rs
}
```

我们需要处理更新用户数据的函数：

```rs
#[put("/users/<_uuid>", format = "text/html", data = "<_user>")]
```

```rs
async fn put_user(mut _db: Connection<DBConnection>, _uuid: &str, _user: Form<User>) -> HtmlResponse {
```

```rs
    todo!("will implement later")
```

```rs
}
```

```rs
#[patch("/users/<_uuid>", format = "text/html", data = "<_user>")]
```

```rs
async fn patch_user(
```

```rs
    mut _db: Connection<DBConnection>,
```

```rs
    _uuid: &str,
```

```rs
    _user: Form<User>,
```

```rs
) -> HtmlResponse {
```

```rs
    todo!("will implement later")
```

```rs
}
```

`PUT`和`PATCH`之间的区别是什么？简单来说，在 REST 中，如果我们想完全替换资源，则使用`PUT`请求，而`PATCH`用于部分更新数据。

最后一个与用户相关的函数是执行`HTTP DELETE`的函数：

```rs
#[delete("/users/<_uuid>", format = "text/html")]
```

```rs
async fn delete_user(mut _db: Connection<DBConnection>, _uuid: &str) -> HtmlResponse {
```

```rs
    todo!("will implement later")
```

```rs
}
```

在创建与用户相关的路由处理函数之后，我们可以扩展我们的需求。

## 制作用户生成内容

只处理用户数据的应用程序并不有趣，所以我们将添加我们的用户上传和删除帖子的能力。每个帖子可以是文本帖子、照片帖子或视频帖子。让我们看看步骤：

1.  定义`Post`的结构：

    ```rs
    #[derive(sqlx::Type, Debug, FromFormField)]
    #[repr(i32)]
    enum PostType {
        Text = 0,
        Photo = 1,
        Video = 2,
    }
    #[derive(FromForm)]
    struct Post {
        uuid: Uuid,
        user_uuid: Uuid,
        post_type: PostType,
        content: String,
        created_at: OurDateTime,
    }
    ```

我们希望区分类型，因此我们添加了 `post_type` 列。我们还想在用户和帖子之间建立关系。因为我们希望用户能够创建许多帖子，我们可以在结构体中创建一个 `user_uuid` 字段。内容将用于存储文本内容或存储上传文件的文件路径。我们将在应用程序实现时处理数据迁移。

1.  每个帖子在 HTML 上的呈现方式可能不同，但它们将在网页上占据相同的位置，所以让我们创建一个 `DisplayPostContent` 特性和三个 `DisplayPostContent` 用于每个新类型：

    ```rs
    trait DisplayPostContent {
        fn raw_html() -> String;
    }
    struct TextPost(Post);
    impl DisplayPostContent for TextPost {
        fn raw_html() -> String {
            todo!("will implement later")
        }
    }
    struct PhotoPost(Post);
    impl DisplayPostContent for PhotoPost {
        fn raw_html() -> String {
            todo!("will implement later")
        }
    }
    struct VideoPost(Post);
    impl DisplayPostContent for VideoPost {
        fn raw_html() -> String {
            todo!("will implement later")
        }
    }
    ```

1.  最后，我们可以添加处理 `Post` 的路由。我们可以创建 `get_post`、`get_posts`、`create_post` 和 `delete_post`。我们还希望这些路由位于用户下：

    ```rs
    #[get("/users/<_user_uuid>/posts/<_uuid>", format = "text/html")]
    async fn get_post(mut _db: Connection<DBConnection>, _user_uuid: &str, _uuid: &str) -> HtmlResponse {
        todo!("will implement later")
    }
    #[get("/users/<_user_uuid>/posts?<_pagination>", format = "text/html")]
    async fn get_posts(
        mut _db: Connection<DBConnection>,
        _user_uuid: &str,
        _pagination: Option<Pagination>,
    ) -> HtmlResponse {
        todo!("will implement later")
    }
    #[post("/users/<_user_uuid>/posts", format = "text/html", data = "<_upload>")]
    async fn create_post(
        mut _db: Connection<DBConnection>,
        _user_uuid: &str,
        _upload: Form<Post>,
    ) -> HtmlResponse {
        todo!("will implement later")
    }
    #[delete("/users/<_user_uuid>/posts/<_uuid>", format = "text/html")]
    async fn delete_post(
        mut _db: Connection<DBConnection>,
        _user_uuid: &str,
        _uuid: &str,
    ) -> HtmlResponse {
        todo!("will implement later")
    }
    ```

在添加与帖子相关的类型和函数后，我们可以在下一小节中最终创建应用程序骨架。

## 完成应用程序

不要忘记将这些路由添加到 Rocket 初始化过程中：

```rs
async fn rocket() -> Rocket<Build> {
```

```rs
   rocket::build().attach(DBConnection::init()).mount(
```

```rs
        "/",
```

```rs
        routes![
```

```rs
            get_user,
```

```rs
            get_users,
```

```rs
            new_user,
```

```rs
            create_user,
```

```rs
            edit_user,
```

```rs
            put_user,
```

```rs
            patch_user,
```

```rs
            delete_user,
```

```rs
            get_post,
```

```rs
            get_posts,
```

```rs
            create_post,
```

```rs
            delete_post,
```

```rs
        ],
```

```rs
    )
```

```rs
}
```

我们还希望通过路由提供上传的文件：

```rs
use rocket::fs::{NamedFile, TempFile};
```

```rs
...
```

```rs
#[get("/<_filename>")]
```

```rs
async fn assets(_filename: &str) -> NamedFile {
```

```rs
    todo!("will implement later")
```

```rs
}
```

```rs
async fn rocket() -> Rocket<Build> {
```

```rs
    rocket::build()
```

```rs
        ...
```

```rs
        .mount("/assets", routes![assets])
```

```rs
}
```

是时候添加我们的默认错误处理了！其他框架通常为 HTTP 状态码 `404`、`422` 和 `500` 提供默认错误处理器。让我们为这些代码创建一个处理器：

```rs
use rocket::request::Request;
```

```rs
...
```

```rs
#[catch(404)]
```

```rs
fn not_found(_: &Request) -> RawHtml<String> {
```

```rs
    todo!("will implement later")
```

```rs
}
```

```rs
#[catch(422)]
```

```rs
fn unprocessable_entity(_: &Request) -> RawHtml<String> {
```

```rs
    todo!("will implement later")
```

```rs
}
```

```rs
#[catch(500)]
```

```rs
fn internal_server_error(_: &Request) -> RawHtml<String> {
```

```rs
    todo!("will implement later")
```

```rs
}
```

```rs
async fn rocket() -> Rocket<Build> {
```

```rs
    rocket::build()
```

```rs
        ...
```

```rs
        .register(
```

```rs
            "/",
```

```rs
catchers![not_found, unprocessable_entity, 
```

```rs
            internal_server_error],
```

```rs
        )
```

```rs
}
```

当我们使用 Cargo 的 `run` 命令运行应用程序时，应用程序应该能够正确启动。但是，当我们查看 `src/main.rs` 文件时，该文件包含许多函数和类型定义。我们将在下一节中将我们的应用程序模块化。

# 模块化 Rocket 应用程序

记得在 *第一章*，*介绍 Rust 语言*，当我们使用模块创建应用程序时？应用程序源代码的一个功能是将其用作应用程序开发人员的文档。易于阅读的代码可以轻松进一步开发并与团队中的其他人共享。

编译器不关心程序是在一个文件中还是多个文件中；生成的应用程序二进制文件是相同的。然而，在单个长文件上工作的程序员很容易感到困惑。

我们将把应用程序源代码拆分成更小的文件，并将文件分类到不同的模块中。程序员来自不同的背景，他们可能有自己关于如何拆分应用程序源代码的范式。例如，习惯于编写 Java 程序的程序员可能更喜欢根据逻辑实体或类组织代码。习惯于模型-视图-控制器（MVC）框架的人可能更喜欢将文件放在模型、视图和控制器文件夹中。习惯于清洁架构的人可能会尝试将代码组织成层。但最终，真正重要的是你组织代码的方式被你合作的人接受，并且他们都能舒适且容易地使用相同的源代码。

Rocket 没有关于如何组织代码的具体指南，但我们可以观察到两个可以用来模块化应用程序的东西。第一个是`Cargo`项目包布局约定，第二个是 Rocket 组件本身。

根据`Cargo`文档([`doc.rust-lang.org/cargo/guide/project-layout.html`](https://doc.rust-lang.org/cargo/guide/project-layout.html))，包布局应该是这样的：

```rs
┌── Cargo.lock
```

```rs
├── Cargo.toml
```

```rs
├── src/
```

```rs
│   ├── lib.rs
```

```rs
│   ├── main.rs
```

```rs
│   └── bin/
```

```rs
│       ├── named-executable.rs
```

```rs
│       ├── another-executable.rs
```

```rs
│       └── multi-file-executable/
```

```rs
│           ├── main.rs
```

```rs
│           └── some_module.rs
```

```rs
├── benches/
```

```rs
│   ├── large-input.rs
```

```rs
│   └── multi-file-bench/
```

```rs
│       ├── main.rs
```

```rs
│       └── bench_module.rs
```

```rs
├── examples/
```

```rs
│   ├── simple.rs
```

```rs
│   └── multi-file-example/
```

```rs
│       ├── main.rs
```

```rs
│       └── ex_module.rs
```

```rs
└── tests/
```

```rs
    ├── some-integration-tests.rs
```

```rs
    └── multi-file-test/
```

```rs
        ├── main.rs
```

```rs
        └── test_module.rs
```

由于我们还没有基准测试、示例或测试，让我们专注于`src`文件夹。我们可以将应用程序拆分为`src/main.rs`中的可执行文件和`src/lib.rs`中的库。在可执行项目中，制作一个只调用库的小型可执行代码是非常常见的。

我们已经知道 Rocket 有不同部分，所以将 Rocket 组件拆分到它们自己的模块中是个好主意。让我们将源代码组织到这些文件和文件夹中：

```rs
┌── Cargo.lock
```

```rs
├── Cargo.toml
```

```rs
└── src/
```

```rs
    ├── lib.rs
```

```rs
    ├── main.rs
```

```rs
    ├── catchers
```

```rs
    │   └── put catchers modules here
```

```rs
    ├── fairings
```

```rs
    │   └── put fairings modules here
```

```rs
    ├── models
```

```rs
    │   └── put requests, responses, and database related modules here
```

```rs
    ├── routes
```

```rs
    │   └── put route handling functions and modules here
```

```rs
    ├── states
```

```rs
    │   └── put states modules here
```

```rs
    ├── traits
```

```rs
    │   └── put our traits here
```

```rs
    └── views
```

```rs
        └── put our templates here
```

1.  首先，编辑`Cargo.toml`文件：

    ```rs
    [package]
    ...
    [[bin]]
    name = "our_application"
    path = "src/main.rs"
    [lib]
    name = "our_application"
    path = "src/lib.rs"
    [dependencies]
    ...
    ```

1.  创建`src/lib.rs`文件以及以下文件夹：`src/catchers`、`src/fairings`、`src/models`、`src/routes`、`src/states`、`src/traits`和`src/views`。

1.  之后，在每个文件夹内创建一个`mod.rs`文件：`src/catchers/mod.rs`、`src/fairings/mod.rs`、`src/models/mod.rs`、`src/routes/mod.rs`、`src/states/mod.rs`、`src/traits/mod.rs`和`src/views/mod.rs`。

1.  然后，编辑`src/lib.rs`：

    ```rs
    #[macro_use]
    extern crate rocket;
    pub mod catchers;
    pub mod fairings;
    pub mod models;
    pub mod routes;
    pub mod states;
    pub mod traits;
    ```

1.  首先编写数据库连接。编辑`src/fairings/mod.rs`：

    ```rs
    pub mod db;
    ```

1.  创建一个新文件，`src/fairings/db.rs`，并像在`src/main.rs`中之前定义的连接一样编写文件：

    ```rs
    use rocket_db_pools::{sqlx::PgPool, Database};
    #[derive(Database)]
    #[database("main_connection")]
    pub struct DBConnection(PgPool);
    ```

注意，我们只使用了比`src/main.rs`更少的模块。我们还添加了`pub`关键字，以便从其他模块或`src/main.rs`中访问结构体。

1.  因为特质将要被结构体使用，我们需要先定义特质。在`src/traits/mod.rs`中，从`src/main.rs`复制特质：

    ```rs
    pub trait DisplayPostContent {
        fn raw_html() -> String;
    }
    ```

1.  之后，让我们将所有用于请求和响应的结构体移动到`src/models`文件夹中。按照以下方式编辑`src/models/mod.rs`：

    ```rs
    pub mod our_date_time;
    pub mod pagination;
    pub mod photo_post;
    pub mod post;
    pub mod post_type;
    pub mod text_post;
    pub mod user;
    pub mod user_status;
    pub mod video_post;
    ```

1.  然后，创建文件并将`src/main.rs`中的定义复制到这些文件中。第一个是`src/models/our_date_time.rs`：

    ```rs
    use chrono::{offset::Utc, DateTime};
    use rocket::form::{self, DataField, FromFormField, ValueField};
    #[derive(Debug)]
    pub struct OurDateTime(DateTime<Utc>);
    #[rocket::async_trait]
    impl<'r> FromFormField<'r> for OurDateTime {
        fn from_value(_: ValueField<'r>) -> form::
        Result<'r, Self> {
            todo!("will implement later")
        }
        async fn from_data(_: DataField<'r, '_>) -> 
        form::Result<'r, Self> {
            todo!("will implement later")
        }
    }
    ```

1.  接下来是`src/models/pagination.rs`：

    ```rs
    use super::our_date_time::OurDateTime;
    #[derive(FromForm)]
    pub struct Pagination {
        pub next: OurDateTime,
        pub limit: usize,
    }
    ```

注意`use`声明使用了`super`关键字。Rust 模块按照层次结构组织，一个模块包含其他模块。`super`关键字用于访问包含当前模块的模块。`super`关键字可以链式使用，例如，`use super::super::SomeModule;`。

1.  之后，编写`src/models/post_type.rs`：

    ```rs
    use rocket::form::FromFormField;
    use rocket_db_pools::sqlx;
    #[derive(sqlx::Type, Debug, FromFormField)]
    #[repr(i32)]
    pub enum PostType {
        Text = 0,
        Photo = 1,
        Video = 2,
    }
    ```

1.  还要编写`src/models/post.rs`：

    ```rs
    use super::our_date_time::OurDateTime;
    use super::post_type::PostType;
    use rocket::form::FromForm;
    use uuid::Uuid;
    #[derive(FromForm)]
    pub struct Post {
        pub uuid: Uuid,
        pub user_uuid: Uuid,
        pub post_type: PostType,
        pub content: String,
        pub created_at: OurDateTime,
    }
    ```

然后，编写`src/models/user_status.rs`：

```rs
use rocket::form::FromFormField;
use rocket_db_pools::sqlx;
#[derive(sqlx::Type, Debug, FromFormField)]
#[repr(i32)]
pub enum UserStatus {
    Inactive = 0,
    Active = 1,
}
```

1.  编写`src/models/user.rs`：

    ```rs
    use super::our_date_time::OurDateTime;
    use super::user_status::UserStatus;
    use rocket::form::FromForm;
    use rocket_db_pools::sqlx::FromRow;
    use uuid::Uuid;
    #[derive(Debug, FromRow, FromForm)]
    pub struct User {
        pub uuid: Uuid,
        pub username: String,
        pub email: String,
        pub password_hash: String,
        pub description: Option<String>,
        pub status: UserStatus,
        pub created_at: OurDateTime,
        pub updated_at: OurDateTime,
    }
    ```

然后，编写三个`post`新类型，`src/models/photo_post.rs`、`src/models/text_post.rs`和`src/models/video_post.rs`：

```rs
use super::post::Post;
use crate::traits::DisplayPostContent;
pub struct PhotoPost(Post);
impl DisplayPostContent for PhotoPost {
    fn raw_html() -> String {
        todo!("will implement later")
    }
}
use super::post::Post;
use crate::traits::DisplayPostContent;
pub struct TextPost(Post);
impl DisplayPostContent for TextPost {
    fn raw_html() -> String {
        todo!("will implement later")
    }
}
use super::post::Post;
use crate::traits::DisplayPostContent;
pub struct VideoPost(Post);
impl DisplayPostContent for VideoPost {
    fn raw_html() -> String {
        todo!("will implement later")
    }
}
```

在所有三个文件中，我们在`use`声明中使用`crate`关键字。我们之前已经讨论过`super`关键字；`crate`关键字是指我们正在工作的当前库，即`our_application`库。在 Rust 2015 版本中，它写作双分号（`::`），但自从 Rust 2018 版本以来，`::`变为了`crate`。现在，`::`表示外部 crate 的根路径，例如，`::rocket::fs::NamedFile;`。

除了`super`、`::`和`crate`之外，还有一些其他的`use`声明：`self`和`Self`。我们可以使用`self`来避免在代码中引用项目时的歧义，如下面的例子所示：

```rs
use super::haha;
mod a {
    fn haha() {}
    fn other_func() {
        self::haha();
    }
}
```

`Self`用于在特性中引用关联类型，如下面的例子所示：

```rs
trait A {
  type Any;
  fn any(&self) -> Self::Any;
}
struct B;
impl A for B {
  type Any = usize;
  fn any(&self) -> self::Any {
    100
  }
}
```

1.  现在，让我们回到应用程序骨架。在所有结构体之后，是时候为应用程序编写路由了。修改`src/routes/mod.rs`：

    ```rs
    use rocket::fs::NamedFile;
    use rocket::http::Status;
    use rocket::response::content::RawHtml;
    pub mod post;
    pub mod user;
    type HtmlResponse = Result<RawHtml<String>, Status>;
    #[get("/<_filename>")]
    pub async fn assets(_filename: &str) -> NamedFile {
        todo!("will implement later")
    }
    ```

我们可以将处理资产的函数放在它们自己的 Rust 文件中，但由于只有一个函数且非常简单，我们只需将函数放在`mod.rs`文件中。

1.  接下来，创建并编写`src/routes/post.rs`：

    ```rs
    use super::HtmlResponse;
    use crate::fairings::db::DBConnection;
    use crate::models::{pagination::Pagination, post::Post};
    use rocket::form::Form;
    use rocket_db_pools::Connection;
    #[get("/users/<_user_uuid>/posts/<_uuid>", format = "text/html")]
    pub async fn get_post(
        mut _db: Connection<DBConnection>,
        _user_uuid: &str,
        _uuid: &str,
    ) -> HtmlResponse {
        todo!("will implement later")
    }
    #[get("/users/<_user_uuid>/posts?<_pagination>", format = "text/html")]
    pub async fn get_posts(
        mut _db: Connection<DBConnection>,
        _user_uuid: &str,
        _pagination: Option<Pagination>,
    ) -> HtmlResponse {
        todo!("will implement later")
    }
    #[post("/users/<_user_uuid>/posts", format = "text/html", data = "<_upload>")]
    pub async fn create_post(
        mut _db: Connection<DBConnection>,
        _user_uuid: &str,
        _upload: Form<Post>,
    ) -> HtmlResponse {
        todo!("will implement later")
    }
    #[delete("/users/<_user_uuid>/posts/<_uuid>", format = "text/html")]
    pub async fn delete_post(
        mut _db: Connection<DBConnection>,
        _user_uuid: &str,
        _uuid: &str,
    ) -> HtmlResponse {
        todo!("will implement later")
    }
    ```

1.  创建并编写`src/routes/user.rs`：

    ```rs
    use super::HtmlResponse;
    use crate::fairings::db::DBConnection;
    use crate::models::{pagination::Pagination, user::User};
    use rocket::form::Form;
    use rocket_db_pools::Connection;
    #[get("/users/<_uuid>", format = "text/html")]
    pub async fn get_user(mut _db: Connection<DBConnection>, _uuid: &str) -> HtmlResponse {
        todo!("will implement later")
    }
    #[get("/users?<_pagination>", format = "text/html")]
    pub async fn get_users(
        mut _db: Connection<DBConnection>,
        _pagination: Option<Pagination>,
    ) -> HtmlResponse {
        todo!("will implement later")
    }
    #[get("/users/new", format = "text/html")]
    pub async fn new_user(mut _db: Connection<DBConnection>) -> HtmlResponse {
        todo!("will implement later")
    }
    #[post("/users", format = "text/html", data = "<_user>")]
    pub async fn create_user(mut _db: Connection<DBConnection>, _user: Form<User>) -> HtmlResponse {
        todo!("will implement later")
    }
    #[get("/users/edit/<_uuid>", format = "text/html")]
    pub async fn edit_user(mut _db: Connection<DBConnection>, _uuid: &str) -> HtmlResponse {
        todo!("will implement later")
    }
    #[put("/users/<_uuid>", format = "text/html", data = "<_user>")]
    pub async fn put_user(
        mut _db: Connection<DBConnection>,
        _uuid: &str,
        _user: Form<User>,
    ) -> HtmlResponse {
        todo!("will implement later")
    }
    #[patch("/users/<_uuid>", format = "text/html", data = "<_user>")]
    pub async fn patch_user(
        mut _db: Connection<DBConnection>,
        _uuid: &str,
        _user: Form<User>,
    ) -> HtmlResponse {
        todo!("will implement later")
    }
    #[delete("/users/<_uuid>", format = "text/html")]
    pub async fn delete_user(mut _db: Connection<DBConnection>, _uuid: &str) -> HtmlResponse {
        todo!("will implement later")
    }
    ```

1.  为了最终完成库，请在`src/catchers/mod.rs`中添加捕获器：

    ```rs
    use rocket::request::Request;
    use rocket::response::content::RawHtml;
    #[catch(404)]
    pub fn not_found(_: &Request) -> RawHtml<String> {
        todo!("will implement later")
    }
    #[catch(422)]
    pub fn unprocessable_entity(_: &Request) -> RawHtml<String> {
        todo!("will implement later")
    }
    #[catch(500)]
    pub fn internal_server_error(_: &Request) -> RawHtml<String> {
        todo!("will implement later")
    }
    ```

1.  当库准备就绪时，我们可以修改`src/main.rs`本身：

    ```rs
    #[macro_use]
    extern crate rocket;
    use our_application::catchers;
    use our_application::fairings::db::DBConnection;
    use our_application::routes::{self, post, user};
    use rocket::{Build, Rocket};
    use rocket_db_pools::Database;
    #[launch]
    async fn rocket() -> Rocket<Build> {
        rocket::build()
            .attach(DBConnection::init())
            .mount(
                "/",
                routes![
                    user::get_user,
                    user::get_users,
                    user::new_user,
                    user::create_user,
                    user::edit_user,
                    user::put_user,
                    user::patch_user,
                    user::delete_user,
                    post::get_post,
                    post::get_posts,
                    post::create_post,
                    post::delete_post,
                ],
            )
            .mount("/assets", routes![routes::assets])
            .register(
                "/",
                catchers![
                    catchers::not_found,
                    catchers::unprocessable_entity,
                    catchers::internal_server_error
                ],
            )
    }
    ```

我们的`src/main.rs`文件变得更加简洁。

现在，如果我们想添加更多的结构体或路由，我们可以在相应的文件夹中轻松地添加新的模块。我们还可以添加更多的状态或公平性，并轻松找到这些项目的文件位置。

# 摘要

在本章中，我们学习了如何设计应用程序、创建 Rocket 应用程序骨架以及将 Rust 应用程序组织成更小的可管理模块。

我们还学习了诸如 CRUD 和 RESTful 应用程序、Rust `enum`区分符以及 Rust 路径限定符等概念。

希望在阅读本章之后，您可以将这些概念应用到帮助您更好地组织代码中。

我们将在接下来的章节中开始实现这个应用程序，并学习更多关于 Rust 和 Rocket 概念，如模板、请求守卫、cookies 和 JSON。
