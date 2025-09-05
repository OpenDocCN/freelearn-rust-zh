

# 第九章：测试我们的应用程序端点和组件

我们的任务 Rust 应用程序现在完全工作。我们对我们的第一个版本感到满意，因为它管理着身份验证、不同用户及其待办事项列表，并记录我们的流程以供检查。然而，一个网络开发者的工作永远不会结束。

虽然我们现在已经完成了为我们的应用程序添加功能的工作，但我们知道旅程并没有在这里结束。在未来这本书之外的迭代中，我们可能希望添加团队、新的状态、每个用户的多列表等。然而，当我们添加这些功能时，我们必须确保我们的旧应用程序的行为保持不变，除非我们主动改变它。这是通过构建测试来实现的。

在本章中，我们将构建检查我们现有行为的测试，设置陷阱，如果应用程序的行为在没有我们主动改变的情况下发生变化，则会抛出错误并报告给我们。这防止我们在添加新功能或更改代码后破坏应用程序并将其推送到服务器。

在本章中，我们将涵盖以下主题：

+   构建我们的单元测试

+   构建 JWT 单元测试

+   在 Postman 中编写功能 API 测试

+   使用 Newman 自动化 Postman 测试

+   构建整个自动化测试管道

到本章结束时，我们将了解如何在 Rust 中构建单元测试，通过一系列边缘情况详细检查我们的结构体。如果我们的结构体以我们预料之外的方式表现，我们的单元测试将向我们报告。

# 技术要求

在本章中，我们将基于第八章中构建的代码，即*构建 RESTful 服务*。这可以在[`github.com/PacktPublishing/Rust-Web-Programming-2nd-Edition/tree/main/chapter08/caching`](https://github.com/PacktPublishing/Rust-Web-Programming-2nd-Edition/tree/main/chapter08/caching)找到。

Node 和 NPM 也需要用于安装和运行自动化 API 测试，可以在[`nodejs.org/en/download/`](https://nodejs.org/en/download/)找到。

我们还将运行自动化测试管道的一部分，使用 Python。Python 可以从[`www.python.org/downloads/`](https://www.python.org/downloads/)下载并安装。

您可以在此处找到本章使用的完整源代码：[`github.com/PacktPublishing/Rust-Web-Programming-2nd-Edition/tree/main/chapter09`](https://github.com/PacktPublishing/Rust-Web-Programming-2nd-Edition/tree/main/chapter09)。

# 构建我们的单元测试

在本节中，我们将探讨单元测试的概念以及如何构建包含测试函数的单元测试模块。在这里，我们不会为我们的应用程序实现 100%的单元测试覆盖率。在我们的应用程序中，有些地方可以通过我们的功能测试来覆盖，例如 API 端点和 JSON 序列化。然而，单元测试在我们的应用程序的某些部分仍然很重要。

单元测试使我们能够更详细地查看我们的某些流程。正如我们在*第八章*“构建 RESTful 服务”中看到的那样，一个功能测试可能按预期的方式从端到端工作，但可能会有我们不希望出现的边缘情况和行为。这在上一章中有所体现，我们看到了应用程序在只需要一个 `GET` 请求的情况下进行了两次 `GET` 调用。

在我们的单元测试中，我们将逐一分解流程，模拟某些参数，并测试结果。这些测试是完全隔离的。这种优势在于我们可以快速测试一系列参数，而无需每次都运行整个流程。这也帮助我们精确地确定应用程序失败的位置和配置。单元测试对于测试驱动开发也很有用，在那里我们逐步构建一个功能的组件，运行单元测试，并根据测试结果修改组件。

在大型、复杂的系统中，这可以节省大量时间，因为你不需要启动应用程序并运行整个系统来查找错误或处理边缘情况。

然而，在我们过于兴奋之前，我们必须承认单元测试是一种工具，而不是一种生活方式，并且使用它有一些局限性。测试的质量取决于它们的模拟。如果我们不模拟真实的交互，那么单元测试可能会通过，但应用程序可能会失败。单元测试很重要，但它们也必须与功能测试相结合。

Rust 仍然是一门新兴语言，因此到目前为止，单元测试的支持并不像 Python 或 Java 等其他语言那样先进。例如，在使用 Python 时，我们可以在测试的任何阶段轻松地模拟任何文件中的任何对象。有了这些模拟，我们可以定义结果并监控交互。虽然 Rust 并没有这样现成的模拟，但这并不意味着我们不能进行单元测试。

*拙匠常怪其工具*。成功的单元测试背后的工艺是构建我们的代码，使各个代码块尽可能独立，这样各个部分就会尽可能具有自主性。正因为这种缺乏依赖性，测试可以很容易地进行，而无需复杂的模拟系统。

首先，我们可以测试我们的待办事项结构体。如您所记得，我们有 `done` 和 `pending` 结构体，它们继承了一个 `base` 结构体。我们可以从单元测试没有依赖的结构体开始，然后向下移动到有依赖的其他结构体。在我们的 `src/to_do/structs/base.rs` 文件中，我们可以在文件底部定义 `base` 结构体的单元测试，如下面的代码所示：

```rs
#[cfg(test)]
mod base_tests {
    use super::Base;
    use super::TaskStatus;
    #[test]
    fn new() {
        let expected_title = String::from("test title");
        let expected_status = TaskStatus::DONE;
        let new_base_struct = Base{
            title: expected_title.clone(),
            status: TaskStatus::DONE
        };
        assert_eq!(expected_title,
                   new_base_struct.title);
        assert_eq!(expected_status,
                   new_base_struct.status);
    }
}
```

在前面的代码中，我们仅仅创建了一个结构体，并评估了该结构体的字段，确保它们是我们预期的。我们可以看到我们创建了一个带有`#[cfg(test)]`属性的`test`模块。`#[cfg(test)]`属性是一个条件检查，只有当我们运行`cargo test`时，代码才是活跃的。如果我们不运行`cargo test`，则带有`#[cfg(test)]`属性的代码不会被编译。

在模块内部，我们将从`base_tests`模块外部的文件中导入`Base`结构体，该模块仍然位于文件中。在 Rust 的世界里，使用`super`导入我们正在测试的内容是典型的。有一个既定的标准，即测试代码应该位于同一文件中被测试代码的下方。然后，我们将通过在`new`函数上添加`#[test]`属性来测试`Base::new`函数。

这是我们第一次介绍属性。**属性**简单来说就是应用于模块和函数的元数据。这些元数据通过提供信息来帮助编译器。在这种情况下，它是在告诉编译器这个模块是一个测试模块，而这个函数是一个单独的测试。

然而，如果我们运行前面的代码，它将不会工作。这是因为`TaskStatus`枚举中没有实现`Eq`特质，这意味着我们无法执行以下代码行：

```rs
assert_eq!(expected_status, new_base_struct.status);
```

这也意味着我们无法在两个`TaskStatus`枚举之间使用`==`运算符。因此，在我们尝试运行测试之前，我们必须在`src/to_do/structs/enums.rs`文件中实现`TaskStatus`枚举的`Eq`特质，以下为相关代码：

```rs
#[derive(Clone, Eq, Debug)]
pub enum TaskStatus {
    DONE,
    PENDING
}
```

我们可以看到我们已经实现了`Eq`和`Debug`特质，这些特质对于`assert_eq!`宏是必需的。然而，我们的测试仍然无法运行，因为我们还没有定义两个`TaskStatus`枚举之间等价的规则。我们可以通过简单地将`PartialEq`特质添加到我们的`derive`注解中来实现`PartialEq`特质。然而，我们应该探索如何编写我们自己的自定义逻辑。为了定义等价规则，我们在`PartialEq`特质下实现`eq`函数，以下为相关代码：

```rs
impl PartialEq for TaskStatus {
    fn eq(&self, other: &Self) -> bool {
        match self {
            TaskStatus::DONE => {
                match other {
                    &TaskStatus::DONE => return true,
                    &TaskStatus::PENDING => false
                }
            },
            TaskStatus::PENDING => {
                match other {
                    &TaskStatus::DONE => return false,
                    &TaskStatus::PENDING => true
                }
            }
        }
    }
}
```

在这里，我们可以看到我们通过两个`match`语句成功确认了`TaskStatus`枚举是否等于正在比较的其他`TaskStatus`枚举。在`eq`函数中使用`==`运算符似乎更直观；然而，使用`==`运算符会调用`eq`函数，从而导致无限循环。如果你在`eq`函数中使用`==`运算符，代码仍然可以编译，但如果你运行它，你会得到以下无用的错误：

```rs
fatal runtime error: stack overflow
```

我们现在实际上创建了一个新的`base`结构体，然后检查字段是否符合我们的预期。要运行此操作，请使用`cargo test`功能，指向我们想要测试的文件，以下为相关命令：

```rs
cargo test to_do::structs::base
```

我们将得到以下输出：

```rs
running 1 test
test to_do::structs::base::base_tests::new ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0
filtered out; finished in 0.00s
```

我们可以看到我们的测试已经运行并通过了。现在，我们将继续编写模块其余部分的测试，这些部分是`Done`和`Pending`结构体。现在是时候看看你能否在`src/to_do/structs/done.rs`文件中编写一个基本的单元测试了。如果你已经尝试在`src/to_do/structs/done.rs`文件中为`Done`结构体编写单元测试，你的代码应该看起来像以下代码：

```rs
#[cfg(test)]
mod done_tests {
    use super::Done;
    use super::TaskStatus;
    #[test]
    fn new() {
        let new_base_struct = Done::new("test title");
        assert_eq!(String::from("test title"),
                   new_base_struct.super_struct.title);
        assert_eq!(TaskStatus::DONE,
                   new_base_struct.super_struct.status);
    }
}
```

我们可以使用以下命令运行这两个测试：

```rs
cargo test
```

这给出了以下输出：

```rs
running 2 tests
test to_do::structs::base::base_tests::new ... ok
test to_do::structs::done::done_tests::new ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; 0
measured; 0 filtered out; finished in 0.00s
```

运行`cargo test`将在所有 Rust 文件中运行所有测试。我们可以看到现在所有的测试都已经运行并通过了。

现在我们已经进行了一些基本的测试，让我们看看我们可以测试的其他模块。我们的 JSON 序列化和视图可以通过**Postman**在我们的功能测试中进行测试。我们的数据库模型没有我们故意定义的任何高级功能。

# 构建 JWT 单元测试

我们所有的模型所做的只是读取和写入数据库。这已经被证明是可行的。我们唯一剩下的要单元测试的模块是`auth`模块。在这里，我们有一些基于输入有多个结果的逻辑。我们还必须做一些模拟，因为一些函数接受`actix_web`结构体，这些结构体有特定的字段和函数。幸运的是，`actix_web`有一个测试模块，它使我们能够模拟请求。

## 构建测试配置

在我们开始为 JWT 构建单元测试之前，我们必须记住，有一个对`config`文件的依赖，以获取密钥。单元测试必须是隔离的。它们不应该需要传递正确的参数才能工作。它们应该每次都是隔离地工作。正因为如此，我们将在`src/config.rs`文件中为我们的`Config`结构体构建一个`new`函数。编码测试的大纲将看起来像以下代码：

```rs
impl Config {
    // existing function reading from file
    #[cfg(not(test))]
    pub fn new() -> Config {
        . . .
    }
    // new function for testing
    #[cfg(test)]
    pub fn new() -> Config {
        . . .
    }
}
```

上述大纲显示有两个`new`函数。我们的新`new`函数在运行测试时被编译，如果服务器以正常方式运行，则旧`new`函数被编译。我们的测试`new`函数有标准值硬编码在以下代码中：

```rs
let mut map = HashMap::new();
map.insert(String::from("DB_URL"),
           serde_yaml::from_str(
           "postgres://username:password@localhost:5433/
           to_do").unwrap());
map.insert(String::from("SECRET_KEY"),
           serde_yaml::from_str("secret").unwrap());
map.insert(String::from("EXPIRE_MINUTES"),
           serde_yaml::from_str("120").unwrap());
map.insert(String::from("REDIS_URL"),
           serde_yaml::from_str("redis://127.0.0.1/")
           .unwrap());
return Config {map}
```

这些默认函数与我们的开发`config`文件相同；然而，我们知道这些变量将是一致的。我们在运行测试时不需要传递任何东西，我们也不存在读取另一个文件的风险。现在我们的测试已经配置好了，我们可以定义要求，包括 JWT 测试的配置。

## 定义 JWT 测试的要求

现在我们已经为测试安全地构建了`Config`结构体，我们可以转到我们的`src/jwt.rs`文件，并使用以下代码定义测试的导入：

```rs
#[cfg(test)]
mod jwt_tests {
    use std::str::FromStr;
    use super::{JwToken, Config};
    use actix_web::{HttpRequest, HttpResponse,
                    test::TestRequest, web, App};
    use actix_web::http::header::{HeaderValue,
                                  HeaderName, ContentType};
    use actix_web::test::{init_service, call_service};
    use actix_web;
    use serde_json::json;
    use serde::{Deserialize, Serialize};
    #[derive(Debug, Serialize, Deserialize)]
    pub struct ResponseFromTest {
        pub user_id: i32,
        pub exp_minutes: i32
    }
    . . .
}
```

通过前面的代码，我们可以导入一系列 `actix_web` 结构体和函数，使我们能够创建伪造的 HTTP 请求并将它们发送到伪造的应用程序以测试 `JwToken` 结构体在 HTTP 请求过程中的工作情况。我们还将定义一个 `ResponseFromTest` 结构体，它可以被处理为 JSON，以从 HTTP 请求中提取用户 ID，因为 `JwToken` 结构体包含了用户 ID。`ResponseFromTest` 结构体是我们期望得到的 HTTP 响应，因此我们正在紧密模拟响应对象。

现在我们已经导入了所有需要的，我们可以用以下代码定义测试的轮廓：

```rs
#[cfg(test)]
mod jwt_tests {
    . . .
    #[test]
    fn get_key() {
        . . .
    }
    #[test]
    fn get_exp() {
        . . .
    }
    #[test]
    fn decode_incorrect_token() {
        . . .
    }
    #[test]
    fn encode_decode() {
        . . .
    }
    async fn test_handler(token: JwToken,
                          _: HttpRequest) -> HttpResponse {
        . . .
    }
    #[actix_web::test]
    async fn test_no_token_request() {
        . . .
    }
    #[actix_web::test]
    async fn test_passing_token_request() {
        . . .
    }
    #[actix_web::test]
    async fn test_false_token_request() {
        . . .
    }
}
```

在这里，我们可以看到我们测试了获取密钥和令牌的编码和解码。它们是 `JwToken` 结构体的原生函数，并且根据我们之前所讨论的，你应该能够自己编写它们。其他函数被装饰为 `#[actix_web::test]`。这意味着我们将创建伪造的 HTTP 请求来测试我们的 `JwToken` 如何实现 `FromRequest` 特性。现在，没有什么阻止我们编写测试，我们将在下一节中介绍。

## 为 JWT 构建基本功能测试

我们将从最基础的测试开始，获取密钥，其形式如下：

```rs
#[test]
fn get_key() {
    assert_eq!(String::from("secret"), JwToken::get_key());
}
```

我们必须记住 `"secret"` 是在 `Config::new` 函数中硬编码的密钥，用于测试实现。如果 `Config::new` 测试函数工作正常，上述测试也将工作正常。获取过期时间也可能很重要。因为我们直接依赖于从 `config` 中提取的过期分钟数，所以以下测试将确保我们返回 120 分钟：

```rs
#[test]
fn get_exp() {
    let config = Config::new();
    let minutes = config.map.get("EXPIRE_MINUTES")
                      .unwrap().as_i64().unwrap();
    assert_eq!(120, minutes);
}
```

我们现在可以继续测试如何处理无效令牌，以下是一个测试示例：

```rs
#[test]
fn decode_incorrect_token() {
    let encoded_token: String =
        String::from("invalid_token");
    match JwToken::from_token(encoded_token) {
        Err(message) => assert_eq!("InvalidToken",
                                    message),
        _ => panic!(
            "Incorrect token should not be able to be
             encoded"
             )
    }
}
```

在这里，我们传递一个 `"invalid_token"` 字符串，它应该会失败解码过程，因为它显然不是一个有效的令牌。然后我们将匹配结果。如果结果是错误，我们将断言错误信息是无效令牌的结果。如果有任何其他输出而不是错误，那么我们将抛出一个错误，使测试失败，因为我们期望解码会失败。

现在我们已经为我们的 `JwToken` 结构体函数编写了两个测试，这是一个很好的时机让你尝试编写编码和解码令牌的测试。如果你尝试编写了编码和解码测试，它应该看起来像以下代码：

```rs
#[test]
fn encode_decode() {
    let test_token = JwToken::new(5);
    let encoded_token = test_token.encode();
    let new_token =
        JwToken::from_token(encoded_token).unwrap();
    assert_eq!(5, new_token.user_id);
}
```

前面的测试实际上将登录和认证请求过程简化为围绕令牌。我们创建一个新的带有用户 ID 的令牌，对令牌进行编码，然后解码令牌以测试我们传递到令牌中的数据是否与我们解码时得到的数据相同。如果不相同，则测试将失败。

现在我们已经完成了对`JwToken`结构体函数的测试，我们可以继续测试`JwToken`结构体如何实现`FromRequest`特质。在我们这样做之前，我们必须定义一个基本的视图函数，该函数将仅处理`JwToken`的认证，然后返回令牌中的用户 ID，以下代码所示：

```rs
async fn test_handler(token: JwToken,
                      _: HttpRequest) -> HttpResponse {
    return HttpResponse::Ok().json(json!({"user_id":
                                           token.user_id,
                                          "exp_minutes":
                                           60}))
}
```

这并不是什么新东西，事实上，这个大纲也是我们定义应用程序中视图的方式。有了我们的基本测试定义，我们可以继续构建针对 Web 请求的测试。

## 构建 Web 请求的测试

我们现在可以使用以下代码来测试我们的测试视图，看看它如何处理头部没有令牌的请求：

```rs
#[actix_web::test]
async fn test_no_token_request() {
    let app = init_service(App::new().route("/", web::get()
                               .to(test_handler))).await;
    let req = TestRequest::default()
        .insert_header(ContentType::plaintext())
        .to_request();
    let resp = call_service(&app, req).await;
    assert_eq!("401", resp.status().as_str());
}
```

在前面的代码中，我们可以看到我们可以创建一个假服务器并将我们的`test_handler`测试视图附加到它上。然后我们可以创建一个没有令牌的假请求。然后我们将使用假请求调用服务器，并断言请求的响应代码为未授权。我们现在可以创建一个插入有效令牌的测试，以下代码所示：

```rs
#[actix_web::test]
async fn test_passing_token_request() {
    let test_token = JwToken::new(5);
    let encoded_token = test_token.encode();
    let app = init_service(App::new().route("/", web::get()
                               .to(test_handler))).await;
    let mut req = TestRequest::default()
        .insert_header(ContentType::plaintext())
        .to_request();
    let header_name = HeaderName::from_str("token")
                                            .unwrap();
    let header_value = HeaderValue::from_str(encoded_token
                                             .as_str())
                                             .unwrap();
    req.headers_mut().insert(header_name, header_value);
    let resp: ResponseFromTest = actix_web::test::
        call_and_read_body_json(&app, req).await;
    assert_eq!(5, resp.user_id);
}
```

在这里，我们可以看到我们创建了一个有效的令牌。我们可以创建我们的假服务器并将我们的`test_handler`函数附加到那个假服务器上。然后我们将创建一个可以变异的请求。然后，我们将令牌插入到头部，并使用`call_and_read_body_json`函数调用假服务器，使用假请求。必须注意的是，当我们调用`call_and_read_body_json`函数时，我们声明在`resp`变量名下返回的类型是`ResponseFromTest`。然后我们断言用户 ID 来自请求响应。

现在我们已经看到了如何创建带有头部的假 HTTP 请求，这是一个很好的机会让你尝试构建一个请求带有无法解码的假令牌的测试。如果你已经尝试过，它应该看起来像以下代码：

```rs
#[actix_web::test]
async fn test_false_token_request() {
    let app = init_service(App::new().route("/", web::get()
                  .to(test_handler))).await;
    let mut req = TestRequest::default()
        .insert_header(ContentType::plaintext())
        .to_request();
    let header_name = HeaderName::from_str("token")
        .unwrap();
    let header_value = HeaderValue::from_str("test")
        .unwrap();
    req.headers_mut().insert(header_name, header_value);
    let resp = call_service(&app, req).await;
    assert_eq!("401", resp.status().as_str());
}
```

观察以下代码，我们可以看到我们使用在通过令牌请求测试中概述的方法将一个假令牌插入到头部，并在没有提供令牌的测试中使用未授权断言。如果我们现在运行所有测试，我们应该得到以下输出：

```rs
running 9 tests
test to_do::structs::base::base_tests::new ... ok
test to_do::structs::done::done_tests::new ... ok
test to_do::structs::pending::pending_tests::new ... ok
test jwt::jwt_tests::get_key ... ok
test jwt::jwt_tests::decode_incorrect_token ... ok
test jwt::jwt_tests::encode_decode ... ok
test jwt::jwt_tests::test_no_token_request ... ok
test jwt::jwt_tests::test_false_token_request ... ok
test jwt::jwt_tests::test_passing_token_request ... ok
test result: ok. 9 passed; 0 failed; 0 ignored;
0 measured; 0 filtered out; finished in 0.00s
```

从前面的输出中，我们的`jwt`和`to_do`模块现在已经完全进行了单元测试。考虑到 Rust 仍然是一种新的语言，我们设法无痛地单元测试了我们的代码，因为我们以模块化的方式结构了我们的代码。

`actix_web`提供的`tests` crate 使我们能够快速轻松地测试边缘情况。在本节中，我们测试了我们的函数如何处理缺少令牌、假令牌和正确令牌的请求。我们亲眼见证了 Rust 如何使我们能够在代码上运行单元测试。

所有的配置都是通过`cargo`完成的。我们不需要设置路径，安装额外的模块，或配置环境变量。我们只需使用`test`属性定义模块，并运行`cargo test`命令。然而，我们必须记住，我们的视图和 JSON 序列化代码没有进行单元测试。这就是我们切换到 Postman 来测试我们的 API 端点的原因。

# 在 Postman 中编写测试

在本节中，我们将使用 Postman 实现功能集成测试，以测试我们的 API 端点。这将测试我们的 JSON 处理和数据库访问。为此，我们将遵循以下步骤：

1.  我们将不得不为我们的 Postman 测试创建一个测试用户。我们可以使用以下 JSON 体来完成此操作：

    ```rs
    {
    ```

    ```rs
        "name": "maxwell",
    ```

    ```rs
        "email": "maxwellflitton@gmail.com",
    ```

    ```rs
        "password": "test"
    ```

    ```rs
    }
    ```

1.  我们需要向`http://127.0.0.1:8000/v1/user/create` URL 添加一个`POST`请求。一旦完成，我们就可以使用我们的登录端点进行 Postman 测试。现在我们已经创建了测试用户，我们必须从`http://127.0.0.1:8000/v1/auth/login` URL 的`POST`请求响应头中获取令牌：

    ```rs
    {
    ```

    ```rs
        "username": "maxwell",
    ```

    ```rs
        "password": "test"
    ```

    ```rs
    }
    ```

这给我们以下 Postman 布局：

![图 9.1 – 创建新的使用 Postman 请求](img/Figure_9.1_B18722.jpg)

图 9.1 – 创建新的使用 Postman 请求

使用此令牌，我们拥有创建我们的 Postman 集合所需的所有信息。Postman 是一组 API 请求。在这个集合中，我们可以使用用户令牌作为认证，将所有待办事项 API 调用组合在一起。调用结果如下：

![图 9.2 – 创建新的使用 Postman 响应](img/Figure_9.2_B18722.jpg)

图 9.2 – 创建新的使用 Postman 的响应

1.  我们可以使用以下 Postman 按钮创建我们的集合，即**+** **新建集合**：

![图 9.3 – 创建新的 Postman 集合](img/Figure_9.3_B18722.jpg)

图 9.3 – 创建新的 Postman 集合

1.  点击此按钮后，我们必须确保我们的用户令牌已为集合定义，因为所有待办事项 API 调用都需要令牌。这可以通过使用 API 调用的**授权**配置来完成，如下面的截图所示：

![图 9.4 – 在新的 Postman 集合中定义 AUTH 凭据](img/Figure_9.4_B18722.jpg)

图 9.4 – 在新的 Postman 集合中定义 AUTH 凭据

我们可以看到，我们只是将令牌复制粘贴到以**token**为键的值中，这将插入到请求的头部。现在这个令牌应该被传递到集合中的所有请求中。这个集合现在存储在左侧导航栏的**集合**选项卡下。

1.  我们现在已经配置了我们的集合，并且现在可以通过点击此截图所示的灰色**添加请求**按钮在集合下添加请求：

![图 9.5 – 为我们的 Postman 集合创建新的请求](img/Figure_9.5_B18722.jpg)

图 9.5 – 为我们的 Postman 集合创建新的请求

现在，我们必须考虑我们测试流程的方法，因为这必须是一个自包含的过程。

## 编写测试的有序请求

我们的要求将按以下顺序进行：

1.  **创建**：创建一个待办事项，然后检查返回值以查看它是否已正确存储。

1.  **创建**：创建另一个待办事项，检查返回值以查看前一个是否已存储，并且流程可以处理两个项目。

1.  **创建**：创建另一个与另一个项目具有相同标题的待办事项，检查响应以确保我们的应用程序不会存储具有相同标题的重复待办事项。

1.  **编辑**：编辑一个项目，检查响应以查看编辑后的项目是否已更改状态为*完成*，并且是否已存储在正确的列表中。

1.  **编辑**：编辑第二个项目，以查看*编辑*效果是否是永久的，以及*完成*列表是否支持两个项目。

1.  **编辑**：编辑应用程序中不存在的项目，以查看应用程序是否正确处理这种情况。

1.  **删除**：删除一个待办事项以查看响应是否不再返回被删除的待办事项，这意味着它不再存储在数据库中。

1.  **删除**：删除最后一个待办事项，检查响应以查看是否没有剩余项目，这表明*删除*操作是永久的。

我们需要运行前面的测试，因为它们依赖于前面的操作是正确的。当我们为集合创建请求时，我们必须清楚请求正在做什么，它处于哪个步骤，以及它是什么类型的请求。例如，创建我们的第一个*创建*测试将如下所示：

![图 9.6 – 创建我们的第一个 Postman 创建请求](img/Figure_9.6_B18722.jpg)

图 9.6 – 创建我们的第一个 Postman 创建请求

如我们所见，步骤通过下划线附加了类型。然后我们将测试列表中的测试描述放入**请求描述（可选）**字段。在定义请求时，你可能会意识到 API 密钥不在请求的标题中。

这是因为它位于请求的隐藏自动生成标题中。我们的第一个请求必须是一个`POST`请求，带有`http://127.0.0.1:8000/v1/item/create/washing` URL。

这创建了待办事项*洗涤*。然而，在我们点击**发送**按钮之前，我们必须切换到 Postman 请求中的**测试**选项卡，就在**设置**选项卡的左侧，以便编写以下截图所示的测试：

![图 9.7 – 在 Postman 中访问测试脚本](img/Figure_9.7_B18722.jpg)

图 9.7 – 在 Postman 中访问测试脚本

我们必须用 JavaScript 编写测试。然而，我们可以通过在测试脚本中键入`pm`来访问 Postman 的`test`库。首先，在测试脚本顶部，我们需要处理请求，这是通过以下代码完成的：

```rs
var result = pm.response.json()
```

通过前面的行，我们可以在整个测试脚本中访问响应 JSON。为了全面测试我们的请求，我们需要遵循以下步骤：

1.  首先，我们需要检查响应的基本内容。我们的第一个测试是检查响应是否为`200`。这可以通过以下代码完成：

    ```rs
    pm.test("response is ok", function () {
    ```

    ```rs
        pm.response.to.have.status(200);
    ```

    ```rs
    });
    ```

在这里，我们定义测试描述。然后，定义测试运行的函数。

1.  然后，我们检查响应中的数据长度。在先前的测试之后，我们将定义以下代码来检查待办事项的长度是否为 1：

    ```rs
    pm.test("returns one pending item", function(){
    ```

    ```rs
        if (result["pending_items"].length !== 1){
    ```

    ```rs
            throw new Error(
    ```

    ```rs
            "returns the wrong number of pending items");
    ```

    ```rs
        }
    ```

    ```rs
    })
    ```

在前面的代码中，我们进行了一个简单的长度检查，如果长度不是一，则抛出错误，因为我们只期望`pending_items`列表中有一个待办事项。

1.  然后，我们在以下代码中检查待办事项的标题和状态：

    ```rs
    pm.test("Pending item has the correct title", function(){
    ```

    ```rs
        if (result["pending_items"][0]["title"] !==
    ```

    ```rs
            "washing"){
    ```

    ```rs
            throw new Error(
    ```

    ```rs
            "title of the pending item is not 'washing'");
    ```

    ```rs
        }
    ```

    ```rs
    })
    ```

    ```rs
    pm.test("Pending item has the correct status",
    ```

    ```rs
             function()
    ```

    ```rs
        {
    ```

    ```rs
            if (result["pending_items"][0]["status"] !==
    ```

    ```rs
                "PENDING"){
    ```

    ```rs
                throw new Error(
    ```

    ```rs
                "status of the pending item is not
    ```

    ```rs
                    'pending'");
    ```

    ```rs
        }
    ```

    ```rs
    })
    ```

在前面的代码中，如果状态或标题不符合我们的预期，我们将抛出错误。现在我们已经满足了待办事项的测试，我们可以继续对已完成事项进行测试。

1.  由于我们的已完成事项应该是零，测试的定义如下：

    ```rs
    pm.test("returns zero done items", function(){
    ```

    ```rs
        if (result["done_items"].length !== 0){
    ```

    ```rs
            throw new Error(
    ```

    ```rs
            "returns the wrong number of done items");
    ```

    ```rs
        }
    ```

    ```rs
    })
    ```

在前面的代码中，我们只是确保`done_items`数组长度为零。

1.  现在，我们必须检查已完成和待办事项的数量。这可以通过以下代码完成：

    ```rs
    pm.test("checking pending item count", function(){
    ```

    ```rs
        if (result["pending_item_count"] !== 1){
    ```

    ```rs
            throw new Error(
    ```

    ```rs
            "pending_item_count needs to be one");
    ```

    ```rs
        }
    ```

    ```rs
    })
    ```

    ```rs
    pm.test("checking done item count", function(){
    ```

    ```rs
        if (result["done_item_count"] !== 0){
    ```

    ```rs
            throw new Error(
    ```

    ```rs
            "done_item_count needs to be zero");
    ```

    ```rs
        }
    ```

    ```rs
    })
    ```

现在我们已经构建了测试，我们可以通过点击 Postman 中的**发送**按钮来发出请求，以获取以下测试输出：

![图 9.8 – Postman 测试输出](img/Figure_9.8_B18722.jpg)

图 9.8 – Postman 测试输出

我们可以看到，我们的测试描述和测试状态被突出显示。如果你得到一个错误，状态将是红色，显示**FAIL**。现在我们的第一个创建测试已经完成，我们可以创建第二个创建测试。

## 创建一个 HTTP 请求的测试

然后，我们可以使用此 URL 创建`2_create`测试：`http://127.0.0.1:8000/v1/item/create/cooking`。这是一个尝试使用我们在上一步中探索的测试方法自己构建测试的好机会。如果你尝试构建测试，它们应该看起来像以下代码：

```rs
var result = pm.response.json()
pm.test("response is ok", function () {
    pm.response.to.have.status(200);
});
pm.test("returns two pending item", function(){
    if (result["pending_items"].length !== 2){
        throw new Error(
        "returns the wrong number of pending items");
    }
})
pm.test("Pending item has the correct title", function(){
    if (result["pending_items"][0]["title"] !== "washing"){
        throw new Error(
        "title of the pending item is not 'washing'");
    }
})
pm.test("Pending item has the correct status", function(){
    if (result["pending_items"][0]["status"] !==
        "PENDING"){
        throw new Error(
        "status of the pending item is not 'pending'");
    }
})
pm.test("Pending item has the correct title", function(){
    if (result["pending_items"][1]["title"] !== "cooking"){
        throw new Error(
        "title of the pending item is not 'cooking'");
    }
})
pm.test("Pending item has the correct status", function(){
    if (result["pending_items"][1]["status"] !==
        "PENDING"){
        throw new Error(
        "status of the pending item is not 'pending'");
    }
})
pm.test("returns zero done items", function(){
    if (result["done_items"].length !== 0){
        throw new Error(
        "returns the wrong number of done items");
    }
})
pm.test("checking pending item count", function(){
    if (result["pending_item_count"].length === 1){
        throw new Error(
        "pending_item_count needs to be one");
    }
})
pm.test("checking done item count", function(){
    if (result["done_item_count"].length === 0){
        throw new Error(
        "done_item_count needs to be zero");
    }
})
```

我们可以看到，我们在第二个待办事项上添加了一些额外的测试。前面的测试也直接适用于`3_create`测试，因为重复创建将与我们将使用与`2_create`相同的 URL 相同。

前面的测试在这些测试中需要相当多的重复，稍微改变数组的长度、项目计数和数组内的属性。这是一个练习基本 Postman 测试的好机会。如果你需要将你的测试与我的测试进行交叉引用，你可以在以下 URL 的 JSON 文件中评估它们：[`github.com/PacktPublishing/Rust-Web-Programming-2nd-Edition/blob/main/chapter09/building_test_pipeline/web_app/scripts/to_do_items.postman_collection.json`](https://github.com/PacktPublishing/Rust-Web-Programming-2nd-Edition/blob/main/chapter09/building_test_pipeline/web_app/scripts/to_do_items.postman_collection.json)。

在本节中，我们为 Postman 测试当 API 调用时执行了一系列步骤。这不仅对我们应用程序有用。Postman 可以访问互联网上的任何 API。因此，您可以使用 Postman 测试来监控实时服务器和第三方 API。

现在，如果必须手动每次运行所有这些测试，可能会很费力。我们可以使用**新曼**自动化运行和检查这个集合中的所有测试。如果我们自动化这些集合，我们可以在每天特定时间运行测试，以实时服务器和我们所依赖的第三方 API，并在我们的服务器或第三方 API 出现问题时提醒我们。

新曼将为我们在这个领域进一步开发提供一个良好的基础。在下一节中，我们将导出集合，并使用新曼按顺序运行导出集合中的所有 API 测试。

# 使用新曼自动化 Postman 测试

为了自动化一系列测试，在本节中，我们将按照正确的顺序导出我们的待办事项 Postman 集合。但首先，我们必须将集合导出为 JSON 文件。这可以通过在 Postman 的左侧导航栏中点击我们的集合，然后点击灰色显示的**导出**按钮来完成，如以下截图所示：

![图 9.9 – 导出我们的 Postman 集合](img/Figure_9.9_B18722.jpg)

图 9.9 – 导出我们的 Postman 集合

现在我们已经导出了集合，我们可以快速检查它以查看文件是如何结构的。以下代码定义了测试套件的头部：

```rs
"info": {
    "_postman_id": "bab28260-c096-49b9-81e6-b56fc5f60e9d",
    "name": "to_do_items",
    "schema": "https://schema.getpostman.com
    /json/collection/v2.1.0/collection.json",
    "_exporter_id": "3356974"
},
```

上述代码告诉 Postman 需要什么模式来运行测试。如果代码被导入到 Postman 中，ID 和名称将可见。然后文件继续通过以下代码定义单个测试：

```rs
"item": [
    {
        "name": "1_create",
        "event": [
            {
                "listen": "test",
                "script": {
                    "exec": [
                        "var result = pm.response.json()",
                        . . .
                    ],
                    "type": "text/javascript"
                }
            }
        ],
        "request": {
            "method": "POST",
            "header": [
                {
                    "key": "token",
                    "value": "eyJhbGciOiJIUzI1NiJ9
                    .eyJ1c2VyX2lkIjo2fQ.
                    uVo7u877IT2GEMpB_gxVtxhMAYAJD8
                    W_XiUoNvR7_iM",
                    "type": "text",
                    "disabled": true
                }
            ],
            "url": {
                "raw": "http://127.0.0.1:8000/
                v1/item/create/washing",
                "protocol": "http",
                "host": ["127", "0", "0", "1"],
                "port": "8000",
                "path": ["v1", "item", "create", "washing"]
            },
            "description": "create a to-do item,
            and then check the
            return to see if it is stored correctly "
        },
        "response": []
    },
```

从上述代码中，我们可以看到我们的测试、方法、URL、头部等都在一个数组中定义。快速检查`item`数组将显示测试将按照我们想要的顺序执行。

现在，我们可以简单地使用新曼运行它。我们可以使用以下命令安装新曼：

```rs
npm install -g newman
```

注意

必须注意，上述命令是一个全局安装，有时可能会出现问题。为了避免这种情况，你可以设置一个包含以下内容的`package.json`文件：

```rs
{
  "name": "newman testing",
  "description": "",
  "version": "0.1.0",
  "scripts": {
    "test": "newman run to_do_items.
             postman_collection.json"
  },
  "dependencies": {
    "newman": "5.3.2"
  }
}
```

使用这个`package.json`，我们已经定义了测试命令和新曼依赖。我们可以使用以下命令在本地上安装我们的依赖项：

```rs
npm install
```

这将在`node_modules`目录下安装我们所需的所有内容。我们不必直接运行新曼测试命令，可以使用`package.json`中定义的测试命令，使用以下命令：

```rs
npm run test
```

现在我们已经安装了新曼，我们可以使用以下命令运行测试集合对导出的集合 JSON 文件进行测试：

```rs
newman run to_do_items.postman_collection.json
```

上述命令运行所有测试并给出状态报告。每个描述都打印出来，测试的状态也在测试旁边表示。以下是一个典型的 API 测试评估的打印输出：

```rs
→ 1_create
    POST http://127.0.0.1:8000/v1/item/create/washing
    [200 OK, 226B, 115ms]
    ✓ response is ok
    ✓ returns one pending item
    ✓ Pending item has the correct title
    ✓ Pending item has the correct status
    ✓ returns zero done items
    ✓ checking pending item count
    ✓ checking done item count
```

上述输出给出了名称、方法、URL 和响应。在这里，所有这些都通过了。如果其中任何一个没有通过，那么测试描述将显示一个 *叉* 而不是 *勾*。我们还得到了以下摘要：

![图 9.10 – Newman 摘要](img/Figure_9.10_B18722.jpg)

图 9.10 – Newman 摘要

我们可以看到所有测试都通过了。通过这种方式，我们已经成功自动化了我们的功能测试，使我们能够以最小的努力测试完整的流程。然而，我们所做的是不可维护的。例如，我们的令牌将过期，这意味着如果我们在本月稍后运行测试，它们将失败。在下一节中，我们将构建一个完整的自动化流程，该流程将构建我们的服务器，更新我们的令牌，并运行我们的测试。

# 构建整个自动化测试流程

在开发和测试方面，我们需要一个可以轻松拆解和重建的环境。没有什么比在你的本地机器上构建数据库中的数据来进一步开发使用这些数据的功能更糟糕的了。然而，数据库容器可能会意外删除，或者你可能编写了一些损坏数据的代码。然后，你必须花费大量时间重新创建数据，才能回到之前的状态。如果系统复杂且缺少文档，你可能会忘记重新创建数据所需的步骤。如果你在开发和测试时对销毁本地数据库并重新开始感到不舒服，那么就有问题，而且你被抓住只是时间问题。在本节中，我们将创建一个单个 Bash 脚本，该脚本执行以下操作：

1.  在后台启动数据库 Docker 容器。

1.  编译 Rust 服务器。

1.  运行单元测试。

1.  启动 Rust 服务器运行。

1.  将迁移运行在 Docker 中的数据库。

1.  发送 HTTP 请求创建用户。

1.  发送 HTTP 请求进行登录并获取令牌。

1.  使用登录令牌更新 Newman JSON 文件。

1.  运行 Newman 测试。

1.  删除整个过程中产生的文件。

1.  停止 Rust 服务器运行。

1.  停止并销毁整个过程中运行的 Docker 容器。

上述列表中列出了许多步骤。浏览这个列表，直观地似乎应该将我们要探索的代码块分解成步骤；然而，我们将几乎在单个 Bash 脚本中运行所有这些步骤。许多前面概述的步骤可以用一行 Bash 代码实现。将代码分解成步骤将是多余的。现在我们已经拥有了所有需要的步骤，我们可以设置我们的测试基础设施。首先，我们需要在 `web_app` 根目录中 `src` 目录旁边设置一个 `scripts` 目录。然后在 `scripts` 目录中，我们需要一个 `run_test_pipeline.sh` 脚本，该脚本将运行主要的测试过程。我们还需要将我们的 Newman JSON `config` 文件放在 `scripts` 目录中。

我们将使用 `bash` 来编排整个测试流程，这是编排测试任务的最佳工具。在我们的 `scripts/run_test_pipeline.sh` 脚本中，我们将从以下代码开始：

```rs
#!/bin/bash
# move to directory of the project
SCRIPTPATH="$( cd "$(dirname "$0")" ; pwd -P )"
cd $SCRIPTPATH
cd ..
```

在前面的代码中，我们告诉计算机代码块是一个 Bash 脚本，使用 `#!/bin/bash` shebang 行。Bash 脚本从调用它的 Bash 脚本的当前工作目录运行。我们可以从多个目录调用脚本，因此我们需要确保我们获取到脚本所在的目录，即 `scripts` 目录，将其分配给名为 `SCRIPTPATH` 的变量，移动到该目录，然后使用 `cd..` 命令向上移动一个目录，以便位于包含 Docker、配置和 Cargo 文件的父目录中。然后我们可以使用 `-d` 标志在后台启动 Docker 容器，并循环直到数据库接受连接，以下为相关代码：

```rs
# spin up docker and hold script until accepting connections
docker-compose up -d
until pg_isready -h localhost -p 5433 -U username
do
  echo "Waiting for postgres"
  sleep 2;
done
```

现在我们已经启动了 Docker 容器，我们可以继续构建我们的 Rust 服务器。首先，我们可以编译 Rust 服务器并使用以下代码运行我们的单元测试：

```rs
cargo build
cargo test
```

单元测试运行完毕后，我们可以使用以下代码在后台运行我们的服务器：

```rs
# run server in background
cargo run config.yml &
SERVER_PID=$!
sleep 5
```

命令末尾的 `&` 使得 `cargo run config.yml` 在后台运行。然后我们获取 `cargo run config.yml` 命令的进程 ID，并将其分配给变量 `SERVER_PID`。然后我们等待 5 秒以确保服务器已准备好接受连接。在我们向服务器发送任何 API 调用之前，我们必须使用以下代码运行数据库迁移：

```rs
diesel migration run
```

然后我们回到 `scripts` 目录，向我们的服务器发送一个创建用户的 API 调用：

```rs
# create the user
curl --location --request POST 'http://localhost:8000/v1/user/create' \
--header 'Content-Type: application/json' \
--data-raw '{
    "name": "maxwell",
    "email": "maxwellflitton@gmail.com",
    "password": "test"
}'
```

如果你想知道如何在 Bash 中使用 `curl` 发送 HTTP 请求，你可以使用 Postman 工具自动生成它们。在 Postman 工具的右侧，你可以看到一个 **代码** 按钮，如下面的截图所示：

![图 9.11 – 代码生成工具](img/Figure_9.11_B18722.jpg)

图 9.11 – 代码生成工具

一旦你点击了代码标签，就会出现一个下拉菜单，你可以从中选择多种语言。一旦你选择了你想要的编程语言，你的 API 调用就会以代码片段的形式显示在你选择的编程语言中，然后你可以复制并粘贴。

现在我们已经创建了用户，我们可以登录并使用以下代码将令牌存储在 `fresh_token.json` 文件中；然而，需要注意的是，`curl` 首先需要被安装：

```rs
# login getting a fresh token
echo $(curl --location --request GET 'http://localhost:8000/v1/auth/login' \
--header 'Content-Type: application/json' \
--data-raw '{
    "username": "maxwell",
    "password": "test"
}') > ./fresh_token.json
```

这里发生的情况是我们可以将 API 调用的结果包装在一个变量 `$(...)` 中。然后我们通过 `echo $(...) > ./fresh_token.json` 将其输出并写入文件。然后我们可以将新的令牌插入到 Newman 数据中，并使用以下代码运行 Newman API 测试：

```rs
TOKEN=$(jq '.token' fresh_token.json)
jq '.auth.apikey[0].value = '"$TOKEN"''
to_do_items.postman_collection.json > test_newman.json
newman run test_newman.json
```

我们的测试现在完成了。我们可以清理测试运行时创建的文件，销毁 Docker 容器，并使用以下代码停止运行的服务器：

```rs
rm ./test_newman.json
rm ./fresh_token.json
# shut down rust server
kill $SERVER_PID
cd ..
docker-compose down
```

注意

在我们运行 Bash 脚本之前，`curl`和`jq`都需要安装。如果您使用 Linux，可能需要运行以下命令：

```rs
sudo chmod +x ./run_test_pipeline.sh
```

我们可以使用以下命令运行我们的测试脚本：

```rs
sh run_test_pipeline.sh
```

展示整个打印输出只会无谓地填满书籍。然而，我们可以在以下屏幕截图中看到测试打印输出的结尾：

![图 9.12 – 测试管道输出](img/Figure_9.12_B18722.jpg)

图 9.12 – 测试管道输出

在这里，打印输出清楚地表明纽曼测试已经运行并通过。测试完成后，服务器被关闭，支持服务器的 Docker 容器也被停止并移除。如果您想将此日志写入`txt`文件，可以使用以下命令：

```rs
sh run_test_pipeline.sh > full_log.txt
```

这就是了！一个完全工作的测试管道，它自动化了我们的服务器设置、测试和清理。因为我们将其编写为简单的 Bash 测试管道，所以我们可以将这些步骤集成到自动化管道中，如 Travis、Jenkins 或 GitHub Actions。这些管道工具在执行`pull`请求和合并时自动触发。

# 摘要

在本章中，我们了解了我们应用程序的工作流程和组件，将它们分解以便我们可以为正确部分选择正确的工具。我们使用了单元测试，以便可以快速检查几个边缘情况，以了解每个函数和结构体如何与其他部分交互。

我们还直接使用单元测试检查了我们的自定义结构体。然后，我们使用`actix_web`测试结构体来模拟请求，以查看使用结构体并处理请求的函数如何工作。然而，当我们来到主 API 视图模块时，我们转向了 Postman。

这是因为我们的 API 端点是简单的。它们创建、编辑和删除待办事项。我们可以通过直接进行 API 调用并检查响应来直接评估此过程。我们能够直接评估 JSON 处理以接受和返回数据。我们还能够使用这些 Postman 测试评估数据库中的数据查询、写入和更新。

Postman 使我们能够快速高效地测试一系列流程。我们甚至通过 Newman 自动化这个过程来加快测试速度。然而，必须注意的是，这种方法并不是万能的。如果 API 视图函数变得更加复杂，有更多的移动部件，例如与另一个 API 或服务通信，那么 Newman 方法将需要重新设计。需要考虑触发模拟此类过程的环境变量，以便我们可以快速测试一系列边缘情况。

如果系统随着我们结构体的依赖关系增长而增长，将需要模拟对象。这就是我们创建一个假的结构体或函数并定义测试输出的地方。为此，我们需要一个外部 crate，例如`mockall`。关于这个 crate 的文档在本章的*进一步阅读*部分有所介绍。

我们的应用程序现在完全运行，并有一系列测试。现在，我们剩下的就是将我们的应用程序部署到服务器上。

在下一章中，我们将使用 *Docker* 在 **Amazon** **Web** **Services** (**AWS**) 上设置一个服务器，将我们的应用程序部署到服务器上。我们将介绍设置 AWS 配置、运行测试，并在测试通过的情况下将我们的应用程序部署到服务器上的过程。

# 问题

1.  如果我们可以手动操作应用程序，为什么还要费心进行单元测试？

1.  单元测试和功能测试有什么区别？

1.  单元测试有哪些优点？

1.  单元测试有哪些缺点？

1.  功能测试有哪些优点？

1.  功能测试有哪些缺点？

1.  构建单元测试的合理方法是什么？

# 答案

1.  当涉及到手动测试时，你可能会忘记运行某个特定程序。运行测试标准化了我们的标准，并使我们能够将它们集成到持续集成工具中，以确保新代码不会破坏服务器，因为如果代码失败，持续集成可以阻止新代码的合并。

1.  单元测试将单个组件（如函数和结构体）隔离开来。然后，通过一系列模拟输入对这些函数和结构体进行评估，以了解组件如何与不同的输入交互。功能测试评估系统，调用 API 端点，并检查响应。

1.  单元测试轻量级，不需要整个系统运行。它们可以快速测试一系列边缘情况。单元测试还可以精确地隔离错误发生的位置。

1.  单元测试基本上是使用虚构输入的隔离测试。如果系统中的输入类型更改但单元测试未更新，那么这个测试在应该失败时实际上会通过。单元测试也不评估系统是如何运行的。

1.  功能测试确保整个基础设施按预期协同工作。例如，我们配置和连接数据库的方式可能存在问题。使用单元测试，这些问题可能会被忽略。此外，尽管模拟确保了隔离的测试，但单元测试的模拟可能过时。这意味着模拟的函数可能会返回更新版本中没有的数据。因此，单元测试会通过，但功能测试不会，因为它们测试一切。

1.  功能测试需要具有像数据库这样的基础设施来运行。还必须有设置和拆卸函数。例如，功能测试将影响数据库中存储的数据。在测试结束时，需要在再次运行测试之前清除数据库。这可能会增加复杂性，并可能需要在不同操作之间使用“粘合”代码。

1.  我们首先测试那些没有依赖的 struct 和函数。一旦这些测试完成，我们就知道我们对它们很熟悉。然后我们转向那些具有我们之前测试过的依赖的函数和 struct。采用这种方法，我们知道我们正在编写的当前测试不会因为依赖而失败。

# 进一步阅读

+   Mockall 文档：[`docs.rs/mockall/0.9.0/mockall/`](https://docs.rs/mockall/0.9.0/mockall/)

+   Github Actions 文档：[`github.com/features/actions`](https://github.com/features/actions)

+   Travis 文档：[`docs.travis-ci.com/user/for-beginners/`](https://docs.travis-ci.com/user/for-beginners/)

)

+   Circle CI 文档：[`circleci.com/docs/`](https://circleci.com/docs/)

+   Jenkins 文档：[`www.jenkins.io/doc/`](https://www.jenkins.io/doc/)
