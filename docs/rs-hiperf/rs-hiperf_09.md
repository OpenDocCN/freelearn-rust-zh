# 创建您的自己的宏

在前面的章节中，我们看到了宏和元编程如何使您的生活变得更加轻松。我们看到了两种宏，一种是减少所需样板代码的宏，另一种是能加快最终代码的宏。现在是时候学习如何创建自己的宏了。

在本章中，您将学习如何创建自己的标准宏，如何创建自己的过程宏和自定义的 derive，以及最后如何使用夜间功能来创建自己的插件。您还将了解新的声明式宏是如何工作的。

本章分为三个部分：

+   **宏系统**：理解 `macro_rules!{}` 宏

+   **过程宏**：学习如何创建自己的自定义 derive

+   **夜间元编程**：插件和声明式宏

# 创建您自己的标准宏

自 Rust 1.0 以来，我们有一个非常好的宏系统。宏允许我们将一些代码应用到多个类型或表达式中，因为它们通过在编译时展开自身来实现。这意味着当您使用宏时，您实际上在编译开始之前就写了很多代码。这有两个主要好处，首先，通过变得更小和重用代码，代码库可以更容易维护，其次，由于宏在开始创建目标代码之前展开，您可以在语法级别进行抽象。

例如，您可以有这样一个函数：

```rs
fn add_one(input: u32) -> u32 {
    input + 1
}
```

这个函数将输入限制为 `u32` 类型，并将返回类型限制为 `u32`。我们可以通过使用泛型添加一些其他可接受类型，如果我们使用 `Add` 特性，它可能接受 `&u32`。宏允许我们为任何可以写入 `+` 符号左侧的元素创建这种代码，并且它将为每种类型的元素生成不同的代码，为每种情况创建不同的代码。

要创建一个宏，您需要使用语言内置的宏，即 `macro_rules!{}` 宏。这个宏接受新宏的名称作为第一个参数，以及包含宏代码的代码块作为第二个元素。第一次看到这个语法时，它可能有点复杂，但可以很快学会。让我们从一个与之前看到的函数做同样事情的宏开始：

```rs
macro_rules! add_one {
    ($input:expr) => {
        $input + 1
    }
}
```

您现在可以从 `main()` 函数中调用该宏，通过调用 `add_one!(integer);`。请注意，即使在同一文件中，宏也需要在第一次调用之前定义。它将适用于任何整数，这是函数所做不到的。

让我们分析一下语法是如何工作的。在新宏（`add_one`）名称之后的代码块中，我们可以看到两个部分。在第一个部分，在 `=>` 的左边，我们看到括号内的 `$input:expr`。然后，在右边，我们看到一个 Rust 代码块，我们在其中执行实际的加法操作。

左侧部分的工作方式（在某些方面）类似于模式匹配。你可以添加任何字符组合和一些变量，所有这些变量都以美元符号（`$`）开头，并在冒号后显示变量的类型。在这种情况下，唯一的变量是 `$input` 变量，它是一个表达式。这意味着你可以在那里插入任何类型的表达式，它将被写入右侧的代码中，用表达式替换变量。

# 宏变体

如你所见，它并没有你想象的那么复杂。正如我写的，你可以在 `macro_rules!{}` 侧的左侧几乎有任意模式。不仅如此，你还可以有多个模式，就像是一个匹配语句，所以如果其中一个匹配，它将被展开。让我们通过创建一个宏来了解它是如何工作的，这个宏根据我们如何调用它，将给定的整数加一或加二：

```rs
macro_rules! add {
    {one to $input:expr} => ($input + 1);
    {two to $input:expr} => ($input + 2);
}

fn main() {
    println!("Add one: {}", add!(one to 25/5));
    println!("Add two: {}", add!(two to 25/5));
}
```

你可以看到宏的几个明显的变化。首先，我们在宏中交换了花括号和括号。这是因为在一个宏中，你可以使用可互换的花括号（`{` 和 `}`）、方括号（`[` 和 `]`）和括号（`(` 和 `)`）。不仅如此，你还可以在调用宏时使用它们。你可能已经使用过 `vec![]` 宏和 `format!()` 宏，我们在上一章中看到了 `lazy_static!{}` 宏。我们在这里使用方括号和括号只是为了约定，但我们可以以相同的方式调用 `vec!{}` 或 `format![]` 宏，因为我们可以在任何宏调用中使用花括号、方括号和括号。

第二次更改是在我们的左侧模式中添加一些额外的文本。我们现在通过编写文本 `one to` 或 `two to` 来调用我们的宏，所以我也将 `one` 的冗余从宏名称中移除，并称之为 `add!()`。这意味着我们现在用文字文本调用我们的宏。这不是有效的 Rust，但由于我们正在使用宏，我们在编译器试图理解实际的 Rust 代码之前修改了我们正在编写的代码，生成的代码是有效的。我们可以在模式中添加任何不结束模式的文本（例如括号或花括号）。

最后的更改是添加第二种可能的模式。现在我们可以添加一个或两个，唯一的区别是宏定义的右侧现在必须以每个模式的尾随分号结尾（最后一个可选），以分隔每个选项。

在示例中我还添加了一个小细节，那就是在`main()`函数中调用宏。如您所见，我本可以将`one`或`two`加到`5`上，但我写`25/5`是有原因的。在编译这段代码时，这将展开为`25/5 + 1`（或者如果你使用第二个变体，是`2`）。这将在编译时进行优化，因为它将知道`25/5 + 1`是`6`，但编译器将接收到这个表达式，而不是最终结果。宏系统不会计算表达式的结果；它将简单地复制你给出的结果代码，并将其传递给下一个编译阶段。

当你创建的宏调用另一个宏时，你应该特别注意这一点。它们将会递归地展开，一层套一层，因此编译器将接收到一大堆最终的 Rust 代码，这些代码需要被优化。在上一章中我们看到的 CLAP crate 中发现了与此相关的问题，因为指数展开给它们的可执行文件添加了大量的冗余代码。一旦他们发现其他宏内部有太多的宏展开并且修复了这个问题，他们通过超过 50%的比例减少了他们的二进制贡献的大小。

宏允许额外的定制层。你可以重复参数多次。这在`vec![]`宏中很常见，例如，在编译时创建一个新向量。你可以写像`vec![3, 4, 76, 87];`这样的东西。`vec![]`宏是如何处理未指定数量的参数的？

# 复杂的宏

我们可以通过在宏定义的左侧模式中添加一个`*`来指定我们想要多个表达式，表示零个或多个匹配，或者添加一个`+`来表示一个或多个匹配。让我们看看我们如何通过简化的`my_vec![]`宏来实现这一点：

```rs
macro_rules! my_vec {
    ($($x: expr),*) => {{
        let mut vector = Vec::new();
        $(vector.push($x);)*
        vector
    }}
}
```

让我们看看这里发生了什么。首先，我们看到在左侧，我们有两个变量，由两个`$`符号表示。第一个指的是实际的重复。每个以逗号分隔的表达式将生成一个`$x`变量。然后，在右侧，我们使用各种重复来将`$x`向量为每个接收到的表达式推送一次。

右侧还有一个新事物。如您所见，宏展开开始和结束于一个双花括号，而不是只使用一个。这是因为一旦宏展开，它将用给定表达式替换一个新表达式：即生成的表达式。由于我们想要返回我们创建的向量，我们需要一个新的作用域，其中最后一句话将是作用域执行后的值。你将在下一个代码片段中更清楚地看到这一点。

我们可以用`main()`函数来调用这段代码：

```rs
fn main() {
    let my_vector = my_vec![4, 8, 15, 16, 23, 42];
    println!("Vector test: {:?}", my_vector);
}
```

它将被扩展到以下代码：

```rs
fn main() {
    let my_vector = {
        let mut vector = Vec::new();
        vector.push(4);
        vector.push(8);
        vector.push(15);
        vector.push(16);
        vector.push(23);
        vector.push(42);
        vector
    };
    println!("Vector test: {:?}", my_vector);
}
```

如您所见，我们需要这些额外的花括号来创建一个作用域，以便返回向量，这样它就会被分配给`my_vector`绑定。

你可以在左侧表达式中有多重重复模式，并且它们将根据需要重复使用。在官方 Rust 书的第一个版本中有一个很好的例子说明了这种行为，我在这里进行了改编：

```rs
macro_rules! add_to_vec {
    ($( $x:expr; [ $( $y:expr ),* ]);* ) => {
        &[ $($( $x + $y ),*),* ]
    }
}
```

在这个例子中，宏可以接收一个或多个`$x; [$y1, $y2,...]`输入。所以，对于每个输入，它将有一个表达式，然后是一个分号，然后是一个带有多个子表达式并用逗号分隔的括号，最后是另一个括号和一个分号。但宏对这个输入做了什么？让我们检查它的右侧。

如你所见，这将创建多个重复。我们可以看到它创建了一个切片（`&[T]`），无论我们给它什么，所以所有我们使用的表达式都必须是同一类型。然后，它将遍历所有的`$x`变量，每个输入组一个。所以如果我们只给它一个输入，它将为分号左侧的表达式迭代一次。然后，它将为与`$x`表达式关联的每个`$y`表达式迭代一次，将它们添加到`+`运算符中，并将结果包含在切片中。

如果这太复杂而难以理解，让我们看一个例子。假设我们用`65; [22, 34]`作为输入调用宏。在这种情况下，`65`将是`$x`，而`22`、`24`等将是与`65`关联的`$y`变量。因此，结果将是一个这样的切片：`&[65+22, 65+34]`。或者，如果我们计算结果：`&[87, 99]`。

如果我们使用`65; [22, 34]; 23; [56, 35]`作为输入提供两组变量，在第一次迭代中，`$x`将是`65`，而在第二次迭代中，它将是`23`。`64`的`$y`变量将是`22`和`34`，如前所述，与`23`关联的变量将是`56`和`35`。这意味着最终的切片将是`&[87, 99, 79, 58]`，其中`87`和`99`与之前相同，而`79`和`58`是添加`23`到`56`和`23`到`35`的扩展。

这比函数提供了更多的灵活性，但请记住，所有这些都会在编译时展开，这可能会使你的编译时间更长，如果使用的宏重复了太多代码，最终的代码库会更大且运行得更慢。无论如何，它还有更多的灵活性。

到目前为止，所有变量都为`expr`类型。我们通过声明`$x:expr`和`$y:expr`来使用它，但正如你可以想象的，还有其他类型的宏变量。列表如下：

+   `expr`：可以在等号后书写的表达式，例如`76+4`或`if a==1 {"something"} else {"other thing"}`。

+   `ident`：一个标识符或绑定名称，例如`foo`或`bar`。

+   `path`：一个合格路径。这将是一个你可以在使用句子中写的路径，例如`foo::bar::MyStruct`或`foo::bar::my_func`。

+   `ty`：一种类型，例如`u64`或`MyStruct`。它也可以是类型的路径。

+   `pat`：可以在等号左侧或匹配表达式中书写的模式，例如`Some(t)`或`(a, b, _)`。

+   `stmt`: 一个完整的语句，例如一个 `let` 绑定，如 `let a = 43;`。

+   `block`: 一个可以包含多个语句并在花括号之间有可选表达式的块元素，例如 `{vec.push(33); vec.len()}`。

+   `item`: Rust 所称的 *items*。例如，函数或类型声明、完整的模块或特质定义。

+   `meta`: 一个元元素，你可以将其写入属性（`#[]`）中。例如，`cfg(feature = "foo")`。

+   `tt`: 任何最终将被宏模式解析的标记树，这意味着几乎可以是任何东西。这对于创建递归宏非常有用。

如你所想，这些类型的宏变量中有些是重叠的，有些则比其他更具体。使用将在宏的右侧进行验证，在展开时，因为你可能会尝试在一个必须使用表达式的地方使用语句，即使你可能也会使用一个标识符，例如。

还有一些额外的规则，正如我们可以在 Rust 文档中看到的那样（[`doc.rust-lang.org/book/first-edition/macros.html#syntactic-requirements`](https://doc.rust-lang.org/book/first-edition/macros.html#syntactic-requirements)）。语句和表达式只能由 `=>`、逗号或分号跟随。类型和路径只能由 `=>`、`as` 或 `where` 关键字、或任何逗号、`=`、`|`、`;`、`:`、`>`、`[` 或 `{` 跟随。最后，模式只能由 `=>`、`if` 或 `in` 关键字、或任何逗号、`=` 或 `|` 跟随。

让我们通过实现一个针对我们可以创建的货币类型的 `Mul` 特质来将这个概念付诸实践。这是一个我们在创建分形信用数字货币时所做的某些工作的改编示例。在这种情况下，我们将查看 `Amount` 类型（[`github.com/FractalGlobal/utils-rs/blob/49955ead9eef2d9373cc9386b90ac02b4d5745b4/src/amount.rs#L99-L102`](https://github.com/FractalGlobal/utils-rs/blob/49955ead9eef2d9373cc9386b90ac02b4d5745b4/src/amount.rs#L99-L102)）的实现，它表示货币金额。让我们从基本类型定义开始：

```rs
#[derive(Copy, Clone, PartialEq, Eq, PartialOrd, Ord)]
pub struct Amount {
    value: u64,
}
```

这个金额可以被最多三位小数整除，但它始终是一个精确值。我们应该能够将一个 `Amount` 添加到当前的 `Amount`，或者从中减去。我不会解释这些简单的实现，但有一个实现中宏可以大有帮助。我们应该能够将金额乘以任何正整数，因此我们应该为 `u8`、`u16`、`u32` 和 `u64` 类型实现 `Mul` 特质。不仅如此，我们还应该能够实现 `Div` 和 `Rem` 特质，但我会省略这些，因为它们稍微复杂一些。你可以在前面链接的实现中查看它们。

`Amount` 与整数的乘法所做的唯一事情就是将值乘以给定的整数。让我们看看 `u8` 的简单实现：

```rs
use std::ops::Mul;

impl Mul<u8> for Amount {
    type Output = Self;

    fn mul(self, rhs: u8) -> Self::Output {
        Self { value: self.value * rhs as u64 }
    }
}

impl Mul<Amount> for u8 {
    type Output = Amount;

    fn mul(self, rhs: Amount) -> Self::Output {
        Self::Output { value: self as u64 * rhs.value }
    }
}
```

如您所见，我两种方式都实现了，这样您可以将`Amount`放在乘法符号的左边或右边。如果我们必须对所有整数都这样做，那将是一种时间和代码的巨大浪费。而且，如果我们必须修改其中一个实现（特别是对于`Rem`函数），在多个代码点上进行修改将会很麻烦。让我们使用宏来帮助我们。

我们可以定义一个宏，`impl_mul_int!{}`，它将接收一个整数类型的列表，然后在这些类型和`Amount`类型之间实现双向的`Mul`特质。让我们看看：

```rs
macro_rules! impl_mul_int {
    ($($t:ty)*) => ($(
        impl Mul<$t> for Amount {
            type Output = Self;

            fn mul(self, rhs: $t) -> Self::Output {
                Self { value: self.value * rhs as u64 }
            }
        }

        impl Mul<Amount> for $t {
            type Output = Amount;

            fn mul(self, rhs: Amount) -> Self::Output {
                Self::Output { value: self as u64 * rhs.value }
            }
        }
    )*)
}

impl_mul_int! { u8 u16 u32 u64 usize }
```

如您所见，我们特别要求给定的元素是类型，然后我们为所有这些元素实现特质。所以，对于您想要为多个类型实现的所有代码，您不妨尝试这种方法，因为它可以节省您大量的代码，并使其更易于维护。

# 创建过程宏

我们已经看到了标准宏可以为我们的 crate 做什么。我们可以创建复杂的编译时代码，既可以减少我们代码的冗长性，使其更易于维护，还可以通过在编译时而不是在运行时执行操作来提高最终可执行文件的性能。

然而，标准宏只能做到如此。使用它们，您只能修改一些 Rust 语法标记处理，但您仍然受限于`macro_rules!{}`宏所能理解的内容。这就是过程宏（也称为宏 1.1 或自定义派生）发挥作用的地方。

使用过程宏，您可以为编译器在结构体或枚举中派生其名称时调用的库创建库。您可以有效地创建一个自定义特质，`derive`。

# 实现一个简单的特质

让我们看看如何通过为结构体或枚举实现一个简单的特质来实现这一点。我们将要实现的特质如下：

```rs
trait TypeName {
    fn type_name() -> &'static str;
}
```

这个`trait`应该返回当前结构体或枚举的名称作为字符串。手动实现它非常简单，但手动逐个为我们的类型实现它没有意义。最好的做法是使用过程宏来派生它。让我们看看我们如何做到这一点。

首先，我们需要创建一个库 crate。按照惯例，它应该有父 crate 的名称，然后是`-derive`后缀。在这种情况下，我们没有为 crate 命名，但让我们称这个库为`type-name-derive`：

```rs
cargo new type-name-derive
```

这应该在`src/`文件夹旁边创建一个新的文件夹，命名为`type-name-derive`。现在我们可以将其添加到`Cargo.toml`文件中：

```rs
[dependencies]
type-name-derive = { path = "./type-name-derive" }
```

在我们的 crate 的`main.rs`文件中，我们需要添加 crate 并使用其宏：

```rs
#[macro_use]
extern crate type_name_derive;

trait TypeName {
    fn type_name() -> &'static str;
}
```

现在，让我们开始实际的派生开发。我们需要使用两个 crate——`syn`和`quote`。我们将把它们添加到`type-name-derive`目录内的`Cargo.toml`文件中：

```rs
[dependencies]
syn = "0.12.10"
quote = "0.4.2"
```

`syn` crate 为我们提供了一些有用的类型和函数，用于处理 Rust 的源树和标记流。我们需要这个，因为我们的所有宏都将看到有关我们结构体或枚举源代码的大量信息。我们需要解析它并从中获取信息。`syn` crate 是一个`Nom`解析器，它将 Rust 源代码转换为我们可以轻松使用的东西。

`quote` crate 为我们提供了`quote!{}`宏，我们将使用它来直接实现实现代码的源代码。它基本上允许我们编写几乎*正常*的 Rust 代码，而不是编译器标记来实现特性。

在`Cargo.toml`文件中，我们还需要包含其他一些额外信息。我们需要通知 Cargo 和 Rust 编译器，这个 crate 是一个过程宏 crate。为此，我们需要在文件中添加以下内容：

```rs
[lib]
proc-macro = true
```

然后，我们需要从`./type-name-derive/src/lib.rs`文件的基本架构开始：

```rs
extern crate proc_macro;
extern crate syn;
#[macro_use]
extern crate quote;

use proc_macro::TokenStream;

#[proc_macro_derive(TypeName)]
pub fn type_name(input: TokenStream) -> TokenStream {
    // some code
}
```

如您所见，我们首先导入了所需的定义。`proc_macro` crate 是编译器内置的，为我们提供了`TokenStream`类型。此类型表示 Rust 标记（源文件中的字符）的流。然后我们导入了`syn`和`quote` crate。如我们之前所见，我们需要使用 quote crate 中的`quote!{}`宏，因此我们也导入了宏。

实现自定义 derive 的语法非常简单。我们需要定义一个具有`proc_macro_derive`属性值的函数，该值是我们想要派生的特性。该函数将获取标记流的拥有权，并返回另一个（或相同的）标记流，以便编译器可以稍后处理新生成的 Rust 代码。

为了实现这个特性，我更喜欢将标记解析和实际的特性实现分为两个函数。为此，让我们首先在我们的`type_name()`函数中编写代码：

```rs
    // Parse the input tokens into a syntax tree
    let input = syn::parse(input).unwrap();

    // Build the output
    let expanded = impl_type_name(&input);

    // Hand the output tokens back to the compiler
    expanded.into()
```

标记流首先被转换为`DeriveInput`结构。此结构包含从输入标记流正确解析的数据；反序列化到此处的信息，我们将添加`#[derive]`属性。在生产环境中，这些`unwrap()`函数可能需要更改为`expect()`，这样我们就可以在出错时添加一些文本。

之后，我们使用这些信息来调用我们尚未定义的`impl_type_name()`函数。该函数将接受有关结构体或枚举的信息，并返回一系列标记。由于我们将使用 quote crate 来创建这些标记，因此它们需要稍后转换为 Rust 编译器标记流并返回给编译器。

现在让我们实现`impl_type_name()`函数：

```rs
fn impl_type_name(ast: &syn::DeriveInput) -> quote::Tokens {
    let name = &ast.ident;
    quote! {
        impl TypeName for #name {
            fn type_name() -> &'static str {
                stringify!(#name)
            }
        }
    }
}
```

如您所见，这个实现非常简单。我们从**高级源树**（**AST**）中获取结构的名称，然后在`quote!{}`宏中使用它，在那里我们为具有该名称的结构或枚举实现`TypeName`宏。`stringify!()`宏是一个本地的 Rust 宏，它获取一个标记并返回其编译时字符串表示形式。在这种情况下，是`name`。

让我们通过在我们的父 crate 的`main()`函数中添加一些代码并添加几个将派生我们实现的类型来测试一下这行是否有效：

```rs
#[derive(TypeName)]
struct Alice;

#[derive(TypeName)]
enum Bob {}

fn main() {
    println!("Alice's name is {}", Alice::type_name());
    println!("Bob's name is {}", Bob::type_name());
}
```

如果您执行`cargo run`，您将看到以下输出，正如预期的那样：

```rs
Alice's name is Alice
Bob's name is Bob
```

# 实现复杂派生

`syn`和`quote` crate 允许进行非常复杂的派生。不仅如此，我们不一定需要派生一个特质；我们可以为给定的结构或枚举实现任何类型的代码。这意味着我们可以派生构建器模式，正如我们在上一章中看到的，这实际上会为我们的结构创建一个新的结构或`Getters`和`Setters`。

这就是我们接下来将要做的。我们将创建结构并给它们一些方法来`get`和`set`结构的字段。在 Rust 中，惯例是`Getters`应该有字段的名字，没有前缀或后缀，如`get`。所以对于名为`foo`的字段，`Getter`应该是`foo()`。对于可变的`Getters`，函数应该有`_mut`后缀，所以在这种情况下它将是`foo_mut()`。设置器应该以`set_`前缀开头，所以对于名为`bar`的字段，它应该是`set_bar()`。

让我们从创建这个新的`getset-derive`过程宏 crate 开始。和之前一样，记得在`Cargo.toml`文件的`[lib]`部分中将`proc_macro`变量设置为 true，并将`syn`和`quote` crate 作为依赖项添加。

# 实现获取器

我们将添加两个派生，`Getters`和`Setters`。我们将从第一个开始，创建所需的模板代码：

```rs
extern crate proc_macro;
extern crate syn;
#[macro_use]
extern crate quote;

use proc_macro::TokenStream;

#[proc_macro_derive(Getters)]
pub fn derive_getters(input: TokenStream) -> TokenStream {
    // Parse the input tokens into a syntax tree
    let input = syn::parse(input).unwrap();

    // Build the output
    let expanded = impl_getters(&input);

    // Hand the output tokens back to the compiler
    expanded.into()
}

fn impl_getters(ast: &syn::DeriveInput) -> quote::Tokens {
    let name = &ast.ident;
    unimplemented!()
}
```

我们不仅需要结构的名称，还需要添加到结构中的任何进一步泛型以及`where`子句。在先前的例子中我们没有处理这些，但在更复杂的例子中我们应该添加它们。幸运的是，`syn` crate 为我们提供了所有需要的东西。

让我们在`impl_getters()`函数中编写下一部分代码：

```rs
fn impl_getters(ast: &syn::DeriveInput) -> quote::Tokens {
    use syn::{Data, Fields};

    let name = &ast.ident;
    let (impl_generics, ty_generics, where_clause) =
                            ast.generics.split_for_impl();

    match ast.data {
        Data::Struct(ref structure) => {
            if let Fields::Named(ref fields) = structure.fields {
                let getters: Vec<_> =
                                fields.named.iter()
                                            .map(generate_getter)
                                            .collect();

                quote! {
                    impl #impl_generics #name #ty_generics
                        #where_clause {
                        #(#getters)*
                    }
                }
            } else {
                panic!("you cannot implement getters for unit \
                        or tuple structs");
            }
        },
        Data::Union(ref _union) => {
            unimplemented!("sorry, getters are not implemented \
                            for unions yet");
        }
        Data::Enum(ref _enum) => {
            panic!("you cannot derive getters for enumerations");
        }
    }
}

fn generate_getter(field: &syn::Field) -> quote::Tokens {
    unimplemented!("getters not yet implemented")
}
```

这里有很多事情在进行。首先，如您所见，我们从`ast.generics`字段中获取泛型，并在`quote!`宏中使用它们。然后我们检查我们有什么类型的数据。我们不能为枚举、单元结构或没有命名字段的结构（如`Foo(T)`）实现获取器或设置器，所以在这些情况下我们会 panic。尽管目前仍然无法为联合派生任何东西，但我们可以使用`syn` crate 特别过滤选项，所以我们只是添加它以供语言未来的潜在更改。

在具有命名字段的结构的情形下，我们得到一个字段列表，并为每个字段实现一个 getter。为此，我们将它们映射到在底部定义但尚未实现的`generate_getter()`函数。

一旦我们有了 getter 列表，我们就调用 quote 的`!{}`宏来生成令牌。正如你所见，我们为`impl`块添加了泛型，这样如果结构体中有任何泛型，例如`Bar<T, F, G>`，它们就会被添加到实现中。

要添加从向量中获取的所有 getter，我们使用`#(#var)*`语法，与`macro_rules!{}`宏类似，它会依次添加。我们可以使用这种语法与任何实现了`IntoIterator`特质的类型一起使用，在这种情况下，是`Vec<quote::Tokens>`。

因此，现在我们必须实际实现一个 getter。我们有`generate_getter()`函数，它接收一个`syn::Field`，因此我们有了所需的所有信息。该函数将返回`quote::Tokens`，因此我们需要在内部使用`quote!{}`宏。如果你一直在跟随，你可以自己实现它，通过检查`syn`包的文档在`docs.rs`。让我们看看完全实现后的样子：

```rs
fn generate_getter(field: &syn::Field) -> quote::Tokens {
    let name = field.ident
                .expect("named fields must have a name");
    let ty = &field.ty;

    quote! {
        fn #name(&self) -> &#ty {
            &self.#name
        }
    }
}
```

如你所见，这真的很简单。我们获取属性或字段的标识符或名称，鉴于我们只为具有命名字段的结构体实现它，这个名称应该存在，然后获取字段的类型。然后我们创建 getter 并返回内部数据的引用。

我们可以通过添加对具有借用对应类型的异常处理来进一步改进，例如`String`或`PathBuf`，分别返回`&str`和`Path`，但我认为这并不值得。

我们还可以将字段的文档添加到生成的 getter 中。为此，我们将使用`field.attrs`变量并获取名为`doc`的属性的值，这是包含文档文本的属性。然而，这并不那么容易，因为属性的名称被存储为路径，我们需要将其转换为字符串。但我邀请你尝试使用`syn`包的文档。

# 实现 setter

本练习的第二部分将是实现我们结构体的 setter。通常，这是通过为每个字段创建一个`set_{field}()`函数来完成的。此外，使用泛型是常见的做法，这样它们就可以与许多不同的类型一起使用。例如，对于`String`字段，如果我们不需要总是使用实际的`String`类型，而是可以使用`&str`、`Cow<_, str>`或`Box<str>`，那就太好了。

我们只需要将输入声明为`Into<String>`。这使得事情变得稍微复杂一些，但我们的 API 看起来会更好。要实现新的 setter，只需稍微修改一下我们之前看到的代码，大部分代码都会被复制。

为了避免这种情况，我们将使用策略模式，这样我们只需简单地将`generate_getter()`函数改为`generate_setter()`，每个字段一个。我还将字段检索移动到了一个新的函数中。让我们看看它看起来如何：

```rs
use syn::{DeriveInput, Fields, Field, Data};
use syn::punctuated::Punctuated;
use syn::token::Comma;

fn get_fields(ast: &DeriveInput) -> &Punctuated<Field, Comma> {
    match ast.data {
        Data::Struct(ref structure) => {
            if let Fields::Named(ref fields) = structure.fields {
                &fields.named
            } else {
                panic!("you cannot implement setters or getters \
                        for unit or tuple structs");
            }
        },
        Data::Union(ref _union) => {
            unimplemented!("sorry, setters and getters are not \
                            implemented for unions yet");
        }
        Data::Enum(ref _enum) => {
            panic!("you cannot derive setters or getters for \
                    enumerations");
        }
    }
}
```

好多了，对吧？它返回一个字段迭代器，这正是我们函数所需要的。我们现在创建一个方法实现函数，它将接收一个函数作为参数，并将用于每个字段：

```rs
fn impl_methods<F>(ast: &DeriveInput, strategy: F) -> Tokens
where F: FnMut(&Field) -> Tokens {
    let methods: Vec<_> = get_fields(ast).iter()
                                         .map(strategy)
                                         .collect();
    let name = &ast.ident;
    let (impl_generics, ty_generics, where_clause) =
                            ast.generics.split_for_impl();

    quote! {
        impl #impl_generics #name #ty_generics #where_clause {
            #(#methods)*
        }
    }
}
```

如你所见，除了对绑定进行小的命名更改以使意义更清晰之外，唯一的重大变化是在函数的`where`签名中添加了一个`FnMut`，这将作为获取器或设置器的实现者。因此，为了调用这个新函数，我们需要更改`derive_getters()`方法并添加新的`derive_setters()`函数：

```rs
#[proc_macro_derive(Getters)]
pub fn derive_getters(input: TokenStream) -> TokenStream {
    // Parse the input tokens into a syntax tree
    let input = syn::parse(input).unwrap();

    // Build the output
    let expanded = impl_methods(&input, generate_getter);

    // Hand the output tokens back to the compiler
    expanded.into()
}

#[proc_macro_derive(Setters)]
pub fn derive_setters(input: TokenStream) -> TokenStream {
    // Parse the input tokens into a syntax tree
    let input = syn::parse(input).unwrap();

    // Build the output
    let expanded = impl_methods(&input, generate_setter);

    // Hand the output tokens back to the compiler
    expanded.into()
}
```

如你所见，这两种方法完全相同，只是在调用`impl_methods()`函数时，它们使用了不同的策略。第一个将生成获取器，第二个生成设置器。最后，让我们看看`generate_setters()`函数将是什么样子：

```rs
use syn::Ident;

fn generate_setter(field: &Field) -> Tokens {
    let name = field.ident
                .expect("named fields must have a name");
    let fn_name = Ident::from(format!("set_{}", name));
    let ty = &field.ty;

    quote! {
        fn #fn_name<T>(&mut self, value: T) where T: Into<#ty> {
            self.#name = value.into();
        }
    }
}
```

代码在大多数方面与`generate_getter()`函数类似，但有一些不同。首先，函数名与`name`不同，因为它需要`set_`前缀。为此，我们创建了一个带有字段名的字符串，并使用该名称创建了一个标识符。

我们通过使用新的函数名，使用可变的`self`，并向函数添加一个新的输入变量（值）来构建设置器。由于我们希望这个值是通用的，我们使用在 where 子句中定义的`T`类型，它是一个可以转换为我们字段类型的类型（`Into<#ty>`）。我们最终将转换后的值赋给我们的字段。

让我们通过在父 crate 的`main.rs`文件中创建一个简短的示例来查看这个获取器和设置器设置是如何工作的。我们在`Cargo.toml`文件中添加了过程宏依赖项，并定义了一个结构体：

```rs
#[macro_use]
extern crate getset_derive;

#[derive(Debug, Getters, Setters)]
struct Alice {
    x: String,
    y: u32,
}
```

没有什么特别的地方；除了`Getters`和`Setters`派生。正如你所见，我们不需要派生一个实际的特质。现在，我们添加一个简单的`main()`函数来测试代码：

```rs
fn main() {
    let mut alice = Alice {
        x: "this is a name".to_owned(),
        y: 34
    };
    println!("Alice: {{ x: {}, y: {} }}",
             alice.x(),
             alice.y());

    alice.set_x("testing str");
    alice.set_y(15u8);
    println!("{:?}", alice);
}
```

我们创建了一个新的`Alice`结构体，并在其中设置了两个字段。当打印这个结构体时，我们可以看到可以直接使用`Alice::x()`和`Alice::y()`获取器。注意，双大括号用于转义。

然后，由于我们有一个可变变量，我们使用设置器来改变`x`和`y`字段的值。正如你所见，我们不必提供`String`或`u32`；我们可以提供任何可以直接转换为这些类型而不会失败的类型。最后，由于我们为`Alice`实现了`Debug`特质，我们可以不使用获取器就打印其内容。执行`cargo run`后的结果应该是以下内容：

```rs
Alice: { x: this is a name, y: 34 }
Alice { x: "testing str", y: 15 }
```

过程宏或自定义派生允许进行非常复杂的代码生成，并且您甚至可以进一步自定义用户体验。正如我们在上一章中看到的，使用`serde`包，我们可以使用`#[serde]`属性。您可以通过在`#[proc_macro_derive]`属性中定义它们来向您的派生包添加自定义属性，如下所示：

```rs
#[proc_macro_derive(Setters, attributes(generic))]
```

然后，您可以通过检查结构体/枚举或其字段在`Field`、`DeriveInput`、`Variant`或`FieldValue`结构中的`attrs`字段来使用它们。例如，您可以允许开发者决定他们是否希望在设置器中使用泛型，或者微调应该泛型的属性。

更多信息可以在官方文档中找到，请参阅[`doc.rust-lang.org/book/first-edition/procedural-macros.html#custom-attributes`](https://doc.rust-lang.org/book/first-edition/procedural-macros.html#custom-attributes)。

# 夜间 Rust 中的元编程

到目前为止，我们一直停留在稳定版 Rust 中，因为它允许向前兼容。尽管如此，也有一些夜间功能可以帮助我们提高对我们生成的代码的控制。尽管如此，它们都是实验性的，它们在稳定之前可能会改变或甚至被删除。

因此，您应该考虑到使用夜间功能可能会在未来破坏您的代码，并且为了与新的 Rust 版本兼容，维护它将需要更多的努力。尽管如此，我们将快速查看即将加入 Rust 的两个新功能。

# 理解编译器插件

Rust 夜间编译器接受加载扩展，称为**插件**。它们实际上改变了编译器的行为，因此它们可以修改语言本身。插件是一个 crate，类似于我们之前创建的过程宏 crate。

过程宏或标准宏与插件之间的区别在于，虽然前两者修改了它们所提供的 Rust 代码，但插件能够执行额外的计算，这可以大大提高 crate 的性能。

由于这些插件在编译器内部加载，它们将能够访问标准宏没有的大量信息。此外，这需要使用`rustc`编译器和`libsyntax`库作为外部 crate，这意味着在编译插件时，您将加载大量的编译器代码。

因此，不要将您的插件作为外部 crate 添加到您的二进制文件中，因为这将创建一个包含大量编译器代码的巨大可执行文件。要使用不作为库添加的插件，您需要夜间编译器，并且必须将`#![feature(plugin)]`和`#![plugin({plugin_name})]`属性添加到您的 crate 中。

当开发一个新插件时，您需要在`Cargo.toml`文件中创建带有一些额外信息的 crate，就像我们为过程宏所做的那样：

```rs
[lib]
plugin = true
```

然后，在 `lib.rs` 文件中，你需要导入所需的库并定义插件注册器。插件正常工作的最低要求如下：

```rs
#![crate_type="dylib"]
#![feature(plugin_registrar, rustc_private)]

extern crate syntax;
extern crate syntax_pos;
extern crate rustc;
extern crate rustc_plugin;

use rustc_plugin::Registry;

#[plugin_registrar]
pub fn plugin_registrar(reg: &mut Registry) {
    unimplemented!()
}
```

尽管在这个例子中，`rustc`、`syntax` 和 `syntax_pos` crate 没有被使用，但在开发插件时，你几乎肯定会需要它们，因为它们提供了你改变任何行为所需所需的类型。`Registry` 对象允许你注册多种新的语言项，例如宏或语法扩展。对于每一个，你都需要定义一个函数来接收编译器标记并修改它们以产生所需的结果。

`#![crate_type = "dylib"]` 属性告诉编译器使用 crate 创建动态库，而不是正常的静态库。这使得库可以被编译器加载。`plugin_registrar` 夜间功能允许我们创建实际的插件注册器函数，而 `rustc_private` 功能允许我们使用私有的 Rust 编译器类型，这样我们就可以使用编译器内部功能。

在撰写本文时，这些 crate 的唯一在线文档是由 Manish Goregaokar 提供的，但这种情况很快就会改变。在他的网站上，你可以找到 `rustc_plugin` 的 API 文档（[`manishearth.github.io/rust-internals-docs/rustc_plugin/index.html`](https://manishearth.github.io/rust-internals-docs/rustc_plugin/index.html)）、`rustc` 的 API 文档（[`manishearth.github.io/rust-internals-docs/rustc/index.html`](https://manishearth.github.io/rust-internals-docs/rustc/index.html)）、`syntax` 的 API 文档（[`manishearth.github.io/rust-internals-docs/syntax/index.html`](https://manishearth.github.io/rust-internals-docs/syntax/index.html)）和 `syntax_pos` 的 API 文档（[`manishearth.github.io/rust-internals-docs/syntax_pos/index.html`](https://manishearth.github.io/rust-internals-docs/syntax_pos/index.html)）。我邀请你阅读这些 crate 的 API，并为编译器创建小型插件。不过，请记住，语法可能会改变，这会使维护变得更加困难。

# 声明式宏

接下来即将加入 Rust 的功能是声明式宏，即宏 2.0 或示例宏。确实，有些人也将标准宏称为声明式宏，因为它们基于相同的原则。但我希望让大家知道这种区别，以便了解这些新宏将为语言带来的改进。

这些新宏引入了 `macro` 关键字，其工作方式将与 `macro_rules!{}` 宏类似，但使用的语法更接近函数语法，而不是当前的语法。不仅如此，它还将模块化引入宏中，这样你就可以在同一个 crate 中拥有两个同名宏，只要它们位于不同的模块中。这种额外的模块化将使 crate 之间的集成变得更加容易。

很遗憾，这些宏还没有提出语法建议，当前 nightly 的实现也不过是未来将要到来的内容的占位符。我邀请你关注这些新宏的标准化进程，甚至通过为社区做出贡献来定义它们的未来语法。

# 摘要

在本章中，你学习了如何创建自己的宏。首先，我们了解了标准宏，并创建了一些可以帮助你更快地开发并编写更高效代码的宏。然后，你学习了过程宏以及如何为你自己的结构和枚举推导出代码。最后，你发现了两个可能在未来稳定版 Rust 中出现的特性，目前可以在 nightly Rust 中使用——插件和声明式宏。

在接下来的两个章节中，我们将讨论 Rust 中的并发。正如你将看到的，一旦我们的单线程代码足够快，向更快执行迈出的下一步就是在并行中计算它。
