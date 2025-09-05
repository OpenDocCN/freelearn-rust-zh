# 第四章：使用 Serde Crate 进行数据序列化和反序列化

微服务可以与客户端或彼此交互。要实现交互，你必须选择一个协议和一个格式，以便从一个通信参与者向另一个通信参与者发送消息。有许多格式和 RPC 框架可以简化交互过程。在本章中，我们将发现 `serde` Crate 的功能，它可以帮助你使结构体可序列化和可反序列化，并兼容不同的格式，如 JSON、CBOR、MessagePack 和 BSON。

本章将涵盖以下主题：

+   如何序列化和反序列化数据

+   如何使自定义类型可序列化

+   应选择哪些序列化格式以及应避免哪些格式

# 技术要求

在本章中，我们将探讨 `serde` Crate 家族中的一些可用功能。这个家族包括不可分割的一对——`serde` 和 `serde_derive` Crate。它还包括如 `serde_json`、`serde_yaml` 和 `toml` 这样的 Crate，它们为你提供了对特殊格式（如 JSON、YAML 和 **`TOML`**）的支持。所有这些 Crate 都是纯 Rust 编写的，并且不需要任何外部依赖。

您可以从 GitHub 获取本章示例的源代码：[`github.com/PacktPublishing/Hands-On-Microservices-with-Rust-2018/tree/master/Chapter04`](https://github.com/PacktPublishing/Hands-On-Microservices-with-Rust-2018/tree/master/Chapter04)[.](https://github.com/PacktPublishing/Hands-On-Microservices-with-Rust-2018/tree/master/Chapter4)

# 与微服务交互的数据格式

微服务可以与不同的参与者交互，例如客户端、其他微服务和第三方 API。通常，交互是通过网络使用特定格式的序列化数据消息来执行的。在本节中，我们将学习如何选择这些交互的格式。我们还将探索 `serde` Crate 的基本功能——如何使我们的结构体可序列化并使用特定格式。

# Serde Crate

当我开始在我的项目中使用 Rust 时，我通常使用流行的 `rustc_serialize` Crate。这并不坏，但我发现它不够灵活。例如，我无法在我的结构体中使用泛型数据类型。`serde` Crate 的创建是为了消除 `rustc_serialize` Crate 的不足。自从 `serde` Crate 达到 1.0 版本分支以来，`serde` 一直是 Rust 中序列化和反序列化的主要 Crate。

我们在上一章中使用这个 Crate 来反序列化配置文件。现在我们将使用它来将请求数据和响应数据转换为文本或二进制数据。

为了探索序列化，我们将使用一个生成随机数的微服务。我们将重写一个非常简单的版本，不包含日志记录、读取参数、环境变量或配置文件。它将使用 HTTP 主体来指定随机值的范围和随机分布，以生成一个数字。

我们的服务将只处理 `/random` 路径的请求。它将期望请求和响应都使用 JSON 格式。如前所述，`serde` crate 为代码提供了序列化能力。我们还需要 `serde_derive` crate 来自动推导序列化方法。`serde` crate 只包含核心类型和特性，以使序列化过程通用和可重用，但特定格式是在其他 crate 中实现的。我们将使用 `serde_json`，它提供了一个 JSON 格式的 `serializer`。

复制最小随机数生成服务的代码，并将以下依赖项添加到 `Cargo.toml` 中：

```rs
[dependencies]
futures = "0.1"
hyper = "0.12"
rand = "0.5"
serde = "1.0"
serde_derive = "1.0"
serde_json = "1.0"
```

将以下 crate 导入到 `main.rs` 源文件中：

```rs
extern crate futures;
extern crate hyper;
extern crate rand;
#[macro_use]
extern crate serde_derive;
extern crate serde_json;
```

如你所见，我们从 `serde_derive` 和 `serde_json` crate 中导入了一个宏来使用 JSON `serializer`。我们不需要导入 `serde` crate，因为我们不会直接使用它，但使用宏是必要的。现在我们可以查看代码的不同部分。首先，我们将检查请求和响应类型。之后，我们将实现一个处理器来使用它。

# 序列化响应

服务返回以 `f64` 类型表示的随机数。我们希望将其打包到一个 JSON 对象中，因为我们可能需要向对象中添加更多字段。在 JavaScript 中使用对象很简单。声明 `RngResponse` 结构体并添加 `#[derive(Serialize)]` 属性。这使得这个结构体可序列化，如下所示：

```rs
#[derive(Serialize)]
struct RngResponse {
    value: f64,
}
```

为了使这个结构体可反序列化，我们应该推导 `Deserialize` 特性。如果你想要使用相同的类型进行请求和响应，反序列化可能很有用。如果你想在服务器和客户端中使用相同的类型，那么推导 `Serialize` 和 `Deserialize` 都是很重要的。

序列化的对象将被表示为以下字符串：

```rs
{ "value": 0.123456 }
```

# 反序列化请求

服务将支持复杂的请求，你可以指定分布和参数。让我们将这个枚举添加到源文件中：

```rs
#[derive(Deserialize)]
enum RngRequest {
    Uniform {
        range: Range<i32>,
    },
    Normal {
        mean: f64,
        std_dev: f64,
    },
    Bernoulli {
        p: f64,
    },
}
```

你可能想知道序列化值看起来像什么。`serde_derive` 提供了额外的属性，你可以使用这些属性来调整序列化格式。当前的 `deserializer` 期望一个 `RngRequest` 实例，如下所示：

```rs
RngRequest::Uniform {
        range: 1..10,
}
```

这将被表示为以下字符串：

```rs
{ "Uniform": { "range": { "start": 1, "end": 10 } } }
```

如果你从头开始创建自己的协议，由于你可以轻松地使其符合由 `serde` crate 自动生成的序列化器的任何限制，因此布局不会有任何问题。然而，如果你必须使用现有的协议，你可以尝试向声明中添加额外的属性。如果这不起作用，你可以手动实现 `Serialize` 或 `Deserialize` 特性。例如，假设我们想使用以下请求格式：

```rs
{ "distribution": "uniform", "parameters": { "start": 1, "end": 10 } } }
```

将 `serde` 属性添加到 `RngRequest` 声明中，这将使 `deserializer` 支持前面的格式。代码将如下所示：

```rs
#[derive(Deserialize)]
#[serde(tag = "distribution", content = "parameters", rename_all = "lowercase")]
enum RngRequest {
    Uniform {
        #[serde(flatten)]
        range: Range<i32>,
    },
    Normal {
        mean: f64,
        std_dev: f64,
    },
    Bernoulli {
        p: f64,
    },
}
```

现在，枚举使用上述请求格式。`serde_derive` 包中有很多属性，探索它们非常重要。

# 调整序列化

`serde_derive` 支持许多属性，可以帮助你避免为结构体手动实现 `serializer` 或 `deserializer`。在本节中，我们将详细探讨有用的属性。我们将学习如何更改变体的字母大小写，如何移除嵌套级别，以及如何为枚举的标签和内容使用特定名称。

# 更改名称的大小写

`serde_derive` 将代码中字段的名称映射到数据字段。例如，结构体中的 `title` 字段将期望数据对象中的 `title` 字段。如果协议使用其他名称作为字段，你必须考虑到这一点以避免警告。例如，协议中的一个结构体可能包含 `stdDev` 字段，但如果你在 Rust 中使用这个字段的名称，你会得到以下警告：

```rs
warning: variable `stdDev` should have a snake case name such as `std_dev`
```

你可以通过添加 `#![allow(non_snake_case)]` 属性来修复这个问题，但这会使代码看起来不美观。更好的解决方案是使用 `#[serde(rename="stdDev")]` 属性，并仅对序列化和反序列化使用其他命名约定。

重命名属性有两种变体：

+   更改所有变体的命名约定

+   更改字段的名称

要更改枚举的所有变体，请添加 `#[serde(rename_all="...")]` 属性并使用以下值之一：`"lowercase"`、`"PascalCase"`、`"camelCase"`、`"snake_case"`、`"SCREAMING_SNAKE_CASE"` 或 `"kebab-case"`。为了更具有代表性，命名值将根据其自身约定的规则编写。

要更改字段的名称，请使用 `#[serde(rename="...")]` 属性并指定在序列化过程中使用的字段名称。你可以在 第十七章 中看到一个此属性使用示例，*使用 AWS Lambda 的有界微服务*。

使用重命名的一个原因是当字段名称在 Rust 中是关键字时。例如，一个结构体不能包含名为 `type` 的字段，因为它是一个关键字。你可以在结构体中将它重命名为 `typ` 并添加 `#[serde(rename="type")]` 到它。

# 移除嵌套

自动派生的 `serializer` 使用与你的类型相同的嵌套结构。如果你需要减少嵌套级别，你可以设置 `#[serde(flatten)]` 属性以使用没有封装对象的字段。在之前的示例中，我们使用了标准库中的 `Range` 类型来设置生成随机值的范围，但我们还希望在序列化数据中看到实现细节。为此，我们需要 `Range` 的 `start` 和 `end` 字段。我们向该字段添加了此属性以删除结构中的 `{ "range": ... }` 级别。

对于枚举，`serde_derive` 使用标签作为对象的名称。例如，以下 JSON-RPC 包含两个变体：

```rs
#[derive(Serialize, Deserialize)]
enum RpcRequest {
    Request { id: u32, method: String, params: Vec<Value> },
    Notification { id: u32, method: String, params: Vec<Value> },
}
```

`params` 字段包含一个由 `serde_json::Value` 类型表示的任何 JSON 值的数组，我们将在本章后面探讨此类型。如果你序列化此结构体的实例，它将包括变体的名称。例如，考虑以下内容：

```rs
{ "Request": { "id": 1, "method": "get_user", "params": [123] } }
```

此请求与 JSON-RPC 规范不兼容 ([`www.jsonrpc.org/specification#request_object`](https://www.jsonrpc.org/specification#request_object))。我们可以使用 `#[serde(untagged)]` 属性删除封装对象，结构体将如下所示：

```rs
{ "id": 1, "method": "get_user", "params": [123] }
```

此更改后，此序列化数据可以发送为 JSON-RPC。但是，如果你仍然希望将变体值保留在序列化数据形式中，你必须使用另一种方法，这将在下一节中描述。

# 使用特定的名称为标签和内容命名

在我们的示例中，我们希望在序列化数据形式中有两个字段：`distribution` 和 `parameters`。在第一个字段中，我们希望保存枚举的变体，但重命名使其为小写。在第二个字段中，我们将保留特定变体的参数。

为了实现这一点，你可以编写自己的 `Serializer` 和 `Deserializer`，我们将在本章后面探讨这种方法。然而，在这种情况下，我们可以使用 `#[serde(tag = "...")]` 和 `#[serde(context = "...")]` 属性。`context` 只能与 `tag` 配对使用。

我们已经将此添加到我们的 `RngRequest` 中：

```rs
#[serde(tag = "distribution", content = "parameters", rename_all = "lowercase")]
```

此属性指定了序列化对象的 `distribution` 键以保存枚举的变体。枚举的变体移动到序列化对象的 `parameters` 字段。最后一个属性 `rename_all` 改变了变体名称的大小写。如果没有重命名，我们将被迫使用标题大小写来表示分布，例如使用 `"Uniform"` 而不是更整洁的 `"uniform"`。

有时，序列化的值必须包含具有动态结构的对象。例如，JSON 格式支持自由数据结构。我不喜欢未指定的结构，但为了创建与现有服务兼容的服务，我们可能需要它们。

# 任何值

如果你希望保持数据的一部分未序列化，但你不知道数据的结构，并且希望在运行时稍后探索它，你可以使用通用的 `serde_json::Value`，它表示任何类型的值。例如，`serde_json` 包含一个 `Value` 对象和一个从 `Value` 实例反序列化类型的方法。这可能在对表示进行重新排序之前完全反序列化困难的反序列化情况中很有用。

要使用通用的 `Value`，请将其添加到你的结构体中。例如，考虑以下内容：

```rs
#[derive(Deserialize)]
struct Response {
    id: u32,
    result: serde_json::Value,
}
```

在这里，我们使用了 `serde_json` 包的通用值。当你需要将其反序列化为 `User` 结构体，例如，你可以使用 `serde_json::from_value` 函数：

```rs
let u: User = serde_json::from_value(&response)?;
```

在本节中，我们学习了反序列化过程。现在是时候向我们的服务器添加一个处理器来处理请求了。这个处理器将反序列化数据，生成一个随机值，并以序列化的形式将数据返回给客户端。

如果我写一个代理服务，我应该反序列化和序列化请求以将它们不变地发送到另一个服务吗？这取决于服务的目的。序列化和反序列化会占用大量的 CPU 资源。如果服务用于平衡请求，你不需要请求的内部数据。特别是如果你只使用 HTTP 头选择请求的目的地。然而，你可能想使用反序列化数据的处理优势——例如，你可以在将其发送到其他微服务之前修补数据的一些值。

# 使用超

我们添加了`RngRequest`和`Response`类型，并实现了`Serialize`和`Deserialize`特性。现在我们可以在处理器中使用它们与`serde_json`一起。在本节中，我们将探讨如何获取请求的全部内容，将其反序列化为特定类型的实例，以及序列化响应。

# 从流中读取主体

事实上，`hyper`包中请求的主体是一个流。你无法立即获取全部内容，但你可以读取传入的块，将它们写入一个向量，并使用结果数据集作为一个单一对象。

我们无法访问整个主体，因为它可能是一个巨大的数据块，我们无法将其保存在内存中。我们的服务可以用于上传多个太字节的数据文件或进行视频流传输。

由于我们使用的是异步方法，我们不能在读取流到末尾时阻塞当前线程。这是因为这将阻塞线程并导致程序停止工作，因为相同的线程用于轮询流。

`serde`包不支持连续数据流的反序列化，但你可以直接创建并使用一个`Deserializer`实例来处理无限流。

要从流中读取数据，你必须获取一个`Stream`实例并将其放入一个`Future`对象中，该对象将收集该流中的数据。我们将在下一章中更详细地探讨这个主题。让我们实现一个从`Stream`实例收集数据的`Future`。将以下代码添加到处理器中`/random`路径的`match`表达式的分支：

```rs
(&Method::POST, "/random") => {
    let body = req.into_body().concat2()
        .map(|chunks| {
            let res = serde_json::from_slice::<RngRequest>(chunks.as_ref())
                .map(handle_request)
                .and_then(|resp| serde_json::to_string(&resp));
            match res {
                Ok(body) => {
                    Response::new(body.into())
                },
                Err(err) => {
                    Response::builder()
                        .status(StatusCode::UNPROCESSABLE_ENTITY)
                        .body(err.to_string().into())
                        .unwrap()
                },
            }
        });
    Box::new(body)
}
```

`Request`实例有一个`into_body`方法，它返回请求的主体。我们使用了`Body`类型来表示我们的处理器的主体。`Body`类型是一个实现了`Stream`特性的块流。它有一个`concat2`方法，可以将所有块连接成一个单一对象。这是因为`Chunk`类型实现了`Extend`特性，并且可以与其他`Chunk`扩展。`concat2`方法将`Stream`转换为`Future`。

如果你还不熟悉`futures` crate，你可以在下一章中了解更多关于它的信息。现在，你可以将`Future`对象视为一个稍后完成的`Result`。你可以将`Stream`视为一个没有引用任何项的`Iterator`，它必须从数据流中轮询下一个项。

在我们取完请求的全部正文后，我们可以使用反序列化。`serde_derive`派生一个通用的`deserializer`，因此我们必须使用一个 crate 来获取特定的序列化格式。在这种情况下，我们将使用 JSON 格式，所以我们将使用`serde_json` crate。这包括一个`from_slice`函数，它为我们的类型创建一个`Deserializer`，并使用它从缓冲区中读取一个实例。

`from_slice`方法返回一个`Result<T, serde_json::Error>`，我们将`map`这个结果到我们自己的`handle_request`函数，该函数读取请求并生成响应。我们将在本节稍后讨论这个函数。

当结果准备好时，我们使用`serde_json::to_string`函数将响应转换为 JSON 字符串。我们使用`and_then`，因为`to_string`返回一个 Result，我们必须处理任何错误。

我们现在有一个`Result`，它包含一个序列化的响应或者如果发生错误则包含`serde_json::Error`。我们将使用`match`表达式来返回一个成功的`Response`，如果响应被成功创建并序列化为一个`String`，或者返回一个带有`UNPROCESSABLE_ENTITY`状态和错误信息的响应体。

在我们的案例中，我们创建了一个没有结果的`Future`对象。我们必须将这个未来添加到一个 reactor 中以便执行它。

在讨论前面的代码时，我们提到了`handle_request`函数。让我们更仔细地看看这个函数的实现：

```rs
fn handle_request(request: RngRequest) -> RngResponse {
    let mut rng = rand::thread_rng();
    let value = {
        match request {
            RngRequest::Uniform { range } => {
                rng.sample(Uniform::from(range)) as f64
            },
            RngRequest::Normal { mean, std_dev } => {
                rng.sample(Normal::new(mean, std_dev)) as f64
            },
            RngRequest::Bernoulli { p } => {
                rng.sample(Bernoulli::new(p)) as i8 as f64
            },
        }
    };
    RngResponse { value }
}
```

函数接受一个`RngRequest`值。实现的第一行使用`rand::thread_rng`函数创建一个随机数生成器实例。我们将使用`sample`方法生成一个随机值。

我们的请求支持三种类型的分布：`Uniform`、`Normal`和`Bernoulli`。我们使用了解构模式来获取请求的参数以创建一个分布实例。之后，我们使用反序列化的请求进行采样，并将结果转换为`f64`类型以打包到`RngResponse`值中。

# 自定义类型

当你的微服务使用自定义数据结构时，它需要一个自定义的序列化格式。你可以通过实现`Serialize`和`Deserialize`特质，或者通过添加特殊属性到你的结构体或结构体的字段中来自定义序列化。我们将在这里探讨这两种方法。

# 自定义序列化

我们将扩展我们的随机数生成服务，增加两个功能——生成随机颜色和打乱字节数组。对于第一个功能，我们需要添加一个`Color`结构体来保存颜色分量。创建一个新的文件，`color.rs`，并将以下代码添加到其中：

```rs
#[derive(Clone, PartialEq, Eq)]
pub struct Color {
    pub red: u8,
    pub green: u8,
    pub blue: u8,
}
```

添加两个我们稍后会用到的常量颜色：

```rs
pub const WHITE: Color = Color { red: 0xFF, green: 0xFF, blue: 0xFF };
pub const BLACK: Color = Color { red: 0x00, green: 0x00, blue: 0x00 };
```

结构体还实现了 `PartialEq` 和 `Eq` 来比较值与这些常量。

我们将使用与 CSS 兼容的文本表示颜色。我们将支持十六进制格式的 RGB 颜色以及两种文本颜色：`black` 和 `white`。要将颜色转换为字符串，为 `Color` 实现 `Display` 特性：

```rs
impl fmt::Display for Color {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            &WHITE => f.write_str("white"),
            &BLACK => f.write_str("black"),
            color => {
                write!(f, "#{:02X}{:02X}{:02X}", color.red, color.green, color.blue)
            },
        }
    }
}
```

此实现使用 `'#'` 前缀将三个颜色分量写入字符串。每个颜色分量字节都以十六进制格式写入，对于非重要数字使用 `'0'` 前缀，宽度为两个字符。

现在我们可以使用这个格式化程序来实现 `Color` 结构体的 `Serialize` 特性：

```rs
impl Serialize for Color {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        serializer.serialize_str(&self.to_string())
    }
}
```

此 `Serialize` 实现调用 `Serializer` 的 `serialize_str` 方法来将颜色的十六进制表示存储到字符串中。在实现自定义反序列化之前，将所有必要的导入添加到 `color.rs` 文件中：

```rs
use std::fmt;
use std::str::FromStr;
use std::num::ParseIntError;
use serde::{de::{self, Visitor}, Deserialize, Deserializer, Serialize, Serializer};
```

# 自定义反序列化

我们的 `Color` 类型必须能够从字符串转换。我们可以通过实现 `FromStr` 特性来实现这一点，这使得我们可以调用 `str` 的 `parse` 方法来从字符串解析结构体：

```rs
impl FromStr for Color {
    type Err = ColorError;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        match s {
            "white" => Ok(WHITE.to_owned()),
            "black" => Ok(BLACK.to_owned()),
            s if s.starts_with("#") && s.len() == 7 => {
                let red = u8::from_str_radix(&s[1..3], 16)?;
                let green = u8::from_str_radix(&s[3..5], 16)?;
                let blue = u8::from_str_radix(&s[5..7], 16)?;
                Ok(Color { red, green, blue })
            },
            other => {
                Err(ColorError::InvalidValue { value: other.to_owned() })
            },
        }
    }
}
```

在此实现中，我们使用具有四个分支的匹配表达式来检查情况。我们指示表达式应该具有 `"white"` 或 `"black"` 的文本值，或者它可以以 `#` 开头，并且应该包含恰好七个字符。否则，应返回错误以指示提供了不受支持的格式。

要实现 `Deserialization` 特性，我们需要添加 `ColorVisitor` 结构体，该结构体实现了 `serde` crate 的 `Visitor` 特性。`Visitor` 特性用于从不同的输入值中提取特定类型的值。例如，我们可以使用 `u32` 和 `str` 输入类型来反序列化十进制值。以下示例中的 `ColorVisitor` 尝试将传入的字符串解析为颜色。它具有以下实现：

```rs
struct ColorVisitor;

impl<'de> Visitor<'de> for ColorVisitor {
    type Value = Color;

    fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
        formatter.write_str("a color value expected")
    }

    fn visit_str<E>(self, value: &str) -> Result<Self::Value, E> where E: de::Error {
        value.parse::<Color>().map_err(|err| de::Error::custom(err.to_string()))
    }

    fn visit_string<E>(self, value: String) -> Result<Self::Value, E> where E: de::Error {
        self.visit_str(value.as_ref())
    }
}
```

如您所见，我们使用了 `str` 的 `parse` 方法，该方法与实现了 `FromStr` 特性的类型一起工作，将字符串转换为 `Color` 实例。我们实现了两个方法来从不同类型的字符串中提取值——第一个用于 `String` 实例，第二个用于 `str` 引用。现在我们可以将 `Color` 类型作为其他可反序列化结构体的字段添加。在我们更详细地查看处理二进制数据之前，让我们先仔细看看 `ColorError` 类型。

# 使用 failure crate 的自定义错误类型

之前的解析需要一个自己的错误类型来覆盖两种错误——数字解析错误和无效变体。让我们在本节中声明 `ColorError` 类型。

Rust 中的错误处理特别简单。有一个 `Result` 类型，它将成功和失败的结果包装在单个实体中。`Result` 将任何类型解释为错误类型，您可以使用 `try!` 宏或 `?` 操作符将一个结果转换为另一个结果。但是，将不同的错误类型连接起来要复杂得多。有一个 `std::error::Error` 特性，它为所有错误提供了一个通用接口，但它有点笨拙。为了以更用户友好的方式创建错误，您可以使用 `failure` 包。

此包有助于错误处理，并包含广泛的 `failure::Error` 类型，这些类型与其他实现 `std::Error::Error` 特性的错误兼容。您可以将实现此特性的任何错误类型转换为通用的 `failure::Error`。此包还包括可以用于派生您自己的错误类型和 `failure::Fail` 特性的宏，例如 `Backtrace`，它提供了有关错误运行时主要原因的额外信息。

在 `color.rs` 文件中声明此类型：

```rs
#[derive(Debug, Fail)]
pub enum ColorError {
    #[fail(display = "parse color's component error: {}", _0)]
    InvalidComponent(#[cause] ParseIntError),
    #[fail(display = "invalid value: {}", value)]
    InvalidValue {
        value: String,
    },
}
```

`ColorError` 枚举有两个变体：`InvalidComponent` 用于解析问题，如果提供了错误值，则为 `InvalidValue`。为了实现此类型的必要错误特性，我们使用 `#[derive(Debug, Fail)]` 属性从 `Fail` 特性派生。`Fail` 派生的 `Debug` 特性实现也是必要的。

为了创建错误消息，我们添加了 `fail` 属性，它带有 `display` 参数，该参数期望一个包含参数以插入到格式字符串中的消息。对于字段，您可以使用名称，例如 `value`，以及带有下划线前缀的数字以指示它们的字段位置。例如，要插入第一个字段，请使用名称 `0`。要将字段标记为嵌套错误，请使用 `#[cause]` 属性。

从 `Fail` 特性派生不会为用作 `ColorError` 枚举变体的类型实现 `From`。您应该自己这样做：

```rs
impl From<ParseIntError> for ColorError {
    fn from(err: ParseIntError) -> Self {
        ColorError::InvalidComponent(err)
    }
}
```

`ColorError` 类型现在可以使用 `?` 操作符，并且我们可以将随机颜色生成添加到我们的微服务中，同时进行二进制数组的洗牌。

# 二进制数据

在改进微服务之前，将所有必要的依赖项添加到 `Cargo.toml` 文件中：

```rs
failure = "0.1"
futures = "0.1"
hyper = "0.12"
rand = "0.5"
serde = "1.0"
serde_derive = "1.0"
serde_json = "1.0"
base64 = "0.9"
base64-serde = "0.3"
```

我们使用了很多与 Rust 和 crates 一起工作得很好的依赖项。如您所注意到的，我们已添加了 `base64` 包和 `base64-serde` 包。第一个是一个二进制到文本的转换器，第二个是 `serde` 序列化过程中的转换器所必需的。将这些全部导入到 `main.rs` 文件中：

```rs
#[macro_use]
extern crate failure;
extern crate futures;
extern crate hyper;
extern crate rand;
extern crate serde;
#[macro_use]
extern crate serde_derive;
extern crate serde_json;
extern crate base64;
#[macro_use]
extern crate base64_serde;

mod color;

use std::ops::Range;
use std::cmp::{max, min};
use futures::{future, Future, Stream};
use hyper::{Body, Error, Method, Request, Response, Server, StatusCode};
use hyper::service::service_fn;
use rand::Rng;
use rand::distributions::{Bernoulli, Normal, Uniform};
use base64::STANDARD;
use color::Color;
```

我们还添加了 `color` 模块，并使用该模块中的 `color:Color`。我们还从 `failure` 和 `base64_serde` 包中导入宏。

为颜色生成和数组洗牌向 `RngRequest` 枚举添加两个额外变体：

```rs
#[derive(Deserialize)]
#[serde(tag = "distribution", content = "parameters", rename_all = "lowercase")]
enum RngRequest {
    Uniform {
        #[serde(flatten)]
        range: Range<i32>,
    },
    Normal {
        mean: f64,
        std_dev: f64,
    },
    Bernoulli {
        p: f64,
    },
    Shuffle {
        #[serde(with = "Base64Standard")]
        data: Vec<u8>,
    },
    Color {
        from: Color,
        to: Color,
    },
}
```

`Shuffle` 变体的字段数据为 `Vec<u8>` 类型。由于 JSON 不支持二进制数据，我们必须将其转换为文本。我们添加了 `#[serde(with = "Base64Standard")]` 属性，这要求我们使用 `Base64Standard` 类型进行反序列化。您可以使用自己的序列化和反序列化函数自定义字段；现在，我们必须声明 `Base64Standard`：

```rs
base64_serde_type!(Base64Standard, STANDARD);
```

我们必须声明它，因为 `base64_serde` 不包含预定义的反序列化器。这是因为 Base64 需要额外的参数，这些参数不能有通用值。

`Color` 变体包含两个字段，可以用来指定颜色生成的范围。

向 `RngResponse` 枚举添加一些新的响应变体：

```rs
#[derive(Serialize)]
#[serde(rename_all = "lowercase")]
enum RngResponse {
    Value(f64),
    #[serde(with = "Base64Standard")]
    Bytes(Vec<u8>),
    Color(Color),
}
```

现在，我们必须通过添加额外的变体来改进 `handle_request` 函数：

```rs
fn handle_request(request: RngRequest) -> RngResponse {
    let mut rng = rand::thread_rng();
    match request {
        RngRequest::Uniform { range } => {
            let value = rng.sample(Uniform::from(range)) as f64;
            RngResponse::Value(value)
        },
        RngRequest::Normal { mean, std_dev } => {
            let value = rng.sample(Normal::new(mean, std_dev)) as f64;
            RngResponse::Value(value)
        },
        RngRequest::Bernoulli { p } => {
            let value = rng.sample(Bernoulli::new(p)) as i8 as f64;
            RngResponse::Value(value)
        },
        RngRequest::Shuffle { mut data } => {
            rng.shuffle(&mut data);
            RngResponse::Bytes(data)
        },
        RngRequest::Color { from, to } => {
            let red = rng.sample(color_range(from.red, to.red));
            let green = rng.sample(color_range(from.green, to.green));
            let blue = rng.sample(color_range(from.blue, to.blue));
            RngResponse::Color(Color { red, green, blue})
        },
    }
}
```

我们在这里稍微重构了代码，并增加了两个额外的分支。第一个分支，对于 `RngRequest::Shuffle` 变体，使用 `Rng` 特性的 `shuffle` 方法来打乱传入的二进制数据，并将其作为转换为 Base64 文本返回。

第二个变体，`RngRequest::Color`，使用我们将声明的 `color_range` 函数。此分支在范围内生成三种颜色并返回生成的颜色。让我们来探索 `color_range` 函数：

```rs
fn color_range(from: u8, to: u8) -> Uniform<u8> {
    let (from, to) = (min(from, to), max(from, to));
    Uniform::new_inclusive(from, to)
}
```

此函数使用 `from` 和 `to` 值创建一个新的 `Uniform` 分布，包含的范围。我们现在准备编译和测试我们的微服务。

# 编译、运行和测试

使用 `cargo run` 命令编译此示例并运行它。使用 `curl` 向服务发送请求。在第一个请求中，我们将使用均匀分布生成一个随机数：

```rs
$ curl --header "Content-Type: application/json" --request POST \
 --data '{"distribution": "uniform", "parameters": {"start": -100, "end": 100}}' \
 http://localhost:8080/random
```

我们向 `localhost:8080/random` URL 发送了一个 `POST` 请求，带有 JSON 主体。这将返回 `{"value":-55.0}`。

下一个命令请求对 `"1234567890"` 二进制字符串进行 Base64 打乱：

```rs
$ curl --header "Content-Type: application/json" --request POST \
 --data '{"distribution": "shuffle", "parameters": { "data": "MTIzNDU2Nzg5MA==" } }' \
 http://localhost:8080/random
```

预期的响应将是 `{"bytes":"MDk3NjgxNDMyNQ=="}`, 这等于字符串 `"0976814325"`. 你将得到这个请求的另一个值。

下一个请求将采用一个随机颜色：

```rs
$ curl --header "Content-Type: application/json" --request POST \
 --data '{"distribution": "color", "parameters": { "from": "black", "to": "#EC670F" } }' \
 http://localhost:8080/random
```

这里，我们使用了颜色值的两种表示：字符串值 `"black"` 和十六进制值 `"#EC670F"`。响应将类似于 `{"color":"#194A09"}`。

最后一个示例显示了如果我们尝试发送一个不支持值的请求会发生什么：

```rs
$ curl --header "Content-Type: application/json" --request POST \
 --data '{"distribution": "gamma", "parameters": { "shape": 2.0, "scale": 5.0 } }' \
 http://localhost:8080/random
```

由于服务不支持 `"gamma"` 分布，它将返回错误信息 `"unknown variant `gamma`, expected one of `uniform`, `normal`, `bernoulli`, `shuffle`, `color` at line 1 column 24"`。

# 多种格式的微服务

有时微服务必须灵活并支持多种格式。例如，一些现代客户端使用 JSON，但有些需要 XML 或其他格式。在本节中，我们将通过添加 **Concise Binary Object Representation** （**CBOR**）序列化格式来改进我们的微服务。

**CBOR**是一种基于 JSON 的二进制数据序列化格式。它更紧凑，支持二进制字符串，运行速度更快，并被定义为标准。您可以在[`tools.ietf.org/html/rfc7049`](https://tools.ietf.org/html/rfc7049)上了解更多信息。

# 不同的格式

我们需要两个额外的包：`queryst`用于从查询字符串中解析参数，以及支持 CBOR 序列化格式的`serde_cbor`包。将这些添加到您的`Cargo.toml`中：

```rs
queryst = "2.0"
 serde_cbor = "0.8"
```

此外，在`main.rs`中导入它们：

```rs
extern crate queryst;
extern crate serde_cbor;
```

我们不会在处理程序中直接使用`serde_json::to_string`，而是将其移动到一个单独的函数中，该函数根据预期的格式序列化数据：

```rs
fn serialize(format: &str, resp: &RngResponse) -> Result<Vec<u8>, Error> {
    match format {
        "json" => {
            Ok(serde_json::to_vec(resp)?)
        },
        "cbor" => {
            Ok(serde_cbor::to_vec(resp)?)
        },
        _ => {
            Err(format_err!("unsupported format {}", format))
        },
    }
}
```

在这段代码中，我们使用了一个匹配表达式来检测格式。值得注意的是，我们将`String`结果更改为二进制类型`Vec<u8>`。我们还使用了`failure::Error`作为错误类型，因为`serde_json`和`serde_cbor`都有自己的错误类型，我们可以使用`?`运算符将它们转换为通用错误。

如果提供的格式未知，我们可以使用`failure`包的`format_err!`宏构建一个`Error`。这个宏就像`println!`函数一样工作，但它基于字符串值创建一个通用错误。

我们还在导入部分更改了`Error`类型。之前它是来自`hyper`包的`hyper::Error`类型，但现在我们将使用`failure::Error`类型，并为错误使用包名前缀。

# 解析查询

HTTP URI 可以包含一个查询字符串，其中包含我们可以用来调整请求的参数。`Request`类型有一个`uri`方法，如果可用，它返回一个查询字符串。我们添加了`queryst`包，它将查询字符串解析为`serde_json::Value`。我们将使用这个值从查询字符串中提取`"format"`参数。如果没有提供格式，我们将使用默认值`"json"`。将格式提取块添加到处理`/random`路径请求的分支中，并使用我们之前声明的`serialize`函数：

```rs
(&Method::POST, "/random") => {
    let format = {
        let uri = req.uri().query().unwrap_or("");
        let query = queryst::parse(uri).unwrap_or(Value::Null);
        query["format"].as_str().unwrap_or("json").to_string()
    };
    let body = req.into_body().concat2()
        .map(move |chunks| {
            let res = serde_json::from_slice::<RngRequest>(chunks.as_ref())
                .map(handle_request)
                .map_err(Error::from)
                .and_then(move |resp| serialize(&format, &resp));
            match res {
                Ok(body) => {
                    Response::new(body.into())
                },
                Err(err) => {
                    Response::builder()
                        .status(StatusCode::UNPROCESSABLE_ENTITY)
                        .body(err.to_string().into())
                        .unwrap()
                },
            }
        });
    Box::new(body)
},
```

此代码从查询字符串中提取格式值，处理请求，并使用所选格式返回序列化值。

# 检查不同的格式

编译代码，运行它，并使用`curl`检查结果。首先，让我们检查传统的 JSON 格式：

```rs
$ curl --header "Content-Type: application/json" --request POST \
 --data '{"distribution": "uniform", "parameters": {"start": -100, "end": 100}}' \
 "http://localhost:8080/random?format=json"
```

这将返回我们之前看到的 JSON 响应：`{"value":-19.0}`。下一个请求将返回一个 CBOR 值：

```rs
$ curl --header "Content-Type: application/json" --request POST \
 --data '{"distribution": "uniform", "parameters": {"start": -100, "end": 100}}' \
 "http://localhost:8080/random?format=cbor"
```

这个命令不会在控制台打印响应，因为它是以二进制格式。您将看到以下警告消息：

```rs
Warning: Binary output can mess up your terminal. Use "--output -" to tell 
Warning: curl to output it to your terminal anyway, or consider "--output 
Warning: <FILE>" to save to a file.
```

让我们尝试以 XML 格式请求一个响应：

```rs
$ curl --header "Content-Type: application/json" --request POST \
 --data '{"distribution": "uniform", "parameters": {"start": -100, "end": 100}}' \
 "http://localhost:8080/random?format=xml"
```

这已经正确工作；它打印了`unsupported format xml`来指示它不支持 XML 格式。现在让我们继续讨论`serde`值的转码，并看看为什么 XML 不是`serde`广泛支持的格式。

# 转码

有时，你可能会遇到需要将一种格式转换为另一种格式的情况。在这种情况下，你可以使用`serde_transcode` crate，它通过标准的`serde`序列化过程将一种格式转换为另一种格式。该 crate 有一个`transcode`函数，它期望一个`serializer`和一个`deserializer`作为参数。你可以如下使用它：

```rs
let mut deserializer = serde_json::Deserializer::from_reader(reader);
let mut serializer = serde_cbor::Serializer::pretty(writer);
serde_transcode::transcode(&mut deserializer, &mut serializer).unwrap();
```

此代码将传入的 JSON 数据转换为 CBOR 数据。你可以在以下链接中了解更多关于此 crate 的信息：[`crates.io/crates/serde-transcode`](https://crates.io/crates/serde-transcode)。

# XML 支持

`serde`对 XML 的支持并不很好。主要原因在于格式的复杂性。它有如此多的规则和例外，以至于你不能用简单形式描述预期的格式。然而，也有一些实现与`serde`不兼容。以下链接解释了流式传输、读取和写入 XML 数据：[`crates.io/crates/xml-rs`](https://crates.io/crates/xml-rs)和[`crates.io/crates/quick-xml`](https://crates.io/crates/quick-xml)。

另一种与`serde`基础设施不兼容的格式是 Protocol Buffers ([`developers.google.com/protocol-buffers/`](https://crates.io/crates/protobuf))。开发者经常出于性能原因和为了在不同应用程序中使用一个数据方案而选择此格式。要在 Rust 代码中使用此格式，请尝试使用`protobuf` crate：[`crates.io/crates/protobuf`](https://crates.io/crates/protobuf)。

在我看来，出于以下原因，最好使用与 Rust 中的`serde`crate 兼容的格式：

+   在代码中使用它更简单。

+   结构体不需要方案，因为它们是严格的。

+   你可以使用一个具有确定协议的独立 crate。

你不应该遵循`serde`方法的唯一情况是，如果你必须支持一个与`serde`不兼容的格式，但已在现有服务或客户端中使用。

# 摘要

在本章中，我们讨论了使用`serde`crate 进行序列化和反序列化过程。我们探讨了`serde`如何支持多种格式，并可以自动为结构体或枚举派生`Serialize`和`Deserialize`实现。我们实现了一个服务，该服务从以 JSON 格式序列化的传入参数生成随机数。

之后，你学习了如何自己实现这些特性，并为洗牌数组或生成随机颜色添加额外功能。最后，我们讨论了如何在单个处理器中支持多种格式。

在下一章中，你将学习如何使用异步代码的完整潜力，并用 Rust 编写可以同时处理数千个客户端的微服务。
