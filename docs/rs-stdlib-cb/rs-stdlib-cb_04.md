# 第四章：序列化

在本章中，我们将介绍以下食谱：

+   处理 CSV

+   使用 Serde 进行序列化基础

+   处理 TOML

+   处理 JSON

+   动态构建 JSON

# 简介

重新发明轮子是没有意义的。许多程序已经提供了大量的功能，它们很乐意与您的程序交互。当然，如果您无法与他们通信，这种提供就毫无价值。

在本章中，我们将探讨 Rust 生态系统中最重要的格式，以便您能够轻松与其他服务进行交流。

# 处理 CSV

存储简单且小型数据集的一个好方法就是 CSV。如果您正在使用像 Microsoft Excel 这样的电子表格应用程序，这个格式也很感兴趣，因为它们对导入和导出各种 CSV 格式有很好的支持。

# 入门

您可能已经知道 CSV 是什么，但稍微复习一下也无妨。

该格式的想法是将值表的所有行都写下来作为*记录*。在记录内部，每个列项都写下来，并用逗号分隔。这就是该格式名称的由来——逗号分隔值。

让我们做一个例子。在下面的代码中，我们将编写一个 CSV 文件，比较太阳系中各种行星与我们的地球。半径、距离太阳和重力为`1`表示*与地球完全相同*。以表格形式写出，我们的值看起来是这样的：

| **name** | **radius** | **distance_from_sun** | **gravity** |
| --- | --- | --- | --- |
| 水星 | 0.38 | 0.47 | 0.38 |
| 金星 | 0.95 | 0.73 | 0.9 |
| 地球 | 1 | 1 | 1 |
| 火星 | 0.53 | 1.67 | 0.38 |
| 木星 | 11.21 | 5.46 | 2.53 |
| 土星 | 9.45 | 10.12 | 1.07 |
| 天王星 | 4.01 | 20.11 | 0.89 |
| 海王星 | 3.88 | 30.33 | 1.14 |

将每一行取出来，用逗号分隔值，将它们各自放在单独的一行上，您就得到了 CSV 文件：

```rs
name,radius,distance_from_sun,gravity
Mercury,0.38,0.47,0.38
Venus,0.95,0.73,0.9
Earth,1,1,1
Mars,0.53,1.67,0.38
Jupiter,11.21,5.46,2.53
Saturn,9.45,10.12,1.07
Uranus,4.01,20.11,0.89
Neptune,3.88,30.33,1.14
```

如您所见，标题（`planet,radius,distance_from_sun,gravity`）简单地写成第一条记录。

# 如何操作...

1.  使用`cargo new chapter_four`创建一个 Rust 项目，在本章中对其进行操作。

1.  导航到新创建的`chapter_four`文件夹。在本章的剩余部分，我们将假设您的命令行当前位于此目录。

1.  在`src`文件夹内，创建一个名为`bin`的新文件夹。

1.  删除生成的`lib.rs`文件，因为我们不是创建一个库。

1.  打开之前为您生成的`Cargo.toml`文件。

1.  在`[dependencies]`下添加以下行：

```rs
 csv = "1.0.0-beta.5"
```

1.  如果您愿意，可以访问`csv`的 crates.io 页面([`crates.io/crates/csv`](https://crates.io/crates/csv))，检查最新版本并使用它。

1.  在`src/bin`文件夹中，创建一个名为`csv.rs`的文件。

1.  添加以下代码，并使用`cargo run --bin csv`运行它：

```rs
1    extern crate csv;
2 
3    use std::io::{BufReader, BufWriter, Read, Seek, SeekFrom,
  Write};
4    use std::fs::OpenOptions;
5 
6    fn main() {
7      let file = OpenOptions::new()
8        .read(true)
9        .write(true)
10       .create(true)
11       .open("solar_system_compared_to_earth.csv")
12       .expect("failed to create csv file");
13 
14     let buf_writer = BufWriter::new(&file);
15     write_records(buf_writer).expect("Failed to write csv");
16 
17     let mut buf_reader = BufReader::new(&file);
18     buf_reader
19       .seek(SeekFrom::Start(0))
20       .expect("Failed to jump to the beginning of the csv");
21     read_records(buf_reader).expect("Failed to read csv");
22   }
23 
24   fn write_records<W>(writer: W) -> csv::Result<()>
25   where
26   W: Write,
27   {
28     let mut wtr = csv::Writer::from_writer(writer);
29 
30     // The header is just a normal record
31     wtr.write_record(&["name", "radius", "distance_from_sun", 
32     "gravity"])?;
33 
34     wtr.write_record(&["Mercury", "0.38", "0.47", "0.38"])?;
35     wtr.write_record(&["Venus", "0.95", "0.73", "0.9"])?;
36     wtr.write_record(&["Earth", "1", "1", "1"])?;
37     wtr.write_record(&["Mars", "0.53", "1.67", "0.38"])?;
38     wtr.write_record(&["Jupiter", "11.21", "5.46", "2.53"])?;
39     wtr.write_record(&["Saturn", "9.45", "10.12", "1.07"])?;
40     wtr.write_record(&["Uranus", "4.01", "20.11", "0.89"])?;
41     wtr.write_record(&["Neptune", "3.88", "30.33", "1.14"])?;
42     wtr.flush()?;
43     Ok(())
44   }
45 
46   fn read_records<R>(reader: R) -> csv::Result<()>
47   where
48   R: Read,
49   {
50     let mut rdr = csv::Reader::from_reader(reader);
51     println!("Comparing planets in the solar system with the 
52     earth");
53     println!("where a value of '1' means 'equal to earth'");
54     for result in rdr.records() {
55       println!("-------");
56       let record = result?;
57       if let Some(name) = record.get(0) {
58         println!("Name: {}", name);
59       }
60       if let Some(radius) = record.get(1) {
61         println!("Radius: {}", radius);
62       }
63       if let Some(distance) = record.get(2) {
64         println!("Distance from sun: {}", distance);
65       }
66       if let Some(gravity) = record.get(3) {
67         println!("Surface gravity: {}", gravity);
68       }
69     }
70     Ok(())
71   }
```

# 它是如何工作的...

首先，我们准备我们的文件[9]及其`OpenOptions`，以便我们可以在文件上同时拥有`read`和`write`访问权限。您会记得这一点来自第三章，*处理文件和文件系统*；*处理文本文件*。

然后我们写入 CSV。我们通过将任何类型的`Write`包装在`csv::Writer`[30]中来完成此操作。然后您可以使用`write_record`来写入任何可以表示为`&[u8]`迭代器的数据类型。大多数情况下，这只是一个字符串数组。

在读取时，我们同样将一个`Read`包装在`csv::Read`中。`records()`方法返回一个`Result`的`StringRecord`迭代器。这样，您可以决定如何处理格式不正确的记录。在我们的例子中，我们简单地跳过它。最后，我们在一个记录上调用`get()`以获取某个字段。如果没有在指定的索引处有条目，或者它超出了范围，这将返回`None`。

# 还有更多...

如果您需要读取或写入自定义 CSV 格式，例如使用制表符而不是逗号作为分隔符的格式，您可以使用`WriterBuilder`和`ReaderBuilder`来自定义预期的格式。如果您计划使用 Microsoft Excel，请务必记住这一点，因为它在选择分隔符方面具有令人烦恼的区域不一致性([`stackoverflow.com/questions/10140999/csv-with-comma-or-semicolon`](https://stackoverflow.com/questions/10140999/csv-with-comma-or-semicolon))。

当处理 CSV 和 Microsoft Excel 时，请小心，并在将其交给 Excel 之前清理您的数据。尽管 CSV 被定义为没有控制标识符的纯数据，但 Excel 在导入 CSV 时会解释并执行宏。关于由此可能打开的可能攻击向量，请参阅[`georgemauer.net/2017/10/07/csv-injection.html`](http://georgemauer.net/2017/10/07/csv-injection.html)。

如果 Windows 应用程序拒绝接受`csv`默认使用的`\n`终止符，这也会很有用。在这种情况下，只需在构建者中指定以下代码即可使用 Windows 本地的`\r\n`终止符：

```rs
.terminator(csv::Terminator::CRLF)
```

`csv`crate 允许您比本配方中显示的更灵活地操作您的数据。例如，您可以在`StringRecord`中动态插入新字段。我们故意不详细探讨这些可能性，因为 CSV 格式并不适用于这类数据操作。如果您需要做比简单的导入/导出更多的事情，您应该使用更合适的格式，例如 JSON，我们将在本章中也探讨这一点。

# 参见

+   *使用构建者模式*的配方在第一章，*学习基础知识*

+   *处理文本文件*的配方在第三章，*处理文件和文件系统*

# 使用 Serde 进行序列化基础知识

Rust 中所有序列化事物的**事实标准**是 Serde 框架。本章中的其他所有食谱都将部分使用它。为了让你熟悉 Serde 的做事方式，我们将使用它重写上一个食谱。在本章的后面部分，我们将详细了解 Serde 的工作原理，以便实现将数据反序列化到自定义格式。

# 如何做到...

1.  打开之前为你生成的 `Cargo.toml` 文件。

1.  在 `[dependencies]` 下，添加以下行：

```rs
serde = "1.0.24"
serde_derive = "1.0.24"
```

1.  如果你还没有在上一个食谱中这样做，请添加以下行：

```rs
csv = "1.0.0-beta.5"

```

1.  如果你想，可以去 Serde 的 crates.io 网页（[`crates.io/crates/serde`](https://crates.io/crates/serde)）、`serde_derive`（[`crates.io/crates/serde_derive`](https://crates.io/crates/serde_derive)）和 CSV（[`crates.io/crates/csv`](https://crates.io/crates/csv)）查看最新版本，并使用这些版本。

1.  在 `bin` 文件夹中，创建一个名为 `serde_csv.rs` 的文件。

1.  添加以下代码，并用 `cargo run --bin serde_csv` 运行它：

```rs
1    extern crate csv;
2    extern crate serde;
3    #[macro_use]
4    extern crate serde_derive;
5 
6    use std::io::{BufReader, BufWriter, Read, Seek, SeekFrom, 
     Write};
7    use std::fs::OpenOptions;
8 
9    #[derive(Serialize, Deserialize)]
10   struct Planet {
11     name: String,
12     radius: f32,
13     distance_from_sun: f32,
14     gravity: f32,
15   }
16 
17   fn main() {
18     let file = OpenOptions::new()
19       .read(true)
20       .write(true)
21       .create(true)
22       .open("solar_system_compared_to_earth.csv")
23       .expect("failed to create csv file");
24 
25     let buf_writer = BufWriter::new(&file);
26     write_records(buf_writer).expect("Failed to write csv");
27 
28     let mut buf_reader = BufReader::new(&file);
29     buf_reader
30       .seek(SeekFrom::Start(0))
31       .expect("Failed to jump to the beginning of the csv");
32     read_records(buf_reader).expect("Failed to read csv");
33   }
34 
35   fn write_records<W>(writer: W) -> csv::Result<()>
36   where
37   W: Write,
38  {
39     let mut wtr = csv::Writer::from_writer(writer);
40 
41     // No need to specify a header; Serde creates it for us
42     wtr.serialize(Planet {
43       name: "Mercury".to_string(),
44       radius: 0.38,
45       distance_from_sun: 0.47,
46       gravity: 0.38,
47     })?;
48     wtr.serialize(Planet {
49       name: "Venus".to_string(),
50       radius: 0.95,
51       distance_from_sun: 0.73,
52       gravity: 0.9,
53     })?;
54     wtr.serialize(Planet {
55       name: "Earth".to_string(),
56       radius: 1.0,
57       distance_from_sun: 1.0,
58       gravity: 1.0,
59     })?;
60     wtr.serialize(Planet {
61       name: "Mars".to_string(),
62       radius: 0.53,
63       distance_from_sun: 1.67,
64       gravity: 0.38,
65     })?;
66     wtr.serialize(Planet {
67       name: "Jupiter".to_string(),
68       radius: 11.21,
69       distance_from_sun: 5.46,
70       gravity: 2.53,
71     })?;
72     wtr.serialize(Planet {
73       name: "Saturn".to_string(),
74       radius: 9.45,
75       distance_from_sun: 10.12,
76       gravity: 1.07,
77    })?;
78    wtr.serialize(Planet {
79      name: "Uranus".to_string(),
80      radius: 4.01,
81      distance_from_sun: 20.11,
82      gravity: 0.89,
83    })?;
84    wtr.serialize(Planet {
85      name: "Neptune".to_string(),
86      radius: 3.88,
87      distance_from_sun: 30.33,
88      gravity: 1.14,
89    })?;
90    wtr.flush()?;
91    Ok(())
92  }
93 
94  fn read_records<R>(reader: R) -> csv::Result<()>
95  where
96  R: Read,
97  {
98    let mut rdr = csv::Reader::from_reader(reader);
99    println!("Comparing planets in the solar system with the 
   earth");
100   println!("where a value of '1' means 'equal to earth'");
101   for result in rdr.deserialize() {
102     println!("-------");
103     let planet: Planet = result?;
104     println!("Name: {}", planet.name);
105     println!("Radius: {}", planet.radius);
106     println!("Distance from sun: {}", planet.distance_from_sun);
107     println!("Surface gravity: {}", planet.gravity);
108   }
109   Ok(())
110 }
```

# 它是如何工作的...

本食谱中的代码读取和写入与上一个食谱完全相同的 CSV。唯一的区别是我们如何处理单个记录。Serde 通过允许我们使用普通的 Rust 结构体来帮助我们。我们唯一需要做的是从 `Serialize` 和 `Deserialize` [9] 中派生我们的 `Planet` 结构体。其余的将由 Serde 自动处理。

由于我们现在使用实际的 Rust 结构体来表示一个行星，我们通过调用 `serialize` 并传入一个结构体来创建记录，而不是像以前那样使用 `write_record` [42]。看起来是不是更易于阅读了？如果你认为示例变得有点冗长，你可以像在第一章中描述的那样，将实际对象创建隐藏在构造函数后面，*学习基础知识*；*使用构造函数模式*。

当读取 CSV 时，我们也不再需要手动访问 `StringRecord` 的字段。相反，`deserialize()` 返回一个已反序列化 `Planet` 对象的 `Result` 迭代器。再次，看看这变得多么易于阅读。

如你所猜，你应该尽可能使用 Serde，因为它通过提供可读性和编译时类型安全来帮助你尽早捕获可能的错误。

# 还有更多...

Serde 允许你通过注释你的字段来稍微调整序列化过程。例如，如果你无法通过写入 `#[serde(default)]` 在其声明上方为其提供一个标准值。在一个结构体中，它看起来像这样：

```rs
#[derive(Serialize, Deserialize)]
struct Foo {
    bar: i32,
    #[serde(default)]
    baz: i32,
}
```

如果 `baz` 没有被解析，它的 `Default::default` 值（参见 第一章，*学习基础知识*；提供默认实现）将被使用。你可以用注解做的另一件有用的事情是改变期望的大小写约定。默认情况下，Serde 会期望 Rust 的 `snake_case`，但是，你可以通过用 `#[serde(rename_all = "PascalCase")]` 注解一个 `struct` 或 `enum` 来改变这一点。你可以在这样的 `struct` 上使用它：

```rs
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "PascalCase")]
struct Stats {
    number_of_clicks: i32,
    total_time_played: i32,
}
```

这将，而不是解析 `number_of_clicks` 和 `total_time_played`，期望 `NumberOfClicks` 和 `TotalTimePlayed` 键被调用。Serde 支持的其他可能的比 `PascalCase` 更多的案例约定包括小写、*camelCase* 和 *SCREAMING_SNAKE_CASE*。

有许多不同且有用的属性。如果你想，你可以在 [`serde.rs/attributes.html`](https://serde.rs/attributes.html) 上熟悉它们。

你可以使用 Serde 提供惯用的序列化和反序列化，然而，讨论所有最佳实践将涵盖一个单独章节。如果你想深入了解这些内容，Serde 在 [`serde.rs/data-format.html`](https://serde.rs/data-format.html) 上有一个很好的关于如何做到这一点的说明。

# 参见

+   *使用构造函数模式* 和 *提供默认实现* 的配方在 第一章，*学习基础知识*

+   *在第三章的“*处理文本文件*”中工作，*处理文件和文件系统*

# 与 TOML 一起工作

你喜欢 INI 文件的简洁性，但希望它们有正式的规范并且有更多功能吗？汤姆·普雷斯顿-沃纳（GitHub 和 Gravatar 等服务的创始人）也是这样想的。他创建了汤姆的明显、最小化语言，简称 TOML。这种相对较新的格式在新项目中越来越受欢迎。实际上，你现在已经多次使用它了：Cargo 的依赖关系在每一个项目的 `Cargo.toml` 文件中指定！

# 入门

在其核心，TOML 完全是关于 *键值* 对。这是你可以创建的最简单的 TOML 文件：

```rs
message = "Hello World"
```

在这里，键信息具有 `"Hello World"` 值。值也可以是一个数组：

```rs
messages: ["Hello", "World", "out", "there"]
```

一组键值被称为 *表*. 下面的 TOML 允许 `smileys` 表包含 `happy` 键和 `":)"` 值，以及 `sad` 键和 `":("` 值：

```rs
[smileys]
happy = ":)"
sad = ":("
```

一个特别小的表可以 *内联*，也就是说，在一行中写出来。最后一个例子与以下内容完全相同：

```rs
smileys = { happy = ":)", sad = ":(" }
```

表可以通过用点分隔它们的名称来嵌套：

```rs
[servers]
  [servers.production]
  ip = "192.168.0.1"
  [servers.beta]
  ip = "192.169.0.2"
  [servers.testing]
  ip = "192.169.0.3"
```

TOML 的一个优点是，如果你需要指定更多信息，你可以将任何键转换为表。例如，Cargo 本身期望在声明依赖版本时这样做。例如，如果你想使用 `rocket_contrib`，这是流行的 Rust `rocket` 网络框架的辅助包，版本为 0.3.3，你会这样写：

```rs
[dependencies]
rocket_contrib = 0.3.3
```

然而，如果你想要指定 `rocket_contrib` 中要包含的确切功能，你需要将其写成 `dependencies` 的子表。以下 TOML 会告诉 `Cargo` 使用其 JSON 序列化功能：

```rs
[dependencies]
[dependencies.rocket_contrib]
version = "0.3.3"
default-features = false
features = ["json"]
```

TOML 带来的另一个优点是它的空白不重要，也就是说，你可以按任何方式缩进文件。你甚至可以通过以下方式添加注释：从行的开头开始：

```rs
# some comment
```

如果你想要进一步探索格式，TOML 语法的全部内容在 [`github.com/toml-lang/toml`](https://github.com/toml-lang/toml) 中指定。

# 如何做到...

1.  打开为你生成的 `Cargo.toml` 文件

1.  在 `[dependencies]` 下添加以下行：

```rs
toml = "0.4.5"
```

1.  如果你还没有这样做，也请添加以下行：

```rs
serde = "1.0.24"
serde_derive = "1.0.24"
```

1.  如果你愿意，你可以访问 TOML ([`crates.io/crates/toml`](https://crates.io/crates/toml))、Serde ([`crates.io/crates/serde`](https://crates.io/crates/serde)) 和 `serde_derive` ([`crates.io/crates/serde_derive`](https://crates.io/crates/serde_derive)) 的 crates.io 网页，检查最新版本并使用这些版本。

1.  在 `bin` 文件夹中创建一个名为 `toml.rs` 的文件

1.  添加以下代码并使用 `cargo run --bin toml` 运行它：

```rs
1    #[macro_use]
2    extern crate serde_derive;
3    extern crate toml;
4
5    use std::io::{BufReader, BufWriter, Read, Seek, SeekFrom, 
     Write};
6    use std::fs::OpenOptions;
```

1.  这些是我们将在整个配方中使用的结构：

```rs
8    #[derive(Serialize, Deserialize)]
9    struct Preferences {
10     person: Person,
11     language: Language,
12     privacy: Privacy,
13   }
14 
15   #[derive(Serialize, Deserialize)]
16   struct Person {
17     name: String,
18     email: String,
19   }
20 
21   #[derive(Serialize, Deserialize)]
22   struct Language {
23     display: String,
24     autocorrect: Option<Vec<String>>,
25   }
26 
27   #[derive(Serialize, Deserialize)]
28   struct Privacy {
29     share_anonymous_statistics: bool,
30     public_name: bool,
31     public_email: bool,
32   }
```

1.  准备一个新的文件并调用其他函数：

```rs
34   fn main() {
35     let file = OpenOptions::new()
36      .read(true)
37      .write(true)
38      .create(true)
39      .open("preferences.toml")
40      .expect("failed to create TOML file");
41 
42     let buf_writer = BufWriter::new(&file);
43     write_toml(buf_writer).expect("Failed to write TOML");
44 
45     let mut buf_reader = BufReader::new(&file);
46     buf_reader
47       .seek(SeekFrom::Start(0))
48       .expect("Failed to jump to the beginning of the TOML 
         file");
49     read_toml(buf_reader).expect("Failed to read TOML");
50   }
```

1.  将我们的结构保存为 TOML 文件：

```rs
52   type SerializeResult<T> = Result<T, toml::ser::Error>;
53   fn write_toml<W>(mut writer: W) -> SerializeResult<()>
54   where
55   W: Write,
56   {
57     let preferences = Preferences {
58       person: Person {
59         name: "Jan Nils Ferner".to_string(),
60         email: "jn_ferner@hotmail.de".to_string(),
61       },
62       language: Language {
63         display: "en-GB".to_string(),
64         autocorrect: Some(vec![
65           "en-GB".to_string(),
66           "en-US".to_string(),
67           "de-CH".to_string(),
68         ]),
69       },
70       privacy: Privacy {
71         share_anonymous_statistics: false,
72         public_name: true,
73         public_email: true,
74       },
75     };
76 
77     let toml = toml::to_string(&preferences)?;
78     writer
79       .write_all(toml.as_bytes())
80       .expect("Failed to write file");
81     Ok(())
82   }
```

1.  `读取` 我们刚刚创建的 TOML 文件：

```rs
84   type DeserializeResult<T> = Result<T, toml::de::Error>;
85   fn read_toml<R>(mut reader: R) -> DeserializeResult<()>
86   where
87   R: Read,
88   {
89     let mut toml = String::new();
90     reader
91       .read_to_string(&mut toml)
92       .expect("Failed to read TOML");
93     let preferences: Preferences = toml::from_str(&toml)?;
94 
95     println!("Personal data:");
96     let person = &preferences.person;
97     println!(" Name: {}", person.name);
98     println!(" Email: {}", person.email);
99 
100     println!("\nLanguage preferences:");
101     let language = &preferences.language;
102     println!(" Display language: {}", language.display);
103     println!(" Autocorrect priority: {:?}", 
        language.autocorrect);
104 
105 
106     println!("\nPrivacy settings:");
107     let privacy = &preferences.privacy;
108     println!(
109       " Share anonymous usage statistics: {}",
110         privacy.share_anonymous_statistics
111     );
112     println!(" Display name publically: {}", 
        privacy.public_name);
113     println!(" Display email publically: {}", 
        privacy.public_email);
114 
115     Ok(())
116  }
```

# 它是如何工作的...

如往常一样，使用 Serde，我们首先需要声明我们计划使用的结构 [8 to 32]。

在序列化过程中，我们可以直接调用 Serde 的 `to_string` 方法，因为 TOML 重新导出它们 [77]。这返回一个 `String`，然后我们可以将其写入文件 [79]。Serde 的 `from_str` 也是如此，当类型注解时，它接受一个 `&str` 并将其转换为结构体。

# 还有更多...

你可能已经注意到，在阅读或编写此配方时我们没有使用 try-运算符（`?`）。这是因为函数期望的错误类型，`se::Error` [77] 和 `de::Error`[93]，与 `std::io::Error` 不兼容。在第六章，*处理错误*；*提供用户定义的错误类型*中，我们将探讨如何通过返回包含其他提到的错误类型的自己的错误类型来避免这种情况。

在此配方中使用的 TOML crate 与 Cargo 本身使用的相同。如果你对 Cargo 如何解析其自己的 `Cargo.toml` 文件感兴趣，可以查看 [`github.com/rust-lang/cargo/blob/master/src/cargo/util/toml/mod.rs`](https://github.com/rust-lang/cargo/blob/master/src/cargo/util/toml/mod.rs)。

# 参见

+   在第三章，*处理文件和文件系统*中的*处理文本文件*配方

+   *提供用户定义的错误类型* 配方在 第六章，*处理错误*

# 处理 JSON

大多数 Web API 和许多本地 API 现在都使用 JSON。当设计供其他程序消费的数据时，它应该是你的首选格式，因为它轻量级、简单、易于使用和理解，并且在各种编程语言中都有出色的库支持，尤其是 JavaScript。

# 准备工作

JSON 是在大多数 Web 通信通过发送 XML 通过 Java 或 Flash 等浏览器插件完成的时候被创建的。这很麻烦，并且使得交换的信息相当庞大。JSLint 的创造者、著名作品《JavaScript: The Good Parts》的作者 Douglas Crockford 在 2000 年初决定，是时候有一个轻量级且易于与 JavaScript 集成的格式了。他将自己定位在 JavaScript 的小子集上，即它定义对象的方式，并在此基础上稍作扩展，形成了 JavaScript 对象表示法或 JSON。是的，你没有看错；JSON *不是* JavaScript 的子集，因为它接受 JavaScript 不接受的事物。你可以在 [`timelessrepo.com/json-isnt-a-javascript-subset.`](http://timelessrepo.com/json-isnt-a-javascript-subset) 上了解更多关于这一点。

这个故事悲哀的讽刺之处在于，今天我们已经回到了原点：我们最好的 Web 开发实践包括一个错综复杂的任务运行器、框架和转换器迷宫，这些在理论上都很不错，但最终却变成了一个庞大的混乱。但这又是另一个故事了。

JSON 建立在两种结构之上：

+   由 `{` 和 `}` 包围的一组键值对，这被称为 *对象*，它本身也可以是一个值

+   被称为 *数组* 的由 `[` 和 `]` 包围的值列表

这可能会让你想起我们之前讨论的 TOML 语法，你可能会问自己在什么情况下应该选择其中一个。答案是，当你的数据将要被工具自动读取时，JSON 是一个好的格式，而当你的数据旨在由人类手动读取和修改时，TOML 是出色的。

注意，JSON *不允许* 注释。这很有道理，因为注释无论如何都是不可读的。

一个包含值、子对象和数组的 JSON 对象示例可能是以下内容：

```rs
{
  name: "John",
  age: 23,
  pets: [
    {
      name: "Sparky",
      species: "Dog",
      age: 2
    },
    {
      name: "Speedy",
      species: "Turtle",
      age: 47,
      colour: "Green"
    },
    {
      name: "Meows",
      species: "Cat",
      colour: "Orange"
    }
  ]
}
```

我们将在下面的代码中编写和读取这个精确的示例。宠物定义的不一致性是有意为之的，因为许多 Web API 在某些情况下会省略某些键。

# 如何做到这一点...

1.  打开之前为你生成的 `Cargo.toml` 文件

1.  在 `[dependencies]` 下，添加以下行：

```rs
serde_json = "1.0.8"
```

1.  如果你还没有这样做，也请添加以下行：

```rs
serde = "1.0.24"
serde_derive = "1.0.24"
```

1.  如果你想的话，你可以访问 `serde_json` ([`crates.io/crates/serde_json`](https://crates.io/crates/serde_json))、Serde ([`crates.io/crates/serde`](https://crates.io/crates/serde)) 和 `serde_derive` ([`crates.io/crates/serde_derive`](https://crates.io/crates/serde_derive)) 的 crates.io 网页，检查最新版本并使用这些版本。

1.  在`bin`文件夹中，创建一个名为`json.rs`的文件

1.  添加以下代码，并用`cargo run --bin json`运行：

```rs
1    extern crate serde;
2    extern crate serde_json;
3
4    #[macro_use]
5    extern crate serde_derive;
6
7    use std::io::{BufReader, BufWriter, Read, Seek, SeekFrom, 
     Write};
8    use std::fs::OpenOptions;
```

1.  这些是我们将在整个食谱中使用的结构：

```rs
10   #[derive(Serialize, Deserialize)]
11   struct PetOwner {
12     name: String,
13     age: u8,
14     pets: Vec<Pet>,
15   }
16 
17   #[derive(Serialize, Deserialize)]
18   struct Pet {
19     name: String,
20     species: AllowedSpecies,
21     // It is usual for many JSON keys to be optional
22     age: Option<u8>,
23     colour: Option<String>,
24   }
25 
26   #[derive(Debug, Serialize, Deserialize)]
27   enum AllowedSpecies {
28     Dog,
29     Turtle,
30     Cat,
31   }
```

1.  准备一个新的文件并调用其他函数：

```rs
33   fn main() {
34     let file = OpenOptions::new()
35       .read(true)
36       .write(true)
37       .create(true)
38       .open("pet_owner.json")
39       .expect("failed to create JSON file");
40 
41     let buf_writer = BufWriter::new(&file);
42     write_json(buf_writer).expect("Failed to write JSON");
43 
44     let mut buf_reader = BufReader::new(&file);
45     buf_reader
46       .seek(SeekFrom::Start(0))
47       .expect("Failed to jump to the beginning of the JSON 
          file");
48     read_json(buf_reader).expect("Failed to read JSON");
49   }
```

1.  将我们的结构保存为 JSON 文件：

```rs
52   fn write_json<W>(mut writer: W) -> serde_json::Result<()>
53   where
54   W: Write,
55   {
56     let pet_owner = PetOwner {
57       name: "John".to_string(),
58       age: 23,
59       pets: vec![
60         Pet {
61           name: "Waldo".to_string(),
62           species: AllowedSpecies::Dog,
63           age: Some(2),
64           colour: None,
65         },
66         Pet {
67           name: "Speedy".to_string(),
68           species: AllowedSpecies::Turtle,
69           age: Some(47),
70           colour: Some("Green".to_string()),
71         },
72         Pet {
73           name: "Meows".to_string(),
74           species: AllowedSpecies::Cat,
75           age: None,
76           colour: Some("Orange".to_string()),
77         },
78       ],
79     };
80 
81     let json = serde_json::to_string(&pet_owner)?;
82     writer
83      .write_all(json.as_bytes())
84      .expect("Failed to write file");
85     Ok(())
86   }
```

1.  读取我们刚刚创建的 JSON 文件：

```rs
88   fn read_json<R>(mut reader: R) -> serde_json::Result<()>
89   where
90   R: Read,
91   {
92     let mut json = String::new();
93     reader
94       .read_to_string(&mut json)
95       .expect("Failed to read TOML");
96     let pet_owner: PetOwner = serde_json::from_str(&json)?;
97 
98     println!("Pet owner profile:");
99     println!(" Name: {}", pet_owner.name);
100    println!(" Age: {}", pet_owner.age);
101 
102    println!("\nPets:");
103    for pet in pet_owner.pets {
104      println!(" Name: {}", pet.name);
105      println!(" Species: {:?}", pet.species);
106      if let Some(age) = pet.age {
107        println!(" Age: {}", age);
108      }
109      if let Some(colour) = pet.colour {
110        println!(" Colour: {}", colour);
111      }
112      println!();
113    }
114    Ok(())
115  }
```

# 它是如何工作的...

注意这个食谱看起来几乎和上一个完全一样？除了结构，唯一的显著区别是我们调用了`serde_json::to_string()` [81]而不是`toml::to_string()`，以及`serde_json::from_str()` [96]而不是`toml::from_str()`。这正是像 Serde 这样的精心设计的框架的美丽之处：自定义序列化和反序列化代码隐藏在特质定义之后，我们可以使用相同的 API 而不必关心内部实现细节。

除了这些，没有其他需要说的，因为之前已经说过了，所以我们不会介绍其他格式。所有重要的格式都支持 Serde，所以你可以像使用本章中的其他格式一样使用它们。有关所有支持格式的完整列表，请参阅[`docs.serde.rs/serde/index.html`](https://docs.serde.rs/serde/index.html)。

# 还有更多...

JSON 没有类似`enum`的概念。然而，由于许多语言*确实*使用它们，多年来已经出现了多种处理从 JSON 到`enum`转换的约定。Serde 允许你通过枚举上的注解来支持这些约定。有关支持的转换的完整列表，请访问[`serde.rs/enum-representations.html`](https://serde.rs/enum-representations.html)。

# 参见

+   在第三章的*处理文本文件*食谱中，*处理文件和文件系统*

+   在第六章的*处理错误*食谱中，*提供用户定义的错误类型*

# 动态构建 JSON

当一个 JSON API 使用一个考虑不周到的模式和不一致的对象设计时，你可能会得到一个大多数成员都是`Option`的巨大结构。如果你发现自己只向这样的服务发送数据，那么动态地逐个构建你的 JSON 属性可能会容易一些。

# 如何操作...

1.  打开之前为你生成的`Cargo.toml`文件

1.  在`[dependencies]`部分，如果你还没有这样做，请添加以下行：

```rs
serde_json = "1.0.8"

```

1.  如果你想，你可以访问`serde_json`的 crates.io 网页([`crates.io/crates/serde_json`](https://crates.io/crates/serde_json))来检查最新版本，并使用那个版本

1.  在`bin`文件夹中，创建一个名为`dynamic_json.rs`的文件

1.  添加以下代码，并用`cargo run --bin dynamic_json`运行：

```rs
1    #[macro_use] 
2    extern crate serde_json;
3 
4    use std::io::{self, BufRead};
5    use std::collections::HashMap;
6 
7    fn main() {
8      // A HashMap is the same as a JSON without any schema
9      let mut key_value_map = HashMap::new();
10     let stdin = io::stdin();
11     println!("Enter a key and a value");
12     for input in stdin.lock().lines() {
13       let input = input.expect("Failed to read line");
14       let key_value: Vec<_> = input.split_whitespace().collect();
15       let key = key_value[0].to_string();
16       let value = key_value[1].to_string();
17 
18       println!("Saving key-value pair: {} -> {}", key, value);
19       // The json! macro lets us convert a value into its JSON 
            representation
20       key_value_map.insert(key, json!(value));
21       println!(
22         "Enter another pair or stop by pressing '{}'",
23          END_OF_TRANSMISSION
24       );
25     }
26     // to_string_pretty returns a JSON with nicely readable 
          whitespace
27     let json =
28     serde_json::to_string_pretty(&key_value_map).expect("Failed 
       to convert HashMap into JSON");
29     println!("Your input has been made into the following 
       JSON:");
30     println!("{}", json);
31   }
32 
33   #[cfg(target_os = "windows")]
34   const END_OF_TRANSMISSION: &str = "Ctrl Z";
35 
36   #[cfg(not(target_os = "windows"))]
37   const END_OF_TRANSMISSION: &str = "Ctrl D";
```

# 它是如何工作的...

在这个例子中，用户可以输入任意数量的键值对，直到他们决定停止，此时他们会以 JSON 的形式收到他们的输入。你可以输入的一些示例输入包括：

```rs
name abraham
age 49
fav_colour red
hello world
(press 'Ctrl Z' on Windows or 'Ctrl D' on Unix)
```

使用 `#[cfg(target_os = "some_operating_system")]` 来处理特定于操作系统的环境。在这个配方中，我们使用它来有条件地编译 `END_OF_TRANSMISSION` 常量，在 Windows 上与 Unix 上不同。这个键组合告诉操作系统停止当前输入流。

这个程序的想法是，一个没有明确定义模式的 JSON 对象不过是一个 `HashMap<String, String>`[9]。现在，`serde_json` 不接受 `String` 作为值，因为这不够通用。相反，它需要一个 `serde_json::Value`，你可以通过在几乎任何类型上调用 `json!` 宏来轻松构建 [20]。

当我们完成时，我们不再像以前那样调用 `serde_json::to_string()`，而是使用 `serde_json::to_string_pretty()` [28]，因为这会产生效率较低但可读性更强的 JSON。记住，JSON 并不主要是供人类阅读的，这就是为什么 Serde 序列化它的默认方式是完全不带任何空白的。如果你对确切差异感到好奇，可以随意将 `to_string_pretty()` 改为 `to_string()` 并比较结果。

# 参见

+   *从标准输入读取* 的配方在 第一章，*学习基础知识*

+   *作为迭代器访问集合* 和 *使用 HashMap* 的配方在 第二章，*与集合一起工作*
