存储和检索数据

任何软件应用的典型需求是通过读取/写入数据文件或数据流或通过查询/操作数据库来输入/输出数据。至于文件和流，非结构化数据，甚至二进制数据，很难操作，因此不建议使用。

此外，由于供应商锁定风险，不建议使用专有数据格式，因此应仅使用标准数据格式。幸运的是，在这些情况下，有免费的 Rust 库可以提供帮助。有 Rust crate 可以操作一些最受欢迎的文件格式，例如 TOML、JSON 和 XML。

在数据库方面，有 Rust crate 可以用来操作一些最受欢迎的数据库，例如 SQLite、PostgreSQL 和 Redis。

在本章中，你将学习以下内容：

+   如何从 TOML 文件中读取配置数据

+   如何读取或写入 JSON 数据文件

+   如何读取 XML 数据文件

+   如何查询或操作 SQLite 数据库中的数据

+   如何查询或操作 PostgreSQL 数据库中的数据

+   如何查询或操作 Redis 数据库中的数据

# 技术要求

当你运行 SQLite 代码时，需要安装 SQLite 运行时库。然而，安装 SQLite 交互式管理器也是有用的（尽管不是必需的）。你可以从[`www.sqlite.org/download.html`](https://www.sqlite.org/download.html)下载 SQLite 工具的预编译二进制文件。然而，版本 3.11 或更高版本是理想的。

请注意，如果你使用的是基于 Debian 的 Linux 发行版，应该安装`libsqlite3-dev`包。

当你运行 PostgreSQL 代码时，还需要安装和运行 PostgreSQL **数据库管理系统**（**DBMS**）。与 SQLite 类似，安装 PostgreSQL 交互式管理器是有用的，但不是必需的。你可以从[`www.postgresql.org/download/`](https://www.postgresql.org/download/)下载 PostgreSQL DBMS 的预编译二进制文件。然而，版本 7.4 或更高版本是可以接受的。

当你运行 Redis 代码时，安装和运行 Redis 服务器是必要的。你可以从[`redis.io/download`](https://redis.io/download)下载它。

本章的完整源代码可以在[`github.com/PacktPublishing/Creative-Projects-for-Rust-Programmers`](https://github.com/PacktPublishing/Creative-Projects-for-Rust-Programmers)存储库的`Chapter02`文件夹中找到。在这个文件夹中，每个项目都有一个子文件夹，还有一个名为`data`的文件夹，其中包含我们将用作项目输入的数据。

# 项目概述

在本章中，我们将探讨如何构建一个程序，该程序将 JSON 文件和 XML 文件加载到三个数据库中：SQLite 数据库、PostgreSQL 数据库和 Redis 键值存储。为了避免将文件名和位置以及数据库凭据硬编码到程序中，我们将从 TOML 配置文件中加载它们。

最后一个项目命名为 `transformer`，但我们将通过几个初步的小项目来解释这一点：

+   `toml_dynamic` 和 `toml_static`: 这两种方式以不同的方式读取一个 TOML 文件。

+   `json_dynamic` 和 `json_static`: 这两种方式以不同的方式读取一个 JSON 文件。

+   `xml_example`: 这将读取一个 XML 文件。

+   `sqlite_example`: 这将在 SQLite 数据库中创建两个表，向它们插入记录并查询它们。

+   `postgresql_example`: 这将在 PostgreSQL 数据库中创建两个表，向它们插入记录并查询它们。

+   `redis_example`: 这向一个键值存储添加一些数据并查询它。

# 读取一个 TOML 文件

在文件系统中存储信息的一个简单且易于维护的方法是使用文本文件。这对于数据量不超过 100 KB 的情况也非常高效。然而，在文本文件中存储信息存在几个相互竞争的标准，如 INI、CSV、JSON、XML、YAML 等。

Cargo 使用的格式是 TOML。这是一个非常强大的格式，被许多 Rust 开发者用来存储他们应用程序的配置数据。它设计为手动编写，使用文本编辑器，但它也可以很容易地由应用程序编写。

`toml_dynamic` 和 `toml_static` 项目（使用 `toml` 包）从 TOML 文件加载数据。在配置软件应用程序时读取 TOML 文件非常有用，这正是我们将要做的。我们将使用 `data/config.toml` 文件，它包含本章所有项目的所有参数。

您也可以通过使用代码来创建或修改一个 TOML 文件，但我们不会这么做。在某些场景下，能够修改 TOML 文件可能很有用，例如保存用户偏好设置。

重要的是要考虑，当 TOML 文件被程序修改时，它会经历重大的重构：

+   它获得了特定的格式，你可能不喜欢。

+   它失去了所有的注释。

+   它的项目按字母顺序排序。

因此，如果您想同时使用 TOML 格式手动编辑参数和程序保存的数据，您最好使用两个不同的文件：

+   一个仅由人类编辑的

+   主要由你的软件编辑，但偶尔也由人类编辑

本章描述了两个项目，在这些项目中使用不同的技术读取 TOML 文件。这些技术将在两种不同的情况下使用：

+   在我们不确定文件中包含哪些字段的情况下，我们想要探索它。在这种情况下，我们使用 `toml_dynamic` 程序。

+   在另一种情况下，在我们的程序中，我们精确地描述了文件中应包含哪些字段，并且不接受不同的格式。在这种情况下，我们使用`toml_static`程序。

## 使用 toml_dynamic

本节的目的在于在我们想要探索该文件内容时读取位于`data`文件夹中的`config.toml`文件。该文件的最初三行如下：

```rs
[input]
xml_file = "../data/sales.xml"
json_file = "../data/sales.json"
```

在这些行之后，文件包含其他部分。其中之一是`[postgresql]`部分，它包含以下行：

```rs
database = "Rust2018"
```

要运行此项目，请进入`toml_dynamic`文件夹，并输入`cargo run ../data/config.toml`。应该打印出长输出。它将从以下行开始：

```rs
Original: Table(
    {
        "input": Table(
            {
                "json_file": String(
                    "../data/sales.json",
                ),
                "xml_file": String(
                    "../data/sales.xml",
                ),
            },
        ),
```

注意，这仅仅是`config.toml`文件前三行的详细表示。此输出随后继续发出文件其余部分的类似表示。在打印出表示读取的文件的整个数据结构之后，输出中添加了以下行：

```rs
 [Postgresql].Database: Rust2018
```

这是读取文件时加载数据结构的具体查询结果。

让我们看看`toml_dynamic`程序的代码：

1.  声明一个变量，该变量将包含整个文件的描述。此变量在接下来的三个语句中初始化：

```rs
let config_const_values =
```

1.  我们将命令行中的第一个参数中的文件路径添加到`config_path`中。然后，我们将此文件的 内容加载到`config_text`字符串中，并将此字符串解析为`toml::Value`结构。这是一个递归结构，因为它可以在其字段中具有`Value`属性：

```rs
{
    let config_path = std::env::args().nth(1).unwrap();
    let config_text = 
     std::fs::read_to_string(&config_path).unwrap();
    config_text.parse::<toml::Value>().unwrap()
};
```

1.  然后使用调试结构化格式（`:#?`）打印此结构，并从中检索一个值：

```rs
println!("Original: {:#?}", config_const_values);
println!("[Postgresql].Database: {}",
    config_const_values.get("postgresql").unwrap()
    .get("database").unwrap()
    .as_str().unwrap());
```

注意，要获取包含在`"postgresql"`部分中的`"database"`项的值，需要编写大量的代码。`get`函数需要查找一个字符串，这可能会失败。这就是不确定性的代价。

## 使用 toml_static

另一方面，如果我们对 TOML 文件的组织结构非常有信心，我们应该使用项目中展示的另一种技术，即`toml_static`。

要运行它，请打开`toml_static`文件夹，并输入`cargo run ../data/config.toml`。程序将仅打印以下行：

```rs
[postgresql].database: Rust2018
```

此项目使用两个额外的 crate：

+   `serde`: 这启用了基本序列化/反序列化操作。

+   `serde_derive`: 这提供了一个名为**custom-derive**的强大附加功能，允许您使用结构体进行序列化/反序列化。

`serde`是标准的序列化/反序列化库。**序列化**是将程序中的数据结构转换为字符串（或流）的过程。**反序列化**是相反的过程；它是将字符串（或流）转换为程序中的某些数据结构的过程。

要读取 TOML 文件，我们需要使用反序列化。

在这两个项目中，我们不需要使用序列化，因为我们不打算编写 TOML 文件。

在代码中，首先，为`data/config.toml`文件中包含的任何部分定义了一个结构。该文件包含`Input`、`Redis`、`Sqlite`和`Postgresql`部分，因此我们声明了与我们要读取的文件部分一样多的 Rust 结构；然后，定义了`Config`结构来表示整个文件，这些部分作为其成员。

例如，这是`Input`部分的架构：

```rs
#[allow(unused)]
#[derive(Deserialize)]
struct Input {
    xml_file: String,
    json_file: String,
}
```

注意，前面的声明前面有两个属性。

使用`allow(unused)`属性来防止编译器警告我们关于以下结构中未使用的字段。这对于我们避免这些嘈杂的警告很方便。使用`derive(Deserialize)`属性来激活`serde`对以下结构的自动反序列化。

在这些声明之后，可以编写以下代码行：

```rs
toml::from_str(&config_text).unwrap()
```

这将调用`from_str`函数，该函数将文件的文本解析为结构。此表达式中未指定该结构的类型，但其值被分配给`main`函数第一行中声明的变量：

```rs
 let config_const_values: Config =
```

因此，它的类型是`Config`。

如果文件内容与结构类型之间存在任何差异，将在此操作中视为错误。因此，如果此操作成功，对结构的任何其他操作都不会失败。

虽然前面的程序（`toml_dynamic`）具有类似于 Python 或 JavaScript 的动态类型，但此程序具有类似于 Rust 或 C++的静态类型。

静态类型的优势体现在最后一条语句中，通过简单地编写`config_const_values.postgresql.database`，就获得了与先前项目长语句相同的行为。

# 读取和写入 JSON 文件

对于存储比配置文件中存储的数据更复杂的数据，JSON 格式更为合适。这种格式相当流行，尤其是在使用 JavaScript 语言的人中。

我们将读取并解析`data/sales.json`文件。此文件包含一个单个匿名对象，该对象包含两个数组——`"products"`和`"sales"`。

`"products"`数组包含两个对象，每个对象都有三个字段：

```rs
  "products": [
    {
      "id": 591,
      "category": "fruit",
      "name": "orange"
    },
    {
      "id": 190,
      "category": "furniture",
      "name": "chair"
    }
  ],
```

`"sales"`数组包含三个对象，每个对象包含五个字段：

```rs
"sales": [
    {
      "id": "2020-7110",
      "product_id": 190,
      "date": 1234527890,
      "quantity": 2.0,
      "unit": "u."
    },
    {
      "id": "2020-2871",
      "product_id": 591,
      "date": 1234567590,
      "quantity": 2.14,
      "unit": "Kg"
    },
    {
      "id": "2020-2583",
      "product_id": 190,
      "date": 1234563890,
      "quantity": 4.0,
      "unit": "u."
    }
  ]
```

数组中的信息涉及一些要出售的产品以及与这些产品相关的销售交易。请注意，每个销售的第二字段（`"product_id"`）是对一个产品的引用，因此它应该在相应的产品对象创建后进行处理。

我们将看到具有相同行为的两个程序。它们读取 JSON 文件，将第二个销售对象的数量增加`1.5`，然后将整个更新后的结构保存到另一个 JSON 文件中。

类似于 TOML 格式的情况，对于 JSON 文件也可以使用动态解析技术，其中任何数据字段的存在的类型由应用程序代码检查，以及静态解析技术，其中它使用反序列化库来检查任何字段的存在的类型。

因此，我们有两个项目：`json_dynamic` 和 `json_static`。要运行每个项目，打开其文件夹，并输入 `cargo run ../data/sales.json ../data/sales2.json`。程序将不会打印任何内容，但它将读取命令行中指定的第一个文件，并创建指定的第二个文件。

创建的文件与读取的文件相似，但有以下不同之处：

+   由 `json_dynamic` 创建的文件的字段按字母顺序排序，而由 `json_static` 创建的文件的字段按 Rust 数据结构的相同顺序排序。

+   第二次销售的量从 `2.14` 增加到 `3.64`。

+   在两个创建的文件中，最后的空行都被移除了。

现在，我们可以看到序列化和反序列化的两种技术的实现。

## `json_dynamic` 项目

让我们看看这个项目的源代码：

1.  此项目从命令行获取两个文件的路径名——现有的 JSON 文件（`"input_path"`)，将其读入内存结构中，以及一个要创建的 JSON 文件（`"output_path"`)，通过保存修改后的结构来创建。 

1.  然后，将输入文件加载到名为 `sales_and_products_text` 的字符串中，并使用 `serde_json::from_str::<Value>` 函数将字符串解析为表示 JSON 文件的动态类型结构。此结构存储在 `sales_and_products` 本地变量中。

假设我们想要改变第二个销售交易的销量，增加 `1.5` 公斤：

1.  首先，我们必须使用以下表达式获取此值：

```rs
sales_and_products["sales"][1]["quantity"]
```

1.  这检索了通用对象的 `"sales"` 子对象。它是一个包含三个对象的数组。

1.  然后，这个表达式获取这个数组的第二个项目（从零开始计数 `[1]`）。这是一个表示单个销售交易的对象。

1.  然后，它获取销售交易对象的 `"quantity"` 子对象。

1.  我们达到的值具有动态类型，我们认为应该是 `serde_json::Value::Number`，因此我们与该类型进行模式匹配，指定 `if let Value::Number(n)` 子句。

1.  如果一切顺利，匹配成功，我们将得到一个名为 `n` 的变量——包含一个数字，或者可以通过使用 `as_f64` 函数将其转换为 Rust 浮点数的东西。最后，我们可以增加 Rust 数字，然后使用 `from_f64` 函数从它创建一个 JSON 数字。然后我们可以使用相同的表达式将此对象分配给 JSON 结构：

```rs
sales_and_products["sales"][1]["quantity"]
    = Value::Number(Number::from_f64(
        n.as_f64().unwrap() + 1.5).unwrap());
```

1.  程序的最后一条语句将 JSON 结构保存到文件中。在这里，使用了 `serde_json::to_string_pretty` 函数。正如其名所示，这个函数添加了格式化空白（空格和新行），使得生成的 JSON 文件更易于人类阅读。还有一个 `serde_json::to_string` 函数，它创建了一个更紧凑的相同信息版本。对于人类来说，阅读起来更困难，但对于计算机来说处理速度更快：

```rs
std::fs::write(
    output_path,
    serde_json::to_string_pretty(&sales_and_products).unwrap(),
).unwrap();
```

## `json_static` 项目

如果我们确信我们的程序知道 JSON 文件的结构，可以使用静态类型技术，这是在 `json_static` 项目中展示的。这里的情形与处理 TOML 文件的项目类似。

静态版本源代码首先声明了三个结构体——每个结构体对应于我们将要处理的 JSON 文件中包含的对象类型。每个结构体前面都有以下属性：

```rs
#[derive(Deserialize, Serialize, Debug)]
```

让我们理解前面的代码片段：

+   `Deserialize` 特性是必需的，用于将 JSON 字符串解析（即读取）到这个结构体中。

+   `Serialize` 特性是必需的，用于将这个结构体格式化（即写入）为 JSON 字符串。

+   `Debug` 特性非常适合在调试跟踪中打印这个结构体。

使用 `serde_json::from_str::<SalesAndProducts>` 函数解析 JSON 字符串。然后，增加已售橙子的数量的代码变得相当简单：

```rs
sales_and_products.sales[1].quantity += 1.5
```

程序的其余部分保持不变。

# 读取 XML 文件

另一个非常流行的文本格式是 XML。不幸的是，没有稳定的序列化/反序列化库来管理 XML 格式。然而，这并不一定是缺点。实际上，XML 格式通常用于存储大型数据集；实际上非常大，以至于在我们开始将数据转换为内部格式之前，加载所有这些数据可能是不高效的。在这些情况下，扫描文件或传入的流并读取时进行处理可能更有效。

`xml_example` 项目是一个相当复杂的程序，它扫描命令行上指定的 XML 文件，并以过程式的方式将文件中的信息加载到 Rust 数据结构中。它的目的是读取 `../data/sales.xml` 文件。这个文件的结构与我们在上一节中寻找的 JSON 文件相对应。以下是一些该文件的摘录：

```rs
<?xml version="1.0" encoding="utf-8"?>
<sales-and-products>
    <product>
        <id>862</id>
    </product>
    <sale>
        <id>2020-3987</id>
    </sale>
</sales-and-products>
```

所有 XML 文件的第一行都有一个头部，然后是一个根元素；在这个例子中，根元素被命名为 `sales-and-products`。这个元素包含两种类型的元素——`product` 和 `sale`。这两种类型的元素都有特定的子元素，它们是对应数据的字段。在这个例子中，只显示了 `id` 字段。

要运行项目，打开其文件夹，输入 `cargo run ../data/sales.xml`。控制台将打印出一些行。前四行应该是这样的：

```rs
Got product.id: 862.
Got product.category: fruit.
Got product.name: cherry.
  Exit product: Product { id: 862, category: "fruit", name: "cherry" }
```

这些描述了指定 XML 文件的内容。特别是，程序找到了一个 ID 为`862`的产品，然后检测到它是一种水果，然后是樱桃，最后，当整个产品被读取完毕后，整个表示产品的结构被打印出来。对于销售也会有类似的输出。

解析仅使用`xml-rs` crate 执行。这个 crate 提供了一个解析机制，如下面的代码片段所示：

```rs
let file = std::fs::File::open(pathname).unwrap();
let file = std::io::BufReader::new(file);
let parser = EventReader::new(file);
for event in parser {
    match &location_item {
        LocationItem::Other => ...
        LocationItem::InProduct => ...
        LocationItem::InSale => ...
    }
}
```

`EventReader`类型的对象扫描缓冲文件，并在解析过程中执行每一步时生成一个事件。应用程序代码根据其需求处理这些类型的事件。

这个 crate 使用**事件**这个词，但**转换**这个词可能更准确地描述了解析器提取的数据。

一个复杂的语言难以解析，但对我们这样的简单数据语言，解析过程中的情况可以通过状态机来建模。为此，源代码中声明了三个`enum`变量：`location_item`，类型为`LocationItem`；`location_product`，类型为`LocationProduct`；以及`location_sale`，类型为`LocationSale`。

第一个表示解析的一般位置。我们可能处于一个产品内部（`InProduct`），处于一个销售内部（`InSale`），或者处于两者之外（`Other`）。如果我们处于产品内部，`LocationProduct`枚举表示当前产品内部解析的位置。这可以是在任何允许的字段内，或者在外部所有字段之外。对于销售也会有类似的状态。

迭代遇到几种类型的事件。主要的有以下几种：

+   `XmlEvent::StartElement`：表示一个 XML 元素开始。它被开始元素的名称和该元素的可能的属性所装饰。

+   `XmlEvent::EndElement`：表示一个 XML 元素结束。它被结束元素的名称所装饰。

+   `XmlEvent::Characters`：表示一个元素的文本内容可用。它被该可用文本所装饰。

程序声明了一个可变的`product`结构体，类型为`Product`，和一个可变的`sale`结构体，类型为`Sale`。它们使用默认值初始化。每当有字符可用时，它们被存储在当前结构体的相应字段中。

例如，考虑一个`location_item`的值为`LocationItem::InProduct`而`location_product`的值为`LocationProduct::InCategory`的情况——即，我们处于一个产品的类别中。在这种情况下，可能会有类别的名称或类别的结束。要获取类别的名称，代码中包含了一个`match`语句的模式：

```rs
Ok(XmlEvent::Characters(characters)) => {
    product.category = characters.clone();
    println!("Got product.category: {}.", characters);
}
```

在这个语句中，`characters`变量获取类别的名称，并将其克隆赋值给`product.category`字段。然后，名称被打印到控制台。

# 访问数据库

当文本文件很小且不需要经常更改时，文本文件是好的。实际上，更改文本文件的唯一方法是在其末尾追加内容或完全重写它。如果您想快速更改大型数据集中的信息，唯一的方法是使用数据库管理器。在本节中，我们将通过一个简单的示例学习如何操作 SQLite 数据库。

但首先，让我们看看三种流行的、广泛的数据库管理器类别：

+   **单用户数据库**：这些数据库将所有数据库存储在一个单独的文件中，该文件必须可通过应用程序代码访问。数据库代码被链接到应用程序中（它可能是一个静态链接库或动态链接库）。一次只允许一个用户访问它，并且所有用户都具有管理权限。要将数据库移动到任何地方，只需移动文件即可。在这个类别中最受欢迎的选择是 SQLite 和 Microsoft Access。

+   **数据库管理系统（DBMS）**：这是一个必须作为服务启动的过程。多个客户端可以同时连接到它，并且它们可以同时应用更改而不会导致数据损坏。它需要更多的存储空间、更多的内存以及更长的启动时间（对于服务器）。在这个类别中有几个流行的选择，例如 Oracle、Microsoft SQL Server、IBM DB2、MySQL 和 PostgreSQL。

+   **键值存储**：这是一个必须作为服务启动的过程。多个客户端可以同时连接到它，并且可以同时应用更改。它本质上是一个大型的内存哈希表，可以被其他进程查询，并且可以选择将其数据存储在文件中，并在重启时重新加载。这个类别比其他两个类别不太受欢迎，但随着高性能网站后端的需求增加，它正在获得更多的关注。最受欢迎的选择之一是 Redis。

在接下来的几节中，我们将向您展示如何访问 SQLite 单用户数据库（在`sqlite_example`项目中）、PostgreSQL 数据库管理系统（在`postgreSQL_example`项目中）和 Redis 键值存储（在`redis_example`项目中）。然后，在`transformer`项目中，将使用这三种类型的数据库。

# 访问 SQLite 数据库

本节的源代码位于`sqlite_example`项目中。要运行它，打开其文件夹并输入`cargo run`。

这将在当前文件夹中创建`sales.db`文件。此文件包含一个 SQLite 数据库。然后，它将在该数据库中创建`Products`和`Sales`表，并在每个表中插入一行，然后对数据库执行查询。查询要求获取所有销售信息，并将每个销售与其相关联的产品结合起来。对于每个提取的行，将在控制台上打印一行，显示销售的日期时间、销售重量和关联产品的名称。由于数据库中只有一个销售记录，您将看到以下行被打印出来：

```rs
At instant 1234567890, 7.439 Kg of pears were sold. 
```

此项目仅使用 `rusqlite` 包。其名称是 **Rust SQLite** 的缩写。要使用此包，`Cargo.toml` 文件必须包含以下行：

```rs
rusqlite = "0.23"
```

## 实现项目

让我们看看 `sqlite_example` 项目的代码是如何工作的。`main` 函数相当简单：

```rs
fn main() -> Result<()> {
    let conn = create_db()?;
    populate_db(&conn)?;
    print_db(&conn)?;
    Ok(())
}
```

它调用 `create_db` 来打开或创建一个包含空表的数据库，并打开并返回到此数据库的连接。

然后，它调用 `populate_db` 来将行插入由该连接引用的数据库的表中。

然后，它调用 `print_db` 来执行对这个数据库的查询，并打印出该查询提取的数据。

`create_db` 函数虽然长，但很容易理解：

```rs
fn create_db() -> Result<Connection> {
    let database_file = "sales.db";
    let conn = Connection::open(database_file)?;
    let _ = conn.execute("DROP TABLE Sales", params![]);
    let _ = conn.execute("DROP TABLE Products", params![]);
    conn.execute(
        "CREATE TABLE Products (
            id INTEGER PRIMARY KEY,
            category TEXT NOT NULL,
            name TEXT NOT NULL UNIQUE)",
        params![],
    )?;
    conn.execute(
        "CREATE TABLE Sales (
            id TEXT PRIMARY KEY,
            product_id INTEGER NOT NULL REFERENCES Products,
            sale_date BIGINT NOT NULL,
            quantity DOUBLE PRECISION NOT NULL,
            unit TEXT NOT NULL)",
        params![],
    )?;
    Ok(conn)
}
```

`Connection::open` 函数简单地使用一个指向 SQLite 数据库文件的路径来打开一个连接。如果此文件不存在，它将被创建。正如你所见，创建的 `sales.db` 文件非常小。通常，DBMS 的空数据库比这大 1,000 倍。

要执行数据操作命令，调用连接的 `execute` 方法。它的第一个参数是一个 SQL 语句，可能包含一些参数，指定为 `$1`、`$2`、`$3` 等。函数的第二个参数是一个值切片的引用，用于替换这些参数。

当然，如果没有参数，参数值列表必须为空。第一个参数值（索引为 `0`）替换 `$1` 参数，第二个替换 `$2` 参数，依此类推。

注意，参数化 SQL 语句的参数可以是不同的数据类型（数值、字母数字、BLOBs 等），但 Rust 集合只能包含相同数据类型的对象。因此，使用 `params!` 宏来执行一些魔法操作。`execute` 方法的第二个参数的数据类型必须是可以迭代的集合，其项目实现了 `ToSql` 特性。实现此特性的对象，正如其名所示，可以用作 SQL 语句的参数。`rusqlite` 包含了许多 Rust 基本类型（如数字和字符串）的此特性的实现。

例如，`params!(34, "abc")` 表达式生成一个可以迭代的集合。这个迭代的第一个项目可以转换成一个包含数字 `34` 的对象，这个数字可以用来替换一个数值类型的 SQL 参数。第二个项目可以转换成一个包含 `"abc"` 字符串的对象，这个字符串可以用来替换一个字母数字类型的 SQL 参数。

现在，让我们看看 `populate_db` 函数。它包含将行插入数据库的语句。以下是其中之一：

```rs
conn.execute(
    "INSERT INTO Products (
        id, category, name
        ) VALUES ($1, $2, $3)",
    params![1, "fruit", "pears"],
)?;
```

如前所述，此语句将执行以下 SQL 语句：

```rs
INSERT INTO Products (
        id, category, name
        ) VALUES (1, 'fruit', 'pears')
```

最后，我们看到整个 `print_db` 函数，它比其他函数更复杂：

```rs
fn print_db(conn: &Connection) -> Result<()> {
    let mut command = conn.prepare(
        "SELECT p.name, s.unit, s.quantity, s.sale_date
        FROM Sales s
        LEFT JOIN Products p
        ON p.id = s.product_id
        ORDER BY s.sale_date",
    )?;
    for sale_with_product in command.query_map(params![], |row| {
        Ok(SaleWithProduct {
            category: "".to_string(),
            name: row.get(0)?,
            quantity: row.get(2)?,
            unit: row.get(1)?,
            date: row.get(3)?,
        })
    })? {
        if let Ok(item) = sale_with_product {
            println!(
                "At instant {}, {} {} of {} were sold.",
                item.date, item.quantity, item.unit, item.name
            );
        }
    }
    Ok(())
}
```

要执行 SQL 查询，首先必须通过调用连接的 `prepare` 方法来准备 `SELECT` SQL 语句，将其转换为高效的内部格式，使用 `Statement` 数据类型。此对象被分配给 `command` 变量。准备好的语句必须是可变的，以便允许以下参数的替换。然而，在这种情况下，我们没有任何参数。

一个查询可以生成多行，我们希望逐行处理，因此必须从这个命令创建一个迭代器。这是通过调用命令的 `query_map` 方法来完成的。此方法接收两个参数——一个参数值切片和一个闭包——并返回一个迭代器。`query_map` 函数执行两个任务——首先，它替换指定的参数，然后使用闭包将提取的每一行映射（或转换）为更方便的结构。但在我们的情况下，我们没有要替换的参数，所以我们只使用 `SaleWithProduct` 类型创建一个特定的结构。要从行中提取字段，使用 `get` 方法。它具有在 `SELECT` 查询中指定的字段上的零基索引。此结构是迭代器返回的对象，用于查询提取的任何行，并将其分配给名为 `sale_with_product` 的迭代变量。

现在我们已经学习了如何访问 SQLite 数据库，让我们来检查 PostgreSQL 数据库管理系统。

# 访问 PostgreSQL 数据库

我们在 SQLite 数据库中做的事情与我们将要在 PostgreSQL 数据库中做的事情相似。这是因为它们都基于 SQL 语言，但主要是因为 SQLite 被设计成与 PostgreSQL 类似。将应用程序从 PostgreSQL 转换为 SQLite 可能会更困难，因为前者有许多后者没有的先进功能。

在本节中，我们将转换上一节的示例，使其与 PostgreSQL 数据库而不是 SQLite 一起工作。因此，我们将解释其中的差异。

本节源代码位于 `postgresql_example` 文件夹中。要运行它，打开其文件夹并输入 `cargo run`。这将执行与 `sqlite_example` 中看到的基本相同的操作，因此创建并填充数据库后，将打印以下内容：

```rs
At instant 1234567890, 7.439 Kg of pears were sold.
```

## 项目的实现

此项目仅使用名为 `postgres` 的 crate。其名称是 `postgresql` 名称的流行缩写。

创建到 PostgreSQL 数据库的连接与创建到 SQLite 数据库的连接非常不同。因为后者只是一个文件，所以你以类似打开文件的方式执行，你应该写`Connection::open(<db 文件路径>)`。相反，要连接到 PostgreSQL 数据库，你需要访问一个运行服务器的计算机，然后访问该服务器监听的 TCP 端口，然后你需要在此服务器上指定你的凭据（你的用户名和密码）。可选地，然后你可以指定你想要使用此服务器管理的哪个数据库。

因此，调用的一般形式是`Connection::connect(<URL>, <TlsMode>)`，其中 URL 可以是例如`postgres://postgres:post@localhost:5432/Rust2018`。URL 的一般形式是`postgres://username[:password]@host[:port][/database]`，其中密码、端口和数据库部分是可选的。`TlsMode`参数指定连接是否必须加密。

端口是可选的，因为它默认值为 `5432`。另一个区别是，这个 crate 不使用`params!`宏。相反，它允许我们指定一个切片的引用。在这种情况下，它是一个空切片（`&[]`），因为我们不需要指定参数。

表创建和填充过程与`sqlite_example`中执行的方式相似。然而，查询是不同的。这是`print_db`函数的主体：

```rs
for row in &conn.query(
    "SELECT p.name, s.unit, s.quantity, s.sale_date
    FROM Sales s
    LEFT JOIN Products p
    ON p.id = s.product_id
    ORDER BY s.sale_date",
    &[],
)? {
    let sale_with_product = SaleWithProduct {
        category: "".to_string(),
        name: row.get(0),
        quantity: row.get(2),
        unit: row.get(1),
        date: row.get(3),
    };
    println!(
        "At instant {}, {} {} of {} were sold.",
        sale_with_product.date,
        sale_with_product.quantity,
        sale_with_product.unit,
        sale_with_product.name
    );
}
```

在 PostgreSQL 中，连接类的`query`方法执行参数替换，类似于`execute`方法，但它不将行映射到结构中。相反，它返回一个迭代器，可以立即在`for`语句中使用。然后，在循环体中，可以使用`row`变量（如示例中所示）来填充一个结构体。

既然我们已经知道如何访问 SQLite 和 PostgreSQL 数据库中的数据，让我们看看如何从 Redis 存储中存储和检索数据。

# 从 Redis 存储中存储和检索数据

一些应用程序需要某些类型的数据具有非常快的响应时间；比数据库管理系统（DBMS）能提供的还要快。通常，针对单个用户的 DBMS 会足够快，但对于某些应用程序（通常是大规模 Web 应用程序）来说，有数百个并发查询和许多并发更新。你可以使用多台计算机，但必须在它们之间保持数据的一致性，而保持一致性可能会造成性能瓶颈。

解决这个问题的方法是用一个键值存储，这是一个非常简单的数据库，可以在网络上进行复制。这可以将数据保存在内存中以最大化速度，但它也支持将数据保存到文件中的选项。这可以避免在服务器停止时丢失信息。

键值存储类似于 Rust 标准库中的`HashMap`集合，但它由一个服务器进程管理，该进程可能运行在不同的计算机上。查询是客户端和服务器之间交换的消息。Redis 是最常用的键值存储之一。

该项目的源代码位于`redis_example`文件夹中。要运行它，请打开文件夹并输入`cargo run`。这将打印以下内容：

```rs
a string, 4567, 12345, Err(Response was of incompatible type: "Response type not string compatible." (response was nil)), false.
```

这只是在当前计算机上创建一个数据存储，并在其中存储以下三个键值对：

+   `"aKey"`，与`"a string"`相关联

+   `"anotherKey"`，与`4567`相关联

+   `45`，与`12345`相关联

然后，它查询以下键：

+   `"aKey"`，它获得一个`"a string"`值

+   `"anotherKey"`，它获得一个`4567`值

+   `45`，它获得一个`12345`值

+   `40`，它获得一个错误

然后，它查询存储中是否存在`40`键，它返回`false`。

## 实施项目

在这个项目中只使用了`redis`crate。

代码相当简短且简单。让我们看看它是如何工作的：

```rs
fn main() -> redis::RedisResult<()> {
    let client = redis::Client::open("redis://localhost/")?;
    let mut conn = client.get_connection()?;
```

首先，必须获取一个客户端。对`redis::Client::open`的调用接收一个 URL 并仅检查此 URL 是否有效。如果 URL 有效，则返回一个`redis::Client`对象，该对象没有打开的连接。然后，客户端的`get_connection`方法尝试连接，如果成功，则返回一个打开的连接。

任何连接本质上都有三个重要的方法：

+   `set`: 这尝试存储一个键值对。

+   `get`: 这尝试检索与指定键关联的值。

+   `exists`: 这尝试检测指定的键是否存在于存储中，而不检索其关联的值。

然后，`set`被调用了三次，键和值的类型不同：

```rs
conn.set("aKey", "a string")?;
conn.set("anotherKey", 4567)?;
conn.set(45, 12345)?;
```

最后，`get`被调用了四次，`exists`被调用了一次。前三次调用获取存储的值。第四次调用指定了一个不存在的值，因此返回了一个 null 值，它不能转换为所需的`String`，因此生成了一个错误：

```rs
conn.get::<_, String>("aKey")?,
conn.get::<_, u64>("anotherKey")?,
conn.get::<_, u16>(45)?,
conn.get::<_, String>(40),
conn.exists::<_, bool>(40)?);
```

你可以始终检查错误以找出你的键是否存在，但更干净的方法是调用`exists`方法，它返回一个布尔值，指定键是否存在。

通过这种方式，我们现在知道如何使用 Rust crates 通过最流行的数据库访问、存储和检索数据。

# 将所有这些放在一起

你现在应该知道足够的信息来构建一个示例，它执行了本章开头所描述的内容。我们学习了以下内容：

+   如何读取 TOML 文件以参数化程序

+   如何将有关产品和销售的数据加载到内存中，这些数据指定在 JSON 文件和 XML 文件中

+   如何将所有这些数据存储在三个地方：一个 SQLite 数据库文件、一个 PostgreSQL 数据库和一个 Redis 键值存储

完整示例的源代码可以在 `transformer` 项目中找到。要运行它，打开其文件夹并输入 `cargo run ../data/config.toml`。如果一切顺利，它将重新创建并填充 `data/sales.db` 文件中包含的 SQLite 数据库，PostgreSQL 数据库，该数据库可以通过 `localhost` 的 `5432` 端口访问，并命名为 `Rust2018`，以及 Redis 存储库，可以从 `localhost` 访问。然后，它将查询 SQLite 和 PostgreSQL 数据库中它们的表中的行数，并将打印以下内容：

```rs
SQLite #Products=4\. 
SQLite #Sales=5\. 
PostgreSQL #Products=4\. 
PostgreSQL #Sales=5\. 
```

因此，我们现在已经看到了一个相当广泛的数据操作示例。

# 摘要

在本章中，我们探讨了访问流行文本格式（TOML、JSON 和 XML）或由流行数据库管理器（SQLite、PostgreSQL 和 Redis）管理的数据的一些基本技术。当然，还存在许多其他文件格式和数据库管理器，关于这些格式和数据库管理器还有很多东西要学习。尽管如此，你现在应该已经掌握了它们的功能。这些技术对许多类型的应用程序都很有用。

在下一章中，我们将学习如何使用 REST 架构构建 Web 后端服务。为了使该章节独立，我们将仅使用框架来接收和响应 Web 请求，而不使用数据库。当然，这相当不切实际；但通过将这些 Web 技术与本章中介绍的技术相结合，你可以构建一个现实世界的 Web 服务。

# 问题

1.  为什么用程序方式更改用户编辑的 TOML 文件不是一个好主意？

1.  在什么情况下使用 TOML 或 JSON 文件的动态类型解析更好，而在什么情况下使用静态类型解析更好？

1.  在什么情况下需要从 `Serialize` 和 `Deserialize` 特性派生结构？

1.  什么是 JSON 字符串的漂亮生成？

1.  为什么使用流解析器而不是单次调用解析器可能更好？

1.  在什么情况下 SQLite 是更好的选择，而在什么情况下使用 PostgreSQL 更好？

1.  将 SQL 命令传递给 SQLite 数据库管理器的参数类型是什么？

1.  `query` 方法在 PostgreSQL 数据库上做什么？

1.  读取和写入 Redis 键值存储中值的函数名称是什么？

1.  你能尝试编写一个程序，从命令行获取一个 ID，查询 SQLite、PostgreSQL 或 Redis 数据库以获取该 ID，并打印有关找到的数据的一些信息吗？
