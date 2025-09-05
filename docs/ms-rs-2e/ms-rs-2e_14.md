# 使用 Rust 与数据库交互

在本章中，我们将讨论为什么数据库对现代应用程序至关重要。我们将介绍 Rust 生态系统中的几个 crate，这些 crate 允许与数据库交互。然后，我们将继续讨论我们在上一章中开发的 linksnap API 服务器，并通过一个方便的库将其数据库支持集成进去。这将使我们能够持久化发送到我们 API 的新书签。

在本章中，我们将涵盖以下主题：

+   使用 `rusqlite` 进行 SQLite 集成

+   使用 Rust 与 PostgreSQL 交互

+   数据库连接池

+   使用 `diesel` crate 进行对象关系映射

+   将 diesel 与 linksnap API 服务器集成

# 我们为什么需要数据持久化？

“有时，优雅的实现仅仅是一个函数。不是一个方法。不是一个类。不是一个框架。仅仅是一个函数。” —— 约翰·卡马克

现代应用程序通常数据密集。正如许多人所说，数据是新石油。数据库支持的服务无处不在，从社交游戏到云存储，再到电子商务，以及医疗保健等等。所有这些服务都需要正确地存储和检索数据。它们存储的数据必须易于检索，并且必须保证一致性和持久性。

数据库是构建基于数据的应用程序并提供用户期望的强大基础的解决方案。在我们用 Rust 构建涉及数据库的任何东西之前，我们需要了解一些基础知识。一般来说，数据库是一组表。表是数据组织的单元。

组织到表中的数据仅适用于关系数据库。其他数据库，如 NoSQL 和基于图的数据库，使用更灵活的文档模型来存储和组织数据。

数据被组织成逻辑实体，称为表。表通常代表现实世界中的实体。这些实体可以具有各种属性，这些属性在表中取代了列的位置。这些实体还可以与其他实体有联系。一个表中的列可以引用另一个表中的列。对任何数据库的更改都通过一个特定的 DSL（领域特定语言）执行，称为**结构化查询语言（SQL）**。SQL 查询还允许您使用查询子句（如 JOIN）将查询扩展到多个表。这些都是基础知识。用户与数据库支持的应用程序交互的常见模式是**CRUD**模式，即**创建、读取、更新和删除**。这些是用户在应用程序中对数据库执行的最常见操作。

SQL 是一种声明式的方法，用于在数据库上执行事务。事务是一组对数据库的修改，这些修改必须原子性地发生，或者在中间出现任何故障时根本不发生。在任何应用程序中编写数据库事务的直观方法是使用原始 SQL 查询。然而，有一种更好的方法来做这件事，它被称为**对象关系映射（ORM）**。这是一种使用本地语言抽象和类型访问数据库的技术，这些类型几乎与 SQL 语法和语义一一对应。语言提供了用于与 SQL 对话的高级库，允许你用它们的本地语言编写查询，然后这些查询被翻译成原始 SQL 查询。在传统的面向对象语言中，你的对象变成了会说话的 SQL 对象。这些库被称为对象关系映射器。许多这些库存在于主流语言中，如 Java 的 Hibernate、Ruby 的 Active Record、Python 的 SQLAlchemy 等。使用 ORM 可以减少在使用原始 SQL 查询时出现任何错误的可能性。然而，ORM 也受到无法将自身完全映射到语言的对象模型和数据库模型的限制。因此，ORM 库应尽量减少它们在交互数据库时提供的抽象量，并将一些部分留给原始 SQL 查询。

Rust 生态系统提供了许多高质量解决方案来管理和构建持久化应用程序。我们将在下一节中探讨其中的一些。

# SQLite

SQLite 是一个非常轻量级的嵌入式数据库。它不需要特殊的数据库管理系统，你就可以使用它。SQLite 创建的数据库可以表示为文件或内存中的数据库，你不需要连接到外部远程端点或本地套接字连接来使用数据库。它服务于与传统客户端-服务器数据库引擎（如 MySQL 或 PostgreSQL）不同的目标受众，并且是应用需要本地存储数据但又要安全且高效检索的用例的首选解决方案。Android 平台是 SQLite 的重度使用者，允许移动应用程序在应用程序内存储用户的偏好或配置。它也被许多需要存储任何类型状态并保证持久性的桌面应用程序所使用。

Rust 社区为我们提供了连接和与 SQLite 数据库交互的几种选择。我们将选择 `rusqlite` crate，该 crate 可在 `crates.io` 上找到，网址为 [`crates.io/crates/rusqlite`](https://crates.io/crates/rusqlite)。此 crate 支持 SQLite 版本 3.6.8 及以上。其 API 不能被视为 ORM，但可以被视为 ORM 提供的中间层抽象，因为它有助于隐藏实际 SQLite API 的许多细节。与许多其他关系型数据库系统相比，SQLite 的类型系统是动态的。这意味着列没有类型，但每个单独的值都有类型。技术上，SQLite 将存储类和数据类型分开，但这主要是实现细节，我们可以简单地从类型的角度思考，而不会离真相太远。

`rusqlite` crate 提供了 `FromSql` 和 `ToSql` 特性，用于在 SQLite 和 Rust 类型之间转换对象。它还提供了以下开箱即用的实现，用于大多数标准库类型和原语：

| **描述** | **SQLite** | **Rust** |
| --- | --- | --- |
| 空值 | `NULL` | `rusqlite::types::Null` |
| 1、2、3、4、6 或 8 字节有符号整数 | `INTEGER` | `i32`（可能截断）和 `i64` |
| 8 字节 IEEE 浮点数 | `REAL` | `f64` |
| UTF-8、UTF-16BE 或 UTF-16LE 字符串 | `TEXT` | `String` 和 `&str` |
| 字节字符串 | `BLOB` | `Vec<u8>` 和 `&[u8]` |

在掌握了 `rusqlite` crate 的基础知识之后，让我们看看它是如何应用的。

我们将通过运行 `cargo new rusqlite_demo` 来创建一个新的项目。我们的程序从标准输入接收一个格式正确的逗号分隔值（**CSV**）格式的书籍列表，将其存储在 SQLite 中，然后使用 SQL 查询的过滤器检索数据子集。首先，让我们创建我们的表创建和删除查询以及我们的 `Book` 结构体，它将存储从查询中检索到的数据：

```rs
// rusqlite_demo/src/main.rs

use std::io;
use std::io::BufRead;

use rusqlite::Error;
use rusqlite::{Connection, NO_PARAMS};

const CREATE_TABLE: &str = "CREATE TABLE books 
                            (id INTEGER PRIMARY KEY,
                            title TEXT NOT NULL,
                            author TEXT NOT NULL,
                            year INTEGER NOT NULL)";

const DROP_TABLE: &str = "DROP TABLE IF EXISTS books";

#[derive(Debug)]
struct Book {
    id: u32,
    title: String,
    author: String,
    year: u16
}
```

我们定义了两个常量，`CREATE_TABLE` 和 `DROP_TABLE`，分别包含创建 `books` 表和删除它的原始 SQL 查询。然后我们有书籍结构体，它包含以下字段：

+   `id`: 它作为主键使用，可以在将书籍插入我们的书籍表中时区分不同的书籍

+   `title`: 书籍的标题

+   `author`: 该书的作者

+   `year`: 出版年份

接下来，让我们看看我们的 `main` 函数：

```rs
// rusqlite_demo/src/main.rs

fn main() {
    let conn = Connection::open("./books").unwrap();
    init_database(&conn);
    insert(&conn);
    query(&conn);
}
```

首先，我们通过调用 `Connection::open` 并提供一个路径，`"./books"`，来创建当前目录中的数据库，从而打开我们的 SQLite 数据库连接。接下来，我们调用 `init_database()`，传递对 `conn` 的引用，它如下定义：

```rs
fn init_database(conn: &Connection) {
    conn.execute(CREATE_TABLE, NO_PARAMS).unwrap();
}
```

然后，我们调用 `insert` 方法，传递我们的 `conn`。最后，我们调用 `query` 方法，查询我们的 `books` 数据库。

这是我们的 `insert` 函数方法：

```rs
fn insert(conn: &Connection) {
    let stdin = io::stdin();
    let lines = stdin.lock().lines();
    for line in lines {
        let elems = line.unwrap();
        let elems: Vec<&str> = elems.split(",").collect();
        if elems.len() == 4 {
            let _ = conn.execute(
                "INSERT INTO books (id, title, author, year) VALUES (?1, ?2, ?3, ?4)",
                &[&elems[0], &elems[1], &elems[2], &elems[3]],
            );
        }
    }
}
```

在`insert`中，我们首先锁定`stdout`，然后遍历行。每行用逗号分隔。随后，我们在`conn`上调用`execute`方法，传入一个插入查询字符串。在查询字符串中，我们使用模板变量`?1`、`?2`等，其对应的值是从`elems`向量中取出的。如果收集到的元素数量达到`4`，我们使用原始 SQL 查询插入书籍，并为模板变量提供`elems` `Vec`中的相应值。

接下来，我们的`query`函数定义如下：

```rs
fn query(conn: &Connection) {
    let mut stmt = conn
        .prepare("SELECT id, title, author, year FROM books WHERE year >= ?1")
        .unwrap();
    let movie_iter = stmt
        .query_map(&[&2013], |row| Book {
            id: row.get(0),
            title: row.get(1),
            author: row.get(2),
            year: row.get(3),
        })
        .unwrap();

    for movie in movie_iter.filter_map(extract_ok) {
        println!("Found book {:?}", movie);
    }
}
```

查询函数接收`conn`参数，我们在其上调用`prepare`方法，传入原始的 SQL 查询字符串。在这里，我们正在过滤那些出版年份大于给定年份的书籍。我们将这个查询存储在`stmt`中。接下来，我们在这个类型上调用`query_map`，传入一个只包含数字`2013`的数组引用，这个数字代表我们要过滤书籍的年份。正如你所见，这里的 API 有点不自然。`query_map`的第二个参数是一个闭包，它是一个`Row`类型。在闭包内部，我们从`row`实例中提取相应的字段，并从中构建一个`Book`实例。这返回一个迭代器，我们将其存储在`movie_iter`中。最后，我们遍历`movie_iter`，使用`extract_ok`辅助方法过滤任何失败的值。这个方法定义如下：

```rs
fn extract_ok(p: Result<Book, Error>) -> Option<Book> {
    if p.is_ok() {
        Some(p.unwrap())
    } else {
        None
    }
}
```

然后，我们打印书籍。完整的代码如下：

```rs
// rusqlite_demo/src/main.rs

use std::io;
use std::io::BufRead;

use rusqlite::Error;
use rusqlite::{Connection, NO_PARAMS};

const CREATE_TABLE: &str = "CREATE TABLE IF NOT EXISTS books 
                            (id INTEGER PRIMARY KEY,
                            title TEXT NOT NULL,
                            author TEXT NOT NULL,
                            year INTEGER NOT NULL)";

#[derive(Debug)]
struct Book {
    id: u32,
    title: String,
    author: String,
    year: u16,
}

fn extract_ok(p: Result<Book, Error>) -> Option<Book> {
    if p.is_ok() {
        Some(p.unwrap())
    } else {
        None
    }
}

fn insert(conn: &Connection) {
    let stdin = io::stdin();
    let lines = stdin.lock().lines();
    for line in lines {
        let elems = line.unwrap();
        let elems: Vec<&str> = elems.split(",").collect();
        if elems.len() > 2 {
            let _ = conn.execute(
                "INSERT INTO books (id, title, author, year) VALUES (?1, ?2, ?3, ?4)",
                &[&elems[0], &elems[1], &elems[2], &elems[3]],
            );
        }
    }
}

fn init_database(conn: &Connection) {
    conn.execute(CREATE_TABLE, NO_PARAMS).unwrap();
}

fn query(conn: &Connection) {
    let mut stmt = conn
        .prepare("SELECT id, title, author, year FROM books WHERE year >= ?1")
        .unwrap();
    let movie_iter = stmt
        .query_map(&[&2013], |row| Book {
            id: row.get(0),
            title: row.get(1),
            author: row.get(2),
            year: row.get(3),
        })
        .unwrap();

    for movie in movie_iter.filter_map(extract_ok) {
        println!("Found book {:?}", movie);
    }
}

fn main() {
    let conn = Connection::open("./books").unwrap();
    init_database(&conn);
    insert(&conn);
    query(&conn);
}
```

我们在同一目录下还有一个 books.csv 文件。我们可以通过以下命令运行它：

```rs
cargo run < books.csv
```

下面是程序运行后的输出：

![](img/9605d0c0-ab0d-4296-b8fe-93a031b7a0f0.png)

这远非一个代表性的真实世界数据库支持的应用程序，它只是为了演示库的使用。一个真实世界的应用程序不会从标准输入读取，查询例程会有更好的错误处理。

这只是一个简要的演示，说明如何使用`rusqlite`crate 通过 Rust 使用 SQLite 数据库。API 不是非常强类型，但这是我们目前唯一的选择。接下来，我们将看看 SQLite 的“大哥”——PostgreSQL 数据库管理系统。

# PostgreSQL

虽然 SQLite 适合原型设计和简单的用例，但一个真正的关系型数据库管理系统可以使开发者的生活更加轻松。这样一个复杂的数据库系统是 PostgreSQL。要在 Rust 中集成 PostgreSQL，我们可以在*crates.io*上找到`postgres`crate。它是一个本地的 Rust 客户端，这意味着它不依赖于 C 库，而是在 Rust 中实现了整个协议。如果 API 看起来与`rusqlite`crate 相似，这是故意的；SQLite 客户端的 API 实际上是基于 PostgreSQL 客户端的。`postgres`crate 支持一些 PostgreSQL 的独特功能，如位向量、时间字段、JSON 支持和 UUID。

在本节中，我们将通过创建一个初始化 PostgreSQL 数据库并在数据库上进行一些插入和查询的示例程序来探索与 PostgreSQL 的交互。我们假设您已经在系统上设置了数据库。本例中使用的 PostgreSQL 版本是 9.5。

要安装 PostgreSQL 数据库系统，以下 DigitalOcean 文章推荐：[`www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-16-04`](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-16-04)。

Postgres 自带一个名为`psql`的命令行工具，可以用来运行查询、检查表、管理角色、查看系统信息等等。您可以通过在 psql 提示符内运行以下命令来查看您系统上运行的 PostgreSQL 版本。首先，我们将通过运行以下命令来启动`psql`：

```rs
$ sudo -u postgres psql
```

一旦我们进入 psql，我们就在提示符下运行以下命令：

```rs
postgres=# SELECT version();
```

运行上述命令，会得到以下输出：

```rs
PostgreSQL 9.5.14 on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 5.4.0-6ubuntu1~16.04.10) 5.4.0 20160609, 64-bit
```

为了使这个例子更简单，我们将重用我们在`rusqlite`演示中使用的相同的书籍数据。在以下示例中，我们将使用默认的`"postgres"`用户，密码为`"postgres"`。您需要将以下示例修改以匹配您的新用户。让我们通过运行`cargo new postgres_demo`来启动一个新的项目。以下是我们的`Cargo.toml`中的依赖项：

```rs
# postgres_demo/Cargo.toml

[dependencies]
postgres = "0.15.2"
serde = { version = "1.0.82"}
serde_derive = "1.0.82"
serde_json = "1.0.33"
```

让我们逐行查看`main.rs`中的代码：

```rs
// postgres_demo/src/main.rs

use postgres::{Connection, TlsMode};

const DROP_TABLE: &str = "DROP TABLE IF EXISTS books";
const CONNECTION: &str = "postgres://postgres:postgres@localhost:5432";
const CREATE_TABLE: &str = "CREATE TABLE IF NOT EXISTS books 
                            (id SERIAL PRIMARY KEY,
                            title VARCHAR NOT NULL,
                            author VARCHAR NOT NULL,
                            year SERIAL)";

#[derive(Debug)]
struct Book {
    id: i32,
    title: String,
    author: String,
    year: i32
}

fn reset_db(conn: &Connection) {
    let _ = conn.execute(DROP_TABLE, &[]).unwrap();
    let _ = conn.execute(CREATE_TABLE, &[]).unwrap();
}
```

我们有一系列用于连接数据库以及创建和删除书籍表的字符串常量。接下来是我们的`main`函数：

```rs
// postgres_demo/src/main.rs

fn main() {
    let conn = Connection::connect(CONNECTION, TlsMode::None).unwrap();
    reset_db(&conn);

    let book = Book {
        id: 3,
        title: "A programmers introduction to mathematics".to_string(),
        author: "Dr. Jeremy Kun".to_string(),
        year: 2018
    };

    conn.execute("INSERT INTO books (id, title, author, year) VALUES ($1, $2, $3, $4)",
                 &[&book.id, &book.title, &book.author, &book.year]).unwrap();

    for row in &conn.query("SELECT id, title, author, year FROM books", &[]).unwrap() {
        let book = Book {
            id: row.get(0),
            title: row.get(1),
            author: row.get(2),
            year: row.get(3)
        };
        println!("{:?}", book);
    }
}
```

由于我们在这里没有使用 ORM，而是一个低级接口，我们需要手动将值解包到数据库查询中。让我们运行这个程序：

![](img/21ff948c-f012-4cce-83bb-1f2f8a075cd5.png)

这是程序的输出，以及随后在`psql`中对表的查询，以显示其内容：

![](img/bbef2889-a031-4572-a8e0-efeae1da3fd9.png)

首先，我们在 psql 提示符下使用`\dt`命令列出我们的数据库。之后，我们使用查询，即`"select * from books"`。

这就是使用 Rust 与 PostgreSQL 交互的基础。接下来，让我们探索如何通过使用连接池的概念来提高我们的数据库查询效率。

# 使用 r2d2 进行连接池

每次新事务发生时都打开和关闭数据库连接很快就会成为瓶颈。通常，打开数据库连接是一个昂贵的操作。这主要是因为需要在两端的套接字连接上创建所需的 TCP 握手。如果数据库托管在远程服务器上，这通常是情况，开销就更加昂贵。如果我们能够为发送到我们数据库的每个后续请求重用连接，我们可能会大大减少延迟。缓解这种开销的有效方法之一是采用数据库连接池。当进程需要新的连接时，它从连接池中获取现有的连接。当进程完成与数据库的必要操作后，这个连接句柄就会回到池中以便以后使用。

在 Rust 中，我们有`r2d2`crate，它利用特质提供了一种通用的方式来维护各种数据库的连接池。它提供了各种后端作为子 crate，并支持 PostgreSQL、Redis、MySQL、MongoDB、SQLite 以及一些其他已知的数据库系统。`r2d2`的架构由两部分组成：一个通用部分和一个特定后端部分。后端代码通过实现 r2d2 的`ManageConnection`特质，并为特定后端添加连接管理器来附加到通用部分。特质如下：

```rs
pub trait ManageConnection: Send + Sync + 'static {
    type Connection: Send + 'static;
    type Error: Error + 'static;
    fn connect(&self) -> Result<Self::Connection, Self::Error>;
    fn is_valid(&self, conn: &mut Self::Connection) -> Result<(), Self::Error>;
    fn has_broken(&self, conn: &mut Self::Connection) -> bool;
}
```

查看特质定义，我们需要指定一个`Connection`类型，它必须是`Send`和`'static`，以及一个`Error`类型。我们还有三个方法：`connect`、`is_valid`和`has_broken`。`connect`方法返回来自底层后端 crate 的`Connection`类型；例如，对于 Postgres 后端，它将是`postgres::Connection`类型。`Error`类型是一个枚举，它指定了在连接阶段或检查连接有效性期间可能发生的所有可能的`Error`情况。

为了演示目的，我们将查看如何使用`r2d2`crate，首先检查如何使用池连接到 PostgreSQL。我们将从上一节中的代码开始，并修改它以使用连接池，其中我们从`8`个线程中执行 SQL 查询。

下面是使用`r2d2-postgres`后端 crate 的池化和线程化实现的完整代码：

```rs
// r2d2_demo/src/main.rs

use std::thread;
use r2d2_postgres::{TlsMode, PostgresConnectionManager};
use std::time::Duration;

const DROP_TABLE: &str = "DROP TABLE IF EXISTS books";

const CREATE_TABLE: &str = "CREATE TABLE IF NOT EXISTS books 
                            (id SERIAL PRIMARY KEY,
                            title VARCHAR NOT NULL,
                            author VARCHAR NOT NULL,
                            year SERIAL)";

#[derive(Debug)]
struct Book {
    id: i32,
    title: String,
    author: String,
    year: i32
}

fn main() {
    let manager = PostgresConnectionManager::new("postgres://postgres:postgres@localhost:5432",
                                                 TlsMode::None).unwrap();
    let pool = r2d2::Pool::new(manager).unwrap();
    let conn = pool.get().unwrap();

    let _ = conn.execute(DROP_TABLE, &[]).unwrap();
    let _ = conn.execute(CREATE_TABLE, &[]).unwrap();

    thread::spawn(move || {
        let book = Book {
            id: 3,
            title: "A programmers introduction to mathematics".to_string(),
            author: "Dr. Jeremy Kun".to_string(),
            year: 2018
        };
        conn.execute("INSERT INTO books (id, title, author, year) VALUES ($1, $2, $3, $4)",
                    &[&book.id, &book.title, &book.author, &book.year]).unwrap();                                         
    });

    thread::sleep(Duration::from_millis(100));
    for _ in 0..8 {
        let conn = pool.get().unwrap();
        thread::spawn(move || {
            for row in &conn.query("SELECT id, title, author, year FROM books", &[]).unwrap() {
                let book = Book {
                    id: row.get(0),
                    title: row.get(1),
                    author: row.get(2),
                    year: row.get(3)
                };
                println!("{:?}", book);
            }
        });
    }
}

```

代码与上一个示例相比相当直接，除了我们现在会启动 8 个线程来对我们的数据库执行选择查询。池的大小被配置为`8`，这意味着`SELECT`查询线程可以通过重用连接来并发执行 8 个查询。

到目前为止，我们主要使用原始 SQL 查询从 Rust 与数据库交互。但是，有一个更方便的强类型方法，通过一个名为 diesel 的 ORM 库与数据库交互。让我们接下来探索一下。

# Postgres 和 diesel ORM

使用低级数据库库和原始 SQL 查询编写复杂的应用程序是犯许多错误的原因。Diesel 是一个 Rust 的 ORM（对象关系映射器）和查询构建器。它大量使用过程宏。它在编译时检测大多数数据库交互错误，并且在大多数情况下能够生成非常高效的代码，有时甚至能超越 C 语言的底层访问。这是因为它能够将通常在运行时进行的检查移动到编译时。在撰写本文时，diesel 支持开箱即用的 PostgreSQL、MySQL 和 SQLite。

我们将把数据库支持集成到我们在 第十三章，*使用 Rust 构建 Web 应用程序* 中开发的 linksnap 服务器中。我们将使用 diesel 以类型安全的方式与我们的 postgres 数据库通信。我们将从 第十三章，*使用 Rust 构建 Web 应用程序* 中复制 `linksnap` 项目，并将其重命名为 `linksnap_v2`。我们不会详细介绍完整的源代码，只会介绍影响数据库与 diesel 集成的部分。其余的代码库与上一章完全相同。

diesel 项目由许多组件组成。首先，我们有一个名为 `diesel-cli` 的命令行工具，它可以自动化创建数据库和执行任何必要的数据库迁移的过程。

现在，在我们开始实现与数据库通信的例程之前，我们需要安装 `diesel-cli` 工具，这将设置我们的数据库及其内部的表。我们可以通过运行以下命令来安装它：

```rs
cargo install diesel_cli --no-default-features --features postgres
```

我们只使用此 CLI 工具的 postgres 功能和 `--features` 标志。`Cargo` 将获取并构建 `diesel_cli` 及其依赖项，并将其安装到用户 Cargo 的默认二进制位置，通常是 `~/.cargo/bin/` 目录。

在我们的 `linksnap_v2` 目录中，我们将在目录根部的 `.env` 文件中添加数据库的连接 URL，其内容如下：

```rs
DATABASE_URL=postgres://postgres:postgres@localhost/linksnap
```

我们在 postgres 中的数据库名为 `linksnap`，用户名和密码都是 `postgres`。这绝对不是访问数据库的安全方式，建议您在生产环境中使用最佳安全实践来设置您的 postgres 数据库。

我们还需要在 `Cargo.toml` 文件中添加 `diesel` 作为依赖项，以及 `dotenv` 库。`dotenv` 包通过 *dotfiles* 处理本地配置。以下是我们的 `Cargo.toml` 文件：

```rs
# linksnap_v2/Cargo.toml

[dependencies]
actix = "0.7"
actix-web = "0.7"

futures = "0.1"
env_logger = "0.5"
bytes = "0.4"
serde = "1.0.80"
serde_json = "1.0.33"
serde_derive = "1.0.80"
url = "1.7.2"
lazy_static = "1.2.0"
log = "0.4.6"
chrono = { version="0.4" }
diesel = { version = "1.3.3", features = ["extras", "postgres", "r2d2"] }
dotenv = "0.13.0"
```

注意 diesel 包中我们使用的 `"postgres"` 和 `"r2d2"` 功能。接下来，我们将运行 `diesel setup`：

```rs
Creating migrations directory at: /home/creativcoder/book/Mastering-RUST-Second-Edition/Chapter14/linksnap_v2/migrations
```

这将在根目录下创建一个 diesel.toml 文件，其内容如下：

```rs
# linksnap_v2/diesel.toml

# For documentation on how to configure this file,
# see diesel.rs/guides/configuring-diesel-cli

[print_schema]
file = "src/schema.rs"
```

Diesel 依赖于宏来提供一些出色的功能，例如在编译时提供额外的安全和性能。为了能够做到这一点，它需要编译时对数据库的访问。这就是为什么 `.env` 文件很重要的原因。`diesel setup` 命令通过读取数据库并将其写入一个名为 `schema.rs` 的文件来自动生成模型类型。模型通常分为查询和插入结构体。两者都使用 derive 宏来为不同的用例生成模型代码。此外，基于 diesel 的应用程序需要代码来连接到数据库，以及一组数据库迁移来构建和维护数据库表。

现在，让我们添加一个迁移，通过运行以下代码来创建我们的表：

```rs
diesel migration generate linksnap
```

此命令生成一个新的迁移，包含两个空的 `up.sql` 和 `down.sql` 迁移文件：

```rs
Creating migrations/2019-01-30-045330_linksnap/up.sql
Creating migrations/2019-01-30-045330_linksnap/down.sql
```

迁移文件只是普通的 SQL，所以我们可以直接放入我们之前的 `CREATE TABLE` 命令：

```rs
-- linksnap_v2/migrations/2019-01-30-045330_linksnap_db/up.sql

CREATE TABLE linksnap (
  id SERIAL PRIMARY KEY,
  title VARCHAR NOT NULL,
  url TEXT NOT NULL,
  added TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
)
```

`down.sql` 文件应该包含相应的 `DROP TABLE`：

```rs
-- linksnap_v2/migrations/2019-01-30-045330_linksnap_db/down.sqlDROP TABLE linksnap
```

一旦我们这样做，我们就必须运行：

```rs
$ diesel migration run
```

这将通过从数据库中读取来创建一个 `schema.rs` 文件：

```rs
// linksnap_v2/src/schema.rs

table! {
    linksnap (id) {
        id -> Int4,
        title -> Varchar,
        url -> Text,
        added -> Timestamp,
    }
}
```

`table!` 宏为具有 `id` 作为主键的 `linksnap` 表生成代码。它还指定了列名，包括 `id`、`title`、`url` 和 `added`。

现在，我们可以为这个表编写一个模型。在 diesel 中，模型可以存在于任何可见的模块中，但我们将遵循将它们放在 `src/models.rs` 中的约定。以下是我们的 `Link` 模型的内容：

```rs
// linksnap_v2/src/models.rs

use chrono::prelude::*;
use diesel::prelude::Queryable;
use chrono::NaiveDateTime;
use diesel;
use diesel::pg::PgConnection;
use diesel::prelude::*;
use crate::state::AddLink;
use crate::schema::linksnap;
use crate::schema::linksnap::dsl::{linksnap as get_links};
use serde_derive::{Serialize, Deserialize};

pub type LinkId = i32;

#[derive(Queryable, Debug)]
pub struct Link {
    pub id: i32,
    pub title: String,
    pub url: String,
    pub added: NaiveDateTime,
}

impl Link {
    pub fn add_link(new_link: AddLink, conn: &PgConnection) -> QueryResult<usize> {
        diesel::insert_into(linksnap::table)
            .values(&new_link)
            .execute(conn)
    }

    pub fn get_links(conn: &PgConnection) -> QueryResult<Vec<Link>> {
        get_links.order(linksnap::id.desc()).load::<Link>(conn)
    }

    pub fn rm_link(id: LinkId, conn: &PgConnection) -> QueryResult<usize> {
        diesel::delete(get_links.find(id)).execute(conn)
    }
}
```

我们创建了一个 `Link` 结构体，它可以用来查询数据库。它还包含各种方法，当我们的服务器在相应的端点上收到请求时会被调用。

接下来，`state.rs` 文件包含 diesel 和 postgres 特定的代码：

```rs
// linksnap_v2/src/state.rs

use diesel::pg::PgConnection;
use actix::Addr;
use actix::SyncArbiter;
use std::env;
use diesel::r2d2::{ConnectionManager, Pool, PoolError, PooledConnection};
use actix::{Handler, Message};
use crate::models::Link;
use serde_derive::{Serialize, Deserialize};
use crate::schema::linksnap;
use std::ops::Deref;

const DB_THREADS: usize = 3;

use actix_web::{error, Error};
use actix::Actor;
use actix::SyncContext;

// Using this we create the connection pool.
pub type PgPool = Pool<ConnectionManager<PgConnection>>;
type PgPooledConnection = PooledConnection<ConnectionManager<PgConnection>>;
pub struct Db(pub PgPool);

// We define a get conn a convenient method to get
impl Db {
    pub fn get_conn(&self) -> Result<PgPooledConnection, Error> {
        self.0.get().map_err(|e| error::ErrorInternalServerError(e))
    }
}
```

首先，我们为我们的 PostgreSQL 连接池创建了一组方便的别名。我们有一个相同的 `Db` 结构体，它封装了 PgPool 类型。PgPool 类型是 diesel 中 `r2d2` 模块的一个 `ConnectionManager`。在 `Db` 结构体上，我们还定义了 `get_conn` 方法，它返回一个指向池连接的引用。

继续阅读同一文件：

```rs

// We then implement the Actor trait on the actor.
impl Actor for Db {
    type Context = SyncContext<Self>;
}

pub fn init_pool(database_url: &str) -> Result<PgPool, PoolError> {
    let manager = ConnectionManager::<PgConnection>::new(database_url);
    Pool::builder().build(manager)
}

// This type is simply wraps a Addr what we 
#[derive(Clone)]
pub struct State {
    pub inner: Addr<Db>
}

impl State {
    // The init method creates 
    pub fn init() -> State {
        let database_url = env::var("DATABASE_URL").expect("DATABASE_URL must be set");
        let pool = init_pool(&database_url).expect("Failed to create pool");
        let addr = SyncArbiter::start(DB_THREADS, move || Db(pool.clone()));
        let state = State {
            inner: addr.clone()
        };
        state
    }
    pub fn get(&self) -> &Addr<Db> {
        &self.inner
    }
}
```

我们有熟悉的 State 类型，但这里的 init 方法不同。它首先访问 `DATABASE_URL` 环境变量，并尝试使用 `pool` 中的 `database_url` 来建立一个池连接。然后我们启动一个 `SyncArbiter` 线程，克隆 `pool`。最后，我们将 `state` 实例返回给调用者。

除了这些，我们不需要对之前的 linksnap 代码库做太多修改。让我们试运行一下我们的新服务器。我们将使用 curl 将一个 Link 插入到我们的服务器中，如下所示：

```rs
curl --header "Content-Type: application/json" \
 --request POST \
 --data '{"title":"rust webpage","url":"https://rust-lang.org"}' \
 127.0.0.1:8080/add
```

现在为了确认这个链接已经到达我们的 postgres 数据库，让我们从 psql 提示符中查询这个条目：

![图片](img/1acba38a-6390-427f-856b-61382f6c1174.png)

太棒了！我们的 curl 请求已经到达了 postgres 数据库。

虽然 Diesel 的入门门槛有点高，但随着越来越多的示例和文献出现，情况应该会得到改善。

# 摘要

在本章中，我们简要地探讨了使用 Rust 通过低级别的 SQLite 和 PostgreSQL 库 crate 来执行基本数据库交互的几种方法。我们看到了如何使用 r2d2 来增强数据库连接性，并使用连接池。最后，我们使用 Diesel，一个安全且高效的 ORM，创建了一个小型应用程序。

在下一章中，我们将探讨如何使用名为 WebAssembly 的前沿技术使 Rust 能够在网络上运行。
