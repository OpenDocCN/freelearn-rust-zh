# 第四章：数据序列化、反序列化和解析

在上一章中，我们介绍了在 Rust 中编写简单的套接字服务器。传输协议，如 TCP 和 UDP，仅提供传输消息的机制，因此需要更高层次的协议来实际构建和发送这些消息。此外，TCP 和 UDP 协议始终处理字节；我们在将字符串发送到套接字之前调用 `as_bytes` 时看到了这一点。将数据转换为可以存储或传输的格式（在网络的案例中是字节流）的过程称为序列化。相反的过程是反序列化，它将原始数据格式转换为数据结构。任何网络软件都必须处理接收到的或即将发送的数据的序列化和反序列化。对于更复杂的数据类型，如用户定义的数据类型，或甚至是简单的集合类型，这种简单的转换并不总是可能的。Rust 生态系统有一些特殊的包可以处理各种情况。

在本章中，我们将涵盖以下主题：

+   使用 Serde 进行序列化和反序列化。我们将从基本用法开始，然后继续介绍如何使用 Serde 编写自定义序列化器。

+   使用 nom 解析文本数据。

+   最后一个主题将是解析二进制数据，这是网络中非常常用的一种技术。

# 使用 Serde 进行序列化和反序列化

Serde 是 Rust 中序列化和反序列化数据的既定标准方式。Serde 支持多种数据结构，它可以直接将这些数据结构序列化成多种给定的数据格式（包括 JSON、TOML 和 CSV）。理解 Serde 的最简单方式是将其视为一个可逆函数，它将给定的数据结构转换成字节流。除了标准数据类型外，Serde 还提供了一些宏，这些宏可以应用于用户定义的数据类型，使它们（反）序列化。

在 第二章，“Rust 及其生态系统简介”中，我们讨论了如何使用过程宏为给定的数据类型实现自定义推导。Serde 使用该机制提供两个自定义推导，分别命名为 `Serialize` 和 `Deserialize`，这些推导可以应用于由 Serde 支持的数据类型组成的用户定义数据类型。让我们看看一个小例子，看看它是如何工作的。我们首先使用 Cargo 创建一个空项目：

```rs
$ cargo new --bin serde-basic
```

这是 Cargo 清单应该看起来像的样子：

```rs
[package]
name = "serde-basic"
version = "0.1.0"
authors = ["Foo<foo@bar.com>"]

[dependencies]
serde = "1.0"
serde_derive = "1.0"
serde_json = "1.0"
serde_yaml = "0.7.1"
```

`serde` 包是 Serde 生态系统的核心。`serde_derive` 包提供了必要的工具，使用过程宏来推导 `Serialize` 和 `Deserialize`。接下来的两个包分别提供了 Serde 特定的功能，用于将数据序列化和反序列化为 JSON 和 YAML：

```rs
// chapter4/serde-basic/src/main.rs

#[macro_use]
extern crate serde_derive;

extern crate serde;
extern crate serde_json;
extern crate serde_yaml;

// We will serialize and deserialize instances of
// this struct
#[derive(Serialize, Deserialize, Debug)]
struct ServerConfig {
    workers: u64,
    ignore: bool,
    auth_server: Option<String>
}

fn main() {
    let config = ServerConfig {
                workers: 100, 
                ignore: false, 
                auth_server: Some("auth.server.io".to_string())
            };
    {
        println!("To and from YAML");
        let serialized = serde_yaml::to_string(&config).unwrap();
        println!("{}", serialized);
        let deserialized: ServerConfig =
        serde_yaml::from_str(&serialized).unwrap();
        println!("{:?}", deserialized);
    }
    println!("\n\n");
    {
        println!("To and from JSON");
        let serialized = serde_json::to_string(&config).unwrap();
        println!("{}", serialized);
        let deserialized: ServerConfig =
        serde_json::from_str(&serialized).unwrap();
        println!("{:?}", deserialized);
    }
}
```

由于 `serde_derive` crate 导出宏，我们需要用 `macro_use` 声明来标记它；然后我们声明所有依赖项为 `extern` crate。设置好这些后，我们可以定义我们的自定义数据类型。在这种情况下，我们感兴趣的是具有不同类型参数的服务器配置。`auth_server` 参数是可选的，这就是为什么它被包裹在 `Option` 中。我们的 struct 从 Serde 继承了两个特质，还继承了编译器提供的 `Debug` 特质，我们将在反序列化后使用它来显示。在我们的主函数中，我们实例化我们的类，并对其调用 `serde_yaml::to_string` 来将其序列化为字符串；其逆操作是 `serde_yaml::from_str`。

一个示例运行应该看起来像这样：

```rs
$ cargo run
   Compiling serde-basic v0.1.0 (file:///Users/Abhishek/Desktop/rust-book/src/chapter4/serde-basic)
    Finished dev [unoptimized + debuginfo] target(s) in 1.88 secs
     Running `target/debug/serde-basic`
To and from YAML
---
workers: 100
ignore: false
auth_server: auth.server.io
ServerConfig { workers: 100, ignore: false, auth_server: Some("auth.server.io") }

To and from JSON
{"workers":100,"ignore":false,"auth_server":"auth.server.io"}
ServerConfig { workers: 100, ignore: false, auth_server: Some("auth.server.io") }
```

让我们继续一个更高级的示例，展示如何在网络上使用 Serde。在这个例子中，我们将设置一个 TCP 服务器和一个客户端。这部分将与我们上一章中做的一样。但这次，我们的 TCP 服务器将作为一个计算器运行，它接受一个沿三个轴有三个分量的 3D 空间中的点，并返回它在同一参考系中与原点的距离。让我们这样设置我们的 Cargo 项目：

```rs
$ cargo new --bin serde-server
```

清单应该看起来像这样：

```rs
$ cat Cargo.toml
[package]
name = "serde-server"
version = "0.1.0"
authors = ["Foo <foo@bar.com>"]

[dependencies]
serde = "1.0"
serde_derive = "1.0"
serde_json = "1.0"
```

通过这种方式，我们就可以继续定义我们的代码。在这个例子中，服务器和客户端将在同一个二进制文件中。应用程序将接受一个标志，指示它应该作为服务器还是客户端运行。正如我们在上一章中做的那样，在服务器的情况下，我们将绑定到已知端口上的所有本地接口并监听传入的连接。客户端的情况将连接到该已知端口，并在控制台上等待用户输入。客户端期望输入为三个整数，每个轴一个，用逗号分隔。在获取输入后，客户端构建一个给定定义的 struct，使用 Serde 进行序列化，并将字节流发送到服务器。服务器将流反序列化为相同类型的 struct。然后它计算距离并发送回结果，客户端随后显示该结果。代码如下：

```rs
// chapter4/serde-server/src/main.rs

#[macro_use]
extern crate serde_derive;

extern crate serde;
extern crate serde_json;

use std::net::{TcpListener, TcpStream};
use std::io::{stdin, BufRead, BufReader, Error, Write};
use std::{env, str, thread};

#[derive(Serialize, Deserialize, Debug)]
struct Point3D {
    x: u32,
    y: u32,
    z: u32,
}

// Like previous examples of vanilla TCP servers, this function handles
// a single client.
fn handle_client(stream: TcpStream) -> Result<(), Error> {
    println!("Incoming connection from: {}", stream.peer_addr()?);
    let mut data = Vec::new();
    let mut stream = BufReader::new(stream);

    loop {
        data.clear();

        let bytes_read = stream.read_until(b'\n', &mut data)?;
        if bytes_read == 0 {
            return Ok(());
        }
        let input: Point3D = serde_json::from_slice(&data)?;
        let value = input.x.pow(2) + input.y.pow(2) + input.z.pow(2);

        write!(stream.get_mut(), "{}", f64::from(value).sqrt())?;
        write!(stream.get_mut(), "{}", "\n")?;
    }
}

fn main() {
    let args: Vec<_> = env::args().collect();
    if args.len() != 2 {
        eprintln!("Please provide --client or --server as argument");
        std::process::exit(1);
    }
    // The server case
    if args[1] == "--server" {
        let listener = TcpListener::bind("0.0.0.0:8888").expect("Could
        not bind");
        for stream in listener.incoming() {
            match stream {
                Err(e) => eprintln!("failed: {}", e),
                Ok(stream) => {
                    thread::spawn(move || {
                        handle_client(stream).unwrap_or_else(|error|
                        eprintln!("{:?}", error));
                    });
                }
            }
        }
    }
    // Client case begins here
    else if args[1] == "--client" {
        let mut stream = TcpStream::connect("127.0.0.1:8888").expect("Could not connect to server");
        println!("Please provide a 3D point as three comma separated
        integers");
        loop {
            let mut input = String::new();
            let mut buffer: Vec<u8> = Vec::new();
            stdin()
                .read_line(&mut input)
                .expect("Failed to read from stdin");
            let parts: Vec<&str> = input
                                    .trim_matches('\n')
                                    .split(',')
                                    .collect();
            let point = Point3D {
                x: parts[0].parse().unwrap(),
                y: parts[1].parse().unwrap(),
                z: parts[2].parse().unwrap(),
            };
            stream
                .write_all(serde_json::to_string(&point).unwrap().as_bytes())
                .expect("Failed to write to server");
            stream.write_all(b"\n").expect("Failed to write to
            server");

            let mut reader = BufReader::new(&stream);
            reader
                .read_until(b'\n', &mut buffer)
                .expect("Could not read into buffer");
            let input = str::from_utf8(&buffer).expect("Could not write
            buffer as string");
            if input == "" {
                eprintln!("Empty response from server");
            }
            print!("Response from server {}", input);
        }
    }
}
```

我们首先设置 Serde，就像在上一例子中做的那样。然后我们定义我们的 3D 点为一个包含三个元素的 struct。在我们的主函数中，我们处理 CLI 参数，并根据传递的内容分支到客户端或服务器。在两种情况下，我们都通过发送换行符来表示传输结束。客户端从 `stdin` 读取一行，清理它，并在循环中创建 struct 的实例。在两种情况下，我们都用 `BufReader` 包装我们的流，以便更容易处理。我们使用 Cargo 运行我们的代码。服务器上的一个示例会话如下：

```rs
server$ cargo run -- --server
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/serde-server --server`
Incoming connection from: 127.0.0.1:49630
```

在客户端方面，我们看到以下与服务器交互。正如预期的那样，客户端读取输入，对其进行序列化，并将其发送到服务器。然后它等待响应，并在收到响应时将结果打印到标准输出：

```rs
client$ cargo run -- --client
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/serde-server --client`
Please provide a 3D point as three comma separated integers
1,2,3
Response from server 3.7416573867739413
3,4,5
Response from server 7.0710678118654755
4,5,6
Response from server 8.774964387392123
```

# 自定义序列化和反序列化

如我们之前所见，Serde 通过宏为所有原始数据类型以及许多复杂数据类型提供了内置的序列化和反序列化功能。然而，在某些情况下，Serde 可能无法自动实现。这可能发生在更复杂的数据类型上。在这些情况下，您将需要手动实现这些。这些示例展示了 Serde 的高级用法，这也允许在输出中重命名字段。对于日常使用，几乎从不必要使用这些高级功能。这些可能更常见于网络通信，处理新的协议等情况。

假设我们有一个包含三个字段的结构体。我们将假设 Serde 无法实现 `Serialize` 和 `Deserialize`，因此我们需要手动实现这些。我们使用 Cargo 初始化我们的项目：

```rs
$ cargo new --bin serde-custom
```

然后，我们声明我们的依赖项；生成的 Cargo 配置文件应如下所示：

```rs
[package]
name = "serde-custom"
version = "0.1.0"
authors = ["Foo <foo@bar.com>"]

[dependencies]
serde = "1.0"
serde_derive = "1.0"
serde_json = "1.0"
serde_test = "1.0"
```

我们的结构体看起来像这样：

```rs
// chapter4/serde-custom/src/main.rs

// We will implement custom serialization and deserialization
// for this struct
#[derive(Debug, PartialEq)]
struct KubeConfig {
    port: u8,
    healthz_port: u8,
    max_pods: u8,
}
```

我们需要为 Serde 使用内部功能而派生 `Debug` 和 `PartialEq`。在现实世界中，可能还需要手动实现这些。现在，我们需要为 kubeconfig 实现 `Serialize` 特征。这个特征看起来像这样：

```rs
pub trait Serialize {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
        where S: Serializer;
}
```

序列化我们的结构体的基本工作流程将是简单地序列化结构体名称，然后是每个元素，然后按此顺序发出序列化结束的信号。Serde 内置了可以与所有基本类型一起工作的序列化方法，因此实现不需要担心处理内置类型。让我们看看我们如何序列化我们的结构体：

```rs
// chapter4/serde-custom/src/main.rs

// Implementing Serialize for our custom struct defines
// how instances of that struct should be serialized.
// In essence, serialization of an object is equal to
// sum of the serializations of it's components
impl Serialize for KubeConfig {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
        where S: Serializer
    {
        let mut state = serializer.serialize_struct("KubeConfig", 3)?;
        state.serialize_field("port", &self.port)?;
        state.serialize_field("healthz_port", &self.healthz_port)?;
        state.serialize_field("max_pods", &self.max_pods)?;
        state.end()
    }
}
```

结构体的序列化将始终以调用 `serialize_struct` 方法开始，该方法以结构体名称和字段数量作为参数（对于其他类型也有类似命名的函数）。然后，我们按顺序序列化每个字段，同时传递一个将在结果 json 中使用的键名。一旦完成，我们调用特殊的 `end` 方法作为信号。

实现反序列化稍微复杂一些，有一些样板代码。相关的特征看起来像这样：

```rs
pub trait Deserialize<'de>: Sized {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
        where D: Deserializer<'de>;
}
```

对于类型的实现，需要实现访问者模式。Serde 定义了一个特殊的 `Visitor` 特征，如下所示样本所示。请注意，这为所有内置类型提供了 `visit_*` 方法，但在此处未显示。此外，在以下样本中，我们使用符号 `...` 来表示还有更多方法，这些方法对于我们的讨论并不重要。

```rs
pub trait Visitor<'de>: Sized {
    type Value;
    fn expecting(&self, formatter: &mut Formatter) -> Result;
    fn visit_bool<E>(self, v: bool) -> Result<Self::Value,
     E>
    where
        E: Error,
    { }
    ...
}
```

这个特征的实现被反序列化器内部使用，以构建结果类型。在我们的情况下，它将如下所示：

```rs
// chapter4/serde-custom/src/main.rs

// Implementing Deserialize for our struct defines how
// an instance of the struct should be created from an
// input stream of bytes
impl<'de> Deserialize<'de> for KubeConfig {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
        where D: Deserializer<'de>
    {
        enum Field { Port, HealthzPort, MaxPods };

        impl<'de> Deserialize<'de> for Field {
            fn deserialize<D>(deserializer: D) ->
            Result<Field,
            D::Error>
                where D: Deserializer<'de>
            {
                struct FieldVisitor;

                impl<'de> Visitor<'de> for FieldVisitor {
                    type Value = Field;

                    fn expecting(&self, formatter: &mut fmt::Formatter)
                    -> fmt::Result {
                        formatter.write_str("`port` or `healthz_port`
                        or `max_pods`")
                    }

                    fn visit_str<E>(self, value: &str) ->
                    Result<Field,
                    E>
                        where E: de::Error
                    {
                        match value {
                            "port" => Ok(Field::Port),
                            "healthz_port" =>
                            Ok(Field::HealthzPort),
                            "max_pods" => Ok(Field::MaxPods),
                            _ => Err(de::Error::unknown_field(value, 
                            FIELDS)),
                        }
                    }
                }

                deserializer.deserialize_identifier(FieldVisitor)
            }
        }
}
```

现在，反序列化器的输入是 json，它可以被视为一个映射。因此，我们只需要实现 `Visitor` 特征中的 `visit_map`。如果将任何非 json 数据传递给我们的反序列化器，它将在调用该特征中的某些其他函数时出错。大部分之前的实现都是样板代码。它归结为几个部分：为字段实现 `Visitor`，以及实现 `visit_str`（因为我们的所有字段都是字符串）。在这个阶段，我们应该能够反序列化单个字段。第二部分是实现整体结构的 `Visitor`，并实现 `visit_map`。在所有情况下都必须适当地处理错误。最后，我们可以调用 `deserializer.deserialize_struct` 并传递结构体的名称、字段列表以及整个结构的访问者实现。

此实现将看起来像这样：

```rs
// chapter4/serde-custom/src/main.rs

impl<'de> Deserialize<'de> for KubeConfig {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
        where D: Deserializer<'de>
    {
        struct KubeConfigVisitor;

        impl<'de> Visitor<'de> for KubeConfigVisitor {
            type Value = KubeConfig;

            fn expecting(&self, formatter: &mut fmt::Formatter) -> 
            fmt::Result {
                formatter.write_str("struct KubeConfig")
            }

            fn visit_map<V>(self, mut map: V) ->
            Result<KubeConfig, 
            V::Error>
                where V: MapAccess<'de>
            {
                let mut port = None;
                let mut hport = None;
                let mut max = None;
                while let Some(key) = map.next_key()? {
                    match key {
                        Field::Port => {
                            if port.is_some() {
                                return 
                            Err(de::Error::duplicate_field("port"));
                            }
                            port = Some(map.next_value()?);
                        }
                        Field::HealthzPort => {
                            if hport.is_some() {
                                return
                            Err(de::Error::duplicate_field
                            ("healthz_port"));
                            }
                            hport = Some(map.next_value()?);
                        }
                        Field::MaxPods => {
                            if max.is_some() {
                                return 
                                Err(de::Error::duplicate_field
                                ("max_pods"));
                            }
                            max = Some(map.next_value()?);
                        }
                    }
                }
                let port = port.ok_or_else(||
                de::Error::missing_field("port"))?;
                let hport = hport.ok_or_else(||
                de::Error::missing_field("healthz_port"))?;
                let max = max.ok_or_else(||
                de::Error::missing_field("max_pods"))?;
                Ok(KubeConfig {port: port, healthz_port: hport,
                max_pods: max})
            }
        }

        const FIELDS: &'static [&'static str] = &["port",
        "healthz_port", "max_pods"];
        deserializer.deserialize_struct("KubeConfig", FIELDS,
        KubeConfigVisitor)
    }
}
```

Serde 还提供了一个可以用于使用类似令牌流界面的接口来对自定义序列化器和反序列化器进行单元测试的 crate。要使用它，我们需要将 `serde_test` 添加到我们的 `Cargo.toml` 中，并在我们的主文件中将其声明为外部 crate。以下是我们反序列化器的测试用例：

```rs
// chapter4/serde-custom/src/main.rs

#[test]
fn test_ser_de() {
    let c = KubeConfig { port: 10, healthz_port: 11, max_pods: 12};

    assert_de_tokens(&c, &[
        Token::Struct { name: "KubeConfig", len: 3 },
        Token::Str("port"),
        Token::U8(10),
        Token::Str("healthz_port"),
        Token::U8(11),
        Token::Str("max_pods"),
        Token::U8(12),
        Token::StructEnd,
    ]);
}
```

`assert_de_tokens` 调用检查给定的令牌流是否反序列化为我们的结构体，从而测试我们的反序列化器。我们还可以添加一个主函数来驱动序列化器，如下所示：

```rs
// chapter4/serde-custom/src/main.rs

fn main() {
    let c = KubeConfig { port: 10, healthz_port: 11, max_pods: 12};
    let serialized = serde_json::to_string(&c).unwrap();
    println!("{:?}", serialized);
}
```

所有这些现在都可以使用 Cargo 运行。使用 `cargo test` 应该运行我们刚刚编写的测试，它应该通过。`cargo run` 应该运行主函数并打印序列化的 json：

```rs
$ cargo test
   Compiling serde-custom v0.1.0 (file:///serde-custom)
    Finished dev [unoptimized + debuginfo] target(s) in 0.61 secs
     Running target/debug/deps/serde_custom-81ee5105cf257563

running 1 test
test test_ser_de ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

$ cargo run
   Compiling serde-custom v0.1.0 (file:///serde-custom)
    Finished dev [unoptimized + debuginfo] target(s) in 0.54 secs
     Running `target/debug/serde-custom`
    "{\"port\":10,\"healthz_port\":11,\"max_pods\":12}"
```

# 解析文本数据

数据解析是与反序列化密切相关的问题。思考解析的最常见方式是从形式语法开始，并基于此构建解析器。这导致了一个自下而上的解析器，其中较小的规则解析整个输入的较小组件。一个最终的组合规则按照给定的顺序将所有较小的规则组合起来形成最终的解析器。这种形式定义有限规则集的方式称为 **解析表达式语法**（**PEG**）。这确保了解析是无歧义的；如果解析成功，则只有一个有效的解析树。在 Rust 生态系统中，有几种不同的方法可以实现 PEG，每种方法都有其自身的优点和缺点。第一种方法是使用宏来定义解析的领域特定语言。

此方法通过新的宏系统很好地与编译器集成，并可以生成快速代码。然而，这通常更难调试和维护。由于此方法不允许重载运算符，实现必须定义一个 DSL，这可能会给学习者带来更大的认知负担。第二种方法是使用特征系统。这种方法有助于定义自定义运算符，并且通常更容易调试和维护。使用第一种方法的解析器示例是 nom；使用第二种方法的解析器示例是 pom 和 pest。

我们解析的使用场景主要在网络应用程序的上下文中。在这些情况下，有时处理原始字符串（或字节流）并解析所需信息比反序列化到复杂的数据结构更有用。这种情况的一个常见例子是任何基于文本的协议，如 HTTP。服务器可能会通过套接字接收一个原始请求作为字节流，并将其解析以提取信息。在本节中，我们将研究 Rust 生态系统中的常见解析技术。

现在，nom 是一个解析器组合框架，这意味着它可以组合较小的解析器来构建更强大的解析器。这是一个自下而上的方法，通常从编写非常具体的解析器开始，这些解析器从输入中解析一个定义良好的东西。然后框架提供方法将这些小型解析器链接成一个完整的解析器。这种方法与 lex 和 yacc 的情况中的自顶向下方法形成对比，在那里人们会从定义语法开始。它可以处理字节流（二进制数据）或字符串，并提供 Rust 的所有常用保证。让我们从解析一个简单的字符串开始，在这个例子中是一个 HTTP GET 或 POST 请求。像所有 cargo 项目一样，我们首先设置结构：

```rs
$ cargo new --bin nom-http
```

然后我们将添加我们的依赖项（在这个例子中是 nom）。生成的清单应该看起来像这样：

```rs
$ cat Cargo.toml
[package]
name = "nom-http"
version = "0.1.0"
authors = ["Foo<foo@bar.com>"]

[dependencies.nom]
version = "3.2.1"
features = ["verbose-errors", "nightly"]
```

该包提供了一些额外的功能，这些功能在调试时通常很有用；默认情况下这些功能是禁用的，可以通过传递列表到`features`标志来打开，如前例所示。现在，让我们继续到我们的主文件：

```rs
// chapter4/nom-http/src/main.rs

#[macro_use]
extern crate nom;

use std::str;
use nom::{ErrorKind, IResult};

#[derive(Debug)]
enum Method {
    GET,
    POST,
}

#[derive(Debug)]
struct Request {
    method: Method,
    url: String,
    version: String,
}

// A parser that parses method out of a HTT request
named!(parse_method<&[u8], Method>,
       return_error!(ErrorKind::Custom(12), alt!(map!(tag!("GET"), |_| Method::GET) | map!(tag!("POST"), |_| Method::POST))));

// A parser that parses the request part
named!(parse_request<&[u8], Request>, ws!(do_parse!(
    method: parse_method >>
    url: map_res!(take_until!(" "), str::from_utf8) >>
    tag!("HTTP/") >>
    version: map_res!(take_until!("\r"), str::from_utf8) >>
    (Request { method: method, url: url.into(), version: version.into() })
)));

// Driver function for running the overall parser
fn run_parser(input: &str) {
    match parse_request(input.as_bytes()) {
      IResult::Done(rest, value) => println!("Rest: {:?} Value: {:?}",
      rest, value),
      IResult::Error(err) => println!("{:?}", err),
      IResult::Incomplete(needed) => println!("{:?}", needed)
    }
}

fn main() {
    let get = "GET /home/ HTTP/1.1\r\n";
    run_parser(get);
    let post = "POST /update/ HTTP/1.1\r\n";
    run_parser(post);
    let wrong = "WRONG /wrong/ HTTP/1.1\r\n";
    run_parser(wrong);
}
```

如可能显而易见，nom 在代码生成中大量使用宏，其中最重要的一个是`named!`，它接受一个函数签名并根据该签名定义一个解析器。nom 解析器返回`IResult`类型的实例；这被定义为枚举，并具有三个变体：

+   `Done(rest, value)`变体表示当前解析器成功的情形。在这种情况下，值将包含当前解析的值，而剩余的将包含需要解析的剩余输入。

+   `Error(Err<E>)`变体表示解析过程中的错误。底层错误将包含错误代码、错误位置以及更多信息。在一个大的解析树中，这也可以包含指向更多错误的指针。

+   最后一个变体`Incomplete(needed)`表示由于某种原因解析不完整的情况。需要的枚举再次有两个变体；第一个变体表示不知道需要多少数据。第二个变体表示所需数据的精确大小。

我们从 HTTP 方法的表示和完整的请求作为结构体开始。在我们的玩具示例中，我们只处理 GET 和 POST 方法，忽略其他所有内容。然后我们定义了一个用于 HTTP 方法的解析器；我们的解析器将接受一个字节数组切片并返回`Method`枚举。这很简单，只需读取输入并查找字符串 GET 或 POST。在每种情况下，基础解析器都是使用`tag!`宏构建的，该宏解析输入以提取给定的字符串。如果解析成功，我们使用`map!`宏将结果转换为`Method`，该宏将解析器的结果映射到函数。现在，对于解析方法，我们可能有一个 POST 或一个 GET，但不可能两者都有。我们使用`alt!`宏来表示之前构建的两个解析器的逻辑或。`alt!`宏将构建一个解析器，如果其构成宏中的任何一个可以解析给定的输入，则解析输入。最后，所有这些都被`return_error!`宏包裹起来，如果在当前解析器中解析失败，则提前返回，而不是传递到树中的下一个解析器。

然后，我们继续通过定义`parse_request`来解析整个请求。我们首先使用`ws!`宏从输入中去除额外的空白。然后我们调用`do_parse!`宏，该宏链接着多个子解析器。这个与其他组合器不同，因为它允许存储中间解析器的结果。这在构建结构体实例的同时返回结果时很有用。在`do_parse!`中，我们首先调用`parse_method`并将结果存储在一个变量中。从请求中移除方法后，我们应该在找到对象位置之前遇到空白的空格。这由`take_until!(" ")`调用处理，它消耗输入直到找到空格。结果使用`map_res!`转换为`str`。列表中的下一个解析器是使用`tag!`宏移除序列`HTTP/`的解析器。接下来，我们通过读取输入直到看到`\r`来解析 HTTP 版本，并将其映射回`str`。一旦完成所有解析，我们就构建一个`Request`对象并返回它。注意在解析器序列中使用`>>`符号作为分隔符。

我们还定义了一个名为`run_parser`的辅助函数，用于在给定输入中运行我们的解析器并打印结果。该函数调用解析器并匹配结果以显示结果结构或错误。然后我们定义我们的主函数，包含三个 HTTP 请求，前两个是有效的，最后一个无效，因为方法错误。运行此函数后，输出如下：

```rs
$ cargo run
   Compiling nom-http v0.1.0 (file:///Users/Abhishek/Desktop/rust-book/src/ch4/nom-http)
    Finished dev [unoptimized + debuginfo] target(s) in 0.60 secs
     Running `target/debug/nom-http`
Rest: [] Value: Request { method: GET, url: "/home/", version: "1.1" }
Rest: [] Value: Request { method: POST, url: "/update/", version: "1.1" }
NodePosition(Custom(128), [87, 82, 79, 78, 71, 32, 47, 119, 114, 111, 110, 103, 47, 32, 72, 84, 84, 80, 47, 49, 46, 49, 13, 10], [Position(Alt, [87, 82, 79, 78, 71, 32, 47, 119, 114, 111, 110, 103, 47, 32, 72, 84, 84, 80, 47, 49, 46, 49, 13, 10])])
```

在前两种情况下，所有内容都按预期解析，并返回了结果。正如预期的那样，在最后一种情况下解析失败，并返回了自定义错误。

正如我们之前讨论的，nom 的一个常见问题是调试，因为宏的调试要困难得多。宏还鼓励使用特定的 DSL（如使用 `>>` 分隔符），这可能会让一些人难以使用。在撰写本文时，nom 的一些错误消息在查找给定解析器的问题时并不足够有帮助。这些将在未来肯定会有所改进，但在此期间，nom 提供了一些辅助宏来帮助调试。

例如，`dbg!` 在底层解析器没有返回 `Done` 时打印结果和输入。`dbg_dump!` 宏类似，但还会打印出输入缓冲区的十六进制转储。根据我们的经验，可以使用一些技术进行调试：

+   通过向 `rustc` 传递编译器选项来扩展宏。Cargo 通过以下调用启用此功能：`cargo rustc -- -Z unstable-options --pretty=expanded` 将展开并格式化打印给定项目中的所有宏。有人可能会发现展开宏以跟踪执行和调试是有用的。Cargo 中相关的命令 `rustc -- -Z trace-macros` 仅展开宏。

+   独立运行较小的解析器。给定一系列解析器和另一个组合这些解析器的解析器，运行每个子解析器直到其中一个出错可能更容易。然后，可以继续调试仅失败的较小解析器。这在隔离故障时非常有用。

+   使用提供的调试宏 `dbg!` 和 `dbg_dump!`。这些可以像调试打印语句一样用于跟踪执行。

`pretty=expanded` 目前是一个不稳定的编译器选项。在未来某个时候，它将被稳定化（或删除）。在这种情况下，将不需要传递 `-Z unstable-options` 标志来使用它。

让我们看看另一个名为 `pom` 的解析器组合器的例子。正如我们之前讨论的，这个解析器组合器在很大程度上依赖于特性和运算符重载来实现解析器组合。在撰写本文时，当前版本是 1.1.0，我们将使用这个版本作为我们的示例项目。像往常一样，第一步是设置我们的项目并将 `pom` 添加到我们的依赖项中：

```rs
$ cargo new --bin pom-string
```

`Cargo.toml` 文件将看起来像这样：

```rs
[package]
name = "pom-string"
version = "0.1.0"
authors = ["Foo<foo@bar.com>"]

[dependencies]
pom = "1.1.0"
```

在这个例子中，我们将解析一个示例 HTTP 请求，就像上次一样。它看起来是这样的：

```rs
// chapter4/pom-string/src/main.rs

extern crate pom;

use pom::DataInput;
use pom::parser::{sym, one_of, seq};
use pom::parser::*;

use std::str;

// Represents one or more occurrence of an empty whitespace
fn space() -> Parser<'static, u8, ()> {
    one_of(b" \t\r\n").repeat(0..).discard()
}

// Represents a string in all lower case
fn string() -> Parser<'static, u8, String> {
    one_of(b"abcdefghijklmnopqrstuvwxyz").repeat(0..).convert(String::from_utf8)
}

fn main() {
    let get = b"GET /home/ HTTP/1.1\r\n";
    let mut input = DataInput::new(get);
    let parser = (seq(b"GET") | seq(b"POST")) * space() * sym(b'/') *
    string() * sym(b'/') * space() * seq(b"HTTP/1.1");
    let output = parser.parse(&mut input);
    println!("{:?}", str::from_utf8(&output.unwrap()));
}
```

我们首先声明对`pom`的依赖。在我们的主函数中，我们将最终的解析器定义为多个子解析器的序列。`*`运算符被重载，以便按顺序应用多个解析器。`seq`运算符是一个内置解析器，用于匹配输入中的给定字符串。`|`运算符对两个操作数进行逻辑或操作。我们定义了一个名为`space()`的函数，它表示输入中的空格。此函数接受每种空格字符中的一个，重复 0 次或多次，然后丢弃它。因此，该函数返回一个没有返回类型的`Parser`，表示为`()`。字符串函数被定义为英语字母中的一个字符，重复 0 次或多次，然后转换为`std::String`。

这个函数的返回类型是一个`Parser`，它有一个`String`，正如预期的那样。设置好这些之后，我们的主解析器将有一个空格，后面跟着符号`/`，然后是一个字符串，一个符号`/`，再次是一个空格，并以序列`HTTP/1.1`结束。正如预期的那样，当我们用我们编写的解析器解析一个示例字符串时，它产生了一个`Ok`：

```rs
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/pom-string`
Ok("HTTP/1.1")
```

基于 PEG 的解析器组合器可能更容易调试和操作。它们还倾向于产生更好的错误消息，但不幸的是，目前它们还不够成熟。围绕它们的社区不如 nom 周围的社区大。因此，在 nom 问题上通常更容易获得帮助。最终，选择适合他们的解决方案取决于程序员。

# 解析二进制数据

一个相关的问题是解析二进制数据。这种情况下常见的例子包括解析二进制文件和二进制协议。让我们看看`nom`如何被用来解析二进制数据。在我们的玩具示例中，我们将编写一个解析 IPv6 头部的解析器。我们的`Cargo.toml`将和上次一样。使用 CLI 设置项目：

```rs
$ cargo new --bin nom-ipv6
```

我们的主要文件将看起来像这样：

```rs
// chapter4/nom-ipv6/src/main.rs

#[macro_use]
extern crate nom;

use std::net::Ipv6Addr;

use nom::IResult;

// Struct representing an IPv6 header
#[derive(Debug, PartialEq, Eq)]
pub struct IPv6Header {
    version: u8,
    traffic_class: u8,
    flow_label: u32,
    payload_length: u16,
    next_header: u8,
    hop_limit: u8,
    source_addr: Ipv6Addr,
    dest_addr: Ipv6Addr,
}

// Converts a given slice of [u8] to an array of 16 u8 given by
// [u8; 16]
fn slice_to_array(input: &[u8]) -> [u8; 16] {
    let mut array = [0u8; 16];
    for (&x, p) in input.iter().zip(array.iter_mut()) {
        *p = x;
    }
    array
}

// Converts a reference to a slice [u8] to an instance of
// std::net::Ipv6Addr
fn to_ipv6_address(i: &[u8]) -> Ipv6Addr {
    let arr = slice_to_array(i);
    Ipv6Addr::from(arr)
}

// Parsers for each individual section of the header
named!(parse_version<&[u8], u8>, bits!(take_bits!(u8, 4)));
named!(parse_traffic_class<&[u8], u8>, bits!(take_bits!(u8, 8)));
named!(parse_flow_label<&[u8], u32>, bits!(take_bits!(u32, 20)));
named!(parse_payload_length<&[u8], u16>, bits!(take_bits!(u16, 16)));
named!(parse_next_header<&[u8], u8>, bits!(take_bits!(u8, 8)));
named!(parse_hop_limit<&[u8], u8>, bits!(take_bits!(u8, 8)));
named!(parse_address<&[u8], Ipv6Addr>, map!(take!(16), to_ipv6_address));

// The primary parser
named!(ipparse<&[u8], IPv6Header>,
       do_parse!(
            ver: parse_version >>
            cls: parse_traffic_class >>
            lbl: parse_flow_label >>
            len: parse_payload_length >>
            hdr: parse_next_header >>
            lim: parse_hop_limit >>
            src: parse_address >>
            dst: parse_address >>
              (IPv6Header {
                  version: ver,
                  traffic_class: cls,
                  flow_label: lbl,
                  payload_length: len,
                  next_header: hdr,
                  hop_limit: lim,
                  source_addr: src,
                  dest_addr : dst
              })
));

// Wrapper for the parser
pub fn parse_ipv6_header(i: &[u8]) -> IResult<&[u8], IPv6Header> {
    ipparse(i)
}

fn main() {
    const EMPTY_SLICE: &'static [u8] = &[];
    let bytes = [0x60,
                 0x00,
                 0x08, 0x19,
                 0x80, 0x00, 0x14, 0x06,
                 0x40,
                 0x2a, 0x02, 0x0c, 0x7d, 0x2e, 0x5d, 0x5d, 0x00,
                 0x24, 0xec, 0x4d, 0xd1, 0xc8, 0xdf, 0xbe, 0x75,
                 0x2a, 0x00, 0x14, 0x50, 0x40, 0x0c, 0x0c, 0x0b,
                 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0xbd
                 ];

    let expected = IPv6Header {
        version: 6,
        traffic_class: 0,
        flow_label: 33176,
        payload_length: 20,
        next_header: 6,
        hop_limit: 64,
        source_addr:
        "2a02:c7d:2e5d:5d00:24ec:4dd1:c8df:be75".parse().unwrap(),
        dest_addr: "2a00:1450:400c:c0b::bd".parse().unwrap(),
    };
    assert_eq!(ipparse(&bytes), IResult::Done(EMPTY_SLICE, expected));
}
```

在这里，我们首先声明一个结构体，用于表示在 RFC 2460 中定义的 IPv6 固定头部（[`tools.ietf.org/html/rfc2460`](https://tools.ietf.org/html/rfc2460)）。我们首先定义一个辅助函数`to_ipv6_address`，它接受一个`u8`切片并将其转换为 IPv6 地址。为此，我们需要另一个辅助函数，该函数将切片转换为固定大小的数组（在这个例子中是`16`）。设置好这些之后，我们定义了多个解析器，使用`named!`宏来解析结构体成员的每个成员。

`parse_version` 函数接收一个字节数组切片并返回一个 `u8` 类型的版本。这是通过从输入中读取 4 位作为 `u8`，使用 `take_bits!` 宏来完成的。然后它被 `bits!` 宏包装，该宏将输入转换为底层解析器的位流。以同样的方式，我们继续定义头部结构中所有其他字段的解析器。对于每一个，我们根据 RFC 中它们占用的位数将它们转换为足够大的类型。解析地址的最后一种情况不同。在这里，我们使用 `take!` 宏读取 16 个字节并将它们映射到 `to_ipv6_address` 函数，使用 `map!` 宏将字节流转换为地址。

到目前为止，所有用于解析整个结构体的小片段都已准备就绪，我们可以使用 `do_parse!` 宏来定义一个函数。在那里，我们在临时变量中累积结果并构建一个 `IPv6Header` 结构体的实例，然后返回。在我们的主函数中，有一个从 IPv6 数据包转储中取出的字节数组，它应该代表一个有效的 IPv6 头部。我们使用定义的解析器来解析它，并断言输出与预期相符。因此，我们解析器的成功运行之前不会抛出异常。

让我们回顾一下到目前为止使用的所有 `nom` 宏：

| **宏** | **目的** |
| --- | --- |
| `named!` | 通过组合较小的函数创建一个解析函数。这（或其变体）总是链中的顶级调用。 |
| `ws!` | 启用解析器消耗标记之间的所有空白字符（`\t`、`\r` 和 `\n`）。 |
| `do_parse!` | 以给定顺序应用子解析器，可以存储中间结果。 |
| `tag!` | 声明一个静态字节序列，封装的解析器应该识别。 |
| `take_until!` | 消耗输入直到给定的标记。 |
| `take_bits!` | 从输入中消耗给定数量的位并将它们转换为给定的类型。 |
| `take!` | 从输入中消耗指定数量的字节。 |
| `map_res!` | 将函数（返回结果）映射到解析器的输出。 |
| `map!` | 将函数映射到解析器的输出。 |
| `bits!` | 将给定的切片转换为位流。 |

# 摘要

在本节中，我们更详细地研究了处理数据的方法。具体来说，是序列化和解析。在撰写本文时，Serde 和相关 crate 是 Rust 社区支持的数据序列化和反序列化方式，而 `nom` 是最常用的解析组合器。这些工具在 nightly 编译器上通常会生成更好的错误信息，并且当开启一些功能标志时，因为它们通常依赖于一些仅夜间的前沿特性。随着时间的推移，这些特性将可用于稳定编译器，并且这些工具将无缝工作。

在下一章中，我们将讨论在套接字上理解传入数据之后的下一步。通常情况下，这涉及到处理应用层协议。
