# 第七章：*第七章*：Rust 和 Rocket 中的错误处理

在上一章中，我们学习了如何创建端点和 SQL 查询来处理`User`实体的管理。在本章中，我们将学习更多关于 Rust 和 Rocket 中的错误处理。学习本章的概念后，你将能够实现 Rocket 应用程序中的错误处理。

我们还将讨论更多在 Rust 和 Rocket 中处理错误的方法，包括使用`panic!`宏来指示不可恢复的错误，以及使用`Option`、`Result`、创建自定义`Error`类型和记录生成的错误来捕获`panic!`宏。

在本章中，我们将涵盖以下主要主题：

+   使用 panic!

+   使用 Option

+   返回 Result

+   创建自定义错误类型

+   记录错误

# 技术要求

对于本章，我们与上一章有相同的技术要求。我们需要一个 Rust 编译器、一个文本编辑器、一个 HTTP 客户端和一个 PostgreSQL 数据库服务器。

你可以在[`github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter07`](https://github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter07)找到本章的源代码。

# 使用 panic!

要理解 Rust 中的错误处理，我们需要从`panic!`宏开始。当应用程序遇到不可恢复的错误且继续应用程序没有意义时，我们可以使用`panic!`宏。如果应用程序遇到`panic!`，应用程序将发出后迹并终止。

让我们尝试在上一章创建的程序中使用`panic!`。假设我们希望在初始化 Rocket 之前让应用程序读取一个秘密文件。如果应用程序找不到这个秘密文件，它将不会继续。

让我们开始吧：

1.  在`src/main.rs`中添加以下行：

    ```rs
    use std::env;
    ```

1.  在`rocket()`函数中的同一文件，添加以下行：

    ```rs
    let secret_file_path = env::current_dir().unwrap().join("secret_file");
    if !secret_file_path.exists() {
        panic!("secret does not exists");
    }
    ```

1.  之后，尝试在当前工作目录下不创建名为`secret_file`的空文件的情况下执行`cargo run`。你应该看到以下输出：

    ```rs
    thread 'main' panicked at 'secret does not exists', src/main.rs:15:9
    note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
    ```

1.  现在，尝试使用`RUST_BACKTRACE=1 cargo run`再次运行应用程序。你应该在终端中看到类似于以下的后迹输出：

    ```rs
    RUST_BACKTRACE=1 cargo run 
    Finished dev [unoptimized + debuginfo] target(s) 
        in 0.18s
         Running `target/debug/our_application`
    thread 'main' panicked at 'secret does not exists', src/main.rs:15:9
    stack backtrace:
    ...
      14: our_application::main
                 at ./src/main.rs:12:36
      15: core::ops::function::FnOnce::call_once
                 at /rustc/59eed8a2aac0230a8b5
                 3e89d4e99d55912ba6b35/library/core/
                 src/ops/function.rs:227:5
    note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
    ```

1.  有时候，我们不想在`panic!`宏中恐慌后进行资源释放，因为我们希望应用程序尽快退出。我们可以通过在`Cargo.toml`中设置`panic = "abort"`来跳过释放，在使用的配置文件下。设置此配置将使我们的二进制文件更小，退出更快，操作系统将在稍后清理它。让我们尝试这样做。在`Cargo.toml`中设置以下行并再次运行应用程序：

    ```rs
    [profile.dev]
    panic = "abort"
    ```

现在我们知道了如何使用`panic!`，让我们看看如何在下一节中捕获它。

## 捕获 panic!

除了使用 `panic!`，我们还可以在 Rust 代码中使用 `todo!` 和 `unimplemented!` 宏。这些宏在原型设计时非常有用，因为它们会在调用 `panic!` 的同时允许代码在编译时进行类型检查。

但是，为什么当我们使用 `todo!` 调用一个路由时，Rocket 不会关闭呢？如果我们检查 Rocket 的源代码，会发现 `src::panic` 中有一个 `catch_unwind` 函数，它可以用来捕获抛出异常的函数。让我们看看 Rocket 源代码中的这段代码，`core/lib/src/server.rs`:

```rs
let fut = std::panic::catch_unwind(move || run())
```

```rs
         .map_err(|e| panic_info!(name, e))
```

```rs
         .ok()?;
```

在这里，`run()` 是一个路由处理函数。每次我们调用一个抛出异常的路由时，前面的程序会将异常转换为结果的 `Err` 变体。尝试移除我们之前添加的 `secret_file_path` 程序并运行应用程序。现在，创建一个用户并尝试进入用户帖子。例如，创建一个具有 `95a54c16-e830-45c9-ba1d-5242c0e4c18f` UUID 的用户。尝试打开 `http://127.0.0.1/users/95a54c16-e830-45c9-ba1d-5242c0e4c18f/posts`。由于我们只在函数体中放置了 `todo!("will implement later")`，应用程序将抛出异常，但前面的 `catch_unwind` 函数将捕获这个异常并将其转换为错误。请注意，如果我们在 `Cargo.toml` 中设置 `panic = "abort"`，则 `catch_unwind` 将不会工作。

在常规的工作流程中，我们通常不希望使用 `panic!`，因为抛出异常会中断一切，程序将无法继续。如果 Rocket 框架没有捕获 `panic!`，并且其中一个路由处理函数抛出异常，那么这个单独的错误将关闭应用程序，并且将没有东西来处理其他请求。但是，如果我们想在遇到不可恢复的错误时终止 Rocket 应用程序，我们应该如何做呢？让我们看看下一节如何实现。

## 使用关闭

为了在路由处理函数遇到不可恢复的错误时平稳关闭，我们可以使用 `rocket::Shutdown` 请求守卫。记住，请求守卫是我们提供给路由处理函数的参数。

要查看 `Shutdown` 请求守卫的实际效果，让我们尝试在我们的应用程序中实现它。使用之前的应用程序，在 `src/routes/mod.rs` 中添加一个新的路由 `/shutdown`:

```rs
use rocket::Shutdown;
```

```rs
...
```

```rs
#[get("/shutdown")]
```

```rs
pub async fn shutdown(shutdown: Shutdown) -> &'static str {
```

```rs
    // suppose this variable is from function which 
```

```rs
    // produces irrecoverable error
```

```rs
    let result: Result<&str, &str> = Err("err");
```

```rs
    if result.is_err() {
```

```rs
        shutdown.notify();
```

```rs
        return "Shutting down the application.";
```

```rs
    }
```

```rs
    return "Not doing anything.";
```

```rs
}
```

尝试在 `src/main.rs` 中添加 `shutdown()` 函数。之后，重新运行应用程序，并向 `/shutdown` 发送 HTTP 请求，同时监控终端上的应用程序输出。应用程序应该能够平稳关闭。

在接下来的两个部分中，我们将看看如何使用 `Option` 和 `Result` 作为处理错误的替代方法。

# 使用 Option

在编程中，一个例程可能会产生正确的结果或遇到问题。一个经典的例子是除以零。在数学上，除以零是未定义的。如果一个应用程序有一个除以某物的例程，并且该例程遇到零作为输入，则应用程序不能返回任何数字。我们希望应用程序返回另一种类型而不是数字。我们需要一种可以持有多种数据变体的类型。

在 Rust 中，我们可以定义一个 `enum` 类型，这是一种可以有不同的数据变体的类型。一个 `enum` 类型可能如下所示：

```rs
enum Shapes {
```

```rs
    None,
```

```rs
    Point(i8),
```

```rs
    Line(i8, i8),
```

```rs
    Rectangle {
```

```rs
        top: (i8, i8),
```

```rs
        length: u8,
```

```rs
        height: u8,
```

```rs
    },
```

```rs
}
```

`Point` 和 `Line` 被说成有 `Rectangle`，而 `Rectangle` 也可以称为 **类似结构体的枚举** 变体。

如果 `enum` 的所有成员都没有数据，我们可以在成员上添加一个区分符。以下是一个例子：

```rs
enum Color {
```

```rs
    Red,         // 0
```

```rs
    Green = 127, // 127
```

```rs
    Blue,        // 128
```

```rs
}
```

我们可以将 `enum` 赋值给一个变量，并在函数中使用该变量，如下所示：

```rs
fn do_something(color: Color) -> Shapes {
```

```rs
    let rectangle = Shapes::Rectangle {
```

```rs
        top: (0, 2),
```

```rs
        length: 10,
```

```rs
        height: 8,
```

```rs
    };
```

```rs
    match color {
```

```rs
        Color::Red => Shapes::None,
```

```rs
        Color::Green => Shapes::Point(10),
```

```rs
        _ => rectangle,
```

```rs
    }
```

```rs
}
```

回到错误处理，我们可以使用 `enum` 来传达我们的代码中存在错误。回到除以零的情况，这里有一个例子：

```rs
enum Maybe {
```

```rs
    WeCannotDoIt,
```

```rs
    WeCanDoIt(i8),
```

```rs
}
```

```rs
fn check_divisible(input: i8) -> Maybe {
```

```rs
    if input == 0 {
```

```rs
        return Maybe::WeCannotDoIt;
```

```rs
    }
```

```rs
    Maybe::WeCanDoIt(input)
```

```rs
}
```

之前模式返回或不返回内容的情况非常常见，因此 Rust 在标准库中有一个自己的枚举来表示我们是否有内容，称为 `std::option::Option`：

```rs
pub enum Option<T> {
```

```rs
    None,
```

```rs
    Some(T),
```

```rs
}
```

`Some(T)` 用于传达我们拥有 `T`，而 `None` 显然用于传达我们没有 `T`。我们在之前的代码中使用了 `Option`。例如，我们在 `User` 结构体中使用了它：

```rs
struct User {
```

```rs
    ...
```

```rs
    description: Option<String>,
```

```rs
    ...
```

```rs
}
```

我们还使用了 `Option` 作为函数参数或返回类型：

```rs
find_all(..., pagination: Option<Pagination>) -> (..., Option<Pagination>), ... {}
```

我们可以使用 `Option` 做很多事情。假设我们有两个变量，`we_have_it` 和 `we_do_not_have_it`：

```rs
let we_have_it: Option<usize> = Some(1);
```

```rs
let we_do_not_have_it: Option<usize> = None;
```

+   我们可以做的事情之一是模式匹配并使用其内容：

    ```rs
    match we_have_it {
        Some(t) => println!("The value = {}", t),
        None => println!("We don't have it"),
    };
    ```

+   如果我们关心 `we_have_it` 的内容，我们可以以更方便的方式处理它：

    ```rs
    if let Some(t) = we_have_it {
        println!("The value = {}", t);
    }
    ```

+   如果内部类型实现了 `std::cmp::Eq` 和 `std::cmp::Ord`，则 `Option` 可以进行比较，即内部类型可以使用 `==`、`!=`、`>` 等比较运算符进行比较。注意我们使用了 `assert!`，这是一个用于测试的宏：

    ```rs
    assert!(we_have_it != we_do_not_have_it);
    ```

+   我们可以检查一个变量是 `Some` 还是 `None`：

    ```rs
    assert!(we_have_it.is_some());
    assert!(we_do_not_have_it.is_none());
    ```

+   我们也可以通过展开 `Option` 来获取内容。但是，有一个注意事项；展开 `None` 将会引发恐慌，所以在展开 `Option` 时要小心。注意我们使用了 `assert_eq!`，这是一个用于测试的宏，用于确保相等性：

    ```rs
    assert_eq!(we_have_it.unwrap(), 1);
    // assert_eq!(we_do_not_have_it.unwrap(), 1); 
    // will panic
    ```

+   我们还可以使用 `expect()` 方法。此方法与 `unwrap()` 的行为相同，但我们可以使用自定义消息：

    ```rs
    assert_eq!(we_have_it.expect("Oh no!"), 1);
    // assert_eq!(we_do_not_have_it.expect("Oh no!"), 1); // will panic
    ```

+   我们可以展开并设置默认值，这样在展开 `None` 时就不会引发恐慌：

    ```rs
    assert_eq!(we_have_it.unwrap_or(42), 1);
    assert_eq!(we_do_not_have_it.unwrap_or(42), 42);
    ```

+   我们可以使用闭包展开并设置默认值：

    ```rs
    let x = 42;
    assert_eq!(we_have_it.unwrap_or_else(|| x), 1);
    assert_eq!(we_do_not_have_it.unwrap_or_else(|| x), 42);
    ```

+   我们可以使用 `map()`、`map_or()` 或 `map_or_else()` 将包含的值转换为其他类型：

    ```rs
    assert_eq!(we_have_it.map(|v| format!("The value = {}", v)), Some("The value = 1".to_string()));
    assert_eq!(we_do_not_have_it.map(|v| format!("The value = {}", v)), None);
    assert_eq!(we_have_it.map_or("Oh no!".to_string(), |v| format!("The value = {}", v)), "The value = 1".to_string());
    assert_eq!(we_do_not_have_it.map_or("Oh no!".to_string(), |v| format!("The value = {}", v)), "Oh no!".to_string());
    assert_eq!(we_have_it.map_or_else(|| "Oh no!".to_string(), |v| format!("The value = {}", v)), "The value = 1".to_string());
    assert_eq!(we_do_not_have_it.map_or_else(|| "Oh no!".to_string(), |v| format!("The value = {}", v)), "Oh no!".to_string());
    ```

还有其他重要的方法，你可以在 `std::option::Option` 的文档中查看。尽管我们可以使用 `Option` 来处理有或没有某种情况的情况，但它并不能传达“出了问题”的消息。我们可以在下一部分使用与 `Option` 类似的另一个类型来实现这一点。

# 返回 Result

在 Rust 中，我们有 `std::result::Result` 枚举，它的工作方式类似于 `Option`，但 `Result` 类型更多的是说“我们有它”或“我们有这个错误”。就像 `Option` 一样，`Result` 是可能的 `T` 类型或可能的 `E` 错误的枚举类型：

```rs
enum Result<T, E> {
```

```rs
   Ok(T),
```

```rs
   Err(E),
```

```rs
}
```

回到除以零的问题，看看以下简单的例子：

```rs
fn division(a: usize, b: usize) -> Result<f64, String> {
```

```rs
    if b == 0 {
```

```rs
        return Err(String::from("division by zero"));
```

```rs
    }
```

```rs
    return Ok(a as f64 / b as f64);
```

```rs
}
```

我们不希望除以 `0`，所以我们在前面的函数中返回一个错误。

与 `Option` 类似，`Result` 有许多方便的特性我们可以使用。假设我们有 `we_have_it` 和 `we_have_error` 变量：

```rs
let we_have_it: Result<usize, &'static str> = Ok(1);
```

```rs
let we_have_error: Result<usize, &'static str> = Err("Oh no!");
```

+   我们可以使用模式匹配来获取值或错误：

    ```rs
    match we_have_it {
        Ok(v) => println!("The value = {}", v),
        Err(e) => println!("The error = {}", e),
    };
    ```

+   或者，我们可以使用 `if let` 来解构并获取值或错误：

    ```rs
    if let Ok(v) = we_have_it {
        println!("The value = {}", v);
    }
    if let Err(e) = we_have_error {
        println!("The error = {}", e);
    }
    ```

+   我们可以比较 `Ok` 变体和 `Err` 变体：

    ```rs
    assert!(we_have_it != we_have_error);
    ```

+   我们可以检查一个变量是 `Ok` 变体还是 `Err` 变体：

    ```rs
    assert!(we_have_it.is_ok());
    assert!(we_have_error.is_err());
    ```

+   我们可以将 `Result` 转换为 `Option`：

    ```rs
    assert_eq!(we_have_it.ok(), Some(1));
    assert_eq!(we_have_error.ok(), None);
    assert_eq!(we_have_it.err(), None);
    assert_eq!(we_have_error.err(), Some("Oh no!"));
    ```

+   就像 `Option` 一样，我们可以使用 `unwrap()`、`unwrap_or()` 或 `unwrap_or_else()`：

    ```rs
    assert_eq!(we_have_it.unwrap(), 1);
    // assert_eq!(we_have_error.unwrap(), 1); 
    // panic
    assert_eq!(we_have_it.expect("Oh no!"), 1);
    // assert_eq!(we_have_error.expect("Oh no!"), 1);
    // panic
    assert_eq!(we_have_it.unwrap_or(0), 1);
    assert_eq!(we_have_error.unwrap_or(0), 0);
    assert_eq!(we_have_it.unwrap_or_else(|_| 0), 1);
    assert_eq!(we_have_error.unwrap_or_else(|_| 0), 0);
    ```

+   此外，我们可以使用 `map()`、`map_err()`、`map_or()` 或 `map_or_else()`：

    ```rs
    assert_eq!(we_have_it.map(|v| format!("The value = {}", v)), Ok("The value = 1".to_string()));
    assert_eq!(
        we_have_error.map(|v| format!("The error = {}", 
        v)),
        Err("Oh no!")
    );
    assert_eq!(we_have_it.map_err(|s| s.len()), Ok(1));
    assert_eq!(we_have_error.map_err(|s| s.len()), Err(6));
    assert_eq!(we_have_it.map_or("Default value".to_string(), |v| format!("The value = {}", v)), "The value = 1".to_string());
    assert_eq!(we_have_error.map_or("Default value".to_string(), |v| format!("The value = {}", v)), "Default value".to_string());
    assert_eq!(we_have_it.map_or_else(|_| "Default value".to_string(), |v| format!("The value = {}", v)), "The value = 1".to_string());
    assert_eq!(we_have_error.map_or_else(|_| "Default value".to_string(), |v| format!("The value = {}", v)), "Default value".to_string());
    ```

除了 `std::result::Result` 文档中的这些方法之外，还有其他重要的方法。请务必查看它们，因为 `Option` 和 `Result` 在 Rust 和 Rocket 中非常重要。

将字符串或数字作为错误返回在某些情况下可能是可接受的，但大多数情况下，我们希望有一个真实的错误类型，包括错误信息和可能的回溯，这样我们就可以进一步处理。在下一节中，我们将学习（并使用）`Error`特质，并在我们的应用程序中返回动态错误类型。

# 创建自定义错误类型

Rust 有一个特质来统一通过提供 `std::error::Error` 特质来传播错误。由于 `Error` 特质被定义为 `pub trait Error: Debug + Display`，任何实现了 `Error` 的类型也应该实现 `Debug` 和 `Display` 特质。

让我们看看如何通过创建一个新的模块来创建自定义错误类型：

1.  在 `src/lib.rs` 中，添加新的 `errors` 模块：

    ```rs
    pub mod errors;
    ```

1.  然后，创建一个新的文件夹，`src/errors`，并添加 `src/errors/mod.rs` 和 `src/errors/our_error.rs` 文件。在 `src/errors/mod.rs` 中，添加以下行：

    ```rs
    pub mod our_error;
    ```

1.  在 `src/errors/our_error.rs` 中，为 `error` 添加自定义类型：

    ```rs
    use rocket::http::Status;
    use std::error::Error;
    use std::fmt;
    #[derive(Debug)]
    pub struct OurError {
        pub status: Status,
        pub message: String,
        debug: Option<Box<dyn Error>>,
    }
    impl fmt::Display for OurError {
        fn fmt(&self, f: &mut fmt::Formatter<'_>) -> 
        fmt::Result {
            write!(f, "{}", &self.message)
        }
    }
    ```

1.  然后，我们可以为 `OurError` 实现 `Error` 特质。在 `src/errors/our_error.rs` 中，添加以下行：

    ```rs
    impl Error for OurError {
        fn source(&self) -> Option<&(dyn Error + 'static)> {
            if self.debug.is_some() {
                self.debug.as_ref().unwrap().source();
            }
            None
        }
    }
    ```

目前，对于 `User` 模块，我们为每个方法返回一个 `Result<..., Box<dyn Error>>` 动态错误。这是一个使用任何实现了 `Error` 的类型来返回错误，然后使用 `Box` 将实例放入堆中的常见模式。

这种方法的缺点是我们只能使用 `Error` 特性提供的方法，即 `source()`。我们希望能够使用 `OurError` 的状态、消息和调试信息。

1.  因此，让我们给 `OurError` 添加几个构建方法。在 `src/errors/our_error.rs` 文件中添加以下行：

    ```rs
    impl OurError {
        fn new_error_with_status(status: Status, message: 
        String, debug: Option<Box<dyn Error>>) -> Self {
            OurError {
                status,
                message,
                debug,
            }
        }
        pub fn new_bad_request_error(message: String, 
        debug: Option<Box<dyn Error>>) -> Self {
            Self::new_error_with_status(Status::
            BadRequest, message, debug)
        }
        pub fn new_not_found_error(message: String,
        debug: Option<Box<dyn Error>>) -> Self {
            Self::new_error_with_status(Status::NotFound, 
            message, debug)
        }
        pub fn new_internal_server_error(
            message: String,
            debug: Option<Box<dyn Error>>,
        ) -> Self {
            Self::new_error_with_status(Status::
            InternalServerError, message, debug)
        }
    }
    ```

1.  如果我们查看 `src/models/user.rs` 文件，有三个错误来源：`sqlx::Error`、`uuid::Error` 和 `argon2`。让我们为 `sqlx::Error` 和 `uuid::Error` 创建到 `OurError` 的转换。在 `src/errors/our_error.rs` 文件中添加以下 `use` 指令：

    ```rs
    use sqlx::Error as sqlxError;
    use uuid::Error as uuidError;
    ```

1.  在同一文件 `src/errors/our_error.rs` 中，添加以下行：

    ```rs
    impl OurError {
        ...
        pub fn from_uuid_error(e: uuidError) -> Self {
            OurError::new_bad_request_error(
                String::from("Something went wrong"),
                Some(Box::new(e)))
        }
    }
    ```

1.  对于 `sqlx::Error`，我们希望将 `not_found` 错误转换为 HTTP 状态 `404`，并将重复索引错误转换为 HTTP 状态 400bad request。在 `src/errors/our_error.rs` 文件中添加以下行：

    ```rs
    use std::borrow::Cow;
    ....
    impl OurError {
        ....
        pub fn from_sqlx_error(e: sqlxError) -> Self {
            match e {
                sqlxError::RowNotFound => {
                    OurError::new_not_found_error(
                        String::from("Not found"),
                        Some(Box::new(e)))
                }
                sqlxError::Database(db) => {
                    if db.code().unwrap_or(Cow::
                    Borrowed("2300")).starts_with("23") {
                        return OurError::new_bad_
                        request_error(
                            String::from("Cannot create or 
                            update resource"),
                            Some(Box::new(db)),
                        );
                    }
                    OurError::new_internal_server_error(
                        String::from("Something went 
                        wrong"),
                        Some(Box::new(db)),
                    )
                }
                _ => OurError::new_internal_server_error(
                    String::from("Something went wrong"),
                    Some(Box::new(e)),
                ),
            }
        }
    }
    ```

1.  在修改我们的 `User` 实体之前，我们需要做一件事。Rust 中的某些 crate 默认不编译 `std` 库，以使生成的二进制文件更小，并嵌入到物联网（IoT）设备或 WebAssembly 中。例如，`argon2` crate 默认不包含 `Error` 特性的实现，因此我们需要启用 `std` 功能。在 `Cargo.toml` 中，修改 `argon2` 依赖项以启用 `std` 库功能：

    ```rs
    argon2 = {version = "0.3", features = ["std"]}
    ```

1.  在 `src/models/user.rs` 文件中，删除 `use std::error::Error;` 并将其替换为 `use crate::errors::our_error::OurError;`。然后，我们可以将 `User` 的方法替换为使用 `OurError`。以下是一个示例：

    ```rs
    pub async fn find(connection: &mut PgConnection, uuid: &str) -> Result<Self, OurError> {
        let parsed_uuid = Uuid::parse_str(
        uuid).map_err(OurError::from_uuid_error)?;
        let query_str = "SELECT * FROM users WHERE uuid = 
        $1";
        Ok(sqlx::query_as::<_, Self>(query_str)
            .bind(parsed_uuid)
            .fetch_one(connection)
            .await
            .map_err(OurError::from_sqlx_error)?)
    }
    ```

1.  对于 `argon2` 错误，我们可以创建一个函数或方法，或者手动转换。例如，在 `src/models/user.rs` 文件中，我们可以这样做：

    ```rs
    let password_hash = argon2
        .hash_password(new_user.password.as_bytes(), 
         &salt)
        .map_err(|e| {
            OurError::new_internal_server_error(
                String::from("Something went wrong"),
                Some(Box::new(e)),
            )
        })?;
    ```

将所有方法更改为使用 `OurError`。提醒一下：你可以在 GitHub 仓库中找到 `src/models/user.rs` 的完整源代码。[`github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter07`](https://github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter07)。

1.  然后，在 `src/routes/user.rs` 文件中，我们将使用 `OurError` 的状态和消息。因为 `Error` 类型已经实现了 `Display` 特性，我们可以在 `format!()` 中直接使用 `e`。以下是一个示例：

    ```rs
    pub async fn get_user(...) -> HtmlResponse {
    ...
        let user = User::find(connection, 
        uuid).await.map_err(|e| e.status)?;
    ...
    }
    ...
    pub async fn delete_user(...) -> Result<Flash<Redirect>, Flash<Redirect>> {
    ...
        User::destroy(connection, uuid)
            .await
            .map_err(|e| Flash::error(Redirect::to("/
             users"), format!("<div>{}</div>", e)))?;
    ...
    }
    ```

你可以在 GitHub 仓库中找到 `src/routes/user.rs` 的完整源代码。[`github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter07`](https://github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter07)。现在我们已经实现了错误处理，可能是一个尝试实现之前在 `src/catchers/mod.rs` 中定义的捕获器的良好时机，以显示默认错误给用户。你还可以在源代码中查看默认捕获器的示例。

在一个应用程序中，跟踪和记录错误是维护应用程序的重要部分。由于我们实现了 `Error` 特性，我们可以在应用程序中记录错误的 `source()`。让我们在下一节中看看如何做到这一点。

# 记录错误

在 Rust 中，有一个日志 crate 提供了应用程序日志的接口。日志提供了五个宏：`error!`、`warn!`、`info!`、`debug!` 和 `trace!`。应用程序可以根据严重性创建日志并过滤需要记录的内容，同样也基于严重性。例如，如果我们基于 `warn` 过滤，那么我们只记录 `error!` 和 `warn!` 并忽略其余内容。由于日志 crate 并没有实现日志记录本身，人们通常使用另一个 crate 来进行实际的实现。在日志 crate 的文档中，我们可以找到其他可用的日志 crate 的示例：`env_logger`、`simple_logger`、`simplelog`、`pretty_env_logger`、`stderrlog`、`flexi_logger`、`log4rs`、`fern`、`syslog` 和 `slog-stdlog`。

让我们在应用程序中实现自定义日志记录。我们将使用 `fern` crate 进行日志记录，并将其包装在 `async_log` 中以实现异步日志记录：

1.  首先，在 `Cargo.toml` 中添加以下这些 crate：

    ```rs
    async-log = "2.0.0"
    fern = "0.6"
    log = "0.4"
    ```

1.  在 `Rocket.toml` 中添加 `log_level` 的配置：

    ```rs
    log_level = "normal"
    ```

1.  然后，我们可以在应用程序中创建一个初始化全局日志记录器的函数。在 `src/main.rs` 中，创建一个名为 `setup_logger` 的新函数：

    ```rs
    fn setup_logger() {}
    ```

1.  在函数内部，让我们初始化日志记录器：

    ```rs
    use log::LevelFilter;
    ...
    let (level, logger) = fern::Dispatch::new()
        .format(move |out, message, record| {
            out.finish(format_args!(
                "[{date}] [{level}][{target}] [{
                 message}]",
                date = chrono::Local::now().format("[
                %Y-%m-%d][%H:%M:%S%.3f]"),
                target = record.target(),
                level = record.level(),
                message = message
            ))
        })
        .level(LevelFilter::Info)
        .chain(std::io::stdout())
        .chain(
            fern::log_file("logs/application.log")
                .unwrap_or_else(|_| panic!("Cannot open 
                logs/application.log")),
        )
        .into_log();
    ```

首先，我们创建一个 `fern::Dispatch` 的新实例。之后，我们使用 `format()` 方法配置输出格式。设置输出格式后，我们使用 `level()` 方法设置日志级别。

对于日志记录器，我们不仅希望将日志输出到操作系统的 `stdout`，还希望写入日志文件。我们可以使用 `chain()` 方法来实现。为了避免恐慌，别忘了在应用程序目录中创建一个 `logs` 文件夹。

1.  在设置好日志级别和日志记录器后，我们将其包装在 `async_log` 中：

    ```rs
    async_log::Logger::wrap(logger, || 0).start(level).unwrap();
    ```

1.  当 `OurError` 被创建时，我们将记录它。在 `src/errors/our_error.rs` 中添加以下行：

    ```rs
    impl OurError {
        fn new_error_with_status(...) ... {
            if debug.is_some() {
                log::error!("Error: {:?}", &debug);
            }
            ...
        }
    }
    ```

1.  将 `setup_logger()` 函数添加到 `src/main.rs` 中：

    ```rs
    async fn rocket() -> Rocket<Build> {
        setup_logger();
    ...
    }
    ```

1.  现在，让我们尝试在应用程序日志中查看 `OurError`。尝试创建具有相同用户名的用户；应用程序应该在终端和 `logs/application.log` 中发出类似以下的重名用户错误：

    ```rs
    [[2021-11-21][17:50:49.366]] [ERROR][our_application::errors::our_error]
    [Error: Some(PgDatabaseError { severity: Error, code: "23505", message:
    "duplicate key value violates unique constraint \"users_username_key\""
    , detail: Some("Key (username)=(karuna) already exists."), hint: None, p
    osition: None, where: None, schema: Some("public"), table: Some("users")
    , column: None, data_type: None, constraint: Some("users_username_key"),
    file: Some("nbtinsert.c"), line: Some(649), routine: Some("_bt_check_un
    ique") })]
    ```

现在我们已经学会了如何记录错误，我们可以实现日志功能来改进应用程序。例如，我们可能想要创建服务器端分析，或者我们可以将日志与第三方监控服务结合，以改进操作并创建商业智能。

# 概述

在本章中，我们学习了一些在 Rust 和 Rocket 应用程序中处理错误的方法。我们可以使用 `panic!`、`Option` 和 `Result` 来传播错误并创建错误处理。

我们还学习了如何创建一个实现 `Error` 特质的自定义类型。该类型可以存储另一个错误，创建一个错误链。

最后，我们学习了在应用程序中记录错误的方法。我们还可以使用日志功能来改进应用程序本身。

我们的用户页面看起来不错，但到处使用`String`显得有些繁琐，所以在下章中，我们将学习如何使用 CSS、JavaScript 以及应用中的其他资源进行模板化。
