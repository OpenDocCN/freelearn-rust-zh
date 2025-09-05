# 第一章：*第一章*：介绍 Rust 语言

几乎每个程序员都听说过 **Rust** 编程语言，甚至尝试或使用过它。每次都说“Rust 编程语言”有点繁琐，所以从现在起我们就叫它 Rust，或者 Rust 语言。

在本章中，我们将简要介绍 Rust，以帮助你是 Rust 语言的新手，或者如果你已经尝试过，作为复习。本章也可能对经验丰富的 Rust 语言程序员有所帮助。在章节的后面部分，我们将学习如何安装 Rust 工具链并创建一个简单的程序来介绍 Rust 语言的特性。然后，我们将使用第三方库来增强我们的一个程序，最后，我们将了解如何为 Rust 语言及其库获取帮助。

在本章中，我们将涵盖以下主要主题：

+   Rust 语言的概述

+   安装 Rust 编译器和工具链

+   编写 Hello World

+   探索 Rust 包和 Cargo

+   探索其他工具和获取帮助的地方

# 技术要求

要跟随本书的内容，你需要一台运行类似 Unix 的操作系统（如 Linux、macOS 或安装了 Windows Subsystem for Linux (WSLv1 或 WSLv2) 的 Windows）的计算机。不用担心 Rust 编译器和工具链；如果尚未安装，我们将在本章中安装它。

本章的代码可以在 [`github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter01`](https://github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter01) 找到。

# Rust 语言的概述

要使用 **Rocket** 框架构建网络应用程序，我们首先需要了解一点 Rust 语言，因为 Rocket 是用那种语言构建的。根据 [`www.rust-lang.org`](https://www.rust-lang.org)，Rust 语言是 "*一种赋予每个人构建可靠和高效软件能力的语言*。" 它始于 2006 年左右程序员 Graydon Hoare 的一个个人项目，他是 Mozilla 的员工。Mozilla 基金会看到了这种语言对他们的产品的潜力；他们在 2009 年开始赞助这个项目，并在 2010 年向公众宣布。

自从诞生以来，Rust 的重点始终是性能和安全。构建一个网络浏览器并不是一件容易的事情；一种不安全的语言可以拥有非常快的性能，但如果没有适当的安全措施，使用系统语言的程序员可能会犯很多错误，例如丢失指针引用。Rust 被设计成一种系统语言，并从较老的语言中吸取了许多教训。在较老的语言中，你可以轻易地因为空指针而“自伤”，而且语言中没有任何东西可以阻止你编译这样的错误。相比之下，在 Rust 语言中，你不能编写出会导致空指针的代码，因为这将会在编译时被检测到，你必须修复实现以使其能够编译。

Rust 语言的设计在很大程度上借鉴了函数式编程范式以及面向对象编程范式。例如，它具有函数式语言的一些元素，如闭包和迭代器。你可以轻松地创建一个纯函数，并将该函数作为另一个函数的参数使用；有语法可以轻松创建闭包和数据类型，如`Option`或`Result`。

另一方面，没有类定义，但你可以轻松地定义一个数据类型，例如，一个**结构体**。在定义了这种数据类型之后，你可以创建一个块来实现其方法。

尽管没有继承，但你可以通过使用`MakeSound`特质轻松地将对象分组。然后，你可以通过编写方法签名来确定该特质中应该有哪些方法。如果你定义一个数据类型，例如，一个名为`Cow`的结构体，你可以告诉编译器它实现了`MakeSound`特质。因为你说`Cow`结构体实现了`MakeSound`特质，所以你必须为`Cow`结构体实现特质中定义的方法。听起来像面向对象语言，对吧？

Rust 语言在发布稳定版本（Rust 1.0）之前经历了多次迭代（2015 年 5 月 15 日）。在发布稳定版本之前，一些早期的语言设计被废弃了。在某个时刻，Rust 有一个类特性，但在稳定发布之前被废弃了，因为 Rust 的设计被改为数据和行为分离。你写数据（例如，以`struct`或`enum`类型的形式），然后你分别写行为（例如，`impl`）。为了将这些`impl`归类到同一组中，我们可以创建一个**特质**。因此，你可以从面向对象语言中获得所有想要的功能。此外，Rust 曾经有过垃圾回收，但后来因为使用了另一种设计模式而被废弃。当对象超出作用域时，例如退出函数，它们会自动释放。这种类型的自动内存管理使得垃圾回收变得不必要。

在第一个稳定版本发布后，人们添加了更多功能，使 Rust 更加易用和可用。其中最大的变化是**async/await**，它在 1.39 版本中发布。这个特性对于开发处理 I/O 的应用程序非常有用，而 Web 应用程序编程处理大量的 I/O。Web 应用程序必须处理数据库和网络连接、从文件中读取等。人们普遍认为，async/await 是使语言适合 Web 编程的最需要的特性之一，因为在 async/await 中，程序不需要创建新的线程，但它也不像传统函数那样阻塞。

另一个重要特性是`const fn`，这是一个将在编译时而不是运行时评估的函数。

近年来，许多大型公司开始建立 Rust 开发者的人才库，这突出了它在商业中的重要性。

## 为什么使用 Rust 语言？

那么，为什么我们应该使用 Rust 语言进行 Web 应用程序开发？现有的成熟语言不是足够好吗？以下是一些人们想要使用 Rust 语言创建 Web 应用程序的原因：

+   安全性

+   无垃圾回收

+   速度

+   多线程和异步编程

+   静态类型

### 安全性

虽然使用系统编程语言编写应用程序有其优势，因为它们功能强大（程序员可以访问程序的基本构建块，例如分配计算机内存以存储重要数据，然后在不使用时立即释放该内存），但很容易出错。

在传统的系统语言中，没有任何东西可以阻止程序在内存中存储数据，创建指向该数据的指针，释放内存中存储的数据，然后尝试通过该指针再次访问数据。数据已经消失，但指针仍然指向内存的该部分。

经验丰富的程序员很容易在简单的程序中找到这样的错误。一些公司强迫他们的程序员使用静态分析工具来检查代码中的此类错误。但是，随着编程技术的日益复杂，应用程序的复杂性也在增长，这些类型的错误仍然可以在许多应用程序中找到。近年来发现的一些高调的漏洞和黑客攻击，如 *Heartbleed*，如果我们使用内存安全语言，就可以防止。

Rust 是一种内存安全语言，因为它对程序员如何编写代码有一系列规则。例如，当代码编译时，它会检查变量的生命周期，如果另一个变量仍然尝试访问已经超出作用域的数据，编译器将显示错误。2020 年，博士后研究员 Ralf Jung 已经进行了第一次正式验证，证明 Rust 语言确实是一种安全语言。内置的数据类型，如 `Option` 或 `Result`，以安全的方式处理类似 null 的行为。

### 无垃圾回收

许多程序员由于安全问题，会创建和使用不同的内存管理技术。其中一种技术是垃圾回收。其理念很简单：内存管理在运行时自动完成，这样程序员就不必考虑内存管理。程序员只需创建一个变量，当变量不再使用时，运行时系统会自动将其从内存中移除。

垃圾回收是计算中的一个有趣且重要的部分。有许多技术，如引用计数和追踪。例如，Java 除了官方的垃圾回收器外，还有几个第三方垃圾回收器。

这种语言设计选择的问题在于垃圾回收通常需要大量的计算资源。例如，一部分内存因为垃圾收集器还没有回收这部分内存而暂时不可用。或者，更糟糕的是，垃圾收集器无法从堆中移除已使用的内存，因此它会积累，大多数计算机内存将变得不可用，或者我们通常所说的**内存泄漏**。在**停止世界**的垃圾回收机制中，整个程序执行被暂停，以便垃圾收集器回收内存，之后程序执行继续。因此，有些人发现很难用这种语言开发实时应用程序。

Rust 采取了一种不同的方法，称为**资源获取即初始化**（**RAII**），这意味着一个对象一旦超出作用域就会自动释放。例如，如果你编写一个函数，在该函数中创建的对象将在函数退出时被释放。但显然，这使得 Rust 与手动释放内存的编程语言或具有垃圾回收的编程语言非常不同。

### 速度

如果你习惯于使用解释型语言或具有垃圾回收的语言进行 Web 开发，你可能会说我们不需要担心计算性能，因为 Web 开发是 I/O 密集型的；换句话说，瓶颈在于应用程序访问数据库、磁盘或另一个网络时，因为它们的速度比 CPU 或内存慢。

这个谚语可能主要是正确的，但一切都取决于应用程序的使用。如果你的应用程序处理大量的 JSON，处理是 CPU 密集型的，这意味着它受限于 CPU 的速度，而不是磁盘访问速度或网络连接速度。如果你关心应用程序的安全性，你可能需要处理散列和加密，这些都是 CPU 密集型的。如果你正在为在线流媒体服务编写后端应用程序，你希望应用程序尽可能优化。如果你正在编写服务于数百万用户的应用程序，你希望应用程序非常优化，并且尽可能快地返回响应。

Rust 语言是一种编译型语言，因此编译器会将程序转换为机器代码，计算机处理器可以执行。编译型语言通常比解释型语言运行得更快，因为在解释型语言中，当运行时二进制将程序解释为本地机器代码时，会有额外的开销。在现代解释器中，通过使用现代技术，如**即时编译器**（**JIT**）来加速程序执行，速度差距已经缩小，但在像 Ruby 这样的动态语言中，它仍然比使用编译型语言慢。

### 多线程和异步编程

在传统编程中，同步编程意味着应用程序必须等待 CPU 处理完一个任务。在 Web 应用程序中，服务器必须等待 HTTP 请求被处理并响应；只有在此之后，它才会继续处理另一个 HTTP 请求。如果应用程序只是直接创建响应，如简单的文本，这并不是问题。但当 Web 应用程序需要花费一些时间来处理时，它必须等待数据库服务器响应，必须等待文件在服务器上完全写入，以及必须等待对第三方 API 服务的 API 调用成功完成，这时就会变成问题。

克服等待问题的方法之一是多线程。单个进程可以创建多个线程，这些线程共享一些资源。Rust 语言被设计成易于创建安全的多线程应用程序。它通过多个容器，如 `Arc`，来简化线程间数据传递。

多线程的问题在于，创建一个线程意味着分配大量的 CPU、内存和操作系统资源，或者如俗话所说，成本高昂。解决方案是使用一种称为**异步编程**的不同技术，其中单个线程被不同的任务重用，而不必等待第一个任务完成。人们可以很容易地在 Rust 中编写异步程序，因为自 2019 年 11 月 7 日起，它已被纳入语言中。

### 静态类型

在编程语言中，动态类型语言是指在运行时检查变量类型的语言，而静态类型语言则在编译时检查数据类型。

动态类型意味着编写代码更容易，但也更容易出错。通常，程序员需要在动态类型语言中编写更多的单元测试来补偿编译时没有检查类型。动态类型语言也被认为成本更高，因为每次函数被调用时，程序都需要检查传递的参数。因此，优化动态类型语言变得困难。

相反，Rust 是静态类型的，因此很难犯将字符串作为数字传递的错误。编译器可以在应用程序发布之前优化生成的机器代码，并显著减少编程错误。

现在我们已经概述了 Rust 语言及其与其他语言的比较优势，让我们学习如何安装 Rust 编译器工具链，该工具链将用于编译 Rust 程序。我们将在这本书中使用这个工具链。

# 安装 Rust 编译器工具链

让我们从安装 Rust 编译器工具链开始。Rust 有三个官方渠道：*稳定版*、*测试版*和*夜间版*。Rust 语言使用 Git 作为其版本控制系统。人们将新功能和错误修复添加到 master 分支。每晚，master 分支的源代码都会被编译并发布到夜间版渠道。六周后，代码将从 beta 分支分叉出来，编译并发布到 beta 渠道。然后，人们将在 beta 发布版中运行各种测试，通常是在他们的 CI（持续集成）安装中。如果发现错误，修复将被提交到 master 分支，然后回滚到 beta 分支。第一次 beta 分支六周后，稳定版将从 beta 分支创建。

我们将在整本书中使用稳定渠道的编译器，但如果您想尝试冒险，您也可以使用其他渠道。不过，如果您使用其他渠道，我们无法保证我们将要创建的程序会编译，因为人们会添加新功能，并且新版本中可能引入了回归。

在您的系统中安装 Rust 工具链有几种方法，例如从头开始引导和编译，或者使用您的操作系统包管理器。但是，在您的系统中安装 Rust 工具链的推荐方法是使用`rustup`。

在其网站上的定义（[`rustup.rs`](https://rustup.rs)）非常简单："*rustup 是系统编程语言 Rust 的安装程序*。"现在，让我们尝试按照这些说明来安装`rustup`。

## 在 Linux OS 或 macOS 上安装 rustup

如果您使用的是 Debian 10 Linux 发行版，则这些说明适用，但如果您已经使用其他 Linux 发行版，我们将假设您已经熟悉 Linux 操作系统，并且可以适应适合您 Linux 发行版的这些说明：

1.  打开您选择的终端。

1.  通过输入以下命令确保您已安装 cURL：

    ```rs
    curl
    ```

1.  如果 cURL 尚未安装，让我们来安装它：

    ```rs
    apt install curl
    ```

如果您使用的是 macOS，您可能已经安装了 cURL。

1.  之后，按照[`rustup.rs`](https://rustup.rs)上的说明进行操作：

    ```rs
    curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
    ```

1.  然后，它会显示问候和信息，您可以自定义；目前，我们只是使用默认设置：

    ```rs
    ...
    1) Proceed with installation (default)
    2) Customize installation
    3) Cancel installation
    >
    ```

1.  输入`1`以使用默认安装。

1.  之后，重新加载您的终端或在此当前终端中输入以下内容：

    ```rs
    source $HOME/.cargo/env
    ```

1.  您可以通过在终端中输入`rustup`来确认安装是否成功，您应该会看到 rustup 的使用说明。

1.  现在，让我们安装稳定的 Rust 工具链。在终端中输入以下内容：

    ```rs
    rustup toolchain install stable
    ```

1.  在工具链安装到您的操作系统后，让我们确认我们是否可以运行 Rust 编译器。在终端中输入`rustc`，您应该会看到如何使用它的说明。

## 安装不同的工具链和组件

目前，我们已经安装了稳定工具链，但还有两个其他默认通道可以安装：*nightly*和*beta*。

有时，你可能出于各种原因想要使用不同的工具链。也许你想尝试一个新功能，或者也许你想测试你的应用程序对即将到来的 Rust 版本的回归。你可以简单地使用`rustup`来安装它：

```rs
rustup toolchain install nightly
```

每个工具链都有组件，其中一些是工具链必需的，例如`rustc`，这是 Rust 编译器。其他组件默认不安装，例如`clippy`，它提供了`rustc`编译器未提供的更多检查，并给出代码风格建议。安装它也非常简单；你可以使用`rustup component add <component>`，如本例所示：

```rs
rustup default stable
rustup component add clippy
```

## 更新工具链、rustup 和组件

Rust 工具链有一个大约每三个月（六周加六周）的定期发布计划，但有时会有针对重大错误修复或安全问题的紧急发布。因此，有时你需要更新你的工具链。更新非常简单。此命令还将更新工具链中安装的组件：

```rs
rustup update
```

除了工具链之外，`rustup`本身也可能需要更新。你可以通过输入以下内容来更新它：

```rs
rustup self update
```

现在我们已经将 Rust 编译器工具链安装到我们的系统中，让我们编写我们的第一个 Rust 程序！

# 编写 Hello World！

在本节中，我们将编写一个非常基础的程序，*Hello World！*。在我们成功编译之后，我们将编写一个更复杂的程序，以查看 Rust 语言的基本功能。让我们按照以下指示进行操作：

1.  让我们创建一个新的文件夹，例如，`01HelloWorld`。

1.  在文件夹内创建一个新的文件，并将其命名为`main.rs`。

1.  让我们在 Rust 中编写我们的第一个代码：

    ```rs
    fn main() { 
        println!("Hello World!");
    }
    ```

1.  之后，保存你的文件，在同一文件夹中，打开你的终端，并使用`rustc`命令编译代码：

    ```rs
    rustc main.rs
    ```

1.  你可以看到文件夹内有一个名为`main`的文件；从你的终端运行该文件：

    ```rs
    ./main
    ```

1.  恭喜！你刚刚用 Rust 语言编写了你的第一个`Hello World`程序。

接下来，我们将提高我们的 Rust 语言水平；我们将展示带有控制流、模块和其他功能的 Rust 基本应用程序。

## 编写更复杂的程序

当然，在制作了`Hello World`程序之后，我们应该尝试编写一个更复杂的程序，看看我们能用这个语言做什么。我们想要编写一个程序，它可以捕获用户输入的内容，使用选定的算法对其进行加密，并将输出返回到终端：

1.  让我们创建一个新的文件夹，例如，`02ComplexProgram`。之后，再次创建`main.rs`文件并再次添加`main`函数：

    ```rs
    fn main() {}
    ```

1.  然后，使用`std::io`模块编写程序的这部分，提示用户输入他们想要加密的字符串：

    ```rs
    use std::io;
    fn main() {
        println!("Input the string you want to encrypt:");
        let mut user_input = String::new();
        io::stdin()
            .read_line(&mut user_input)
            .expect("Cannot read input");
        println!("Your encrypted string: {}", user_input);
    }
    ```

让我们逐行探索我们所写的：

1.  第一行，`use std::io;`，是在告诉我们的程序，我们将在程序中使用 `std::io` 模块。除非我们明确表示不使用它，否则 `std` 应该默认包含在程序中。

1.  `let...` 行是一个变量声明。当我们定义一个变量在 Rust 中时，该变量默认是不可变的，因此我们必须添加 `mut` 关键字来使其可变。`user_input` 是变量名，而此语句的右侧是初始化一个新的空 `String` 实例。注意我们是如何直接初始化变量的。Rust 允许分离声明和初始化，但那种形式不是惯用的，因为程序员可能会尝试使用未初始化的变量，而 Rust 禁止使用未初始化的变量。因此，代码将无法编译。

1.  下一段代码，即 `stdin()` 函数，初始化了 `std::io::Stdin` 结构体。它从终端读取输入并将其放入 `user_input` 变量中。注意 `read_line()` 的签名接受 `&mut String`。我们必须明确告诉编译器我们正在传递一个可变引用，因为 Rust 的借用检查器，我们将在稍后的 *第九章**,* *显示用户帖子* 中讨论。`read_line()` 的输出是 `std::result::Result`，这是一个有两个变体的枚举，`Ok(T)` 和 `Err(E)`。`Result` 中的一个方法是 `expect()`，它返回一个泛型类型 `T`，如果它是 `Err` 变体，则它将结合传递的消息引发一个泛型错误 `E` 并导致程序崩溃。

1.  Rust 语言中有两个非常普遍且重要的枚举类型（`std::result::Result` 和 `std::option::Option`），因此默认情况下，我们可以在程序中使用它们而不需要指定 `use`。

接下来，我们想要能够加密输入，但到目前为止，我们还不知道我们想要使用哪种加密。我们想要做的第一件事是创建一个 **特质**，这是 Rust 语言中的一种特定代码，它告诉编译器一个类型可以有什么功能：

1.  创建模块有两种方式：创建 `module_name.rs` 或创建一个名为 `module_name` 的文件夹并在其中添加一个 `mod.rs` 文件。让我们创建一个名为 `encryptor` 的文件夹并创建一个名为 `mod.rs` 的新文件。由于我们想要稍后添加类型和实现，让我们使用第二种方式。让我们在 `mod.rs` 中写下这些内容：

    ```rs
    pub trait Encryptable {
        fn encrypt(&self) -> String;
    }
    ```

1.  默认情况下，类型或特质是私有的，但我们在 `main.rs` 中想要使用它，并在不同的文件中实现加密器，所以我们应该通过添加 `pub` 关键字来将特质标记为公共的。

1.  该特质有一个名为 `encrypt()` 的函数，它以自引用作为参数并返回 `String`。

1.  现在，我们应该在 `main.rs` 中定义这个新模块。将此行放在 `fn main` 块之前：

    ```rs
    pub mod encryptor;
    ```

1.  然后，让我们创建一个简单的类型，它实现了`Encryptable`特质。记得凯撒密码，其中密码替换一个字母为另一个字母吗？让我们实现最简单的一个叫做`ROT13`的，其中它将`'a'`转换为`'n'`，将`'n'`转换为`'a'`，将`'b'`转换为`'o'`和将`'o'`转换为`'b'`，依此类推。在`mod.rs`文件中写下以下内容：

    ```rs
    pub mod rot13;
    ```

1.  让我们在`encryptor`文件夹内创建另一个名为`rot13.rs`的文件。

1.  我们想要定义一个简单的结构体，它只包含一个数据项，即一个字符串，并告诉编译器这个结构体正在实现`Encryptable`特质。将此代码放入`rot13.rs`文件中：

    ```rs
    pub struct Rot13(pub String);
    impl super::Encryptable for Rot13 {}
    ```

你可能会注意到我们在模块声明、特质声明、结构体声明和字段声明中到处都使用了`pub`。

1.  接下来，让我们尝试编译我们的程序：

    ```rs
    > rustc main.rs 
    error[E0046]: not all trait items implemented, missing: `encrypt`
     --> encryptor/rot13.rs:3:1
      |
    3 | impl super::Encryptable for Rot13 {}
    | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ missing 
      `encrypt` in implementation
    | 
     ::: encryptor/mod.rs:6:5
      |
    6 |     fn encrypt(&self) -> String;
      |     ----------------------------------------------
      ------ `encrypt` from trait
    error: aborting due to previous error
    For more information about this error, try `rustc --explain E0046`.
    ```

这里发生了什么？显然，编译器在我们的代码中找到了错误。Rust 的一个优点是它提供了有用的编译器消息。你可以看到错误发生的行，我们代码错误的原因，有时甚至建议修复我们的代码。我们知道我们必须为`Rot13`类型实现`super::Encryptable`特质。

如果你想看到更多信息，运行前面错误中显示的命令，`rustc --explain E0046`，编译器将显示关于那个特定错误的更多信息。

1.  现在，我们可以继续实现我们的`Rot13`加密。首先，让我们将特质的签名放入我们的实现中：

    ```rs
    impl super::Encryptable for Rot13 {
        fn encrypt(&self) -> String {
        }
    }
    ```

这种加密的策略是对字符串中的每个字符进行迭代，如果它在`'n'`或`'N'`之前有一个字符，就将其字符值加 13，如果它在`'n'`或`'N'`或其后有字符，就减去 13。Rust 语言默认处理 Unicode 字符串，所以程序应该有一个限制，只能操作拉丁字母。

1.  在我们的第一次迭代中，我们想要分配一个新的字符串，获取原始`String`的长度，从零索引开始，应用一个转换，将其推入一个新的字符串，并重复直到结束：

    ```rs
    fn encrypt(&self) -> String {
        let mut new_string = String::new();
        let len = self.0.len();
        for i in 0..len {
            if (self.0[i] >= 'a' && self.0[i] < 'n') || 
            (self.0[i] >= 'A' && self.0[i] < 'N') {
                new_string.push((self.0[i] as u8 + 13) as 
                char);
            } else if (self.0[i] >= 'n' && self.0[i] < 
            'z') || (self.0[i] >= 'N' && self.0[i] < 'Z') 
            {
                new_string.push((self.0[i] as u8 - 13) as 
                char);
            } else {
                new_string.push(self.0[i]);
            }
        } 
        new_string
    }
    ```

1.  让我们尝试编译那个程序。你很快会发现它不起作用，所有的错误都是`String`不能通过`usize`索引。记住 Rust 默认处理 Unicode 吗？索引一个字符串将会产生各种复杂情况，因为 Unicode 字符有不同的尺寸：有些是 1 字节，但有些可以是 2、3 或 4 字节。关于索引，我们究竟在说什么？索引是指`String`中的字节位置、grapheme 还是 Unicode 标量值？

在 Rust 语言中，我们有原始类型，如 `u8`、`char`、`fn`、`str` 以及更多。除了这些原始类型之外，Rust 还在标准库中定义了许多模块，例如 `string`、`io`、`os`、`fmt` 和 `thread`。这些模块包含了编程的许多构建块。例如，`std::string::String` 结构体处理 `String`。重要的编程概念，如比较和迭代，也定义在这些模块中，例如，`std::cmp::Eq` 用于比较类型的实例。Rust 语言还有 `std::iter::Iterator` 使类型可迭代。幸运的是，对于 `String`，我们已经有了一个进行迭代的方法。

1.  让我们稍微修改一下我们的代码：

    ```rs
    fn encrypt(&self) -> String {
        let mut new_string = String::new();
        for ch in self.0.chars() {
            if (ch >= 'a' && ch < 'n') || (ch >= 'A' &&
            ch < 'N') {
                new_string.push((ch as u8 + 13) as char);
            } else if (ch >= 'n' && ch < 'z') || (ch >= 
            'N' && ch < 'Z') {
                new_string.push((ch as u8 - 13) as char);
            } else {
                new_string.push(ch);
            }
        }
        new_string
    }
    ```

1.  有两种返回方式；第一种是使用 `return` 关键字，如 `return new_string;`，或者我们可以在函数的最后一行不写分号。你会看到第二种形式更常见。

1.  上述代码运行正常，但我们可以让它更加符合 Rust 的风格。首先，让我们在不使用 `for` 循环的情况下处理迭代器。让我们移除新的字符串初始化，并使用 `map()` 方法。任何实现了 `std::iter::Iterator` 的类型都将有一个接受闭包作为参数并返回 `std::iter::Map` 的 `map()` 方法。然后我们可以使用 `collect()` 方法将闭包的结果收集到它自己的 `String` 中：

    ```rs
    fn encrypt(&self) -> Result<String, Box<dyn Error>> {
        self.0
            .chars()
            .map(|ch| {
                if (ch >= 'a' && ch < 'n') || (ch >= 'A' 
                && ch < 'N') {
                    (ch as u8 + 13) as char
                } else if (ch >= 'n' && ch < 'z') || (
                ch >= 'N' && ch < 'Z') {
                    (ch as u8 - 13) as char
                } else {
                    ch
                }
            })
            .collect()
    }
    ```

`map()` 方法接受形式为 `|x|...` 的闭包。然后我们使用从 `chars()` 获取的捕获的单独项进行处理。

如果你看看闭包，你会发现我们也没有使用 `return` 关键字。如果我们不在分支中放置分号，并且它是最后一项，它将被视为返回值。

使用 `if` 块是好的，但我们可以让它更加符合 Rust 的风格。Rust 语言的一个优点是强大的 `match` 控制流。

1.  让我们再次更改代码：

    ```rs
    fn encrypt(&self) -> String {
        self.0
            .chars()
            .map(|ch| match ch {
                'a'..='m' | 'A'..='M' => (ch as u8 + 13) 
                as char,
                'n'..='z' | 'N'..='Z' => (ch as u8 - 13) 
                as char,
                _ => ch,
            })
            .collect()
    }
    ```

这看起来更干净。管道 (`|`) 操作符是一个分隔符，用于匹配臂中的项。Rust 匹配器是详尽的，这意味着编译器会检查匹配器是否包含所有可能的匹配值。在这种情况下，这意味着 Unicode 中的所有字符。尝试移除最后一个臂并编译它，看看如果不包含集合中的项会发生什么。

你可以使用 `..` 或 `..=` 来定义一个范围。前者表示我们排除了最后一个元素，而后者表示我们包含了最后一个元素。

1.  现在我们已经实现了我们的简单加密器，让我们在主应用程序中使用它：

    ```rs
    fn main() {
        ...
        io::stdin()
        .read_line(&mut user_input)
        .expect("Cannot read input");
        println!(
            "Your encrypted string: {}",
            encryptor::rot13::Rot13(user_input).encrypt()
        );
    }
    ```

目前，当我们尝试编译它时，编译器会显示一个错误。基本上，编译器是在说，如果特性行不在作用域内，就不能使用特性行为，编译器提供的帮助信息显示了我们需要做什么。

1.  将以下行放在 `main()` 函数上方，编译器应该生成一个没有错误的二进制文件：

    ```rs
    use encryptor::Encryptable;
    ```

1.  让我们尝试运行可执行文件：

    ```rs
    > ./main
    Input the string you want to encrypt:
    asdf123
    Your encrypted string: nfqs123
    > ./main
    Input the string you want to encrypt:
    nfqs123
    Your encrypted string: asdf123
    ```

我们已经完成了我们的程序，并使用现实世界的加密对其进行了改进。在下一节中，我们将学习如何搜索和使用第三方库，并将它们集成到我们的应用程序中。

# 包和 Cargo

现在我们已经知道如何在 Rust 中创建一个简单的程序，让我们探索 **Cargo**，Rust 的包管理器。Cargo 是一个 **命令行应用程序**，用于管理您的应用程序依赖项并编译您的代码。

Rust 在 [`crates.io`](https://crates.io) 有一个社区包注册处。您可以使用该网站搜索您可以在应用程序中使用的库。别忘了检查您想要使用的库或应用程序的许可证。如果您在该网站上注册，您可以使用 Cargo 公开分发您的库或二进制文件。

我们如何将 Cargo 安装到我们的系统中？好消息是，如果您使用 `rustup` 在稳定通道中安装 Rust 工具链，Cargo 已经安装好了。

## Cargo 包布局

让我们在我们的应用程序中尝试使用 Cargo。首先，让我们复制我们之前编写的应用程序：

```rs
cp -r 02ComplexProgram  03Packages
cd 03Packages
cargo init . --name our_package
```

由于我们已经有了一个现有的应用程序，我们可以使用 `cargo init` 初始化我们的现有应用程序。注意我们添加了 `--name` 选项，因为我们正在将数字作为文件夹名称的前缀，而 Rust 包名称不能以数字开头。

如果我们正在创建一个新的应用程序，我们可以使用 `cargo new package_name` 命令。要创建一个仅包含库的包而不是二进制包，您可以将 `--lib` 选项传递给 `cargo new`。

您将看到文件夹内有两个新文件，`Cargo.toml` 和 `Cargo.lock`。`.toml` 文件是一种常用作配置文件的文件格式。`lock` 文件是由 Cargo 自动生成的，我们通常不会手动更改其内容。通常也会将 `Cargo.lock` 添加到您的源代码版本控制应用程序的忽略列表中，例如 `.gitignore`。

让我们检查 `Cargo.toml` 文件的内容：

```rs
[package]
```

```rs
name = "our_package"
```

```rs
version = "0.1.0"
```

```rs
edition = "2021"
```

```rs
# See more keys and their definitions at
```

```rs
https://doc.rust-lang.org/cargo/reference/manifest.html
```

```rs
[dependencies]
```

```rs
[[bin]]
```

```rs
name = "our_package"
```

```rs
path = "main.rs"
```

如您所见，我们可以为我们的应用程序定义基本内容，例如 `name` 和 `version`。我们还可以添加重要信息，如作者、主页、仓库等。我们还可以添加在 Cargo 应用程序中想要使用的依赖项。

突出的一个特点是版本配置。Rust 版本是一个可选的标记，用于将具有相同兼容性的各种 Rust 语言版本分组。当 Rust 1.0 发布时，编译器没有能力知道 `async` 和 `await` 关键字。在添加了 `async` 和 `await` 之后，它为旧编译器带来了各种问题。解决这个问题的方法是引入 Rust 版本。已经定义了三个版本：2015、2018 和 2021。

目前，Rust 编译器可以完美地编译我们的包，但它并不非常符合惯例，因为 Cargo 项目在文件和文件夹名称及结构上有约定。让我们稍微改变一下文件和目录结构：

1.  预期一个包位于`src`目录中。让我们将`Cargo.toml`文件中的`[[bin]]`路径从`"main.rs"`更改为`"src/main.rs"`。

1.  在我们的应用程序文件夹内创建`src`目录。然后，将`main.rs`文件和`encryptor`文件夹移动到`src`文件夹。

1.  在`Cargo.toml`中的`[[bin]]`之后添加这些行：

    ```rs
    [lib]
    name = "our_package"
    path = "src/lib.rs"
    ```

1.  让我们创建`src/lib.rs`文件，并将此行从`src/main.rs`移动到`src/lib.rs`：

    ```rs
    pub mod encryptor;
    ```

1.  然后，我们可以在`main.rs`文件中简化使用`rot13`和`Encryptable`模块：

    ```rs
    use our_package::encryptor::{rot13, Encryptable};
    use std::io;
    fn main() {
        ...
        println!(
            "Your encrypted string: {}",
            rot13::Rot13(user_input).encrypt()
        );
    }
    ```

1.  我们可以通过在命令行中输入`cargo check`来检查是否有错误阻止代码编译。它应该产生类似以下的内容：

    ```rs
    > cargo check
    Checking our_package v0.1.0 
        (/Users/karuna/Chapter01/03Packages)
    Finished dev [unoptimized + debuginfo] target(s) 
        in 1.01s
    ```

1.  之后，我们可以使用`cargo build`命令构建二进制文件。由于我们没有在命令中指定任何选项，默认的二进制文件应该是未优化的，并包含调试符号。生成的二进制文件的默认位置是在工作区根目录下的`target`文件夹：

    ```rs
    $ cargo build
    Compiling our_package v0.1.0 
       (/Users/karuna/Chapter01/03Packages)
    Finished dev [unoptimized + debuginfo] target(s) 
        in 5.09s
    ```

然后，你可以按照以下方式在`target`文件夹中运行二进制文件：

```rs
./target/debug/our_package
```

`debug`默认由开发配置启用，`our_package`是我们指定在`Cargo.toml`中的名称。

如果你想要创建一个发布版本的二进制文件，你可以指定`--release`选项，使用`cargo build --release`。你可以在`./target/release/our_package`中找到发布版本的二进制文件。

你也可以输入`cargo run`，这将为你编译和运行应用程序。

现在我们已经安排好了应用程序结构，让我们通过使用第三方 crate 为我们的应用程序添加实际的加密功能。

## 使用第三方 crate

在我们使用第三方模块实现另一个加密器之前，让我们稍微修改一下我们的应用程序。将之前的`03Packages`文件夹复制到新的文件夹`04Crates`中，并使用该文件夹进行以下步骤：

1.  我们将重命名我们的 Encryptor 特质为 Cipher 特质并修改函数。原因是我们只需要考虑类型的输出，而不是加密过程本身：

    +   让我们将`src/lib.rs`的内容更改为`pub mod cipher;`。

    +   之后，将`encryptor`文件夹重命名为`cipher`。

    +   然后，将 Encryptable 特质修改为以下内容：

        ```rs
        pub trait Cipher {
            fn original_string(&self) -> String;
            fn encrypted_string(&self) -> String;
        }
        ```

事实上，我们只需要函数来显示原始字符串和加密字符串。我们不需要在类型本身暴露加密。

1.  之后，让我们也将`src/cipher/rot13.rs`修改为使用重命名的特质：

    ```rs
    impl super::Cipher for Rot13 {
        fn original_string(&self) -> String {
            String::from(&self.0)
        }
        fn encrypted_string(&self) -> String {
            self.0
                .chars()
                .map(|ch| match ch {
                    'a'..='m' | 'A'..='M' => (ch as u8 + 
                    13) as char,
                    'n'..='z' | 'N'..='Z' => (ch as u8 – 
                    13) as char,
                    _ => ch,
                })
                .collect()
        }
    }
    ```

1.  让我们也修改`main.rs`以使用新的特性和函数：

    ```rs
    use our_package::cipher::{rot13, Cipher};
    …
    fn main() {
        …
        println!(
            "Your encrypted string: {}",
            rot13::Rot13(user_input).encrypted_string()
        );
    }
    ```

下一步是确定我们想要为我们新的类型使用什么加密和库。我们可以访问[`crates.io`](https://crates.io)并搜索可用的包。在网站上搜索实际的加密算法后，我们找到了[`crates.io/crates/rsa`](https://crates.io/crates/rsa)。我们发现 RSA 算法是一个安全的算法，该包有良好的文档，并且已经由安全研究人员审核过，许可证与我们的需求兼容，并且下载量巨大。除了检查这个库的源代码外，所有迹象都表明这是一个很好的包来使用。幸运的是，在那个页面的右侧有一个安装部分。除了`rsa`包外，我们还将使用`rand`包，因为 RSA 算法需要一个随机数生成器。由于生成的加密是字节形式，我们必须以某种方式将其编码为`string`。常见的方法之一是使用`base64`。

1.  在我们的`Cargo.toml`文件中，在`[dependencies]`部分添加以下行：

    ```rs
    rsa = "0.5.0"
    rand = "0.8.4"
    base64 = "0.13.0"
    ```

1.  下一步应该是添加一个新的模块并使用`rsa`包。但是，对于这个类型，我们想稍作修改。首先，我们想要创建一个**关联函数**，在其他语言中可能被称为构造函数。我们想要在这个函数中加密输入字符串并将加密后的字符串存储在一个字段中。有句话说是所有不在处理中的数据都应该默认加密，但事实是，作为程序员，我们很少这样做。

由于 RSA 加密涉及字节操作，存在错误的可能性，因此关联函数的返回值应该被包裹在`Result`类型中。虽然没有编译器规则，但如果一个函数不能失败，其返回值应该是直接的。无论函数是否可以产生结果，返回值应该是`Option`，但如果函数可以产生错误，使用`Result`会更好。

`encrypted_string()`方法应该返回存储的加密字符串，而`original_string()`方法应该解密存储的字符串并返回纯文本。

在`src/cipher/mod.rs`中，将代码更改为以下内容：

```rs
pub trait Cipher {
    fn original_string(&self) -> Result<String, 
    Box<dyn Error>>;
    fn encrypted_string(&self) -> Result<String, 
    Box<dyn Error>>;
}
```

1.  由于我们改变了特性的定义，我们不得不将`src/cipher/rot13.rs`中的代码也进行更改。将代码更改为以下内容：

    ```rs
    use std::error::Error;
    pub struct Rot13(pub String);
    impl super::Cipher for Rot13 {
        fn original_string(&self) -> Result<String, 
        Box<dyn Error>> {
            Ok(String::from(&self.0))
        }
    fn encrypted_string(&self) -> Result<String, 
        Box<dyn Error>> {
            Ok(self
                .0
                ...
                .collect())
        }
    }
    ```

1.  让我们在`src/cipher/mod.rs`文件中添加以下行：

    ```rs
    pub mod rsa;
    ```

1.  之后，在`cipher`文件夹内创建`rsa.rs`文件，并在其中创建`Rsa`结构体。请注意，我们使用`Rsa`而不是`RSA`作为类型名。按照惯例，类型名应使用`CamelCase`格式：

    ```rs
    use std::error::Error;
    pub struct Rsa {
        data: String,
    }
    impl Rsa {
        pub fn new(input: String) -> Result<Self, Box<
        dyn Error>> {
            unimplemented!();
        }
    }
    impl super::Cipher for Rsa {
        fn original_string(&self) -> Result<String, ()> {
           unimplemented!();
        }
        fn encrypted_string(&self) -> Result<String, ()> {
            Ok(String::from(&self.data))
        }
    }
    ```

我们可以观察到一些事情。首先，`data`字段没有`pub`关键字，因为我们想将其设置为私有。你可以看到我们有两个`impl`块：一个是用于定义`Rsa`类型的自身方法，另一个是用于实现`Cipher`特质。

此外，`new()`函数没有`self`、`mut self`、`&self`或`&mut self`作为第一个参数。在其他语言中，将其视为一个静态方法。此方法返回`Result`，要么是`Ok(Self)`，要么是`Box<dyn Error>`。`Self`实例是`Rsa`结构体的实例，但我们将稍后讨论`Box<dyn Error>`，当我们在*第七章*中讨论 Rust 和 Rocket 的错误处理时。现在，我们还没有实现此方法，因此使用了`unimplemented!()`宏。Rust 中的宏看起来像函数，但有一个额外的感叹号（!）。

1.  现在，让我们实现关联函数。修改`src/cipher/rsa.rs`：

    ```rs
    use rand::rngs::OsRng;
    use rsa::{PaddingScheme, PublicKey, RsaPrivateKey};
    use std::error::Error;
    const KEY_SIZE: usize = 2048;
    pub struct Rsa {
        data: String,
        private_key: RsaPrivateKey,
    }
    impl Rsa {
         pub fn new(input: String) -> Result<Self, Box<
        dyn Error>> {
            let mut rng = OsRng;
            let private_key = RsaPrivateKey::new(&mut rng, 
            KEY_SIZE)?;
            let public_key = private_key.to_public_key();
            let input_bytes = input.as_bytes();
            let encrypted_data =
                public_key.encrypt(&mut rng, PaddingScheme
                ::new_pkcs1v15_encrypt(), input_bytes)?;
            let encoded_data = 
            base64::encode(encrypted_data);
            Ok(Self {
                data: encoded_data,
                private_key,
            })
        }
    }
    ```

我们首先声明我们将要使用的各种类型。之后，我们定义一个常量来表示我们将使用什么大小的密钥。

如果你理解 RSA 算法，你已经知道它是一个非对称算法，这意味着我们有两个密钥：一个公钥和一个私钥。我们使用公钥加密数据，使用私钥解密数据。我们可以生成公钥并将其提供给另一方，但我们不希望将私钥提供给另一方。这意味着我们必须将私钥存储在结构体内部。

`new()`的实现相当直接。我们首先声明一个随机数生成器，`rng`。然后，我们生成 RSA 私钥。但请注意私钥初始化时的问号运算符（`?`）。如果一个函数返回`Result`，我们可以在调用其内部任何方法或函数后快速返回由该函数生成的错误，只需在该函数后使用（`?`）即可。

然后，我们从私钥生成 RSA 公钥，将输入字符串编码为字节，并加密数据。由于加密数据可能会导致错误，我们再次使用问号运算符。然后，我们将加密的字节编码为`base64`字符串，并初始化`Self`，这意味着`Rsa`结构体本身。

1.  现在，让我们实现`original_string()`方法。我们应该做与我们创建结构体时相反的操作：

    ```rs
    fn original_string(&self) -> Result<String, Box<dyn Error>> {
        let decoded_data = base64::decode(&self.data)?;
        let decrypted_data = self
            .private_key
            .decrypt(PaddingScheme::
            new_pkcs1v15_encrypt(), &decoded_data)?;
        Ok(String::from_utf8(decrypted_data)?)
    }
    ```

首先，我们解码`data`字段中的`base64`编码字符串。然后，我们解密解码的字节并将它们转换回字符串。

1.  现在我们已经完成了`Rsa`类型，让我们在`main.rs`文件中使用它：

    ```rs
    fn main() {
        ...
        println!(
            "Your encrypted string: {}",
            rot13::Rot13(user_input).encrypted_
            string().unwrap()
        );
        println!("Input the string you want to encrypt:");
        let mut user_input = String::new();
        io::stdin()
            .read_line(&mut user_input)
            .expect("Cannot read input");
        let encrypted_input = rsa::Rsa::new(
        user_input).expect("");
        let encrypted_string = encrypted_input.encrypted_
        string().expect("");
        println!("Your encrypted string: {}", 
        encrypted_string);
        let decrypted_string = encrypted_input
        .original_string().expect("");
        println!("Your original string: {}", 
        decrypted_string);
    }
    ```

一些读者可能会想知道为什么我们重新声明了`user_input`变量。简单的解释是 Rust 已经将资源移动到了新的`Rot13`类型，而 Rust 不允许重复使用已移动的值。你可以尝试注释掉第二个变量声明并编译应用程序以查看解释。我们将在*第九章*中更详细地讨论 Rust 的借用检查器和移动。

现在，尝试通过输入`cargo run`来运行程序：

```rs
$ cargo run
   Compiling cfg-if v1.0.0
   Compiling subtle v2.4.1
   Compiling const-oid v0.6.0
   Compiling ppv-lite86 v0.2.10
   ...
Compiling our_package v0.1.0 
   (/Users/karuna//Chapter01/04Crates)
Finished dev [unoptimized + debuginfo] target(s) 
    in 3.17s
     Running `target/debug/our_package`
Input the string you want to encrypt:
first
Your encrypted string: svefg
Input the string you want to encrypt:
second
Your encrypted string: lhhb9RvG9zI75U2VC3FxvfUujw0cVqqZFgPXhNixQTF7RoVBEJh2inn7sEefDB7eNlQcf09lD2nULfgc2mK55ZE+UUcYzbMDu45oTaPiDPog4L6FRVpbQR27bkOj9Bq1KS+QAvRtxtTbTa1L5/OigZbqBc2QOm2yHLCimMPeZKhLBtK2whhtzIDM8l5AYTBg+rA688ZfB7ZI4FSRm4/h22kNzSPo1DECI04ZBprAq4hWHxEKRwtn5TkRLhClGFLSYKkY7Ajjr3EOf4QfkUvFFhZ0qRDndPI5c9RecavofVLxECrYfv5ygYRmW3B1cJn4vcBhVKfQF0JQ+vs+FuTUpw==
Your original string: second
```

你会发现 Cargo 会自动下载依赖并逐个构建它们。此外，你可能注意到使用`Rsa`类型加密花费了一些时间。Rust 不是应该是一个快速的系统语言吗？RSA 算法本身就是一个慢速算法，但这并不是速度慢的真正原因。因为我们是在开发配置下运行程序，Rust 编译器生成一个包含所有调试信息的应用程序二进制文件，并且不对生成的二进制文件进行优化。另一方面，如果你使用`--release`标志构建应用程序，编译器会生成一个优化的应用程序二进制文件并删除调试符号。使用发布标志编译的结果二进制文件应该比调试二进制文件执行得更快。试着亲自做一下，这样你会记住如何构建发布二进制文件。

在本节中，我们学习了 Cargo 和第三方包，所以接下来，让我们了解一下如何找到我们使用的工具的帮助和文档。

# 工具和获取帮助

现在我们已经创建了一个相当简单的应用程序，你可能想知道我们可以使用哪些工具进行开发，以及如何了解更多关于 Rust 和获取帮助。

## 工具

除了 Cargo，我们还可以使用一些其他工具来开发 Rust 应用程序：

+   **rustfmt**

这个程序用于格式化你的源代码，使其遵循 Rust 风格指南。你可以通过使用`rustup`（`rustup component add rustfmt`）来安装它。然后，你可以将其集成到你的 favorite text editor 或者在命令行中使用它。你可以在[`github.com/rust-lang/rustfmt`](https://github.com/rust-lang/rustfmt)了解更多关于`rustfmt`的信息。

+   **clippy**

这个名字让你想起了什么吗？`clippy`通过使用各种 lint 规则对你的 Cargo 应用程序进行 linting 非常有用。目前，你可以使用超过 450 个 lint 规则。你可以使用以下命令安装它：`rustup component add clippy`。之后，你可以通过运行`cargo clippy`在 Cargo 应用程序中使用它。你能在我们之前写的 Cargo 应用程序中尝试一下吗？你可以在[`github.com/rust-lang/rust-clippy`](https://github.com/rust-lang/rust-clippy)了解更多关于`clippy`的信息。

## 文本编辑器

很可能，你选择的文本编辑器已经支持 Rust 语言，或者至少支持 Rust 的语法高亮。如果你想添加诸如跳转到定义、跳转到实现、符号搜索和代码补全等重要功能，你可以安装 Rust 语言服务器。大多数流行的文本编辑器已经支持语言服务器，所以你只需在你的文本编辑器中安装扩展或其他集成方法即可：

+   **Rust 语言服务器**

你可以使用`rustup`命令安装它：`rustup component add rls rust-analysis rust-src`。然后，你可以将其集成到你的文本编辑器中。例如，如果你正在使用`rls`。

你可以在[`github.com/rust-lang/rls`](https://github.com/rust-lang/rls)了解更多关于它的信息。

+   **Rust analyzer**

这个应用程序有望成为 Rust 语言服务器 2.0。截至本书编写时，它仍被视为处于 alpha 阶段，但根据我的经验，这个应用程序与常规更新配合得很好。你可以在[`github.com/rust-analyzer/rust-analyzer/releases`](https://github.com/rust-analyzer/rust-analyzer/releases)找到这个应用程序的可执行文件，然后配置你的编辑器语言服务器以使用此应用程序。你可以在[`rust-analyzer.github.io`](https://rust-analyzer.github.io)了解更多关于它的信息。

## 获取帮助和文档

有几份重要的文档，你可能想要阅读以获取帮助或参考：

+   **《Rust 编程语言书籍》**：如果你想了解更多关于 Rust 编程语言的信息，这本书是你想要阅读的。你可以在[`doc.rust-lang.org/book/`](https://doc.rust-lang.org/book/)在线找到它。

+   **Rust 示例**：这份文档是关于 Rust 语言及其标准库功能概念的示例集合。你可以在[`doc.rust-lang.org/rust-by-example/index.html`](https://doc.rust-lang.org/rust-by-example/index.html)在线阅读它。

+   **标准库文档**：作为一名程序员，你将参考这份标准库文档。你可以了解更多关于标准库、它们的模块、函数签名、标准库函数的功能、阅读示例等内容。你可以在[`doc.rust-lang.org/std/index.html`](https://doc.rust-lang.org/std/index.html)找到它。

+   `Cargo.toml`清单格式，你可以在[`doc.rust-lang.org/cargo/index.html`](https://doc.rust-lang.org/cargo/index.html)了解更多关于它的信息。

+   **Rust 风格指南**：Rust 语言，像其他编程语言一样，有风格指南。这些指南告诉程序员命名约定是什么，关于空白，如何使用常量，以及 Rust 程序的其它惯用约定。你可以在[`doc.rust-lang.org/1.0.0/style/`](https://doc.rust-lang.org/1.0.0/style/)了解更多关于它的信息。

+   我们之前使用的`rsa`包。要找到该库的文档，你可以访问[`crates.io`](https://crates.io)，搜索该包的页面，然后转到右侧面板并进入文档部分。或者，你可以访问[`docs.rs`](https://docs.rs)，搜索包名并找到它的文档。

+   `rustup`（`rustup component add rust-docs`）。然后，你可以使用`rustup doc`命令在离线状态下打开文档。如果你想离线打开标准库文档，你可以输入`rustup doc --std`。还有其他可以打开的文档；尝试使用`rustup doc --help`来查看它们是什么。

+   **Rust 用户论坛**：如果你想得到帮助或帮助其他 Rust 程序员，你可以在互联网上找到所有这些。在[`users.rust-lang.org/`](https://users.rust-lang.org/)有一个专门的论坛来讨论与 Rust 相关的话题。

# 摘要

在本章中，我们对 Rust 语言进行了简要概述。我们学习了 Rust 工具链及其安装方法，以及 Rust 开发所需的工具。之后，我们创建了两个简单的程序，使用了 Cargo，并导入第三方模块以改进我们的程序。现在，你已经可以用 Rust 语言编写小程序了，去探索吧！尝试创建更多程序或对语言进行实验。你可以尝试使用 *Rust by Example* 来查看我们可以在程序中使用哪些特性。在随后的章节中，我们将学习更多关于 Rocket 的内容，这是一个用 Rust 语言编写的网络框架。
