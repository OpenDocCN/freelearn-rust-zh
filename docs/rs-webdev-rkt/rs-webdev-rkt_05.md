# *第四章*：构建、点燃和发射 Rocket

许多网络应用程序需要某种可以反复使用的对象管理，无论是数据库服务器的连接池、内存存储的连接、第三方服务器的 HTTP 客户端，还是任何其他对象。网络应用程序中的另一个常见功能是**中间件**。

本章中，我们将讨论两个 Rocket 功能（状态和火箭头），它们作为 Rocket 的可重用对象管理和中间件部分。我们还将学习如何创建和使用数据库服务器的连接，这在几乎所有网络应用程序中都非常重要。

完成本章后，我们希望您能够使用和实现 Rocket 网络框架的可重用对象管理和中间件部分。我们还希望您能够连接到您自己选择的数据库。

本章我们将涵盖以下主要主题：

+   管理状态

+   与数据库一起工作

+   安装 Rocket 火箭头

# 技术要求

除了 Rust 编译器、文本编辑器和 HTTP 客户端等常规要求外，从本章开始，我们将与数据库一起工作。本书中我们将使用的数据库是 PostgreSQL，您可以从[`www.postgresql.org/`](https://www.postgresql.org/)下载它，通过您的操作系统包管理器安装它，或使用第三方服务器，如**亚马逊网络服务**（**AWS**）、Microsoft Azure 或**谷歌云平台**（**GCP**）。

我们将看到如何连接到其他**关系数据库管理系统**（**RDBMSs**）如 SQLite、MySQL 或 Microsoft SQL Server，您可以将课程代码调整为适合这些 RDBMS，但使用 PostgreSQL 更容易理解。

您可以在[`github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter04`](https://github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter04)找到本章的源代码。

# 管理状态

在网络应用程序中，通常程序员需要创建一个在请求/响应生命周期中可以重复使用的对象。在 Rocket 网络框架中，这个对象被称为**状态**。状态可以是任何东西，例如数据库连接池、存储各种客户统计信息的对象、存储到内存存储的连接的对象、用于发送**简单邮件传输协议**（**SMTP**）电子邮件的客户端，等等。

我们可以告诉 Rocket 维护状态，这被称为**管理状态**。创建管理状态的过程相当简单。我们需要初始化一个对象，告诉 Rocket 管理它，并在路由中使用它。一个注意事项是，我们可以从不同类型管理多个状态，但 Rocket 只能管理 Rust 类型的单个实例。

让我们直接尝试一下。我们将有一个访问计数器状态，并告诉 Rocket 管理它，并为每个传入的请求增加计数器。我们可以重用前一章中的应用程序，将 `Chapter03/15ErrorCatcher` 中的程序复制到 `Chapter04/01State` 中，并在 `Cargo.toml` 中将应用程序重命名为 `chapter4`。

在 `src/main.rs` 中，定义一个结构体来保存访问计数器的值。为了状态能够工作，要求是 `T: Send + Sync + 'static`：

```rs
use std::sync::atomic::AtomicU64;
```

```rs
...
```

```rs
struct VisitorCounter {
```

```rs
    visitor: AtomicU64,
```

```rs
}
```

我们已经知道 `'static` 是一个生命周期标记，但 `Send + Sync` 是什么？

在现代计算中，由于其复杂性，程序可能会以非预期的方式执行。例如，多线程使得很难知道变量的值是否在另一个线程上被更改。现代 CPU 也执行分支预测，并同时执行多个指令。复杂的编译器也会重新排列生成的二进制代码执行流程以优化结果。为了克服这些问题，在 Rust 语言中需要某种同步机制。

Rust 语言有特性和内存容器来解决同步问题，这取决于程序员如何期望应用程序工作。我们可能想在堆中创建一个对象，并在多个其他对象中共享对该对象的引用。例如，我们创建对象 `x`，并在其他对象的字段 `y` 和 `z` 中使用 `x` 的引用 `&x`。这会创建另一个问题，因为程序可以在其他例程中删除 `x`，从而使程序变得不稳定。解决方案是为不同的用例创建不同的容器。这些包括 `std::cell::Rc` 和 `std::box::Box` 等。

`std::marker::Send` 是这些特性之一。`Send` 特性确保任何 `T` 类型在转移到另一个线程时是安全的。几乎 `std` 库中的所有类型都是 `Send`，但有少数例外，例如 `std::rc::Rc` 和 `std::cell::UnsafeCell`。Rc 是一个单线程的引用计数指针。

同时，`std::marker::Sync` 表示 `T` 类型可以在多个线程之间安全共享。只有当 `&T` 引用可以安全地发送到另一个线程时，这才会成立。并非所有 `Send` 类型都是 `Sync`。例如，`std::cell::Cell` 和 `std::cell::RefCell` 是 `Send` 但不是 `Sync`。

`Both` `Send + Sync` 都是 `Send + Sync` 吗？这些类型自动成为 `Send` 类型。除了原始指针、`Rc`、`Cell` 和 `RefCell` 之外，`std` 库中的几乎所有类型都是 `Send + Sync`。

`AtomicU64`是什么？使用常规的`u64`类型，尽管它是`Send + Sync`，但线程之间没有同步，因此可能会发生数据竞争条件。例如，两个线程同时访问相同的变量`x`（其值为`64`），并且它们将值增加一。我们期望结果是`66`（因为有两条线程），但由于线程之间没有同步，最终结果是不可预测的。它可以是`65`或`66`。

`std::sync`模块中的类型提供了一些在多个线程之间共享更新的方法，包括`std::sync::Mutex`、`std::sync::RwLock`和`std::sync::atomic`类型。我们还可以使用可能比标准库提供更好速度的其他库，例如`parking_lot`crate。

现在我们已经定义了`VisitorCounter`，让我们初始化它并告诉 Rocket 将其作为状态管理。在`rocket()`函数中写入以下代码：

```rs
fn rocket() -> Rocket<Build> {
```

```rs
    let visitor_counter = VisitorCounter {
```

```rs
        visitor: AtomicU64::new(0),
```

```rs
    };
```

```rs
    rocket::build()
```

```rs
        .manage(visitor_counter)
```

```rs
        .mount("/", routes![user, users, favicon])
```

```rs
        .register("/", catchers![not_found, forbidden])
```

```rs
}
```

在我们告诉 Rocket 管理状态之后，我们可以在路线处理函数中使用它。在前一章中，我们学习了必须在使用`function`参数时使用的动态段。在路线处理函数中，我们还可以使用其他参数，我们称之为**请求守卫**。它们被称为*守卫*，因为如果请求没有通过守卫内的验证，请求将被拒绝，并返回错误响应。

任何实现了`rocket::request::FromRequest`的类型都可以被视为请求守卫。传入的请求将与从左到右的每个请求守卫进行验证，如果请求无效，将短路并返回错误。

假设我们有一个如下所示的路线处理函数：

```rs
#[get("/<param1>")]
```

```rs
fn handler1(param1: u8, type1: Guard1, type2: Guard2) {}
```

`Guard1`和`Guard2`类型是请求守卫。然后，传入的请求将与`Guard1`方法进行验证，如果发生错误，将立即返回适当的错误响应。

我们将在整本书中学习并实现请求守卫，但在这个章节中，我们将只使用请求守卫而不进行实现。`FromRequest`已经为`rocket::State<T>`实现，因此我们可以在路线处理函数中使用它。

现在我们已经学习了为什么在路线处理函数中使用`State`，让我们在我们的函数中使用它。我们想要设置访问者计数器，以便每个请求都应该增加计数器：

```rs
use rocket::{Build, Rocket, State};
```

```rs
#[get("/user/<uuid>", rank = 1, format = "text/plain")]
```

```rs
fn user<'a>(counter: &State<VisitorCounter>, uuid: &'a str) -> Option<&'a User> {
```

```rs
    counter.visitor.fetch_add(1, Ordering::Relaxed);
```

```rs
    println!("The number of visitor is: {}", counter.
```

```rs
    visitor.load(Ordering::Relaxed));
```

```rs
    USERS.get(uuid)
```

```rs
}
```

为什么我们添加`'a`生命周期？我们正在添加一个新的引用参数，而 Rust 无法推断返回的`&User`应该遵循哪个生命周期。在这种情况下，我们表示`User`引用的生命周期应该与`uuid`相同。

在函数内部，我们使用 AtomicU64 的`fetch_add()`方法来增加访问者的值，并使用 AtomicU64 的`load()`方法打印该值。

让我们在`users()`函数中也添加相同的处理，但由于我们与`user()`函数有完全相同的流程，所以让我们创建另一个函数：

```rs
impl VisitorCounter {
```

```rs
    fn increment_counter(&self) {
```

```rs
        self.visitor.fetch_add(1, Ordering::Relaxed);
```

```rs
        println!(
```

```rs
            "The number of visitor is: {}",
```

```rs
            self.visitor.load(Ordering::Relaxed)
```

```rs
        );
```

```rs
    }
```

```rs
}
```

```rs
...
```

```rs
fn user<'a>(counter: &State<VisitorCounter>, uuid: &'a str) -> Option<&'a User> {
```

```rs
    counter.increment_counter();
```

```rs
    ...
```

```rs
}
```

```rs
...
```

```rs
fn users<'a>(
```

```rs
    counter: &State<VisitorCounter>,
```

```rs
    name_grade: NameGrade,
```

```rs
    filters: Option<Filters>,
```

```rs
) -> Result<NewUser<'a>, Status> {
```

```rs
    counter.increment_counter();
```

```rs
    ...
```

```rs
}
```

此示例与`Atomic`类型配合良好，但如果您需要处理更复杂的数据类型，例如`String`、`Vec`或`Struct`，请尝试使用标准库或第三方 crate（如`parking_lot`）中的`Mutex`或`RwLock`。

现在我们已经知道了在 Rocket 中`State`是什么，让我们通过结合数据库服务器来扩展我们的应用程序。我们将使用`State`来存储数据库连接。

# 与数据库一起工作

目前，在我们的应用程序中，我们正在使用静态变量存储用户数据。这非常麻烦，因为它不够灵活，我们无法轻松更新数据。大多数现代处理数据的应用程序将使用某种持久存储，无论是基于文件系统的存储、面向文档的数据库还是传统的 RDBMS。

Rust 有许多库可以连接到各种数据库或类似存储。有`postgres`crate，它是 Rust 的 PostgreSQL 客户端。还有其他客户端，如`mongodb`和`redis`。对于`diesel`，它可以连接到各种数据库系统。对于连接池管理，有`deadpool`和`r2d2`crate。所有 crate 都有其优势和局限性，例如没有异步应用程序。

在本书中，我们将使用`sqlx`连接到 RDBMS。`sqlx`声称是一个 Rust 的 SQL 工具包。它为客户端提供了连接到各种 RDBMS 的抽象，它有一个连接池特质，并且还可以用于将类型转换为查询以及将查询响应转换为 Rust 类型。

如本章*技术要求*部分所述，我们将使用 PostgreSQL 作为我们的 RDBMS，因此请准备 PostgreSQL 的连接信息。

之后，按照以下步骤将我们的应用程序转换为使用数据库：

1.  我们将再次使用我们的应用程序。我们首先想做的事情是在终端中输入以下命令来安装`sqlx-cli`：

    ```rs
    cargo install sqlx-cli
    ```

`sqlx-cli`是一个有用的命令行应用程序，用于创建数据库、创建迁移和运行迁移。它不如其他成熟框架中的迁移工具复杂，但它非常有效地完成了工作。

1.  准备连接信息，并在您的终端中设置`DATABASE_URL`环境变量。`DATABASE_URL`的格式应如下所示，具体取决于您使用的 RDBMS：

    ```rs
    postgres://username:password@localhost:port/db_name?connect_options
    ```

对于`connect_options`，它以查询形式存在，相关信息可以在[`www.postgresql.org/docs/current/libpq-connect.html#LIBPQ-CONNSTRING`](https://www.postgresql.org/docs/current/libpq-connect.html#LIBPQ-CONNSTRING)找到。其他 RDBMS 的`DATABASE_URL`格式可能如下所示：

```rs
mysql://username:password@localhost:port/db_name
```

或者`sqlite::memory:`或`sqlite://path/to/file.db?connect_options`或`sqlite:///path/to/file.db?connect_options`。SQLite 的连接选项可以在[`www.sqlite.org/uri.html`](https://www.sqlite.org/uri.html)找到。

1.  使用以下命令创建一个新的数据库：

    ```rs
    sqlx database create
    ```

1.  我们可以使用以下命令创建一个名为`create_users`的迁移：

    ```rs
    sqlx migrate add create_users
    ```

1.  `sqlx` CLI 将在我们的应用程序根目录内创建一个名为 `migrations` 的新文件夹，在该文件夹内，将有一个符合 `timestamp_migration_name.sql` 模式的文件。在我们的示例中，文件名将看起来像 `migrations/20210923055406_create_users.sql`。在文件中，我们可以编写 SQL 查询来创建或修改 `User` 结构体，所以让我们将以下代码写入 SQL 文件：

    ```rs
    CREATE TABLE IF NOT EXISTS users
    (
        uuid   UUID PRIMARY KEY,
        name   VARCHAR NOT NULL,
        age    SMALLINT NOT NULL DEFAULT 0,
        grade  SMALLINT NOT NULL DEFAULT 0,
        active BOOL NOT NULL DEFAULT TRUE
    );
    CREATE INDEX name_active_idx ON users(name, active);
    ```

我们如何知道数据库列类型和 Rust 类型之间的映射关系？`sqlx` 提供了自己的映射；我们可以在 [`docs.rs/sqlx`](https://docs.rs/sqlx) 找到相关文档。该包为支持的数据库提供了优秀的模块。我们可以在顶部的搜索栏中搜索它；例如，我们可以在 [`docs.rs/sqlx/0.5.7/sqlx/postgres/index.html`](https://docs.rs/sqlx/0.5.7/sqlx/postgres/index.html) 找到 PostgreSQL 的文档。在该页面上，我们可以看到有 `types` 模块，我们可以查看它们。

1.  在我们编写迁移文件的内容后，我们可以使用以下命令运行迁移：

    ```rs
    sqlx migrate run
    ```

1.  迁移后，检查生成的数据库表，看看表模式是否正确。让我们插入上一章的数据。也可以随意用您选择的数据填充表格。

1.  迁移后，将 `sqlx` 包包含在我们的 `Cargo.toml` 文件中。由于我们将使用 PostgreSQL 的 `uuid` 类型，我们还应该包含 `uuid` 包。如果您想启用其他 RDBMS，请查看该包的 API 文档：

    ```rs
    sqlx = {version = "0.5.7", features = ["postgres", "uuid", "runtime-tokio-rustls"]}
    uuid = "0.8.2"
    ```

1.  我们可以从 `Cargo.toml` 中删除 `lazy_static`，并从 `src/main.rs` 文件中移除 `lazy_static!`、`USERS` 和 `HashMap` 的引用。我们不需要这些，我们只将从我们之前插入的数据库中检索 `User` 数据。使用以下 `SQL INSERT` 语法插入之前用户的数据：

    ```rs
    INSERT INTO public.users
    (uuid, name, age, grade, active)
    VALUES('3e3dd4ae-3c37-40c6-aa64-7061f284ce28'::uuid, 'John Doe', 18, 1, true);
    ```

1.  修改 `User` 结构体以符合我们创建的数据库：

    ```rs
    use sqlx::FromRow;
    use uuid::Uuid;
    ...
    #[derive(Debug, FromRow)]
    struct User {
        uuid: Uuid,
        name: String,
        age: i16,
        grade: i16,
        active: bool,
    }
    ```

当 `sqlx` 从数据库检索结果时，它将被存储在 `sqlx::Database::Row` 中。这种类型可以转换为任何实现了 `sqlx::FromRow` 的类型。幸运的是，只要所有成员实现了 `sqlx::Decode`，我们就可以推导出 `FromRow`。有一些异常情况，我们可以用来覆盖 `FromRow`。以下是一个示例：

```rs
#[derive(Debug, FromRow)]
#[sqlx(rename_all = "camelCase")]
struct User {
    uuid: Uuid,
    name: String,
    age: i16,
    grade: i16,
    #[sqlx(rename = "active")]
    present: bool,
    #[sqlx(default)]
    not_in_database: String,
}
```

对于 `rename_all`，我们可以使用以下选项：`snake_case`、`lowercase`、`UPPERCASE`、`camelCase`、`PascalCase`、`SCREAMING_SNAKE_CASE` 和 `kebab-case`。

当我们具有不同的列名和类型成员名时，使用 `rename`。如果一个成员在数据库中没有列，并且该类型实现了 `std::default::Default` 特性，我们可以使用 `default` 指令。

为什么我们使用 `i16`？答案是 PostgreSQL 类型没有映射到 Rust 的 `u8` 类型。我们可以使用 `i8`，使用更大的 `i16` 类型，或者尝试为 `u8` 实现 `Decode`。在这种情况下，我们选择使用 `i16` 类型。

我们希望程序从环境变量中读取连接信息 (`DATABASE_URL`)。在 *第二章*，*构建我们的第一个 Rocket Web 应用程序*，我们学习了如何使用标准配置来配置 Rocket，但这次，我们想添加额外的配置。我们可以从在 `Cargo.toml` 中将 `serde` 添加到应用程序依赖项开始。

`serde` 是 Rust 中使用最广泛且最重要的库之一。其名称来源于 *序列化和反序列化*。它用于涉及序列化和反序列化的任何事物。它可以用于将 Rust 类型实例转换为字节表示，反之亦然，转换为 JSON 和反之亦然，转换为 YAML，以及任何其他类型，只要它们实现了 `serde` 特性。它还可以用于将实现 `serde` 特性的一个类型转换为另一个实现 `serde` 特性的类型。

如果你想查看 `serde` 的文档，你可以在他们的网站上找到它，网址为 [`serde.rs`](https://serde.rs)。

`serde` 的文档中提到了许多对多种数据格式的原生或第三方支持，例如 JSON、Bincode、CBOR、YAML、MessagePack、TOML、Pickle、RON、BSON、Avro、JSON5、Postcard、URL 查询字符串、Envy、Envy Store、S-expressions、D-Bus 的二进制线格式和 FlexBuffers。

让我们在 `Cargo.toml` 中添加以下行以将 `serde` 包含到我们的应用程序中：

```rs
[dependencies]
...
serde = "1.0.130"
```

1.  之后，创建一个将用于包含我们自定义配置的结构体：

    ```rs
    use serde::Deserialize;
    ...
    #[derive(Deserialize)]
    struct Config {
        database_url: String,
    }
    ```

`serde` 已经提供了可以在 `derive` 属性中使用的 `Deserialize` 宏。到目前为止，我们已经使用了大量提供库的宏，这些库可以在 `derive` 属性中使用，例如 `Debug`、`FromRow`、`Deserialize`。宏系统是 Rust 的重要特性之一。

1.  实现读取配置并将其映射到 `rocket()` 函数的例程：

    ```rs
    fn rocket() -> Rocket<Build> {
        let our_rocket = rocket::build();
        let config: Config = our_rocket
            .figment()
            .extract()
            .expect("Incorrect Rocket.toml 
             configuration");
        ...
        our_rocket
            .manage(visitor_counter)
        ...
    }
    ```

1.  现在应用程序可以从环境变量中获取 `DATABASE_URL` 信息，是时候初始化数据库连接池并告诉 Rocket 管理它了。编写以下行：

    ```rs
    use sqlx::postgres::PgPoolOptions;
    ...
    async fn rocket() -> Rocket<Build> {
        ...
        let config: Config = rocket_frame
            .figment()
            .extract()
            .expect("Incorrect Rocket.toml 
            configuration");
        let pool = PgPoolOptions::new()
            .max_connections(5)
            .connect(&config.database_url)
            .await
            .expect("Failed to connect to database");
        ...
        rocket_frame
            .manage(visitor_counter)
            .manage(pool)
        ...
    }
    ```

我们使用 `PgPoolOptions` 初始化连接池。其他数据库可以使用它们对应的类型，例如 `sqlx::mysql::MySqlPoolOptions` 或 `sqlx::sqlite::SqlitePoolOptions`。`connect()` 方法是一个 `async` 方法，因此我们必须使 `rocket()` 也异步，以便能够使用结果。

之后，在 `rocket()` 函数内部，我们告诉 Rocket 管理连接池。

1.  在使用数据库连接之前，我们使用了 `lazy_static` 并创建了 `user` 对象作为 `USERS` 哈希表的引用。现在，我们将使用数据库中的数据，因此我们需要使用具体对象而不是引用。从 `User` 和 `NewUser` 结构体的 `Responder` 实现中移除 ampersand (`&`)：

    ```rs
    impl<'r> Responder<'r, 'r> for User { ... }
    struct NewUser(Vec<User>);
    impl<'r> Responder<'r, 'r> for NewUser { ... }
    ```

1.  现在，是时候实现 `user()` 函数以使用数据库连接池从数据库中查询了。按照以下方式修改 `user()` 函数：

    ```rs
    async fn user(
        counter: &State<VisitorCounter>,
        pool: &rocket::State<PgPool>,
        uuid: &str,
    ) -> Result<User, Status> {
        ...
        let parsed_uuid = Uuid::parse_str(uuid)
        .map_err(|_| Status::BadRequest)?;
        let user = sqlx::query_as!(
            User,
            "SELECT * FROM users WHERE uuid = $1",
            parsed_uuid
        )
        .fetch_one(pool.inner())
        .await;
        user.map_err(|_| Status::NotFound)
    }
    ```

我们在函数参数中包含了连接池管理状态。之后，我们将 UUID `&str` 参数解析为 `Uuid` 实例。如果解析 `uuid` 参数时发生错误，我们将错误更改为 `Status::BadRequest` 并返回错误。

然后，我们使用 `query_as!` 宏向数据库服务器发送查询并将结果转换为 `User` 实例。我们可以使用许多 `sqlx` 宏，例如 `query!`、`query_file!`、`query_as_unchecked!` 和 `query_file_as!`。您可以在我们之前提到的 `sqlx` API 文档中找到这些宏的文档。

使用此宏的格式如下：`query_as!(RustType, "prepared statement", bind parameter1, ...)`。如果您不需要将结果作为 Rust 类型获取，可以使用 `query!` 宏代替。

然后，我们使用 `fetch_one()` 方法。如果您想执行而不是查询，例如更新或删除行，可以使用 `execute()` 方法。如果您想获取所有结果，可以使用 `fetch_all()` 方法。您可以在 `sqlx::query::Query` 结构的文档中找到其他可用的方法和它们的文档。

我们可以选择保留 `user()` 函数返回的 `Option<User>` 并使用 `user.ok()`，或者我们将返回值更改为 `Status::SomeStatus`。由于我们将返回类型更改为 `Ok(user)` 或 `Err(some_error)`，我们可以直接返回 `Ok(user)` 变体，但我们需要使用 `map_err(|_| Status::NotFound)` 将错误更改为 `Status` 类型。

您可能正在想，如果我们向服务器发送原始 SQL 查询，是否可能进行 SQL 注入攻击？是否可能错误地获取任何用户输入并执行 `sqlx::query_as::<_, User>("SELECT * FROM users WHERE name = ?").bind``).fetch_one(pool.inner())`？

答案是否定的。`sqlx` 预编译并缓存了每个语句。由于使用预编译语句的结果，它比常规 SQL 查询更安全，并且返回的类型也是我们从 RDBMS 服务器期望的类型。

1.  让我们再修改一下 `users()` 函数。像 `user()` 函数一样，我们希望该函数是 `async` 并从 Rocket 管理状态中获取连接池。我们还想从 `NewUser` 中移除生命周期，因为我们不再引用 `USERS`：

    ```rs
    async fn users(
        counter: &State<VisitorCounter>,
        pool: &rocket::State<PgPool>,
        name_grade: NameGrade<'_>,
        filters: Option<Filters>,
    ) -> Result<NewUser, Status> {…}
    ```

1.  之后，我们可以准备预编译语句。如果客户端发送 `filters` 请求，我们将更多条件追加到 `WHERE` 子句中。对于 PostgreSQL，预编译语句使用 `$1`、`$2` 等等，但对于其他 RDBMS，您可以使用 `?` 作为预编译语句：

    ```rs
    ...
    let mut query_str = String::from("SELECT * FROM users WHERE name LIKE $1 AND grade = $2");
    if filters.is_some() {
        query_str.push_str(" AND age = $3 AND active = 
        $4");
    }
    ```

1.  接下来，编写执行查询的代码，但绑定参数的数量可能会根据是否存在过滤器而改变；我们使用`query_as`函数，这样我们就可以使用`if`分支。我们还添加了`%name%`作为名称绑定参数，因为我们使用了 SQL 语句中的`LIKE`运算符。我们还需要将`u8`类型转换为`i16`类型。最后，我们使用`fetch_all`方法检索所有结果。`query_as!`宏和查询函数的不错之处在于，它们都根据`fetch_one`或`fetch_all`返回`Vec<T>`或不是：

    ```rs
    ...
    let mut query = sqlx::query_as::<_, User>(&query_str)
        .bind(format!("%{}%", &name_grade.name))
        .bind(name_grade.grade as i16);
    if let Some(fts) = &filters {
        query = query.bind(fts.age as i16).bind(fts.
        active);
    }
    let unwrapped_users = query.fetch_all(pool.inner()).await;
    let users: Vec<User> = unwrapped_users.map_err(|_| Status::InternalServerError)?;
    ```

1.  我们可以像往常一样返回结果：

    ```rs
    ...
    if users.is_empty() {
        Err(Status::NotFound)
    } else {
        Ok(NewUser(users))
    }
    ```

现在，让我们再次尝试调用`user()`和`users()`端点。它应该像我们使用`HashMap`时一样工作。由于我们在连接池上写下`connect()`之后没有修改连接选项，SQL 输出被写入终端：

```rs
/* SQLx ping */; rows: 0, elapsed: 944.711µs
SELECT * FROM users …; rows: 1, elapsed: 11.187ms
SELECT
  *
FROM
  users
WHERE
  uuid = $1
```

这里有一些更多的输出：

```rs
/* SQLx ping */; rows: 0, elapsed: 524.114µs
SELECT * FROM users …; rows: 1, elapsed: 2.435ms
SELECT
  *
FROM
  users
WHERE
  name LIKE $1
  AND grade = $2
```

在这本书中，我们不会使用 ORM；相反，我们将只使用`sqlx`，因为这对于本书的范围已经足够了。如果你想在应用程序中使用 ORM，你可以使用来自[`github.com/NyxCode/ormx`](https://github.com/NyxCode/ormx)或[`www.sea-ql.org/SeaORM/`](https://www.sea-ql.org/SeaORM/)的 ORM 和查询构建器。

现在我们已经了解了`State`以及如何使用`State`来使用数据库，现在是时候学习 Rocket 的另一个中间件功能，即附加整流罩。

# 附加 Rocket 整流罩

在现实生活中，火箭整流罩是一种用于保护火箭有效载荷的头部锥体。在 Rocket 框架中，整流罩不是用来保护有效载荷的，而是用来钩入请求生命周期的任何部分并重写有效载荷。整流罩在其他 Web 框架中类似于中间件，但有一些不同之处。

其他框架的中间件可能能够注入任何任意数据。在 Rocket 中，整流罩可以用来修改请求，但不能用来添加不属于请求的信息。例如，我们可以使用整流罩在请求或响应中添加新的 HTTP 头。

一些 Web 框架可能能够终止并直接响应传入的请求，但在 Rocket 中，整流罩不能直接停止传入的请求；请求必须通过路由处理函数，然后路由可以创建适当的响应。

我们可以通过为类型实现`rocket::fairing::Fairing`来创建一个整流罩。让我们首先看看特质的签名：

```rs
 #[crate::async_trait]
```

```rs
pub trait Fairing: Send + Sync + Any + 'static {
```

```rs
    fn info(&self) -> Info;
```

```rs
    async fn on_ignite(&self, rocket: Rocket<Build>) ->
```

```rs
    Result { Ok(rocket) }
```

```rs
    async fn on_liftoff(&self, _rocket: &Rocket<Orbit>) { }
```

```rs
    async fn on_request(&self, _req: &mut Request<'_>, 
```

```rs
    _data: &mut Data<'_>) {}
```

```rs
    async fn on_response<'r>(&self, _req: &'r Request<'_>, 
```

```rs
    _res: &mut Response<'r>) {}
```

```rs
}
```

有一些类型我们不太熟悉，例如`Build`和`Orbit`。这些类型与 Rocket 中的阶段相关。

## Rocket 阶段

我们想要讨论的类型是`Build`和`Orbit`，它们的完整模块路径是`rocket::Orbit`和`rocket::Build`。这些类型是什么？Rocket 实例的签名是`Rocket<P: Phase>`，这意味着任何实现了`rocket::Phase`的`P`类型。

`Phase`是一个`pub trait SomeTrait: private::Sealed {}`。

`Phase` 特征被密封，因为 Rocket 作者只打算在 Rocket 应用程序中有三个阶段：`rocket::Build`、`rocket::Ignite` 和 `rocket::Orbit`。

我们通过 `rocket::build()` 初始化一个 Rocket 实例，它使用 `Config::figment()` 默认配置，或者使用 `rocket::custom<T: Provider>(provider: T)`，它使用自定义配置提供者。在这个阶段，我们还可以使用 `configure<T: Provider>(self, provider: T)` 将生成的实例与自定义配置链式连接。然后我们可以使用 `mount()` 添加路由，使用 `register()` 注册捕获器，使用 `manage()` 管理状态，并使用 `attach()` 附加防热罩。

之后，我们可以通过 `ignite()` 方法将 Rocket 阶段更改为 `Ignite`。在这个阶段，我们有一个具有最终配置的 Rocket 实例。然后我们可以通过 `launch()` 方法将 Rocket 发送到 `Orbit` 阶段，或者返回 `Rocket<Build>` 并使用 `#[launch]` 属性。我们还可以跳过 `Ignite` 阶段，并在 `build()` 之后直接使用 `launch()`。

让我们回顾一下到目前为止我们创建的代码：

```rs
#[launch]
```

```rs
async fn rocket() -> Rocket<Build> {
```

```rs
    let our_rocket = rocket::build();
```

```rs
    …
```

```rs
    our_rocket
```

```rs
        .manage(visitor_counter)
```

```rs
        .manage(pool)
```

```rs
        .mount("/", routes![user, users, favicon])
```

```rs
        .register("/", catchers![not_found, forbidden])
```

```rs
}
```

这个函数生成 `Rocket<Build>`，而 `#[launch]` 属性生成使用 `launch()` 的代码。

这个小节的结论是，Rocket 阶段从 `Build` 到 `Ignite` 到 `Launch`。这些阶段与防热罩有何关系？让我们在下一个小节中讨论这个问题。

## 防热罩回调

实现防热罩的任何类型都必须实现一个强制函数，`info()`，它返回 `rocket::fairing::Info`。`Info` 结构体被定义为如下：

```rs
pub struct Info {
```

```rs
    pub name: &'static str,
```

```rs
    pub kind: Kind,
```

```rs
}
```

`rocket::fairing::Kind` 被定义为只是一个空的 struct，`pub struct Kind(_);`，但 `Kind` 包含了 `Kind::Ignite`、`Kind::Liftoff`、`Kind::Request`、`Kind::Response` 和 `Kind::Singleton`。

关联常量是什么？在 Rust 中，我们可以声明**关联项**，这些是在特征中声明的或在实现中定义的项。例如，我们有这样一段代码：

```rs
struct Something {
```

```rs
    item: u8
```

```rs
}
```

```rs
impl Something {
```

```rs
    fn new() -> Something {
```

```rs
        Something {
```

```rs
            item: 8,
```

```rs
        }
```

```rs
    }
```

```rs
}
```

我们可以使用 `Something::new()` `self` 作为第一个参数。我们已经实现了一个关联方法几次。

我们也可以定义一个**关联类型**如下：

```rs
trait SuperTrait {
```

```rs
    type Super;
```

```rs
}
```

```rs
struct Something;
```

```rs
struct Some;
```

```rs
impl SuperTrait for Something {
```

```rs
    type Super = Some;
```

```rs
}
```

最后，我们可以有一个 `rocket::fairing::Kind`：

```rs
pub struct Kind(usize);
```

```rs
impl Kind {
```

```rs
    pub const Ignite: Kind = Kind(1 << 0);
```

```rs
    pub const Liftoff: Kind = Kind(1 << 1);
```

```rs
    pub const Request: Kind = Kind(1 << 2);
```

```rs
    pub const Response: Kind = Kind(1 << 3);
```

```rs
    pub const Singleton: Kind = Kind(1 << 4);
```

```rs
    ...
```

```rs
}
```

让我们回到 `Info`。我们可以创建一个 `Info` 实例如下：

```rs
Info {
```

```rs
    name: "Request Response Tracker",
```

```rs
    kind: Kind::Request | Kind::Response,
```

```rs
}
```

我们说的是 `kind` 的值是 `kind` 关联常量之间 `OR` 位运算的结果。`Kind::Request` 是 `1<<2`，这意味着二进制中的 `100` 或十进制中的 `4`。`Kind::Response` 是 `1<<3`，这意味着二进制中的 `1000` 或十进制中的 `8`。`0100 | 1000` 的结果是二进制中的 `1100` 或十进制中的 `12`。有了这些知识，我们可以将 `Info` 实例的 `kind` 值从 `00000` 设置到 `11111`。

使用位运算设置配置是一个非常常见的将多个值打包到一个变量中的设计模式。一些其他语言甚至将这个设计模式变成自己的类型，并称之为**bitset**。

在实现`Fairing`特质的类型中，必须实现的方法是`info()`，它返回`Info`实例。我们还需要根据我们定义的`kind`实例实现`on_ignite()`、`on_liftoff()`、`on_request()`和`on_response()`。在我们的情况下，这意味着我们必须实现`on_request()`和`on_response()`。

Rocket 在不同的场合执行我们的公平器方法。如果我们有`on_ignite()`，它将在发射前执行。这种类型的公平器是特殊的，因为`on_ignite()`返回`Result`，如果返回的变体是`Err`，则可以中止发射。

对于`on_liftoff()`，此方法将在发射后执行，这意味着当 Rocket 处于`Orbit`阶段时。

如果我们有`on_request()`，它将在 Rocket 获取请求但请求尚未路由之前执行。此方法将可以访问`Request`和`Data`，这意味着我们可以修改这两个项目。

当路由处理程序创建响应但尚未将其发送到 HTTP 客户端时，将执行`on_response()`。此回调可以访问`Request`和`Response`实例。

`Kind::Singleton`是特殊的。我们可以创建同一类型的多个公平器实例并将它们附加到 Rocket 上。但是，我们可能只想允许添加一个`Fairing`实现类型的实例。我们可以使用`Kind::Singleton`，这将确保只有最后附加的此类型的实例将被添加。

现在我们对 Rocket 阶段和`Fairing`回调有了更多的了解，让我们在下一小节中实现`Fairing`特质。

## 实现并附加公平器

目前，我们的 Rocket 应用程序管理`VisitorCounter`，但我们没有将`State<VisitorCounter>`添加到`favicon()`函数中。我们可能还想添加新的路由处理函数，但将`State<VisitorCounter>`作为每个路由处理函数的参数参数是很繁琐的。

我们可以将`VisitorCounter`从受管理状态改为公平器。同时，让我们假设我们的应用程序中还有另一个要求。我们希望为请求和响应添加自定义头，用于内部日志记录。我们可以通过添加另一个公平器来更改传入的请求和响应来实现它。

首先，让我们稍微组织一下我们的模块使用。我们需要添加与公平器相关的模块，`rocket::http::Header`、`rocket::Build`和`rocket::Orbit`，这样我们就可以为我们的`VisitorCounter`公平器和另一个修改请求和响应的公平器使用它们：

```rs
use rocket::fairing::{self, Fairing, Info, Kind};
```

```rs
use rocket::fs::{relative, NamedFile};
```

```rs
use rocket::http::{ContentType, Header, Status};
```

```rs
use rocket::request::{FromParam, Request};
```

```rs
use rocket::response::{self, Responder, Response};
```

```rs
use rocket::{Build, Data, Orbit, Rocket, State};
```

为`VisitorCounter`添加`Fairing`特质实现。由于此特质是一个`async`特质，我们需要用`#[rocket::async_trait]`装饰`impl`：

```rs
#[rocket::async_trait]
```

```rs
impl Fairing for VisitorCounter {
```

```rs
    fn info(&self) -> Info {
```

```rs
        Info {
```

```rs
            name: "Visitor Counter",
```

```rs
            kind: Kind::Ignite | Kind::Liftoff | Kind::
```

```rs
            Request,
```

```rs
        }
```

```rs
    }
```

```rs
}
```

我们添加了强制性的 `info()` 方法，它返回 `Info` 实例。在 `Info` 实例内部，我们实际上只需要 `Kind::Request`，因为我们只需要为每个传入请求增加访问者计数器。但这次，我们还添加了 `Kind::Ignite` 和 `Kind::Liftoff`，因为我们想看到回调何时执行。

然后，我们可以在 `impl Fairing` 块内添加这些回调：

```rs
async fn on_ignite(&self, rocket: Rocket<Build>) -> fairing::Result {
```

```rs
    println!("Setting up visitor counter");
```

```rs
    Ok(rocket)
```

```rs
}
```

```rs
async fn on_liftoff(&self, _: &Rocket<Orbit>) {
```

```rs
    println!("Finish setting up visitor counter");
```

```rs
}
```

```rs
async fn on_request(&self, _: &mut Request<'_>, _: &mut Data<'_>) {
```

```rs
    self.increment_counter();
```

```rs
}
```

`on_ignite()` 方法的返回类型是什么？`rocket::fairing::Result` 被定义为 `Result<T = Rocket<Build>, E = Rocket<Build>> = Result<T, E>`。此方法用于控制程序是否继续。例如，我们可以检查与第三方服务器的连接以确保其就绪。如果第三方服务器准备好接受连接，我们可以返回 `Ok(rocket)`。但是，如果第三方服务器不可用，我们可以返回 `Err(rocket)` 以停止 Rocket 的启动。注意，`on_liftoff()`、`on_request()` 和 `on_response()` 没有返回类型，因为 `Fairing` 被设计为仅在构建 Rocket 时失败。

对于 `on_liftoff()`，我们只想将一些信息打印到应用程序输出中。对于 `on_request()`，我们承担这个舱盖的真实目的：为每个请求增加计数器。

在实现 `Fairing` 特性之后，我们可以从 `user()` 和 `users()` 函数参数中移除 `counter: &State<VisitorCounter>`。我们还需要从这些函数的主体中移除 `counter.increment_counter();`。

在我们修改了 `user()` 和 `users()` 函数之后，我们可以将舱盖附加到 Rocket 应用程序上。在 Rocket 初始化代码中将 `manage(visitor_counter)` 改为 `attach(visitor_counter)`。

是时候看看舱盖的实际效果了！首先，看看初始化序列。你可以看到 `on_ignite()` 在开始时执行，而 `on_liftoff()` 在一切准备就绪后执行：

```rs
> cargo run
...
Setting up visitor counter
 Configured for debug.
...
 Fairings:
   >> Visitor Counter (ignite, liftoff, request)
...
Finish setting up visitor counter
 Rocket has launched from http://127.0.0.1:8000
```

之后，再次尝试调用我们的路由处理函数，以查看计数器再次增加：

```rs
> curl http://127.0.0.1:8000/user/3e3dd4ae-3c37-40c6-aa64-7061f284ce28
```

此外，在 Rocket 输出中，我们可以看到当我们将其用作状态时，它会增加：

```rs
The number of visitor is: 2
```

现在，让我们实现第二个用例，将跟踪 ID 注入到我们的请求和响应中。

首先，修改 `Cargo.toml` 以确保 `uuid` 包可以生成一个随机的 UUID：

```rs
uuid = {version = "0.8.2", features = ["v4"]}
```

之后，在 `src/main.rs` 中，我们可以定义我们想要注入的头部名称以及作为舱盖工作的类型：

```rs
const X_TRACE_ID: &str = "X-TRACE-ID";
```

```rs
struct XTraceId {}
```

之后，我们可以为 `XtraceId` 实现一个 `Fairing` 特性。这次，我们希望有 `on_request()` 和 `on_response()` 回调：

```rs
#[rocket::async_trait]
```

```rs
impl Fairing for XTraceId {
```

```rs
    fn info(&self) -> Info {
```

```rs
        Info {
```

```rs
            name: "X-TRACE-ID Injector",
```

```rs
            kind: Kind::Request | Kind::Response,
```

```rs
        }
```

```rs
    }
```

```rs
}
```

现在，在 `impl Fairing` 块内编写 `on_request()` 和 `on_response()` 的实现：

```rs
async fn on_request(&self, req: &mut Request<'_>, _: &mut Data<'_>) {
```

```rs
    let header = Header::new(X_TRACE_ID, 
```

```rs
    Uuid::new_v4().to_hyphenated().to_string());
```

```rs
    req.add_header(header);
```

```rs
}
```

```rs
async fn on_response<'r>(&self, req: &'r Request<'_>, res: &mut Response<'r>) {
```

```rs
    let header = req.headers().get_one(X_TRACE_ID).
```

```rs
    unwrap();
```

```rs
    res.set_header(Header::new(X_TRACE_ID, header));
```

```rs
}
```

在 `on_request()` 中，我们生成一个随机 UUID，并将生成的字符串作为请求头部之一注入。在 `on_response()` 中，我们将具有相同头部的响应注入到请求中。

不要忘记初始化并将这个新的舱盖附加到 Rocket 构建和启动过程中：

```rs
let x_trace_id = XTraceId {};
```

```rs
our_rocket
```

```rs
    ...
```

```rs
    .attach(visitor_counter)
```

```rs
    .attach(x_trace_id)
```

```rs
    ...
```

重新运行应用程序。我们应该在应用程序输出中看到一个新舱盖，并在 HTTP 响应中包含 `"x-trace-id"`：

```rs
...
 Fairings:
   >> X-TRACE-ID Injector (request, response)
   >> Visitor Counter (ignite, liftoff, request)
...
```

这里是另一个例子：

```rs
curl -v http://127.0.0.1:8000/user/3e3dd4ae-3c37-40c6-aa64-7061f284ce28
...
< x-trace-id: 28c0d523-13cc-4132-ab0a-3bb9ae6153a9
...
```

请注意，我们可以在应用程序中使用 `State` 和 `Fairing`。只有在我们需要为每个请求调用它时才使用 `Fairing`。

之前，我们创建了一个连接池，并告诉 Rocket 使用管理状态来管理它，但 Rocket 已经有通过其内置数据库连接 `rocket_db_pools` 连接到数据库的方法，`rocket_db_pools` 是一种公平的连接方式。让我们看看在下一部分如何实现它。

## 使用 rocket_db_pools 连接到数据库

Rocket 通过使用 `rocket_db_pools` 提供了一种连接到某些 RDBMS 的官方方法。该 crate 为 Rocket 提供了数据库驱动程序的集成。我们将学习如何使用这个 crate 来连接到数据库。让我们将之前创建的连接池从使用状态改为使用公平连接：

1.  我们不需要 `serde`，因为 `rocket_db_pools` 已经有自己的配置。从 `Cargo.toml` 中删除 `serde` 并添加 `rocket_db_pools` 作为依赖项：

    ```rs
    [dependencies]
    rocket = "0.5.0-rc.1"
    rocket_db_pools = {version = "0.5.0-rc.1", features = ["sqlx_postgres"]}
    ...
    ```

你也可以使用不同的功能，如 `sqlx_mysql`、`sqlx_sqlite`、`sqlx_mssql`、`deadpool_postgres`、`deadpool_redis` 或 `mongodb`。

1.  在 `Rocket.toml` 中，删除包含 `database_url` 配置的行，并替换为以下行：

    ```rs
    [debug.databases.main_connection]
    url = "postgres://username:password@localhost/rocket"
    ```

如果你喜欢，可以使用 `default.databases.main_connection`，你也可以将 `main_connection` 改成你喜欢的任何名称。

1.  在 Cargo 库项目中，我们可以使用 `pub use something;` 语法在 `our_library` 中重新导出某些内容，然后另一个库可以通过 `our_library::something` 使用它。删除这些 `use sqlx...` 和 `use serde...` 行，因为 `rocket_db_pools` 已经重新导出了 `sqlx`，我们也不再需要 `serde`：

    ```rs
    use serde::Deserialize;
    ...
    use sqlx::postgres::{PgPool, PgPoolOptions};
    use sqlx::FromRow;
    ```

1.  添加以下行来使用 `rocket_db_pools`。注意，我们可以在代码中多行声明 `use`：

    ```rs
    use rocket_db_pools::{
        sqlx,
        sqlx::{FromRow, PgPool},
        Connection, Database,
    };
    ```

1.  删除 `Config` 结构体的声明，并添加以下行来声明数据库连接类型：

    ```rs
    #[derive(Database)]
    #[database("main_connection")]
    struct DBConnection(PgPool);
    ```

数据库为 `DBConnection` 类型自动生成 `rocket_db_pools::Database` 实现。注意，我们写的是连接名称 `"main_connection"`，就像我们在 `Rocket.toml` 中设置的那样。

1.  在 `rocket()` 函数中移除配置和连接池初始化：

    ```rs
    let config: Config = our_rocket
        .figment()
        .extract()
        .expect("Incorrect Rocket.toml configuration");
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&config.database_url)
        .await
        .expect("Failed to connect to database");
    ```

1.  在 `rocket()` 函数中添加 `DBConnection::init()` 并将其附加到 Rocket：

    ```rs
    async fn rocket() -> Rocket<Build> {
        ...
        rocket::build()
            .attach(DBConnection::init())
            .attach(visitor_counter)
            ...
    }
    ```

1.  将 `user()` 和 `users()` 函数修改为使用 `rocket_db_pools::Connection` 请求保护：

    ```rs
    async fn user(mut db: Connection<DBConnection>, uuid: &str) -> Result<User, Status> {
        ...
        let user = sqlx::query_as!(User, "SELECT * FROM 
        users WHERE uuid = $1", parsed_uuid)
            .fetch_one(&mut *db)
            .await;
        ...
    }
    ...
    #[get("/users/<name_grade>?<filters..>")]
    async fn users(
        mut db: Connection<DBConnection>,
        ...
    ) -> Result<NewUser, Status> {
        ...
        let unwrapped_users = query.fetch_all(&mut 
        *db).await;
        ...
    }
    ```

应用程序应该像我们使用状态管理连接池时一样工作，但有一些细微的差别。以下是我们的输出：

```rs
 Fairings:
   ...
   >> 'main_connection' Database Pool (ignite)
```

我们可以看到应用程序输出中有一个新的公平连接，但应用程序输出中没有准备好的 SQL 语句。

# 摘要

在这一章中，我们学习了两个 Rocket 组件，`State` 和 `Fairing`。我们可以在构建火箭时管理状态对象和附加公平连接，在路由处理函数中使用 `state` 对象，并使用 `fairing` 函数在构建后、启动后、请求时和响应时执行回调。

我们还创建了计数器状态并在路由处理函数中使用它们。我们还学习了如何使用 `sqlx`，进行了数据库迁移，创建了数据库连接池状态，并使用 `state` 查询数据库。

此后，我们更多地了解了 Rocket 初始化过程以及构建、点燃和发射阶段。

最后，我们将计数器状态改为公平器，并创建了一个新的公平器来向传入的请求和传出的响应中注入自定义的 HTTP 头部。

拥有了这些知识，你可以在路由处理函数之间创建可重用的对象，并创建一个可以在请求和响应之间全局执行的方法。

我们的 `src/main.rs` 文件正在变得越来越大和复杂；我们将在下一章学习如何以模块的方式管理我们的 Rust 代码，并规划一个更复杂的应用程序。
