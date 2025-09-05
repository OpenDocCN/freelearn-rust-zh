# 第五章：处理错误和其他结果

在每种编程语言中，处理错误始终是一个有趣的挑战。有许多风格可供选择：返回数值、异常（软件中断）、结果和选项类型等。每种方式都需要不同的架构，并对性能、可读性和可维护性有影响。Rust 的方法——就像许多函数式编程语言一样——基于将失败集成到常规工作流程中。这意味着无论返回值如何，错误都不是一个特殊情况，而是集成到处理中。`Option` 和 `Result` 是中心类型，允许返回结果以及错误。`panic!` 是一个额外的宏，在无法/不应继续时立即停止线程。

在本章中，我们将介绍一些基本的配方和架构，以有效地使用 Rust 的错误处理，使你的代码易于阅读、理解和维护。因此，在本章中，你可以期待学习以下配方：

+   负责任地恐慌

+   处理多个错误

+   与异常结果一起工作

+   无缝错误处理

+   自定义错误

+   弹性编程

+   与外部 crate 进行错误处理

+   在 Option 和 Result 之间移动

# 负责任地恐慌

有时，执行线程无法继续执行。这可能是由于无效的配置文件、无响应的同伴或服务器，或与操作系统相关的错误。Rust 有许多方法可以恐慌，无论是显式还是隐式。最普遍的一个可能是 `unwrap()` 对于多个 `Option` 类型及其相关类型，它在出错或 `None` 时会恐慌。然而，对于更复杂的程序，控制恐慌（例如，通过避免多个 `unwrap()` 调用和使用它的库）是至关重要的，而 `panic!` 宏支持这一点。

# 如何做到...

让我们看看我们如何可以控制多个 `panic!` 实例：

1.  使用 `cargo new panicking-responsibly --lib` 创建一个新的项目，并用 VS Code 打开它。

1.  打开 `src/lib.rs` 并将默认测试替换为常规、直接的恐慌实例：

```rs
#[cfg(test)]
mod tests {

    #[test]
    #[should_panic]
    fn test_regular_panic() {
        panic!();
    }
}
```

1.  还有许多其他停止程序的方法。让我们添加另一个 `test` 实例：

```rs
    #[test]
    #[should_panic]
    fn test_unwrap() {
        // panics if "None"
        None::<i32>.unwrap();
    }
```

1.  然而，这些恐慌都有一个通用的错误消息，这并不很有助于了解应用程序正在做什么。使用 `expect()` 可以让你提供一个错误消息来解释错误的起因：

```rs
    #[test]
    #[should_panic(expected = "Unwrap with a message")]
    fn test_expect() {
        None::<i32>.expect("Unwrap with a message");
    }
```

1.  `panic!` 宏提供了一种类似的方式来解释突然的停止：

```rs
    #[test]
    #[should_panic(expected = "Everything is lost!")]
    fn test_panic_message() {
        panic!("Everything is lost!");
    }

    #[test]
    #[should_panic(expected = "String formatting also works")]
    fn test_panic_format() {
        panic!("{} formatting also works.", "String");
    }
```

1.  宏也可以返回数值，这对于可以检查这些值的 Unix 类操作系统来说非常重要。添加另一个测试以返回一个整数代码来指示特定的失败：

```rs
    #[test]
    #[should_panic]
    fn test_panic_return_value() {
        panic!(42);
    }
```

1.  基于无效值停止程序的另一种很好的方法是使用 `assert!` 宏。它应该从编写测试中很熟悉，所以让我们添加一些来看看 Rust 的变体：

```rs
    #[test]
    #[should_panic]
    fn test_assert() {
        assert!(1 == 2);
    }

    #[test]
    #[should_panic]
    fn test_assert_eq() {
        assert_eq!(1, 2);
    }

    #[test]
    #[should_panic]
    fn test_assert_neq() {
        assert_ne!(1, 1);
    }
```

1.  最后一步，像往常一样，使用`cargo test`编译并运行我们刚刚编写的代码。输出显示测试是否通过（它们应该通过）：

```rs
$ cargo test
 Compiling panicking-responsibly v0.1.0 (Rust- Cookbook/Chapter05
 /panicking-responsibly)
Finished dev [unoptimized + debuginfo] target(s) in 0.29s
Running target/debug/deps/panicking_responsibly-6ec385e96e6ee9cd

running 9 tests
test tests::test_assert ... ok
test tests::test_assert_eq ... ok
test tests::test_assert_neq ... ok
test tests::test_panic_format ... ok
test tests::test_expect ... ok
test tests::test_panic_message ... ok
test tests::test_panic_return_value ... ok
test tests::test_regular_panic ... ok
test tests::test_unwrap ... ok

test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests panicking-responsibly

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

但这如何让我们能够负责任地恐慌呢？让我们看看它是如何工作的。

# 它是如何工作的...

多亏了 Rust 检查 panic 结果的能力，我们可以验证消息和 panic 发生的事实。从*步骤 2*到*步骤 4*，我们只是在使用各种（常见）方法恐慌，例如`unwrap()` ([`doc.rust-lang.org/std/option/enum.Option.html#method.unwrap`](https://doc.rust-lang.org/std/option/enum.Option.html#method.unwrap))或`panic!()` ([`doc.rust-lang.org/std/macro.panic.html`](https://doc.rust-lang.org/std/macro.panic.html))。这些方法返回的消息，如`'called `Option::unwrap()` on a `None` value', src/libcore/option.rs:347:21`或`panicked at 'explicit panic', src/lib.rs:64:9`，并不容易调试。

然而，有一个名为`unwrap()`的`expect()`变体，它接受一个`&str`参数作为用户用于进一步调试问题的简单消息。*步骤 4*到*步骤 6*展示了如何结合消息和返回值。在*步骤 7*中，我们介绍了额外的`assert!`宏，它通常在测试中看到，但也进入生产系统以防止罕见且无法恢复的值。

停止线程或程序的执行应该是最后的手段，尤其是在你为他人创建库时。想想看——一些错误导致第三方库中出现意外的值，然后引发恐慌并使服务立即意外停止。想象一下，如果这种情况是由于调用`unwrap()`而不是使用更健壮的方法而发生的。

我们已经成功地学会了如何负责任地恐慌。现在，让我们继续下一个菜谱。

# 处理多个错误

当一个应用程序变得更加复杂并包含第三方框架时，需要一致地处理各种错误类型，而不需要对每个错误都设置条件。例如，一个网络服务的大量可能错误可能会冒泡到处理器，在那里它们需要被转换成带有信息性消息的 HTTP 代码。这些预期的错误可能从解析错误到无效的认证细节，失败的数据库连接，或者带有错误代码的应用程序特定错误。在这个菜谱中，我们将介绍如何使用包装器来处理这些各种错误。

# 如何做到这一点...

让我们分几个步骤创建一个错误包装器：

1.  使用 VS Code 打开你用`cargo new multiple-errors`创建的项目。

1.  打开`src/main.rs`并添加一些顶部的导入：

```rs
use std::fmt;
use std::io;
use std::error::Error;
```

1.  在我们的应用程序中，我们将处理三种用户定义的错误。让我们在导入之后立即声明它们：

```rs
#[derive(Debug)]
pub struct InvalidDeviceIdError(usize);
#[derive(Debug)]
pub struct DeviceNotPresentError(usize);
#[derive(Debug)]
pub struct UnexpectedDeviceStateError {}
```

1.  现在是包装器的时间：由于我们正在处理某种事物的多个变体，`enum`将完美地满足这个目的：

```rs

#[derive(Debug)]
pub enum ErrorWrapper {
    Io(io::Error),
    Db(InvalidDeviceIdError),
    Device(DeviceNotPresentError), 
    Agent(UnexpectedDeviceStateError)
}
```

1.  然而，如果有一个与其他错误相同的接口那就更好了，所以让我们实现`std::error::Error`特质：

```rs
impl Error for ErrorWrapper {
    fn description(&self) -> &str {
        match *self {
            ErrorWrapper::Io(ref e) => e.description(),
            ErrorWrapper::Db(_) | ErrorWrapper::Device(_) => "No 
             device present with this id, check formatting.",
            _ => "Unexpected error. Sorry for the inconvenience."
        }
    }
}
```

1.  特性使得必须实现`std::fmt::Display`，因此这将是我们下一个`impl`块：

```rs
impl fmt::Display for ErrorWrapper {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match *self {
            ErrorWrapper::Io(ref e) => write!(f, "{} [{}]", e, 
                self.description()), 
            ErrorWrapper::Db(ref e) => write!(f, "Device with id \"
             {}\" not found [{}]", e.0, self.description()),
            ErrorWrapper::Device(ref e) => write!(f, "Device with
             id\"{}\" is currently unavailable [{}]", e.0,         
             self.description()),
            ErrorWrapper::Agent(_) => write!(f, "Unexpected device 
             state [{}]", self.description())
        }
    }
}
```

1.  现在，我们想看到我们劳动的结果。按照以下方式替换现有的`main`函数：

```rs
fn main() {
    println!("{}",     
    ErrorWrapper::Io(io::Error::from(io::ErrorKind::InvalidData)));
    println!("{}", ErrorWrapper::Db(InvalidDeviceIdError(42)));
    println!("{}", ErrorWrapper::Device
     (DeviceNotPresentError(42)));
    println!("{}", ErrorWrapper::Agent(UnexpectedDeviceStateError 
{}));
}
```

1.  最后，我们执行`cargo run`以查看输出是否与我们之前预期的相符：

```rs
$ cargo run
 Compiling multiple-errors v0.1.0 (Rust-Cookbook/Chapter05
 /multiple-errors)
 Finished dev [unoptimized + debuginfo] target(s) in 0.34s
 Running `target/debug/multiple-errors`
invalid data [invalid data]
Device with id "42" not found [No device present with this id, check formatting.]
Device with id "42" is currently unavailable [No device present with this id, check formatting.]
Unexpected device state [Unexpected error. Sorry for the inconvenience.]
```

现在，让我们深入幕后，更好地理解代码。

# 它是如何工作的...

多个错误一开始可能看起来不是什么大问题，但对于一个清晰、易读的架构，有必要以某种方式解决它们。一个封装可能变体的枚举已被证明是最实用的解决方案，通过实现`std::error::Error`（以及`std::fmt::Display`要求），新错误类型的处理应该无缝。在*步骤 3*到*6*中，我们以简约的方式展示了所需特性的示例实现。*步骤 7*展示了如何使用封装枚举以及如何使用`Display`和`Error`实现来帮助匹配变体。

实现`Error`特性将在未来允许有趣的特点，包括递归嵌套。检查[`doc.rust-lang.org/std/error/trait.Error.html#method.source`](https://doc.rust-lang.org/std/error/trait.Error.html#method.source)文档以了解更多信息。通常，如果我们能避免，我们不会创建这些错误变体，这就是为什么有支持性 crate 负责所有样板代码——我们将在本章的另一道菜中介绍这一点。

让我们继续到下一个菜谱，以补充我们在处理多个错误方面的新技能！

# 与异常结果一起工作

除了`Option`类型外，`Result`类型还可以有两个自定义类型，这意味着`Result`提供了关于错误原因的额外信息。这比返回单个类型实例或`None`的`Option`更具有表达性。然而，这个`None`实例可以意味着从*处理失败*到*输入错误*的任何东西。这样，`Result`类型可以被视为与其他语言中的异常类似系统，但它们是程序常规工作流程的一部分。一个例子是搜索，其中可能发生多种情况：

+   找到了所需值。

+   未找到所需值。

+   集合无效。

+   该值无效。

如何有效地使用`Result`类型？让我们在这个菜谱中找出答案！

# 如何做...

这里有一些使用`Result`和`Option`的步骤：

1.  使用`cargo new exceptional-results --lib`创建一个新的项目，并用 VS Code 打开它。

1.  打开`src/lib.rs`并在`test`模块之前添加一个函数：

```rs
/// 
/// Finds a needle in a haystack, returns -1 on error 
/// 
pub fn bad_practice_find(needle: &str, haystack: &str) -> i32 {
    haystack.find(needle).map(|p| p as i32).unwrap_or(-1)
}
```

1.  如其名所示，这不是在 Rust 中传达失败的最佳方式。那么更好的方式是什么呢？一个答案是利用`Option`枚举。在第一个函数下面添加另一个函数：

```rs
/// 
/// Finds a needle in a haystack, returns None on error 
/// 
pub fn better_find(needle: &str, haystack: &str) -> Option<usize> {
    haystack.find(needle)
}
```

1.  这使得推理预期的返回值成为可能，但 Rust 允许更丰富的变化——例如`Result`类型。将以下内容添加到当前函数集合中：

```rs
#[derive(Debug, PartialEq)]
pub enum FindError {
    EmptyNeedle,
    EmptyHaystack,
    NotFound,
}

/// 
/// Finds a needle in a haystack, returns a proper Result 
/// 
pub fn best_find(needle: &str, haystack: &str) -> Result<usize, FindError> {
    if needle.len() <= 0 {
        Err(FindError::EmptyNeedle)
    } else if haystack.len() <= 0 {
        Err(FindError::EmptyHaystack)
    } else {
        haystack.find(needle).map_or(Err(FindError::NotFound), |n| 
    Ok(n))
    }
}
```

1.  现在我们实现了几种相同函数的变体，让我们来测试它们。对于第一个函数，将以下内容添加到 `test` 模块中，并替换现有的（默认）测试：

```rs
    use super::*;

    #[test]
    fn test_bad_practice() {
        assert_eq!(bad_practice_find("a", "hello world"), -1);
        assert_eq!(bad_practice_find("e", "hello world"), 1);
        assert_eq!(bad_practice_find("", "hello world"), 0);
        assert_eq!(bad_practice_find("a", ""), -1);
    }
```

1.  其他测试函数看起来非常相似。为了保持一致的结果并展示返回类型之间的差异，将这些添加到 `test` 模块中：

```rs
    #[test]
    fn test_better_practice() {
        assert_eq!(better_find("a", "hello world"), None);
        assert_eq!(better_find("e", "hello world"), Some(1));
        assert_eq!(better_find("", "hello world"), Some(0)); 
        assert_eq!(better_find("a", ""), None); 
    }

    #[test]
    fn test_best_practice() {
        assert_eq!(best_find("a", "hello world"), 
        Err(FindError::NotFound));
        assert_eq!(best_find("e", "hello world"), Ok(1));
        assert_eq!(best_find("", "hello world"), 
        Err(FindError::EmptyNeedle));
        assert_eq!(best_find("e", ""), 
        Err(FindError::EmptyHaystack)); 
    }
```

1.  让我们运行 `cargo test` 来查看测试结果：

```rs
$ cargo test
Compiling exceptional-results v0.1.0 (Rust-Cookbook/Chapter05
 /exceptional-results)
Finished dev [unoptimized + debuginfo] target(s) in 0.53s
Running target/debug/deps/exceptional_results-97ca0d7b67ae4b8b

running 3 tests
test tests::test_best_practice ... ok
test tests::test_bad_practice ... ok
test tests::test_better_practice ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests exceptional-results

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

现在，让我们看看幕后发生了什么，以便更好地理解代码。

# 它是如何工作的...

Rust 和许多其他编程语言使用 `Result` 类型来一次性传达多个函数结果。这样，函数可以像设计的那样返回，而不需要（意外的）跳跃，例如异常机制。

在这个菜谱的 *第 3 步* 中，我们展示了在其他语言中常见的错误通信方式（例如，Java）——然而，正如我们在测试 (*第 6 步*) 中所看到的，空字符串的结果是意外的（`0` 而不是 `-1`）。在第 *3 步* 中，我们定义了一个更好的返回类型，但这足够吗？不，还不够。在第 *4 步* 中，我们实现了函数的最佳版本，其中每个 `Result` 类型都易于解释和明确定义。

一个如何使用 `Result` 的更伟大的例子可以在标准库中找到。它是 `slice` 特性上的 `quick_search` 函数，它返回找到项的位置的 `Ok()` 和应该找到项的位置的 `Err()`。有关更多详细信息，请查看[`doc.rust-lang.org/std/primitive.slice.html#method.binary_search`](https://doc.rust-lang.org/std/primitive.slice.html#method.binary_search)文档。

一旦你掌握了使用多个 `Result` 和 `Option` 类型来传达成功和失败之外的信息的技巧，其他人会喜欢你的表达性 API。继续学习，进入下一个菜谱。

# 无缝错误处理

异常在许多程序中是一个特殊情况：它们有自己的执行路径，程序可以随时跳入这个路径。但这理想吗？这取决于 `try` 块的大小（或 whatever 的名称）；这可能会覆盖几个语句，调试运行时异常很快就会变得不有趣。实现安全错误处理的一种更好的方式可能是将错误集成到函数调用的结果中——这种做法已经在 C 函数中看到，其中参数执行数据传输，返回代码表示成功/失败。较新的、更函数式的方法建议类似于 Rust 中的 `Result` 类型——它带有用于优雅地处理各种结果的功能。这使得错误成为函数的预期结果，并使错误处理无需为每个调用添加额外的 `if` 条件。

在这个菜谱中，我们将介绍几种无缝处理错误的方法。

# 如何做到这一点...

让我们通过一些步骤来无缝处理错误：

1.  使用 `cargo new exceptional-results --lib` 创建一个新的项目，并用 VS Code 打开它。

1.  打开`src/lib.rs`并替换现有的测试为新测试：

```rs
    #[test]
    fn positive_results() {
       // code goes here
    }
```

1.  如其名所示，我们将在函数体中添加一些正面的结果测试。让我们从声明和简单的内容开始。用以下内容替换前面的`// code goes here`部分：

```rs
        let ok: Result<i32, f32> = Ok(42);

        assert_eq!(ok.and_then(|r| Ok(r + 1)), Ok(43));
        assert_eq!(ok.map(|r| r + 1), Ok(43));
```

1.  让我们添加一些更多的变化，因为多个`Result`类型可以表现得就像布尔值一样。在`good_results`测试中添加一些更多的代码：

```rs
        // Boolean operations with Results. Take a close look at 
        // what's returned
        assert_eq!(ok.and(Ok(43)), Ok(43));
        let err: Result<i32, f32> = Err(-42.0);
        assert_eq!(ok.and(err), err);
        assert_eq!(ok.or(err), ok);
```

1.  然而，有好结果的地方，也可能会有坏结果！在`Result`类型的情况下，这关乎`Err`变体。添加另一个名为`negative_results`的空测试：

```rs
    #[test]
    fn negative_results() {
        // code goes here
    }
```

1.  就像之前一样，我们将`//code goes here`注释替换为一些实际的测试：

```rs
        let err: Result<i32, f32> = Err(-42.0);
        let ok: Result<i32, f32> = Ok(-41);

        assert_eq!(err.or_else(|r| Ok(r as i32 + 1)), ok);
        assert_eq!(err.map(|r| r + 1), Err(-42.0));
        assert_eq!(err.map_err(|r| r + 1.0), Err(-41.0));
```

1.  除了正面的结果外，负面的结果通常有自己的函数，例如`map_err`。与它相反，布尔函数表现一致，并将`Err`结果视为假。将以下内容添加到`negative_results`测试中：

```rs
        let err2: Result<i32, f32> = Err(43.0);
        let ok: Result<i32, f32> = Ok(42);
        assert_eq!(err.and(err2), err);
        assert_eq!(err.and(ok), err);
        assert_eq!(err.or(ok), ok);
```

1.  作为最后一步，我们运行`cargo test`来查看测试结果：

```rs
$ cargo test
 Compiling seamless-errors v0.1.0 (Rust-Cookbook/Chapter05
  /seamless-errors)
 Finished dev [unoptimized + debuginfo] target(s) in 0.37s
 Running target/debug/deps/seamless_errors-7a2931598a808519

running 2 tests
test tests::positive_results ... ok
test tests::negative_results ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests seamless-errors

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

你想知道更多吗？继续阅读以了解它是如何工作的。

# 它是如何工作的...

`Result`类型对于创建将所有可能函数结果集成到常规工作流程中的代码非常重要。这消除了对异常的特殊处理需求，使代码更加简洁，更容易推理。由于这些类型事先已知，库可以提供专门的函数，这正是我们在本菜谱中要探讨的。

在前几个步骤（*步骤 2* 到 *步骤 4*），我们正在处理正面的结果，这意味着被`Ok`枚举变体包裹的值。首先，我们介绍了`and_then`函数，它提供了各种函数的链式调用，这些函数只有在初始`Result`为`Ok`时才应该执行。如果链中的某个函数返回`Err`值，则`Err`结果会被传递下去，跳过正面的处理器（如`and_then`和`map`）。同样，`map()`允许在`Result`类型内进行转换。`map`和`and_then`都只允许将`Result<i32, i32>`转换为`Result<MyOwnType, i32>`，但不能单独转换为`MyOwnType`。最后，测试覆盖了表中总结的多个`Result`类型的布尔运算：

| **A** | **B** | **A and B** |
| --- | --- | --- |
| `Ok` | `Ok` | `Ok` (B) |
| `Ok` | `Err` | `Err` |
| `Err` | `Ok` | `Err`  |
| `Err` | `Err` | `Err` (A) |
| `Ok` | `Ok` | `Ok` (A) |

剩余的步骤（*步骤 5* 到 *步骤 7*）展示了与负面的结果类型`Err`相同的处理过程：`map()`只处理`Ok`结果，`map_err()`转换`Err`。其特殊情况是`or_else()`函数，它在返回`Err`时执行提供的闭包。测试的最后部分涵盖了多个`Result`类型的布尔函数，并展示了它们如何与各种`Err`参数一起工作。

现在我们已经看到了许多处理`Ok`和`Err`的不同变体，让我们继续下一个菜谱。

# 自定义错误

虽然`Result`类型对`Err`分支返回的类型不关心，但返回`String`实例作为错误信息也不是最佳选择。典型的错误有几个需要考虑的因素：

+   是否有根本原因或错误？

+   什么是错误信息？

+   是否有更深入的消息要输出？

标准库的错误都遵循来自`std::error::Error`的通用特质——让我们看看它们是如何实现的。

# 如何做...

定义错误类型并不难——只需遵循以下步骤：

1.  使用`cargo new custom-errors`创建一个新的项目，并用 VS Code 打开它。

1.  使用 VS Code，打开`src/main.rs`并创建一个名为`MyError`的基本结构体：

```rs
use std::fmt;
use std::error::Error;

#[derive(Debug)]
pub struct MyError {
    code: usize,
}
```

1.  我们可以实现一个`Error`特质，如下所示：

```rs
impl Error for MyError {
    fn description(&self) -> &str {
        "Occurs when someone makes a mistake"
 }
}
```

1.  然而，特质还要求我们（除了我们推导出的`Debug`之外）实现`std::fmt::Display`：

```rs
impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
       write!(f, "Error code {:#X}", self.code) 
    }
}
```

1.  最后，让我们看看这些特质是如何发挥作用的，并替换`main`函数：

```rs
fn main() {
    println!("Display: {}", MyError{ code: 1535 });
    println!("Debug: {:?}", MyError{ code: 42 });
    println!("Description: {:?}", (MyError{ code: 42 
    }).description());
}
```

1.  然后，我们可以使用`cargo run`看到一切是如何协同工作的：

```rs
$ cargo run
Compiling custom-errors v0.1.0 (Rust-Cookbook/Chapter05/custom-errors)
 Finished dev [unoptimized + debuginfo] target(s) in 0.23s
 Running `target/debug/custom-errors`
Display: Error code 0x5FF
Debug: MyError { code: 42 }
Description: "Occurs when someone makes a mistake"
```

让我们看看是否可以深入了解这个简短的食谱。

# 它是如何工作的...

虽然任何类型在`Result`分支中都可以正常工作，但 Rust 提供了一个错误特质，可以用于更好地集成到其他 crate 中。一个例子是`actix_web`框架的错误处理（[`actix.rs/docs/errors/`](https://actix.rs/docs/errors/)），它既与`std::error::Error`一起工作，也与其自己的类型一起工作（我们将在第八章 Chapter 8，*Web 的安全编程*）进行更深入的探讨）。

此外，`Error`特质还提供了嵌套，并且，通过动态分派，所有`Errors`都可以遵循一个通用的 API。在*步骤 2*中，我们声明类型并推导出（强制性的）`Debug`特质。在*步骤 3*和*步骤 4*中，剩余的实现遵循。其余的食谱执行代码。

在这个简短而甜美的食谱中，我们可以创建自定义错误类型。现在，让我们继续下一个食谱。

# 弹性编程

返回`Result`或`Option`将始终遵循一个特定的模式，该模式生成大量的模板代码——特别是对于不确定的操作，如读取或创建文件和搜索值。特别是，该模式会产生大量使用早期返回的代码（记得`goto`吗？）或嵌套语句，这两种情况都会产生难以推理的代码。因此，Rust 库的早期版本实现了一个`try!`宏，它已被`?`运算符所取代，作为快速早期返回的选项。让我们看看这如何影响代码。

# 如何做...

按照以下步骤编写更健壮的程序：

1.  使用`cargo new resilient-programming`创建一个新的项目，并用 VS Code 打开它。

1.  打开`src/main.rs`以添加一个函数：

```rs
use std::fs;
use std::io;

fn print_file_contents_qm(filename: &str) -> Result<(), io::Error> {
    let contents = fs::read_to_string(filename)?;
    println!("File contents, external fn: {:?}", contents);
    Ok(())
}
```

1.  前面的函数在找到文件时会打印文件内容；除此之外，我们还需要调用这个函数。为此，用以下内容替换现有的`main`函数：

```rs
fn main() -> Result<(), std::io::Error> {
    println!("Ok: {:?}", print_file_contents_qm("testfile.txt"));
    println!("Err: {:?}", print_file_contents_qm("not-a-file"));

    let contents = fs::read_to_string("testfile.txt")?;
    println!("File contents, main fn: {:?}", contents);
    Ok(())
}
```

1.  就这样——运行`cargo run`以找出结果：

```rs
$ cargo run
 Compiling resilient-programming v0.1.0 (Rust-Cookbook/Chapter05
  /resilient-programming)
 Finished dev [unoptimized + debuginfo] target(s) in 0.21s
 Running `target/debug/resilient-programming`
File contents, external fn: "Hello World!"
Ok: Ok(())
Err: Err(Os { code: 2, kind: NotFound, message: "No such file or directory" })
File contents, main fn: "Hello World!"
```

现在，让我们深入了解代码，以更好地理解它。

# 它是如何工作的...

在这四个步骤中，我们看到了问号运算符的使用以及它是如何避免与守卫相关的典型模板代码。在*步骤 3*中，我们创建了一个函数，如果找到了文件（并且可读），它会打印文件内容；通过使用`?`运算符，我们可以跳过检查返回值并在必要时退出函数——所有这些操作都通过简单的`?`运算符完成。

在*步骤 4*中，我们不仅调用了之前创建的函数，而且还打印了结果以展示其工作方式。此外，相同的模式也应用于（特殊的）`main`函数，它现在有返回值。因此，`?`不仅限于子函数，还可以应用于整个应用程序。

只需几个简单的步骤，我们就看到了如何安全地使用`?`运算符来解包`Result`。现在，让我们继续下一个菜谱。

# 使用外部 crate 进行错误处理

在现代程序中，创建和包装错误是一个常见的任务。然而，正如我们在本章的各种菜谱中所看到的，处理每一个可能的案例以及关心可能返回的每个可能的变体可能会相当繁琐。这个问题是众所周知的，Rust 社区已经找到了使这变得更容易的方法。我们将在下一章（第六章，*使用宏表达自己*）中涉及宏，但创建错误类型很大程度上依赖于使用宏。此外，这个菜谱与之前的菜谱（*处理多个错误*）相呼应，以展示代码中的差异。

# 如何做到这一点...

让我们通过几个步骤引入一些外部 crate 来更好地处理错误：

1.  使用`cargo new external-crates`创建一个新的项目，并用 VS Code 打开它。

1.  编辑`Cargo.toml`以添加`quick-error`依赖项：

```rs
[dependencies]
quick-error = "1.2"
```

1.  要使用`quick-error`中提供的宏，我们需要显式地导入它们。将以下`use`语句添加到`src/main.rs`中：

```rs
#[macro_use] extern crate quick_error;

use std::convert::From;
use std::io;
```

1.  然后，我们将一步添加所有我们想要在`quick_error!`宏中声明的错误：

```rs
quick_error! {
    #[derive(Debug)]
    pub enum ErrorWrapper {
        InvalidDeviceIdError(device_id: usize) {
            from(device_id: usize) -> (device_id)
            description("No device present with this id, check 
            formatting.")
        }

        DeviceNotPresentError(device_id: usize) {
            display("Device with id \"{}\" not found", device_id)
        }

        UnexpectedDeviceStateError {}

        Io(err: io::Error) {
            from(kind: io::ErrorKind) -> (io::Error::from(kind))
            description(err.description())
            display("I/O Error: {}", err)
        } 
    }
}
```

1.  代码只有在添加了`main`函数后才是完整的：

```rs
fn main() {
    println!("(IOError) {}", 
    ErrorWrapper::from(io::ErrorKind::InvalidData));
    println!("(InvalidDeviceIdError) {}", 
    ErrorWrapper::InvalidDeviceIdError(42));
    println!("(DeviceNotPresentError) {}", 
    ErrorWrapper::DeviceNotPresentError(42));
    println!("(UnexpectedDeviceStateError) {}", 
    ErrorWrapper::UnexpectedDeviceStateError {});
}
```

1.  使用`cargo run`来查找程序的输出：

```rs
$ cargo run
 Compiling external-crates v0.1.0 (Rust-Cookbook/Chapter05
  /external-crates)
 Finished dev [unoptimized + debuginfo] target(s) in 0.27s
  Running `target/debug/external-crates`
(IOError) I/O Error: invalid data
(InvalidDeviceIdError) No device present with this id, check formatting.
(DeviceNotPresentError) Device with id "42" not found
(UnexpectedDeviceStateError) UnexpectedDeviceStateError
```

你理解代码了吗？让我们找出它是如何工作的。

# 它是如何工作的...

与我们之前声明多个错误的菜谱相比，这个声明要短得多，并且有几个额外的优点。第一个优点是，每个错误类型都可以使用`From`特质创建（*步骤 4*中的第一个`IOError`）。其次，每个类型都会自动生成错误名称的描述和`Display`实现（参见*步骤 3*中的`UnexpectedDeviceStateError`，然后是*步骤 5*）。这并不完美，但作为第一步是不错的。

在底层，`quick-error` 生成一个处理所有可能情况的枚举，并在必要时生成实现。查看 `main` 宏——相当令人印象深刻（[`tailhook.github.io/quick-error/quick_error/macro.quick_error.html`](http://tailhook.github.io/quick-error/quick_error/macro.quick_error.html)）！为了根据您的需求定制 `quick-error` 的使用，请查看他们的其余文档，网址为 [`tailhook.github.io/quick-error/quick_error/index.html`](http://tailhook.github.io/quick-error/quick_error/index.html)。或者，还有 `error-chain` crate ([`github.com/rust-lang-nursery/error-chain`](https://github.com/rust-lang-nursery/error-chain))，它采用不同的方法来创建这些错误类型。这两种选项中的任何一种都可以让您极大地提高错误的可读性和实现速度，同时移除所有样板代码。

我们已经成功地学习了如何通过使用外部 crate 来改进我们的错误处理。现在，让我们继续到下一个菜谱。

# 在 Option 和 Result 之间切换

当函数需要返回二进制结果时，选择使用 `Result` 或 `Option`。两者都可以传达函数调用失败的信息——但前者提供了过多的具体信息，而后者可能提供的信息过少。虽然这是一个针对特定情况做出的决定，但 Rust 的类型提供了在它们之间轻松转换的工具。让我们在这道菜谱中逐一了解它们。

# 如何做到这一点...

在几个快速步骤中，您将了解如何在不同之间切换 `Option` 和 `Result`：

1.  使用 `cargo new options-results --lib` 创建一个新的项目，并用 VS Code 打开它。

1.  让我们编辑 `src/lib.rs` 并将现有的测试（在 `mod tests` 内部）替换为以下内容：

```rs
    #[derive(Debug, Eq, PartialEq, Copy, Clone)]
    struct MyError;

    #[test]
    fn transposing() {
        // code will follow
    }
```

1.  我们需要将 `// code will follow` 替换为如何使用 `transpose()` 函数的示例：

```rs
        let this: Result<Option<i32>, MyError> = Ok(Some(42));
        let other: Option<Result<i32, MyError>> = Some(Ok(42));
        assert_eq!(this, other.transpose());
```

1.  这也适用于 `Err`，为了证明这一点，将其添加到 `transpose()` 测试中：

```rs
        let this: Result<Option<i32>, MyError> = Err(MyError);
        let other: Option<Result<i32, MyError>> = Some(Err(MyError));
        assert_eq!(this, other.transpose());
```

1.  剩下的特殊情况是 `None`。使用以下内容完成 `transpose()` 测试：

```rs
assert_eq!(None::<Result<i32, MyError>>.transpose(), Ok(None::
 <i32>));
```

1.  在两种类型之间切换不仅涉及转置——还有更复杂的方法可以做到这一点。创建另一个 `test`：

```rs
    #[test]
    fn conversion() {
        // more to follow
    }
```

1.  作为第一次测试，让我们用可以替代 `unwrap()` 的内容替换 `// more to follow`：

```rs
        let opt = Some(42);
        assert_eq!(opt.ok_or(MyError), Ok(42));

        let res: Result<i32, MyError> = Ok(42);
        assert_eq!(res.ok(), opt);
        assert_eq!(res.err(), None);
```

1.  为了完成转换测试，也请将以下内容添加到 `test` 中。这些是转换，但来自 `Err` 方面：

```rs
        let opt: Option<i32> = None;
        assert_eq!(opt.ok_or(MyError), Err(MyError));

        let res: Result<i32, MyError> = Err(MyError);
        assert_eq!(res.ok(), None);
        assert_eq!(res.err(), Some(MyError));
```

1.  最后，我们应该使用 `cargo test` 运行代码，并查看成功的测试结果：

```rs
$ cargo test
Compiling options-results v0.1.0 (Rust-Cookbook/Chapter05/options-results)
 Finished dev [unoptimized + debuginfo] target(s) in 0.44s
 Running target/debug/deps/options_results-111cad5a9a9f6792

running 2 tests
test tests::conversion ... ok
test tests::transposing ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests options-results

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

现在，让我们深入幕后，更好地理解代码。

# 它是如何工作的...

虽然关于何时使用 `Option` 和何时使用 `Result` 的讨论发生在较高层面，但 Rust 通过几个函数支持这两种类型之间的转换。除了 `map()`、`and_then()` 等函数（在本章的 *无缝错误处理* 部分讨论过）之外，这些函数还提供了处理各种错误的有效且优雅的方法。在 *步骤 1* 到 *步骤 4* 中，我们逐渐构建了一个简单的测试，展示了转置函数的应用性。这使得通过一个函数调用就能从 `Ok(Some(42))` 切换到 `Some(Ok(42))`（注意细微的区别）。同样，调用中的 `Err` 变体从常规的 `Err(MyError)` 函数变为 `Some(Err(MyError))`。

剩余的步骤（*步骤 6* 到 *步骤 8*）展示了在两种类型之间转换的更传统方法。这包括获取 `Ok` 和 `Err` 的值，以及为正结果提供错误实例。一般来说，这些函数足以替换大多数 `unwrap()` 或 `expect()` 调用，并且程序中只有一个执行路径，无需求助于 `if` 和 `match` 条件语句。这增加了鲁棒性和可读性的额外优势，你的未来同事和用户会感谢你！
