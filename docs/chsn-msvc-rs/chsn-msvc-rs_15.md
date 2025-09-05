# 第十五章：将服务器打包到容器中

使用 Rust 创建的微服务部署起来相当简单：只需为你的服务器构建一个二进制文件，将该二进制文件上传到服务器，然后启动它即可。但这并不是真实应用中的灵活方法。首先，你的微服务可能需要文件、模板和配置。另一方面，你可能希望使用不同操作系统的服务器。在这种情况下，你将不得不为每个系统构建一个二进制文件。为了减少部署中出现的问题，现代微服务被打包到容器中，并使用虚拟化来启动。虚拟化有助于简化一组微服务的部署。此外，它还可以帮助扩展微服务，因为要运行微服务的额外实例，你只需启动另一个容器副本即可。

本章将带你沉浸于使用 Rust 微服务构建 Docker 镜像的过程。我们将探讨以下内容：

+   使用 Docker 编译微服务。

+   在容器中准备必要的 Rust 版本。

+   减少使用 Rust 微服务构建镜像的时间。在我们准备好镜像后，我们将为多个微服务创建镜像。

+   为 Docker Compose 工具创建一个 compose 文件，以引导一组微服务，展示如何运行由多个相互交互的微服务组成的复杂项目。

+   配置一组微服务并添加一个数据库实例，以便这些微服务可以将持久状态存储到数据库中。

# 技术要求

本章需要完整安装 Docker 并使用 Docker Compose 工具。由于我们将使用 Docker 容器构建微服务，因此不需要 Rust 编译器，但如果你想在本地构建和测试任何微服务或在不修改 `docker-compose.yml` 文件的情况下调整配置参数，拥有 nightly Rust 编译器会更好。

要安装 Docker，请遵循此处针对您操作系统的说明：[`docs.docker.com/compose/install/`](https://docs.docker.com/compose/install/)。

要安装 Docker Compose 工具，请查看以下文档：[`docs.docker.com/compose/install/`](https://docs.docker.com/compose/install/)。

你可以在 GitHub 项目的 `Chapter15` 文件夹中找到本章的示例：[`github.com/PacktPublishing/Hands-On-Microservices-with-Rust-2018/`](https://github.com/PacktPublishing/Hands-On-Microservices-with-Rust-2018/)。

# 使用微服务构建 Docker 镜像

在本章的第一部分，我们将构建一个带有必要版本的 Rust 编译器的 Docker 镜像，并构建一个带有编译微服务的镜像。我们将使用来自其他章节的一组微服务来展示如何将使用不同框架创建的微服务连接起来。我们将使用来自 第九章 的 *Simple REST Definition and Request Routing with Frameworks* 和 *emails*、*content* 微服务，以及来自 第十一章 的 *Involving Concurrency with Actors and Actix Crate* 的 *router* 微服务，并且我们将调整它们以使其可配置。此外，我们还将添加一个 `dbsync` 微服务，它将对数据库执行所有必要的迁移，因为我们将会使用两个使用 `diesel` crate 的数据库的微服务，如果两个微服务都尝试为其自己的模式应用迁移，将会发生冲突。那是因为我们将使用单个数据库，但如果你为每个微服务使用单独的数据库（不一定是不同的数据库管理应用程序，但只是数据库文件），你可以为每个数据库使用单独的迁移集。现在是准备带有夜间 Rust 编译器的镜像的时候了。

# 使用 Rust 编译器创建镜像

Docker Hub 上有许多现成的镜像。你还可以在这里找到官方镜像：[`hub.docker.com/_/rust/`](https://hub.docker.com/_/rust/)。但我们将创建自己的镜像，因为官方镜像只包含稳定的编译器版本。如果你觉得足够用，使用官方镜像会更好，但如果你使用像 `diesel` 这样的需要 Rust 编译器夜间版本的 crate，你将不得不构建自己的镜像来构建微服务。

创建一个新的 `Dockerfile` 并向其中添加以下内容：

```rs
FROM buildpack-deps:stretch

ENV RUSTUP_HOME=/usr/local/rustup \
    CARGO_HOME=/usr/local/cargo \
    PATH=/usr/local/cargo/bin:$PATH

RUN set -eux; \
    url="https://static.rust-lang.org/rustup/dist/x86_64-unknown-linux-gnu/rustup-init"; \
    wget "$url"; \
    chmod +x rustup-init; \
    ./rustup-init -y --no-modify-path --default-toolchain nightly; \
    rm rustup-init; \
    chmod -R a+w $RUSTUP_HOME $CARGO_HOME; \
    rustup --version; \
    cargo --version; \
    rustc --version;
```

我从官方的 Rust Docker 镜像中借用了这个 `Dockerfile`，位于此处：[`github.com/rust-lang-nursery/docker-rust-nightly/blob/master/nightly/Dockerfile`](https://github.com/rust-lang-nursery/docker-rust-nightly/blob/master/nightly/Dockerfile)。这个文件是使用 Rust 编译器创建镜像时良好实践的起点。

我们的 Rust 镜像是基于 `buildpack-deps` 镜像的，它包含了开发者常用到的所有必要依赖。这个依赖在第一行通过 `FROM` 命令指示。

`buildpack-deps` 是基于 Ubuntu（基于 Debian 的免费开源 Linux 发行版）的官方 Docker 镜像。该镜像包含 OpenSSL 和 curl 等库的大量头文件，以及包含所有必要证书的软件包等。它作为 Docker 镜像的构建环境非常有用。

下一个包含 `ENV` 命令的行，在镜像中设置了三个环境变量：

+   `RUSTUP_HOME`：设置 `rustup` 工具的根文件夹，其中包含配置并安装工具链

+   `CARGO_HOME`：包含 `cargo` 工具使用的缓存文件

+   `PATH`：包含可执行二进制文件路径的系统环境变量

我们通过设置这些环境变量将所有实用程序目标文件夹设置为`/usr/local`。

我们在这里使用`rustup`实用程序来初始化 Rust 环境。它是一个官方的 Rust 安装工具，可以帮助您维护和保持多个 Rust 安装的更新。在我看来，使用`rustup`是本地或容器中安装 Rust 的最佳方式。

最后一个`Dockerfile`命令，`RUN`，很复杂，我们将逐行分析这组命令。第一个 shell 命令如下：

```rs
set -eux
```

由于 Ubuntu 的默认 shell 是 Bash shell，我们可以设置三个有用的标志：

+   `-e`：此标志告诉 shell 仅在上一条命令成功完成后才运行下一条（命令）

+   `-u`：使用此标志，如果命令尝试展开未设置的变量，shell 将打印错误到`stderr`

+   `-x`：使用此标志，shell 将在运行之前将每个命令打印到`stderr`

接下来的三行下载`rustup-init`二进制文件，并将可执行标志设置为下载的文件：

```rs
url="https://static.rust-lang.org/rustup/dist/x86_64-unknown-linux-gnu/rustup-init"; \
wget "$url"; \
chmod +x rustup-init; \
```

下一个对运行`rustup-init`命令带有参数，并在运行后删除二进制文件的配对：

```rs
./rustup-init -y --no-modify-path --default-toolchain nightly; \
rm rustup-init; \
```

以下标志被使用：

+   `-y`：抑制任何确认提示

+   `--no-modify-path`：不会修改`PATH`环境变量（我们之前手动设置过，用于镜像）

+   `--default-toolchain`：默认工具链的类型（我们将使用`nightly`）

剩余的行设置`RUSTUP_HOME`和`CARGO_HOME`文件夹的写权限，并打印所有已安装工具的版本：

```rs
chmod -R a+w $RUSTUP_HOME $CARGO_HOME; \
rustup --version; \
cargo --version; \
rustc --version;
```

现在，您可以构建`Dockerfile`以获取包含预配置 Rust 编译器的镜像：

```rs
docker build -t rust:nightly  .
```

此命令需要一些时间才能完成，但完成后，您将拥有一个可以用于构建微服务镜像的镜像。如果您输入`docker images`命令，您将看到如下内容：

```rs
REPOSITORY   TAG       IMAGE ID       CREATED             SIZE
rust         nightly   91e52fb2cea5   About an hour ago   1.67GB
```

现在，我们将使用标记为`rust:nightly`的镜像，并从中创建微服务的镜像。让我们先为用户微服务创建一个镜像。

# 用户微服务镜像

用户微服务为用户提供注册功能。本章包含来自第九章的修改版用户微服务，*使用框架进行简单的 REST 定义和请求路由*。由于此服务需要数据库并使用`diesel`包与之交互，我们需要在构建镜像的过程中使用`diesel.toml`配置文件。

# .dockerignore

由于 Docker 会复制构建文件夹中的所有文件，我们必须添加包含要避免复制的路径模式的`.dockerignore`文件。例如，跳过`target`构建文件夹是有用的，因为它可能包含大型项目数 GB 的数据，但无论如何，我们不需要所有这些，因为我们将会使用带有 Rust 编译器的镜像来构建微服务。添加`.dockerignore`文件：

```rs
target
 Cargo.lock
 **/*.rs.bk
 files
 *.db
```

在下一章中，我们将忽略所有 Rust 的构建工件（例如`target`、`Cargo.lock`和`rustfmt`工具产生的`*.bk`文件等），我们将探索持续集成工具。我们还包含了两个模式：`files`——如果尝试在本地运行，这个文件夹将由微服务创建来存储文件，`*.db`——对于 SQLite 数据库来说，这不是一个必要的模式，因为此版本使用的是 PostgreSQL 而不是 SQLite，但如果以后出于测试原因想要支持两种数据库，则很有用。

# Dockerfile

现在一切准备就绪，可以构建并将微服务打包到镜像中。为此，将`Dockerfile`文件添加到包含微服务的文件夹中，并在其中添加以下行：

```rs
FROM rust:nightly

RUN USER=root cargo new --bin users-microservice
WORKDIR /users-microservice
COPY ./Cargo.toml ./Cargo.toml
RUN cargo build

RUN rm src/*.rs
COPY ./src ./src
COPY ./diesel.toml ./diesel.toml
RUN rm ./target/debug/deps/users_microservice*
RUN cargo build

CMD ["./target/debug/users-microservice"]

EXPOSE 8000
```

我们基于本章早期创建的`rust:nightly`镜像创建了该镜像。我们使用`FROM`命令设置了它。下一行创建了一个新的 crate：

```rs
RUN USER=root cargo new --bin users-microservice
```

你可能会问我们为什么这样做而没有使用现有的 crate。那是因为我们将在容器内部重现 crate 的创建过程，首先构建依赖项，以避免在添加任何微服务源代码的微小更改时重建它们的漫长过程。这种方法将为你节省大量时间。将`Cargo.toml`复制到镜像中，并构建所有依赖项，而不包括微服务的源代码（因为我们还没有复制它们）：

```rs
WORKDIR /users-microservice
COPY ./Cargo.toml ./Cargo.toml
RUN cargo build
```

下一个命令集将源代码和`diesel.toml`文件添加到镜像中，删除之前的构建结果，并使用新源代码重新构建 crate：

```rs
RUN rm src/*.rs
COPY ./src ./src
COPY ./diesel.toml ./diesel.toml
RUN rm ./target/debug/deps/users_microservice*
RUN cargo build
```

在这个时刻，这个镜像包含了一个微服务的二进制文件，我们可以将其用作启动容器的起始命令：

```rs
CMD ["./target/debug/users-microservice"]
```

默认情况下，容器不会打开端口，你不能通过另一个容器连接到它，也不能将容器的端口转发到本地端口。由于我们的微服务从端口 8000 开始，我们必须使用以下命令来暴露它：

```rs
EXPOSE 8000
```

镜像已准备好构建和运行容器。让我们开始吧。

# 构建镜像

我们已经准备好了 Dockerfile 来构建一个镜像，该镜像首先构建我们微服务的所有依赖项，然后构建所有源代码。要启动这个过程，你必须使用 Docker 的`build`命令：

```rs
docker build -t users-microservice:latest 
```

当你运行这个命令时，你会看到 Docker 如何准备文件来构建镜像，并构建所有依赖项，但只针对没有微服务源代码的空 crate：

```rs
Sending build context to Docker daemon  13.82kB
Step 1/12 : FROM rust:nightly
 ---> 91e52fb2cea5
Step 2/12 : RUN USER=root cargo new --bin users-microservice
 ---> Running in 3ff6b18a9c72
     Created binary (application) `users-microservice` package
Removing intermediate container 3ff6b18a9c72
 ---> 85f700c4a567
Step 3/12 : WORKDIR /users-microservice
 ---> Running in eff894de0a40
Removing intermediate container eff894de0a40
 ---> 66366486b1e2
Step 4/12 : COPY ./Cargo.toml ./Cargo.toml
 ---> 8864ae055d16
Step 5/12 : RUN cargo build
 ---> Running in 1f1150ae4661
    Updating crates.io index
 Downloading crates ...
 Compiling crates ...
   Compiling users-microservice v0.1.0 (/users-microservice)
    Finished dev [unoptimized + debuginfo] target(s) in 2m 37s
Removing intermediate container 1f1150ae4661
 ---> 7868ea6bf9b3
```

我们的镜像总共需要 12 个步骤来构建微服务。正如你所见，依赖项的构建需要两分半钟。这并不快。但我们不需要重复这个步骤，直到`Cargo.toml`发生变化。接下来的步骤将微服务的源代码复制到容器中，并使用预构建的依赖项构建它们：

```rs
Step 6/12 : RUN rm src/*.rs
 ---> Running in 5b7d9a1f96cf
Removing intermediate container 5b7d9a1f96cf
 ---> b03e7d0b23cc
Step 7/12 : COPY ./src ./src
 ---> 2212e3db5223
Step 8/12 : COPY ./diesel.toml ./diesel.toml
 ---> 5d4c59d31614
Step 9/12 : RUN rm ./target/debug/deps/users_microservice*
 ---> Running in 6bc9df93ebc1
Removing intermediate container 6bc9df93ebc1
 ---> c2e3d67d3bf8
Step 10/12 : RUN cargo build
 ---> Running in b985b6c793d1
   Compiling users-microservice v0.1.0 (/users-microservice)
    Finished dev [unoptimized + debuginfo] target(s) in 4.98s
Removing intermediate container b985b6c793d1
 ---> 553156f97943
Step 11/12 : CMD ["./target/debug/users-microservice"]
 ---> Running in c36ff8e44db3
Removing intermediate container c36ff8e44db3
 ---> 56e7eb1144aa
Step 12/12 : EXPOSE 8000
 ---> Running in 5e76a47a0ded
Removing intermediate container 5e76a47a0ded
 ---> 4b6fc8aa6f1b
Successfully built 4b6fc8aa6f1b
Successfully tagged users-microservice:latest
```

如输出所示，构建微服务只需 5 秒钟。这足够快，你可以根据需要重新构建它多次。由于镜像已经构建，我们可以启动一个包含微服务的容器。

# 启动容器

我们构建的镜像已经存储在 Docker 中，我们可以使用`docker images`命令查看它：

```rs
REPOSITORY           TAG       IMAGE ID       CREATED         SIZE
users-microservice   latest    4b6fc8aa6f1b   7 minutes ago   2.3GB
rust                 nightly   91e52fb2cea5   3 hours ago     1.67GB
```

要从镜像中启动微服务，请使用以下命令：

```rs
docker run -it --rm -p 8080:8000 users-microservice
```

带有微服务实例的容器将启动，但它不会工作，因为我们还没有运行一个带有数据库实例的容器。我们不会手动连接容器，因为这属于 Docker 使用的细微之处，你可以在 Docker 的文档中了解更多信息；然而，我们将在本章后面的*组合微服务集*部分学习如何使用 Docker Compose 工具连接容器。

你可能也会问：为什么我们的微服务这么庞大？我们将在本章后面尝试减小它。但现在我们应该将其他微服务打包到镜像中。

# 内容微服务镜像

我们将要使用的第二个微服务是我们在第九章，*使用框架进行简单的 REST 定义和请求路由*中创建的内容微服务。我们还为使用 PostgreSQL 数据库准备了这项服务。我们从上一个示例中借用了`dockerignore`文件，并为此微服务修改了`Dockerfile`文件。请看以下代码：

```rs
FROM rust:nightly

RUN USER=root cargo new --bin content-microservice
WORKDIR /content-microservice
COPY ./Cargo.toml ./Cargo.toml
RUN cargo build

RUN rm src/*.rs
COPY ./src ./src
RUN rm ./target/debug/deps/content_microservice*
RUN cargo build

CMD ["./target/debug/content-microservice"]
EXPOSE 8000
```

如你所见，这个`Dockerfile`与上一个镜像的`Dockerfile`相同，但它有一个区别：它没有复制任何配置文件。我们将使用 Rocket 框架，但我们将使用 Docker Compose 文件中的环境变量设置所有参数。

你可以使用以下命令构建此镜像以检查其工作情况：

```rs
 docker build -t content-microservice:latest .
```

但构建这个镜像并不是必要的，因为我们不会手动启动容器——我们将使用 Docker Compose。让我们也将一个邮件微服务打包到镜像中。

# 邮件微服务镜像

邮件微服务不使用`diesel`crate，我们可以使用官方的 Rust 镜像来构建微服务。此外，邮件微服务有模板，用于准备电子邮件的内容。我们将使用相同的`.dockerignore`文件，但会从上一个示例中复制`Dockerfile`并添加一些与邮件微服务相关的更改：

```rs
FROM rust:1.30.1

RUN USER=root cargo new --bin mails-microservice
WORKDIR /mails-microservice
COPY ./Cargo.toml ./Cargo.toml
RUN cargo build

RUN rm src/*.rs
COPY ./src ./src
COPY ./templates ./templates
RUN rm ./target/debug/deps/mails_microservice*
RUN cargo build

CMD ["./target/debug/mails-microservice"]
```

我们是从`rust:1.30.1`镜像创建了这个镜像。编译器的稳定版本适合编译这个简单的微服务。我们还添加了一个命令，将所有模板复制到镜像中：

```rs
COPY ./templates ./templates
```

现在，我们可以准备带有路由微服务的镜像。

# 路由微服务镜像

如果你记得，我们在第十一章，*使用 Actors 和 Actix Crate 涉及并发*中创建了路由微服务，我们探索了 Actix 框架的功能。我们将路由微服务修改为与其他微服务一起工作——我们添加了一个`Config`和一个`State`，它们与处理器共享配置值。此外，改进后的路由微服务还服务于静态文件夹中的资源。我们还需要将这个文件夹复制到镜像中。请看路由微服务的`Dockerfile`：

```rs
FROM rust:1.30.1

RUN USER=root cargo new --bin router-microservice
WORKDIR /router-microservice
COPY ./Cargo.toml ./Cargo.toml
RUN cargo build

RUN rm src/*.rs
COPY ./src ./src
COPY ./static ./static
RUN rm ./target/debug/deps/router_microservice*
RUN cargo build

CMD ["./target/debug/router-microservice"]

EXPOSE 8000
```

我们还使用了官方的 Rust 镜像和稳定的编译器。与之前的例子相比，你将注意到的唯一区别是将`static`文件夹复制到镜像中。我们使用了与之前例子相同的`.dockerignore`文件。

我们已经为所有微服务构建了镜像，但我们还需要添加一个将迁移应用到数据库的工作者。我们稍后将会使用 Docker Compose 来自动应用所有迁移。让我们创建这个 Docker 镜像。

# DBSync 工作者镜像

DBSync 工作者的唯一功能是等待与数据库的连接并应用所有迁移。我们也将这个工作者打包到 Docker 镜像中，以便在下一节中我们将创建的 compose 文件中使用。

# 依赖项

工作者需要以下依赖项：

```rs
clap = "2.32"
config = "0.9"
diesel = { version = "¹.1.0", features = ["postgres", "r2d2"] }
diesel_migrations = "1.3"
env_logger = "0.6"
failure = "0.1"
log = "0.4"
postgres = "0.15"
r2d2 = "0.8"
serde = "1.0"
serde_derive = "1.0"
```

我们需要`diesel` crate 的`diesel_migrations`来将所有迁移嵌入到代码中。这不是必需的，但很有用。我们需要`config`和`serde` crate 来配置工作者。其他 crate 更常见，你可以在之前的章节中看到我们如何使用它们。

将这些依赖项添加到`Cargo.toml`并导入在`main`函数中将使用的类型：

```rs
use diesel::prelude::*;
use diesel::connection::Connection;
use failure::{format_err, Error};
use log::debug;
use serde_derive::Deserialize;
```

现在让我们创建一段代码，它将等待数据库连接并应用所有嵌入的迁移。

# 主函数

在创建主函数之前，我们必须使用`embed_migrations!`宏调用嵌入迁移：

```rs
embed_migrations!();
```

这个调用创建了一个`embedded_migrations`模块，其中包含一个`run`函数，该函数将所有迁移应用到数据库。但在我们使用它之前，让我们添加`Config`结构体来从配置文件或环境变量中读取数据库连接链接，使用`config` crate：

```rs
#[derive(Deserialize)]
struct Config {
    database: Option<String>,
}
```

这个结构体只包含一个参数——一个可选的`String`类型的数据库连接链接。我们稍后将会使用 Docker Compose 设置这个参数，通过`DBSYNC_DATABASE`环境变量。我们在`main`函数中添加了`DBSYNC`前缀。查看`main`函数的完整代码：

```rs
fn main() -> Result<(), Error> {
    env_logger::init();
    let mut config = config::Config::default();
    config.merge(config::Environment::with_prefix("DBSYNC"))?;
    let config: Config = config.try_into()?;
    let db_address = config.database.unwrap_or("postgres://localhost/".into());
    debug!("Waiting for database...");
    loop {
        let conn: Result<PgConnection, _> = Connection::establish(&db_address);
        if let Ok(conn) = conn {
            debug!("Database connected");
            embedded_migrations::run(&conn)?;
            break;
        }
    }
    debug!("Database migrated");
    Ok(())
}
```

在前面的代码中，我们初始化了`env_logger`以将信息打印到 stderr。之后，我们从一个`config`模块创建了一个通用的`Config`实例，并使用`DBSYNC`前缀合并环境变量。如果配置合并成功，我们尝试将其转换为之前声明的我们自己的`Config`类型的值。我们将使用配置来提取数据库连接的链接。如果没有提供值，我们将使用`postgres://localhost/`链接。

当连接链接就绪时，我们使用循环尝试连接到数据库。我们将尝试连接，直到成功为止，因为我们将会使用这个工作者与 Docker Compose 一起使用，尽管我们将启动一个包含数据库的容器，但在数据库实例启动时，它可能不可用。我们使用循环等待连接就绪。

当连接就绪时，我们使用它通过`embedded_migrations`模块的`run`方法应用内嵌迁移。在迁移应用后，我们打破循环并停止工作进程。

我们已经准备好了所有微服务以供启动，但它们的缺点是它们的源代码也保留在镜像中。如果我们想隐藏微服务的实现细节，这就不太好了。让我们探索一种使用镜像构建缓存隐藏微服务源的技术。

# 隐藏微服务源代码

在镜像内构建微服务的主要缺点是，所有源代码和构建工件都将对任何有权访问 Docker 镜像的人可用。如果你想删除源代码和其他构建工件，你可以使用两种方法之一。

1.  第一种方法是通过带有 Rust 编译器的 Docker 镜像构建所有源代码，通过链接虚拟卷提供对源代码的访问。在 Docker 中，你可以使用`docker run`命令的`-v`参数将任何本地文件夹映射到容器内的一个卷。这种方法的不利之处在于，Docker 在容器内使用另一个 ID，这与你在本地会话中的 ID 不同。它可能会创建你无法删除的文件，除非更改用户 ID。此外，这种方法更难维护。但如果只需要编译结果，它是有用的。如果你计划在容器内运行微服务，最好在镜像内构建一切。

1.  第二种方法涉及使用 Docker 构建一切，但使用构建缓存来获取编译结果并将其放入新创建的容器中。让我们探索实现此方法的`Dockerfile`：

```rs
FROM rust:nightly as builder

RUN USER=root cargo new --bin dbsync-worker
WORKDIR /dbsync-worker
COPY ./Cargo.toml ./Cargo.toml
RUN cargo build

RUN rm src/*.rs
COPY ./src ./src
COPY ./migrations ./migrations
COPY ./diesel.toml ./diesel.toml
RUN rm ./target/debug/deps/dbsync_worker*
RUN cargo build

FROM buildpack-deps:stretch

COPY --from=builder /dbsync-worker/target/debug/dbsync-worker  /app/
ENV RUST_LOG=debug
EXPOSE 8000
ENTRYPOINT ["/app/dbsync-worker"]
```

我们使用了 dbsync 微服务的`Dockerfile`，文件的第一部分与原始文件相同，但有一个小的改进——我们将该名称设置为我们在第一行构建的镜像：

```rs
FROM rust:nightly as builder
```

现在，我们可以使用`builder`名称来使用镜像的缓存数据。

在本节之后，我们从一个原本用于构建前面的`rust:nightly`镜像的`buildpack-deps`镜像开始创建一个新的空镜像。我们使用带有`--from`参数的`COPY`命令从构建镜像中复制一个可执行二进制文件，其中我们设置了镜像的名称：

```rs
COPY --from=builder /dbsync-worker/target/debug/dbsync-worker  /app/
```

此命令将二进制文件复制到镜像内部的`/app`文件夹中，我们可以将其用作容器的入口点：

```rs
ENTRYPOINT ["/app/dbsync-worker"]
```

我们还设置了`RUST_LOG`环境变量并公开了端口。通过传递此`Dockerfile`的名称并使用 Docker 构建命令的`-f`参数来构建此镜像，你将得到一个包含微服务单个二进制文件的镜像。换句话说，这种方法允许我们构建微服务并重新使用编译的二进制文件来创建新镜像。你现在已经知道足够的信息来将微服务打包到镜像中，现在我们可以探索 Docker Compose 启动一组微服务并将所有启动的容器相互连接的能力。

# 组合微服务集

Docker Compose 是一个部署和运行一组可以相互连接的微服务的出色工具。它帮助你在可读的 YAML 文件中定义具有配置参数的多容器应用程序。你不仅限于本地部署，还可以在 Docker 守护进程也在运行的远程服务器上部署它。

在本章的这一节中，我们将所有包含数据库的微服务打包成一个单一的应用程序。你将学习如何为 Rust 框架和日志记录器设置变量，如何连接微服务，如何定义启动容器的顺序，如何读取运行中的应用程序的日志，以及如何为测试和生产使用不同的配置。

# 应用程序定义

Docker Compose 是一个与应用程序的 YAML 定义一起工作的工具。一个 YAML 文件可以包含容器、网络和卷的声明。我们将使用版本`3.6`。创建一个`docker-compose.test.yml`文件并添加以下部分：

```rs
version: "3.6"
services:
    # the place for containers definition
```

在`services`部分，我们将添加所有我们的微服务。让我们看看每个容器配置。

# 数据库容器

我们的应用程序需要一个数据库实例。用户和内容微服务都使用 PostgreSQL 数据库，dbsync 工作器在必要时应用所有迁移。查看以下设置：

```rs
db:
  image: postgres:latest
  restart: always
  environment:
    - POSTGRES_USER=postgres
    - POSTGRES_PASSWORD=password
  ports:
    - 5432:5432
```

我们使用官方的 PostgreSQL 镜像。如果数据库失败，它也需要重启。我们将`restart`策略设置为`always`，这意味着如果容器失败，它将被重启。我们还使用环境变量设置了用户名和密码。

由于我们创建了用于测试目的的 compose 文件，我们将容器的一个端口转发到外部，以便使用本地客户端连接到数据库。

# 一个带有电子邮件服务器的容器

我们需要 SMTP 服务器来为我们的邮件服务。我们使用带有 Postfix 邮件服务器的`juanluisbaptiste/postfix`镜像。如果服务器失败，它也需要重启，我们将`restart`策略设置为`always`。查看以下代码：

```rs
smtp:
  image: juanluisbaptiste/postfix
  restart: always
  environment:
    - SMTP_SERVER=smtp.example.com
    - SMTP_USERNAME=admin@example.com
    - SMTP_PASSWORD=password
    - SERVER_HOSTNAME=smtp.example.com
  ports:
    - "2525:25"
```

我们还使用环境变量配置服务器，并设置了服务器名称、用户名、密码和主机名。为了测试邮件服务器，我们将邮件服务器的端口`25`转发到本地的`2525`端口。

# DBSync 工作容器

现在我们可以添加应用到数据库实例的 dbsync 工作器。我们使用一个本地镜像，该镜像将使用`./microservices/dbsync`文件夹中的`Dockerfile`构建，我们将其用作`build`参数的值。这个工作器依赖于一个数据库容器（称为`db`），我们使用`depends_on`参数设置这个依赖关系。

依赖关系并不意味着当必要的应用程序准备好工作时，依赖的容器将被启动。它仅指容器启动的顺序；你的微服务需要的应用程序可能尚未准备好。你必须控制应用程序的可用性，就像我们对 dbsync 所做的那样，通过一个尝试连接到数据库直到其可用的循环。

此外，我们还设置了`RUST_LOG`变量，以过滤比`debug`级别低一级的消息，并且只打印与`dbsync_worker`模块相关的消息：

```rs
dbsync:
  build: ./microservices/dbsync
  depends_on:
    - db
  environment:
    - RUST_LOG=dbsync_worker=debug
    - RUST_BACKTRACE=1
    - DBSYNC_DATABASE=postgresql://postgres:password@db:5432
```

我们还通过设置`RUST_BACKTRACE`变量激活了回溯打印功能。

最后一个变量设置了一个连接到数据库的连接链接。正如你所见，我们使用主机的`db`名称，因为 Docker 配置容器以解析名称并匹配其他容器的名称，所以你不需要设置或记住容器的 IP 地址。你可以使用容器的名称作为主机名。

# 邮件微服务容器

向用户发送邮件的微服务是基于存储在`./microservices/mails`文件夹中的`Dockerfile`镜像构建的。此微服务依赖于`smtp`容器，但此微服务不会检查邮件服务是否已准备好工作。如果你想要检查邮件服务器是否已准备好，请在开始任何活动之前添加一段尝试连接到 SMTP 服务器的代码。查看以下设置：

```rs
mails:
  build: ./microservices/mails
  depends_on:
    - smtp
  environment:
    - RUST_LOG=mails_microservice=debug
    - RUST_BACKTRACE=1
    - MAILS_ADDRESS=0.0.0.0:8000
    - MAILS_SMTP_ADDRESS=smtp:2525
    - MAILS_SMTP_LOGIN=admin@example.com
    - MAILS_SMTP_PASSWORD=password
  ports:
    - 8002:8000
```

我们还通过环境变量配置了一个微服务，并将端口`8002`映射到容器的`8000`端口。你可以使用端口`8002`来检查微服务是否启动并正常工作。

# 用户微服务容器

用户微服务是由我们之前创建的`Dockerfile`构建的。此微服务依赖于两个其他容器——dbsync 和 mails。首先，我们需要在数据库中有一个用户表来保存用户记录；其次，我们需要有能力向用户发送电子邮件通知。我们还设置了`USERS_ADDRESS`变量中的套接字地址和`USERS_DATABASE`变量中的连接链接：

```rs
users:
  build: ./microservices/users
  environment:
    - RUST_LOG=users_microservice=debug
    - RUST_BACKTRACE=1
    - USERS_ADDRESS=0.0.0.0:8000
    - USERS_DATABASE=postgresql://postgres:password@db:5432
  depends_on:
    - dbsync
    - mails
  ports:
    - 8001:8000
```

此外，还有一个设置，将容器的端口`8000`映射到本地端口`8001`，你可以使用这个端口来测试访问微服务。

# 内容微服务容器

内容微服务是用`./microservices/content`文件夹中的`Dockerfile`文件构建的。我们也在本章的早期创建了此文件。由于内容微服务基于 Rocket 框架，我们可以使用带有`ROCKET`前缀的环境变量来配置微服务：

```rs
content:
  build: ./microservices/content
  depends_on:
    - dbsync
  ports:
    - 8888:8000
  environment:
    - RUST_LOG=content_microservice=debug
    - RUST_BACKTRACE=1
    - ROCKET_ADDRESS=0.0.0.0
    - ROCKET_PORT=8000
    - ROCKET_DATABASES={postgres_database={url="postgresql://postgres:password@db:5432"}}
  ports:
    - 8003:8000
```

此微服务使用数据库，并依赖于`dbsync`容器，而`dbsync`容器又依赖于包含数据库实例的`db`容器。我们打开端口`8003`以在 Docker 外部访问此微服务。

# 路由微服务容器

在我们启动整个应用程序之前，我们将配置最后一个服务，即路由微服务。此服务依赖于用户和内容微服务，因为路由代理请求这些微服务：

```rs
router:
  build: ./microservices/router
  depends_on:
    - users
    - content
  environment:
    - RUST_LOG=router_microservice=debug
    - RUST_BACKTRACE=1
    - ROUTER_ADDRESS=0.0.0.0:8000
    - ROUTER_USERS=http://users:8000
    - ROUTER_CONTENT=http://content:8000
  ports:
    - 8000:8000
```

我们还配置了`router_microservice`命名空间的`debug`级别日志，开启了回溯打印，设置了绑定此微服务的套接字地址，并使用配置支持的变量设置了用户和内容微服务的路径。我们使用容器名称作为主机名，因为 Docker Compose 配置容器通过名称相互访问。我们还转发端口`8000`到相同的系统端口。现在我们可以启动包含所有容器的应用程序。

# 运行应用程序

要运行应用程序，我们将使用 Docker Compose 工具，该工具必须已安装（你可以在本章的技术要求部分找到有用的链接）。如果实用工具安装成功，你将拥有`docker-compose`命令。将目录更改为名为`docker-compose.test.yml`的目录，并运行`up`子命令：

```rs
docker-compose -f docker-compose.test.yml up
```

因此，如果需要，它将构建所有镜像并启动应用程序：

```rs
Creating network "deploy_default" with the default driver
Creating deploy_smtp_1 ... done
Creating deploy_db_1    ... done
Creating deploy_mails_1  ... done
Creating deploy_dbsync_1 ... done
Creating deploy_users_1   ... done
Creating deploy_content_1 ... done
Creating deploy_router_1  ... done
Attaching to deploy_smtp_1, deploy_db_1, deploy_mails_1, deploy_dbsync_1, deploy_content_1, deploy_users_1, deploy_router_1
```

当所有容器启动后，你将在终端中看到所有容器的日志，日志前缀为容器名称：

```rs
smtp_1     | Setting configuration option smtp_sasl_password_maps with value: hash:\/etc\/postfix\/sasl_passwd
mails_1    | [2018-12-24T19:08:20Z DEBUG mails_microservice] Waiting for SMTP server
smtp_1     | Setting configuration option smtp_sasl_security_options with value: noanonymous
dbsync_1   | [2018-12-24T19:08:20Z DEBUG dbsync_worker] Waiting for database...
db_1       | 
db_1       | fixing permissions on existing directory /var/lib/postgresql/data ... ok
mails_1    | [2018-12-24T19:08:20Z DEBUG mails_microservice] SMTP connected
smtp_1     | Adding SASL authentication configuration
mails_1    | Listening on http://0.0.0.0:8000
mails_1    | Ctrl-C to shutdown server
content_1  | Configured for development.
router_1   | DEBUG 2018-12-24T19:08:22Z: router_microservice: Started http server: 0.0.0.0:8000
content_1  | Rocket has launched from http://0.0.0.0:8000
users_1    | [2018-12-24T19:08:24Z DEBUG users_microservice] Starting microservice...
```

现在应用程序已启动，你可以使用此链接通过浏览器连接到它：`http://localhost:8000`。

要停止应用程序，使用*Ctrl+C*键组合。这将启动终止过程，你将在终端中看到它：

```rs
Gracefully stopping... (press Ctrl+C again to force)
Stopping deploy_router_1  ... done
Stopping deploy_users_1   ... done
Stopping deploy_content_1 ... done
Stopping deploy_mails_1   ... done
Stopping deploy_db_1      ... done
Stopping deploy_smtp_1    ... do
```

如果你重新启动应用程序，数据库将是空的。为什么？因为我们把数据库存储在容器的临时文件系统中。如果你需要持久性，你可以将本地文件夹附加到容器作为虚拟卷。让我们探索这个功能。

# 向应用程序添加持久状态

我们创建了一个由微服务组成的应用程序，并且没有持久状态——每次重启时应用程序都是空的。解决这个问题很简单：将持久卷映射到容器的文件夹。由于我们应用程序的没有任何微服务将数据保存在文件中，但 PostgreSQL 数据库是，我们只需要将一个文件夹附加到数据库容器。将`docker-compose.test.yml`复制到`docker-compose.prod.yml`，并添加以下更改：

```rs
services:
  db:
    image: postgres:latest
    restart: always
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    volumes:
      - database_data:/var/lib/postgresql/data
  # other containers definition
volumes:
  database_data:
    driver: local
```

我们将名为`database_data`的卷附加到数据库容器的`/var/lib/postgresql/data`路径。PostgreSQL 默认使用此路径来存储数据库文件。要声明持久卷，我们使用名为卷的`volume`部分。我们将`driver`参数设置为`local`以将数据保留在本地硬盘上。现在数据在重启之间保存。

我们还移除了所有微服务的端口转发，除了路由微服务，因为所有微服务都可通过 Docker 的内部虚拟网络访问，只有路由器需要在外部容器中可用。

# 在后台运行应用程序

我们通过将终端附加到容器的输出启动了应用程序，但如果您想将应用程序部署到远程服务器，这就不方便了。要分离终端，请在启动时使用`-d`参数：

```rs
docker-compose -f docker-compose.prod.yml up -d
```

这将启动具有持久状态的应用程序，并打印如下内容：

```rs
Starting deploy_db_1   ... done
Starting deploy_smtp_1   ... done
Starting deploy_dbsync_1 ... done
Starting deploy_mails_1   ... done
Starting deploy_content_1 ... done
Starting deploy_users_1   ... done
Starting deploy_router_1  ... done
```

它也会从终端分离。您可能会问：我如何使用`env_logger`和`log`crate 读取微服务打印的日志？请使用以下命令，并在末尾加上服务名称：

```rs
docker-compose -f docker-compose.prod.yml logs users
```

此命令将打印`users_1`容器的日志，它代表应用程序的用户服务。您可以使用`grep`命令过滤日志中的不必要记录。

由于应用程序已从终端分离，您应该使用向下命令来停止应用程序：

```rs
docker-compose -f docker-compose.test.yml stop
```

这将停止所有容器并输出以下内容：

```rs
Stopping deploy_router_1  ... done
Stopping deploy_users_1   ... done
Stopping deploy_content_1 ... done
Stopping deploy_mails_1   ... done
Stopping deploy_db_1      ... done
Stopping deploy_smtp_1    ... done
```

应用程序已停止，现在您知道如何使用 Docker Compose 工具运行多容器应用程序。如果您想了解更多关于在本地和远程机器上使用 Docker Compose 的信息，请阅读这本书。

# 摘要

本章向您介绍了如何使用 Docker 构建图像并运行自己的微服务容器。我们将我们在第九章，*使用框架进行简单的 REST 定义和请求路由*，和第十一章，*使用 Actors 和 Actix Crate 处理并发*中创建的所有微服务打包在一起，并学习了如何手动构建图像和启动容器。我们还添加了 dbsync 工作进程，它应用了所有必要的迁移，并为用户和内容微服务准备了数据库。

此外，我们还考虑了隐藏微服务源代码的方法，并使用容器的缓存将编译的二进制文件复制到空镜像中，而不构建工件。

在本章的后半部分，我们学习了如何一次性运行多个具有必要依赖项（如数据库和邮件服务器）的微服务。我们使用 Docker Compose 工具描述了具有运行顺序和端口转发的微服务集的配置。我们还学习了如何将卷附加到服务（容器）上，以存储持久数据，并允许您在没有任何数据丢失风险的情况下重新启动应用程序。

在下一章中，我们将学习如何使用持续集成工具自动化构建微服务，帮助您更快地交付产品的最新版本。
