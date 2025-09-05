# *第九章*：为 Rust 结构化 Python Flask 应用程序

在上一章中，我们成功使用 Rust 解决了一个实际问题。然而，我们也学到了一个重要的教训，那就是良好的代码实现，例如添加向量或合并数据框，以及第三方模块，如 `NumPy`，可以比糟糕的自编 Rust 解决方案表现得更好。然而，我们知道在实现与实现之间比较，Rust 的速度远快于 Python。我们已经了解了如何将 Rust 与标准 Python 脚本融合。然而，Python 的用途远不止运行脚本。Python 的一个流行用途是用于 Web 应用程序。

在本章中，我们将使用 NGINX、数据库和由 `Celery` 包实现的信使总线来构建一个 Flask Web 应用程序。这个信使总线将允许我们的应用程序在处理重任务的同时返回 Web HTTP 请求。Web 应用程序和信使总线将被封装在 Docker 容器中，并部署到 `docker-compose`。然而，如果需要，我们也可以将应用程序部署到云平台。

在本章中，我们将涵盖以下主题：

+   构建基本的 Flask 应用程序

+   定义数据库访问层

+   构建消息总线

本章将使我们能够为具有各种功能和服务的可部署 Python Web 应用程序打下基础。这个基础使我们能够发现如何将 Rust 与封装在 Docker 容器中的 Python Web 应用程序融合。

# 技术要求

本章的代码和数据可以在[`github.com/PacktPublishing/Speed-up-your-Python-with-Rust/tree/main/chapter_nine`](https://github.com/PacktPublishing/Speed-up-your-Python-with-Rust/tree/main/chapter_nine)找到。

此外，我们将在 Docker 的基础上使用 `docker-compose` 来编排我们的 Docker 容器。可以通过遵循[`docs.docker.com/compose/install/`](https://docs.docker.com/compose/install/)中的说明来安装。

在本章中，我们将构建一个 Docker 容器中的 Flask 应用程序，该应用程序可在 GitHub 仓库[`github.com/maxwellflitton/fib-flask`](https://github.com/maxwellflitton/fib-flask)中找到。

# 构建基本的 Flask 应用程序

在我们开始向应用程序添加任何其他功能，如数据库之前，我们必须确保我们可以运行一个基本的 Flask 应用程序，并包含我们所需的一切。这个应用程序将接受一个数字并返回一个斐波那契数。此外，我们还需要确保如果我们要部署它，这个应用程序可以在自己的 Docker 容器中运行。在本节结束时，我们的应用程序应该具有以下结构：

```rs
├── deployment
```

```rs
│   ├── docker-compose.yml
```

```rs
│   └── nginx
```

```rs
│       ├── Dockerfile
```

```rs
│       └── nginx.conf
```

```rs
├── src
```

```rs
│   ├── Dockerfile
```

```rs
│   ├── __init__.py
```

```rs
│   ├── app.py
```

```rs
│   ├── fib_calcs
```

```rs
│   │   ├── __init__.py
```

```rs
│   │   └── fib_calculation.py
```

```rs
│   └── requirements.txt
```

在这里，你可以看到应用程序位于`src`目录中。在运行我们的应用程序时，我们必须确保`PYTHONPATH`路径设置为`src`。我们部署所需的代码位于`deployment`目录中。为了构建一个可以在 Docker 中运行的应用程序，请执行以下步骤：

1.  为我们的应用程序构建一个入口点。

1.  构建斐波那契数计算模块。

1.  为我们的应用程序构建一个 Docker 镜像。

1.  构建我们的 NGINX 服务。

完成所有这些步骤后，我们将拥有一个可以在服务器上运行的基本 Flask 应用程序。现在，让我们在以下子节中详细探讨这些步骤。

## 构建应用程序的入口点

这里是执行步骤：

1.  在我们可以构建入口点之前，我们需要使用以下命令安装 Flask 模块：

    ```rs
    pip install flask
    ```

1.  一旦完成，我们就有了创建基本 Flask 应用程序所需的一切，通过在`src/app.py`文件中定义入口点来创建它，如下所示：

    ```rs
    from flask import Flask
    app = Flask(__name__)
    @app.route("/")
    def home():
        return "home for the fib calculator"
    if __name__ == "__main__":
        app.run(use_reloader=True, port=5002, \
          threaded=True)
    ```

在这里，观察我们可以使用装饰器定义一个基本路由。我们可以通过运行`src/app.py`脚本来运行我们的应用程序；这将使我们的服务器在本地运行，使我们能够访问我们定义的所有路由。将`http://127.0.0.1:5002` URL 传递到我们的浏览器中，将给出以下视图：

![图 9.1 – 我们本地 Flask 服务器的主视图](img/Figure_9.01_B17720.jpg)

图 9.1 – 我们本地 Flask 服务器的主视图

现在我们基本的服务器正在运行，我们可以继续构建斐波那契数计算器模块。

## 构建我们的斐波那契数计算器模块

这里是执行步骤：

1.  在这个例子中，我们的应用程序很简单。因此，我们可以在一个文件中的一个类中定义我们模块的功能。我们使用以下代码在`src/fib_calcs/fib_calculation.py`文件中定义它：

    ```rs
    class FibCalculation:
        def __init__(self, input_number: int) -> None:
            self.input_number: int = input_number
            self.fib_number: int = self.recur_fib(
                n=self.input_number
            )
        @staticmethod
        def recur_fib(n: int) -> int:
            if n <= 1:
                return n
            else:
                return (FibCalculation.recur_fib(n - 1) +
                        FibCalculation.recur_fib(n - 2))
    ```

    在这里，请注意，我们的类仅仅接受一个输入数字，并自动将计算出的斐波那契数填充到`self.fib_number`属性中。

1.  一旦完成，我们就可以定义一个视图，它通过 URL 接受一个整数，将其传递给我们的`FibCalculation`类，并使用以下代码将计算出的斐波那契数作为字符串返回给用户在`src/app.py`文件中：

    ```rs
    from fib_calcs.fib_calculation import FibCalculation
    . . .
    @app.route("/calculate/<int:number>")
    def calculate(number):
        calc = FibCalculation(input_number=number)
        return f"you entered {calc.input_number} " \
               f"which has a Fibonacci number of " \
               f"{calc.fib_number}"
    ```

1.  重新运行我们的服务器，并将`http://127.0.0.1:5002/calculate/10` URL 传递到我们的浏览器中，将给出以下视图：

![图 9.2 – 计算我们本地 Flask 服务器的视图](img/Figure_9.02_B17720.jpg)

图 9.2 – 计算我们本地 Flask 服务器的视图

现在，我们的应用程序执行其预期功能：它根据输入计算斐波那契数。Flask 的视图还有更多功能；然而，这本书不是一本网络开发教科书。如果您想学习如何构建更全面的 API 端点，我们建议您查看 Flask API 和`Marshmallow`包。这两个包的参考信息可以在*进一步阅读*部分找到。现在，我们需要使我们的应用程序可部署，以便我们可以在各种设置中使用它作为下一步。

## 为我们的应用程序构建 Docker 镜像

为了使我们的应用程序可用，我们必须构建一个接受请求的应用程序 Docker 镜像。然后，我们必须使用另一个充当入口的容器调用来保护它。NGINX 执行负载均衡、缓存、流式传输和流量重定向。我们的应用程序将使用 Gunicorn 包运行，该包本质上同时运行多个应用程序工作进程。对于每个请求，NGINX 询问请求应该发送到哪个 Gunicorn 工作进程，并将其重定向，如下面的图所示：

![图 9.3 – 我们应用程序请求的流程]

](img/Figure_9.03_B17720.jpg)

图 9.3 – 我们应用程序请求的流程

我们可以通过执行以下步骤来实现前面图中定义的布局：

1.  在我们构建 Docker 镜像之前，我们必须确保处理我们应用程序的要求。因此，我们必须使用以下命令使用`pip`安装 Gunicorn：

    ```rs
    pip install gunicorn
    ```

1.  我们必须确保我们处于`src`目录，因为我们将要使用以下命令将所有应用程序依赖项放入一个名为`requirements.txt`的文件中：

    ```rs
     pip freeze > requirements.txt
    ```

    这为我们提供了一个包含所有应用程序运行所需依赖项列表的文本文件。目前，我们只需要 Flask 和 Gunicorn。

1.  使用这个方法，我们可以开始编写我们的 Dockerfile，以便构建我们应用程序的应用程序镜像。首先，在我们的`src/Dockerfile`文件中，我们应该使用以下代码定义所需的操作系统：

    ```rs
    FROM python:3.6.13-stretch
    ```

    这意味着我们的镜像正在运行一个安装了 Python 的简化版 Linux。

1.  现在我们有了正确的操作系统，我们应该定义我们的应用程序目录，并使用以下代码将所有应用程序文件复制到镜像中：

    ```rs
    # Set the working directory to /app
    WORKDIR /app
    # Copy the current directory contents into the 
      container at /app
    ADD . /app
    ```

1.  现在我们所有的应用程序文件都在镜像中，我们安装系统更新，然后安装`python-dev`包。这是为了我们可以包含以下代码中的扩展：

    ```rs
    RUN apt-get update -y
    RUN apt-get install -y python3-dev python-dev gcc
    ```

    这将使我们能够在应用程序中编译我们的 Rust 代码并使用数据库二进制文件。

    我们的系统现在已经设置好了，我们可以继续使用以下代码安装我们的需求：

    ```rs
    RUN pip install --upgrade pip setuptools wheel
    RUN pip install -r requirements.txt
    ```

    一切准备就绪，我们可以定义我们的系统。没有任何东西阻止我们运行我们的应用程序。

1.  要做到这一点，我们需要公开端口并使用以下代码运行我们的应用程序：

    ```rs
    EXPOSE 5002
    CMD ["gunicorn", "-w 4", "-b", "0.0.0.0:5002", \
      "app:app"]
    ```

    注意，当我们从镜像创建容器时，我们使用列表中定义的参数运行 `CMD`。我们声明我们有四个工作进程，使用 `-w 4` 参数。然后，我们定义我们正在监听的 URL 和端口。我们的最后一个参数是 `app:app`。这表示我们的应用程序位于 `app.py` 文件中，并且在该文件中的应用程序是名为 `app` 的 `Flask` 对象。

1.  我们现在可以使用以下命令构建我们的应用程序镜像：

    ```rs
    flask-fib tag. 
    ```

1.  我们可以使用以下命令检查我们的镜像：

    ```rs
    docker image ls
    ```

    运行此命令会给我们一个以下形式的镜像：

    ```rs
    REPOSITORY      TAG         IMAGE ID
    flask-fib       latest      0cdb0c979ac1
    CREATED          SIZE
    33 minutes ago   1.05GB
    ```

这很重要。我们将在运行应用程序时需要引用我们的镜像，我们将在下一部分定义它。

## 构建我们的 NGINX 服务

当谈到 Docker 和 NGINX 时，我们很幸运，因为我们不需要构建一个定义 NGINX 镜像的 Dockerfile。NGINX 已经发布了一个官方镜像，我们可以免费下载和使用。然而，我们确实需要修改其配置。NGINX 非常重要；这是因为它赋予我们控制如何处理传入请求的能力。我们可以根据 URL 的部分将请求重定向到不同的服务。此外，我们可以控制数据的大小、连接的持续时间并配置 HTTPS 流量。NGINX 还可以作为负载均衡器。在这个例子中，我们将以最简单的格式配置 NGINX 以使其运行。然而，必须注意的是，NGINX 本身是一个庞大的主题；在 *进一步阅读* 部分提供了一个有用的 NGINX 书籍的参考。

我们可以通过以下步骤构建我们的 NGINX 服务并将其连接到我们的 Flask 应用程序：

1.  我们将使用 `deployment/nginx/nginx.conf` 文件中的代码配置我们的 NGINX 容器。在这个文件中，我们声明我们的工作进程和错误日志，如下所示：

    ```rs
    worker_processes  auto;
    error_log  /var/log/nginx/error.log warn;
    ```

    在这里，我们已将 `worker_processes` 定义为 `auto`。这是自动检测可用的 CPU 核心数，将进程数设置为 CPU 核心数。

1.  现在，我们必须使用以下代码定义一个工作进程一次可以处理的最大连接数：

    ```rs
    events {
        worker_connections  512;
    }
    ```

    必须注意的是，这里选择的是 NGINX 的默认端口号。

1.  现在我们所剩下的只是定义我们的 HTTP 监听器。这可以通过以下代码实现：

    ```rs
    http {
        server {
               listen 80;
               location / {
                    proxy_pass http://flask_app:5002/;
               }
        }
    }
    ```

    在这里，请注意我们监听的是端口 `80`，这是标准的对外监听端口。然后，我们声明如果我们的 URL 有任何模式，我们就将其传递到端口 `5002` 的 `flask_app` 容器。如果我们愿意，可以在 `http` 部分堆叠多个位置。例如，如果我们有另一个应用，我们可以使用以下代码将请求路由到其他应用，如果 URL 尾部以 `/another_app/` 开头：

    ```rs
               location /another_app {
                    proxy_pass http://another_app:5002/;
               }
               location / {
                    proxy_pass http://flask_app:5002/;
               }
    ```

我们 NGINX 的配置文件已经完成。再次强调，还有很多其他的配置参数；我们只是运行了最基本的部分。更多关于这些参数的资源在 *进一步阅读* 部分中有所说明。考虑到我们的 NGINX 配置文件已经完成，对于下一步，我们必须在 Flask 应用程序旁边运行它。

## 连接并运行我们的 Nginx 服务

要一起运行我们的应用程序和 NGINX，我们将使用 `docker-compose`。这允许我们同时定义多个 Docker 容器，这些容器可以相互通信。没有什么阻止我们在服务器上运行 `docker-compose` 来实现基本设置。然而，如果需要，更高级的系统如 Kubernetes 可以帮助跨多个服务器对 Docker 容器进行编排。此外，不同的云平台提供开箱即用的负载均衡器。执行以下步骤：

1.  在我们的 `deployment/docker-compose.yml` 文件中，我们使用以下代码声明我们正在使用的 `docker-compose` 版本：

    ```rs
    version: "3.7"
    ```

1.  现在实施完成后，我们可以定义我们的服务，包括我们的第一个服务，即我们的 Flask 应用程序。这是以下代码定义的：

    ```rs
    services:
        flask_app:
            container_name: fib-calculator
            image: "flask-fib:latest"
            restart: always
            ports:
              - "5002:5002"
            expose:
                - 5002
    ```

    在前面的代码中，我们引用了我们使用最新版本构建的镜像。例如，如果我们更改了镜像并重新构建它，那么我们的 `docker-compose` 设置将使用这个镜像。我们还给它一个容器名称，这样我们就可以在检查正在运行的容器时知道容器状态。此外，我们声明我们通过端口 `5002` 接受流量，并将其路由到我们的容器端口 `5002`。因为我们选择了这条路径，所以我们还暴露了端口 `5002`。如果我们现在运行我们的 `docker-compose` 设置，我们可以通过 `http://localhost:5002` URL 访问我们的应用程序。然而，如果它在服务器上运行，并且端口 `5002` 对外部流量不可访问，那么我们就无法访问它。

1.  考虑到这一点，我们可以在 `deployment/docker-compose.yml` 文件中使用以下代码定义我们的 NGINX：

    ```rs
        nginx:
            container_name: 'nginx'
            image: "nginx:1.13.5"
            ports:
              - "80:80"
            links:
                - flask_app
            depends_on:
                - flask_app
            volumes:
              - ./nginx/nginx.conf:/etc/nginx/nginx.conf
    ```

    在这里，你可以看到我们依赖于第三方 NGINX 镜像，并且我们将外部端口 `80` 路由到端口 `80`。我们还链接到我们的 Flask 应用程序，并依赖于它，这意味着 `docker-compose` 将确保在我们的 NGINX 服务运行之前，我们的 Flask 应用程序已经启动并运行。在 `volumes` 部分中，我们用我们在上一步定义的配置文件替换了标准配置文件。因此，我们的 NGINX 服务将运行我们定义的配置。必须注意的是，每次我们运行 `docker-compose` 时，这个配置切换都会发生。这意味着如果我们更改了配置文件然后再次运行 `docker-compose`，我们会看到变化。所以，我们已经做了所有事情来让我们的应用程序启动并运行。现在我们可以测试它了。

1.  测试我们的应用程序就像运行以下命令一样简单：

    ```rs
    5002, and boots up workers to process requests. Following this, our NGINX service looks for a range of configurations before concluding that the configuration is complete and that it is ready to start up. Also, note that the NGINX started after our Flask application was started. This is because we stated that our NGINX was dependent on our Flask application when building our docker-compose file.Now, we can directly hit our localhost URL without having to specify a port because we are listening to the outside port of `80` with our NGINX. This gives us results similar to the following:
    ```

![图 9.4 – 与我们的完全容器化的 Flask 应用程序交互![图片](img/Figure_9.04_B17720.jpg)

图 9.4 – 与我们的完全容器化 Flask 应用程序交互

现在我们有一个完全容器化的应用程序正在运行。这处于就绪状态，因此在下章中，我们可以测试我们的 Rust 与应用程序的集成是否在实际场景中真正有效。现在我们的应用程序已经运行起来，我们可以继续构建我们的数据访问层。这将使我们能够从数据库存储和获取数据。

# 定义我们的数据访问层

现在我们有一个应用程序，它接受一个数字并基于它计算斐波那契数。然而，数据库查找比计算更快。我们将利用这个事实通过最初在提交数字时执行数据库查找来优化我们的应用程序。如果它不在那里，我们将计算这个数字，将其存储在数据库中，然后将其返回给用户。在我们开始构建之前，我们必须使用 `pip` 安装以下包：

+   `pyml`：这个包帮助我们从 `.yml` 文件中加载应用程序的参数。

+   `sqlalchemy`：这个包使我们的应用程序能够将 Python 对象映射到数据库以存储和查询。

+   `alembic`：这个包帮助跟踪和将应用程序对数据库的更改应用到数据库中。

+   `psycopg2-binary`：这是将使我们的应用程序能够连接到数据库的二进制文件。

现在我们已经安装了所有需要的，我们可以通过以下步骤使我们的应用程序能够存储和获取斐波那契数：

1.  在 `docker-compose` 中定义一个 PostgreSQL 数据库。

1.  构建一个配置加载系统。

1.  定义一个数据访问层。

1.  构建数据库模型。

1.  设置应用程序数据库迁移系统。

1.  将数据库访问层应用到斐波那契计算视图。

完成这些步骤后，我们的应用程序将呈现以下形式：

```rs
├── deployment
```

```rs
│   . . .
```

```rs
├── docker-compose.yml
```

```rs
├── src
```

```rs
│   . . .
```

```rs
│   ├── config.py
```

```rs
│   ├── config.yml
```

```rs
│   ├── data_access.py
```

```rs
│   ├── fib_calcs
```

```rs
│   │   . . .
```

```rs
│   ├── models
```

```rs
│   │   ├── __init__.py
```

```rs
│   │   └── database
```

```rs
│   │       ├── __init__.py
```

```rs
│   │       └── fib_entry.py
```

```rs
│   └── requirements.txt
```

我们的部署文件结构没有改变。我们已经在根目录中添加了一个 `docker-compose.yml` 文件，这样我们就可以在我们开发应用程序时访问数据库。除此之外，我们还添加了一个数据访问文件，使我们能够连接到数据库，以及一个 `models` 模块，使我们能够将对象映射到数据库。这种结构将导致一个具有数据库访问权限的容器化 Flask 应用程序。接下来，我们将开始定义我们的数据库 Docker 容器。

## 在 docker-compose 中定义一个 PostgreSQL 数据库

为了定义我们的数据库容器，我们将以下代码应用到 `deployment/docker-compose.yml` 文件和 `docker-compose.yml` 文件中：

```rs
    Postgres:
```

```rs
      container_name: 'fib-dev-Postgres
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
        - '5432:5432'
```

```rs
      environment:
```

```rs
        - 'POSTGRES_USER=user'
```

```rs
        - 'POSTGRES_DB=fib'
```

```rs
        - 'POSTGRES_PASSWORD=password'
```

在这里，你可以观察到我们依赖于官方的第三方 Postgres 镜像。与我们在 NGINX 服务中定义配置文件的方式不同，我们使用环境变量来定义密码、数据库名和用户。当我们运行本地环境并开发我们的应用程序时，我们将在根目录下运行我们的`docker-compose`文件。现在我们已经定义了数据库；在下一节中，我们可以构建我们的配置系统。

## 构建配置加载系统

实质上，我们的配置系统通过以下步骤在 Flask 应用程序内部从`.yml`文件加载参数：

1.  根据系统不同，我们的应用程序可能需要不同的参数。因此，我们必须构建一个对象，该对象从`.yml`文件中加载参数，并在整个应用程序中以字典的形式提供服务。在我们的`src/config.py`文件中，首先，我们使用以下代码导入所需的模块：

    ```rs
    import os
    import sys
    from typing import Dict, List
    import yaml
    ```

    我们将使用`sys`模块来接收在运行应用程序时传递给我们的参数。我们使用`os`模块来检查我们指定的配置文件是否存在。

1.  我们可以使用以下代码构建全局参数对象：

    ```rs
    class GlobalParams(dict):
        def __init__(self) -> None:
            super().__init__()
            self.update(self.get_yml_file())
        @staticmethod
        def get_yml_file() -> Dict:
            file_name = sys.argv[-1]
            if ".yml" not in file_name:
                file_name = "config.yml"
            if os.path.isfile(file_name):
                with open("./{}".format(file_name)) as \
                  file:
                    data = yaml.load(file, 
                           Loader=yaml.FullLoader)
                return data
            raise FileNotFoundError(
                "{} config file is not available".
                    format(file_name)
            )
        @property
        def database_meta(self) -> Dict[str, str]:
            db_string: str = self.get("DB_URL")
            buffer: List[str] = db_string.split("/")
            second_buffer: List[str] = buffer[- \
              2].split(":")
            third_buffer: List[str] = \
              second_buffer[1].split("@")
            return {
                "DB_URL": db_string,
                "DB_NAME": buffer[-1],
                "DB_USER": second_buffer[0],
                "DB_PASSWORD": third_buffer[0],
                "DB_LOCATION":f"{third_buffer[1]} \
                  :{second_buffer[-1]}",
            }
    ```

    在这里，你可以观察到我们的`GlobalParams`类直接继承自字典类。这意味着我们拥有了字典的所有功能。除此之外，请注意，我们并没有在我们的 Python 程序中传递任何参数来指定要加载哪个`.yml`文件；相反，我们简单地回退到标准的`config.yml`文件。这是因为我们将使用我们的配置文件进行数据库迁移。在执行数据库迁移时，传递我们的参数将会很困难。如果我们想更改配置，最好是获取新数据并将其写入配置文件。

1.  现在我们已经定义了配置参数类，我们可以使用以下代码将数据库 URL 添加到我们的`src/config.yml`文件中：

    ```rs
    DB_URL: \
    "postgresql://user:password@localhost:5432/fib"
    ```

现在我们已经可以访问数据库 URL，在下一步中，我们可以构建我们的数据库访问层。

## 构建我们的数据访问层

我们将对数据库的访问定义在`src/data_access.py`文件中。一旦完成，我们就可以在任何 Flask 应用程序的位置导入数据访问层。这样我们就可以在 Flask 应用程序的任何地方访问数据库。我们可以通过以下步骤构建它：

1.  首先，我们必须使用以下代码导入所需的模块：

    ```rs
    from flask import _app_ctx_stack
    from sqlalchemy import create_engine
    from sqlalchemy.ext.declarative import 
    declarative_base
    from sqlalchemy.orm import sessionmaker, 
    scoped_session
    from config import GlobalParams
    ```

    在这里，我们将使用`_app_ctx_stack`对象来确保我们的会话处于 Flask 请求的上下文中。在此之后，我们导入所有其他的`sqlalchemy`依赖项，以确保我们的访问有一个会话创建者和一个引擎。由于这本书的重点是融合 Rust 和 Python，我们只是使用 SQLAlchemy 来探索 Rust 与数据库的集成，所以我们必须避免对数据库管理进行过多的细节描述。然而，我们应该能够感受到会话、引擎和基的作用。

1.  现在我们已经导入了所有需要的，我们可以使用以下代码构建我们的数据库引擎：

    ```rs
    class DbEngine:
        def __init__(self) -> None:
            params = GlobalParams()
            self.base = declarative_base()
            self.engine = create_engine(params.get
              ("DB_URL"),
                                       echo=True,
                                       pool_recycle=3600,
                                       pool_size=2,
                                       max_overflow=1,
                                       connect_args={
                                      'connect_timeout': 5
                                        })
            self.session = scoped_session(sessionmaker(
                bind=self.engine
            ), scopefunc=_app_ctx_stack)
            self.url = params.get("DB_URL")
    dal = DbEngine()
    ```

    现在我们有一个可以给我们数据库会话、数据库连接和基的类。然而，必须注意的是，我们初始化了`DbEngine`类并将其分配给`dal`变量；然而，我们没有在这个文件外部导入`DbEngine`类。相反，我们导入`dal`变量以用于与数据库的交互。如果我们在这个文件外部初始化时导入`DbEngine`类，并在需要与数据库交互时使用它，我们将为每个请求创建多个数据库会话，并且这些会话将难以关闭。现在我们的数据库连接已经定义，在下一步中，我们可以继续构建我们的数据库模型。

1.  在我们的数据库模型中，我们可以有一个唯一的 ID、输入数字和斐波那契数字。我们的模型在`src/models/database/fib_entry.py`文件中定义，以下代码所示：

    ```rs
    from typing import Dict
    from sqlalchemy import Column, Integer
    from data_access import dal
    class FibEntry(dal.base):
        __tablename__ = "fib_entries"
        id = Column(Integer, primary_key=True)
        input_number = Column(Integer)
        calculated_number = Column(Integer)
        @property
        def package(self) -> Dict[str, int]:
            return {
                "input_number": self.input_number,
                "calculated_number": \
                  self.calculated_number
            }
    ```

在这里，你可以看到代码非常直接。我们将`dal.base`传递到我们的模型中，以便将模型添加到元数据中。然后，我们定义将在数据库中的表名以及模型字段，它们是`id`、`input_number`和`calculated_number`。我们的数据库模型现在已经定义好了，因此我们可以在整个应用程序中使用它。此外，我们将在下一步中使用这个来管理数据库迁移。

## 设置应用程序数据库迁移系统

迁移是跟踪我们对数据库所做的所有更改的有用工具。如果我们对数据库模型进行更改或定义一个模型，我们需要将这些更改翻译到我们的数据库中。我们可以通过执行以下步骤来实现这一点：

1.  对于我们的数据库管理，我们将依赖`alembic`包。一旦我们导航到`src/`目录内部，我们运行以下命令：

    ```rs
    src/alembic/env.py file; we are going to alter this so that we can connect our alembic scripts and commands to our database. 
    ```

1.  接下来，我们必须导入`os`和`sys`模块，因为我们将会使用它们来导入我们的模型和加载我们的配置文件。我们使用以下代码导入模块：

    ```rs
    import sys
    import os
    ```

1.  在此之后，我们使用`os`模块将`src/`目录中的路径附加到以下代码：

    ```rs
    from alembic import context
    # this is the Alembic Config object, which provides
    # access to the values within the .ini file in use.
    Config = context.config
    # Interpret the config file for Python logging.
    # This line sets up loggers basically.
    fileConfig(config.config_file_name)
    # add the src to our import path
    sys.path.append(os.path.join(
        os.path.dirname(os.path.abspath(__file__)), 
        "../")
    )
    ```

1.  现在我们已经配置了导入路径，我们可以导入我们的参数和数据库引擎。然后，我们使用以下代码将我们的数据库 URL 添加到`alembic`数据库 URL 中：

    ```rs
    # config the database url for migrations
    from config import GlobalParams
    params = GlobalParams()
    section = config.config_ini_section
    db_params = params.database_meta
    config.set_section_option(section, 'sqlalchemy.url', 
                              params.get('DB_URL'))
    from data_access import dal
    db_engine = dal
    from models.database.fib_entry import FibEntry
    target_metadata = db_engine.base.metadata
    ```

1.  通过这种方式，你可以观察到自动生成的函数获取了我们的配置，然后执行以下代码进行迁移：

    ```rs
    def run_migrations_offline():
        url = config.get_main_option("sqlalchemy.url")
        context.configure(
            url=url,
            target_metadata=target_metadata,
            literal_binds=True,
            dialect_opts={"paramstyle": "named"},
            render_as_batch=True
        )
        with context.begin_transaction():
            context.run_migrations()
    ```

1.  现在我们已经将配置系统链接到我们的数据库迁移，我们必须确保 `docker-compose` 正在运行，因为我们的数据库必须是活跃的。我们可以使用以下命令生成迁移：

    ```rs
    src/alembic/versions/ file, there is an autogenerated script that creates our table with the following code:

    ```

    # 修订标识符，由 Alembic 使用。

    Revision = '40b83d85c278'

    down_revision = None

    branch_labels = None

    depends_on = None

    def upgrade():

    op.create_table('fib_entries',

    sa.Column('id', sa.Integer(), nullable=False),

    sa.Column('input_number', sa.Integer(),\

    nullable=True),

    sa.Column('calculated_number', sa.Integer(), \

    nullable=True),

    sa.PrimaryKeyConstraint('id')

    )

    def downgrade():

    op.drop_table('fib_entries')

    ```rs

    Here, if we upgrade, the `upgrade` function will run, and if we downgrade, the `downgrade` function will run. We can upgrade our database using the following command:

    ```

    alembic upgrade head

    ```rs

    This gives us the following printout:

    ```

    INFO  [alembic.runtime.migration] Context impl

    PostgresqlImpl.

    INFO  [alembic.runtime.migration] 将假设

    事务性 DDL。

    INFO  [alembic.runtime.migration] 正在运行升级 ->

    40b83d85c278, create-fib-entry

    ```rs

    ```

我们的迁移已成功。在下一步中，我们将在应用程序中与数据库进行交互。

## 构建数据库模型

现在我们有一个应用了我们的应用程序模型的数据库，我们可以在应用程序中与之交互。这可以通过将数据访问层和数据模型导入使用它们的视图来完成，并且，嗯，使用它们：

1.  对于我们的示例，我们将在 `src/app/app.py` 文件中实现我们的视图。首先，我们使用以下代码导入数据访问层和模型：

    ```rs
    from data_access import dal
    from models.database.fib_entry import FibEntry
    ```

    通过这些导入，我们可以修改我们的计算视图以检查数字是否存在于数据库中，如果存在，则从数据库中返回该数字。

1.  如果数据库中没有可用，那么我们计算它，将结果保存到数据库中，并使用以下代码返回结果：

    ```rs
    @app.route("/calculate/<int:number>")
    def calculate(number):
        fib_calc = dal.session.query(FibEntry).filter_by(
               input_number=number).one_or_none()
        if fib_calc is None:
            calc = FibCalculation(input_number=number)
            new_calc = FibEntry(input_number=number,
                      calculated_number=calc.fib_number)
            dal.session.add(new_calc)
            dal.session.commit()
            return f"you entered {calc.input_number} " \
                   f"which has a Fibonacci number of " \
                   f"{calc.fib_number}"
        return f"you entered {fib_calc.input_number} " 
           f"which has an existing Fibonacci number of " 
               f"{fib_calc.calculated_number}"
    ```

    在这里，你可以观察到我们与数据库的交互非常直接。

1.  现在，我们必须确保当我们的请求完成后，我们的数据库会话已过期、关闭并删除，使用以下代码：

    ```rs
    @app.teardown_request
    def teardown_request(*args, **kwargs):
        dal.session.expire_all()
        dal.session.remove()
        dal.session.close()
    ```

因此，我们与数据库的交互是安全且完全正常的。你现在已经了解了使用我们的应用程序与数据库交互的基础。你可以通过阅读 SQLAlchemy 文档中关于数据库、其他数据库查询和插入的特定语法来达到其他更复杂的数据库查询。如果我们在本地上运行我们的应用程序并两次点击计算视图，我们将得到第一次和第二次的结果，如下面的截图所示：

![Figure 9.5 – The top part is the first request (calculated), and the 底部部分是第二个请求（数据库调用）](img/Figure_9.05_B17720.jpg)

Figure 9.5 – 顶部部分是第一次请求（计算），底部部分是第二次请求（数据库调用）

我们的数据库正在按照我们预期的样子工作。现在应用程序已经完全运行，如果你愿意的话，可以继续到下一节，因为这样已经足够测试 Flask 应用程序中的 Rust 代码了，我们将在下一章中这样做。然而，如果你想了解我们如何在部署部分应用数据库，我们将在下一节中介绍这一点。

## 将数据库访问层应用于 fib 计算视图

将数据库添加到我们的部署中，就是将其添加到我们的 `docker-compose` 部署中，并更新我们的配置文件以映射到 `docker-compose` 部署中的数据库服务。我们可以通过以下步骤实现这一点：

1.  首先，我们必须使用以下代码重构我们的 `deployment/docker-compose.yml` 文件：

    ```rs
    services:
        flask_app:
            container_name: fib-calculator
            image: "flask-fib:latest"
            restart: always
            ports:
              - "5002:5002"
            expose:
                - 5002
            depends_on:
              - postgres
            links:
              - postgres
        nginx:
            . . .
        postgres:
            container_name: 'fib-live-postgres'
            image: 'postgres:11.2'
            restart: always
            ports:
                - '5432:5432'
            environment:
                - 'POSTGRES_USER=user'
                - 'POSTGRES_DB=fib'
                - 'POSTGRES_PASSWORD=password'
    ```

    你可以观察到我们为数据库容器取了一个稍微不同的名字。这是为了确保不会与我们的开发数据库发生冲突。此外，我们还声明我们的 Flask 应用程序依赖于并链接到我们的数据库。

1.  我们还必须将我们的 Flask 应用程序指向 Docker 数据库。为此，我们必须为 Flask 应用程序有一个不同的配置文件。我们可以通过在 Flask 应用的 `src/Dockerfile` 中管理配置文件切换。这可以通过以下代码完成：

    ```rs
    # Copy the current directory contents into the 
      container at /app
    ADD . /app
    RUN rm ./config.yml
    RUN mv live_config.yml config.yml
    . . .
    ```

    在这里，我们删除了 `config.yml` 文件，然后将 `live_config.yml` 文件的名称更改为 `config.yml`。

1.  然后，我们必须创建我们的 `src/live_config.yml` 文件，内容如下：

    ```rs
    DB_URL: "postgresql://user:password@postgres:5432/fib"
    ```

    在这里，我们将 `@localhost` 改为 `@postgres`，因为我们的服务分类名为 `postgres`。

1.  在此之后，我们可以使用以下命令重新构建我们的 Flask 镜像：

    ```rs
    docker build . -t flask-fib
    ```

1.  现在，我们可以运行我们的 `docker-compose` 部署，但我们必须在 `docker-compose` 部署运行时运行我们的迁移。这是因为我们的 Flask 应用程序在尝试对数据库进行查询之前不会引发错误，所以如果我们不发出请求，在迁移之前运行 `docker-compose` 是可以的。当我们进行迁移时，我们必须在 `docker-compose` 运行时进行；否则，迁移将无法连接到数据库。我们可以在 `docker-compose` 运行时使用以下命令运行迁移：

    ```rs
    docker exec -it fib-calculator alembic upgrade head
    ```

这将在我们的 Flask 容器上运行迁移。不建议你只将实时配置文件放在你的应用程序代码中。我偏爱的方法是将加密的配置文件上传到 AWS S3，并在 Kubernetes 启动 pod 时拉取它。这超出了本书的范围，因为这不是一本关于网络开发的书籍。然而，如果需要的话，重要的是要记住这种方法，以便进一步阅读。

目前，使用我们的 Flask 应用程序计算斐波那契数并没有太多可抱怨的。然而，当我们尝试计算一个大数时，我们将等待很长时间，这将使请求挂起。为了防止这种情况发生，在下一节中，我们将实现一个消息总线。这样，当我们的应用程序在后台处理大数时，我们可以返回一个消息告诉用户耐心等待。

# 构建消息总线

对于本节，我们将使用`Celery`和`Redis`包来构建和运行我们的消息总线。一旦我们完成本节，我们的机制将采取以下类似的形式：

![Figure 9.6 – A message bus with Flask and Celery]

![Figure 9.06_B17720.jpg]

图 9.6 – 带有 Flask 和`Celery`的消息总线

如前图所示，我们有两个进程正在运行。一个是运行我们的 Flask 应用程序，另一个是运行`Celery`，它处理队列和任务处理。为了使这一切工作，我们将执行以下步骤：

1.  为 Flask 构建`Celery`代理。

1.  为**Celery**构建一个斐波那契计算任务。

1.  使用`Celery`更新我们的计算视图。

1.  在 Docker 中定义我们的`Celery`服务。

在我们开始这些步骤之前，我们必须使用`pip`安装以下包：

+   `Celery`：这是我们将要使用的消息总线代理。

+   `Redis`：这是`Celery`将要使用的存储系统。

现在我们已经安装了所有需求，我们必须记得使用`Celery`和 Redis 更新`src/requirements.txt`文件以供我们的 Docker 构建使用。现在我们已经安装了所有依赖项，我们可以开始构建我们的`Celery`代理，如下所示。

## 为 Flask 构建 Celery 代理。

实质上，我们的`Celery`代理是一个存储系统，它将存储我们发送给它的任务相关数据。我们可以通过以下步骤设置我们的存储系统并将其连接到我们的`Celery`系统：

1.  在构建我们的任务队列时，我们将构建自己的模块。在`src/`目录内，我们的任务队列模块将具有以下结构：

    ```rs
    └── task_queue
        ├── __init__.py
        ├── engine.py
        └── fib_calc_task.py
    ```

    我们的`engine.py`文件将托管一个考虑 Flask 应用程序上下文的`Celery`构造函数。

1.  我们将在`fib_calc_task.py`文件中构建我们的斐波那契计算`Celery`任务。在我们的`engine.py`文件中，我们可以使用以下代码构建构造函数：

    ```rs
    from celery import Celery
    from config import GlobalParams
    def make_celery(flask_app):
        params = GlobalParams()
        celery = Celery(
            backend=params.get("QUEUE_BACKEND"),
            broker=params.get("QUEUE_BROKER")
        )
        celery.conf.update(flask_app.config)
        class ContextTask(celery.Task):
            def __call__(self, *args, **kwargs):
                with flask_app.app_context():
                    return self.run(*args, **kwargs)
        celery.Task = ContextTask
        return celery
    ```

    `backend`和`broker`参数将指向存储；我们将在稍后定义它们。在这里，你可以观察到我们必须将 Flask 应用程序传递给函数，构建`Celery`类，并将`Celery`任务对象与 Flask 应用程序上下文融合，然后返回它。当涉及到定义运行我们的`Celery`进程的入口点时，我们应该将其放置在与我们的 Flask 应用程序相同的文件中。这是因为我们希望使用相同的 Docker 构建和图像来构建 Flask 应用程序和`Celery`进程。

1.  要实现这一点，我们在`src/app.py`文件中导入我们的`Celery`构造函数，并通过它传递 Flask 应用程序，使用以下代码：

    ```rs
    . . .
    from task_queue.engine import make_celery
    app = Flask(__name__)
    celery = make_celery(app)
    . . .
    ```

1.  现在，当我们运行我们的`Celery`代理时，我们将将其指向我们的`src/app.py`文件以及其中的`Celery`对象。此外，我们必须定义我们的后端存储系统。因为我们使用 Redis，我们可以在`src/config.yml`文件中使用以下代码定义这些参数：

    ```rs
    QUEUE_BACKEND: "redis://localhost:6379/0"
    QUEUE_BROKER: "redis://localhost:6379/0"
    ```

现在我们已经定义了我们的`Celery`代理，在下一步中，我们可以构建我们的 Fibonacci 计算任务。

## 为 Celery 构建 Fibonacci 计算任务

当涉及到运行我们的`Celery`任务时，我们需要构建另一个构造函数。然而，我们不是传递我们的 Flask 应用程序，而是传递我们的`Celery`代理。我们可以在`src/task_queue/fib_calc_task.py`文件中使用以下代码实现这一点：

```rs
from data_access import dal
```

```rs
from fib_calcs.fib_calculation import FibCalculation
```

```rs
from models.database.fib_entry import FibEntry
```

```rs
def create_calculate_fib(input_celery):
```

```rs
    @input_celery.task()
```

```rs
    def calculate_fib(number):
```

```rs
        calculation = FibCalculation(input_number=number)
```

```rs
        fib_entry = FibEntry(
```

```rs
            input_number=calculation.input_number,
```

```rs
            calculated_number=calculation.fib_number
```

```rs
        )
```

```rs
        dal.session.add(fib_entry)
```

```rs
        dal.session.commit()
```

```rs
    return calculate_fib
```

上述逻辑类似于我们的标准计算视图。我们可以将其导入到我们的`src/app.py`文件中，并使用以下代码将我们的`Celery`代理传递给它：

```rs
. . .
```

```rs
from task_queue.engine import make_celery
```

```rs
app = Flask(__name__)
```

```rs
celery = make_celery(app)
```

```rs
from task_queue.fib_calc_task import create_calculate_fib
```

```rs
calculate_fib = create_calculate_fib(input_celery=celery)
```

```rs
. . .
```

现在我们已经定义了任务，并将其与`Celery`代理和 Flask 应用程序融合，在下一步中，我们可以将我们的`Celery`任务添加到计算视图中，如果输入的数字太大。

## 更新我们的计算视图

在我们的视图中，我们必须检查输入的数字是否小于`31`且不在数据库中。如果是这样，我们运行我们现有的标准代码。然而，如果输入的数字是`30`或更高，我们将计算发送到`Celery`代理，并返回一条消息告诉用户它已被发送到队列。我们可以使用以下代码完成此操作：

```rs
@app.route("/calculate/<int:number>")
```

```rs
def calculate(number):
```

```rs
    fib_calc = dal.session.query(FibEntry).filter_by(
```

```rs
                       input_number=number).one_or_none()
```

```rs
    if fib_calc is None:
```

```rs
        if number < 31:
```

```rs
            calc = FibCalculation(input_number=number)
```

```rs
            new_calc = FibEntry(input_number=number,
```

```rs
                                calculated_number=calc.
```

```rs
                                fib_number)
```

```rs
            dal.session.add(new_calc)
```

```rs
            dal.session.commit()
```

```rs
            return f"you entered {calc.input_number} " \
```

```rs
                   f"which has a Fibonacci number of " \
```

```rs
                   f"{calc.fib_number}"
```

```rs
        calculate_fib.delay(number)
```

```rs
        return "calculate fib sent to queue because " \
```

```rs
               "it's above 30"
```

```rs
    return f"you entered {fib_calc.input_number} " \
```

```rs
           f"which has an existing Fibonacci number of " \
```

```rs
           f"{fib_calc.calculated_number}"
```

现在我们已经完全构建了带有任务的`Celery`进程。在下一步中，我们将在`docker-compose`中定义我们的 Redis 服务。

## 在 Docker 中定义我们的 Celery 服务

当涉及到我们的`Celery`服务时，请记住我们使用了 Redis 作为存储机制。考虑到这一点，我们在开发的`docker-compose.yml`文件中定义我们的 Redis 服务，使用以下代码：

```rs
. . .
```

```rs
    redis:
```

```rs
      container_name: 'main-dev-redis'
```

```rs
      image: 'redis:5.0.3'
```

```rs
      ports:
```

```rs
        - '6379:6379'
```

现在以开发模式运行整个系统需要运行我们项目根目录下的开发`docker-compose`文件。此外，我们通过运行 Python 中的`app.py`文件来运行 Flask 应用程序，其中`PYTHONPATH`设置为`src`。

在此之后，我们打开另一个终端窗口，导航到`src`目录内的终端，并运行以下命令：

```rs
celery -A app.celery worker -l info
```

这是我们将`Celery`指向`app.py`文件的地方。我们声明对象名为`Celery`，它是一个工作进程，日志级别为`info`。运行此命令会给出以下输出：

```rs
-------------- celery@maxwells-MacBook-Pro.
```

```rs
--- ***** ----- local v5.1.2 (sun-harmonics)
```

```rs
-- ******* ---- Darwin-20.2.0-x86_64-i386-64bit
```

```rs
- *** --- * --- 2021-08-22 23:24:14
```

```rs
- ** ---------- [config]
```

```rs
- ** ---------- .> app:         __main__:0x7fd0796d0ed0
```

```rs
- ** ---------- .> transport:   redis://localhost:6379/0
```

```rs
- ** ---------- .> results:     redis://localhost:6379/0
```

```rs
- *** --- * --- .> concurrency: 4 (prefork)
```

```rs
-- ******* ---- .> task events: OFF (enable -E to
```

```rs
--- ***** -----    monitor tasks in this worker)
```

```rs
 -------------- [queues]
```

```rs
                .> celery  exchange=celery(direct) 
```

```rs
key=celery               
```

```rs
[tasks]
```

```rs
  . task_queue.fib_calc_task.calculate_fib
```

```rs
[2021-08-22 23:24:14,385: INFO/MainProcess] Connected 
```

```rs
to redis://localhost:6379/0
```

```rs
[2021-08-22 23:24:14,410: INFO/MainProcess] mingle: 
```

```rs
searching for neighbors
```

```rs
[2021-08-22 23:24:15,476: INFO/MainProcess] mingle: 
```

```rs
all alone
```

```rs
[2021-08-22 23:24:15,514: INFO/MainProcess] 
```

```rs
celery@maxwells-MacBook-Pro.local ready.
```

```rs
[2021-08-22 23:24:39,822: INFO/MainProcess] 
```

```rs
Task task_queue.fib_calc_task.calculate_fib
```

```rs
[c3241a5f-3208-48f7-9b0a-822c30aef94e] received
```

上述输出显示我们的任务已被注册，并且启动了四个进程。使用大于`30`的数字点击计算视图，我们得到以下视图：

![Figure 9.7 – 底部显示使用 Celery 的第一个请求，顶部显示使用 Celery 的请求]

使用 Celery 的第二次请求

![Figure 9.07_B17720.jpg](img/Figure_9.07_B17720.jpg)

图 9.7 – 底部显示的是使用 `Celery` 的第一个请求，顶部显示的是使用 `Celery` 的第二个请求

现在，我们的 Flask 应用程序与数据库和 `Celery` 消息总线完全在本地运行。如果您愿意，可以在这里停止，因为这对于在下一章测试 `Celery` 中的 Rust 代码已经足够了。但是，如果您想学习如何将 `Celery` 应用于部署部分，请继续本节。

将 `Celery` 应用于我们的 `docker-compose` 部署很简单。记住，我们有一个相同的入口点，因此不需要新的镜像。我们只需更改启动我们的 `Celery` 容器时运行的命令。这可以在我们的 `deployment/docker-compose.yml` 文件中使用以下代码完成：

```rs
. . .
```

```rs
    main_cache:
```

```rs
        container_name: 'main-live-redis'
```

```rs
        image: 'redis:5.0.3'
```

```rs
        ports:
```

```rs
            - '6379:6379'
```

```rs
    queue_worker:
```

```rs
        container_name: fib-worker
```

```rs
        image: "flask-fib:latest"
```

```rs
        restart: always
```

```rs
        entrypoint: "celery -A app.celery worker -l info"
```

```rs
        ports:
```

```rs
            - "5003:5003"
```

```rs
        expose:
```

```rs
            - 5003
```

```rs
        depends_on:
```

```rs
            - main_cache
```

```rs
        links:
```

```rs
            - main_cache
```

在这里，您可以观察到我们为我们的 `queue_worker` 服务拉取了相同的镜像。然而，我们使用 `docker-compose` 中的 `entrypoint` 标签更改了 Docker 构建中的 `CMD` 标签。因此，当我们的 `queue_worker` 服务构建时，它将运行 `Celery` 命令来运行 `Celery` 工作器，而不是运行 Flask 网络应用程序。在此之后，我们需要在 `live_config.yml` 文件中添加一些更多参数，如下所示：

```rs
QUEUE_BACKEND: "redis://main_cache:6379/0"
```

```rs
QUEUE_BROKER: "redis://main_cache:6379/0"
```

在这里，我们将我们的 Redis 服务命名为不同于 localhost。这样做是为了确保我们的打包 `Celery` 工作器和 Flask 应用程序在 `docker-compose` 部署中连接到我们的 Redis 服务。运行 `docker-compose` 部署后，我们可以用 `localhost` 代替 `127.0.0.1:5002` 重复 *图 9.6* 中展示的请求。有了这个，我们的 Flask 应用程序就准备好与数据库和任务队列一起部署了。技术上讲，我们的设置可以在服务器上部署和使用。我已经这样做过，并且一切正常。然而，对于更高级的系统和控制，建议您进行一些进一步的阅读。关于在 Docker 中部署 Flask 应用程序到云服务（如 Amazon Web Services）的附加参考资料列在 *进一步阅读* 部分。

# 摘要

在本章中，我们构建了一个 Python Flask 应用程序，该应用程序可以访问数据库和消息总线，以便在后台排队处理重任务。在此之后，我们将我们的服务封装在 Docker 容器中，并通过一个简单的 `docker-compose` 文件与 NGINX 部署。此外，我们还学会了如何使用相同的构建在同一 Dockerfile 中构建我们的 `Celery` 工作器和 Flask 应用程序。这使得我们的代码更容易维护和部署。我们还使用 `alembic` 和配置文件管理我们的数据库迁移，然后在部署我们的应用程序时切换到另一个配置文件。虽然这不是一本网络开发教科书，但我们已经涵盖了构建 Flask 网络应用程序的所有基本要素。

关于数据库查询、数据序列化或 HTML 和 CSS 渲染的更多细节，在 Flask 文档中以简单的方式进行了介绍。我们已经涵盖了所有困难的部分。现在，我们可以尝试 Rust 以及它如何与 Python 网络应用程序融合，不仅限于开发环境，还包括应用程序在 Docker 容器中运行并与其他 Docker 容器通信的实时环境。在下一章中，我们将 Rust 与我们的 Flask 应用程序融合。这样，它就可以与开发和部署设置一起工作。

# 问题

1.  当我们从开发环境切换到 `docker-compose` 上的部署环境以与另一个服务通信时，我们在 URI 中做了什么改变？

1.  我们为什么使用配置文件？

1.  我们真的需要 `alembic` 来管理数据库吗？

1.  我们需要对我们的数据库引擎做些什么，以确保数据库不会因悬挂会话而充满？

1.  我们是否需要 Redis 来管理我们的 `Celery` 工作进程？

# 答案

1.  我们将 URI 中的 `localhost` 部分切换到 `docker-compose` 服务的标签。

1.  配置文件使我们能够轻松切换上下文；例如，从开发环境切换到实时环境。此外，如果我们的 `Celery` 服务需要出于某种原因与不同的数据库通信，这可以以最小的努力完成；只需更改配置文件即可。这还是一个安全问题。硬编码数据库 URI 将将这些凭证暴露给任何有权访问代码并位于 GitHub 仓库历史记录中的人。将配置文件存储在不同的空间，如 AWS S3，当服务部署时会被拉取。

1.  技术上，不是的。我们只需简单地编写 SQL 脚本并按顺序运行它们。当我从事金融科技工作时，这实际上是我们必须做的事情。虽然这可以给你更多的自由，但它确实需要更多的时间，并且更容易出错。使用 `alembic` 将为你节省时间、错误和大部分工作。

1.  我们在定义引擎的同一文件中初始化我们的数据库引擎一次。我们永远不会再次初始化它，并且在我们需要的地方导入这个初始化的引擎。不这样做会导致我们的数据库因悬挂会话而陷入停滞，并且会显示一些不太有帮助的错误信息，这些信息会让你在互联网上四处奔波，寻找模糊的半成品答案。此外，我们必须在 Flask 的 teardown 函数中关闭所有请求的会话。

1.  是的，也不是。我们需要一种存储机制，例如 Redis；然而，如果需要，我们也可以使用 RabbitMQ 或 MongoDB 来代替 Redis。

# 进一步阅读

+   *《Nginx HTTP 服务器 - 第四版：利用 Nginx 的力量》* by Fjordvald M. and Nedelcu C. (2018) (Packt)

+   官方 Flask 文档 – Pallets (2021)：[`flask.palletsprojects.com/en/2.0.x/`](https://flask.palletsprojects.com/en/2.0.x/%0D)

+   *《Python 实战 Docker 微服务》* by Jaime Buelta (2019) (Packt)

+   *AWS 认证开发者 - 助理指南 - 第二版* 由 Vipul Tankariya 和 Bhavin Parmar 著（2019 年）（Packt）

+   SQLAlchemy 查询参考文档（2021 年）：[`docs.sqlalchemy.org/en/14/orm/loading_objects.html`](https://docs.sqlalchemy.org/en/14/orm/loading_objects.html%0D)
