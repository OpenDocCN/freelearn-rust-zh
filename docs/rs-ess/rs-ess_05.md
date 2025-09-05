# 第五章. 使用高阶函数和参数化泛化代码

现在我们已经建立了数据结构和控制结构，我们可以开始探索 Rust 的函数式和面向对象特性，这使得它成为一种真正表达性的语言。在本章中，我们将涵盖以下主题：

+   高阶函数和闭包

+   迭代器

+   消费者和适配器

+   泛型数据结构和函数

+   错误处理

+   结构体上的方法

+   特质

+   使用特质约束

+   内置特质和运算符重载

# 高阶函数和闭包

到目前为止，我们知道如何使用函数，如下面的示例所示，其中 `triples` 函数改变了我们的 `strength`，但只有当 `triples` 的返回值赋给 `strength` 时：

```rs
// see code in Chapter 5/code/higher_functions.rs
let mut strength = 26;
println!("My tripled strength equals {}",triples(strength)); // 78
println!("My strength is still {}", strength); // 26
strength = triples(strength);
println!("My strength is now {}", strength); // 78
```

将 `triples` 定义为 `fn triples(s: i32) -> i32 { 3 * s }`，`s` 代表力量。

假设我们的玩家砸碎了一块惊人的力量宝石，使得他的力量翻倍，并且结果的力量再次翻倍，因此我们可以写 `triples(triples(s))`。我们也可以写一个函数来做这件事，但有一个更通用的函数，让我们称它为 `again`，它可以在其结果上应用某种函数 `f`，`F` 类型，使我们能够创建各种新的游戏技巧，如下所示：

```rs
fn again (f: F, s: i32) -> i32  { f(f(s)) }
```

然而，这还不够信息给 Rust；编译器会要求我们解释 `F` 类型是什么。我们可以在参数列表之前添加 `<F: Fn(i32) -> i32>` 来使这一点明确：

```rs
fn again<F: Fn(i32) -> i32>(f: F , s: i32) -> i32 {
 f(f(s))
}

```

`< >`（尖括号）之间的表达式告诉我们 `F` 是一个函数，`Fn`，它接受 `i32` 作为参数并返回一个 `i32` 函数。

现在看看 `triples` 的定义。这正是这个函数所做的（`triples` 有 `F` 类型的签名），因此我们可以再次调用，将 `triples` 作为第一个参数：

```rs
strength = again(triples, strength);
println!("I got so lucky to turn my strength into {}", strength); // 702 (= 3 * 3 * 78)
```

`again` 函数是一个 *高阶函数*，这意味着它是一个接受另一个函数（或多个函数）作为参数的函数。

通常，像 `triples` 这样的简单函数甚至没有定义为命名函数：

```rs
  strength = 78;
 let triples = |n| { 3 * n };
  strength = again(triples, strength);
  println!("My strength is now {}", strength); // 702
```

这里，我们有一个 *匿名函数* 或 *闭包*，`|n| { 3 * n }`，它接受一个 `n` 参数并返回其三倍值。`||`（竖线）标记闭包的开始，它们包含传递给它的参数（当没有参数时，它写作 `||`）。没有必要指出参数的类型或返回值，因为闭包可以从其被调用的上下文中推断它们的类型。

`triples` 函数仅是一个对名称的绑定，这样我们就可以在另一段代码中引用闭包。我们甚至可以省略那个名称，将闭包内联，如下所示：

```rs
  strength = 78;
 strength = again(|n| { 3 * n }, strength);
  println!("My strength is now {}", strength); // 702
```

闭包使用 `n` 参数调用，该参数的值为 `s`，它是 `strength` 的一个副本。大括号也可以省略，以简化闭包如下：

```rs
strength = again(|n| 3 * n , strength);
```

那么，为什么它被称为闭包？这在以下示例中变得更加明显：

```rs
   let x: i32 = 42;
   let print_add = |s| { 
      println!("x is {}", x); 
      x + s
    }; 
   let res = print_add(strength);
   // here the closure is called and "x is 42" is printed
   assert_eq!(res, 744); // 42 + 702
```

`print_add()` 闭包有一个参数并返回一个 32 位整数。`print_add` 闭包知道 `x` 的值以及在其周围作用域中可用的所有其他变量——它将它们“封闭”起来。没有参数的闭包有一个空参数列表，`||`。

此外，还有一种特殊的闭包称为移动闭包，它由 `move` 关键字指示。一个普通的闭包只需要对其封装的变量的引用，但移动闭包会获取所有封装变量的所有权。

之前的例子将使用移动闭包如下编写：

```rs
    let m: i32 = 42;
 let print_add_move = move |s| { 
      println!("m is {}", m); 
      m + s
    };
    let res = print_add_move(strength); // strength == 702
    assert_eq!(res, 744); // 42 + 702
```

移动闭包主要用于程序与不同的并发线程一起工作时（你可以在 第八章 的 *并发和并行* 中看到这一点）。 

正如你将在以下章节中看到的那样，Rust 中广泛使用高阶函数和闭包，因为它们可以使代码更加简洁和易读，并且它们对于泛化计算非常有用。

# 迭代器

`Iterator` 是一个对象，它按顺序返回集合中的项目，从第一个项目到最后一个项目。为了返回下一个项目，它使用 `next()` 方法。在这里，我们有使用 `Option` 的机会：因为迭代器在某个 `next()` 调用可能没有更多值，所以 `next()` 返回 `Option`：当有值时返回 `Some(value)`，当没有更多值时返回 `None`。

具有这种行为的简单对象是一个数字范围，`0...n`。每次我们使用 `for` 循环，例如 `for i in 0...n`，底层的迭代器机制就会被激活。让我们看一个例子：

```rs
// see code in Chapter 5/code/iterators.rs
    let mut rng = 0..7;
 println!("> {:?}", rng.next()); // prints Some(0)
    println!("> {:?}", rng.next()); // prints Some(1)
    for n in rng {
      print!("{} - ", n);
    } // prints 2 - 3 - 4 - 5 - 6 -
```

在这里，我们看到 `next()` 在工作，它产生 0、1 等等；`for` 循环继续到结束。

进行以下练习：

在前面的例子中，我们看到 `next()` 返回一个 `Some` 对象，这是 `Option` 类型的变体（参见 第四章 的 *结果和 Option* 部分，*结构化数据和匹配模式*）。使用 `next()` 在 `rng` 上编写一个无限循环，看看会发生什么。你将如何打破无限循环？使用 `Option` 值上的 `match`。 （例如，请参阅 `Chapter 5/exercises/range_next.rs`）。实际上，我们在这个练习之前看到的 `for` 循环是这个 `loop – match` 构造的语法糖。

迭代器也是遍历数组或切片的首选方式。让我们回顾一下来自 第四章 的外星人数组，`let aliens = ["Cherfer", "Fynock", "Shirack", "Zuxu"];`，*结构化数据和匹配模式*。而不是使用索引逐个显示所有项目，让我们使用 `iter()` 函数以迭代器的方式来做：

```rs
for alien in aliens.iter() {
 print!("{} / ", alien)
 // process alien
}

```

它打印出：`Cherfer / Fynock / Shirack / Zuxu /`。外星变量是`&str`类型，它依次引用每个项目。（技术上，这里它是`&&str`类型，因为项目本身是`&str`类型，但这与这里要说明的点无关。）这要高效得多，也更安全，因为 Rust 不需要进行索引边界检查，我们总是确定在数组的内存中移动。

更短的方法是写：

```rs
for alien in &aliens {
  print!("{} / ", alien)
}
```

外星数组也是`&str`类型，但`print!`宏会自动解引用。如果你想按相反的顺序打印它们，请使用`aliens.iter().rev()`。我们在上一章中遇到的其它迭代器是`Strings`上的`chars()`和`split()`方法。

迭代器天生是惰性的；除非被询问，否则不会生成值，我们通过调用`next()`方法或在循环中应用`for`来询问它们。这很有意义，因为我们不希望在以下绑定中分配一百万个整数：

```rs
let rng = 0..1000_000; // _ makes the number 1000000 more readable
```

我们只想在我们需要的时候分配内存。

# 消费者和适配器

现在，我们将看到一些示例，说明为什么迭代器如此有用。迭代器是惰性的，必须通过调用一个*消费者*来激活以开始使用值。让我们从 0 到 999 的数字范围开始。为了将其转换为向量，我们应用`collect()`消费者：

```rs
// see code in Chapter 5/code/adapters_consumers.rs
let rng = 0..1000;
let rngvec = rng.collect::<Vec<i32>>();
println!("{:?}", rngvec);
```

它打印出范围（我们用...缩短了输出）：[0, 1, 2, 3, 4, ... , 999]

`collect()`遍历整个迭代器，并将所有元素收集到一个容器中，这里在`Vec<i32>`类型中。这个容器不必是迭代器。注意，我们用`Vec<i32>`表示向量的项目类型，但我们也可以写成`Vec<_>`。`collect::<Vec<i32>>()`的表示法是新的；它表示`collect`是一个参数化方法，可以与泛型类型一起工作，正如你将在下一节中看到的。这一行也可以写成：

```rs
let rngvec: Vec<i32> = rng.collect();
```

`find()`消费者获取迭代器中第一个满足其条件（这里，`>= 42`）的值，并将其作为`Option`函数返回，例如：

```rs
  let forty_two = rng.find(|n| *n >= 42);
 println!("{:?}", forty_two);  // prints out Some(42)

```

`find`的值是一个`Option`函数，因为条件可能对所有项目都为假，然后它将返回一个`None`值。条件被包裹在一个`|n| *n >= 42`闭包中，该闭包通过一个`n`引用应用于迭代器的每个项目；这就是为什么我们必须解引用`*n`来获取值。

假设我们只想在我们的范围内有偶数，通过在每个项目上测试闭包条件来生成一个新的范围。这可以通过`filter()`函数来完成，它是一个适配器，因为它从旧迭代器生成一个新的迭代器。它的结果可以像任何迭代器一样收集：

```rs
  let rng_even = rng.filter(|n| is_even(*n))
 .collect::<Vec<i32>>();
  println!("{:?}", rng_even);
```

在这里，`is_even`是以下函数：

```rs
fn is_even(n: i32) -> bool {
  n % 2 == 0
}
```

这打印出：`[0, 2, 4, ..., 996, 998]`，这表明奇数被过滤掉了。

注意我们如何只需在`filter()`的结果上应用`.collect()`就能链式调用我们的消费者/适配器。

现在，如果我们想对结果迭代器中的每个项进行立方运算`(n * n * n)`，我们会怎么做？我们可以通过使用`map()`函数将闭包应用于每个项来生成一个新的范围：

```rs
  let rng_even_pow3 = rng.filter(|n| is_even(*n))
 .map(|n| n * n * n)
                         .collect::<Vec<i32>>();
  println!("{:?}", rng_even_pow3);
```

现在打印出：`[0, 8, 64, ..., 988047936, 994011992]`。

如果你只想获取前五个结果，请在`collect`函数之前插入一个`take(5)`适配器。结果向量将包含`[0, 8, 64, 216, 512]`。

因此，如果你在编译时看到警告`未使用的必须使用的结果：迭代适配器是惰性的，除非被消费不会做任何事情`，你就知道该怎么做——调用一个消费者！

要查看所有消费者和适配器，请查阅`std::iter`模块的文档。

执行以下练习：

另一个非常强大的消费者是`fold()`函数。以下示例计算前一百个整数的和。它从一个基值 0 开始，这也是求和累加器的初始值，然后迭代并添加每个`n`项到和中：

```rs
let sum = (0..101).fold(0, |sum, n| sum + n);
println!("{}", sum); // prints out 5050
```

现在，计算从 1 到 6 的整数的所有立方的乘积。结果应该是 1,728,000，但要注意基值！作为第二个练习，从`[1, 9, 2, 3, 14, 12]`数组中减去所有项，从 0 开始（即 0, 1, 9, 2，以此类推）。这将得到`41`。（作为一个提示，记住迭代器项是一个引用；对于一些示例代码，请参阅`第五章/练习/fold.rs`）。

# 泛型数据结构和函数

泛型是编写一次代码的能力，无需或部分指定类型，以便代码可以用于许多不同的类型。Rust 具有丰富的这种能力，并将其应用于数据结构和函数。

如果一个复合数据结构的项的类型可以是通用的`<T>`类型，则该数据结构是泛型的。`T`可以是`i32`、`f64`、`String`或我们自定义的`Person`等结构体类型。因此，我们不仅可以有`Vec<f64>`，还可以有`Vec<Person>`。如果你将`T`指定为具体类型，那么你必须将数据结构定义中所有出现的`T`替换为该类型。

我们的数据结构可以用泛型`<T>`类型参数化，因此它有多个具体定义——它是多态的。Rust 广泛使用这一概念，我们在第四章中已经遇到过，当我们讨论数组、向量、切片以及`Result`和`Option`类型时，我们称之为“结构化数据和模式匹配”。

假设你想定义一个具有两个字段的结构体，第一个和第二个，但你希望保持这些字段的类型泛型。我们可以这样定义：

```rs
// see code in Chapter 5/code/generics.rs
struct Pair<T> {
 first: T,
 second: T,
}

```

我们现在可以定义一对魔法数字，或一对魔术师，或我们想要的任何东西，如下所示：

```rs
let magic_pair: Pair<u32> = Pair { first: 7, second: 42 };
let pair_of_magicians: Pair<&str> = Pair { first: "Gandalf", second: "Sauron" };
```

如果我们想要编写与泛型数据结构一起工作的函数，它们也必须是泛型的，对吧？作为一个简单的例子，我们如何编写一个返回一对中第二个元素的函数？我们可以这样做：

```rs
fn second<T>(pair: Pair<T>) {
 pair.second;
}
```

我们可以将其调用为`let a = second(magic_pair);`，产生`42`。

注意函数名后面的`<T>`字符；这就是泛型函数的声明方式。

现在，让我们来探讨为什么`Option`和`Result`如此强大。再次给出`Option`类型的定义：

```rs
enum Option<T> {
    Some(T),
    None
}
```

从这个例子中，我们可以定义多个具体类型如下：

```rs
let x: Option<i8> = Some(5);
let pi: Option<f64> = Some(3.14159265359);
let none: Option<f64> = None;
let none2 = None::<f64>;
let name: Option<&str> = Some("Joyce");
```

当类型与值不匹配时，会发生类型不匹配错误，类似于在`let magic: Option<f32> = Some(42)`中使用的`Option`。

我们可以定义一个`Person`结构体如下：

```rs
struct Person {
  name: &'static str,
  id:   i32
}
```

我们也可以定义几个`Person`对象如下：

```rs
let p1 = Person{ name: "James Bond", id: 7 };
let p2 = Person{ name: "Vin Diesel", id: 12 };
let p3 = Person{ name: "Robin Hood", id: 42 };
```

然后，使用这些对象，我们可以为`Person`创建`Option`或向量：

```rs
let op1: Option<Person> = Some(p1);
let pvec: Vec<Person> = vec![p2, p3];
```

在你期望得到一个值，但有可能不会得到值的情况下，你应该使用`Option`类型。一个典型的场景是用户输入。

与此相关的是我们在第四章的“结果和选项”部分首次遇到的`Result`类型，即第四章中的“结构化数据与模式匹配”。当计算应该返回一个结果时，可以使用它，但如果出现错误，它也可以返回一个错误。`Result`是通过以下两个泛型类型——`T`和`E`——定义的：

```rs
enum Result<T, E> {
    Ok(T),
    Err(E)
}
```

这再次显示了 Rust 对安全性的承诺；如果它是`Ok`，它将返回一个`T`类型的值，如果存在问题，则返回一个`E`类型的错误（这通常是一个错误消息字符串）。因此，我们也可以将它们读作`Ok(what)`和`Err(why)`，其中`what`具有`T`类型，而`why`具有`E`类型。

那么，为什么`Option`和`Result`是 Rust 的杀手级特性呢？记得从第四章中的“结果和选项”部分，在*结构化数据与模式匹配*节中，我们是如何在获取数字输入时使用`Option`的？这里再次给出：

```rs
let input_num: Result<u32, _> = buf.trim().parse();
```

在其他语言，如 Java 或 C#中，将输入解析为数字可能会导致异常（当输入包含非数字字符或为空或 null 时），你将不得不使用资源密集型的`try/catch`来处理它。

在 Rust 中，`parse()`的结果是一个`Result`，我们只需使用`match`来测试`Result`的返回值，这是一个更简单的机制：

```rs
match input_num {
 Ok(num) => println!("{}", num),
 Err(ex) => println!("Please input an integer number! {}", ex)
};
```

这里是另一个如何使用`Result`返回错误条件的例子。我们使用`std::num::Float::sqrt()`函数计算浮点数的平方根：

```rs
fn sqroot(r: f32) -> Result<f32, String> {
  if r < 0.0 { 
    return Err("Number cannot be negative!".to_string()); 
  }
  Ok(Float::sqrt(r))
}
```

我们通过返回一个`Err`值来防止对负数取平方根（这将给出 NaN，即“不是一个数字”）。

```rs
let m = sqroot(42.0);
```

这将打印出：“42 的平方根是 6.480741”。

在调用代码中，我们使用我们信任的模式匹配机制来区分这两种情况：

```rs
match m {
   Ok(sq) => println!("The square root of 42 is {}", sq),
   Err(str) => println!("{}", str)
}
```

使用`let m = sqroot(-5.0);`时，错误消息会打印为“数字不能为负！”。

### 注意

对于`Option`和`Result`值，使用`match`可以确保没有空值或错误可以在你的代码中传播，这避免了空指针运行时错误或其他异常导致程序崩溃。

# 错误处理

一个 Rust 程序必须尽可能准备好处理未预见的错误，但意外的事情总是可能发生的，比如整数除以零：

```rs
// see code in Chapter 5/code/errors.rs
let x = 3;
let y = 0;
x / y; 
```

当这种情况发生时，程序会停止并显示以下消息：“`<main>`线程恐慌于'尝试除以零'”。

## 潘克

可能会出现一种非常糟糕的情况（比如除以零），以至于继续运行程序已经没有意义，也就是说，我们无法从错误中恢复。在这种情况下，我们可以调用`panic!("message")`宏，这将释放线程拥有的所有资源，报告消息，然后使程序退出。我们可以改进之前的代码如下：

```rs
if (y == 0) { panic!("Division by 0 occurred, exiting"); }
println!("{}", div(x, y));
```

这里，`div`是以下函数：

```rs
fn div(x: i32, y: i32) -> f32 {
  (x / y) as f32
}
```

还可以使用许多其他宏，如`assert!`系列，来指示这种不受欢迎的条件：

```rs
assert!(x == 5); //thread <main> panicked at assertion failed: x == 5
assert!( x == 5, "x is not equal to 5!");
// thread <main> panicked at "x is not equal to 5!"
assert_eq!(x, 5); // thread '<main>' panicked at 'assertion failed: (left: `3`, right: `5`)',
```

当条件不成立时，它们会导致恐慌情况并退出。如果`assert!`的第二个参数提供了错误消息，则会打印出来，否则会给出通用消息，“断言失败”。`assert!`函数主要用于测试前置和后置条件。

代码中那些通常不会被执行的部分可以包含`unreachable!`宏，当它被执行时会引发恐慌：

```rs
unreachable!(); 
// thread '<main>' panicked at 'internal error: entered unreachable code'

```

## 失败

在大多数情况下，我们希望尝试从错误中恢复并让程序继续运行。幸运的是，我们已经在第四章的“结果和选项”部分以及本章的“通用数据结构和函数”部分看到了处理这种错误的基本技术。

当我们期望一个值时，可以使用`Option<T>`枚举；此时，如果存在值，则给出`Some(T)`枚举，如果没有值或发生失败，则返回`None`值。这样，Rust 强制将“无”以清晰和语法上可识别的形式出现，避免了空指针运行时错误。

`Result<T, E>`枚举可以在正常（成功）情况下返回`Ok(T)`值，在失败情况下返回`Err(E)`值，包含有关错误的信息。在前一节的示例中，我们使用了 Result 来安全地从键盘读取值并创建一个计算数字平方根的安全函数。

# 结构体上的方法

现在，我们将看看 Rust 如何满足那些习惯于 `object.method()` 语法而不是 `function(object)` 的面向对象开发者。在 Rust 中，我们可以在结构体上定义 *方法*，这基本上与传统概念中的 `class` 相当。

假设我们正在开发一个游戏，游戏动作发生在遥远太阳系中的一个星球上，那里居住着敌对外星人。为了这个游戏，让我们定义一个 `Alien` 结构体如下：

```rs
// see code in Chapter 5/code/methods.rs
struct Alien {
  health: u32,
  damage: u32
}
```

在这里，`health` 是外星人的状态，而 `damage` 是当它攻击时你的健康减少的量。我们可以创建一个外星人如下：

```rs
let mut bork = Alien{ health: 100, damage: 5 }; 
```

`health` 参数不能超过 `100`，但当我们创建结构体实例时，我们无法强制这个约束。解决方案是为外星人定义一个 `new` 方法，这样我们就可以测试这个值：

```rs
impl Alien {
 fn new(mut h: u32, d: u32) -> Alien {
 // constraints:
 if h > 100 { h = 100; }
 Alien { health: h, damage: d }
 }
}

```

我们可以按照以下方式构建一个新的 `Alien` 数组：

```rs
let mut berserk = Alien::new(150, 15);

```

我们在 `impl Alien` 块内部定义了 `new` 方法（以及所有其他方法），这个块与 `Alien` 结构体的定义是分开的。在应用了所有约束之后，它返回一个 `Alien` 对象。我们通过 `Alien::new()` 在 `Alien` 结构体本身上调用它。由于它是一个 *静态方法*，所以我们不会在 `Alien` 实例上调用它。这样一个新的方法与面向对象语言中的构造函数非常相似。之所以叫它 `new`，仅仅是因为惯例，因为我们本可以叫它 `create()` 或 `give_birth()`。另一个静态方法可以是所有外星人给出的警告：

```rs
  fn warn() -> &'static str {
    "Leave this planet immediately or perish!"
  }
```

这可以这样调用：

```rs
println!("{}", Alien::warn());

```

当一个特定的外星人攻击时，我们可以为该外星人定义一个方法如下：

```rs
fn attack(&self) {
  println!("I attack! Your health lowers with {} damage points.", self.damage);
}
```

然后按照以下方式在 alien `berserk` 上调用它：`berserk.attack();`。将 `berserk`（调用方法时所在的 `Alien` 对象）的引用作为 `&self` 传递给方法。实际上，`self` 与 Python 中的 `self` 或 Java 或 C# 中的 `this` 类似。实例方法总是有 `&self` 作为参数，与静态方法相对。

在这里，对象以不可变的方式传递，但如果攻击你也会降低外星人的健康值怎么办？让我们添加第二个攻击方法：

```rs
fn attack(&self) {
  self.health -= 10;
}
```

然而，Rust 报错，指出有两个编译错误。首先，它说，`cannot assign to immutable field self.health`。我们可以通过像这样传递一个可变引用来解决这个问题：`fn attack(&mut self)`。但现在 Rust 抱怨，`duplicate definition of value 'attack'`。这意味着 Rust 不允许有两个同名的方法；Rust 中没有方法重载。这是由于类型推断的工作方式造成的。

通过将名称改为 `attack_and_suffer`，我们得到以下内容：

```rs
fn attack_and_suffer(&mut self, damage_from_other: u32) {
  self.health -= damage_from_other;
}
```

在调用 `berserk.attack_and_suffer(31);` 之后，berserk 的健康值现在是 69（其中 `31` 是另一个攻击外星人对 berserk 造成的伤害点数）。

没有方法重载意味着我们只能定义一个新函数（这本身也是可选的）。我们可以为我们的构造函数发明不同的名字，这在代码文档方面是好的。否则，你可以选择所谓的`Builder`模式，你可以在[`doc.rust-lang.org/book/method-syntax.html#builder-pattern`](http://doc.rust-lang.org/book/method-syntax.html#builder-pattern)找到更多信息。

### 注意

注意，在 Rust 中，方法也可以定义在元组和枚举上。

执行以下练习：

复数，如 2 + 5i（i 是-1 的平方根），有一个实部（这里为 2）和一个虚部（5）；两者都是浮点数。定义一个`Complex`结构体和一些方法：

+   一个`new`方法来构造一个复数。

+   一个`to_string`方法，用于打印复数，如 2 + 5i 或 2 – 5i（作为一个提示，使用与`println!`相同方式的`format!`宏，但它返回一个`String`。）

+   一个`add`方法来添加两个复数；这是一个新的复数，其实部是操作数的实部之和，同样适用于虚部。

+   一个`times_ten`方法，通过将两部分都乘以 10 来改变对象本身（作为一个提示，仔细思考这个方法参数。）

+   作为额外奖励，创建一个`abs`方法，用于计算复数的绝对值。（请参阅[`en.wikipedia.org/wiki/Absolute_value`](http://en.wikipedia.org/wiki/Absolute_value)。）

现在，测试你的方法！（有关示例代码，请参阅`第五章/练习/complex.rs`。）Rust 在 crate `num`中定义了一个`Complex`类型。

# 特质

如果我们的游戏真的非常多样化呢？也就是说，除了外星人，我们还有僵尸和捕食者，不用说，他们都想攻击。我们能否将它们的共同行为抽象成它们都拥有的东西？当然，在 Rust 中，我们说它们有一个共同的特质，这类似于其他语言中的接口或超类。让我们称这个特质为`Monster`，因为它们都想攻击，第一个版本可以是这样的：

```rs
// see code in Chapter 5/code/traits.rs
trait Monster {
 fn attack(&self);
}

```

特质只包含方法的描述，即它们的类型声明或签名，但它没有真正的实现。这是逻辑的，因为僵尸、捕食者和外星人可能各自有自己的攻击方法。所以，在函数签名之后的`{}`之间没有代码体，但不要忘记用`;`关闭它。

当我们想要为`Alien`结构体实现`Monster`特质时，我们编写以下代码：

```rs
impl Monster for Alien {

}

```

当我们编译这个程序时，Rust 抛出`not all trait items implemented, missing: 'attack'`错误。这很好，因为 Rust 提醒我们哪些特质的函数我们忘记了实现。以下代码可以使其通过：

```rs
impl Monster for Alien {
  fn attack(&self) {
    println!("I attack! Your health lowers with {} damage points.", self.damage);
  }
}
```

因此，为类型实现的特性必须提供实际代码，该代码将在调用`Alien`对象上的该方法时执行。如果僵尸攻击是两倍糟糕，其`Monster`实现可能如下所示：

```rs
impl Monster for Zombies {
  fn attack(&self) {
    println!("I bite you! Your health lowers with {} damage points.", 2 * self.damage);
  }
}
```

我们可以向特性添加其他方法，例如`new`方法、`noise`方法和`attack_with_sound`方法：

```rs
trait Monster {
    fn new(hlt: u32, dam: u32) -> Self;
    fn attack(&self);
    fn noise(&self) -> &'static str;
 fn attacks_with_sound(&self) {
 println!("The Monster attacks by making an awkward sound {}", self.noise());
 }
}
```

注意，在`new`方法中，生成的对象是`Self`类型，在特性的实际实现中，它成为`Alien`或`Zombie`实现类型。

出现在特性中的函数被称为方法。方法与函数的不同之处在于它们有`&self`作为参数；这意味着它们有被调用的对象作为参数，例如，`fn noise(&self) -> &'static str`。当我们用`zmb1.noise()`调用它时，`zmb1`对象成为 self。

特性可以为方法提供默认代码（类似于这里的`attack_with_sound`方法）。实现类型可以选择采用此默认代码或用自己的版本覆盖它。特性方法中的代码也可以使用`self.method()`调用特性中的其他方法，类似于`attack_with_sound`，其中调用了`self.noise()`。

对于`Zombie`类型，`Monster`特性的完整实现可能如下所示：

```rs
impl Monster for Zombie {
  fn new(mut h: u32, d: u32) -> Zombie {
    // constraints:
    if h > 100 { h = 100; }
    Zombie { health: h, damage: d }
  }
  fn attack(&self) {
    println!("The Zombie bites! Your health lowers with {} damage points.", 2 * self.damage);
  }
  fn noise(&self) -> &'static str {
    "Aaargh!"
  }
}
```

这里是我们游戏场景的一个简短片段：

```rs
let zmb1 = Zombie { health: 75, damage: 15 };
println!("Oh no, I hear: {}", zmb1.noise());
zmb1.attack();
```

它打印出：`Oh no, I hear: Aaargh!`

`The Zombie bites! Your health lowers with 30 damage points.`。

特性不仅限于结构体；它们可以应用于任何类型。一个类型也可以实现多个不同的特性。所有实现的不同方法都被编译成针对它们类型的特定版本，因此编译后，例如，存在针对`Alien`、`Zombie`和`Predator`的新方法。

实现特性中的所有方法可能是繁琐的工作。例如，我们可能希望以这种方式显示我们的生物：

```rs
println!("{:?}", zmb1);
```

不幸的是，这给我们带来了`the trait 'core::fmt::Debug' is not implemented for the type 'Zombie' compiler`错误。所以，从消息中，我们可以推断出这个`{:?}`使用了一个`Debug`特性。如果我们查看文档，我们会发现我们必须实现一个`fmt`方法（指定格式化对象的方式）。然而，编译器在这里又一次帮助我们；如果我们用属性`#[derive(Debug)]`前缀我们的`Zombie`结构定义，那么将自动生成默认代码版本：

```rs
#[derive(Debug)]
struct Zombie { health: u32, damage: u32 }
```

`println!("{:?}", zmb1);`片段现在显示为：`Zombie { health: 75, damage: 15 }`。

这也适用于一系列其他特性。（参见本章的*内置特性和运算符重载*部分和[`rustbyexample.com/trait/derive.html`](http://rustbyexample.com/trait/derive.html)。）

# 使用特性约束

在*泛型数据结构和函数*部分，我们创建了一个`sqroot`函数来计算 32 位浮点数的平方根：

```rs
fn sqroot(r: f32) -> Result<f32, String> {
  if r < 0.0 {
    return Err("Number cannot be negative!".to_string()); 
  }
  Ok(f32::sqrt(r))
}
```

如果我们想计算一个 `f64` 数字的平方根呢？为每种类型制作不同的版本将非常不实用。第一次尝试可能是将 `f32` 替换为泛型类型 `<T>`：

```rs
// see code in Chapter 5/code/trait_constraints.rs
extern crate num;
use num::traits::Float;
fn sqroot<T>(r: T) -> Result<T, String> {
  if r < 0.0 { 
    return Err("Number cannot be negative!".to_string()); 
  }
  Ok(num::traits::Float::sqrt(r))
}
```

然而，Rust 不会同意，因为它对 `T` 一无所知，并且会给出多个错误（`num` 是一个外部库，它通过 `extern crate num` 导入，请参阅第七章 Chapter 7，*组织代码和宏*）：

```rs
binary operation `<` cannot be applied to type `T`
the trait `core::marker::Copy` is not implemented for the type `T`
the trait `core::num::NumCast` is not implemented for the type `T`
…

```

所缺少的所有特性都由 `Float` 特性实现。我们可以断言 `T` 必须实现此特性，作为 `fn sqroot<T: num::traits::Float>`。这被称为在 `T` 类型上放置特性约束或特性界限，这确保了函数可以使用指定特性的所有方法。

为了尽可能通用，我们还使用了 `num::traits::Float` 特性中存在的特殊指示符 `0`，它被命名为 `num::zero();` 因此，我们的函数现在如下所示：

```rs
fn sqroot<T: num::traits::Float>(r: T) -> Result<T, String> {
  if r < num::zero() { 
    return Err("Number cannot be negative!".to_string()); 
  }
  Ok(num::traits::Float::sqrt(r))
}
```

这对以下两个调用都适用：

```rs
println!("The square root of {} is {:?}", 42.0f32, sqroot(42.0f32) );
println!("The square root of {} is {:?}", 42.0f64, sqroot(42.0f64) );
```

这将输出如下：

```rs
The square root of 42 is Ok(6.480741)
The square root of 42 is Ok(6.480741)

```

然而，如果我们尝试如下调用 `sqroot` 在一个整数上，我们会得到一个错误：

```rs
println!("The square root of {} is {:?}", 42, sqroot(42) );
```

我们得到一个错误，“`std::num::Float` 特性未在类型 `_` 上实现” [E0277]，因为整数不是 `Float` 类型。

我们的 `sqroot` 函数是泛型的，适用于任何 `Float` 类型。编译器为它应该与之一起工作的任何类型创建不同的可执行 `sqroot` 方法——在这种情况下，`f32` 和 `f64`。当函数调用是多态的，即函数可以接受不同类型的参数时，Rust 应用此机制。这被称为 `静态` 分发，并且没有运行时开销。这应该与 Java 接口的工作方式形成对比，其中分发是在运行时由 Java 虚拟机动态执行的。然而，Rust 也有动态分发的形式；有关更多详细信息，请参阅 [`doc.rust-lang.org/1.0.0-beta/book/static-and-dynamic-dispatch.html`](http://doc.rust-lang.org/1.0.0-beta/book/static-and-dynamic-dispatch.html)。

以另一种方式编写相同的特性约束是使用 `where` 子句，如下所示：

```rs
fn sqroot<T>(r: T) -> Result<T, String> where T: num::traits::Float { … }

```

为什么存在这种其他形式？嗯，可能存在多个泛型 `T` 和 `U` 类型。此外，每种类型都可以被约束到多个特性（特性之间用 `+` 连接），例如 `Trait1`、`Trait2` 等等，就像在这个虚构的例子中：

```rs
fn multc<T: Trait1, U: Trait1 + Trait2>(x: T, y: U) {}
```

使用 `where` 语法，这可以更易于阅读，如下所示：

```rs
fn multc<T, U>(x: T, y: U) where T: Trait1, U: Trait1 + Trait2 {}
```

执行以下练习：

定义一个具有 `draw` 方法的 `Draw` 特性。定义具有整数字段的 `S1` 结构体类型和具有浮点字段 `S2` 结构体类型。

为 `S1` 和 `S2` 实现特性 `Draw`（绘制打印值，并被 *** 所包围）。

创建一个泛型 `draw_object` 函数，它接受任何实现了 `Draw` 的对象。

测试这些！（请参阅第五章的示例代码 `exercises/draw_trait.rs`）

# 内置特性和运算符重载

Rust 标准库中充满了特性，它们被广泛应用于各个地方。例如，有用于：

+   比较对象（`Eq` 和 `PartialEq` 特性）。

+   排序对象（`Ord` 和 `PartialOrd` 特性）。

+   创建一个空对象（`Default` 特性）。

+   使用 `{:?}` 格式化值（`Debug` 特性，它定义了一个 `fmt` 方法）。

+   复制一个对象（`Clone` 特性）。

+   添加对象（`Add` 特性，它定义了一个 `add` 方法）

    ### 注意

    `+` 运算符只是使用的一个好方法；`add: n + m` 与 `n.add(m)` 相同。所以，如果我们实现了 `Add` 特性，我们就可以使用 `+` 运算符；这被称为运算符重载。许多其他特性也可以用来重载运算符，例如 `Sub(-)`、`Mul(*)`、`Deref (*v)`、`Index([])` 等等。

+   当对象超出作用域时释放对象资源（换句话说，`Drop` 特性，即对象有一个析构函数）

在 *迭代器* 部分中，我们描述了迭代器的工作原理，并在范围和数组上使用了它。实际上，迭代器在 Rust 的 `std::iter::Iterator` 中也被定义为一个特性。从迭代器的文档（参考 [`doc.rust-lang.org/core/iter/trait.Iterator.html`](http://doc.rust-lang.org/core/iter/trait.Iterator.html)）中，我们看到我们只需要定义 `next()` 方法，该方法将迭代器向前推进以返回下一个值作为选项。当 `next()` 为你的对象类型实现时，我们就可以使用 `for in` 循环来遍历对象。

# 摘要

在本章中，我们学习了各种技术，通过使用高阶函数、闭包、迭代器和泛型类型和函数来使我们的代码更加灵活。然后我们回顾了利用泛型类型的基本错误处理机制。

我们还发现了 Rust 的面向对象特性，通过在结构体上定义方法和实现特性。最后，我们看到了特性是 Rust 的结构化概念。

在下一章中，我们将揭示 Rust 语言的瑰宝，这些瑰宝构成了其内存安全行为的基础。
