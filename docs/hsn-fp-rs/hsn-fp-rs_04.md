# 泛型和多态性

参数化，也称为**泛型**或**多态性**，是继控制流和数据结构之后的第三大重要语言特性。参数化解决了早期语言中的复制粘贴问题。此功能允许遵循“不要重复自己”的良好程序设计原则。

在本章中，我们将探讨参数化如何帮助我们设计能够随变化而演化的稳健程序，而不是与变化作斗争。不会引入新的项目需求。本章将完全反思，查看项目当前的结构，如何改进，以及参数化如何具体帮助。

本章的学习成果如下：

+   理解泛化代数数据类型

+   理解参数化多态性

+   理解参数化生命周期

+   理解参数化特性

+   理解模糊方法解析

# 技术要求

运行提供的示例需要 Rust 的最近版本：

[Rust 安装指南](https://www.rust-lang.org/en-US/install.html)

本章的代码也可在 GitHub 上找到：

[《RUST 函数式编程实战》](https://github.com/PacktPublishing/Hands-On-Functional-Programming-in-RUST)

每章的`README.md`文件中也包含了具体的安装和构建说明。

# 在空闲时间保持生产力

在客户就谈判和可能接受你的项目提案做出最终决定之前，将有一段时间。在这段时间里，你的管理层鼓励你利用这段时间回顾你的工作，并为将电梯控制器集成到真实电梯中做好准备。

你对直接电梯控制接口了解不多，客户特别提到可能有多个分包商设计不同的电梯。在这个阶段做出假设可能会导致浪费精力，因此，你决定重新考虑你的代码，寻找消除任何假设的机会。

特性接口的参数化和使用有助于实现这一抽象目标。在这段时间内，你决定让团队学习参数化，并考虑如何将其应用于改进本项目或后续项目。

# 了解泛型

泛型是一种编写适用于不同类型上下文的代码的设施，参数化允许程序员编写对涉及代码定义的数据结构和代码段做出较少假设的代码。例如，一个非常模糊的概念将是加法概念。当程序员编写`a + b`时，这意味着什么？在 Rust 中，可以为几乎任何类型实现`Add`特质。只要存在与`a`和`b`的类型兼容的`Add`特质的实现，则此特质将定义操作。在这种模式中，我们可以编写以最抽象的术语定义概念的泛型代码，允许稍后定义的数据和方法与该代码接口而无需更改。

完全泛型代码的一个主要例子是内置的容器数据结构。向量和 HashMap 必须必然知道它们存储的对象的类型。然而，如果对底层数据结构或存储项的方法有任何假设，这将非常限制性。因此，容器的参数化允许容器及其方法显式声明期望从存储类型中获得的特征界限。存储项的所有其他特征都将被参数化。

# 调查泛型

泛型是指参数化面向对象编程语言中的类。Rust 没有类的确切等效物。然而，如果从那种意义上使用，数据类型与特质的组合概念与类非常相似。因此，在 Rust 中，泛型将指数据类型和特质的参数化。

以面向对象编程（OOP）的常见示例，让我们看看动物王国。在以下代码中，我们将定义一些动物和它们可以执行的动作。首先，让我们定义两种动物：

```rs
struct Cat
{
   weight: f64,
   speed: f64
}

struct Dog
{
   weight: f64,
   speed: f64
}
```

现在，让我们定义一个`动物特征`及其实现。所有动物都将具有`max_speed`方法。以下是代码：

```rs
trait Animal
{
   fn max_speed(&self) -> f64;
}

impl Animal for Cat
{
   fn max_speed(&self) -> f64
   {
      self.speed
   }
}

impl Animal for Dog
{
   fn max_speed(&self) -> f64
   {
      self.speed
   }
}
```

在这里，我们定义了 Rust 中面向对象编程（OOP）接口的等效物。然而，我们没有对任何东西进行参数化，因此这里不应该有任何泛型内容。我们将添加以下代码，一个定义动物追逐玩具概念的特质。首先，我们将定义玩具的概念。这将遵循与前面代码相同的面向对象模式：

```rs
struct SqueakyToy
{
   weight: f64
}

struct Stick
{
   weight: f64
}

trait Toy
{
   fn weight(&self) -> f64;
}

impl Toy for SqueakyToy
{
   fn weight(&self) -> f64
   {
      self.weight
   }
}

impl Toy for Stick
{
   fn weight(&self) -> f64
   {
      self.weight
   }
}
```

现在，我们有两个特质，每个特质都有两种可能的实现。让我们定义一个动物追逐玩具的动作。已经定义了多个可能的动物和多个可能的玩具，因此我们需要使用泛型定义。结构定义还通过特质界限约束每个参数，这为`struct`添加了额外的信息；现在，我们可以保证每个动物都将实现`Animal`特质，同样，每个玩具也将实现`Toy`。我们还将定义一些使用参数化特质方法的关联逻辑。以下是代码：

```rs
struct AnimalChasingToy<A: Animal, T: Toy>
{
   animal: A,
   toy: T
}

trait AnimalChasesToy<A: Animal, T: Toy>
{
   fn chase(&self);
}

impl<A: Animal, T: Toy> AnimalChasesToy<A, T> for AnimalChasingToy<A, T>
{
   fn chase(&self)
   {
      println!("chase")
   }
}
```

在这一点上，我们已经定义了一个通用的 `struct` 和 `trait`，它接受类型，只知道一些关于每个对象特质的有限信息。可以指定多个特质或没有任何特质来声明所有预期的接口。可以使用 `'l + Trait1 + Trait2` 语法来声明多个特质或生命周期界限。

# 探究参数多态

另一个参数化的常见应用是函数。出于我们想要参数化数据结构或特质的原因，我们也应该考虑函数的参数化。函数的参数化被称为**参数多态**。多态在希腊语中意为多种形式，有时在现代用法中，它也可以意味着多个箭头。这个词表明一个函数有多个实现或多个基类型签名。

对于一个参数化函数的简单示例，我们可以想象一个泛型乘以三的函数。以下是实现：

```rs
fn raise_by_three<T: Mul + Copy>(x: T) -> T
where T: std::ops::Mul<Output=T>
{
   x * x * x
}
```

在这里，`raise_by_three` 函数不知道 `Mul` 做什么。`Mul` 是一个特质和抽象行为，它还指定了一个关联类型，`Output`。在这里无法泛型地提升 `x.pow(3)`，因为 `x` 可能不是数值类型。至少，我们不知道 `x` 是浮点类型还是整型。因此，我们使用可用的 `Mul` 特质将 `x` 乘以三次。这看起来可能有些奇怪，但在上下文中这个概念会变得清晰。

首先，考虑应用于浮点型和整型。这种用法很简单，但似乎还没有什么用处。我们已经有了一个工作的 `raise by three` 表达式，只要我们知道并拥有原始的浮点型或整型。那么，我们为什么不直接使用内置的表达式呢？首先，让我们比较两种选项的代码：

```rs
raise_by_three(10);
(10 as u64).pow(3);

raise_by_three(3.0);
(3.0 as f64).powi(3);
```

第二种选择似乎更可取，而且确实是。然而，第二种选择也假设我们知道每个参数的 `u64` 或 `f64` 的完整类型。让我们看看如果我们擦除一些类型信息会发生什么：

```rs
#[derive(Copy,Clone)]
struct Raiseable<T: Mul + Copy>
{
   x: T
}

impl<T: Mul + Copy> std::ops::Mul for Raiseable<T>
where T: std::ops::Mul<Output=T>
{
   type Output = Raiseable<T>;
   fn mul(self, rhs: Self) -> Self::Output
   {
      Raiseable { x: self.x * rhs.x }
   }
}

let x = Raiseable { x: 10 as u64 };
raise_by_three(x);
//no method named pow
//x.pow(3);

```

```rs

let x = Raiseable { x: 3.0 as f64 };
raise_by_three(x);
//no method named powi
//x.powi(3); 
```

在我们失去对底层类型的访问后，我们很快就会在可以执行的操作方面受到限制。泛型编程在长期减少工作量方面很出色；然而，它也要求非常明确地声明和实现所有使用的接口。在这里，你可以看到我们必须将 `Copy` 声明为特质界限，这意味着能够将变量从一个内存位置复制到另一个位置。另一个低级特质是 `Sized`，它表示数据在编译时有一个已知的常量大小。

如果我们查看 `HashMap` 的声明，我们可以看到为什么这种抽象通常是必要的：

```rs
impl<K: Hash + Eq, V> HashMap<K, V, RandomState>
```

每个哈希键必须实现 `Hash` 和 `Eq`，这意味着它必须是可哈希的且可比较的。除此之外，没有其他特质被期望，因此整个数据结构保持非常通用。

正如函数可以被参数化一样，作为参数的函数也可以被参数化。函数作为参数有两种一般形式——闭包和函数指针。函数指针不允许携带状态。闭包可以携带状态，但大小是独立于其声明的类型的。函数指针可以自动提升为闭包：

```rs
fn foo<X>(x: X) -> X
{
   x
}

fn bar<X>(f: fn(X) -> X, x: X) -> X
{
   f(x)
}

foo(1);
bar(foo,1);
```

闭包也可以以类似的方式参数化。这种情况更为常见。如果你在考虑是否使用函数指针或闭包，请使用闭包。函数指针总是可以被提升为闭包。此外，这段代码引入了 `where` 语法；`where` 子句允许以更可读的形式声明特质界限。以下是代码：

```rs
fn baz<X,F>(f: F, x: X) -> X
where F: Fn(X) -> X
{
   f(x)
}

baz(|x| x, 1);
baz(foo, 1);
```

在这里，我们可以看到将函数指针包装成闭包是多么容易。闭包是一个很好的抽象，并且当正确使用时非常强大。

# 研究泛化代数数据类型

有时，我们希望类型系统携带比正常更多的信息。如果我们看看编译过程，类型位于程序代码和程序可执行文件之间。代码在编译前可以以文本文件的形式存在，或者像 Rust 宏那样操作的抽象语法树。程序可执行文件由所有 Rust 原始元素（如表达式、函数、数据类型、特质等）的组合结果组成。

正在中间，可以引入一个新概念，称为**代数数据类型**（**ADTs**）。技术上，ADTs 是 Rust 原始元素的扩展，尽管重要的是要注意 ADTs 使用了多少额外的类型信息。这种技术涉及将额外的类型信息保留到可执行文件中。额外的运行时决策是向动态类型迈进的一步，并放弃了静态编译可用的优化。结果是编程原语稍微低效，但也是一个可以描述其他情况下难以接近的概念的原语。

让我们看看一个例子——延迟计算。当我们描述不同值和表达式的关联时，我们通常只是直接将这段代码写入程序中。然而，如果我们想将代码步骤与执行步骤分开，我们会怎么做？为了实现这一点，我们开始构建一个称为**领域特定语言**的东西。

以一个具体的例子来说，假设你正在构建一个用于 JavaScript 的 JIT（动态编译）解释器。Mozilla 项目有几个项目是专门针对用 Rust 构建的 JS 引擎的（[`blog.mozilla.org/javascript/2017/10/20/holyjit-a-new-hope/`](https://blog.mozilla.org/javascript/2017/10/20/holyjit-a-new-hope/))。这是一个非常适合 Rust 的实际应用。要在 JIT 编译的解释器中使用 ADT，我们希望得到两件事：

+   在解释器中直接评估 ADT 表达式

+   如果选择了编译 ADT 表达式

因此，我们的 JavaScript 表达式中的任何部分都可以在任何时候被解释或编译。如果一个表达式被编译，那么我们希望所有后续的评估都使用编译版本。实现这一点的关键是给类型系统增加一些额外的权重。这些重型类型定义是 ADT 概念的精髓。以下是一个使用 ADT 定义 JavaScript 非常小子集的示例：

```rs
struct JSJIT(u64);

enum JSJITorExpr {
   Jit { label: Box<JSJIT> },
   Expr { expr: Box<JSExpr> }
}

enum JSExpr {
   Integer { value: u64 },
   String { value: String },
   OperatorAdd { lexpr: Box<JSJITorExpr>, rexpr: Box<JSJITorExpr> },
   OperatorMul { lexpr: Box<JSJITorExpr>, rexpr: Box<JSJITorExpr> }
}
```

在这里，我们可以看到每个中间表达式都有足够的信息来评估，同时也有足够的信息来编译。我们本可以将`Add`或`Mul`运算符包装在闭包中，但这将不允许 JIT 优化。我们需要在这里保持完整的表示，以便允许 JIT 编译。此外，请注意程序在决定评估表达式或调用编译代码之间的间接引用。

下一步是为每种表达式形式实施一个评估程序。我们可以将其分解为特性，或者定义一个更大的函数作为评估。为了保持函数式风格，我们将定义一个单一函数。为了评估一个表达式，我们将使用对`JSJITorExpr`表达式的模式匹配。这种即时编译表达式可以分解为通过调用`jump`函数运行的代码地址，或者必须动态评估的表达式。这种模式为我们提供了两者的最佳结合，将编译代码和解释代码混合在一起。代码如下：

```rs
fn jump(l: JSJIT) -> JSJITorExpr
{
   //jump to compiled code
   //this depends on implementation
   //so we will just leave this as a stub
   JSJITorExpr::Jit { label: JSJIT(0) }
}

fn eval(e: JSJITorExpr) -> JSJITorExpr
{
   match e
   {
      JSJITorExpr::Jit { label: label } => jump(label),
      JSJITorExpr::Expr { expr: expr } => {
         let rawexpr = *expr;
         match rawexpr
         {
            JSExpr::Integer {..} => JSJITorExpr::Expr { expr: Box::new(rawexpr) },
            JSExpr::String {..} => JSJITorExpr::Expr { expr: Box::new(rawexpr) },
            JSExpr::OperatorAdd { lexpr: l, rexpr: r } => {
               let l = eval(*l);
               let r = eval(*r);
               //call add op codes for possible l,r representations
               //should return wrapped value from above
               JSJITorExpr::Jit { label: JSJIT(0) }
            }
            JSExpr::OperatorMul { lexpr: l, rexpr: r } => {
               let l = eval(*l);
               let r = eval(*r);
               //call mul op codes for possible l,r representations
               //should return wrapped value from above
               JSJITorExpr::Jit { label: JSJIT(0) }
            }
         }
      }
   }
}
```

ADT 概念的另一个例子是异构列表。异构列表不像其他泛型容器，如向量。Rust 向量是同质的，意味着所有项目都必须具有相同的类型。相比之下，异构列表可以包含任何类型的元素混合。这听起来可能像元组，但元组具有固定的长度和平坦的类型签名。同样，异构列表必须具有在编译时已知长度和类型签名，但这种知识可以逐步实现。异构列表允许在部分了解列表类型的情况下工作，参数化它们不需要的知识。

这里是一个异构列表的示例实现：

```rs
pub trait HList: Sized {}

pub struct HNil;
impl HList for HNil {}

pub struct HCons<H, T> {
   pub head: H,
   pub tail: T,
}
impl<H, T: HList> HList for HCons<H, T> {}
impl<H, T> HCons<H, T> {
   pub fn pop(self) -> (H, T) {
      (self.head, self.tail)
   }
}
```

注意这个定义故意使用特性来隐藏类型信息，没有这些信息，这样的定义将是不可能的。一个`HList`的声明看起来如下：

```rs
let hl = HCons {
   head: 2,
   tail: HCons {
      head: "abcd".to_string(),
      tail: HNil
   }
};

let (h1,t1) = hl.pop();
let (h2,t2) = t1.pop();
//this would fail
//HNil has no .pop method
//t2.pop();
```

Rust 在类型检查方面有时可能有点僵化。然而，也有许多解决方案允许复杂的行为，这些行为一开始可能看起来是不可能的。

# 调查参数化生命周期

生命周期可能会很快变得复杂。例如，当生命周期用作参数时，它被称为**参数化生命周期**。为了涵盖最常见的问题，我们将生命周期概念分解为四个不同的概念：

+   在基础类型上的生命周期

+   泛型类型上的生命周期

+   特性上的生命周期

+   生命周期子类型

# 在基础类型上定义生命周期

基础类型是一个没有参数的类型。在基础类型上定义生命周期是最简单的情况。所有特质、字段、大小以及任何其他信息对于组类型都是直接可用的。

这里是一个在基础类型上声明生命周期的函数：

```rs
fn ground_lifetime<'a>(x: &'a u64) -> &'a u64
{
   x
}

let x = 3;
ground_lifetime(&x);
```

声明生命周期通常是不必要的。有时，声明生命周期是必要的。推断规则很复杂，有时还会扩展，所以我们现在暂时忽略那部分。

# 在泛型类型上定义生命周期

在泛型类型上声明生命周期需要考虑一个额外的因素。所有具有指定生命周期的泛型类型都必须参数化为具有该生命周期。参数声明必须与参数的使用方式兼容。

这里有一个会失败的示例：

```rs
struct Ref<'a, T>(&'a T);
```

结构定义使用了具有生命周期`'a`的参数`T`；然而，参数`T`并不需要与`'a`具有兼容的生命周期。参数`T`必须由其自己的生命周期约束。这样做后，代码如下：

```rs
struct Ref<'a, T: 'a>(&'a T);
```

现在，参数`T`具有与`'a`兼容的显式约束，代码将能够编译。

# 在特质上定义生命周期

在定义、实现和实例化实现特质的对象时，对象和特质都可能需要生命周期。通常，可以从对象的生命周期推断出特质的生命周期。当这不可能时，程序员必须声明一个与所有其他约束兼容的特质生命周期。代码如下：

```rs
trait Red { }

struct Ball<'a> {
   diameter: &'a i32,
}

impl<'a> Red for Ball<'a> { }

static num: i32 = 5;
let obj = Box::new(Ball { diameter: &num }) as Box<Red + 'static>;
```

# 定义生命周期子类型

有可能存在一个对象，它本身需要一个较长的生命周期，但同时也需要某些组件或方法具有较短的生命周期。这可以通过参数化多个生命周期来实现。这通常工作得很好，除非生命周期发生冲突。以下是一个多个生命周期的示例：

```rs
struct Context<'s>(&'s mut String);

impl<'s> Context<'s>
{
   fn mutate<'c>(&mut self, cs: &'c mut String) -> &'c mut String
```

```rs

   {
      let swap_a = self.0.pop().unwrap();
      let swap_b = cs.pop().unwrap();
      self.0.push(swap_b);
      cs.push(swap_a);
      cs
   }
}

fn main() {
   let mut s = "outside string context abc".to_string();
   {
      //temporary context
      let mut c = Context(&mut s);
      {
         //further temporary context
         let mut s2 = "inside string context def".to_string();
         c.mutate(&mut s2);
         println!("s2 {}", s2);
      }
   }
   println!("s {}", s);
}
```

# 调查参数化类型

在这一点上，了解到所有数据类型声明都可以参数化并不令人惊讶。需要注意的是，在声明参数化数据类型时，生命周期参数必须位于泛型参数之前。以下代码展示了这一点：

```rs
type TFoo<'a, A: 'a> = (&'a A, u64);

struct SFoo<'a, A: 'a>(&'a A);

struct SBar<'a, A: 'a>
{
   x: &'a A
}

enum EFoo<'a, A: 'a>
{
   X { x: &'a A },
   Y { y: &'a A },
```

```rs

}
```

我们也看到了特质如何进行参数化。然而，当一个数据类型和特质都需要参数来实现时会发生什么？这有一个特殊的语法，涉及三个参数列表，看起来如下：

```rs
struct SBaz<'a, 'b, A: 'a, B: 'b>
{
   a: &'a A,
   b: &'b B,
}

trait TBaz<'a, 'b, A: 'a, B: 'b>
{
   fn baz(&self);
}

impl<'a, 'b, A: 'a, B: 'b>
TBaz<'a, 'b, A, B>
for SBaz<'a, 'b, A, B>
{
   fn baz(&self){}
}
```

我们还应提及一个特殊情况，那就是方法歧义的情况。当一个类型实现了多个特质时，可能存在多个同名的方法。为了访问不同的方法，在调用时必须指定要使用哪个`trait`。以下是一个示例：

```rs
trait Foo {
   fn f(&self);
}

trait Bar {
   fn f(&self);
}

struct Baz;

impl Foo for Baz {
   fn f(&self) { println!("Baz’s impl of Foo"); }
}

impl Bar for Baz {
   fn f(&self) { println!("Baz’s impl of Bar"); }
}
```

```rs

let b = Baz;
```

要调用方法，我们必须使用称为**通用函数调用语法**的东西。该语法有两种形式，一种较短，另一种较长。短形式通常足以解决所有但最复杂的情况。以下是一个与前面的类型定义相匹配的示例：

```rs
Foo::f(&b);
Bar::f(&b);

<Baz as Foo>::f(&b);
<Baz as Bar>::f(&b);
```

此外，还有一些不太文档化的语法形式（[`matematikaadit.github.io/posts/rust-turbofish.html`](https://matematikaadit.github.io/posts/rust-turbofish.html)）适用于需要显式提供参数的各种场景。目前 Rust 没有直接的类型注解，因此编译器需要时提供提示。

# 应用参数化概念

我们已经探讨了泛型和参数化的概念。让我们扫描一下项目，看看是否有任何概念适合使用。

# 参数化数据

参数化数据允许我们仅声明所需的最小语义信息量。我们不是指定一个类型，而是指定一个具有特质的泛型参数。让我们首先查看`physics.rs`中的类型声明：

```rs
#[derive(Clone,Serialize,Deserialize,Debug)]
pub enum MotorInput
{
   Up { voltage: f64 },
   Down { voltage: f64 }
}

#[derive(Clone,Serialize,Deserialize,Debug)]
pub struct ElevatorSpecification
{
   pub floor_count: u64,
   pub floor_height: f64,
```

```rs

   pub carriage_weight: f64
}
#[derive(Clone,Serialize,Deserialize,Debug)]
pub struct ElevatorState
{
   pub timestamp: f64,
   pub location: f64,
   pub velocity: f64,
   pub acceleration: f64,
   pub motor_input: MotorInput
}

pub type FloorRequests = Vec<u64>;
```

如果我们记得，当我们设计新的`MotorInput`实现时使用了`physics.rs`，我们应该注意到一个问题。我们希望将`MotorInput`的行为抽象在特质之后；然而，`ElevatorState`指定了一个特定的实现。让我们重新定义`ElevatorState`以使用`motor_input`的泛型类型。该参数应该实现`MotorInput`的所有特质，因此将如下所示：

```rs
#[derive(Clone,Serialize,Deserialize,Debug)]
pub struct ElevatorState<MI: MotorForce + MotorVoltage + Clone, 'a serde::Serialize, 'a serde::Deserialize + Debug>
{
   pub timestamp: f64,
   pub location: f64,
   pub velocity: f64,
   pub acceleration: f64,
   pub motor_input: MI
}
```

这乍一看可能看起来可以接受，但现在`MotorInput`参数和所有特质都必须在提及任何包装`MotorInput`或`ElevatorState`类型的任何地方声明。我们得到了参数的爆炸。必须有一种更好的方法。

在这种情况下，参数爆炸看起来如下，在每个类型声明、特质声明、实现、函数或表达式中：

```rs
pub trait MotorController
<MI: MotorForce + MotorVoltage + Clone, 'a serde::Serialize, 'a serde::Deserialize + Debug>
{
   fn init(&mut self, esp: ElevatorSpecification, est: ElevatorState<MI>);
   fn poll(&mut self, est: ElevatorState<MI>, dst: u64) -> MI;
```

```rs

}

pub trait DataRecorder
<MI: MotorForce + MotorVoltage + Clone, 'a serde::Serialize, 'a serde::Deserialize + Debug>
{
   fn init(&mut self, esp: ElevatorSpecification, est: ElevatorState<MI>);
   fn poll(&mut self, est: ElevatorState<MI>, dst: u64);
}

impl MotorController
<MI: MotorForce + MotorVoltage + Clone, 'a serde::Serialize, 'a serde::Deserialize + Debug>
for SimpleMotorController
<MI: MotorForce + MotorVoltage + Clone, 'a serde::Serialize, 'a serde::Deserialize + Debug>
{
   ...
} 
```

这只是针对一个参数！幸运的是，还有另一个解决这个问题的方法。该技术使用一种称为**特质对象**的东西。特质对象是一个实现了特质的对象，但在编译时没有已知类型。由于特质对象没有具体类型，因此不需要参数化。特质对象的缺点是它们不能被大小化，因此通常必须通过 Box 或其他大小容器间接处理。任何尝试大小化特质对象的行为都会导致编译器错误。同样，任何具有静态方法或不是对象安全的特质都不能与特质对象一起使用。

我们可以将`MotorInput`和`ElevatorState`对象重写为使用特质对象，如下所示：

```rs
#[derive(Clone,Serialize,Deserialize,Debug)]
pub enum SimpleMotorInput
{
   Up { voltage: f64 },
   Down { voltage: f64 }
}

pub trait MotorInput: MotorForce + MotorVoltage
{
}

```

```rs

impl MotorInput for SimpleMotorInput {}
pub struct ElevatorState
{
   pub timestamp: f64,
   pub location: f64,
   pub velocity: f64,
   pub acceleration: f64,
   pub motor_input: Box<MotorInput>
}
```

在这里，我们声明一个`MotorInput`特质有两个子特质来指定行为。我们的`ElevatorState`声明不需要参数；然而，`MotorInput`特质对象必须被包裹在一个`Box`中。由于编译器无法为`MotorInput`特质对象的大小进行编译，这一层间接性是必需的。此外，因为`MotorInput`没有实现`Sized`，它不能使用`Clone`或`serde`宏。我们需要对一些代码进行修改以适应这一点，但这并不令人难以承受。

# 参数化函数和特质对象

在我们的电机控制器中，我们对电机做出了另一个无根据的假设。即，每个电压输入将产生一个平坦的力。电机控制器中的可疑代码如下所示：

```rs
let target_voltage = target_force / 8.0;
```

关于电机比预期更有效率或更无效率的假设可能是错误的。同样，生成的力与电压成线性关系的假设也不太可能。为了满足我们的电机控制器和物理模拟的要求，我们需要一个函数来考虑所使用的物理电机，并将电压转换为力。同样，我们还需要一个逆函数将目标力转换为目标电压。我们可以简单地按以下方式编写这些函数：

```rs
pub fn force_of_voltage(v: f64) -> f64
{
   8.0 * v
}

pub fn voltage_of_force(v: f64) -> f64
{
   v / 8.0
}
```

这看起来很棒，但它并不符合抽象物理电机概念的目标。我们应该将这些函数定义为接口上的方法。这样，我们就可以再次使用特质对象模式来抽象电机的类型，以及电机的类型参数。代码如下：

```rs
pub trait Motor
{
   fn force_of_voltage(&self, v: f64) -> f64;
   fn voltage_of_force(&self, v: f64) -> f64;
}

pub struct SimpleMotor;
impl Motor for SimpleMotor
{
   fn force_of_voltage(&self, v: f64) -> f64
   {
      8.0 * v
   }
   fn voltage_of_force(&self, v: f64) -> f64
   {
      v / 8.0
   }
}
```

在声明了`Motor`特质及其实现之后，我们可以将这个定义与`ElevatorSpecification`结构体集成。结果是如下所示：

```rs
pub struct ElevatorSpecification
{
   pub floor_count: u64,
   pub floor_height: f64,
   pub carriage_weight: f64,
   pub motor: Box<Motor>
}
```

再次，我们失去了使用某些`derive`宏的能力，但至少类型签名要干净得多。现在，电机控制器中的使用支持多个电机：

```rs
let target_voltage = self.esp.motor.voltage_of_force(target_force);
```

我们可以看到，在参数化或泛型行为的不同类型之间可能存在一些潜在的权衡。一方面，参数可能会很快变得难以跟踪。另一方面，特质对象破坏了许多具有诸如`derive`宏、非对象安全特性、需要具体类型等功能的语言。选择正确的工具是一个重要的决定，需要权衡每个选项的优点。

# 参数化特质和实现

现在，我们已经成功地将`Motor`和`MotorInput`实现为特质对象。然而，为了实现这一点，我们牺牲了诸如`Clone`、`Serialize`、`Deserialize`和`Debug`等美好的特性。我们能否恢复这些功能？

首先，让我们尝试复制这些功能。我们将把这些捆绑的特质称为`ElevatorStateClone`和`ElevatorSpecificationClone`。签名可能看起来如下（特质实现可以在`src/physics.rs`文件中找到）：

```rs
pub trait ElevatorStateClone
{
   fn clone(&self) -> ElevatorState;
   fn dump(&self) -> (f64,f64,f64,f64,f64);
   fn load((f64,f64,f64,f64,f64)) -> ElevatorState;
}

pub trait ElevatorSpecificationClone
{
   fn clone(&self) -> ElevatorSpecification;
   fn dump(&self) -> (u64,f64,f64,u64);
   fn load((u64,f64,f64,u64)) -> ElevatorSpecification;
}

impl ElevatorStateClone for ElevatorState {
   ...
}
```

这些特质提供了最基本的功能，让我们能够回到之前使用序列化和复制语义的地方。主要的缺点是每个定义都很冗长。此外，序列化变成了一个元组，而不是直接在正确的类型之间来回转换。

那么，特质对象究竟有什么问题呢？我们知道它们必须被`Box`类型包裹以规避未知的大小。这是问题所在吗？这里有一个程序来测试这个理论：

```rs
#[derive(Serialize,Deserialize)]
struct Foo
{
   bar: Box<u64>
}
```

所以，`Box`类型可以序列化。那么，问题肯定出在特质对象上。让我们用特质对象来做同样的尝试，看看会发生什么：

```rs
trait T {}

#[derive(Serialize,Deserialize)]
struct S1;
impl T for S1 {}

#[derive(Serialize,Deserialize)]
struct S2;
impl T for S2 {}

#[derive(Serialize,Deserialize)]
struct Container
{
   field: Box<T>
}
```

当编译这个最后的片段时，我们得到错误：“为`T`没有实现`serde::Deserialize<'_>`特质”。因此，我们可以看到单独的结构体`S1`和`S2`都实现了`Deserialize`，但这个信息被隐藏了。特质对象`T`本身必须实现`Deserialize`。

在尝试序列化特质对象`T`的第一步中，我们可以遵循编写自定义序列化的说明。结果应该是以下类似的内容：

```rs
impl Serialize for Box<T>
{
   fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
   where S: Serializer
   {
      serializer.serialize_unit_struct("S1")
   }
}

struct S1Visitor;
impl<'de> Visitor<'de> for S1Visitor {
   type Value = Box<T>;

   fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result
   {
      formatter.write_str("an S1 structure")
   }
   fn visit_unit<E>(self) -> Result<Self::Value, E>
   where E: de::Error
```

```rs

   {
      Result::Ok(Box::new(S1))
   }
}

impl<'de> Deserialize<'de> for Box<T> {
   fn deserialize<D>(deserializer: D) -> Result<Box<T>, D::Error>
   where D: Deserializer<'de>
   {
      deserializer.deserialize_unit_struct("S1", S1Visitor)
   }
}

let bt: Box<T> = Box::new(S1);
let s = serde_json::to_string(&bt).unwrap();
let bt: Box<T> = serde_json::from_str(s.as_str()).unwrap();
```

这有点乱，但重要的是我们想要将`S1`或`S2`写入序列化器，并检查那些标签以反序列化。本质上，我们试图创建一个仅用于序列化的辅助枚举。某种方式下，序列化器需要知道`T`是`S1`还是`S2`通过接口，所以为什么不反过来提供一个返回枚举的方法呢？枚举也可以通过宏进行序列化，所以我们可以将自动序列化传递给`T`。让我们尝试一下，从类型和特质定义开始，如下所示：

```rs
#[derive(Clone,Serialize,Deserialize)]
enum T_Enum
{
   S1(S1),
   S2(S2),
}

trait T
{
   fn as_enum(&self) -> T_Enum;
}

#[derive(Clone,Serialize,Deserialize)]
struct S1;
impl T for S1
{
   fn as_enum(&self) -> T_Enum
   {
      T_Enum::S1(self.clone())
   }
```

```rs

}
#[derive(Clone,Serialize,Deserialize)]
struct S2;
impl T for S2
{
   fn as_enum(&self) -> T_Enum
   {
      T_Enum::S2(self.clone())
   }
}
```

在这里，我们可以看到允许在特质对象上有一个将对象转换为枚举的方法没有问题。这种关系是自然的，并提供了一个逃生门，可以在特质对象及其内部表示之间来回转换。现在，为了实现序列化，我们只需要包装和展开枚举序列化器：

```rs
impl Serialize for Box<T>
{
   fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
   where S: Serializer
   {
      self.as_enum().serialize(serializer)
   }
}

impl<'de> Deserialize<'de> for Box<T>
{
   fn deserialize<D>(deserializer: D) -> Result<Box<T>, D::Error>
   where D: Deserializer<'de>
   {
      let result = T_Enum::deserialize(deserializer);
      match result
      {
         Result::Ok(te) => {
            match te {
               T_Enum::S1(s1) => Result::Ok(Box::new(s1.clone())),
               T_Enum::S2(s2) => Result::Ok(Box::new(s2.clone()))
            }
         }
         Result::Err(err) => Result::Err(err)
      }
   }
}
```

那并没有那么糟糕，对吧？使用这项技术，我们可以在保持对数据直接访问和宏推导特质优势的同时，将参数隐藏在特质对象后面。这里有一点点样板代码。幸运的是，对于每个宏，无论你使用什么类型，代码几乎都是相同的。记住这一点；它可能很有用。

# 摘要

在本章中，我们探讨了泛型和参数化编程的基本和深入概念。我们学习了如何向类型、特质、函数和实现声明中添加生命周期、类型和特质参数。我们还考察了根据需要选择性地保留或隐藏类型信息的高级技术。

将这些概念应用于电梯模拟，我们观察到了参数化和泛型如何创建完全抽象的接口。通过使用特质对象，可以完全将特质接口与任何实现分离。我们还观察到了参数化和泛型的缺点或困难。过度使用参数化可能导致参数泄漏，可能需要与接口交互的所有代码本身也变得参数化。另一方面，我们观察到使用特质对象擦除类型信息时的困难。选择保留正确数量的信息很重要。

在下一章中，我们将学习具有复杂要求的实际项目结构。客户将对项目提案做出回应，而您的团队将对新的需求做出回应。

# 问题

1.  什么是代数数据类型？

1.  什么是多态性？

1.  什么是参数多态性？

1.  什么是基类型？

1.  什么是通用函数调用语法？

1.  特质对象的可能类型签名有哪些？

1.  有哪两种方法可以隐藏类型信息？

1.  如何声明子特质？
