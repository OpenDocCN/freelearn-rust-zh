# *第六章*：安装和使用 Binaryen

在编译过程中，编译语言会产生它们自己的**中间表示**（**IR**）。然后编译器会优化 IR 以生成优化代码。在传递给 LLVM 之前，编译器应将此 IR 转换为 LLVM 理解的内容（LLVM IR）。LLVM 优化 LLVM IR 并生成原生代码（如 WebAssembly 二进制文件）。这些不同级别的多个 IR 生成和优化使得编译过程变慢且不太有效。Binaryen 试图消除这些多个 IR 生成，并使用自己的 IR。

**(WebAssembly) 二进制 + Emscripten = Binaryen**

Binaryen 是一个用于 WebAssembly 的编译器和工具链基础设施库，用 C++ 编写。它的目标是使编译到 WebAssembly 变得简单、快速和有效。

- Binaryen 的 GitHub 仓库 ([`github.com/WebAssembly/binaryen`](https://github.com/WebAssembly/binaryen))

Binaryen 使用自己的 IR 版本。Binaryen 的 IR 是 WebAssembly 的一个子集。因此，它使得将 Binaryen 编译到 WebAssembly 变得更快、更简单。Binaryen 的 IR 使用紧凑的数据结构，并考虑到现代 CPU 架构。也就是说，可以使用所有可用的 CPU 核心并行生成和优化 WebAssembly 二进制文件。

此外，Binaryen 的优化器有许多可以通过显著改进代码的遍历。Binaryen 的优化器使用诸如局部着色以合并局部变量、删除死代码以及在可能的情况下预计算表达式等技术。Binaryen 还提供了一种缩小 WebAssembly 二进制文件的方法。

Binaryen 使用简单。它接受 WebAssembly 二进制文件，甚至控制图来生成高度优化的 WebAssembly 二进制文件。Binaryen 还提供了 Binaryen.js，它使得从 JavaScript 使用 Binaryen 成为可能。类似于 WABT，Binaryen 包含一套不同的工具，这些工具在处理 WebAssembly 时非常有用。

这些工具链实用程序有助于解析 WebAssembly 二进制文件，然后进一步优化它，并最终输出高度优化的 WebAssembly 二进制文件（换句话说，wasm-to-wasm 优化器），当浏览器没有 WebAssembly 支持时，提供 WebAssembly 的 polyfill。

在本章中，我们将了解如何安装和使用 Binaryen 提供的各种工具。了解 Binaryen 及其提供的工具将帮助您在性能和大小方面优化 WebAssembly 二进制文件。本章将涵盖以下部分：

+   安装和使用 Binaryen

+   `wasm-as`

+   `wasm-dis`

+   `wasm-opt`

+   `wasm2js`

# 技术要求

我们将需要安装 Binaryen 和 Visual C++。您可以在 GitHub 上找到本章中提供的代码文件，网址为 [`github.com/PacktPublishing/Practical-WebAssembly`](https://github.com/PacktPublishing/Practical-WebAssembly)。

# 安装和使用 Binaryen

为了安装 Binaryen，首先从 GitHub 克隆仓库：

```rs
$ git clone https://github.com/WebAssembly/binaryen
```

在将仓库克隆后，进入以下文件夹：

```rs
$ cd binaryen
```

## Linux/macOS

通过运行带有 `binaryen` 文件夹路径的 `cmake` 命令来生成项目构建系统：

```rs
$ cmake .
```

接下来，使用 `make` 命令构建项目：

```rs
$ make .
```

这将在 `bin` 文件夹中生成所有二进制文件。

## Windows

对于 Windows，一旦克隆了存储库，我们将创建一个 `build` 目录并进入其中：

```rs
$ mkdir build
$ cd build
```

默认情况下，Windows 没有可用的 cmake 命令。安装 Visual C++ 工具以使 `cmake` 命令在系统中可用。要安装 Visual C++ 工具，请查看以下链接：[`docs.microsoft.com/en-us/cpp/build/cmake-projects-in-visual-studio?view=msvc-160&viewFallbackFrom=vs-2019`](https://docs.microsoft.com/en-us/cpp/build/cmake-projects-in-visual-studio?view=msvc-160&viewFallbackFrom=vs-2019)。然后，在 `build` 文件夹中运行以下命令：

```rs
$ "<path-to-visual-studio-root>\Common7\IDE\CommonExtensions\
  Microsoft\CMake\CMake\bin\cmake.exe" ..
```

上述命令将在 `build` 目录中生成所有必要的构建文件。然后，我们可以使用 `cmake` 生成的 `binaryen.vcxproj` 文件构建项目：

```rs
$ msbuild binaryen.vcxproj
```

生成的二进制文件包括以下内容：

```rs
$ tree -L 1
├── wasm-as
├── wasm-ctor-eval
├── wasm-dis
├── wasm-emscripten-finalize
├── wasm-metadce
├── wasm-opt
├── wasm-reduce
├── wasm-shell
└── wasm2js 
```

Binaryen 生成的各种工具如下：

+   `wasm-as` – 此工具与 WABT 中的 `wat2wasm` 类似。该工具将 WebAssembly 文本格式 (`.wast`) 转换为 WebAssembly 二进制格式 (`.wasm`)。

+   `wasm-ctor-eval` – 此工具在预编译时执行 C++ 全局构造函数，并使其就绪。这种优化加快了 WebAssembly 的执行速度。

+   `wasm-dis` – 此工具与 wabt 中的 `wasm2wat` 类似。也就是说，它将 WebAssembly 二进制格式 (`.wasm`) 转换为 WebAssembly 文本格式 (`.wat`)。

+   `wasm-emscripten-finalize` – 此工具对给定的 `.wasm` 文件执行 Emscripten 特定的转换。

+   `wasm-metadce` – 此工具从提供的 WebAssembly 二进制文件中删除死代码。

+   `wasm-opt` – 此工具优化提供的 WebAssembly 二进制文件。

+   `wasm-reduce` – 此工具将给定的 WebAssembly 二进制文件减小到更小的二进制文件。

+   `wasm-shell` – 此工具创建一个可以加载和解释 WebAssembly 代码的壳。

+   `wasm2js` – 此工具在 polyfill 中非常有用。它将 WebAssembly 转换为 JavaScript 编译器。

+   `binaryen.js` – 一个独立的 JavaScript 库，它公开 Binaryen 方法以创建和优化 WebAssembly 模块。此 JavaScript 文件就像任何其他可以加载到浏览器中的 JavaScript 文件一样。

现在我们已经构建并生成了 Binaryen 提供的工具，让我们探索生成的工具。

# wasm-as

`wasm-as` 工具将 WAST 转换为 WASM。让我们看看步骤：

1.  让我们创建一个名为 `binaryen-playground` 的新文件夹，并进入该文件夹：

    ```rs
    $ mkdir binaryen-playground
    $ cd binaryen-playground
    ```

1.  创建一个名为 `add.wat` 的 `.wat` 文件：

    ```rs
    $ touch add.wat
    ```

1.  将以下内容添加到 `add.wat`：

    ```rs
    (module
        (func $add (param $x i32) (param $y i32) 
          (result i32)
            (i32.add
                (local.get $x)
                (local.get $y)
            )
        )
    )
    ```

1.  使用 `wasm-as` 二进制将 Web Assembly 文本格式转换为 WebAssembly 模块：

    ```rs
    $ /path/to/build/directory/of/binaryen/wasm-as add.wat
    ```

生成 `add.wasm` 文件：

```rs
00 61 73 6d 01 00 00 00 01 07 01 60 02 7f 7f 01
7f 03 02 01 00 0a 09 01 07 00 20 00 20 01 6a 0b
```

注意

生成的二进制文件大小仅为 32 字节。

`wasm-as` 首先验证给定的文件（`.wat`），然后将其转换为 `.wasm` 文件。要检查 `wasm-as` 支持的各种选项，我们可以运行以下命令：

```rs
$ /path/to/build/directory/of/binaryen/wasm-as --help
wasm-as INFILE
Assemble a .wat (WebAssembly text format) into a .wasm (WebAssembly binary
format)
Options:
  --version                        Output version information     and exit
  --help,-h                        Show this help message and     exit
  --debug,-d                       Print debug information to     stderr
....
  --output,-o                      Output file (stdout if not 
    specified)
  --validate,-v                    Control validation of the 
    output module
  --debuginfo,-g                   Emit names section and debug
    info
  --source-map,-sm                 Emit source map to the 
    specified file
  --source-map-url,-su             Use specified string as 
    source map URL
  --symbolmap,-s                   Emit a symbol map (indexes 
    => names)
```

如果我们需要将 WebAssembly 模块文件生成在不同的名称下，我们将使用 `-o` 选项和文件名。例如，`wasm-as add.wat -o customAdd.wasm` 将生成 `customAdd.wasm`。

`wasm-as` 也提供了详细的输出，清楚地解释了 WebAssembly 模块的结构。为了查看 WebAssembly 模块的结构，我们使用 `-d` 选项运行它：

```rs
$ /path/to/build/directory/of/binaryen/wasm-as add.wat -d
Loading 'add.wat'...
s-parsing...
w-parsing...
Validating...
writing...
writing binary to add.wasm
Opening 'add.wasm'
== writeHeader
...
== writeTypes
...
== writeFunctionSignatures
...
== writeFunctions
...
finishUp
Done.
```

之前的输出是关于如何生成二进制的详细描述。首先，它加载了给定的 `.wat` 文件。之后，它解析并验证文件。最后，它读取了头部、类型、函数签名和函数后创建了 `add.wasm` 并写入了头部、类型、函数签名和函数。在生成二进制文件时，我们可以通过使用适当的 `enable-*` 和 `disable-*` 选项来启用编译器包括新的和闪亮的功能，并禁用各种现有功能。此外，您可以使用 `--sm` 选项生成 `sourcemap`。

现在我们已经看到了如何将 WAST 转换为 WASM，让我们看看如何将 WASM 转换为 WAST。

# wasm-dis

`wasm-dis` 工具将 WAST 转换为 WASM。在这里，我们将使用之前示例中创建的 `add.wasm` 文件。让我们看看步骤：

1.  为了将 WebAssembly 模块转换为 WebAssembly 文本格式，使用 `wasm-dis` 二进制文件，运行以下命令：

    ```rs
    $ /path/to/build/directory/of/binaryen/wasm-dis add.wasm -o gen-add.wast
    ```

1.  我们使用 `-o` 选项和文件名（`gen-add.wast`）生成 `gen-add.wast` 文件：

    ```rs
    (module
    (type $i32_i32_=>_i32 (func (param i32 i32) 
      (result i32)))
    (func $0 (param $0 i32) (param $1 i32) (result i32)
               (i32.add  (local.get $0)  (local.get $1) )
    )
    )
    ```

1.  `wasm-dis` 首先验证给定的文件（`.wasm`），然后将其转换为 `.wat` 文件。要检查 `wasm-dis` 支持的各种选项，请运行以下命令：

    ```rs
    $ /path/to/build/directory/of/binaryen/wasm-dis --help
    wasm-dis INFILE 
    Un-assemble a .wasm (WebAssembly binary format) into a .wat (WebAssembly text
    format)
    Options:
      --version        Output version information and exit
      --help,-h        Show this help message and exit
      --debug,-d       Print debug information to stderr
      --output,-o      Output file (stdout if not specified)
      --source-map,-sm Consume source map from the specified file to add location
    ```

1.  `wasm-dis` 还提供了详细的输出，清楚地解释了 WebAssembly 模块的结构。为了查看 WebAssembly 模块的结构，我们使用 `-d` 选项运行它：

    ```rs
    $ /path/to/build/directory/of/binaryen/wasm-dis
      add.wat -o gen-add.wast -d
    parsing binary...
    reading binary from add.wasm
    Loading 'add.wasm'...
    == readHeader
    ...
    == readSignatures
    ...
    == readFunctionSignatures
    ...
    == processExpressions
    ...
    == processExpressions finished
    end function bodies
    Printing...
    Opening 'gen-add.wast'
    Done.
    ```

之前的输出是关于如何生成 `.wast` 文件的详细描述。首先，它加载了给定的 `.wasm` 文件。之后，它解析并验证文件。最后，在读取了头部、类型、函数签名和函数后创建了 `gen-add.wast`。

在生成文件时，我们可以通过使用适当的 `enable-*` 和 `disable-*` 选项来启用编译器包括新的和闪亮的功能，并分别禁用各种现有功能。

此外，我们还可以使用 `--sm <filename>` 选项输入 `sourcemap`。

现在我们已经看到了如何将 WASM 转换为 WAST，让我们看看如何进一步优化 WebAssembly 二进制文件。

# wasm-opt

`wasm-opt` 工具是一个 `wasm-to-wasm` 优化器。它将接收一个 WebAssembly 模块作为输入，并在其上运行转换过程以优化并生成优化的 WebAssembly 模块。让我们看看步骤：

1.  让我们先创建 `inline-optimizer.wast` 文件并添加以下内容：

    ```rs
    (module
        (func $parent (export "parent") (result i32)
            (i32.add
                (call $child)
                (i32.const 13)
            )
        )
        (func $child (result i32) (call $grandChild))
        (func $grandChild (result i32) (call
           $greatGrandChild))
        (func $greatGrandChild (result i32) (i32.const 7))
    )
    ```

1.  要生成 WebAssembly 模块，我们将运行以下命令：

    ```rs
      optimizer.wast -o inline.wasm --print
    (module
    (type $0 (func (result i32)))
    (export "parent" (func $parent))
    (func $parent (; 0 ;) (type $0) (result i32)
    (i32.add
    (call $child)
    (i32.const 13)
      )
    )
    (func $child (; 1 ;) (type $0) (result i32)
      (call $grandChild)
    )
    (func $grandChild (; 2 ;) (type $0) (result i32)
      (call $greatGrandChild)
    )
    (func $greatGrandChild (; 3 ;) (type $0) (result i32)
      (i32.const 7)
    )
    )
    ```

这将生成 `inline.wasm`。`--print` 选项在转换成 WebAssembly 二进制之前打印出 WebAssembly 文本格式。我们还传递了 `-o` 选项，将 WebAssembly 模块输出为 `inline.wasm`：

```rs
60B inline-optimize.wasm
273B inline-optimize.wat
```

这个生成的二进制文件在内存中占用 60 字节。

1.  我们可以使用 `--inlining-optimizing` 选项进一步优化二进制文件：

    ```rs
      optimizer.wast -o inline.wasm --print --inlining-
      optimizing
    ```

这将优化函数并将函数内联到二进制文件被调用的地方。让我们检查生成的文件大小：

```rs
39B inline-optimize.wasm
273B inline-optimize.wat
```

生成的文件只有 39 字节，比原始二进制文件少了 35%。

1.  要检查 `wasm-opt` 支持的各种选项，请运行以下命令：

    ```rs
    /path/to/bin/folder/of/binaryen/wasm-opt –help
    ```

`wasm-opt` 工具帮助我们进一步优化 WebAssembly 二进制文件。接下来，让我们探索 `wasm2js` 工具。

# wasm2js

`wasm2js` 工具将 WASM/WAST 文件转换为 JavaScript 文件。让我们看看步骤：

1.  创建一个名为 `add-with-export.wast` 的文件：

    ```rs
    $ touch add-with-export.wast
    ```

然后，添加以下代码：

```rs
(module
    (export "add" (func $add))
    (func $add (param $x i32) (param $y i32) 
      (result i32)
        (i32.add
            (local.get $x)
            (local.get $y)
        )
    )
)
```

1.  为了使用 `wasm2js` 将 WebAssembly 文本格式转换为 JavaScript，运行以下命令：

    ```rs
    $ /path/to/build/directory/of/binaryen/wasm2js add-
      with-export.wast
    ```

这将打印出生成的 JavaScript：

```rs
function asmFunc(global, env) {
var Math_imul = global.Math.imul;
var Math_fround = global.Math.fround;
var Math_abs = global.Math.abs;
var Math_clz32 = global.Math.clz32;
var Math_min = global.Math.min;
var Math_max = global.Math.max;
var Math_floor = global.Math.floor;
var Math_ceil = global.Math.ceil;
var Math_sqrt = global.Math.sqrt;
var abort = env.abort;
var nan = global.NaN;
var infinity = global.Infinity;
function add(x, y) {
  x = x | 0;
  y = y | 0;
  return x + y | 0 | 0;
}
return {
  "add": add
};
}
var retasmFunc = asmFunc({
    Math,
    Int8Array,
    Uint8Array,
    Int16Array,
    Uint16Array,
    Int32Array,
    Uint32Array,
    Float32Array,
    Float64Array,
    NaN,
    Infinity
  }, {
    abort: function() { throw new Error('abort'); }
  });
export var add = retasmFunc.add;
```

`asmFunc` 函数被定义了。在 `asmFunc` 中，我们从全局对象中导入数学函数。之后，我们有一个 `add` 函数。`add` 函数初始化 `x` 和 `y`。该函数返回两个值的和。最后，我们返回 `add` 函数。

注意

生成的 JavaScript 是 `asmjs` 而不是常规 JavaScript。我们还在 JavaScript 中从全局命名空间导入了大量函数到 `asmFunc`。

`wasm2js` 工具使得从 WebAssembly 模块生成 JavaScript 变得容易。生成的 JavaScript 模块比其常规 JavaScript 对应物更快。这可以用作不支持 WebAssembly 的浏览器的 polyfill。

# 摘要

在本章中，我们看到了如何安装 Binaryen 以及 Binaryen 工具包提供的各种工具。Binaryen 使得将 WebAssembly 模块转换为各种格式变得更加容易。这是一个使您的 WebAssembly 之旅更加轻松和高效的工具。

在下一章中，我们将开始我们的 Rust 和 WebAssembly 之旅。
