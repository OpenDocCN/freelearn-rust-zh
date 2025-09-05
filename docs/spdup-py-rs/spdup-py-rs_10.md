# *第十章*：将 Rust 注入 Python Flask 应用

在 *第九章* 中，*为 Rust 结构化 Python Flask 应用*，我们设置了一个基本的 Python 网络应用程序，该应用程序可以使用 Docker 部署。在本章中，我们将把 Rust 融入该网络应用程序的各个方面。这意味着我们将磨练定义可以安装使用 `pip` 的 Rust 包的技能。有了这些包，我们将把 Rust 代码插入我们的 Flask 和 Celery 容器中。我们还将直接使用 Rust 与现有的数据库交互，无需担心迁移。这是因为我们的 Rust 包将镜像现有数据库的模式。我们需要 Rust `nightly` 版本来编译我们的包，因此我们还将学习如何在构建 Flask 镜像时管理 Rust `nightly`。我们还将学习如何使用来自私有 GitHub 仓库的 Rust 包。

在本章中，我们将涵盖以下主题：

+   将 Rust 融入 Flask 和 Celery

+   使用 Rust 部署 Flask 和 Celery

+   使用私有 GitHub 仓库部署

+   将 Rust 与数据访问融合

+   在 Flask 中部署 Rust `nightly`

了解这些主题将使我们能够在 Python 网络应用程序中使用我们的 Rust 包，以便可以在 Docker 中部署。这将使我们的 Rust 技能直接与实际世界接轨，使我们能够加快 Python 网络应用程序的速度，而无需重写整个基础设施。如果你是 Python 网络开发者，你将在阅读本章后能够上班并开始将 Rust 注入网络应用程序，以引入快速、安全的代码而风险不大。

# 技术要求

以下为本章的技术要求：

+   本章的代码和数据可以在 [`github.com/PacktPublishing/Speed-up-your-Python-with-Rust/tree/main/chapter_ten`](https://github.com/PacktPublishing/Speed-up-your-Python-with-Rust/tree/main/chapter_ten) 找到。

+   在本章中，你将构建一个 Docker 容器中的 Flask 应用程序。这可以通过以下 GitHub 仓库获取：[`github.com/maxwellflitton/fib-flask`](https://github.com/maxwellflitton/fib-flask)。

# 将 Rust 融入 Flask 和 Celery

我们将通过使用 `pip` 安装我们的 Rust 斐波那契计算库来将 Rust 融入我们的 Flask 应用程序。然后我们将在我们的视图和 Celery 任务中使用它。这将加快我们的 Flask 应用程序，而无需对我们基础设施进行大的改动。为了实现这一点，我们将执行以下步骤：

1.  定义我们对 Rust 斐波那契数计算包的依赖。

1.  使用 Rust 构建我们的计算模块。

1.  在我们的 Flask 应用程序中使用 Rust 创建一个计算视图。

1.  将 Rust 插入我们的 Celery 任务。

通过这种方式，我们将拥有一个由于 Rust 而加速的 Flask 应用程序。让我们开始吧！

## 定义我们对 Rust 斐波那契数计算包的依赖

当涉及到我们的 Rust 依赖项时，可能会诱使我们直接将 Rust 依赖项放入我们的 `requirements.txt` 文件中。然而，这可能会变得令人困惑。此外，我们正在使用一个自动化的过程来更新我们的 `requirements.txt` 文件。这可能会使我们的 GitHub 仓库从 `requirements.txt` 文件中被清除。我们必须记住，我们的 `requirements.txt` 文件只是一个文本文件。因此，没有任何东西阻止我们添加另一个文本文件，列出我们的 GitHub 仓库，并使用它来安装我们的应用程序所依赖的 GitHub 仓库。为此，我们将使用以下依赖项填充我们的 `src/git_repos.txt` 文件：

```rs
git+https://github.com/maxwellflitton/flitton-fib-rs@main
```

现在，我们可以使用以下命令安装我们的 GitHub 仓库依赖项：

```rs
pip install -r git_repos.txt
```

这将导致我们的 Python 系统下载 GitHub 仓库并在我们的 Python 包中编译它。我们现在知道哪些 GitHub 仓库在为我们的应用程序提供动力，因此我们可以开始使用自动化工具来更新我们的 `requirements.txt` 文件。现在我们已经安装了 Rust 包，我们可以开始构建一个将使用 Rust 的计算模块。

## 使用 Rust 构建我们的计算模型

我们的计算模块将具有以下结构：

```rs
src
```

```rs
├── fib_calcs
```

```rs
│   ├── __init__.py
```

```rs
│   ├── enums.py
```

```rs
│   └── fib_calculation.py
```

我们已经在上一章的 `fib_calculation.py` 文件中有了我们的 Python 计算代码。然而，我们现在支持 Rust 和 Python 的实现。

要做到这一点，我们首先将在我们的 `enums.py` 文件中定义一个枚举，如下所示：

```rs
from enum import Enum
```

```rs
class CalculationMethod(Enum):
```

```rs
    PYTHON = "python"
```

```rs
    RUST = "rust"
```

使用这个枚举，我们可以继续添加方法。例如，如果我们稍后开发微服务并有一个单独的服务器来计算我们的斐波那契数，我们可以在枚举中添加一个 API 调用，并在我们的计算接口中支持它。根据配置文件，我们可以在它们之间切换。现在我们已经定义了枚举，我们可以在 `src/fib_calcs/__init__.py` 文件中构建我们的接口：

1.  首先，我们必须使用以下代码导入我们需要的内容：

    ```rs
    import time
    from flitton_fib_rs.flitton_fib_rs import \
      fibonacci_number
    from fib_calcs.enums import CalculationMethod
    from fib_calcs.fib_calculation import FibCalculation
    ```

    在这里，我们使用了 `time` 模块来计时一个过程运行所需的时间。我们还导入了我们的 Python 和 Rust 实现来进行计算。最后，我们导入了我们的枚举来映射我们使用了哪种方法。

1.  在所有这些之后，我们可以在我们的 `src/fib_calcs/__init__.py` 文件中开始构建时间处理函数，如下所示：

    ```rs
    def _time_process(processor, input_number):
        start = time.time()
        calc = processor(input_number)
        finish = time.time()
        time_taken = finish - start
        return calc, time_taken
    ```

    在这里，我们接受了一个名为 `processor` 的计算函数参数，并将 `input_number` 参数传递给该函数。我们还计时了这个过程，并返回了斐波那契数。现在我们已经完成了这个，我们可以构建一个函数来处理输入字符串，并将其转换为我们的枚举。我们并不总是将字符串传递给我们的接口，但如果我们可以从配置文件中加载一个表示我们想要哪种处理类型的字符串，这将很重要。

1.  我们可以使用以下代码定义我们的处理方法：

    ```rs
    def _process_method(input_method):
        calc_enum = CalculationMethod._value2member_map_.
                    get(input_method)
        if calc_enum is None:
            raise ValueError(
                f"{input_method} is not supported, "
                f"please choose from "
         f"{CalculationMethod._value2member_map_.keys()}")
        return calc_enum
    ```

    这里，我们可以看到我们的字符串存储在`_value2member_map_`映射的键值中。如果它不在键中，那么我们的枚举将不支持它，并且方法将抛出错误。然而，如果它存在，我们返回与键值关联的枚举。

1.  现在，我们可以使用以下代码定义我们的接口的两个辅助函数：

    ```rs
    def calc_fib_num(input_number, method):
        if isinstance(method, str):
            method = _process_method(input_method=method)
        if method == CalculationMethod.PYTHON:
            calc, time_taken = _time_process(
                processor=FibCalculation,
                input_number=input_number
            )
            return calc.fib_number, time_taken
        elif method == CalculationMethod.RUST:
            calc, time_taken = _time_process(
                processor=fibonacci_number,
                input_number=input_number
            )
            return calc, time_taken
    ```

这里，如果我们传递一个字符串作为我们的方法，我们可以将其转换为枚举。如果枚举指向 Python，我们可以将我们的 Python 计算对象以及输入数字传递到我们的`_time_process`函数中。然后，我们可以返回斐波那契数和所需时间。如果枚举指向 Rust，我们可以执行相同的操作，但使用 Rust 函数。通过这种方法，我们可以添加和移除功能。例如，我们可以用指向另一个不计时过程的计算函数的参数来切换计时过程，如果我们想在不计时的情况下仅执行计算，那么这个过程将不会计时。然而，对于这个例子，我们将使用计时过程来比较速度。现在，我们已经构建了我们的接口，我们可以使用这个接口创建我们的计算视图。

## 使用 Rust 创建计算视图

我们在`src/app.py`文件中托管我们的视图。首先，我们将使用以下代码导入我们的接口：

```rs
from fib_calcs import calc_fib_num
```

```rs
from fib_calcs.enums import CalculationMethod
```

使用这个新接口和枚举，我们可以通过以下代码对我们的标准计算视图进行修改：

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
        if number < 50:
```

```rs
            fib_number, time_taken = calc_fib_num(
```

```rs
                input_number=number,
```

```rs
                method=CalculationMethod.PYTHON
```

```rs
            )
```

```rs
            . . .
```

```rs
            return f"you entered {number} " \
```

```rs
                   f"which has a Fibonacci number of " \
```

```rs
                   f"{fib_number} which took {time_taken}"
```

```rs
    . . .
```

在这里，我们使用的是新接口。正因为如此，我们还可以返回执行计算所需的时间。现在，我们可以构建我们的 Rust 计算视图。它将具有与标准计算视图相同的形式，这意味着您可以根据传递到 URL 的参数重构它，以便在同一个视图中拥有 Rust 和 Python 计算方法。如果不这样做，我们的 Rust 计算视图将采用以下代码的形式：

```rs
@app.route("/rust/calculate/<int:number>")
```

```rs
def rust_calculate(number):
```

```rs
    . . .
```

```rs
    if fib_calc is None:
```

```rs
        if number < 50:
```

```rs
            fib_number, time_taken = calc_fib_num(
```

```rs
                input_number=number,
```

```rs
                method=CalculationMethod.RUST
```

```rs
            )
```

```rs
            . . .
```

上述代码中的点表明，这是在标准计算函数中使用的相同代码。现在，我们的 Rust 包已经与我们的 Flask 应用融合，我们可以将 Rust 插入到我们的 Celery 任务中。

## 将 Rust 插入我们的 Celery 任务

当涉及到我们的 Celery 后台任务时，我们不必担心计时。由于接口和配置，我们必须使用以下代码将参数和接口导入到`src/task_queue/fib_calc_task.py`文件中：

```rs
from config import GlobalParams
```

```rs
from fib_calcs import calc_fib_num
```

通过这样，我们现在可以使用以下代码重构我们的 Celery 任务：

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
        params = GlobalParams()
```

```rs
        fib_number, _ = calc_fib_num(input_number=number,
```

```rs
                                     method=params.get(
```

```rs
                                     "CELERY_METHOD", 
```

```rs
                                     "rust"))
```

```rs
        fib_entry = FibEntry(input_number=number,
```

```rs
            calculated_number=fib_number)
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

这里，我们可以看到我们获取了全局参数。我们将`CELERY_METHOD`全局参数传递到`params`中。考虑到参数是从字典类继承的，我们可以使用内置的`get`方法。如果我们没有在配置文件中定义`CELERY_METHOD`，我们可以将默认计算方法设置为`rust`。

应用程序现在已完全集成，这意味着我们可以测试我们的应用程序。我们必须记住运行我们的开发 `docker-compose` 环境、Flask 应用程序和 Celery 工作进程。访问我们的两个视图将给出以下输出：

![](img/B17720_10_01.jpg)

图 10.1 – Flask、Python 和 Rust 请求

在前面的屏幕截图中，我们可以看到我们的 Rust 调用速度提高了四倍，尽管 Rust 请求的数量更高。现在我们有一个使用 Rust 加速计算的运行中的 Python 应用程序。然而，如果我们不能部署它，这并不很有用。互联网上充满了半成品教程，它们教你如何在开发环境中表面地做某事，而在生产环境中却无法使用或配置它。在下一节中，我们将配置我们的 Docker 环境，以便我们可以部署我们的应用程序。

# 使用 Rust 部署 Flask 和 Celery

为了让我们的 Flask 应用程序的 Docker 镜像支持 Rust 包，我们需要对 `src/Dockerfile` 文件进行一些修改。查看这个文件，我们可以看到我们的镜像基于 `python:3.6.13-stretch` 构建。这本质上是一个安装了 Python 的 Linux 环境。当我们看到这一点时，我们意识到我们可以对我们的 Docker 镜像环境有信心。如果我们能在 Linux 上做到这一点，那么在 Docker 镜像上做到这一点的高概率也很高。考虑到这一点，我们在 `src/Dockerfile` 文件中必须安装 Rust 并使用以下代码注册 `cargo`：

```rs
. . .
```

```rs
RUN apt-get update -y
```

```rs
RUN apt-get install -y python3-dev python-dev gcc
```

```rs
# setup rust
```

```rs
RUN curl https://sh.rustup.rs -sSf | bash -s -- -y –profile 
```

```rs
  minimal –no-modify-path
```

```rs
# Add .cargo/bin to PATH
```

```rs
ENV PATH="/root/.cargo/bin:${PATH}"
```

```rs
. . .
```

幸运的是，Rust 非常容易安装。记住，`apt-get install -y python3-dev python-dev gcc` 命令允许我们在使用 Python 时使用编译扩展。现在我们已经这样做，我们可以使用以下代码拉取和编译我们的 Rust 包：

```rs
. . .
```

```rs
# Install the dependencies
```

```rs
RUN pip install --upgrade pip setuptools wheel
```

```rs
RUN pip install -r requirements.txt
```

```rs
RUN pip install -r git_repos.txt
```

```rs
. . .
```

其他一切保持不变。现在我们的图片已经准备好，可以使用以下命令在 `src/` 目录的根目录下构建：

```rs
docker build . -t flask-fib
```

这将重建我们的 Flask 应用程序的 Docker 镜像。在构建过程中可能会跳过一些部分。不用担心——Docker 缓存了镜像构建中未更改的层。这用以下打印输出表示：

```rs
Step 1/14 : FROM python:3.6.13-stretch
```

```rs
 ---> b45d914a4516
```

```rs
Step 2/14 : WORKDIR /app
```

```rs
 ---> Using cache
```

```rs
 ---> b0331f8a005d
```

```rs
Step 3/14 : ADD . /app
```

```rs
 ---> Using cache
```

一旦某个步骤被更改，其后的每个步骤都将重新运行，因为中断的步骤可能会改变后续步骤的结果。请注意，当 `pip` 安装我们的 Rust 包时，构建可能会挂起。这是因为包正在编译。你可能已经注意到，我们每次安装 Rust 包时都必须这样做。在下一章中，我们将探讨更优的发行策略。现在，如果我们在我们部署目录中运行 `docker-compose`，我们将看到我们可以无任何问题地使用我们的 Rust Flask 容器。

# 使用私有 GitHub 仓库进行部署

如果你正在为一个副项目、公司或付费功能编码，你将需要与私有 GitHub 仓库合作。这很合理，因为我们不希望人们免费访问你或你的公司打算收费的仓库。然而，如果我们把我们的 Rust Fibonacci 包的 GitHub 仓库设置为私有，使用`docker image rm YOUR_IMAGE_ID_HERE`命令删除所有我们的 Flask 镜像，然后再次运行我们的`docker build . -t flask-fib`命令，我们会得到以下输出：

```rs
Collecting git+https://github.com/maxwellflitton/flitton-
```

```rs
fib-rs@main
```

```rs
Running command git clone -q 
```

```rs
https://github.com/maxwellflitton/
```

```rs
flitton-fib-rs /tmp/pip-req-build-ctmjnoq0
```

```rs
Cloning https://github.com/maxwellflitton/flitton-fib-rs 
```

```rs
(to revision main) to /tmp/pip-req-build-ctmjnoq0
```

```rs
fatal: could not read Username for 'https://github.com': 
```

```rs
No such device or address
```

这是因为我们正在构建的基于 Linux 的隔离 Docker 镜像没有登录 GitHub，尽管我们已经登录了。因此，正在构建的镜像无法从 GitHub 仓库中拉取包。我们可以通过参数将我们的 GitHub 凭据传递到构建过程中，但这将出现在镜像构建层中。因此，任何可以访问我们的镜像的人都可以查看我们的 GitHub 凭据。这是一个安全隐患。Docker 确实有一些关于传递机密的文档。然而，在撰写本书时，文档稀少且复杂。一个更直接的方法是在镜像外部克隆我们的`flitton-fib-rs`包，并将其传递到 Docker 镜像构建过程中，如下所示：

![](img/B17720_10_02.jpg)

图 10.2 – 私有仓库镜像构建流程

如果我们要使用 GitHub Actions 或 Travis 这样的持续集成工具，那么我们可以运行前面图表中描述的过程，并将 GitHub 凭据作为机密传递。GitHub Actions 和 Travis 以高效和简单的方式处理机密。如果我们本地构建，就像在这个例子中一样，那么我们应该已经登录 GitHub，因为我们在这个项目中直接在 Flask 项目上工作。为了执行前面图表中描述的过程，我们必须执行以下步骤：

1.  构建一个 Bash 脚本，以协调前面图表中描述的过程。

1.  在我们的 Dockerfile 中重新配置 Rust Fib 包的安装。

这是我们使用私有 GitHub 仓库进行我们的 Web 应用程序构建的最直接方法。我们将从查看 Bash 脚本开始。

## 构建一个 Bash 脚本，以协调整个过程

我们的脚本存放在`src/build_image.sh`中。首先，我们必须声明这是一个 Bash 脚本，代码应该在 Flask 应用程序的目录中运行。为此，我们必须使用以下代码更改到包含脚本的目录：

```rs
#!/usr/bin/env bash
```

```rs
SCRIPTPATH="$( cd "$(dirname "$0")" ; pwd -P )"
```

```rs
cd $SCRIPTPATH
```

现在，我们必须使用以下代码克隆我们的包并从仓库中删除我们的`.git`文件：

```rs
git clone https://github.com/maxwellflitton/flitton-fib-
```

```rs
  rs.git
```

```rs
rm -rf ./flitton-fib-rs/.git
```

现在，我们的包只是一个目录。我们准备好构建我们的 Docker 镜像。然而，如果我们这样做，它可能不会工作，因为我们的文件可能被缓存。为了防止这种情况发生，我们可以不带缓存运行我们的构建，然后在构建后使用以下代码删除我们的克隆包：

```rs
docker build . --no-cache -t flask-fib
```

```rs
rm -rf ./flitton-fib-rs
```

我们需要运行这个脚本来构建我们的 Flask 应用程序。然而，如果我们现在就运行构建，它将不会工作，因为我们的 Dockerfile 仍然会尝试从 GitHub 拉取目录。为了解决这个问题，我们将继续进行第二步。

## 在我们的 Dockerfile 中重新配置 Rust Fib 包安装

在我们的`src/Dockerfile`文件中，我们必须删除`RUN pip install -r git_repos.txt`行，因为这会阻止我们的镜像构建尝试从 GitHub 仓库中拉取。现在，我们可以使用以下代码`pip install`安装已传递的本地目录，然后删除它：

```rs
RUN pip install ./flitton-fib-rs
```

```rs
RUN rm -rf ./flitton-fib-rs
```

现在，我们可以通过运行以下命令来构建我们的 Flask 镜像：

```rs
sh build_image.sh
```

这将产生一个长打印输出，最终会告诉我们镜像已成功构建。运行我们的部署`docker-compose`文件将确认这一点。你可能希望从不同的 Git 分支安装我们的包。这可以通过向我们的`src/build_image.sh`文件中添加三行来完成，如下所示：

```rs
. . .
```

```rs
git clone –branch $1 
```

```rs
  https://github.com/maxwellflitton/flitton-fib-rs.git
```

```rs
cd flitton-fib-rs
```

```rs
cd ..
```

```rs
rm -rf ./flitton-fib-rs/.git
```

```rs
. . .
```

在这里，我们克隆了包含分支的仓库，该分支的名称基于传递给脚本的参数。一旦我们完成这个操作，我们可以通过删除`.git`文件来移除 Git 历史记录。

我们现在将 Rust 包完全融合到我们的 Python Web 应用程序中，在 Docker 中。安装我们的 Rust 包时构建镜像的一个额外好处是，它不需要每次使用镜像时都进行编译。

注意

在减少构建方面，我们可以更进一步，尽管这是可选的。你不必这样做来完成这一章。目前，我们正在安装 Rust 并编译我们的 Rust Python 包以进行斐波那契计算。我们可以通过为一系列 Linux 发行版和 Python 版本构建 wheel 来避免每次都需要安装 Rust 和编译。这可以通过拉取 ManyLinux Docker 镜像并使用它们来编译我们的包到多个发行版中完成。

如何将 Rust 设置工具文档中详细说明的步骤应用于用 Rust 编写的 Python `pip`包（请参阅*进一步阅读*部分）。一旦完成这些步骤，你将在`dist`目录中获得一系列的 wheel 文件。将 3.6 版本的 wheel 文件复制粘贴到 Flask 的`src`目录中，并指示 Dockerfile 在构建时将其复制到镜像中。完成这些操作后，你可以直接将`pip install`命令指向复制到镜像构建中的 wheel 文件。安装过程将几乎是瞬间的。

虽然将 Rust 与我们的 Flask 应用程序融合确实很有用，因为我们现在有一个真实世界的例子，展示了我们的 Rust 代码如何在部署环境中使用，但我们还可以更进一步。在下一节中，我们将让我们的 Rust 代码与我们的数据库交互。

# 将 Rust 与数据访问融合

在网络应用程序中，访问数据库是过程的一个重要部分。我们可以导入在`src/data_access.py`文件中创建的`dal`对象，并将其传递给我们的 Rust 函数，通过 Python 执行数据库操作。虽然技术上可行，但这并不是最佳方案，因为我们将不得不浪费时间从数据库查询中提取对象，检查它们，并将它们转换为 Rust 结构体。然后我们必须将 Rust 结构体转换为 Python 对象，然后再将它们插入数据库。这是一大堆与 Python 交互的冗余代码，减少了其速度优势。

由于数据库是 Python 网络应用程序的外部组件，并且它包含有关其模式的信息，我们可以通过使用`diesel` Rust 包完全绕过 Python 的实现，根据实时数据库自动在 Rust 中编写我们的模式和数据库模型。我们还可以使用`diesel`来管理数据库连接。因此，我们可以直接与数据库交互，减少我们对 Python 的依赖，加快我们的代码执行速度，并减少我们需要编写的代码量。为了实现这一点，我们必须执行以下步骤：

1.  设置我们的数据库克隆包。

1.  设置我们的`diesel`环境。

1.  自动生成并配置我们的数据库模型和模式。

1.  在 Rust 中定义我们的数据库连接。

1.  创建一个 Rust 函数，该函数获取所有斐波那契记录并返回它们。

一旦我们完成这些步骤，我们将拥有一个可以添加到我们的 Flask 应用程序构建中并按需使用的与数据库交互的 Rust 包。我们将首先设置我们的数据库克隆包。

## 设置我们的数据库克隆包

我们现在应该熟悉设置标准的 Rust 包用于 Python。对于我们的数据库包，我们将有以下布局：

```rs
├── Cargo.toml
```

```rs
├── diesel.toml
```

```rs
├── rust_db_cloning
```

```rs
│   └── __init__.py
```

```rs
├── setup.py
```

```rs
├── src
```

```rs
│   ├── database.rs
```

```rs
│   ├── lib.rs
```

```rs
│   ├── models.rs
```

```rs
│   └── schema.rs
```

```rs
├── .env
```

你现在应该知道一些这些文件的作用。新文件有以下用途：

+   `database.rs`：包含返回数据库连接的函数

+   `models.rs`：包含定义数据库模型、字段以及数据库表中单个行行为的结构体

+   `schema.rs`：包含数据库表的模式

+   `.env`：包含用于**命令行界面**（CLI）交互的数据库 URL

+   `Diesel.toml`：包含我们`diesel` CLI 的配置

现在，我们可以将注意力转向`setup.py`文件。查看包布局，你应该能够自己定义此文件，我鼓励你尝试一下。以下是一个示例的裸骨`setup.py`文件，它是使此包能够通过`pip`安装所必需的：

```rs
#!/usr/bin/env python
```

```rs
from setuptools import dist
```

```rs
dist.Distribution().fetch_build_eggs(['setuptools_rust'])
```

```rs
from setuptools import setup
```

```rs
from setuptools_rust import Binding, RustExtension
```

```rs
setup(
```

```rs
    name="rust-database-cloning",
```

```rs
    version="0.1",
```

```rs
    rust_extensions=[RustExtension(
```

```rs
        ".rust_db_cloning.rust_db_cloning",
```

```rs
        path="Cargo.toml", binding=Binding.PyO3)],
```

```rs
    packages=["rust_db_cloning"],
```

```rs
    zip_safe=False,
```

```rs
)
```

因此，我们的`rust_db_cloning/__init__.py`文件包含以下代码：

```rs
from .rust_db_cloning import *
```

现在，我们可以转向我们的`Cargo.toml`文件，该文件将列出一些你熟悉的依赖项，以及新的`diesel`依赖项：

```rs
[package]
```

```rs
name = "rust_db_cloning"
```

```rs
version = "0.1.0"
```

```rs
authors = ["maxwellflitton"]
```

```rs
edition = "2018"
```

```rs
[dependencies]
```

```rs
diesel = { version = "1.4.4", features = ["postgres"] }
```

```rs
dotenv = "0.15.0"
```

```rs
[lib]
```

```rs
name = "rust_db_cloning"
```

```rs
crate-type = ["cdylib"]
```

```rs
[dependencies.pyo3]
```

```rs
version = "0.13.2"
```

```rs
features = ["extension-module"]
```

这样，我们已经为我们的包定义了通过`pip`安装的基本内容。它现在还没有安装，因为我们`src/lib.rs`文件中没有任何内容，但在最终步骤中我们将填写这个文件。现在，我们可以继续到下一步，即设置我们的`diesel`环境。

## 设置`diesel`环境

我们将从我们的开发数据库克隆我们的模式，这样我们就可以将 URL 硬编码到我们的`.env`文件中，如下所示：

```rs
DATABASE_URL=postgresql://user:password@localhost:5432/fib
```

由于这个数据库配置永远不会进入生产环境，它仅仅用于从开发数据库生成模式和模型，所以如果这个 URL 落入错误的手中，也没有关系。将这个硬编码到你的 GitHub 仓库中并不是世界末日。考虑到这一点，我们可以在`diesel.toml`文件中使用以下代码来定义我们希望模式打印的位置：

```rs
[print_schema]
```

```rs
file = "src/schema.rs"
```

现在我们已经写下了所有需要的内容，我们可以开始安装和运行`diesel` CLI。在安装和编译`diesel`时可能会遇到编译错误。如果你在阅读这段内容时遇到这种情况，你可以通过切换到 Rust `nightly`来绕过这些编译错误。Rust `nightly`提供了 Rust 的最新版本；然而，它不够稳定。因此，你应该尝试在不切换到`nightly`的情况下遵循这些步骤，但如果发现你需要这样做，你可以通过以下代码安装它来切换到`nightly`：

```rs
rustup toolchain install nightly
```

安装完成后，我们可以使用以下命令切换到`nightly`：

```rs
rustup default nightly
```

你的 Rust 编译现在将在`nightly`模式下运行。回到设置我们的`diesel`环境，我们必须使用以下命令安装`diesel` CLI：

```rs
cargo install diesel_cli --no-default-features 
```

```rs
--features postgres
```

现在，我们可以使用 CLI 结合`.env`文件中的 URL 来与我们的数据库进行交互。

## 自动生成和配置我们的数据库模型和模式

在这一步，我们将与 Docker 中的开发数据库进行交互。考虑到这一点，在继续之前，你需要打开另一个终端，并在`flask-fib`仓库中运行开发`docker-compose`环境。运行此命令将运行我们将要连接的数据库，以便我们可以访问数据库模式和模型。现在 CLI 已经安装，我们可以使用以下命令打印模式：

```rs
diesel print-schema > src/schema.rs
```

终端将不会有打印输出，但如果我们打开`src/schema.rs`文件，我们会看到以下代码：

```rs
table! {
```

```rs
    alembic_version (version_num) {
```

```rs
        version_num -> Varchar,
```

```rs
    }
```

```rs
}
```

```rs
table! {
```

```rs
    fib_entries (id) {
```

```rs
        id -> Int4,
```

```rs
        input_number -> Nullable<Int4>,
```

```rs
        calculated_number -> Nullable<Int4>,
```

```rs
    }
```

```rs
}
```

```rs
allow_tables_to_appear_in_same_query!(
```

```rs
    alembic_version,
```

```rs
    fib_entries,
```

```rs
);
```

在这里，我们可以看到我们的`alembic`版本作为单独的表存在于模式中。这就是`alembic`跟踪迁移的方式。我们还可以看到我们的`fib_entries`表已经被映射。虽然我们可以不使用`diesel` CLI 自己完成这项工作，但它是一个救星，确保模式始终与数据库保持最新。这也在大型、复杂的数据库中节省了时间，并减少了错误。

现在我们已经定义了我们的模式，我们可以使用以下命令来定义我们的模型：

```rs
diesel_ext > src/models.rs
```

这给我们以下代码：

```rs
#![allow(unused)]
```

```rs
#![allow(clippy::all)]
```

```rs
#[derive(Queryable, Debug, Identifiable)]
```

```rs
#[primary_key(version_num)]
```

```rs
pub struct AlembicVersion {
```

```rs
    pub version_num: String,
```

```rs
}
```

```rs
#[derive(Queryable, Debug)]
```

```rs
pub struct FibEntry {
```

```rs
    pub id: i32,
```

```rs
    pub input_number: Option<i32>,
```

```rs
    pub calculated_number: Option<i32>,
```

```rs
}
```

这并不完全完美，我们必须做一些修改。模型没有定义表。`diesel`假设表名只是模型名称的复数形式。例如，如果我们有一个名为*test*的数据模型，那么`diesel`会假设表名为*tests*。然而，对我们来说，情况并非如此，因为我们已经在上一章运行迁移时在我们的 Flask 应用程序中明确定义了我们的表。我们还可以删除两个`allow`宏，因为我们不会使用这个功能。相反，我们将导入我们的模式并在`table`宏中定义它们。经过这次调整后，我们的`src/models.rs`文件应该看起来像这样：

```rs
use crate::schema::fib_entries;
```

```rs
use crate::schema::alembic_version;
```

```rs
#[derive(Queryable, Debug, Identifiable)]
```

```rs
#[primary_key(version_num)]
```

```rs
#[table_name="alembic_version"]
```

```rs
pub struct AlembicVersion {
```

```rs
    pub version_num: String,
```

```rs
}
```

```rs
#[derive(Queryable, Debug, Identifiable)]
```

```rs
#[table_name="fib_entries"]
```

```rs
pub struct FibEntry {
```

```rs
    pub id: i32,
```

```rs
    pub input_number: Option<i32>,
```

```rs
    pub calculated_number: Option<i32>,
```

```rs
}
```

我们的模式和模式现在已准备好在我们的 Rust 包中使用。考虑到这一点，我们可以继续下一步，即定义我们的数据库连接。

## 在 Rust 中定义我们的数据库连接

我们的数据库连接通常会从环境中获取数据库 URL，并使用它来建立连接。然而，这是一个附加到我们的 Flask 应用程序的 Rust 包。没有必要再有一个需要加载的敏感信息。因此，为了避免额外的复杂性和另一个安全漏洞，我们将仅从 Flask 应用程序传递数据库 URL 来建立连接，因为 Flask 应用程序已经在管理配置和加载敏感数据了。我们的整个数据库连接可以在我们的`src/database.rs`文件中处理。首先，我们必须用以下代码导入我们需要的内容：

```rs
use diesel::prelude::*;
```

```rs
use diesel::pg::PgConnection;
```

`prelude`帮助我们使用`diesel`宏，而`PgConnection`是我们将返回以获取数据库连接的东西。有了这个，我们可以用以下代码构建我们的数据库连接函数：

```rs
pub fn establish_connection(url: String) -> PgConnection {
```

```rs
    PgConnection::establish(&url)
```

```rs
      .expect(&format!("Error connecting to {}", url))
```

```rs
}
```

这可以在我们需要数据库连接的任何地方导入。在这个时候，我们可以开始创建一个函数，该函数获取所有记录并以字典的形式返回它们。

## 创建一个 Rust 函数，该函数获取所有斐波那契记录并返回它们

为了避免在这个例子中过度复杂化，我们将所有内容都在`src/lib.rs`文件中完成。然而，建议你在更复杂的包中构建一些模块并将它们导入到`src/lib.rs`文件中。首先，我们将导入构建函数并绑定所需的所有内容，以下代码：

```rs
#[macro_use] extern crate diesel;
```

```rs
extern crate dotenv;
```

```rs
use diesel::prelude::*;
```

```rs
use pyo3::prelude::*;
```

```rs
use pyo3::wrap_pyfunction;
```

```rs
use pyo3::types::PyDict;
```

```rs
mod database;
```

```rs
mod schema;
```

```rs
mod models;
```

```rs
use database::establish_connection;
```

```rs
use models::FibEntry;
```

```rs
use schema::fib_entries;
```

在这里，导入的顺序很重要。我们直接导入带有宏使用的`diesel` crate。因此，像`database`和`schema`这样的文件不会出错，因为它们使用了`diesel`宏。在我们的例子中，我们没有使用`dotenv`，因为我们是从 Python 系统传递数据库 URL。然而，如果你想要从环境中获取数据库 URL，了解这一点是有用的。然后，我们可以导入所需的`pyo3`宏和结构体，以及我们定义的结构体和函数。有了这些导入，我们可以用以下代码定义我们的`get_fib_entries`函数：

```rs
#[pyfunction]
```

```rs
fn get_fib_entries(url: String, py: Python) -> Vec<&PyDict> 
```

```rs
{
```

```rs
   let connection = establish_connection(url);  
```

```rs
   let fibs = fib_entries::table
```

```rs
       .order(fib_entries::columns::input_number.asc())
```

```rs
       .load::<FibEntry>(&connection)
```

```rs
       .unwrap();
```

```rs
   let mut buffer = Vec::new();
```

```rs
   for i in fibs {
```

```rs
       let placeholder = PyDict::new(py);
```

```rs
       placeholder.set_item("input number", 
```

```rs
         i.input_number.unwrap());
```

```rs
       placeholder.set_item("fib number", 
```

```rs
          i.calculated_number.unwrap());
```

```rs
       buffer.push(placeholder);
```

```rs
   }
```

```rs
}
```

使用 Python 构建字典列表并不新鲜，函数的定义也是如此。然而，新的地方在于建立连接，使用模式列对其进行排序，并将其作为`FibEntry`结构体的列表加载。我们将连接的引用传递到查询中，并在它返回结果时解包它。如果需要，我们可以向它链式调用更多函数，例如`.filter`。`diesel`文档很好地涵盖了你可以执行的不同类型的查询和插入。一旦我们完成这些，我们可以使用以下代码将其添加到我们的`rust_db_cloning`模块中：

```rs
#[pymodule]
```

```rs
fn rust_db_cloning(py: Python, m: &PyModule) 
```

```rs
    -> PyResult<()> {
```

```rs
    m.add_wrapped(wrap_pyfunction!(get_fib_enteries));
```

```rs
    Ok(())
```

```rs
}
```

这样，我们的代码就准备好上传到 GitHub 仓库并在我们的 Flask 应用程序中使用。

现在，我们可以在定义 Dockerfile 之前快速测试我们的包是否工作。首先，我们需要在我们的 Flask 应用程序虚拟环境中`pip`安装它。这是你可能会遇到编译问题的另一个地方。为了解决这个问题，你可能需要切换到 Rust `nightly`来`pip`安装你刚刚构建的包。一旦我们的包被安装，我们可以通过向 Flask 应用程序添加一个简单的`get`视图来检查它。在我们的 Flask 应用程序的`src/app.py`文件中，我们可以使用以下代码导入我们的函数：

```rs
from rust_db_cloning import get_fib_entries
```

现在，我们可以使用以下代码定义我们的`get`视图：

```rs
@app.route("/get")
```

```rs
def get():
```

```rs
    return str(get_fib_entries(dal.url))
```

记得在上一章中，我们定义了`dal`的`url`属性，使用的是从`.yml`配置文件加载的`GlobalParams`中的 URL。我们必须将其转换为字符串；否则，Flask 序列化将无法处理它。在开发`docker-compose`环境中运行此代码将给出以下输出：

![图 10.3 – 从我们的 Flask 应用程序获取的简单视图](img/B17720_10_03.jpg)

图 10.3 – 从我们的 Flask 应用程序获取的简单视图

你可能会有不同的数字，这取决于你数据库中的内容。然而，我们这里有一个 Rust 包，它能跟上数据库的变化，并能直接与数据库交互。现在，这个功能在我们的开发环境中已经工作，我们可以开始打包我们的 Rust `nightly`包以进行部署。

# 在 Flask 中部署 Rust nightly

要打包我们的`nightly`数据库 Rust 包以便部署，我们必须在我们的构建 Bash 脚本中添加我们 GitHub 仓库的另一个副本，安装`nightly`，并在使用`pip`安装数据库包时切换到它。你可能会猜到，通过在我们的 Bash 脚本中克隆数据库 GitHub 仓库，我们将实现什么。

作为参考，我们的`src/build_image.sh`文件将采用以下代码的形式：

```rs
. . .
```

```rs
git clone https://github.com/maxwellflitton/
```

```rs
flitton-fib-rs.git
```

```rs
git clone https://github.com/maxwellflitton/
```

```rs
rust-db-cloning.git
```

```rs
rm -rf ./flitton-fib-rs/.git
```

```rs
rm -rf ./rust-db-cloning/.git
```

```rs
docker build . --no-cache -t flask-fib
```

```rs
rm -rf ./flitton-fib-rs
```

```rs
rm -rf ./rust-db-cloning
```

在这里，我们可以看到我们仅仅添加了克隆`rust-db-cloning`仓库的代码，从那个`rust-db-cloning`仓库中移除了`.git`文件，然后在镜像构建完成后移除了`rust-db-cloning`仓库。当我们谈到我们的 Dockerfile 时，这些步骤将保持不变。唯一的区别是，在安装我们的常规 Rust 包之后，我们安装`nightly`，切换到它，然后安装我们的数据库包。这可以通过以下代码实现：

```rs
. . .
```

```rs
RUN pip install ./flitton-fib-rs
```

```rs
RUN rm -rf ./flitton-fib-rs
```

```rs
RUN rustup toolchain install nightly
```

```rs
RUN rustup default nightly
```

```rs
RUN pip install ./rust-db-cloning
```

```rs
RUN rm -rf ./rust-db-cloning
```

```rs
. . .
```

尽管一个是用常规 Rust 编译的，而另一个是用 Rust `nightly`编译的，但当应用程序运行时，它们都会正常运行。构建这个镜像并在部署`docker-compose`环境中运行它将显示容器将处理 Rust 计算视图并从数据库视图中获取它而没有任何问题。有了这个，我们现在拥有了将 Rust 融合到 Python Web 应用程序并部署它们的全部工具。

# 摘要

在本章中，我们将所有 Rust 融合技能用于构建打包到 Python Web 应用程序 Docker 镜像中的包。我们将 Rust 包直接附加到 Web 应用程序，然后到 Celery 工作进程，当我们要求 Web 应用程序计算斐波那契数时，这导致了显著的加速。

然后，我们改变了我们的构建过程，在构建我们的 Python Web 应用程序镜像时从私有 GitHub 仓库中获取 Rust 包。最后，我们用 Rust 直接连接到数据库，并使用 Rust `nightly`来编译它。我们设法将其包含在我们的 Python Web 应用程序 Docker 构建中。这使我们不仅能够将 Rust 融合到可部署的 Web 应用程序中，还能使用 Rust `nightly`和数据库来解决问题

考虑到这一点，我们现在可以使用这本书中学到的知识来生产 Web 应用程序。你现在可以开始用 Rust 编码，并将你的 Rust 包插入到可以部署在 Docker 中的现有 Python Web 应用程序中，而无需对 Python Web 应用程序的构建过程进行重大更改。

在你的日常工作中明天使用 Rust 来解决速度瓶颈或确保代码在实时 Python Web 应用程序中的一致性和安全性，这是你可以做到的。你现在可以将最快的内存安全编程语言带入你的 Python 项目中，而无需彻底改造现有的系统。你现在能够弥合维护现有经过验证的系统与前沿语言之间的差距。在下一章和最后一章中，我们将介绍一些最佳实践。但现在，你知道了改变你的项目或组织的核心概念。

# 问题

1.  直接在 Rust 中连接到数据库如何减少代码？

1.  为什么我们不能直接在我们的 Docker 镜像构建 Dockerfile 中传递登录凭证？

1.  在本章中，我们没有进行任何迁移。我们是如何将数据库的模型和模式映射到我们的 Rust 模块中，以及我们如何继续跟踪数据库的变化？

1.  为什么我们将数据库 URL 传递给我们的 Rust 数据库包，而不是从配置文件或环境中加载它？

1.  如果我们要将 Rust 与 Django、bottle 或 FastAPI Python Web 应用融合，我们是否需要做任何额外的事情？

# 答案

1.  使用 Rust 直接连接到数据库减少了我们必须编写的代码量，因为我们不需要检查 Python 数据库调用返回的 Python 对象。我们也不需要在将数据插入数据库之前将数据打包成 Python 对象。这实际上消除了我们在与数据库交互时必须编写的整个代码层。

1.  如果有人获取了我们的镜像，他们可以访问构建的层。结果，他们可以访问传递给构建的参数。这意味着他们可以看到我们用来登录的凭证。

1.  我们使用了`diesel`包来连接到数据库，并自动根据连接的数据库打印出模式和模型。我们可以重复这样做，以保持与新的数据库迁移同步。

1.  我们必须记住，我们的 Rust 数据库包是我们 Python Web 应用的补充。我们的 Python Web 应用已经加载了数据库 URL。将凭证加载到我们的包中只是增加了另一个安全漏洞的可能性，而没有带来任何优势。

1.  不，我们的融合方法与`pip`安装过程和数据库映射过程完全分离。

# 进一步阅读

+   Rust 的 Diesel 文档（2021）：Crate Diesel: [`diesel.rs`](https://diesel.rs)

+   Rust 的设置工具文档（2021）：使用 wheels 分发 Rust Python 包：[`pypi.org/project/setuptools-rust/`](https://pypi.org/project/setuptools-rust/)

+   ManyLinux GitHub (2021): [`github.com/pypa/manylinux`](https://github.com/pypa/manylinux%0D)
