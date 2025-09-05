# 第十四章：外部函数接口

由于 Rust 是一种主要设计用于在服务器上工作的语言，而大多数服务器上的库（目前）都不是用 Rust 编写的，因此 Rust 应用程序能够利用用其他语言编写的库是有意义的。在本章中，我们将探讨如何做到这一点。

具体来说，我们将涵盖以下内容：

+   学习我们如何利用其他库

+   理解使用用其他语言编写的代码的陷阱。

+   在尽可能的范围内确保我们的代码保持安全。

就像之前的章节一样，源代码将可供你检查。你还将找到一个用 C 编写的、可在 Windows、macOS 和 Linux 上编译的小型库。这个库并不做什么，但它让你了解系统是如何工作的。其他库（如`ImageMagick`）也是以完全相同的方式工作的。

让我们开始吧！

# 介绍我们的简单库

库有三种类型：Windows 上的`.dll`（动态链接库）、`.so`（共享对象）和`.a`——`.a`和`.so`通常在 Unix 类型系统（包括 macOS）上找到。

我们的库非常简单；它作为一个计算库——你将值传递给正确的函数，然后返回结果。这不是火箭科学，但足以证明我们要做什么。

当使用外部库时，我们需要使用`unsafe`指令。Rust 无法控制外部库提供的内容，因此如果我们使用标准代码，编译器将不允许编译。

作为开发者，使用外部库必须谨慎处理。

# 三步程序

在你的 Rust 应用程序中使用库基本上有三个步骤：

1.  包含依赖项。

1.  编写使用库的代码。

1.  将你的应用程序构建为链接到库。

最困难的是第二个阶段，因为它需要编写代码、回调代码和其他这样的包装器来使用库。

# 包含依赖项

就像使用`Prelude`未提供的任何库一样，编译器必须知道库的存在。正如我们在第八章《Rust 应用程序生命周期》中所做的那样，我们通过在`Cargo.toml`文件中包含，让编译器知道预期一个外部库，如下所示：

```rs
[dependency] 
libc = "0.2.0" 
```

引号中的是库版本。这很有用，因为它使得编译的 Rust 应用程序只能运行在特定版本的库上，这保证了所需的代码将在库中。缺点是，为了始终确保库可用，编译的二进制文件需要附带该库。在这种情况下（以及大多数外部库的情况），需要添加`libc`。

我们还需要将以下行添加到将要调用函数的源文件中：

```rs
extern crate libc; 
```

# 创建代码。

这一部分的代码在`第十四章/firstexample`。

当我们处理来自我们应用程序外部的代码时，我们需要能够告诉编译器类似“嘿，看看，构建这段代码，并留下一个钩子，这个钩子可能存在也可能不存在，可能需要也可能不需要这些参数，但希望它能返回一些东西。”这就像给了拿着你签名支票的骗子一张空白支票，希望他们不会在上面填写并兑现！

在 Rust 中，我们通过使用链接指令并将函数放在`extern`块中来做到这一点。`extern`块内的代码调用库中持有的函数。它必须与库中函数的名称相同：

```rs
[link(name="mathlib")] 
extern 
 { 
     fn add_two_int_numbers(a: i32, b: i32) -> i32; 
} 
```

然后，可以使用以下方式访问此代码：

```rs
fn main() 
 { 
    let ans = unsafe { add_two_int_numbers(10,20) }; 
    println!("10 + 20 = {}", ans); 
} 
```

# 链接]的作用是什么？

这是一个指令，告诉编译器代码将要链接到一个名为引号内内容的库。你不需要在引号内使用`mathlib.dll`、`mathlib.so`或`mathlib.a`这样的名称，只需要没有扩展名的名称。

可用的三种不同类型的链接（称为模型，并在名称后的`kind`参数中定义）包括：*动态*、*静态*和*框架*（尽管后者仅适用于 macOS）。下表总结了它们各自的作用。在大多数情况下，使用的是`动态`类型。

| **类型** | **示例** | **注意事项** |
| --- | --- | --- |
| 动态 | `[link(name="foo")]` | 这是默认选项。编译的二进制文件会创建*钩子*，这些钩子将链接到平台安装的库版本。 |
| 静态 | `[link(name="foo", kind="static")]` | 这些是`.a`文件。当应用程序构建时，会创建二进制文件，但平台库文件不需要分发。 |
| 框架 | `[link(name="foo", kind="framework")]` | 仅限 macOS。这将是一个`.dylib`文件，其处理方式与动态库相同。 |

# 那有什么大不了的？这很简单！

虽然表面上使用外部库通过 FFI 并不是什么难事，但它确实带来了一系列问题。为什么即使我们引用库中的已知名称，我们还需要在块上使用`unsafe`注释？

就像我们一次又一次地看到的那样，与 Rust 相比，编译器为开发者做了很多工作，这在许多其他编译器中是看不到的。它确保线程安全，特定操作可以完成，缓冲区不会溢出，我们不会留下未分配的内存或尝试两次释放内存，以及许多其他确保我们的代码尽可能运行并保持稳定（从可靠性角度）的事情。

不幸的是，对于外部库，编译器所能做的只是期望从链接库中得到一些东西。线程可能会挂起或是不安全；没有保证，如果我传递了 6 和 0 给一个类似的除法函数，返回的将是一个数字，而且几乎任何其他事情都可能出错。

通过使用`unsafe`，我们向编译器承诺，当它链接代码时，它链接到的将正确绑定。

# 让我们扩展一下内容

`extern` 块可以包含 Rust 应用程序使用的库所需的方法（或方法数量）。

每当向 `extern` 块添加新函数时，测试被包含的函数总是一个好主意。这可以通过单元测试或通过将函数添加到 `extern` 块并在 `main` 中调用该函数来实现。

我们也可以有多个包含库函数的 Rust 源文件。

例如，在 `Source1.rs` 文件中进行修改：

```rs
//Source1.rs
[link(name="mylib")] 
extern  
{ 
    fn some_method(a: f32) → f32; 
    fn some_other_method(a: i32, b: f32, c: f64) → f64; 
} 
```

现在，在 `Source2.rs` 文件中进行修改：

```rs
[link(name="mylib")] 
extern 
 { 
    fn some_other_method(a: i32, b: f32, c: f64) → f64; 
    fn some_text_method() → String; 
} 
```

只要包含链接行，这就不会引起问题。

# 如果类型不匹配会发生什么？

当你在 32 位平台上构建库时，不能保证 `int` 的大小与 64 位平台上的 `int` 的大小相同。它们通常会是相同的，但并不能保证。一个简单的例子如下：

```rs
sizeof(char) == 1 
sizeof(short) <= sizeof(int) <= sizeof(long) <= sizeof(long long) 
```

因此，短整型可以与长整型大小相同！然而，更常见的情况是，`int` 将是平台字大小（在 32 位处理器上为 32 位，在 64 位处理器上为 64 位）。

浮点数的值更为严格，并符合 IEEE 754 标准。

如果 Rust 应用程序是在 64 位平台上构建的，而库是 32 位的，通常不会有问题。然而，如果情况相反，则可能发生溢出的情况。这种情况不太可能发生，但值得注意。

# 我们能否使事情更安全？

我们可以采取一种策略来尝试使事情稍微安全一些。

考虑我们的原始 `extern` 代码：

```rs
[link(name="mathlib")] 
extern 
 { 
     fn add_two_int_numbers(a: i32, b: i32) -> i32; 
} 
```

这段代码正在调用原始 C API，正如讨论的那样，任何对此的调用都必须标记为 `unsafe`。这是不安全的，因为调用被认为是 **低级** 的。

在编程语言方面，语言越低级，它就越接近处理器理解的语言（汇编器被认为是实际有用的最低级语言，除非直接将原始二进制代码插入内存位置）。在这里，我们正在以最低级别暴露库。

为了使调用更安全，我们使用了一种称为 **包装** 的方法。

# 包装器

当使用为其他语言设计的库时，包装器非常常见。它们通过暴露一个高级函数名来隐藏下面的真正方法。这个暴露的函数名通常被称为库接口 API。通过仅暴露高级函数名，Rust 能够将不安全的部分与其他部分隔离开来。

# 一个实际例子

库中的一个方法接受一个 `int` 值的向量来执行平均值、中位数和众数计算，然后返回一个包含这些值的 `float` 值数组。然而，我们需要首先验证这些值（本质上，测试数组不为空）并且有五个或更多的值。这将返回一个布尔值。

代码的不安全版本将是：

```rs
[link(name="mathlib")] 
extern 
 { 
     fn check_mean_mode_median(a: Vec<i32>) -> bool; 
} 
```

我们可以非常简单地为此创建一个包装器：

```rs
pub fn calc_mean_mode_median_check(a:Vec<int32>) -> bool 
 { 
    unsafe 
 { 
        check_mean_mode_median(a) == 0; 
    } 
} 
```

我们将安全函数暴露给代码，并隐藏（包装）不安全的部分。一旦我们返回一个 true 值，我们就知道数据可以进行计算。

现在，这是一段相当无意义的代码（它只是一个测试，以确保我们在向量中有正确的参数数量）。让我们修改这个包装器，使其返回一个`Vec<f32>`，它将包含`-999f`、`-999f`或`-999f`，如果检查失败，或者包含向量的平均值、中位数和众数。

然而，问题是原始库是用 C 编写的，所以我们需要将结果作为数组获取，然后将其放入向量中。

第一部分是进行第一次检查：

```rs
pub fn mean_mode_median(a: Vec<int32>) -> Vec<f32> 
{ 
    // we can have this result outside of the unsafe as it is a guaranteed parameter 
    // it must be mutable as it is used to store the result if the result is returned 
    let mut result = vec![-999f32, -999f32, -999f32]; 
    unsafe 
 { 
        if check_mean_mode_median(a) != 0 
 { 
             return result; 
        } 
 else 
 { 
             let res = calc_mean_median_mode(a); 

result = res.to_vec(); 
             return result; 

        } 
    } 
} 
```

不仅我们现在只有一个对外部库的调用，我们还保证了编译器所需的保证。

# 访问全局变量

在 C 库中，经常会有用于版本细节和特定构建代码等用途的全局变量。Rust 可以以与其他变量类似的方式访问这些变量：

```rs
extern crate libc; 
#[link(name = "mathlib")] 
extern { 
    static code_version: libc::c_int; 
} 
fn main() { 
    println!("You have mathlib version {} installed.", code_version as i32); 
} 
```

# 自己清理

虽然数学库是一个非常简单的例子，但有时你可能需要使用返回大量数据块的库（例如，如果你创建了一个用于与`ImageMagick`（一个常用且功能强大的图形库）一起工作的包装器）。当库返回时，结果会被传递给 Rust 应用程序，你需要手动释放。

为了帮助你，Rust 提供了`Drop`特质。

# 丢弃它！

`Drop`特质是一个非常简单的特质：

```rs
pub trait Drop  
{ 
    fn drop(&mut self); 
} 
```

就像所有特质一样，在使用特质之前，它需要一个`impl`：

```rs
struct FreeMemory; 
impl Drop for FreeMemory 
{ 
     fn drop(&mut self); 
} 
```

在这一点上，我们调用我们的`pub fn`，它从`ImageMagick`返回一个数据块。一旦我们用完那个内存块，我们必须释放它。我们将在一个名为`graphics_block`的变量中存储数据。要释放`graphics_block`中的块，我们使用：

```rs
graphics_block  = FreeMemory; 
```

当`graphics_block`超出作用域时，内存将被释放。

值得指出的是，`panic!`在回滚内存时会调用`drop`。因此，如果你在`drop`中有`panic!`，那么它很可能会中止。

# 在 FFI 中监控外部进程

在你使用计算机的时间里，你无疑会看到以下这样的图像：

![](img/00101.jpeg)

这些进度条以类似的方式工作。比如说，你有一个有五个相等部分的进程，或者你正在从互联网下载文件。随着部分完成或下载的代码量增加，条形图和百分比会使用一种称为**回调**的编程技术更新。

回调的实现方式取决于所使用的语言。例如，在事件驱动的语言中，进程将发出信号或生成接收者监听的事件。当接收到信号/事件时，用户界面会更新。

Rust 没有不同；它能够在使用 FFI 时使用回调。Rust 能够处理同步和异步回调。还可以将回调定位到 Rust 对象。

# 定位同步回调

同步回调是最容易定位的，因为它们通常总是在同一个线程上。因此，我们不必处理代码比平时更不安全的情况，这在异步回调中通常是常见的情况。

这一部分的代码在 `Chapter 14/synccallback` 中。Linux、macOS 和 Windows 的构建说明包含在源示例中。

首先，让我们处理 Rust 代码的这一部分。在这里，我们有三个部分：

1.  回调函数本身：

```rs
extern fn my_callback(percent: i32) 
 { 
     println!("Process is now {}% complete", percent); 
} 
```

1.  调用外部代码：

```rs
[link(name="external_lib")] 
extern 
 { 
     fn register_callback(call: extern fn(i32)) -> i32; 
     fn do_callback_trigger(); 
} 
```

1.  启动代码：

```rs
fn main() 
 { 
    unsafe 
 { 
         register_callback(my_callback); 
         do_callback_trigger(); 
    } 
} 
```

`register_callback(my_callback)` 和 `fn register_callback(call: extern fn(i32)) ->→ i32;` 在第一眼看起来可能很奇怪。在正常的函数调用中，花括号内的参数被传递到接收函数中，然后接收函数对它们进行处理。

在这里，我们正在将一个函数作为参数传递，这实际上是不可以做的（或者至少不应该）。然而，回调是不同的，因为函数由于 `extern` 修饰符的存在，被视为一个指针，它将外部库返回的值作为自己的参数。

# 定位 Rust 对象

在最后一个例子中，我们有一个监听单个 `int` 的回调。但是，如果我们想监听外部库中的复杂对象（例如，一个结构体）会发生什么呢？我们不能返回一个结构体，但我们可以有一个可以映射到回调的 Rust 对象。

这比同步回调稍微复杂一些：

1.  创建将映射到我们感兴趣的外部结构体的结构体：

```rs
#[repr(C)] // this is a name used within the extern in (2) 
struct MyObject 
 { 
    a: i32, 
    // and anything else you want to get back from the library 
    // just make sure you add them into the call back 
} 
```

1.  创建回调；`result` 是指向可变 `myobject` 的指针：

```rs
extern "C" fn callback(result: *mut MyObject, a: i32)  
{ 
     unsafe 
     { 
          (*result).a = a; 
     } 
} 
```

1.  创建库的 `extern` 函数：

```rs
#[link(name="external_lib")] 
extern 
 { 
    fn register_callback(result: mut MyObject, cback: extern fn(mut MyObject, i32)); 
    fn start_callback(); 
} 
```

1.  创建调用代码：

```rs
fn main() 
 { 
    // we need to create an object for the callback 
    let mystruct = Box::new (MyObject{a: 5i32}); 
    unsafe { 
        register_callback(&mut *mystruct, callback); 
        start_callback(); 
    } 
} 
```

# 从其他语言调用 Rust

Rust 也可以从其他语言中被调用，这是一个简单的过程。唯一的限制是使用的名称必须是未解析的。如果你还记得 第八章，“Rust 应用程序生命周期”，当你使用泛型时，编译器会生成必要的代码以确保链接器正常工作。它是通过混淆名称来做到这一点的，以确保在需要时编译和调用正确的代码。

解析（Unmangling）是相反的过程；它保留正在使用的函数的名称：

```rs
#[no_mangle] 
pub extern fn hello_world() -> *const u8 { 
    "Hello, world!\0".as_ptr() 
} 
```

这可以从你自己的（非 Rust）应用程序中调用。

# 处理未知情况

C 开发者并不总是在具有 *强类型* 的函数之间传递参数；相反，他们传递 `void*` 类型。然后在接收函数中将其转换为某种具体类型。从某种意义上说，这与在函数之间传递泛型类型非常相似。

如果你想访问一个以 `void*` 作为参数类型的库中的函数，这些必须以不同的方式处理。

例如，C 函数可能是：

```rs
void output_data(void *data); 
void transformed_data(void *data); 
```

由于 Rust 中没有与 `void*` 相同的东西，我们需要使用一个可变指针：

```rs
extern crate libc; 
extern "C" 
 { 
    pub fn output_data(arg: *mut libc::c_void); 
    pub fn transformed_data(arg: *mut libc::c_void); 
} 
```

这将完成这项工作。

# C 结构体

在本章的早期，我们使用 `struct` 作为参数。在 C 中，没有阻止开发者将结构体作为参数传递：

```rs
struct MyStruct; 
struct MyOtherStruct; 
void pass_struct(struct MyStruct *arg); 
void pass_struct2(struct MyOtherStruct *arg2); 
```

`MyStruct` 和 `MyOtherStruct` 被称为不透明结构体。名称是公开的，但私有部分不是。

在 Rust 中处理 `struct` 并不像你最初想象的那么简单，但也不是那么困难。唯一的区别是我们在与 C 库接口时使用一个空的 `enum` 而不是 `struct`。这创建了一个不透明的类型，用于存储来自 C 不透明类型的信息。由于 `enum` 是空的，我们无法实例化它，更重要的是，由于 `MyStruct` 和 `MyOtherStruct` 并不相同，我们有了类型安全，因此不能将它们混淆：

```rs
enum MyStruct {}; 
enum MyOtherStruct {}; 
extern "C" 
{ 
    pub fn pass_struct(arg: *mut MyStruct); 
    pub fn pass_struct2(arg: *mut MyOtherStruct); 
} 
```

# 摘要

在本章中，我们介绍的内容不仅使 Rust 成为开发应用的优秀选择，而且通过使用非 Rust 库，也使其成为一种灵活且强大的语言。存在一些陷阱（例如需要使用 unsafe 和必须非常小心处理 panic! 代码），但优点远多于缺点。

对于本文的用途，`.dll` 仅用于 Windows。.NET 框架也使用 `.dll` 文件，如果它们不包含任何特定于 Windows 的内容，也可以在 macOS 和 Linux 上使用。
