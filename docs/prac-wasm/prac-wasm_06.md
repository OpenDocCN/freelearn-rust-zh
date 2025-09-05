# *第四章*：理解 WebAssembly 二进制工具包

Rust 编译器链将 Rust 代码转换为 WebAssembly 二进制。但生成的二进制文件在大小和性能上都进行了优化。理解、调试和验证二进制代码（它是一堆十六进制数字）是困难的。将 WebAssembly 二进制转换回原始源代码非常困难。**WebAssembly 二进制工具包**（**WABT**）有助于将 WebAssembly 二进制转换为人类可读的格式，例如**WebAssembly 文本**（**WAST**）格式或 C 本地代码。

注意

这里的本地代码并不指代原始的真实来源；相反，它指的是机器解释的 C 本地代码。

WebAssembly 二进制工具包（WebAssembly Binary Toolkit）简称为 WABT，发音为"*wabbit*"。WABT 提供了一套用于转换、分析和测试 WebAssembly 二进制文件的工具。

在本章中，我们将探讨 WABT 以及它是如何帮助将 WebAssembly 二进制转换为各种格式，以及为什么它是有用的。本章将涵盖以下主要主题：

+   开始使用 WABT

+   将 WAST 转换为 WASM

+   将 WASM 转换为 WAST

+   将 WASM 转换为 C

+   将 WAST 转换为 JSON

+   了解 WABT 提供的其他一些工具

# 技术要求

你可以在 GitHub 上找到本章中存在的代码文件，链接为[`github.com/PacktPublishing/Practical-WebAssembly`](https://github.com/PacktPublishing/Practical-WebAssembly)。

# 开始使用 WABT

让我们先安装 WABT，然后探索 WABT 工具提供的各种选项。

## 安装 WABT

为了安装 WABT，首先从 GitHub 克隆仓库：

```rs
$ git clone --recursive https://github.com/WebAssembly/wabt
```

注意

我们在这里使用`--recursive`标志，因为它确保在创建克隆之后，仓库中的所有子模块（如`test-suite`）都被初始化。

进入克隆的仓库，创建一个名为`build`的文件夹，然后进入`build`文件夹。这是我们生成二进制文件的地方：

```rs
$ cd wabt
$ mkdir build
$ cd build
```

注意

你还需要安装 CMake。有关更多说明，请参阅[`cmake.org/download/`](https://cmake.org/download/)。

要使用 CMake 构建二进制文件，我们首先需要生成构建系统。我们指定`cmake`命令的源。然后，CMake 将构建树并为指定的源生成一个构建系统，使用`CMakeLists.txt`文件。

### Linux 或 macOS

为了生成项目构建系统，我们运行带有`wabt`文件夹路径的`cmake`命令。`cmake`命令接受相对路径和绝对路径。我们在这里使用相对路径（`..`）：

```rs
$ cmake ..
```

现在我们可以使用`cmake build`来构建项目。`cmake build`利用生成的项目二进制树来生成二进制文件：

```rs
Usage: cmake --build <dir> [options] [-- [native-options]]
Options:
  <dir> = Project binary directory to be built.
  --parallel [<jobs>], -j [<jobs>]
        = Build in parallel using the given number of jobs.
                   If <jobs> is omitted the native build
                   tool's
                   default number is used.
                   The CMAKE_BUILD_PARALLEL_LEVEL
                   environment variable
                   specifies a default parallel level when
                   this option
                   is not given.
  --target <tgt>..., -t <tgt>...
                 = Build <tgt> instead of default targets.
  --config <cfg> = For multi-configuration tools, choose
    <cfg>.
  --clean-first  = Build target 'clean' first, then build.
                   (To clean only, use --target 'clean'.)
  --verbose, -v  = Enable verbose output - if supported –
    including the build commands to be executed.
  --             = Pass remaining options to the native
    tool.
```

`cmake build`命令需要`<dir>`选项来生成二进制文件。`cmake build`命令接受前面代码块中列出的标志：

```rs
$ cmake --build .
....
[100%] Built target spectest-interp-copy-to-bin
```

### Windows

安装 CMake 和 Visual Studio（>= 2015）。然后，在`build`文件夹内运行`cmake`：

```rs
$ cmake [wabt project root] -DCMAKE_BUILD_TYPE=[config] –
  DCMAKE_INSTALL_PREFIX=[install directory] -G [generator]
```

`[config]`参数可以是`DEBUG`或`RELEASE`。

`[安装目录]`参数应该是您想要安装二进制文件的文件夹。

`[generator]`参数应该是您想要生成的项目类型，例如，Visual Studio 14 2015。您可以通过运行`cmake –help`来查看可用生成器的列表。

这将在指定的文件夹内构建和安装所有必需的可执行文件：

```rs
$ cd build
$ cmake --build .. --config RELEASE --target install
```

一旦成功安装了所有 WABT 工具，您可以将它们添加到您的路径中或从它们的路径中调用它们。

`build`文件夹包含以下二进制文件：

```rs
$ tree -L 1
├── dummy
├── hexfloat_test
├── spectest-interp
├── wabt-unittests
├── wasm-c-api-global
├── wasm-c-api-hello
├── wasm-c-api-hostref
├── wasm-c-api-memory
├── wasm-c-api-multi
├── wasm-c-api-reflect
├── wasm-c-api-serialize
├── wasm-c-api-start
├── wasm-c-api-table
├── wasm-c-api-threads
├── wasm-c-api-trap
├── wasm-decompile
├── wasm-interp
├── wasm-objdump
├── wasm-opcodecnt
├── wasm-strip
├── wasm-validate
├── wasm2c
├── wasm2wat
├── wast2json
├── wat-desugar
└── wat2wasm
```

这确实是一个庞大的二进制文件列表。让我们详细看看每个二进制文件能做什么：

+   `wat2wasm` – 这个工具帮助将 WAST 格式转换为**WebAssembly 模块**（**WASM**）。

+   `wat-desugar` – 这个工具读取 WASM S 表达式文件并格式化它。

+   `wast2json` – 这个工具验证并将 WAST 格式转换为 JSON 格式。

+   `wasm2wat` – 这个工具将 WASM 转换为 WAST 格式。

+   `wasm2c` – 这个工具将 WASM 转换为 C 原生代码。

+   `wasm-validate` – 这个工具验证给定的 WebAssembly 是否按照规范构建。

+   `wasm-strip` – 正如我们在上一章中看到的，WASM 由多个部分组成。模块中的自定义部分仅用于关于模块及其生成过程中使用的工具的额外元信息。`wasm-strip`从 WASM 中移除自定义部分。

+   `wasm-opcodecnt` – 这个工具读取 WASM 并计算 WebAssembly 模块中指令操作码的使用情况。

+   `wasm-objdump` – 这个工具帮助打印关于 WASM 二进制文件的信息。它与 objdump ([`en.wikipedia.org/wiki/Objdump`](https://en.wikipedia.org/wiki/Objdump))类似，但用于 WebAssembly 模块。

+   `wasm-interp` – 这个工具使用基于堆栈的解释器解码并运行 WebAssembly 二进制文件。

+   `wasm-decompile` – 这个工具帮助将 WASM 二进制文件反编译成可读的类似 C 的语法。

以下是一些测试二进制文件：

+   `hexfloat_test`

+   `spectest-interp`

+   `wabt-unittests`

    注意

    探索 WABT 支持的各个提案，请访问[`github.com/WebAssembly/wabt#supported-proposals`](https://github.com/WebAssembly/wabt#supported-proposals)。

我们已经构建了 WABT 并生成了工具。现在，让我们探索最重要和最有用的工具。

# 将 WAST 转换为 WASM

`wat2wasm`帮助将 WAST 格式转换为 WASM。让我们试一试：

1.  创建一个名为`wabt-playground`的新文件夹并进入该文件夹：

    ```rs
    $ mkdir wabt-playground
    $ cd wabt-playground
    ```

1.  创建一个名为`add.wat`的`.wat`文件：

    ```rs
    $ touch add.wat
    ```

1.  将以下内容添加到`add.wat`：

    ```rs
    (module
    (func $add (param $lhs i32) (param $rhs i32) (result i32)
     get_local $lhs 
      get_local $rhs
        i32.add
    )
    )
    ```

1.  使用`wat2wasm`二进制文件将 WAST 格式转换为 WASM：

    ```rs
    $ /path/to/build/directory/of/wabt/wat2wasm add.wat
    ```

这将在`add.wasm`文件中生成有效的 WebAssembly 二进制文件：

```rs
00 61 73 6d 01 00 00 00 01 07 01 60 02 7f 7f 017f 03
 02 01 00 0a 09 01 07 00 20 00 20 01 6a 0b
```

注意，生成的二进制文件大小为 32 字节。

1.  WABT 读取 WAST 格式文件（`.wat`）并将其转换为 WebAssembly 模块（`.wasm`）。`wat2wasm`首先验证给定的文件（`.wat`），然后将其转换为`.wasm`文件。要检查`wat2wasm`支持的选项，我们可以运行以下命令：

    ```rs
    $ /path/to/build/directory/of/wabt/wat2wasm --help
    usage: wat2wasm [options] filename
      read a file in the wasm text format, check it for
      errors, and
      convert it to the wasm binary format.
    examples:
      # parse and typecheck test.wat
      $ wat2wasm test.wat
      # parse test.wat and write to binary file test.wasm
      $ wat2wasm test.wat -o test.wasm
      # parse spec-test.wast, and write verbose output to
        stdout (including
      # the meaning of every byte)
      $ wat2wasm spec-test.wast -v
    options:
          --help              Print this help message
          --version           Print version information
      -v, --verbose       Use multiple times for more info
          --debug-parser      Turn on debugging the parser
                              of wat files
      ...
          --debug-names       Write debug names to the
                              generated binary file
          --no-check          Don't check for invalid
                              modules
    ```

如果我们需要以不同的名称生成 WASM 文件，可以使用 `-o` 选项和文件名。例如，`wat2wasm add.wat -o add.wasm` 将生成 `add.wasm`。

1.  `wat2wasm` 还提供了详细的输出，清楚地解释了 WASM 的结构。为了查看 WASM 的结构，我们使用 `-v` 选项运行它：

    ```rs
     $ /path/to/build/directory/of/wabt/wat2wasm add.wat -v
    0000000: 0061 736d     ; WASM_BINARY_MAGIC
    0000004: 0100 0000     ; WASM_BINARY_VERSION
    ; section "Type" (1)
    0000008: 01            ; section code
    0000009: 00            ; section size (guess)
    000000a: 01            ; num types
    ; type 0
    000000b: 60            ; func
    000000c: 02            ; num params
    000000d: 7f            ; i32
    000000e: 7f            ; i32
    000000f: 01            ; num results
    0000010: 7f            ; i32
    0000009: 07            ; FIXUP section size
    ; section "Function" (3)
    0000011: 03            ; section code
    0000012: 00            ; section size (guess)
    0000013: 01            ; num functions
    0000014: 00            ; function 0 signature index
    0000012: 02            ; FIXUP section size
    ; section "Code" (10)
    0000015: 0a            ; section code
    0000016: 00            ; section size (guess)
    0000017: 01            ; num functions
    ; function body 0
    0000018: 00            ; func body size (guess)
    0000019: 00            ; local decl count
    000001a: 20            ; local.get
    000001b: 00            ; local index
    000001c: 20            ; local.get
    000001d: 01            ; local index
    000001e: 6a            ; i32.add
    000001f: 0b            ; end
    0000018: 07            ; FIXUP func body size
    0000016: 09            ; FIXUP section size
    ```

前面的输出是对生成的二进制的详细描述。最左边的七个数字是索引，后面跟着一个冒号。接下来的两个字符是实际的二进制代码，然后是注释。注释描述了二进制（操作）码的功能。

前两行指定 `wasm_magic_header` 及其版本。下一个部分是 `Type` 部分。`Type` 部分定义了部分 ID，后面跟着部分大小，然后是类型块的数量。在我们的例子中，我们只有一个类型。`type 0` 部分定义了 `add` 函数的类型签名。

然后，我们有 `Function` 部分。在 `Function` 部分，我们有部分 ID，后面跟着部分大小，然后是函数的数量。函数部分没有函数体。函数体在 `Code` 部分定义。

在生成二进制时，我们可以通过使用适当的 `enable-*` 和 `disable-*` 选项分别启用新功能和禁用现有功能来让编译器包含新功能和禁用现有功能。

注意

你也可以在网上查看版本，在 [`webassembly.github.io/wabt/demo/wat2wasm/`](https://webassembly.github.io/wabt/demo/wat2wasm/) 探索 WABT 工具。

我们已经将 WAST 转换成了 WASM。现在，让我们探索如何使用 `wasm2wat` 将 WASM 转换为 WAST。

# 将 WASM 转换为 WAST

有时，为了调试或理解，我们需要知道 WASM 正在做什么。WABT 有一个 `wasm2wat` 转换器。它可以帮助将 WASM 转换为 WAST 格式：

```rs
$ /path/to/build/directory/of/wabt/wasm2wat add.wasm
(module
  (type (;0;) (func (param i32 i32) (result i32)))
  (func (;0;) (type 0) (param i32 i32) (result i32)
    local.get 0
    local.get 1
    i32.add))
```

运行前面的命令将 `add.wasm` 转换回 WAST 格式，并在控制台打印输出。

如果你想将其保存为文件，可以使用 `-o` 标志：

```rs
$ /path/to/build/directory/of/wabt/wasm2wat add.wasm -o new_
  add.wat
```

此命令创建一个 `new_add.wat` 文件。

要检查 `wasm2wat` 支持的各种选项，我们可以运行以下命令：

```rs
$ wasm2wat --help
usage: wasm2wat [options] filename

  Read a file in the WebAssembly binary format, and convert 
  it to
  the WebAssembly text format.

examples:
  # parse binary file test.wasm and write text file test.wast
  $ wasm2wat test.wasm -o test.wat

  # parse test.wasm, write test.wat, but ignore the debug names, if any
  $ wasm2wat test.wasm --no-debug-names -o test.wat

options:
      --help                                Print this help 
        message
      --version                             Print version 
        information
  -v, --verbose                             Use multiple times 
   for more info
  -o, --output=FILENAME                     Output file for the 
   generated wast file, by default use stdout
  -f, --fold-exprs                          Write folded 
   expressions where possible
  ....
      --no-debug-names                      Ignore debug names 
        in the binary file
      --ignore-custom-section-errors        Ignore errors in 
        custom sections
      --generate-names                      Give auto-generated
        names to non-named functions, types, etc.
      --no-check                            Don't check for 
        invalid modules
```

注意

`wasm2wat` 和 `wat2wasm` 几乎有相同的选项。

使用 `-v` 选项运行前面的命令将打印 WAST 格式的 AST 语法：

```rs
$ /path/to/build/directory/of/wabt/wasm2wat add.wasm -o new_
  add.wat -v
BeginModule(version: 1)
  BeginTypeSection(7)
    OnTypeCount(1)
    OnType(index: 0, params: [i32, i32], results: [i32])
  EndTypeSection
  BeginFunctionSection(2)
    OnFunctionCount(1)
    OnFunction(index: 0, sig_index: 0)
  EndFunctionSection
  BeginCodeSection(9)
    OnFunctionBodyCount(1)
    BeginFunctionBody(0, size:7)
    OnLocalDeclCount(0)
    OnLocalGetExpr(index: 0)
    OnLocalGetExpr(index: 1)
    OnBinaryExpr("i32.add" (106))
    EndFunctionBody(0)
  EndCodeSection
  BeginCustomSection('name', size: 28)
    BeginNamesSection(28)
      OnNameSubsection(index: 0, type: function, size:6)
      OnFunctionNameSubsection(index:0, nametype:1, size:6)
      OnFunctionNamesCount(1)
      OnFunctionName(index: 0, name: "add")
      OnNameSubsection(index: 1, type: local, size:13)
      OnLocalNameSubsection(index:1, nametype:2, size:13)
      OnLocalNameFunctionCount(1)
      OnLocalNameLocalCount(index: 0, count: 2)
      OnLocalName(func_index: 0, local_index: 0, name: "lhs")
      OnLocalName(func_index: 0, local_index: 1, name: "rhs")
    EndNamesSection
  EndCustomSection
EndModule
```

整个代码块被包裹在 `BeginModule` 和 `EndModule` 之间。`BeginModule` 包含 WebAssembly 二进制的版本。

在 `BeginModule` 内部，`BeginTypeSection` 从类型的部分索引（即 `7`）开始，后面跟着 `OnTypeCount`，定义的类型数量。然后，我们有 `OnType` 的实际类型定义。我们通过 `EndTypeSection` 结束类型部分。

然后，我们有由 `BeginFunctionSection` 和 `EndFunctionSection` 标记的 `Function` 部分。这部分包含函数计数（`OnFunctionCount`）和函数定义（`OnFunction`）。

最后，我们有代码部分，它包含函数的实际主体。代码部分以`BeginCodeSection`开始，以`EndCodeSection`结束。

有时，WASM 可能包含调试名称。我们可以使用`--no-debug-names`标志来忽略它们：

```rs
 $ /path/to/build/directory/of/wabt/wasm2wat add.wasm -o new_
  add.wat -v --no-debug-names
BeginModule(version: 1)
  BeginTypeSection(7)
    OnTypeCount(1)
    OnType(index: 0, params: [i32, i32], results: [i32])
  EndTypeSection
  BeginFunctionSection(2)
    OnFunctionCount(1)
    OnFunction(index: 0, sig_index: 0)
  EndFunctionSection
  BeginCodeSection(9)
    OnFunctionBodyCount(1)
    BeginFunctionBody(0, size:7)
    OnLocalDeclCount(0)
    OnLocalGetExpr(index: 0)
    OnLocalGetExpr(index: 1)
    OnBinaryExpr("i32.add" (106))
    EndFunctionBody(0)
  EndCodeSection
  BeginCustomSection('name', size: 28)
  EndCustomSection
EndModule
```

注意`BeginCustomSection`和`EndCustomSection`。与之前的输出比较，它缺少`NamesSection`。现在，让我们查看`wasm2wat`工具提供的各种选项。

## -f 或--fold-exprs

作为功能编程的大粉丝，这是可用的最酷的选项之一。它折叠表达式；也就是说，它将表达式 1 >> 表达式 2 >> 操作转换为操作 >> 表达式 1 >> 表达式 2。

让我们看看实际操作：

1.  创建一个名为`fold.wat`的 WAST 文件，并填充以下内容：

    ```rs
    (module
        (func $fold (result i32)
            i32.const 22
            i32.const 20
            i32.add
        )
    )
    ```

1.  首先，使用`wat2wasm`将其转换为 WASM：

    ```rs
    $ /path/to/build/directory/of/wabt/wat2wasm -v
      fold.wat
    ; some contents
    0000018: 41                                        ;
      i32.const
    0000019: 16                                        ;
      i32 literal
    000001a: 41                                        ;
      i32.const
    000001b: 14                                        ;
      i32 literal
    000001c: 6a                                        ;
      i32.add
    ; other contents
    ```

这将创建`fold.wasm`。

1.  现在，使用`wasm2wat`将 WASM 转换为 WAST 格式，并传递`-f`选项：

    ```rs
    $ /path/to/build/directory/of/wabt/wasm2wat -v
      fold.wasm -o converted_fold.wat -f
    ```

这将创建一个名为`converted_fold.wat`的文件：

```rs
(module
    (type (;0;) (func (result i32)))
    (func (;0;) (type 0) (result i32)
        (i32.add
            (i32.const 1)
            (i32.const 2))))
```

而不是使用`i32.const 1`（表达式 1）和`i32.const 2`（表达式 2）然后执行`i32.add`（操作），这生成一个输出，`i32.add`（操作），然后是`i32.const 1`（表达式 1）和`i32.const 2`（表达式 2）。

在生成`wat`时，我们可以通过使用适当的`enable-*`和`disable-*`选项来启用编译器包括新的和闪亮的功能，并禁用各种现有功能。

我们已将 WASM 转换为 WAST。现在，让我们探索如何使用`wasm2c`将 WASM 转换为本地代码（C）。

# 将 WASM 转换为 C

WABT 有一个`wasm2c`转换器，可以将 WASM 转换为 C 源代码和头文件。

让我们创建一个`simple.wat`文件：

```rs
$ touch simple.wat
```

将以下内容添加到`simple.wat`中：

```rs
(module
    (func $uanswer (result i32)
        i32.const 22
        i32.const 20
        i32.add
    )
)
```

`wat`在这里定义了一个`uanswer`函数，将`22`和`20`相加得到`42`作为答案。让我们使用`wat2wasm`创建一个 WebAssembly 二进制文件：

```rs
$ /path/to/build/directory/of/wabt/wat2wasm simple.wat -o 
  simple.wasm
```

这生成了`simple.wasm`二进制文件。现在，使用`wasm2c`将二进制文件转换为 C 代码：

```rs
$ /path/to/build/directory/of/wabt/wasm2c simple.wasm -o 
  simple.c
```

这生成了`simple.c`和`simple.h`。

注意

`simple.c`和`simple.h`可能看起来很大。记住这是一个自动生成的文件，它包含了程序运行所需的所有必要的头文件和配置。

## simple.h

`simple.h`（头文件）包括头文件的标准化样板代码。它还包括`_cplusplus`条件，以防止 C++中的名称修饰：

```rs
#ifndef SIMPLE_H_GENERATED_
#define SIMPLE_H_GENERATED_
#ifdef __cplusplus
extern "C" {
#endif

...

#ifdef __cplusplus
}
#endif
#endif
```

由于我们使用了`i32.const`和`i32.add`，头文件也导入了`stdint.h`。它包括`wasm-rt.h`。`wasm-rt.h`头文件导入了必要的 WASM 运行时变量。

接下来，我们可以指定一个模块前缀。当使用多个模块时，模块前缀很有用。由于我们只有一个模块，我们使用一个空前缀：

```rs

#ifndef WASM_RT_MODULE_PREFIX
#define WASM_RT_MODULE_PREFIX
#endif

#define WASM_RT_PASTE_(x, y) x ## y
#define WASM_RT_PASTE(x, y) WASM_RT_PASTE_(x, y)
#define WASM_RT_ADD_PREFIX(x) WASM_RT_PASTE(WASM_RT_MODULE_PREFIX, x)
```

接下来，我们有一些针对 WASM 支持的多种数字格式的 typedef：

```rs
typedef uint8_t u8;
typedef int8_t s8;
typedef uint16_t u16;
typedef int16_t s16;
typedef uint32_t u32;
typedef int32_t s32;
typedef uint64_t u64;
typedef int64_t s64;
typedef float f32;
typedef double f64;
```

## simple.c

`simple.c`提供了从 WASM 二进制生成的实际 C 代码。生成的代码具有以下内容：

1.  我们将需要以下库列表在代码中使用：

    ```rs
    #include <math.h>
    #include <string.h>

    #include "simple.h"
    ```

1.  接下来，我们定义当发生错误时调用的陷阱：

    ```rs
    #define TRAP(x) (wasm_rt_trap(WASM_RT_TRAP_##x), 0)
    ```

1.  然后，我们定义 `PROLOGUE`、`EPILOGUE` 和 `UNREACHABLE_TRAP`，分别是在执行开始前、执行后以及执行遇到不可达异常时调用的：

    ```rs
    #define FUNC_PROLOGUE                                            \
      if (++wasm_rt_call_stack_depth >
        WASM_RT_MAX_CALL_STACK_DEPTH) \
        TRAP(EXHAUSTION)

    #define FUNC_EPILOGUE --wasm_rt_call_stack_depth

    #define UNREACHABLE TRAP(UNREACHABLE)
    ```

`WASM_RT_MAX_CALL_STACK_DEPTH` 是堆栈的最大深度。默认值为 `500`，但我们可以更改它。注意，如果达到限制，则会抛出异常。

1.  接下来，我们定义内存操作：

    ```rs
        addr) {   \
        addr, t2 value) { \
        MEMCHECK(mem, addr, t1);       
    \
        t1 wrapped = (t1)value;      
    \
        __builtin_memcpy(&mem->data[addr], &wrapped,
          sizeof(t1));          \
      }
    ```

`MEMCHECK` 检查内存。`DEFINE_LOAD` 和 `DEFINE_STORE` 块定义了如何在内存中加载和存储值。

1.  接下来，我们为示例中具有的各种数据类型定义了一组加载和存储操作：

    ```rs
    DEFINE_LOAD(i32_load, u32, u32, u32);
    DEFINE_LOAD(i64_load, u64, u64, u64);
    ...
    DEFINE_LOAD(i64_load32_u, u32, u64, u64);
    DEFINE_STORE(i32_store, u32, u32);
    DEFINE_STORE(i64_store, u64, u64);
    ...
    DEFINE_STORE(i64_store32, u32, u64);
    ```

然后，我们定义各种数据类型支持的各种函数，例如 `TRUNC`、`DIV` 和 `REM`。

1.  接下来，我们使用 `func_types` 初始化函数类型：

    ```rs
    static u32 func_types[1];

    static void init_func_types(void) {
      func_types[0] = wasm_rt_register_func_type(0, 1,
        WASM_RT_I32);
    }
    ```

这在 WASM 中注册了（结果 i32）类型。

1.  接下来，我们初始化 `globals`、`memory`、`table` 和 `exports`：

    ```rs
    static void init_globals(void) {
    }

    static void init_memory(void) {
    }

    static void init_table(void) {
      uint32_t offset;
    }

    static void init_exports(void) {
    }
    ```

1.  现在，我们实现实际的功能：

    ```rs
    static u32 w2c_f0(void) {
      FUNC_PROLOGUE;
      u32 w2c_i0, w2c_i1;
      w2c_i0 = 22u;
      w2c_i1 = 20u;
      w2c_i0 += w2c_i1;
      FUNC_EPILOGUE;
      return w2c_i0;
    }
    ```

函数是静态的。它在执行前调用 `FUNC_PROLOGUE`。然后，它创建两个变量（都是无符号 u32）。然后，我们定义两个变量的值，分别是 `22` 和 `20`。之后，我们将它们相加。一旦执行完成，我们调用 `FUNC_EPILOGUE`。最后，我们返回值。

注意

由于我们在 `wat` 中没有导出任何内容，因此 `init_exports` 是空的。

我们已经将 WASM 转换为 C。生成的代码与原始代码略有不同。让我们探索如何使用 `wast2json` 将 WAST 转换为 JSON。

# 将 WAST 转换为 JSON

`wast2json` 工具读取 WAST 格式并解析它，检查错误，然后将 WAST 转换为 JSON 文件。它为 WAST 文件生成一个 JSON 和 WASM 文件。然后，它将 JSON 中的 WASM 链接起来：

```rs
$ /path/to/build/directory/of/wabt/wast2json add.wat -o add.
  json
$ cat add.json
{"source_filename": "add.wat",
"commands": [
{"type": "module", "line": 1, "filename": "add.0.wasm"}]}
```

要检查 `wast2json` 支持的各种选项，请运行以下命令：

```rs
$ /path/to/build/directory/of/wabt/wast2json --help
usage: wast2json [options] filename

  read a file in the wasm spec test format, check it for 
  errors, and
  convert it to a JSON file and associated wasm binary files.

examples:
  # parse spec-test.wast, and write files to spec-test.json. 
  Modules are
  # written to spec-test.0.wasm, spec-test.1.wasm, etc.
  $ wast2json spec-test.wast -o spec-test.json

options:
      --help                                Print this help 
        message
      --version                             Print version 
        information
  -v, --verbose                             Use multiple times 
   for more info
      --debug-parser                        Turn on debugging 
        the parser of wast files
      --enable-exceptions                   Enable Experimental 
        exception handling
      --disable-mutable-globals             Disable Import/
        export mutable globals
      --disable-saturating-float-to-int     Disable Saturating
        float-to-int operators
      --disable-sign-extension              Disable Sign-
        extension operators
      --enable-simd                         Enable SIMD support
      --enable-threads                      Enable Threading 
        support
      --disable-multi-value                 Disable Multi-value
      --enable-tail-call                    Enable Tail-call 
        support
      --enable-bulk-memory                  Enable Bulk-memory 
        operations
      --enable-reference-types              Enable Reference 
        types (externref)
      --enable-annotations                  Enable Custom 
        annotation syntax
      --enable-gc                           Enable Garbage 
        collection
      --enable-memory64                     Enable 64-bit 
        memory
      --enable-all                          Enable all features
  -o, --output=FILE                         output JSON file
  -r, --relocatable                         Create a 
   relocatable wasm binary (suitable for linking with e.g. lld)
      --no-canonicalize-leb128s             Write all LEB128 
        sizes as 5-bytes instead of their minimal size
      --debug-names                         Write debug names 
        to the generated binary file
      --no-check                            Don't check for 
        invalid modules
```

这些是常用的 WABT 工具。WABT 还提供了一些其他工具，有助于调试和更好地理解 WASM。

# 理解 WABT 提供的几个其他工具

除了转换器之外，WABT 还提供了一些工具，帮助我们更好地理解 WASM。在本节中，让我们探索 WABT 提供的以下工具：

+   `wasm-objdump`

+   `wasm-strip`

+   `wasm-validate`

+   `wasm-interp`

## `wasm-objdump`

目标代码不过是计算机语言中的一系列指令或语句。目标代码是编译器产生的。目标代码随后收集在一起，并存储在目标文件中。目标文件是链接和调试信息的元数据持有者。目标文件中的机器代码不能直接执行，但在调试时提供有价值的信息，并有助于链接。

注意

`objdump` 是 POSIX 系统中可用的工具，它提供了一种解汇编二进制格式并打印运行代码汇编格式的方法。

`wasm-objdump` 工具提供以下选项：

```rs
$ /path/to/build/directory/of/wabt/wasm-objdump --help
usage: wasm-objdump [options] filename+

  Print information about the contents of wasm binaries.

examples:
  $ wasm-objdump test.wasm

options:
      --help                   Print this help message
      --version                Print version information
  -h, --headers                Print headers
  -j, --section=SECTION        Select just one section
  -s, --full-contents          Print raw section contents
  -d, --disassemble            Disassemble function bodies
      --debug                  Print extra debug information
  -x, --details                Show section details
  -r, --reloc                  Show relocations inline with 
   disassembly
```

至少应提供以下选项之一给 `wasm-objdump` 命令：

```rs
-d/--disassemble
-h/--headers
-x/--details
-s/--full-contents
```

`-h` 选项打印 WASM 中所有可用的头信息。例如，在我们的 add 示例（`add.wasm`）中，我们有以下内容：

```rs
$ /path/to/build/directory/of/wabt/wasm-objdump add.wasm -h
add.wasm: file format wasm 0x1

Sections:

     Type start=0x0000000a end=0x00000011 (size=0x00000007)
  count: 1
Function start=0x00000013 end=0x00000015 (size=0x00000002) 
  count: 1
     Code start=0x00000017 end=0x00000020 (size=0x00000009) 
  count: 1
```

这里，我们有三个部分可用：

+   `Type`

+   `Function`

+   `Code`

`-d` 选项打印函数的实际主体：

```rs

$/path/to/build/directory/of/wabt/wasm-objdump add.wasm -d
add.wasm: file format wasm 0x1

Code Disassembly:

000019 func[0]:
00001a: 20 00 | local.get 0
00001c: 20 01 | local.get 1
00001e: 6a | i32.add
00001f: 0b | end
```

它反汇编汇编函数并仅打印函数主体。

`-x` 选项打印 WebAssembly 二进制文件的段详细信息：

```rs
$ /path/to/build/directory/of/wabt/wasm-objdump add.wasm -x

add.wasm: file format wasm 0x1

Section Details:

Type[1]:
- type[0] (i32, i32) -> i32
Function[1]:
- func[0] sig=0
Code[1]:
- func[0] size=7
```

`-s` 选项打印所有可用的段的内容：

```rs
$ /path/to/build/directory/of/wabt/wasm-objdump add.wasm -s

add.wasm: file format wasm 0x1

Contents of section Type:
000000a: 0160 027f 7f01 7f .`.....

Contents of section Function:
0000013: 0100 ..

Contents of section Code:
0000017: 0107 0020 0020 016a 0b ... . .j.
```

## wasm-strip

WASM 中的自定义部分包含有关函数和所有在 WASM 中定义的局部变量的名称的信息。它可能包含有关构建和 WASM 如何创建的信息。这是附加信息。它会使二进制文件膨胀，但不会添加任何功能。

我们可以使用 `wasm-strip` 工具删除自定义部分以减小二进制文件大小：

1.  创建一个包含以下内容的 `wat` 文件：

    ```rs
    $ touch simple.wat
    (module
        (func $fold (result i32)
            i32.const 22
            i32.const 20
            i32.add
        )
    )
    ```

1.  现在，使用 `wat2wasm` 将其转换为 WASM：

    ```rs
    $ /path/to/build/directory/of/wabt/wat2wasm simple.wat
      --debug-names
    $ l simple.wasm
    51B simple.wasm
    ```

注意

`--debug-names` 选项提供的生成自定义部分并将其添加到生成的二进制文件中。

之前的命令生成了一个 `simple.wasm` 文件，其大小为 51 字节。

1.  现在，让我们使用以下命令从二进制文件中移除自定义部分：

    ```rs
    $ /path/to/build/directory/of/wabt/wasm-strip add.wasm
    $ l simple.wasm
    30B simple.wasm
    ```

如您所见，它删除了 21 字节的不必要信息。一些 WASM 生成器添加自定义部分以获得更好的调试体验，但在生产部署时，我们不需要自定义部分。使用 `wasm-strip` 来删除它。

## wasm-validate

我们可以使用 `wasm-validate` 验证 WASM：

1.  使用以下内容创建 `error.wasm`:

    ```rs
    00 61 73 6d 03 00 00 00
                |
            Error value
    ```

1.  现在，运行 `wasm-validate` 检查 WASM 是否有效：

    ```rs
    $ /path/to/build/directory/of/wabt/wasm-validate
      error.wasm
    0000004: error: bad magic value
    ```

1.  `wasm-validate` 工具提供以下选项：

    ```rs
    usage: wasm-validate [options] filename
    ```

1.  读取 WebAssembly 二进制格式的文件并验证它：

    ```rs
    examples:
      # validate binary file test.wasm
      $ wasm-validate test.wasm

    options:
          --help                Print this help message
          --version             Print version information
    -v, --verbose             Use multiple times for 
       more info
          --enable-exceptions   Enable Experimental
            exception handling
          --disable-mutable-globals     Disable
            Import/export mutable globals
          --disable-saturating-float-to-int        Disable
            Saturating float-to-int operators
          --disable-sign-extension                 Disable
            Sign-extension operators
          --enable-simd                            Enable
            SIMD support
          --enable-threads                         Enable
            Threading support
          --disable-multi-value                    Disable
            Multi-value
          --enable-tail-call                       Enable
            Tail-call support
          --enable-bulk-memory                     Enable
            Bulk-memory operations
          --enable-reference-types                 Enable
            Reference types (externref)
          --enable-annotations                     Enable
            Custom annotation syntax
          --enable-gc                              Enable
            Garbage collection
          --enable-memory64                        Enable
            64-bit memory
          --enable-all                             Enable
            all features
          --no-debug-names                         Ignore
            debug names in the binary file
          --ignore-custom-section-errors           Ignore
            errors in custom sections
    ```

## wasm-interp

`wasm-interp` 读取 WebAssembly 二进制格式的文件并在基于堆栈的解释器中运行它。`wasm-interp` 工具解析二进制文件，然后进行类型检查：

```rs
$ /path/to/build/directory/of/wabt/wasm-interp add.wasm -v
BeginModule(version: 1)
  BeginTypeSection(7)
    OnTypeCount(1)
    OnType(index: 0, params: [i32, i32], results: [i32])
  EndTypeSection
  BeginFunctionSection(2)
    OnFunctionCount(1)
    OnFunction(index: 0, sig_index: 0)
  EndFunctionSection
  BeginCodeSection(9)
    OnFunctionBodyCount(1)
    BeginFunctionBody(0, size:7)
    OnLocalDeclCount(0)
    OnLocalGetExpr(index: 0)
    OnLocalGetExpr(index: 1)
    OnBinaryExpr("i32.add" (106))
    EndFunctionBody(0)
  EndCodeSection
EndModule
   0| local.get $2
   8| local.get $2
  16| i32.add %[-2], %[-1]
  20| drop_keep $2 $1
  32| return
```

最后五行是堆栈解释器执行代码的方式。

`wasm-interp` 工具提供以下选项：

```rs

usage: wasm-interp [options] filename [arg]...

  read a file in the wasm binary format and run it in a stack-
  based
  interpreter.

examples:
  ...

options:
      --help                                Print this help 
        message
      --version                             Print version 
        information
  ...
```

WABT 提供了一系列工具，使 WASM 更易于理解、调试和转换为各种可读格式。它是允许开发者更好地探索 WASM 的最重要的工具包之一。

# 摘要

在本章中，我们了解了 WABT 是什么以及如何安装和使用它提供的各种工具。WABT 工具在 WebAssembly 生态系统中非常重要，因为它提供了一个将不可读的紧凑二进制文件转换为可读的展开源代码的简单选项。

在下一章中，我们将探索 WASM 内部的各种部分。
