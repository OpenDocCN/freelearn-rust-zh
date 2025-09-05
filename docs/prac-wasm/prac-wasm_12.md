# *第九章*：Rust 和 WebAssembly 之间的边界跨越

到目前为止，我们只看到了在 JavaScript 和 WebAssembly 之间共享简单数字的示例。在上一节中，我们看到了 `wasm-bindgen` 如何轻松地将字符串从 Rust 传递到 JavaScript。在本章中，我们将探讨 `wasm-bindgen` 如何通过 Rust 使在 JavaScript 和 WebAssembly 之间转换更复杂的数据类型变得更加容易。本章将涵盖以下内容：

+   使用 Rust 与 JavaScript 共享类

+   使用 JavaScript 与 Rust 共享类

+   通过 WebAssembly 调用 JavaScript API

+   通过 WebAssembly 调用闭包

+   将 JavaScript 函数导入 Rust

+   通过 WebAssembly 调用 Web API

# 技术要求

您可以在 GitHub 上找到本章中包含的代码文件，链接为 [`github.com/PacktPublishing/Practical-WebAssembly/tree/main/09-rust-wasm-boundary`](https://github.com/PacktPublishing/Practical-WebAssembly/tree/main/09-rust-wasm-boundary)。

# 使用 Rust 与 JavaScript 共享类

`wasm-bindgen` 通过简单的注解使 JavaScript 与 Rust 以及反之亦然共享类变得容易。它处理所有样板代码，例如将值从 JavaScript 转换为 WebAssembly 或从 WebAssembly 转换为 JavaScript，复杂的内存操作以及容易出错的指针算术。因此，`wasm-bindgen` 使一切变得简单。

让我们看看在 JavaScript 和 WebAssembly（从 Rust）之间共享类有多简单：

1.  创建一个新的项目：

    ```rs
    $ cargo new --lib class_world
    Created library `class_world` package
    ```

1.  为项目定义 `wasm-bindgen` 依赖项。打开 `cargo.toml` 文件并添加以下内容：

    ```rs
    [package]
    name = "class_world"
    version = "0.1.0"
    authors = ["Sendil Kumar"]
    edition = "2018"
    [lib]
    crate-type = ["cdylib"]
    [dependencies]
    wasm-bindgen = "0.2.68"
    ```

1.  打开 `src/lib.rs` 文件，并将其内容替换为以下内容：

    ```rs
    use wasm_bindgen::prelude::*;
    #[wasm_bindgen]
    pub struct Point {
        x: i32,
        y: i32,
    }
    #[wasm_bindgen]
    impl Point {
        pub fn new(x: i32, y: i32) -> Point {
            Point { x: x, y: y}
        }
        pub fn get_x(&self) -> i32 {
            self.x
        }
        pub fn get_y(&self) -> i32 {
            self.y
        }

        pub fn set_x(&mut self, x: i32) {
            self.x = x;
        }

        pub fn set_y(&mut self, y:i32) {
            self.y = y;
        }

        pub fn add(&mut self, p: Point) {
            self.x = self.x + p.x;
            self.y = self.y + p.y;
         }
    }
    ```

    注意

    参数前的 `&mut` 指定参数（在这种情况下，`self`）是一个可变引用。

Rust 没有类，但我们可以通过结构体定义一个类。`Point` 结构体包含获取器、设置器和 `add` 函数。这是只有添加了 `#[wasm_bindgen]` 注解的正常 Rust 代码。

注意

函数和结构体被显式标记为 `pub`。`pub` 修饰符表示函数是公共的，并将被导出。

1.  使用 Cargo 生成 WebAssembly 模块：

    ```rs
    $ cargo build --target=wasm32-unknown-unknown
    ```

1.  使用 `wasm-bindgen` CLI 生成 WebAssembly 模块对应的绑定文件：

    ```rs
    $ wasm-bindgen target/wasm32-unknown-
      unknown/debug/class_world.wasm --out-dir .
    ```

这将生成与上一章中看到的类似的绑定文件和类型定义文件。让我们首先看看 `class_world.js` 文件。此文件将与之前章节中生成的文件类似，除了 `Point` 类。`Point` 类中包含所有获取器、设置器和 `add` 函数。这些函数使用它们的引用指针。

此外，`wasm-bindgen` 生成一个名为 `__wrap` 的静态方法，它创建 `Point` 类对象并将其指针附加到它。它添加了一个自由方法，该方法反过来调用 WebAssembly 模块内的 `__wbg_point_free` 方法。此方法负责释放 `Point` 对象或类占用的内存。

创建以下文件。我们将在其他部分也使用它们：

1.  创建`webpack.config.js`。它包含 webpack 配置：

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
        mode: 'development'
    };
    ```

1.  创建`package.json`并添加以下内容：

    ```rs
    {
        "scripts": {
            "build": "webpack",
            "serve": "webpack-dev-server"
        },
        "dependencies": {
            "html-webpack-plugin": "³.2.0",
            "webpack": "⁴.41.5",
            "webpack-cli": "³.3.10",
            "webpack-dev-server": "³.10.1"
        }
    }
    ```

1.  创建一个`index.js`文件：

    ```rs
    $ touch index.js
    ```

1.  然后，再次运行`npm install`。用以下内容修改`index.js`：

    ```rs
    import("./class_world").then(({Point}) => {
    const p1 = Point.new(10, 10);
    console.log(p1.get_x(), p1.get_y());
    const p2 = Point.new(3, 3);
    p1.add(p2);
    console.log(p1.get_x(), p1.get_y());
    });
    ```

我们在`Point`类中调用新方法，并传递`x`和`y`参数。我们打印`x`和`y`坐标。这将打印`10, 10`。然后，我们将创建另一个点（`p2`）。最后，我们调用`add`函数，并传递点`p2`。这将打印`13, 13`。

1.  获取器方法使用指针从共享数组中获取值：

    ```rs
    get_x() {
        return wasm.point_get_x(this.ptr);
    }
    ```

1.  在设置器方法中，我们传递指针和值。由于我们在这里只是传递一个数字，所以不需要额外的转换：

    ```rs
    set_x(arg0) {
        return wasm.point_set_x(this.ptr, arg0);
    }
    ```

1.  在`add`函数的情况下，我们获取参数，获取`Point`对象的指针，并将其传递给 WebAssembly 模块：

    ```rs
    add(arg0) {
        const ptr0 = arg0.ptr;
        arg0.ptr = 0;
        return wasm.point_add(this.ptr, ptr0);
    }
    ```

`wasm-bindgen`使将类转换为 WebAssembly 模块变得简单。我们已经看到了如何在 Rust 中与 JavaScript 共享一个类。现在，我们将看到如何从 JavaScript 与 Rust 共享一个类。

# 使用 JavaScript 与 Rust 共享类

使用`#[wasm_bindgen]`，JavaScript 类与 Rust 的共享也变得简单。让我们看看如何实现它。

JavaScript 类是具有一些方法的对象。Rust 是一种强类型语言。这意味着 Rust 编译器需要具体的绑定。如果没有它们，编译器会报错，因为它需要了解对象的生命周期。我们需要一种方法来确保编译器在运行时可以访问这个 API。

外部 C 函数块在这里很有帮助。extern C 使得函数名在 Rust 中可用。

在这个例子中，让我们看看如何将 JavaScript 中的类与 Rust 共享：

1.  让我们创建一个新的项目：

    ```rs
    $ cargo new --lib class_from_js_world
    Created library `class_from_js_world` package
    ```

1.  为项目定义`wasm-bindgen`依赖项。打开`cargo.toml`文件，并添加以下内容：

    ```rs
    [package]
    name = "class_from_js_world"
    version = "0.1.0"
    authors = ["Sendil Kumar"]
    edition = "2018"
    [lib]
    crate-type = ["cdylib"]
    [dependencies]
    wasm-bindgen = "0.2.68"
    ```

请将上一节中的`package.json`、`index.js`和`webpack-config.js`复制过来。然后，运行`npm install`。

1.  打开`src/lib.rs`文件，并用以下内容替换其内容：

    ```rs
    use wasm_bindgen::prelude::*;
    #[wasm_bindgen(module = "./point")] . // 1
    extern "C" {
         pub type Point; // 2

        #[wasm_bindgen(constructor)] //3
        fn new(x: i32, y: i32) -> Point;

        #[wasm_bindgen(method, getter)] //4
        fn get_x(this: &Point) -> i32;

        #[wasm_bindgen(method, getter)]
        fn get_y(this: &Point) -> i32;

        #[wasm_bindgen(method, setter)] //5
        fn set_x(this: &Point, x:i32) -> i32;

        #[wasm_bindgen(method, setter)]
        fn set_y(this: &Point, y:i32) -> i32;

        #[wasm_bindgen(method)] // 6
        fn add(this: &Point, p: Point);
    }

    #[wasm_bindgen]
    fn get_precious_point() -> Point { //7
        let p = Point::new(10, 10);
        let p1 = Point::new(3, 3);
        p.add(p1); // 8
        p
    }
    ```

在`//1`处，我们正在导入 JavaScript 模块。这将导入一个 JavaScript 文件，`point.js`。注意，这个文件应该位于与`Cargo.toml`相同的目录中。然后，我们创建一个 extern C 块来定义我们需要使用的方法。

我们首先在块中声明一个类型（`pub type Point;`）。现在，我们可以在 Rust 代码中像使用任何其他类型一样使用它。之后，我们定义一系列函数。我们首先定义构造函数。我们将构造函数作为参数传递给`#[wasm_bindgen]`注解。定义一个接受参数并返回之前声明的类型的函数。这将绑定到 Point 类型的命名空间，我们可以在 Rust 函数内部调用`Point::new(x, y);`。

然后，我们定义获取器和设置器（分别对应`//4`和`//5`）。我们甚至可以定义一个方法；这些与 JavaScript 侧上的函数类似。然后，我们有`add`函数。

注意

外部 C 块内的所有函数都是完全类型化的。

最后，我们使用 `#[wasm_bindgen]` 注解导出 `get_precious_point()` 函数。在 `get_precious_point` 函数中，我们使用 `Point::new(x, y)` 创建两个 `Point`，然后使用 `p1.add(p2)` 添加两个点。

我们可以像之前一样从 JavaScript 中调用它。我们还需要在 JavaScript 端定义一个 `Point` 类。

1.  使用以下内容创建 `Point.js`：

    ```rs
    export class Point {
        constructor(x, y) {
            this.x = x;
            this.y = y;
        }

        get_x() {
            return this.x;
        }

        get_y() {
            return this.y;
        }

        set_x(x) {
            this.x = x;
        }

        set_y(y) {
            this.y = y;
        }

        add(p1) {
            this.x += p1.x;
            this.y += p1.y;
        }
    }
    ```

1.  最后，用以下内容替换 `index.js`：

    ```rs
    import("./class_from_js_world").then(module => {
        console.log(module.get_precious_point());
    });
    ```

1.  现在，运行以下命令以启动服务器：

    ```rs
    $ npm run serve
    ```

1.  打开浏览器并运行 `http://localhost:8000`。打开开发者控制台以查看打印的对象类。

1.  让我们看看 `#[wasm_bindgen]` 宏是如何扩展代码的：

    ```rs
    $ cargo expand --target=wasm32-unknown-unknown >
      expanded.rs
    ```

这里发生了一些有趣的事情。

首先，`type` 点被转换成一个结构体。这与我们在上一个示例中所做的是类似的。但是，结构体的成员是 `JsValue` 而不是 `x` 和 `y`。这是因为 `wasm_bindgen` 不会知道这个 `Point` 类正在实例化什么。因此，它创建一个 JavaScript 对象并将其作为成员：

```rs
pub struct Point {
    obj: ::wasm_bindgen::JsValue,
}
```

它还定义了如何构造 `Point` 对象以及如何解引用它。这对于 WebAssembly 运行时知道何时分配和何时解引用它是有用的。

所定义的所有方法都被转换成 `Point` 结构体的实现。正如你所看到的，方法声明中有大量的 unsafe 代码。这是因为 Rust 代码直接与原始指针交互：

```rs
fn new(x: i32, y: i32) -> Point {
#[link(wasm_import_module =
  "__wbindgen_placeholder__")]
extern "C" {
fn __wbg_new_3ffc5ccd013f4db7(x:<i32 as
 ::wasm_bindgen::convert::IntoWasmAbi>::Abi, y:<i32 as
 ::wasm_bindgen::convert::IntoWasmAbi>::Abi) -> <Point
 as ::wasm_bindgen::convert::FromWasmAbi>::Abi;
}

unsafe {
let _ret = {
let mut __stack =
  ::wasm_bindgen::convert::GlobalStack::new();
let x = <i32 as
  ::wasm_bindgen::convert::IntoWasmAbi>::into_abi
  (x, &mut __stack);
let y = <i32 as
  ::wasm_bindgen::convert::IntoWasmAbi>::into_abi
  (y, &mut __stack);
__wbg_new_3ffc5ccd013f4db7(x, y)
};
<Point as
 ::wasm_bindgen::convert::FromWasmAbi>::from_abi(_ret,
 &mut ::wasm_bindgen::convert::GlobalStack::new())
}
}
```

在前面的代码中展示了由 `#[wasm_bindgen(constructor)]` 宏生成的代码。它首先将代码与 extern C 块链接起来。然后，参数被转换，以便在 WebAssembly 中推断。

然后，我们有 unsafe 块。首先，在全局栈中预留空间。然后，`x` 和 `y` 都被转换成 `IntoWasmAbi` 类型。

`IntoWasmAbi` 是一个 trait，用于任何可以转换为可以直接跨越 WebAssembly ABI 的类型的任何东西，例如 u32 或 f64。然后，调用 JavaScript 中的函数。然后，使用 `FromWasmAbi` 将返回值转换成 `Point` 类型。

`FromWasmAbi` 是一个 trait，用于任何可以从 WebAssembly ABI 边界恢复值的任何东西；例如，Rust u8 可以从 WebAssembly ABI u32 类型中恢复。

我们已经看到了如何使用 Rust 与 JavaScript 共享一个类。现在，我们将看到我们如何在 Rust 中调用 JavaScript API。

# 通过 WebAssembly 调用 JavaScript API

JavaScript 提供了丰富的 API 来处理对象、数组、映射、集合等。如果我们想在 Rust 中使用或定义它们，那么我们需要提供必要的绑定。手工制作这些绑定将是一个巨大的过程。但是，如果我们已经有了这些 API 的绑定呢？这是一个既适用于 Node.js 也适用于浏览器环境的通用 API，它将创建一个平台，我们可以在这个平台上完全用 Rust 编写代码，并使用 `wasm_bindgen` 来创建必要的代码。

rustwasm 团队对此的答案是 js-sys crate。

js-sys 包包含所有由 ECMAScript 标准保证在每一个 JavaScript 环境中存在的全局 API 的原始 `#[wasm_bindgen]` 绑定。 – RustWASM

它们提供了对 JavaScript 的标准内置对象的绑定，包括它们的方法和属性。

在这个例子中，让我们看看如何通过 WebAssembly 调用 JavaScript API：

1.  使用 `cargo new` 命令创建一个默认项目：

    ```rs
    $ cargo new --lib jsapi
    ```

1.  按照上一个示例，复制 `webpack.config.js`、`index.js` 和 `package.json`。然后，在您最喜欢的编辑器中打开生成的项目。

1.  修改 `Cargo.toml` 的内容：

    ```rs
    [package]
    name = "jsapi"
    version = "0.1.0"
    authors = ["Sendil Kumar"]
    edition = "2018"

    [lib]
    crate-type = ["cdylib"]

    [dependencies]
    wasm-bindgen = "0.2.68"
    js-sys = "0.3.45"
    ```

1.  现在，打开 `src/lib.rs` 并将其替换为以下内容。我们可以在 Rust 中使用以下代码片段创建一个 JavaScript 映射：

    ```rs
    use wasm_bindgen::prelude::*;

    use js_sys::Map;

    #[wasm_bindgen]
    pub fn new_js_map() -> Map {
        Map::new()
    }
    ```

在 `wasm_bindgen` 导入中，我们使用 `use js_sys::Map;` 从 `js_sys` 包中导入了 map。

1.  然后，我们定义 `new_js_map` 函数，它将返回一个新的映射：

    ```rs
    #[wasm_bindgen]
    pub fn set_get_js_map() -> JsValue {
        let map = Map::new();
        map.set(&"foo".into(), &"bar".into());
        map.get(&"foo".into())
    }
    ```

`set_get_js_map` 函数创建一个新的映射，在映射中设置一个值，然后返回设置的值。

注意，这里的返回类型是 `JsValue`。这是 Rust 中用于指定 JavaScript 值的包装器。另外，请注意，我们将字符串传递给 trait 函数 get 和 set。当在 JavaScript 中调用时，这将返回 `bar` 作为输出。

1.  现在，我们也在 Rust 代码中使用 `for_each` 运行映射，如下所示：

    ```rs
    #[wasm_bindgen]
    pub fn run_through_map() -> f64 {
        let map = Map::new();
        map.set(&1.into(), &1.into());
        map.set(&2.into(), &2.into());
        map.set(&3.into(), &3.into());
        map.set(&4.into(), &4.into());
        map.set(&5.into(), &5.into());
        let mut res: f64 = 0.0;

        map.for_each(&mut |value, _| {
            res = res + value.as_f64().unwrap();
        });

        res
    }
    ```

这创建了一个映射，然后使用值 `1`、`2`、`3`、`4` 和 `5` 加载映射。然后，它遍历创建的映射并添加值。这将产生输出 `15`（即 1 + 2 + 3 + 4 + 5）。

1.  最后，我们将 `index.js` 替换为以下内容：

    ```rs
    import("./jsapi").then(module => {
        let m = module.new_js_map();
        m.set("Hi", "Hi");
        console.log(m); // prints Map { "Hi" ->  "Hi" }
        console.log(module.set_get_js_map());  // prints
          "bar"
        console.log(module.run_through_map()); // prints
          15
    });
    ```

在浏览器上运行此代码将打印结果。请参阅控制台日志附近的注释。

让我们从生成的 JavaScript 绑定文件开始。生成的绑定 JavaScript 文件的结构几乎与上一节相同，但导出了更多函数。

堆对象在这里用作栈。所有与 WebAssembly 模块共享或引用的 JavaScript 对象都存储在这个堆中。还重要的是要注意，一旦访问了值，它就会从堆中弹出。

```rs
function takeObject(idx) {
    const ret = getObject(idx);
    dropObject(idx);
    return ret;
}
```

`takeObject` 函数用于从堆中获取对象。它首先获取给定索引处的对象。然后，它从该堆索引（即弹出）中移除对象。最后，它返回值 `ret`。

同样，我们可以在 Rust 中使用 JavaScript API。仅针对常见的 JavaScript API（包括 Node.js 和浏览器）生成绑定。

我们已经看到了如何在 Rust 中调用 JavaScript API。现在，我们将看到如何通过 WebAssembly 调用 Rust 闭包。

# 通过 WebAssembly 调用闭包

官方的 Rust 书籍将闭包定义为如下：

闭包是无名函数，你可以将其保存到变量中，或者将其作为参数传递给其他函数。- 《Rust 编程语言》（涵盖 Rust 2018）由 Steve Klabnik 和 Carol Nichols 编著 ([`doc.rust-lang.org/book/ch13-00-functional-features.html`](https://doc.rust-lang.org/book/ch13-00-functional-features.html))

MDN 将 JavaScript 中的闭包定义为如下：

闭包是函数和其声明时所处词法环境的组合。- MDN Web 文档 ([`developer.mozilla.org/en-US/docs/Web/JavaScript/Closures#closure`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures#closure))

通常，闭包是自包含的功能块，可以在代码中传递和使用。它们可以捕获并存储定义它们上下文中的变量引用。

闭包和函数相似，除了一个细微的区别。闭包会在第一次创建时捕获状态。然后，每次调用闭包时，它都会覆盖这个捕获的状态。

闭包是有状态的函数。当你创建闭包时，它会捕获状态。然后，我们可以像传递任何其他函数一样传递闭包。当闭包被调用时，它会覆盖这个捕获的状态并执行（即使闭包在捕获状态之外被调用）。这是闭包在 JavaScript 函数式编程方面使用越来越多的一个重要原因。

闭包使得数据封装、高阶函数和记忆化变得容易。（听起来像是函数式编程，对吧？ ;))

让我们看看如何从 JavaScript 到 Rust 以及从 Rust 到 JavaScript 共享闭包：

1.  创建一个新的项目：

    ```rs
    $ cargo new --lib closure_world
         Created library `closure_world` package
    ```

1.  为项目定义 `wasm-bindgen` 依赖项。让我们打开 `cargo.toml` 文件，并添加加粗的内容：

    ```rs
    [package]
    name = "closure_world"
    version = "0.1.0"
    authors = ["Sendil Kumar"]
    edition = "2018"

    [lib]
    crate-type = ["cdylib"]

    [dependencies]
    wasm-bindgen = "0.2.38"
    js-sys = "0.3.15"
    ```

我们需要 js-sys 包来将闭包从 JavaScript 复制到 Rust。请从上一节复制 `package.json`、`index.js` 和 `webpack-config.js`。然后，运行 `npm install`。

1.  然后，我们打开 `src/lib.rs` 文件，并添加来自我们的 `Point` 类示例的内容，以及一个接受 JavaScript 闭包函数作为参数的额外方法：

    ```rs
    use wasm_bindgen::prelude::*;

    #[wasm_bindgen]
    pub struct Point {
        x: i32,
        y: i32,
    }

    #[wasm_bindgen]
    impl Point {
        pub fn new(x: i32, y: i32) -> Point {
            Point { x: x, y: y}
        }

        pub fn get_x(&self) -> i32 {
            self.x
        }

        pub fn get_y(&self) -> i32 {
            self.y
        }

        pub fn set_x(&mut self, x: i32) {
            self.x = x;
        }

        pub fn set_y(&mut self, y:i32) {
            self.y = y;
        }

        pub fn add(&mut self, p: Point) {
            self.x = self.x + p.x;
            self.y = self.y + p.y;
         }

        pub fn distance(&self, js_func: js_sys::Function)
          -> JsValue {
            let this = JsValue::NULL;
            let x = JsValue::from(self.x);
            let y = JsValue::from(self.y);
            js_func.call2(&this, &x, &y).unwrap()
        }
    }
    ```

现在，我们将更改 `index.js` 以使用闭包调用 `distance` 函数：

```rs
import("./closure_world").then(({Point}) => {
     const p1 = Point.new(13, 10);
     console.log(p1.distance((x, y) => x - y));
});
```

让我们使用 `npm run serve` 启动 webpack 服务器。这将打印出 `3`。

js-sys 包提供了一种使用 apply 和 call 方法调用 JavaScript 函数的选项。这正是我们通过调用 `js_func.call2(&this, &x, &y)` 来实现的。

Rust 没有函数重载。这意味着我们必须根据传递的参数数量使用不同的方法名。因此，`js-sys` 为我们提供了 `call1`、`call2`、`call3` 等等，分别接受 `1`、`2`、`3` 等等个参数。

在 Rust 中调用 JavaScript 函数将返回 `Result<JsValue, Error>`。我们将展开结果以获取 JsValue 并返回它。`wasm-bindgen` 将创建必要的绑定，以便将值作为数字在 JavaScript 中返回。

另一方面，将闭包从 Rust 传递到 JavaScript 需要一些额外的信息和选项。

`wasm-bindgen` 在这里支持两种变体：

+   堆栈生命周期闭包

+   堆分配的闭包

让我们看看它们实际上意味着什么：

+   一旦将闭包传递给导入的 JavaScript 函数后，堆栈生命周期闭包不应再次被 JavaScript 调用。这是因为一旦函数（闭包）返回，Rust 将使闭包失效。任何未来的调用都将导致异常。换句话说，堆栈生命周期闭包是短暂的，一旦被访问，它们就会超出上下文。

+   另一方面，堆分配的闭包对于多次调用内存非常有用。在这里，其有效性与 Rust 中闭包的生命周期相关联。一旦 Rust 中的闭包被丢弃，闭包将进行解分配，垃圾回收器将进行垃圾回收。这将反过来使 JavaScript 中的闭包（函数）失效。一旦失效，任何进一步尝试访问闭包或内存的操作都将引发异常。

堆栈生命周期和堆分配的闭包都支持 `Fn` 和 `FnMut` 闭包、参数和返回值。

我们已经看到了如何调用闭包函数。现在，我们将了解如何将 JavaScript 中的函数导入 Rust。

# 将 JavaScript 函数导入 Rust

在某些地方，JavaScript 比 WebAssembly 快，因为没有边界跨越和实例化单独运行时环境的开销。JavaScript 在其自身环境中运行得更自然。

JavaScript 生态系统非常庞大。有成千上万的库是用 JavaScript 创建并经过实战检验的（当然，并非全部），这使得 JavaScript 变得容易（这里的“容易”是主观的）。

WebAssembly 解决了前端世界最重要的一个问题，即“一致”的性能问题。但它并不是 JavaScript 的完全替代品。WebAssembly 帮助 JavaScript 提供更好和更一致的性能。

JavaScript 将在大多数地方成为默认选择。提供允许两个系统无缝集成的生态系统很重要。我们已经看到了如何将 JavaScript 中的类导入 Rust。同样，我们可以使用 `wasm-bindgen` 将任何内容从 JavaScript 导入 Rust。最重要的是，我们可以在 Rust 代码中更自然地使用这些导入的 JavaScript 函数。

在这个例子中，让我们看看如何将 JavaScript 函数导入 Rust：

1.  创建一个新的项目：

    ```rs
    $ cargo new --lib import_js_world
         Created library `import_js_world` package
    ```

1.  为项目定义 `wasm-bindgen` 依赖项。让我们打开 `cargo.toml` 文件，并添加以下加粗内容：

    ```rs

    [package]
    name = "import_js_world"
    version = "0.1.0"
    authors = ["Sendil Kumar"]
    edition = "2018"

    [lib]
    crate-type = ["cdylib"]

    [dependencies]
    wasm-bindgen = "0.2.38"
    ```

1.  请将上一节中的 `package.json`、`index.js` 和 `webpack-config.js` 复制过来。然后，运行 `npm install`。接着，打开 `src/lib.rs` 文件，并将其内容替换为以下内容：

    ```rs
    use wasm_bindgen::prelude::*;

    #[wasm_bindgen(module = "./array")]
    extern "C" {
        fn topArray() -> f64;
        fn getNumber() -> i32;
        fn lowerCase(str: &str) -> String;
    }

    #[wasm_bindgen]
    pub fn sum_of_square_root() -> f64 {
        let n = getNumber();
        let mut sum = 0;

        for _ in 0..n {
            sum = sum + (topArray().sqrt() as i64);
        } 
        sum
    }

    #[wasm_bindgen]
    pub fn some_string_to_share() -> String {
        lowerCase("HEYA! I AM ALL CAPS")
    }
    ```

我们首先导入`wasm_bindgen`库。然后，我们定义 extern C 块来定义 FFI 函数（即我们从 JavaScript 中导入的函数）。在 extern C 块内部，我们定义与 Rust 编译器理解相似的函数签名。我们还使用`#[wasm_bindgen(module = "./array")]`来注释 extern C 块。这有助于`wasm-bindgen` CLI 理解函数的定义和导出位置。它将使用这些信息并创建必要的链接。

1.  `array.js` 文件与 `cargo.toml` 文件位于同一目录下。我们将如下定义 `array.js`：

    ```rs
    let someGlobalArray = [1, 4, 9, 16, 25];

    export function getNumber() {
        return someGlobalArray.length;
    }

    export function topArray() {
        return someGlobalArray.sort().pop();
    }

    export lowerCase(str) {
        return str.toLowerCase();
    }
    ```

之前提到的功能应该在 JavaScript 文件中导出。

我们然后在 Rust 中声明一个函数（`sum_of_square_root`），并将其作为生成的 WebAssembly 模块中的函数导出。我们首先从 JavaScript 中调用 `getNumber()` 方法。我们使用返回值，然后运行 `for` 循环遍历数组的长度。对于每个循环，我们调用 `topArray` 从数组中获取最小元素。然后，我们取这个数字的平方根（这在 Rust 代码中发生）。将它们加起来并返回总和（例如我们之前看到的 `15`）。

1.  我们将用以下内容替换`index.js`：

    ```rs
    import("./import_js_world").then(module => {
        console.log(module.sum_of_square_root());
        console.log(module.some_string_to_share());
    });
    ```

1.  打开生成的绑定 JavaScript 文件。你会发现，`getNumber` 和 `topArray` 函数在生成的绑定 JavaScript 文件中不可用。这主要是因为我们只是在 JavaScript 和 WebAssembly 模块之间共享数字。因此，在这种情况下，边界跨越更加自然发生。

1.  `wasm-bindgen` 还会根据内存对象字节数进行必要的移位和解析字节缓冲区。对于 `Uint32Array`，指针和内存的计算如下：

    ```rs
    const rustptr = mem[retptr / 4];
    const rustlen = mem[retptr / 4 + 1];
    ```

1.  对于 `BigInt64Array`，指针和内存的计算如下：

    ```rs
    const rustptr = mem[retptr / 8];
    const rustlen = mem[retptr / 8 + 1];
    ```

我们已经了解了如何在 Rust 中导入 JavaScript 函数。现在，我们将看看如何在 Rust 中调用 web API。

# 通过 WebAssembly 调用 Web API

网络的发展是惊人的，其增长归功于其开放标准。今天，网络提供了数百个 API，这使得网络开发者能够轻松地为音频、视频、画布、SVG、USB、电池等开发。

网络是普遍存在的。它不断地被实验和改变，以使其对开发者和公司来说既吸引人又易于使用。`web-sys`包提供了对目前网络上几乎所有 API 的访问。

`web-sys`包提供了对 Web 所有 API 的原始绑定：从 DOM 操作到 WebGL、Web Audio、计时器、fetch 等！ – web-sys crates.io ([`crates.io/crates/web-sys`](https://crates.io/crates/web-sys))

WebIDL 接口定义被转换为`wasm-bindgen`的内部**抽象语法树**（**ASTs**）。然后，这些 ASTs 被用来创建零开销的 Rust 和 JavaScript 粘合代码。

在这个绑定代码的帮助下，我们可以调用和操作网络 API。将网络 API 转换为 Rust 确保了参数和返回值的类型信息被正确且安全地处理。

在这个例子中，让我们通过 WebAssembly 调用一个网络 API：

1.  使用`cargo new`命令创建一个默认项目：

    ```rs
    $ cargo new --lib web_sys_api
        Created library `web_sys_api` package
    ```

1.  将`webpack.config.js`、`index.js`和`package.json`的内容与`jsapi`部分（在上面的部分）类似地复制过来。现在，我们将打开生成的项目到我们最喜欢的编辑器中。让我们更改`cargo.toml`的内容：

    ```rs
    [package]
    name = "web_sys_api"
    version = "0.1.0"
    authors = ["Sendil Kumar"]
    edition = "2018"

    [lib]
    crate-type = ["cdylib"]

    [dependencies]
    wasm-bindgen = "0.2.38"

    [dependencies.web-sys]
    version = "0.3.4"
    features = [
        'Document',
        'Element',
        'HtmlElement',
        'Node',
        'Window',
    ]
    ```

这里的主要区别在于，我们不仅定义了依赖及其版本，还定义了我们将在此示例中使用的功能。

我们为什么需要它？由于网络生态系统中存在大量的 API，我们不希望携带所有这些 API 的绑定。绑定文件仅用于列出的功能。

1.  让我们打开`src/lib.rs`并将文件替换为以下内容：

    ```rs
    use wasm_bindgen::prelude::*;

    #[wasm_bindgen]
    pub fn draw(percent: i32) -> Result<web_sys::Element,
      JsValue> {
        let window = web_sys::window().unwrap();
        let document = window.document().unwrap();

        let div = document.create_element("div")?;
        let ns = Some("http://www.w3.org/2000/svg");

        div.set_attribute("class", "pie")?;

        let svg = document.create_element_ns( ns, "svg")?;
        svg.set_attribute("height", "100")?;
        svg.set_attribute("width", "100")?;
        svg.set_attribute("viewBox", "0 0 32 32")?;

        let circle = document.create_element_ns(ns,
          "circle")?;
        circle.set_attribute("r", "16")?;
        circle.set_attribute("cx", "16")?;
        circle.set_attribute("cy", "16")?;
        circle.set_attribute("stroke-dasharray",
          &(percent.to_string().to_owned() +" 100"))?;

        svg.append_child(&circle)?;

        div.append_child(&svg)?;

        Ok(div)
    }
    ```

我们首先使用`web_sys::window()`获取窗口。最后的 unwrap 确保窗口可用。如果不可用，它将抛出一个错误。之后，我们从窗口对象中获取文档。然后，我们使用`document.createElement`创建一个`div`元素。然后，我们创建一个 SVG 和圆形文档元素，并将圆形添加到 SVG 元素中。最后，我们将 SVG 作为子元素添加到`div`元素中，并返回`div`元素。

API 与 Web API 非常相似，只是方法名使用的是蛇形命名法而不是驼峰命名法。

1.  我们将更改`index.js`以使用此元素作为 Web 组件：

    ```rs
    import("./web_sys_api").then(module => {
        class Pie extends HTMLElement {
            constructor() {
                super();
                let shadow = this.attachShadow({ mode:
                  'open' });
                let style =
                  document.createElement('style');

                style.textContent = `
                        svg {
                            width:100px;
                            height: 100px;
                            background: yellowgreen;
                            border-radius: 50%;
                        }

                        circle {
                            fill: yellowgreen;
                            stroke: #655;
                            stroke-width: 32;
                        }`;

               shadow.appendChild(module.draw(this.
               getAttribute
               ('value'));
               shadow.appendChild(style);
           }
       }

        customElements.define('pie-chart', Pie);

        setInterval(() => {
            let r = Math.floor(Math.random() * 100);
            document.getElementsByTagName('body')[0].
              innerHTML = `
                <pie-chart value='${r}' />`;
        }, 1000);
    });
    ```

那么，我们在这里做了什么？我们首先导入绑定文件，这将反过来初始化 WebAssembly 模块。一旦 WebAssembly 模块初始化完成，我们创建一个扩展 HTML 元素的`Pie`类。在类的构造函数中，我们调用`super`方法。然后，我们创建一个阴影 DOM。我们在阴影 DOM 中添加一个样式元素，然后定义元素的样式。

我们继续将样式元素附加到阴影元素上，然后添加从 Rust 代码导出的元素。然后我们将其注册为名为`pie-chart`的自定义元素。最后，我们将自定义元素附加到文档的主体中，以查看饼图被显示出来。

1.  现在，运行以下命令：

    ```rs
    $ npm run serve
    ```

打开浏览器查看饼图。

# 摘要

在本章中，我们看到了`wasm-bindgen`如何使 JavaScript 和 Rust 之间共享复杂对象变得容易。注释使得标记一个函数以导出/导入 JavaScript 和 WebAssembly 变得简单。我们还看到了如何使用 js-sys 和 web-sys Cargo 在 Rust 代码中轻松调用 JavaScript 和 Web API。

在下一章中，我们将看到如何在 Rust 中优化生成的 WebAssembly 模块。
