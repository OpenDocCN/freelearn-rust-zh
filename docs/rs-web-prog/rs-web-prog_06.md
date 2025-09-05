# 6

# 使用 PostgreSQL 实现数据持久化

到本书的这一阶段，我们应用程序的前端已经定义，我们的应用程序在表面上运行良好。然而，我们知道我们的应用程序是从一个 **JSON** 文件中读取和写入的。

在本章中，我们将放弃我们的 JSON 文件，并引入一个 `create`、`edit` 和 `delete` 端点来与数据库而不是 JSON 文件进行交互。

在本章中，我们将涵盖以下主题：

+   构建我们的 PostgreSQL 数据库

+   使用 Diesel 连接到 PostgreSQL

+   将我们的应用程序连接到 PostgreSQL

+   配置我们的应用程序

+   管理数据库连接池

到本章结束时，你将能够管理一个应用程序，该应用程序使用数据模型在 PostgreSQL 数据库中执行读取、写入和删除数据。如果我们对数据模型进行更改，我们将能够通过迁移来管理它们。一旦完成这些操作，你将能够使用连接池优化你的数据库连接，并在无法建立数据库连接之前拒绝 HTTP 请求。

# 技术要求

在本章中，我们将使用 **Docker** 来定义、运行一个 PostgreSQL 数据库，并运行它。这将使我们的应用程序能够与本地机器上的数据库进行交互。可以通过遵循[`docs.docker.com/engine/install/`](https://docs.docker.com/engine/install/)中的说明来安装 Docker。

我们还将在 Docker 之上使用 `docker-compose` 来编排我们的 Docker 容器。可以通过遵循[`docs.docker.com/compose/install/`](https://docs.docker.com/compose/install/)中的说明来安装 `docker-compose`。

本章的代码文件可以在[`github.com/PacktPublishing/Rust-Web-Programming-2nd-Edition/tree/main/chapter06`](https://github.com/PacktPublishing/Rust-Web-Programming-2nd-Edition/tree/main/chapter06)找到。

# 构建我们的 PostgreSQL 数据库

到本书的这一阶段，我们一直在使用 JSON 文件来存储我们的待办事项。到目前为止，这已经很好地为我们服务了。事实上，我们没有理由在整个书中使用 JSON 文件来完成任务。然而，如果你在生产项目中使用 JSON 文件，你将遇到一些缺点。

## 为什么我们应该使用合适的数据库

如果我们对 JSON 文件的读写操作增加，我们可能会遇到一些并发问题和数据损坏。此外，也没有对数据类型进行检查。因此，另一个开发者可以编写一个将不同数据写入 JSON 文件的功能，而没有任何阻碍。

迁移也存在一个问题。如果我们想给待办事项添加时间戳，这将只会影响我们插入到 JSON 文件中的新待办事项。因此，一些待办事项将具有时间戳，而另一些则不会，这将在我们的应用程序中引入错误。我们的 JSON 文件在过滤方面也存在限制。

目前，我们只是读取整个数据文件，修改整个数据集中的某个条目，并将整个数据集写入**JSON**文件。这并不有效，并且扩展性不好。它还阻碍了我们将这些待办事项与另一个数据模型（如用户）链接起来。此外，我们目前只能使用状态进行搜索。如果我们使用一个与待办事项数据库链接的用户表的 SQL 数据库，我们就能根据用户、状态或标题过滤待办事项。我们甚至可以使用它们的组合。当我们运行数据库时，我们将使用 Docker。那么，为什么我们应该使用 Docker 呢？

## 为什么使用 Docker？

要理解为什么我们会使用 Docker，我们首先需要了解 Docker 是什么。Docker 本质上具有像虚拟机一样工作的容器，但方式更加具体和细致。Docker 容器隔离单个应用程序及其所有依赖项。应用程序在 Docker 容器内运行。然后，Docker 容器可以相互通信。由于 Docker 容器共享单个公共**操作系统**（**OS**），它们彼此以及与整个操作系统隔离开来，这意味着 Docker 应用程序相比虚拟机使用更少的内存。由于 Docker 容器，我们可以使我们的应用程序更加便携。如果 Docker 容器在您的机器上运行，它也会在另一个安装了 Docker 的机器上运行。我们还可以打包我们的应用程序，这意味着不需要单独安装为我们的应用程序运行所必需的额外包，包括操作系统级别的依赖项。因此，Docker 在 Web 开发中为我们提供了极大的灵活性，因为我们可以在本地机器上模拟服务器和数据库。

## 如何使用 Docker 运行数据库

考虑到所有这些，通过额外的步骤来设置和运行一个 SQL 数据库是有意义的。为此，我们将使用 Docker：这是一个帮助我们创建和使用容器的工具。容器本身是 Linux 技术，它将应用程序及其整个运行时环境打包和隔离。容器在技术上是一个隔离的文件系统，但为了帮助可视化我们在本章中所做的工作，你可以把它们想象成迷你轻量级虚拟机。这些容器是由可以从**Docker Hub**下载的镜像构成的。在从这些镜像启动容器之前，我们可以将我们自己的代码插入到这些镜像中，如下面的图所示：

![图 6.1 – Docker 镜像和容器之间的关系](img/Figure_6.1_B18722.jpg)

图 6.1 – Docker 镜像和容器之间的关系

使用 Docker，我们可以下载一个镜像，例如 PostgreSQL 数据库，并在我们的开发环境中运行它。由于 Docker，我们可以根据需要启动多个数据库和应用程序，然后关闭它们。首先，我们需要通过在终端运行以下命令来盘点我们的容器：

```rs
docker container ls -a
```

如果 Docker 是全新安装，我们将得到以下输出：

```rs
CONTAINER ID  IMAGE  COMMAND  CREATED  STATUS  PORTS  NAMES
```

如我们所见，我们没有容器。我们还需要盘点我们的镜像。这可以通过运行以下终端命令来完成：

```rs
docker image ls
```

前面的命令给出了以下输出：

```rs
REPOSITORY  TAG  IMAGE ID  CREATED  SIZE
```

再次强调，如果 Docker 是全新安装，那么将没有容器。

我们还可以通过其他方式在 Docker 中创建数据库。例如，我们可以创建自己的已安装的`docker-compose`。使用`docker-compose`将使数据库定义变得简单。它还将使我们能够添加更多容器和服务。为了定义我们的 PostgreSQL 数据库，我们在根目录中编写以下`docker-compose.yml`文件：

```rs
version: "3.7"
services:
  postgres:
    container_name: 'to-do-postgres'
    image: 'postgres:11.2'
    restart: always
    ports:
      - '5432:5432'
    environment:
      - 'POSTGRES_USER=username'
      - 'POSTGRES_DB=to_do'
      - 'POSTGRES_PASSWORD=password'
```

在前面的代码中，文件顶部我们定义了版本。较老的版本，如*2*或*1*，在文件布局上具有不同的风格。不同的版本也支持不同的参数。在撰写本书时，版本*3*是最新的版本。以下 URL 涵盖了每个`docker-compose`版本之间的变化：[`docs.docker.com/compose/compose-file/compose-versioning/`](https://docs.docker.com/compose/compose-file/compose-versioning/).

然后我们定义嵌套在`postgres`标签下的数据库服务。如`postgres`和`services`之类的标签表示字典，列表使用`-`定义每个元素。如果我们把我们的`docker-compose`文件转换为 JSON，它将具有以下结构：

```rs
{
  "version": "3.7",
  "services": {
    "postgres": {
      "container_name": "to-do-postgres",
      "image": "postgres:11.2",
      "restart": "always",
      "ports": [
        "5432:5432"
      ],
      "environment": [
        "POSTGRES_USER=username",
        "POSTGRES_DB=to_do",
        "POSTGRES_PASSWORD=password"
      ]
    }
  }
}
```

在前面的代码中，我们可以看到我们的服务是一个字典的字典，表示每个服务。因此，我们可以推断出我们不能有两个具有相同名称的标签，因为我们不能有两个相同的字典键。前面的代码还告诉我们，我们可以继续堆叠具有自己参数的服务标签。

## 在 Docker 中运行数据库

使用我们的数据库服务，我们有一个名称，因此当我们查看我们的容器时，我们知道每个容器相对于服务（如服务器或数据库）正在做什么。在配置数据库和构建它方面，我们幸运地拉取了官方的`postgres`镜像。这个镜像为我们配置了一切，Docker 会从仓库中拉取它。这个镜像就像一个蓝图。我们可以从这个我们拉取的一个镜像中启动多个具有自己参数的容器。然后我们定义重启策略为总是。这意味着当容器退出时，容器的重启策略将被触发。我们还可以将其定义为仅在失败或停止时重启。

应该注意的是，Docker 容器有自己的端口，这些端口对机器是关闭的。然而，我们可以公开容器的端口，并将公开的端口映射到 Docker 容器内部的内部端口。考虑到这些特性，我们可以定义我们的端口。

然而，在我们的例子中，我们将保持我们的定义简单。我们将声明我们接受端口`5432`上进入 Docker 容器的流量，并将其路由到内部端口`5432`。然后我们将定义我们的环境变量，包括用户名、数据库名称和密码。虽然我们在这本书中使用的是通用、易于记忆的密码和用户名，但如果要推送到生产环境，建议您切换到更安全的密码和用户名。我们可以通过运行以下命令来构建和启动我们的系统，该命令将我们的`docker-compose`文件所在的根目录：

```rs
docker-compose up
```

前面的命令将从存储库中拉取`postgres`镜像并开始构建数据库。在一系列日志消息之后，终端应该显示以下输出：

```rs
LOG:  listening on IPv4 address "0.0.0.0", port 5432
LOG:  listening on IPv6 address "::", port 5432
LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
LOG:  database system was shut down at 2022-04-23 17:36:45 UTC
LOG:  database system is ready to accept connections
```

如您所见，日期和时间会变化。然而，这里告诉我们的是我们的数据库已经准备好接受连接。是的，这真的很简单。因此，Docker 的采用是不可避免的。点击*Ctrl* + *C*将停止我们的`docker-compose`；从而关闭我们的`postgres`容器。

我们现在使用以下命令列出所有我们的容器：

```rs
docker container ls -a
```

前面的命令给出了以下输出：

```rs
CONTAINER ID        IMAGE               COMMAND                  
c99f3528690f        postgres:11.2       "docker-entrypoint.s…"
CREATED             STATUS                          PORTS
4 hours ago         Exited (0) About a minute ago        
NAMES
to-do-postgres
```

在前面的输出中，我们可以看到所有参数都在那里。然而，端口却是空的，因为我们已经停止了我们的服务。

## 探索 Docker 中的路由和端口

如果我们再次启动我们的服务并在另一个终端中列出我们的容器，端口`5432`将位于`PORTS`标签下。我们必须注意`CONTAINER ID`对 Docker 容器的引用，因为它是唯一的，对于每个容器都是不同的/随机的。如果我们需要访问日志，我们需要引用这些。当我们运行`docker-compose up`时，我们实际上使用以下结构：

![图 6.2 – docker-compose 服务我们的数据库](img/Figure_6.2_B18722.jpg)

图 6.2 – docker-compose 服务我们的数据库

在*图 6.2*中，我们可以看到我们的`docker-compose`使用一个独特的项目名称来保持容器和网络在其命名空间中。必须注意的是，我们的容器正在本地主机上运行。因此，如果我们想调用由`docker-compose`管理的容器，我们必须发起一个本地主机请求。然而，我们必须调用`docker-compose`开放的端口，`docker-compose`会将它路由到`docker-compose`.`yml`文件中定义的端口。例如，我们有两个数据库，以下是一个`yml`文件：

```rs
version: "3.7"
services:
  postgres:
    container_name: 'to-do-postgres'
    image: 'postgres:11.2'
    restart: always
    ports:
      - '5432:5432'
    environment:
      - 'POSTGRES_USER=username'
      - 'POSTGRES_DB=to_do'
      - 'POSTGRES_PASSWORD=password'
  postgres_two:
    container_name: 'to-do-postgres_two'
    image: 'postgres:11.2'
    restart: always
    ports:
      - '5433:5432'
    environment:
      - 'POSTGRES_USER=username'
      - 'POSTGRES_DB=to_do'
      - 'POSTGRES_PASSWORD=password'
```

在前面的代码中，我们可以看到我们的两个数据库都通过端口`5432`接受进入其容器的流量。然而，会有冲突，所以我们打开的端口之一是端口`5433`，它被路由到第二个数据库容器的端口`5432`，这给我们以下布局：

![图 6.3 – docker-compose 服务多个数据库](img/Figure_6.3_B18722.jpg)

图 6.3 – docker-compose 服务多个数据库

这种路由在运行多个容器时给我们提供了灵活性。我们不会为我们的待办事项应用运行多个数据库，因此我们应该删除我们的`postgres_two`服务。一旦我们删除了`postgres_two`服务，我们就可以再次运行`docker-compose`，然后使用以下命令列出我们的容器：

```rs
docker image ls
```

前面的命令现在将给出以下输出：

```rs
REPOSITORY          TAG                 IMAGE ID    
postgres            11.2                3eda284d1840
CREATED             SIZE
17 months ago       312MB
```

在前面的输出中，我们可以看到我们的镜像是从`postgres`仓库拉取的。我们还有一个为该镜像生成的唯一/随机 ID，以及该镜像创建的日期。

现在我们已经基本了解了如何让我们的数据库启动并运行，我们可以使用以下命令在后台运行`docker-compose`：

```rs
docker-compose up -d
```

前面的命令只是告诉我们哪些容器已经被启动，以下为输出：

```rs
Starting to-do-postgres ... done
```

我们可以通过以下输出查看我们的状态：

```rs
STATUS              PORTS                    NAMES
Up About a minute   0.0.0.0:5432->5432/tcp   to-do-postgres
```

在前面的输出中，其他标签都是相同的，但我们还可以看到`STATUS`标签告诉我们容器运行了多长时间，以及它占用了哪个端口。尽管`docker-compose`在后台运行，但这并不意味着我们无法看到正在发生的事情。我们可以通过调用`logs`命令并使用以下命令引用容器的 ID 来随时访问容器的日志：

```rs
docker logs c99f3528690f
```

前面的命令应该给出与我们的标准`docker-compose up`命令相同的输出。要停止`docker-compose`，我们可以运行`stop`命令，如下所示：

```rs
docker-compose stop
```

前面的命令将停止我们的`docker-compose`中的容器。必须注意的是，这与下面的`down`命令不同：

```rs
docker-compose down
```

`down`命令也会停止我们的容器。然而，`down`命令会删除容器。如果我们的数据库容器被删除，我们也会丢失所有数据。

有一个名为`volumes`的配置参数可以在容器删除时防止我们的数据被删除；然而，这对于我们在计算机上的本地开发并不是必需的。实际上，你可能会想要定期从你的笔记本电脑上删除容器和镜像。*我曾在我的笔记本电脑上执行了一次删除不再使用的容器和镜像的操作，这释放了* *23GB!*

在我们的本地开发机器上的 Docker 容器应该被视为临时的。虽然 Docker 容器比标准虚拟机更轻量级，但它们并不是免费的。Docker 在我们本地机器上运行背后的想法是，我们可以模拟在服务器上运行我们的应用程序的情况。如果它在我们的笔记本电脑上的 Docker 中运行，我们可以确信它也会在我们的服务器上运行，特别是如果服务器是由一个生产就绪的 Docker 编排工具（如**Kubernetes**）管理的。

## 使用 Bash 脚本在后台运行 Docker

Docker 还可以帮助进行一致的测试和开发。我们希望每次运行测试时都能得到相同的结果。我们还希望能够轻松地让其他开发者入职，并使他们能够快速、轻松地关闭和启动容器以支持开发。我个人看到，如果不支持简单的关闭和启动程序，开发可能会被推迟。例如，当处理复杂的应用程序时，我们添加并测试的代码可能会损坏数据库。回滚可能不可行，删除数据库并重新开始将是一件痛苦的事情，因为重建这些数据可能需要很长时间。开发者甚至可能不记得最初是如何构建这些数据的。有多种方法可以防止这种情况发生，我们将在*第九章*中介绍，*测试我们的应用程序端点和组件*。

现在，我们将构建一个脚本，在后台启动我们的数据库，等待直到数据库连接就绪，然后关闭我们的数据库。这将为我们构建管道、测试和入职包以开始开发奠定基础。为此，我们将在 Rust Web 应用的根目录中创建一个名为`scripts`的目录。然后我们可以创建一个`scripts/wait_for_database.sh`文件，包含以下代码：

```rs
#!/bin/bash
cd ..
docker-compose up -d
until pg_isready -h localhost -p 5432 -U username
do
  echo "Waiting for postgres"
  sleep 2;
done
echo "docker is now running"
docker-compose down
```

使用前面的代码，我们将脚本的当前工作目录从`scripts`目录移动到我们的根目录。然后我们在后台启动`docker-compose`。接下来，我们循环，通过使用`pq_isready`命令 ping 端口`5432`，等待直到我们的数据库准备好接受连接。

重要注意事项

`pg_isready` Bash 命令可能不在您的计算机上可用。`pg_isready`命令通常与 PostgreSQL 客户端的安装一起提供。或者，您可以使用以下 Docker 命令代替`pg_isready`：

`until docker run -it postgres --add-host host.docker.internal:host-gateway docker.io/postgres:14-alpine -h localhost -U` `username pg_isready`

这里发生的情况是，我们正在使用`postgres` Docker 镜像来运行我们的数据库检查，以确保我们的数据库已准备好接受连接。

当我们的数据库启动后，我们在控制台打印出我们的数据库正在运行，然后关闭我们的`docker-compose`，销毁数据库容器。运行执行`wait_for_database.sh` Bash 脚本的命令将给出以下输出：

```rs
❯ sh wait_for_database.sh 
[+] Running 0/0
 ⠋ Network web_app_default  Creating        0.2s
 ⠿ Container to-do-postgres      Started    1.5s
localhost:5432 - no response
Waiting for postgres
localhost:5432 - no response
Waiting for postgres
localhost:5432 - accepting connections
docker is now running
[+] Running 1/1
 ⠿ Container to-do-postgres  Removed        1.2s
 ⠿ Network web_app_default   Removed  
```

从前面的输出中，考虑到我们告诉循环在每次迭代中暂停 2 秒，我们可以推断出我们的新启动的数据库接受连接大约需要 4 秒钟。因此，我们可以说我们已经掌握了使用 Docker 管理本地数据库的基本技能。

在本节中，我们设置了我们的环境。我们也充分理解了 Docker 的基本知识，只需几个简单的命令就能构建、监控、关闭和删除我们的数据库。现在，我们可以继续到下一节，我们将使用 Rust 和`diesel` crate 与我们的数据库进行交互。

# 使用 Diesel 连接到 PostgreSQL

现在我们的数据库正在运行，我们将在本节中构建与该数据库的连接。为此，我们将使用`diesel` crate。`diesel` crate 使我们能够使我们的 Rust 代码连接到数据库。我们使用`diesel` crate 而不是其他 crate，因为`diesel` crate 是最成熟的，拥有大量的支持和文档。为此，让我们按照以下步骤进行：

1.  首先，我们将利用`diesel` crate。为此，我们可以在`cargo.toml`文件中添加以下依赖项：

    ```rs
    diesel = { version = "1.4.8", features = ["postgres", 
    ```

    ```rs
                                              "chrono", 
    ```

    ```rs
                                              "r2d2"] }
    ```

    ```rs
    dotenv = "0.15.0"
    ```

    ```rs
    chrono = "0.4.19"
    ```

在前面的代码中，我们在`diesel` crate 中包含了`postgres`功能。`diesel` crate 的定义还包括了`chrono`和`r2d2`功能。`chrono`功能使我们的 Rust 代码能够利用 datetime 结构体。`r2d2`功能使我们能够执行连接池。我们将在本章末尾介绍连接池。我们还包含了`dotenv` crate。这个 crate 使我们能够在`.env`文件中定义变量，这些变量随后将被传递到我们的程序中。我们将使用它来传递数据库凭据，然后传递到进程中。

1.  现在我们需要安装`diesel`客户端，以便通过我们的终端而不是应用程序运行迁移到数据库。我们可以使用以下命令来完成：

    ```rs
    cargo install diesel_cli --no-default-features 
    ```

    ```rs
        --features postgres
    ```

1.  我们现在需要定义环境变量`DATABASE_URL` URL。这将使我们的客户端命令能够使用以下命令连接到数据库：

    ```rs
    echo DATABASE_URL=postgres://username:password@localhost/to_do 
    ```

    ```rs
        > .env
    ```

在前面的 URL 中，我们的用户名表示为 username，密码表示为 password。我们的数据库运行在我们的计算机上，表示为`localhost`，我们的数据库名为`to_do`。这将在根目录中创建一个`.env`文件，输出以下内容：

```rs
DATABASE_URL=postgres://username:password@localhost/to_do
```

1.  现在我们已经定义了变量，我们可以开始设置我们的数据库。我们需要使用`docker-compose up`命令启动我们的数据库容器。然后，我们使用以下命令设置我们的数据库：

    ```rs
    diesel setup
    ```

前面的命令在根目录中创建了一个`migrations`目录，其结构如下：

```rs
── migrations
│   └── 00000000000000_diesel_initial_setup
│       ├── down.sql
│       └── up.sql
```

当迁移升级时，会触发`up.sql`文件，而当迁移降级时，会触发`down.sql`文件。

1.  现在，我们需要创建我们的迁移来创建我们的待办事项。这可以通过命令我们的客户端使用以下命令生成迁移来完成：

    ```rs
    diesel migration generate create_to_do_items
    ```

一旦运行，我们应在控制台获得以下输出：

```rs
Creating migrations/2022-04-23-201747_create_to_do_items/up.sql
Creating migrations/2022-04-23-201747_create_to_do_items/down.sql
```

前面的命令在我们的迁移中给出了以下文件结构：

```rs
├── migrations
│   ├── 00000000000000_diesel_initial_setup
│   │   ├── down.sql
│   │   └── up.sql
│   └── 2022-04-23-201747_create_to_do_items
│       ├── down.sql
│       └── up.sql
```

从字面上看，使用`diesel`包，我们可能觉得不得不创建自己的 SQL 文件可能是不幸的。然而，这迫使我们养成良好的习惯。虽然允许包自动编写 SQL 代码更容易，但这会将我们的应用程序与数据库耦合起来。例如，我曾经在重构一个微服务系统时工作。我遇到的问题是，我正在使用一个 Python 包来管理所有数据库迁移。然而，我想更改服务器的代码。你不会感到惊讶，我将服务器从 Python 切换到 Rust。然而，因为迁移是由 Python 库自动生成的，我不得不构建辅助 Docker 容器，这些容器至今仍在每次发布新版本时启动，并在构建过程中将数据库模式复制到 Rust 应用程序中。这是很混乱的。这也解释了为什么当我作为金融科技研发软件工程师时，我们必须手动编写所有 SQL 代码的原因。

数据库与我们的应用程序是分开的。正因为如此，我们应该将它们隔离，这样我们就不会把手动编写 SQL 代码看作是一种阻碍。在学习一项将为你未来节省头痛的好技能时，要接受它。我们必须记住不要强迫一个工具适用于所有事情，它必须是适合正确工作的正确工具。在我们的`create to-do items`迁移文件夹中，我们在`up.sql`文件中定义我们的`to_do`表，使用以下 SQL 条目：

```rs
CREATE TABLE to_do (
  id SERIAL PRIMARY KEY,
  title VARCHAR NOT NULL,
  status VARCHAR NOT NULL,
  date timestamp NOT NULL DEFAULT NOW()
)
```

在前面的代码中，我们有一个唯一的`id`字段，然后是`title`和`status`。我们还添加了一个`date`字段，默认情况下，该字段包含将待办事项插入表中的当前时间。这些字段被包含在一个`CREATE TABLE`命令中。在我们的`down.sql`文件中，如果我们正在降级迁移，我们需要使用以下 SQL 命令来删除表：

```rs
DROP TABLE to_do
```

现在我们已经为我们的`up.sql`和`down.sql`文件编写了 SQL 代码，我们可以用以下图表来描述我们的代码做了什么：

![图 6.4 – up.sql 和 down.sql 脚本的效果](img/Figure_6.4_B18722.jpg)

图 6.4 – up.sql 和 down.sql 脚本的效果

1.  现在我们已经准备好了迁移，我们可以使用以下终端命令来运行它：

    ```rs
    diesel migration run
    ```

上述命令将运行迁移，创建`to_do`表。有时，我们可能在 SQL 中引入不同的字段类型。为了纠正这一点，我们可以在`up.sql`和`down.sql`文件中更改 SQL，并运行以下`redo`终端命令：

```rs
diesel migration redo
```

上述命令将运行`down.sql`文件，然后运行`up.sql`文件。

1.  接下来，我们可以在我们的数据库 Docker 容器中运行命令来检查我们的数据库是否具有我们定义的正确字段的`to_do`表。我们可以通过在数据库 Docker 容器上直接运行命令来实现这一点。我们可以使用以下终端命令进入容器，使用`username`用户名，同时指向`to_do`数据库：

    ```rs
    docker exec -it 5fdeda6cfe43 psql -U username to_do
    ```

必须注意的是，在前面的命令中，我的容器 ID 是`5fdeda6cfe43`，但你的容器 ID 将不同。如果你没有输入正确的容器 ID，你将不会在正确的数据库上运行正确的数据库命令。运行此命令后，我们得到一个具有以下提示的 shell 界面：

```rs
to_do=#
```

在前面的提示之后，当我们输入`\c`时，我们将连接到数据库。这通常会用一条语句表示我们现在已连接到`to_do`数据库并以`username`用户身份连接。最后，输入`\d`将列出关系，在终端中显示以下表：

```rs
Schema |            Name            |   Type   |  Owner 
--------+----------------------------+----------+----------
 public | __diesel_schema_migrations | table    | username
 public | to_do                      | table    | username
 public | to_do_id_seq               | sequence | username
```

从前面的表中，我们可以看到有一个迁移表来跟踪数据库的迁移版本。

1.  我们还有一个`to_do`表格和`to_do`项 ID 的序列。要检查模式，我们只需要输入`\d+ to_do`，在终端中会得到以下模式：

    ```rs
    Table "public.to_do"
    ```

    ```rs
     Column |            Type             | Collation |
    ```

    ```rs
    --------+-----------------------------+-----------+
    ```

    ```rs
     id     | integer                     |           |
    ```

    ```rs
     title  | character varying           |           |
    ```

    ```rs
     status | character varying           |           |
    ```

    ```rs
     date   | timestamp without time zone |           |
    ```

    ```rs
    | Nullable |              Default              | Storage  |
    ```

    ```rs
    +----------+-----------------------------------+----------+
    ```

    ```rs
    | not null | nextval('to_do_id_seq'::regclass) | plain    | 
    ```

    ```rs
    | not null |                                   | extended | 
    ```

    ```rs
    | not null || extended | 
    ```

    ```rs
    | not null | now()                             | plain    |            
    ```

    ```rs
    Indexes:
    ```

    ```rs
        "to_do_pkey" PRIMARY KEY, btree (id)
    ```

在前面的表中，我们可以看到我们的模式与我们设计的完全一致。我们在`date`列中得到了更多一点的信息，明确指出我们没有得到待办事项创建的时间。鉴于我们的迁移已经成功，我们应该在下一步探索迁移如何在数据库中显示。

1.  我们可以通过运行以下 SQL 命令来检查我们的迁移表：

    ```rs
    SELECT * FROM  __diesel_schema_migrations;
    ```

前面的命令给出了以下表作为输出：

```rs
    version     |           run_on           
----------------+----------------------------
 00000000000000 | 2022-04-25 23:10:07.774008
 20220424171018 | 2022-04-25 23:29:55.167845
```

如您所见，这些迁移对于调试很有用，因为有时我们可能会忘记在更新数据模型后运行迁移。为了进一步探索，我们可以在 Docker 容器外部使用以下命令回滚我们的最后一个迁移：

```rs
diesel migration revert
```

然后我们得到以下打印输出，告知我们已执行回滚操作：

```rs
Rolling back migration 2022-04-24-171018_create_to_do_items
```

现在我们已经回滚了迁移，我们的迁移表将如下所示：

```rs
    version     |           run_on 
----------------+----------------------------
 00000000000000 | 2022-04-25 23:10:07.774008
```

在前面的打印输出中，我们可以看到我们的最后一个迁移已被移除。因此，我们可以推断迁移表不是一个日志。它只跟踪当前活动的迁移。迁移的跟踪将帮助`diesel`客户端在运行迁移时执行正确的迁移。

在本节中，我们使用`diesel`客户端连接到 Docker 容器中的数据库。然后我们在环境文件中定义了数据库 URL。接下来，我们初始化了一些迁移并在数据库中创建了一个表。更好的是，我们直接连接到了 Docker 容器，在那里我们可以运行一系列命令来探索我们的数据库。现在，我们的数据库通过终端中的客户端完全交互式，我们可以开始构建我们的`to-do`项数据库模型，以便我们的 Rust 应用程序可以与数据库交互。

# 将我们的应用程序连接到 PostgreSQL

在上一节中，我们成功使用终端连接到 PostgreSQL 数据库。然而，我们现在需要我们的应用程序管理待办事项的数据库读写操作。在本节中，我们将我们的应用程序连接到在 Docker 中运行的数据库。为了连接，我们必须构建一个建立连接并返回它的函数。必须强调的是，管理数据库连接和配置有更好的方法，我们将在本章末尾介绍。现在，我们将实现最简单的数据库连接以使我们的应用程序运行。在 `src/database.rs` 文件中，我们使用以下代码定义函数：

```rs
use diesel::prelude::*;
use diesel::pg::PgConnection;
use dotenv::dotenv;
use std::env;
pub fn establish_connection() -> PgConnection {
    dotenv().ok();
    let database_url = env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");
    PgConnection::establish(&database_url)
        .unwrap_or_else(
        |_| panic!("Error connecting to {}", 
        database_url))
}
```

在前面的代码中，首先，你可能注意到 `use diesel::prelude::*;` 导入命令。此命令导入了连接、表达式、查询、序列化和结果结构体的一系列。一旦完成所需的导入，我们定义 `connection` 函数。首先，我们需要确保程序在无法使用 `dotenv().();` 命令加载环境时不会抛出错误。

完成此操作后，我们从环境变量中获取数据库 URL 并使用数据库 URL 的引用建立连接。然后我们 `unwrap` 结果，如果我们无法做到这一点，我们可能会恐慌地显示数据库 URL，因为我们想确保我们使用的是正确的 URL 和正确的参数。由于连接是函数中的最后一个语句，这就是返回的内容。

现在我们已经建立了自己的连接，我们需要定义模式。这将把 `to-do` 项的变量映射到数据类型。我们可以在 `src/schema.rs` 文件中使用以下代码定义我们的模式：

```rs
table! {
    to_do (id) {
        id -> Int4,
        title -> Varchar,
        status -> Varchar,
        date -> Timestamp,
    }
}
```

在前面的代码中，我们使用了 `diesel` 宏 `table!`，它指定了一个表的存在。这个映射很简单，我们将在未来的数据库查询和插入中引用列和表使用此模式。

现在我们已经建立了数据库连接并定义了模式，我们必须在 `src/main.rs` 文件中声明它们，如下所示：

```rs
#[macro_use] extern crate diesel;
extern crate dotenv;
use actix_web::{App, HttpServer};
use actix_service::Service;
use actix_cors::Cors;
mod schema;
mod database;
mod views;
mod to_do;
mod state;
mod processes;
mod json_serialization;
mod jwt;
```

在前面的代码中，我们的第一个导入还启用了过程宏。如果我们不使用 `#[macro_use]` 标签，那么我们无法在其他文件中引用我们的模式。我们的模式定义也无法使用表宏。我们还导入了 `dotenv` crate。我们保留了在 *第五章* 中创建的模块，*在浏览器中显示内容*。我们还定义了我们的模式和数据库模块。完成这些后，我们就需要开始构建我们的数据模型了。

## 创建我们的数据模型

我们将使用我们的数据模型在 Rust 中定义数据库数据的参数和行为。它们本质上充当数据库和 Rust 应用程序之间的桥梁，如下面的图所示：

![图 6.5 – 模型、模式和数据库之间的关系](img/Figure_6.5_B18722.jpg)

图 6.5 – 模型、模式和数据库之间的关系

在本节中，我们将定义待办事项的数据模型。然而，我们需要使我们的应用程序能够根据需要添加更多数据模型。为此，我们执行以下步骤：

1.  我们定义了一个新的待办事项数据模型结构体。

1.  然后，我们为新的待办事项结构体定义了一个构造函数。

1.  最后，我们定义了一个待办事项数据模型结构体。

在开始编写任何代码之前，我们在`src`目录中定义了以下文件结构，如下所示：

```rs
├── models
│   ├── item
│   │   ├── item.rs
│   │   ├── mod.rs
│   │   └── new_item.rs
│   └── mod.rs
```

在前述结构中，每个数据模型在`models`目录下都有一个目录。在该目录内，我们有两个文件定义了模型。一个用于新的插入，另一个用于管理数据库周围的数据。新的插入数据模型没有 ID 字段。

没有 ID 字段，因为数据库将为项目分配一个 ID；我们不会在之前定义它。然而，当我们与数据库中的项目交互时，我们将得到它们的 ID，我们可能想按 ID 进行筛选。因此，现有的数据项模型包含一个 ID 字段。我们可以在`new_item.rs`文件中用以下代码定义我们的新项目数据模型：

```rs
use crate::schema::to_do;
use chrono::{NaiveDateTime, Utc};
#[derive(Insertable)]
#[table_name="to_do"]
pub struct NewItem {
    pub title: String,
    pub status: String,
    pub date: NaiveDateTime
}
impl NewItem {
    pub fn new(title: String) -> NewItem {
        let now = Utc::now().naive_local();
        return NewItem{
            title, 
            status: String::from("PENDING"), 
            date: now
        }
    }
}
```

如前述代码所示，我们导入我们的表定义，因为我们将要引用它。然后我们使用`title`和`status`定义我们的新项目，它们应该是字符串类型。`chrono`库用于定义我们的`date`字段为`NaiveDateTime`。然后我们使用`diesel`宏定义表属于此结构体的`"to_do"`表。不要被这个定义使用引号的事实所迷惑。

如果我们不导入我们的模式，应用程序将无法编译，因为它将无法理解引用。我们还添加了另一个`diesel`宏，声明我们允许使用`Insertable`标签将数据插入数据库。如前所述，我们不会向此宏添加任何更多标签，因为我们只想让这个结构体插入数据。

我们还添加了一个`new`函数，使我们能够定义创建新结构的标准规则。例如，我们只将创建待处理状态的新项目。这减少了创建恶意状态的风险。如果我们想以后扩展，`new`函数可以接受一个`status`输入，并通过匹配语句运行它，如果状态不是我们愿意接受的状态之一，则抛出错误。我们还自动声明`date`字段是创建项目的日期。

有了这个想法，我们可以在`item.rs`文件中定义我们的项目数据模型，如下所示：

```rs
use crate::schema::to_do;
use chrono::NaiveDateTime;
#[derive(Queryable, Identifiable)]
#[table_name="to_do"]
pub struct Item {
    pub id: i32,
    pub title: String,
    pub status: String,
    pub date: NaiveDateTime
}
```

如前述代码所示，`NewItem`和`Item`结构体之间的唯一区别是我们没有构造函数；我们将`Insertable`标签替换为`Queryable`和`Identifiable`，并在结构体中添加了一个`id`字段。为了使这些对应用程序的其余部分可用，我们在`models/item/mod.rs`文件中定义了以下代码：

```rs
pub mod new_item;
pub mod item;
```

然后，在`models/mod.rs`文件中，我们用以下代码使`item`模块对其他模块和`main.rs`文件公开：

```rs
pub mod item;
```

接下来，我们在`main.rs`文件中用以下行代码定义我们的模型模块：

```rs
mod models;
```

现在我们可以在整个应用程序中访问我们的数据模型。我们还将读取和写入数据库的行为锁定。接下来，我们可以继续导入这些数据模型并在我们的应用程序中使用它们。

## 从数据库获取数据

当与数据库交互时，可能需要一些时间来习惯我们这样做的方式。不同的**对象关系映射器**（**ORMs**）和语言有不同的怪癖。虽然基本原理相同，但这些 ORM 的语法可能会有很大差异。因此，仅仅使用 ORM 计时并不能使您更有信心，也不能解决更复杂的问题。我们可以从最简单的机制开始，即从表中获取所有数据。

为了探索这一点，我们可以从`to_do`表获取所有项目，并在每个视图的末尾返回它们。我们在*第四章*，*处理 HTTP 请求*中定义了这种机制。就我们的应用程序而言，您会记得我们有一个`get_state`函数，它为我们的前端打包待办事项。这个`get_state`函数位于`src/json_serialization/to_do_items.rs`文件中的`ToDoItems`结构体中。最初，我们必须用以下代码导入所需的项：

```rs
use crate::diesel;
use diesel::prelude::*;
use crate::database::establish_connection;
use crate::models::item::item::Item;
use crate::schema::to_do;
```

在前面的代码中，我们导入了`diesel`包和宏，这使我们能够构建数据库查询。然后我们导入`establish_connection`函数以连接到数据库，然后导入模式和数据库模型以进行查询和处理数据。然后我们可以用以下代码重构我们的`get_state`函数：

```rs
pub fn get_state() -> ToDoItems {
    let connection = establish_connection();
    let mut array_buffer = Vec::new();
    let items = to_do::table
        .order(to_do::columns::id.asc())
        .load::<Item>(&connection).unwrap();
    for item in items {
        let status = 
            TaskStatus::new(&item.status.as_str());
        let item = to_do_factory(&item.title, status);
        array_buffer.push(item);
    }                            
    return ToDoItems::new(array_buffer)
}
```

在前面的代码中，我们首先建立连接。一旦建立了数据库连接，我们就从我们的表获取并基于它构建一个数据库查询。查询的第一部分定义了顺序。正如我们所见，我们的表也可以传递列的引用，这些列也有它们自己的函数。

我们首先定义将用于加载数据的结构体，并传入连接的引用。因为宏在`load`中定义了结构体，如果我们把`NewItem`结构体传递给`load`函数，我们会得到一个错误，因为`Queryable`宏没有为该结构体启用。

还可以注意到，`load`函数在其函数括号之前有一个前缀，其中参数通过`load::<Item>(&connection)`传递。这个前缀概念在*第一章*，*Rust 快速入门*，*使用特性行验证*部分，*步骤 4*中有所介绍，但到目前为止，您可以使用以下代码：

```rs
fn some_function<T: SomeType>(some_input: &T) -> () {
    . . .
}
```

传递到前缀`<Item>`的类型定义了接受的输入类型。这意味着当编译器运行时，为传递给我们的`load`函数的每个类型编译一个单独的函数。

然后，我们直接解包 `load` 函数，得到数据库中的项目向量。有了数据库中的数据，我们通过构造项目结构体并将它们追加到我们的缓冲区中来进行循环。一旦完成，我们从缓冲区中构造 `ToDoItems` JSON 架构并返回它。现在我们已经实施了这一变更，所有我们的视图都将直接从数据库返回数据。如果我们运行这个程序，将不会显示任何项目。如果我们尝试创建任何项目，它们将不会显示。然而，尽管它们没有被显示，我们所做的是从数据库中获取数据，并以我们想要的 JSON 结构进行序列化。这是从数据库返回数据并以标准方式将其返回给请求者的基础。这是在 Rust 中构建的 API 的骨架。因为我们不再依赖于从 JSON 状态文件中读取项目时读取，我们可以从 `src/json_serialization/to_do_items.rs` 文件中删除以下导入：

```rs
use crate::state::read_file;
use serde_json::value::Value;
use serde_json::Map;
```

我们移除了前面的导入，因为我们没有重构任何其他端点。因此，`create` 端点将正确触发；然而，该端点只是在 JSON 状态文件中创建项目，而 `return_state` 已经不再读取这些项目。为了再次启用创建功能，我们必须重构 `create` 端点以将新项目插入到数据库中。

# 插入到数据库中

在本节中，我们将构建并创建一个视图来创建待办事项。如果我们还记得创建待办事项的规则，我们不想创建重复的待办事项。这可以通过唯一约束来实现。然而，目前保持简单是好的。相反，我们将基于传递给视图的标题对数据库执行一个筛选。然后我们进行检查，如果没有返回结果，我们将向数据库中插入一个新的 `to-do` 事项。我们通过重构 `views/to_do/create.rs` 文件中的代码来完成这项工作。首先，我们重新配置导入，如下面的代码所示：

```rs
use crate::diesel;
use diesel::prelude::*;
use actix_web::HttpResponse;
use actix_web::HttpRequest;
use crate::json_serialization::to_do_items::ToDoItems;
use crate::database::establish_connection;
use crate::models::item::new_item::NewItem;
use crate::models::item::item::Item;
use crate::schema::to_do;
```

在前面的代码中，我们导入了必要的 `diesel` 导入以执行上一节中描述的查询。然后，我们导入了 `actix-web` 结构体，以便视图处理请求并定义结果。接着，我们导入了与数据库交互的数据库结构体和函数。现在我们已经拥有了所有这些，我们可以开始工作在 `create` 视图上了。在我们的 `pub async fn create` 函数内部，我们首先从请求中获取 `to-do` 项目的标题的两个引用：

```rs
pub async fn create(req: HttpRequest) -> HttpResponse {
    let title: String = req.match_info().get("title"
    ).unwrap().to_string();
    . . .
}
```

一旦我们从前面代码中的 URL 中提取了标题，我们就建立数据库连接，并使用该连接对我们的表进行数据库调用，如下面的代码所示：

```rs
let connection = establish_connection();
let items = to_do::table
    .filter(to_do::columns::title.eq(&title.as_str()))
    .order(to_do::columns::id.asc())
    .load::<Item>(&connection)
    .unwrap();
```

如前述代码所示，查询基本上与上一节的查询相同。然而，我们有一个`filter`部分，它引用我们的`title`列，必须等于我们的`title`。如果正在创建的项目确实是新的，则不会创建任何项目；因此，结果的长度将为零。随后，如果长度为零，我们应该创建一个`NewItem`数据模型，然后将其插入数据库，以便在函数结束时返回状态，如下面的代码所示：

```rs
if items.len() == 0 {
    let new_post = NewItem::new(title);
    let _ = 
     diesel::insert_into(to_do::table).values(&new_post)
     .execute(&connection);
}
return HttpResponse::Ok().json(ToDoItems::get_state())
```

我们可以看到 Diesel 有一个`insert`函数，它接受表和我们对构建的数据模型的引用。因为我们已经切换到数据库存储，所以我们可以删除以下导入，因为它们不再需要：

```rs
use serde_json::value::Value;
use serde_json::Map;
use crate::state::read_file;
use crate::processes::process_input;
use crate::to_do::{to_do_factory, enums::TaskStatus};
```

现在，使用我们的应用，我们将能够创建待办事项，然后看到这些事项在我们的应用前端弹出。因此，我们可以看到我们的`create`和`get state`函数正在工作；它们正在与我们的数据库交互。如果你遇到麻烦，一个常见的错误是忘记启动我们的`docker-compose`。(*注意：请记住做这件事，否则，应用将无法连接到数据库，因为它没有运行*)。然而，我们无法将待办事项的状态编辑为`完成`。我们将不得不在数据库中编辑我们的数据来完成这个操作。

## 编辑数据库

当我们编辑我们的数据时，我们将从数据库中获取数据模型，然后使用 Diesel 的数据库调用函数来编辑条目。为了使我们的`edit`函数与数据库交互，我们可以在`views/to_do/edit.rs`文件中编辑我们的视图。我们首先重构导入，如下面的代码所示：

```rs
use crate::diesel;
use diesel::prelude::*;
use actix_web::{web, HttpResponse};
use crate::json_serialization::{to_do_item::ToDoItem, 
                                to_do_items::ToDoItems};
use crate::jwt::JwToken;
use crate::database::establish_connection;
use crate::schema::to_do;
```

如前述代码所示，出现了一个模式。这个模式是，我们导入依赖项来处理身份验证、数据库连接和模式，然后构建一个执行我们想要的操作的函数。我们已经覆盖了导入及其含义。在我们的`edit`视图中，我们这次必须只获取一个对标题的引用，如下面的代码所示：

```rs
pub async fn edit(to_do_item: web::Json<ToDoItem>, 
    token: JwToken) -> HttpResponse {
    let connection = establish_connection();
    let results = to_do::table.filter(to_do::columns::title
        .eq(&to_do_item.title));

    let _ = diesel::update(results)
        .set(to_do::columns::status.eq("DONE"))
        .execute(&connection);
    return HttpResponse::Ok().json(ToDoItems::get_state())
}
```

在前述代码中，我们可以看到我们在不加载数据的情况下对数据库执行了过滤。这意味着`results`变量是一个`UpdateStatement`结构体。然后我们使用这个`UpdateStatement`结构体来在数据库中更新项目为`完成`。鉴于我们不再使用 JSON 文件，我们可以在`views/to_do/edit.rs`文件中删除以下导入：

```rs
use crate::processes::process_input;
use crate::to_do::{to_do_factory, enums::TaskStatus};
use serde_json::value::Value;
use serde_json::Map;
use crate::state::read_file;
```

在前述代码中，我们可以看到我们调用了`update`函数，并用从数据库获取的结果填充它。然后我们将`status`列设置为`完成`，然后使用连接的引用来执行。现在我们可以使用这个来编辑我们的`to-do`事项，使它们移动到完成列表。然而，我们无法删除它们。为了做到这一点，我们将不得不重构我们的最终端点，以完全重构我们的应用以连接到数据库。

## 删除数据

在删除数据时，我们将采取与上一节编辑时相同的方法。我们将从数据库中获取一个项目，通过 Diesel 的 `delete` 函数传递，然后返回状态。目前，我们应该对这种方法感到舒适，因此建议你在 `views/to_do/delete.rs` 文件中自己尝试实现它。以下是导入部分的代码：

```rs
use crate::diesel;
use diesel::prelude::*;
use actix_web::{web, HttpResponse};
use crate::database::establish_connection;
use crate::schema::to_do;
use crate::json_serialization::{to_do_item::ToDoItem, 
                                to_do_items::ToDoItems};
use crate::jwt::JwToken;
use crate::models::item::item::Item;
```

在前面的代码中，我们依赖于 Diesel 包和预导入，这样我们就可以使用 Diesel 宏。没有 `prelude`，我们就无法使用模式。然后我们导入返回数据给客户端所需的 Actix Web 结构体。然后导入我们构建来管理待办事项数据的包。对于 `delete` 函数，代码如下：

```rs
pub async fn delete(to_do_item: web::Json<ToDoItem>, 
                    token: JwToken) -> HttpResponse {
    let connection = establish_connection();
    let items = to_do::table
        .filter(to_do::columns::title.eq(
                &to_do_item.title.as_str()))
        .order(to_do::columns::id.asc())
        .load::<Item>(&connection)
        .unwrap();
    let _ = diesel::delete(&items[0]).execute(&connection);
    return HttpResponse::Ok().json(ToDoItems::get_state())
}
```

为了进行质量控制，让我们按照以下步骤进行：

1.  在文本输入框中输入 `buy canoe` 并点击 **创建** 按钮。

1.  在文本输入框中输入 `go dragon boat racing` 并点击 **创建** 按钮。

1.  点击 **buy canoe** 项上的 **编辑** 按钮。完成此操作后，我们应该在前端看到以下输出：

![图 6.6 – 预期输出](img/Figure_6.6_B18722.jpg)

图 6.6 – 预期输出

在前面的图中，我们已经购买了我们的独木舟，但还没有参加龙舟比赛。现在，我们的应用程序与我们的 PostgreSQL 数据库无缝工作。我们可以创建、编辑和删除我们的待办事项。由于在之前的章节中定义了结构，移除数据库的 JSON 文件机制并没有花费太多工作。数据处理和返回请求已经就绪。当我们现在运行应用程序时，你可能已经意识到在编译时会有以下输出：

```rs
warning: function is never used: `read_file`
  --> src/state.rs:17:8
   |
17 | pub fn read_file(file_name: &str) -> Map<String, Value> {
. . .
warning: function is never used: `process_pending`
  --> src/processes.rs:18:4
   |
18 | fn process_pending(item: Pending, command: String, 
                        state: &Map<String, Value>) {
```

根据前面的输出，所发生的情况是我们不再使用 `src/state.rs` 文件或 `src/processes.rs` 文件。这些文件不再被使用，因为 `src/database.rs` 文件现在正在使用真实数据库来管理我们的存储、删除和编辑持久数据。我们还使用实现了 Diesel 的 `Queryable` 和 `Identifiable` 特性的数据库模型结构体将数据库中的数据处理到结构体中。因此，我们可以删除 `src/state.rs` 和 `src/database.rs` 文件，因为它们不再需要。我们也不再需要 `src/to_do/traits` 模块，因此也可以将其删除。接下来，我们可以删除所有对特质的引用。删除引用实际上意味着从 `src/to_do/mod.rs` 文件中删除特质，并从 `to_do` 模块的结构体中删除这些特质的导入和实现。例如，要从 `src/to_do/structs/pending.rs` 文件中删除特质，我们只需删除以下导入：

```rs
use super::super::traits::get::Get;
use super::super::traits::edit::Edit;
use super::super::traits::create::Create;
```

然后，我们可以删除以下特质的实现：

```rs
impl Get for Pending {}
impl Edit for Pending {}
impl Create for Pending {}
```

我们还必须在`src/to_do/structs/done.rs`文件中移除特性。我们现在有机会欣赏我们结构良好、独立代码的回报。我们可以轻松地插入不同的持久化存储方法，因为我们只需删除几个文件和特性即可简单地移除现有的存储方法。然后我们只是移除了特性的实现。就是这样。我们不必深入函数和修改代码行。因为我们的读取功能在特性中，通过移除特性的实现，我们只需移除特性的实现就能完全切断应用程序。以这种方式结构代码在将来真的很划算。在过去，我不得不从一个服务器中提取代码到另一个服务器中，以重构微服务或升级或切换方法，例如存储选项。拥有独立的代码使重构变得更容易、更快、更不容易出错。

我们的应用程序现在与数据库完全兼容。然而，我们可以改进我们数据库的实现。在我们这样做之前，然而，我们应该在下一节重构我们的配置代码。

# 配置我们的应用程序

目前，我们正在将数据库 URL 存储在`.env`文件中。这很好，但我们也可以使用`yaml`配置文件。在开发网络服务器时，它们将在不同的环境中运行。例如，目前，我们正在在我们的本地机器上运行我们的 Rust 服务器。然而，稍后我们将把服务器打包成我们自己的 Docker 镜像并在云上部署它。由于微服务基础设施，我们可能会使用在一个集群中构建的服务器，并将其插入到连接到不同数据库的不同集群中。因此，定义所有内外部流量的配置文件变得至关重要。当我们部署我们的服务器时，我们可以向文件存储发出请求，例如 AWS S3，以获取服务器的正确配置文件。必须注意的是，环境变量可以优先用于部署，因为它们可以被传递到容器中。我们将在第十三章“干净 Web 应用程序仓库的最佳实践”中介绍如何使用环境变量配置 Web 应用程序。现在，我们将专注于使用配置文件。我们还需要使我们的服务器在加载配置文件方面具有灵活性。例如，我们不应该需要将配置文件放在特定的目录中，并具有特定的名称才能加载到我们的服务器中。我们应该为正确的上下文正确命名我们的配置文件，以降低错误配置文件被加载到服务器中的风险。我们的文件路径可以在启动服务器时传递。因为我们正在使用`yaml`文件，所以我们需要在`Cargo.toml`文件中定义`serde_yaml`依赖项，如下所示：

```rs
serde_yaml = "0.8.23"
```

现在我们可以读取`yaml`文件了，我们可以构建自己的配置模块，该模块将`yaml`文件值加载到 HashMap 中。这可以通过一个结构体来完成；因此，我们应该将配置模块放入一个文件中，即`src/config.rs`文件。首先，我们使用以下代码导入所需的模块：

```rs
use std::collections::HashMap;
use std::env;
use serde_yaml;
```

在前面的代码中，我们使用`env`来捕获传递给程序的环境变量，使用`HashMap`来存储配置文件中的数据，并使用`serde_yaml`包来处理配置文件中的`yaml`值。然后，我们使用以下代码定义包含我们配置数据的结构体：

```rs
pub struct Config {
    pub map: HashMap<String, serde_yaml::Value>
}
```

在前面的代码中，我们可以看到我们的数据键的值是`String`，而属于这些键的值是`yaml`值。然后我们为我们的结构体构建一个构造函数，它接受传递给程序的最后一个参数，根据传递给程序的文件路径打开文件，并使用以下代码将文件中的数据加载到我们的`map`字段中：

```rs
impl Config {
    pub fn new() -> Config {
        let args: Vec<String> = env::args().collect();
        let file_path = &args[args.len() - 1];
        let file = std::fs::File::open(file_path).unwrap();
        let map: HashMap<String, serde_yaml::Value> = 
            serde_yaml::
                 from_reader(file).unwrap();
        return Config{map}
    }
}
```

现在已经定义了`config`结构体，我们可以在`src/main.rs`文件中使用以下代码行来定义我们的`config`模块：

```rs
mod config;
```

然后，我们必须重构我们的`src/database.rs`文件，以便从`yaml`配置文件中加载。我们重构的导入形式如下：

```rs
use diesel::prelude::*;
use diesel::pg::PgConnection;
use crate::config::Config;
```

我们可以看到，所有对`env`的引用都已移除，因为现在这已在我们的`config`模块中处理。然后我们加载我们的文件，获取我们的`DB_URL`键，并直接展开与`DB_URL`键关联的变量的结果，将`yaml`值转换为字符串并直接展开转换的结果。我们直接展开获取和转换函数，因为如果它们失败，我们无论如何都无法连接到数据库。如果我们无法连接，我们希望尽快知道错误，并显示一个清晰的错误消息，说明错误发生的位置。现在，我们可以将我们的数据库 URL 移入 Rust 应用程序根目录下的`config.yml`文件中，内容如下：

```rs
DB_URL: postgres://username:password@localhost:5433/to_do
```

接下来，我们可以使用以下命令运行我们的应用程序：

```rs
cargo run config.yml
```

`config.yml`文件是配置文件的路径。如果你运行`docker-compose`和前端，你会看到我们的配置文件中的数据库 URL 正在被加载，并且我们的应用程序已连接到数据库。然而，这个数据库连接有一个问题。每次我们执行`establish_connection`函数时，我们都会与我们的数据库建立连接。这可以工作；然而，这不是最优的。在下一节中，我们将通过数据库连接池来提高数据库连接的效率。

# 构建数据库连接池

在本节中，我们将创建一个数据库连接池。数据库连接池是一定数量的数据库连接。当我们的应用程序需要数据库连接时，它会从池中取出连接，当应用程序不再需要连接时，它会将连接放回池中。如果池中没有剩余的连接，应用程序将等待直到有可用的连接，如下面的图示所示：

![图 6.7 – 限制为三个连接的数据库连接池](img/Figure_6.7_B18722.jpg)

图 6.7 – 限制为三个连接的数据库连接池

在我们重构数据库连接之前，我们需要在我们的`Cargo.toml`文件中安装以下依赖项：

```rs
lazy_static = "1.4.0"
```

然后我们使用以下代码在`src/database.rs`文件中定义我们的导入：

```rs
use actix_web::dev::Payload;
use actix_web::error::ErrorServiceUnavailable;
use actix_web::{Error, FromRequest, HttpRequest};
use futures::future::{Ready, ok, err};
use lazy_static::lazy_static;
use diesel::{
   r2d2::{Pool, ConnectionManager, PooledConnection},
   pg::PgConnection,
};
use crate::config::Config;
```

从前面代码中定义的导入中，你认为我们在编写新的数据库连接时会做什么？到目前为止，这是一个停下来思考你可以用这些导入做什么的好时机。

如果你还记得，第一个导入块将在请求击中视图之前建立数据库连接。第二个导入块使我们能够定义我们的数据库连接池。最后的`Config`参数是为了获取连接的数据库 URL。现在我们的导入已经完成，我们可以使用以下代码定义连接池结构体：

```rs
type PgPool = Pool<ConnectionManager<PgConnection>>;
pub struct DbConnection {
   pub db_connection: PgPool,
}
```

在前面的代码中，我们声明我们的`PgPool`结构体是一个连接管理器，它管理池内的连接。然后我们使用以下代码构建我们的连接，这是一个静态引用：

```rs
lazy_static! {
    pub static ref DBCONNECTION: DbConnection = {
        let connection_string = 
            Config::new().map.get("DB_URL").unwrap()
            .as_str().unwrap().to_string();
      DbConnection {
          db_connection: PgPool::builder().max_size(8)
          .build(ConnectionManager::new(connection_string))
          .expect("failed to create db connection_pool")
       }
   };
}
```

在前面的代码中，我们从配置文件中获取 URL 并构建一个返回并因此分配给`DBCONNECTION`静态引用变量的连接池。因为这是一个静态引用，所以我们的`DBCONNECTION`变量的生命周期与服务器匹配。现在我们可以重构`establish_connection`函数，使其从数据库连接池中获取连接，以下是一段代码：

```rs
pub fn establish_connection() -> 
  PooledConnection<ConnectionManager<PgConnection>>{
    return DBCONNECTION.db_connection.get().unwrap()
}
```

在前面的代码中，我们可以看到我们返回了一个`PooledConnection`结构体。然而，我们不想每次需要时都调用`establish_connection`函数。我们还想在 HTTP 请求击中视图之前拒绝请求，如果我们无法建立连接，因为我们不想加载无法处理的视图。就像我们的`JWToken`结构体一样，我们可以创建一个结构体来创建数据库连接并将该数据库连接传递到视图中。我们的结构体有一个字段，即一个池连接，如下所示：

```rs
pub struct DB {
   pub connection: PooledConnection<ConnectionManager<PgConnection>>
}
```

使用这个`DB`结构体，我们可以像处理我们的`JWToken`结构体一样实现`FromRequest`特性，以下是一段代码：

```rs
impl FromRequest for DB {
  type Error = Error;
  type Future = Ready<Result<DB, Error>>;
  fn from_request(_: &HttpRequest, _: &mut Payload) -> 
      Self::Future{
      match DBCONNECTION.db_connection.get() {
         Ok(connection) => {
            return ok(DB{connection})
         },
         Err(_) => {
            return err(ErrorServiceUnavailable(
            "could not make connection to database"))
         }
      }
  }
}
```

在这里，我们并没有直接解包获取数据库连接。相反，如果连接时出现错误，我们返回一个带有有用信息的错误。如果我们的连接成功，我们随后返回它。我们可以在我们的视图中实现这一点。为了避免重复代码，我们只需使用`edit`视图，但我们应该将这种方法应用到所有我们的视图中。首先，我们定义以下导入：

```rs
use crate::diesel;
use diesel::prelude::*;
use actix_web::{web, HttpResponse};
use crate::json_serialization::{to_do_item::ToDoItem, 
                                to_do_items::ToDoItems};
use crate::jwt::JwToken;
use crate::schema::to_do;
use crate::database::DB;
```

在前面的代码中，我们可以看到我们导入了`DB`结构体。现在，我们的`edit`视图应该如下所示：

```rs
pub async fn edit(to_do_item: web::Json<ToDoItem>, 
    token: JwToken, db: DB) -> HttpResponse {
    let results = to_do::table.filter(to_do::columns::title
        .eq(&to_do_item.title));

    let _ = diesel::update(results)
        .set(to_do::columns::status.eq("DONE"))
        .execute(&db.connection);
    return HttpResponse::Ok().json(ToDoItems::get_state())
}
```

在前面的代码中，我们可以看到我们直接引用了`DB`结构体的`connection`字段。事实上，我们可以通过将`DB`结构体传递到视图中，就像我们的认证令牌一样，简单地获取一个池化连接到我们的视图中。

# 摘要

在本章中，我们构建了一个开发环境，我们的应用程序可以通过 Docker 与数据库进行交互。一旦完成这项工作，我们就探索了容器和镜像的列表，以检查我们的系统运行情况。然后，我们使用`diesel`包创建了迁移。之后，我们安装了`diesel`客户端，并将数据库 URL 定义为环境变量，以便我们的 Rust 应用程序和迁移可以直接连接到数据库容器。

然后，我们运行了迁移，并定义了在迁移运行时将触发的 SQL 脚本，然后运行了它们。一旦完成所有这些，我们再次检查数据库容器，以查看迁移是否实际上已执行。然后，我们在 Rust 中定义了数据模型，并重构了我们的 API 端点，以便它们在数据库上执行`get`、`edit`、`create`和`delete`操作，以跟踪待办事项。

我们在这里所做的是升级了我们的数据库存储系统。我们离拥有一个生产就绪的系统又近了一步，因为我们不再依赖于 JSON 文件来存储我们的数据。你现在拥有了执行数据库管理任务所需的技能，这些任务使你能够管理更改、凭证/访问和模式。我们还对数据库执行了所有必要的操作，以便运行一个创建、获取、更新和删除数据的应用程序。这些技能可以直接转移到你希望在 Rust Web 项目中进行的任何其他项目中。然后，我们通过使数据库连接受连接池限制来增强我们新开发的数据库技能。最后，通过实现`FromRequest`特质，使实现数据库连接结构体变得简单，这样其他开发者只需通过将结构体传递到视图中作为参数即可实现我们的连接。如果无法建立数据库连接，视图也得到了保护。

在下一章中，我们将在此基础上构建用户身份验证系统，以便我们可以在访问应用程序时创建用户和检查凭证。我们将结合使用数据库、从头部提取数据、浏览器存储和路由，以确保用户必须登录才能访问待办事项。

# 问题

1.  使用数据库而不是 JSON 文件有哪些优势？

1.  你如何创建迁移？

1.  我们如何检查迁移？

1.  如果我们想在 Rust 中创建一个具有姓名和年龄的用户数据模型，我们应该怎么做？

1.  什么是连接池，为什么我们应该使用它？

# 答案

1.  数据库在同时进行多读和多写方面具有优势。数据库还会在插入之前检查数据，看它是否处于正确的格式，并且我们可以使用关联表进行高级查询。

1.  我们安装`diesel`客户端并在`.env`文件中定义数据库 URL。然后我们使用客户端创建迁移，并编写所需的模式以供迁移使用。接着我们运行迁移。

1.  我们使用数据库的容器 ID 来访问容器。然后我们列出表；如果所需的表存在，那么这是一个迁移已运行的迹象。我们还可以检查数据库中的迁移表，以查看它最后一次运行的时间。

1.  我们定义一个`NewUser`结构体，其中姓名是一个字符串，年龄是一个整数。然后我们创建一个`User`结构体，具有相同的字段和一个额外的整数字段，即 ID。

1.  连接池会收集一定数量的连接，这些连接连接到数据库。然后我们的应用程序将这些连接传递给需要它们的线程。这样可以保持连接到数据库的连接数量有限，以避免数据库过载。
