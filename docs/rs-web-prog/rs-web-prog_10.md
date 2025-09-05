

# 第十章：在 AWS 上部署我们的应用程序

在许多教程和教育材料中，部署很少被涉及。这是因为有很多可变的部分，过程可能相当脆弱。在提及部署时，参考其他资源可能更为方便。

在本章中，我们将介绍如何在 **Amazon Web Services** (**AWS**) 上的服务器上自动化部署，并从那里构建和连接数据库。必须强调的是，部署和云计算是很大的主题——关于它们有整本书的论述。

在本章中，我们将达到可以部署和运行我们的应用程序以供他人使用的地方。学习如何在服务器上部署应用程序是最后一步。这就是您将您一直在开发的应用程序转变为一个实用现实的过程，可以供全世界的人们使用。

在本章中，我们将涵盖以下主题：

+   设置我们的构建环境

+   使用 Docker 管理我们的软件

+   在 AWS 上部署我们的应用程序

到本章结束时，您将能够将您的代码打包成 Docker 镜像，并在 AWS 上的服务器实例上部署，以便其他用户可以访问。您还将能够使用 **Terraform** 配置基础设施，这是一个用于将服务器和数据库等云计算基础设施定义为代码的工具。完成本章后，您将拥有几个构建脚本，这些脚本使用 Terraform 创建构建和部署服务器，将 Terraform 构建中的数据传递到配置文件中，**SSH** 到这些服务器，并运行一系列命令，导致数据库迁移并在我们的服务器上启动 Docker 容器。

# 技术要求

在本章中，我们将基于第 *第九章* 中构建的代码，*测试我们的应用程序端点和组件* 进行扩展。您可以在以下网址找到它：[`github.com/PacktPublishing/Rust-Web-Programming-2nd-Edition/tree/main/chapter09/building_test_pipeline`](https://github.com/PacktPublishing/Rust-Web-Programming-2nd-Edition/tree/main/chapter09/building_test_pipeline)。

本章的代码可以在以下网址找到：[`github.com/PacktPublishing/Rust-Web-Programming-2nd-Edition/tree/main/chapter10`](https://github.com/PacktPublishing/Rust-Web-Programming-2nd-Edition/tree/main/chapter10)。

本章还有以下要求：

+   我们将使用 Terraform 自动化服务器的构建。因此，我们需要使用以下网址安装 Terraform：[`learn.hashicorp.com/tutorials/terraform/install-cli`](https://learn.hashicorp.com/tutorials/terraform/install-cli)。

+   当我们使用 Terraform 时，它将对 AWS 基础设施进行调用。我们需要 AWS 认证，这将通过 AWS 客户端来完成。在本章中，我们将使用 AWS 客户端，可以使用以下网址安装：[`docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html`](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)。

+   你还需要一个 Docker Hub 账户，这样我们就可以打包和部署我们的应用程序。这可以在以下网址找到：[`hub.docker.com/`](https://hub.docker.com/)。

+   由于你将在服务器上部署应用程序，你需要注册一个 AWS 账户。这可以在以下网址完成：[`aws.amazon.com/`](https://aws.amazon.com/)。

# 设置我们的构建环境

到目前为止，我们一直使用 `cargo run` 命令运行我们的应用程序。这一直运行得很好，但你可能已经注意到我们的应用程序速度并不快。事实上，当我们尝试登录到应用程序时，它相对较慢。这似乎与我们学习 Rust 来开发更快应用程序的目标相悖。

到目前为止，它看起来并不快。这是因为我们没有运行应用程序的优化版本。我们可以通过添加 `--release` 标签来实现这一点。结果，我们使用以下命令运行我们的优化应用程序：

```rs
cargo run --release config.yml
```

在这里，我们注意到编译需要更长的时间。每次我们更改代码，以及在开发过程中运行这个命令，都不是理想的；因此，我们一直使用 `cargo run` 命令在调试模式下构建和运行。然而，现在我们的优化应用程序正在运行，我们可以看到登录过程要快得多。虽然我们可以在本地运行服务器，但我们的目标是部署我们的应用程序到服务器上。为了在我们的应用程序上运行，我们必须在 Docker 镜像中构建我们的应用程序。为了确保我们的 Docker 镜像构建过程顺利，我们将在 AWS 上使用在线计算单元。在我们运行构建过程之前，我们需要执行以下步骤：

1.  为 AWS **安全外壳**（**SSH**）设置 AWS **弹性计算** **云**（**EC2**）密钥

1.  为我们的本地计算机设置 AWS 客户端

1.  编写一个 Terraform 脚本来构建我们的构建服务器

1.  编写一个 Python 脚本来管理构建

1.  编写一个 Bash 脚本来在服务器上编排构建过程

一旦我们完成了上述步骤，我们就能运行一个自动化的流水线，该流水线会在 AWS 上创建一个构建 EC2 服务器，然后构建我们的 Rust 应用程序。首先，让我们开始创建服务器的 SSH 密钥。

## 为 AWS EC2 实例设置 AWS SSH 密钥

如果我们想在服务器上运行命令，我们将不得不通过**超文本传输协议**（**HTTP**）使用 SSH 协议连接到服务器。然而，我们不能让任何人访问我们的服务器，因为它将不安全。为了阻止任何人连接到我们的服务器并运行他们想要的任何命令，我们只允许拥有 SSH 密钥的用户连接到我们的服务器。为了创建我们的密钥，我们需要登录到 AWS 控制台并导航到我们的 EC2 仪表板，这可以通过搜索框访问，如下面的屏幕截图所示：

![图 10.1 – 使用搜索框导航到 EC2 仪表板](img/Figure_10.1_B18722.jpg)

图 10.1 – 使用搜索框导航到 EC2 仪表板

注意

必须注意，AWS 可能看起来与本章中的截图不同，因为 AWS 不断更改 UI。然而，基本概念将是相同的。

一旦我们导航到 EC2 仪表板，我们可以在视图左侧面板的**网络与安全**部分导航到**密钥对**，如下面的屏幕截图所示：

![图 10.2 – 导航到密钥对](img/Figure_10.2_B18722.jpg)

图 10.2 – 导航到密钥对

一旦我们导航到**密钥对**部分，将有一个您已拥有的密钥对列表。如果您之前构建过 EC2 实例，您可能已经看到一些列出了。如果您之前从未构建过 EC2 实例，那么列表将为空。在屏幕右上角，您可以通过点击以下屏幕截图所示的**创建密钥对**按钮来创建一个密钥：

![图 10.3 – 允许创建密钥对的按钮](img/Figure_10.3_B18722.jpg)

图 10.3 – 允许创建密钥对的按钮

一旦我们点击了这个按钮，我们将看到以下表单：

![图 10.4 – 创建密钥对表单](img/Figure_10.4_B18722.jpg)

图 10.4 – 创建密钥对表单

在前面的屏幕截图中，我们可以看到我们已将我们的`remotebuild`密钥命名；我们还声明我们的密钥具有`.pem`格式，并且位于`.ssh`目录下。在`.ssh`目录内，我们可以使用以下命令创建一个`keys`目录：

```rs
mkdir "$HOME"/.ssh/keys/
```

`"$HOME"`环境变量始终在 Bash shell 中可用，它表示用户的家目录。这意味着如果我们将这些 SSH 密钥存储在我们刚刚创建的目录中，其他用户在计算机上以不同的用户名登录时无法访问这些密钥。我们现在可以导航到我们的密钥已下载的位置，并使用以下命令将其复制到`keys`目录：

```rs
cp ./remotebuild.pem "$HOME"/.ssh/keys/
```

然后，我们必须更改密钥的权限，以便只有文件的所有者才能使用 600 代码读取文件，这样我们才能使用密钥进行 SSH 操作，以下命令即可实现：

```rs
chmod 600 "$HOME"/.ssh/keys/remotebuild.pem
```

现在我们有一个 SSH 密钥存储在我们的 `.ssh/keys` 目录中，我们可以使用正确的权限访问这个密钥来访问我们创建的服务器。现在我们有了这个，我们需要通过设置我们的 AWS 客户端来获取对 AWS 服务的程序访问。

## 设置我们的 AWS 客户端

我们将使用 Terraform 自动化我们的服务器构建。为了做到这一点，Terraform 将调用 AWS 基础设施。这意味着我们必须在我们的本地机器上安装 AWS 客户端。在我们配置客户端之前，我们需要获取一些程序用户密钥。如果我们没有获取程序访问权限，我们的代码将不会被授权在我们的 AWS 账户上构建基础设施。获取用户密钥的第一步是通过搜索框导航到**IAM**部分，如下面的截图所示：

![图 10.5 – 导航到 IAM 部分](img/Figure_10.5_B18722.jpg)

图 10.5 – 导航到 IAM 部分

一旦我们导航到**IAM**部分，我们将看到以下布局：

![图 10.6 – IAM 部分视图](img/Figure_10.6_B18722.jpg)

图 10.6 – IAM 部分视图

在前面的截图中，我们可以看到我的用户启用了**多因素认证**（**MFA**），并且我定期更换访问密钥。如果你是 AWS 的新手，这个清单可能不会令人满意，建议你遵循 AWS 提供的安全建议。我们现在必须访问视图左侧的**用户**选项，并通过点击屏幕右上角的**添加用户**按钮来创建新用户，如下面的截图所示：

![图 10.7 – 创建用户](img/Figure_10.7_B18722.jpg)

图 10.7 – 创建用户

然后，我们可以创建一个用户。用户的名字不重要，但你必须确保他们有程序访问权限，通过检查**访问密钥 - 程序访问**选项，如下面的截图所示：

![图 10.8 – 创建用户的第一个阶段](img/Figure_10.8_B18722.jpg)

图 10.8 – 创建用户的第一个阶段

一旦我们突出显示了程序访问并定义了用户名，我们就可以继续到权限部分，如下面的截图所示：

![图 10.9 – 定义权限](img/Figure_10.9_B18722.jpg)

图 10.9 – 定义权限

我们可以看到，我们已经赋予了用户 `AdministratorAccess` 权限，这将使我们能够创建和销毁服务器和数据库。用户创建过程中的其余步骤都很简单，您只需点击 **下一步** 即可完成。一旦用户创建成功，您将获得一个访问密钥和一个秘密访问密钥。请注意，将这些密钥记录在安全的位置，例如密码管理器中，因为您将无法再次在 AWS 网站上看到您的秘密访问密钥，并且在我们配置本地计算机上的 AWS 客户端时需要它们。现在我们已经有了用户密钥，我们可以使用以下命令配置我们的 AWS 客户端：

```rs
aws configure
```

然后 AWS 客户端将提示您在需要时输入用户密钥。完成此操作后，您的 AWS 客户端配置完成，我们就可以在本地计算机上以编程方式使用 AWS 功能。现在，我们已经准备好在下一节开始使用 Terraform 创建服务器。

## 设置 Terraform 构建

当涉及到在 AWS 上构建基础设施时，我们可以在 EC2 控制台中简单地点击和选择。然而，这并不是我们所希望的。如果您像我一样，当我点击一系列配置设置时，除非我记录下来，否则我会忘记我做了什么，而且说实话，记录您点击的内容并不是您期待的事情。即使您比我更好，您会记录下来，但当您更改配置时，您可能会忘记返回更新该更改的文档。点击和选择也很耗时。如果我们想创建一些基础设施，然后在一周后销毁它，然后在一个月后再重新创建它，如果我们必须点击和选择，我们可能会不愿意去操作它，而且我们的服务器账单会更高。这就是 **基础设施即代码**（**IaC**）发挥作用的地方。我们仍然需要像前几节那样进行一些点击和选择。如果我们不点击和选择来设置程序访问权限，我们就无法进行任何程序访问。

现在我们有了程序访问权限，我们可以构建 `build` 目录，该目录应位于我们的 `web_app` 和 `front_end` 目录旁边。在 `build` 目录中，我们可以在 `build/main.tf` 文件中定义构建服务器的基础设施。必须注意的是，`.tf` 扩展名是 Terraform 文件的标准扩展名。首先，我们使用以下代码定义正在使用哪个版本的 Terraform：

```rs
terraform {
  required_version = ">= 1.1.3"
}
```

现在我们已经定义了 Terraform 版本，我们可以声明正在使用 AWS 模块。Terraform 注册表中包含各种平台的模块，包括 Google Cloud 和 Microsoft Azure。任何人都可以构建模块并抽象基础设施，以便在 Terraform 注册表中下载。我们的 AWS 模块使用声明采用以下形式：

```rs
provider "aws" {
    version = ">= 2.28.1"
    region = "eu-west-2"
}
```

如果另一个区域更适合您，您可能想要选择不同的区域。您可以通过以下链接找到 AWS 上可用于 EC2 的所有区域：[`docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html`](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html)。

我只是用 `"eu-west-2"` 来进行演示。现在我们可以用以下代码构建我们的 EC2 实例：

```rs
resource "aws_instance" "build_server" {
    ami = "ami-0fdbd8587b1cf431e"
    instance_type = "t2.medium"
    key_name = "remotebuild"
    user_data = file("server_build.sh")
    tags = {
      Name = "to-do build server"
    }
}
```

`resource` 声明我们正在定义一个要构建的资源，而 `aws_instance` 表明我们正在使用 AWS 模块中的 EC2 实例模板。可以通过以下链接找到可用的 AWS Terraform 模块及其文档：[`registry.terraform.io/providers/hashicorp/aws/latest/docs`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)。

`build_server` 是我们给它起的名字。我们可以在 Terraform 脚本的任何地方引用 `build_server`，Terraform 将确定资源构建的顺序，以确保所有引用都被考虑在内。我们可以看到我们引用了上一节中定义的 `"remotebuild"` 键。如果我们想的话，我们可以创建多个可以通过一个密钥访问的 EC2 实例。我们还声明了名称，这样当我们查看我们的 EC2 实例时，我们就知道服务器是用来做什么的。我们还必须注意，`user_data` 是在新的 EC2 服务器构建完成后将运行的 Bash 脚本。`ami` 部分是对正在使用的操作系统类型和版本的引用。除非您使用的是相同的区域，否则请不要直接复制示例中的我的 **Amazon Machine Image** (**AMI**) ID，因为 AMI ID 可能会根据区域而变化。如果您想找到 AMI ID，请转到您的 EC2 仪表板并点击 **启动实例**，这将导致以下窗口：

![图 10.10 – 启动实例](img/Figure_10.10_B18722.jpg)

图 10.10 – 启动实例

在这里，我们可以看到我们选择了 **Amazon Linux**。您必须选择这个选项；否则，您的构建脚本将无法工作。如果我们放大查看，我们可以看到 AMI ID 是可见的，如下所示：

![图 10.11 – 获取服务器的 AMI ID](img/Figure_10.11_B18722.jpg)

图 10.11 – 获取服务器的 AMI ID

这将是您想要启动的区域的 Amazon Linux 操作系统的 AMI ID。您也可以看到没有任何阻止您在其他 Terraform 项目中使用其他操作系统的。如果您以前构建过 EC2 实例，您将知道除非我们将弹性 IP 绑定到 EC2 实例，否则 IP 地址是随机的。我们将不得不生成我们 EC2 实例的 IP 地址输出，这样我们就可以连接到它。我们的输出是用以下代码定义的：

```rs
output "ec2_global_ips" {
  value = ["${aws_instance.build_server.*.public_ip}"]
}
```

在这里，我们可以看到我们使用`aws_instance.build_server`引用我们的构建服务器。关于 Terraform 输出的进一步阅读可以在*进一步阅读*部分找到。到目前为止，我们的 Terraform 构建几乎完成了。我们必须记住，我们需要构建一个`server_build.sh`脚本，这个脚本将在 EC2 实例构建完成后在 EC2 实例上运行。在我们的`/build/server_build.sh`文件中，我们可以使用以下代码安装服务器所需的基本要求：

```rs
#!/bin/bash
sudo yum update -y
sudo yum install git -y
sudo yum install cmake -y
sudo yum install tree -y
sudo yum install vim -y
sudo yum install tmux -y
sudo yum install make automake gcc gcc-c++ kernel-devel -y
```

使用前面的包，我们将能够使用`tree`在服务器上导航并查看文件树；我们还将能够执行`git`操作，打开文件并使用`vim`编辑它们，如果需要的话，通过一个终端打开多个面板。其他包使我们能够编译我们的 Rust 代码。我们还必须注意，我们在每个安装中都附加了一个`-y`标签。这是告诉计算机跳过输入提示并输入默认答案。这意味着我们可以没有问题地在后台运行此脚本。现在我们必须使用以下代码安装 PostgreSQL 驱动程序：

```rs
sudo amazon-linux-extras install postgresql10 vim epel -y
sudo yum install -y postgresql-server postgresql-devel -y
```

我们几乎拥有了我们所需要的一切。在下一节中，我们将使用 Docker 构建和打包我们的应用程序。这可以通过以下代码完成：

```rs
sudo amazon-linux-extras install docker
sudo service docker start
sudo usermod -a -G docker ec2-user
```

在这里，我们可以看到我们安装了 Docker，启动了 Docker 服务，然后使用 Docker 服务注册我们的用户，这样我们就不必在每一个 Docker 命令中使用`sudo`了。既然我们已经安装了 Docker，我们不妨为了完整性安装`docker-compose`。这可以在我们脚本的末尾通过以下代码完成：

```rs
sudo curl -L "https://github.com/docker/compose/releases
/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)"
-o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

`curl`命令下载`docker-compose`。然后我们使用`chmod`命令使`docker-compose`可执行。构建脚本几乎完成了。然而，运行构建脚本中定义的所有命令将需要一段时间。有可能在构建脚本完成之前我们能够通过 SSH 连接到我们的服务器。因此，我们应该将一个`FINISHED`字符串写入一个文件，以通知其他进程服务器已经安装了所有已安装的包。我们可以用以下代码写入我们的标志：

```rs
echo "FINISHED" > home/ec2-user/output.txt
```

我们现在已经构建了将要为我们构建应用程序以进行部署的基础设施。在下一节中，我们将构建脚本，这些脚本将使用我们配置的构建服务器来编排我们的应用程序构建。

## 编写我们的 Python 应用程序构建脚本

从技术上讲，我们可以尝试将我们的应用程序构建脚本放入我们在上一节中编写的相同脚本中。然而，我们希望保持我们的脚本独立。例如，如果我们打算使用一个`build/run_build.py`脚本，我们首先通过以下代码导入我们需要的所有内容：

```rs
from subprocess import Popen
from pathlib import Path
import json
import time
DIRECTORY_PATH = Path(__file__).resolve().parent
```

现在我们有了`build`目录的绝对路径，我们可以通过`Popen`类运行 Bash 命令，并且我们可以加载 JSON。现在我们已经导入了所有内容，我们可以使用以下代码运行我们的 Terraform 命令：

```rs
init_process = Popen(f"cd {DIRECTORY_PATH} && terraform init",
                     shell=True)
init_process.wait()
apply_process = Popen(f"cd {DIRECTORY_PATH} && terraform apply",
                      shell=True)
apply_process.wait()
```

在每个命令中，我们进入目录路径然后执行 Terraform 命令。然后我们等待命令完成再继续执行下一个命令。虽然 Python 是一个慢速、简单且非强类型的语言，但在这里它增加了很多功能。我们可以同时运行多个 Bash 命令，并在需要时稍后等待它们。我们还可以轻松地从进程提取数据，操作这些数据并将其输入到另一个命令中。前一个片段中执行的两个 Terraform 命令是 `init` 和 `apply`。`init` 命令设置 Terraform 状态以记录正在发生的事情并下载我们需要的模块，在这个例子中是 AWS。`apply` 命令运行 Terraform 构建，这将构建我们的 EC2 服务器。

一旦我们的 EC2 实例构建完成，我们可以获取 Terraform 的输出并将其写入 JSON 文件，然后使用以下代码从该 JSON 文件加载数据：

```rs
produce_output = Popen(f"cd {DIRECTORY_PATH} && terraform
                        output -json > {DIRECTORY_PATH}/
                        output.json", shell=True)
produce_output.wait()
with open(f"{DIRECTORY_PATH}/output.json", "r") as file:
    data = json.loads(file.read())
server_ip = data["ec2_global_ips"]["value"][0][0]
```

现在我们已经获得了服务器 IP，我们可以 SSH 连接到这台服务器并让它执行我们的构建。然而，可能会存在一些并发问题。在 Terraform 构建完成但服务器尚未准备好接受连接的短暂时间内，因此，我们只需要脚本等待一小段时间后再继续执行以下代码：

```rs
print("waiting for server to be built")
time.sleep(5)
print("attempting to enter server")
```

完成这些后，我们需要将我们的服务器 IP 传递给另一个管理构建的 Bash 脚本，然后在之后使用以下代码销毁服务器：

```rs
build_process = Popen(f"cd {DIRECTORY_PATH} &&
                      sh ./run_build.sh {server_ip}",
                      shell=True)
build_process.wait()
destroy_process = Popen(f"cd {DIRECTORY_PATH} &&
                        terraform destroy", shell=True)
destroy_process.wait()
```

我们现在已经构建了构建的编排和定义构建基础设施的 Terraform 脚本。现在我们必须构建最终的构建脚本，该脚本将在服务器上运行构建命令。

## 编写我们的 Bash 部署脚本

我们的 Bash 部署脚本必须接受构建服务器的 IP 地址，SSH 连接到服务器，并在服务器上运行一系列命令。它还必须将我们的代码复制到服务器上进行构建。我们可以从前面的代码中看到，我们可以在 `/build/run_build.sh` 文件中构建我们的构建 Bash 脚本。首先，我们从标准的样板代码开始：

```rs
#!/usr/bin/env bash
SCRIPTPATH="$( cd "$(dirname "$0")" ; pwd -P )"
cd $SCRIPTPATH
```

使用这个样板代码，我们声明这是一个 Bash 脚本，并且其余的 Bash 代码将在 `build` 目录中运行。然后我们使用以下代码上传 Rust 应用程序的代码：

```rs
rm -rf ../web_app/target/
scp -i "~/.ssh/keys/remotebuild.pem" -r
../web_app ec2-user@$1:/home/ec2-user/web_app
```

在这里，我们可以看到我们移除了 `target` 目录。正如我们所记得的，`target` 目录是在我们构建 Rust 应用程序时创建的；我们不需要从本地构建上传构建文件。然后我们使用 `scp` 命令复制我们的 Rust 代码。我们访问脚本传入的第一个参数，即 `$1`。记住，我们传入的是 IP 地址，所以 `$1` 是 IP 地址。然后我们通过以下代码 SSH 连接到我们的服务器并在该服务器上运行命令：

```rs
ssh -i "~/.ssh/keys/remotebuild.pem" -t ec2-user@$1 << EOF
  until [ -f ./output.txt ]
  do
      sleep 2
  done
  echo "File found"
  curl https://sh.rustup.rs -sSf | bash -s -- -y
  source ~/.cargo/env
  cd web_app
  cargo build --release
EOF
```

在这里，我们可以看到我们在每次迭代中循环休眠 2 秒钟，直到`output.txt`文件出现。一旦`output.txt`文件出现，我们就知道 Terraform 的构建脚本已经完成，我们可以开始我们的构建。我们通过在控制台回显`"File found"`来发出这个信号。然后我们安装 Rust，使用`source`命令将我们的`cargo`命令加载到我们的 shell 中，移动到`web_app`目录，并构建我们的 Rust 应用程序。

注意

如果需要，我们可以通过在`run_build.sh`脚本中使用`jq`命令并插入以下代码来去除 Python 依赖：

```rs
. . .
SCRIPTPATH="$( cd "$(dirname "$0")" ; pwd -P )"
cd $SCRIPTPATH
terraform init
terraform apply
terraform output -json > ./output.json
IP_ADDRESS=$(jq --raw-output '.ec2_global_ips.value[0][0]' output.json)
echo $IP_ADDRESS
echo "waiting for server to be built"
sleep 5
echo "attempting to enter server"
rm -rf ../web_app/target/
scp -i "~/.ssh/keys/remotebuild.pem" -r
../web_app ec2-user@$IP_ADDRESS:/home/ec2-user/web_app
. . .
```

需要注意的是，对`$1`变量的引用会被替换为`$IP_ADDRESS`。

要在没有 Python 依赖的情况下运行我们的管道，我们只需要运行以下命令：

```rs
Sh run_build.sh
```

然而，在接下来的章节中，我们将依赖于我们的 Python 脚本。

现在我们已经构建了构建管道，我们可以使用以下命令来运行它：

```rs
python3 run_build.py
```

警告

Terraform 有时可能会表现得有些情绪化；如果它失败了，你可能需要再次运行它。有时，我不得不运行 Terraform 三次以上才能完全正常工作。Terraform 会存储每一个动作，所以请不要担心——再次运行 Python 脚本不会导致服务器重复创建。

当运行此命令时，你将在过程中的三个不同点被提示输入`yes`。第一次是为了批准使用 Terraform 构建服务器，第二次是为了将服务器的 IP 地址添加到已知主机列表以批准 SSH 连接，最后一次是为了批准服务器的销毁。通常来说，在这个书中展示打印输出是有意义的；然而，这里的打印输出很长，可能需要占用多页。此外，打印输出是显而易见的。Terraform 明确说明了正在构建的内容，复制和构建过程也是详尽的，使用 Terraform 销毁服务器的过程也会打印出来。当你运行这个构建管道时，会发生什么将非常清楚。你也可能注意到创建了一系列 Terraform 文件。这些文件跟踪我们在 AWS 平台上构建的资源的状态。如果你删除这些状态文件，你就无法知道构建了什么，并且会创建重复的服务器。它还会阻止 Terraform 进行清理。在我工作的地方，在撰写本文时，我们使用 Terraform 构建用于计算地理区域财务损失风险的大规模数据模型。正在处理的数据可以超过每块数太字节。我们使用 Terraform 启动一系列强大的计算机，将数据通过它们运行（这可能需要几天时间），然后在完成时关闭它们。需要多个人监控这个过程，因此我们的 Terraform 状态存储在一个`state.tf`文件中：

```rs
terraform {
    backend "s3" {
    bucket = "some_bucket"
    key    = "some/ptaht/terraform.tfstate"
    region = "eu-west-1"
  }
}
```

需要注意的是，你的账户需要有权访问定义的存储桶。

现在，是时候构建我们的前端应用程序了。看看我们刚刚所做的工作，我们所需做的只是将上传我们的前端代码到构建服务器并构建前端应用程序的步骤添加到 `/build/run_build.sh` 文件中。在这个阶段，你应该能够自己编写这段代码。现在，停止阅读并尝试构建它将是一个很好的时间利用方式。如果你已经尝试过，它应该看起来像这里显示的代码：

```rs
rm -rf ../front_end/node_modules/
scp -i "~/.ssh/keys/remotebuild.pem" -r
../front_end ec2-user@$1:/home/ec2-user/front_end
ssh -i "~/.ssh/keys/remotebuild.pem" -t ec2-user@$1 << EOF
  curl -o- https://raw.githubusercontent.com/nvm-sh
  /nvm/v0.37.2/install.sh | bash
  . ~/.nvm/nvm.sh
  nvm install --lts
  cd front_end
  npm install
EOF
```

在这里，我们移除了 node 模块，将代码复制到服务器上，安装 node，然后运行我们应用程序的 `install` 命令。现在，我们的构建管道完全工作后，我们可以继续将我们的软件包装在 Docker 中，以便我们可以将软件打包并部署。

# 使用 Docker 管理我们的软件

到目前为止，我们一直在使用 Docker 来管理我们的 PostgreSQL 和 Redis 数据库。当涉及到运行我们的前端和 Rust 服务器时，我们只是在我们的本地计算机上直接运行它们。然而，当涉及到在远程服务器上运行我们的应用程序时，它更简单、更容易分发。在我们开始在服务器上部署我们的 Docker 镜像之前，我们需要在本地构建和运行它们，这从编写我们的 Docker 镜像文件开始。

## 编写 Docker 镜像文件

在我们继续之前，必须注意的是，这里执行的方法是构建 Rust 服务器 Docker 镜像的最简单、最不优化的方式，因为我们正在处理许多新概念。我们在 *第十三章* 中介绍了构建 Rust 服务器 Docker 镜像的优化方法，*最佳实践：干净的 Web 应用程序仓库*。当涉及到构建 Docker 镜像时，我们需要一个 Dockerfile。这是我们定义构建我们的镜像所需的步骤的地方。在我们的 `web_app/Dockerfile` 文件中，我们基本上借用了一个基础镜像，然后在上面运行我们的命令，以便我们的应用程序能够工作。我们可以使用以下代码定义基础镜像和运行我们的数据库交互的要求：

```rs
FROM rust:1.61
RUN apt-get update -yqq && apt-get install -yqq cmake g++
RUN cargo install diesel_cli --no-default-features --features postgres
```

在我们的 Docker 构建中，我们以官方的 `rust` 镜像开始。然后我们更新 `apt` 以便我们可以下载所有可用的包。然后我们安装 `g++` 和 `diesel` 客户端，以便我们的数据库操作能够工作。然后我们将 Rust 应用程序的代码和配置文件复制过来，并使用以下代码定义我们的工作目录：

```rs
COPY . .
WORKDIR .
```

现在，我们有以下代码来构建我们的 Rust 应用程序：

```rs
RUN cargo clean
RUN cargo build --release
```

现在我们已经完成了构建，我们将 `target` 目录中的静态二进制文件移动到我们的主目录，删除过量的代码，如 `target` 和 `src` 目录，并使用以下代码允许静态二进制文件可执行：

```rs
RUN cp ./target/release/web_app ./web_app
RUN rm -rf ./target
RUN rm -rf ./src
RUN chmod +x ./web_app
```

现在一切都已经完成，我们可以暴露运行 Web 服务器的端口，以便通过容器暴露服务器并执行以下代码启动 Docker 镜像时运行的命令：

```rs
EXPOSE 8000
CMD ["./web_app", "config.yml"]
```

现在我们已经编写了 Docker 镜像文件，我们可以继续构建 Docker 镜像。

## 构建 Docker 镜像

在我们可以运行这个之前，我们需要删除我们的构建脚本。我们的 Docker 镜像构建只剩下一件事。当我们把代码复制到 Docker 镜像中时，我们知道 `target` 目录中有许多我们不需要在镜像中的代码和文件。我们可以通过在 `.dockerignore` 文件中添加以下代码来避免复制 `target` 目录：

```rs
target
```

如果我们尝试使用构建脚本编译我们的应用程序，Docker 将将自己陷入无限文件循环，然后超时。这意味着我们的 `main.rs` 文件中的 `ALLOWED_VERSION` 变量采用以下形式：

```rs
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    const ALLOWED_VERSION: &'static str = "v1";
    . . .
```

然后，我们必须在 `Cargo.toml` 文件中取消注释我们的 `build` 依赖项，并使用以下代码，并完全删除 `build.rs` 文件：

```rs
[package]
name = "web_app"
version = "0.1.0"
edition = "2021"
# build = "build.rs"
```

现在，我们已经准备好构建我们的镜像；我们导航到 Dockerfile 所在的位置，并运行以下命令：

```rs
docker build . -t rust_app
```

此命令执行当前目录中 Dockerfile 定义的构建。镜像被标记为 `rust_app`。我们可以使用以下命令列出我们的镜像：

```rs
docker image ls
```

这将给出以下输出：

```rs
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
rust_app     latest    c2e465545e30   2 hours ago    2.69GB
. . .
```

然后，我们可以测试我们的应用程序是否正确构建；我们只需运行以下命令：

```rs
docker run rust_app
```

这直接运行我们的 Rust 应用程序。我们的应用程序应该立即崩溃，并显示以下错误：

```rs
thread 'main' panicked at 'called `Result::unwrap()`
on an `Err` value: Connection refused (os error 111)',
src/counter.rs:24:47 note: run with `RUST_BACKTRACE=1`
environment variable to display a backtrace
```

我们可以看到，错误不是来自我们的构建，而是来自计数器文件与 Redis 的连接问题。这令人放心，并且当我们在两个数据库上运行我们的 Rust 应用程序时，它将工作。

在 Docker 中有一个多层级构建的方法。这就是我们从 `rust` 基础镜像开始，构建我们的应用程序，然后将构建移动到另一个没有依赖项的 Docker 镜像中。结果是我们的服务器镜像通常只有 100 MB，而不是多个 GB。然而，我们的应用程序有很多依赖项，这种多层级构建方法会导致多个驱动错误。我们在 *第十三章* 中探讨了构建小型镜像的最佳实践，*清洁 Web 应用程序仓库*。

## 使用 Terraform 构建 EC2 构建服务器

我们现在已经在本地构建了我们的 Rust Docker 镜像。我们可以在构建服务器上构建它。在我们这样做之前，我们必须增加构建服务器硬盘的大小；否则，镜像将因为空间不足而拒绝构建。这可以通过在我们的 `/build/main.tf` 文件中添加一个根块设备来完成，如下面的代码片段所示：

```rs
resource "aws_instance" "build_server" {
    ami = "ami-0fdbd8587b1cf431e"
    instance_type = "t2.medium"
    key_name = "remotebuild"
    user_data = file("server_build.sh")
    tags = {
      Name = "to-do build server"
    }
    # root disk
    root_block_device {
      volume_size           = "150"
      volume_type           = "gp2"
      delete_on_termination = true
    }
}
```

`gp2` 是 AWS 支持的 SSD 版本，我们正在使用。`150` 是我们连接到服务器的 GB 数。这将足够构建我们的 Docker 镜像，剩下的只是构建构建我们的 Docker 镜像的管道。

## 使用 Bash 进行构建编排

在这一点上，我们还将优化我们的 `/build/run_build.sh` 文件中的构建 Bash 脚本。首先，我们不删除 `target` 目录；相反，我们使用以下代码有选择性地上传到我们的服务器：

```rs
#!/usr/bin/env bash
SCRIPTPATH="$( cd "$(dirname "$0")" ; pwd -P )"
cd $SCRIPTPATH
ssh -i "~/.ssh/keys/remotebuild.pem" -t ec2-user@$1 "mkdir web_app"
scp -i "~/.ssh/keys/remotebuild.pem" -r
../web_app/src ec2-user@$1:/home/ec2-user/web_app/src
scp -i "~/.ssh/keys/remotebuild.pem"
../web_app/Cargo.toml ec2-user@$1:/home/ec2-user/web_app/Cargo.toml
scp -i "~/.ssh/keys/remotebuild.pem"
../web_app/config.yml ec2-user@$1:/home/ec2-user/web_app/config.yml
scp -i "~/.ssh/keys/remotebuild.pem"
../web_app/Dockerfile ec2-user@$1:/home/ec2-user/web_app/Dockerfile
```

在这里，我们可以看到我们创建了 `web_app` 目录，然后上传了构建 Rust Docker 镜像所需的文件和目录。然后我们需要使用以下代码连接到服务器来安装 Rust：

```rs
echo "installing Rust"
ssh -i "~/.ssh/keys/remotebuild.pem" -t ec2-user@$1 << EOF
  curl https://sh.rustup.rs -sSf | bash -s -- -y
  until [ -f ./output.txt ]
  do
      sleep 2
  done
  echo "File found"
EOF
echo "Rust has been installed"
```

在这里，我们可以看到我们在安装 Rust 之前阻止脚本直到服务器准备好所有安装。这意味着我们在构建服务器其余部分的同时运行 Rust 的安装，从而节省时间。然后我们退出与构建服务器的连接。最后，我们再次连接到服务器，并使用以下代码构建 Docker 镜像：

```rs
echo "building Rust Docker image"
ssh -i "~/.ssh/keys/remotebuild.pem" -t ec2-user@$1 << EOF
  cd web_app
  docker build . -t rust_app
EOF
echo "Docker image built"
```

我们已经在我们的构建服务器上构建了构建 Rust Docker 镜像的管道。这看起来像有很多步骤，你是对的。我们可以使用不同的目标操作系统和芯片架构在本地构建我们的镜像。在这里探索它将打断我们试图实现的目标的流程，但有关使用不同目标编译的更多信息将在 *进一步阅读* 部分提供。

## 为 React 前端编写 Docker 镜像文件

目前，我们将使用 Docker 打包我们的前端应用程序。在我们的 `front_end/Dockerfile` 文件中，我们继承 node 基础镜像，复制代码，并使用以下代码定义工作目录：

```rs
FROM node:17.0.0
WORKDIR .
COPY . ./
```

然后，我们安装一个 `serve` 包来提供 web 应用构建文件、构建应用程序所需的模块，并使用以下代码构建 React 应用程序：

```rs
RUN npm install -g serve
RUN npm install
RUN npm run react-build
```

然后，我们使用以下代码公开端口和服务器到我们的应用程序：

```rs
EXPOSE 4000
CMD ["serve", "-s", "build", "-l", "4000"]
```

然后，我们可以使用以下命令构建我们的 Docker 镜像：

```rs
docker build . -t front_end
```

然后，我们可以使用以下命令运行最近构建的镜像：

```rs
docker run -p 80:4000 front_end
```

这将容器的外部端口 (`80`) 路由到本地暴露的端口 `4000`。当我们的镜像运行时，我们得到以下输出：

```rs
INFO: Accepting connections at http://localhost:4000
```

这表明我们的镜像正在容器中运行。我们可以通过访问本地主机来访问我们的前端容器，本地主机端口为 `80`，如下截图所示：

![图 10.12 – 使用本地主机访问我们的前端容器](img/Figure_10.12_B18722.jpg)

图 10.12 – 使用本地主机访问我们的前端容器

然而，我们无法对其进行任何操作，因为我们的 Rust 服务器尚未运行。现在，我们可以将构建前端所执行的步骤提升到我们的 `/build/run_build.sh` 脚本中，以便我们的构建管道构建前端镜像。这是一个尝试自己添加步骤的好机会。我们将在构建服务器上安装 node，然后执行前端构建步骤。

如果你尝试在我们的管道中集成我们的 React 构建，它应该看起来像以下代码：

```rs
echo "copying React app"
rm -rf ../front_end/node_modules/
scp -i "~/.ssh/keys/remotebuild.pem" -r
../front_end ec2-user@$1:/home/ec2-user/front_end
echo "React app copied"
echo "installing node on build server"
ssh -i "~/.ssh/keys/remotebuild.pem" -t ec2-user@$1 << EOF
  curl -o- https://raw.githubusercontent.com/nvm-sh/nvm
  /v0.37.2/install.sh | bash
  . ~/.nvm/nvm.sh
  nvm install --lts
EOF
echo "node installed"
echo "building front-end on server"
ssh -i "~/.ssh/keys/remotebuild.pem" -t ec2-user@$1 << EOF
  cd front_end
  docker build . -t front_end
EOF
echo "front-end Docker image has been built"
```

我们可以在我们的实现中更加优化；然而，前面的代码是最简单的应用。首先，我们将需要的代码复制到构建服务器上。然后我们连接到构建服务器来安装 node。安装 node 后，我们再次连接到服务器，进入 React 应用程序目录并构建我们的 Docker 镜像。

我们的建设管道现在正在工作。想想我们在这里取得的成就——我们构建了一个管道，它构建了一个构建服务器；然后我们将我们的代码复制到构建服务器上，构建了 Docker 镜像，然后在构建完成后销毁了服务器。尽管这个流水线并不完美，但我们已经探索了一些强大的工具，这些工具将使您能够自动化任务，并减轻我们在本节中涵盖的其他 CI 流水线工具中的许多代码。然而，现在我们只是在构建 Docker 镜像，然后与服务器一起销毁它们。在下一节中，我们将部署我们的镜像到**Docker Hub**。

## 将镜像部署到 Docker Hub

在我们能够从 Docker Hub 拉取我们的镜像之前，我们必须先将我们的镜像推送到 Docker Hub。在我们能够将我们的镜像推送到 Docker Hub 之前，我们必须创建一个 Docker Hub 仓库。在 Docker Hub 上注册我们的镜像非常简单。登录后，我们点击右上角的**创建仓库**按钮，如图所示：

![图 10.13 – 在 Docker Hub 上创建新的仓库](img/Figure_10.13_B18722.jpg)

图 10.13 – 在 Docker Hub 上创建新的仓库

一旦我们点击了它，我们使用以下配置定义仓库：

![图 10.14 – 定义新的 Docker 仓库](img/Figure_10.14_B18722.jpg)

图 10.14 – 定义新的 Docker 仓库

我们可以看到，有一个选项可以通过点击前面截图中的*GitHub*按钮将我们的仓库与 GitHub 连接起来。前面截图中的**已连接**GitHub 状态简单意味着我的 GitHub 已经连接到我的 Docker Hub 账户。这意味着每次成功的拉取请求完成后，镜像都会用代码重新构建，然后发送到仓库。如果你正在构建一个完全自动化的流水线，这可能会很有帮助。然而，对于这本书，我们不会连接我们的 GitHub 仓库。我们将将其推送到我们的构建服务器。如果你正在部署 React 应用程序，你还需要为我们的前端创建一个 Docker Hub 仓库。

现在我们已经定义了我们的 Docker Hub 仓库，我们需要在我们的构建管道中添加一个 Docker 登录。这意味着我们必须将我们的 Docker Hub 密码和用户名传递到我们的 Python 构建脚本中。然后我们的 Python 脚本可以将 Docker Hub 凭证传递到构建 Bash 脚本中。这个构建 Bash 脚本将登录到构建服务器上的 Docker，以便我们可以将我们的镜像推送到我们的 Docker Hub。在我们的`/build/run_build.py`文件中，我们使用以下代码定义传递给 Python 脚本的参数：

```rs
. . .
import argparse
. . .
parser = argparse.ArgumentParser(
                    description='Run the build'
                  )
parser.add_argument('--u', action='store',
                    help='docker username',
                    type=str, required=True)
parser.add_argument('--p', action='store',
                    help='docker password',
                    type=str, required=True)
args = parser.parse_args()
```

我们可以看到，我们已经将`required`设置为`True`，这意味着除非提供这两个参数，否则 Python 脚本不会运行。如果我们向 Python 脚本提供一个`-h`参数，那么在前面代码中定义的参数将带有帮助信息打印出来。现在我们已经摄入了 Docker 凭证，我们可以在`/build/run_build.py`文件中将它们传递到我们的构建 Bash 脚本中，以下是对我们代码的以下修改：

```rs
. . .
print("attempting to enter server")
build_process = Popen(f"cd {DIRECTORY_PATH} && sh ./run_build.sh
    {server_ip} {args.u} {args.p}", shell=True)
build_process.wait()
destroy_process = Popen(f"cd {DIRECTORY_PATH} && terraform destroy",
    shell=True)
. . .
```

在这里，我们可以看到 Docker 用户名是通过`args.u`属性访问的，而 Docker 密码是通过`args.p`属性访问的。既然我们已经将 Docker 凭证传递到我们的构建 Bash 脚本中，我们需要使用这些凭证来推送我们的镜像。在我们的`/build/run_build.sh`文件中，我们应该在 Docker 安装在我们的构建服务器上后，使用以下代码进行登录：

```rs
. . .
echo "Rust has been installed"
echo "logging in to Docker"
ssh -i "~/.ssh/keys/remotebuild.pem" -t ec2-user@$1 << EOF
  echo $3 | docker login --username $2 --password-stdin
EOF
echo "logged in to Docker"
echo "building Rust Docker image"
. . .
```

在前面的代码中，我们使用`–password-stdin`将密码管道输入到 Docker 登录中。`stdin`确保密码不会存储在日志中，这使得它稍微安全一些。然后我们可以使用以下代码构建、标记并推送我们的 Rust 应用程序的更新：

```rs
echo "building Rust Docker image"
ssh -i "~/.ssh/keys/remotebuild.pem" -t ec2-user@$1 << EOF
  cd web_app
  docker build . -t rust_app
  docker tag rust_app:latest maxwellflitton/to_do_actix:latest
  docker push maxwellflitton/to_do_actix:latest
EOF
echo "Docker image built"
```

在这里，我们像往常一样构建 Rust 镜像。然后我们将 Rust 应用程序镜像标记为最新发布版本，并将其推送到 Docker 仓库。我们还必须将我们的前端应用程序推送到 Docker Hub。在这个时候，这是一个写代码推送前端镜像的好机会。如果你尝试推送前端镜像，它应该看起来像以下代码片段：

```rs
echo "building front-end on server"
ssh -i "~/.ssh/keys/remotebuild.pem" -t ec2-user@$1 << EOF
  cd front_end
  docker build . -t front_end
  docker tag front_end:latest maxwellflitton/to_do_react:latest
  docker push maxwellflitton/to_do_react:latest
EOF
echo "front-end Docker image has been built"
```

我们现在已经编写了所有必要的代码来构建两个镜像并将它们推送到我们的 Docker 仓库。我们可以使用以下命令运行我们的 Python 构建脚本：

```rs
python3 run_build.py --u some_username --p some_password
```

再次强调，打印输出很长，但我们可以检查我们的 Docker Hub 仓库以查看我们的镜像是否已推送。正如以下屏幕截图所示，Docker Hub 仓库会显示镜像何时被推送：

![图 10.15 – 推送 Docker 仓库的视图](img/Figure_10.15_B18722.jpg)

图 10.15 – 推送 Docker 仓库的视图

现在我们已经将镜像推送到我们的 Docker Hub 仓库！这意味着我们可以将它们拉到任何我们需要使用的计算机上，就像我们在`docker-compose`文件中拉取 PostgreSQL 镜像时做的那样。我们现在应该将我们的镜像拉到服务器上，以便其他人可以访问和使用我们的应用程序。在下一节中，我们将部署我们的应用程序以供外部使用。

# 在 AWS 上部署我们的应用程序

尽管我们已经将 Rust 应用程序打包到 Docker 中，但我们还没有在 Docker 容器中运行我们的 Rust 应用程序。在我们将 Rust 应用程序运行在 AWS 服务器上之前，我们应该在本地运行我们的 Rust 应用程序。这将帮助我们了解简单的部署是如何工作的，而无需构建服务器。

## 在本地运行我们的应用程序

当涉及到在本地运行我们的应用程序时，我们将使用以下布局的`docker-compose`：

![图 10.16 – 本地部署结构](img/Figure_10.16_B18722.jpg)

图 10.16 – 本地部署结构

在这里，我们可以看到 NGINX 容器从`docker-compose`网络外部接收流量，并将流量导向适当的容器。现在我们理解了我们的结构，我们可以定义我们的`docker-compose`文件。首先，我们需要在`build`、`front_end`和`web_app`目录旁边创建一个名为`deployment`的目录。在我们的`deployment`目录中，我们的`docker-compose.yml`文件的一般布局如下所示：

```rs
services:
  nginx:
    . . .
  postgres_production:
    . . .
  redis_production:
      . . .
  rust_app:
    . . .
  front_end:
    . . .
```

我们可以从 NGINX 服务开始。既然我们知道外部端口是`80`，那么我们的 NGINX 容器监听外部端口`80`是有意义的，以下代码实现了这一点：

```rs
  nginx:
    container_name: 'nginx-rust'
    image: "nginx:latest"
    ports:
      - "80:80"
    links:
      - rust_app
      - front_end
    volumes:
      - ./nginx_config.conf:/etc/nginx/nginx.conf
```

我们可以看到我们获取了最新的 NGINX 镜像。我们还链接到`front_end`和`rust_app`容器，因为我们将会将这些 HTTP 请求转发到这些容器。还必须注意的是，我们有一个卷。这是在容器外部与容器内部的目录共享卷的地方。所以，这个卷定义意味着我们的`deploy/nginx_config.conf`文件可以在 NGINX 容器的`etc/nginx/nginx.conf`目录中访问。有了这个卷，我们可以在`deploy/nginx_config.conf`文件中配置 NGINX 路由规则。

首先，我们将工作进程的数量设置为`auto`。如果需要，我们可以手动定义工作进程的数量。`auto`会检测可用的 CPU 核心数量，并将数量设置为该值，以下代码实现了这一点：

```rs
worker_processes  auto;
error_log  /var/log/nginx/error.log warn;
```

`error_log`指令定义了日志记录到特定的文件。我们不仅定义了文件，还声明了写入日志文件所需的最小级别，即`warning`。默认情况下，写入文件所需的日志级别是`error`。现在我们可以继续在`deploy/nginx_config.conf`文件中定义上下文。在`events`上下文中，我们定义了工作进程一次可以处理的连接的最大数量。这是通过以下代码实现的：

```rs
events {
    worker_connections  512;
}
```

我们定义的工作进程数量是 NGINX 设置的默认数量。现在这已经完成，我们可以继续到我们的`http`上下文。在这里，我们定义了`server`上下文。在这个上下文中，我们指示服务器监听端口`80`，这是监听外部流量的端口，以下代码实现了这一点：

```rs
http {
    server {
        listen 80;
        location /v1 {
            proxy_pass http://rust_app:8000/v1;
        }
        location / {
            proxy_pass http://front_end:4000/;
        }
    }
}
```

在这里，我们可以看到，如果端点的 URL 以`/v1/`开头，我们就将其通过 Rust 服务器转发。必须注意的是，我们将`v1`传递给 Rust 服务器。如果我们没有将`v1`传递给 Rust 服务器，当它到达 Rust 服务器时，URL 中会缺少`v1`。如果 URL 中不包含`v1`，那么我们就将其转发到我们的`front_end`容器。我们的 NGINX 容器现在已准备好管理我们的`docker-compose`系统中的流量。在我们继续到前端和后端服务之前，我们需要定义 Redis 和 PostgreSQL 数据库。这里没有什么新的，所以在这个时候，你可以尝试自己定义它们。如果你已经定义了，那么你的代码应该看起来像这样：

```rs
  postgres_production:
    container_name: 'to-do-postgres-production'
    image: 'postgres:latest'
    restart: always
    ports:
      - '5433:5432'
    environment:
      - 'POSTGRES_USER=username'
      - 'POSTGRES_DB=to_do'
      - 'POSTGRES_PASSWORD=password'
    expose:
      - 5433
  redis_production:
      container_name: 'to-do-redis'
      image: 'redis:5.0.5'
      ports:
        - '6379:6379'
      volumes:
    	  - ./data/redis:/tmp
```

之前在本地开发中使用的数据库定义与此相同。有了这些数据库，我们可以使用以下代码定义我们的 Rust 应用程序：

```rs
  rust_app:
    container_name: rust_app
    build: "../web_app"
    restart: always
    ports:
      - "8000:8000"
    links:
      - postgres_production
      - redis_production
    expose:
      - 8000
    volumes:
      - ./rust_config.yml:/config.yml
```

在这里，我们可以看到我们没有定义一个镜像。而不是声明镜像，我们指向一个构建。`build` 标签是我们指向 `deploy/rust_config.yml` 文件的地方，其形式如下：

```rs
DB_URL: postgres://username:password@postgres_production/to_do
SECRET_KEY: secret
EXPIRE_MINUTES: 120
REDIS_URL: redis://redis_production/
```

在这里，我们可以看到我们引用的是 `docker-compose` 系统中定义的服务名称，而不是 URL。我们还必须在 `web_app/src/main.rs` 文件中将我们的 Rust 应用程序的地址更改为零，如下所示：

```rs
})
.bind("0.0.0.0:8000")?
.run()
```

然后，我们必须在我们的 `web_app/Dockerfile` 文件中删除 Docker 构建中的配置文件，如下所示：

```rs
RUN rm config.yml
```

如果我们不这样做，那么我们的 Rust 应用程序将无法与 NGINX 容器连接。现在，我们的 Rust 服务器已经定义完毕，我们可以继续在 `docker-compose.yml` 文件中定义前端应用程序，如下所示：

```rs
  front_end:
    container_name: front_end
    image: "maxwellflitton/to_do_react:latest"
    restart: always
    ports:
      - "4000:4000"
    expose:
      - 4000
```

在这里，我们看到我们引用了 Docker Hub 中的镜像并公开了端口。现在，我们的本地系统已经定义，我们可以通过运行以下命令来运行我们的系统并与之交互：

```rs
docker-compose up
```

所有的构建和运行都将自动完成。在我们能够与我们的系统交互之前，我们需要在我们的 Rust 应用程序构建中运行以下命令的 `diesel` 迁移：

```rs
diesel migration run
```

然后，我们需要使用以下 `curl` 命令创建一个用户：

```rs
curl --location --request POST 'http://localhost/v1/user/create' \
--header 'Content-Type: application/json' \
--data-raw '{
    "name": "maxwell",
    "email": "maxwellflitton@gmail.com",
    "password": "test"
}'
```

我们现在已经准备好与我们的应用程序进行交互。我们可以看到，没有端口引用的 localhost 在以下屏幕截图中的工作情况：

![图 10.17 – 通过浏览器访问我们的 docker-compose 系统](img/Figure_10.17_B18722.jpg)

图 10.17 – 通过浏览器访问我们的 docker-compose 系统

如果你的 NGINX 正在运行，你应该能够登录并像以前一样与待办事项应用程序进行交互。我们现在能够将我们的系统部署到 AWS，以便在下一节中其他人可以访问和使用我们的待办事项应用程序。

## 在 AWS 上运行我们的应用程序

我们可以通过执行以下步骤在 AWS 上部署我们的应用程序：

1.  构建服务器

1.  在那个服务器上运行我们的 `docker-compose` 系统

1.  在那个服务器上运行数据库迁移

1.  创建用户

一旦我们完成了这些步骤，我们就能访问远程服务器上的应用程序。然而，在我们这样做之前，我们必须修改 React 应用程序。目前，我们的 React 应用程序通过 `127.0.0.1` 向 localhost 发起 API 调用。当我们使用远程服务器时，这将不起作用，因为我们必须调用我们部署应用程序的服务器。为此，我们可以在 React 应用程序中提取 API 调用的位置，并使用以下代码更新 API 调用的 URL 根：

```rs
axios.get(window.location.href + "/v1/item/get",
```

这里发生的事情是 `window.location.href` 返回当前位置，这将是我们应用程序部署在的服务器 IP 地址，或者如果我们是在本地计算机上开发，则是 localhost。以下文件有需要更新的 API 调用：

+   `src/components/LoginForm.js`

+   `src/components/CreateToDoitem.js`

+   `src/components/ToDoitem.js`

+   `src/App.js`

更新这些文件后，我们可以在 `build` 目录下通过运行以下命令来运行另一个构建：

```rs
python3 run_build.py --u some_username --p some_password
```

一旦我们的构建完成，我们的两个镜像都将更新。现在我们可以移动到我们的 `deployment` 目录，并使用以下文件来完善它：

+   `main.tf`：这应该与 `build` 目录中的 `main.tf` 文件相同，除了服务器有一个不同的标签

+   `run_build.py`：这应该与 `build` 目录中的 `run_build.py` 文件相同，除了 `run_build.py` 脚本末尾的 *销毁服务器* 过程

+   `server_build.sh`：这应该与 `build` 目录中的 `server_build.sh` 脚本相同，因为我们希望我们的服务器拥有与构建镜像时相同的环境

+   `deployment-compose.yml`：这应该与 `deployment` 目录中的 `docker-compose.yml` 文件相同，除了 `rust_app` 服务有一个镜像标签而不是构建标签，并且镜像标签应该是 `maxwellflitton/to_do_actix:latest`

+   `.env`：这应该与 `web_app` 目录中的 `.env` 文件相同，我们需要它来执行数据库迁移

现在，我们已经准备好编写 `run_build.sh` 文件，这将使我们能够部署我们的应用程序，运行迁移，并创建一个用户。首先，我们从一些标准的样板代码开始，以确保我们处于正确的目录，如下所示：

```rs
#!/usr/bin/env bash
SCRIPTPATH="$( cd "$(dirname "$0")" ; pwd -P )"
cd $SCRIPTPATH
```

然后，我们复制启动我们的 `docker-compose` 系统所需的文件，并使用以下代码执行数据库迁移：

```rs
scp -i "~/.ssh/keys/remotebuild.pem"
./deployment-compose.yml ec2-user@$1:/home/ec2-user/docker-compose.yml
scp -i "~/.ssh/keys/remotebuild.pem"
./rust_config.yml ec2-user@$1:/home/ec2-user/rust_config.yml
scp -i "~/.ssh/keys/remotebuild.pem"
./.env ec2-user@$1:/home/ec2-user/.env
scp -i "~/.ssh/keys/remotebuild.pem"
./nginx_config.conf ec2-user@$1:/home/ec2-user/nginx_config.conf
scp -i "~/.ssh/keys/remotebuild.pem" -r
../web_app/migrations ec2-user@$1:/home/ec2-user/migrations
```

这些都不应该令人惊讶，因为我们需要所有前面的文件来运行我们的 `docker-compose` 系统。然后我们安装 Rust 并等待服务器构建完成，使用以下代码：

```rs
echo "installing Rust"
ssh -i "~/.ssh/keys/remotebuild.pem" -t ec2-user@$1 << EOF
  curl https://sh.rustup.rs -sSf | bash -s -- -y
  until [ -f ./output.txt ]
  do
      sleep 2
  done
  echo "File found"
EOF
echo "Rust has been installed"
```

再次，这并不是我们没有见过的新的东西。然后我们使用以下代码安装 `diesel` 客户端：

```rs
echo "installing diesel"
ssh -i "~/.ssh/keys/remotebuild.pem" -t ec2-user@$1 << EOF
  cargo install diesel_cli --no-default-features --features postgres
EOF
echo "diesel has been installed"
```

然后，我们登录到 Docker，启动我们的 `docker-compose` 系统，运行我们的迁移，然后使用以下代码创建一个用户：

```rs
echo "building system"
ssh -i "~/.ssh/keys/remotebuild.pem" -t ec2-user@$1 << EOF
  echo $3 | docker login --username $2 --password-stdin
  docker-compose up -d
  sleep 2
  diesel migration run
  curl --location --request POST 'http://localhost/v1/user/create' \
  --header 'Content-Type: application/json' \
  --data-raw '{
      "name": "maxwell",
      "email": "maxwellflitton@gmail.com",
      "password": "test"
  }'
EOF
echo "system has been built"
```

这样，我们的系统就部署完成了！我们可以通过将 `output.json` 文件中的服务器 IP 地址放入浏览器来访问我们的应用程序。我们可以登录并使用我们的待办事项应用程序，就像我们在本地计算机上运行系统时一样，如下面的截图所示：

![图 10.18 – 我们在部署服务器上的应用程序](img/Figure_10.18_B18722.jpg)

图 10.18 – 我们在部署服务器上的应用程序

如我们所见，连接不安全，我们的浏览器正在给我们发出警告，因为我们没有实现 HTTPS 协议。这是因为我们的连接未加密。我们将在下一章中介绍如何加密我们的连接。

## 编写我们的应用程序构建脚本

目前，我们的应用程序在 EC2 实例上本地运行数据库。这有几个问题。首先，这意味着 EC2 是状态化的。如果我们拆毁实例，我们将丢失所有数据。

其次，如果我们擦除实例上的容器，我们可能会丢失所有数据。数据脆弱性并不是这里唯一的问题。假设我们的流量急剧增加，我们需要更多的计算实例来管理它。这可以通过使用 NGINX 作为两个实例之间的负载均衡器来实现，如下面的图所示：

![图 10.19 – 为我们的系统加倍 EC2 实例](img/Figure_10.19_B18722.jpg)

图 10.19 – 为我们的系统加倍 EC2 实例

如您所见，这里的问题是访问随机数据。如果用户一创建了一个项目，并且这个请求击中了左侧的实例，它将存储在左侧的数据库中。然而，用户一可以随后发起一个`GET`请求，这个请求击中了右侧的实例。第二个请求将无法访问第一个请求中创建的项目。用户将根据请求击中的实例访问随机状态。

这可以通过从我们的`docker-compose`文件中删除数据库并在其外部创建数据库来解决，如下面的图所示：

![图 10.20 – 我们的新改进系统](img/Figure_10.20_B18722.jpg)

图 10.20 – 我们的新改进系统

现在，我们有一个单一的数据真相，我们的 EC2 实例是无状态的，这意味着我们有自由根据需要创建和删除实例。

当涉及到将 AWS 数据库添加到我们的部署中时，我们必须在 Terraform 中构建我们的数据库，然后将有关构建的数据库的信息传递到我们的`deployment/.env`文件以进行数据库迁移，以及传递到我们的`deployment/rust_config.yml`文件以供我们的 Rust 服务器访问。首先，我们必须将数据库定义添加到我们的`deployment/main.tf`文件中，如下面的代码所示：

```rs
resource "aws_db_instance" "main_db" {
  instance_class         = "db.t3.micro"
  allocated_storage      = 5
  engine                 = "postgres"
  username               = var.db_username
  password               = var.db_password
  db_name                = "to_do"
  publicly_accessible    = true
  skip_final_snapshot    = true
  tags = {
      Name = "to-do production database"
    }
}
```

在前面的代码中定义的字段很简单，除了`allocated_storage`字段，这是分配给数据库的 GB 数。我们还可以看到我们使用了带有`var`变量的变量。这意味着我们必须在运行 Terraform 构建时传递密码和用户名。我们需要在`deployment/variables.tf`文件中定义我们的输入变量，如下面的代码所示：

```rs
variable "db_password" {
    description = "The password for the database"
    default = "password"
}
variable "db_username" {
    description = "The username for the database"
    default = "username"
}
```

这些变量有默认值，因此我们不需要传递变量。但是，如果我们想传递变量，我们可以使用以下布局：

```rs
-var"db_password=some_password" -var"db_username=some_username"
```

现在，我们必须将这些参数传递到所需的文件和 Terraform 构建中。这正是 Python 开始发光的地方。我们将读取和写入 YAML 文件，因此我们必须使用以下命令安装 YAML Python 包：

```rs
pip install pyyaml
```

然后，我们在脚本的顶部导入这个包到我们的`deployment/run_build.py`文件中，使用以下代码：

```rs
import yaml
```

然后，我们从名为`database.json`的 JSON 文件中加载数据库参数，并使用以下代码创建我们的`vars`命令字符串：

```rs
with open("./database.json") as json_file:
    db_data = json.load(json_file)
params = f' -var="db_password={db_data["password"]}"
            -var="db_username={db_data["user"]}"'
```

您可以为您的`deployment/database.json`文件制定任何所需的参数；我最近一直在尝试 GitHub Copilot，这是一个自动填充代码的 AI 配对程序员，这给了我以下参数：

```rs
{
    "user": "Santiago",
    "password": "1234567890",
    "host": "localhost",
    "port": "5432",
    "database": "test"
}
```

我不知道谁是`Santiago`，但 Copilot AI 显然认为`Santiago`是正确的用户，所以我将使用它。回到我们的`deployment/run_build.py`文件，我们必须通过更新以下代码将我们的参数传递给 Terraform 的`apply`命令：

```rs
apply_process = Popen(f"cd {DIRECTORY_PATH} && terraform apply" + params, shell=True)
```

在运行 Terraform 构建的进程完成后，我们将该构建的输出存储在一个 JSON 文件中。然后我们创建自己的数据库 URL，并使用以下代码将此 URL 写入一个文本文件：

```rs
. . .
produce_output.wait()
with open(f"{DIRECTORY_PATH}/output.json", "r") as file:
    data = json.loads(file.read())
database_url = f"postgresql://{db_data['user']}:{db_data['password']}
    @{data['db_endpoint']['value'][0]}/to_do"
with open("./database.txt", "w") as text_file:
    text_file.write("DATABASE_URL=" + database_url)
```

我们现在唯一需要做的事情是使用以下代码更新我们的 Rust 应用程序配置数据：

```rs
with open("./rust_config.yml") as yaml_file:
    config = yaml.load(yaml_file, Loader=yaml.FullLoader)
config["DB_URL"] = database_url
with open("./rust_config.yml", "w") as yaml_file:
    yaml.dump(config, yaml_file, default_flow_style=False)
```

在我们的管道中，只剩下一个更改，这是在我们的`deployment/run_build.sh`文件中。我们不是将我们的本地`.env`文件复制到部署服务器，而是使用以下代码复制我们的`deployment/database.txt`文件：

```rs
scp -i "~/.ssh/keys/remotebuild.pem"
./database.txt ec2-user@$1:/home/ec2-user/.env
```

再次运行我们的部署将部署我们的服务器并将其连接到我们创建的 AWS 数据库。同样，这些构建脚本可能很脆弱。有时，在将其中一个文件复制到部署服务器时可能会被拒绝连接，这可能导致整个管道中断。因为我们自己编写了所有步骤并理解每个步骤，如果发生中断，手动解决问题或再次尝试运行 Python 构建脚本不会花费我们太多时间。

# 摘要

我们终于到达了旅程的终点。我们已经创建了自己的 Docker 镜像，打包了我们的 Rust 应用程序。然后我们在本地计算机上运行了这个镜像，并受到 NGINX 容器的保护。然后我们将其部署到 Docker Hub 账户，使我们能够将其部署到我们设置的 AWS 服务器上。

必须注意，我们已经完成了配置容器和通过 SSH 访问服务器的漫长步骤。这使得我们能够将此过程应用于其他平台，因为我们的通用方法并非仅针对 AWS。我们只是使用 AWS 来设置服务器。然而，如果我们使用其他提供商设置服务器，我们仍然能够在服务器上安装 Docker，将我们的镜像部署到上面，并使用 NGINX 和数据库连接来运行它。

然而，作为开发者，我们的工作永远不会结束，我们还能做更多的事情。不过，我们已经涵盖了构建 Rust 网络应用程序的核心基础知识，并实现了从零开始构建和自动化部署。

考虑到这一点，几乎没有阻碍开发者使用 Rust 构建 Web 应用程序。可以添加前端框架来提高前端功能，还可以添加额外的模块来增加我们的应用程序的功能和 API 端点。我们现在有一个坚实的基础来构建各种应用程序，并进一步阅读相关主题，以使我们能够开发 Rust Web 开发的技能和知识。

我们正处于 Rust 和 Web 开发的一个激动人心的时期，希望到达这个阶段后，你会有信心推动 Rust 在 Web 开发领域的进步。在下一章中，我们将使用 HTTPS 加密我们的 Web 流量到应用程序。

# 进一步阅读

+   Rust 编译到不同目标文档：[`doc.rust-lang.org/rustc/targets/index.html`](https://doc.rust-lang.org/rustc/targets/index.html)

+   GitHub Actions 文档：[`github.com/features/actions`](https://github.com/features/actions)

+   Travis CI 文档：[`docs.travis-ci.com/user/for-beginners/`](https://docs.travis-ci.com/user/for-beginners/)

+   CircleCI 文档：[`circleci.com/docs/`](https://circleci.com/docs/)

+   Jenkins 文档：[`www.jenkins.io/doc/`](https://www.jenkins.io/doc/)

+   Terraform 输出文档：[`developer.hashicorp.com/terraform/language/values/outputs`](https://developer.hashicorp.com/terraform/language/values/outputs)

+   *AWS 认证开发者 - 副总裁指南*，第二版，*V. Tankariya* 和 *B. Parmar* (2019)，*Packt Publishing*，*第五章*，*开始使用弹性计算云 (EC2)*，第 165 页

+   *AWS 认证开发者 - 副总裁指南*，第二版，*V. Tankariya* 和 *B. Parmar* (2019)，*Packt Publishing*，*第十章*，*AWS 关系型数据库服务 (RDS)*，第 333 页

+   *AWS 认证开发者 - 副总裁指南*，第二版，*V. Tankariya* 和 *B. Parmar* (2019)，*Packt Publishing*，*第二十一章*，*开始使用 AWS CodeDeploy*，第 657 页

+   *精通 Kubernetes*，*G. Sayfan* (2020)，*Packt Publishing*

+   *开始使用 Terraform*，*K. Shirinkin* (2017)，*Packt Publishing*

+   *Nginx HTTP 服务器*，第四版，*M. Fjordvald* 和 *C. Nedelcu* (2018)，*Packt Publishing*
