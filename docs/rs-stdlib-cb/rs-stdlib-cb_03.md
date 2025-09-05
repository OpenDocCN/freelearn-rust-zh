# 第三章：处理文件和文件系统

在本章中，我们将涵盖以下菜谱：

+   处理文本文件

+   处理字节

+   处理二进制文件

+   压缩和解压缩数据

+   遍历文件系统

+   使用 glob 模式查找文件

# 简介

在大数据、机器学习和云服务时代，你不能依赖你的所有数据始终在内存中。相反，你需要能够有效地检查和遍历文件系统，并在你方便的时候操作其内容。

在阅读了这一章之后，你将能够做到的事情包括在子目录中配置具有不同命名变体的文件、以高效的二进制格式保存你的数据、读取其他程序生成的协议，以及压缩你的数据以便以快速的速度通过互联网发送。

# 处理文本文件

在这个菜谱中，我们将学习如何读取、写入、创建、截断和追加文本文件。掌握了这些知识，你将能够将所有其他菜谱应用于文件，而不是内存中的字符串。

# 如何去做...

1.  使用 `cargo new chapter-three` 创建一个在本章中工作的 Rust 项目。

1.  导航到新创建的 `chapter-three` 文件夹。在本章的其余部分，我们将假设你的命令行当前位于这个目录。

1.  在 `src` 文件夹内，创建一个名为 `bin` 的新文件夹。

1.  删除生成的 `lib.rs` 文件，因为我们不是创建一个库。

1.  在 `src/bin` 文件夹中，创建一个名为 `text_files.rs` 的文件。

1.  添加以下代码，并用 `cargo run --bin text_files` 运行它：

```rs
1    use std::fs::{File, OpenOptions};
2    use std::io::{self, BufReader, BufWriter, Lines, Write};
3    use std::io::prelude::*;
4 
5    fn main() {
6      // Create a file and fill it with data
7      let path = "./foo.txt";
8      println!("Writing some data to '{}'", path);
9      write_file(path, "Hello World!\n").expect("Failed to write to 
        file");
10     // Read entire file as a string
11     let content = read_file(path).expect("Failed to read file");
12     println!("The file '{}' contains:", path);
13     println!("{}", content);
14 
15     // Overwrite the file
16     println!("Writing new data to '{}'", path);
17     write_file(path, "New content\n").expect("Failed to write to 
        file");
18     let content = read_file(path).expect("Failed to read file");
19     println!("The file '{}' now contains:", path);
20     println!("{}", content);
21 
22     // Append data to the file
23     println!("Appending data to '{}'", path);
24     append_file(path, "Some more content\n").expect("Failed to 
        append to file");
25     println!("The file '{}' now contains:", path);
26     // Read file line by line as an iterator
27     let lines = read_file_iterator(path).expect("Failed to read 
        file");
28     for line in lines {
29       println!("{}", line.expect("Failed to read line"));
30     }
31 
32     append_and_read(path, "Last line in the file, 
        goodbye").expect("Failed to read and write file");
       }
```

1.  这些是 `main()` 函数调用的函数：

```rs
37   fn read_file(path: &str) -> io::Result<String> {
38     // open() opens the file in read-only mode
39     let file = File::open(path)?;
40     // Wrap the file in a BufReader
41     // to read in an efficient way
42     let mut buf_reader = BufReader::new(file);
43     let mut content = String::new();
44     buf_reader.read_to_string(&mut content)?;
45     Ok(content)
46   }
47 
48   fn read_file_iterator(path: &str) ->
       io::Result<Lines<BufReader<File>>> {
49     let file = File::open(path)?;
50     let buf_reader = BufReader::new(file);
51     // lines() returns an iterator over lines
52     Ok(buf_reader.lines())
53   }
54 
55 
56   fn write_file(path: &str, content: &str) -> io::Result<()> {
57     // create() opens a file with the standard options
58     // to create, write and truncate a file
59     let file = File::create(path)?;
60     // Wrap the file in a BufReader
61     // to read in an efficient way
62     let mut buf_writer = BufWriter::new(file);
63     buf_writer.write_all(content.as_bytes())?;
64     Ok(())
65   }
66 
67   fn append_file(path: &str, content: &str) -> io::Result<()> {
68     // OpenOptions lets you set all options individually
69     let file = OpenOptions::new().append(true).open(path)?;
70     let mut buf_writer = BufWriter::new(file);
71     buf_writer.write_all(content.as_bytes())?;
72     Ok(())
73   }
```

1.  在同一个句柄上进行读取和写入：

```rs
76   fn append_and_read(path: &str, content: &str) -> io::Result<()
     {
       let file = 
77       OpenOptions::new().read(true).append(true).open(path)?;
78     // Passing a reference of the file will not move it
79     // allowing you to create both a reader and a writer
80     let mut buf_reader = BufReader::new(&file);
81     let mut buf_writer = BufWriter::new(&file);
82 
83     let mut file_content = String::new();
84     buf_reader.read_to_string(&mut file_content)?; 
85     println!("File before appending:\n{}", file_content);
86 
87     // Appending will shift your positional pointer
88     // so you have to save and restore it
89     let pos = buf_reader.seek(SeekFrom::Current(0))?;
90     buf_writer.write_all(content.as_bytes())?;
91     // Flushing forces the write to happen right now
92     buf_writer.flush()?;
93     buf_reader.seek(SeekFrom::Start(pos))?;
94 
95     buf_reader.read_to_string(&mut file_content)?;
96     println!("File after appending:\n{}", file_content);
97 
98     Ok(())
99   }
```

# 它是如何工作的...

我们的 `main` 函数分为三个部分：

1.  创建一个文件。

1.  覆盖文件，在这个上下文中被称为 *截断*。

1.  向文件中追加。

在前两部分中，我们将整个文件内容加载到一个单独的 `String` 中并显示它 [11 和 18]。在最后一部分中，我们遍历文件中的单个行并打印它们 [28]。

`File::open()` 以只读模式打开文件，并返回对该文件的句柄 [39]。因为这个句柄实现了 `Read` 特性，我们现在可以直接使用 `read_to_string` 将其读入字符串。然而，在我们的示例中，我们首先将其包装在一个 `BufReader` [42] 中。这是因为专门的读取器可以通过收集读取指令来大大提高其资源访问的性能，这被称为 *缓冲*，并且可以批量执行它们。对于第一个读取示例 `read_file` [37]，这根本没有任何区别，因为我们无论如何都是一次性读取的。我们仍然使用它，因为这是一种良好的实践，它允许我们在不担心性能的情况下，灵活地更改函数的精确读取机制。如果你想看到一个 `BufReader` 确实做了些事情的函数，可以向下看一点，到 `read_file_iterator` [48]。它看起来是逐行读取文件的。当处理大文件时，这将是一个非常低效的操作，这就是为什么 `BufReader` 实际上是一次性读取大块文件，然后逐行返回该段的原因。结果是优化了文件读取，我们甚至没有注意到或关心后台发生了什么，这非常方便。

`File::create()` 如果文件不存在则创建新文件，否则截断文件。在任何情况下，它都返回与 `File::open()` 之前相同的 `File` 处理符。另一个相似之处在于我们围绕它包装的 `BufWriter`。就像 `BufReader` 一样，即使没有它我们也能访问底层文件，但使用它来优化未来的访问也是有益的。

除了以只读或截断模式打开文件之外，还有更多选项。我们可以通过使用 `OpenOptions` [69] 创建文件句柄来使用它们，这使用了我们在第一章 *学习基础知识* 中探索的 *使用构建器模式* 部分中的构建器模式。在我们的示例中，我们对 `append` 模式感兴趣，它允许我们在每次访问时而不是覆盖它来向文件添加内容。

要查看所有可用选项的完整列表，请参阅 OpenOption 文档：

[`doc.rust-lang.org/std/fs/struct.OpenOptions.html`](https://doc.rust-lang.org/std/fs/struct.OpenOptions.html)。

我们可以在同一个文件处理符上进行读写操作。为此，在创建 `ReadBuf` 和 `WriteBuf` 时，我们传递文件的一个引用而不是移动它，因为否则缓冲区会消耗处理符，使得共享变得不可能：

```rs
let mut buf_reader = BufReader::new(&file);
let mut buf_writer = BufWriter::new(&file);
```

在进行此操作时，请注意在同一个句柄上进行追加和读取。当追加时，存储当前读取位置的内部指针可能会偏移。如果你想先读取，然后追加，然后继续读取，你应该在写入之前保存当前位置，然后在之后恢复它。

我们可以通过调用`seek(SeekFrom::Current(0))`[89]来访问文件中的当前位置。`seek`通过一定数量的字节移动我们的指针并返回其新位置。`SeekFrom::Current(0)`意味着我们想要移动的距离正好是当前位置的零字节远。由于这个原因，因为我们根本不移动，所以`seek`将返回我们的当前位置。

然后，我们使用`flush`[92]来追加我们的数据。我们必须调用这个方法，因为`BufWriter`通常会等待实际写入直到它被丢弃，也就是说，它不再在作用域内。由于我们想在发生之前读取，我们使用`flush`来强制写入。

最后，我们通过从之前的位置恢复，再次进行定位，准备好再次读取：

```rs
buf_reader.seek(SeekFrom::Start(pos))?;
```

我邀请您运行代码，查看结果，然后与注释掉此行后的输出进行比较。

# 还有更多...

我们可以在程序开始时打开一个单独的文件句柄，并将其传递给需要它的每个函数，而不是在每个函数中打开一个新的文件句柄。这是一个权衡——如果我们不反复锁定和解锁文件，我们会获得更好的性能。反过来，我们不允许其他进程在我们程序运行时访问我们的文件。

# 另请参阅

+   *使用构建者模式*的配方在第一章，*学习基础知识*

# 处理字节

当您设计自己的协议或使用现有的协议时，您必须能够舒适地移动和操作二进制缓冲区。幸运的是，扩展的标准库生态系统提供了`byteorder`包，以满足您所有二进制需求的各种读写功能。

# 准备就绪

在本章中，我们将讨论*端序性*。这是描述缓冲区中值排列顺序的一种方式。有两种方式来排列它们：

+   首先放最小的数字（*小端序*）

+   首先放最大的一个（*大端序*）

让我们尝试一个例子。假设我们想要保存十六进制值`0x90AB12CD`。我们首先必须将其分成`0x90`、`0xAB`、`0x12`和`0xCD`的位。现在我们可以按最大的值首先存储它们（大端序），`0x90 - 0xAB - 0x12 - 0xCD`，或者我们可以先写最小的数字（小端序），`0xCD - 0x12 - 0xAB - 0x90`。

如您所见，这是一组完全相同的值，但顺序相反。如果这个简短的说明让您感到困惑，我建议您查看马里兰大学计算机科学系的这篇优秀解释：[`web.archive.org/web/20170808042522/http://www.cs.umd.edu/class/sum2003/cmsc311/Notes/Data/endian.html.`](https://web.archive.org/web/20170808042522/http://www.cs.umd.edu/class/sum2003/cmsc311/Notes/Data/endian.html)

没有更好的字节序。它们在不同的领域中被使用：例如，Intel 这样的微处理器使用 Little Endian，而 TCP、IPv4、IPv6 和 UDP 这样的互联网协议使用 Big Endian。这不是规则，而是一种为了保持向后兼容而维护的约定。因此，存在例外。

当设计您自己的协议时，请根据类似协议的字节序来定位，选择一个并简单地坚持使用它。

# 如何做到这一点...

按照以下步骤：

1.  打开之前为您生成的 `Cargo.toml` 文件。

1.  在 `[dependencies]` 下，添加以下行：

```rs
byteorder = "1.1.0"
```

1.  如果您愿意，可以访问 `byteorder` 的 crates.io 页面 ([`crates.io/crates/byteorder`](https://crates.io/crates/byteorder)) 检查最新版本，并使用该版本。

1.  在 `bin` 文件夹中，创建一个名为 `bytes.rs` 的文件。

1.  添加以下代码，并使用 `cargo run --bin bytes` 运行它：

```rs
1    extern crate byteorder;
2    use std::io::{Cursor, Seek, SeekFrom};
3    use byteorder::{BigEndian, LittleEndian, ReadBytesExt,
     WriteBytesExt};
4 
5    fn main() {
6      let binary_nums = vec![2, 3, 12, 8, 5, 0];
7      // Wrap a binary collection in a cursor
8      // to provide seek functionality
9      let mut buff = Cursor::new(binary_nums);
10     let first_byte = buff.read_u8().expect("Failed to read
         byte");
11     println!("first byte in binary: {:b}", first_byte);
12 
13     // Reading advances the internal position,
14     // so now we read the second
15     let second_byte_as_int = buff.read_i8().expect("Failed to
         read byte as int");
16     println!("second byte as int: {}", second_byte_as_int);
17 
18     // Overwrite the current position
19     println!("Before: {:?}", buff);
20     buff.write_u8(123).expect("Failed to overwrite a byte");
21     println!("After: {:?}", buff);
22 
23 
24     // Set and get the current position
25     println!("Old position: {}", buff.position());
26     buff.set_position(0);
27     println!("New position: {}", buff.position());
28 
29     // This also works using the Seek API
30     buff.seek(SeekFrom::End(0)).expect("Failed to seek end");
31     println!("Last position: {}", buff.position());
32 
33     // Read and write in specific endianness
34     buff.set_position(0);
35     let as_u32 = buff.read_u32::<LittleEndian>()
36       .expect("Failed to read bytes");
37     println!(
38       "First four bytes as u32 in little endian order:\t{}",
39        as_u32
40     );
41 
42     buff.set_position(0);
43     let as_u32 = buff.read_u32::<BigEndian>().expect("Failed to
         read bytes");
44     println!("First four bytes as u32 in big endian order:\t{}", 
         as_u32);
45 
46     println!("Before appending: {:?}", buff);
47     buff.seek(SeekFrom::End(0)).expect("Failed to seek end");
48     buff.write_f32::<LittleEndian>(-33.4)
49       .expect("Failed to write to end");
50     println!("After appending: {:?}", buff);
51 
52     // Read a sequence of bytes into another buffer
53     let mut read_buffer = [0; 5];
54     buff.set_position(0);
55     buff.read_u16_into::<LittleEndian>(&mut read_buffer)
56       .expect("Failed to read all bytes");
57     println!(
58       "All bytes as u16s in little endian order: {:?}",
59        read_buffer
60     );
61   }
```

# 它是如何工作的...

首先，我们需要一个二进制源。在我们的例子中，我们简单地使用一个向量。然后，我们将它包装到一个 `Cursor` 中，因为它为我们提供了一个 `Seek` 实现和一些方便的方法。

`Cursor` 有一个内部位置计数器，用于跟踪我们此刻正在访问的字节。正如预期的那样，它从零开始。使用 `read_u8` 和 `read_i8`，我们可以将当前字节读取为无符号或带符号的数字。这将使位置前进一个字节。这两个函数做的是同一件事，但返回的类型不同。

您注意到我们没有使用 `{:b}` 作为格式化参数 [11] 来打印返回的字节吗？

```rs
println!("first byte in binary: {:b}", first_byte);
```

通过这样做，我们告诉底层的 `format!` 宏将我们的字节解释为二进制，这就是为什么它会打印 `10` 而不是 `2`。如果您愿意，尝试在我们的其他打印调用中将 `{}` 替换为 `{:b}` 并比较结果。

当前位置可以通过 `position()` [25] 读取，并通过 `set_position()` 设置。您还可以使用我们在上一道菜谱中介绍的更详细的 `Seek` API 来操作您的位置 [30]。当使用 `SeekFrom::End` 时，请记住这不会从末尾开始倒计数。例如，`SeekFrom::End(1)` 将指向缓冲区末尾之后的一个字节，而不是之前。这种行为是这样定义的，因为，也许有些令人惊讶，越过缓冲区是合法的。这在写入时可能很有用，因为它将简单地用零填充缓冲区末尾和光标位置之间的空间。

当处理多个字节时，您需要通过类型注解指定字节序。读取或写入将根据读取或写入的字节数前进位置，这就是为什么在我们的示例代码中，我们需要频繁地使用 `set_position(0)` 重置位置。请注意，当您写入到末尾时，您总是会简单地扩展缓冲区 [48]。

如果您知道您想要读取一个非常特定的字节数，比如在解析一个定义良好的协议时，您可以通过提供一个固定大小的数组并通过在 `read` 后添加 `_into` 来填充它来实现这一点，如下所示：

```rs
// Read exactly five bytes
let mut read_buffer = [0; 5];
buff.read_u16_into::<LittleEndian>(&mut read_buffer).expect("Failed to fill buffer");
```

在这样做的时候，如果缓冲区没有完全填满，读取将返回一个错误，在这种情况下，其内容是未定义的。

# 还有更多...

在字节序 crate 中有各种别名，可以简化您的字节序注释。`BE`别名，表示大端，和`LE`别名，表示小端，如果您不想输入太多，它们很有用。另一方面，如果您经常忘记在哪里使用哪种字节序，您可以使用`NativeEndian`，它会设置为操作系统的默认字节序，以及`NetworkEndian`，用于大端。

要使用它们，您必须像这样将它们拖入作用域：

```rs
use byteorder::{BE, LE, NativeEndian, NetworkEndian};
```

# 处理二进制文件

我们现在将结合在前两章中学到的知识，以便解析和编写二进制文件。当你计划实现自定义、手动处理文件类型，如 PDF、种子文件和 ZIP 文件时，这将变得至关重要。在设计适用于您自己用例的自定义文件类型时，它也会派上用场。

# 如何做...

1.  如果您在上一个章节中还没有做，请打开之前为您生成的`Cargo.toml`文件。

1.  在`[dependencies]`部分下，添加以下行：

```rs
byteorder = "1.1.0"
```

1.  如果您愿意，您可以访问 byteorder 的 crates.io 页面([`crates.io/crates/byteorder`](https://crates.io/crates/byteorder))，检查最新版本，并使用那个版本。

1.  在`bin`文件夹中，创建一个名为`binary_files.rs`的文件。

1.  添加以下代码，并用`cargo run --bin binary_files`运行它：

```rs
1    extern crate byteorder;
2    use byteorder::{ByteOrder, ReadBytesExt, WriteBytesExt, BE,
     LE};
3    use std::fs::File;
4    use std::io::{self, BufReader, BufWriter, Read};
5    use std::io::prelude::*;
6 
7 
8    fn main() {
9      let path = "./bar.bin";
10     write_dummy_protocol(path).expect("Failed write file");
11     let payload = read_protocol(path).expect("Failed to read
         file");
12     print!("The protocol contained the following payload: ");
13     for num in payload {
14       print!("0x{:X} ", num);
15     }
16     println!();
17   }
```

1.  创建一个二进制文件：

```rs
19    // Write a simple custom protocol
20    fn write_dummy_protocol(path: &str) -> io::Result<()> {
21      let file = File::create(path)?;
22      let mut buf_writer = BufWriter::new(file);
23 
24      // Let's say our binary file starts with a magic string
25      // to show readers that this is our protocoll
26      let magic = b"MyProtocol";
27      buf_writer.write_all(magic)?;
28 
29      // Now comes another magic value to indicate
30      // our endianness
31      let endianness = b"LE";
32      buf_writer.write_all(endianness)?;
33 
34      // Let's fill it with two numbers in u32
35      buf_writer.write_u32::<LE>(0xDEAD)?;
36      buf_writer.write_u32::<LE>(0xBEEF)?;
37 
38      Ok(())
39    }
```

1.  读取和解析文件：

```rs
42   fn read_protocol(path: &str) -> io::Result<Vec<u32>> {
43     let file = File::open(path)?;
44     let mut buf_reader = BufReader::new(file);
45 
46     // Our protocol has to begin with a certain string
47     // Namely "MyProtocol", which is 10 bytes long
48     let mut start = [0u8; 10];
49     buf_reader.read_exact(&mut start)?;
50     if &start != b"MyProtocol" {
51       return Err(io::Error::new(
52         io::ErrorKind::Other,
53         "Protocol didn't start with the expected magic string",
54       ));
55     }
56 
57     // Now comes the endianness indicator
58     let mut endian = [0u8; 2];
59     buf_reader.read_exact(&mut endian)?;
60     match &endian {
61       b"LE" => read_protocoll_payload::<LE, _>(&mut buf_reader),
62       b"BE" => read_protocoll_payload::<BE, _>(&mut buf_reader),
63       _ => Err(io::Error::new(
64         io::ErrorKind::Other,
65         "Failed to parse endianness",
66       )),
67     }
68   }
69 
70   // Read as much of the payload as possible
71   fn read_protocoll_payload<E, R>(reader: &mut R) ->
       io::Result<Vec<u32>>
72   where
73   E: ByteOrder,
74   R: ReadBytesExt,
75   {
76     let mut payload = Vec::new();
77     const SIZE_OF_U32: usize = 4;
78     loop {
79     let mut raw_payload = [0; SIZE_OF_U32];
80     // Read the next 4 bytes
81     match reader.read(&mut raw_payload)? {
82     // Zero means we reached the end
83     0 => return Ok(payload),
84     // SIZE_OF_U32 means we read a complete number
85     SIZE_OF_U32 => {
86       let as_u32 = raw_payload.as_ref().read_u32::<E>()?;
87       payload.push(as_u32)
88     }
89     // Anything else means the last element was not
90     // a valid u32
91     _ => {
92       return Err(io::Error::new(
93       io::ErrorKind::UnexpectedEof,
94         "Payload ended unexpectedly",
95       ))
96     }
97   }
98  }
99 }
```

# 它是如何工作的...

为了演示如何读取和写入二进制文件，我们将创建一个简单的自定义二进制协议。它将以所谓的*魔数*开始，即某个硬编码的值。我们的魔数将是`MyProtocol`字符串的二进制表示。我们可以在字符串前加上`b`来告诉 Rust，我们希望文本以二进制切片(`&[u8]`)的形式表示，而不是字符串切片(`&str`) [26]。

许多协议和文件以魔数开始，以指示它们是什么。例如，`.zip`文件的内部头以魔数十六进制数`0x50`和`0x4B`开始。这些代表 ASCII 中的首字母*PH*，是创建者 Phil Katz 名字的缩写。另一个例子是 PDF；它以`0x25`、`0x50`、`0x44`和`0x46`开始，代表`PDF%`，后面跟着版本号。

之后，我们接着使用`LE`或`BE`的二元表示来告诉读取者其余数据的字节序 [31]。最后，我们有有效载荷，它只是任意数量的`u32`数字，按照上述字节序编码 [35 和 36]。通过在我们的数字前加上`0x`，我们告诉 Rust 将其视为十六进制数，并将其转换为十进制。因此，Rust 将`0xDEAD`视为与`57005`相同的值。

让我们把所有这些都放在一起，并写入一个包含`MyProtocolLE5700548879`的二进制文件。根据我们的协议，我们还可以创建其他文件，例如`MyProtocolBE92341739241842518425`或`MyProtocolLE0000`等。

如果你阅读了之前的食谱，`write_dummy_protocol`应该很容易理解。我们使用标准库中的古老的好`write_all`来写入我们的二进制文本，以及`byteorder`中的`write_u32`来写入需要端序的值。

协议的读取被分为`read_protocol`和`read_protocol_payload`函数。第一个通过读取魔数来验证协议的有效性，然后调用后者，后者读取剩余的数字作为有效载荷。

我们如下验证魔数：

1.  正如我们所知，我们知道了用于魔数的确切大小，因此准备相应大小的缓冲区。

1.  用同样数量的字节填充它们。

1.  将字节与预期的魔数进行比较。

1.  如果它们不匹配，则返回错误。

解析完两个魔数后，我们可以解析有效载荷中包含的实际*数据*。记住，我们将其定义为任意数量的`32`位（`= 4`字节）长，无符号数。为了解析它们，我们将反复读取最多四个字节到名为`raw_payload`的缓冲区中。然后我们将检查实际读取的字节数。在我们的情况下，这个数字可以有以下三种形式，正如我们的`match`所优雅地展示的那样。

我们感兴趣的第一个值是零，这意味着没有更多的字节可以读取，也就是说，我们已经到达了末尾。在这种情况下，我们可以返回我们的有效载荷：

```rs
// Zero means we reached the end
0 => return Ok(payload),
```

第二个值是`SIZE_OF_U32`，我们之前将其定义为四。接收到这个值意味着我们的读取器已经成功地将四个字节读入一个四字节长的缓冲区中。这意味着我们已经成功读取了一个值！让我们将其解析为`u32`并将其推送到我们的`payload`向量中：

```rs
// SIZE_OF_U32 means we read a complete number
SIZE_OF_U32 => {
    let as_u32 = raw_payload.as_ref().read_u32::<E>()?;
    payload.push(as_u32)
}
```

我们必须在我们的缓冲区上调用`as_ref()`，因为固定大小的数组没有实现`Read`。由于切片实现了该特性和数组引用隐式可转换为切片，我们工作在`raw_payload`的引用上。

我们可以预期的第三个和最后一个值是除了零或四之外的所有值。在这种情况下，读取器无法读取四个字节，这意味着我们的缓冲区在`u32`之外结束，并且格式不正确。我们可以通过返回错误来对此做出反应：

```rs
// Anything else means the last element was not
// a valid u32
_ => {
    return Err(io::Error::new(
        io::ErrorKind::UnexpectedEof,
        "Payload ended unexpectedly",
        ))
    }
}
```

# 还有更多...

当读取格式不正确的协议时，我们重用`io::ErrorKind`来显示到底出了什么问题。在第六章的食谱中，*处理错误*，你将学习如何提供自己的错误以更好地分离你的失败区域。如果你想，你可以现在阅读它们，然后返回这里来改进我们的代码。

需要推送到自己的变体的错误是：

+   `InvalidStart`

+   `InvalidEndianness`

+   `UnexpectedEndOfPayload`

代码的另一个改进是将我们所有的字符串，即`MyProtocol`、`LE`和`BE`，放入它们自己的常量中，如下行所示：

```rs
const PROTOCOL_START: &[u8] = b"MyProtocol";
```

在这个配方和某些其他配方中提供的代码不使用很多常量，因为它们在打印形式中证明是有些难以理解的。然而，在实际代码库中，务必始终将你发现自己复制粘贴的字符串放入自己的常量中！

# 参见

+   在[第六章](https://cdp.packtpub.com/rust_standard_library_cookbook/wp-admin/post.php?post=81&action=edit#post_151)中提供用户定义的错误类型*配方*，*处理错误*

# 压缩和解压缩数据

在今天这个网站臃肿和每天出现新的 Web 框架的时代，许多网站感觉比它们实际使用（和应该使用）的要慢得多。减轻这种情况的一种方法是在发送资源之前压缩它们，然后在收到时解压缩。这已经成为（通常被忽视的）网络标准。为此，这个配方将教你如何使用不同的算法压缩和解压缩任何类型的数据。

# 如何做到...

按照以下步骤操作：

1.  打开之前为你生成的`Cargo.toml`文件。

1.  在`[dependencies]`下添加以下行：

    ```rs
     flate2 = "0.2.20"
    ```

    如果你想，你可以去`flate2`的 crates.io 页面([`crates.io/crates/flate2`](https://crates.io/crates/flate2))查看最新版本，并使用那个版本。

1.  在`bin`文件夹中，创建一个名为`compression.rs`的文件。

1.  添加以下代码，并用`cargo run --bin compression`运行它：

```rs
1    extern crate flate2;
2 
3    use std::io::{self, SeekFrom};
4    use std::io::prelude::*;
5 
6    use flate2::{Compression, FlateReadExt};
7    use flate2::write::ZlibEncoder;
8    use flate2::read::ZlibDecoder;
9 
10   use std::fs::{File, OpenOptions};
11   use std::io::{BufReader, BufWriter, Read};
12 
13   fn main() {
14     let bytes = b"I have a dream that one day this nation will
         rise up, \
15     and live out the true meaning of its creed";
16     println!("Original: {:?}", bytes.as_ref());
17     // Conpress some bytes
18     let encoded = encode_bytes(bytes.as_ref()).expect("Failed to
         encode bytes");
19     println!("Encoded: {:?}", encoded);
20     // Decompress them again
21     let decoded = decode_bytes(&encoded).expect("Failed to decode
         bytes");
22     println!("Decoded: {:?}", decoded);
23 
24     // Open file to compress
25     let original = File::open("ferris.png").expect("Failed to
         open file");
26     let mut original_reader = BufReader::new(original);
27 
28     // Compress it
29     let data = encode_file(&mut original_reader).expect("Failed
         to encode file");
30 
31     // Write compressed file to disk
32     let encoded = OpenOptions::new()
33       .read(true)
34       .write(true)
35       .create(true)
36       .open("ferris_encoded.zlib")
37       .expect("Failed to create encoded file");
38     let mut encoded_reader = BufReader::new(&encoded);
39     let mut encoded_writer = BufWriter::new(&encoded);
40     encoded_writer
41       .write_all(&data)
42       .expect("Failed to write encoded file");
43 
44 
45     // Jump back to the beginning of the compressed file
46     encoded_reader
47       .seek(SeekFrom::Start(0))
48       .expect("Failed to reset file");
49 
50     // Decompress it
51     let data = decode_file(&mut encoded_reader).expect("Failed to
         decode file");
52 
53     // Write the decompressed file to disk
54     let mut decoded =
         File::create("ferris_decoded.png").expect("Failed to create
           decoded file");
55     decoded
56       .write_all(&data)
57       .expect("Failed to write decoded file");
58   }
```

1.  这些是执行实际编码和解码的函数：

```rs
61   fn encode_bytes(bytes: &[u8]) -> io::Result<Vec<u8>> {
62     // You can choose your compression algorithm and it's
         efficiency
63     let mut encoder = ZlibEncoder::new(Vec::new(),
         Compression::Default);
64     encoder.write_all(bytes)?;
65     encoder.finish()
66   }
67 
68   fn decode_bytes(bytes: &[u8]) -> io::Result<Vec<u8>> {
69     let mut encoder = ZlibDecoder::new(bytes);
70     let mut buffer = Vec::new();
71     encoder.read_to_end(&mut buffer)?;
72     Ok(buffer)
73   }
74 
75 
76   fn encode_file(file: &mut Read) -> io::Result<Vec<u8>> {
77     // Files have a built-in encoder
78     let mut encoded = file.zlib_encode(Compression::Best);
79     let mut buffer = Vec::new();
80     encoded.read_to_end(&mut buffer)?;
81     Ok(buffer)
82   }
83 
84   fn decode_file(file: &mut Read) -> io::Result<Vec<u8>> {
85     let mut buffer = Vec::new();
86     // Files have a built-in decoder
87     file.zlib_decode().read_to_end(&mut buffer)?;
88     Ok(buffer)
89   }
```

# 它是如何工作的...

大多数`main`函数的工作只是重复你从上一章学到的内容。真正的操作发生在下面。在`encode_bytes`中，你可以看到如何使用*编码器*。你可以尽可能多地写入它，并在完成后调用`finish`。

`flate2`为你提供了几个压缩选项。你可以通过传递的`Compression`实例来选择你的压缩强度：

```rs
let mut encoder = ZlibEncoder::new(Vec::new(), Compression::Default);
```

`Default`是在速度和大小之间的折衷。你的其他选项是`Best`、`Fast`和`None`。此外，你可以指定使用的编码算法。`flate2`支持 zlib，我们在本配方中使用，gzip 和平滑的 deflate。如果你想使用除 zlib 之外的算法，只需将所有提及它的地方替换为另一个支持的算法。例如，如果你想将前面的代码重写为使用 gzip，它看起来会像这样：

```rs
use flate2::write::GzEncoder;
let mut encoder = GzEncoder::new(Vec::new(), Compression::Default);
```

有关特定编码器如何调用的完整列表，请访问`flate2`的文档[`docs.rs/flate2/`](https://docs.rs/flate2/)。

由于人们通常会倾向于压缩或解压缩整个文件而不是字节缓冲区，因此有一些方便的方法可以做到这一点。实际上，它们是在实现了`Read`接口的每个类型上实现的，这意味着你也可以在`BufReader`和许多其他类型上使用它们。`encode_file`和`decode_file`使用以下行中的`zlib`：

```rs
let mut encoded = file.zlib_encode(Compression::Best);
file.zlib_decode().read_to_end(&mut buffer)?;
```

这同样适用于 `gzip` 和 `deflate` 算法。

在我们的例子中，我们正在压缩和解压缩 `ferris.png`，这是 Rust 面具的图片：

![图片](img/11ce3c14-a708-46ef-b5d4-7250ea181f6b.jpg)

你可以在 GitHub 仓库 [`github.com/SirRade/rust-standard-library-cookbook`](https://github.com/SirRade/rust-standard-library-cookbook/tree/master/chapter_three) 中找到它，或者你可以使用任何你想要的文件。如果你想要验证压缩效果，你可以查看原始文件、压缩文件和解压缩文件，以检查压缩文件有多小，以及原始文件和解压缩文件是否相同。

# 还有更多...

当前的 `encode_something` 和 `decode_something` 函数被设计得尽可能简单易用。然而，尽管我们可以直接将数据管道输入到写入器中，它们仍然通过分配 `Vec<u8>` 来浪费一些性能。当编写库时，通过这种方式添加方法，为用户提供两种可能性会很好：

```rs
use std::io::Write;
fn encode_file_into(file: &mut Read, target: &mut Write) -> io::Result<()> {
    // Files have a built-in encoder
    let mut encoded = file.zlib_encode(Compression::Best);
    io::copy(&mut encoded, target)?;
    Ok(())
}
```

用户可以像这样调用它们：

```rs
// Compress it
encode_file_into(&mut original_reader, &mut encoded_writer)
    .expect("Failed to encode file");
```

# 遍历文件系统

到目前为止，我们总是为我们的代码提供某个文件的静态位置。然而，现实世界很少如此可预测，当处理散布在不同文件夹中的数据时，需要进行一些挖掘。

`walkdir` 通过将操作系统的文件系统表示的复杂性和不一致性统一到一个公共 API 下，帮助我们抽象化这些问题，我们将在本配方中学习这个 API。

# 准备工作

这个配方大量使用了迭代器来操作数据流。如果你还不熟悉它们或需要快速复习，你应该在继续之前阅读第二章 *使用集合* 中的 *将集合作为迭代器访问* 部分。

# 如何做...

1.  打开为你生成的 `Cargo.toml` 文件。

1.  在 `[dependencies]` 下，添加以下行：

```rs
walkdir = "2.0.1"
```

1.  如果你想，你可以去 `walkdir` 的 crates.io 页面（[`crates.io/crates/walkdir`](https://crates.io/crates/walkdir)）查看最新版本，并使用那个版本。

1.  在 `bin` 文件夹中，创建一个名为 `traverse_files.rs` 的文件。

1.  添加以下代码，并使用 `cargo run --bin traverse_files` 运行它：

```rs
1    extern crate walkdir;
2    use walkdir::{DirEntry, WalkDir};
3 
4    fn main() {
5      println!("All file paths in this directory:");
6      for entry in WalkDir::new(".") {
7        if let Ok(entry) = entry {
8          println!("{}", entry.path().display());
9        }
10     }
11 
12     println!("All non-hidden file names in this directory:");
13     WalkDir::new("../chapter_three")
14       .into_iter()
15       .filter_entry(|entry| !is_hidden(entry)) // Look only at 
          non-hidden enthries
16       .filter_map(Result::ok) // Keep all entries we have access to
17       .for_each(|entry| {
18         // Convert the name returned by theOS into a Rust string
19         // If there are any non-UTF8 symbols in it, replace them 
              with placeholders
20         let name = entry.file_name().to_string_lossy();
21           println!("{}", name)
22       });
23 
24       println!("Paths of all subdirectories in this directory:");
25       WalkDir::new(".")
26         .into_iter()
27         .filter_entry(is_dir) // Look only at directories
28         .filter_map(Result::ok) // Keep all entries we have 
            access to
29         .for_each(|entry| {
30           let path = entry.path().display();
31           println!("{}", path)
32         });
33 
34       let are_any_readonly = WalkDir::new("..")
35         .into_iter()
36         .filter_map(Result::ok) // Keep all entries we have 
            access to
37         .filter(|e| has_file_name(e, "vector.rs")) // Get the 
            ones with a certain name
38         .filter_map(|e| e.metadata().ok()) // Get metadata if the 
            OS allows it
39         .any(|e| e.permissions().readonly()); // Check if at 
            least one entry is readonly
40       println!(
41         "Are any the files called 'vector.rs' readonly? {}",
42          are_any_readonly
43       );
44 
45      let total_size = WalkDir::new(".")
46        .into_iter()
47        .filter_map(Result::ok) // Keep all entries we have access 
           to
48        .filter_map(|entry| entry.metadata().ok()) // Get metadata
           if supported
49        .filter(|metadata| metadata.is_file()) // Keep all files
50        .fold(0, |acc, m| acc + m.len()); // Accumulate sizes
51 
52      println!("Size of current directory: {} bytes", total_size);
53    }
```

1.  现在，让我们来看看这个配方中使用的谓词：

```rs
55   fn is_hidden(entry: &DirEntry) -> bool {
56     entry
57       .file_name()
58       .to_str()
59       .map(|s| s.starts_with('.'))
60       .unwrap_or(false) // Return false if the filename is 
          invalid UTF8
61   }
62 
63   fn is_dir(entry: &DirEntry) -> bool {
64     entry.file_type().is_dir()
65   }
66 
67   fn has_file_name(entry: &DirEntry, name: &str) -> bool {
68     // Check if file name contains valid unicode
69     match entry.file_name().to_str() {
70     Some(entry_name) => entry_name == name,
71     None => false,
72   }
73 }
```

# 它是如何工作的...

`walkdir` 由三个重要的类型组成：

+   `WalkDir`：一个构建器（参见第一章 *使用构建器模式* 部分，*学习基础知识*），用于你的目录遍历器

+   `IntoIter`：由构建器创建的迭代器

+   `DirEntry`：表示单个文件夹或文件

如果你只想操作根文件夹下的所有条目列表，例如在行 [6] 的第一个例子中，你可以隐式地直接将 `WalkDir` 作为 `DirEntry` 不同实例的迭代器使用：

```rs
for entry in WalkDir::new(".") {
    if let Ok(entry) = entry {
        println!("{}", entry.path().display());
    }
}
```

如你所见，迭代器并不直接给你一个 `DirEntry`，而是一个 `Result`。这是因为有些情况下访问文件或文件夹可能会很困难。例如，操作系统可能禁止你读取文件夹的内容，隐藏其中的文件。或者一个符号链接，你可以通过在 `WalkDir` 实例上调用 `follow_links(true)` 来启用它，它可能指向父目录，这可能导致无限循环。

对于这个配方中的错误，我们的解决方案策略很简单——我们只是忽略它们，并继续处理没有报告任何问题的其他条目。

当你提取实际的条目时，它可以告诉你很多关于它自己的信息。其中之一就是它的路径。记住，尽管 `.path()` [8] 并不仅仅返回路径作为一个字符串。实际上，它返回一个本地的 Rust `Path` 结构体，可以用于进一步分析。例如，你可以通过在它上面调用 `.extension()` 来读取文件路径的扩展名。或者你可以通过调用 `.parent()` 来获取它的父目录。你可以通过探索 [`doc.rust-lang.org/std/path/struct.Path.html`](https://doc.rust-lang.org/std/path/struct.Path.html) 上的 `Path` 文档来自由探索可能性。在我们的案例中，我们只是通过调用 `.display()` 来将其显示为一个简单的字符串。

当我们使用 `into_iter()` 显式地将 `WalkDir` 转换为迭代器时，我们可以访问一个其他迭代器没有的特殊方法：`filter_entry`。它是对 `filter` 的优化，因为它在遍历期间被调用。当它的谓词对一个目录返回 `false` 时，遍历者根本不会进入该目录！这样，在遍历大型文件系统时，你可以获得很多性能提升。在配方中，我们在寻找非隐藏文件 [15] 时使用了它。如果你只需要操作文件而永远不操作目录，你应该使用普通的 `filter`。

我们根据 Unix 习惯定义 *隐藏文件* 为所有以点开头的目录和文件。因此，它们有时也被称为 *点文件*。

在这两种情况下，你的过滤都需要一个谓词。它们通常被放在自己的函数中，以保持简单性和可重用性。

注意，`walkdir` 并不仅仅给我们一个普通的字符串形式的文件名。相反，它返回一个 `OsStr`。这是一种特殊的字符串，当 Rust 直接与操作系统交互时，Rust 会使用这种字符串。这种类型的存在是因为一些操作系统允许文件名中包含无效的 UTF-8。在 Rust 中查看这类文件时，你有两种选择——让 Rust 尝试将它们转换为 UTF-8，并用 Unicode 替换字符（�）替换所有无效字符，或者自行处理错误。你可以通过在 `OsStr` 上调用 `to_string_lossy` 来选择第一种方法 [20]。第二种方法可以通过调用 `to_str` 并检查返回的 `Option` 来访问，就像我们在 `has_file_name` 中做的那样，我们只是简单地丢弃无效的名称。

在这道菜谱中，你可以看到一个精彩的例子，说明何时选择`for_each`方法调用（在第一章的*将集合作为迭代器访问*部分讨论过，*学习基础知识*，*处理集合*）而不是`for`循环——我们的大部分迭代器调用都是链在一起的，因此`for_each`调用可以自然地链接到迭代器中。

# 还有更多...

如果你计划仅在 Unix 上发布你的应用程序，你可以通过`.metadata().unwrap().permissions()`调用在条目上访问额外的权限。具体来说，你可以通过调用`.mode()`来查看确切的`st_mode`位，并通过调用`set_mode()`使用一组新的位来更改它们。

# 相关内容

+   [第一章](https://cdp.packtpub.com/rust_standard_library_cookbook/wp-admin/post.php?post=81&action=edit#post_24)的*使用构建器模式*菜谱，*学习基础知识*

+   [第二章](https://cdp.packtpub.com/rust_standard_library_cookbook/wp-admin/post.php?post=81&action=edit#post_47)的*将集合作为迭代器访问*菜谱，*处理集合*

# 使用 glob 模式查找文件

如你所注意到的，使用`walkdir`根据文件名过滤文件有时可能有点笨拙。幸运的是，你可以通过使用`glob`crate 来大大简化这一点，它将 Unix 中的标题模式直接带入 Rust。

# 如何做到这一点...

按以下步骤操作：

1.  打开之前为你生成的`Cargo.toml`文件。

1.  在`[dependencies]`下添加以下行：

```rs
glob = "0.2.11"
```

1.  如果你愿意，你可以访问 glob 的 crates.io 页面([`crates.io/crates/glob`](https://crates.io/crates/glob))，查看最新版本并使用它。

1.  在`bin`文件夹中，创建一个名为`glob.rs`的文件。

1.  添加以下代码，并使用`cargo run --bin glob`运行它：

```rs
1    extern crate glob;
2    use glob::{glob, glob_with, MatchOptions};
3 
4    fn main() {
5      println!("All all Rust files in all subdirectories:");
6      for entry in glob("**/*.rs").expect("Failed to read glob
       pattern") {
7        match entry {
8          Ok(path) => println!("{:?}", path.display()),
9          Err(e) => println!("Failed to read file: {:?}", e),
10       }
11     }
12 
13     // Set the glob to be case insensitive and ignore hidden
          files
14     let options = MatchOptions {
15       case_sensitive: false,
16       require_literal_leading_dot: true,
17       ..Default::default()
18     };
19 
20 
21     println!(
22       "All files that contain the word \"ferris\" case
          insensitive \
23        and don't contain an underscore:"
24     );
25     for entry in glob_with("*Ferris[!_]*",
       &options).expect("Failed to read glob pattern") {
26       if let Ok(path) = entry {
27         println!("{:?}", path.display())
28       }
29     }
30   }
```

# 它是如何工作的...

这个 crate 相当小且简单。使用`glob(...)`，你可以通过指定一个`glob`模式来创建一个迭代器，遍历所有匹配的文件。如果你不熟悉它们，但记得之前（在第一章的*使用正则表达式查询*部分中）的 regex 菜谱（*学习基础知识*），可以将它们视为非常简化的正则表达式，主要用于文件名。其语法在维基百科上有很好的描述：[`en.wikipedia.org/wiki/Glob_(programming)`](https://en.wikipedia.org/wiki/Glob_%28programming%29)。

与之前的`WalkDir`一样，`glob`迭代器返回一个`Result`，因为程序可能没有权限读取文件系统条目。在`Result`中包含一个`Path`，我们也在上一道菜谱中提到了它。如果你想读取文件内容，请参考本章的第一道菜谱，它涉及文件操作。

使用`glob_with`，你可以指定一个`MatchOptions`实例来更改`glob`搜索文件的方式。你可以切换的最有用的选项是：

+   `case_sensitive`：默认情况下是启用的，它控制是否应该将小写字母（abcd）和大写字母（ABCD）视为不同或相同。

+   `require_literal_leading_dot`：默认情况下是禁用的，当设置时，禁止通配符匹配文件名中的前导点。这用于您想忽略用户的隐藏文件时。

您可以在`MatchOption`的文档中查看其余选项：[`doc.rust-lang.org/glob/glob/struct.MatchOptions.html`](https://doc.rust-lang.org/glob/glob/struct.MatchOptions.html).

如果您已经设置了您关心的选项，您可以通过使用在第一章*学习基础知识*中*提供默认实现*部分讨论的`..Default::default()`更新语法，将剩余的选项保留为默认值。

# 参见

+   在第一章*学习基础知识*中的*使用正则表达式查询*和*提供默认实现*食谱
