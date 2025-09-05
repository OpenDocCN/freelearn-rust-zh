# 进一步学习高级 Rust

对于 Rust 语言对热衷于学习的用户提出的困难，毫无疑问。然而，如果您正在阅读这篇文章，您已经走在了大多数人的前面，并投入了所需的时间来提高。这种语言及其强制您思考内存的方式将把新的概念引入到您的编程习惯中。Rust 并不一定提供新的工具来完成事情，但借用和所有权规则帮助我们更多地关注作用域、生命周期以及适当地释放内存，无论语言如何。因此，让我们深入探讨 Rust 中更高级的概念，以完成对语言的全面理解——何时、为什么以及如何应用以下概念：

+   使用枚举创建有意义的数字

+   没有空值

+   复杂的条件与模式匹配

+   实现自定义迭代器

+   高效地过滤和转换序列

+   以不安全的方式读取内存

+   共享所有权

+   共享可变所有权

+   带有显式生命周期的引用

+   使用特质界限强制行为

+   与泛型数据类型一起工作

# 使用枚举创建有意义的数字

枚举，简称为枚举，是许多语言中常见的编程结构。这些类型的特殊情况允许将数字映射到名称。这可以用来将常量绑定到单个名称下，并允许我们声明值作为变体。例如，我们可以在枚举 `MathConstants` 中有 π，以及欧拉数作为变体。Rust 并无不同，但它可以走得更远。Rust 不仅仅依赖于 *命名数字*，它允许枚举具有与其他 Rust 类型相同的灵活性。让我们看看这在实践中意味着什么。

# 如何做到这一点...

按照以下步骤来探索枚举：

1.  使用 `cargo new enums --lib` 创建一个新的项目，并在 Visual Studio Code 或您选择的任何 IDE 中打开此文件夹。

1.  打开 `src/lib.rs` 并声明一个包含一些数据的枚举：

```rs
use std::io;

pub enum ApplicationError {
    Code { full: usize, short: u16 },
    Message(String),
    IOWrapper(io::Error),
    Unknown
}
```

1.  除了声明之外，我们还实现了一个简单的函数：

```rs
impl ApplicationError {

    pub fn print_kind(&self, mut to: &mut impl io::Write) -> 
    io::Result<()> {
        let kind = match self {
            ApplicationError::Code { full: _, short: _ } => "Code",
            ApplicationError::Unknown => "Unknown",
            ApplicationError::IOWrapper(_) => "IOWrapper",
            ApplicationError::Message(_) => "Message"
        };
        write!(&mut to, "{}", kind)?; 
        Ok(())
    }
}
```

1.  现在，我们还需要对枚举做一些处理，所以让我们实现一个名为 `do_work` 的虚拟函数：

```rs
pub fn do_work(choice: i32) -> Result<(), ApplicationError> {
    if choice < -100 {

            Err(ApplicationError::IOWrapper(io::Error::
             from(io::ErrorKind::Other
  )))
    } else if choice == 42 {
        Err(ApplicationError::Code { full: choice as usize, short: 
        (choice % u16::max_value() as i32) as u16 } )
    } else if choice > 42 {
        Err(ApplicationError::Message(
            format!("{} lead to a terrible error", choice)
        ))
    } else {
        Err(ApplicationError::Unknown)
    }
}
```

1.  未经测试，一切都是假的！现在，添加一些测试来展示枚举强大的匹配功能，从 `do_work()` 函数开始：

```rs

#[cfg(test)]
mod tests {
    use super::{ApplicationError, do_work};
    use std::io;

    #[test]
    fn test_do_work() {
        let choice = 10;
        if let Err(error) = do_work(choice) {
            match error {
                ApplicationError::Code { full: code, short: _ } => 
                assert_eq!(choice as usize, code),
                // the following arm matches both variants (OR)
                ApplicationError::Unknown | 
                ApplicationError::IOWrapper(_) => assert!(choice < 
                42),
                ApplicationError::Message(msg) => 
                assert_eq!(format!
                ("{} lead to a terrible error", choice), msg)
            }
        }
    }
```

对于 `get_kind()` 函数，我们还需要一个测试：

```rs
    #[test]
    fn test_application_error_get_kind() {
        let mut target = vec![];
        let _ = ApplicationError::Code { full: 100, short: 100 
        }.print_kind(&mut target);
        assert_eq!(String::from_utf8(target).unwrap(), 
        "Code".to_string());

        let mut target = vec![];
        let _ = ApplicationError::Message("0".to_string()).
        print_kind(&mut target);
        assert_eq!(String::from_utf8(target).unwrap(), 
        "Message".to_string());

        let mut target = vec![];
        let _ = ApplicationError::Unknown.print_kind(&mut target);
        assert_eq!(String::from_utf8(target).unwrap(), 
        "Unknown".to_string());

        let mut target = vec![];
        let error = io::Error::from(io::ErrorKind::WriteZero);
        let _ = ApplicationError::IOWrapper(error).print_kind(&mut 
        target);
        assert_eq!(String::from_utf8(target).unwrap(), 
        "IOWrapper".to_string());

    }
}
```

1.  在项目的根目录下调用 `cargo test`，我们可以观察到输出：

```rs
$ cargo test
   Compiling enums v0.1.0 (Rust-Cookbook/Chapter02/enums)
    Finished dev [unoptimized + debuginfo] target(s) in 0.61s
     Running target/debug/deps/enums-af52cbd5cd8d54cb

running 2 tests
test tests::test_do_work ... ok
test tests::test_application_error_get_kind ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests enums

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

现在，让我们看看枚举在底层是如何工作的。

# 如何工作...

Rust 中的枚举封装了选择——就像在任何语言中一样。然而，它们在许多方面与常规结构的行为相似：

+   它们可以为特性和函数拥有 `impl` 块。

+   无名和命名的属性可以携带不同的值。

这些方面使它们成为所有选择的真正优秀候选人，无论是配置值、标志、常量还是像*第 2 步*中那样包装错误。其他语言中的典型枚举将名称映射到你选择的数值，但 Rust 更进一步。Rust 的枚举不仅可以有数值，甚至可以有命名属性。看看*第 2 步*中的定义：

```rs
pub enum ApplicationError {
    Code { full: usize, short: u16 },
    Message(String),
    IOWrapper(io::Error),
    Unknown
}
```

`ApplicationError::Code`有两个属性，一个叫做`full`，另一个叫做`short`——就像任何其他`struct`实例一样可赋值。第二个和第三个变体，`Message`和`IOWrapper`，封装了另一个类型实例，一个是`String`，另一个是`std::io::Error`，类似于元组。

能够在匹配子句中工作的附加能力使这些结构非常有用，尤其是在代码库很大且可读性很重要的情况下——例如在*第 3 步*中，我们在枚举类型中实现了一个函数。这个函数将显式的枚举实例映射到字符串，以便更容易打印。

*第 4 步*实现了一个辅助函数，为我们提供了不同类型的错误和值来处理，这是我们*第 5 步*中创建这些函数的两个广泛测试所必需的。在那里，我们使用`match`子句（这个子句将在本章后面的食谱中讨论）从错误中提取值，并在单个分支中匹配多个枚举变体。此外，我们还创建了一个测试来展示`print_kind()`函数通过使用`Vec`作为流（归功于它实现了`Write`特质）是如何工作的。

我们成功地学习了如何使用枚举创建有意义的数字。现在，让我们继续下一个食谱。

# 没有 null

函数式语言通常没有**null**的概念，简单的理由是它总是一个特殊情况。如果你严格遵循函数式原则，每个输入都必须有一个可工作的输出——但是 null 是什么？它是错误吗？还是正常操作参数内的一个负结果？

作为遗留特性，null 自从 C/C++时代就有，当时指针实际上可以指向（无效的）地址，`0`。然而，许多新的语言试图摆脱这一点。Rust 没有 null，也没有使用`Option`类型的正常返回值。错误的情况由`Result`类型覆盖，我们为此专门写了一章，第五章，*处理错误和其他结果*。

# 如何做到这一点...

由于我们正在探索内置库功能，我们将创建几个测试来涵盖所有内容：

1.  使用`cargo new not-null --lib`创建一个新的项目，并使用 Visual Studio code 打开项目文件夹。

1.  首先，让我们看看`unwrap()`做了什么，并用以下代码替换`src/lib.rs`中的默认测试：

```rs
    #[test]
    #[should_panic]
    fn option_unwrap() {
        // Options to unwrap Options
        assert_eq!(Some(10).unwrap(), 10);
        assert_eq!(None.unwrap_or(10), 10);
        assert_eq!(None.unwrap_or_else(|| 5 * 2), 10);

        Option::<i32>::None.unwrap();
        Option::<i32>::None.expect("Better say something when 
        panicking");
    }
```

1.  `Option`也很好地封装了值，有时获取它们可能很复杂（或者简单地说，很冗长）。这里有几种获取值的方法：

```rs
    #[test]
    fn option_working_with_values() {
        let mut o = Some(42);

        let nr = o.take();
        assert!(o.is_none());
        assert_eq!(nr, Some(42));

        let mut o = Some(42);
        assert_eq!(o.replace(1535), Some(42));
        assert_eq!(o, Some(1535));

        let o = Some(1535);
        assert_eq!(o.map(|v| format!("{:#x}", v)), 
        Some("0x5ff".to_owned()));

        let o = Some(1535);
        match o.ok_or("Nope") {
            Ok(nr) => assert_eq!(nr, 1535),
            Err(_) => assert!(false)
        }
    }
```

1.  由于它们的函数式起源，在单值或集合上工作通常并不重要，`Option`在某些方面也表现得像集合：

```rs
    #[test]
    fn option_sequentials() {
        let a = Some(42);
        let b = Some(1535);
        // boolean logic with options. Note the returned values
        assert_eq!(a.and(b), Some(1535));
        assert_eq!(a.and(Option::<i32>::None), None);
        assert_eq!(a.or(None), Some(42));
        assert_eq!(a.or(b), Some(42));
        assert_eq!(None.or(a), Some(42));
        let new_a = a.and_then(|v| Some(v + 100))
                     .filter(|&v| v != 42);

        assert_eq!(new_a, Some(142));
        let mut a_iter = new_a.iter();
        assert_eq!(a_iter.next(), Some(&142));
        assert_eq!(a_iter.next(), None);
    }
```

1.  最后，在`Option`上使用`match`子句非常流行且通常是必要的：

```rs
    #[test]
    fn option_pattern_matching() {

        // Some trivial pattern matching since this is common

        match Some(100) {
            Some(v) => assert_eq!(v, 100),
            None => assert!(false) 
        };

        if let Some(v) = Some(42) {
            assert_eq!(v, 42);
        }
        else {
            assert!(false);
        }
    }
```

1.  要看到它全部工作，我们也应该运行`cargo test`：

```rs
$ cargo test
   Compiling not-null v0.1.0 (Rust-Cookbook/Chapter02/not-null)
    Finished dev [unoptimized + debuginfo] target(s) in 0.58s
     Running target/debug/deps/not_null-ed3a746487e7e3fc

running 4 tests
test tests::option_pattern_matching ... ok
test tests::option_sequentials ... ok
test tests::option_unwrap ... ok
test tests::option_working_with_values ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests not-null

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

现在，让我们深入了解代码，以更好地理解它。

# 它是如何工作的...

`Options` 在我们的初步惊讶中，是一个枚举类型。虽然这几乎保证了良好的`match`兼容性，但在其他方面枚举的行为与结构体非常相似。在*步骤 2*中，我们看到这不仅仅是一个普通的枚举，而是一个类型枚举——这迫使我们为`None`添加类型声明。*步骤 2*还展示了如何从`Option`类型中获取值，包括带和不带恐慌的情况。`unwrap()`是一个流行的选择，但它有一些变体，当遇到`None`时不会使线程停止。

`unwrap()`始终是一件危险的事情，并且只应在非生产代码中使用。它会引发恐慌，可能导致整个程序突然、意外地停止，甚至不会留下适当的错误消息。如果你希望程序停止，`expect()`是一个更好的选择，因为它允许你添加一个简单的消息。这就是为什么我们在单元测试中添加了`#[should_panic]`属性，这样我们就可以向你证明它实际上会引发恐慌（否则测试会失败）。

*步骤 3*展示了几种非侵入性的方法来*展开*`Option`的值。特别是由于`unwrap()`返回所有者值同时销毁`Option`本身，如果`Option`仍然是数据结构的一部分并且只是临时持有值，其他方法可能更有用。`take()`是为这些情况设计的，用`None`替换值，类似于`replace()`，它对替换值做同样的处理。此外，还有`map()`，它允许你直接处理值（如果存在），并忽略通常的`if`-then 或`match`结构，这些结构会增加大量的代码冗余（参考*步骤 5*）。

*步骤 4*中间有一个有趣的细节：`Options`可以像布尔值一样执行逻辑运算，类似于 Python，其中 AND/OR 运算返回特定的操作数([`docs.python.org/3/reference/expressions.html#boolean-operations`](https://docs.python.org/3/reference/expressions.html#boolean-operations))。最后但同样重要的是，`Options`也可以使用迭代器像集合一样处理。

Rust 的选项非常灵活，通过查看文档([`doc.rust-lang.org/std/option/index.html`](https://doc.rust-lang.org/std/option/index.html))，你可以找到许多不同的方法来动态转换值，而不需要繁琐的`if`、`let`和`match`保护子句。

现在我们已经成功地了解到 Rust 中没有空值，让我们继续学习下一个菜谱。

# 复杂的条件与模式匹配

如前一个示例所示，模式匹配与枚举一起非常有用。然而，还有更多！模式匹配是一种起源于函数式语言的构造，它减少了在`struct`中分配属性时在条件分支和赋值之间的许多选择。这些步骤同时进行，减少了屏幕上的代码量，并创建了一种类似于高阶`switch-case`语句的东西。

# 如何做到这一点...

只需遵循几个步骤，就可以了解更多关于模式匹配的信息：

1.  使用`cargo new pattern-matching`创建一个新的二进制项目。这次，我们将运行一个实际的可执行文件！再次，使用 Visual Studio Code 或其他编辑器打开项目。

1.  让我们看看字面量匹配。就像其他语言中的`switch-case`语句一样，每个匹配臂也可以匹配字面量：

```rs
fn literal_match(choice: usize) -> String {
    match choice {
        0 | 1 => "zero or one".to_owned(),
        2 ... 9 => "two to nine".to_owned(),
        10 => "ten".to_owned(),
        _ => "anything else".to_owned()
    }
}
```

1.  然而，模式匹配比这要强大得多。例如，可以提取元组元素并选择性地匹配：

```rs
fn tuple_match(choices: (i32, i32, i32, i32)) -> String {
    match choices {
        (_, second, _, fourth) => format!("Numbers at positions 1 
        and 3 are {} and {} respectively", second, fourth)
    }
}
```

1.  **解构**（将属性从`struct`移动到它们自己的变量中）是与结构和枚举结合使用的一个强大功能。首先，这有助于在单个匹配臂中将多个变量分配给分配给结构实例属性的值。现在，让我们定义一些结构和枚举：

```rs
enum Background {
    Color(u8, u8, u8),
    Image(&'static str),
}

enum UserType {
    Casual,
    Power
}

struct MyApp {
    theme: Background,
    user_type: UserType,
    secret_user_id: usize
}
```

然后，可以在解构匹配中匹配单个属性。枚举也工作得很好——然而，务必涵盖所有可能的变体；编译器会注意到（或使用特殊`_`来匹配所有）。匹配也是从上到下进行的，所以首先应用的规则将被执行。以下代码片段匹配了我们刚才定义的结构体的变体。如果检测到特定的用户类型和主题，它将匹配并分配变量：

```rs
fn destructuring_match(app: MyApp) -> String {
    match app {
        MyApp { user_type: UserType::Power, 
                secret_user_id: uid, 
                theme: Background::Color(b1, b2, b3) } => 
            format!("A power user with id >{}< and color background 
            (#{:02x}{:02x}{:02x})", uid, b1, b2, b3),
        MyApp { user_type: UserType::Power, 
                secret_user_id: uid, 
                theme: Background::Image(path) } => 
            format!("A power user with id >{}< and image background 
            (path: {})", uid, path),
        MyApp { user_type: _, secret_user_id: uid, .. } => format!
        ("A regular user with id >{}<, individual backgrounds not 
        supported", uid), 
    }
}
```

1.  在强大的正则匹配之上，守卫也可以强制执行某些条件。类似于解构，我们可以添加更多约束：

```rs
fn guarded_match(app: MyApp) -> String { 
    match app {
        MyApp { secret_user_id: uid, .. } if uid <= 100 => "You are 
        an early bird!".to_owned(),
        MyApp { .. } => "Thank you for also joining".to_owned()
    }
}
```

1.  到目前为止，借用和所有权并不是一个重要的问题。然而，到目前为止的所有`match`子句都采取了所有权并将其转移到匹配臂的作用域（`=>`之后的内容），除非你返回它，否则外部作用域无法对其进行任何其他操作。为了解决这个问题，也可以匹配引用：

```rs
fn reference_match(m: &Option<&str>) -> String {
    match m {
        Some(ref s) => s.to_string(),
        _ => "Nothing".to_string()
    }
}
```

1.  为了形成一个完整的循环，我们还没有匹配特定的字面量类型：字符串字面量。由于它们的堆分配，它们在本质上与`i32`或`usize`等类型不同。在语法上，它们看起来与其他任何匹配形式没有区别：

```rs
fn literal_str_match(choice: &str) -> String {
    match choice {
        "" => "Power lifting".to_owned(),
        "" => "Football".to_owned(),
        "" => "BJJ".to_owned(),
        _ => "Competitive BBQ".to_owned()
    }
}
```

1.  现在，让我们将所有这些结合起来，构建一个`main`函数，该函数使用正确的参数调用各种函数。让我们先打印一些简单的匹配：

```rs
pub fn main() {
    let opt = Some(42);
    match opt {
        Some(nr) => println!("Got {}", nr),
        _ => println!("Found None") 
    }
    println!();
    println!("Literal match for 0: {}", literal_match(0));
    println!("Literal match for 10: {}", literal_match(10));
    println!("Literal match for 100: {}", literal_match(100));

    println!();
    println!("Literal match for 0: {}", tuple_match((0, 10, 0, 
    100)));

    println!();
    let mystr = Some("Hello");
    println!("Matching on a reference: {}", 
    reference_match(&mystr));
    println!("It's still owned here: {:?}", mystr);
```

接下来，我们还可以打印解构后的匹配：

```rs
    println!();
    let power = MyApp {
        secret_user_id: 99,
        theme: Background::Color(255, 255, 0),
        user_type: UserType::Power
    };
    println!("Destructuring a power user: {}", 
    destructuring_match(power));

    let casual = MyApp {
        secret_user_id: 10,
        theme: Background::Image("my/fav/image.png"),
        user_type: UserType::Casual
    };
    println!("Destructuring a casual user: {}", 
    destructuring_match(casual));

    let power2 = MyApp {
        secret_user_id: 150,
        theme: Background::Image("a/great/landscape.png"),
        user_type: UserType::Power
    };
    println!("Destructuring another power user: {}", 
    destructuring_match(power2));
```

最后，让我们看看关于守卫和 UTF 符号的文本字面量匹配：

```rs
    println!();
    let early = MyApp {
        secret_user_id: 4,
        theme: Background::Color(255, 255, 0),
        user_type: UserType::Power
    };
    println!("Guarded matching (early): {}", guarded_match(early));

     let not_so_early = MyApp {
        secret_user_id: 1003942,
        theme: Background::Color(255, 255, 0),
        user_type: UserType::Power
    };
    println!("Guarded matching (late): {}", 
    guarded_match(not_so_early));
    println!();

    println!("Literal match for : {}", literal_str_match(""));
    println!("Literal match for : {}", literal_str_match(""));
    println!("Literal match for : {}", literal_str_match(""));
    println!("Literal match for : {}", literal_str_match(""));
}
```

1.  最后一步再次涉及到运行程序。由于这不是一个库项目，结果将在命令行上打印。您可以随意更改`main`函数中的任何变量，以查看它如何影响输出。以下是输出*应该*是的样子：

```rs
$ cargo run
   Compiling pattern-matching v0.1.0 (Rust-
   Cookbook/Chapter02/pattern-matching)
    Finished dev [unoptimized + debuginfo] target(s) in 0.43s
     Running `target/debug/pattern-matching`
Got 42

Literal match for 0: zero or one
Literal match for 10: ten
Literal match for 100: anything else

Literal match for 0: Numbers at positions 1 and 3 are 10 and 100 respectively

Matching on a reference: Hello
It's still owned here: Some("Hello")

Destructuring a power user: A power user with id >99< and color background (#ffff00)
Destructuring a casual user: A regular user with id >10<, individual backgrounds not supported
Destructuring another power user: A power user with id >150< and image background (path: a/great/landscape.png)

Guarded matching (early): You are an early bird!
Guarded matching (late): Thank you for also joining

Literal match for : BJJ
Literal match for : Football
Literal match for : Power lifting
Literal match for : Competitive BBQ
```

现在，让我们幕后看看，以更好地理解代码。

# 它是如何工作的...

自从我们在 Scala 编程语言中遇到模式匹配以来，我们就爱上了它的简单性。作为函数式编程的主要支柱，这项技术提供了一种快速转换值的方法，而不会牺牲 Rust 的类型安全。

在**步骤 2**和**步骤 7**中的字面匹配是一种节省`if-else`链的好方法。然而，最常见的情况可能是为了解包`Result`或`Option`类型以提取封装的值。虽然使用`|`符号可以实现多个匹配，但有一些特殊操作符可以匹配特定的变体：`...`表示一个范围，而`..`表示跳过结构体中剩余的成员。`_`几乎总是忽略特定事物的通配符，作为一个`match`子句，它是一个通配符，应该放在最后。在**步骤 3**中，我们进行了大量的元组解包；我们使用`_`代替变量名来跳过一些匹配。

以类似的方式，**步骤 4**设置了并使用 Rust 的`match`子句（也称为解构）来匹配类型内的属性。这个特性支持嵌套，并允许我们从复杂的结构体实例中提取值和子结构体。真不错！

然而，通常不是通过在类型上匹配然后仅在`match`分支中处理解包的值来完成的。相反，将匹配条件排列整齐是处理类型内允许值的更好方法。Rust 的`match`子句支持为此目的的守卫。**步骤 5**展示了它们的能力。

**步骤 8**和**步骤 9**都展示了之前实现的`match`函数的使用。我们强烈建议您亲自进行一些实验，看看会发生什么变化。类型匹配允许在没有冗长的安全措施或解决方案的情况下实现复杂的架构，这正是我们想要的！

我们已经成功地学习了使用模式匹配的复杂条件。现在，让我们继续下一个菜谱。

# 实现自定义迭代器

优秀语言的真正力量在于它让程序员如何与标准库和通用生态系统中的类型进行集成。做到这一点的一种方法是通过迭代器模式：由四人帮在其《设计模式》一书中定义（Addison-Wesley Professional，1994 年），迭代器是通过一个指针在集合中移动的封装。Rust 在`Iterator`特质之上提供了一系列实现。让我们看看我们如何仅用几行代码利用这种力量。

# 准备工作

我们将为我们在早期菜谱中构建的链表构建一个迭代器。我们建议使用`Chapter01/testing`项目，或者跟随我们构建迭代器的过程。如果你太忙，无法这样做，完整的解决方案可以在`Chapter02/custom-iterators`中找到。这些路径指的是本书的 GitHub 仓库[`github.com/PacktPublishing/Rust-Programming-Cookbook`](https://github.com/PacktPublishing/Rust-Programming-Cookbook)。

# 如何去做...

迭代器通常是它们自己的结构体，由于可能有不同类型（例如，用于返回引用而不是拥有值），因此在架构上也是一个很好的选择：

1.  让我们为`List<T>`的迭代器创建一个结构体：

```rs
pub struct ConsumingListIterator<T>
where
    T: Clone + Sized,
{
    list: List<T>,
}

impl<T> ConsumingListIterator<T>
where
    T: Clone + Sized,
{
    fn new(list: List<T>) -> ConsumingListIterator<T> {
        ConsumingListIterator { list: list }
    }
}
```

1.  到目前为止，这只是一个缺少迭代器应有的所有功能的普通`struct`。它们的定义性质是一个`next()`函数，它前进内部指针并返回它刚刚移动过的值。按照典型的 Rust 风格，返回的值被包裹在一个`Option`中，一旦集合中的项目用完，它就变为`None`。让我们实现`Iterator`特质以获得所有这些功能：

```rs
impl<T> Iterator for ConsumingListIterator<T>
where
    T: Clone + Sized,
{
    type Item = T;

    fn next(&mut self) -> Option<T> {
        self.list.pop_front()
    }
}
```

1.  目前，我们可以实例化`ConsumingListIterator`并将我们的`List`实例传递给它，这样它就能很好地工作。然而，这还远非无缝集成！Rust 标准库提供了一个额外的特质来实现`IntoIterator`。通过实现这个特质的函数，即使是`for`循环也知道该怎么做，它看起来就像任何其他集合，并且可以轻松互换：

```rs
impl<T> IntoIterator for List<T>
where
    T: Clone + Sized,
{
    type Item = T;
    type IntoIter = ConsumingListIterator<Self::Item>;

    fn into_iter(self) -> Self::IntoIter {
        ConsumingListIterator::new(self)
    }
}
```

1.  最后，我们需要编写一个测试来证明一切正常工作。让我们将其添加到现有的测试套件中：

```rs

    fn new_list(n: usize, value: Option<usize>) -> List<usize>{
        let mut list = List::new_empty();
        for i in 1..=n {
            if let Some(v) = value {
                list.append(v);
            } else {
                list.append(i);
            }
        }
        return list;
    }

    #[test]
    fn test_list_iterator() {
        let list = new_list(4, None);
        assert_eq!(list.length, 4);

        let mut iter = list.into_iter();
        assert_eq!(iter.next(), Some(1));
        assert_eq!(iter.next(), Some(2));
        assert_eq!(iter.next(), Some(3));
        assert_eq!(iter.next(), Some(4));
        assert_eq!(iter.next(), None);

        let list = new_list(4, Some(1));
        assert_eq!(list.length, 4);

        for item in list {
            assert_eq!(item, 1);
        } 

        let list = new_list(4, Some(1));
        assert_eq!(list.length, 4);
        assert_eq!(list.into_iter().fold(0, |s, e| s + e), 4);
    }
```

1.  运行测试将展示这种集成工作得有多好。`cargo test`命令的输出展示了这一点：

```rs
$ cargo test
    Finished dev [unoptimized + debuginfo] target(s) in 0.02s
     Running target/debug/deps/custom_iterators-77e564edad00bd16

running 7 tests
test tests::bench_list_append ... ok
test tests::test_list_append ... ok
test tests::test_list_new_empty ... ok
test tests::test_list_split ... ok
test tests::test_list_iterator ... ok
test tests::test_list_split_panics ... ok
test tests::test_list_pop_front ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests custom-iterators

running 5 tests
test src/lib.rs - List (line 52) ... ignored
test src/lib.rs - List<T>::append (line 107) ... ok
test src/lib.rs - List<T>::new_empty (line 80) ... ok
test src/lib.rs - List<T>::pop_front (line 134) ... ok
test src/lib.rs - List<T>::split (line 173) ... ok

test result: ok. 4 passed; 0 failed; 1 ignored; 0 measured; 0 filtered out
```

下一个部分将更深入地探讨幕后发生的事情！

# 它是如何工作的...

迭代器是向自定义数据结构提供高级功能的好方法。凭借它们简单统一的接口，集合类型也可以轻松切换，程序员不必为每个数据结构都适应新的 API。

通过在*步骤 1*和*步骤 2*中实现`Iterator`特质，可以很容易地提供对集合元素的精确访问级别。在这个菜谱的案例中（类似于`Vec<T>`），它将完全消耗列表并逐个移除项目，从前面开始。

在 *步骤 3* 中，我们实现了 `IntoIterator`，这是一个使这种结构可用于 `for` 循环和其他调用 `into_iter()` 的用户的特例。并非每个集合都实现了这个特例以提供多个不同的迭代器；例如，`Vec<T>` 的第二个迭代器是基于引用的，并且只能通过类型上的 `iter()` 函数访问。顺便说一句，引用是一种数据类型，就像实际实例一样，所以在这种情况下，一切都关于类型定义。这些定义是在特例实现中通过 `type Item` 声明（所谓的**关联类型**：[`doc.rust-lang.org/rust-by-example/generics/assoc_items/types.html`](https://doc.rust-lang.org/rust-by-example/generics/assoc_items/types.html))完成的。这些类型被称为关联类型，可以使用 `Self::Item` 来引用——就像泛型一样，但没有额外的语法冗余。

使用这些接口，你可以访问一个大型函数库，这些函数只假设存在一个正在工作的迭代器！查看 *步骤 4* 和 *步骤 5* 以查看使用迭代器在新建列表类型上的实现和结果。

我们已经成功学习了如何实现自定义迭代器。现在，让我们继续下一个示例。

# 高效地过滤和转换序列

在上一个示例中，我们讨论了实现自定义迭代器，现在是时候利用它们提供的函数了。迭代器可以一次性转换、过滤、归约或简单地转换底层元素，从而使其成为一个非常高效的尝试。

# 准备工作

首先，使用 `cargo new iteration --lib` 创建一个新的项目，并将以下内容添加到项目目录中新建的 `Cargo.toml` 文件：

```rs
[dev-dependencies]
rand = "⁰.5"
```

这将在项目中添加对 `rand` ([`github.com/rust-random/rand`](https://github.com/rust-random/rand)) 包的依赖，首次运行 `cargo test` 时将安装。在 Visual Studio Code 中打开整个项目（或 `src/lib.rs` 文件）。

# 如何实现...

通过四个简单的步骤，我们就能在 Rust 中过滤和转换集合：

1.  为了使用迭代器，你首先必须检索它！让我们这样做并实现一个测试，快速展示迭代器在常规 Rust `Vec<T>` 上的工作方式：

```rs
    #[test]
    fn getting_the_iterator() {
        let v = vec![10, 10, 10];
        let mut iter = v.iter();
        assert_eq!(iter.next(), Some(&10));
        assert_eq!(iter.next(), Some(&10));
        assert_eq!(iter.next(), Some(&10));
        assert_eq!(iter.next(), None);

        for i in v {
            assert_eq!(i, 10);
        }
    }
```

1.  添加了一个测试后，让我们进一步探讨迭代器函数的概念。它们是可组合的，并允许你在单个迭代中执行多个步骤（想想在单个 `for` 循环中添加更多东西）。此外，结果类型可以与开始时完全不同！以下是添加到项目中以执行一些数据转换的另一个测试：

```rs
    fn count_files(path: &String) -> usize {
        path.len()
    }

    #[test]
    fn data_transformations() {
        let v = vec![10, 10, 10];
        let hexed = v.iter().map(|i| format!("{:x}", i));
        assert_eq!(
            hexed.collect::<Vec<String>>(),
            vec!["a".to_string(), "a".to_string(), "a".to_string()]
        );
        assert_eq!(v.iter().fold(0, |p, c| p + c), 30);
        let dirs = vec![
            "/home/alice".to_string(),
            "/home/bob".to_string(),
            "/home/carl".to_string(),
            "/home/debra".to_string(),
        ];

        let file_counter = dirs.iter().map(count_files);

        let dir_file_counts: Vec<(&String, usize)> = 
        dirs.iter().zip(file_counter).collect();

        assert_eq!(
            dir_file_counts,
            vec![
                (&"/home/alice".to_string(), 11),
                (&"/home/bob".to_string(), 9),
                (&"/home/carl".to_string(), 10),
                (&"/home/debra".to_string(), 11)
            ]
        )
    }
```

1.  作为最后一步，让我们也看看一些过滤和拆分。在我们的个人经验中，这些证明是最有用的——它们大大减少了代码的冗余。以下是一些代码：

```rs
    #[test]
    fn data_filtering() {
        let data = vec![1, 2, 3, 4, 5, 6, 7, 8];
        assert!(data.iter().filter(|&n| n % 2 == 0).all(|&n| n % 2 
        == 0));

        assert_eq!(data.iter().find(|&&n| n == 5), Some(&5));
        assert_eq!(data.iter().find(|&&n| n == 0), None);
        assert_eq!(data.iter().position(|&n| n == 5), Some(4));

        assert_eq!(data.iter().skip(1).next(), Some(&2));
        let mut data_iter = data.iter().take(2);
        assert_eq!(data_iter.next(), Some(&1));
        assert_eq!(data_iter.next(), Some(&2));
        assert_eq!(data_iter.next(), None);

        let (validation, train): (Vec<i32>, Vec<i32>) = data
            .iter()
            .partition(|&_| (rand::random::<f32>() % 1.0) > 0.8);

        assert!(train.len() > validation.len());
    }
```

1.  和往常一样，我们希望看到示例工作！运行 `cargo test` 来实现这一点：

```rs
$ cargo test
   Compiling libc v0.2.50
   Compiling rand_core v0.4.0
   Compiling iteration v0.1.0 (Rust-Cookbook/Chapter02/iteration)
   Compiling rand_core v0.3.1
   Compiling rand v0.5.6
    Finished dev [unoptimized + debuginfo] target(s) in 5.44s
     Running target/debug/deps/iteration-a23e5d58a97c9435

running 3 tests
test tests::data_transformations ... ok
test tests::getting_the_iterator ... ok
test tests::data_filtering ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests iteration

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

你想知道更多吗？让我们看看它是如何工作的。

# 它是如何工作的...

Rust 的迭代器受到了函数式编程语言的强烈启发，这使得它们非常便于使用。作为一个迭代器，每个操作都是逐个元素顺序应用的，但仅当迭代器被向前移动时。本食谱中展示了多种类型的操作。其中最重要的如下：

+   `map()`操作执行值或类型的转换，它们非常常见且易于使用。

+   `filter()`与许多类似操作一样，执行一个谓词（一个返回布尔值的函数），以确定一个元素是否应该包含在输出中。例如`find()`、`take_while()`、`skip_while()`和`any()`。

+   聚合函数，如`fold()`、`sum()`、`min()`和`max()`，用于将整个迭代器的所有内容缩减为一个单一的对象。这可能是一个数字（例如`sum()`）或者一个哈希表（例如，通过使用`fold()`）。

+   `chain()`、`zip()`、`fuse()`以及许多其他操作将迭代器组合起来，以便可以在单个循环中迭代。通常，我们使用这些操作是因为在其他情况下可能需要多次遍历。

这种更函数式的编程风格不仅减少了需要编写的代码量，还充当了一个通用词汇表：在条件适用时，不是通过遍历整个将项目推入先前定义列表的`for`循环，而是对`filter()`的函数调用告诉读者可以期待什么。*步骤 2*和*步骤 3*展示了根据不同用例转换（*步骤 2*）或过滤（*步骤 3*）集合的不同函数调用。

此外，迭代器可以被链式连接起来，因此对`iterator.filter().map().fold()`的调用并不罕见，并且通常比执行相同操作的循环更容易理解。作为最后一步，大多数迭代器都被收集到它们的目标集合或变量类型中。`collect()`评估整个链，这意味着它的执行是昂贵的。由于整个主题非常具体于当前的任务，请查看我们编写的代码和结果/调用，以充分利用它。*步骤 4*只展示了运行测试，但真正的故事在代码中。

完成！我们已经成功学习了如何高效地过滤和转换序列。继续学习下一个食谱，了解更多内容！

# 以不安全的方式读取内存。

`unsafe`是 Rust 中的一个概念，其中一些编译器安全机制被关闭。这些**超能力**使 Rust 更接近 C 语言操纵（几乎）任意内存部分的能力。`unsafe`本身使一个作用域（或函数）能够使用这四种超能力（参见[`doc.rust-lang.org/book/ch19-01-unsafe-rust.html`](https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html)）：

+   取消原始指针的引用。

+   调用一个`unsafe`函数或方法。

+   访问或修改一个可变静态变量。

+   实现一个不安全的特性。

在大多数项目中，`unsafe` 只需要用于使用 **FFI**（即 **Foreign Function Interface**），因为它超出了借用检查器的范围。无论如何，在这个菜谱中，我们将探索一些读取内存的 `unsafe` 方法。

# 如何做到这一点...

只需几个步骤，我们就会进入 `unsafe` 状态：

1.  使用 `cargo new unsafe-ways --lib` 创建一个新的库项目。使用 Visual Studio Code 或其他编辑器打开该项目。

1.  打开 `src/libr.rs` 在测试模块之前添加以下函数：

```rs
#![allow(dead_code)]
use std::slice;

fn split_into_equal_parts<T>(slice: &mut [T], parts: usize) -> Vec<&mut [T]> {
    let len = slice.len();
    assert!(parts <= len);
    let step = len / parts;
    unsafe {
        let ptr = slice.as_mut_ptr();

        (0..step + 1)
            .map(|i| {
                let offset = (i * step) as isize;
                let a = ptr.offset(offset);
                slice::from_raw_parts_mut(a, step)
            })
            .collect()
    }
}
```

1.  准备就绪后，我们现在必须在 `mod tests {}` 中添加一些测试：

```rs

#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn test_split_into_equal_parts() {
        let mut v = vec![1, 2, 3, 4, 5, 6];
        assert_eq!(
            split_into_equal_parts(&mut v, 3),
            &[&[1, 2], &[3, 4], &[5, 6]]
        );
    }
}
```

1.  回忆一下 `unsafe` 的超级能力，我们可以尝试改变读取内存的方式。让我们添加这个测试来看看它是如何工作的：

```rs
#[test]
fn test_str_to_bytes_horribly_unsafe() {
    let bytes = unsafe { std::mem::transmute::<&str, &[u8]>("Going 
               off the menu") };
    assert_eq!(
        bytes,
            &[
                71, 111, 105, 110, 103, 32, 111, 102, 102, 32, 116, 
                104, 101, 32, 109, 101, 110, 117
            ]
        );
    }

```

1.  最后一步是在运行 `cargo test` 后查看积极的测试结果：

```rs
$ cargo test
   Compiling unsafe-ways v0.1.0 (Rust-Cookbook/Chapter02/unsafe-ways)
    Finished dev [unoptimized + debuginfo] target(s) in 0.41s
     Running target/debug/deps/unsafe_ways-e7a1d3ffcc456d53

running 2 tests
test tests::test_str_to_bytes_horribly_unsafe ... ok
test tests::test_split_into_equal_parts ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests unsafe-ways

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

安全性是 Rust 中的一个重要概念，让我们来看看使用 `unsafe` 我们会做出哪些权衡。

# 它是如何工作的...

虽然 `unsafe` 是一种在某种程度上使解决方案更简单的方法，但这本书 ([`rust-unofficial.github.io/too-many-lists/index.html`](https://rust-unofficial.github.io/too-many-lists/index.html)) 通过像链表这样简单的东西完美地描述了安全编程的限制。

Rust 是一种安全的编程语言，这意味着编译器会确保所有内存都被计算在内。因此，程序不可能获得同一内存地址的多个可变引用，使用已释放的内存，或者出现类型安全问题等。这使 Rust 能够避免未定义的行为。然而，对于一些有限的使用场景，这些约束禁止了有效的使用案例，这就是为什么 `unsafe` 会放宽一些保证，以适应一些只有 C 语言才允许的操作。

在 *步骤 1* 中设置好项目后，我们在 *步骤 2* 中添加第一个函数。它的目的是类似于 `chunks()` ([`doc.rust-lang.org/std/primitive.slice.html#method.chunks_mut`](https://doc.rust-lang.org/std/primitive.slice.html#method.chunks_mut))，但与迭代器不同，我们立即返回整个集合，这在示例中是可以的，但在实现生产版本时应该考虑。我们的函数将提供的（可变）切片分割成 `parts` 个大小相等的块，并返回对它们的可变引用。由于输入也是对整个内存部分的可变引用，因此我们将有 `parts + 1` 个对同一内存区域的可变引用；显然，这是违反了安全的 Rust！此外，这个函数还允许使用 `ptr.offset()` 调用超出分配的内存（这涉及到指针算术）。

在 *步骤 3* 中创建的测试中，我们展示了它编译和执行没有出现任何重大问题。*步骤 4* 提供了另一个不安全代码的例子：不进行类型转换更改数据类型。`transmute` ([`doc.rust-lang.org/std/mem/fn.transmute.html`](https://doc.rust-lang.org/std/mem/fn.transmute.html)) 函数可以轻松地更改变量的数据类型，并带来所有随之而来的后果。如果我们把类型改为其他类型，比如 `u64`，我们最终会得到一个完全不同的结果，并读取不属于程序的内存。在 *步骤 5* 中，我们运行整个测试套件。

`unsafe` Rust 可以用来从数据结构中获取最后一点性能，进行一些神奇的二进制打包，或者实现 `Send` 和 `Sync` ([`doc.rust-lang.org/std/mem/fn.transmute.html`](https://doc.rust-lang.org/std/mem/fn.transmute.html))。无论你打算用 `unsafe` 做什么，查看 nomicon ([`doc.rust-lang.org/nightly/nomicon/`](https://doc.rust-lang.org/nightly/nomicon/)) 以深入了解其深度。

拥有了这些知识，让我们继续下一个菜谱。

# 共享所有权

所有权和借用是 Rust 的基本概念；它们是无需运行时垃圾回收的原因。作为一个快速入门：它们是如何工作的？简而言之：作用域。Rust（以及许多其他语言）使用（嵌套）作用域来确定变量的有效性，因此它不能在作用域之外使用（如函数）。在 Rust 中，这些作用域 *拥有* 它们的变量，因此当作用域结束时它们将消失。为了使程序能够 *移动* 值，它可以将其所有权转移到嵌套作用域或返回给父作用域。

对于临时转移（和多用户查看），Rust 有 **借用**，它创建了一个指向拥有值的引用。然而，这些引用功能较弱，有时更复杂，难以维护（例如，引用能否比原始值存活得更久？），这可能是编译器抱怨的原因。

在这个菜谱中，我们通过使用引用计数器来共享所有权，只有当计数器达到零时才会丢弃变量，来解决这个问题。

# 准备工作

使用 `new sharing-ownership --lib` 创建一个新的库项目，并在你最喜欢的编辑器中打开该目录。我们还将使用 `nightly` 编译器进行基准测试，因此强烈建议运行 `rustup default nightly`。

要启用基准测试，请将 `#![feature(test)]` 添加到 `lib.rs` 文件的顶部。

# 如何做到这一点...

理解共享所有权只需要八个步骤：

1.  在相对年轻的 Rust 生态系统里，API 和函数签名并不总是最有效的，尤其是在它们需要一定的内存布局知识时。因此，考虑一个简单的 `length` 函数（将其添加到 `mod tests` 范围内）：

```rs
    /// 
    /// A length function that takes ownership of the input 
    /// variable
    /// 
    fn length(s: String) -> usize {
        s.len()
    } 
```

虽然不是必需的，但该函数要求你将你的拥有变量传递到作用域内。

1.  幸运的是，如果你在函数调用后仍然需要所有权，`clone()` 函数已经为你准备好了。顺便说一句，这类似于一个循环，在第一次迭代中所有权被移动，这意味着在第二次迭代时它就**消失了**——导致编译器错误。让我们添加一个简单的测试来展示这些移动：

```rs
    #[test]
    fn cloning() {
        let s = "abcdef".to_owned();
        assert_eq!(length(s), 6);
        // s is now "gone", we can't use it anymore
        // therefore we can't use it in a loop either!
        // ... unless we clone s - at a cost! (see benchmark)
        let s = "abcdef".to_owned();

        for _ in 0..10 {
            // clone is typically an expensive deep copy
            assert_eq!(length(s.clone()), 6);
        }
    }
```

1.  这可行，但会创建大量字符串的副本，然后很快就会丢弃它们。这会导致资源浪费，并且对于足够大的字符串，会减慢程序的速度。为了建立一个基线，让我们通过添加基准测试来检查这一点：

```rs

    extern crate test;
    use std::rc::Rc;
    use test::{black_box, Bencher};

    #[bench]
    fn bench_string_clone(b: &mut Bencher) {
        let s: String = (0..100_000).map(|_| 'a').collect();
        b.iter(|| {
            black_box(length(s.clone()));
        });
    }
```

1.  一些 API 需要输入变量的所有权，但没有语义意义。例如，*步骤 1*中的 `length` 函数假装需要变量所有权，但除非可变性也是必要的，否则 Rust 的 `std::rc::Rc`（简称**引用计数**）类型是一个很好的选择，可以避免重量级克隆或从调用范围中移除所有权。让我们通过创建一个更好的 `length` 函数来试试看：

```rs

    ///
    /// The same length function, taking ownership of a Rc
    /// 
    fn rc_length(s: Rc<String>) -> usize {
        s.len() // calls to the wrapped object require no additions 
    }
```

1.  我们现在可以在将类型传递到函数之后继续使用 `owned` 类型：

```rs
     #[test]
    fn refcounting() {
        let s = Rc::new("abcdef".to_owned());
        // we can clone Rc (reference counters) with low cost
        assert_eq!(rc_length(s.clone()), 6);

        for _ in 0..10 {
            // clone is typically an expensive deep copy
            assert_eq!(rc_length(s.clone()), 6);
        }
    }
```

1.  在我们创建了一个基线基准测试之后，我们当然想知道 `Rc` 版本的表现如何：

```rs
    #[bench]
    fn bench_string_rc(b: &mut Bencher) {
        let s: String = (0..100_000).map(|_| 'a').collect();
        let rc_s = Rc::new(s);
        b.iter(|| {
            black_box(rc_length(rc_s.clone()));
        });
    }
```

1.  首先，我们应该通过运行 `cargo test` 来检查实现是否正确：

```rs
$ cargo test
   Compiling sharing-ownership v0.1.0 (Rust-
   Cookbook/Chapter02/sharing-ownership)
    Finished dev [unoptimized + debuginfo] target(s) in 0.81s
     Running target/debug/deps/sharing_ownership-f029377019c63d62

running 4 tests
test tests::cloning ... ok
test tests::refcounting ... ok
test tests::bench_string_rc ... ok
test tests::bench_string_clone ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests sharing-ownership

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

1.  现在，我们可以检查哪种变体更快，以及它们之间的区别：

```rs
$ cargo bench
   Compiling sharing-ownership v0.1.0 (Rust-
   Cookbook/Chapter02/sharing-ownership)
    Finished release [optimized] target(s) in 0.54s
     Running target/release/deps/sharing_ownership-68bc8eb23caa9948

running 4 tests
test tests::cloning ... ignored
test tests::refcounting ... ignored
test tests::bench_string_clone ... bench: 2,703 ns/iter (+/- 289)
test tests::bench_string_rc ... bench: 1 ns/iter (+/- 0)

test result: ok. 0 passed; 0 failed; 2 ignored; 2 measured; 0 filtered out
```

在我们探索了 `Rc` 的共享所有权之后，让我们深入了解它们。

# 它是如何工作的...

令人印象深刻的基准测试结果并非偶然：`Rc` 对象是堆上位置的智能指针，尽管我们仍然调用 `clone` 来进行**深拷贝**，但 `Rc` 只会复制一个指针并增加对其的引用计数。而实际示例函数保持简单，这样我们就不必担心它，但它确实具有我们经常遇到的复杂函数的所有特性。我们在*步骤 1*中定义了第一个版本，它只适用于拥有内存（输入参数不是引用），在*步骤 2*和*步骤 3*中展示了*步骤 1*中选择的 API 的后果：如果我们想保留（传递数据的）数据副本，我们需要调用 `clone` 函数。

在*步骤 4*到*步骤 6*中，我们使用 Rust 中的一个称为 `Rc` 的构造函数做等效操作。拥有其中之一意味着你拥有指针位置的所有权，但不是实际值，这使得整个构造非常轻量。实际上，一次为原始值分配内存并从多个位置指向它是提高需要大量移动字符串的应用程序性能的常见方法。这在*步骤 7*和*步骤 8*中可以观察到，在那里我们执行测试和基准测试。

仍然有一个注意事项。`Rc` 构造函数不允许可变所有权，这个问题我们将在下一个菜谱中解决。

# 共享可变所有权

共享所有权对于只读数据来说很棒。然而，有时需要可变性，Rust 提供了一种实现这一点的绝佳方式。如果你还记得所有权和借用规则，如果有一个可变引用，它必须是唯一的引用，以避免异常。

这通常是借用检查器介入的地方：在编译时，它确保条件成立。这是 Rust 引入内部可变性模式的地方。通过将数据包装进`RefCell`或`Cell`类型的对象中，可以动态地分配不可变和可变访问。让我们看看这在实践中是如何工作的。

# 准备工作

使用`cargo new --lib mut-shared-ownership`创建一个新的库项目，并在你喜欢的编辑器中打开`src/lib.rs`。为了启用基准测试，请使用`rustup default nightly`切换到`nightly` Rust，并在`lib.rs`文件的顶部添加`#![feature(test)]`（这有助于使用基准测试类型所需的类型）。

# 如何做...

让我们创建一个测试，以在几个步骤中确定共享可变所有权的最佳方式：

1.  让我们在测试模块内部创建几个新函数：

```rs
    use std::cell::{Cell, RefCell};
    use std::borrow::Cow;
    use std::ptr::eq;

    fn min_sum_cow(min: i32, v: &mut Cow<[i32]>) {
        let sum: i32 = v.iter().sum();
        if sum < min {
            v.to_mut().push(min - sum);
        }
    }

    fn min_sum_refcell(min: i32, v: &RefCell<Vec<i32>>) {
        let sum: i32 = v.borrow().iter().sum();
        if sum < min {
            v.borrow_mut().push(min - sum);
        }
    }

    fn min_sum_cell(min: i32, v: &Cell<Vec<i32>>) {
        let mut vec = v.take();
        let sum: i32 = vec.iter().sum();
        if sum < min {
            vec.push(min - sum);
        }
        v.set(vec);
    }
```

1.  这些函数根据传入的数据动态地（基于特定条件，如总和至少需要是*X*）修改整数列表，并依赖于三种共享可变所有权的方式。让我们探索这些在外部是如何表现的！`Cell`对象（以及`RefCell`对象）只是返回值的引用或所有权的包装器：

```rs
    #[test]
    fn about_cells() {
        // we allocate memory and use a RefCell to dynamically
        // manage ownership
        let ref_cell = RefCell::new(vec![10, 20, 30]);

        // mutable borrows are fine,
        min_sum_refcell(70, &ref_cell);

        // they are equal!
        assert!(ref_cell.borrow().eq(&vec![10, 20, 30, 10]));

        // cells are a bit different
        let cell = Cell::from(vec![10, 20, 30]);

        // pass the immutable cell into the function
        min_sum_cell(70, &cell);

        // unwrap
        let v = cell.into_inner();

        // check the contents, and they changed!
        assert_eq!(v, vec![10, 20, 30, 10]);
    }
```

1.  由于这与其他编程语言非常相似，在这些语言中可以自由传递引用，我们也应该知道其注意事项。一个重要的方面是，如果借用检查失败，这些`Cell`线程会引发恐慌，这至少会使当前线程突然停止。在几行代码中，这看起来是这样的：

```rs
    #[test]
    #[should_panic]
    fn failing_cells() {
        let ref_cell = RefCell::new(vec![10, 20, 30]);

        // multiple borrows are fine
        let _v = ref_cell.borrow();
        min_sum_refcell(60, &ref_cell);

        // ... until they are mutable borrows
        min_sum_refcell(70, &ref_cell); // panics!
    }
```

1.  直观地看，这些单元格应该会增加运行时开销，因此比常规的预编译借用检查要慢。为了确认这一点，让我们添加一个基准测试：

```rs
    extern crate test;
    use test::{ Bencher};

    #[bench]
    fn bench_regular_push(b: &mut Bencher) {
        let mut v = vec![];
        b.iter(|| {
            for _ in 0..1_000 {
                v.push(10);
            }
        });
    }

    #[bench]
    fn bench_refcell_push(b: &mut Bencher) {
        let v = RefCell::new(vec![]);
        b.iter(|| {
            for _ in 0..1_000 {
                v.borrow_mut().push(10);
            }
        });
    }

    #[bench]
    fn bench_cell_push(b: &mut Bencher) {
        let v = Cell::new(vec![]);
        b.iter(|| {
            for _ in 0..1_000 {
                let mut vec = v.take();
                vec.push(10);
                v.set(vec);
            }
        });
    }
```

1.  然而，我们没有解决`Cell`中未预见的恐慌的危险，这在复杂的应用中可能是禁止性的。这就是`Cow`出现的地方。`Cow`是一种**写时复制（Copy-on-Write）**类型，如果请求可变访问，它会通过惰性克隆来替换它所包装的值。通过使用这个`struct`，我们可以确保避免使用此代码时的恐慌：

```rs
    #[test]
    fn handling_cows() {
        let v = vec![10, 20, 30];

        let mut cow = Cow::from(&v);
        assert!(eq(&v[..], &*cow));

        min_sum_cow(70, &mut cow);

        assert_eq!(v, vec![10, 20, 30]);
        assert_eq!(cow, vec![10, 20, 30, 10]);
        assert!(!eq(&v[..], &*cow));

        let v2 = cow.into_owned();

        let mut cow2 = Cow::from(&v2);
        min_sum_cow(70, &mut cow2);

        assert_eq!(cow2, v2);
        assert!(eq(&v2[..], &*cow2));
    }
```

1.  最后，让我们通过运行`cargo test`来验证测试和基准是否成功：

```rs
$ cargo test
   Compiling mut-sharing-ownership v0.1.0 (Rust-
   Cookbook/Chapter02/mut-sharing-ownership)
    Finished dev [unoptimized + debuginfo] target(s) in 0.81s
     Running target/debug/deps/mut_sharing_ownership-
     d086077040f0bd34

running 6 tests
test tests::about_cells ... ok
test tests::bench_cell_push ... ok
test tests::bench_refcell_push ... ok
test tests::failing_cells ... ok
test tests::handling_cows ... ok
test tests::bench_regular_push ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests mut-sharing-ownership

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

1.  让我们看看`cargo bench`输出的基准时间：

```rs
$ cargo bench
    Finished release [optimized] target(s) in 0.02s
     Running target/release/deps/mut_sharing_ownership-
     61f1f68a32def1a8

running 6 tests
test tests::about_cells ... ignored
test tests::failing_cells ... ignored
test tests::handling_cows ... ignored
test tests::bench_cell_push ... bench: 10,352 ns/iter (+/- 595)
test tests::bench_refcell_push ... bench: 3,141 ns/iter (+/- 6,389)
test tests::bench_regular_push ... bench: 3,341 ns/iter (+/- 124)

test result: ok. 0 passed; 0 failed; 3 ignored; 3 measured; 0 filtered out
```

以各种方式共享内存是复杂的，所以让我们更深入地了解它们是如何工作的。

# 它是如何工作的...

这个食谱就像一个大的基准或测试方案：在*步骤 1*中，我们定义要测试的函数，每个函数都有不同的输入参数，但具有相同的行为；它将`Vec`填充到最小总和。这些参数反映了不同的共享所有权方式，包括`RefCell`、`Cell`和`Cow`。

*步骤 2*和*步骤 3*创建了仅针对`RefCell`和`Cell`采用的不同方式处理和失败这些值的测试。*步骤 5*与`Cow`类型类似；这些都是测试您自己理论的绝佳机会！

在*步骤 4*和*步骤 6*中，我们正在创建和运行我们在本菜谱中创建的函数的基准测试和测试。结果是令人惊讶的。事实上，我们尝试了不同的计算机和版本，并得出相同的结论：`RefCell`几乎与常规方式检索可变引用（运行时行为导致更高的方差）一样快。`Cell`参数的减速也是预期的；它们在每次迭代中都将整个数据移入和移出——这也是我们可以从`Cow`期望的，所以请随意尝试它。

`Cell`对象和`RefCell`对象将数据移动到堆内存，并使用引用（指针）来访问这些值，通常需要额外的跳跃。然而，它们提供了与 C#、Java 或其他类似语言相似的方式来移动对象引用，同时提供了舒适度。

我们希望您已经成功学习了关于共享可变所有权的知识。现在，让我们继续到下一个菜谱。

# 使用显式生命周期进行引用

生命周期在许多语言中都很常见，通常决定一个变量是否可以在作用域外使用。在 Rust 中，由于借用和所有权模型广泛使用生命周期和作用域来自动管理内存，所以情况要复杂一些。我们开发者想要避免保留内存并将内容克隆到其中的低效和潜在的减速，而不是使用引用。然而，这会导致一条棘手的路径，因为当原始值超出作用域时，引用会发生什么？

由于编译器无法从代码中推断出这些信息，您必须帮助它并注释代码，以便它可以检查正确的使用。让我们看看这会是什么样子。

# 如何做到这一点...

生命周期可以通过几个步骤来探索：

1.  使用`cargo new lifetimes --lib`创建一个新的项目，并在您喜欢的编辑器中打开它。

1.  让我们从一个非常简单的函数开始，该函数接受一个可能不会比函数长寿的引用！让我们确保函数和输入参数具有相同的生命周期：

```rs
// declaring a lifetime is optional here, since the compiler automates this

///
/// Compute the arithmetic mean
/// 
pub fn mean<'a>(numbers: &'a [f32]) -> Option<f32> {
    if numbers.len() > 0 {
        let sum: f32 = numbers.iter().sum();
        Some(sum / numbers.len() as f32)
    } else {
        None
    }
} 
```

1.  在结构体中需要声明生命周期。因此，我们首先定义基础`struct`。它为包含的类型提供了生命周期注解：

```rs
///
/// Our almost generic statistics toolkit
/// 
pub struct StatisticsToolkit<'a> {
    base: &'a [f64],
}
```

1.  接下来是实现，它继续了生命周期规范。首先，我们实现构造函数（`new()`）：

```rs
impl<'a> StatisticsToolkit<'a> {

    pub fn new(base: &'a [f64]) -> 
     Option<StatisticsToolkit> {
        if base.len() < 3 {
            None
        } else {
            Some(StatisticsToolkit { base: base })
        }
    }
```

然后，我们希望实现方差计算以及标准差和平均值：

```rs
    pub fn var(&self) -> f64 {
        let mean = self.mean();

        let ssq: f64 = self.base.iter().map(|i| (i - 
        mean).powi(2)).sum();
        return ssq / self.base.len() as f64;
    }

    pub fn std(&self) -> f64 {
        self.var().sqrt()
    }

    pub fn mean(&self) -> f64 {
        let sum: f64 = self.base.iter().sum();

        sum / self.base.len() as f64
    }
```

作为最后的操作，我们添加了中位数计算：

```rs
    pub fn median(&self) -> f64 {
        let mut clone = self.base.to_vec();

        // .sort() is not implemented for floats
        clone.sort_by(|a, b| a.partial_cmp(b).unwrap()); 

        let m = clone.len() / 2;
        if clone.len() % 2 == 0 {
            clone[m]
        } else {
            (clone[m] + clone[m - 1]) / 2.0
        }
    }
}
```

1.  就这样！需要一些测试以确保我们可以确信一切按预期工作。让我们从几个辅助函数和计算平均值的测试开始：

```rs
#[cfg(test)]
mod tests {

    use super::*;

    ///
    /// a normal distribution created with numpy, with mu = 
    /// 42 and 
    /// sigma = 3.14 
    /// 
    fn numpy_normal_distribution() -> Vec<f64> {
        vec![
            43.67221552, 46.40865622, 43.44603147, 
            43.16162571, 
            40.94815816, 44.585914 , 45.84833022, 
            37.77765835, 
            40.23715928, 48.08791899, 44.80964938, 
            42.13753315, 
            38.80713956, 39.16183586, 42.61511209, 
            42.25099062, 
            41.2240736 , 44.59644304, 41.27516889, 
            36.21238554
        ]
    }

    #[test]
    fn mean_tests() {
        // testing some aspects of the mean function
        assert_eq!(mean(&vec![1.0, 2.0, 3.0]), Some(2.0));
        assert_eq!(mean(&vec![]), None);
        assert_eq!(mean(&vec![0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 
        0.0]), 
        Some(0.0));
    }
```

然后，我们对新函数进行一些测试：

```rs
    #[test]
    fn statisticstoolkit_new() {
        // require >= 3 elements in an array for a 
        // plausible normal distribution
        assert!(StatisticsToolkit::new(&vec![]).is_none());
        assert!(StatisticsToolkit::new(&vec![2.0, 
         2.0]).is_none());

        // a working example
        assert!(StatisticsToolkit::new(&vec![1.0, 2.0, 
         1.0]).is_some());

        // not a normal distribution, but we don't mind
        assert!(StatisticsToolkit::new(&vec![2.0, 1.0, 
         2.0]).is_some());
    }
```

接下来，让我们测试实际的统计数据。在一个函数中，我们开始使用一些特殊的数据输入：

```rs
    #[test]
    fn statisticstoolkit_statistics() {
         // simple best case test
        let a_sample = vec![1.0, 2.0, 1.0];
        let nd = StatisticsToolkit::
         new(&a_sample).unwrap();
        assert_eq!(nd.var(), 0.2222222222222222);
        assert_eq!(nd.std(), 0.4714045207910317);
        assert_eq!(nd.mean(), 1.3333333333333333);
        assert_eq!(nd.median(), 1.0);

        // no variance
        let a_sample = vec![1.0, 1.0, 1.0];
        let nd = StatisticsToolkit::
         new(&a_sample).unwrap();
        assert_eq!(nd.var(), 0.0);
        assert_eq!(nd.std(), 0.0);
        assert_eq!(nd.mean(), 1.0);
        assert_eq!(nd.median(), 1.0);
```

为了检查更复杂的数据输入（例如，偏斜分布或边缘情况），让我们进一步扩展测试：

```rs
        // double check with a real library
        let a_sample = numpy_normal_distribution();
        let nd = 
         StatisticsToolkit::new(&a_sample).unwrap();
        assert_eq!(nd.var(), 8.580276516670548);
        assert_eq!(nd.std(), 2.9292109034124785);
        assert_eq!(nd.mean(), 42.36319998250001);
        assert_eq!(nd.median(), 42.61511209);

        // skewed distribution
        let a_sample = vec![1.0, 1.0, 5.0];
        let nd = 
         StatisticsToolkit::new(&a_sample).unwrap();
        assert_eq!(nd.var(), 3.555555555555556);
        assert_eq!(nd.std(), 1.8856180831641267);
        assert_eq!(nd.mean(), 2.3333333333333335);
        assert_eq!(nd.median(), 1.0);

        // median with even collection length
        let a_sample = vec![1.0, 2.0, 3.0, 4.0] ;
        let nd = 
         StatisticsToolkit::new(&a_sample).unwrap();
        assert_eq!(nd.var(), 1.25);
        assert_eq!(nd.std(), 1.118033988749895);
        assert_eq!(nd.mean(), 2.5);
        assert_eq!(nd.median(), 3.0);
    }
}
```

1.  使用`cargo test`运行测试并验证它们是否成功：

```rs
$ cargo test
   Compiling lifetimes v0.1.0 (Rust-Cookbook/Chapter02/lifetimes)
    Finished dev [unoptimized + debuginfo] target(s) in 1.16s
     Running target/debug/deps/lifetimes-69291f4a8f0af715

running 3 tests
test tests::mean_tests ... ok
test tests::statisticstoolkit_new ... ok
test tests::statisticstoolkit_statistics ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests lifetimes

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

与生命周期一起工作很复杂，所以让我们深入了解代码以更好地理解。

# 它是如何工作的...

在这个食谱中，我们创建了一个简单的统计工具箱，它允许快速准确地分析正态分布样本。然而，这个例子只是为了说明生命周期的有用性和相对简单的方法。在*步骤 2*中，我们创建了一个计算给定集合平均值的函数。由于生命周期可以从使用函数/变量中推断出来，因此显式指定生命周期是可选的。尽管如此，该函数明确地将输入参数的生命周期绑定到函数的生命周期，要求任何传入的引用都要比`mean()`长命。

*步骤 3*和*步骤 4*展示了如何在结构体及其实现中处理生命周期。由于类型实例可以很容易地比存储的引用长命（每个甚至可能需要不同的生命周期），因此显式指定生命周期变得必要。生命周期必须在每一步中声明；在结构体声明中，在`impl`块中，以及在使用它们的函数中。生命周期的名称将它们绑定在一起。从某种意义上说，它创建了一个与类型实例的生命周期相关的虚拟作用域。

生命周期注解很有用但很冗长，这使得处理引用有时很繁琐。然而，一旦注解到位，程序可以更加高效，接口可以更加方便，移除`clone()`方法和其他东西。

生命周期名称的选择（`'a'`）是常见的，但也是任意的。除了预定义的`'static'`之外，每个单词都同样适用，而且可读的选择绝对更好。

与显式生命周期一起工作并不太难，对吧？我们建议你继续实验，直到你准备好进入下一个食谱。

# 使用特性界限强制行为

当构建复杂架构时，先决条件行为非常常见。在 Rust 中，这意味着我们无法构建泛型或其他类型，除非它们符合某些先前的行为，换句话说，我们需要能够指定所需的特性。特性界限是实现这一目标的一种方式——即使你之前跳过了许多食谱，你也已经看到了多个这样的例子。

# 如何做到...

按照以下步骤学习更多关于特性的知识：

1.  使用`cargo new trait-bounds`创建一个新的项目，并在你喜欢的编辑器中打开它。

1.  编辑`src/main.rs`以添加以下代码，其中我们可以轻松地打印变量的调试格式，因为编译时需要实现该格式：

```rs
///
/// A simple print function for printing debug formatted variables
/// 
fn log_debug<T: Debug>(t: T) {
    println!("{:?}", t);
}
```

1.  如果我们使用自定义类型，例如 `struct AnotherType(usize)` 来调用它，编译器会很快抱怨：

```rs
$ cargo run
   Compiling trait-bounds v0.1.0 (Rust-Cookbook/Chapter02/trait-bounds)
error[E0277]: `AnotherType` doesn't implement `std::fmt::Debug`
  --> src/main.rs:35:5
   |
35 | log_debug(b);
   | ^^^^^^^^^ `AnotherType` cannot be formatted using `{:?}`
   |
   = help: the trait `std::fmt::Debug` is not implemented for `AnotherType`
   = note: add `#[derive(Debug)]` or manually implement `std::fmt::Debug`
note: required by `log_debug`
  --> src/main.rs:11:1
   |
11 | fn log_debug<T: Debug>(t: T) {
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^

error: aborting due to previous error

For more information about this error, try `rustc --explain E0277`.
error: Could not compile `trait-bounds`.

To learn more, run the command again with --verbose.
```

1.  为了解决这个问题，我们可以实现或推导 `Debug` 特质，正如错误信息中所述。对于标准类型的组合，推导实现是非常常见的。在特质中，特质的界限变得更有趣：

```rs
///
/// An interface that can be used for quick and easy logging
/// 
pub trait Loggable: Debug + Sized {
    fn log(self) {
        println!("{:?}", &self)
    }
}
```

1.  然后，我们可以创建并实现一个合适的数据类型：

```rs
#[derive(Debug)]
struct ArbitraryType {
    v: Vec<i32>
}

impl ArbitraryType {
    pub fn new() -> ArbitraryType {
        ArbitraryType {
            v: vec![1,2,3,4]
        }
    }
}
impl Loggable for ArbitraryType {}
```

1.  接下来，让我们在 `main` 函数中将代码串联起来：

```rs
fn main() {
    let a = ArbitraryType::new();
    a.log();
    let b = AnotherType(2);
    log_debug(b);
}
```

1.  执行 `cargo run` 并确定输出是否符合你的预期：

```rs
$ cargo run
   Compiling trait-bounds v0.1.0 (Rust-Cookbook/Chapter02/trait-
   bounds)
    Finished dev [unoptimized + debuginfo] target(s) in 0.38s
     Running `target/debug/trait-bounds`
     ArbitraryType { v: [1, 2, 3, 4] }
     AnotherType(2)
```

在创建示例程序后，让我们探索特质界限的背景。

# 它是如何工作的...

特质界限指定了实现者实现时的要求。通过这种方式，我们可以在不了解其结构的情况下对泛型类型调用函数。

在 *第 2 步* 中，我们要求任何参数类型都必须实现 `std::fmt::Debug` 特质，以便能够使用调试格式化程序进行打印。然而，这并不很好地推广，我们还需要要求任何 *其他* 函数也实现该特性。这就是为什么在 *第 4 步* 中，我们要求任何实现 `Loggable` 特质的类型也实现 `Debug`。

因此，我们可以在特质的函数中使用所有必需的特质，这使得扩展更加容易，并提供了所有类型实现特质以保持兼容性的能力。在 *第 5 步* 中，我们正在为创建的类型实现 `Loggable` 特质，并在后续步骤中使用它。

关于所需特质的决策对于公共 API 以及编写良好设计和可维护的代码同样重要。注意真正需要哪些类型以及如何提供它们，这将导致更好的接口和类型。注意两个类型界限之间的 `+` 符号；它要求在实现 `Loggable` 时存在（以及更多，如果添加了更多的 `+` 符号）这两个（以及更多）特质。

我们已经成功地学习了如何使用特质界限强制行为。现在，让我们继续下一个菜谱。

# 与泛型数据类型一起工作

Rust 的函数重载比其他语言更奇特。你不需要用不同的类型签名重新定义相同的函数，通过指定泛型实现的实际类型，你可以达到相同的结果。泛型是提供更通用接口的绝佳方式，由于编译器的有用消息，实现起来并不复杂。

在这个菜谱中，我们将以泛型方式实现动态数组（例如 `Vec<T>`）。

# 如何做到这一点...

只需几个步骤就能学会如何使用泛型：

1.  首先，使用 `cargo new generics --lib` 创建一个新的库项目，并在 Visual Studio Code 中打开项目文件夹。

1.  动态数组是一种你每天都会使用的数据结构。在 Rust 中，它的实现被称为 `Vec<T>`，而其他语言则称之为 `ArrayList` 或 `List`。首先，让我们建立基本结构：

```rs
use std::boxed::Box;
use std::cmp;
use std::ops::Index;

const MIN_SIZE: usize = 10;

type Node<T> = Option<T>;

pub struct DynamicArray<T>
where
    T: Sized + Clone,
{
    buf: Box<[Node<T>]>,
    cap: usize,
    pub length: usize,
}
```

1.  如`struct`定义所示，主要元素是一个类型为`T`的盒子，这是一个泛型类型。让我们看看实现看起来像什么：

```rs
impl<T> DynamicArray<T>
where
    T: Sized + Clone,
{
    pub fn new_empty() -> DynamicArray<T> {
        DynamicArray {
            buf: vec![None; MIN_SIZE].into_boxed_slice(),
            length: 0,
            cap: MIN_SIZE,
        }
    }

    fn grow(&mut self, min_cap: usize) {
        let old_cap = self.buf.len();
        let mut new_cap = old_cap + (old_cap >> 1);

        new_cap = cmp::max(new_cap, min_cap);
        new_cap = cmp::min(new_cap, usize::max_value());
        let current = self.buf.clone();
        self.cap = new_cap;

        self.buf = vec![None; new_cap].into_boxed_slice();
        self.buf[..current.len()].clone_from_slice(&current);
    }

    pub fn append(&mut self, value: T) {
        if self.length == self.cap {
            self.grow(self.length + 1);
        }
        self.buf[self.length] = Some(value);
        self.length += 1;
    }

    pub fn at(&mut self, index: usize) -> Node<T> {
        if self.length > index {
            self.buf[index].clone()
        } else {
            None
        }
    }
}
```

1.  到目前为止，非常直接。我们不会使用类型名称，而是简单地使用`T`。如果我们想为泛型定义实现一个特定类型，会发生什么？让我们为`usize`类型实现`Index`操作（Rust 中的一个特质）。此外，一个`clone`操作在将来会非常有帮助，所以让我们也添加那个：

```rs
impl<T> Index<usize> for DynamicArray<T>
where
    T: Sized + Clone,
{
    type Output = Node<T>;

    fn index(&self, index: usize) -> &Self::Output {
        if self.length > index {
            &self.buf[index]
        } else {
            &None
        }
    }
}

impl<T> Clone for DynamicArray<T>
where
    T: Sized + Clone,
{
    fn clone(&self) -> Self {
        DynamicArray {
            buf: self.buf.clone(),
            cap: self.cap,
            length: self.length,
        }
    }
}
```

1.  为了确保所有这些都能正常工作，我们没有犯任何错误，让我们为每个实现的功能进行一些测试：

```rs

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn dynamic_array_clone() {
        let mut list = DynamicArray::new_empty();
        list.append(3.14);
        let mut list2 = list.clone();
        list2.append(42.0);
        assert_eq!(list[0], Some(3.14));
        assert_eq!(list[1], None);

        assert_eq!(list2[0], Some(3.14));
        assert_eq!(list2[1], Some(42.0));
    }

    #[test]
    fn dynamic_array_index() {
        let mut list = DynamicArray::new_empty();
        list.append(3.14);

        assert_eq!(list[0], Some(3.14));
        let mut list = DynamicArray::new_empty();
        list.append("Hello");
        assert_eq!(list[0], Some("Hello"));
        assert_eq!(list[1], None);
    }
```

现在，让我们添加更多的测试：

```rs
    #[test]
    fn dynamic_array_2d_array() {
        let mut list = DynamicArray::new_empty();
        let mut sublist = DynamicArray::new_empty();
        sublist.append(3.14);
        list.append(sublist);

        assert_eq!(list.at(0).unwrap().at(0), Some(3.14));
        assert_eq!(list[0].as_ref().unwrap()[0], Some(3.14));

    }

    #[test]
    fn dynamic_array_append() {
        let mut list = DynamicArray::new_empty();
        let max: usize = 1_000;
        for i in 0..max {
            list.append(i as u64);
        }
        assert_eq!(list.length, max);
    }

    #[test]
    fn dynamic_array_at() {
        let mut list = DynamicArray::new_empty();
        let max: usize = 1_000;
        for i in 0..max {
            list.append(i as u64);
        }
        assert_eq!(list.length, max);
        for i in 0..max {
            assert_eq!(list.at(i), Some(i as u64));
        }
        assert_eq!(list.at(max + 1), None);
    }
}
```

1.  一旦实现了测试，我们就可以使用`cargo test`成功运行测试：

```rs
$ cargo test
   Compiling generics v0.1.0 (Rust-Cookbook/Chapter02/generics)
    Finished dev [unoptimized + debuginfo] target(s) in 0.82s
     Running target/debug/deps/generics-0c9bbd42843c67d5

running 5 tests
test tests::dynamic_array_2d_array ... ok
test tests::dynamic_array_index ... ok
test tests::dynamic_array_append ... ok
test tests::dynamic_array_clone ... ok
test tests::dynamic_array_at ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests generics

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

现在，让我们看看从幕后使用泛型。

# 它是如何工作的...

泛型在 Rust 中工作得非常好，除了冗长的符号外，它们非常方便。实际上，你会发现它们无处不在，随着你在 Rust 中的进步，对更好、更通用接口的需求也会增加。

在*第 2 步*中，我们正在创建一个修改后的动态数组（来自书籍*《动手学数据结构与算法：Rust 语言实现》*：[`www.packtpub.com/application-development/hands-data-structures-and-algorithms-rust`](https://www.packtpub.com/application-development/hands-data-structures-and-algorithms-rust)），它使用泛型类型。在代码中使用泛型类型就像使用任何其他类型一样，用`T`代替`i32`。然而，如前所述，编译器期望`T`类型具有某些行为，例如实现`Clone`，这在结构体和实现的`where`子句中指定。在更复杂的使用案例中，可能会有多个块，用于`T`实现`Clone`和未实现`Clone`的情况，但这将超出菜谱的范围。*第 3 步*显示了动态数组类型的泛型实现以及`Clone`和`Sized`特质如何发挥作用。

在*第 4 步*实现`Index`特质时，某些事情变得更加明显。首先，我们为特质实现头指定`usize`类型。因此，只有当有人使用`usize`变量（或常量/文字）进行索引时，才会实现此特质，从而排除了任何负值。第二个方面是关联类型，它本身具有泛型类型。

泛型的一个重要方面是术语`Sized`。当大小在编译时已知时，Rust 中的变量是`Sized`的，因此编译器知道要分配多少内存。无尺寸类型在编译时具有未知的大小；也就是说，它们是动态分配的，可能在运行时增长。例如，`str`或类型为`[T]`的切片。它们的实际大小可以改变，这就是为什么它们总是位于一个固定大小的引用之后，即指针。如果需要`Sized`，则只能使用无尺寸类型的引用（`&str`，`&[T]`），但还有`?Sized`来使这种行为可选。

然后，*步骤 5*和*步骤 6*创建一些测试并运行它们。这些测试表明动态数组的主要功能仍然正常工作，我们鼓励你尝试解决其中关于代码的任何疑问。

如果你想了解更多关于动态数组以及它是如何增长（它的大小会翻倍，就像 Java 的`ArrayList`一样）的细节，请查看《Rust 编程中的动手实践数据结构与算法》，在那里可以更详细地解释这个动态数组以及其他数据结构。
