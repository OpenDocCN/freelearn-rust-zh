# 第七章：与数据库的可靠集成

持久化微服务需要存储和加载数据。如果您想保持这些数据组织有序且可靠，您应该使用数据库。Rust 有支持流行数据库的第三方 crate，在本章中，您将了解如何使用 Rust 与不同的数据库进行交互，包括以下内容：

+   PostgreSQL

+   MySQL

+   Redis

+   MongoDB

+   DynamoDB

我们将创建一些实用工具，允许您向数据库中插入或删除数据，以及查询数据库中存储的数据。

# 技术要求

在本章中，您需要数据库实例来运行我们的示例。最有效的方法是使用 Docker 进行数据库的运行和测试。您可以在本地安装数据库，但由于我们还需要在后续章节中使用 Docker，因此最好从本章开始安装并使用它。

我们将使用以下来自 Docker Hub 的官方镜像：

+   `postgres:11`

+   `mysql:8`

+   `redis:5`

+   `mongo:4`

+   `amazon/dynamodb-local:latest`

您可以在 Docker Hub 的仓库页面了解更多关于这些镜像的信息：[`hub.docker.com/`](https://hub.docker.com/).

我们还将使用作为 Amazon Web Services 一部分提供的`DynamoDB`数据库：[`aws.amazon.com/dynamodb/`](https://aws.amazon.com/dynamodb/).

如果您想与数据库交互以检查我们的示例是否成功运行，您还必须为每个数据库安装相应的客户端。

您可以在 GitHub 上的`Chapter07`文件夹中找到本章的所有示例：[`github.com/PacktPublishing/Hands-On-Microservices-with-Rust-2018/`](https://github.com/PacktPublishing/Hands-On-Microservices-with-Rust-2018/).

# PostgreSQL

PostgreSQL 是一个可靠且成熟的数据库。在本节中，我们将探讨如何在容器中启动该数据库的一个实例，以及如何使用第三方 crate 从 Rust 连接到它。我们将查看与该数据库的简单交互，以及使用连接池以获得额外性能的使用。我们将使用 Docker 启动数据库的一个实例，并创建一个工具，用于向表中添加记录，并在将它们打印到控制台之前查询已添加的记录列表。

# 设置测试数据库

要创建我们的数据库，您可以使用 Docker，它会自动拉取包含预安装 PostgreSQL 数据库的所有必要层。需要注意的是，PostgreSQL 在 Docker Hub 上有官方镜像，您应该选择使用这些镜像而不是非官方镜像，因为后者有更大的恶意更新风险。

我们需要启动一个包含 PostgreSQL 数据库实例的容器。您可以使用以下命令来完成此操作：

```rs
docker run -it --rm --name test-pg -p 5432:5432 postgres
```

这个命令做了什么？它从一个 `postgres` 镜像（最新版本）启动一个容器，并使用本地主机的 `5432` 端口将其转发到容器的内部端口 `5432`（即镜像暴露的端口）。我们还使用 `--name` 参数设置了一个名称。我们给容器命名为 `test-pg`。你可以稍后使用这个名称来停止容器。`--rm` 标志将在容器停止时删除与容器关联的匿名卷。为了能够从终端与数据库交互，我们添加了 `-it` 标志。

数据库实例将启动并在终端打印出类似以下内容：

```rs
creating subdirectories ... ok
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting dynamic shared memory implementation ... posix
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
........
```

数据库现在已准备好使用。如果你本地有 `psql` 客户端，你可以使用它来检查。该镜像的默认参数如下：

```rs
psql --host localhost --port 5432 --username postgres
```

如果你不再需要数据库，可以使用以下命令来关闭它：

```rs
docker stop test-pg
```

但现在不要关闭它——让我们用 Rust 连接到它。

# 简单的数据库交互

与数据库交互的最简单方式是直接创建一个到数据库的单个连接。简单交互是一个直接的数据库连接，它不使用连接池或其他抽象来最大化性能。

要连接到 PostgreSQL 数据库，我们可以使用两个 crate：`postgres` 或 `r2d2_postgres`。第一个是一个通用的连接驱动程序。第二个，`r2d2_postgres`，是 `r2d2` 连接池 crate 的一个 crate。我们将首先直接使用 `postgres` crate，而不使用 `r2d2` crate 的池，然后创建一个简单的实用工具来创建一个表，在添加命令来操作该表中的数据之前。

# 添加依赖项

让我们创建一个新的项目，包含所有必要的依赖项。我们将创建一个用于管理数据库中用户的二进制实用工具。创建一个新的二进制 crate：

```rs
cargo new --bin users
```

接下来，添加依赖项：

```rs
cargo add clap postgres
```

但等等！货物中不包含 `add` 命令。我已经安装了用于管理依赖项的 `cargo-edit` 工具。你可以使用以下命令来完成此操作：

```rs
cargo install cargo-edit
```

上述命令安装了 `cargo-edit` 工具。如果你没有安装它，你的本地 `cargo` 将不会有 `add` 命令。安装 `cargo-edit` 工具并添加 `postgres` 依赖项。你也可以通过编辑 `Cargo.toml` 文件手动添加依赖项，但鉴于我们将要创建更复杂的项目，`cargo-edit` 工具可以帮助我们节省时间。

Cargo 工具可以在以下位置找到：[`github.com/killercup/cargo-edit`](https://github.com/killercup/cargo-edit)。此工具包含三个有用的命令来管理依赖项：`add`用于添加依赖项，`rm`用于删除不必要的依赖项，`upgrade`用于将依赖项的版本升级到最新版本。此外，使用 Rust 的令人惊叹的 2018 版，你不需要使用`extern crate ...`声明。你可以简单地添加或删除任何 crate，并且它们将立即在所有模块中可用。但是，如果你添加了一个不需要的 crate，并且最终忘记了它怎么办？由于 Rust 编译器允许未使用的 crate，你可以在你的 crate 中添加以下 crate-wide 属性，`#![deny(unused_extern_crates)]`，以防你意外地添加了一个不会使用的 crate。

此外，添加`clap` crate。我们需要它来解析我们的工具的参数。按照以下方式添加所有必要类型的用法：

```rs
extern crate clap;
extern crate postgres;

use clap::{
    crate_authors, crate_description, crate_name, crate_version,
    App, AppSettings, Arg, SubCommand,
};
use postgres::{Connection, Error, TlsMode};
```

所有必要的依赖都已安装，并且我们已经导入了我们的类型，因此我们可以创建到数据库的第一个连接。

# 创建连接

在你可以在数据库上执行任何查询之前，你必须与你在容器中启动的数据库建立连接。使用以下命令创建一个新的`Connection`实例：

```rs
let conn = Connection::connect("postgres://postgres@localhost:5432", TlsMode::None).unwrap();
```

创建的`Connection`实例有`execute`和`query`方法。第一个方法用于执行 SQL 语句；第二个用于使用 SQL 查询数据。由于我们想要管理用户，让我们添加三个我们将与`Connection`实例一起使用的函数：`create_table`、`create_user`和`list_users`。

第一个函数`create_table`为用户创建一个表：

```rs
fn create_table(conn: &Connection) -> Result<(), Error> {
    conn.execute("CREATE TABLE users (
                    id SERIAL PRIMARY KEY,
                    name VARCHAR NOT NULL,
                    email VARCHAR NOT NULL
                  )", &[])
        .map(drop)
}
```

此函数使用一个`Connection`实例来执行一个创建`users`表的语句。由于我们不需要结果，我们可以简单地使用`Result`上的`map`命令来`drop`它。正如你所见，我们使用了一个不可变引用来引用连接，因为`Connection`包含对共享结构的引用，所以我们不需要改变这个值来与数据库交互。

关于使用哪种方法的讨论有很多：使用运行时锁和 Mutex 的不可变引用，还是即使需要运行时锁也使用可变引用。一些 crate 使用第一种方法，而其他 crate 使用第二种。在我看来，将你的方法适应于它将被调用的环境是好的。在某些情况下，避免可变引用可能更方便，但在大多数情况下，要求接口对象（如`postgres` crate 中的`Connection`）的可变性更安全。crate 的开发者也有一个计划将引用改为可变引用。你可以在这里了解更多信息：[`github.com/sfackler/rust-postgres/issues/346`](https://github.com/sfackler/rust-postgres/issues/346)。

下一个函数是`create_user`：

```rs
fn create_user(conn: &Connection, name: &str, email: &str) -> Result<(), Error> {
    conn.execute("INSERT INTO users (name, email) VALUES ($1, $2)",
                 &[&name, &email])
        .map(drop)
}
```

这个函数也使用了`Connection`的`execute`方法来插入一个值，但它还向调用中添加了参数来填充提供的语句（`create_table`函数将这些参数留空）。执行的结果被丢弃，我们只保留`Error`。如果请求返回插入记录的标识符，你可能需要返回值。

最后一个函数`list_users`查询数据库以从`users`表中获取用户列表。

```rs
fn list_users(conn: &Connection) -> Result<Vec<(String, String)>, Error> {
    let res = conn.query("SELECT name, email FROM users", &[])?.into_iter()
        .map(|row| (row.get(0), row.get(1)))
        .collect();
    Ok(res)
}
```

这个函数`list_users`使用了`Connection`的`query`方法。在这里，我们使用了一个简单的`SELECT` SQL 语句，将其转换为行的迭代器，并提取用户的名称和电子邮件地址对。

# 使用工具包装

我们已经准备好了所有查询，现在我们可以将它们在一个具有命令行界面的二进制工具中连接起来。在下面的代码中，我们将使用`clap` crate 解析一些参数，并运行函数来管理已建立的`Connection`的`users`表中的用户。

我们的工具将支持三个命令。将它们的名称声明为常量：

```rs
const CMD_CREATE: &str = "create";
const CMD_ADD: &str = "add";
const CMD_LIST: &str = "list";
```

现在，我们可以使用`clap` crate 创建`main`函数来解析我们的命令行参数：

```rs
fn main() -> Result<(), Error> {

    let matches = App::new(crate_name!())
        .version(crate_version!())
        .author(crate_authors!())
        .about(crate_description!())
        .setting(AppSettings::SubcommandRequired)
        .arg(
            Arg::with_name("database")
            .short("d")
            .long("db")
            .value_name("ADDR")
            .help("Sets an address of db connection")
            .takes_value(true),
            )
        .subcommand(SubCommand::with_name(CMD_CREATE).about("create users table"))
        .subcommand(SubCommand::with_name(CMD_ADD).about("add user to the table")
                    .arg(Arg::with_name("NAME")
                         .help("Sets the name of a user")
                         .required(true)
                         .index(1))
                    .arg(Arg::with_name("EMAIL")
                         .help("Sets the email of a user")
                         .required(true)
                         .index(2)))
        .subcommand(SubCommand::with_name(CMD_LIST).about("print list of users"))
        .get_matches();
    // Add connection here
}
```

如果失败，`main`函数返回`postgres::Error`，因为我们将要进行的所有操作都与我们的 Postgres 数据库连接相关。在这里，我们创建了一个`clap::App`实例，并添加了一个`--database`参数，让用户更改连接地址。我们还添加了三个子命令`create`、`add`和`list`，以及`add`命令的额外参数，该参数需要用户的名称和电子邮件地址，以便我们可以将其插入数据库。

要创建一个`Connection`实例，我们使用数据库参数来提取用户通过`--db`命令行参数提供的连接 URL，如果没有提供，我们将使用默认 URL 值`postgres://postgres@localhost:5432`：

```rs
let addr = matches.value_of("database")
    .unwrap_or("postgres://postgres@localhost:5432");
let conn = Connection::connect(addr, TlsMode::None)?;
```

我们使用了一个带有地址的`Connection::connect`方法，并将`TlsMode`参数设置为`TlsMode::None`，因为我们演示中不使用 TLS。我们创建了一个名为`conn`的`Connection`实例来调用我们的函数与数据库交互。

最后，我们可以为子命令添加分支：

```rs
match matches.subcommand() {
    (CMD_CREATE, _) => {
        create_table(&conn)?;
    }
    (CMD_ADD, Some(matches)) => {
        let name = matches.value_of("NAME").unwrap();
        let email = matches.value_of("EMAIL").unwrap();
        create_user(&conn, name, email)?;
    }
    (CMD_LIST, _) => {
        let list = list_users(&conn)?;
        for (name, email) in list {
            println!("Name: {:20}    Email: {:20}", name, email);
        }
    }
    _ => {
        matches.usage(); // but unreachable
    }
}
Ok(())
```

第一分支匹配`crate`子命令，通过调用`create_table`函数创建一个表。

第二分支是针对`add`子命令的。它提取用户名称和电子邮件地址所需的参数对，并调用`create_user`函数来创建一个包含提供值的用户记录。我们使用`unwrap`来提取它，因为这两个参数都是必需的。

倒数第二个分支处理`list`命令，通过调用`list_users`函数来获取用户列表。在取值之后，它在一个`for`循环中使用，将所有用户的记录打印到控制台。

最后一个分支不可达，因为我们将`AppSettings::SubcommandRequired`设置为`clap::App`，但我们保留它以保持一致性。如果你想在子命令值未设置时提供默认行为，这特别有用。

# 编译和运行

在本章的开头，我们启动了一个 PostgreSQL 数据库实例，我们将使用它来检查我们的工具。使用以下命令编译我们创建的示例并打印可用的子命令：

```rs
cargo run -- --helpYou will see the next output:
USAGE:
 users [OPTIONS] <SUBCOMMAND>
FLAGS:
 -h, --help       Prints help information
 -V, --version    Prints version information
OPTIONS:
 -d, --db <ADDR>    Sets an address of db connection
SUBCOMMANDS:
 add       add user to the table
 create    create users table
 help      Prints this message or the help of the given subcommand(s)
 list      print list of users
```

Cargo 看起来是一个管理应用程序数据库的可爱工具。让我们用它创建一个表格，如下所示：

```rs
cargo run -- create
```

此命令创建一个`users`表。如果你再次尝试运行它，你会得到一个错误：

```rs
Error: Error(Db(DbError { severity: "ERROR", parsed_severity: Some(Error), code: SqlState("42P07"), message: "relation \"users\" already exists", detail: None, hint: None, position: None, where_: None, schema: None, table: None, column: None, datatype: None, constraint: None, file: Some("heap.c"), line: Some(1084), routine: Some("heap_create_with_catalog") }))
```

如果你使用`psql`客户端检查你的表，你将看到我们数据库中的表：

```rs
postgres=# \dt
 List of relations
 Schema | Name  | Type  |  Owner 
--------+-------+-------+----------
 public | users | table | postgres
(1 row)
```

要添加新用户，使用以下参数调用`add`子命令：

```rs
cargo run -- add user-1 user-1@example.com
cargo run -- add user-2 user-2@example.com
cargo run -- add user-3 user-3@example.com
```

我们添加了三个用户，如果你输入`list`子命令，你可以在列表中看到他们：

```rs
cargo run -- list
Name: user-1   Email: user-1@example.com 
Name: user-2   Email: user-2@example.com 
Name: user-3   Email: user-3@example.com  
```

在以下示例中，我们将使用数据库连接池来并行添加多个用户。

# 连接池

我们创建的工具使用数据库的单个连接。对于少量查询来说，它工作得很好。如果你想并行运行多个查询，你必须使用连接池。在本节中，我们通过`import`命令改进了工具，该命令从 CSV 文件导入大量用户数据。我们将使用`r2d2` crate 的`Pool`类型，添加一个读取用户文件的命令，并执行并行将用户添加到表中的语句。

# 创建连接池

要创建连接池，我们将使用可以保存多个连接并为我们从池中提供连接的`r2d2` crate。这个 crate 是泛型的，所以你需要为每个要连接的数据库提供一个特定的实现。`r2d2` crate 可以使用适配器 crate 连接以下数据库：

+   PostgreSQL

+   Redis

+   MySQL

+   SQLite

+   Neo4j

+   Diesel ORM

+   CouchDB

+   MongoDB

+   ODBC

在我们的例子中，我们需要`r2d2-postgres`适配器 crate 来连接到 PostgreSQL 数据库。使用`r2d2` crate 将其添加到我们的依赖项中：

```rs
[dependencies]
clap = "2.32"
csv = "1.0"
failure = "0.1"
postgres = "0.15"
r2d2 = "0.8"
r2d2_postgres = "0.14"
rayon = "1.0"
serde = "1.0"
serde_derive = "1.0"
```

我们还保留了`postgres`依赖项，并添加了`failure`用于错误处理和`rayon`以并行执行 SQL 语句。我们还添加了一套`serde` crate 来从 CSV 文件反序列化`User`记录，以及`csv` crate 来读取该文件。

你将更习惯于使用 Rust 结构体来表示数据库中的数据记录。让我们添加一个`User`类型，它代表数据库中的用户记录，如下所示的结构体：

```rs
#[derive(Deserialize, Debug)]
struct User {
 name: String,
 email: String,
}
```

由于我们有我们的特殊`User`类型，我们可以改进`create_user`和`list_users`函数以使用这种新类型：

```rs
fn create_user(conn: &Connection, user: &User) -> Result<(), Error> {
    conn.execute("INSERT INTO users (name, email) VALUES ($1, $2)",
                 &[&user.name, &user.email])
        .map(drop)
}

fn list_users(conn: &Connection) -> Result<Vec<User>, Error> {
    let res = conn.query("SELECT name, email FROM users", &[])?.into_iter()
        .map(|row| {
            User {
                name: row.get(0),
                email: row.get(1),
            }
        })
        .collect();
    Ok(res)
}
```

它的变化并不大：我们仍然使用相同的`Connection`类型，但我们使用`User`结构体中的字段来填充我们的`create`语句并从我们的`get list`查询中提取值。`create_table`函数没有变化。

为`import`命令添加一个常量：

```rs
const CMD_IMPORT: &str = "import";
```

然后，将其作为`SubCommand`添加到`App`中：

```rs
.subcommand(SubCommand::with_name(CMD_IMPORT).about("import users from csv"))
```

几乎所有分支都有所改变，我们应该探索这些变化。`add`命令创建一个`User`实例来调用`create_user`函数：

```rs
(CMD_ADD, Some(matches)) => {
    let name = matches.value_of("NAME").unwrap().to_owned();
    let email = matches.value_of("EMAIL").unwrap().to_owned();
    let user = User { name, email };
    create_user(&conn, &user)?;
}
```

`list`子命令返回一个`User`结构体实例的列表。我们必须注意这个变化：

```rs
(CMD_LIST, _) => {
    let list = list_users(&conn)?;
    for user in list {
        println!("Name: {:20}    Email: {:20}", user.name, user.email);
    }
}
```

`import`命令更复杂，所以让我们在下一节中更详细地讨论这个问题。

# 使用 rayon crate 进行并行导入

由于我们有连接池，我们可以并行运行对数据库的多个请求。我们将从 CSV 格式的标准输入流中读取用户。让我们为之前声明的`match`表达式添加一个分支，用于`import`子命令，并使用`csv::Reader`打开`stdin`。之后，我们将使用读取器的`deserialize`方法，它返回我们所需类型的反序列化实例的迭代器。在我们的情况下，我们将 CSV 数据反序列化为`User`结构体的列表并将它们推送到向量中：

```rs
(CMD_IMPORT, _) => {
    let mut rdr = csv::Reader::from_reader(io::stdin());
    let mut users = Vec::new();
    for user in rdr.deserialize() {
        users.push(user?);
    }
    // Put parallel statements execution here
}
```

# Rayon

为了并行运行请求，我们将使用`rayon` crate，它提供了具有`par_iter`方法的并行迭代器。并行迭代器将列表分割成在线程池中运行的单独任务：

```rs
users.par_iter()
    .map(|user| -> Result<(), failure::Error> {
        let conn = pool.get()?;
        create_user(&conn, &user)?;
        Ok(())
    })
    .for_each(drop);
```

并行迭代器返回项目的方式与传统迭代器类似。我们可以使用`Pool::get`方法从连接池中获取一个连接，并使用连接的引用调用`create_user`函数。我们在这里也忽略结果，如果任何请求失败，它将被静默跳过，正如在演示中，我们无法处理尚未插入的值。由于我们使用多个连接，如果任何语句失败，我们无法使用事务来回滚更改。

`rayon` crate 看起来非常令人印象深刻且易于使用。您可能会问：*您可以在微服务中使用这个 crate 吗？* 答案是：*是的!* 但请记住，为了收集数据，您必须调用`for_each`方法，这将阻塞当前线程直到所有任务完成。如果您在异步`Future`的上下文中（我们在第五章，*使用 Futures Crate 理解异步操作）中调用它，它将阻塞反应器一段时间。

在下一节中，我们将重写针对 MySQL 数据库的此示例。

# MySQL

MySQL 是最受欢迎的数据库之一，因此 Rust 自然有与之交互的 crate。我推荐您使用两个很好的 crate：`mysql` crate 及其异步版本`mysql_async` crate。

在本节中，我们将重写之前管理用户时支持 MySQL 数据库的示例。我们还将在一个容器中启动数据库的本地实例，并创建一个命令行实用程序，该实用程序连接到数据库实例，发送创建表的查询，并允许我们添加和删除用户。我们将使用最新的 PostgreSQL 示例，它使用`r2d2`连接池。

# 测试数据库

为了启动数据库，我们也将使用 Docker 镜像。您可以在本地安装 MySQL，但容器是一种更灵活的方法，它不会阻塞系统，并且您可以轻松地在几秒钟内为测试目的启动一个空数据库。

MySQL 数据库有一个官方镜像，`mysql`，您可以在以下位置找到：[`hub.docker.com/_/mysql`](https://hub.docker.com/_/mysql)。您可以使用以下命令使用这些镜像加载和运行容器：

```rs
docker run -it --rm --name test-mysql -e MYSQL_ROOT_PASSWORD=password -e MYSQL_DATABASE=test -p 3306:3306 mysql
```

您可以使用环境变量设置两个必要的参数。首先，`MYSQL_ROOT_PASSWORD`环境变量为 root 用户设置密码。其次，`MYSQL_DATABASE`环境变量设置了一个默认数据库的名称，该数据库将在容器首次启动时创建。我们命名我们的容器为`test-mysql`，并将本地端口`3306`映射到容器内的`3306`端口。

为了确保我们的容器已启动，您可以使用本地安装的`mysql`客户端：

```rs
mysql -h 127.0.0.1 -P 3306 -u root -p test
```

之前的命令连接到`127.0.0.1`（为了避免使用套接字）的`3306`端口，用户名为`root`。`-p`参数请求连接的密码。我们为我们的测试容器设置了一个密码，因为数据库镜像需要它。

我们的数据库已准备好使用。您也可以使用以下命令停止它：

```rs
docker stop test-mysql
```

# 使用 r2d2 适配器连接

在上一节中，我们使用了来自`r2d2` crate 的连接池与 PostgreSQL 数据库进行交互。在`r2d2-mysql` crate 中也有一个 MySQL 的连接管理器，允许您使用`r2d2` crate 来使用 MySQL 连接。`r2d2-mysql` crate 基于`mysql` crate。使用连接池与 PostgreSQL 数据库类似简单，但在这里，我们使用`MysqlConnectionManager`作为`r2d2::Pool`的类型参数。让我们修改所有带有查询的函数，以使用我们 MySQL 数据库的连接池。

# 添加依赖项

首先，我们必须添加依赖项以建立到 MySQL 的连接。我们使用与上一个示例中相同的所有依赖项，但将`postgres`替换为`mysql` crate，将`r2d2_postgres`替换为`r2d2_mysql` crate：

```rs
mysql = "14.1"
r2d2_mysql = "9.0"
```

我们仍然需要`csv`、`rayon`、`r2d2`和`serde`家族的 crate。

您还必须声明其他类型，以便在代码中使用，如下所示：

```rs
use mysql::{Conn, Error, Opts, OptsBuilder};
use r2d2_mysql::MysqlConnectionManager;
```

# 数据库交互函数

现在，我们可以用`mysql` crate 的`Conn`替换`postgres` crate 中的`Connection`实例，以提供我们的交互函数。第一个函数`create_table`使用对`Conn`实例的可变引用：

```rs
fn create_table(conn: &mut Conn) -> Result<(), Error> {
    conn.query("CREATE TABLE users (
                    id INT(6) UNSIGNED AUTO_INCREMENT PRIMARY KEY,
                    name VARCHAR(50) NOT NULL,
                    email VARCHAR(50) NOT NULL
                  )")
        .map(drop)
}
```

此外，我们使用了`Conn`连接对象的`query`方法来发送查询。此方法不期望参数。我们仍然忽略查询的成功结果，并用`map`将其`drop`。

下一个函数`create_user`已转换为以下形式：

```rs
fn create_user(conn: &mut Conn, user: &User) -> Result<(), Error> {
     conn.prep_exec("INSERT INTO users (name, email) VALUES (?, ?)",
                  (&user.name, &user.email))
         .map(drop)
 }
```

我们使用`Conn`的`prep_exec`方法，它期望一个参数元组，这些参数是从`User`结构体字段中提取的。如您所见，我们使用了`?`字符来指定插入值的位置。

最后一个函数`list_users`从查询中收集用户。它比 PostgreSQL 版本更复杂。我们使用了返回实现`Iterator`特质的`QueryResult`类型的`query`方法。我们使用这个属性将结果转换为迭代器，并在`Iterator`实现的`try_fold`方法中尝试将值折叠到向量中：

```rs
fn list_users(conn: &mut Conn) -> Result<Vec<User>, Error> {
    conn.query("SELECT name, email FROM users")?
        .into_iter()
        .try_fold(Vec::new(), |mut vec, row| {
            let row = row?;
            let user = User {
                name: row.get_opt(0).unwrap()?,
                email: row.get_opt(1).unwrap()?,
            };
            vec.push(user);
            Ok(vec)
        })
}
```

对于`try_fold`方法调用，我们提供一个闭包，它期望两个参数：第一个是我们通过`try_fold`调用传递的向量，而第二个是一个`Row`实例。我们使用`try_fold`在行转换到用户失败时返回`Error`。

我们使用`Row`对象的`get_opt`方法来获取相应类型的值，并使用`?`运算符从结果中提取它，或者使用`try_fold`返回`Error`。在每次迭代中，我们返回一个包含新附加值的向量。

# 创建连接池

我们将重用前一个示例中的参数解析器，但将重写建立连接的代码，因为我们现在使用的是 MySQL 而不是 PostgreSQL。首先，我们将数据库链接替换为`mysql`方案。我们将使用与启动 MySQL 服务器实例相同的参数来建立连接。

我们将地址字符串转换为`Opts` - 连接的选项，这是用于设置连接参数的 mysql crate 的类型。但是`MysqlConnectionManager`期望我们提供一个`OptsBuilder`对象。看看下面的代码：

```rs
let addr = matches.value_of("database")
    .unwrap_or("mysql://root:password@localhost:3306/test");
let opts = Opts::from_url(addr)?;
let builder = OptsBuilder::from_opts(opts);
let manager = MysqlConnectionManager::new(builder);
let pool = r2d2::Pool::new(manager)?;
let mut conn = pool.get()?;
```

现在，我们可以使用`builder`创建`MysqlConnectionManager`，并且我们可以使用`manager`实例创建`r2d2::Pool`。我们还得到一个可变的`conn`引用，以便为子命令提供它。

好消息是，这已经足够开始了。我们不需要在我们的分支中做任何改变，除了引用的类型。现在，我们必须传递一个可变的引用到连接：

```rs
(CMD_CRATE, _) => {
    create_table(&mut conn)?;
}
```

尝试启动并检查工具的工作情况。我们将提供一个具有以下格式的 CSV 文件作为输入：

```rs
name,email
user01,user01@example.com
user02,user02@example.com
user03,user03@example.com
```

如果你想检查数据库是否真的发生了变化，请尝试从我们的 CSV 文件导入用户数据：

```rs
cargo run -- import < users.csv
```

你可以使用`mysql`客户端打印`users`表：

```rs
mysql> SELECT * FROM users;
+----+--------+--------------------+
| id | name   | email              |
+----+--------+--------------------+
|  1 | user01 | user01@example.com |
|  2 | user03 | user03@example.com |
|  3 | user08 | user08@example.com |
|  4 | user06 | user06@example.com |
|  5 | user02 | user02@example.com |
|  6 | user07 | user07@example.com |
|  7 | user04 | user04@example.com |
|  8 | user09 | user09@example.com |
|  9 | user10 | user10@example.com |
| 10 | user05 | user05@example.com |
+----+--------+--------------------+
10 rows in set (0.00 sec)
```

它工作了！正如你所见，用户以不可预测的顺序被添加，因为我们使用了多个连接和真正的并发。现在你了解了如何使用 SQL 数据库。是时候看看如何通过`r2d2`crate 与 NoSQL 数据库交互了。

# Redis

当编写微服务时，你可能有时需要一个可以按键存储值的存储库；例如，如果你想存储会话，你可以存储会话的保护标识符，并在持久缓存中保留有关用户的其他信息。如果会话数据丢失不是问题；相反，定期清理会话是一个最佳实践，以防用户的会话标识符被盗。

Redis 是一个流行的内存数据结构存储，适用于此用例。它可以用作数据库、消息代理或缓存。在下一节中，我们将使用 Docker 运行 Redis 实例并创建一个命令行工具，帮助管理 Redis 中的用户会话。

# 为测试设置数据库

Redis 在 Docker Hub 上有一个官方镜像，名为`redis`。要创建和运行容器，请使用以下命令：

```rs
docker run -it --rm --name test-redis -p 6379:6379 redis
```

此命令从`redis`镜像运行一个名为`test-redis`的容器，并将本地端口`6379`转发到容器的内部端口`6379`。

关于 Redis 的一个有趣的事实是，它使用一个非常简单和直接的交互协议。您甚至可以使用 `telnet` 与 Redis 交互：

```rs
telnet 127.0.0.1 6379
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
SET session-1 "Rust"
+OK
GET session-1
$4
Rust
^]
```

原生客户端更易于使用，但它期望与原始协议相同的命令。

要关闭运行 Redis 的容器，请使用以下命令：

```rs
docker stop test-redis
```

让我们创建一个用于管理 Redis 会话的工具。

# 创建连接池

我们已经在 Docker 容器中启动了一个 Redis 实例，因此现在我们可以开始创建一个命令行工具，允许我们连接到该数据库实例并将一些信息放入其中。这个实用程序将不同于我们为 PostgreSQL 和 MySQL 创建的工具，因为 Redis 不使用 SQL 语言。我们将使用 Redis 中可用的特定 API 方法。

在本节中，我们将创建一个新的二进制 crate 并添加使用 `r2d2::Pool` 从 Redis 设置或获取数据的函数。之后，我们将根据用户指定的命令行参数作为子命令来调用它们。

# 依赖项

创建一个新的二进制 crate，并将所有必要的依赖项添加到该 crate 的`Cargo.toml`文件中：

```rs
[dependencies]
clap = "2.32"
failure = "0.1"
r2d2 = "0.8"
r2d2_redis = "0.8"
redis = "0.9"
```

我们添加了本章前面示例中使用的依赖项——`clap`、`failure` 和 `r2d2`。此外，我们还需要`redis`和`r2d2_redis` crate，它们包含 Redis 的连接管理器，以便我们可以使用 `r2d2` crate 的 `Pool`。

接下来，让我们导入创建工具所需的类型：

```rs
use clap::{
    crate_authors, crate_description, crate_name, crate_version,
    App, AppSettings, Arg, SubCommand,
};
use redis::{Commands, Connection, RedisError};
use r2d2_redis::RedisConnectionManager;
use std::collections::HashMap;
```

注意一些类型的用法。我们将 `Connection` 作为主要连接类型导入，我们将使用它来连接到 Redis 实例。我们还从 `r2d2_redis` crate 中导入了 `RedisConnectionManager`。此类型允许 `Pool` 创建新的连接。您应该注意的最后一件事情是 `Command` trait。这个 trait 包含反映 Redis 客户端 API 的方法。方法名称与 Redis 协议中相同（但全部小写），如前节中手动测试的那样。`Command` trait，由 `Connection` 结构体实现，允许您调用 Redis API 的方法。

Redis 支持许多命令。您可以在[`redis.io/commands`](https://redis.io/commands)找到完整的列表。`redis` crate 提供了其中大部分作为 `Command` trait 的方法。

# 添加命令和交互函数

我们正在为 Redis 创建的工具将支持三个命令：

+   `add` - 添加新的会话记录

+   `remove` - 通过键（即用户名）删除会话记录

+   `list` - 打印所有会话记录

我们需要为每个子命令的名称定义常量，以防止代码中字符串的错误：

```rs
const SESSIONS: &str = "sessions";
const CMD_ADD: &str = "add";
const CMD_REMOVE: &str = "remove";
const CMD_LIST: &str = "list";
```

此列表还包含 `SESSION` 常量，作为 Redis 中 `HashMap` 的名称。现在，我们可以声明用于操作会话数据的函数。

# 数据操作函数

我们的例子需要三个函数。第一个函数，`add_session`，在令牌和用户 ID 之间添加关联：

```rs
fn add_session(conn: &Connection, token: &str, uid: &str) -> Result<(), RedisError> {
    conn.hset(SESSIONS, token, uid)
}
```

此函数仅调用 `Connection` 的 `hset` 方法，并通过 `token` 键在 `SESSIONS` 映射中设置 `uid` 值。如果设置操作出现错误，则返回 `RedisError`。

下一个函数，`remove_session`，也非常简单，它调用了 `Connection` 的 `hdel` 方法：

```rs
fn remove_session(conn: &Connection, token: &str) -> Result<(), RedisError> {
    conn.hdel(SESSIONS, token)
}
```

此函数从 `SESSIONS` 映射中删除具有 `token` 键的记录。

最后一个函数，`list_sessions`，从 `SESSION` 映射中返回所有令牌-uid 对，作为一个 `HashMap` 实例。它使用 `Connection` 的 `hgetall` 方法，该方法调用 Redis 中的 `HGETALL` 方法：

```rs
fn list_sessions(conn: &Connection) -> Result<HashMap<String, String>, RedisError> {
     conn.hgetall(SESSIONS)
}
```

如您所见，所有函数都映射到原始 Redis 命令，看起来非常简单。但所有函数在后台也做得很好，将值转换为相应的 Rust 类型。

现在，我们可以为会话工具创建一个参数解析器。

# 解析参数

由于我们的命令支持三个子命令，我们必须将它们添加到 `clap::App` 实例中：

```rs
let matches = App::new(crate_name!())
    .version(crate_version!())
    .author(crate_authors!())
    .about(crate_description!())
    .setting(AppSettings::SubcommandRequired)
    .arg(
        Arg::with_name("database")
        .short("d")
        .long("db")
        .value_name("ADDR")
        .help("Sets an address of db connection")
        .takes_value(true),
        )
    .subcommand(SubCommand::with_name(CMD_ADD).about("add a session")
                .arg(Arg::with_name("TOKEN")
                     .help("Sets the token of a user")
                     .required(true)
                     .index(1))
                .arg(Arg::with_name("UID")
                     .help("Sets the uid of a user")
                     .required(true)
                     .index(2)))
    .subcommand(SubCommand::with_name(CMD_REMOVE).about("remove a session")
                .arg(Arg::with_name("TOKEN")
                     .help("Sets the token of a user")
                     .required(true)
                     .index(1)))
    .subcommand(SubCommand::with_name(CMD_LIST).about("print list of sessions"))
    .get_matches();
```

如前例所示，这也可以使用带有 Redis 连接链接的 `--database` 参数。它支持两个子命令。`add` 子命令期望一个会话 `TOKEN` 和用户的 `UID`。`remove` 命令期望一个会话 `TOKEN`，仅用于从映射中删除它。`list` 命令不期望任何参数，并打印会话列表。

想象这个例子中的数据结构是一个会话缓存，它持有 `token` 和 `uid` 之间的关联。在授权后，我们可以将令牌作为安全 cookie 发送，并为每个微服务提供的令牌提取用户的 `uid`，以实现微服务之间的松耦合。我们将在稍后详细探讨这个概念。

现在，我们准备好使用 `r2d2::Pool` 连接到 Redis。

# 连接到 Redis

`r2d2` 连接到 Redis 的方式与其他数据库类似：

```rs
let addr = matches.value_of("database")
    .unwrap_or("redis://127.0.0.1/");
let manager = RedisConnectionManager::new(addr)?;
let pool = r2d2::Pool::builder().build(manager)?;
let conn = pool.get()?;
```

我们从 `--database` 参数中获取地址，但如果它未设置，我们将使用默认值 `redis://127.0.0.1/`。之后，我们将创建一个新的 `RedisConnectionManager` 实例，并将其传递给 `Pool::new` 方法。

# 执行子命令

我们使用分支的结构来匹配我们之前示例中的子命令：

```rs
match matches.subcommand() {
    (CMD_ADD, Some(matches)) => {
        let token = matches.value_of("TOKEN").unwrap();
        let uid = matches.value_of("UID").unwrap();
        add_session(&conn, token, uid)?;
    }
    (CMD_REMOVE, Some(matches)) => {
        let token = matches.value_of("TOKEN").unwrap();
        remove_session(&conn, token)?;
    }
    (CMD_LIST, _) => {
        println!("LIST");
        let sessions = list_sessions(&conn)?;
        for (token, uid) in sessions {
            println!("Token: {:20}   Uid: {:20}", token, uid);
        }
    }
    _ => { matches.usage(); }
}
```

对于 `add` 子命令，我们从参数中提取 `TOKEN` 和 `UID` 值，并将它们与 `Connector` 的引用一起传递给 `add_session` 函数。对于 `remove` 子命令，我们仅提取 `TOKEN` 值，并使用相应的参数调用 `remove_session` 函数。对于 `list` 子命令，我们直接调用 `list_session` 函数，因为我们不需要任何额外的参数来从映射中获取所有值。这返回一个对向量。对中的第一个元素包含 `token`，第二个包含 `uid`。我们使用固定宽度指定符 `{:20}` 打印值。

# 测试我们的 Redis 示例

让我们编译并测试这个工具。我们将添加三个用户会话：

```rs
cargo run -- add 7vQ2MhnRcyYeTptp a73bbfe3-df6a-4dea-93a8-cb4ea3998a53
cargo run -- add pTySt8FI7TIqId4N 0f3688be-0efc-4744-829c-be5d177e0e1c
cargo run -- add zJx3mBRpJ9WTkwGU f985a744-6648-4d0a-af5c-0b71aecdbcba
```

要打印列表，请运行 `list` 命令：

```rs
cargo run -- list
```

使用此方法，您将看到您创建的所有会话：

```rs
LIST
Token: pTySt8FI7TIqId4N       Uid: 0f3688be-0efc-4744-829c-be5d177e0e1c
Token: zJx3mBRpJ9WTkwGU       Uid: f985a744-6648-4d0a-af5c-0b71aecdbcba
Token: 7vQ2MhnRcyYeTptp       Uid: a73bbfe3-df6a-4dea-93a8-cb4ea3998a53
```

我们已经学习了如何使用 Redis。它对于存储用于缓存的消息很有用。接下来，我们将查看最后一个 NoSQL 数据库：MongoDB。

# MongoDB

MongoDB 是一个流行的 NoSQL 数据库，具有出色的功能和良好的性能。它非常适合结构快速变化的数据，如下所示：

+   运营智能（日志和报告）

+   产品数据管理（产品目录、层次结构和类别）

+   内容管理系统（帖子、评论和其他记录）

我们将创建一个示例来存储用户的操作。

# 为测试启动数据库

我们将使用官方 Docker 镜像来启动 MongoDB 实例。您可以使用以下命令简单地完成此操作：

```rs
docker run -it --rm --name test-mongo -p 27017:27017 mongo
```

此命令从 `mongo` 镜像运行一个名为 `test-mongo` 的容器，并将本地端口 `27017` 转发到容器的相同内部端口。容器关闭后，容器产生的数据将被删除。

如果您有一个 `mongo` 客户端，您可以使用它连接到容器内的数据库实例：

```rs
mongo 127.0.0.1:27017/admin
```

当您需要关闭容器时，使用 `docker` 的 `stop` 子命令并指定容器的 `name`：

```rs
docker stop test-mongo
```

如果您将容器附加到带有 `-it` 参数的终端，则可以使用 *Ctrl *+ *C* 终止它，就像我之前做的那样。

现在，我们可以看看如何使用 `mongo` 和 `r2d2-mongo` crate 连接到数据库。

# 使用 r2d2 池连接到数据库

按照惯例，我们将使用 `r2d2` crate 中的 `Pool`，但在本例（以及 Redis 示例）中，我们不会同时使用多个连接。将所有必要的依赖项添加到新的二进制 crate：

```rs
[dependencies]
bson = "0.13"
chrono = { version = "0.4", features = ["serde"] }
clap = "2.32"
failure = "0.1"
mongodb = "0.3"
r2d2 = "0.8"
r2d2-mongodb = "0.1"
serde = "1.0"
serde_derive = "1.0"
url = "1.7"
```

列表并不短。除了您已经熟悉的 crates 之外，我们还添加了 `bson`、`chrono` 和 `url` crates。第一个 crate 我们需要与数据库中的数据一起工作；第二个，用于使用 `Utc` 类型；最后一个用于将 URL 字符串拆分成片段。

按照以下方式导入所有必要的类型：

```rs
use chrono::offset::Utc;
use clap::{
    crate_authors, crate_description, crate_name, crate_version,
    App, AppSettings, Arg, SubCommand,
};
use mongodb::Error;
use mongodb::db::{Database, ThreadedDatabase};
use r2d2::Pool;
use r2d2_mongodb::{ConnectionOptionsBuilder, MongodbConnectionManager};
use url::Url;
```

此用户的日志工具将支持两个命令：`add` 用于添加记录，`list` 用于打印所有记录的列表。添加以下必要的常量：

```rs
const CMD_ADD: &str = "add";
const CMD_LIST: &str = "list";
```

要设置和获取结构化数据，我们需要声明一个 `Activity` 结构体，该结构体将用于创建 BSON 文档以及从 BSON 数据中恢复它，因为 MongoDB 使用此格式进行数据交互。`Activity` 结构体有三个字段，`user_id`、`activity` 和 `datetime`：

```rs
#[derive(Deserialize, Debug)]
struct Activity {
    user_id: String,
    activity: String,
    datetime: String,
}
```

# 交互函数

由于我们已声明结构体，我们可以添加与数据库一起工作的函数。我们将添加的第一个函数是 `add_activity`，它将活动记录添加到数据库中：

```rs
fn add_activity(conn: &Database, activity: Activity) -> Result<(), Error> {
    let doc = doc! {
        "user_id": activity.user_id,
        "activity": activity.activity,
        "datetime": activity.datetime,
    };
    let coll = conn.collection("activities");
    coll.insert_one(doc, None).map(drop)
}
```

此函数仅将 `Activity` 结构体转换为 BSON 文档，通过从结构体中提取字段并使用相同字段构建 BSON 文档来实现。我们可以为结构体推导出 `Serialize` 特性以使用自动序列化，但为了演示目的，我使用了 `doc!` 宏来展示你可以添加一个可以即时构建的自由格式文档。

要添加 `Activity`，我们通过 `collection()` 方法从 `Database` 实例中获取一个名为 `activities` 的集合，并调用 `Collection` 的 `insert_one` 方法来添加记录。

下一个方法是 `list_activities`。此方法使用 `Database` 实例来查找 `activities` 集合中的所有值。我们使用 `Collection` 的 `find()` 方法来获取数据，但请确保将过滤器（第一个参数）设置为 `None`，将选项（第二个参数）设置为 `None`，以获取集合中的所有值。

你可以调整这些参数进行过滤，或者限制你检索的记录数量：

```rs
fn list_activities(conn: &Database) -> Result<Vec<Activity>, Error> {
    conn.collection("activities").find(None, None)?
        .try_fold(Vec::new(), |mut vec, doc| {
            let doc = doc?;
            let activity: Activity = bson::from_bson(bson::Bson::Document(doc))?;
            vec.push(activity);
            Ok(vec)
        })
}
```

要将 `find` 查询返回的每个记录转换为 BSON 文档，我们可以使用 `bson::from_bson` 方法，因为我们已经为 `Activity` 结构体推导出 `Deserialize` 特性。`try_fold` 方法允许我们在转换失败时中断折叠。我们将所有成功转换的值推送到我们提供给 `try_fold` 方法调用的第一个参数的向量中。现在，我们可以解析参数，以便我们可以准备一个池来调用声明的交互函数。

# 解析参数

我们的工具期望两个子命令：`add` 和 `list`。让我们将它们添加到 `clap::App` 实例中。像所有之前的示例一样，我们也添加了一个 `--database` 参数来设置连接 URL。请看以下代码：

```rs
let matches = App::new(crate_name!())
    .version(crate_version!())
    .author(crate_authors!())
    .about(crate_description!())
    .setting(AppSettings::SubcommandRequired)
    .arg(
        Arg::with_name("database")
        .short("d")
        .long("db")
        .value_name("ADDR")
        .help("Sets an address of db connection")
        .takes_value(true),
        )
    .subcommand(SubCommand::with_name(CMD_ADD).about("add user to the table")
                .arg(Arg::with_name("USER_ID")
                     .help("Sets the id of a user")
                     .required(true)
                     .index(1))
                .arg(Arg::with_name("ACTIVITY")
                     .help("Sets the activity of a user")
                     .required(true)
                     .index(2)))
    .subcommand(SubCommand::with_name(CMD_LIST).about("print activities list of users"))
    .get_matches();
```

`add` 子命令期望两个参数：`USER_ID` 和 `ACTIVITY`。在 `Activity` 结构体中，这两个参数都表示为 `String` 类型的值。我们将要求这些参数，但我们将获取任何提供的值，没有任何限制。`list` 子命令没有额外的参数。

# 创建连接池

要连接到数据库，我们从 `--database` 命令行参数中提取连接 URL。如果没有设置，我们使用默认值 `mongodb://localhost:27017/admin`：

```rs
let addr = matches.value_of("database")
    .unwrap_or("mongodb://localhost:27017/admin");
let url = Url::parse(addr)?;
```

但我们也将它解析到 `Url` 结构体中。这是必要的，因为 MongoDB 连接期望选项集通过单独的值来收集：

```rs
let opts = ConnectionOptionsBuilder::new()
    .with_host(url.host_str().unwrap_or("localhost"))
    .with_port(url.port().unwrap_or(27017))
    .with_db(&url.path()[1..])
    .build();

let manager = MongodbConnectionManager::new(opts);

let pool = Pool::builder()
    .max_size(4)
    .build(manager)?;

let conn = pool.get()?;
```

在前面的代码中，我们创建了一个新的`ConnectionOptionsBuilder`实例，并用从解析的`Url`实例中获取的值填充它。我们设置了`host`、`port`和`db`名称。如您所见，我们跳过了路径的第一个字符，以便将其用作数据库的名称。调用`build`方法来构建`ConnectionOptions`结构体。现在，我们可以创建一个`MongodbConnectionManager`实例，并使用它来创建一个`Pool`实例。但是，在这个例子中，我们调用的是`builder`方法，而不是`new`，以向您展示如何设置`Pool`实例中的连接数。我们将此设置为`4`。之后，我们调用`build`方法来创建一个`Pool`实例。与之前的示例一样，我们调用`Pool`的`get`方法来从一个池中获取`Database`连接对象。

# 实现子命令

子命令的实现很简单。对于`add`子命令，我们提取两个参数，`USER_ID`和`ACTIVITY`，并使用它们来创建一个`Activity`结构体实例。我们还使用`Utc::now`方法获取当前时间，并将其保存到`Activity`的`datetime`字段中。最后，我们调用`add_activity`方法将`Activity`实例添加到 MongoDB 数据库中：

```rs
match matches.subcommand() {
    (CMD_ADD, Some(matches)) => {
        let user_id = matches.value_of("USER_ID").unwrap().to_owned();
        let activity = matches.value_of("ACTIVITY").unwrap().to_owned();
        let activity = Activity {
            user_id,
            activity,
            datetime: Utc::now().to_string(),
        };
        add_activity(&conn, activity)?;
    }
    (CMD_LIST, _) => {
        let list = list_activities(&conn)?;
        for item in list {
            println!("User: {:20}    Activity: {:20}    DateTime: {:20}",
                     item.user_id, item.activity, item.datetime);
        }
    }
    _ => { matches.usage(); }
}
```

列表子命令调用`list_activities`函数，然后遍历所有记录并将它们打印到终端。日志工具已完成 - 我们现在可以测试它了。

# 测试

使用以下命令编译和运行工具：

```rs
cargo run -- add 43fb507d-4cee-431a-a7eb-af31a1eeed02 "Logged In"
cargo run -- add 43fb507d-4cee-431a-a7eb-af31a1eeed02 "Added contact information"
cargo run -- add 43fb507d-4cee-431a-a7eb-af31a1eeed02 "E-mail confirmed"
```

使用以下命令打印添加的记录列表：

```rs
cargo run -- list
```

这将打印以下输出：

```rs
User: 43fb507d-4cee-431a-a7eb-af31a1eeed02   DateTime: 2018-11-30 14:19:26.245957656 UTC    Activity: Logged In
User: 43fb507d-4cee-431a-a7eb-af31a1eeed02   DateTime: 2018-11-30 14:19:42.249548906 UTC   Activity: Added contact information
User: 43fb507d-4cee-431a-a7eb-af31a1eeed02   DateTime: 2018-11-30 14:19:59.035373758 UTC   Activity: E-mail confirmed
```

你也可以使用`mongo`客户端来检查结果：

```rs
mongo admin
> db.activities.find()
{ "_id" : ObjectId("5c0146ee6531339934e7090c"), "user_id" : "43fb507d-4cee-431a-a7eb-af31a1eeed02", "activity" : "Logged In", "datetime" : "2018-11-30 14:19:26.245957656 UTC" }
{ "_id" : ObjectId("5c0146fe653133b8345ed772"), "user_id" : "43fb507d-4cee-431a-a7eb-af31a1eeed02", "activity" : "Added contact information", "datetime" : "2018-11-30 14:19:42.249548906 UTC" }
{ "_id" : ObjectId("5c01470f653133cf34391c1f"), "user_id" : "43fb507d-4cee-431a-a7eb-af31a1eeed02", "activity" : "E-mail confirmed", "datetime" : "2018-11-30 14:19:59.035373758 UTC" }
```

你做到了！它运行得很好！现在，你知道如何使用 Rust 的所有流行数据库。在下一章中，我们将通过**对象关系映射**（**ORM**）来提高这方面的知识，ORM 有助于简化数据库结构声明、交互和迁移。

# DynamoDB

在本章中，我们使用了本地数据库实例。自己维护数据库的缺点是你还必须自己处理可伸缩性。有许多服务提供流行的数据库，这些数据库可以自动伸缩以满足你的需求。但并非每个数据库都可以无限制地增长：传统的 SQL 数据库在表变得很大时通常会经历速度性能问题。对于大型数据集，你应该选择使用设计时就提供可伸缩性的键值数据库（如 NoSQL）。在本节中，我们将探讨使用由 Amazon 创建的`DynamoDB`的用法，以提供作为一个服务的易于伸缩的数据库。

要使用 AWS 服务，你需要 AWS SDK，但 Rust 没有官方的 SDK，因此我们将使用`rusoto`包，它为 Rust 提供了 AWS API。让我们首先将本章中较早创建的工具移植到`DynamoDB`。首先，我们应该在`DynamoDB`实例中创建一个表。

# 为测试设置数据库

由于 AWS 服务是付费的，因此对于开发或测试你的应用程序，最好在本地启动`DynamoDB`数据库的一个实例。Docker Hub 上有`DynamoDB`的镜像。使用以下命令运行实例：

```rs
docker run -it --rm --name test-dynamodb -p 8000:8000 amazon/dynamodb-local
```

此命令创建一个数据库实例，并将容器的`8000`端口转发到相同数字的本地端口。

要与这个数据库实例一起工作，你需要 AWS CLI 工具。可以使用[`docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html`](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)中的说明进行安装。在 Linux 上，我使用以下命令：

```rs
pip install awscli --upgrade --user
```

此命令不需要安装管理权限。在我安装了工具之后，我创建了一个具有程序访问权限的用户，具体细节如下：[`console.aws.amazon.com/iam/home#/users$new?step=details`](https://console.aws.amazon.com/iam/home#/users%24new?step=details)。你可以在[`docs.aws.amazon.com/IAM/latest/UserGuide/getting-started_create-admin-group.html`](https://docs.aws.amazon.com/IAM/latest/UserGuide/getting-started_create-admin-group.html)中了解更多关于创建用户账户以访问 AWS API 的信息。

当你有程序访问权限的用户时，你可以使用`configure`子命令配置 AWS CLI：

```rs
aws configure
AWS Access Key ID [None]: <your-access-key>
AWS Secret Access Key [None]: <your-secret-key>
Default region name [None]: us-east-1
Default output format [None]: json
```

子命令会要求你提供用户凭据、默认区域和所需的输出格式。根据适当的情况填写这些字段。

现在，我们可以使用 AWS CLI 工具创建一个表格。将以下命令输入到控制台：

```rs
aws dynamodb create-table --cli-input-json file://table.json --endpoint-url http://localhost:8000 --region custom
```

此命令从本地数据库中`table.json`文件的一个 JSON 格式的声明创建一个表格，端点为`localhost:8000`。这是我们已经启动的容器的地址。查看此表格声明文件的内容：

```rs
{
    "TableName" : "Locations",
    "KeySchema": [
        {
            "AttributeName": "Uid",
            "KeyType": "HASH"
        },
        {
            "AttributeName": "TimeStamp",
            "KeyType": "RANGE"
        }
    ],
    "AttributeDefinitions": [
        {
            "AttributeName": "Uid",
            "AttributeType": "S"
        },
        {
            "AttributeName": "TimeStamp",
            "AttributeType": "S"
        }
    ],
    "ProvisionedThroughput": {
        "ReadCapacityUnits": 1,
        "WriteCapacityUnits": 1
    }
}
```

此文件包含一个具有两个必需属性的表格声明：

+   `Uid` - 此属性存储用户标识符。此属性将用作分区键。

+   `TimeStamp` - 此属性存储当位置数据被生成时的戳记。此属性将用作排序键以对记录进行排序。

你可以使用以下命令检查数据库实例是否包含此新表格：

```rs
aws dynamodb list-tables --endpoint-url http://localhost:8000 --region custom
```

它打印出数据库实例包含的表格列表，但我们的列表相当短，因为我们只有一个表格：

```rs
{
    "TableNames": [
        "Locations"
    ]
}
```

数据库已准备就绪。现在，我们将创建一个工具，使用 Rust 向此表格添加记录。

# 连接到 DynamoDB

在本节中，我们将创建一个工具，用于向我们的`DynamoDB`数据库中的表格添加记录，并打印出表格中的所有记录。首先，我们需要添加所有必要的 crate。

# 添加依赖项

要与 AWS API 一起工作，我们将使用`rusoto`crate。实际上，它不是一个单独的 crate，而是一组 crate，其中每个 crate 都覆盖 AWS API 的一些功能。基本 crate 是`rusoto_core`，其中包含表示 AWS API 端点地址的`Region`结构体。`Region`通常对其他 crate 是必要的。此外，`rusoto_core`crate 重新导出`rusoto_credential`crate，它包含用于加载和管理 AWS 凭证以访问 API 的类型。

要与`DynamoDB`数据库交互，我们需要添加`rusoto_dynamodb`依赖项。完整的列表如下：

```rs
chrono = "0.4"

clap = "2.32"

failure = "0.1"

rusoto_core = "0.36.0"

rusoto_dynamodb = "0.36.0"
```

我们还添加了`chrono`依赖项来生成时间戳并将它们转换为 ISO-8601 格式的字符串。我们使用`clap`crate 来解析命令行参数，并使用`failure`crate 从`main`函数返回一个通用的`Error`类型。

我们需要在我们的代码中以下类型：

```rs
use chrono::Utc;

use clap::{App, AppSettings, Arg, SubCommand,
    crate_authors, crate_description, crate_name, crate_version};

use failure::{Error, format_err};

use rusoto_core::Region;

use rusoto_dynamodb::{AttributeValue, DynamoDb, DynamoDbClient,
    QueryInput, UpdateItemInput};

use std::collections::HashMap;
```

值得注意的是从`rusoto_core`和`rusoto_dynamodb`crate 导入的类型。我们导入了`Region`结构体，它用于设置 AWS 端点的位置。`DynamoDb`特质和`DynamoDbClient`用于获取对数据库的访问权限。`AttributeValue`是一种用于表示存储在 DynamoDB 表中的值的类型。`QueryInput`是一个结构体，用于准备`query`，而`UpdateItemInput`是一个结构体，用于准备`update_item`请求。

让我们添加与`DynamoDB`数据库交互的函数。

# 交互函数

在本节中，我们将创建一个工具，该工具将位置记录存储到数据库中，并查询特定用户的位置点。为了在代码中表示位置，我们声明以下`Location`结构体：

```rs
#[derive(Debug)]
struct Location {
    user_id: String,
    timestamp: String,
    longitude: String,
    latitude: String,
}
```

这个结构体保存`user_id`，它代表分区键，以及`timestamp`，它代表排序键。

`DynamoDB`是一个键值存储，其中每个记录都有一个唯一的键。当你声明表时，你必须决定哪些属性将作为记录的键。你可以选择最多两个键。第一个是必需的，代表用于在数据库分区之间分配数据的分区键。第二个键是可选的，代表用于在表中排序项的属性。

`rusoto_dynamodb`crate 包含一个`AttributeValue`结构体，它在查询和结果中用于插入或从表中提取数据。由于表中的每个记录（即每个项）都是属性名称到属性值的集合，我们将添加`from_map`方法将属性`HashMap`转换为我们的`Location`类型：

```rs
impl Location {
    fn from_map(map: HashMap<String, AttributeValue>) -> Result<Location, Error> {
        let user_id = map
            .get("Uid")
            .ok_or_else(|| format_err!("No Uid in record"))
            .and_then(attr_to_string)?;
        let timestamp = map
            .get("TimeStamp")
            .ok_or_else(|| format_err!("No TimeStamp in record"))
            .and_then(attr_to_string)?;
        let latitude = map
            .get("Latitude")
            .ok_or_else(|| format_err!("No Latitude in record"))
            .and_then(attr_to_string)?;
        let longitude = map
            .get("Longitude")
            .ok_or_else(|| format_err!("No Longitude in record"))
            .and_then(attr_to_string)?;
        let location = Location { user_id, timestamp, longitude, latitude };
        Ok(location)
    }
}
```

我们需要四个属性：`Uid`、`TimeStamp`、`Longitude`和`Latitude`。我们使用`attr_to_string`方法从映射中提取每个属性并将其转换为`Location`实例：

```rs
fn attr_to_string(attr: &AttributeValue) -> Result<String, Error> {
    if let Some(value) = &attr.s {
        Ok(value.to_owned())
    } else {
        Err(format_err!("no string value"))
    }
}
```

`AttributeValue`结构体包含多个字段，用于不同类型的值：

+   `b` - 一个由`Vec<u8>`表示的二进制值

+   `bool` - 一个具有`bool`类型的布尔值

+   `bs` - 一个二进制集，但表示为`Vec<Vec<u8>>`

+   `l` - 一个`Vec<AttributeValue>`类型的属性列表

+   `m` - 一个`HashMap<String, AttributeValue>`类型的属性映射

+   `n` - 一个以`String`类型存储的数字，以保持精确值而不丢失任何精度

+   `ns` - 一个作为`Vec<String>`的数字集合

+   `null` - 用于表示空值，并以`bool`存储，这意味着值是 null

+   `s` - 一个字符串，类型为`String`

+   `ss` - 一个字符串集合，类型为`Vec<String>`

你可能会注意到没有为时间戳指定数据类型。这是真的，因为`DynamoDB`为大多数数据类型使用字符串。

我们使用`s`字段来处理我们将通过`add_location`函数添加的字符串值：

```rs
fn add_location(conn: &DynamoDbClient, location: Location) -> Result<(), Error> {
    let mut key: HashMap<String, AttributeValue> = HashMap::new();
    key.insert("Uid".into(), s_attr(location.user_id));
    key.insert("TimeStamp".into(), s_attr(location.timestamp));
    let expression = format!("SET Latitude = :y, Longitude = :x");
    let mut values = HashMap::new();
    values.insert(":y".into(), s_attr(location.latitude));
    values.insert(":x".into(), s_attr(location.longitude));
    let update = UpdateItemInput {
        table_name: "Locations".into(),
        key,
        update_expression: Some(expression),
        expression_attribute_values: Some(values),
        ..Default::default()
    };
    conn.update_item(update)
        .sync()
        .map(drop)
        .map_err(Error::from)
}
```

这个函数期望两个参数：数据库客户端的引用，以及要存储的`Location`实例。我们必须手动准备数据，以便将其作为属性映射进行存储，因为`DynamoDbClient`只接受`AttributeValue`类型的值。键中包含的属性被插入到`HashMap`中，值从`Location`实例中提取，并使用具有以下声明的`s_attr`函数转换为`AttributeValue`：

```rs
fn s_attr(s: String) -> AttributeValue {
    AttributeValue {
        s: Some(s),
        ..Default::default()
    }
}
```

在我们填充了`key`映射之后，我们可以用表达式设置其他属性。为了将属性设置到项目上，我们必须在`DynamoDB`语法中指定它们，例如`SET Longitude = :x, Latitude = :y`。这个表达式意味着我们添加了两个名为`Longitude`和`Latitude`的属性。在上面的表达式中，我们使用了占位符`:x`和`:y`，这些将被我们从`HashMap`中传递的实际值所替换。

更多关于表达式的信息可以在这里找到：[`docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.html`](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.html)。

当所有准备好的数据都准备好后，我们填充`UpdateItemInput`结构体，并将`table_name`设置为`"Locations"`，因为`update_item`方法需要这个参数。

`update_item`方法返回`RusotoFuture`，它实现了我们在第五章中探讨的`Future`特质，即*使用 Futures Crate 理解异步操作*。你可以在异步应用程序中使用`rusoto` crate。由于我们在这个例子中没有使用 reactor 或异步操作，我们将调用`RusotoFuture`的`sync`方法，这将阻塞当前线程并等待`Result`。

我们已经实现了一个向表中创建新数据项的方法，现在我们需要一个函数来检索这个表中的数据。以下`list_locations`函数从`Locations`表中获取特定用户的`Location`列表：

```rs
fn list_locations(conn: &DynamoDbClient, user_id: String) -> Result<Vec<Location>, Error> {
    let expression = format!("Uid = :uid");
    let mut values = HashMap::new();
    values.insert(":uid".into(), s_attr(user_id));
    let query = QueryInput {
        table_name: "Locations".into(),
        key_condition_expression: Some(expression),
        expression_attribute_values: Some(values),
        ..Default::default()
    };
    let items = conn.query(query).sync()?
        .items
        .ok_or_else(|| format_err!("No Items"))?;
    let mut locations = Vec::new();
    for item in items {
        let location = Location::from_map(item)?;
        locations.push(location);
    }
    Ok(locations)
}
```

`list_locations`函数期望`DynamoDbClient`实例的引用和一个包含用户 ID 的字符串。如果表中存在请求用户的条目，它们将作为`Vec`类型的条目返回，并转换为`Location`类型。

在这个函数中，我们使用`DynamoDbClient`的`query`方法，它期望一个`QueryInput`结构体作为参数。我们用表的名称、键表达式的条件以及填充该表达式的值来填充它。我们使用一个简单的`Uid = :uid`表达式来查询具有相应`Uid`分区键值的项。我们使用一个`:uid`占位符并创建一个带有`:uid`键和`user_id`值的`HashMap`实例，该值通过`s_attr`函数调用转换为`AttributeValue`。

现在，我们有两个函数来插入和查询数据。我们将使用它们来实现一个与`DynamoDB`交互的命令行工具。让我们从解析工具的参数开始。

# 解析命令行参数

AWS 被划分为区域，每个区域都有自己的端点来连接服务。我们的工具将支持两个参数来设置区域和端点：

```rs
.arg(
   Arg::with_name("region")
   .long("region")
   .value_name("REGION")
   .help("Sets a region")
   .takes_value(true),
   )
.arg(
   Arg::with_name("endpoint")
   .long("endpoint-url")
   .value_name("URL")
   .help("Sets an endpoint url")
   .takes_value(true),
   )
```

我们将这两个都添加到`App`实例中。该工具将支持两个命令：添加新项和打印所有项。第一个子命令是`add`，它期望三个参数：`USER_ID`、`LONGITUDE`和`LATITUDE`：

```rs
.subcommand(SubCommand::with_name(CMD_ADD).about("add geo record to the table")
           .arg(Arg::with_name("USER_ID")
                .help("Sets the id of a user")
                .required(true)
                .index(1))
           .arg(Arg::with_name("LATITUDE")
                .help("Sets a latitudelongitude of location")
                .required(true)
                .index(2))
           .arg(Arg::with_name("LONGITUDE")
                .help("Sets a longitude of location")
                .required(true)
                .index(3)))
```

`list`子命令只需要在参数中提供`USER_ID`：

```rs
.subcommand(SubCommand::with_name(CMD_LIST).about("print all records for the user")
           .arg(Arg::with_name("USER_ID")
                .help("User if to filter records")
                .required(true)
                .index(1)))
```

将所有前面的代码添加到`main`函数中。我们可以使用这些参数来创建一个`Region`实例，我们可以使用它来与`DynamoDB`建立连接：

```rs
let region = matches.value_of("endpoint").map(|endpoint| {
     Region::Custom {
         name: "custom".into(),
         endpoint: endpoint.into(),
     }
 }).ok_or_else(|| format_err!("Region not set"))
 .or_else(|_| {
     matches.value_of("region")
         .unwrap_or("us-east-1")
         .parse()
 })?;
```

代码按照以下逻辑工作：如果用户设置了`--endpoint-url`参数，我们创建一个具有自定义名称的`Region`并提供一个`endpoint`值。如果没有设置`endpoint`，我们尝试将`--region`参数解析为`Region`实例，或者默认使用`us-east-1`值。

AWS 非常重视区域值，如果你在一个区域创建了一个表，你就无法从另一个区域访问那个表。我们为区域使用了一个自定义名称，但对于生产工具来说，最好使用`~/.aws/config`文件或提供自定义这些设置的灵活性。

现在，我们可以使用`Region`值来创建一个`DynamoDbClient`实例：

```rs
let client = DynamoDbClient::new(region);
```

`DynamoDbClient`结构体用于向我们的`DynamoDB`实例发送查询。我们将在命令的实现中使用这个实例。你还记得解析命令行参数的`match`表达式吗？首先为`add`子命令添加这个实现，它将新项放入表中，如下所示：

```rs
(CMD_ADD, Some(matches)) => {
     let user_id = matches.value_of("USER_ID").unwrap().to_owned();
     let timestamp = Utc::now().to_string();
     let latitude = matches.value_of("LATITUDE").unwrap().to_owned();
     let longitude = matches.value_of("LONGITUDE").unwrap().to_owned();
     let location = Location { user_id, timestamp, latitude, longitude };
     add_location(&client, location)?;
 }
```

实现很简单——我们提取所有提供的参数，使用`Utc::now`调用生成时间戳，并将其转换为 ISO-8601 格式的`String`类型。最后，我们填充`Location`实例并调用我们之前声明的`add_location`函数。

你是否曾经想过为什么数据库使用 ISO-8601 格式来表示日期，这些日期看起来像`YEAR-MONTH-DATE HOUR:MINUTE:SECOND`？这是因为以这种格式存储在字符串中的日期如果按字母顺序排序，则是按时间顺序排序的。这非常方便：你可以排序日期，将最早的放在顶部，最新的放在底部。

我们仍然需要实现`list`子命令：

```rs
(CMD_LIST, Some(matches)) => {
     let user_id = matches.value_of("USER_ID").unwrap().to_owned();
     let locations = list_locations(&client, user_id)?;
     for location in locations {
         println!("{:?}", location);
     }
 }
```

此命令提取 `USER_ID` 参数，并使用提供的 `user_id` 值调用 `list_locations` 函数。最后，我们遍历所有位置并将它们打印到终端。

实现已完成，我们现在可以尝试它了。

# 测试

为了测试该工具，使用 Docker 启动 `DynamoDB` 实例并创建一个表，就像我们在本章中之前所做的那样。让我们添加两个用户的四个位置：

```rs
cargo run -- --endpoint-url http://localhost:8000 add 651B4984-1252-4ECE-90E7-0C8B58541E7C 52.73169 41.44326
cargo run -- --endpoint-url http://localhost:8000 add 651B4984-1252-4ECE-90E7-0C8B58541E7C 52.73213 41.44443
cargo run -- --endpoint-url http://localhost:8000 add 651B4984-1252-4ECE-90E7-0C8B58541E7C 52.73124 41.44435
cargo run -- --endpoint-url http://localhost:8000 add 7E3E27D0-D002-43C4-A0DF-415B2F5FF94D 35.652832 139.839478
```

我们还将 `--endpoint-url` 参数设置为指向我们的客户端到本地的 `DynamoDB` 实例。当所有记录都已添加后，我们可以使用 `list` 子命令来打印指定用户的全部值：

```rs
cargo run -- --endpoint-url http://localhost:8000 list 651B4984-1252-4ECE-90E7-0C8B58541E7C
```

此命令将打印出类似以下内容：

```rs
Location { user_id: "651B4984-1252-4ECE-90E7-0C8B58541E7C", timestamp: "2019-01-04 19:58:26.278518362 UTC", latitude: "52.73169", longitude: "41.44326" }
Location { user_id: "651B4984-1252-4ECE-90E7-0C8B58541E7C", timestamp: "2019-01-04 19:58:42.559125438 UTC", latitude: "52.73213", longitude: "41.44443" }
Location { user_id: "651B4984-1252-4ECE-90E7-0C8B58541E7C", timestamp: "2019-01-04 19:58:55.730794942 UTC", latitude: "52.73124", longitude: "41.44435" }
```

如您所见，我们已按顺序检索了所有值，因为我们使用了 `TimeStamp` 属性作为表的排序键。现在，您已经具备了创建使用数据库的微服务的能力，但如果您使用 SQL 数据库，您还可以添加一个额外的抽象层，并以原生 Rust 结构体的形式与数据库记录一起工作，而无需编写粘合代码。在下一章中，我们将通过对象关系映射来检查这种方法。

# 概述

在本章中，我们涵盖了与数据库相关的大量内容。我们首先创建了一个到 PostgreSQL 的普通连接。之后，我们使用 `r2d2` crate 添加了一个连接池，并利用 `rayon` crate 并行执行 SQL 语句。我们创建了一个用于管理我们的 `users` 数据库的工具，并为其 MySQL 数据库重新实现了它。

我们还掌握了一些与 NoSQL 数据库交互的方法，特别是 Redis 和 MongoDB。

我们最后探索的数据库是 DynamoDB，它是亚马逊网络服务的一部分，并且可以非常容易地进行扩展。

对于所有示例，我们都在容器中运行数据库实例，因为这是最简单的方式来测试与数据库的交互。我们尚未在微服务中使用数据库连接，因为这需要一个单独的线程来避免阻塞。我们将在第十章，*微服务中的后台任务和线程池*中学习如何使用异步代码来使用后台任务。

在下一章中，我们将探讨使用数据库的不同方法——使用 `diesel` crate 进行对象关系映射。
