# 函数式编程 – 一种比较

**函数式编程**（**FP**）是继**面向对象编程**（**OOP**）之后的第二大流行编程范式。多年来，这两个范式被分离到不同的语言中，以避免混合。多范式语言试图支持这两种方法。Rust 就是这样一种语言。

广义上，函数式编程强调使用可组合和最大可重用函数来定义程序行为。使用这些技术，我们将展示函数式编程如何巧妙地解决了许多常见但困难的问题。本章将概述本书中提出的多数概念。剩余的章节将致力于帮助您掌握每种技术。

我们希望提供的成果如下：

+   能够使用函数式风格来减少代码的重量和复杂性

+   能够通过利用安全抽象编写健壮且安全的代码

+   能够使用函数式原则来构建复杂的项目

# 技术要求

运行提供的示例需要一个较新的 Rust 版本，可以在以下链接找到：

[`www.rust-lang.org/en-US/install.html`](https://www.rust-lang.org/en-US/install.html)

本章的代码也可在 GitHub 上找到，链接如下：

[`github.com/PacktPublishing/Hands-On-Functional-Programming-in-RUST`](https://github.com/PacktPublishing/Hands-On-Functional-Programming-in-RUST)

每章的`README.md`文件中也包含了具体的安装和构建说明。

# 减少代码的重量和复杂性

函数式编程可以大大减少完成任务所需的代码量和复杂性。特别是在 Rust 中，正确应用函数式原则可能会简化通常复杂的设计要求，并使编程成为一种更加高效和有回报的体验。

# 使泛型更加通用

使泛型更加通用与起源于函数式语言的参数化数据结构和函数的实践相关。在 Rust 和其他语言中，这被称为**泛型**。类型和函数都可以进行参数化。可以在泛型类型上放置一个或多个约束，以指示特性和生命周期的要求。

没有泛型的情况下，结构定义可能会变得冗余。以下是对三个定义了“点”这一公共概念的结构的定义。然而，这些结构使用了不同的数值类型，因此单一的概念在`intro_generics.rs`中扩展成了三个独立的`PointN`类型定义：

```rs
struct PointU32 
{
    x: u32,
    y: u32
}

struct PointF32
{
    x: f32,
    y: f32
}

struct PointI32
{
    x: i32,
    y: i32
}
```

相反，我们可以使用泛型来删除重复代码并使代码更加健壮。泛型代码更容易适应新的要求，因为许多行为（以及因此的需求）可以被参数化。如果需要更改，最好是只更改一行而不是一百行。

此代码片段定义了一个参数化的 `Point` 结构体。现在，单个定义可以捕获 `Point` 在 `intro_generics.rs` 中所有可能的数值类型：

```rs
struct Point<T>
{
    x: T,
    y: T
}
```

没有泛型，函数也存在问题。

这里是一个简单的平方数字的函数。然而，为了捕获可能的数值类型，我们在 `intro_generics.rs` 中定义了三个不同的函数：

```rs
fn foo_u32(x: u32) -> u32
{
    x*x
}

fn foo_f32(x: f32) -> f32
{
    x*x
}

fn foo_i32(x: i32) -> i32
{
    x*x
}
```

函数参数，如这个例子所示，可能需要特质界限（指定一个或多个特质的约束）以允许在函数体中使用该类型上的任何行为。

这里是重新定义的 `foo` 函数，带有参数化类型。单个函数可以定义所有数值类型的操作。在 `intro_generics.rs` 中，甚至对于乘法或复制等基本操作，也必须显式设置界限：

```rs
fn foo<T>(x: T) -> T
where T: std::ops::Mul<Output=T> + Copy
{
    x*x
}
```

函数本身也可以作为参数传递。我们称之为高阶函数。

这里是一个接受函数和参数的简单函数，然后使用参数调用该函数，并返回结果。注意特质界限 `Fn`，表示提供的函数是一个闭包。为了使对象可调用，它必须在 `intro_generics.rs` 中实现 `fn`、`Fn`、`FnMut` 或 `FnOnce` 中的一个特质：

```rs
fn bar<F,T>(f: F, x: T) -> T
where F: Fn(T) -> T
{
    f(x)
}
```

# 函数作为值

函数是函数式编程的主要特性。具体来说，函数作为值是整个范式的基石。忽略许多细节，我们还将在此处引入术语 **闭包** 以供将来参考。闭包是一个充当函数的对象，实现了 `fn`、`Fn`、`FnMut` 或 `FnOnce`。

可以使用内置的闭包语法定义简单的闭包。这种语法的好处是，如果允许，`fn`、`Fn`、`FnMut` 和 `FnOnce` 特质将自动实现。这种语法非常适合简短的数据操作。

这里是一个从 `0` 到 `10` 的范围迭代器，映射到平方值。平方操作是通过将内联闭包定义发送到迭代器的 `map` 函数来应用的。此表达式的结果将是一个迭代器。以下是在 `intro_functions.rs` 中的一个表达式：

```rs
(0..10).map(|x| x*x);
```

如果使用块语法，闭包也可以有复杂的主体和语句。

这里是一个从 `0` 到 `10` 的迭代器，使用复杂方程映射。提供的映射闭包包括一个函数定义和一个变量绑定，在 `intro_functions.rs` 中：

```rs
(0..10).map(|x| {
    fn f(y: u32) -> u32 {
        y*y
    }
    let z = f(x+1) * f(x+2);
    z*z
}
```

可以定义接受闭包作为参数的函数或方法。为了将闭包用作可调用的函数，必须指定 `Fn`、`FnMut` 或 `FnOnce` 的界限。

这里是一个接受函数 `g` 和参数 `x` 的 HoF 定义。该定义将 `g` 和 `x` 限制为处理 `u32` 类型，并定义了一些涉及调用 `g` 的数学运算。还提供了一个 `f` HoF 的调用示例，如下所示，使用 `intro_functions.rs` 中的简单内联闭包定义：

```rs
fn f<T>(g: T, x: u32) -> u32
where T: Fn(u32) -> u32
{
    g(x+1) * g(x+2)
}

fn main()
{
   f(|x|{ x*x }, 2);
}
```

标准库的许多部分，尤其是迭代器，鼓励大量使用函数作为参数。

这是一个从 `0` 到 `10` 的迭代器，后面跟着许多链式迭代器组合器。`map` 函数从原始值返回一个新值。`inspect` 查看一个值，不改变它，但允许副作用。`filter` 跳过所有不满足谓词的值。`filter_map` 使用单个函数进行过滤和映射。`fold` 从一个初始值开始，向左向右工作，将所有结果减少到一个单一值。以下是在 `intro_functions.rs` 中的表达式：

```rs
(0..10).map(|x| x*x)
       .inspect(|x|{ println!("value {}", *x) })
       .filter(|x| *x<3)
       .filter_map(|x| Some(x))
       .fold(0, |x,y| x+y);

```

# 迭代器

迭代器是面向对象语言的一个常见特性，Rust 良好地支持这个概念。Rust 迭代器也是以函数式编程为设计理念的，允许程序员编写更易读的代码。这里强调的具体概念是 **可组合性**。当迭代器可以被操作、转换和组合时，`for` 循环的混乱可以被单个函数调用所取代。这些例子可以在 `intro_iterators.rs` 文件中找到。这如下表所示：

| **带有描述的功能名称** | **示例** |
| --- | --- |
| 连接两个迭代器：`first...second` | `(0..10).chain(10..20);` |
| `zip` 函数将两个迭代器组合成元组对，迭代到最短迭代器的末尾：(a1,b1), (a2, b2), ... | `(0..10).zip(10..20);` |
| `enumerate` 函数是 `zip` 的一个特例，它创建带编号的元组 (0, a1),(1,a2), … | `(0..10).enumerate();` |
| `inspect` 函数在迭代过程中将一个函数应用于迭代器中的所有值 | `(0..10).inspect( | x | { println!("value {}", *x) });` |
| `map` 函数将一个函数应用于每个元素，返回结果 | `(0..10).map( | x | x*x);` |
| `filter` 函数限制元素为满足谓词的元素 | `(0..10).filter( | x | *x<3);` |
| `fold` 函数将所有值累积到一个单一的结果中 | `(0..10).fold(0, | x,y | x+y);` |

| 当你想应用迭代器时，你可以使用 `for` 循环或调用 `collect` | `for i in (0..10) {}`

`(0..10).collect::<Vec<u64>>();` |

# 紧凑易读的表达式

在函数式语言中，所有项都是表达式。函数体中没有语句，只有一个单独的表达式。所有控制流运算符随后都表示为具有返回值的表达式。在 Rust 中，这几乎就是情况；唯一不是表达式的是 `let` 语句和项目声明。

这两个语句都可以包裹在块中，以创建一个表达式以及任何其他项。以下是一个例子，在 `intro_expressions.rs` 中：

```rs
let x = {
    fn f(x: u32) -> u32 {
        x * x
    }
    let y = f(5);
    y * 3
};
```

这种嵌套格式在野外不常见，但它说明了 Rust 语法宽容的本质。

回到函数式风格表达式的概念，重点始终应该放在编写易于阅读的代码上，无需太多麻烦或冗余。当其他人，或者你自己在以后的时间，来阅读你的代码时，它应该立即就能理解。理想情况下，代码应该能够自我说明。如果你发现自己不断地在代码和注释中重复写相同的代码，那么你应该重新考虑你的编程实践是否真正有效。

要开始一些函数式表达式的例子，让我们看看大多数语言中存在的一个表达式，三元条件运算符。在一个普通的`if`语句中，条件必须占据它自己的行，因此不能用作子表达式。

以下是一个传统的`if`语句，在`intro_expressions.rs`中初始化一个变量：

```rs
let x;
if true {
    x = 1;
} else {
    x = 2;
}
```

使用三元运算符，这个赋值可以移动到一行，如下所示在`intro_expressions.rs`中：

```rs
let x = if true { 1 } else { 2 };
```

几乎 Rust 中的每个从面向对象编程（OOP）来的语句也是一个表达式——`if`、`for`、`while`等等。在 Rust 中可以看到的一个更独特的表达式，在面向对象语言中不常见的是直接构造表达式。所有 Rust 类型都可以通过单个表达式实例化。构造器只在特定情况下是必要的，例如，当内部字段需要复杂的初始化时。以下是一个简单的`struct`和`intro_expressions.rs`中的等效元组：

```rs
struct MyStruct
{
    a: u32,
    b: f32,
    c: String
}

fn main()
{
    MyStruct {
        a: 1,
        b: 1.0,
        c: "".to_string()
    };

    (1, 1.0, "".to_string());
}
```

函数式语言中的另一个独特表达式是模式匹配。模式匹配可以被认为是一个更强大的`switch`语句版本。任何表达式都可以发送到一个模式表达式中，并在执行`分支`表达式之前解构以将内部信息绑定到局部变量中。模式表达式非常适合与枚举一起工作。这两者是一对完美的搭档。

以下代码片段定义了一个`Term`，作为一个标记联合的表达式选项。在主函数中，构建了一个`Term` `t`，然后与一个模式表达式进行匹配。注意`intro_expressions.rs`中标记联合的定义与模式表达式内部的匹配的语法相似性：

```rs
enum Term
{
    TermVal { value: String },
    TermVar { symbol: String },
    TermApp { f: Box<Term>, x: Box<Term> },
    TermAbs { arg: String, body: Box<Term> }
}

fn main()
{
    let mut t = Term::TermVar {
        symbol: "".to_string()
    };
    match t {
        Term::TermVal { value: v1 } => v1,
        Term::TermVar { symbol: v1 } => v1,
        Term::TermApp { f: ref v1, x: ref v2 } =>
            "TermApp(?,?)".to_string(),
        Term::TermAbs { arg: ref mut v1, body: ref mut v2 } =>  
            "TermAbs(?,?)".to_string()
    };
}
```

# 严格的抽象意味着安全的抽象

拥有更严格的类型系统并不意味着代码会有更多的要求或更复杂。与其说是严格的类型，不如考虑使用“表达性类型”这个术语。表达性类型为编译器提供了更多信息。这些额外的信息允许编译器在编程时提供额外的帮助。这些额外的信息还允许一个非常丰富的元编程系统。所有这些都是在更安全、更健壮的代码的明显好处之上。

# 作用域数据绑定

Rust 中的变量比大多数其他语言处理得更为严格。全局变量几乎完全不允许。局部变量受到密切监控，以确保在超出作用域之前，分配的数据结构被正确地解构，但不是更早。这种跟踪变量适当作用域的概念被称为**所有权**和**生命周期**。

在一个简单的例子中，分配内存的数据结构在超出作用域时会自动解构。在`intro_binding.rs`文件中不需要手动内存管理：

```rs
fn scoped() {
    vec![1, 2, 3];
}
```

在一个稍微复杂一点的例子中，分配的数据结构可以作为返回值传递，或者被引用，等等。这些简单的作用域的例外也必须在`intro_binding.rs`文件中考虑到：

```rs
fn scoped2() -> Vec<u32>
{
    vec![1, 2, 3]
}
```

这种使用跟踪可能会变得复杂（且不可决），因此 Rust 有一些规则来限制变量何时可以逃离上下文。我们称之为**复杂规则所有权**。它可以以下面的代码来解释，在`intro_binding.rs`文件中：

```rs
fn scoped3()
{
    let v1 = vec![1, 2, 3];
    let v2 = v1;
    //it is now illegal to reference v1
    //ownership has been transferred to v2
}
```

当无法或不需要转移所有权时，`clone`特质鼓励在`intro_binding.rs`文件中创建被引用数据的副本：

```rs
fn scoped4()
{
    vec![1, 2, 3].clone();
    "".to_string().clone();
}
```

克隆或复制并不是一个完美的解决方案，并且会带来性能开销。为了使 Rust 更快，它已经相当快了，我们还有借用这个概念。借用是一种机制，它承诺在某个特定点将所有权返回，以直接引用某些数据。引用由一个和号表示。考虑以下示例，在`intro_binding.rs`文件中：

```rs
fn scoped5()
{
   fn foo(v1: &Vec<u32>)
   {
       for v in v1
       {
           println!("{}", v);
       }
   }

   let v1 = vec![1, 2, 3];
   foo(&v1);

   //v1 is still valid
   //ownership has been returned
   v1;
}
```

严格的拥有权的另一个好处是安全的并发。每个绑定都由一个特定的线程拥有，并且可以使用`move`关键字将这种所有权转移到新的线程。这已经在以下代码中解释过，在`intro_binding.rs`文件中：

```rs
use std::thread;

fn thread1()
{
   let v = vec![1, 2, 3];

   let handle = thread::spawn(move || {
      println!("Here's a vector: {:?}", v);
   });

   handle.join().ok();
}

```

为了在线程之间共享信息，程序员有两个主要的选择。

首先，程序员可以使用传统的锁和原子引用的组合。这已经在以下代码中解释过，在`intro_binding.rs`文件中：

```rs
use std::sync::{Mutex, Arc};
use std::thread;

fn thread2()
{

   let counter = Arc::new(Mutex::new(0));
   let mut handles = vec![];

   for _ in 0..10 {
      let counter = Arc::clone(&counter);
      let handle = thread::spawn(move || {
         let mut num = counter.lock().unwrap();
         *num += 1;
      });
      handles.push(handle);
   }

   for handle in handles {
      handle.join().unwrap();
   }

   println!("Result: {}", *counter.lock().unwrap());
}
```

其次，通道提供了一个很好的机制，用于线程之间的消息传递和作业排队。`send`特质也自动应用于大多数对象。考虑以下代码，在`intro_binding.rs`文件中：

```rs
use std::thread;
use std::sync::mpsc::channel;

fn thread3() {

    let (sender, receiver) = channel();
    let handle = thread::spawn(move ||{

        //do work
        let v = vec![1, 2, 3];
        sender.send(v).unwrap();

    });

    handle.join().ok();
    receiver.recv().unwrap();
}
```

所有这些并发都是类型安全的，并且由编译器强制执行。你可以尽可能多地使用线程，如果你不小心尝试创建竞态条件或简单的死锁，编译器会阻止你。我们称之为**无畏并发**。

# 代数数据类型

除了结构/对象和函数/方法之外，Rust 的函数式编程还包括对可定义类型和结构的丰富扩展。元组提供了定义简单匿名结构的简写。枚举提供了一种类型安全的联合复杂数据结构的方法，并增加了构造标签以帮助模式匹配的额外好处。标准库对泛型编程有广泛的支持，从基本类型到集合。甚至对象系统特性也是面向对象（OOP）概念中的类和函数式编程（FP）概念中的类型类的混合。函数式风格无处不在，即使你不在 Rust 中寻找，你也可能会不知不觉地使用这些功能。

`type`别名可以帮助创建复杂类型的简称。或者，可以使用`newtype`结构模式来创建具有不同非等效类型的别名。以下是一个例子，在`intro_datatypes.rs`文件中：

```rs
//alias
type Name = String;

//newtype
struct NewName(String);
```

一个`struct`，即使参数化，当仅用于将多个值存储到单个对象中时，也可能变得重复。这可以在`intro_datatypes.rs`文件中看到：

```rs
struct Data1
{
    a: i32,
    b: f64,
    c: String
}

struct Data2
{
    a: u32,
    b: String,
    c: f64
}
```

元组有助于消除冗余的结构定义。使用元组不需要先前的类型定义。以下是一个例子，在`intro_datatypes.rs`文件中：

```rs
//alias to tuples
type Tuple1 = (i32, f64, String);
type Tuple2 = (u32, String, f64);

//named tuples
struct New1(i32, f64, String);
struct New2(u32, String, f64);
```

可以通过实现正确的特性为任何类型实现标准运算符。以下是一个例子，在`intro_datatypes.rs`文件中：

```rs
use std::ops::Mul;

struct Point
{
    x: i32,
    y: i32
}

impl Mul for Point
{
    type Output = Point;
    fn mul(self, other: Point) -> Point
    {
        Point
        {
            x: self.x * other.x,
            y: self.y * other.y
        }
    }
}
```

标准库集合和许多其他内置类型都是泛型的，例如`intro_datatypes.rs`中的`HashMap`：

```rs
use std::collections::HashMap;

type CustomHashMap = HashMap<i32,u32>;
```

枚举是多个类型的类型安全联合。请注意，递归的`enum`定义必须将内部值包裹在容器中，如`Box`，否则大小将是无限的。以下是如何表示的，在`intro_datatypes.rs`文件中：

```rs
enum BTree<T>
{
    Branch { val:T, left:Box<BTree<T>>, right:Box<BTree<T>> },
    Leaf { val: T }
}
```

标签联合也用于更复杂的数据结构。以下是一个例子，在`intro_datatypes.rs`文件中：

```rs
enum Term
{
    TermVal { value: String },
    TermVar { symbol: String },
    TermApp { f: Box<Term>, x: Box<Term> },
    TermAbs { arg: String, body: Box<Term> }
}
```

特性有点像面向对象中的类（OOP），以下是一个代码示例，在`intro_datatypes.rs`文件中：

```rs
trait Data1Trait
{
    //constructors
    fn new(a: i32, b: f64, c: String) -> Self;

    //methods
    fn get_a(&self) -> i32;
    fn get_b(&self) -> f64;
    fn get_c(&self) -> String;
}
```

特性也像类型类（FP），以下是一个代码片段，在`intro_datatypes.rs`文件中：

```rs
trait BehaviorOfShow
{
    fn show(&self) -> String;
}
```

# 混合面向对象编程和函数式编程

如前所述，Rust 支持面向对象和函数式编程风格的许多方面。数据类型和函数对任何范式都是中立的。特性和特质专门支持这两种风格的混合。

首先，在面向对象风格中，使用`struct`、`trait`和`impl`定义一个简单的类和构造函数以及一些方法可以完成。这通过以下代码片段进行解释，在`intro_mixoopfp.rs`文件中：

```rs
struct MyObject
{
    a: u32,
    b: f32,
    c: String
}

trait MyObjectTrait
{
    fn new(a: u32, b: f32, c: String) -> Self;
    fn get_a(&self) -> u32;
    fn get_b(&self) -> f32;
    fn get_c(&self) -> String;
}

impl MyObjectTrait for MyObject
{
    fn new(a: u32, b: f32, c: String) -> Self
    {
        MyObject { a:a, b:b, c:c }
    }

    fn get_a(&self) -> u32
    {
        self.a
    }

    fn get_b(&self) -> f32
    {
        self.b
    }

    fn get_c(&self) -> String
    {
        self.c.clone()
    }
}
```

在对象上添加对函数式编程的支持就像定义特性和使用函数式语言特性的方法一样简单。例如，接受一个闭包可以成为当适当使用时的一种很好的抽象。以下是一个例子，在`intro_mixoopfp.rs`文件中：

```rs
trait MyObjectApply
{
    fn apply<F,R>(&self, f:F) -> R
    where F: Fn(u32,f32,String) -> R;
}

impl MyObjectApply for MyObject
{
    fn apply<F,R>(&self, f:F) -> R
    where F: Fn(u32,f32,String) -> R
    {
        f(self.a, self.b, self.c.clone())
    }
}
```

# 改进项目架构

函数式程序鼓励良好的项目架构和原则性设计模式。使用函数式编程的构建块通常可以减少需要做出的设计选择，使得好的选项变得明显。

“应该只有一个 - 最好是唯一一个 - 明显的方法来做这件事。”

– *PEP 20*

# 文件层次结构、模块和命名空间设计

Rust 程序主要以两种方式编译。第一种是使用 `rustc` 编译单个文件。第二种是使用 `cargo` 描述整个包以进行编译。我们假设这里的项目是使用 `cargo` 构建的，如下所示：

1.  要开始一个包，你首先在一个目录中创建一个 `Cargo.toml` 文件。从现在起，这个目录将是你的包目录。这是一个配置文件，它将告诉编译器应该将哪些代码、资源和额外信息包含到包中：

```rs
[package]
name = "fp_rust"
version = "0.0.1"

```

1.  在完成基本配置之后，你现在可以使用 `cargo build` 来编译整个项目。你决定将代码文件放在哪里，以及如何命名它们，取决于你如何在模块命名空间中引用它们。每个文件都会被赋予自己的模块 `mod`。你还可以在文件内部嵌套模块：

```rs
mod inner_module
{
    fn f1()
    {
        println!("inner module function");
    }
}

```

1.  在这些步骤之后，项目可以作为 cargo 依赖项添加，模块内部可以使用命名空间来公开符号。考虑以下代码片段：

```rs
extern crate package;
use package::inner_module::f1;
```

这些是 Rust 模块的基本构建块，但这与函数式编程有什么关系呢？

以函数式风格设计项目是一个过程，并且适合某些常规。通常，项目架构师会首先设计核心数据结构，在复杂情况下也会设计物理结构（代码/服务将在此处运行）。一旦数据布局被详细概述，就可以规划核心函数/常规（例如程序的行为）。到这一点，如果在架构阶段进行编码，可能会有一些代码尚未实现。最终阶段涉及用正确的行为替换这个模拟代码。

按照这个分阶段的发展过程，我们还可以看到典型的文件布局正在形成。在实际程序中，通常将这些阶段从上到下书写。尽管作者们不太可能在这些明确的阶段中进行规划，但由于简单起见，这仍然是一个常见的模式。考虑以下示例：

```rs
//trait definitions

//data structure and trait implementations

//functions

//main
```

将定义分组如下可能有助于标准化文件布局并提高可读性。在长文件中来回搜索符号定义是编程中常见但令人不快的一部分。这同样也是一个可以预防的问题。

# 函数式设计模式

除了文件布局之外，还有许多功能设计模式有助于减少代码重量和冗余。当正确使用时，这些原则可以帮助阐明设计决策，并使架构更加健壮。大多数设计模式都是单一责任原则的变体。这可以有多种形式，具体取决于上下文，但意图是相同的；编写做一件事做得好的代码，然后根据需要重用这段代码。我已如下解释：

+   **纯函数**：这些是没有副作用或逻辑依赖（除了函数参数之外）的函数。副作用是指影响函数之外任何内容的状态变化，除了返回值。纯函数很有用，因为它们可以被随意抛来抛去，组合，并且通常可以不加顾虑地使用，而不会产生意外的影响。

纯函数可能出现的最糟糕的事情是返回值不好，或者在极端情况下，栈溢出。

使用纯函数，即使使用不当，也难以引发错误。考虑以下纯函数的示例，在`intro_patterns.rs`中：

```rs
fn pure_function1(x: u32) -> u32
{
    x * x
}

fn impure_function(x: u32) -> u32
{
    println!("x = {}", x);
    x * x
}
```

+   **不可变性**：不可变性是一种有助于鼓励纯函数的模式。Rust 变量绑定默认是不可变的。这是 Rust 鼓励你避免可变状态的一种不太微妙的方式。不要这样做。如果你绝对必须，可以使用`mut`关键字标记变量以允许重新赋值。这可以通过以下示例在`intro_patterns.rs`中展示：

```rs
let immutable_v1 = 1;
//immutable_v1 = 2; //invalid

let mut mutable_v2 = 1;
mutable_v2 = 2;
```

+   **函数组合**：函数组合是一种模式，其中一个函数的输出连接到另一个函数的输入。以这种方式，函数可以串联起来，通过简单的步骤创建复杂的效果。这可以通过以下代码片段在`intro_patterns.rs`中展示：

```rs
let fsin = |x: f64| x.sin();
let fabs = |x: f64| x.abs();

//feed output of one into the other
let transform = |x: f64| fabs(fsin(x));
```

+   **高阶函数**：这些之前已经提到过，但我们还没有使用这个术语。高阶函数是接受一个函数作为参数的函数。许多迭代器方法都是高阶函数。考虑以下示例，在`intro_patterns.rs`中：

```rs
fn filter<P>(self, predicate: P) -> Filter<Self, P>
where P: FnMut(&Self::Item) -> bool
{ ... }
```

+   **函子**：如果你能越过这个名字，这些是一个简单而有效的设计模式。它们也非常灵活。这个概念在整体上可能难以捕捉，但你可能将函子视为*函数的逆*。一个函数定义了一个转换，接受数据，并返回转换的结果。函子定义数据，接受一个函数，并返回转换的结果。函子的一个常见例子是绑定在容器上的`map`方法，例如在`Vec`上。以下是一个示例，在`intro_patterns.rs`中：

```rs
let mut c = 0;
for _ in vec!['a', 'b', 'c'].into_iter()
   .map(|letter| {
      c += 1; (letter, c)
   }){};
```

“单子是端内函子的范畴中的幺半群，问题是什么？”

– *菲利普·瓦德勒*

+   **Monads**：Monads 是学习 FP 的人常见的绊脚石。Monads 和 functors 可能是你深入理论数学之旅中可能遇到的第一个词。我们不会深入那个领域。对我们来说，monads 只是一个有两个方法的`trait`。以下代码展示了这一点，位于`intro_patterns.rs`文件中：

```rs
trait Monad<A> {
    fn return_(t: A) -> Self;
    //:: A -> Monad<A>

    fn bind<MB,B>(m: Self, f: Fn(A) -> MB) -> MB
    where MB: Monad<B>;
    //:: Monad<A> -> (A -> Monad<B>) -> Monad<B>
}
```

如果这还不能帮助你澄清问题（很可能不能），monad 有两个方法。第一个方法是构造函数。第二个方法允许你绑定一个操作以创建另一个 monad。许多常见的特性都有隐藏的半 monad，但通过使概念明确化，这个概念就变成了一个强大的设计模式，而不是一个混乱的反模式。不要试图重新发明你不需要的东西。

+   **函数 currying**：对于来自面向对象或命令式语言背景的人来说，函数 currying 可能是一个陌生的技术。这种混淆的原因是，在许多函数式语言中，函数默认是 curry 的，而其他语言则不是这样。Rust 函数默认不是 curry 的。

curry 函数和非 curry 函数的区别在于 curry 函数是逐个传入参数，而非 curry 函数则是一次性传入所有参数。观察一个普通的 Rust 函数定义，我们可以看到它并不是 curry 函数。考虑以下代码，位于`intro_patterns.rs`文件中：

```rs
fn not_curried(p1: u32, p2: u32) -> u32
{
    p1 + p2
}

fn main()
{
   //and calling it
   not_curried(1, 2);
}
```

一个`curried`函数逐个接受参数，如下所示，位于`intro_patterns.rs`文件中：

```rs
fn curried(p1: u32) -> Box<Fn(u32) -> u32>
{
    Box::new(move |p2: u32| {
        p1 + p2
    })
}

fn main()
{
   //and calling it
   curried(1)(2);
}
```

Curry 函数可以用作函数工厂。前几个参数配置了最终函数应该如何表现。结果是允许简短配置复杂操作符的模式。Currying 通过将单个函数转换为多个组件来补充所有其他设计模式。

+   **惰性评估**：惰性评估在其他语言中在技术上也是可能的。然而，由于语言障碍，它很少在 FP 之外看到。正常表达式和惰性表达式的区别在于惰性表达式只有在被访问时才会被评估。以下是一个简单的惰性实现，位于`intro_patterns.rs`文件中的函数调用之后：

```rs
let x = { println!("side effect"); 1 + 2 };

let y = ||{ println!("side effect"); 1 + 2 };
```

第二个表达式只有在函数被调用时才会被评估，此时代码才会解析。对于惰性表达式，副作用发生在解析时而不是初始化时。这是一个懒惰实现的糟糕例子，因此我们将在后面的章节中进一步详细说明。这种模式相当常见，一些操作符和数据结构需要惰性才能工作。一个必要的惰性例子是可能无法以其他方式创建的惰性列表。内置的 Rust 数值迭代器（惰性列表）很好地使用了这一点：（0..）。

缓存（Memoization）是我们在这里将要介绍的最后一个模式。它可能更多地被视为一种优化而不是设计模式，但由于其普遍性，我们在这里应该提到它。一个缓存的函数只计算一次唯一的结果。一个简单的实现是使用哈希表来保护函数。如果参数和结果已经在哈希表中，则跳过函数调用并直接从哈希表中返回结果。否则，计算结果，将其放入哈希表，并返回。这个过程可以在任何语言中手动实现，但 Rust 宏允许我们一次性编写缓存代码，并通过应用此宏来重用该代码。这可以通过以下代码片段展示，位于 `intro_patterns.rs` 文件中：

```rs
#[macro_use] extern crate cached;
#[macro_use] extern crate lazy_static;

cached! {
    FIB;
    fn fib(n: u64) -> u64 = {
        if n==0 || n==1 { return n }
        fib(n-1) + fib(n-2)
    }
}

fn main()
{
   fib(30);
}
```

此示例使用了两个 crate 和许多宏。我们不会在本书的最后一部分完全解释这里发生的一切。宏和元编程有很多可能性。缓存函数结果是起点。

# 元编程

在 Rust 中，元编程这个术语经常与宏（macros）这个术语重叠。Rust 中有两种主要的宏类型可用：

+   递归

+   过程式

这两种类型的宏都接受一个**抽象语法树**（**AST**）作为输入，并生成一个或多个 AST。

一个常用的宏是 `println`。通过宏将可变数量的参数和类型与格式字符串连接起来，以产生格式化的输出。要调用这样的递归宏，就像调用函数一样调用宏，只是在参数前加上一个 `!`。宏应用可以由 `[]` 或 `{}` 包围：

```rs
vec!["this is a macro", 1, 2];
```

递归宏通过 `macro_rules!` 语句定义。`macro_rules` 定义内部的格式与模式匹配表达式非常相似。唯一的区别是 `macro_rules!` 匹配语法而不是数据。我们可以使用此格式来定义 `vec` 宏的简化版本。以下是在 `intro_metaprogramming.rs` 文件中的代码片段展示：

```rs
macro_rules! my_vec_macro
{
    ( $( $x:expr ),* ) =>
    {
        {
            let mut temp_vec = Vec::new();
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    }
}
```

此定义仅接受并匹配一个模式。它期望一个逗号分隔的表达式列表。语法模式 `( $( $x: expr ),* )` 匹配逗号分隔的表达式列表，并将结果存储在复数变量 `$x` 中。在表达式的主体中，有一个单独的块。该块定义了一个新的 `vec`，然后通过迭代 `$x*` 将每个 `$x` 推入 `vec`，最后，该块将其结果作为返回值。以下是在 `intro_metaprogramming.rs` 文件中的宏及其展开：

```rs
//this
my_vec_macro!(1, 2, 3);

//is the same as this
{
    let mut temp_vec = Vec::new();
    temp_vec.push(1);
    temp_vec.push(2);
    temp_vec.push(3);
    temp_vec
}
```

重要的一点是，表达式作为代码移动，而不是作为值移动，因此副作用将移动到评估上下文，而不是定义上下文。

递归宏模式匹配标记字符串。根据匹配的标记执行不同的分支是可能的。一个简单的匹配案例如下，位于 `intro_metaprogramming.rs` 文件中：

```rs
macro_rules! my_macro_branch
{
    (1 $e:expr) => (println!("mode 1: {}", $e));
    (2 $e:expr) => (println!("mode 2: {}", $e));
}

fn main()
{
    my_macro_branch!(1 "abc");
    my_macro_branch!(2 "def");
}

```

递归宏的名字来源于宏中的递归，所以当然我们可以调用我们正在定义的宏。递归宏可以是一种快速定义领域特定语言的途径。考虑以下代码片段，在`intro_metaprogramming.rs`中：

```rs
enum DSLTerm {
    TVar { symbol: String },
    TAbs { param: String, body: Box<DSLTerm> },
    TApp { f: Box<DSLTerm>, x: Box<DSLTerm> }
}

macro_rules! dsl
{
    ( ( $($e:tt)* ) ) => (dsl!( $($e)* ));
    ( $e:ident ) => (DSLTerm::TVar {
        symbol: stringify!($e).to_string()
    });
    ( fn $p:ident . $b:tt ) => (DSLTerm::TAbs {
        param: stringify!($p).to_string(),
        body: Box::new(dsl!($b))
    });
    ( $f:tt $x:tt ) => (DSLTerm::TApp {
        f: Box::new(dsl!($f)),
        x: Box::new(dsl!($x))
    });
} 
```

宏定义的第二种形式是过程宏。递归宏可以被认为是一种好的语法，有助于定义过程宏。另一方面，过程宏是最通用的形式。你可以用过程宏做很多递归形式不可能做到的事情。

在这里，我们可以获取`struct`的`TypeName`，并使用它来自动生成特质实现。以下是宏定义，在`intro_metaprogramming.rs`中：

```rs
#![crate_type = "proc-macro"]
extern crate proc_macro;
extern crate syn;
#[macro_use]
extern crate quote;
use proc_macro::TokenStream;
#[proc_macro_derive(TypeName)]

pub fn type_name(input: TokenStream) -> TokenStream
{
    // Parse token stream into input AST
    let ast = syn::parse(input).unwrap();
    // Generate output AST
    impl_typename(&ast).into()
}

fn impl_typename(ast: &syn::DeriveInput) -> quote::Tokens
{
    let name = &ast.ident;
    quote!
    {
        impl TypeName for #name
        {
            fn typename() -> String
            {
                stringify!(#name).to_string()
            }
        }
    }
}
```

相应的宏调用在`intro_metaprogramming.rs`中看起来如下：

```rs
#[macro_use]
extern crate metaderive;

pub trait TypeName
{
    fn typename() -> String;
}

#[derive(TypeName)]
struct MyStructA
{
    a: u32,
    b: f32
}
```

如你所见，过程宏的设置稍微复杂一些。然而，好处是所有处理都是直接用正常的 Rust 代码完成的。这些宏允许在编译前以非结构化格式使用任何语法信息来生成更多的代码结构。

过程宏被处理为独立的模块，在正常的编译器执行期间预编译和执行。提供给每个宏的信息是局部化的，所以

整个程序的考虑是不可能的。然而，可用的本地信息足以实现一些相当复杂的效果。

# 摘要

在本章中，我们简要概述了本书中将要出现的主要概念。从代码示例中，你现在应该能够直观地识别函数式风格。我们还提到了一些为什么这些概念有用的原因。在接下来的章节中，我们将提供完整的环境，说明何时以及为什么每种技术是合适的。在那个环境中，我们还将提供掌握这些技术并开始使用函数式实践所需的知识。

从本章中，我们学会了尽可能地进行参数化，并且知道函数可以用作参数，通过组合简单行为来定义复杂行为，并且在 Rust 中只要编译通过，就可以随意使用线程。

本书的结构是先介绍简单的概念，然后随着书的继续，一些概念可能会变得更加抽象或技术化。此外，所有技术都将在一个持续的项目环境中介绍。该项目将控制电梯系统，随着书的进展，需求将逐渐变得更加严格。

# 问题

1.  函数是什么？

1.  函子是什么？

1.  什么是元组？

1.  为标记联合设计了哪种控制流表达式？

1.  函数作为参数的函数叫什么名字？

1.  在记忆化的`fib(20)`中，`fib`会被调用多少次？

1.  可以通过通道发送哪些数据类型？

1.  为什么函数在从函数返回时需要装箱？

1.  `move`关键字的作用是什么？

1.  两个变量怎么可能共享一个单一变量的所有权？

# 进一步阅读

Packt 为学习 Rust 提供了许多其他优秀资源：

+   [`www.packtpub.com/application-development/rust-programming-example`](https://www.packtpub.com/application-development/rust-programming-example)

+   [`www.packtpub.com/application-development/learning-rust`](https://www.packtpub.com/application-development/learning-rust)

对于基本文档和教程，请参考此处：

+   教程：[`doc.rust-lang.org/book/first-edition/`](https://doc.rust-lang.org/book/first-edition/)

+   文档：[`doc.rust-lang.org/stable/reference/`](https://doc.rust-lang.org/stable/reference/)
