

# 第十六章：在 TCP 之上构建协议

在上一章中，我们使用了 Tokio 框架来支持异步 actor 模型。我们的 Tokio 框架接受基本流量，并在消息处理完毕后将这些消息发送给 actor。然而，我们的 TCP 处理是基本的。如果你对 TCP 的唯一接触就是这本书，那么你不应该在这个基本的 TCP 流程上构建复杂的系统。在本章中，我们将完全专注于如何在 TCP 连接上打包、发送和读取数据。

在本章中，我们将涵盖以下主题：

+   设置 TCP 客户端和回声服务器

+   使用结构体在 TCP 上处理字节

+   在 TCP 上创建帧以分隔消息

+   在 TCP 之上构建 HTTP 帧

到本章结束时，你将能够使用多种不同的方法打包、发送和读取通过 TCP 发送的数据。你将能够理解如何将你的数据分割成可以处理为结构的帧。最后，你将能够构建一个包含 URL 和方法头的 HTTP 帧，以及包含数据的主体。这将使你能够在发送 TCP 数据时构建所需的数据结构。

# 技术要求

在本章中，我们将纯粹专注于如何在 TCP 连接上处理数据。因此，我们不会依赖任何之前的代码，因为我们正在构建自己的回声服务器。

本章的代码可以在[`github.com/PacktPublishing/Rust-Web-Programming-2nd-Edition/tree/main/chapter16`](https://github.com/PacktPublishing/Rust-Web-Programming-2nd-Edition/tree/main/chapter16)找到。

# 设置我们的 TCP 客户端和服务器

为了探索发送和处理 TCP 上的字节，我们将创建一个基本的回声服务器和客户端。我们将放弃上一章中构建的任何复杂逻辑，因为我们不需要在尝试探索发送、接收和处理字节的想法时被复杂逻辑分散注意力。

在一个新的目录中，我们应该有两个 cargo 项目——一个用于服务器，另一个用于客户端。它们可以采用以下文件结构：

```rs
├── client
│   ├── Cargo.toml
│   └── src
│       └── main.rs
└── server
    ├── Cargo.toml
    └── src
        └── main.rs
```

两个项目都将使用相同的 Tokio 依赖，因此两个项目应该在它们的`Cargo.toml`文件中定义以下依赖：

```rs
[dependencies]
tokio = { version = "1", features = ["full"] }
```

我们现在需要构建回声服务器的基本机制。这是客户端向服务器发送消息的地方。服务器随后处理客户端发送的消息，重新打包消息，并将相同的消息发送回客户端。我们将从构建我们的服务器开始。

## 设置我们的 TCP 服务器

我们可以通过以下代码在`server/src/main.rs`文件中定义服务器：首先导入我们需要的所有内容：

```rs
use tokio::net::TcpListener;
use tokio::io::{BufReader, AsyncBufReadExt, AsyncWriteExt};
```

这样做是为了我们可以监听传入的 TCP 流，从该流量中读取字节，并将其写回发送消息的客户。然后，我们需要利用 Tokio 运行时来监听传入的流量，如果收到消息则启动一个线程。如果你完成了上一章，这是一个很好的机会尝试自己完成这个任务，因为我们已经涵盖了创建监听传入流量的 TCP 服务器的概念。

如果你尝试编写 TCP 服务器的基础知识，你的代码应该看起来像这样：

```rs
#[tokio::main]
async fn main() {
    let addr = "127.0.0.1:8080".to_string();
    let socket = TcpListener::bind(&addr).await.unwrap();
    println!("Listening on: {}", addr);
    while let Ok((mut stream, peer)) =
        socket.accept().await {
        println!("Incoming connection from: {}",
                  peer.to_string());
        tokio::spawn(async move {
            . . .
        });
    }
}
```

在这里，我们应该熟悉以下概念：我们创建一个监听器，将其绑定到地址，然后等待传入的消息，当收到消息时创建一个线程。在线程内部，我们通过以下代码循环处理传入的消息，直到出现带有以下代码的新行：

```rs
println!("thread starting {} starting", peer.to_string());
let (reader, mut writer) = stream.split();
let mut buf_reader = BufReader::new(reader);
let mut buf = vec![];
loop {
    match buf_reader.read_until(b'\n', &mut buf).await {
        Ok(n) => {
            if n == 0 {
                println!("EOF received");
                break;
            }
            let buf_string = String::from_utf8_lossy(&buf);
            writer.write_all(buf_string.as_bytes())
                .await.unwrap();
            buf.clear();
        },
        Err(e) => println!("Error receiving message: {}", e)
    }
}
println!("thread {} finishing", peer.to_string());
```

到现在为止，这段代码应该不会让你感到惊讶。如果你对前面代码中涵盖的任何概念不熟悉，建议你阅读上一章。

现在我们已经定义了一个基本的回声服务器，并且它已经准备好运行，这意味着我们可以将注意力转向在 `client/src/main.rs` 文件中创建我们的客户端代码。我们需要相同的 structs 和 traits 来使客户端工作。然后，我们需要向 TCP 服务器发送一个标准的文本消息。这是一个尝试自己实现客户端的好时机，而且之前已经多次涵盖了构建客户端所需的所有内容。

## 设置我们的 TCP 客户端

如果你尝试自己构建客户端，你应该已经导入了以下 structs 和 traits：

```rs
use tokio::net::TcpStream;
use tokio::io::{BufReader, AsyncBufReadExt, AsyncWriteExt};
use std::error::Error;
```

然后，我们必须建立 TCP 连接，发送一条消息，等待消息被发送回来，然后使用 Tokio 运行时将其打印出来：

```rs
#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    let mut stream =
        TcpStream::connect("127.0.0.1:8080").await?;
    let (reader, mut writer) = stream.split();
    println!("stream starting");
    writer.write_all(b"this is a test\n").await?;
    println!("sent data");
    let mut buf_reader = BufReader::new(reader);
    let mut buf = vec![];
    println!("reading data");
    let _ = buf_reader.read_until(b'\n', &mut
        buf).await.unwrap();
    let message = String::from_utf8_lossy(&buf);
    println!("{}", message);
    Ok(())
}
```

现在我们有一个运行中的客户端和服务器。为了测试客户端和服务器是否工作正常，我们必须在一个终端中启动服务器，然后在另一个终端中运行客户端，通过运行 `cargo run`。服务器将显示以下输出：

```rs
stream starting
sent data
reading data
this is a test
```

我们服务器的输出将如下所示：

```rs
Listening on: 127.0.0.1:8080
Incoming connection from: 127.0.0.1:60545
thread starting 127.0.0.1:60545 starting
EOF received
thread 127.0.0.1:60545 finishing
```

这样，我们就有一个基本的回声服务器和客户端正在工作。现在我们可以专注于打包、解包和处理字节。在下一节中，我们将探讨使用 structs 标准化处理消息的基本方法。

# 使用 structs 处理字节

在上一章中，我们向服务器发送字符串。然而，结果是我们必须将单个值解析成我们需要的类型。由于字符串解析没有很好地结构化，其他开发者不清楚我们的消息结构。我们可以通过定义一个可以发送到 TCP 通道的 struct 来使我们的消息结构更清晰。通过在发送 struct 本身之前将其转换为二进制格式，我们可以实现通过 TCP 通道发送 struct。这也被称为序列化数据。

如果我们要将结构体转换为二进制格式，首先，我们需要利用`serde`和`bincode`包。有了我们新的包，客户端和服务器`Cargo.toml`文件应该包含以下依赖项：

```rs
[dependencies]
serde = { version = "1.0.144", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
bincode = "1.3.3"
```

将会使用`serde`包来序列化结构体，同时使用`bincode`包将我们的消息结构体转换为二进制格式。现在，我们的依赖项已经定义好了，我们可以开始创建消息发送客户端。

## 创建消息发送客户端

我们可以构建`client/src/main.rs`文件以通过 TCP 发送结构体。首先，我们必须导入我们需要的内容：

```rs
. . .
use serde::{Serialize, Deserialize};
use bincode;
```

在准备好导入之后，我们可以使用以下代码定义我们的`Message`结构体：

```rs
#[derive(Serialize, Deserialize, Debug)]
struct Message {
    pub ticker: String,
    pub amount: f32
}
```

我们`Message`结构体的定义形式与我们在 Actix 服务器上处理 HTTP 请求 JSON 主体的结构体类似。然而，这次我们不会使用 Actix Web 的结构体和特性来处理结构体。

我们的`Message`结构体现在可以在`main`函数中使用。记住，在我们的`main`函数中，我们有一个由以下代码创建的 TCP 流：

```rs
let mut stream = TcpStream::connect("127.0.0.1:8080").await?;
let (reader, mut writer) = stream.split();
```

现在我们已经建立了连接，我们可以创建我们的`Message`结构体，并将`Message`结构体转换为二进制格式：

```rs
let message = Message{ticker: String::from("BYND"),
                      amount: 3.2};
let message_bin = bincode::serialize(&message).unwrap();
```

我们的`Message`结构体现在已经是二进制格式。然后，我们必须使用以下代码将我们的消息通过 TCP 流发送出去：

```rs
println!("stream starting");
writer.write_all(&message_bin).await?;
writer.write_all(b"\n").await?;
println!("sent data");
```

注意，我们发送了消息然后换行。这是因为我们的服务器将会读取直到出现新行。如果我们不发送新行，那么程序将会挂起并且永远不会完成。

现在我们已经发送了消息，我们可以等待直到我们再次收到消息。然后，我们必须从二进制格式构造`Message`结构体，并使用以下代码打印出构造的`Message`结构体：

```rs
let mut buf_reader = BufReader::new(reader);
let mut buf = vec![];
println!("reading data");
let _ = buf_reader.read_until(b'\n',
    &mut buf).await.unwrap();
println!("{:?}", bincode::deserialize::<Message>(&buf));
```

我们的客户现在已准备好向服务器发送消息。

## 在服务器中处理消息

当涉及到更新我们的服务器代码时，我们的目标是解包消息，打印消息，然后将消息转换为二进制格式以发送回客户端。在这个阶段，你应该能够自己实现这个更改。这是一个复习我们所覆盖内容的好机会。

如果你尝试首先在`server/src/main.rs`文件中实现服务器上的消息处理，你应该已经导入了以下额外的需求：

```rs
use serde::{Serialize, Deserialize};
use bincode;
```

然后，你应该已经定义了`Message`结构体，如下所示：

```rs
#[derive(Serialize, Deserialize, Debug)]
struct Message {
    pub ticker: String,
    pub amount: f32
}
```

现在，我们只需要处理消息，打印消息，然后将消息返回给客户端。我们可以通过在线程中的循环内管理消息来处理所有这些过程。

```rs
let message =
    bincode::deserialize::<Message>(&buf).unwrap();
println!("{:?}", message);
let message_bin = bincode::serialize(&message).unwrap();
writer.write_all(&message_bin).await.unwrap();
writer.write_all(b"\n").await.unwrap();
buf.clear();
```

这里，我们使用了与客户端相同的方法，但方向相反——也就是说，我们首先将二进制格式转换为结构体，然后在最后将其转换回二进制格式。

如果我们运行服务器然后运行客户端，我们的服务器将给出以下输出：

```rs
Listening on: 127.0.0.1:8080
Incoming connection from: 127.0.0.1:50973
thread starting 127.0.0.1:50973 starting
Message { ticker: "BYND", amount: 3.2 }
EOF received
thread 127.0.0.1:50973 finishing
```

我们的客户端给出了以下输出：

```rs
stream starting
sent data
reading data
Ok(Message { ticker: "BYND", amount: 3.2 })
```

在这里，我们可以看到我们的`Message`结构体可以被发送、接收，然后再次发送，而不会有任何妥协。这给我们的 TCP 流量带来了另一个层次的复杂性，因为我们可以为我们的消息创建更复杂的结构。例如，我们的消息中的一个字段可以是 HashMap，另一个字段可以是另一个结构体的向量，如果该结构体实现了`serde`特性。我们可以根据需要更改`Message`结构体的结构，而无需重写解包和打包消息的协议。其他开发者只需查看我们的`Message`结构体，就可以知道通过 TCP 通道发送了什么。现在我们已经改进了通过 TCP 发送消息的方式，我们可以使用帧结构将流分割成帧。

# 利用帧结构

到目前为止，我们通过 TCP 发送结构体，并用换行符分隔这些消息。本质上，这是帧结构的最基本形式。然而，存在一些缺点。我们必须记住要添加一个分隔符，例如换行符；否则，我们的程序将无限期地挂起。我们还冒着在消息数据中包含分隔符的情况下，过早地将消息分割成两个消息的风险。例如，当我们使用换行符分隔消息时，在消息中包含换行符或任何特殊字符或字节以表示需要将流分割成可序列化的包的情况并非不可想象。为了防止这些问题，我们可以使用 Tokio 提供的内置帧结构支持。

在本节中，我们将重新编写客户端和服务器，因为消息的发送和接收将会改变。如果我们试图将我们的新方法插入到客户端现有的代码中，很容易导致混淆。在我们编写客户端和服务器之前，我们必须更新客户端和服务器中的`Cargo.toml`文件中的依赖项：

```rs
[dependencies]
tokio-util = {version = "0.7.4", features = ["full"] }
serde = { version = "1.0.144", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
futures = "0.3.24"
bincode = "1.3.3"
bytes = "1.2.1"
```

在这里，我们使用了更多的 crate。我们将在本节剩余的代码中介绍它们的需求。为了掌握帧结构，我们将从一个简单的任务开始，即重新编写我们的客户端以支持帧结构。

## 重新编写我们的客户端以支持帧结构

记住，我们将在`client/src/main.rs`文件中编写整个客户端。首先，我们必须使用以下代码从 Tokio 中导入所需的模块：

```rs
use tokio::net::TcpStream;
use tokio_util::codec::{BytesCodec, Decoder};
```

`TcpStream`用于连接到我们的服务器。`BytesCodec`结构体用于通过连接传输原始字节。我们将使用`BytesCodec`结构体来配置分帧。`Decoder`是一个特质，用于解码我们通过连接接受的字节。然而，当涉及到通过连接发送数据时，我们可以传递结构体、字符串或其他任何必须转换为字节的内容。因此，我们必须检查`BytesCodec`结构体的实现，查看`BytesCodec`的源代码。源代码可以通过查看文档或在编辑器中控制点击或悬停于`BytesCodec`结构体上来检查。当我们检查`BytesCodec`结构体的源代码时，我们将看到以下`Encode`实现：

```rs
impl Encoder<Bytes> for BytesCodec {
    . . .
}
impl Encoder<BytesMut> for BytesCodec {
    . . .
}
```

在这里，我们只能通过`BytesCodec`结构体使用`Bytes`或`BytesMut`通过连接发送数据。我们可以为`BytesCodec`实现`Encode`来发送其他类型的数据；然而，对于我们的用例来说，这过于复杂，而且直接通过我们的连接发送`Bytes`更为合理。然而，在我们编写更多代码之前，我们不妨检查`Bytes`的实现，以了解分帧是如何工作的。`Bytes`的`Encode`实现形式如下：

```rs
impl Encoder<Bytes> for BytesCodec {
    type Error = io::Error;
    fn encode(&mut self, data: Bytes, buf: &mut BytesMut)
              -> Result<(), io::Error> {
        buf.reserve(data.len());
        buf.put(data);
        Ok(())
    }
}
```

在这里，我们可以看到正在传递的数据长度被保留在缓冲区中。然后，数据被放入缓冲区。

现在我们已经了解了我们将如何使用分帧来编码和解码我们的消息，我们需要从`futures`和`bytes`包中导入特质，以使我们能够处理我们的消息：

```rs
use futures::sink::SinkExt;
use futures::StreamExt;
use bytes::Bytes;
```

`SinkExt`和`StreamExt`特质基本上使我们能够异步地从流中接收消息。`Bytes`结构体会将我们的序列化消息包装起来以便发送。然后，我们必须导入这些特质以启用消息的序列化，并定义我们的消息结构体：

```rs
use serde::{Serialize, Deserialize};
use bincode;
use std::error::Error;
#[derive(Serialize, Deserialize, Debug)]
struct Message {
    pub ticker: String,
    pub amount: f32
}
```

我们现在拥有开始工作所需的全部内容，来构建我们的运行时。请记住，我们的主要运行时具有以下大纲：

```rs
#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    . . .
    Ok(())
}
```

在我们的运行时内部，我们最初建立一个 TCP 连接，并使用以下代码定义分帧：

```rs
let stream = TcpStream::connect("127.0.0.1:8080").await?;
let mut framed = BytesCodec::new().framed(stream);
```

然后，我们定义我们的消息，序列化消息，并使用以下代码将消息包装在`Bytes`中：

```rs
let message = Message{ticker: String::from("BYND"),
                      amount: 3.2};
let message_bin = bincode::serialize(&message).unwrap();
let sending_message = Bytes::from(message_bin);
```

然后，我们可以发送我们的消息，等待消息被发送回来，然后反序列化消息并使用以下代码将其打印出来：

```rs
framed.send(sending_message).await.unwrap();
let message = framed.next().await.unwrap().unwrap();
let message =
    bincode::deserialize::<Message>(&message).unwrap();
println!("{:?}", message);
```

在所有这些之后，我们的客户端已经构建完成。我们可以看到，我们不必担心换行符或其他任何分隔符。在通过 TCP 发送和接收消息时，我们的代码既干净又直接。现在，我们的客户端已经构建完成，我们可以继续构建服务器，以便它能够处理分帧。

## 重新编写我们的服务器以支持分帧

当谈到构建支持帧处理的服务器时，它与我们在上一节中编写的代码有很多重叠。现在，尝试自己构建服务器是一个好时机。构建服务器需要将我们在上一节中编写的帧处理逻辑实现到现有的服务器代码中。

如果你尝试重写服务器，首先，你应该已经导入了以下结构体和特质：

```rs
use tokio::net::TcpListener;
use tokio_util::codec::{BytesCodec, Decoder};
use futures::StreamExt;
use futures::sink::SinkExt;
use bytes::Bytes;
use serde::{Serialize, Deserialize};
use bincode;
```

注意，我们导入的`Decoder`特质允许我们在字节编解码器上调用`.framed`。这里没有什么应该让你感到陌生的。一旦我们有了必要的导入，我们必须使用以下代码定义相同的`Message`结构体：

```rs
#[derive(Serialize, Deserialize, Debug)]
struct Message {
    pub ticker: String,
    pub amount: f32
}
```

现在，我们必须使用以下代码定义服务器运行时的轮廓：

```rs
#[tokio::main]
async fn main() {
    let addr = "127.0.0.1:8080".to_string();
    let listener = TcpListener::bind(&addr).await.unwrap();
    println!("Listening on: {}", addr);
    loop {
        let (socket, _) = listener.accept().await.unwrap();
        tokio::spawn(async move {
            . . .
        });
    }
}
```

在这里，我们可以看到监听器正在循环以接受流量，并在接收到消息时创建线程，就像之前的服务器实现一样。在我们的线程中，我们使用以下代码读取我们的帧化消息：

```rs
let mut framed = BytesCodec::new().framed(socket);
let message = framed.next().await.unwrap();
    match message {
        Ok(bytes) => {
            . . .
        },
        Err(err) => println!("Socket closed with error:
                              {:?}", err),
    }
println!("Socket received FIN packet and closed
           connection");
```

如我们所见，我们不再有`while`循环。这是因为我们的帧处理管理了消息之间的分割。

一旦我们从连接中提取出我们的字节，我们必须实现我们在客户端中做的相同逻辑，即处理我们的消息，打印它，再次处理它，然后将其发送回客户端：

```rs
let message =
    bincode::deserialize::<Message>(&bytes).unwrap();
println!("{:?}", message);
let message_bin = bincode::serialize(&message).unwrap();
let sending_message = Bytes::from(message_bin);
framed.send(sending_message).await.unwrap();
```

现在我们有一个正在工作的客户端和服务器，它们利用了帧处理。如果我们启动服务器然后运行客户端，客户端将给出以下打印输出：

```rs
Message { ticker: "BYND", amount: 3.2 }
```

我们的服务器将给出以下打印输出：

```rs
Listening on: 127.0.0.1:8080
Message { ticker: "BYND", amount: 3.2 }
Socket received FIN packet and closed connection
```

我们的服务器和客户端现在支持帧处理。我们已经走了很长的路。现在，我们在这个章节中只剩下一个概念需要探索，那就是使用 TCP 构建 HTTP 帧。

# 在 TCP 之上构建 HTTP 帧

在我们探索这本书中的 Tokio 框架之前，我们使用 HTTP 在服务器之间发送和接收数据。HTTP 协议本质上是在 TCP 之上构建的。在本节中，虽然我们将创建一个 HTTP 帧，但我们不会完全模仿 HTTP 协议。相反，为了防止代码过多，我们将创建一个基本的 HTTP 帧来理解创建 HTTP 帧时使用的机制。还必须强调，这只是为了教育目的。TCP 对我们协议来说很好，但如果你想使用 HTTP 处理器，使用现成的 HTTP 处理器（如 Hyper）会更快速、更安全、更不容易出错。我们将在下一章中介绍如何使用 Hyper HTTP 处理器与 Tokio 一起使用。

当涉及到 HTTP 请求时，一个请求通常有一个头部和一个主体。当我们发送请求时，头部将告诉我们正在使用什么方法以及与请求关联的 URL。为了定义我们的 HTTP 帧，我们需要在服务器和客户端上定义相同的帧结构体。因此，我们必须为`client/src/http_frame.rs`和`server/src/http_frame.rs`文件编写相同的代码。首先，我们必须使用以下代码导入需要的序列化特质：

```rs
use serde::{Serialize, Deserialize};
```

然后，我们必须使用以下代码定义我们的 HTTP 帧：

```rs
#[derive(Serialize, Deserialize, Debug)]
pub struct HttpFrame {
    pub header: Header,
    pub body: Body
}
```

正如我们所见，我们在`HttpFrame`结构体中定义了一个头和体。我们使用以下代码定义头和体结构体：

```rs
#[derive(Serialize, Deserialize, Debug)]
pub struct Header {
    pub method: String,
    pub uri: String,
}
#[derive(Serialize, Deserialize, Debug)]
pub struct Body {
    pub ticker: String,
    pub amount: f32,
}
```

我们的基本 HTTP 帧现在已经完成，我们可以使用以下代码将 HTTP 帧导入客户端和服务器中的`main.rs`文件：

```rs
mod http_frame;
use http_frame::{HttpFrame, Header, Body};
```

我们将首先在客户端的`main.rs`文件中发送我们的 HTTP 帧，以下代码：

```rs
let stream = TcpStream::connect("127.0.0.1:8080").await?;
let mut framed = BytesCodec::new().framed(stream);
let message = HttpFrame{
    header: Header{
        method: "POST".to_string(),
        uri: "www.freshcutswags.com/stock/purchase".to_string()
    },
    body: Body{
        ticker: "BYND".to_string(),
        amount: 3.2,
    }
};
let message_bin = bincode::serialize(&message).unwrap();
let sending_message = Bytes::from(message_bin);
framed.send(sending_message).await.unwrap();
```

我们可以看到我们的 HTTP 帧开始看起来像我们在 Actix 服务器接收请求时将要处理的 HTTP 请求。对于我们的服务器中的`main.rs`文件，几乎没有变化。我们只需要重新定义以下代码中正在反序列化的结构体：

```rs
let message = bincode::deserialize::<HttpFrame>(&bytes).unwrap();
println!("{:?}", message);
let message_bin = bincode::serialize(&message).unwrap();
let sending_message = Bytes::from(message_bin);
framed.send(sending_message).await.unwrap();
```

如果我们运行我们的服务器和客户端程序，我们会得到以下服务器输出：

```rs
Listening on: 127.0.0.1:8080
HttpFrame { header: Header {
                method: "POST",
                uri: "www.freshcutswags.com/stock/purchase"
            },
            body: Body {
                ticker: "BYND",
                amount: 3.2
            }
        }
Socket received FIN packet and closed connection
```

我们的客户端程序将给出以下输出：

```rs
HttpFrame { header: Header {
                method: "POST",
                uri: "www.freshcutswags.com/stock/purchase"
            },
            body: Body {
                ticker: "BYND",
                amount: 3.2
            }
        }
```

我们可以看到我们的数据没有损坏。我们现在已经涵盖了所有核心必要的方法和技巧，以便在 TCP 上打包、发送和读取数据时更加灵活。

# 摘要

在本章中，我们构建了一个基本的 TCP 客户端，向回声服务器发送和接收数据。我们首先发送基本的字符串数据，并用分隔符分隔消息。然后，我们通过序列化结构体增加了通过 TCP 连接发送的数据的复杂性。这使得我们能够拥有更复杂的数据结构。这种序列化还减少了获取所需格式的消息数据所需的处理。例如，在前一章中，我们在收到消息后会将字符串解析为浮点数。有了结构体，我们没有任何阻止我们将浮点数列表作为字段的理由，在消息序列化后，该字段将包含一个浮点数列表，而不需要任何额外的代码行。

结构体的序列化对于我们处理大多数问题已经足够，但我们探索了分帧技术，这样我们就不必依赖分隔符来区分我们通过 TCP 发送的消息。有了分帧，我们构建了一个基本的 HTTP 帧来可视化我们可以用帧做什么，以及 HTTP 是如何建立在 TCP 之上的。我们必须记住，实现 HTTP 协议比我们在本章中所做的工作要复杂得多，建议我们利用 crate 中已有的 HTTP 处理器来处理和 HTTP 流量。

在下一章中，我们将使用已建立的 Hyper crate 和 Tokio 运行时框架来处理 HTTP 流量。

# 进一步阅读

Tokio 分帧文档：[`tokio.rs/tokio/tutorial/framing`](https://tokio.rs/tokio/tutorial/framing)。

# 问题

1.  使用分帧与使用分隔符相比有什么优势？

1.  为什么我们将序列化的消息包装在`Bytes`结构体中？

1.  我们如何能够发送一个字符串作为帧？

# 答案

1.  如果我们使用换行符等分隔符，通过 TCP 发送的数据中可能包含消息中的换行符。消息中存在换行符的问题意味着在接收到消息的末尾之前，消息已经被分割。帧结构解决了这个问题。

1.  我们不得不将序列化的消息包装进一个 `Bytes` 结构体中，因为 `Encode` 特性并未为任何其他数据类型实现。

1.  实现字符串的 `Encode` 特性是最简单的方法。在实现 `Encode` 特性时，我们序列化字符串，然后将字符串包装进一个 `Bytes` 结构体中，在缓冲区中预留序列化字符串的长度，然后将序列化后的字符串放入缓冲区。
