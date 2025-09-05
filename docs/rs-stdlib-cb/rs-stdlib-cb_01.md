# 第一章：学习基础知识

在本章中，我们将介绍以下食谱：

+   字符串连接

+   使用 format! 宏

+   提供默认实现

+   使用构造器模式

+   使用构建器模式

+   通过简单线程实现并行

+   生成随机数

+   使用正则表达式查询

+   访问命令行

+   与环境变量交互

+   从 stdin 读取

+   接受可变数量的参数

# 简介

有些代码片段和思维模式一次又一次地证明是某种编程语言的基础。我们将从查看 Rust 中的一些此类技术开始这本书。它们对于编写优雅和灵活的代码至关重要，你几乎会在你处理的任何项目中使用其中的一些。

接下来的章节将在此基础上构建，并与 Rust 的零成本抽象协同工作，这些抽象与高级语言中的抽象一样强大。我们还将探讨标准库的复杂内部方面，并在无畏并发和谨慎使用 `unsafe` 块的帮助下实现我们自己的类似结构，这使我们能够以与某些系统语言（如 C 语言）相同的低级别工作。

# 字符串连接

在系统编程语言中，字符串操作通常比在脚本语言中要复杂一些，Rust 也不例外。有多种方法可以实现，所有这些方法都以不同的方式管理涉及的资源。

# 准备工作

我们将假设在本书的其余部分，你已打开编辑器，准备就绪的最新 Rust 编译器，以及可用的命令行。截至写作时，最新版本是 `1.24.1`。由于 Rust 对向后兼容性的强大保证，你可以放心，所有展示的食谱（除第十章 使用实验性夜间功能外）都将始终以相同的方式工作。你可以通过 [`www.rustup.rs`](https://www.rustup.rs) 下载最新的编译器和其命令行工具。

# 如何做到...

1.  使用 `cargo new chapter-one` 创建一个 Rust 项目以在本章中工作

1.  导航到新创建的 `chapter-one` 文件夹。在本章的其余部分，我们将假设你的命令行当前位于此目录下

1.  在 `src` 文件夹内，创建一个名为 `bin` 的新文件夹

1.  删除生成的 `lib.rs` 文件，因为我们没有创建库

1.  在 `src/bin` 文件夹中，创建一个名为 `concat.rs` 的文件

1.  添加以下代码，并使用 `cargo run --bin concat` 运行它：

```rs
1  fn main() {
2   by_moving();
3   by_cloning();
4   by_mutating();
5  }
6
7  fn by_moving() {
8   let hello = "hello ".to_string();
9   let world = "world!";
10
11   // Moving hello into a new variable
12   let hello_world = hello + world;
13   // Hello CANNOT be used anymore
14   println!("{}", hello_world); // Prints "hello world!"
15  }
16
17  fn by_cloning() {
18   let hello = "hello ".to_string();
19   let world = "world!";
20
21   // Creating a copy of hello and moving it into a new variable
22   let hello_world = hello.clone() + world;
23   // Hello can still be used
24   println!("{}", hello_world); // Prints "hello world!"
25  }
26
27  fn by_mutating() {
28   let mut hello = "hello ".to_string();
29   let world = "world!";
30
31   // hello gets modified in place
32   hello.push_str(world);
33   // hello is both usable and modifiable
34   println!("{}", hello); // Prints "hello world!"
35  }
```

# 它是如何工作的...

在所有函数中，我们首先为可变长度的字符串分配内存。

我们通过创建一个字符串切片 (`&str`) 并对其应用 `to_string` 函数来实现这一点 [8, 18 和 28]。

Rust 中连接字符串的第一种方法，如 `by_moving` 函数所示，是通过将分配的内存以及一个额外的字符串切片**移动**到一个新的变量 [12] 中。这有几个优点：

+   它非常直接且清晰，因为它遵循常见的编程约定，即使用 `+` 操作符进行连接

+   它只使用不可变数据。请记住，始终尝试编写尽可能少的状态行为代码，因为它会导致更健壮和可重用的代码库

+   它重用了 `hello` 分配的内存 [8]，这使得它非常高效

因此，在可能的情况下，应该优先选择这种方式进行连接。

那么，为什么我们还要列出其他连接字符串的方法呢？好吧，我很高兴你问了，亲爱的读者。虽然这种方法很优雅，但它有两个缺点：

+   在第 [12] 行之后，`hello` 就不再可用，因为它已经被移动。这意味着你不能再以任何方式读取它

+   有时候，你可能实际上更喜欢可变数据，以便在小型、封闭的环境中使用状态

剩下的两个函数分别解决一个关注点。

`by_cloning`[17] 几乎与第一个函数相同，但它将分配的字符串 [22] 克隆到一个临时对象中，在这个过程中分配新的内存，然后将其移动，这样原始的 `hello` 就不会被修改，仍然可访问。当然，这会带来运行时冗余内存分配的代价。

`by_mutating`[27] 是解决我们问题的有状态方法。它就地执行涉及的内存管理，这意味着性能应该与 `by_moving` 相同。最后，它将 `hello` 设置为可变，以便进行进一步更改。你可能注意到这个函数看起来不像其他函数那么优雅，因为它没有使用 `+` 操作符。这是故意的，因为 Rust 尝试通过其设计推动你通过移动数据来创建新变量，而不是修改现有变量。如前所述，你只有在真正需要可变数据或想在非常小且可管理的环境中引入状态时才应该这样做。

# 使用格式化宏

还有另一种组合字符串的方法，也可以用来与其他数据类型（如数字）组合。

# 如何操作...

1.  在 `src/bin` 文件夹中，创建一个名为 `format.rs` 的文件

1.  添加以下代码，并使用 `cargo run --bin format` 运行它

```rs
1  fn main() {
2    let colour = "red";
3    // The '{}' it the formatted string gets replaced by the
   parameter
4    let favourite = format!("My favourite colour is {}", colour);
5    println!("{}", favourite);
6     
7    // You can add multiple parameters, which will be
8    // put in place one after another
9    let hello = "hello ";
10   let world = "world!";
11   let hello_world = format!("{}{}", hello, world);
12   println!("{}", hello_world); // Prints "hello world!"
13     
14   // format! can concatenate any data types that
15   // implement the 'Display' trait, such as numbers
16   let favourite_num = format!("My favourite number is {}", 42);
17   println!("{}", favourite_num); // Prints "My favourite number
     is 42"
18     
19   // If you want to include certain parameters multiple times
20   // into the string, you can use positional parameters
21   let duck_duck_goose = format!("{0}, {0}, {0}, {1}!", "duck",
     "goose");
22   println!("{}", duck_duck_goose); // Prints "duck, duck, duck,
     goose!"
23     
24   // You can even name your parameters!
25   let introduction = format!(
26     "My name is {surname}, {forename} {surname}",
27     surname="Bond",
28     forename="James"
29   );
30   println!("{}", introduction) // Prints "My name is Bond, James
     Bond"
31  }
```

# 它是如何工作的...

`format!` 宏通过接受一个填充有格式化参数（例如，`{}`、`{0}` 或 `{foo}`）的格式字符串和一组参数来组合字符串，然后这些参数被插入到**占位符**中。

我们现在将在第 [16] 行的示例中展示这一点：

```rs
format!("My favourite number is {}", 42);
```

让我们分解一下前面的代码行：

+   `"My favourite number is {}"` 是格式字符串

+   `{}` 是格式化参数

+   `42` 是参数

如所示，`format!`不仅与字符串一起工作，还与数字一起工作。实际上，它与实现`Display`特质的任何`struct`一起工作。这意味着，通过你自己提供这样的实现，你可以轻松地使你的数据结构以任何你想要的方式可打印。

默认情况下，`format!`会依次替换一个参数。如果你想覆盖这种行为，你可以使用位置参数，如`{0}`[21]。了解位置是从零开始的，这里的操作非常直接，`{0}`被第一个参数替换，`{1}`被第二个参数替换，以此类推。

有时，当使用大量参数时，这可能会变得有些难以控制。为此，你可以使用命名参数[26]，就像在 Python 中一样。记住，你所有的未命名参数都必须放在你的命名参数之前。例如，以下是不合法的：

```rs
format!("{message} {}", message="Hello there,", "friendo")
```

应该重写为：

```rs
format!("{message} {}", "friendo", message="Hello there,")
 // Returns "hello there, friendo"
```

# 还有更多...

你可以将位置参数与普通参数结合使用，但这可能不是一个好主意，因为它很容易变得难以理解。在这种情况下，行为如下——想象一下`format!`在内部使用一个计数器来确定下一个要放置的参数。每当`format!`遇到一个没有位置的`{}`时，这个计数器就会增加。这个规则导致以下结果：

```rs
format!("{1} {} {0} {}", "a", "b") // Returns "b a a b"
```

如果你想要以不同的格式显示你的数据，还有很多额外的格式化选项。`{:?}`打印出相应参数的`Debug`特质的实现，通常会产生更冗长的输出。`{:.*}`允许你通过参数指定浮点数的十进制精度，如下所示：

```rs
format!("{:.*}", 2, 1.234567) // Returns "1.23"
```

要获取完整列表，请访问[`doc.rust-lang.org/std/fmt/`](https://doc.rust-lang.org/std/fmt/)。

这个配方中的所有信息都适用于`println!`和`print!`，因为它们本质上是一个宏。唯一的区别是`println!`不返回其处理后的字符串，而是直接打印出来！

# 提供默认实现

通常，当处理表示配置的结构时，你不需要关心某些值，只想默默地给它们分配一个标准值。

# 如何做...

1.  在`src/bin`文件夹中，创建一个名为`default.rs`的文件

1.  添加以下代码，并用`cargo run --bin default`运行它：

```rs
1  fn main() {
2    // There's a default value for nearly every primitive type
3    let foo: i32 = Default::default();
4    println!("foo: {}", foo); // Prints "foo: 0"
5 
6 
7    // A struct that derives from Default can be initialized like
     this
8    let pizza: PizzaConfig = Default::default();
9    // Prints "wants_cheese: false
10   println!("wants_cheese: {}", pizza.wants_cheese);
11 
12   // Prints "number_of_olives: 0"
13   println!("number_of_olives: {}", pizza.number_of_olives);
14 
15   // Prints "special_message: "
16   println!("special message: {}", pizza.special_message);
17 
18   let crust_type = match pizza.crust_type {
19     CrustType::Thin => "Nice and thin",
20     CrustType::Thick => "Extra thick and extra filling",
21   };
22   // Prints "crust_type: Nice and thin"
23   println!("crust_type: {}", crust_type);
24 
25 
26   // You can also configure only certain values
27   let custom_pizza = PizzaConfig {
28     number_of_olives: 12,
29     ..Default::default()
30   };
31 
32   // You can define as many values as you want
33   let deluxe_custom_pizza = PizzaConfig {
34     number_of_olives: 12,
35     wants_cheese: true,
36     special_message: "Will you marry me?".to_string(),
37     ..Default::default()
38   };
39
40 }
41 
42  #[derive(Default)]
43  struct PizzaConfig {
44    wants_cheese: bool,
45    number_of_olives: i32,
46    special_message: String,
47    crust_type: CrustType,
48  }
49 
50  // You can implement default easily for your own types
51  enum CrustType {
52    Thin,
53    Thick,
54  }
55  impl Default for CrustType {
56    fn default() -> CrustType {
57      CrustType::Thin
58    }
59  }
```

# 它是如何工作的...

几乎 Rust 中的每个类型都有一个`Default`实现。当你定义自己的`struct`，且它只包含已经具有`Default`的元素时，你可以选择从`Default`派生[42]。在枚举或复杂结构体的情况下，你可以轻松地编写自己的`Default`实现[55]，因为只需提供一个方法。之后，`Default::default()`返回的`struct`会隐式推断为你的类型，前提是你告诉编译器你的类型实际是什么。这就是为什么在行[3]中我们必须写`foo: i32`，否则 Rust 不知道默认对象实际上应该变成什么类型。

如果你只想指定一些元素并让其他元素保持默认值，你可以使用行[29]中的语法。记住，你可以配置和跳过你想要的任意多个值，如行[33 到 37]所示。

# 使用构造函数模式

你可能已经问过自己如何在 Rust 中惯用方式初始化复杂结构体，考虑到它没有构造函数。答案是简单的，有一个构造函数，它只是一种约定而不是规则。Rust 的标准库经常使用这种模式，所以如果我们想有效地使用 std，我们需要理解它。

# 准备工作

在这个菜谱中，我们将讨论用户如何与`struct`交互。当我们在这个上下文中说“用户”时，我们不是指点击你编写应用程序 GUI 的最终用户。我们指的是实例化和操作`struct`的程序员。

# 如何做到这一点...

1.  在`src/bin`文件夹中，创建一个名为`constructor.rs`的文件

1.  添加以下代码，并用`cargo run --bin constructor`运行它：

```rs
1  fn main() {
2    // We don't need to care about
3    // the internal structure of NameLength
4    // Instead, we can just call it's constructor
5    let name_length = NameLength::new("John");
6 
7    // Prints "The name 'John' is '4' characters long"
8    name_length.print();
9  }
10 
11  struct NameLength {
12    name: String,
13    length: usize,
14  }
15 
16  impl NameLength {
17    // The user doesn't need to setup length
18    // We do it for him!
19    fn new(name: &str) -> Self {
20      NameLength {
21        length: name.len(),
22        name,
23      }
24    }
25 
26    fn print(&self) {
27      println!(
28        "The name '{}' is '{}' characters long",
29          self.name,
30          self.length
31      );
32    }
33  }
```

# 它是如何工作的...

如果一个`struct`提供了一个返回`Self`的`new`方法，那么`struct`的使用者将不会配置或依赖于`struct`的成员，因为它们被认为是处于内部*隐藏*状态。

换句话说，如果你看到一个具有`new`函数的`struct`，总是使用它来创建结构体。

这有一个很好的效果，就是允许你更改结构体的任意多个成员，而用户不会注意到任何变化，因为他们本来就不应该查看它们。

使用这种模式的另一个原因是引导用户以正确的方式实例化`struct`。如果有一个只有一大堆成员需要填充值的列表，用户可能会感到有些迷茫。然而，如果有一个只有几个自文档化参数的方法，感觉会更有吸引力。

# 还有更多...

你可能已经注意到，对于我们的示例，我们实际上并不需要一个`length`成员，我们可以在打印时随时计算长度。我们仍然使用这种模式，以说明它在隐藏实现方面的有用性。它的另一个很好的用途是，当`struct`的成员本身有自己的构造函数，并且需要*级联*构造函数调用时。例如，当我们有一个`Vec`作为成员时，就像我们将在本书后面的*使用向量*部分中看到的那样，在第二章，*处理集合*部分。

有时，你的结构体可能需要多种初始化方式。当这种情况发生时，请尝试仍然提供一个`new()`方法作为你的默认构造方式，并将其他选项根据它们与默认方式的差异进行命名。一个很好的例子又是向量，它不仅提供了一个`Vec::new()`构造函数，还提供了一个`Vec::with_capacity(10)`，它为`10`个元素初始化了足够的空间。更多关于这一点的内容，请参阅第二章，*处理集合*部分。

当接受一种字符串类型（无论是`&str`，即借用字符串切片，还是`String`，即拥有字符串）并计划将其存储在你的`struct`中，就像我们在示例中所做的那样，同时考虑一个`Cow`。不，不是那些大牛奶动物朋友。在 Rust 中，`Cow`是一个围绕类型的*写时复制*包装器，这意味着它将尽可能尝试借用类型，只有在绝对必要时才会创建数据的拥有副本，这发生在第一次变异时。这种实际效果是，如果我们以以下方式重写我们的`NameLength`结构体，它将不会关心调用者传递给它的是`&str`还是`String`，而会尝试以最有效的方式工作：

```rs
use std::borrow::Cow;
struct NameLength<'a> {
    name: Cow<'a, str>,
    length: usize,
}

impl<'a> NameLength<'a> {
    // The user doesn't need to setup length
    // We do it for him!
    fn new<S>(name: S) -> Self
    where
        S: Into<Cow<'a, str>>,
    {
        let name: Cow<'a, str> = name.into();
        NameLength {
            length: name.len(),
            name,
        }
    }

    fn print(&self) {
        println!(
            "The name '{}' is '{}' characters long",
            self.name, self.length
        );
    }
}
```

如果你想了解更多关于`Cow`的信息，请查看 Joe Wilm 的这篇易于理解的博客文章：[`jwilm.io/blog/from-str-to-cow/`](https://jwilm.io/blog/from-str-to-cow/)。

在`Cow`代码中使用的`Into`特质将在第五章，*高级数据结构*部分的*将类型转换为彼此*部分中进行解释。

# 参见

+   *使用向量*配方在第二章，*处理集合*

+   *将类型转换为彼此*配方在第五章，*高级数据结构*

# 使用构建器模式

有时你需要介于构造函数的定制和默认实现的隐式性之间的某种东西。构建器模式就出现了，这是 Rust 标准库中经常使用的一种技术，因为它允许调用者流畅地连接他们关心的配置，并让他们忽略他们不关心的细节。

# 如何实现...

1.  在`src/bin`文件夹中，创建一个名为`builder.rs`的文件

1.  添加以下所有代码并使用 `cargo run --bin builder` 运行它：

```rs
1  fn main() {
2    // We can easily create different configurations
3    let normal_burger = BurgerBuilder::new().build();
4    let cheese_burger = BurgerBuilder::new()
       .cheese(true)
       .salad(false)
       .build();
5    let veggie_bigmac = BurgerBuilder::new()
       .vegetarian(true)
       .patty_count(2)
       .build();
6
7    if let Ok(normal_burger) = normal_burger {
8      normal_burger.print();
9    }
10   if let Ok(cheese_burger) = cheese_burger {
11     cheese_burger.print();
12   }
13   if let Ok(veggie_bigmac) = veggie_bigmac {
14     veggie_bigmac.print();
15   }
16
17   // Our builder can perform a check for
18   // invalid configurations
19   let invalid_burger = BurgerBuilder::new()
       .vegetarian(true)
       .bacon(true)
       .build();
20   if let Err(error) = invalid_burger {
21     println!("Failed to print burger: {}", error);
22   }
23
24   // If we omit the last step, we can reuse our builder
25   let cheese_burger_builder = BurgerBuilder::new().cheese(true);
26   for i in 1..10 {
27     let cheese_burger = cheese_burger_builder.build();
28     if let Ok(cheese_burger) = cheese_burger {
29       println!("cheese burger number {} is ready!", i);
30       cheese_burger.print();
31     }
32   }
33 }
```

这是可配置的对象：

```rs
35 struct Burger {
36    patty_count: i32,
37    vegetarian: bool,
38    cheese: bool,
39    bacon: bool,
40    salad: bool,
41 }
42 impl Burger {
43    // This method is just here for illustrative purposes
44    fn print(&self) {
45        let pretty_patties = if self.patty_count == 1 {
46            "patty"
47        } else {
48            "patties"
49        };
50        let pretty_bool = |val| if val { "" } else { "no " };
51        let pretty_vegetarian = if self.vegetarian { "vegetarian " 
           }
          else { "" };
52        println!(
53            "This is a {}burger with {} {}, {}cheese, {}bacon and
              {}salad",
54            pretty_vegetarian,
55            self.patty_count,
56            pretty_patties,
57            pretty_bool(self.cheese),
58            pretty_bool(self.bacon),
59            pretty_bool(self.salad)
60        )
61    }
62 }
```

这就是构建器本身。它用于配置和创建一个 `Burger`：

```rs
64  struct BurgerBuilder {
65    patty_count: i32,
66    vegetarian: bool,
67    cheese: bool,
68    bacon: bool,
69    salad: bool,
70  }
71  impl BurgerBuilder {
72    // in the constructor, we can specify
73    // the standard values
74    fn new() -> Self {
75      BurgerBuilder {
76        patty_count: 1,
77        vegetarian: false,
78        cheese: false,
79        bacon: false,
80        salad: true,
81      }
82    }
83
84    // Now we have to define a method for every
85    // configurable value
86    fn patty_count(mut self, val: i32) -> Self {
87      self.patty_count = val;
88      self
89    }
90
91    fn vegetarian(mut self, val: bool) -> Self {
92      self.vegetarian = val;
93      self
94    }
95    fn cheese(mut self, val: bool) -> Self {
96      self.cheese = val;
97      self
98    }
99    fn bacon(mut self, val: bool) -> Self {
100     self.bacon = val;
101     self
102   }
103   fn salad(mut self, val: bool) -> Self {
104     self.salad = val;
105     self
106   }
107
108   // The final method actually constructs our object
109   fn build(&self) -> Result<Burger, String> {
110     let burger = Burger {
111       patty_count: self.patty_count,
112       vegetarian: self.vegetarian,
113       cheese: self.cheese,
114       bacon: self.bacon,
115       salad: self.salad,
116   };
117   // Check for invalid configuration
118   if burger.vegetarian && burger.bacon {
119     Err("Sorry, but we don't server vegetarian bacon
             yet".to_string())
120     } else {
121       Ok(burger)
122     }
123   }
124 }
```

# 它是如何工作的...

哇，这是一大堆代码！让我们先把它拆分开来。

在第一部分，我们展示了如何使用这种模式轻松地配置一个复杂的对象。我们通过依赖合理的标准值，只指定我们真正关心的内容来实现这一点：

```rs
let normal_burger = BurgerBuilder::new().build();
let cheese_burger = BurgerBuilder::new()
    .cheese(true)
    .salad(false)
    .build();
let veggie_bigmac = BurgerBuilder::new()
    .vegetarian(true)
    .patty_count(2)
    .build();
```

代码读起来相当清晰，不是吗？

在我们的构建器模式版本中，我们返回一个包裹在 `Result` 中的对象，以便告诉世界存在某些无效配置，并且我们的构建器可能无法始终生成有效的产品。正因为如此，我们必须在访问它之前检查汉堡的有效性[7, 10 和 13]。

我们的无效配置是 `vegetarian(true)` 和 `bacon(true)`。不幸的是，我们的餐厅还没有提供素食培根！当你启动程序时，你会看到以下行会打印出一个错误：

```rs
if let Err(error) = invalid_burger {
    println!("Failed to print burger: {}", error);
}
```

如果我们省略最后的 `build` 步骤，我们可以重复使用构建器来构建我们想要的任意数量的对象。[25 到 32]

让我们看看我们是如何实现所有这些的。在 `main` 函数之后的第一件事是我们 `Burger` 结构体的定义。这里没有惊喜，它只是普通的老数据。`print` 方法只是在这里提供一些在运行时的一些漂亮的输出。如果你想的话可以忽略它。

真正的逻辑在 `BurgerBuilder`[64] 中。它应该为每个你想要配置的值有一个成员。由于我们想要配置汉堡的每个方面，我们将有与 `Burger` 完全相同的成员。在构造函数 [74] 中，我们可以指定一些默认值。然后我们为每个配置创建一个方法。最后，在 `build()` [109] 中，我们首先执行一些错误检查。如果配置是正确的，我们返回由所有成员组成的 `Burger`[121]。否则，我们返回一个错误[119]。

# 还有更多...

如果你希望你的对象可以在没有构建器的情况下构建，你也可以为 `Burger` 提供一个 `Default` 实现。`BurgerBuilder::new()` 然后可以简单地返回 `Default::default()`。

在 `build()` 中，如果你的配置本质上不能是无效的，你当然可以直接返回对象，而无需将其包裹在 `Result` 中。

# 通过简单线程实现并行

每年，随着处理器物理核心数量的不断增加，并行性和并发性变得越来越重要。在大多数语言中，编写并行代码是棘手的。非常棘手。但在 Rust 中不是这样，因为它从一开始就被设计为围绕 *无惧并发* 原则。

# 如何做到这一点...

1.  在 `src/bin` 文件夹中，创建一个名为 `parallelism.rs` 的文件

1.  添加以下代码并使用 `cargo run --bin parallelism` 运行它

```rs
1   use std::thread;
2
3   fn main() {
4     // Spawning a thread lets it execute a lambda
5     let child = thread::spawn(|| println!("Hello from a new
      thread!"));
6     println!("Hello from the main thread!");
7     // Joining a child thread with the main thread means
8     // that the main thread waits until the child has
9     // finished it's work
10    child.join().expect("Failed to join the child thread");
11 
12    let sum = parallel_sum(&[1, 2, 3, 4, 5, 6, 7, 8, 9, 10]);
13    println!("The sum of the numbers 1 to 10 is {}", sum);
14  }
15 
16  // We are going to write a function that
17  // sums the numbers in a slice in parallel
18  fn parallel_sum(range: &[i32]) -> i32 {
19    // We are going to use exactly 4 threads to sum the numbers
20    const NUM_THREADS: usize = 4;
21     
22    // If we have less numbers than threads,
23    // there's no point in multithreading them
24    if range.len() < NUM_THREADS {
25      sum_bucket(range)
26    } else {
27        // We define "bucket" as the amount of numbers
28        // we sum in a single thread
29        let bucket_size = range.len() / NUM_THREADS;
30        let mut count = 0;
31        // This vector will keep track of our threads
32        let mut threads = Vec::new();
33        // We try to sum as much as possible in other threads
34        while count + bucket_size < range.len() {
35          let bucket = range[count..count +
                               bucket_size].to_vec();
36          let thread = thread::Builder::new()
37            .name("calculation".to_string())
38            .spawn(move || sum_bucket(&bucket))
39            .expect("Failed to create the thread");
40          threads.push(thread);
41             
42          count += bucket_size
43    }
44    // We are going to sum the rest in the main thread
45    let mut sum = sum_bucket(&range[count..]);
46         
47    // Time to add the results up
48    for thread in threads {
49      sum += thread.join().expect("Failed to join thread");
50    }
51    sum
52  }
53 }
54 
55  // This is the function that will be executed in the threads
56  fn sum_bucket(range: &[i32]) -> i32 {
57    let mut sum = 0;
58    for num in range {
59      sum += *num;
60    }
61     sum
62  }
```

# 它是如何工作的...

你可以通过调用`thread::spawn`来创建一个新的线程，然后它将开始执行提供的 lambda 表达式。这将返回一个`JoinHandle`，你可以用它来，嗯，连接线程。连接一个线程意味着等待线程完成其工作。如果你不连接一个线程，你无法保证它实际上会完成。然而，当设置线程来执行永远不会完成的任务时，比如监听传入的连接，这可能是有效的。

请记住，你不能预知你的线程完成任何工作的顺序。在我们的例子中，无法预测是*来自新线程的问候！*还是*来自主线程的问候！*将被首先打印出来，尽管大多数时候可能是主线程，因为操作系统需要付出一些努力来创建一个新的线程。这就是为什么小型算法在不并行执行时可能会更快。有时，让操作系统创建和管理新线程的开销可能根本不值得。

如行[49]所示，连接一个线程将返回一个包含你的 lambda 返回值的`Result`。

线程也可以被赋予名称。根据你的操作系统，在崩溃的情况下，将显示负责线程的名称。在行[37]中，我们将我们的新求和线程命名为*calculation*。如果其中任何一个崩溃，我们就能快速识别问题。自己试试，在`sum_bucket`的开始处插入对`panic!();`的调用，以故意崩溃程序并运行它。如果你的操作系统支持命名线程，你现在将被告知你的线程*calculation*因为*显式恐慌*而崩溃。

`parallel_sum`是一个函数，它接受一个整数切片并在四个线程上并行地将它们相加。如果你在处理并行算法方面经验有限，这个函数一开始可能很难理解。我邀请你手动将其复制到你的文本编辑器中，并对其进行操作，以便掌握它。如果你仍然感到有些迷茫，不要担心，我们稍后会再次讨论并行性。

将算法调整为并行运行通常伴随着数据竞争的风险。数据竞争被定义为系统中的行为，其输出依赖于外部事件的随机时间。在我们的例子中，存在数据竞争意味着多个线程试图同时访问和修改一个资源。通常，程序员必须分析他们资源的用法并使用外部工具来捕捉所有的数据竞争。相比之下，Rust 的编译器足够智能，可以在编译时捕捉到数据竞争，并在发现一个时停止。这就是为什么我们不得不在行[35]中调用`.to_vec()`的原因：

```rs
let bucket = range[count..count + bucket_size].to_vec();
```

我们将在后面的配方中介绍向量（第二章 Chapter 2，*与集合一起工作*中的*使用向量*部分），所以如果你对此感兴趣，请随时跳转到第二章 Chapter 2，*与集合一起工作*，然后再回来。其本质是我们将数据复制到 `bucket` 中。如果我们在新线程中将引用传递给 `sum_bucket`，我们就会遇到问题，`range` 引用的内存只保证在 `parallel_sum` 内部存在，但我们的线程允许比父线程存活更久。这意味着理论上，如果我们没有在正确的时间 `join` 线程，`sum_bucket` 可能会不幸地太晚被调用，以至于 `range` 已经无效。

这样就会产生数据竞争，因为我们的函数的结果将取决于操作系统决定启动线程的不可控序列。

但不要只听我的话，自己试试。只需将上述行替换为 `let bucket = &range[count..count + bucket_size];` 并尝试编译它。

# 还有更多...

如果你熟悉并行处理，你可能已经注意到我们这里的算法不是很优化。这是故意的，因为编写 `parallel_sum` 的优雅和高效方式需要使用我们尚未讨论的技术。我们将在第七章 Chapter 7，*并行性和 Rayon*中重新审视这个算法，并以专业的方式重写它。在第七章，我们还将学习如何使用锁并发修改资源。

# 参见

+   *使用 RwLocks 并行访问资源*，请参考第七章 Chapter 7 中的配方，*并行性和 Rayon*

# 生成随机数

如前言所述，Rust 核心团队有意将一些功能从标准库中移除，并将其放入自己的外部 crate 中。生成伪随机数就是这样一种功能。

# 如何实现...

1.  打开之前为你生成的 `Cargo.toml` 文件

1.  在 `[dependencies]` 下添加以下行：

```rs
rand = "0.3"
```

1.  如果你想，你可以访问 rand 的 crates.io 页面([`crates.io/crates/rand`](https://crates.io/crates/rand))来检查最新版本，并使用那个版本。

1.  在 `bin` 文件夹中，创建一个名为 `rand.rs` 的文件

1.  添加以下代码，并用 `cargo run --bin rand` 运行它：

```rs
1   extern crate rand;
2 
3   fn main() {
4     // random_num1 will be any integer between
5     // std::i32::MIN and std::i32::MAX
6     let random_num1 = rand::random::<i32>();
7     println!("random_num1: {}", random_num1);
8     let random_num2: i32 = rand::random();
9     println!("random_num2: {}", random_num2);
10    // The initialization of random_num1 and random_num2
11    // is equivalent.
12 
13    // Every primitive data type can be randomized
14    let random_char = rand::random::<char>();
15    // Altough random_char will probably not be
16    // representable on most operating systems
17    println!("random_char: {}", random_char);
18 
19 
20    use rand::Rng;
21    // We can use a reusable generator
22    let mut rng = rand::thread_rng();
23    // This is equivalent to rand::random()
24    if rng.gen() {
25      println!("This message has a 50-50 chance of being
                  printed");
26    }
27    // A generator enables us to use ranges
28    // random_num3 will be between 0 and 9
29    let random_num3 = rng.gen_range(0, 10);
30    println!("random_num3: {}", random_num3);
31 
32    // random_float will be between 0.0 and 0.999999999999...
33    let random_float = rng.gen_range(0.0, 1.0);
34    println!("random_float: {}", random_float);
35 
36    // Per default, the generator uses a uniform distribution,
37    // which should be good enough for nearly all of your
38    // use cases. If you require a particular distribution,
39    // you specify it when creating the generator:
40    let mut chacha_rng = rand::ChaChaRng::new_unseeded();
41    let random_chacha_num = chacha_rng.gen::<i32>();
42    println!("random_chacha_num: {}", random_chacha_num);
43  }
```

# 它是如何工作的...

在你能够使用 `rand` 之前，你必须告诉 Rust 你正在使用 `crate`，方法是编写：

```rs
extern crate rand;
```

之后，`rand` 将提供随机数生成器。我们可以通过调用 `rand::random();` [6] 或直接使用 `rand::thread_rng();` [22] 来访问它。

如果我们选择第一条路线，生成器需要被告知要生成哪种类型。你可以在方法调用中明确指定类型[6]或者注释结果的变量类型[8]。两者都是等效的，并且会产生完全相同的结果。你使用哪一种取决于你。在这本书中，我们将使用第一种约定。

如你在行[29 和 33]中看到的，如果你在调用上下文中类型是明确的，你不需要`if`。

生成的值将在其类型的`MIN`和`MAX`常量之间。对于`i32`来说，这将是从`std::i32::MIN`和`std::i32::MAX`，或者具体数字是-2147483648 和 2147483647。你可以通过调用以下内容来轻松验证这些数字：

```rs
println!("min: {}, max: {}", std::i32::MIN, std::i32::MAX);
```

如你所见，这些数字非常大。对于大多数用途，你可能需要定义自定义限制。你可以走之前讨论的第二条路线，并使用`rand::Rng`来实现[22]。它有一个`gen`方法，实际上这个方法被`rand::random()`隐式调用，还有一个`gen_range()`方法，它接受最小值和最大值。记住，这个范围是非包含的，这意味着最大值永远不会被达到。这就是为什么在行[29]中，`rng.gen_range(0, 10)`只会生成数字 0, 1, 2, 3, 4, 5, 6, 7, 8, 9，而不包括 10。

所述的所有生成随机值的方法都使用**均匀分布**，这意味着范围内的每个数字都有相同的机会被生成。在某些上下文中，使用其他分布是有意义的。你可以在创建生成器时指定生成器的分布[40]。截至出版时，rand crate 支持 ChaCha 和 ISAAC 分布。

# 还有更多...

如果你想要随机填充整个`struct`，你可以使用`rand_derive`辅助 crate 来从**Rand**派生它。然后你可以生成自己的`struct`，就像生成任何其他类型一样。

# 使用正则表达式查询

在解析简单数据格式时，编写正则表达式（或简称*regex*）通常比使用解析器更容易。Rust 通过其`regex`crate 提供了相当不错的支持。

# 准备工作

为了真正理解这一章，你应该熟悉正则表达式。网上有无数免费资源，比如 regexone ([`www.regexone.com/`](https://www.regexone.com/))。

这个配方不会符合 clippy，因为我们故意使正则表达式**过于**简单，因为我们想将配方的重点放在代码上，而不是正则表达式上。一些显示的示例可以被重写以使用`.contains()`。

# 如何做到这一点...

1.  打开之前为你生成的`Cargo.toml`文件。

1.  在`[dependencies]`下添加以下行：

```rs
regex = "0.2"
```

1.  如果你愿意，你可以访问正则表达式的 crates.io 页面([`crates.io/crates/regex`](https://crates.io/crates/regex))来检查最新版本并使用它。

1.  在`bin`文件夹中，创建一个名为`regex.rs`的文件。

1.  添加以下代码，并使用`cargo run --bin regex`运行它：

```rs
1   extern crate regex;
2
3   fn main() {
4     use regex::Regex;
5     // Beginning a string with 'r' makes it a raw string,
6     // in which you don't need to escape any symbols
7     let date_regex =
        Regex::new(r"^\d{2}.\d{2}.\d{4}$").expect("Failed
          to create regex");
8     let date = "15.10.2017";
9     // Check for a match
10    let is_date = date_regex.is_match(date);
11    println!("Is '{}' a date? {}", date, is_date);
12
13    // Let's use capture groups now
14    let date_regex = Regex::new(r"(\d{2}).(\d{2})
        .(\d{4})").expect("Failed to create regex");
15    let text_with_dates = "Alan Turing was born on 23.06.1912 and
          died on 07.06.1954\. \
16      A movie about his life called 'The Imitation Game' came out
          on 14.11.2017";
17    // Iterate over the matches
18    for cap in date_regex.captures_iter(text_with_dates) {
19      println!("Found date {}", &cap[0]);
20      println!("Year: {} Month: {} Day: {}", &cap[3], &cap[2],
          &cap[1]);
21    }
22    // Replace the date format
23    println!("Original text:\t\t{}", text_with_dates);
24    let text_with_indian_dates =
        date_regex.replace_all(text_with_dates, "$1-$2-$3");
25    println!("In indian format:\t{}", text_with_indian_dates);
26
27    // Replacing groups is easier when we name them
28    // ?P<somename> gives a capture group a name
29    let date_regex = Regex::new(r"(?P<day>\d{2}).(?P<month>\d{2})
        .(?P<year>\d{4})")
30      .expect("Failed to create regex");
31    let text_with_american_dates =
        date_regex.replace_all(text_with_dates,
          "$month/$day/$year");
32    println!("In american format:\t{}", 
      text_with_american_dates);
33    let rust_regex = Regex::new(r"(?i)rust").expect("Failed to
        create regex");
34    println!("Do we match RuSt? {}", 
      rust_regex.is_match("RuSt"));
35    use regex::RegexBuilder;
36    let rust_regex = RegexBuilder::new(r"rust")
37      .case_insensitive(true)
38      .build()
39      .expect("Failed to create regex");
40    println!("Do we still match RuSt? {}",
        rust_regex.is_match("RuSt"));
41  }
```

# 它是如何工作的...

你可以通过调用`Regex::new()`并传递一个有效的正则表达式字符串来构建一个正则表达式对象[7]。大多数情况下，你将想要传递一个形式为`r"..."`的原始字符串。原始字符串意味着字符串中的所有符号都被当作字面值处理，而不需要转义。这很重要，因为正则表达式使用反斜杠(`\`)字符来表示一些重要概念，例如数字(`\d`)或空白(`\s`)。然而，Rust 已经使用反斜杠来转义特殊*不可打印*符号，例如换行符(`\n`)或制表符(`\t`)[23]。如果我们想在普通字符串中使用反斜杠，我们必须通过重复它来转义（`\\`）。或者第[14]行的正则表达式必须被重写为：

```rs
"(\\d{2}).(\\d{2}).(\\d{4})"
```

更糟糕的是，如果我们想匹配反斜杠本身，我们也必须因为正则表达式而转义它！在普通字符串中，我们必须四次转义它！（`\\\\`）

我们可以使用原始字符串来避免丢失可读性和混淆，并正常编写我们的正则表达式。实际上，在*每个*正则表达式中使用原始字符串被认为是一种良好的风格，即使它没有反斜杠[33]。这有助于你未来的自己，如果你发现你实际上确实想使用需要反斜杠的功能。

我们可以遍历我们的正则表达式的结果[18]。每次匹配时我们得到的对象是我们捕获组的集合。请注意，零索引始终是整个捕获[19]。第一个索引然后是我们第一个捕获组的字符串，第二个索引是第二个捕获组的字符串，依此类推。[20]。不幸的是，我们没有在索引上获得编译时检查，所以如果我们访问`&cap[4]`，我们的程序会编译，但在运行时崩溃。

在替换时，我们遵循相同的概念：`$0`是整个匹配，`$1`是第一个捕获组的结果，依此类推。为了使我们的生活更轻松，我们可以通过从`?P<somename>`开始给捕获组命名，然后在替换时使用这个名称[29][31]。

你可以指定许多标志，形式为`(?flag)`，用于微调，例如`i`，它使匹配不区分大小写[33]，或者`x`，它在正则表达式中忽略空白。如果你想了解更多，请访问它们的文档([`doc.rust-lang.org/regex/regex/index.html`](https://doc.rust-lang.org/regex/regex/index.html))。不过，大多数情况下，你可以通过使用正则表达式 crate 中的`RegexBuilder`来获得相同的结果[36]。我们在第[33]行和第[36]行生成的两个`rust_regex`对象是等效的。虽然第二个版本确实更冗长，但它一开始就更容易理解。

# 还有更多...

正则表达式通过在创建时将它们的字符串编译成等效的 Rust 代码来工作。出于性能考虑，建议你重用正则表达式，而不是每次使用时都重新创建它们。一个很好的方法是使用`lazy_static`包，我们将在本书的“创建懒静态对象”部分（第五章，[6b8b0c3c-2644-4684-b1f4-b1e08d62450c.xhtml]，*高级数据结构*）中稍后讨论。

注意不要过度使用正则表达式。正如他们所说，“当你只有一把锤子时，一切看起来都像钉子。”如果你解析复杂的数据，正则表达式可以迅速变得难以置信地复杂。当你注意到你的正则表达式变得太大，以至于一眼看不过来时，尝试将其重写为解析器。

# 参见

+   *创建懒静态对象*配方在第五章，**高级数据结构**中

# 访问命令行

总有一天，你将想以某种方式与用户交互。最基本的方法是在通过命令行调用应用程序时让用户传递参数。

# 如何做到...

1.  在`bin`文件夹中，创建一个名为`cli_params.rs`的文件

1.  添加以下代码，并用`cargo run --bin cli_params some_option some_other_option`运行它：

```rs
1   use std::env;
2
3   fn main() {
4     // env::args returns an iterator over the parameters
5     println!("Got following parameters: ");
6     for arg in env::args() {
7       println!("- {}", arg);
8     }
9
10    // We can access specific parameters using the iterator API
11    let mut args = env::args();
12    if let Some(arg) = args.nth(0) {
13      println!("The path to this program is: {}", arg);
14    }
15    if let Some(arg) = args.nth(1) {
16        println!("The first parameter is: {}", arg);
17    }
18    if let Some(arg) = args.nth(2) {
19        println!("The second parameter is: {}", arg);
20    }
21
22    // Or as a vector
23    let args: Vec<_> = env::args().collect();
24    println!("The path to this program is: {}", args[0]);
25    if args.len() > 1 {
26        println!("The first parameter is: {}", args[1]);
27    }
28    if args.len() > 2 {
29        println!("The second parameter is: {}", args[2]);
30    }
31  }
```

# 它是如何工作的...

调用`env::args()`返回一个提供参数的迭代器[6]。按照惯例，大多数操作系统上的第一个命令行参数是可执行文件的路径[12]。

我们可以通过两种方式访问特定的参数：将它们保持为迭代器[11]或`collect`到像`Vec`[23]这样的集合中。不用担心，我们将在第二章，*与集合一起工作*中详细讨论它们。现在，你只需要知道：

+   访问迭代器会强制你在编译时检查元素是否存在，例如，使用`if let`绑定[12]

+   访问向量时在运行时检查其有效性

这意味着我们可以在[25]和[28]中首先检查它们的有效性之前执行[26]和[29]行。试试看，在程序末尾添加`&args[3];`行并运行它。

我们无论如何都会检查长度，因为这被认为是一种良好的风格，以确保预期的参数已被提供。使用迭代器方式访问参数时，你不必担心忘记检查，因为它会强制你这么做。另一方面，通过使用向量，你可以在程序开始时检查一次参数，之后就不必再担心了。

# 还有更多...

如果你正在以*nix 工具的风格构建一个严肃的命令行工具，你将不得不解析很多不同的参数。与其重新发明轮子，你应该看看第三方库，例如 clap ([`crates.io/crates/clap`](https://crates.io/crates/clap))。

# 与环境变量交互

根据《十二要素应用》([`12factor.net/`](https://12factor.net/))，你应该将配置存储在环境中([`12factor.net/config`](https://12factor.net/config))。这意味着你应该将可能在部署之间改变值的值，如端口、域名或数据库句柄，作为环境变量传递。许多程序也使用环境变量来相互通信。

# 如何做...

1.  在 `bin` 文件夹中，创建一个名为 `env_vars.rs` 的文件

1.  添加以下代码，并使用 `cargo run --bin env_vars` 运行它：

```rs
1   use std::env;
2
3   fn main() {
4     // We can iterate over all the env vars for the current
      process
5     println!("Listing all env vars:");
6     for (key, val) in env::vars() {
7       println!("{}: {}", key, val);
8     }
9
10    let key = "PORT";
11    println!("Setting env var {}", key);
12    // Setting an env var for the current process
13    env::set_var(key, "8080");
14
15    print_env_var(key);
16
17    // Removing an env var for the current process
18    println!("Removing env var {}", key);
19    env::remove_var(key);
20
21    print_env_var(key);
22  }
23
24  fn print_env_var(key: &str) {
25    // Accessing an env var
26    match env::var(key) {
27      Ok(val) => println!("{}: {}", key, val),
28      Err(e) => println!("Couldn't print env var {}: {}", key, e),
29    }
30  }
```

# 它是如何工作的...

使用 `env::vars()`，我们可以访问在执行时为当前进程设置的 `env var` 的迭代器 [6]。这个列表相当庞大，正如你运行代码时将看到的，大部分对我们来说都是不相关的。

使用 `env::var()` [26] 访问单个 `env var` 更为实用，它如果请求的变量不存在或包含无效的 Unicode，则返回一个 `Err`。我们可以在第 [21] 行看到这一点，在那里我们尝试打印一个刚刚删除的变量。

因为你的 `env::var` 返回一个 `Result`，你可以通过使用 `unwrap_or_default` 来轻松地为它们设置默认值。一个涉及运行中的流行 Redis ([`redis.io/`](https://redis.io/)) 键值存储地址的现实生活例子如下：

```rs
redis_addr = env::var("REDIS_ADDR")
    .unwrap_or_default("localhost:6379".to_string());
```

请记住，使用 `env::set_var()` [13] 创建 `env var` 和使用 `env::remove_var()` [19] 删除它都只改变我们当前进程的 `env var`。这意味着创建的 `env var` 不会被其他程序读取。这也意味着如果我们不小心删除了一个重要的 `env var`，操作系统的其他部分不会关心，因为它仍然可以访问它。

# 还有更多...

在这个菜谱的开始，我写了关于将你的配置存储在环境中的内容。做这件事的行业标准方式是创建一个名为 `.env` 的文件，其中包含键值对形式的配置，并在构建过程中某个时刻将其加载到进程中。在 Rust 中这样做的一个简单方法是通过使用 dotenv ([`crates.io/crates/dotenv`](https://crates.io/crates/dotenv)) 第三方包。

# 从 stdin 读取

如果你想要创建一个交互式应用程序，使用命令行来原型化你的功能很容易。对于 CLI 程序，这将是所有需要的交互。

# 如何做...

1.  在 `src/bin` 文件夹中，创建一个名为 `stdin.rs` 的文件

1.  添加以下代码，并使用 `cargo run --bin stdin` 运行它：

```rs
1   use std::io;
2   use std::io::prelude::*;
3
4   fn main() {
5     print_single_line("Please enter your forename: ");
6     let forename = read_line_iter();
7
8     print_single_line("Please enter your surname: ");
9     let surname = read_line_buffer();
10
11    print_single_line("Please enter your age: ");
12    let age = read_number();
13
14    println!(
15      "Hello, {} year old human named {} {}!",
16      age, forename, surname
17    );
18  }
19
20  fn print_single_line(text: &str) {
21    // We can print lines without adding a newline
22    print!("{}", text);
23    // However, we need to flush stdout afterwards
24    // in order to guarantee that the data actually displays
25    io::stdout().flush().expect("Failed to flush stdout");
26  }
27
28  fn read_line_iter() -> String {
29    let stdin = io::stdin();
30    // Read one line of input iterator-style
31    let input = stdin.lock().lines().next();
32    input
33      .expect("No lines in buffer")
34      .expect("Failed to read line")
35      .trim()
36      .to_string()
37  }
38
39  fn read_line_buffer() -> String {
40    // Read one line of input buffer-style
41    let mut input = String::new();
42    io::stdin()
43      .read_line(&mut input)
44      .expect("Failed to read line");
45    input.trim().to_string()
46  }
47
48  fn read_number() -> i32 {
49    let stdin = io::stdin();
50    loop {
51      // Iterate over all lines that will be inputted
52      for line in stdin.lock().lines() {
53        let input = line.expect("Failed to read line");
54        // Try to convert a string into a number
55        match input.trim().parse::<i32>() {
56          Ok(num) => return num,
57            Err(e) => println!("Failed to read number: {}", e),
58        }
59      }
60    }
61  }
```

# 它是如何工作的...

为了从标准控制台输入`stdin`读取，我们首先需要获取它的句柄。我们通过调用`io::stdin()`[29]来完成这个操作。想象一下返回的对象是一个全局`stdin`对象的引用。这个全局缓冲区由一个`Mutex`管理，这意味着一次只有一个线程可以访问它（关于这一点，本书将在第七章的*使用 Mutex 并行访问资源*部分中进一步讨论，*并行性和 Rayon*）。我们通过锁定（使用`lock()`）缓冲区来获取这个访问权限，这会返回一个新的句柄[31]。完成这个操作后，我们可以在它上面调用`lines`方法，该方法返回用户将写入的行的迭代器[31 和 52]。关于迭代器的更多信息，请参阅第二章的*作为迭代器访问集合*部分，*处理集合*。

最后，我们可以迭代提交的任意多行，直到达到某种中断条件，否则迭代将永远进行下去。在我们的例子中，一旦输入了有效数字，我们就中断数字检查循环[56]。

如果我们对输入不太挑剔，只想获取下一行，我们有两个选择：

+   我们可以继续使用`lines()`提供的无限迭代器，但只需简单地调用它的`next`方法来获取第一个元素。这附带了一个额外的错误检查，因为一般来说，我们无法保证存在下一个元素。

+   我们可以使用`read_line`来填充现有的缓冲区[43]。这不需要我们先`lock`处理程序，因为它被隐式地完成了。

虽然它们都产生了相同的效果，但你应该选择第一个选项。它更符合惯例，因为它使用迭代器而不是可变状态，这使得它更易于维护和阅读。

顺便提一下，我们在这个配方的一些地方使用`print!`而不是`println!`是出于美观原因[22]。如果你更喜欢在用户输入前显示换行符的外观，你可以避免使用它们。

# 还有更多...

这个配方是基于你希望使用 stdin 进行实时交互的`cli`编写的。如果你计划将一些数据通过管道输入（例如，`cat foo.txt | stdin.rs`在*nix 上），你可以停止将`lines()`返回的迭代器视为无限，并检索单独的行，这与你在上一个配方中检索单独参数的方式类似。

在我们的配方中有各种对`trim()`的调用[35, 45 和 55]。此方法删除前导和尾随空白，以提高我们程序的易用性。我们将在第二章的*使用字符串*部分中详细探讨它，*处理集合*。

# 参见

+   第一章中的*与环境变量交互*配方，*学习基础知识*

+   第二章中的*使用字符串*和*将集合作为迭代器访问配方*，*与集合一起工作*

+   第七章中的*使用 Mutex 并行访问资源配方*，*并行性和 Rayon*

# 接受可变数量的参数

大多数时候，当你想要对一个数据集进行操作时，你会设计一个接受集合的函数。然而，在某些情况下，拥有只接受未绑定数量参数的函数会更好，就像 JavaScript 的*剩余参数*。这个概念被称为**可变参数函数**，在 Rust 中不受支持。但是，我们可以通过定义递归宏来实现它。

# 入门

这个配方中的代码可能很小，但如果你不熟悉宏，它看起来可能像是乱码。如果你还没有学习关于宏的内容或者需要复习，我建议你快速查看官方 Rust 书籍中的相关章节([`doc.rust-lang.org/stable/book/first-edition/macros.html`](https://doc.rust-lang.org/stable/book/first-edition/macros.html))。

# 如何做...

1.  在`src/bin`文件夹中，创建一个名为`variadic.rs`的文件

1.  添加以下代码，并使用`cargo run --bin variadic`运行它：

```rs
1   macro_rules! multiply {
2     // Edge case
3     ( $last:expr ) => { $last };
4
5     ( $head:expr, $($tail:expr), +) => {
6       // Recursive call
7       $head * multiply!($($tail),+)
8     };
9   }
10
11  fn main() {
12    // You can call multiply! with
13    // as many parameters as you want
14    let val = multiply!(2, 4, 8);
15    println!("2*4*8 = {}", val)
16  }
```

# 它是如何工作的...

让我们从我们的意图开始：我们想要创建一个名为 multiply 的宏，它可以接受未定义数量的参数并将它们全部相乘。在宏中，这是通过递归实现的。我们每次递归定义都以**边缘情况**开始，即递归应该停止的参数。大多数时候，这是函数调用不再有意义的地方。在我们的例子中，这是一个单独的参数。想想看，`multiply!(3)`应该返回什么？由于我们没有其他参数与之相乘，所以将它与任何东西相乘都没有意义。我们最好的反应就是简单地返回未修改的参数。

我们的另一个条件是匹配多个参数，一个`$head`和一个用逗号分隔的参数列表，位于`$tail`内部。在这里，我们只定义返回值为`$head`与`$tail`乘积的结果。这将调用`multiply!`与`$tail`和没有`$head`，这意味着在每次调用中我们处理一个更少的参数，直到我们最终达到边缘情况，一个单独的参数。

# 还有更多...

请记住，你应该谨慎使用这种技术。大多数时候，直接接受并操作一个切片会更清晰。然而，与其他宏和更高层次的概念结合使用是有意义的，在这些情况下，“可触摸的事物列表”的类比就失效了。找到一个好的例子很难，因为它们往往非常具体。不过，你可以在书的末尾找到一个例子。

# 参见

+   第十章中的*函数组合*配方，*使用实验性 Nightly 功能*
