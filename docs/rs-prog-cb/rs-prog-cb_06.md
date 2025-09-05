# 第六章：使用宏表达自己

在上一个世纪，许多语言都包含一个预处理器（最著名的是 C/C++），它通常执行简单的文本替换。虽然这对于表达常量（例如 `#define MYCONST 1`）很有用，但当替换变得复杂时（例如，`#define MYCONST 1 + 1` 并将其应用于 `5 * MYCONST` 得到 *5 * 1 + 1 = 6* 而不是预期的 10（来自 *5 * (1 + 1)*））时，可能会导致意想不到的结果。

然而，预处理器允许程序编程（元编程），因此使开发者的工作变得更简单。通过快速定义宏而不是复制粘贴表达式和过多的样板代码，可以减少代码库的大小，并实现可重用的调用——因此减少了错误。为了充分利用 Rust 的类型系统，宏不能简单地搜索和替换文本；它们必须在更高的层面上工作：抽象语法树。这不仅需要不同的调用语法（例如，在调用末尾使用感叹号；例如，`println!`），而且参数 *类型* 也不同。

在这个层面上，我们讨论的是表达式、语句、标识符、类型以及许多可以被传递给宏的内容。然而，最终，宏预处理器仍然在编译前将宏的主体插入到调用作用域中，因此编译器会捕获类型不匹配或借用违规。如果您想了解更多关于宏的信息，请查看博客文章[`blog.x5ff.xyz/blog/easy-programming-with-rust-macros/`](https://blog.x5ff.xyz/blog/easy-programming-with-rust-macros/)、“Rust 宏小书”（[`danielkeep.github.io/tlborm/book/index.html`](https://danielkeep.github.io/tlborm/book/index.html)）和 Rust 书籍（[`doc.rust-lang.org/book/ch19-06-macros.html`](https://doc.rust-lang.org/book/ch19-06-macros.html)）。最好通过尝试来了解宏——我们将在本章中介绍以下内容：

+   在 Rust 中构建自定义宏

+   使用宏进行匹配

+   使用预定义的 Rust 宏

+   使用宏进行代码生成

+   宏重载

+   使用 `repeat` 为参数范围

+   不要重复自己（DRY）

# 在 Rust 中构建自定义宏

之前，我们主要使用预定义的宏——现在是时候看看如何创建自定义宏了。Rust 中有几种类型的宏——基于 derive 的、函数式和属性，它们各自都有相应的用途。在本食谱中，我们将尝试函数式类型以开始。

# 如何实现...

您只需几个步骤就可以创建宏：

1.  在终端（或在 Windows 上的 PowerShell）中运行 `cargo new custom-macros` 并使用 Visual Studio Code 打开该目录。

1.  在编辑器中打开 `src/main.rs` 文件。让我们在文件顶部创建一个新的宏，命名为 `one_plus_one`：

```rs
// A simple macro without arguments
macro_rules! one_plus_one {
    () => { 1 + 1 };
}
```

1.  让我们称这个位于 `main` 函数内部的简单宏为：

```rs
fn main() {
    println!("1 + 1 = {}", one_plus_one!());
}
```

1.  这是一个非常简单的宏，但宏可以做更多的事情！比如一个让我们决定操作的宏。在文件顶部添加一个非常简单的宏：

```rs
// A simple pattern matching argument
macro_rules! one_and_one {
 (plus) => { 1 + 1 };
 (minus) => { 1 - 1 };
 (mult) => { 1 * 1 };
}
```

1.  由于宏的**匹配器**部分中的单词是必需的，我们必须像那样精确地调用宏。在`main`函数内部添加以下内容：

```rs
    println!("1 + 1 = {}", one_and_one!(plus));
    println!("1 - 1 = {}", one_and_one!(minus));
    println!("1 * 1 = {}", one_and_one!(mult));
```

1.  作为最后一部分，我们应该考虑保持事物的有序；创建模块、结构体、文件等。将类似的行为分组是一种常见的组织方式，如果我们想在模块外部使用它，我们需要使其公开可用。就像`pub`关键字一样，宏必须显式导出——但是使用一个属性。将此模块添加到`src/main.rs`中，如下所示：

```rs
mod macros {
    #[macro_export]
    macro_rules! two_plus_two {
        () => { 2 + 2 };
    }
}
```

1.  多亏了导出，我们现在也可以在`main()`中调用这个函数：

```rs
fn main() {
    println!("1 + 1 = {}", one_plus_one!());
    println!("1 + 1 = {}", one_and_one!(plus));
    println!("1 - 1 = {}", one_and_one!(minus));
    println!("1 * 1 = {}", one_and_one!(mult));
    println!("2 + 2 = {}", two_plus_two!());
}
```

1.  通过在项目目录内的终端中发出`cargo run`命令，我们就可以知道它是否成功了：

```rs
$ cargo run
 Compiling custom-macros v0.1.0 (Rust-Cookbook/Chapter06/custom-
  macros)
 Finished dev [unoptimized + debuginfo] target(s) in 0.66s
 Running `target/debug/custom-macros`
1 + 1 = 2
1 + 1 = 2
1 - 1 = 0
1 * 1 = 1
2 + 2 = 4
```

为了更好地理解代码，让我们来解析这些步骤。

# 它是如何工作的...

根据需要，我们使用宏——`macro_rules!`——来创建自定义宏，就像我们在*步骤 3*中所做的那样。一个单独的宏匹配一个模式，并包含三个部分：

+   一个名称（例如，`one_plus_one`)

+   一个匹配器（例如，`(plus) => ...`)

+   一个转录器（例如，`... => { 1 + 1 }`)

调用一个宏始终是通过它的名称后跟一个感叹号（*步骤 4*）来完成的，对于特定的模式，使用所需的字符/单词（*步骤 6*）。请注意，`plus`和其他的不是变量、类型或以其他方式定义的——这给了您创建自己的**领域特定语言**（**DSL**）的能力！关于这一点，本章的其他菜谱中将有更多介绍。

通过调用一个宏，编译器会注意抽象语法树（**AST**）中的位置，并且不是进行纯文本替换，而是在那里插入宏的转录子树。之后，编译器尝试完成编译，导致常规类型安全检查、借用规则执行等，但考虑到宏。这使得作为开发者的您更容易找到错误并将它们追溯到它们起源的宏。

在*步骤 6*中，我们创建了一个模块来导出一个宏——这将会改善代码结构和可维护性，尤其是在较大的代码库中。然而，导出步骤是必需的，因为宏默认是私有的。尝试移除`#[macro_export]`属性来看看会发生什么。

*步骤 8* 展示了如何将项目中的每个宏变体作为比较来调用。有关更多信息，您还可以查看[`blog.rust-lang.org/2018/12/21/Procedural-Macros-in-Rust-2018.html`](https://blog.rust-lang.org/2018/12/21/Procedural-Macros-in-Rust-2018.html)上的博客文章，该文章更详细地介绍了在`crates.io`上提供宏 crate 的内容 ([`crates.io`](https://crates.io))。

现在我们知道了如何在 Rust 中构建自定义宏，我们可以继续到下一个菜谱。

# 使用宏实现匹配

当我们创建自定义宏时，我们已经看到了模式匹配在起作用：只有当特定的单词在编译前存在时，才会执行命令。换句话说，宏系统在它们成为表达式或类型之前将原始文本作为模式进行比较。因此，创建一个领域特定语言（DSL）非常容易。定义一个网络请求处理器？使用模式中的方法名称：`GET`、`POST`、`HEAD`。然而，种类繁多，所以让我们看看在这个菜谱中我们如何定义一些模式！

# 如何做...

通过遵循接下来的几个步骤，您将能够使用宏：

1.  在终端（或在 Windows 上的 PowerShell）中运行`cargo new matching --lib`，然后用 Visual Studio Code 打开该目录。

1.  在`src/lib.rs`中，我们添加一个宏来处理特定类型的输入。在文件顶部插入以下内容：

```rs
macro_rules! strange_patterns {
    (The pattern must match precisely) => { "Text" };
    (42) => { "Numeric" };
    (;<=,<=;) => { "Alpha" };
}
```

1.  显然，应该对其进行测试以查看它是否工作。将`it_works()`测试替换为不同的测试函数：

```rs
    #[test]
    fn test_strange_patterns() {
        assert_eq!(strange_patterns!(The pattern must match 
        precisely), "Text");
        assert_eq!(strange_patterns!(42), "Numeric");
        assert_eq!(strange_patterns!(;<=,<=;), "Alpha");
    }
```

1.  模式也可以包含实际的输入参数：

```rs
macro_rules! compare {
    ($x:literal => $y:block) => { $x == $y };
}
```

1.  一个简单的测试来结束它如下：

```rs
    #[test]
    fn test_compare() {
        assert!(compare!(1 => { 1 }));
    }
```

1.  处理 HTTP 请求始终是一个架构挑战，每个业务案例都需要添加额外的层和特殊路由。正如一些网络框架（[`github.com/seanmonstar/warp`](https://github.com/seanmonstar/warp)）所示，宏可以提供有用的支持，以使处理器能够组合在一起。向文件中添加另一个宏和支持函数——`register_handler()`函数，该函数模拟为我们的假设网络框架注册处理器函数：

```rs
#[derive(Debug)]
pub struct Response(usize);
pub fn register_handler(method: &str, path: &str, handler: &Fn() -> Response ) {}

macro_rules! web {
    (GET $path:literal => $b:block) => { 
     register_handler("GET", $path, &|| $b) };
    (POST $path:literal => $b:block) => { 
     register_handler("POST", $path, &|| $b) };
}
```

1.  为了确保一切正常工作，我们还应该为`web!`宏添加一个测试。当函数为空时，不匹配其包含模式的宏会导致编译时错误：

```rs
    use super::*;

    #[test]
    fn test_web() {
        web!(GET "/" => { Response(200) });
        web!(POST "/" => { Response(403) });
    } 
```

1.  作为最后一步，让我们运行`cargo test`（注意：在文件顶部添加`#![allow(unused_variables, unused_macros)]`以消除警告）：

```rs
$ cargo test
   Compiling matching v0.1.0 (Rust-Cookbook/Chapter06/matching)
    Finished dev [unoptimized + debuginfo] target(s) in 0.31s
     Running target/debug/deps/matching-124bc24094676408

running 3 tests
test tests::test_compare ... ok
test tests::test_strange_patterns ... ok
test tests::test_web ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests matching

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

现在让我们看看代码做了什么。

# 它是如何工作的...

在这个菜谱的*步骤 2*中，我们定义了一个宏，该宏明确提供了引擎可以匹配的不同模式。具体来说，字母数字字符限制为`,`、`;`和`=>`。虽然这允许 Ruby 风格的映射初始化，但它也限制了 DSL 可以拥有的元素。然而，宏仍然非常适合创建处理情况的更表达性的方式。在*步骤 6*和*步骤 7*中，我们展示了使用比常规链式函数调用更表达性的方式创建网络请求处理器的方法。*步骤 4*和*步骤 5*展示了在宏中使用箭头（`=>`）的用法，而*步骤 8*通过运行测试将一切联系起来。

在这个菜谱中，我们为宏调用创建了匹配臂，这些臂使用字面量匹配（而不是在类型上匹配，这将在本章后面介绍）来决定替换项。这表明我们不仅可以在一个臂中使用参数和字面量，还可以在没有常规允许的名称约束的情况下自动化任务。

我们已经成功地学习了如何在宏中实现匹配。现在让我们继续下一个配方。

# 使用预定义的宏

正如我们在本章前面的配方中看到的，宏可以节省大量的编写工作，并提供便利函数，而无需重新思考整个应用程序架构。因此，Rust 标准库提供了许多宏，用于实现可能在其他情况下出人意料地复杂的各种功能。一个例子是跨平台打印——那将如何工作？是否为每个平台都有一个输出控制台文本的等效方式？关于颜色支持呢？默认编码是什么？有很多问题，这是需要可配置的东西很多的一个指标——然而，在典型的程序中，我们只调用`print!("hello")`，它就工作了。让我们看看还有其他什么。

# 如何做到这一点...

按照以下几个步骤来实现这个配方：

1.  在终端（或在 Windows 上的 PowerShell）中运行`cargo new std-macros`，然后用 Visual Studio Code 打开该目录。然后，在项目的`src`目录中创建一个名为`a.txt`的文件，并包含以下内容：

```rs
Hello World!
```

1.  首先，`main()`的默认实现（在`src/main.rs`中）已经为我们提供了一个调用`println!`的宏：

```rs
fn main() {
    println!("Hello, world!");
}
```

1.  我们可以通过打印更多内容来扩展函数。在`main()`函数中`println!`宏调用之后插入以下内容：

```rs
    println!("a vec: {:?}", vec![1, 2, 3]);
    println!("concat: {}", concat!(0, 'x', "5ff"));
    println!("MyStruct stringified: {}", stringify!(MyStruct(10)));
    println!("some random word stringified: {}", stringify!
     (helloworld));

```

1.  `MyStruct`的定义也很简单，涉及标准库中附带的过程宏。在`main()`函数之前插入以下内容：

```rs
#[derive(Debug)]
struct MyStruct(usize);
```

1.  Rust 标准库还包括与外部世界交互的宏。让我们在`main`函数中添加更多调用：

```rs
    println!("Running on Windows? {}", cfg!(windows));
    println!("From a file: {}", include_str!("a.txt"));
    println!("$PATH: {:?}", option_env!("PATH")); 
```

1.  作为最后一步，让我们向已知的`println!`和`assert!`宏添加两个替代方案到`main()`函数中：

```rs
    eprintln!("Oh no!");
    debug_assert!(true);
```

1.  如果你还没有这样做，我们必须使用`cargo run`运行整个项目，以查看一些输出：

```rs
$ cargo run
   Compiling std-macros v0.1.0 (Rust-Cookbook/Chapter06/std-macros)
    Finished dev [unoptimized + debuginfo] target(s) in 0.25s
     Running `target/debug/std-macros`
Hello, world!
a vec: [1, 2, 3]
concat: 0x5ff
MyStruct stringified: MyStruct ( 10 )
some random word stringified: helloworld
Running on Windows? false
From a file: Hello World!
$PATH: Some("/home/cm/.cargo/bin:/home/cm/.cargo/bin:/home/cm/.cargo/bin:/usr/local/bin:/usr/bin:/bin:/home/cm/.cargo/bin:/home/cm/Apps:/home/cm/.local/bin:/home/cm/.cargo/bin:/home/cm/Apps:/home/cm/.local/bin")
Oh no!
```

我们现在应该揭开面纱，更好地理解代码。

# 它是如何工作的...

在`main`函数中，我们现在有几个或多或少为人所知的宏。每个宏都在做我们认为有用的操作。我们将跳过*步骤 2*，因为它只显示了`println!`宏——这是我们一直在使用的。然而，在*步骤 3*中，出现了一些更奇特的宏：

+   `vec!`创建并初始化一个向量，并且著名地使用`[]`来这样做。然而，虽然这样做在视觉上是有意义的，编译器同样接受`vec!()`和`vec!{}`。

+   `concat!`像静态字符串一样从左到右连接字面量。

+   `stringify!`从输入的标记中创建一个字符串字面量，无论这些标记是否存在（参见单词`helloworld`，它被转换成了字符串）。

*第 4 步*包括在 Rust 中使用过程宏。虽然单词*derive*和语法让人联想到经典 OOP 中的继承，但实际上它们并没有派生任何东西，而是提供了实际的实现。对我们来说，`#[derive(Debug)]`无疑是迄今为止最有用的，但还有`PartialEq`、`Eq`和`Clone`，它们紧随其后。

菜谱的*第 5 步*回到了函数式宏：

+   `cfg!`与`#[cfg]`属性类似，这使得在编译时确定条件成为可能，这允许你——例如——包含特定平台的代码。

+   `include_str!`是一个非常有趣的宏。还有其他包含宏，但这个宏非常有用，因为它可以将提供的文件内容作为`'static str`（就像字面量一样）提供翻译。

+   `option_env!`在**编译时**读取环境变量，以提供其值的`Option`结果。请注意，为了反映变量的更改，程序必须重新编译！

*第 6 步*的宏是其他已知流行宏的替代品：

+   `debug_assert!`是`assert!`的一个变体，它不包括在`--release`构建中。

+   `eprintln!`将内容输出到标准错误而不是标准输出。

虽然这是一个相当稳定的选项，但 Rust 标准库的未来版本将包括更多宏，使使用 Rust 更加方便。在撰写本文时，最受欢迎的未完成宏示例是`await!`，由于对`async`/`await`的不同方法，它可能永远不会稳定。请查看文档中的完整列表：[`doc.rust-lang.org/std/#macros`](https://doc.rust-lang.org/std/#macros)。

现在我们已经了解了更多关于使用预定义宏的知识，我们可以继续到下一个菜谱。

# 使用宏进行代码生成

一些派生类型的宏已经向我们展示了我们可以使用宏来生成整个特质的实现。同样，我们也可以使用宏来生成整个结构体和函数，从而避免复制粘贴编程，以及繁琐的样板代码。由于宏是在编译前执行的，生成的代码将相应地进行检查，同时避免了严格类型语言的细节。让我们看看怎么做吧！

# 如何做到这一点……

代码生成可以像这些简单的步骤一样简单：

1.  在终端（或在 Windows 上的 PowerShell）中运行`cargo new code-generation --lib`，然后使用 Visual Studio Code 打开该目录。

1.  打开`src/lib.rs`并添加第一个简单的宏：

```rs
// Repeat the statement that was passed in n times
macro_rules! n_times {
    // `()` indicates that the macro takes no argument.
    ($n: expr, $f: block) => {
        for _ in 0..$n {
            $f()
        }
    }
}
```

1.  让我们再来一个，这次稍微更具有生成性。将以下内容添加到测试模块外部（例如，在之前的宏下面）：

```rs
// Declare a function in a macro!
macro_rules! make_fn {
    ($i: ident, $body: block) => {
        fn $i () $body
    } 
}
```

1.  这两个宏也非常容易使用。让我们用相关的测试替换`tests`模块：

```rs
#[cfg(test)]
mod tests {
    #[test]
    fn test_n_times() {
        let mut i = 0;
        n_times!(5, {
            i += 1;
        });
        assert_eq!(i, 5);
    }

    #[test]
    #[should_panic]
    fn test_failing_make_fn() {
        make_fn!(fail, {assert!(false)});
        fail();
    }

    #[test]
    fn test_make_fn() {
        make_fn!(fail, {assert!(false)});
        // nothing happens if we don't call the function
    }
}
```

1.  到目前为止，宏还没有进行复杂的代码生成。实际上，第一个宏只是重复一个块多次——这已经可以通过迭代器（[`doc.rust-lang.org/std/iter/fn.repeat_with.html`](https://doc.rust-lang.org/std/iter/fn.repeat_with.html)）来实现。第二个宏创建了一个函数，但这同样可以通过闭包语法（[`doc.rust-lang.org/stable/rust-by-example/fn/closures.html`](https://doc.rust-lang.org/stable/rust-by-example/fn/closures.html)）来实现。那么，让我们添加一些更有趣的东西，比如具有`Default`实现的`enum`：

```rs
macro_rules! default_enum {
    ($name: ident, $($variant: ident => $val:expr),+) => {
        #[derive(Eq, PartialEq, Clone, Debug)]
        pub enum $name {
            Invalid,
            $($variant = $val),+
        }

        impl Default for $name {
            fn default() -> Self { $name::Invalid }
        }
    };
}
```

1.  没有什么可以未经测试，所以这里有一个测试来查看它是否按预期工作。将此添加到前面的测试中：

```rs
    #[test]
    fn test_default_enum() {
        default_enum!(Colors, Red => 0xFF0000, Blue => 0x0000FF);
        let color: Colors = Default::default();
        assert_eq!(color, Colors::Invalid);
        assert_eq!(Colors::Red as i32, 0xFF0000);
        assert_eq!(Colors::Blue as i32, 0x0000FF);     
    }
```

1.  如果我们在编写测试，我们也想看到它们在运行：

```rs
$ cargo test
Compiling custom-designators v0.1.0 (Rust-Cookbook/Chapter06/code-generation)
warning: function is never used: `fail`
  --> src/lib.rs:20:9
   |
20 | fn $i () $body
   | ^^^^^^^^^^^^^^
...
56 | make_fn!(fail, {assert!(false)});
   | --------------------------------- in this macro invocation
   |
   = note: #[warn(dead_code)] on by default

    Finished dev [unoptimized + debuginfo] target(s) in 0.30s
     Running target/debug/deps/custom_designators-ebc95554afc8c09a

running 4 tests
test tests::test_default_enum ... ok
test tests::test_make_fn ... ok
test tests::test_failing_make_fn ... ok
test tests::test_n_times ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests custom-designators

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

为了理解代码，让我们谈谈幕后发生的事情。

# 它是如何工作的...

由于编译器在真正编译之前执行宏，我们可以生成最终程序中将出现的代码，但实际上是通过宏调用创建的。这使得我们可以减少样板代码，强制执行默认值（例如实现某些特性、添加元数据等），或者为我们的 crate 的用户提供一个更友好的接口。

在**步骤 2**中，我们创建了一个简单的宏来重复一个块（这些花括号`{ }`及其内容被称为**块**）多次——使用`for`循环。在**步骤 4**中创建的测试显示了它的操作方式和它能做什么——它执行得就像我们在测试中直接写一个`for`循环一样。

**步骤 3**创建了一个更有趣的东西：一个函数。结合**步骤 4**中的测试，我们可以看到宏是如何操作的，并注意以下内容：

+   提供的块是惰性评估的（只有当函数被调用时测试才会失败）。

+   如果函数没有被调用，编译器会抱怨未使用的函数。

+   以这种方式创建参数化函数会导致编译器错误（它找不到值）。

**步骤 5**创建了一个更复杂的宏，能够创建一个完整的`enum`。它允许用户定义变体（甚至可以使用箭头`=>`记法），并添加一个默认值。让我们看看宏期望的模式：`($name: ident, $($variant: ident => $val:expr),+)`。第一个参数（`$name`）是一个标识符，它命名了某个东西（也就是说，标识符的规则被强制执行）。第二个参数是一个重复参数，它至少需要出现一次（由`+`表示），但如果提供了更多实例，它们必须用逗号分隔。这些重复的期望模式如下：标识符，`=>`，和表达式（例如，`bla => 1 + 1`，`Five => 5`，或`blog => 0x5ff`，等等）。

宏内部的内容是`enum`的经典定义，重复参数的插入频率与输入中出现的频率相同。然后，我们可以在`enum`上添加 derive 属性，并实现`std::default::Default`特质（[`doc.rust-lang.org/std/default/trait.Default.html`](https://doc.rust-lang.org/std/default/trait.Default.html)），以便在需要默认值时提供一些合理的东西。

让我们更多地了解宏和参数，然后继续下一个菜谱。

# 宏重载

方法/函数重载是一种技术，它允许有重复的方法/函数名，但每个都有不同的参数。许多静态类型语言，如 C#和 Java，支持这种技术，以便提供多种调用方法，而无需每次都想出一个新名字（或使用泛型）。然而，Rust 不支持函数重载——这是有充分理由的（[`blog.rust-lang.org/2015/05/11/traits.html`](https://blog.rust-lang.org/2015/05/11/traits.html)）。Rust 支持重载的地方是宏模式：你可以创建一个宏，并拥有多个只有输入参数不同的分支。

# 如何做到这一点...

让我们通过几个简单的步骤实现一些重载宏：

1.  在终端（或在 Windows 上的 PowerShell）中运行`cargo new macro-overloading --lib`，然后用 Visual Studio Code 打开该目录。

1.  在`src/lib.rs`中，在`mod tests`模块声明之前添加以下内容：

```rs
#![allow(unused_macros)]

macro_rules! print_debug {
    (stdout, $($o:expr),*) => {
        $(print!("{:?}", $o));*;
        println!();
    };
    (error, $($o:expr),*) => {
        $(eprint!("{:?}", $o));*;
        eprintln!();
    };
    ($stream:expr, $($o:expr),*) => {
        $(let _ = write!($stream, "{:?}", $o));*;
        let _ = writeln!($stream);
    }
}
```

1.  让我们看看我们如何应用这个宏。在`tests`模块内部，让我们通过添加以下单元测试来看看打印宏是否将字符串序列化到流中（替换现有的`it_works`测试）：

```rs
    use std::io::Write;

    #[test]
    fn test_printer() {
        print_debug!(error, "hello std err");
        print_debug!(stdout, "hello std out");
        let mut v = vec![];
        print_debug!(&mut v, "a");
        assert_eq!(v, vec![34, 97, 34, 10]);

    }
```

1.  为了便于未来的测试，我们应该在`tests`模块中添加另一个宏。这次，这个宏是对一个具有静态返回值的函数进行模拟（[`martinfowler.com/articles/mocksArentStubs.html`](https://martinfowler.com/articles/mocksArentStubs.html)）。在之前的测试之后编写这个：

```rs
    macro_rules! mock {
        ($type: ty, $name: ident, $ret_val: ty, $val: block) => {
            pub trait $name {
                fn $name(&self) -> $ret_val;
            }

            impl $name for $type {
                fn $name(&self) -> $ret_val $val
            }
        };
        ($name: ident, $($variant: ident => $type:ty),+) => {
            #[derive(PartialEq, Clone, Debug)]
            struct $name {
                $(pub $variant: $type),+
            }
        };
    }
```

1.  然后，我们应该也测试一下`mock!`宏。在下面添加另一个测试：

```rs
    mock!(String, hello, &'static str, { "Hi!" });
    mock!(HelloWorld, greeting => String, when => u64);

    #[test]
    fn test_mock() {
        let mystr = "Hello".to_owned();
        assert_eq!(mystr.hello(), "Hi!");

        let g = HelloWorld { greeting: "Hello World".to_owned(), 
        when: 1560887098 };

        assert_eq!(g.greeting, "Hello World");
        assert_eq!(g.when, 1560887098);
    }
```

1.  作为最后一步，我们运行`cargo test`来查看它是否工作。然而，这次，我们将`--nocapture`传递给测试工具，以查看打印了什么（对于*步骤 3*）：

```rs
$ cargo test -- --nocapture
Compiling macro-overloading v0.1.0 (Rust-Cookbook/Chapter06
 /macro-overloading)
warning: trait `hello` should have an upper camel case name
  --> src/lib.rs:53:19
   |
53 | mock!(String, hello, &'static str, { "Hi!" });
   | ^^^^^ help: convert the identifier to upper camel case: `Hello`
   |
   = note: #[warn(non_camel_case_types)] on by default

    Finished dev [unoptimized + debuginfo] target(s) in 0.56s
     Running target/debug/deps/macro_overloading-bd8b38e609ddd77c

running 2 tests
"hello std err"
"hello std out"
test tests::test_mock ... ok
test tests::test_printer ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests macro-overloading

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

现在，让我们幕后了解代码，以更好地理解它。

# 它是如何工作的...

重载是一个非常简单的概念——实际上简单到难以找到可用的示例，这些示例不能通过足够复杂的函数来完成。然而，在这个菜谱中，我们认为我们已经找到了一些有用的东西。

在*步骤 2*中，我们创建了一个围绕`println!`和类似函数的包装器，允许通过仅用一个标记来写入标准流，如标准输出和标准错误，或者任何其他任意流类型。此外，这个实现还有一些有趣的细节：

+   每次调用`print!`都会跟一个`;`——除了最后一个，这就是为什么在`*`后面还有一个额外的`;`的原因。

+   该模式允许传入任意数量的表达式。

这个宏可以用来避免重复 `println!("{:?}", "hello")` 只是为了快速查看变量的当前值。此外，它还便于将输出重定向到标准错误。

在 *步骤 3* 中，我们为这个宏调用创建了一个测试。在快速检查中，我们向 `error`、`stdout` 和 `vec!`（这就是为什么我们导入 `std::io::Write`）打印。在那里，我们可以看到末尾的新行，并且它被写为字符串（数字是字节）。在任何调用中，它都找到了所需的宏模式并插入了其内容。

*步骤 4* 创建了一个宏，用于模拟结构体或整个结构体上的函数。这对于隔离测试，真正只测试目标实现而不冒通过尝试实现辅助函数而添加更多错误的风险非常有用。在这种情况下，宏的分支很容易区分。第一个分支创建了一个函数的模拟实现，并匹配它所需的参数：它附加到的类型、函数的标识符、返回类型以及返回该类型的代码块。第二个分支创建了一个结构体，因此只需要一个标识符来命名结构体及其属性及其数据类型。

模拟——或者创建一个模拟对象——是一种测试技术，它允许创建浅层结构来模拟所需的行为。这对于无法以其他方式实现的事物（外部硬件、第三方网络服务等等）或复杂的内部系统（数据库连接和逻辑）非常有用。

接下来，我们必须测试这些结果，这在 *步骤 5* 中完成。在那里，我们调用 `mock!` 宏并定义其行为，以及一个测试来证明它的工作。我们在 *步骤 6* 中运行测试，没有捕获控制台输出：它工作得很好！

我们确信宏的重载很容易学习。现在让我们继续下一个菜谱。

# 使用重复来指定参数范围

Rust 的 `println!` 宏有一个奇特的特点：你可以传入的参数数量没有上限。由于常规 Rust 不支持任意参数范围，它必须是一个宏特性——但是哪个呢？在这个菜谱中，找出如何处理和实现宏的参数范围。

# 如何做到...

经过这几步，你就会知道如何使用参数范围。

1.  在终端（或在 Windows 上的 PowerShell）中运行 `cargo new parameter-ranges --lib` 并使用 Visual Studio Code 打开该目录。

1.  在 `src/lib.rs` 中，添加以下代码以在 `vec!` 风格中初始化一个集合：

```rs
#![allow(unused_macros)]

macro_rules! set {
 ( $( $item:expr ),* ) => {
        {
            let mut s = HashSet::new();
            $(
                s.insert($item);
            )*
            s
        }
    };
}
```

1.  接下来，我们将添加一个简单的宏来创建一个 DTO——数据传输对象：

```rs
macro_rules! dto {
    ($name: ident, $($variant: ident => $type:ty),+) => {
        #[derive(PartialEq, Clone, Debug)]
        pub struct $name {
            $(pub $variant: $type),+
        }

        impl $name {
            pub fn new($($variant:$type),+) -> Self {
                $name {
                    $($variant: $variant),+
                }
             }
        }
    };
}
```

1.  这也需要进行测试，所以让我们添加一个测试来使用新的宏创建一个集合：

```rs
#[cfg(test)]
mod tests {
    use std::collections::HashSet;

    #[test]
    fn test_set() {
        let actual = set!("a", "b", "c", "a");
        let mut desired = HashSet::new();
        desired.insert("a");
        desired.insert("b");
        desired.insert("c");
        assert_eq!(actual, desired); 
    }
}
```

1.  在测试集合初始化后，让我们也测试创建一个 DTO。在之前的测试下添加以下内容：

```rs
    #[test]
    fn test_dto() {
        dto!(Sensordata, value => f32, timestamp => u64);
        let s = Sensordata::new(1.23f32, 123456);
        assert_eq!(s.value, 1.23f32); 
        assert_eq!(s.timestamp, 123456); 
    }
```

1.  作为最后一步，我们也运行 `cargo test` 来展示它的工作情况：

```rs
$ cargo test
 Compiling parameter-ranges v0.1.0 (Rust-Cookbook/Chapter06
  /parameter-ranges)
  Finished dev [unoptimized + debuginfo] target(s) in 1.30s
  Running target/debug/deps/parameter_ranges-7dfb9718c7ca3bc4

running 2 tests
test tests::test_dto ... ok
test tests::test_set ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests parameter-ranges

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

现在，让我们深入了解代码以更好地理解它。

# 它是如何工作的...

Rust 的宏系统中参数范围的工作方式有点像正则表达式。语法有几个部分：`$()`表示重复，其分隔符后面的字符（`,`、`;`和`=>`是允许的），最后是重复期望的限定符（`+`或`*`——就像正则表达式一样，一个是或多个，另一个是零个或多个）。

*第二步*展示了类似于`vec!`的集合初始化宏的实现。在那里，我们期望一个表达式来填充`std::collections::HashSet`，并在子块中返回结果。这是必要的，以便允许诸如变量赋值（在 transcriber 块中不允许直接进行）之类的操作，但不要阻碍传递给宏的参数的展开。与声明类似，展开使用`$()`区域进行，但与分隔符不同，重复限定符直接跟随。其中包含的内容将根据参数的数量运行多次。

第二个宏在*第三步*中定义，并且更加复杂。名称`dto!`（数据传输对象）表示一个业务对象，如数据容器，它仅用于在程序内部传递数据，而不会被发送到程序外部。由于这些 DTO 包含大量的样板代码，它们可以像键值存储一样初始化。通过在参数范围指定中使用`=>`符号，我们可以创建用于在`struct`及其构造函数中创建属性的标识符/类型对。请注意，分隔属性的逗号位于`+`符号之前，因此它也会被重复。

*第四步*展示了调用在*第二步*中设计的宏以填充集合并测试是否正确填充。同样，*第五步*展示了创建和实例化 DTO 实例（称为`Sensordata`的结构体）以及测试以确认属性按预期创建。最后一步通过运行测试来确认这一点。

我们已经成功学习了如何使用 repeat 来处理参数范围。现在让我们继续下一个菜谱。

# 不要重复自己

在之前的菜谱中，我们使用宏来生成几乎任意的代码，从而减少了需要编写的代码量。让我们更深入地探讨这个话题，因为这是一个不仅能够减少错误而且能够实现代码一致性的好方法。每个人都应该做的重复性任务之一是测试（尤其是如果它是面向公众的 API），如果我们复制粘贴这些测试，我们就会暴露自己于错误之中。相反，让我们看看我们如何使用宏生成样板代码来停止重复。

# 如何做...

使用宏进行自动化测试只需几个步骤：

1.  在终端（或在 Windows 上的 PowerShell）中运行`cargo new dry-macros --lib`，然后用 Visual Studio Code 打开该目录。

1.  在`src/lib.rs`中，我们想要创建一个辅助宏并导入所需的库：

```rs
use std::ops::{Add, Mul, Sub};

macro_rules! assert_equal_len {
    // The `tt` (token tree) designator is used for
    // operators and tokens.
    ($a:ident, $b: ident, $func:ident) => (
        assert_eq!($a.len(), $b.len(),
                "{:?}: dimension mismatch: {:?} {:?}",
                stringify!($func),
                ($a.len(),),
                ($b.len(),));
    )
}
```

1.  接下来，我们定义一个宏来自动实现一个操作符。让我们在 `assert_equal_len` 宏下面添加这个宏：

```rs
macro_rules! op {
    ($func:ident, $bound:ident, $method:ident) => (
        pub fn $func<T: $bound<T, Output=T> + Copy>(xs: &mut 
        Vec<T>, ys: &Vec<T>) {
            assert_equal_len!(xs, ys, $func);

            for (x, y) in xs.iter_mut().zip(ys.iter()) {
                *x = $bound::$method(*x, *y);
            }
        }
    )
}
```

1.  现在，让我们调用这个宏并实际生成实现：

```rs
op!(add_assign, Add, add);
op!(mul_assign, Mul, mul);
op!(sub_assign, Sub, sub);
```

1.  在这些函数到位之后，我们现在也可以生成测试用例了！用以下内容替换 `test` 模块：

```rs
#[cfg(test)]
mod test {

    use std::iter;
    macro_rules! test {
        ($func: ident, $x:expr, $y:expr, $z:expr) => {
            #[test]
            fn $func() {
                for size in 0usize..10 {
                    let mut x: Vec<_> = 
                    iter::repeat($x).take(size).collect();
                    let y: Vec<_> = 
                    iter::repeat($y).take(size).collect();
                    let z: Vec<_> = 
                    iter::repeat($z).take(size).collect();

                    super::$func(&mut x, &y);

                    assert_eq!(x, z);
                }
            }
        }
    }

    // Test `add_assign`, `mul_assign` and `sub_assign`
    test!(add_assign, 1u32, 2u32, 3u32);
    test!(mul_assign, 2u32, 3u32, 6u32);
    test!(sub_assign, 3u32, 2u32, 1u32);
}
```

1.  作为最后一步，让我们通过运行 `cargo test` 来查看生成的代码的实际效果，以查看（积极的）测试结果：

```rs
$ cargo test
 Compiling dry-macros v0.1.0 (Rust-Cookbook/Chapter06/dry-macros)
  Finished dev [unoptimized + debuginfo] target(s) in 0.64s
  Running target/debug/deps/dry_macros-bed1682b386b41c3

running 3 tests
test test::add_assign ... ok
test test::mul_assign ... ok
test test::sub_assign ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests dry-macros

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

为了更好地理解代码，让我们分析这些步骤。

# 它是如何工作的...

虽然设计模式、`if-else` 构造和一般而言的 API 设计有助于代码重用，但当需要将标记（例如，某些名称）硬编码以保持松散耦合时，就会变得复杂。Rust 的宏可以帮助解决这个问题。作为一个例子，我们为这些函数生成函数和测试，以避免在文件之间复制和粘贴测试代码。

在 *第 3 步* 中，我们声明了一个宏，它围绕比较两个序列的长度并提供更好的错误信息。*第 4 步* 立即使用这个宏并创建一个具有提供名称的函数，但前提是多个输入 `Vec` 实例的长度匹配。

在 *第 5 步* 中，我们调用宏并提供它们所需的输入：一个名称（用于函数）和泛型绑定的类型。这使用提供的接口创建了函数，而不需要复制和粘贴代码。

*第 6 步* 通过声明 `test` 模块、一个用于生成测试的宏以及最终创建测试代码的调用来创建相关的测试。这允许你在编译之前即时生成测试，这显著减少了静态、重复代码的数量——这一直是测试中的一个问题。最后一步显示了这些测试实际上是在运行 `cargo test` 时创建和执行的。
