# 第十二章：*第十二章*: 测试您的应用程序

确保程序正确运行是编程的一个重要部分。在本章中，我们将学习如何测试 Rust 应用程序。我们将为函数实现一个简单的单元测试，并为创建用户实现一个功能测试。

我们将学习一种简单的技术来调试并找到代码中问题发生的位置。

在学习本章信息后，您将能够为 Rust 和 Rocket 应用程序创建单元测试和功能测试，以确保应用程序按预期工作。您还将学习如何使用`gdb`或`lldb`等调试器调试 Rust 程序。

在本章中，我们将涵盖以下主要主题：

+   测试 Rust 程序

+   测试 Rocket 应用程序

+   调试 Rust 应用程序

# 技术要求

在本章中，我们将进行测试和调试，因此我们需要一个调试器。请为您的操作系统安装`gdb`，GNU 调试器（[`www.sourceware.org/gdb/download/`](https://www.sourceware.org/gdb/download/))）。

您可以在此章节的源代码中找到[`github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter12`](https://github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter12)。

# 测试 Rust 程序

编程的一个重要部分是测试应用程序。有许多种测试，例如单元测试（用于测试单个函数或方法）、功能测试（用于测试应用程序的功能）和集成测试（用于测试各种单元和函数作为一个单一的组合实体）。为了使应用程序尽可能正确，应进行各种测试。

在 Rust 标准库中，有三个宏用于测试：`assert!`、`assert_eq!`和`assert_ne!`。`assert!`宏接受一个或多个参数。第一个参数是任何评估为布尔值的语句，其余的是如果结果不是预期的调试信息。

`assert_eq!`宏比较第一个参数和第二个参数之间的相等性，其余的是如果结果不是预期的调试信息。`assert_ne!`宏是`assert_eq!`的对立面；此宏测试第一个和第二个参数之间的不等性。

让我们看看那些宏在实际中的应用。我们想要测试`src/models/text_post.rs`中`TextPost`模型的`raw_html()`方法。我们想要确保该方法的结果是我们想要的字符串。按照以下步骤测试该方法：

1.  在`src/models/text_post.rs`中添加以下`use`声明：

    ```rs
    use crate::models::our_date_time::OurDateTime;
    use crate::models::post_type::PostType;
    use chrono::{offset::Utc, TimeZone};
    use uuid::Uuid;
    ```

1.  在相同的`src/models/text_post.rs`文件中，我们想要有一个测试函数。为了使函数成为测试函数，使用`#[test]`属性注释该函数。添加函数声明：

    ```rs
    #[test]
    fn test_raw_html() {
    }
    ```

1.  在函数内部，初始化一个`TextPost`实例如下：

    ```rs
    let created_at = OurDateTime(Utc.timestamp_nanos(1431648000000000));
    let post = Post {
        uuid: Uuid::new_v4(),
        user_uuid: Uuid::new_v4(),
        post_type: PostType::Text,
        content: String::from("hello"),
        created_at: created_at,
    };
    let text_post = TextPost::new(&post);
    ```

1.  将`assert!`宏添加到确保结果字符串是我们想要的：

    ```rs
    assert!(
        text_post.raw_html() == 
        String::from("<p>hel1lo</p>"),
        "String is not equal, {}, {}",
        text_post.raw_html(),
        String::from("<p>hello</p>")
    );
    ```

1.  保存文件，通过在终端上运行 `cargo test` 来运行测试。由于我们在测试代码中犯了一个错误 `"<p>hel1lo</p>"`，测试应该失败，如下面的示例所示：

    ```rs
    $ cargo test
       Compiling our_application v0.1.0 (/workspace/
       rocketbook/Chapter12/01RustTesting)
    …
    running 1 test
    test models::text_post::test_raw_html ... FAILED
    failures:
    ---- models::text_post::test_raw_html stdout ----
    thread 'models::text_post::test_raw_html' panicked at 'String is not equal, <p>hello</p>, <p>hello</p>', src/models/text_post.rs:33:5
    note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
    failures:
        models::text_post::test_raw_html
    test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
    error: test failed, to rerun pass '--lib' 
    ```

1.  通过将 `"<p>hel1lo</p>"` 替换为 `"<p>hello</p>"` 来修复 `"<p>hel1lo</p>"`。保存文件并再次运行测试。现在测试应该可以正常工作：

    ```rs
    $ cargo test
       Compiling our_application v0.1.0 (/workspace/
       rocketbook/Chapter12/01RustTesting)
    …
    running 1 test
    test models::text_post::test_raw_html ... ok
    test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
         Running unittests (target/debug/deps/our_
         application-43a2db5b02032f30)
    running 0 tests
    test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
       Doc-tests our_application
    running 0 tests
    test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
    ```

1.  我们想使用 `assert_eq!` 和 `assert_ne!` 宏。`assert_eq!` 宏用于检查第一个参数是否等于第二个参数。`assert_ne!` 宏用于确保第一个参数不等于第二个参数。在 `test_raw_html()` 函数中添加这些宏以查看它们的作用：

    ```rs
    assert_eq!(text_post.raw_html(), String::from("<p>hello</p>"));
    assert_ne!(text_post.raw_html(), String::from("<img>hello</img>"));
    ```

1.  再次运行测试；它应该通过。但是，如果我们查看测试输出，会有警告，如下所示：

    ```rs
    warning: unused import: `crate::models::our_date_time::OurDateTime`
     --> src/models/text_post.rs:1:5
      |
    1 | use crate::models::our_date_time::OurDateTime;
      |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
      |
      = note: `#[warn(unused_imports)]` on by default
    ```

1.  单元测试的一个约定是创建一个测试模块并将模块标记为测试，这样它就不会被编译。在 `src/models/text_post.rs` 中添加一个新的模块：

    ```rs
    #[cfg(test)]
    mod tests {
    }
    ```

1.  在 `src/models/text_post.rs` 文件中，删除这些未使用的 `use` 声明：

    ```rs
    use crate::models::our_date_time::OurDateTime;
    use crate::models::post_type::PostType;
    use chrono::{offset::Utc, TimeZone};
    use uuid::Uuid;
    ```

1.  反之，在 `tests` 模块中的 `src/models/text_post.rs` 中添加所需的 `use` 声明：

    ```rs
    use super::TextPost;
    use crate::models::our_date_time::OurDateTime;
    use crate::models::post::Post;
    use crate::models::post_type::PostType;
    use crate::traits::DisplayPostContent;
    use chrono::{offset::Utc, TimeZone};
    use uuid::Uuid;
    ```

1.  将 `test_raw_html()` 函数移动到 `tests` 模块中。在终端中再次运行 `cargo test`。测试应该没有警告通过，如下面的示例所示：

    ```rs
    $ cargo test
    Finished test [unoptimized + debuginfo] target(s) 
        in 0.34s
         Running unittests (target/debug/deps/our_
         application-40cf18b02419edd7)
    running 1 test
    test models::text_post::tests::test_raw_html ... ok
    test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
         Running unittests (target/debug/deps/our_
         application-77e614e023a036bf)
    running 0 tests
    test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
       Doc-tests our_application
    running 0 tests
    test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
    ```

现在我们已经学会了如何在 Rust 中执行单元测试，我们可以继续在下一节中通过功能测试来测试应用程序。

# 测试 Rocket 应用程序

除了在 `src` 目录中放置测试外，我们还可以在应用程序根目录下的 `tests` 目录中的 Rust 文件中创建测试。当我们运行 `cargo test` 时，命令行会查找 `tests` 目录并运行那里找到的任何测试。人们通常在 `src` 目录中使用测试进行单元测试，并在 `tests` 目录中编写功能测试。

Rocket 框架提供了一个 `rocket::local` 模块，它包含用于向本地 Rocket 应用程序发送请求的模块、结构和方法。向本地 Rocket 应用程序发送非网络请求的主要目的是检查响应并确保响应是我们预期的，主要是为了测试。

让我们尝试通过以下步骤实现应用程序的集成测试：

1.  在应用程序的根目录下，添加一个名为 `tests` 的新目录。在 `tests` 目录内，创建一个名为 `functional_tests.rs` 的文件。

1.  在 `tests/functional_tests.rs` 内部，添加一个新的测试函数，如下所示：

    ```rs
    #[test]
    fn some_test() {
        assert_eq!(2, 1 + 1);
    }
    ```

1.  之后，从命令行保存并运行 `cargo test`。测试应该通过，`cargo test` 的输出应该显示它在 `tests` 目录内运行测试，如下所示：

    ```rs
    ...
    Running tests/functional_tests.rs
    ...
    ```

1.  让我们继续测试 Rocket 应用程序。创建一个名为 `test_rocket` 的测试函数，但由于应用程序是 `async` 的，我们需要不同的测试注解，如下所示：

    ```rs
    #[rocket::async_test]
    async fn test_rocket() {
    }
    ```

1.  我们将把 Rocket 实例放入`rocket::local::asynchronous::Client`实例中。稍后，我们可以使用`Client`实例发送请求并验证响应。但，一个问题在于 Rocket 初始化在`src/main.rs`中，而不是在`our_application`库中。我们可以通过将 Rocket 初始化从`src/main.rs`移动到`src/lib.rs`来解决这个问题。将`src/main.rs`中的代码移动到`src/lib.rs`下的`pub mod`声明下，然后更改任何`use our_application::`到`use crate::`。

之后，将`rocket()`函数重命名为`setup_rocket()`。此外，在`setup_rocket()`函数前面添加`pub`并从`setup_rocket()`函数的顶部移除`#[launch]`。

我们需要一个方法来获取数据库 URL，因此实现`Config`在`src/lib.rs`中的`get_database_url`方法：

```rs
impl Config {
    pub fn get_database_url(&self) -> String {
        self.databases.main_connection.url.clone()
    }
}
```

1.  在`src/main.rs`中，将应用程序更改为使用`setup_rocket()`，如下所示：

    ```rs
    use our_application::setup_rocket;
    use rocket::{Build, Rocket};
    #[launch]
    async fn rocket() -> Rocket<Build> {
        setup_rocket().await
    }
    ```

1.  返回到`tests/functional_test.rs`，添加`our_application`库：

    ```rs
    use our_application;
    ```

然后，在`test_rocket()`函数中初始化一个 Rocket 实例，如下所示：

```rs
let rocket = our_application::setup_rocket().await;
```

1.  我们希望获取数据库连接以截断数据库表，确保测试的干净状态。添加所需的`use`声明：

    ```rs
    use our_application::Config;
    use rocket_db_pools::sqlx::PgConnection;
    use sqlx::postgres::PgPoolOptions;
    ```

然后，在`test_rocket()`函数内部添加以下行：

```rs
let config_wrapper = rocket.figment().extract();
assert!(config_wrapper.is_ok());
let config: Config = config_wrapper.unwrap();
let db_url = config.get_database_url();
let db_wrapper = PgPoolOptions::new()
    .max_connections(5)
    .connect(&db_url)
    .await;
assert!(db_wrapper.is_ok());
let db = db_wrapper.unwrap();
```

1.  我们希望截断`users`表的内容。我们希望为`User`创建一个删除所有数据的方法，但该方法应仅适用于测试。让我们添加一个 trait 来扩展`User`模型。添加`use`声明：

    ```rs
    use our_application::models::user::User;
    ```

添加`ModelCleaner` trait 并为`User`实现`ModelCleaner`，如下所示：

```rs
#[rocket::async_trait]
trait ModelCleaner {
    async fn clear_all(connection: &mut PgConnection) 
    -> Result<(), String>;
}
#[rocket::async_trait]
impl ModelCleaner for User {
    async fn clear_all(connection: &mut PgConnection) 
    -> Result<(), String> {
        let _ = sqlx::query("TRUNCATE users RESTART 
        IDENTITY CASCADE")
            .execute(connection)
            .await
            .map_err(|_| String::from("error 
            truncating databasse"))?;
        Ok(())
    }
}
```

1.  在`test_rocket()`函数中继续，添加以下行：

    ```rs
    let pg_connection_wrapper = db.acquire().await;
    assert!(pg_connection_wrapper.is_ok());
    let mut pg_connection = pg_connection_wrapper.unwrap();
    let clear_result_wrapper = User::clear_all(&mut pg_connection).await;
    assert!(clear_result_wrapper.is_ok());
    ```

1.  在`tests/functional_tests.rs`文件中添加`use`声明：

    ```rs
    use rocket::local::asynchronous::Client;
    ```

然后，在`test_rocket()`函数内部创建一个`Client`实例：

```rs
let client_wrapper = Client::tracked(rocket).await;
assert!(client_wrapper.is_ok());
let client = client_wrapper.unwrap();
```

1.  目前，数据库中的用户数量是`0`。我们希望通过获取`"/users"`并解析 HTML 来进行测试。一个用于解析 HTML 的 crate 是`scraper`。因为我们只想在测试中使用`scraper` crate，所以请在`Cargo.toml`中添加一个新的部分，称为`[dev-dependencies]`，如下所示：

    ```rs
    [dev-dependencies]
    scraper = "0.12.0"
    ```

1.  返回到`tests/functional_test.rs`，我们想要获取`"/users"`响应。添加`use`声明：

    ```rs
    use rocket::http::Status;
    ```

然后，在`test_rocket()`函数内部追加以下行：

```rs
let req = client.get("/users");
let resp = req.dispatch().await;
assert_eq!(resp.status(), Status::Ok);
```

1.  我们希望验证响应体不包含任何用户。如果我们查看`src/views/users/index.html.tera`模板，我们会看到每个用户都有一个`mark` HTML 标签。让我们使用`scraper`通过添加`use`声明来验证响应：

    ```rs
    use scraper::{Html, Selector}; 
    ```

然后，在`test_rocket()`函数内部追加以下行：

```rs
let body_wrapper = resp.into_string().await;
assert!(body_wrapper.is_some());
let body = Html::parse_document(&body_wrapper.unwrap());
let selector = Selector::parse(r#"mark.tag"#).unwrap();
let containers = body.select(&selector);
let num_of_elements = containers.count();
assert_eq!(num_of_elements, 0);
```

1.  我们希望创建一个`post`请求来创建新用户，但一个问题在于应用程序将执行 token 真实性检查，因此我们需要首先从`"/users/new"`页面获取值。将以下行追加以从响应体中获取 token：

    ```rs
    let req = client.get("/users/new");
    let resp = req.dispatch().await;
    assert_eq!(resp.status(), Status::Ok);
    let body_wrapper = resp.into_string().await;
    assert!(body_wrapper.is_some());
    let body = Html::parse_document(&body_wrapper.unwrap());
    let authenticity_token_selector = Selector::parse(r#"input[name="authenticity_token"]"#).unwrap();
    let element_wrapper = body.select(&authenticity_token_selector).next();
    assert!(element_wrapper.is_some());
    let element = element_wrapper.unwrap();
    let value_wrapper = element.value().attr("value");
    assert!(value_wrapper.is_some());
    let authenticity_token = value_wrapper.unwrap();
    ```

1.  使用`authenticity_token`发送`post`请求。添加`use`声明：

    ```rs
    use rocket::http::ContentType;
    ```

1.  然后，将以下行追加到`test_rocket()`函数中：

    ```rs
    let username = "testing123";
    let password = "lkjKLAJ09231478mlasdfkjsdkj";
    let req = client.post("/users")
        .header(ContentType::Form)
        .body(
            format!("authenticity_token={
            }&username={}&email={}@{}
            .com&password={}&password_confirmation={}
            &description=",
        authenticity_token, username, username, username, 
        password, password,
    ));
    let resp = req.dispatch().await;
    assert_eq!(resp.status(), Status::SeeOther);
    ```

1.  最后再次检查 `"/users"` 页面；您应该看到一个用户。追加以下行：

    ```rs
    let req = client.get("/users");
    let resp = req.dispatch().await;
    assert_eq!(resp.status(), Status::Ok);
    let body_wrapper = resp.into_string().await;
    assert!(body_wrapper.is_some());
    let body = Html::parse_document(&body_wrapper.unwrap());
    let selector = Selector::parse(r#"mark.tag"#).unwrap();
    let containers = body.select(&selector);
    let num_of_elements = containers.count();
    assert_eq!(num_of_elements, 1);
    ```

再次尝试运行测试。有时测试会成功：

```rs
$ cargo test
...
     Running tests/functional_tests.rs (target/debug/deps/functional_tests-625b16e4b25b72de)
running 2 tests
test some_test ... ok
...
test test_rocket ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 2.43s
```

但是，有时测试不会成功：

```rs
$ cargo test
...
     Running tests/functional_tests.rs (target/
     debug/deps/functional_tests-625b16e4b25b72de)
running 2 tests
test some_test ... ok
...
test test_rocket ... FAILED
failures:
---- test_rocket stdout ----
thread 'test_rocket' panicked at 'assertion failed: `(left == right)`
  left: `0`,
 right: `1`', tests/functional_tests.rs:115:5
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
thread 'rocket-worker-test-thread' panicked at 'called `Result::unwrap()` on an `Err` value: Disconnected',
/workspace/rocketbook/Chapter12/03RocketTesting/src/lib.rs:137:28
failures:
    test_rocket
test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 2.09s
```

测试为什么失败？我们将在下一节中学习如何调试 Rust 程序。

# 调试 Rust 应用程序

在上一节中，我们学习了如何编写功能测试，但有时测试会失败。我们想知道为什么测试失败了。错误可能发生的两个可能位置是用户创建过程和创建用户后的用户查找过程。

一种调试方法是记录错误可能发生的位置。如果我们记录用户创建过程中所有可能发生的错误（例如，在 `src/routes/user.rs` 的 `create_user()` 函数中），我们会发现身份验证令牌验证有时会产生错误。记录错误的示例如下：

```rs
csrf_token
```

```rs
    .verify(&new_user.authenticity_token)
```

```rs
    .map_err(|err| {
```

```rs
        log::error!("Verify authenticity_token error: {}", 
```

```rs
        err);
```

```rs
        Flash::error(
```

```rs
            Redirect::to("/users/new"),
```

```rs
            "Something went wrong when creating user",
```

```rs
        )
```

```rs
    })?;
```

如果我们继续记录 `verify()` 方法并继续追踪问题的来源，我们最终会发现令牌的 `from_request()` 方法没有产生正确的结果。我们可以通过更改 `src/fairings/csrf.rs` 中的 `from_request()` 方法来修复问题，如下所示：

```rs
async fn from_request(request: &'r Request<'_>) -> Outcome<Self, Self::Error> {
```

```rs
    match request.get_csrf_token() {
```

```rs
        None => Outcome::Failure((Status::Forbidden, ())),
```

```rs
        Some(token) => Outcome::Success(Self(base64::
```

```rs
        encode_config(token, base64::URL_SAFE))),
```

```rs
    }
```

```rs
}
```

显然，记录代码并查找问题并不高效。我们还可以使用调试器，如 `gdb`（GNU 调试器）或 `lldb` 来调试 Rust 程序。`gdb` 可用于 Linux 操作系统，而 `lldb`（LLVM 项目的调试器）可用于 macOS 和 Linux 操作系统。如果您想为 Rust 编程语言使用调试器，请安装这些调试器之一。

Rust 提供了 `rust-gdb`（`gdb` 的包装器）和 `rust-lldb`（`lldb` 的包装器）。这些程序应与 Rust 编译器一起安装。让我们通过以下步骤看看如何使用 `rust-gdb` 的示例：

1.  首先，使用终端上的 `cargo build` 命令构建应用程序。由于我们不是构建发布版本，调试符号应该包含在生成的二进制文件中。

1.  检查生成的二进制文件在源目录目标目录中的位置；例如，如果应用程序的源代码在 `/workspace/rocketbook/Chapter12/03RocketTesting/`，我们可以在 `/workspace/rocketbook/Chapter12/03RocketTesting/target/debug/our_application` 中找到生成的二进制文件。

1.  在终端上像运行 `gdb` 一样运行 `rust-gdb`。以下是一个示例：

    ```rs
    rust-gdb -q target/debug/our_application
    ```

1.  您将看到一个 `gdb` 提示符，如下所示：

    ```rs
    Reading symbols from target/debug/our_application...
    Reading symbols from /workspace/rocketbook/Chapter12/03RocketTesting/target/debug/our_application/Contents/Resources/DWARF/our_application...
    (gdb)
    ```

1.  设置应用程序的断点，例如：

    ```rs
    b /workspace/rocketbook/Chapter12/03RocketTesting/src/lib.rs:143
    ```

1.  您将看到一个提示设置 `our_application` 库的断点，如下所示：

    ```rs
    No source file named /workspace/rocketbook/Chapter12/03RocketTesting/src/lib.rs.
    Make breakpoint pending on future shared library load? (y or [n])
    ```

1.  回复 `y` 并注意以下设置的 `Breakpoint 1`：

    ```rs
    (y or [n]) y
    Breakpoint 1 (/workspace/rocketbook/Chapter12/03RocketTesting/src/lib.rs:143) pending.
    ```

1.  在 `gdb` 提示符下编写 `r` 命令并按 *Enter* 键运行应用程序：

    ```rs
    (gdb) r
    ```

1.  应用程序应该运行，因为它触发了断点，执行停止。我们可以再次使用 `gdb` 提示符来检查 final_rocket，如下例所示：

    ```rs
    Starting program: /workspace/rocketbook/Chapter12/03RocketTesting/target/debug/our_application 
      [Thread debugging using libthread_db enabled]
    Using host libthread_db library 
      "/usr/lib/libthread_db.so.1".
      [New Thread 0x7ffff7c71640 (LWP 50269)]
      ...
      [New Thread 0x7ffff746d640 (LWP 50273)]
      Thread 1 "our_application" hit Breakpoint 1, our_
    application::main::{generator#0} () at 
      src/lib.rs:143
      143         final_rocket
    ```

1.  在调试器提示符中尝试打印一些变量：

    ```rs
    (gdb) p config
    ```

我们可以看到以下打印的结果：

```rs
$1 = our_application::Config {databases: our_application::Databases {main_connection: our_application::MainConnection {url: "postgres://username:password@localhost/rocket"}}, jwt_secret: "+/xbAZJs+e1BA4
gbv2zPrtkkkOhrYmHUGnJIoaL9Qsk="}
```

1.  要退出`gdb`，只需在提示符下输入`quit`并确认退出调试器，如下所示：

    ```rs
    (gdb) quit
    A debugging session is active.
            Inferior 1 [process 50265] will be killed.
    Quit anyway? (y or n) y
    ```

这些调试器有许多更多功能，例如设置多个断点和通过断点逐步执行。您可以在[`www.sourceware.org/gdb/`](https://www.sourceware.org/gdb/)找到有关`gdb`的更多信息，以及[`lldb.llvm.org/`](https://lldb.llvm.org/)有关`lldb`的更多信息。

此外，还有适用于 IDE 或代码编辑器的调试器，例如，Visual Studio Code 的用户可以使用 CodeLLDB ([`marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb`](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb)) 从编辑器方便地单击行并标记断点，并通过专用面板检查变量：

![图 12.1 – CodeLLDB 检查我们的应用程序](img/Figure_12.1_B16825.jpg)

图 12.1 – CodeLLDB 检查我们的应用程序

在任何情况下，使用调试器是编程不可或缺的工具。学会正确使用调试器可以帮助我们更好地使用 Rust 编程语言。

# 摘要

在本章中，我们学习了测试 Rust 程序和 Rocket 应用程序。我们学习了使用如`assert!`这样的宏进行测试。我们还学习了单元测试和功能测试之间的区别。

我们创建了一个功能测试，并了解了在 Rocket 应用程序上进行功能测试的模块。最后，我们学习了一些调试 Rust 应用程序的技术，以帮助修复它。

测试和调试是编程的重要组成部分，因为这些技术可以提高应用程序的正确性。

在所有开发完成后，在下一章中，我们将学习几种将 Rocket 应用程序设置为可供现实世界用户使用的方法。
