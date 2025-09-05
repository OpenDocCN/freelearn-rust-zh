# 第十二章：*第十二章*：编写 unsafe Rust 和 FFI

在上一章中，我们学习了内置在 Rust 标准库中的网络原语，并看到了如何编写通过 TCP 和 UDP 通信的程序。在本章中，我们将通过介绍一些与 **unsafe Rust** 和 **外部函数接口**（**FFIs**）相关的高级主题来结束本书。

我们已经看到 Rust 编译器如何强制执行内存和线程安全的所有权规则。虽然这大多数时候都是一种祝福，但可能存在您想要实现新的低级数据结构或调用用其他语言编写的外部程序的情况。或者，您可能想要执行 Rust 编译器禁止的其他操作，例如取消引用原始指针、修改静态变量或处理未初始化的内存。您是否想过 Rust 标准库本身是如何进行系统调用来管理资源的，当系统调用涉及处理原始指针时？答案在于理解 unsafe Rust 和 FFIs。

在本章中，我们首先将探讨为什么以及如何使用 unsafe Rust 代码。然后，我们将介绍 FFIs 的基础知识，并讨论在使用它们时的特殊注意事项。我们还将编写调用 C 函数的 Rust 代码，以及调用 Rust 函数的 C 程序。

我们将按以下顺序介绍这些主题：

+   介绍 unsafe Rust

+   介绍 FFIs

+   复习安全 FFIs 的指南

+   从 C 调用 Rust（项目）

+   理解 ABI

到本章结束时，您将学会何时以及如何使用 unsafe Rust。您将学习如何通过 FFIs 将 Rust 与其他编程语言接口，并学习如何与之交互。您还将了解一些高级主题的概述，例如 **应用程序二进制接口**（**ABIs**）、条件编译、数据布局约定以及向链接器提供指令。了解这些内容将有助于构建针对不同目标平台的 Rust 二进制文件，以及将 Rust 代码与用其他编程语言编写的代码链接。

# 技术要求

使用以下命令验证 `rustup`、`rustc` 和 `cargo` 是否已正确安装：

```rs
rustup --version
rustc --version 
cargo --version
```

由于本章涉及编译 C 代码和生成二进制文件，您需要在您的开发机器上设置 C 开发环境。设置完成后，运行以下命令以验证安装是否成功：

```rs
gcc --version
```

如果此命令无法成功执行，请重新检查您的安装。

注意

建议在 Windows 平台上开发的人员使用 Linux 虚拟机来尝试本章中的代码。

本节中的代码已在 Ubuntu 20.04（LTS）x64 上进行测试，并应在任何其他 Linux 变体上工作。

本章代码的 Git 仓库可以在 [`github.com/PacktPublishing/Practical-System-Programming-for-Rust-Developers/tree/master/Chapter12`](https://github.com/PacktPublishing/Practical-System-Programming-for-Rust-Developers/tree/master/Chapter12) 找到。

# 介绍不安全 Rust

到目前为止，在这本书中，我们看到了并使用了 Rust 语言，它在编译时强制执行内存和类型安全，并防止各种未定义的行为，例如内存溢出、空或无效指针构造以及数据竞争。这是 *安全* Rust。实际上，Rust 标准库为我们提供了良好的工具和实用程序来编写安全、惯用的 Rust，并有助于保持程序安全（以及您的心情！）。

但在某些情况下，编译器可能会 *妨碍*。Rust 编译器对代码进行保守的静态分析（这意味着 Rust 编译器不介意生成一些假阳性并拒绝有效代码，只要它不让坏代码通过）。您作为程序员知道某段代码是安全的，但编译器认为它是危险的，因此拒绝此代码。这包括系统调用、类型转换和直接操作内存指针等操作，这些操作用于开发几类系统软件。

另一个例子是在嵌入式系统中，寄存器通过固定的内存地址访问并需要指针解引用。因此，为了启用此类操作，Rust 语言提供了 `unsafe` 关键字。对于 Rust 作为系统编程语言，它对于使程序员能够编写低级代码以直接与操作系统接口至关重要，如果需要，可以绕过 Rust 标准库。*这是不安全 Rust*。这是 Rust 语言中不遵循借用检查器规则的部分。

不安全 Rust 可以被视为安全 Rust 的超集。它是超集，因为它允许您做所有在标准 Rust 中可以做的事情，但您可以做更多被 Rust 编译器禁止的事情。实际上，Rust 的编译器和标准库都包含了精心编写的 `unsafe` Rust 代码。

## 您如何区分安全和不安全 Rust 代码？

Rust 提供了一种方便且直观的机制，可以使用 `unsafe` 关键字将代码块封装在 `unsafe` 块中。尝试以下代码：

```rs
fn main() {
    let num = 23;
    let borrowed_num = &num; // immutable reference to num
    let raw_ptr = borrowed_num as *const i32; // cast the 
    // reference borrowed_num to raw pointer
    assert!(*raw_ptr == 23);
}
```

使用 `cargo check`（或从 Rust playground IDE 运行）编译此代码。您将看到以下错误信息：

```rs
error[E0133]: dereference of raw pointer is unsafe and requires unsafe function or block
```

现在我们通过将原始指针的解引用封装在 `unsafe` 块中来修改代码：

```rs
fn main() {
    let num = 23;
    let borrowed_num = &num; // immutable reference to num
    let raw_ptr = borrowed_num as *const i32; // cast 
    // reference borrowed_num to raw pointer
    unsafe {
        assert!(*raw_ptr == 23);
    }
}
```

您将看到现在编译成功，尽管这段代码可能引发未定义的行为。这是因为一旦将某些代码封装在 `unsafe` 块中，编译器就期望程序员确保不安全代码的安全性。

现在我们来看看不安全 Rust 允许的操作类型。

## 不安全 Rust 中的操作

实际上，在*unsafe*类别中只有五个关键操作——解引用原始指针、处理可变静态变量、实现不安全特性和通过 FFI 接口调用外部函数，以及跨 FFI 边界共享联合结构体。

我们将在本节中查看前三个，在下一节中查看最后两个：

+   `*const T`是一个指针类型，对应于安全 Rust 中的`&T`（不可变引用类型），而`*mut T`是一个指针类型，对应于`&mut T`（安全 Rust 中的可变引用类型）。与 Rust 引用类型不同，这些原始指针可以同时具有对值的不可变和可变引用，或者同时有多个指针指向内存中的同一值。当这些指针超出作用域时，没有自动清理内存，这些指针也可以是 null 或指向无效的内存位置。Rust 提供的内存安全保证不适用于这些指针类型。下面将展示如何在`unsafe`块中定义和访问指针的示例：

    ```rs
    fn main() {
        let mut a_number = 5;
        // Create an immutable pointer to the value 5
        let raw_ptr1 = &a_number as *const i32;
        // Create a mutable pointer to the value 5
        let raw_ptr2 = &mut a_number as *mut i32;

        unsafe {
            println!("raw_ptr1 is: {}", *raw_ptr1);
            println!("raw_ptr2 is: {}", *raw_ptr2);
        }
    }
    ```

    您会注意到从这段代码中，我们通过从相应的不可变和可变引用类型进行转换，同时创建了同一值的不可变引用和可变引用。请注意，为了创建原始指针，我们不需要`unsafe`块，但需要解引用它们。这是因为解引用原始指针可能会导致不可预测的行为，因为借用检查器不负责验证其有效性或生命周期。

+   `unsafe`块。在下面的示例中，我们声明了一个可变静态变量，它使用默认值初始化线程数量。然后，在`main()`函数中，我们检查环境变量，如果指定了该变量，它将覆盖默认值。在静态变量中的值覆盖必须包含在`*unsafe*`块中：

    ```rs
    static mut THREAD_COUNT: u32 = 4;
    use std::env::var;
    fn change_thread_count(count: u32) {
        unsafe {
            THREAD_COUNT = count;
        }
    }
    fn main() {
        if let Some(thread_count) = 
            var("THREAD_COUNT").ok() {
            change_thread_count(thread_count.parse::
                <u32>()
                .unwrap());
        };
        unsafe {
            println!("Thread count is: {}", THREAD_COUNT);
        }
    }
    ```

    以下代码片段展示了可变静态变量的声明，`THREAD_COUNT`，初始化为`4`。当`main()`函数执行时，它会查找名为`THREAD_COUNT`的环境变量。如果找到`env`变量，它将调用`change_thread_count()`函数，在`unsafe`块中修改静态变量的值。然后`main()`函数在`unsafe`块中打印出该值。

+   `Send`或`Sync`特性。为了为原始指针实现这两个特性，我们必须使用不安全 Rust，如下所示：

    ```rs
    struct MyStruct(*mut u16);
    unsafe impl Send for MyStruct {}
    unsafe impl Sync for MyStruct {}
    ```

    `unsafe`关键字的原因是因为原始指针具有未跟踪的所有权，这随后成为程序员的职责来跟踪和管理。

与其他编程语言接口相关的不安全 Rust 有两个更多特性，我们将在下一节关于 FFI 的讨论中讨论。

# 介绍 FFI

在本节中，我们将了解 FFI 是什么，然后查看与 FFI 相关的两个不安全 Rust 特性。

为了理解 FFI，让我们看看以下两个示例：

+   有一个用于线性回归的 Rust 编写的快速机器学习算法。一个 Java 或 Python 开发者想要使用这个 Rust 库。这该如何实现？

+   你想在不用 Rust 标准库的情况下（这基本上意味着你想实现标准库中没有的功能或想改进现有功能）进行 Linux 系统调用。你将如何做？

尽管可能有其他方法可以解决这个问题，但一种流行的方法是使用 FFI。

在第一个例子中，你可以用 Java 或 Python 中定义的 FFI 包装 Rust 库。在第二个例子中，Rust 有一个关键字`extern`，可以用它来设置和调用 C 函数的 FFI。让我们看看第二个案例的例子：

```rs
use std::ffi::{CStr, CString};
use std::os::raw::c_char;
extern "C" {
    fn getenv(s: *const c_char) -> *mut c_char;
}
fn main() {
    let c1 = CString::new("MY_VAR").expect("Error");
    unsafe {
        println!("env got is {:?}", CStr::from_ptr(getenv(
            c1.as_ptr())));
    }
}
```

在这里，在`main()`函数中，我们正在调用外部 C 函数`getenv()`（而不是直接使用 Rust 标准库）来检索`MY_VAR`环境变量的值。`getenv()`函数接受一个`*const c_char`类型的参数作为输入。为了创建这种类型，我们首先实例化`CString`类型，传入环境变量的名称，然后使用`as_ptr()`方法将其转换为所需的函数输入参数类型。`getenv()`函数返回一个`*mut c_char`类型的值。为了将其转换为 Rust 兼容的类型，我们使用`Cstr::from_ptr()`函数。

注意这里的两个主要考虑因素：

+   我们在`extern "C"`块中指定对 C 函数的调用。此块包含我们想要调用的函数的签名。请注意，函数中的数据类型不是 Rust 数据类型，而是属于 C 的数据类型。

+   我们从 Rust 标准库中导入了一些模块——`std::ffi`和`std::os::raw`。`ffi`模块提供了与 FFI 绑定相关的实用函数和数据结构，这使得在非 Rust 接口之间进行数据映射变得更容易。我们使用`ffi`模块中的`CString`和`CStr`类型，在 C 之间传输 UTF-8 字符串。`os::raw`模块包含映射到 C 数据类型的平台特定类型，以便与 C 交互的 Rust 代码引用正确的类型。

现在，让我们使用以下命令运行程序：

```rs
MY_VAR="My custom value" cargo -v run --bin ffi
```

你将看到`MY_VAR`的值被打印到控制台。通过这种方式，我们已经成功使用外部 C 函数调用来检索环境变量的值。

回想一下，我们在前面的章节中学习了如何使用 Rust 标准库来获取和设置环境变量。现在我们已经做了类似的事情，但这次是使用 Rust FFI 接口调用 C 库函数。请注意，对 C 函数的调用被包含在一个`unsafe`块中。

到目前为止，我们已经看到了如何在 Rust 中调用 C 函数。稍后，在*从 C 调用 Rust（项目）*部分，我们将看到如何反过来操作，即从 C 调用 Rust 函数。

现在我们来看一下不安全的 Rust 的另一个特性，即定义和访问联合结构体的字段，以便通过 FFI 接口与 C 函数进行通信。

联合结构体是在 C 中使用的，并且不是内存安全的。这是因为在一个联合类型中，你可以将一个`union`的实例设置为其中一个不变量，并以另一个不变量的方式访问它。Rust 不直接在安全 Rust 中提供`union`作为类型。然而，Rust 在安全 Rust 中有一个称为`enum`的联合类型。让我们看看`union`的一个例子：

```rs
#[repr(C)]
union MyUnion {
    f1: u32,
    f2: f32,
}
fn main() {
    let float_num = MyUnion {f2: 2.0};
    let f = unsafe { float_num.f2 };
    println!("f is {:.3}",f);
}
```

在显示的代码中，我们首先使用`repr(C)`注解，这告诉编译器`MyUnion`联合结构体中字段的顺序、大小和对齐方式与 C 语言中预期的一致（我们将在*理解 ABI*部分中讨论更多关于`repr(C)`的内容）。然后我们定义联合结构体的两个不变量：一个是`u32`类型的整数，另一个是`f32`类型的浮点数。对于这个联合结构体的任何给定实例，只有一个这些不变量是有效的。在代码中，我们创建了这个联合结构体的一个实例，使用`float`不变量初始化它，然后从`unsafe`块中访问其值。

使用以下命令运行程序：

```rs
cargo run 
```

你将在终端上看到打印出`f is 2.000`的值。到目前为止，看起来是正确的。现在，让我们尝试将联合结构体作为整数而不是浮点类型来访问。为此，只需更改一行代码。定位以下行：

```rs
let f = unsafe { float_num.f2 };
```

改成以下内容：

```rs
let f = unsafe { float_num.f1 };
```

再次运行程序。这次，你不会得到错误，但你将看到像这样打印出无效值。原因是现在指向的内存位置中的值现在被解释为整数，尽管我们存储了一个浮点数值：

```rs
f is 1073741824
```

在 C 中使用联合结构体是危险的，除非做得非常小心，而 Rust 提供了在`unsafe Rust`中与联合结构体一起工作的能力。

到目前为止，你已经看到了`unsafe Rust`和 FFI 是什么。你也看到了调用`unsafe`和外部函数的例子。在下一节中，我们将讨论创建安全 FFI 接口的指南。

# 审查安全的 FFI 指南

在本节中，我们将探讨在使用 Rust 中的 FFI 与其他语言进行接口时需要记住的一些指南：

+   Rust 中的`extern`关键字本身是不安全的，此类调用必须从`unsafe`块中进行。

+   `#repr(C)`注解对于保持内存安全非常重要。我们之前已经看到了如何使用这个注解的例子。另一个需要注意的事项是，外部函数的参数或返回值应仅使用与 C 兼容的类型。在 Rust 中，与 C 兼容的类型包括整数、浮点数、`repr(C)`注解的结构体和指针。与 C 不兼容的 Rust 类型包括特质对象、动态大小类型和具有字段的枚举。有一些工具，如`rust-bindgen`和`cbindgen`，可以帮助生成 Rust 和 C 之间兼容的类型（有一些注意事项）。

+   `int` 和 `long`，这意味着这些类型的长度根据平台架构而变化。当与使用这些类型的 C 函数交互时，可以使用 Rust 标准库 `std::raw` 模块，该模块提供跨平台的可移植类型别名。`c_char` 和 `c_uint` 是我们在早期示例中使用的原始类型中的两个例子。除了标准库之外，`libc` crate 也为这些数据类型提供了这样的可移植类型别名。

+   **引用和指针**：由于 C 的指针类型和 Rust 的引用类型之间存在差异，Rust 代码在跨 FFI 边界工作时不应使用引用类型，而应使用指针类型。任何解引用指针类型的 Rust 代码必须在使用之前进行空检查。

+   为任何直接传递给外部代码的类型提供 `Drop` 特性。仅使用 `Copy` 类型跨 FFI 边界使用则更为安全。

+   `std::panic::catch_unwind` 或 `#[panic_handler]`（我们在 *第九章*，*管理* *并发性*）中看到）。这将确保 Rust 代码不会在不稳定状态下终止或返回。

+   **将 Rust 库暴露给外语**：将 Rust 库及其函数暴露给外语（如 Java、Python 或 Ruby）应仅通过 C 兼容的 API 完成。

这就结束了关于编写安全 FFI 接口的章节。在下一节中，我们将看到一个从 C 代码中使用 Rust 库的示例。

# 从 C 调用 Rust（项目）

在本节中，我们将演示构建一个 Rust 共享库（在 Linux 上具有 `.so` 扩展名）所需的设置，该库包含 FFI 接口，并从 C 程序中调用它。该 C 程序将是一个简单的程序，仅打印问候消息。示例故意保持简单，以便您（由于您不需要熟悉复杂的 C 语法）可以专注于涉及到的步骤，并便于在各种操作系统环境中验证此第一个 FFI 程序。

这里是我们将采取的步骤，以开发并测试一个 C 程序的工作示例，该程序使用 FFI 接口从 Rust 库中调用函数：

1.  创建一个新的 Cargo lib 项目。

1.  修改 `Cargo.toml` 以指定要构建共享库。

1.  在 Rust 中编写 FFI（以 C 兼容的 API 形式）：

1.  构建 Rust 共享库。

1.  验证 Rust 共享库是否已正确构建。

1.  创建一个调用 Rust 共享库中函数的 C 程序。

1.  指定 Rust 共享库路径构建 C 程序。

1.  设置 `LD_LIBRARY_PATH`。

1.  运行 C 程序。

让我们开始执行上述步骤：

1.  创建一个新的 cargo 项目：

    ```rs
    cargo new --lib ffi && cd ffi
    ```

1.  将以下内容添加到 `Cargo.toml` 中：

    ```rs
    [lib]
    name = "ffitest"
    crate-type = ["dylib"]
    ```

1.  在 `src/lib.rs` 中编写 FFI（以 C 兼容的 API 形式）：

    ```rs
    #[no_mangle]
    pub extern "C" fn see_ffi_in_action() {
        println!("Congrats! You have successfully invoked 
            Rust shared library from a C program");
    }
    ```

    `#[no_mangle]` 注解告诉 Rust 编译器，`see_ffi_in_action()` 函数应该可以被具有相同名称的外部程序访问。否则，默认情况下，Rust 编译器会修改它。

    函数使用了`extern "C"`关键字。如前所述，Rust 编译器使标记为`extern`的任何函数与 C 代码兼容。`extern "C"`中的`"C"`关键字表示目标平台上的标准 C 调用约定。在这个函数中，我们只是打印出问候语。

1.  使用以下命令从`ffi`文件夹构建 Rust 共享库：

    ```rs
    libffitest.so, created in the target/release directory.
    ```

1.  验证共享库是否已正确构建：

    ```rs
    nm command-line utility is used to examine binary files (including libraries and executables) and view the symbols in these object files. Here, we are checking whether the function that we have written is included in the shared library. You should see a result similar to this:

    ```

    （在 Mac 平台上为`.dylib`扩展名。）

    ```rs

    ```

1.  让我们创建一个 C 程序，调用我们构建的 Rust 共享库中的函数。在`ffi`项目文件夹的根目录中创建一个`rustffi.c`文件，并添加以下代码：

    ```rs
    #include "rustffi.h"
    int main(void) {
            see_ffi_in_action();
    }
    ```

    这是一个简单的 C 程序，它包含一个头文件，并有一个`main()`函数，该函数反过来调用一个`see_ffi_in_action()`函数。在这个时候，C 程序不知道这个函数在哪里。当我们构建二进制文件时，我们将提供这个信息给 C 编译器。现在让我们编写这个程序中提到的头文件。在 C 源文件相同的文件夹中创建一个`rustffi.h`文件，并包含以下内容：

    ```rs
    void see_ffi_in_action();
    ```

    此头文件声明了函数签名，表示此函数不返回任何值也不接受任何输入参数。

1.  使用以下命令从项目的根目录构建 C 二进制文件：

    ```rs
    gcc: Invokes the GCC compiler.`-Ltarget/release`: The `–L` flag specifies to the compiler to look for the shared library in the folder target/release.`-lffitest`: The `–l` flag tells the compiler that the name of the shared library is `ffitest`. Note that the actual library built is called `libffitest.so`, but the compiler knows that the `lib` prefix and `.so` suffix are part of the standard shared library name, so it is sufficient to specify `ffitest` for the `–l` flag.`rustffi.c`: This is the source file to be compiled.`-o ffitest`: Tells the compiler to generate the output executable with the name `ffitest`.
    ```

1.  设置`LD_LIBRARY_PATH`环境变量，在 Linux 中指定库将被搜索的路径：

    ```rs
    export LD_LIBRARY_PATH=$(rustc --print sysroot)/lib:target/release:$LD_ LIBRARY_PATH
    ```

1.  使用以下命令运行可执行文件：

    ```rs
    ./ffitest
    ```

你应该在终端上看到以下消息：

```rs
Congrats! You have successfully invoked Rust shared library from a C program
```

如果你已经走到这一步，恭喜你！

你已经使用 Rust 编写了一个包含具有 C 兼容 API 的函数的共享库。然后，你从 C 程序中调用了这个 Rust 库。这就是 FFI（Foreign Function Interface）的作用。

# 理解 ABI

本节简要介绍了 ABI 以及一些与 Rust 相关的（高级）特性，这些特性涉及条件编译选项、数据布局约定和链接选项。

**ABI**是一组约定和标准，编译器和链接器遵循这些约定和标准，用于函数调用约定以及指定数据布局（类型、对齐、偏移）。

要理解 ABI 的重要性，让我们通过 API 进行类比，API 是应用编程中众所周知的概念。当程序需要在源代码级别访问外部组件或库时，它会寻找外部组件暴露的 API 定义。外部组件可以是库或通过网络可访问的外部服务。API 指定了可以调用的函数名称，需要传递给函数调用的参数（包括它们的名称和数据类型），以及函数返回值的类型。

ABI 可以看作是 API 在二进制层面的等价物。编译器和链接器需要一种方式来指定调用程序如何在二进制对象文件中定位被调用函数，以及如何处理参数和返回值（参数的类型和顺序以及返回类型）。但与源代码不同，在生成的二进制文件的情况下，诸如整数长度、填充规则以及函数参数是存储在栈上还是寄存器中等细节会因平台架构（例如，x86、x64、AArch32）和操作系统（例如，Linux 和 Windows）而异。64 位操作系统可以为执行 32 位和 64 位二进制文件具有不同的 ABI。基于 Windows 的程序将不知道如何访问在 Linux 上构建的库，因为它们使用不同的 ABI。

虽然 ABIs 的研究本身就是一个专业化的主题，但了解 ABIs 的重要性以及 Rust 在编写代码时提供哪些功能来指定与 ABI 相关的参数就足够了。我们将涵盖以下内容 – *条件编译选项*、*数据布局约定*和*链接选项*：

+   `cfg`宏。以下是一些`cfg`选项的示例：

    ```rs
    #[cfg(target_arch = "x86_64")]  
    #[cfg(target_os = "linux")] 
    #[cfg(target_family = "windows")] 
    #[cfg(target_env = "gnu")] 
    #[cfg(target_pointer_width = "32")]
    ```

    这些注释被附加到函数声明中，如下例所示：

    ```rs
    // Only if target OS is Linux and architecture is x86, 
    // include this function in build 
    #[cfg(all(target_os = "linux", target_arch = "x86"))] 
    // all conditions must be true  
    fn do_something() { // ... }
    ```

    更多关于各种条件编译选项的详细信息可以在[`doc.rust-lang.org/reference/conditional-compilation.html`](https://doc.rust-lang.org/reference/conditional-compilation.html)找到。

+   `#[repr(Rust)]`。但如果需要通过 FFI 边界传递数据，则接受的标准是使用 C 的数据布局（注释为`#[repr(C)]`）。在这个布局中，字段顺序、大小和对齐方式与 C 程序中的方式相同。这对于确保数据在 FFI 边界上的兼容性非常重要。

    Rust 保证，如果将`#[repr(C)]`属性应用于结构体，则该结构体的布局将与平台在 C 中的表示兼容。有一些自动化工具，如`cbindgen`，可以帮助从 Rust 程序生成 C 数据布局。

+   `link`注释。以下是一个示例：

    ```rs
    #[link(name = "my_library")]
    extern {
        static a_c_function() -> c_int;
    }
    ```

    `#[link(...)]`属性用于指示链接器链接到`my_library`以解析符号。它指导 Rust 编译器如何链接到本地库。此注释还可以用于指定要链接的库的类型（*静态*或*动态*）。以下注释告诉`rustc`链接到名为`my_other_library`的*静态*库：

    ```rs
    #[link(name = "my_other_library", kind = "static")]
    ```

在本节中，我们了解了 ABI 是什么以及它的意义。我们还探讨了如何通过代码中的各种注释来指定对编译器和链接器的指令，例如目标平台、操作系统、数据布局和链接指令。

这就结束了本节。本节的目的是仅介绍与 ABI、FFI 和相关指令有关的一些高级主题。更多详情，请参阅以下链接：[`doc.rust-lang.org/nomicon/`](https://doc.rust-lang.org/nomicon/)。

# 摘要

在这一章中，我们回顾了不安全 Rust 的基础知识，并了解了安全 Rust 和不安全 Rust 之间的关键区别。我们看到了不安全 Rust 如何使我们能够执行在安全 Rust 中不被允许的操作，例如取消引用原始指针、访问或修改静态变量、与联合体一起工作、实现不安全特质以及调用外部函数。我们还探讨了什么是外函数接口，以及如何在 Rust 中编写一个。我们编写了一个从 Rust 调用 C 函数的示例。此外，在示例项目中，我们编写了一个 Rust 共享库，并从 C 程序中调用它。我们看到了如何在 Rust 中编写安全的 FFI 的指南。我们还查看了一些可以用来指定条件编译、数据布局和链接选项的 ABI 和注解。

有了这些，我们结束了这一章，也结束了这本书。

我感谢您与我一起踏上探索 Rust 系统编程世界的旅程，并祝愿您在进一步探索这个主题时一切顺利。
