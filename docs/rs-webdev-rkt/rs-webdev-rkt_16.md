# *第十三章*：启动 Rocket 应用程序

在开发和测试之后，开发的一个重要部分是准备应用程序以服务于其目标用户。在本章中，我们将学习一些生成生产就绪二进制文件的技术。在我们生成二进制文件之后，我们将学习配置位于通用 Web 服务器后面的应用程序。最后，我们将学习如何为 Rocket 应用程序生成 Docker 镜像。

在学习本章中的信息后，您将能够使用 Rust 编译器标志和 Cargo 配置来优化二进制文件。您还将学习准备您的应用程序以服务于其目标用户的技术。

在本章中，我们将涵盖以下主要主题：

+   优化生产二进制文件

+   使用 Rocket 应用程序设置 Apache HTTP 服务器

+   为 Rocket 应用程序生成 Docker 镜像

# 技术要求

在本章中，我们将使用 Apache HTTP 服务器([`httpd.apache.org/`](https://httpd.apache.org/))来处理 HTTP 请求。如果您有基于 Unix 的操作系统，您通常可以在操作系统的软件包管理器中找到 Apache HTTP 服务器。如果您有 Windows 操作系统，以下链接中有推荐的下载：[`httpd.apache.org/docs/2.4/platform/windows.html`](https://httpd.apache.org/docs/2.4/platform/windows.html)。

您还需要使用 OpenSSL 生成一个 TLS（传输层安全性）证书。如果您有基于 Unix 的操作系统，您通常可以使用发行版的软件包管理器找到 OpenSSL 二进制文件。如果您有 Windows 操作系统，您可以在以下链接中找到推荐的二进制文件：[`wiki.openssl.org/index.php/Binaries`](https://wiki.openssl.org/index.php/Binaries)。

对于生成 Docker 镜像，您可以使用以下链接中的 Docker Desktop：[`www.docker.com/products/docker-desktop/`](https://www.docker.com/products/docker-desktop/)。

您可以在以下位置找到本章的源代码：[`github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter13`](https://github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter13)

# 优化生产二进制文件

在我们创建应用程序之后，我们希望准备应用程序以接受真实连接。在软件开发中，有一个生产环境，也称为发布环境或部署环境。生产环境包含系统的配置和软件，使其能够提供给目标客户。在*第二章*，“构建我们的第一个 Rocket Web 应用程序”，我们了解到我们可以告诉 Rust 编译器在编译 Rust 应用程序时构建发布二进制文件。我们可以使用带有额外`--release`标志的`cargo build`或`cargo run`。

为了刷新，Cargo 将读取 `[profile.release]` 部分中的 `Cargo.toml` 配置。我们可以进行一些编译优化来提高生成的镜像：

1.  第一个设置是编译的 `codegen-units` 数量。Rust 编译可能需要很长时间，为了解决这个问题，编译器可能会尝试将其分割成多个部分并行编译。但是，并行编译二进制文件或库可能会遗漏一些优化。默认情况下，`codegen-units` 的数量是 `3`。我们可以牺牲编译速度，将 `codegen-units` 设置为 `1` 以进一步优化生成的二进制文件。例如，在 `Cargo.toml` 中，我们可以有如下设置：

    ```rs
    [profile.release]
    codegen-units = 1
    ```

`codegen` 后端，LLVM，可以执行各种 LTO（链接时优化）以生成优化后的代码输出。要启用 LTO，我们可以设置 `lto = yes` 或 `lto = "fat"`。以下是在 `Cargo.toml` 中 `lto` 的一个示例：

```rs
[profile.release]
lto = "fat"
```

设置优化级别。我们可以设置从 `0`、`1`、`2` 和 `3` 的优化级别，默认值是 `0`（无优化）到 `3`（所有优化），如下所示：

```rs
[profile.release]
opt-level = 3
```

除了优化级别 `0` 到 `3`，我们还可以设置 `"s"` 和 `"z"`，其中 `"s"` 用于二进制大小优化，而 `"z"` 用于二进制大小优化并关闭循环向量化。

禁用 `panic` 回溯。我们可以设置 `panic` 不显示堆栈跟踪。结果是更优化的二进制文件。在 `Cargo.toml` 中设置以下内容以禁用堆栈回溯：

```rs
[profile.release]
panic = "abort"
```

1.  第二个优化是编译正确的架构。CPU 生产商总是会创建一个新的 CPU，具有更好的优化或指令集，这可以提高应用程序的性能。例如，SSE（Streaming SIMD Extensions）指令集是由英特尔在发布英特尔奔腾 III 时引入的。

默认情况下，Rust 编译器会生成具有合理 CPU 支持的二进制文件。但这意味着在编译库或二进制文件时不会使用新的指令集或优化。我们可以告诉 Rust 编译器生成支持目标机器新优化或指令集的二进制文件。

要查看 Rust 编译器支持的目标架构列表，我们可以使用以下命令：

```rs
rustc --print target-cpus
```

例如，如果我们知道目标机器是支持 *znver3* 架构的 AMD Ryzen，我们可以按照以下方式编译 Rust 程序：

```rs
RUSTFLAGS='-C target-cpu=znver -C codegen-units=1' cargo build –release
```

我们使用 `RUSTFLAGS` 环境变量的原因是 `target-cpu` 选项在 `Cargo.toml` 中不被认可。Cargo 还会使用在 `RUSTFLAGS` 环境变量中设置的任何其他 `rustc` 选项。

现在我们知道了如何为生产环境编译 Rust 应用程序，让我们学习如何在 Web 服务器后面部署 Rocket 应用程序。

# 配置 Apache HTTP 服务器与 Rocket 应用程序

我们知道 Rocket 在其配置中支持 TLS，因此我们可以将 TCP 端口设置为`443`，这是默认的 HTTPS 连接端口。在某些情况下，直接运行 Web 应用程序可能是可接受的，例如，当我们想要为微服务提供服务内容时。

我们不想直接运行 Rocket 应用程序的一个原因是因为 Rocket 指南中的这个警告：

Rocket 内置的 TLS 仅实现了 TLS 1.2 和 1.3。它可能不适合生产使用。

Rocket 框架使用的 TLS 库可能由于各种原因（如安全原因或尚未经过审计）不适合生产使用。

除了 TLS 库问题之外，还有其他原因我们不希望直接从 Rocket 服务内容。一个例子是我们想要从同一台计算机上服务多个应用程序。我们也可能想要在同一台机器上服务 PHP 应用程序。

当人们服务 Rust 应用程序时，会使用的一种技术是将它放在一个通用的 Web 服务器后面，该服务器可以执行反向代理：

![图 13.1 - 执行 Rocket 应用程序反向代理的通用 Web 服务器]

![img/Figure_13.1_B16825.jpg](img/Figure_13.1_B16825.jpg)

图 13.1 - 执行 Rocket 应用程序反向代理的通用 Web 服务器

最常用的反向代理应用程序之一是 Apache HTTP 服务器。Apache HTTP 服务器除了反向代理之外，还有其他功能，包括服务静态文件和压缩文件以更快地处理请求。

让我们尝试使用 Apache HTTP 服务器来服务我们的应用程序，并按照以下步骤配置服务器以作为反向代理：

1.  下载适用于您操作系统的 Apache HTTP 服务器或从[`httpd.apache.org/`](https://httpd.apache.org/)下载。

1.  尝试使用以下命令行启动应用程序：

    ```rs
    sudo apachectl -k start
    ```

1.  Apache HTTP 服务器的默认端口是`8080`。使用 cURL 命令检查 Apache 是否正在运行：

    ```rs
    curl http://127.0.0.1:8080/
    ```

1.  Apache HTTP 服务器的功能可以通过模块进行扩展，并且安装了几个模块。我们想要启用几个模块，以便通过反向代理使用 HTTP 请求来访问我们的应用程序。找到`httpd.conf`，这是您操作系统的配置文件。在某些 Linux 发行版中，配置可能位于`/etc/httpd/httpd.conf`。在其他发行版或操作系统上，文件位置可能位于`/usr/local/etc/httpd/httpd.conf`。

编辑`httpd.conf`文件并取消注释以启用所需的模块：

```rs
LoadModule log_config_module libexec/apache2/mod_log_config.so
LoadModule vhost_alias_module libexec/apache2/mod_vhost_alias.so
LoadModule socache_shmcb_module libexec/apache2/mod_socache_shmcb.so
LoadModule ssl_module libexec/apache2/mod_ssl.so
LoadModule xml2enc_module libexec/apache2/mod_xml2enc.so
LoadModule proxy_html_module libexec/apache2/mod_proxy_html.so
LoadModule proxy_module libexec/apache2/mod_proxy.so
LoadModule proxy_connect_module libexec/apache2/mod_proxy_connect.so
LoadModule proxy_http_module libexec/apache2/mod_proxy_http.so
```

1.  在相同的`httpd.conf`文件中，找到这些行并取消注释这些行：

    ```rs
    Include /usr/local/etc/httpd/extra/httpd-vhosts.conf
    Include /usr/local/etc/httpd/extra/httpd-ssl.conf
    Include /usr/local/etc/httpd/extra/proxy-html.conf
    ```

1.  我们需要一个服务器名称。在真实服务器中，我们可以通过从域名注册商购买域名权利并将`ourapplication.example.net`指向它来获取一个域名。按照以下方式编辑`/etc/hosts`和一些测试域名：

    ```rs
    127.0.0.1 ourapplication.example.net
    ```

1.  为您的操作系统安装`openssl`。之后，使用`openssl`命令行生成`ourapplication.example.net`的证书，如下所示：

    ```rs
    openssl req -x509 -out ourapplication.example.com.crt -keyout ourapplication.example.com.key \
      -newkey rsa:2048 -nodes -sha256 \
    -subj '/CN=ourapplication.example.com' -extensions 
      EXT -config <( \
       printf "[dn]\nCN=ourapplication.example
      .com\n[req]\ndistinguished_name = dn\n[EXT]\
      nsubjectAltName=DNS:ourapplication.
      example.com\nkeyUsage=digitalSignature\
      nextendedKeyUsage=serverAuth")
    ```

命令行将生成两个文件，`ourapplication.example.com.crt` 和 `ourapplication.example.com.key`。

1.  生成一个 PEM 文件，该文件格式包含以下证书：

    ```rs
    openssl rsa -in ourapplication.example.com.key -text > ourapplication.example.com.private.pem
    openssl x509 -inform PEM -in ourapplication.example.com.crt > ourapplication.example.com.public.pem 
    ```

1.  编辑 `httpd-vhosts.conf` 文件。该文件可能位于 `/usr/local/etc/httpd/extra/`，具体取决于您的操作系统配置。添加一个新的虚拟主机。我们希望虚拟主机指向 `http://127.0.0.1:8000` 的我们的 Rocket 应用程序。添加以下行：

    ```rs
    <VirtualHost *:443>
        ServerName ourapplication.example.com
        SSLEngine On
        SSLCertificateFile /usr/local/etc/httpd/
        ourapplication.example.com.public.pem
        SSLCertificateKeyFile /usr/local/etc/httpd/
        ourapplication.example.com.private.pem
        SSLProxyEngine On
        ProxyRequests Off
        ProxyVia Off
        <Proxy *>
             Require all granted
        </Proxy>
        ProxyPass "/" "http://127.0.0.1:8000/"
        ProxyPassReverse "/" "http://127.0.0.1:8000/"
    </VirtualHost>
    ```

1.  通过运行以下命令来检查配置是否正确：

    ```rs
    sudo apachectl configtest
    ```

1.  在您的网络浏览器中重新启动并打开 [`ourapplication.example.com`](https://ourapplication.example.com)。网络浏览器可能会抱怨根证书未知。我们可以添加我们生成的证书，以便它在我们的浏览器中被接受。例如，在 Firefox 中，我们可以转到 **首选项** | **隐私和安全** | **查看证书**。之后，选择 **服务器标签** 并点击 **添加异常**。然后，确保 **永久存储此异常** 被选中。最后，点击 **确认安全异常** 以存储安全异常。如果一切顺利，我们可以在浏览器中使用示例域名，如图所示：

![Figure 13.2 – 使用域名和 TLS 证书![img/Figure_13.2_B16825.jpg](img/Figure_13.2_B16825.jpg)

![Figure 13.2 – 使用域名和 TLS 证书

现在我们已经将 Rocket 应用程序部署在反向代理后面，我们可以使用同样的原理与真实服务器一起使用。设置 Apache HTTP 服务器或 NGINX 作为反向代理，并在反向代理后面运行 Rocket 应用程序。

要在操作系统启动时自动运行 Rocket 应用程序，我们可以为操作系统设置某种类型的服务。如果我们运行的是以 systemd 作为服务管理器的 Linux 发行版，例如，我们可以创建一个 `systemd` 服务文件并自动运行应用程序。

在下一节中，我们将学习一种不同的部署应用程序的方法。我们将使用 Docker 打包并创建我们的 Rocket 应用程序的 Docker 容器。

# 为 Rocket 应用程序生成 Docker 镜像

容器化已经是一段时间内用于打包生产应用程序的热门选择。容器化最流行的应用程序之一是 Docker。在本节中，我们将学习如何设置 Docker 以运行我们的 Rocket 应用程序。要使用 `docker` 命令行，请从 [`www.docker.com/products/docker-desktop/`](https://www.docker.com/products/docker-desktop/) 安装 Docker Desktop。

按照以下步骤创建和运行 Rocket 应用程序的 Docker 镜像：

1.  在应用程序的根目录中创建一个 Dockerfile。

1.  我们可以使用一些基础镜像来构建和运行应用程序。我们将使用来自 [`hub.docker.com/_/rust`](https://hub.docker.com/_/rust) 的 Rust 官方 Docker 镜像。对于 Linux 发行版，我们将使用 *Alpine base*，因为它是最小的 Docker 基础镜像之一。

在 Dockerfile 中添加第一行：

```rs
FROM rust:alpine as prepare-stage
```

1.  设置工作目录。将此行追加到 Dockerfile 中：

    ```rs
    WORKDIR /app
    ```

1.  我们可以使用 Cargo 安装依赖项，但还有一种快速编译应用程序的方法。我们可以供应商化依赖项并使用供应商依赖项来构建应用程序。在应用程序源代码的根目录下运行此命令：

    ```rs
    cargo vendor
    ```

1.  我们希望覆盖从互联网到供应商文件夹的依赖项来源。在根应用程序文件夹中创建一个 `.cargo` 文件夹，并在 `.cargo` 文件夹内创建 `config.toml`。

将这些行追加到 `.cargo/config.toml` 文件中：

```rs
[source.crates-io]
replace-with = "vendored-sources"
[source.vendored-sources]
directory = "vendor"
```

1.  我们希望添加构建应用程序为 Docker 镜像所需的文件。我们不需要 `Rocket.toml`、模板或静态文件来构建应用程序。将这些行追加到 Dockerfile 中：

    ```rs
    COPY src src
    COPY Cargo.toml Cargo.toml
    COPY .cargo .cargo
    COPY vendor vendor
    ```

1.  添加构建镜像的指令。我们希望使用另一个阶段并安装依赖项来构建镜像。添加以下行：

    ```rs
    FROM prepare-stage as build-stage
    RUN apk add --no-cache musl-dev
    RUN cargo build --release
    ```

1.  通过运行以下命令尝试构建应用程序：

    ```rs
    docker build .
    ```

1.  在测试之后，向 Dockerfile 中添加一个新的部分以运行应用程序。我们希望打开端口 `8000`。我们还希望添加默认时区并配置用户以运行应用程序。追加以下行：

    ```rs
    FROM rust:alpine
    EXPOSE 8000
    ENV TZ=Asia/Tokyo \
        USER=staff
    RUN addgroup -S $USER \
        && adduser -S -g $USER $USER
    ```

1.  我们希望镜像包含最新的库。将以下行追加到 Dockerfile 中：

    ```rs
    RUN apk update \
        && apk add --no-cache ca-certificates tzdata \
        && rm -rf /var/cache/apk/*
    ```

1.  设置工作目录。将此行追加到 Dockerfile 中：

    ```rs
    WORKDIR /app
    ```

1.  将 `Rocket.toml` 设置为从 `0.0.0.0` 运行。我们希望告诉应用程序使用主机运行的数据库。在 Docker 中，我们可以使用特殊域名 host.docker.internal 来引用主机机器。按照以下方式编辑 `Rocket.toml`：

    ```rs
    [default.databases.main_connection]
    url = "postgres://username:passwordR@host.docker.internal:5432/rocket"
    [release]
    address = "0.0.0.0"
    ```

1.  将生成的二进制文件、`Rocket.toml`、资产和模板复制到最终镜像中。将这些行追加到 Dockerfile 中：

    ```rs
    COPY --from=build-stage /app/target/release/our_application our_application
    COPY Rocket.toml Rocket.toml
    COPY static static
    COPY src/views src/views
    ```

1.  添加存储日志文件的文件夹：

    ```rs
    RUN mkdir logs
    ```

1.  添加更改权限到 `$USER`，如下所示：

    ```rs
    RUN chown -R $USER:$USER /app
    ```

1.  最后，将应用程序的入口点运行到 Dockerfile 中：

    ```rs
    USER $USER
    CMD ["./our_application"]
    ```

1.  使用此命令构建镜像并为其创建标签：

    ```rs
    docker build -t our_application .
    ```

1.  在构建了 Docker 镜像之后，是时候运行它了。使用以下命令行：

    ```rs
    docker run --add-host host.docker.internal:host-gateway -dp 8000:8000 our_application
    ```

一切完成后，我们应该看到 Docker 容器正在运行并显示 `our_application` 输出：

![图 13.3 - Docker Desktop 显示正在运行的容器和我们的应用程序](img/Figure_13.3_B16825.jpg)

图 13.3 - Docker Desktop 显示正在运行的容器和我们的应用程序

使用 Docker 部署 Rocket 应用程序就像部署其他应用程序一样。我们需要复制源代码、构建并运行生成的镜像。有一些操作可以执行以确保正确部署，例如供应商库和打开正确的端口以确保可以向正在运行的容器和容器内运行的应用程序发送请求。

# 摘要

在本章中，我们学习了生产就绪的编译选项。我们可以使用它们来确保生成的二进制文件尽可能优化。我们还学习了如何设置一个通用目的的 HTTP 服务器与 Rocket 应用程序协同工作。最后，我们学习了如何为 Rocket 应用程序创建和运行 Docker 镜像。

在学习这些技术之后，我们将它们扩展到设置 Rocket 应用程序以服务于其目标用户。

在下一章中，我们将学习如何使用 Rust 与 Rocket 应用程序结合创建前端 WebAssembly 应用程序。
