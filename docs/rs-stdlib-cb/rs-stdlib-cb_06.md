# 第六章：处理错误

在本章中，我们将涵盖以下配方：

+   提供用户定义的错误类型

+   提供日志记录

+   创建自定义记录器

+   实现 Drop 特性

+   理解 RAII

# 简介

错误会发生，这是正常的。毕竟，我们都是凡人。生活和编程中的重要事情不是我们犯了什么错误，而是我们如何处理它们。Rust 通过为我们提供一个错误处理概念来帮助我们处理这一原则的编程方面，该概念保证我们在处理可能失败的函数时必须考虑失败的结果，因为它们不直接返回值，而是被包裹在一个必须以某种方式打开的`Result`中。唯一剩下的事情是以一种与这个概念很好地集成的代码方式设计我们的代码。

# 提供用户定义的错误类型

在前面的章节中，我们了解了几种处理必须处理不同类型错误的函数的方法。到目前为止，我们有：

+   遇到它们时直接崩溃

+   只返回一种错误，并将所有其他错误转换为它

+   在`Box`中返回不同类型的错误

大多数这些方法都已被使用，因为我们还没有达到这个配方。现在，我们将学习做事的优选方式，创建一种包含多个子错误的自定义`Error`类型。

# 如何做到这一点...

1.  使用`cargo new chapter-six`创建一个 Rust 项目，在本章中工作。

1.  导航到新创建的`chapter-six`文件夹。在本章的其余部分，我们将假设您的命令行当前位于此目录。

1.  在`src`文件夹内创建一个名为`bin`的新文件夹。

1.  删除生成的`lib.rs`文件，因为我们不是创建一个库。

1.  在`src/bin`文件夹中创建一个名为`custom_error.rs`的文件。

1.  添加以下代码，并使用`cargo run --bin custom_error`运行它：

```rs
1   use std::{error, fmt, io, num, result};
2   use std::fs::File;
3   use std::io::{BufReader, Read};
4 
5   #[derive(Debug)]
6   // This is going to be our custom Error type
7   enum AgeReaderError {
8     Io(io::Error),
9     Parse(num::ParseIntError),
10    NegativeAge(),
11  }
12 
13  // It is common to alias Result in an Error module
14  type Result<T> = result::Result<T, AgeReaderError>;
15 
16  impl error::Error for AgeReaderError {
17    fn description(&self) -> &str {
18      // Defer to the existing description if possible
19      match *self {
20      AgeReaderError::Io(ref err) => err.description(),
21      AgeReaderError::Parse(ref err) => err.description(),
22      // Descriptions should be as short as possible
23      AgeReaderError::NegativeAge() => "Age is negative",
24      }
25    }
26 
27    fn cause(&self) -> Option<&error::Error> {
28      // Return the underlying error, if any
29      match *self {
30        AgeReaderError::Io(ref err) => Some(err),
31        AgeReaderError::Parse(ref err) => Some(err),
32        AgeReaderError::NegativeAge() => None,
33      }
34    }
35  }
36 
37  impl fmt::Display for AgeReaderError {
38    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
39      // Write a detailed description of the problem
40      match *self {
41        AgeReaderError::Io(ref err) => write!(f, "IO error: {}", 
           err),
42        AgeReaderError::Parse(ref err) => write!(f, "Parse 
           error: {}", err),
43        AgeReaderError::NegativeAge() => write!(f, "Logic error: 
           Age cannot be negative"),
44      }
45    }
46  }
47 
48  // Implement From<T> for every sub-error
49  impl From<io::Error> for AgeReaderError {
50    fn from(err: io::Error) -> AgeReaderError {
51    AgeReaderError::Io(err)
52    }
53  }
54 
55  impl From<num::ParseIntError> for AgeReaderError {
56    fn from(err: num::ParseIntError) -> AgeReaderError {
57    AgeReaderError::Parse(err)
58    }
59  }
60 
61  fn main() {
62    // Assuming a file called age.txt exists
63    const FILENAME: &str = "age.txt";
64    let result = read_age(FILENAME);
65    match result {
66      Ok(num) => println!("{} contains the age {}", FILENAME, 
         num),
67      Err(AgeReaderError::Io(err)) => eprintln!("Failed to open 
         the file {}: {}", FILENAME, err),
68      Err(AgeReaderError::Parse(err)) => eprintln!(
69       "Failed to read the contents of {} as a number: {}",
70        FILENAME, err
71         ),
72      Err(AgeReaderError::NegativeAge()) => eprintln!("The age in 
         the file is negative"),
73      }
74  }
75 
76  // Read an age out of a file
77  fn read_age(filename: &str) -> Result<i32> {
78    let file = File::open(filename)?;
79    let mut buf_reader = BufReader::new(file);
80    let mut content = String::new();
81    buf_reader.read_to_string(&mut content)?;
82    let age: i32 = content.trim().parse()?;
83    if age.is_positive() {
84      Ok(age)
85    } else {
86      Err(AgeReaderError::NegativeAge())
87    }
88  }
```

# 它是如何工作的...

我们的示例目的是读取文件`age.txt`，并返回其中写下的数字，假设它代表某种年龄。在这个过程中，我们可能会遇到三种错误：

+   读取文件失败（可能它不存在）

+   读取其内容作为数字失败（它可能包含文本）

+   数字可能是负数

这些可能的错误状态是`Error enum`的可能变体：`AgeReaderError`[7]。通常，我们会根据它们所代表的子错误来命名变体。因为读取文件失败会引发`io::Error`，所以我们把对应的变体命名为`AgeReaderError::Io`[8]。将`&str`解析为`i32`失败会引发`num::ParseIntError`，所以我们把包含的变体命名为`AgeReaderError::Parse`[9]。

这两个`std`错误清楚地展示了错误命名的约定。如果你有一个模块可以返回许多不同的错误，通过它们的完整名称导出它们，例如`num::ParseIntError`。如果你的模块只返回一种`Error`，只需将其导出为`Error`，例如`io::Error`。我们故意不遵循这个约定，因为在配方中，独特的名称`AgeReaderError`使得讨论它更容易。如果这个配方逐个包含在一个 crate 中，我们可以通过将其导出为`pub type Error = AgeReaderError;`来实现传统效果。

我们接下来创建的是我们自己的`Result`[14]的别名：

```rs
type Result<T> = result::Result<T, AgeReaderError>;
```

这是你自己错误的一个极其常见的模式，这使得与它们一起工作变得非常愉快，正如我们在`read_age`的返回类型中看到的那样[77]：

```rs
fn read_age(filename: &str) -> Result<i32> { ... }
```

看起来不错，不是吗？然而，为了使用我们的`enum`作为`Error`，我们首先需要实现它[16]。`Error`特质需要两件事：一个`description`[17]，它是对发生错误的一个简短解释，以及一个`cause`[27]，它简单地是将错误重定向到底层错误（如果有的话）。你可以（并且应该）通过实现`Display`为你的`Error`[37]来提供关于当前问题的详细描述。在这些实现中，如果可能的话，你应该参考底层错误，就像以下行[20]所示：

```rs
AgeReaderError::Io(ref err) => err.description()
```

你需要为良好的自定义`Error`提供每个子错误的`From`实现。在我们的例子中，这将涉及`From<io::Error>`[49]和`From<num::ParseIntError>`[55]。这样，`try`操作符（`?`）将自动为我们转换涉及到的错误。

在实现所有必要的特性之后，你可以在任何函数中返回自定义的`Error`，并使用前面提到的操作符来解包其中的值。在这个例子中，当我们检查`read_age`的结果时，我们不需要`match`返回的值。在一个真正的`main`函数中，我们可能只是简单地调用`.expect("…")`，但我们仍然匹配了单独的错误变体，以便向您展示在使用已知的错误类型时，您如何优雅地应对不同的问题[65 到 73]：

```rs
match result {
    Ok(num) => println!("{} contains the age {}", FILENAME, num),
    Err(AgeReaderError::Io(err)) => eprintln!("Failed to open the file 
    {}: {}", FILENAME, err),
    Err(AgeReaderError::Parse(err)) => eprintln!(
        "Failed to read the contents of {} as a number: {}",
        FILENAME, err
    ),
    Err(AgeReaderError::NegativeAge()) => eprintln!("The age in the file is negative"),
}
```

# 还有更多...

为了组织原因，一个 crate 的`Error`通常被放在一个自定义的`error`模块中，然后直接导出以实现最佳可用性。相关的`lib.rs`条目可能看起来像这样：

```rs
mod error;
pub use error::Error;
```

# 提供日志记录

在一个大型应用程序中，事情迟早会不如预期。但没关系，只要你为用户提供了一个系统，让他们知道出了什么问题，如果可能的话，为什么，那就行了。一个经过时间考验的工具是详细的日志，它允许用户自己指定他们想要看到多少诊断信息。

# 如何做到这一点...

按照以下步骤操作：

1.  打开之前为你生成的`Cargo.toml`文件。

1.  在`[dependencies]`下添加以下行：

```rs
log = "0.4.1"
env_logger = "0.5.3"
```

1.  如果你想，你可以访问`log` ([`crates.io/crates/log`](https://crates.io/crates/log)) 或 `env_logger` ([`crates.io/crates/env_log`](https://crates.io/crates/env_log)) 的 crate.io 页面，检查最新版本并使用那个版本。

1.  在`bin`文件夹中创建一个名为`logging.rs`的文件。

1.  如果你使用的是基于 Unix 的系统，请添加以下代码并使用`RUST_LOG=logging cargo run --bin logging`运行它。否则，在 Windows 上运行`$env:RUST_LOG="logging"; cargo run --bin logging`：

```rs
1   extern crate env_logger;
2   #[macro_use]
3   extern crate log;
4   use log::Level;
5 
6   fn main() {
7     // env_logger's priority levels are:
8     // error > warn > info > debug > trace
9     env_logger::init();
10    // All logging calls log! in the background
11    log!(Level::Debug, "env_logger has been initialized");
12 
13    // There are convenience macros for every logging level 
       however
14    info!("The program has started!");
15 
16    // A log's target is its parent module per default 
17    // ('logging' in our case, as we're in a binary)
18    // We can override this target however:
19    info!(target: "extra_info", "This is additional info that 
       will only show if you \
20    activate info level logging for the extra_info target");
21 
22    warn!("Something that requires your attention happened");
23 
24    // Only execute code if logging level is active
25    if log_enabled!(Level::Debug) {
26    let data = expensive_operation();
27    debug!("The expensive operation returned: \"{}\"", data);
28    }
29 
30    error!("Something terrible happened!");
31  }
32 
33  fn expensive_operation() -> String {
34    trace!("Starting an expensive operation");
35    let data = "Imagine this is a very very expensive 
       task".to_string();
36    trace!("Finished the expensive operation");
37    data
38  }
```

# 它是如何工作的...

Rust 的日志系统基于`log` crate，它为所有日志事物提供了一个共同的*外观*。这意味着它实际上并不提供任何功能，只是提供了接口。实现留给其他 crate，在我们的例子中是`env_logger`。这种外观和实现的分离非常实用，因为任何人都可以创建一种新的、酷的日志方式，这会自动与任何 crate 兼容。

应由你代码的使用者来决定使用的日志实现。如果你编写了一个 crate，不要使用任何实现，而应仅通过`log` crate 来记录所有事物。然后，使用该 crate 的（或他人的）可执行文件可以简单地初始化他们选择的日志记录器[9]，以便实际处理日志调用。

`log` crate 提供了`log!`宏[11]，它接受一个日志`Level`，一个可以像`println!`一样格式化的消息，以及一个可选的`target`。你可以这样记录事物，但使用每个日志级别的便利宏（`error!`、`warn!`、`info!`、`debug!`和`trace!`）更易于阅读，这些宏在后台简单地调用`log!`。日志的`target`[19]是一个额外的属性，有助于日志实现根据主题分组日志。如果你省略了`target`，它默认为当前的`module`。例如，如果你从`foo` crate 记录了某些内容，它的`target`将默认为`foo`。如果你在其子模块`foo::bar`中记录了某些内容，它的`target`将默认为`bar`。如果你然后在`main.rs`中使用了该 crate 并记录了某些内容，它的`target`将默认为`main`。

`log`提供的另一个好处是`log_enabled!`宏，它返回当前活动的日志记录器是否设置为处理特定的警告级别。这在与提供有用信息的昂贵操作的成本下特别有用。

`env_logger` 是 Rust 库提供的日志记录实现。它在其日志上打印到 `stderr`，并在你的终端支持的情况下使用漂亮的颜色来表示不同的日志级别。它依赖于一个 `RUST_LOG` 环境变量来过滤应该显示哪些日志。如果你没有定义这个变量，它将默认为 `error`，这意味着它将只打印来自所有目标的 `error` 级别的日志。正如你所猜到的，其他可能的值包括 `warn`、`info`、`debug` 和 `trace`。然而，这些不仅会过滤指定的级别，还会过滤其 **以上** 的所有级别，其中层次结构定义如下：

```rs
error > warn > info > debug > trace
```

这意味着将你的 `RUST_LOG` 设置为 `warn` 将显示所有 `warn` 和 `error` 级别的日志。将其设置为 `debug` 将显示 `error`、`warn`、`info` 和 `debug` 级别的日志。

你可以将 `RUST_LOG` 设置为目标，而不是错误级别，这将显示所选目标的全部日志，无论它们的日志级别如何。这就是我们在示例中所做的，我们将 `RUST_LOG` 设置为 `logging` 以显示所有带有 `target` 调用 `logging` 的日志，这是我们二进制中所有日志的标准目标。如果你想的话，你可以像这样将级别过滤器与目标过滤器结合使用：`logging=warn`，这将只显示 `logging` 目标中的 `warn` 和 `error` 级别的日志。

你可以使用逗号组合不同的过滤器。如果你想显示此示例的所有日志，你可以将你的变量设置为 `logging,extra_info`，这将过滤 `logging` 和 `extra_info` 目标。

最后，你可以使用斜杠 (`/`) 后跟一个正则表达式来通过内容过滤日志，该正则表达式必须匹配。例如，如果你将 `RUST_LOG` 设置为 `logging=debug/expensive`，则只有具有 `logging` 目标且包含单词 `expensive` 的 `debug` 级别以上的日志将被显示。

哇，这有很多配置！我建议你尝试不同的过滤模式，并运行示例以了解各个部分是如何结合在一起的。如果你需要更多信息，`env_logger` 当前版本中 `RUST_LOG` 值的所有可能性都在 [`docs.rs/env_logger/`](https://docs.rs/env_logger/) 中有文档说明。

# 还有更多...

如果你以前从未使用过日志记录器，你可能想知道某些日志级别之间的区别。当然，你可以根据需要使用它们，但以下约定在许多语言的日志记录器中是常见的：

| **日志级别** | **用法** | **示例** |
| --- | --- | --- |
| `Error` | 发生了一些可能很快终止程序的重大问题。如果应用程序是一个应该始终运行的服务，系统管理员应立即被通知。 | 数据库连接已断开。 |
| `Warn` | 发生了一些不是严重的问题或具有自动修复方法的错误。应该在某个时候有人检查并修复它。 | 一个用户的配置文件包含未识别的选项，这些选项已被忽略。 |
| `Info` | 可能会在以后查看的一些有用信息。这记录了正常条件。 | 用户已启动或停止了一个进程。由于没有提供配置，已使用默认值。 |
| `Debug` | 当尝试解决问题时对程序员或系统管理员有帮助的信息。与许多其他语言相反，调试日志在发布构建中不会被删除。 | 传递给主要函数的参数。应用程序在各个点的当前状态。 |
| `Trace` | 只有在程序员试图追踪错误时才有用的非常低级别的控制流信号。允许重建堆栈跟踪。 | 辅助函数的参数。函数的开始和结束。 |

许多语言也包含一个`Fatal`日志级别。在 Rust 中，传统的`panic!()`用于此目的。如果您想以某种特殊方式记录您的恐慌，可以通过调用`std::panic::set_hook()`并传递您想要的任何功能来简单地将其打印到`stderr`，从而替换对恐慌的常规反应。以下是一个示例：

```rs
    std::panic::set_hook(Box::new(|e| {
        println!("Oh noes, something went wrong D:");
        println!("{:?}", e);
    }));
    panic!("A thing broke");
```

`env_logger`的一个好替代品是`slog`包，它以陡峭的学习曲线为代价提供了出色的可扩展结构化日志。此外，它的输出看起来很漂亮。如果您对此感兴趣，请务必在[`github.com/slog-rs/slog.`](https://github.com/slog-rs/slog)上查看。

# 创建一个自定义日志记录器

有时您或您的用户可能会有非常具体的日志需求。在这个配方中，我们将学习如何创建一个自定义日志记录器以与`log`包一起使用。

# 如何做到这一点...

1.  打开之前为您生成的`Cargo.toml`文件。

1.  在`[dependencies]`部分下，如果您在上一个配方中没有这样做，请添加以下行：

```rs
log = "0.4.1"
```

如果您愿意，您可以访问日志的 crates.io 页面([`crates.io/crates/log`](https://crates.io/crates/log))以检查最新版本，并使用该版本。

1.  在`bin`文件夹中创建一个名为`custom_logger.rs`的文件。

1.  如果您使用的是基于 Unix 的系统，请添加以下代码并使用`RUST_LOG=custom_logger cargo run --bin custom_logger`运行它。否则，在 Windows 上运行`$env:RUST_LOG="custom_logger"; cargo run --bin custom_logger`：

```rs
1   #[macro_use]
2   extern crate log;
3 
4   use log::{Level, Metadata, Record};
5   use std::fs::{File, OpenOptions};
6   use std::io::{self, BufWriter, Write};
7   use std::{error, fmt, result};
8   use std::sync::RwLock;
9   use std::time::{SystemTime, UNIX_EPOCH};
10 
11  // This logger will write logs into a file on disk
12  struct FileLogger {
13    level: Level,
14    writer: RwLock<BufWriter<File>>,
15  }
16 
17  impl log::Log for FileLogger {
18    fn enabled(&self, metadata: &Metadata) -> bool {
19      // Check if the logger is enabled for a certain log level
20      // Here, you could also add own custom filtering based on 
         targets or regex
21      metadata.level() <= self.level
22    }
23 
24    fn log(&self, record: &Record) {
25      if self.enabled(record.metadata()) {
26        let mut writer = self.writer
27          .write()
28          .expect("Failed to unlock log file writer in write 
             mode");
29        let now = SystemTime::now();
30        let timestamp = now.duration_since(UNIX_EPOCH).expect(
31         "Failed to generate timestamp: This system is 
            operating before the unix epoch",
32        );
33        // Write the log into the buffer
34        write!(
35          writer,
36          "{} {} at {}: {}\n",
37          record.level(),
38          timestamp.as_secs(),
39          record.target(),
40          record.args()
41        ).expect("Failed to log to file");
42      }
43      self.flush();
44    }
45 
46    fn flush(&self) {
47      // Write the buffered logs to disk
48      self.writer
49        .write()
50        .expect("Failed to unlock log file writer in write 
               mode")
51        .flush()
52        .expect("Failed to flush log file writer");
53    }
54  }
55 
56  impl FileLogger {
57    // A convenience method to set everything up nicely
58    fn init(level: Level, file_name: &str) -> Result<()> {
59      let file = OpenOptions::new()
60        .create(true)
61        .append(true)
62        .open(file_name)?;
63      let writer = RwLock::new(BufWriter::new(file));
64      let logger = FileLogger { level, writer };
65      // set the global level filter that log uses to optimize 
         ignored logs
66      log::set_max_level(level.to_level_filter());
67      // set this logger as the one used by the log macros
68      log::set_boxed_logger(Box::new(logger))?;
69      Ok(())
70    }
71  }
```

这是我们在日志记录器中使用的自定义错误：

```rs
73  // Our custom error for our FileLogger
74  #[derive(Debug)]
75  enum FileLoggerError {
76    Io(io::Error),
77    SetLogger(log::SetLoggerError),
78  }
79 
80  type Result<T> = result::Result<T, FileLoggerError>;
81  impl error::Error for FileLoggerError {
82    fn description(&self) -> &str {
83      match *self {
84        FileLoggerError::Io(ref err) => err.description(),
85        FileLoggerError::SetLogger(ref err) => 
           err.description(),
86      }
87    }
88  
89    fn cause(&self) -> Option<&error::Error> {
90      match *self {
91        FileLoggerError::Io(ref err) => Some(err),
92        FileLoggerError::SetLogger(ref err) => Some(err),
93      }
94    }
95  }
96 
97  impl fmt::Display for FileLoggerError {
98    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
99      match *self {
100       FileLoggerError::Io(ref err) => write!(f, "IO error: {}", 
           err),
101       FileLoggerError::SetLogger(ref err) => write!(f, "Parse 
           error: {}", err),
102     }
103   }
104 }
105 
106 impl From<io::Error> for FileLoggerError {
107   fn from(err: io::Error) -> FileLoggerError {
108     FileLoggerError::Io(err)
109   }
110 }
111 
112 impl From<log::SetLoggerError> for FileLoggerError {
113   fn from(err: log::SetLoggerError) -> FileLoggerError {
114     FileLoggerError::SetLogger(err)
115   }
116 }
```

初始化和使用日志记录器：

```rs
118 fn main() {
119   FileLogger::init(Level::Info, "log.txt").expect("Failed to 
       init 
      FileLogger");
120   trace!("Beginning the operation");
121   info!("A lightning strikes a body");
122   warn!("It's moving");
123   error!("It's alive!");
124   debug!("Dr. Frankenstein now knows how it feels to be god");
125   trace!("End of the operation");
126 }
```

# 它是如何工作的...

我们的自定义`FileLogger`，正如其名所示，会将日志记录到文件中。它还接受初始化时的最大日志级别。

你不应该习惯直接在磁盘上记录日志。正如 *The Twelve-Factor App 指南* ([`12factor.net/logs`](https://12factor.net/logs)) 所述，日志应该被视为事件流，以原始转储到 `stdout` 的形式。然后，生产环境可以将所有日志流通过 `systemd` 或专门的日志路由器（如 `Logplex` ([`github.com/heroku/logplex`](https://github.com/heroku/logplex)) 或 `Fluentd` [`github.com/fluent/fluentd`](https://github.com/fluent/fluentd)）路由到最终目的地。这些路由器将决定日志是否应该发送到文件、分析系统（如 `Splunk` [`www.splunk.com/`](https://www.splunk.com/)）或数据仓库（如 `Hive` [`hive.apache.org/`](http://hive.apache.org/)）。

每个日志记录器都需要实现 `log::Log` 特性，该特性包括 `enabled`、`log` 和 `flush` 方法。`enabled` 应该返回是否接受某个日志事件。在这里，你可以随心所欲地使用你想要的任何过滤逻辑[18]。这个方法永远不会直接被 `log` 调用，所以它的唯一目的是作为 `log` 方法中的辅助方法，我们将在稍后讨论。`flush` [46] 被以相同的方式处理。它应该应用你在日志中缓存的任何更改，但它永远不会被 `log` 调用。

事实上，如果你的日志记录器不与文件系统或网络交互，它可能只是简单地通过不执行任何操作来实现 `flush`：

```rs
fn flush(&self) {}
```

然而，`Log` 实现的核心是 `log` 方法[24]，因为每当调用日志宏时都会调用它。实现通常从以下行开始，然后是实际的日志记录，最后调用 `self.flush()`[25]：

```rs
if self.enabled(record.metadata()) {
```

我们实际的日志记录操作只是简单地组合当前日志级别、Unix 时间戳、目标地址和日志消息，然后将这些内容写入文件，并在之后刷新。

从技术上来说，我们的 `self.flush()` 调用也应该在 `if` 块内部，但这将需要围绕可变借用 `writer` 的额外作用域，以避免两次借用。由于这与这里的根本课程无关，即如何创建一个日志记录器，我们将它放在块外，以便使示例更易于阅读。顺便说一下，我们从 `RwLock` 借用 `writer` 的方式是 第八章 中 *使用 RwLock 并行访问资源* 的主题。现在，只需知道 `RwLock` 是一个在并行环境（如日志记录器）中安全使用的 `RefCell` 就足够了。

在为 `FileLogger` 实现 `Log` 之后，用户可以使用它作为 `log` 调用的日志记录器。为此，用户需要做两件事：

+   通过`log::set_max_level()` [66]告诉`log`日志级别，这是我们记录器能接受的最大日志级别。这是必需的，因为如果使用超过我们最大级别的日志级别，`log`会在运行时优化我们的记录器上的`.log()`调用。该函数接受一个`LevelFilter`而不是`Level`，这就是为什么我们首先需要用`to_level_filter()`将我们的级别转换的原因 [66]。这种类型的原因在*更多内容...*部分中解释。

+   使用`log::set_boxed_logger()`[68]指定记录器。`log`接受一个盒子，因为它将其记录器实现视为一个特质对象，我们在第五章，*高级数据结构*的*装箱数据*部分讨论过。如果你想要（非常）小的性能提升，你也可以使用`log::set_logger()`，它接受一个`static`，你首先需要通过`lazy_static` crate 创建它。关于这一点，请参阅第五章，*高级数据结构*和配方*创建懒加载静态对象*。

这通常在提供的`.init()`方法上完成，就像`env_logger`一样，我们在第[58]行实现它。

```rs
    fn init(level: Level, file_name: &str) -> Result<()> {
        let file = OpenOptions::new()
            .create(true)
            .append(true)
            .open(file_name)?;
        let writer = RwLock::new(BufWriter::new(file));
        let logger = FileLogger { level, writer };
        log::set_max_level(level.to_level_filter());
        log::set_boxed_logger(Box::new(logger))?;
        Ok(())
    }
```

当我们谈论这个话题时，我们也可以用同样的方法打开文件。其他可能性包括让用户直接将`File`传递给`init`作为参数，或者为了最大的灵活性，使记录器成为一个泛型记录器，它接受任何实现`Write`的流。

然后，我们在随后的行[74 到 116]返回一个自定义错误。

我们记录器的初始化示例可能看起来像这样 [119]：

```rs
FileLogger::init(Level::Info, "log.txt").expect("Failed to init FileLogger");
```

# 更多内容...

为了简单起见，`FileLogger`不对任何目标进行区分。一个更复杂的记录器，如`env_logger`，可以在不同的目标上设置不同的日志级别。为此，`log`为我们提供了`LevelFilter`枚举，它有一个`Off`状态，对应于*为此目标未启用日志记录*。如果你需要创建这样的记录器，务必记住这个枚举。你可以通过查看`env_logger`的源代码来获取一些关于如何实现基于目标的过滤器的灵感，源代码位于[`github.com/sebasmagri/env_logger/blob/master/src/filter/mod.rs`](https://github.com/sebasmagri/env_logger/blob/master/src/filter/mod.rs)。

在一个真正用户友好的记录器中，你希望显示用户自己的本地时间戳。对于与时间测量、时区和日期相关的一切，请查看`chrono` crate，位于[`crates.io/crates/chrono`](https://crates.io/crates/chrono)。

# 参见

+   在第五章，*高级数据结构*中的*装箱数据*配方

+   在第五章**，高级数据结构*中的*创建懒加载静态对象*配方

+   在第七章，*并行和 Rayon*中的*使用 RwLocks 并行访问资源*配方

# 实现 Drop 特性

在传统面向对象语言中，有析构函数，而 Rust 有 `Drop` 特性，它由一个单一的 `drop` 函数组成，当变量的生命周期结束时会被调用。通过实现它，你可以执行任何必要的清理或高级日志记录。你还可以通过 RAII 自动释放资源，正如我们将在下一个菜谱中看到的。

# 如何做...

1.  在 `bin` 文件夹中创建一个名为 `drop.rs` 的文件。

1.  添加以下代码，并使用 `cargo run --bin drop` 运行它：

```rs
1   use std::fmt::Debug;
2 
3   struct CustomSmartPointer<D>
4   where
5   D: Debug,
6   {
7   data: D,
8   }
9 
10  impl<D> CustomSmartPointer<D>
11  where
12  D: Debug,
13  {
14    fn new(data: D) -> Self {
15    CustomSmartPointer { data }
16    }
17  }
18 
19  impl<D> Drop for CustomSmartPointer<D>
20  where
21  D: Debug,
22  {
23    // This will automatically be called when a variable is 
       dropped
24    // It cannot be called manually
25    fn drop(&mut self) {
26    println!("Dropping CustomSmartPointer with data `{:?}`", 
       self.data);
27    }
28  }
29 
30  fn main() {
31    let a = CustomSmartPointer::new("A");
32    let b = CustomSmartPointer::new("B");
33    let c = CustomSmartPointer::new("C");
34    let d = CustomSmartPointer::new("D");
35 
36    // The next line would cause a compiler error,
37    // as destructors cannot be explicitely called
38    // c.drop();
39 
40    // The correct way to drop variables early is the following:
41    std::mem::drop(c);
42  }
```

# 它是如何工作的...

这个例子是从 Rust 书的第二版中稍作修改后得到的([`doc.rust-lang.org/book/second-edition/`](https://doc.rust-lang.org/book/second-edition/))，展示了如何开始实现自定义智能指针。在我们的例子中，它所做的只是在其被销毁时打印存储数据的 `Debug` 信息[26]。我们通过实现 `Drop` 特性及其单一的 `drop` 函数[25]来完成这项工作，编译器会在变量被销毁时自动调用这个函数。所有智能指针都是这样实现的。

变量销毁的时刻几乎总是它离开作用域的时候。因此，我们不能直接调用 `drop` 函数[38]。当它退出作用域时，编译器仍然会调用它，所以清理将会发生两次，导致未定义的行为。如果你需要提前销毁一个变量，你可以通过在它上面调用 `std::mem:drop` 来告诉编译器这样做 [41]。

变量退出其作用域时，会以 **LIFO** 方式被销毁：**后进先出**。这意味着最后声明的变量将是第一个被销毁的。如果我们按照 `a`、`b`、`c` 和 `d` 的顺序分配变量，它们将被销毁的顺序将是 `d`、`c`、`b`、`a`。在我们的例子中，我们提前销毁 `c`[41]，所以我们的顺序变成了 `c`、`d`、`b`、`a`。

# 还有更多...

你想知道一个像 `std::mem::drop` 这样的复杂低级函数是如何实现的吗：

```rs
pub fn drop<T>(_x: T) { }
```

没错，它什么也没做！这样做的原因是它通过值传递 `T`，将其移动到函数中。函数什么也不做，并且它所有的拥有变量都退出了作用域。为 Rust 的借用检查器欢呼！

# 另请参阅

+   在第五章的“*装箱数据*”菜谱中。

+   在第五章的“*与智能指针共享所有权*”菜谱中。

# 理解 RAII

我们可以比简单的析构函数更进一步。我们可以创建可以给用户提供临时访问某些资源或功能的结构体，并在用户完成时自动撤销访问。这个概念被称为 **RAII**，代表 **Resource Acquisition Is Initialization**。换句话说，资源的有效性与变量的生命周期绑定。

# 如何做...

按照以下步骤操作：

1.  打开之前为你生成的 `Cargo.toml` 文件。

1.  在 `bin` 文件夹中创建一个名为 `raii.rs` 的文件。

1.  添加以下代码，并用`cargo run --bin raii`运行它：

```rs
1   use std::ops::Deref;
2 
3   // This represents a low level, close to the metal OS feature 
     that
4   // needs to be locked and unlocked in some way in order to be   
     accessed
5   // and is usually unsafe to use directly
6   struct SomeOsSpecificFunctionalityHandle;
7 
8   // This is a safe wrapper around the low level struct
9   struct SomeOsFunctionality<T> {
10    // The data variable represents whatever useful information
11    // the user might provide to the OS functionality
12    data: T,
13    // The underlying struct is usually not savely movable,
14    // so it's given a constant address in a box
15    inner: Box<SomeOsSpecificFunctionalityHandle>,
16  }
17 
18  // Access to a locked SomeOsFunctionality is wrapped in a guard
19  // that automatically unlocks it when dropped
20  struct SomeOsFunctionalityGuard<'a, T: 'a> {
21    lock: &'a SomeOsFunctionality<T>,
22  }
23 
24  impl SomeOsSpecificFunctionalityHandle {
25    unsafe fn lock(&self) {
26      // Here goes the unsafe low level code
27    }
28    unsafe fn unlock(&self) {
29      // Here goes the unsafe low level code
30    }
31  }
```

现在是`structs`的实现：

```rs

33  impl<T> SomeOsFunctionality<T> {
34    fn new(data: T) -> Self {
35      let handle = SomeOsSpecificFunctionalityHandle;
36        SomeOsFunctionality {
37          data,
38          inner: Box::new(handle),
39        }
40    }
41 
42    fn lock(&self) -> SomeOsFunctionalityGuard<T> {
43      // Lock the underlying resource.
44      unsafe {
45        self.inner.lock();
46      }
47 
48      // Wrap a reference to our locked selves in a guard
49      SomeOsFunctionalityGuard { lock: self }
50    }
51  }
52 
53  // Automatically unlock the underlying resource on drop
54  impl<'a, T> Drop for SomeOsFunctionalityGuard<'a, T> {
55    fn drop(&mut self) {
56      unsafe {
57        self.lock.inner.unlock();
58      }
59    }
60  }
61 
62  // Implementing Deref means we can directly
63  // treat SomeOsFunctionalityGuard as if it was T
64  impl<'a, T> Deref for SomeOsFunctionalityGuard<'a, T> {
65    type Target = T;
66 
67    fn deref(&self) -> &T {
68    &self.lock.data
69    }
70  }

```

最后，实际的用法：

```rs
72  fn main() {
73    let foo = SomeOsFunctionality::new("Hello World");
74    {
75      // Locking foo returns an unlocked guard
76      let bar = foo.lock();
77      // Because of the Deref implementation on the guard,
78      // we can use it as if it was the underlying data
79      println!("The string behind foo is {} characters long", 
         bar.len());
80 
81      // foo is automatically unlocked when we exit this scope
82    }
83    // foo could now be unlocked again if needed
84  }
```

# 它是如何工作的...

嗯，这是一堆复杂的代码。

让我们先介绍参与这个示例的结构：

+   `SomeOsSpecificFunctionalityHandle` [6]代表操作系统的一个未指定功能，该功能操作某些数据，并且可能直接使用是不安全的。我们假设这个功能锁定了一些需要再次解锁的操作系统资源。

+   `SomeOsFunctionality` [9]代表围绕该功能的保护器，以及一些可能对它有用的数据`T`。

+   `SomeOsFunctionalityGuard` [20]是通过使用`lock`函数创建的 RAII 保护器。当它被丢弃时，它将自动解锁底层资源。此外，它可以直接用作数据`T`本身。

这些函数可能看起来有些抽象，因为它们并没有做任何具体的事情，而是作用于某些未指定的操作系统功能。这是因为大多数真正有用的候选者已经存在于标准库中——比如`File`、`RwLock`、`Mutex`等等。剩下的主要是特定领域的使用案例，当编写低级库或处理一些需要自动解锁的特殊、自制的资源时。当你发现自己正在编写这样的代码时，你会欣赏 RAII 的优雅。

结构体的实现引入了一些可能初次遇到时会显得有些令人困惑的新概念。在`SomeOsSpecificFunctionalityHandle`的实现中，我们可以看到一些`unsafe`关键字[25, 28 和 44]：

```rs
impl SomeOsSpecificFunctionalityHandle {
    unsafe fn lock(&self) {
        // Here goes the unsafe low level code
    }
    unsafe fn unlock(&self) {
        // Here goes the unsafe low level code
    }
}
...
fn lock(&self) -> SomeOsFunctionalityGuard<T> {
    // Lock the underlying resource.
    unsafe {
        self.inner.lock();
    }

```

```rs
    // Wrap a reference to our locked selves in a guard
    SomeOsFunctionalityGuard { lock: self }
}
```

让我们从不安全块[44 到 46]开始：

```rs
unsafe {
    self.inner.lock();
}
```

`unsafe`关键字告诉编译器以特殊方式处理前面的代码块。它禁用了借用检查器，并允许你做各种疯狂的事情：像 C 语言一样解引用原始指针，修改可变静态变量，调用不安全函数。作为交换，编译器也不会给你任何保证。例如，它可能会访问无效的内存，导致**SEGFAULT**。如果你想了解更多关于`unsafe`关键字的信息，请查看官方 Rust 书籍第二版中关于它的章节[`doc.rust-lang.org/book/second-edition/ch19-01-unsafe-rust.html`](https://doc.rust-lang.org/book/second-edition/ch19-01-unsafe-rust.html)。

通常来说，应该避免编写不安全的代码。然而，在以下情况下这样做是可以接受的：

+   你正在编写一些直接与操作系统交互的代码，并且你想要创建一个围绕不安全部分的安全包装器，这正是我们在这里所做的事情。

+   你绝对 100%完全确信，在非常具体的上下文中，你所做的是没有问题的，这与编译器的观点相反。

如果你想知道为什么`unsafe`块是空的，那是因为我们在这个配方中没有使用任何实际的操作系统资源。如果你想使用，处理它们的代码将放在那两个空块中。

`unsafe`关键字的其他用途如下[25]：

```rs
unsafe fn lock(&self) { ... }
```

这将函数本身标记为不安全，意味着它只能在不安全的块中调用。记住，在函数中调用`unsafe`代码并不会使函数自动变为不安全，因为函数可能是一个围绕它的安全包装。

现在我们从假设的低级实现`SomeOsSpecificFunctionalityHandle`转移到我们对其安全包装`SomeOsFunctionality`[33]的现实实现。其构造函数没有惊喜（如果你需要复习，请参阅第一章，*学习基础知识*和*使用构造函数模式*配方）：

```rs
fn new(data: T) -> Self {
    let handle = SomeOsSpecificFunctionalityHandle;
    SomeOsFunctionality {
        data,
        inner: Box::new(handle),
    }
}
```

我们只是简单地准备底层的操作系统功能，并将其与用户提供的资料一起存储在我们的`struct`中。我们使用`Box`封装了句柄，因为，如代码中较早的注释（在第 13 行和第 14 行）所述，与操作系统交互的低级结构通常不安全移动。然而，我们不想限制用户移动我们的安全包装，因此，我们通过将句柄放入堆中并通过`Box`来使其可移动，这给它提供了一个永久地址。然后移动的仅仅是指向该地址的智能指针。关于这一点，请阅读第五章，*高级数据结构*和*装箱数据*配方。

实际的封装发生在`lock`方法中：

```rs
fn lock(&self) -> SomeOsFunctionalityGuard<T> {
    // Lock the underlying resource.
    unsafe {
        self.inner.lock();
    }

    // Wrap a reference to our locked selves in a guard
    SomeOsFunctionalityGuard { lock: self }
}
```

当与实际的操作系统功能或自定义资源一起工作时，你希望在这样做之前确保`self.inner.lock()`在这个上下文中是安全的，否则，包装器将不会安全。这也是你可以对`self.data`做有趣事情的地方，你可能会将其与提到的资源结合起来使用。

在锁定我们的东西之后，我们返回一个 RAII 保护器，它包含对我们结构的引用[49]，当它被丢弃时将解锁我们的资源。查看`SomeOsFunctionalityGuard`的实现，你可以看到我们不需要为它实现任何新的函数。我们只需要实现两个特质。我们首先从`Drop`[54]开始，你在之前的配方中已经见过。实现它意味着我们可以在保护器通过`SomeOsFunctionality`的引用被丢弃时解锁资源。再次确保以某种方式安排环境，在调用`self.lock.inner.unlock()`之前确保它实际上是安全的。

由于我们基本上是在创建一个指向`data`的智能指针，我们可以使用`Deref`特质[64]。通过将`Target`设置为`A`实现`Deref for B`允许对`B`的引用被解引用到`A`。或者用稍微不准确的话来说，它让`B`表现得像`A`。在我们的例子中，通过将`Target`设置为`T`实现`Deref for SomeOsFunctionalityGuard`意味着我们可以像使用底层数据一样使用我们的守护者。因为如果实现不当，这可能会给用户带来很大的困惑，Rust 建议您只在对智能指针实现它，而不是其他任何东西。

实现`Deref`对于 RAII 模式来说当然不是强制的，但可能会非常有用，正如我们一会儿将要看到的。

让我们看看我们现在如何使用所有这些花哨的功能：

```rs
fn main() {
    let foo = SomeOsFunctionality::new("Hello World");
    {
        let bar = foo.lock();
        println!("The string behind foo is {} characters long", 
        bar.len());
    }
}
```

用户永远不应该直接使用`SomeOsSpecificFunctionalityHandle`，因为它是不安全的。相反，他可以构建一个`SomeOsFunctionality`的实例，他可以随意传递和存储[73]。每次他需要使用它背后的酷功能时，他都可以在当前作用域中调用`lock`，并且他将收到一个在完成后会为他清理的守护者[81]。因为他实现了`Deref`，所以他可以直接使用守护者，就像它是底层数据一样。在我们的例子中，`data`是一个`&str`，所以我们可以在我们的守护者上直接使用`str`的方法，就像我们在第[79]行所做的那样，通过调用`.len()`。

在这个小范围结束之后，我们的守护者对资源调用`unlock`，因为`foo`独立地仍然存在，我们可以继续以我们想要的任何方式再次锁定它。

# 还有更多...

这个例子是为了与`RwLock`和`Mutex`的实现保持一致。唯一缺少的是为了不让这个配方更加复杂而省略的一个额外的间接层。`SomeOsSpecificFunctionalityHandle`不应该包含`lock`和`unlock`的实际实现，而应该将调用传递给存储的实现，该实现针对您正在使用的任何操作系统。例如，假设您有一个针对基于 Windows 的实现的结构体`windows::SomeOsSpecificFunctionalityHandle`和一个针对基于 Unix 的实现的结构体`unix::SomeOsSpecificFunctionalityHandle`。`SomeOsSpecificFunctionalityHandle`应该根据正在运行的操作系统，有条件地将它的`lock`和`unlock`调用传递到正确的实现。这些可能具有更多功能。Windows 的那个可能有一个`awesome_windows_thing()`函数，这可能对需要它的不幸的 Windows 开发者很有用。Unix 实现可能有一个`confusing_posix_thing()`函数，它做一些只有 Unix 黑客才能理解的非常奇怪的事情。重要的是，我们的`SomeOsSpecificFunctionalityHandle`应该代表实现的一个通用接口。在我们的例子中，这意味着每个支持的操作系统都有能力锁定和解除锁定相关的资源。

# 参见

+   *使用构造器模式* 的配方在 第一章，*学习基础知识*

+   *数据装箱* 的配方在 第五章，*高级数据结构*

+   *使用 RwLocks 并行访问资源* 的配方在 第七章，*并行性与 Rayon*
