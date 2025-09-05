# 第四章 结构化数据和模式匹配

迄今为止我们只使用了简单的数据，但要进行真正的编程，需要更多复合和结构化的数据值。其中包含灵活的数组、元组和枚举，以及表示更类似面向对象语言中对象行为的结构体。选项是另一种重要的类型，用于确保考虑了没有返回值的情况。然后，我们将探讨模式匹配，这是 Rust 中另一种典型的函数式结构。然而，我们将首先更仔细地查看字符串。我们将涵盖以下主题：

+   字符串

+   数组、向量和切片

+   元组

+   结构体

+   枚举

+   从控制台获取输入

+   模式匹配

# 字符串

Rust 处理字符串的方式与其他语言中的字符串处理方式略有不同。所有字符串都是有效的 Unicode（UTF-8）字节序列。它们可以包含空字节，但它们不是以空字符终止，如 C 语言中的那样。Rust 区分两种类型的字符串：

+   直至现在我们所使用的字面量字符串，是类型为`&str`的字符串切片。`&`字符表示字符串切片是字符串的引用。它们是不可变的，并且具有固定的大小。例如，以下绑定声明了字符串切片：

    ```rs
    // from Chapter 4/code/strings.rs
       let magician1 = "Merlin";
       let greeting = "Hello, 世界!";
    ```

    否则，我们关心的是明确注释字符串变量及其类型：

    ```rs
     let magician2: &'static str = "Gandalf";

    ```

    `&'static`命令表示字符串是静态分配的。我们之前在第二章中看到了这种表示法，即当我们声明全局字符串常量时。在那个情况下，指定类型是强制性的，但对于一个`let`绑定来说，它是多余的，因为编译器可以推断类型：

    ```rs
    println!("Magician {} greets magician {} with {}", 
            magician1, magician2, greeting);
    ```

    打印输出：`魔术师梅林向魔术师甘道夫问候，世界！`

    这些字符串与程序的生命周期相同；它们具有程序的静态生命周期。它们在`std::str`模块中进行了描述。

+   另一方面，`String`可以动态地增长大小（实际上是一个缓冲区），因此它必须在堆上分配。我们可以使用以下代码片段创建一个空字符串：

    ```rs
     let mut str1 = String::new();

    ```

    每次字符串增长时，它都必须在内存中重新分配。所以，例如，如果你知道它将开始为 25 个字节，你可以通过以下方式分配这么多的内存来创建字符串：

    ```rs
     let mut str2 = String::with_capacity(25);

    ```

    此类型在`std::string`模块中进行了描述。要将字符串切片转换为 String，请使用`to_string`方法：

    ```rs
     let mut str3 = magician1.to_string();

    ```

    `to_string()`方法可以用于将任何对象转换为`String`（更精确地说，任何实现了`ToString`特质的对象；我们将在下一章讨论特质）。此方法在堆上分配内存。

    如果`str3`是一个 String，那么你可以使用`&str3`或`&str3[..]`从它创建一个字符串切片：

    ```rs
     let sl1 = &str3;

    ```

以这种方式创建的字符串切片可以被视为对`String`的视图。它是 String 内部的引用，对其进行操作没有成本。

我更喜欢这种方式而不是`to_string()`来比较字符串，因为使用`&[..]`不会消耗资源，而`to_string()`会分配堆内存：

```rs
  if &str3[] == magician1 {
    println!("We got the same magician alright!")
  }
```

要构建一个`String`，我们可以使用多种方法，如下所示：

+   `push`方法：将一个字符追加到`String`

+   `push_str`方法：将另一个字符串追加到`String`

您可以在以下代码片段中看到它们的作用：

```rs
let c1 = 'q';  // character c1
str1.push(c1);
println!("{}", str1); // q
str1.push_str(" Level 1 is finished - ");
println!("{}", str1); // q Level 1 is finished - 
str1.push_str("Rise up to Level 2");
println!("{}", str1); // q Level 1 is finished - Rise up to Level 2
```

如果您需要逐个按顺序获取`String`的字符，请使用`chars()`方法。此方法返回一个`Iterator`，因此我们可以使用 for in 循环（参见第二章的*循环*部分，*使用变量和类型*），如下所示：

```rs
for c in magician1.chars() {
    print!("{} - ", c); 
} 
```

打印结果为：`M - e - r - l - i - n -`.

要遍历由空格分隔的`String`部分，我们可以使用`split()`方法，该方法还返回一个`Iterator`：

```rs
 for word in str1.split(" ") {
 print!("{} / ", word);
 }

```

打印结果为：`q` `/ Level / 1 / is / finished / - / Rise / up / to / Level / 2 /`.

要更改与另一个字符串匹配的`String`的第一部分，请使用`replace`方法：

```rs
let str5 = str1.replace("Level", "Floor");

```

此代码为修改后的`str5`字符串分配了新的内存。

当您编写一个接受字符串作为参数的函数时，始终将其声明为字符串切片，这是一个字符串的视图，如下面的代码片段所示：

```rs
  fn how_long(s: &str) -> usize { s.len() }
```

原因是传递`String` `str1`作为参数会分配内存，所以我们最好将其作为切片传递。这样做最简单、最优雅的方式如下：

```rs
println!("Length of str1: {}", how_long(&str1));
```

或者：

```rs
println!("Length of str1: {}", how_long(&str1[..]));
```

请参阅[`doc.rust-lang.org/std/str/`](http://doc.rust-lang.org/std/str/)和[`doc.rust-lang.org/std/string/`](http://doc.rust-lang.org/std/string/)的文档以获取更多功能。以下是一个更清晰地显示两种字符串类型之间差异的方案：

| 字符串 | 字符串切片(&str) |
| --- | --- |
| 可变 – 堆内存分配模块：`std::string` | 固定大小 – `String`的视图 – 引用(&)模块：`std::str` |

# 数组、向量和切片

假设我们有一群外星生物来填充游戏关卡，那么我们可能希望将它们的名称存储在一个方便的列表中。Rust 的数组正是我们所需要的：

```rs
// from Chapter 4/code/arrays.rs
let aliens = ["Cherfer", "Fynock", "Shirack", "Zuxu"];
println!("{:?}", aliens);

```

要创建一个数组，请用逗号分隔不同的项目，并将整个内容括在`[ ]`（方括号）内。所有项目都必须是同一类型。这样的数组必须是固定大小的（这必须在编译时已知）且不能更改；这是存储在一个连续的内存块中。

如果项目需要可修改性，请使用`let mut`声明您的数组；然而，即使在这种情况下，项目数量也不能改变。外星人数组可以是类型注释为`[&str; 4]`的类型，其中第一个参数是项目类型，第二个是它们的数量：

```rs
let aliens: [&str; 4] = ["Cherfer", "Fynock", "Shirack", "Zuxu"];
```

如果我们想用三个`Zuxus`初始化一个数组，这也很简单：

```rs
let zuxus = ["Zuxu"; 3];
```

那么您如何创建一个空数组？如下所示：

```rs
let mut empty: [i32; 0] = [];
println!("{:?}", empty); // []
```

我们还可以通过它们的索引访问单个项目，从 0 开始：

```rs
println!("The first item is: {}", aliens[0]); // Cherfer
println!("The third item is: {}", aliens[2]); // Shirack
```

数组中的元素数量由 `aliens.len()` 给出；那么，你是如何获取最后一个元素的？正是这样！通过使用 `aliens[aliens.len() - 1]`。或者，这也可以通过使用 `aliens.iter().last().unwrap();` 来找到。

数组的指针使用自动解引用，因此你不需要显式地使用 `*`，如以下代码片段所示：

```rs
let pa = &aliens;
println!("Third item via pointer: {}", pa[2]);
```

它会打印出：`通过指针访问的第三个元素：Shirack`。你认为当我们尝试如下方式更改一个元素时会发生什么：

```rs
aliens[2] = "Facehugger";
```

希望你没有认为 Rust 会允许这样做，对吧？除非你明确地告诉它外星人可以通过 `let mut aliens = [...];` 来改变，否则这是可以的！

索引在运行时也会检查是否在数组的界限内，即 0 和 `aliens.len()`；如果不是，程序将因运行时错误或恐慌而崩溃：

```rs
println!("This item does not exist: {}", aliens[10]); // runtime error:
```

它给出了以下输出：

```rs
thread '<main>' panicked at 'index out of bounds: the len is 4 but the index is 10'
```

如果我们想逐个连续地遍历元素并打印它们或对它们进行一些有用的操作，我们可以这样做：

```rs
for ix in 0..aliens.len() {
 println!("Alien no {} is {}", ix, aliens[ix]);
}

```

这可以工作，并且它为我们提供了每个元素的索引，这可能很有用。然而，当我们使用索引来获取每个连续的元素时，Rust 也必须每次检查我们是否仍然在内存数组的界限内。这就是为什么这并不非常高效，在第五章的*迭代器*部分，*使用高阶函数和参数化泛化代码*，我们将看到一种更高效的方法，如下迭代元素：

```rs
for a in aliens.iter() {
 println!("The next alien is {}", a); 
}

```

`for` 循环可以写成更短的形式如下：

```rs
for a in &aliens  { … }

```

## 向量

通常，与可以增长（或缩小）大小的数组一起工作更为实用，因为它是分配在堆上的。Rust 通过 `std::vec` 模块中的 `Vec` 向量类型提供了这一点。这是一个泛型类型，这意味着元素可以具有任何 `T` 类型，其中 `T` 在代码中指定；例如，我们可以有 `Vec<i32>` 类型或 `Vec<&str>` 类型的向量。为了表示这是一个泛型类型，它被写成 `Vec<T>`。同样，所有元素都必须是相同的 `T` 类型。我们可以用两种方式创建一个向量，使用 `new()` 或使用 `vec!` 宏。这些在这里展示：

```rs
let mut numbers: Vec<i32> = Vec::new();
let mut magic_numbers = vec![7i32, 42, 47, 45, 54];
```

在第一种情况下，类型通过 `Vec<i32>` 明确指示；在第二种情况下，这是通过给第一个元素添加 `i32` 后缀来完成的，但这通常是可选的。

我们也可以创建一个新的向量并为其分配一个初始内存大小，如果你事先知道你至少需要那么多元素，这可能会很有用。以下初始化了一个用于有符号整数的向量，并为 25 个整数分配了内存：

```rs
let mut ids: Vec<i32> = Vec::with_capacity(25);
```

我们需要在这里提供类型，否则编译器无法计算所需的内存量。

向量也可以通过 `collect()` 方法和一个范围来从迭代器构建，例如在这个例子中：

```rs
let rgvec: Vec<u32> = (0..7).collect();
println!("Collected the range into: {:?}", rgvec);
```

它会打印出：`将范围收集到：[0, 1, 2, 3, 4, 5, 6]`。

索引、获取长度和遍历向量与数组的工作方式相同。例如，一个遍历向量的 `for` 循环可以简单地写成以下形式：

```rs
let values = vec![1, 2, 3];
for n in values {
      println!("{}", n);
}
```

使用 `push()` 向向量的末尾添加新项，使用 `pop()` 移除最后一个项：

```rs
numbers.push(magic_numbers[1]);
numbers.push(magic_numbers[4]);
println!("{:?}", numbers); // [42, 54]
let fifty_four = numbers.pop();// fifty_four now contains 54
println!("{:?}", numbers); // [42]
```

如果一个函数需要返回许多相同类型的值，你可以用这些值创建一个数组或向量，并返回该对象。

## 切片

如果你想对数组或向量的某个部分进行操作，你可能会首先想到将这部分复制到另一个数组中，但 Rust 有一个更安全、更高效的解决方案；取数组的切片。不需要复制，而是你得到对现有数组的视图，类似于字符串切片是字符串的视图。

例如，假设我只需要从我们的 `magic_numbers` 向量中获取数字 42、47 和 45。那么，我可以采取以下切片：

```rs
let slc = &magic_numbers[1..4]; // only the items 42, 47 and 45

```

起始索引 1 是 42 的索引，最后一个索引 4 指向 54，但这个项不包括在内。`&` 表示我们正在引用现有的内存分配。切片与向量共享以下特点：

+   它们是泛型的，对于 `T` 类型具有 `&[T]` 类型

+   它们的大小不必在编译时已知

## 字符串和数组

在本章的第一节中，我们看到了 `String` 中的字符序列是由 `chars()` 函数给出的。这看起来像数组吗？如果我们查看 `String` 字符的内存分配，它是由数组支持的；它存储为一个字节数组 `Vec<u8>` 的向量。

这意味着我们也可以从 `String` 中取出 `&str` 类型的切片：

```rs
let location = "Middle-Earth";
let part = &location[7..12];
println!("{}", part); // Earth
```

我们可以将切片的字符收集到一个向量中，并按如下方式排序：

```rs
let magician = "Merlin";
let mut chars: Vec<char> = magician.chars().collect();
chars.sort();
for c in chars.iter() {
      print!("{} ", c);
}
```

这会打印出 `M e i l n r`（在排序顺序中，大写字母先于小写字母）。以下是一些使用 `collect()` 方法的其他示例：

```rs
let v: Vec<&str> = "The wizard of Oz".split(' ').collect();
let v: Vec<&str> = "abc1def2ghi".split(|c: char| c.is_numeric()).collect();
```

在这里，`split()` 接收一个闭包来决定在哪个字符上分割。切片类型 `&str` 和 `&[T]` 可以分别看作是 `Strings` 和向量的视图。以下方案比较了我们刚刚遇到的类型（`T` 表示一个泛型类型）：

| 固定大小（栈分配） | 切片 |   | 动态大小（可增长）（堆分配） |
| --- | --- | --- | --- |
|   | `&str``类型: &[u8]` | 是视图到 | `String` |
| 数组类型: `[T;size]` | 切片类型: `&[T]` | 是视图到 | 向量类型: `Vec<T>` |

通过参考 `Chapter 4/exercises/chars_string.rs` 执行以下练习：

+   尝试使用 `[0]` 或 `[4]` 来获取字符串的第一个或第五个字符

+   将 `bytes()` 方法与 `chars()` 方法在 `let greeting = "Hello,` `世界``!";` 字符串上比较

# 元组

如果你想要组合一定数量的不同类型的值，那么你可以将它们收集在一个元组中，元组用括号 `(` `)` 括起来，并用逗号分隔，如下所示：

```rs
// from Chapter 4/code/tuples.rs
let thor = ("Thor", true, 3500u32);
println!("{:?}", thor); // ("Thor", true, 3500)
```

`thor` 的类型是 `(&str, bool, u32)`，即：项类型的元组。要在一个索引上提取一个项，使用点语法：

```rs
println!("{} - {} - {}", thor.0, thor.1, thor.2);

```

另一种将项目提取到其他变量中的方法是通过对元组进行*解构*：

```rs
let (name, _, power) = thor;
println!("{} has {} points of power", name, power);

```

这将打印出：`索尔有 3500 点力量值`。

这里`let`语句将左侧的模式与右侧匹配。`_`表示我们对`thor`的第二个项目不感兴趣。

如果元组具有相同的类型，则它们只能相互赋值或比较。一个单元素元组需要这样写：`let one = (1,);`。

需要返回不同类型一些值的函数可以将它们收集在元组中，并如下返回该元组：

```rs
fn increase_power(name: &str, power: u32) -> (&str, u32) {
  if power > 1000 {
    return (name, power * 3);
  } else {
    return (name, power * 2);
  }
}
```

如果我们用以下代码片段调用它：

```rs
let (god, strength) = increase_power(thor.0, thor.2);
println!("This god {} has now {} strength", god, strength);
```

输出是：`这个神索尔现在有 10500 点力量值`。

通过参考`Chapter 4/exercises/tuples_ex.rs)`中的代码进行以下练习：

+   尝试比较元组`(2, 'a')`和`(5, false)`，并解释错误信息。

+   创建一个空元组。我们之前没有遇到过吗？所以，单元值实际上是一个空元组！

# 结构体

通常，你可能需要在程序中将几个不同类型的值一起保存；例如，玩家的得分。让我们假设得分包含表示玩家健康和他们在哪个级别上玩游戏的数字。然后你可以做的第一件事是为这些元组赋予一个共同的名称，例如`struct Score`，或者更好的是，你可以指示值的类型：`struct Score(i32, u8)`，然后我们可以创建一个得分如下：

```rs
   let score1 = Score(73, 2);
```

这些被称为元组结构体，因为它们非常类似于元组。它们包含的值可以按如下方式提取：

```rs
  // from Chapter 4/code/structs.rs
  let Score(h, l) = score1; // destructure the tuple
  println!("Health {} - Level {}", h, l);
```

这将打印出：`健康 73 - 级别 2`。

只有一个字段（称为新类型）的元组结构体使我们能够创建一个基于旧类型的新类型，这样它们就有相同的内存表示。以下是一个例子：

```rs
struct Kilograms(u32);
let weight = Kilograms(250);
let Kilograms(kgm) = weight; // extracting kgm
println!("weight is {} kilograms", kgm);
```

这将打印：`重量是 250 公斤`。

然而，我们仍然需要记住这些数字的含义以及它们属于哪个玩家。我们可以通过定义一个具有命名字段的`struct`来使编码更加简单：

```rs
   struct Player {
      nname: &'static str, // nickname
      health: i32,
      level: u8
  }
```

这可以在`main()`内部或外部定义，尽管后者更受欢迎。现在，我们可以创建玩家实例或对象如下：

```rs
let mut pl1 = Player{ nname: "Dzenan", health: 73, level: 2 };

```

注意对象周围的括号（`{ }`）和`key: value`语法。`nname`字段是一个常量字符串，Rust 要求我们指出其生命周期，即这个字符串在程序中需要多长时间。我们在全局作用域中使用了`&'static`，来自第二章的*全局常量*部分，*使用变量和类型*。

我们可以使用点符号来访问实例的字段：

```rs
println!("Player {} is at level {}", pl1.nname, pl1.level); 

```

如果字段值可以改变，结构变量必须声明为可变的；例如，当玩家进入新关卡时：

```rs
  pl1.level = 3;
```

按照惯例，结构体的名称始终以大写字母开头，并遵循驼峰命名法。它还定义了一个由其项目类型组成的自己的类型。

与元组一样，结构体也可以在 `let` 绑定中进行解构，例如：

```rs
let Player{ health: ht, nname: nn, .. } = pl1; 
println!("Player {} has health {}", nn, ht); 
```

它将打印出：`Player Dzenan has health 73`。这表明你可以重命名字段，如果你想的话可以重新排序它们，或者省略字段。

指针在访问数据结构元素时执行自动解引用，如下所示：

```rs
    let ps = &Player{ nname: "John", health: 95, level: 1 };
    println!("{} == {}", ps.nname, (*ps).nname);
```

结构体（Structs）与 C 语言中的记录或结构体非常相似，甚至与其他语言中的类相似。在 第五章，*使用高阶函数和参数化泛化代码* 中，我们将看到如何定义结构体上的方法。

参考第四章/exercises/monster.rs 中的代码执行以下练习：

+   定义一个具有健康和伤害字段的 `Monster` 结构体。然后，创建一个 `Monster` 并显示其状态。

# 枚举（Enums）

如果某个值只能是有限数量的命名值之一，则将其定义为枚举。例如，如果我们的游戏需要指南针方向，我们可以定义如下：

```rs
// from Chapter 4/code/enums.rs
enum Compass {
 North, South, East, West
}

```

然后按照 `main()` 或另一个函数中的示例使用它：

```rs
  let direction = Compass::West;
```

枚举的值也可以是其他类型或结构体，如下例所示：

```rs
type species = &'static str;

enum PlanetaryMonster {
  VenusMonster(species, i32),
  MarsMonster(species, i32)
}
let martian = PlanetaryMonster::MarsMonster("Chela", 42);
```

在其他语言中，枚举有时被称为联合类型或代数数据类型。如果我们在一开始就在代码文件中创建一个 `use` 函数：

```rs
use PlanetaryMonster::MarsMonster;
```

然后，类型可以缩短，如下所示：

```rs
let martian = MarsMonster("Chela", 42);
```

枚举在使代码清晰方面非常出色，并且在 Rust 中使用得很多。要有效地在代码中使用它们，请参阅本章的 *匹配模式* 部分。

## Result 和 Option

在这里，我们查看 Rust 代码中普遍存在的两种枚举。*Result* 是在标准库中定义的一种特殊枚举。当执行某些操作，可能以以下两种方式结束时使用：

+   成功的话，则返回一个 `Ok` 值（某种类型 `T`）

+   出错时，则返回一个 `Err` 值（类型 `E`）

由于这种情况很常见，因此提供了这样的规定，即值 `T` 和错误 `E` 类型可以尽可能通用或泛型。Result 枚举定义如下：

```rs
enum Result<T, E> {
 Ok(T),
 Err(E)
}

```

*Option* 是在标准库中定义的另一个枚举。当存在值时使用，但也可能不存在值。例如，假设我们的程序期望从控制台读取一个值。然而，当它意外地作为后台程序运行时，它将永远不会得到输入值。Rust 希望在可能的情况下始终采取安全措施，因此在这种情况下，最好将值读取为具有两种可能性的 Option 枚举：

+   `Some`，如果有值

+   `None`，如果没有值

这个值可以是任何类型 `T`，因此选项再次被定义为泛型类型：

```rs
enum Option<T> {
 Some(T),
 None
}

```

# 从控制台获取输入

假设我们想在游戏开始之前捕获玩家的昵称；我们该如何做？输入/输出功能由`std` crate 中的`io`模块处理。它有一个`stdin()`函数用于从控制台读取输入。这个函数返回一个`Stdin`类型的对象，它是输入流的引用。`Stdin`有一个`read_line(buf)`方法用于读取以换行符结束的完整输入行（当用户按下*Enter*键时）。这个输入被读取到一个 String 缓冲区`buf`中。方法是为定义在特定类型上的函数命名，它使用点符号调用，例如`object.method`（参见第五章，*使用高阶函数和参数化泛化代码*）。

因此，我们的代码将如下所示：

```rs
let mut buf = String::new();
io::stdin().read_line(&mut buf); 
```

然而，这对 Rust 来说还不够好；它给出了警告，`未使用的 result 必须使用`。Rust 首先是一个安全的语言，我们必须准备好应对可能发生的一切。读取一行可能成功并提供输入值，但它也可能失败；例如，如果这段代码在后台运行在机器上，那么没有控制台可以获取输入。

你将如何应对这种情况？嗯，`read_line()`返回一个 Result 值，它可以是正常值（一个`Ok`），当一切正常时，或者是一个错误值（一个`Err`），当出现问题时。为了处理可能出现的错误，我们需要一个`ok()`函数和一个`expect()`函数；`ok()`将 Result 转换为 Option 值（它包含读取的字节数）和`expect()`在发生错误时给出这个值或显示其消息。在 Rust 中，当发生无法恢复的错误时，程序会崩溃，`expect()`中的字符串参数会显示出来，告诉我们错误发生的位置。

这段代码是用 Rust 编写的，采用链式形式（第一次看到可能会觉得有点不寻常），如下所示：

```rs
io::stdin().read_line(&mut buf).ok().expect("Error!");
```

Rust 允许我们将这些连续的调用写在单独的行上，这对大多数人来说可以大大澄清代码：

```rs
   // from Chapter 4/code/input.rs
use std::io;

fn main() {
  println!("What's your name, noble warrior?");
  let mut buf = String::new();
  io::stdin().read_line(&mut buf)
      .ok()
      .expect("Failed to read line");
  println!("{}, that's a mighty name indeed!", buf);
}
```

当我们从命令行运行这段代码时，我们得到以下对话：

```rs
What's your name, noble warrior?
Riddick
Riddick
, that's a mighty name indeed!

```

你能猜到为什么`that's a mighty name indeed!`出现在新的一行上吗？这是因为输入`buf`仍然包含一个换行符，`\n!`幸运的是，我们有一个`trim()`方法来从字符串中删除前后空白。如果我们插入以下代码片段中的行：

```rs
let name = buf.trim();
println!("{}, that's a mighty name indeed!", name);
```

现在我们得到了正确的输出：`Riddick，这确实是一个响亮的名字！`

如果输入不成功，我们的程序将崩溃，并显示以下输出：

```rs
What's your name, noble warrior?
thread '<main>' panicked at 'Failed to read line 
```

我们如何从控制台读取一个正整数？

```rs
// from Chapter 4/code/pattern_match.rs
let mut buf = String::new();
io::stdin().read_line(&mut buf)
 .ok()
 .expect("Failed to read number");
let input_num: Result<u32, _> = buf.trim().parse();

```

我们从控制台读取数字到`buf` String 缓冲区，并使用`trim()`处理值；如果出现问题，`expect()`将显示消息。然而，我们读取的内容仍然是一个`String`，因此我们必须将`String`转换为数字。

在这个情况下，`parse()`方法尝试将输入转换为无符号 32 位整数。它返回的实际上又是一个 Result 值；这可以是整数（`Ok<u32>`）或错误（`Err`），当转换失败时。

我们将在第五章的*泛型*部分遇到更多 Option 和 Result 的例子，*使用高阶函数和参数化泛化代码*。

# 匹配模式

但我们如何测试上一节中的`input_num`，它是一个 Result 类型，是否包含值呢？当值是`Ok(T)`函数时，`unwrap()`函数可以像这样提取`T`：

```rs
println!("Unwrap found {}", input_num.unwrap());

```

打印结果为：`Unwrap found 42`。然而，当结果是`Err`值时，这会导致程序因恐慌而崩溃，具体表现为`thread '<main>' panicked at 'called `Result::unwrap()` on an `Err` value'`。这是不好的！

为了解决这个问题，仅使用复杂的 if-else 结构是不够的；我们需要在这里使用 Rust 的神奇 match，它比其他语言的 switch 有更多的可能性，并且在处理错误时经常使用：

```rs
match input_num {
 Ok(num) => println!("{}", num),
 Err(ex) => println!("Please input an integer number! {}", ex)
};

```

`match`函数测试一个表达式的值与所有可能值。只有第一个匹配分支后面的`=>`之后的代码（可以是一个代码块）被执行。所有分支都由逗号分隔。在这种情况下，打印出与输入相同的数字。没有从一分支到下一分支的跳转，因此不需要 break 语句；这使我们能够避免 C++中常见的错误。

为了继续使用`match`的返回值，我们必须将那个值绑定到一个变量上，这是可能的，因为 match 本身是一个表达式：

```rs
let num = match input_num {
        Ok(num) => num,
        Err(_)  => 0
}; 
```

这个`match`从`input_num`中提取数字，以便我们可以将其与其他数字进行比较或进行计算。两个分支都必须返回相同类型的值；这就是为什么我们在`Err`情况下返回`0`（假设我们期望一个大于 0 的数字）。

获取 Result 或 Option 值的另一种方法是使用`if let`构造，如下所示：

```rs
if let Ok(val) = input_num {
    println!("Matched {:?}!", val);
} else {
    println!("No match!");
}
```

`input_num`函数被解构，如果它包含一个值`val`，则提取该值。在某些情况下，这可以简化代码，但你会失去穷尽匹配检查。相同的原理也可以在`while`循环内部应用，如下所示：

```rs
while let Ok(val) = input_num {
    println!("Matched {:?}!", val);
    if val == 42 { break }
}
```

使用`match`，必须覆盖所有可能值，这在我们使用 Result、Option（`Some`或`None`相当穷尽）或其他枚举值匹配时是成立的。

然而，看看当我们测试一个字符串切片时会发生什么：

```rs
// from Chapter 4/code/pattern_match2.rs
let magician = "Gandalf";
match magician {
      "Gandalf" => println!("A good magician!"),
      "Sauron"  => println!("A magician turned bad!")
}
```

这个`match`在`magician`上给出了一个错误：非穷尽模式：`_`未覆盖。毕竟，除了"Gandalf"和"Sauron"之外，还有其他魔术师！编译器甚至给出了解决方案：使用下划线（`_`）表示所有其他可能性；因此，这是一个完整的匹配：

```rs
match magician {
      "Gandalf" => println!("A good magician!"),
      "Sauron"  => println!("A magician turned bad!"),
      _         => println!("No magician turned up!")
}
```

### 小贴士

为了始终确保安全，在测试变量的可能值或表达式时使用 match！

分支的左侧可以包含多个值，如果它们由`|`符号分隔或以起始值 … 结束值的包含值范围形式书写。以下代码片段展示了这一功能的应用：

```rs
let magical_number: i32 = 42;
match magical_number {
      // Match a single value
       1 => println!("Unity!"),
      // Match several values
       2 | 3 | 5 | 7 | 11 => println!("Ok, these are primes"),
      // Match an inclusive range
       40...42 => println!("It is contained in this range"),
      // Handle the rest of cases
        _ => println!("No magic at all!"),
}
```

这会打印出：“它包含在这个范围内”。匹配的值可以使用`@`符号捕获到变量中（这里为`num`），如下所示：

```rs
  num @ 40...42 => println!("{} is contained in this range", num)
```

这会打印出：“42 包含在这个范围内”。

匹配甚至比这更强大；正在匹配的表达式可以在左侧进行解构，并且这甚至可以与称为*守卫*的`if`条件结合：

```rs
 let loki = ("Loki", true, 800u32); 
    match loki {
 (name, demi, _) if demi => {
                            print!("This is a demigod ");
                            println!("called {}", name);
                        },
 (name, _, _) if name == "Thor" => 
                        println!("This is Thor!"),
 (_, _, pow) if pow <= 1000 => 
                        println!("This is a powerless god"),
        _ => println!("This is something else")
    }
```

这会打印出：“这是一个半神洛基”。

注意，由于`demi`是布尔值，我们不必写`if demi == true`。如果你想在分支中不执行任何操作，则写`=> {}`。解构不仅适用于元组，如这个示例所示，还可以应用于结构体。

执行以下练习：

如果你将`_`分支从最后一个位置向上移动会发生什么？请参见`Chapter 4/exercises/pattern_match.rs`中的示例。

使用`..`和`...`记号可能会令人困惑，所以这里是对 Rust 1.0 中情况的总结：

|   | What works | Does not work |
| --- | --- | --- |
| `for in` | `..` exclusive | `...` |
| `Match` | `...` inclusive | `..` |

# 摘要

在本章中，我们增强了在 Rust 中处理复合数据的能力，从字符串、数组、向量以及它们的切片，到元组、结构体和枚举。我们还发现，模式匹配结合解构和守卫是一个编写清晰、优雅代码的非常强大的工具。

在下一章中，我们将看到函数比我们预期的要强大得多。此外，我们将发现结构体可以通过实现特质来拥有方法，这几乎就像在其他语言中的类和接口一样。
