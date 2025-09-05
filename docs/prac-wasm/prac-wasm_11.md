# 第八章：*第八章*：使用 wasm-pack 打包 WebAssembly

JavaScript 无处不在，但无处不在既是优点也是缺点。有许多不同的生态系统，它们有不同的标准和目的。为所有生态系统构建独特的解决方案并不实际。

尽管如此，JavaScript 社区在这里做得非常出色。社区的付出使 JavaScript 成为首选语言之一。对于像 JavaScript 这样多才多艺的语言，总会有些奇怪的地方（当然，每种语言都有）。当您编写 JavaScript 时，这些地方需要额外的关注和注意。

JavaScript 是动态类型的。这使得避免运行时异常变得困难（几乎不可能）。虽然 TypeScript、Flow 和 Elm 试图在 JavaScript 的动态类型上提供（类型化的）超集，但它们无法完全解决根本问题。

任何语言要成长，都必须快速演变，JavaScript 就是这样做的。快速演变而不破坏现有用法也同样重要，JavaScript 提供了 polyfills 来实现向后兼容。

但创建 polyfills 是一项平凡的任务。还有许多其他平凡的任务，例如打包和包装库、压缩包和懒加载库，仅举几例。打包器为大多数这些问题提供了解决方案。它们充当前端编译器。

到目前为止，我们已经看到了 Rust 如何使创建和运行 WebAssembly 模块变得容易。在本章中，我们将探讨 `wasm-pack`，这是一个使打包和发布 WebAssembly 模块更容易的工具。本章将涵盖以下部分：

+   使用 webpack 打包 WebAssembly 模块

+   使用 Parcel 打包 WebAssembly 模块

+   介绍 `wasm-pack`

+   使用 `wasm-pack` 打包和发布

# 技术要求

您可以在 GitHub 上找到本章中存在的代码文件，网址为[`github.com/PacktPublishing/Practical-WebAssembly`](https://github.com/PacktPublishing/Practical-WebAssembly)。

# 使用 webpack 打包 WebAssembly 模块

webpack 是现代 JavaScript 应用程序的静态模块打包器。那么，它做什么呢？

您可以将 webpack 视为前端的一个非正式编译器。webpack 接受一个应用程序的入口点，逐步运行模块，并构建依赖图。依赖图包含所有模块。这些模块对于应用程序的运行是必要的。

一旦构建了依赖图，webpack 就会输出一个或多个包。webpack 非常灵活，帮助我们根据需要打包或包装 JavaScript，webpack 配置中提供了选项。根据提供的选项，webpack 创建输出。

*听起来很简单，对吧？*

几年前，当我们需要唯一的库就是 jQuery 时，事情很简单。

但由于 JavaScript 的快速演变，现在有很多不同的事情正在发生。底层运行时不尽相同。存在三种不同的浏览器引擎和多种目标。

浏览器引擎的演进速度不同，浏览器支持各种版本的 JavaScript。在某些工作场所的机器上，升级到最新版本的浏览器是被禁止的。这意味着运行的 JavaScript 应用需要在不同的时间进行调整和填充。

基础目标系统需要对您的 JavaScript 代码进行一定的调整才能运行。手动完成所有这些工作将花费很长时间，并且容易出错。

JavaScript 有多种变体，包括 TypeScript 和 CoffeeScript。它们各不相同，但在运行之前都会编译成 JavaScript。基于浏览器的开发需要 CSS、SCSS、SASS 和 LESS。支持所有这些变体并在每次更改后手动编译它们并不是一件容易的事情。

JavaScript 对此的回应是打包器。无论你讨厌它们还是喜欢它们，打包器都能在用 JavaScript 开发时减少负担和混乱。

webpack 为所有这些问题以及更多问题提供了一个解决方案。

webpack 是一个用于打包 JavaScript 应用的工具。它包含加载器和插件，可以帮助转换、添加、删除和操作输出包。webpack 最有趣的部分是其加载器和插件，它们将 webpack 的能力发挥到极致。

加载器允许我们像在 JavaScript 中加载或导入任何其他模块一样加载或导入 Rust、CSS 或 TypeScript 文件。然后 webpack 负责生成支持指定目标环境的包。

插件允许我们优化和管理生成的包。需要注意的是，webpack 完全建立在插件系统之上。

*webpack 如何帮助 WebAssembly？*

webpack 内部依赖于 `webassemblyjs` 库。因此，所有使用 webpack 的应用都已经为 WebAssembly 准备就绪。你只需像加载普通 JavaScript 文件一样开始加载 WebAssembly 文件，webpack 就会处理其余部分。

在 webpack 配置中，我们将定义入口点。然后 webpack 加载入口文件。入口文件中的 `import` 语句根据 JavaScript 的模块解析算法被加载为一个模块。如果导入的模块是 WebAssembly 模块，它将获取模块的内容并将其交给 `webassemblyjs` 编译器。

编译器负责解析和修改 WebAssembly 模块。

你知道吗？

`webassemblyjs` 可以直接解析 WebAssembly 文本格式和 WebAssembly 二进制格式。

编译器生成 **抽象语法树**（**AST**）。生成的 AST 然后进行验证。一旦验证成功，WebAssembly 模块中的任何自定义部分都将被移除。

自定义部分是 WebAssembly 模块内部的一个部分，用户可以在此存储关于 WebAssembly 模块的自定义信息。这些信息可能包括函数和局部变量的名称。浏览器可以使用这些信息来改善调试过程。

webpack 也不支持起始部分。起始部分是 WebAssembly 模块中的一个部分，将在 WebAssembly 模块加载时立即调用。

相反，webpack 创建一个函数，并在 WebAssembly 模块加载后调用它。`webassemblyjs` 移除了起始部分，并将起始函数转换为 WebAssembly 模块上的普通函数。然后，webpack 负责生成调用该函数的包装器，以便模块加载后立即调用。

最后，`webassemblyjs` 还负责优化二进制文件，并从 WebAssembly 模块中消除死代码。

`webassemblyjs` 内置了解释器和 CLI，这使得实验 WebAssembly 模块变得容易。

**代码，刷新，重复。**

这长期以来一直是 Web 开发的流程。实时重新加载为 Web 开发者提供了额外的帮助。一旦代码保存，实时重新加载可以自动编译和重新加载更改。代码可以在多个设备、因素和方向之间共享。在一个地方进行的交互可以自动与其他设备同步。虽然 Web 提供了一种轻松交付软件的媒介，但它以各种形式存在。这些形式包括功能手机、智能手机、平板电脑、笔记本电脑、计算机、超宽显示器、360 度虚拟世界等等。支持所有或其中一些是一项艰巨的任务。实时重新加载就像一双额外的手。

webpack 为添加实时重新加载到您的应用程序提供了多个选项。它为实时重新加载工具，如 BrowserSync，提供了插件。webpack 生态系统在其配置中也提供了监视模式。

一旦启用，监视模式会查找源文件及其目录中发生的任何更改。一旦检测到更改，它将自动重新编译。但监视模式是为了将输入重新编译为输出。

自动重新加载网页是由一个名为 webpack-dev-server 的库提供的。webpack-dev-server 是一个内存中的 Web 服务器。内容是在内存中生成和放置的，而不是在文件系统中的实际文件中。

此外，webpack-dev-server 还支持热模块替换。这允许服务器仅修补浏览器中的更改，而不是进行完整的页面刷新。

让我们看看如何在 WebAssembly 项目中启用实时重新加载：

1.  首先，我们将创建一个新的 Rust 项目：

    ```rs
    $ cargo new --lib live_reload
      Created library `live_reload` package
    ```

1.  一旦创建了项目，就在您喜欢的编辑器中打开它。为了为项目定义 `wasm-bindgen` 依赖项，打开 `Cargo.toml` 文件：

    ```rs
    [package]
    name = "live_reload"
    version = "0.1.0"
    authors = ["Sendil Kumar"]
    edition = "2018"

    [lib]
    crate-type = ["cdylib"]

    [dependencies]
    wasm-bindgen = "0.2.38"
    ```

首先，添加 `[lib]` 部分，并添加 `crate-type = ["cdylib"]`。通过 `crate-type` 选项，我们指示编译器该库是动态的。之后，将 `wasm-bindgen` 依赖项添加到 `[dependencies]` 标签中。

1.  然后，打开 `src/lib.rs` 文件，并用以下内容替换其内容：

    ```rs
    use wasm_bindgen::prelude::*;

    #[wasm_bindgen]
    pub fn hello_world() -> String {
    "Hello World".to_string()
    }
    ```

1.  我们将在这里重用前几章中的简单 Hello World 示例。使用以下命令构建 WASM 模块：

    ```rs
    $ cargo build --target wasm32-unknown-unknown
    $ wasm-bindgen target/wasm32-unknown-
      unknown/debug/live_reload.wasm --out-dir .
    ```

1.  然后创建一个`webpack.config.js`文件来指导 webpack 如何处理和编译文件：

    ```rs
    const path = require('path');
    const HtmlWebpackPlugin = require('html-webpack-
      plugin');

    module.exports = {
        entry: './index.js',
        output: {
            path: path.resolve(__dirname, 'dist'),
            filename: 'bundle.js',
        },
        plugins: [
            new HtmlWebpackPlugin(),
        ],
        experiments: {
           syncWebAssembly: true,
        },
        mode: 'development'
    };
    ```

1.  添加一个`package.json`文件来下载 webpack 依赖项：

    ```rs
    {
        "scripts": {
            "build": "webpack",
            "serve": "webpack-dev-server"
        },
        "devDependencies": {
            "html-webpack-plugin": "⁵.5.0",
            "webpack": "⁵.64.1",
            "webpack-cli": "⁴.9.1",
            "webpack-dev-server": "⁴.5.0"
        }
    }
    ```

    注意

    请使用此处适用的最新版本的依赖项。

1.  创建一个`index.js`文件来加载绑定 JavaScript，该 JavaScript 反过来加载生成的 WebAssembly 模块：

    ```rs
    import("./live_reload").then(module => {
        console.log(module.hello_world());
    });
    ```

1.  现在，转到终端并使用以下命令安装 npm 依赖项：

    ```rs
    $ npm install
    ```

使用以下命令运行 webpack-dev-server:

```rs
$ npm run serve 
```

我们已经使用 webpack-dev-server 来启用自动重新编译。现在我们可以去更改 HTML、CSS 或 JavaScript 文件。一旦我们保存更改，webpack 服务器将编译一切。一旦编译完成，更改将在浏览器中反映出来。

但等等，如果你更改 Rust 文件会发生什么？让我们尝试更改它：

```rs
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn hello_world() -> String {
"Hello Universe".to_string()
}
```

我们在`main.rs`文件中进行了重大更改。是的，我们从*world*更改为*universe*；这不是很大吗？但一旦你保存文件，你将不会在浏览器中看到任何更改。事实上，甚至 webpack 编译器也没有重新编译东西。

webpack 编译器默认查找将在 HTML、CSS 和 JavaScript 文件中发生的更改（在配置文件中定义的东西以及包含在依赖图中的东西）。但它对 Rust 代码一无所知。

我们需要以某种方式告诉 webpack 在 Rust 中查找代码更改。我们可以使用一个插件来完成这个任务，该插件将检查指定文件类型的指定位置中的任何更改。然后，它将重新触发构建过程。我们将使用`wasm-pack-plugin`来完成这个任务。

使用以下命令将`wasm-pack-plugin`依赖项添加到应用程序中：

```rs
$ npm i @wasm-tool/wasm-pack-plugin -D
```

然后，通过`webpack.config.js`文件将此插件钩入 webpack 的插件系统：

```rs
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const WasmPackPlugin = require('@wasm-tool/wasm-pack-
  plugin');

module.exports = {
    entry: './index.js',
    output: {
        path: path.resolve(__dirname, 'dist'),
        filename: 'bundle.js',
    },
    plugins: [
        new HtmlWebpackPlugin(),
        new WasmPackPlugin({
                crateDirectory: path.resolve(__dirname)
        }),
    ],
    experiments: {
       syncWebAssembly: true,
    },
    mode: 'development'
};
```

我们导入`wasm-pack-plugin`。我们指定包含`Cargo.toml`文件的 crate 目录，然后插件将负责自动重新加载部分。要看到它的实际效果，让我们使用`npm run serve`停止并启动 webpack 服务器。

现在，让我们用 Hello Galaxy 编辑`src/main.rs`文件。打开浏览器以查看控制台日志已更改为**Hello Galaxy**。

那么，这里发生了什么？

`wasm-pack-plugin`通过 webpack 的插件系统连接到 webpack。这将与 webpack 编译器一起运行。如果`src`目录中发生任何更改，`wasm-pack-plugin`将运行`wasm-pack`编译来自动将 Rust 代码编译成 WebAssembly 模块。这将触发 webpack 编译器的重新编译。一旦 webpack 编译器重新编译，它将通知`webpack-dev-server`在浏览器中重新加载更改。然后浏览器将自动重新加载更改。

`wasm-pack-plugin`使得在 webpack 中运行 Rust 和 WebAssembly 变得容易。现在，让我们检查如何使用 Parcel 运行 Rust 和 WebAssembly。

# 使用 Parcel 打包 WebAssembly 模块

*Parcel 是一个快速、零配置的 Web 应用程序打包器*`index.html`)，然后从那里构建整个图。

虽然 Webpack 具有基于插件的架构，但 Parcel 具有基于工作进程的架构。这使得 Parcel 比 Webpack 更快，因为它使用多核编译和缓存。

Parcel 还内置了配置以支持 JavaScript、CSS 和 HTML 文件。就像 Webpack 一样，它也有各种插件，我们可以使用这些插件来配置打包器以生成所需的输出。

当需要时，它还内置了对标准 Babel、PostCSS 和 PostHTML 的转换支持。如果需要，我们可以通过插件扩展和更改它们。

Parcel 还具有自动的、开箱即用的热模块替换功能，用于跟踪和记录文件的更改（这些更改由依赖关系图记录）。让我们使用 Parcel 作为打包器来构建 WebAssembly 模块：

1.  我们将首先创建一个新的 Rust 项目：

    ```rs
    $ cargo new --lib live_reload_parcel
      Created library `live_reload_parcel` package
    ```

一旦创建项目，就在您最喜欢的编辑器中打开项目。

1.  要为项目定义`wasm-bindgen`依赖项，打开`Cargo.toml`文件：

    ```rs
    [package]
    name = "live_reload_parcel"
    version = "0.1.0"
    authors = ["Sendil Kumar"]
    edition = "2018"

    [lib]
    crate-type = ["cdylib"]

    [dependencies]
    wasm-bindgen = "0.2.38"
    ```

首先，删除`[dependencies]`标签，并用上面的粗体行替换它。我们正在告诉编译器，生成的库将是动态的，并且它依赖于`wasm-bindgen`库。

1.  然后，我们打开`src/lib.rs`文件，并将其内容替换为以下内容：

    ```rs
    use wasm_bindgen::prelude::*;

    #[wasm_bindgen]
    pub fn hello_world() -> String {
    "Hello World".to_string()
    }
    ```

1.  我们将重用简单的 Hello World 示例。使用以下命令构建`wasm`模块：

    ```rs
    $ cargo build --target wasm32-unknown-unknown
    $ wasm-bindgen target/wasm32-unknown-
      unknown/debug/live_reload_parcel.wasm --out-dir .
    ```

1.  由于 Parcel 支持零配置，我们所需做的只是将 Parcel 依赖项添加到`package.json`中：

    ```rs
    {
        "scripts": {
            "build": "parcel index.html",
            "serve": "parcel build index.html"
        },
        "devDependencies": {
            "parcel-bundler": "¹.12.3"
        }
    }
    ```

由于 Parcel 是零配置打包器，我们只需定义它的入口点。我们在`scripts`部分定义入口点。`serve`命令是我们用于开发目的运行代码的命令。当我们定义`parcel build index.html`时，我们正在通知 Parcel 入口点是`index.html`。

注意

请使用此处适用的最新版本。

1.  然后，我们将创建入口点。我们将创建一个`index.html`文件，正如`package.json`脚本中指定的那样：

    ```rs
    <html>
        <head>
            ...
            <script src="img/index.js"> </script>
        </head>
        <body> ... </body>
    </html>
    ```

1.  创建一个`index.js`文件来加载绑定 JavaScript，它反过来加载生成的 WebAssembly 模块：

    ```rs
    import { module } from './live_reload_parcel.js';
    module.hello_world();
    ```

1.  现在，转到终端。运行以下命令来安装依赖项：

    ```rs
    $ npm install
    ```

使用以下命令运行 Parcel 应用程序：

```rs
$ npm run serve
```

Parcel 的零配置特性使得开始使用 WebAssembly 非常容易。默认情况下，Parcel 支持 `.wasm` 文件。我们甚至可以像导入任何其他 `.js` 文件一样导入 `.wasm` 文件。

重要的是要注意，同步导入 WebAssembly 模块仍然不受支持。但我们可以将导入写为同步导入。内部，Parcel 将生成必要的额外代码，在 JavaScript 执行开始前预加载文件。

这意味着 WebAssembly 文件将是一个单独的包，而不是与打包的 JavaScript 文件一起。

1.  让我们更改 Rust 文件并看看会发生什么：

    ```rs
     use wasm_bindgen::prelude::*;

    #[wasm_bindgen]
    pub fn hello_world() -> String {
    "Hello Universe".to_string()
    }
    ```

1.  保存文件后，您将看不到任何变化。Parcel 无法得知您已更改源代码，编译器也不会做出反应。

要使 Parcel 对 Rust 源代码更改做出反应，我们需要添加一个插件。该插件是 `parcel-plugin-wasm.rs`。

1.  要安装插件，我们可以运行以下命令：

    ```rs
    npm install -D parcel-plugin-wasm.rs
    ```

这将把插件下载到 `node_modules`。这也会在 `package.json` 的 `devDependencies` 中保存插件。

1.  安装完成后，我们需要更改 `index.js`，使其直接查看源代码，而不是从 `Cargo.toml` 文件中引用：

    ```rs
    import { hello_world } from './src/lib.rs';
    // import { hello_world } from './Cargo.toml'; 
    hello_world();
    ```

在这里，我们不是从 WebAssembly 模块导入，而是指定入口 Rust 文件。我们甚至可以指定 `Cargo.toml` 文件的位置，以便 Parcel 在相应位置查找更改。

1.  现在，让我们用 Hello Galaxy 编辑 `src/main.rs` 文件。打开浏览器查看控制台日志如何变为 **Hello Galaxy**。

那么，这里发生了什么？

Parcel 只需要我们应用程序的起点。它将从那里生成依赖图。Parcel 插件会持续查找文件夹中的任何更改。它基本上会查看包含 `Cargo.toml` 文件的文件夹。`Cargo.toml` 的位置是通过 `index.js` 传递给 Parcel 打包器和其插件的。

因此，对 Rust 文件所做的任何更改都将导致以下过程。

当 Rust 文件被保存时，`parcel-plugin-wasm.rs` 内部的监视器会被触发。然后，`parcel-plugin-wasm.rs` 将通过 `wasm-pack` 启动 Rust 代码编译成 WebAssembly 的过程。一旦 `wasm-pack` 编译并生成新的 WebAssembly 代码，插件将通知 Parcel 编译器依赖图中的某个部分发生了变化。

然后，Parcel 编译器会重新编译，这将导致浏览器刷新。浏览器现在显示更改后的消息。

注意，对于 Parcel，我们实际上使用了同步模块导入，而对于 webpack，我们依赖于异步导入。

`parcel-plugin-wasm.rs` 插件使得在 Parcel 中运行 Rust 和 WebAssembly 变得容易。现在，让我们看看如何安装和使用 `wasm-pack` 来打包和发布 WebAssembly 模块。

# 介绍 wasm-pack

为了与 JavaScript 兼容，基于 Rust 的 WebAssembly 应用程序应该能够完全与 JavaScript 世界互操作。没有这一点，开发者将难以在 JavaScript 中启动他们的 WebAssembly 项目。

节点模块完全改变了 JavaScript 世界的视角。它们使得在浏览器和 Node 环境之间开发和共享模块变得更加容易。世界各地的开发者可以在任何地方和任何时候使用这些库。

`wasm-pack` 工具旨在成为构建和与 Rust 生成的 WebAssembly 交互的一站式商店，这些 WebAssembly 可以在浏览器或 Node.js 中与 JavaScript 交互。`- wasm-pack` 网站。[`github.com/rustwasm/wasm-pack`](https://github.com/rustwasm/wasm-pack)

## 你为什么需要 wasm-pack？

`wasm-pack` 使得构建和打包基于 Rust 和 WebAssembly 的项目变得简单。一旦打包，模块就准备好通过 npm 注册表与世界共享——就像那里数百万（甚至数十亿）的 JavaScript 库一样。

## 如何使用 wasm-pack

`wasm-pack` 可作为 Cargo 库使用。如果您正在跟随这本书学习，那么您可能已经安装了 Cargo。要安装 `wasm-pack`，请运行以下命令：

```rs
$ cargo install wasm-pack
```

上述命令将下载、编译并安装 `wasm-pack` 库。一旦安装完成，`wasm-pack` 命令将可用。

要检查 `wasm-pack` 是否正确安装，请运行以下命令：

```rs
$ wasm-pack --version
wasm-pack 0.6.0
```

一旦您安装了 `wasm-pack`，让我们看看如何使用 `wasm-pack` 来构建和打包 Rust 和 WebAssembly 项目：

1.  我们将首先使用 Cargo 生成一个新的项目。要生成项目，请使用以下命令：

    ```rs
    $ cargo new --lib wasm_pack_world
      Created library `wasm_pack_world` package
    ```

一旦项目创建完成，请使用您最喜欢的编辑器打开它。

1.  要为项目定义 `wasm-bindgen` 依赖项，请打开 `cargo.toml` 文件：

    ```rs
    [package]
    name = "wasm_pack_world"
    version = "0.1.0"
    authors = ["Sendil Kumar"]
    edition = "2018"

    [lib]
    crate-type = ["cdylib"]

    [dependencies]
    wasm-bindgen = "0.2.38"
    ```

首先，删除 `[dependencies]` 标签，并用 `wasm-bindgen` 库替换它。我们告诉编译器，正在生成的库将是动态的，并且它依赖于 `wasm-bindgen` 库。

1.  然后，我们打开 `src/lib.rs` 文件，并将内容替换为以下内容：

    ```rs
    use wasm_bindgen::prelude::*;

    #[wasm_bindgen]
    pub fn get_me_universe_answer() -> i32 {
        42
    }
    ```

再次强调，这是一个简单的函数，它返回一个数字（这是万能的答案）。

以前，我们使用 `rustc` 或 Cargo 构建 Rust 和 WebAssembly 应用程序。这会产生一个 WebAssembly 二进制文件。但二进制文件本身并没有用；它需要一个绑定文件。使用 `wasm-bindgen`，我们将生成绑定文件以及 WebAssembly 二进制文件。

这两个步骤是强制性的，但它们很平凡。我们可以用 `wasm-pack` 替换它们。

1.  要使用 `wasm-pack` 构建 WebAssembly 应用程序，请运行以下命令：

    ```rs
    $ wasm-pack build
    ```

当我们运行 `wasm-pack build` 时，会发生以下情况：

1.  `wasm-pack` 首先检查 Rust 编译器是否已安装。如果已安装，那么它会检查 Rust 编译器版本是否大于 1.30。

1.  `wasm-pack` 检查 crate 配置以及库是否指示我们正在生成动态库。

1.  `wasm-pack` 验证是否有任何 `wasm-target` 可用于构建。如果 `wasm32-unknown-unknown` 目标不可用，`wasm-pack` 将下载并添加该目标。

1.  一旦环境准备就绪，`wasm-pack` 然后开始编译模块并构建 WebAssembly 模块和绑定 JavaScript 文件。

注意，`wasm-pack` 命令也会生成 `package.json` 文件。`package.json` 文件看起来类似于以下内容：

```rs
{
"name": "wasm_pack_world",
"collaborators": [
"Sendil Kumar"
],
"version": "0.1.0",
"files": [
     "wasm_pack_world_bg.wasm",
     "wasm_pack_world.js",
     "wasm_pack_world.d.ts"
],
"module": "wasm_pack_world.js",
"types": "wasm_pack_world.d.ts",
"sideEffects": "false"
}
```

1.  最后，如果有的话，它会复制 Readme 和 LICENSE 文件，以确保 Rust 和 WebAssembly 版本之间有共享文档。

`wasm-pack` 还会检查 `wasm-bindgen-cli` 的存在。如果不存在，将使用 Cargo 进行安装。

1.  当构建成功完成后，它将创建一个 `pkg` 目录。在 `pkg` 目录内，它将通过 `wasm-bindgen` 将输出重定向：

    ```rs
    pkg
    ├── package.json
    ├── wasm_pack_world.d.ts
    ├── wasm_pack_world.js
    ├── wasm_pack_world_bg.d.ts
    └── wasm_pack_world_bg.wasm
    ```

现在，这个 `pkg` 文件夹可以像任何其他 JavaScript 模块一样打包和共享。我们将在未来的菜谱中看到如何实现这一点。

`wasm-pack` 是一个打包和发布 WebAssembly 模块的强大工具。现在，让我们来看看如何使用它。

# 使用 wasm-pack 打包和发布

对于库开发者来说，最令人惊叹（当然，也是最重要的）的事情就是打包和发布工件。这就是我们日夜精心制作应用程序，将其发布到世界，接收反馈（无论是负面还是正面），然后根据这些反馈增强应用程序的原因。

任何项目的关键点是其首次发布，这决定了项目的命运。即使它只是一个 MVP，它也会让世界一瞥我们正在做什么，并让我们一瞥未来我们必须做什么。

`wasm-pack` 帮助我们将基于 Rust 和 WebAssembly 的项目构建、打包和发布到 npm 注册表。我们已经看到了 `wasm-pack` 如何通过底层的 `wasm-bindgen` 使将 Rust 构建到 WebAssembly 二进制文件以及绑定 JavaScript 文件变得更加简单。让我们进一步探索我们可以使用其 `pack` 和 `publish` 标志做什么。

`wasm-pack` 提供了一个 `pack` 标志来打包使用 `wasm-pack` 构建命令生成的工件。尽管使用 `wasm-pack` 构建二进制文件不是必需的，但它生成了我们将需要将工件打包成 Node 模块的所有样板代码。

为了使用 `wasm-pack` 打包构建的工件，我们必须运行以下命令，并参考 `pkg`（或我们生成构建工件的那个目录）：

```rs
$ wasm-pack pack pkg
```

我们也可以通过传递 `project_folder/pkg` 作为其参数来运行命令。如果没有参数，`wasm-pack pack` 命令将在它运行的当前工作目录中搜索 `pkg` 目录。

`wasm-pack pack` 命令首先确定提供的文件夹是否是 `pkg` 目录或包含一个 `pkg` 目录作为其直接子目录。如果检查通过，那么 `wasm-pack` 将调用底层的 npm pack 命令，将库打包成一个 npm 包。

为了捆绑 npm 包，我们只需要一个有效的 `package.json` 文件。该文件由 `wasm-pack` 构建命令生成。

我们可以在之前的菜谱中的 `cg-array-world` 示例内部运行 `pack` 命令，并检查会发生什么：

```rs
$ wasm-pack pack
npm notice
npm notice 📦 cg-array-world@0.1.0
npm notice === Tarball Contents ===
npm notice 313B package.json
npm notice 32.7kB cg_array_world_bg.wasm
npm notice 135B cg_array_world.d.ts
npm notice 1.6kB cg_array_world.js
npm notice 1.5kB README.md
npm notice === Tarball Details ===
npm notice name: cg-array-world
npm notice version: 0.1.0
npm notice filename: cg-array-world-0.1.0.tgz
npm notice package size: 16.0 kB
npm notice unpacked size: 36.4 kB
npm notice shasum: 243488f1f5a859b60bb34f39146b35ba720dd8ea
npm notice integrity: sha512-9SFuObzEpi254[...]/lkHq6RKSgnNw==
npm notice total files: 5
npm notice
cg-array-world-0.1.0.tgz
| 🎒 packed up your package!
```

正如你所见，`pack` 命令在 `npm pack` 命令的帮助下，创建了一个包含 `pkg` 文件夹内容的 tarball 包。

一旦我们打包了我们的应用程序，下一步显然就是发布它。为了发布生成的 tarball，`wasm-pack` 有一个 `publish` 选项。

为了发布包，我们必须运行以下命令：

```rs
$ wasm-pack publish
```

`wasm-pack publish`命令将首先检查提供的目录中是否已经存在`pkg`目录。

如果`pkg`目录不存在，那么它会询问你是否想要首先创建包：

```rs
$ wasm-pack publish
Your package hasn't been built, build it? [Y/n]
```

如果你回答`Y`来回答这个问题，那么它会要求你输入你想要生成构建工件文件夹的位置。我们可以给出任何文件夹名称或使用默认值：

```rs
$ wasm-pack publish
Your package hasn't been built, build it? yes
out_dir[default: pkg]:
```

然后，它会询问你的目标，即构建应该生成的目标。你可以在这里选择各种选项，如构建配方中所述：

```rs
$ wasm-pack publish
Your package hasn't been built, build it? yes
out_dir[default: pkg]: .
target[default: browser]:
> browser
nodejs
no-modules
```

根据提供的选项，它将在指定的文件夹中生成工件。

一旦生成工件，它们就准备好使用 npm publish 进行发布了。为了 npm publish 能够正确工作，我们需要进行认证。你可以通过使用 npm login 或`wasm-pack` login 来对 npm 进行认证。

`wasm-pack login`命令将调用底层的 npm login 命令然后创建一个会话：

```rs
$ wasm-pack login
Username: sendilkumarn
Password: *************
login succeeded.
```

`wasm-pack publish`命令还支持两个选项，即以下内容：

+   `-a`或`--access`用于确定要部署的包的访问级别。

这接受`public`或`restricted`：

+   `public` – 使包公开

+   `restricted` – 使包内部化

+   `-t`或`--target`用于支持在构建中产生的各种目标。

因此，`wasm-pack`使得打包和发布 WebAssembly 二进制文件变得简单。

# 摘要

在本章中，我们看到了如何使用 webpack 和 Parcel 等打包器运行 WebAssembly 项目。Parcel 和 webpack 使得 JavaScript 开发者能够轻松运行和开发 Rust 和 WebAssembly 项目。然后，我们安装了`wasm-pack`并使用它来运行项目。最后，我们使用`wasm-pack`将 WebAssembly 模块打包并发布到 npm。

在下一章中，我们将探讨如何使用`wasm-bindgen`在 Rust 和 WebAssembly 之间共享复杂对象。
