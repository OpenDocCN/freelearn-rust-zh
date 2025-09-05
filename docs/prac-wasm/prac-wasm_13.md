# *第十章*：优化 Rust 和 WebAssembly

到目前为止，我们已经看到了 Rust 如何使创建和运行 WebAssembly 模块变得容易，以及 Rust 社区提供的各种工具。在本章中，我们将涵盖以下部分：

+   最小化 WebAssembly 模块

+   分析 WebAssembly 模块中的内存模型

+   使用 Twiggy 分析 WebAssembly 模块

# 技术要求

您可以在 GitHub 上找到本章中存在的代码文件，地址为[`github.com/PacktPublishing/Practical-WebAssembly`](https://github.com/PacktPublishing/Practical-WebAssembly)。

# 最小化 WebAssembly 模块

`wasm-bindgen`是一个完整的套件，为 WebAssembly 模块生成绑定 JavaScript 文件（包括 polyfills）。在前几章中，我们看到了`wasm-bindgen`如何提供库，并使得在 JavaScript 和 WebAssembly 之间传递复杂对象变得容易。但在 WebAssembly 的世界里，优化生成的二进制文件的大小和性能非常重要。

让我们看看我们如何进一步优化 WebAssembly 模块：

1.  使用所有必要的工具链创建 WebAssembly 应用程序：

    ```rs
    $ npm init rust-webpack wasm-rust
    🦀 Rust + 🕸 WebAssembly + Webpack = ❤
    ```

此前命令创建了一个新的基于 Rust 和 JavaScript 的应用程序，webpack 作为打包器。

1.  进入生成的`wasm-rust`目录：

    ```rs
    cd wasm-rust
    ```

Rust 源文件位于`src`目录中，JavaScript 文件位于`js`目录中。我们已经为运行应用程序配置了 webpack。

1.  从`src/lib.rs`中删除所有代码，并用以下内容替换：

    ```rs
    use wasm_bindgen::prelude::*;

    #[cfg(feature = "wee_alloc")]
    #[global_allocator]
    static ALLOC: wee_alloc::WeeAlloc =
      wee_alloc::WeeAlloc::INIT;
    #[wasm_bindgen]
    pub fn is_palindrome(input: &str) -> bool {
        let s = input.to_string().to_lowercase();
        s == s.chars().rev().collect::<String>()
    }
    ```

我们导入`wasm_bindgen`并启用`wee_alloc`，它进行更小的内存分配。

我们继续定义`is_palindrome`函数，它接受`&str`作为输入并返回`bool`。在这个函数内部，我们检查给定的字符串是否是回文。

注意

在[`users.rust-lang.org/t/whats-the-difference-between-string-and-str/10177/9`](https://users.rust-lang.org/t/whats-the-difference-between-string-and-str/10177/9)了解更多关于`&str`和`String`之间的区别。

1.  现在，从`js/index.js`中删除所有行，并用以下内容替换：

    ```rs
    const rust = import('../pkg/index.js');
    rust.then(module => {
        console.log(module.is_palindrome('tattarrattat'));
      // returns true
    });
    ```

    注意

    我们在这里从`../pkg/index.js`导入。`wasm-pack`命令将在`pkg`文件夹内生成`binding`文件和`wasm`文件。

1.  接下来，使用以下命令构建应用程序：

    ```rs
    $ npm run build
    // comments, logs are elided
       Asset     Size   Chunks    Chunk Names
       0.js   9.84 KiB       0  [emitted]
       0fd5cbc32a547ac3295c.module.wasm    115 KiB       0
         [emitted] [immutable]
       index.html  179 bytes          [emitted]
       index.js    901 KiB   index  [emitted]
         index
    ```

您可以使用`npm run start`命令运行应用程序。此命令打开浏览器并加载应用程序。

1.  现在，打开开发者工具并检查控制台中的日志。

Rust 编译器生成的 WebAssembly 模块并未完全优化。我们可以进一步优化 WebAssembly 模块。在 JavaScript 的世界里，每个字节都很重要。

1.  现在，打开`Cargo.toml`并添加以下内容：

    ```rs
    [profile.dev]
    opt-level = 'z'
    lto = true
    debug = false
    ```

此外，完全删除`[profile.release]`部分。`[profile.dev]`部分指导编译器如何对 dev 构建中生成的代码进行性能分析。`[profile.release]`部分仅用于发布构建。

我们指示编译器使用`opt-level = z`来生成代码。`opt-level`设置类似于 LLVM 编译器的`-O1/2/3/...`。

`opt-level`设置的合法选项如下：

+   `0` – 没有优化；同时开启`cfg(debug_assertions)`

+   `1` – 基本优化

+   `2` – 一些优化

+   `3` – 所有优化

+   `s` – 优化二进制大小

+   `z` – 优化二进制大小，但关闭循环向量化

LLVM 支持链接时优化，通过使用整个程序分析来更好地优化代码。但链接时优化是以更长的链接时间为代价的。我们可以通过使用 lto 选项来启用 LLVM 的链接时优化。

LTO 支持以下选项：

+   `false` – 执行“thin local LTO”。这意味着链接时优化仅在本地 crate 上完成。注意：当 Codegen 单元数量为 1 或`opt-level`为 0 时，将不会进行链接时优化。

+   `true`或“fat” – 执行“fat”LTO。这意味着链接时优化是在依赖图中的所有 crates 上完成的。

+   `thin` – 执行`thin`LTO。这是“fat”的一个更快版本，优化速度更快。

+   `off` – 禁用 LTO（链接时优化）。

1.  接下来，运行`npm run build`：

    ```rs
    $ npm run build
    // comments, logs are elided
       Asset   Size   Chunks   Chunk Names
       0.js  9.84 KiB     0  [emitted]
       b5e867dd3d25627d7122.module.wasm  50.8 KiB       0
         [emitted] [immutable]
       index.js   901 KiB   index  [emitted]
         index
    ```

生成的 WebAssembly 二进制文件大小为 50.8 KB。生成的二进制文件大小减少了约 44%。这对我们来说是一个巨大的胜利。我们可以使用 Binaryen 的`wasm-opt`工具进一步优化二进制文件：

```rs
$ /path/to/build/directory/of/binaryen/wasm-opt -Oz
  b5e867dd3d25627d7122.module.wasm -o opt-gen.wasm
$ l
-rw-r--r--    1 sendilkumar  staff    45K May  8 17:43
 opt-gen.wasm
```

它又减少了 5 KB。我们已经使用了`-Oz`传递，但我们还可以传递其他传递来进一步优化生成的二进制文件。

我们已经看到了如何使用 Rust 最小化 WebAssembly 模块。接下来，我们将分析 WebAssembly 模块中的内存模型。

# 分析 WebAssembly 模块中的内存模型

在 JavaScript 引擎内部，WebAssembly 和 JavaScript 在不同的位置运行。跨越 JavaScript 和 WebAssembly 之间的边界总会有一些成本。浏览器厂商实现了酷炫的技巧和解决方案来减少这种成本，但当你应用程序跨越这个边界时，这个边界跨越很快就会成为你应用程序的主要性能瓶颈。设计 WebAssembly 应用程序时，减少边界跨越非常重要。但一旦应用程序增长，管理这个边界跨越就变得困难。为了防止边界跨越，WebAssembly 模块附带内存模块。

WebAssembly 模块中的内存部分是一个线性内存的向量。

线性内存模型是一种内存寻址技术，其中内存组织在一个单一的连续地址空间中。它也被称为平面内存模型。

线性内存模型使得理解、编程和表示内存变得更加容易。但它也有巨大的缺点，例如在内存中重新排列元素的高执行时间和浪费大量内存区域。

在这里，内存代表一个包含未解释数据的原始字节的向量。WebAssembly 使用可调整大小的数组缓冲区来存储内存的原始字节。需要注意的是，创建的这种内存可以从 JavaScript 和 WebAssembly 中访问和修改。

## 使用 Rust 在 JavaScript 和 WebAssembly 之间共享内存

我们已经看到了如何在 JavaScript 和 WebAssembly 之间共享内存。让我们在这个例子中使用 Rust 来共享内存：

1.  使用 Cargo 创建一个新的 Rust 项目：

    ```rs
    $ cargo new --lib memory_world
    ```

1.  在您喜欢的编辑器中打开项目，并将 `src/lib.rs` 替换为以下内容：

    ```rs
    #![no_std]

    use core::panic::PanicInfo;
    use core::slice::from_raw_parts_mut;

    #[no_mangle]
    fn memory_to_js() {
        let obj: &mut [u8];

        unsafe {
          obj = from_raw_parts_mut::<u8>(0 as *mut u8, 1);
        }

        obj[0] = 13;
    }

    #[panic_handler]
    fn panic(_info: &PanicInfo) -> !{
        loop{}
    } 
    ```

Rust 文件以 `#![no_std]` 开始。这指示编译器在生成 WebAssembly 模块时不要包含 Rust 标准库。这将大大减少二进制文件的大小。接下来，我们定义一个名为 `memory_to_js` 的函数。这个函数在内存中创建一个 `obj` 并与 JavaScript 共享。在函数定义中，我们创建一个名为 `obj` 的 `u32` 切片。接下来，我们将一些原始内存分配给 `obj`。在这里，我们处理原始内存。因此，我们将代码包裹在一个 `unsafe` 块中。内存对象是全局的，并且可以被 JavaScript 和 WebAssembly 同时修改。因此，我们使用 `from_raw_parts_mut` 来实例化对象。最后，我们将共享数组缓冲区的第一个元素赋值为一个值。

1.  创建一个 `index.html` 文件，并添加以下内容：

    ```rs
    <script>
        ( async() => {
             const bytes = await fetch("target/wasm32-
             unknown-unknown/debug/memory_world.wasm");
             const response = await bytes.arrayBuffer();
             const result = await
               WebAssembly.instantiate(response, {});
             result.exports.memory_to_js();
             const memObj = new
               UInt8Array(result.exports.memory.buffer, 0)
               .slice(0, 1);
             console.log(memObj[0]); // 13
        })();
    </script>
    ```

我们创建一个匿名异步 JavaScript 函数，该函数在脚本加载时立即被调用。在匿名函数中，我们获取 WebAssembly 二进制文件。接下来，我们创建 `ArrayBuffer` 并将模块实例化到 `result` 对象中。然后，我们在 WebAssembly 模块中调用 `memory_to_js` 方法（注意 `exports` 关键字，因为该函数是从 WebAssembly 模块导出的）。这实例化了内存并将共享数组缓冲区的第一个元素赋值为 `13`：

```rs
const memObj = new
 UInt8Array(result.exports.memory.buffer, 0)
   .slice(0, 1);
console.log(memObj[0]); // 13
```

接下来，我们使用 `result.export.memory.buffer` 调用从 WebAssembly 导出的内存对象，并使用一个新的 `UInt8Array()` 将其转换为 `UInt8Array`。然后，我们使用 `slice(0,1)` 提取第一个元素。这样，我们可以在 JavaScript 和 WebAssembly 之间传递和检索值，而无需任何开销。内存通过 `load` 和 `store` 二进制指令访问。`load` 操作将数据从主内存复制到寄存器。`store` 操作将数据从主内存复制。这些二进制指令通过偏移量和对齐方式访问。对齐是以 2 为底的对数表示。内存地址应该是 4 的倍数。这被称为对齐限制。这种对齐限制使硬件运行得更快。

注意

需要注意的是，WebAssembly 目前只提供 32 位地址范围。在未来，WebAssembly 可能会提供 64 位地址范围。

我们已经看到如何通过在 Rust 中创建内存来在 JavaScript 和 WebAssembly 之间共享内存。接下来，我们将在 JavaScript 端创建内存对象并在 Rust 应用程序中使用它。

## 在 JavaScript 中创建内存对象以在 Rust 应用程序中使用

与 JavaScript 不同，Rust 不是动态类型。在 JavaScript 中创建的内存无法告诉 WebAssembly（或 Rust 代码）要分配什么以及何时释放它们。我们需要明确告知 WebAssembly 如何分配内存，最重要的是，何时以及如何释放它们（以避免任何泄漏）。

我们使用 `WebAssembly.memory()` 构造函数在 JavaScript 中创建内存。内存构造函数接收一个对象来设置默认值。该对象具有以下选项：

+   `initial` – 内存初始大小

+   `maximum` – 内存的最大大小（可选）

+   `shared` – 表示是否使用共享内存

`initial` 和 `maximum` 的单位是 WebAssembly 页面，其中页面指的是 64 KB。

我们按如下方式更改 HTML 文件：

```rs
<script>
     ( async() => {
        const memory = new WebAssembly.Memory({initial: 10,
          maximum:100}); // -> 1
        const bytes = await fetch("target/wasm32-unknown-
          unknown/debug/memory_world.wasm");
        const response = await bytes.arrayBuffer();
        const instance = await
          WebAssembly.instantiate(response, 
          { js: { mem: memory } }); // ->2
        const s = new Set([1, 2, 3]);
        let jsArr = Uint8Array.from(s); // -> 3
        const len = jsArr.length;
        let wasmArrPtr = instance.exports.malloc(length);
          // -> 4
        let wasmArr = new
          Uint8Array(instance.exports.memory.buffer,
          wasmArrPtr, len); // -> 5
        wasmArr.set(jsArr); // -> 6
        const sum = instance.exports.accumulate
          (wasmArrPtr, len); // -> 7
        console.log(sum);
    })();
</script>
```

在 `// -> 1` 中，我们使用 `WebAssembly.Memory()` 构造函数初始化内存。我们传递了内存的初始和最大大小，即 640 KB 和 6.4 MB。

在 `// -> 2` 中，我们正在实例化 WebAssembly 模块以及内存对象。

在 `// -> 3` 中，我们随后创建具有值 `1`、`2` 和 `3` 的 `typedArray` (`UInt8Array`)。

在 `// -> 4` 中，我们看到，由于 WebAssembly 模块对从内存中创建的对象没有任何线索，因此需要分配内存。我们必须在 WebAssembly 中手动编写内存的分配和释放。在这个步骤中，我们发送数组的长度并分配该内存。这给了我们内存位置的指针。

在 `// -> 5` 中，我们使用缓冲区（总可用内存）、内存偏移量（`wasmAttrPtr`）和内存长度创建一个新的 `typedArray`。

在 `// -> 6` 中，我们将本地创建的 `typedArray`（在第 *3* 步中创建）设置为在第 *5* 步中创建的 `typedArray`。

在 `//-> 7` 中，最后，我们将内存指针和长度发送到 WebAssembly 模块，通过使用内存指针和长度从内存中获取值。

在 Rust 端，将 `src/lib.rs` 的内容替换为以下内容：

```rs
use std::alloc::{alloc, dealloc,  Layout};
use std::mem;

#[no_mangle]
fn accumulate(data: *mut u8, len: usize) -> i32 {
    let y = unsafe { std::slice::from_raw_parts(data as
      *const u8, len) };
    let mut sum = 0;
    for i in 0..len {
        sum = sum + y[i];
    }
    sum as i32
}

#[no_mangle]
fn malloc(size: usize) -> *mut u8 {
    let align = std::mem::align_of::<usize>();
    if let Ok(layout) = Layout::from_size_align(size,
      align) {
        unsafe {
            if layout.size() > 0 {
                let ptr = alloc(layout);
                if !ptr.is_null() {
                    return ptr
                }
            } else {
                return align as *mut u8
            }
        }
    }
    std::process::abort
}
```

我们从 `std::alloc` 和 `std::mem` 中导入了 `alloc`、`dealloc` 和 `Layout` 来操作原始内存。第一个函数 `accumulate` 接收 `data`，这是数据开始的指针，以及 `len`，要读取的内存长度。首先，我们使用 `std::slice::from_raw_parts` 通过传递指针 `data` 和长度 `len` 从原始内存中创建一个切片。请注意，这是一个不安全操作。接下来，我们遍历数组中的项并将所有元素相加。最后，我们将值作为 `i32` 返回。

`malloc`函数用于自定义分配内存，因为 WebAssembly 模块对发送的信息类型以及如何读取/理解它一无所知。`malloc`帮助我们按需分配内存，而无需任何恐慌。

使用`python -m http.server`运行前面的代码，并在浏览器中加载网页以在开发者工具中查看结果。

# 使用 Twiggy 分析 WebAssembly 模块

Rust 到 WebAssembly 的二进制文件更有可能创建一个臃肿的二进制文件。在创建 WebAssembly 二进制文件时，应采取适当的注意。在生成二进制文件时，应考虑优化级别、编译时间以及各种其他因素之间的权衡。但大多数先前的工作默认由编译器完成。无论是 Emscripten 还是`rustc`编译器，都会确保消除死代码，并提供各种优化级别的选项（从`-O0`到`z`）。我们可以选择适合我们的一个。

Twiggy 是一个代码大小分析器。它使用调用图来确定函数的来源，并提供关于函数的元信息。元信息包括每个函数的二进制大小及其成本。Twiggy 提供了二进制内容的概述。有了这些信息，我们可以进一步优化二进制文件。让我们安装并使用 Twiggy 来优化二进制文件：

1.  通过运行以下命令安装 Twiggy：

    ```rs
    $ cargo install twiggy
    ```

1.  安装完成后，`twiggy`命令将在命令行中可用，我们可以通过以下命令来检查：

    ```rs
    $ twiggy
    twiggy-opt 0.6.0
    ...
    Use `twiggy` to make your binaries slim!

    USAGE:
        twiggy <SUBCOMMAND>

    FLAGS:
        -h, --help Prints help information
        -V, --version Prints version information

    SUBCOMMANDS:
        diff         Diff the old and new versions of a
                     binary to see what sizes changed.
        dominators   Compute and display the dominator
                     tree for a binary's call graph.
        garbage      Find and display code and data that
                     is not transitively referenced by any
                     exports or public functions.
        help         Prints this message or the help of
                     the given subcommand(s)
        monos        List the generic function
                     monomorphizations that are
                     contributing to code bloat.
    paths        Find and display the call paths 
                     to a function in the given binary's
                     call graph.
        top          List the top code size offenders in a
                     binary.
    ```

1.  创建一个文件夹来测试驱动 Twiggy：

    ```rs
    $ mkdir twiggy-world
    ```

1.  创建一个名为`add.wat`的文件，并添加以下内容：

    ```rs
    $ touch add.wat
    (module
        (func $add (param $lhs i32) (param $rhs i32)
          (result i32)
            get_local $lhs
            get_local $rhs
            i32.add)
        (export "add" (func $add))
    )
    ```

1.  一旦定义了 WebAssembly 文本格式，就可以使用`wabt`将其编译为 WebAssembly 模块：

    ```rs
    $ /path/to/build/directory/of/wabt/wat2wasm add.wat
    ```

1.  前面的命令生成一个`add.wasm`文件。要获取二进制中的调用路径，请使用`paths`选项运行 Twiggy：

    ```rs
    $ twiggy paths add.wasm
    Shallow Bytes │ Shallow % │ Retaining Paths
    ───────────────┼───────────┼───────────────────────────
    9 ┊21.95% ┊ code[0]
    ┊┊⬑ export "add"
    8 ┊19.51% ┊ wasm magic bytes
    6 ┊14.63% ┊ type[0]: (i32, i32) -> i32
    ┊┊⬑ code[0]
    ┊┊⬑ export "add"
    6 ┊14.63% ┊ export "add"
    6 ┊14.63% ┊ code section headers
    3 ┊7.32% ┊ type section headers
    3 ┊7.32% ┊ export section headers
    ```

`twiggy paths`命令显示函数的调用路径、它们在二进制文件中占用的字节数以及它们的百分比。实际添加的代码是 9 字节，它占整个二进制文件大小的 21.95%。

让我们探索 Twiggy 的各种子命令：

+   `top`

+   `monos`

+   `garbage`

## top

`twiggy top`命令将列出每个代码块的大小。它按降序列出函数的大小、在最终二进制文件中的大小百分比以及块部分：

```rs
$ twiggy top add.wasm
Shallow Bytes │ Shallow % │ Item
───────────────┼───────────┼───────────────────────────
9 ┊21.95% ┊ code[0]
8 ┊19.51% ┊ wasm magic bytes
6 ┊14.63% ┊ type[0]: (i32, i32) -> i32
6 ┊14.63% ┊ export "add"
6 ┊14.63% ┊ code section headers
3 ┊7.32% ┊ type section headers
3 ┊7.32% ┊ export section headers
41 ┊100.00% ┊Σ [7 Total Rows]
The usage of the twiggy top is as follows
USAGE: twiggy top <input> -n <max_items> -o
 <output_destination> --format <output_format> --mode
 <parse_mode>
```

使用`-n`后跟要显示的条目数来列出前 n 个详细信息：

```rs
$ twiggy top add.wasm -n 3
Shallow Bytes │ Shallow % │ Item
───────────────┼───────────┼───────────────────────────
9 ┊21.95% ┊ code[0]
8 ┊19.51% ┊ wasm magic bytes
             6 ┊14.63% ┊ type[0]: (i32, i32) -> i32
18 ┊43.90% ┊ ... and 4 more.
41 ┊100.00% ┊Σ [7 Total Rows]
```

类似地，我们可以使用`--format`标志将输出格式化为 JSON 格式：

```rs
$ twiggy top add.wasm -n 3 --format json
[{"name":"code[0]","shallow_size":9,"shallow_size_percent":
21.951219512195124},{"name":"wasm magic
bytes","shallow_size":8,"shallow_size_percent":19.512195121
95122},{"name":"type[0]: (i32, i32) ->
i32","shallow_size":6,"shallow_size_percent":14.63414634146
3413}]
```

当你想追踪最大的代码块并单独优化它们时，`top`命令非常有用。

## monos

在 JavaScript 世界中，单态化可以提高性能。但它也会增加代码大小（例如，在泛型中）。由于我们必须为每种类型动态创建泛型函数的实现，因此在使用泛型和单态代码时必须非常小心。

Twiggy 有一个名为`monos`的子命令，它将列出由于单态化导致的代码膨胀：

```rs
$ twiggy monos pkg/index_bg.wasm
Apprx. Bloat Bytes │ Apprx. Bloat % │ Bytes │ %      │ Monomorphizations
────────────────────┼────────────────┼───────┼────────┼───────────────────────────────────────────────────────────────────
                  4 ┊          0.01% ┊    32 ┊  0.06% ┊ core::ptr::drop_in_place
                    ┊                ┊    28 ┊  0.05% ┊     core::ptr::drop_in_place::h9684ba572bb4c2f9
                    ┊                ┊     4 ┊  0.01% ┊     core::ptr::drop_in_place::h00c08aab80423b88
                  0 ┊          0.00% ┊  5437 ┊ 10.44% ┊ dlmalloc::dlmalloc::Dlmalloc::malloc
                    ┊                ┊  5437 ┊ 10.44% ┊     dlmalloc::dlmalloc::Dlmalloc::malloc::hb0329e71e24f7e2f
                  0 ┊          0.00% ┊  1810 ┊  3.48% ┊ <char as core::fmt::Debug>::fmt
                    ┊                ┊  1810 ┊  3.48% ┊     <char as core::fmt::Debug>::fmt::h5472f29c33f4c4c9
                  0 ┊          0.00% ┊  1126 ┊  2.16% ┊ dlmalloc::dlmalloc::Dlmalloc::free
                    ┊                ┊  1126 ┊  2.16% ┊     dlmalloc::dlmalloc::Dlmalloc::free::h7ab57ecacfa2b1c3
                  0 ┊          0.00% ┊  1123 ┊  2.16% ┊ core::str::slice_error_fail
                    ┊                ┊  1123 ┊  2.16% ┊     core::str::slice_error_fail::h26278b2259fb6582
                  0 ┊          0.00% ┊   921 ┊  1.77% ┊ core::fmt::Formatter::pad
                    ┊                ┊   921 ┊  1.77% ┊     core::fmt::Formatter::pad::hb011277a1901f9f7
                  0 ┊          0.00% ┊   833 ┊  1.60% ┊ dlmalloc::dlmalloc::Dlmalloc::dispose_chunk
                    ┊                ┊   833 ┊  1.60% ┊     dlmalloc::dlmalloc::Dlmalloc::dispose_chunk::he00c681454a3c3b7
                  0 ┊          0.00% ┊   787 ┊  1.51% ┊ core::fmt::write
                    ┊                ┊   787 ┊  1.51% ┊     core::fmt::write::hb395f946a5ce2cab
                  0 ┊          0.00% ┊   754 ┊  1.45% ┊ core::fmt::Formatter::pad_integral
                    ┊                ┊   754 ┊  1.45% ┊     core::fmt::Formatter::pad_integral::h05ee6133195a52bc
                  0 ┊          0.00% ┊   459 ┊  0.88% ┊ alloc::string::String::push
                    ┊                ┊   459 ┊  0.88% ┊     alloc::string::String::push::he03a5b89b77597a1
                  0 ┊          0.00% ┊  4276 ┊  8.21% ┊ ... and 64 more.
                  4 ┊          0.01% ┊ 17558 ┊ 33.73% ┊ Σ [85 Total Rows]
....
```

我们正在使用本章“最小化 WebAssembly 模块”部分中的 `index_bg.wasm` 示例。

`monos` 对于我们理解由泛型参数引起的任何膨胀现象非常有用，这些膨胀现象随后可以被更简单的泛型函数所替代。

## garbage

有时，找到不再使用但出于某些其他原因保留在最终二进制文件中的代码是很重要的。这些函数在某个地方被引用，但未在任何地方使用，编译器将不知道何时何地删除它们。

我们可以使用 Twiggy 的 `garbage` 命令列出所有非传递性引用的代码和数据：

```rs
$ twiggy garbage add.wasm
Bytes │ Size % │ Garbage Item
───────┼────────┼─────────────────────────────────────────
   109 ┊  0.21% ┊ custom section 'producers'
   109 ┊  0.21% ┊ Σ [1 Total Rows]
27818 ┊ 53.44% ┊ 1 potential false-positive data segments
```

WebAssembly 模块由一个数据部分组成。但有时，我们可能不会立即在 WebAssembly 模块中使用这些数据，而是在其他导入数据的地方使用。正如你所看到的，Twiggy 的 `garbage` 子命令显示了这些可能错误的数据值。

# 摘要

在本章中，我们看到了如何使用 Rust 优化 WebAssembly 二进制文件，如何映射 JavaScript 和 Rust 之间的内存，最后是如何使用 Twiggy 分析 WebAssembly 模块。

WebAssembly 生态系统仍处于早期阶段，它承诺提供更好的性能。WebAssembly 二进制文件解决了 JavaScript 生态系统中的几个差距，例如大小高效的紧凑二进制文件、启用流式编译和正确类型化的二进制文件。这些功能使 WebAssembly 更小、更快。另一方面，Rust 提供了生成 WebAssembly 模块的一流支持，而 `wasm-bindgen` 是目前最好的工具，它使得在 Rust 和 WebAssembly 之间传输复杂对象变得更加容易。

我希望你现在已经理解了 WebAssembly 的基础知识以及 Rust 如何使其生成 WebAssembly 模块变得更加容易。我迫不及待地想看看你将使用 Rust 和 WebAssembly 发行的内容。
