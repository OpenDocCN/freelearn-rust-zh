# *第五章*：理解 WebAssembly 模块中的部分

WebAssembly 模块由零个或多个部分组成。每个部分都有其自身的功能。在前几章中，我们看到了如何在 WebAssembly 模块内部定义函数。函数是 WebAssembly 模块内部的一个部分。

在本章中，我们将探讨 WebAssembly 模块内部的各个其他部分。了解 WebAssembly 模块内部的各个部分将使我们更容易识别、调试和编写高效的 WebAssembly 模块。在本章中，我们将涵盖以下部分：

+   导出和导入

+   全局变量

+   开始

+   内存

# 技术要求

你可以在 GitHub 上找到本章中存在的代码文件，地址为[`github.com/PacktPublishing/Practical-WebAssembly/tree/main/05-wasm-sections`](https://github.com/PacktPublishing/Practical-WebAssembly/tree/main/05-wasm-sections)。

# 导出和导入

WebAssembly 模块由导出和导入部分组成。这些部分负责将函数导出和导入到 WebAssembly 模块中。

## 导出

为了从 JavaScript 中调用 WebAssembly 模块中定义的函数，我们需要从 WebAssembly 模块中导出这些函数。导出部分是我们将定义所有从 WebAssembly 模块导出的函数的地方。

让我们回到前一章中的经典`add.wat`示例：

```rs
; add.wat
(module
    (func $add (param $lhs i32) (param $rhs i32) 
      (result i32)
        get_local $lhs
        get_local $rhs
        i32.add)
    (export "add" (func $add))
)
```

这里，我们使用`(export "add" (func $add))`语句导出了`add`函数。为了导出一个函数，我们使用了`export`关键字，后跟函数名称，然后是导出函数本身的指针。

记住 WebAssembly 是紧凑型的。因此，我们可以将导出语句及其本身的函数定义一起表示，如下所示：

```rs
; add.wat
(module
    (func $add (export "add") (param $lhs i32) 
      (param $rhs i32) (result i32)
        get_local $lhs
        get_local $rhs
        i32.add)
)
```

让我们使用 WABT 的`wat2wasm`工具，通过以下命令将 WebAssembly 文本格式转换为 WebAssembly 模块：

```rs
$ /path/to/wabt/bin/wat2wasm add.wat
```

让我们使用`hexdump`工具分析生成的字节码：

```rs
$ hexdump add.wasm
0000000 00 61 73 6d 01 00 00 00 01 07 01 60 02 7f 7f 01
0000010 7f 03 02 01 00 07 07 01 03 61 64 64 00 00 0a 09
0000020 01 07 00 20 00 20 01 6a 0b
0000029
```

如预期，第一个字节是二进制文件的魔术头和版本`00 61 73 6d 01 00 00 00`：

```rs
0000000: 0061 736d                   ; WASM_BINARY_MAGIC
0000004: 0100 0000                   ; WASM_BINARY_VERSION
```

接下来的位是`01`，代表类型部分的索引。随后，我们有类型部分的大小，为`07`。接下来的七个位是类型部分。`01`代表可用的类型定义数量：

```rs
; section "Type" (1)
0000008: 01                          ; section code
0000009: 07                          ; section size
000000a: 01                          ; num types
```

然后，我们有`60`，代表`func`。随后，我们有`02`，代表两个参数。`7f`是定义 i32 类型的操作码。由于两个参数都是 i32 类型，所以我们有连续的`7f`操作码。随后，最后两个位代表返回类型，还有一个`7f`代表 i32：

```rs
; type 0
000000b: 60                          ; func
000000c: 02                          ; num params
000000d: 7f                          ; i32
000000e: 7f                          ; i32
000000f: 01                          ; num results
0000010: 7f                          ; i32
```

在 `type` 部分之后，我们有 `func` 部分。`func` 部分的唯一标识符是 `03`。随后是 `02`，它定义了函数部分的大小。这意味着函数部分的大小仅为 2 位。但我们在 WebAssembly 文本格式中定义了 `add` 的函数定义，而函数的大小超过 2 位。那么这是怎么可能的呢？原因是函数部分没有函数体；相反，它只是定义了可用的函数。函数定义在代码部分。下一个 `01` 定义了模块中定义了一个函数：

```rs
; section "Function" (3)
0000011: 03                          ; section code
0000012: 02                          ; section size
0000013: 01                          ; num functions
0000014: 00                          ; function 0 signature
  index
```

然后，我们有导出部分，它以 `07` 开头。下一个 `07` 代表导出部分的大小。然后，我们定义导出部分中导出的数量。下一个位表示导出函数名的长度。接下来的 `03` 位表示函数名，`add`。然后，导出部分有导出类型和导出函数的索引：

```rs
; section "Export" (7)
0000015: 07                          ; section code
0000016: 07                          ; section size
0000017: 01                          ; num exports
0000018: 03                          ; string length
0000019: 6164 64                add  ; export name
000001c: 00                          ; export kind
000001d: 00                          ; export func index
```

最后一段以 `0a` 开头。`0a` 是代码部分的唯一标识符。代码部分的长度为 `09`。接下来，`01` 代表代码块中定义的函数数量。

接下来，`07` 代表函数定义的长度。接下来的七个位实际上定义了函数块。`00` 表示函数块没有任何局部声明。`20` 是 `get_local` 的操作码，我们使用 `00` 索引，然后再次使用 `20` 操作码来 `get_local` 并使用 `01` 索引。然后，我们使用 `i32.add` 将它们相加。i32 加法的操作码是 `6a`。最后，我们使用 `0b` 来结束函数代码块：

```rs
; section "Code" (10)
000001e: 0a                          ; section code
000001f: 09                          ; section size
0000020: 01                          ; num functions
  ; function body 0
0000021: 07                          ; func body size

0000022: 00                          ; local decl count
0000023: 20                          ; local.get
0000024: 00                          ; local index
0000025: 20                          ; local.get
0000026: 01                          ; local index
0000027: 6a                          ; i32.add
0000028: 0b                          ; end
```

我们已经看到了导出部分在 WebAssembly 模块中的表示。在下一节中，让我们看看导入部分在 WebAssembly 模块中的表示。

## 导入

为了从另一个 WebAssembly 模块或 JavaScript 模块导入一个函数，我们需要在 WebAssembly 模块中导入这些函数。导入部分是我们将所有外部依赖项导入 WebAssembly 模块的地方。

现在，让我们想象一个名为 `jsAdd` 的 JavaScript 模块导出了一个函数。我们可以使用 `import` 关键字导入 `jsAdd` 函数。创建一个名为 `jsAdd.wat` 的文件，并向其中添加以下内容：

```rs
(module
    (func $i (import "imports" "jsAdd") (param i32))
)
```

在这里，我们使用 `func` 关键字定义了一个函数，后面跟着函数的名称，`$i`。我们使用 `$i` 在 WebAssembly 模块内部调用该函数。然后，我们有了 `import` 关键字。`import` 关键字后面跟着模块名称。这里的模块名称指的是 JavaScript 模块，然后是我们要从 JavaScript 模块中导入的函数名称。

最后，我们有 `param`。由于 WebAssembly 模块是强类型的，我们必须在函数定义中定义输入参数和返回类型。

让我们使用 WABT 的 `wat2wasm` 将 WebAssembly 文本格式转换为以下命令的 WebAssembly 模块：

```rs
$ /path/to/wabt/bin/wat2wasm jsAdd.wat
```

让我们使用 `hexdump` 工具分析生成的字节码：

```rs
$ hexdump jsAdd.wasm
0000000 00 61 73 6d 01 00 00 00 01 05 01 60 01 7f 00 02
0000010 11 01 07 69 6d 70 6f 72 74 73 05 6a 73 41 64 64
0000020 00 00
0000022
```

二进制文件由导入部分组成，从索引 16 开始。导入部分以 `02` 开始，因为导入部分的唯一部分索引是 `02`。之后，我们有 `11`，表示二进制中导入部分的大小。下一个位表示导入的数量，`01`。

然后，我们有导入的定义。`07` 这里表示导入函数的长度。接下来的七个位表示导入模块的名称。下一个位表示函数名称的长度，`05`，接下来的五个位表示函数名称。最后，我们有索引的类型和签名：

```rs
; Other information
; section "Import" (2)
000000f: 02                          ; section code
0000010: 11                          ; section size
0000011: 01                          ; num imports
; import header 0
0000012: 07                          ; string length
0000013: 696d 706f 7274 73           imports  ; import
  module name
000001a: 05                          ; string length
000001b: 6a73 4164 64         jsAdd  ; import field name
0000020: 00                          ; import kind
0000021: 00                          ; import signature
  index
```

现在，你可以像在 WebAssembly 模块内部的其他函数一样使用 `$i` 标识符调用 `jsAdd` 函数。

我们已经探讨了如何在 WebAssembly 模块内部定义导入和导出部分，以及它们如何帮助导入和导出函数。现在，让我们探讨如何导入和导出 WebAssembly 模块中的值。

# 全局变量

全局部分是我们可以在 WebAssembly 模块中导入和导出值的地方。在 WebAssembly 模块中，你可以从 JavaScript 导入可变或不可变值。此外，WebAssembly 还支持 `wasmValue`，这是 WebAssembly 模块内部的内部不可变值。

让我们创建一个名为 `globals.wat` 的文件，并将以下内容添加到其中：

```rs
$ touch globals.wat
(module
     (global $mutableValue (import "js" "mutableGlobal")
       (mut i32))
     (global $immutableValue (import "js"
       "immutableGlobal") i32)
     (global $wasmValue i32 (i32.const 10))
     (func (export "getWasmValue") (result i32)
        (global.get $wasmValue))
     (func (export "getMutableValue") (result i32)
        (global.get $mutableValue))
     (func (export "getImmutableValue") (result i32)
        (global.get $immutableValue))
     (func (export "setMutableValue") (param $v i32)
        (global.set $mutableValue
            (local.get $v)))
)
```

我们创建了一个模块（`module`）和三个全局变量：

+   `$mutableValue` – 这个值是从 `js` JavaScript 模块和 `mutableGlobal` 全局变量导入的。我们还定义全局变量为 `mut i32` 类型。

+   `$immutableValue` – 这个值是从 `js` JavaScript 模块和 `immutableGlobal` 全局变量导入的。我们还定义全局变量为 `i32` 类型。

+   `$wasmValue` – 这是一个全局常量。我们定义 `global` 关键字后跟全局变量的名称 `$wasmValue`，然后是 `i32` 类型，最后是实际值（`i32.const 10`）。

    注意

    `$wasmValue` 是不可变的，不能导出到外部世界。

然后，我们有一组帮助获取和设置全局变量的函数。`getWasmValue`、`getImmutableValue` 和 `getMutableValue` 分别获取 `wasmValue` 全局常量、`immutableValue` 全局常量和 `mutableValue` 全局变量的值。

最后，一个将 `mutableValue` 设置为新值的函数是 `setMutableValue`。`setMutableValue` 接收 `param $v` 参数，将值设置为 `$mutableValue`。

使用 WABT 将 WebAssembly 文本格式转换为 WebAssembly 模块，可以使用以下命令：

```rs
$ /path/to/wabt/bin/wat2wasm globals.wat
```

创建一个包含以下内容的 `globals.html` 文件：

```rs
// globals.html
<html>
    <head> </head>
    <body>
        <script>
            async function run() {  }
            run()
        </script>
    </body>
</html>
```

让我们在 `<script>` 标签内定义 `run` 函数。

`WebAssembly.Global`对象代表一个全局变量实例，可以从 JavaScript 访问，并在一个或多个`WebAssembly.Module`实例之间导入/导出。`WebAssembly.Global`构造函数期望一个描述符和值。描述符定义了全局变量的类型和可变性：

注意

这个全局变量构造函数提供了一个选项，可以动态链接多个 WebAssembly 模块。

```rs
let immutableGlobal = new WebAssembly.Global({value:'i32',
  mutable:false}, 1000)
let mutableGlobal = new WebAssembly.Global({value:'i32',
  mutable:true}, 0)
```

我们使用`WebAssembly.Global`构造函数创建两个全局值。它们是`immutableGlobal`和`mutableGlobal`。前者是`mutable:false`，而后者是`mutable:true`。因此，我们可以使用`mutableGlobal.value`更改后者的值，但不能更改前者。如果我们尝试更改`immutableGlobal`的值，我们将收到一个错误：

```rs
mutableGlobal.value = 1337  // valid.
immutableGlobal.value = 7331 // Error
```

之后，我们获取`globals.wasm` WebAssembly 模块。然后，我们使用响应和`arrayBuffer`以及`WebAssembly.instantiate`构造函数实例化`arrayBuffer`。此外，`WebAssembly.instantiate`构造函数接受`importsObject`。我们可以通过`importsObject`发送 JavaScript 模块：

```rs
const response = await fetch('./globals.wasm')
const bytes = await response.arrayBuffer()
const wasm = await WebAssembly.instantiate(bytes, { js: {
  mutableGlobal, immutableGlobal } })
```

在这种情况下，我们发送了`js`模块以及`mutableGlobal`和`immutableGlobal`值。`wasm`变量现在持有 WebAssembly 模块。我们调用`wasm.instance.exports`以获取 WebAssembly 模块中所有导出的函数：

```rs
const {
    getWasmValue,
    getMutableValue,
    setMutableValue,
    getImmutableValue
} =  wasm.instance.exports
```

`getWasmValue`、`getMutableValue`、`setMutableValue`和`getImmutableValue`是 WebAssembly 模块导出的函数。

`getWasmValue`函数返回 WebAssembly 模块内`wasmValue`的值：

```rs
console.log(getWasmValue()) // 10
```

`getMutableValue`和`setMutableValue`函数返回并设置在 JavaScript 中定义并传递到 WebAssembly 模块中的`mutableGlobal`字段：

```rs
console.log(getMutableValue()) // 1337
setMutableValue(1338)
console.log(getMutableValue()) // 1338
```

最后，我们使用`getImmutableValue`函数获取不可变值：

```rs
console.log(getImmutableValue()) // 1000
```

让我们在浏览器中使用以下命令运行一个示例：

```rs
$ python -m http.server
```

现在，启动 URL `http://localhost:8000/globals.html`并打开开发者工具。

WebAssembly 二进制包含一个导入部分。导入部分有一个唯一的标识符`02`，后面是部分的长度，为`2b`（十进制为 43）。接下来的 43 位代表导入部分。

在`000015`索引处的`02`代表导入的数量。然后，我们有两个部分定义了导入的全局函数：

```rs

; section "Import" (2)
0000013: 02                          ; section code
0000014: 2b                          ; section size
0000015: 02                          ; num imports
```

每个全局段由模块字符串长度和模块名称组成，接着是函数字符串长度和函数名称。最后，它包含导入的类型、变量类型和可变性：

```rs

; import header 0
0000016: 02                          ; string length
0000017: 6a73                    js  ; import module name
0000019: 0d                          ; string length
000001a: 6d75 7461 626c 6547 6c6f 6261 6c
         mutableGlobal  ; import field name
0000027: 03                          ; import kind
0000028: 7f                          ; i32
0000029: 01                          ; global mutability
  ; import header 1
000002a: 02                          ; string length
000002b: 6a73                    js  ; import module name
000002d: 0f                          ; string length
000002e: 696d 6d75 7461 626c 6547 6c6f 6261 6c
    immutableGlobal  ; import field name
000003d: 03                          ; import kind
000003e: 7f                          ; i32
000003f: 00                          ; global mutability
```

之后，我们有`Global`部分。`Global`部分具有唯一的部分 ID `6`。下一个位定义了`Global`部分的大小，为`06`。

之后，我们有可用的全局变量数量。全局变量的数量是`01`。这是因为其他两个全局变量被导入。类型、可变性和值是接下来的 4 个字节：

```rs
; section "Global" (6)
0000047: 06                          ; section code
0000048: 06                          ; section size
0000049: 01                          ; num globals
000004a: 7f                          ; i32
000004b: 00                          ; global mutability
000004c: 41                          ; i32.const
000004d: 0a                          ; i32 literal
000004e: 0b                          ; end
```

代码部分内的第一个`function`体如下所示：

```rs
; function body 0
000009c: 04                          ; func body size
000009d: 00                          ; local decl count
000009e: 23                          ; global.get
000009f: 02                          ; global index
00000a0: 0b                          ; end
```

`function` 的长度为四位。前两位 `00` 表示该函数没有局部声明。接下来的 `23` 是获取全局值的操作码。接下来的 `02` 定义了全局值的索引。尽管前面的全局部分指定只有一个全局值，但整个模块会考虑导入的全局值。由于有两个导入的全局值，我们在导入的全局值之后索引局部全局值。因此，`$wasmValue` 全局值的索引为 3。最后，我们使用 `0b` 操作码结束函数代码。同样，第二个和第三个函数体定义了如何获取其他两个导入的全局值：

```rs
; function body 3
00000ab: 06                          ; func body size
00000ac: 00                          ; local decl count
00000ad: 20                          ; local.get
00000ae: 00                          ; local index
00000af: 24                          ; global.set
00000b0: 00                          ; global index
00000b1: 0b                          ; end
```

在函数体 4 中，我们使用 `global.set` 设置全局值，其操作码为 `24`。

我们已经探讨了如何在 WebAssembly 模块中导入和导出值。现在，让我们探索 WebAssembly 模块中的特殊 `start` 函数。

# 起始

Start 是一个特殊函数，它在 WebAssembly 模块初始化后运行。让我们使用与全局变量相同的示例。我们将以下内容添加到 `globals.wat` 文件中：

```rs
(module
    ; Code is elided
    (func $initMutableValue
          (global.set $mutableValue
               (i32.const 200))) 
     (start $initMutableValue)
)
```

我们定义了 `initMutableValue` 函数，该函数将 `mutableValue` 设置为 `200`。之后，我们添加一个起始块，该块以 `startkeyword` 开头，后跟函数的名称。

注意

在起始处引用的函数不应返回任何值。

让我们使用 WABT 将 WebAssembly 文本格式转换为 WebAssembly 模块，以下命令：

```rs
$ /path/to/wabt/bin/wat2wasm globals.wat
```

使用以下命令在浏览器中运行示例：

```rs
$ python -m http.server
```

现在，启动 URL `http://localhost:8000/globals.html` 并打开开发者工具。

起始函数与其他函数类似，但它们没有被分类到任何类型中。类型可能在函数初始化时初始化，也可能不初始化。WebAssembly 模块的起始部分指向一个函数索引（函数部分在函数组件内的位置索引）。

起始函数的节 ID 为 `8`。解码后，起始函数表示模块的起始组件：

```rs
; section "Start" (8)
0000085: 08                          ; section code
0000086: 01                          ; section size
0000087: 03                          ; start func index
```

注意

目前，工具如 webpack 不支持 `start` 函数。起始部分被重写为一个普通函数，然后当打包器本身初始化 JavaScript 时调用该函数。

`start` 是一个有趣且有用的函数，它允许在模块初始化时设置一些值，以防止模块可能引起的不必要的副作用。现在，让我们探索内存部分。内存部分负责在 JavaScript 和 WebAssembly 之间传输内存。

# 内存

在 JavaScript 和 WebAssembly 之间传输数据是一个昂贵的操作。为了减少 JavaScript 和 WebAssembly 模块之间的数据传输，WebAssembly 使用 `sharedArrayBuffer`。使用 `sharedArrayBuffer`，JavaScript 和 WebAssembly 模块都可以访问相同的内存，并使用它来在两者之间共享数据。

WebAssembly 模块的内存部分是一个线性内存的向量。线性内存模型是一种内存寻址技术，其中内存组织在一个单一的连续地址空间中。它也被称为平面内存模型。线性内存模型使得理解、编程和表示内存变得更加容易。但是，线性内存模型存在一个巨大的缺点，即内存中元素重新排列的执行时间很高，并且会浪费内存空间。在这里，内存代表了一个未解释数据的原始字节的向量。它们使用可调整大小的数组缓冲区来存储内存的原始字节。我们使用 `sharedArrayBuffers` 来定义和维护这个内存。

注意

重要的是要注意，这个内存可以通过 JavaScript 和 WebAssembly 访问和修改。

我们使用 `WebAssembly.Memory()` 构造函数来分配内存。构造函数可以接受一个参数，用于定义内存的初始值和最大值，如下所示：

```rs
$ touch memory.html
$ vi memory.html
let memory = new WebAssembly.Memory({initial: 10, maximum: 100})
```

在这里，我们定义 `WebAssembly.Memory` 具有初始内存 `10` 和最大内存 `100`。然后，我们使用以下代码实例化 WebAssembly 模块：

```rs
const response = await fetch('./memory.wasm')
const bytes = await response.arrayBuffer()
const wasm = await WebAssembly.instantiate(bytes, { js: { memory } })
```

与全局示例类似，这里我们传递 `importObject`，它接受 `js` 模块和内存对象。

让我们创建一个名为 `memory.wat` 的新文件，并将以下内容添加到其中：

```rs
(module
     (memory (import "js" "memory") 1)
     (func (export "sum") (param $ptr i32) (param $len i32)
       (result i32)
          (local $end i32)
          (local $sum i32)
          (local.set $end (i32.add (local.get $ptr)
            (i32.mul (local.get $len) (i32.const 4))))
          (block $break (loop $top
               (br_if $break (i32.eq (local.get $ptr)
               (local.get $end)))
               (local.set $sum (i32.add (local.get $sum)
                 (i32.load (local.get $ptr))))
               (local.set $ptr (i32.add (local.get $ptr)
                 (i32.const 4)))
               (br $top)
          ))
          (local.get $sum)
     )
)
```

在模块内部，我们使用名为 `memory` 的名称从 `js` 模块导入内存。之后，我们定义一个名为 sum 的函数并将该函数导出至模块外部。该函数接受两个参数作为输入，并返回一个 i32 类型的输出。第一个参数命名为 `$ptr`，它是一个指向 `sharedArrayBuffer` 中值所在索引的指针。下一个参数是 `$len`，它定义了共享内存的长度。

然后，我们创建两个局部变量，`$end` 和 `$sum`。首先，我们将 `$end` 设置为 `$ptr` 加上 `$len` 的四倍值。然后，我们创建一个块并开始一个循环。循环在 `$end` 的值等于 `$ptr` 的值时结束。然后，我们将 `$sum` 的值通过将现有的 `$sum` 值与 `$ptr` 的值相加来设置。然后，我们将 `$ptr` 增加到下一个值。最后，我们退出循环并返回 `$sum`。

以下代码与 JavaScript 中的以下代码类似：

```rs
function sum(ptr, len) {
    let end = ptr + (len * 4)
    let tmp = 0
    while (ptr < end) {
        tmp = memory[ptr]
        ptr = ptr + 4
    }
    return tmp;
}
```

让我们回到 `memory.html` 并初始化缓冲区：

```rs
let i32Arr = new Uint32Array(memory.buffer)
for (var i = 0; i < 50; i++) {
    i32Arr[i] = i * i * i
}
```

我们使用 `Uint32Array` 和我们创建的内存对象创建一个无符号数组。然后，我们将从 1 到 50 的数字的立方填充到数组缓冲区中：

```rs
var sum = wasm.instance.exports.sum(0, 50)
console.log(sum) // 1500625
```

最后，我们在 WebAssembly 模块内部调用 sum 函数，并要求它提供从 0 开始到长度为 50 的共享数组缓冲区中所有立方数的总和。

让我们使用 WABT 将 WebAssembly 文本格式转换为 WebAssembly 模块，使用以下命令：

```rs
$ /path/to/wabt/bin/wat2wasm memory.wat
```

让我们在浏览器中使用以下命令运行一个示例：

```rs
$ python -m http.server
```

现在，启动 URL `http://localhost:8000/globals.html` 并打开开发者工具。

当我们需要在两个世界之间传输大量数据时，内存部分非常有用。内存部分使得在 WebAssembly 和 JavaScript 世界之间定义、共享和访问内存变得更加容易。

# 摘要

在本章中，我们学习了 WebAssembly 模块中的导入、导出、启动和内存部分。我们看到了它们在 WebAssembly 模块中的结构和定义方式。每个部分都承载着一个特定的功能，理解、分析和调试 WebAssembly 模块时，这些部分是至关重要的。在下一章中，我们将探索 Binaryen。
