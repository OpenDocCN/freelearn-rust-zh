# Rust 内存模型 - 所有权、引用和操作

在上一章，第二章，*Rust 的顺序性能和测试*，我们讨论了影响 Rust 程序顺序性能的因素。我们没有明确地讨论并发性能，因为缺乏关于 Rust 的抽象内存模型如何与机器的真实内存层次结构交互的足够信息。在本章中，我们将讨论 Rust 的内存模型，如何控制类型在内存中的布局，类型是如何被别名的，以及 Rust 的内存安全性是如何工作的。我们将深入研究标准库，以了解这些在实际中是如何体现的。本章还将检查生态系统中的一些常见 crate，这些 crate 将在本书后面的部分对我们感兴趣。请注意，当你阅读这一章时，`rustc` 的实现可能已经改变，这可能会使这里的代码列表与 `rustc` 本身的命名模式不再一致。如果你希望跟随，请查看 SHA `da569fa9ddf8369a9809184d43c600dc06bd4b4d` 的 Rust。

到本章结束时，我们将：

+   研究了 Rust 如何在内存中布局对象

+   讨论了 Rust 指向内存的各种方式及其保证

+   讨论了 Rust 如何分配和释放内存

+   讨论 Rust 如何表示栈和堆分配

+   研究了 `Option`、`Cell`、`CellRef`、`Rc` 和 `Vec` 的内部实现

# 技术要求

这需要有一个工作的 Rust 安装。验证安装的细节在第一章，*预备知识 - 机器架构和 Rust 入门*中有介绍。不需要额外的软件工具。

你可以在 GitHub 上找到本书项目的源代码：[`github.com/PacktPublishing/Hands-On-Concurrency-with-Rust`](https://github.com/PacktPublishing/Hands-On-Concurrency-with-Rust)。本章的源代码位于 `Chapter03`。

# 内存布局

Rust 有一系列机制来在内存中布局复合类型。它们如下：

+   数组

+   枚举

+   结构体

+   元组

这些在内存中的具体布局取决于选择的表现形式。默认情况下，Rust 中的所有内容都是 `repr(Rust)`。所有 `repr(Rust)` 类型都按照 2 的幂次对字节边界对齐。每个类型在内存中至少占用一个字节，然后是两个，然后是四个，以此类推。原始类型——`u8`、`usize`、`bool` 和 `&T`——都与其大小对齐。在 Rust 中，表示结构体的对齐方式取决于最大的字段。考虑以下结构体：

```rs
struct AGC {
  elapsed_time2: u16,
  elapsed_time1: u16,
  wait_list_upper: u32,
  wait_list_lower: u16,
  digital_autopilot: u16,
  fine_scale: u16
}
```

`AGC` 与 `u32` 对齐，并插入适当的填充以匹配 32 位对齐。Rust 将重新排序字段以实现最大打包。枚举不同，受到大量优化的影响，最值得注意的是空指针优化。请参见以下枚举：

```rs
enum AGCInstruction {
  TC,
  TCF,
  CCS(u8),
  B(u16),
  BZF,
}
```

这将按照以下方式布局：

```rs
struct AGCInstructionRepr {
  data: u16,
  tag: u8,
}
```

`data`字段足够宽，可以容纳最大的内部值，而`tag`允许区分不同的情况。当涉及到一个包含非空指针的枚举时，情况会变得复杂，其他情况不能引用相同的指针。`Option<&T>`表示如果在取消引用选项时发现空指针，Rust 可以假设发现了`None`情况。Rust 会优化掉`Option<&T>`的标签。

Rust 支持其他表示形式。`repr(C)`以 C 语言的方式在内存中布局类型，常用于 FFI 项目，我们将在本书后面的内容中看到。`repr(packed)`以`repr(Rust)`类似的方式布局类型，但不会添加填充，对齐仅发生在字节级别。这种表示形式可能会引起未对齐的加载，并对常见 CPU 的性能产生严重影响，特别是本书中我们关注的两种 CPU 架构。其余的表示形式与强制无字段枚举的大小有关——即没有数据在其变体中的枚举——这些表示形式对于确保与 ABI 兼容性有关的大小非常有用。

Rust 的分配默认发生在硬件栈上，这与其他底层语言的做法相同。堆分配必须由程序员显式执行，或者当创建一个包含某种内部存储的新类型时隐式执行。这里有一些复杂性。默认情况下，Rust 类型遵循移动语义：类型位在上下文之间适当地移动。例如：

```rs
fn project_flights() -> Vec<(u16, u8)> {
    let mut t = Vec::new();
    t.push((1968, 2));
    t.push((1969, 4));
    t.push((1970, 1));
    t.push((1971, 2));
    t.push((1972, 2));
    t
}

fn main() {
    let mut total: u8 = 0;
    let flights = project_flights();
    for &(_, flights) in flights.iter() {
        total += flights;
    }
    println!("{}", total);
}
```

`project_flights`函数在堆上分配一个新的`Vec<(u16, u8)>`，填充它，然后将堆分配的向量所有权返回给调用者。这并不意味着`t`的位从`project_flights`的栈帧中复制过来，而是指针`t`从`project_flights`的栈返回到`main`。在 Rust 中，可以通过使用`Copy`特质来实现复制语义。`Copy`类型将在内存中从一个地方复制到另一个地方。Rust 的原始类型是`Copy`——复制它们与移动它们一样快，特别是当类型小于本地指针时。除非你的类型实现了`Drop`特质（定义了类型如何释放自身），否则可以实现你自己的类型的`Copy`。这个限制在 Rust 代码（不使用`unsafe`）中消除了双重释放的可能性。以下代码块为两个用户定义的类型推导了`Copy`，是一个糟糕的随机生成器的示例：

```rs
#[derive(Clone, Copy, PartialEq, Eq)]
enum Project {
    Apollo,
    Gemini,
    Mercury,
}

#[derive(Clone, Copy)]
struct Mission {
    project: Project,
    number: u8,
    duration_days: u8,
}

fn flight() -> Mission {
    Mission {
        project: Project::Apollo,
        number: 8,
        duration_days: 6,
    }
}

fn main() {
    assert_eq!(::std::mem::size_of::<Mission>(), 3);
    let mission = flight();
    if mission.project == Project::Apollo && mission.number == 8 {
        assert_eq!(mission.duration_days, 6);
    }
}
```

# 内存指针

Rust 定义了几种指针类型，每种都有特定的用途。`&T` 是一个共享引用，可能会有许多该共享引用的副本。`&T` 的拥有者不一定拥有 `T`，并且可能无法修改它。共享引用是不可变的。可变引用——写成 `&mut T`——也不一定意味着其他 `&mut T` 必然拥有 `T`，但引用可以用来修改 `T`。对于任何 `T`，可能只有一个 `&mut T` 引用。这使某些代码变得复杂，但意味着 Rust 能够证明两个变量在内存中不重叠，从而解锁了 C/C++ 中不存在的各种优化机会。Rust 引用被设计成编译器能够证明所引用类型的活性：引用不能悬空。这不是 Rust 的原始指针类型——`*const T` 和 `*mut T`——的工作方式，它们与 C 指针类似：它们严格上是内存中的一个地址，对该地址的数据不提供任何保证。因此，许多原始指针的操作都需要 `unsafe` 关键字，并且它们几乎总是仅在性能敏感的代码或 FFI 接口上下文中出现。换句话说，原始指针可能为空；引用永远不会为空。

关于引用的规则通常会在意外地对可变引用进行不可变借用的情况下引起困难。Rust 文档使用以下小型程序来展示这种困难：

```rs
fn main() {
    let mut x: u8 = 5;
    let y: &mut u8 = &mut x;

    *y += 1;

    println!("{}", x);
}
```

`println!` 宏通过引用接收其参数，在这里隐式地创建了一个 `&x`。编译器拒绝了这个程序，因为 `y: &mut u8` 是无效的。如果这个程序能够编译，我们就会面临 `y` 的更新和 `x` 的读取之间的竞争条件，这取决于 CPU 和内存排序。当与结构一起工作时，引用的排他性可能会限制潜在的应用。Rust 允许程序分割结构的借用，只要不重叠的字段不能被别名。

我们在以下简短的程序中展示了这一点：

```rs
use std::fmt;

enum Project {
    Apollo,
    Gemini,
    Mercury,
}

impl fmt::Display for Project {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match *self {
            Project::Apollo => write!(f, "Apollo"),
            Project::Mercury => write!(f, "Mercury"),
            Project::Gemini => write!(f, "Gemini"),
        }
    }
}

struct Mission {
    project: Project,
    number: u8,
    duration_days: u8,
}

fn main() {
    let mut mission = Mission {
        project: Project::Gemini,
        number: 2,
        duration_days: 0,
    };
    let proj: &Project = &mission.project;
    let num: &mut u8 = &mut mission.number;
    let dur: &mut u8 = &mut mission.duration_days;

    *num = 12;
    *dur = 3;

    println!("{} {} flew for {} days", proj, num, dur);
}
```

这种技巧对于通用容器类型来说很难甚至不可能实现。考虑一个映射，其中两个键映射到相同的引用 `T`。或者，现在考虑一个切片：

```rs
use std::fmt;

enum Project {
    Apollo,
    Gemini,
    Mercury,
}

impl fmt::Display for Project {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match *self {
            Project::Apollo => write!(f, "Apollo"),
            Project::Mercury => write!(f, "Mercury"),
            Project::Gemini => write!(f, "Gemini"),
        }
    }
}

struct Mission {
    project: Project,
    number: u8,
    duration_days: u8,
}

fn main() {
    let mut missions: [Mission; 2] = [
        Mission {
            project: Project::Gemini,
            number: 2,
            duration_days: 0,
        },
        Mission {
            project: Project::Gemini,
            number: 12,
            duration_days: 2,
        },
    ];

    let gemini_2 = &mut missions[0];
    let _gemini_12 = &mut missions[1];

    println!(
        "{} {} flew for {} days",
        gemini_2.project, gemini_2.number, gemini_2.duration_days
    );
}
```

这个程序编译失败，错误如下：

```rs
> rustc borrow_split_array.rs
error[E0499]: cannot borrow `missions[..]` as mutable more than once at a time
  --> borrow_split_array.rs:40:27
   |
39 |     let gemini_2 = &mut missions[0];
   |                         ----------- first mutable borrow occurs here
40 |     let _gemini_12 = &mut missions[1];
   |                           ^^^^^^^^^^^ second mutable borrow occurs here
...
46 | }
   | - first borrow ends here

error: aborting due to previous error
```

我们，程序员，知道这是安全的——`gemini_2` 和 `gemini_12` 在内存中不重叠——但是编译器无法证明这一点。如果我们做了以下操作：

```rs
use std::fmt;

enum Project {
    Apollo,
    Gemini,
    Mercury,
}

impl fmt::Display for Project {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match *self {
            Project::Apollo => write!(f, "Apollo"),
            Project::Mercury => write!(f, "Mercury"),
            Project::Gemini => write!(f, "Gemini"),
        }
    }
}

struct Mission {
    project: Project,
    number: u8,
    duration_days: u8,
}

fn main() {
    let gemini_2 = Mission {
        project: Project::Gemini,
        number: 2,
        duration_days: 0,
    };

    let mut missions: [&Mission; 2] = [&gemini_2, &gemini_2];

    let m0 = &mut missions[0];
    let _m1 = &mut missions[1];

    println!(
        "{} {} flew for {} days",
        m0.project, m0.number, m0.duration_days
    );
}
```

根据定义，`missions[0]` 和 `missions[1]` 在内存中重叠。我们，程序员，知道我们正在违反别名规则，而编译器出于保守考虑，假设规则正在被违反。

# 分配和释放内存

分配释放可以通过两种方式之一发生，这取决于类型是在栈上还是在堆上分配的。如果是在栈上，当栈帧本身不再存在时，类型就会被释放。每个 Rust 栈帧都是完全分配但未初始化地来到这个世界，并在其关联的函数退出时离开这个世界。堆分配的类型在最后一个有效的绑定移出作用域时被释放，无论是通过程序的正常流程还是通过程序员显式调用的`std::mem::drop`。`drop`的实现如下：

```rs
fn drop<T>(_x: T) { }
```

`_x`的值被移动到`drop`中——这意味着没有其他对`_x`的借用——然后当`drop`退出时立即超出作用域。然而，显式`drop`不能从作用域中删除项目，因此与结构体的微妙交互将发生，在这些结构体中，Rust 编译器无法证明非重叠别名等情况。`drop`文档讨论了几个案例，值得回顾这些材料。

任何可以被释放的 Rust 类型——这意味着它不是`Copy`——都将实现`Drop`特质，这是一个只有一个函数`drop(&mut self)`的特质。`Drop::drop`不能被显式调用，当类型超出作用域或可以被`std::mem::drop`调用时，它就会被调用，正如之前讨论的那样。

到目前为止，在本章中，我们讨论了内存中类型的布局和在堆上的分配，但没有完全讨论 Rust 内存模型中分配本身是如何工作的。强制堆分配的最简单方法就是使用`std::boxed::Box`。实际上，Rust 对`Box`的文档——也简称为 box——将其描述为 Rust 中最简单的堆分配形式。也就是说，`Box<T>`在堆上为类型`T`分配足够的空间，并作为该分配的所有者。当 box 被丢弃时，`T`的丢弃就会发生。以下是`Box<T>`的定义，直接来自标准库，在文件`src/liballoc/boxed.rs`中：

```rs
pub struct Box<T: ?Sized>(Unique<T>);
```

# 类型的大小

这里有两个重要的事情我们在这本书中还没有遇到——`Sized`和`Unique`。首先，`Sized`，或者更准确地说，`std::marker::Sized`。`Sized`是 Rust 的一个`trait`，它将类型约束为在编译时具有已知的大小。在 Rust 中几乎所有的东西都有一个隐式的`Sized`约束，我们可以检查它。例如：

```rs
use std::mem;

#[allow(dead_code)]
enum Project {
    Mercury { mission: u8 },
    Gemini { mission: u8 },
    Apollo { mission: u8 },
    Shuttle { mission: u8 },
}

fn main() {
    assert_eq!(1, mem::size_of::<u8>());
    assert_eq!(2, mem::size_of::<Project>());

    let ptr_sz = mem::size_of::<usize>();
    assert_eq!(ptr_sz, mem::size_of::<&Project>());

    let vec_sz = mem::size_of::<usize>() * 2 + ptr_sz;
    assert_eq!(vec_sz, mem::size_of::<Vec<Project>>());
}
```

`u8` 是一个字节，Project 是用于区分枚举变体的字节以及内部任务字节，指针是机器字的大小——一个 `usize`，而 `Vec<T>` 保证是一个指针和两个 `usize` 字段，无论 `T` 的大小如何。`Vec<T>` 和 `T` 之间有一些有趣的事情发生，我们将在本章后面深入探讨。请注意，我们几乎在 Rust 中说到了所有东西都有一个隐式的 `Sized` 约束。Rust 支持 *动态大小类型*，这些类型没有已知的大小或对齐。Rust 需要已知的大小和对齐，因此所有 DSTs 都必须位于引用或指针之后。切片——对连续内存的视图——就是这样一种类型。在以下程序中，编译器将无法确定值的切片的大小：

```rs
use std::time::{SystemTime, UNIX_EPOCH};

fn main() {
    let values = vec![0, 1, 2, 3, 4, 5, 7, 8, 9, 10];
    let cur: usize = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap()
        .as_secs() as usize;
    let cap: usize = cur % values.len();

    let slc: &[u8] = &values[0..cap];

    println!("{:?}", slc);
}
```

*切片是内存块的一种视图，该内存块以指针和长度表示，正如原始类型切片的文档所述*。技巧在于长度是在运行时确定的。以下程序将能够编译：

```rs
fn main() {
    let values = vec![0, 1, 2, 3, 4, 5, 7, 8, 9, 10];
    let slc: &[u8] = &values[0..10_000];

    println!("{:?}", slc);
}
```

然而，之前的代码在运行时会引发恐慌：

```rs
> ./past_the_end thread 
'main' panicked at 'index 10000 out of range for slice of length 10', libcore/slice/mod.rs:785:5 note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

Rust 允许程序员将 DSTs（可变大小类型）作为 `struct` 的最后一个字段，如下所示：

```rs
struct ImportantThing {
  tag: u8,
  data: [u8],
}
```

然而，这会导致结构体本身成为 DST。

# 静态和动态分发

在 Rust 中，特质对象同样没有静态已知的大小。特质对象是 Rust 执行动态分发的方式。优先考虑 Rust 是一种静态分发语言，因为存在更多的内联和优化机会——编译器知道得更多。考虑以下代码：

```rs
use std::ops::Add;

fn sd_add<T: Add<Output=T>>(x: T, y: T) -> T {
    x + y
}

fn main() {
    assert_eq!(48, sd_add(16 as u8, 32 as u8));
    assert_eq!(48, sd_add(16 as u64, 32 as u64));
}
```

这里，我们定义了一个函数，`sd_add<T: Add<Output=T>>(x: T, y: T) -> T`。Rust，就像 C++ 一样，将在编译时进行单态化，生成两个 `sd_add` 函数，一个用于 `u8`，另一个用于 `u64`。像 C++ 一样，这可能会增加 Rust 程序的代码大小并减慢编译速度，但好处是在调用者位置允许内联，由于类型特化，可能更有效的实现，以及更少的分支。

当静态分发不可取时，程序员可以构建特质对象以执行动态分发。特质对象没有已知的大小——实际上，它们可以是实现该特质的任何 `T` 类型——因此，就像切片一样，必须位于某种指针之后。在这本书中，动态分发不会得到太多应用。强烈建议读者查阅 `std::raw::TraitObject` 的文档，以获取 Rust 特质对象概念的详细信息。

# 零大小类型

Rust 还支持*零大小类型*或 ZST；没有字段的 struct、单位类型`()`或没有大小的数组都是零大小的。类型没有大小的事实对优化和死树移除是一个福音。Rust API 通常包括如下返回类型：`Result<(), a_mod::ErrorKind>`。这种类型表示，虽然函数可能会出错，但在正常路径下的返回值是单位类型。这些类型在实践中相对较少，但它们是不安全的，Rust 必须意识到它们。许多分配器在请求分配零字节时返回空指针——使得 ZST 的分配与分配器无法找到空闲内存的情况无法区分——并且从 ZST 的指针偏移量为零。这两个考虑因素在本章中都将非常重要。

# 带框的类型

那是`Sized`特性，但`?Sized`是什么意思呢？`?`标志表示相关的特性是可选的。因此，一个框是 Rust 中参数化其他类型`T`的类型，`T`可能具有大小也可能不具有大小。一个框是一种指向堆分配存储的指针。让我们进一步看看它的实现。`Unique<T>`又是怎么回事呢？这个类型是向 Rust 编译器发出信号，表示某些`*mut T`是非空的，并且唯一所有者是`T`，即使`T`是在`Unique`外部分配的。`Unique`的定义如下：

```rs
pub struct Unique<T: ?Sized> {
    pointer: NonZero<*const T>,
    // NOTE: this marker has no consequences for variance, but is
    // necessary for dropck to understand that we logically 
    // own a `T`.
    _marker: PhantomData<T>,
}
```

`NonZero<T>`是一个`rustc`源代码描述为原始指针和整数的包装类型，这些指针和整数永远不会是`NULL`或`0`，这可能会允许某些优化。它以特殊方式注释，以允许本章其他地方讨论的空指针优化。`Unique`也因其使用`PhantomData<T>`而引起兴趣。实际上，`PhantomData`是一个零大小类型，定义为`pub struct PhantomData<T:?Sized>;`。此类型指示 Rust 编译器将`PhantomData<T>`视为拥有`T`，尽管最终`PhantomData`无处存储其新发现的`T`。这对于必须通过维护指向`T`的非零常量指针来拥有`<T>`的`Unique<T>`来说效果很好，但它本身并不在除堆以外的任何地方存储`T`。因此，一个框是一个唯一的、非空的指针，指向内存中某个地方分配的东西，但不在框的存储空间内。

框的内部是编译器内建函数：它们位于分配器和借用检查器之间的交互中，在 Rust 的借用检查器中是一个特殊考虑。考虑到这一点，我们将避免深入挖掘`Box`的内部细节，因为它们会随着编译器版本的更新而变化，而这本书明确不是一本关于`rustc`内部机制的书籍。然而，就我们的目的而言，考虑由框暴露的 API 是有价值的。关键函数包括：

+   `fn from_raw(raw: *mut T) -> Box<T>`

+   `fn from_unique(u: Unique<T>) -> Box<T>`

+   `fn into_raw(b: Box<T>) -> *mut T`

+   `fn into_unique(b: Box<T>) -> Unique<T>`

+   `fn leak<'a>(b: Box<T>) -> &'a mut T`

`from_raw` 和 `from_unique` 都是 unsafe 的。如果原始指针被 *boxed* 多次，或者如果从一个与另一个重叠的指针创建了一个 box，那么从原始指针的转换是不安全的，例如。还有其他可能性。从 `Unique<T>` 的转换是不安全的，因为 `T` 可能或可能不被 `Unique` 所拥有，从而导致 box 不是其内存的唯一所有者。然而，`into_*` 函数是安全的，因为生成的指针将是有效的，但调用者不会对管理内存的生命周期承担全部责任。`Box` 文档指出，调用者可以自己释放内存或将指针转换回它们原来的类型，并允许 Rust 为他们处理。本书将采用后一种方法。最后，还有 `leak`。`leak` 是一个有趣的概念，但在稳定渠道上不可用，但对于将软件部署到嵌入式目标的应用程序来说值得讨论。嵌入式系统的一种常见内存管理策略是在程序的生命周期内预分配所有必要的内存，并且只在该内存上操作。在 Rust 中，如果你想要一个固定大小的未初始化内存：数组和其他原始类型，这是微不足道的。如果你在程序开始时需要堆分配，情况就更加复杂了。这就是 `leak` 的作用：它使内存从 box（堆分配）泄漏到任何你想要的地方。当泄漏的内存打算在程序的生命周期内存在——进入 `static` 生命周期——就没有问题。以下是一个例子，直接来自 `leak` 的文档：

```rs
#![feature(box_leak)]

fn main() {
    let x = Box::new(41);
    let static_ref: &'static mut usize = Box::leak(x);
    *static_ref += 1;
    assert_eq!(*static_ref, 42);
}
```

在这里，我们看到在堆上分配了一个新的 `usize`，泄漏到 `static_ref`——一个具有静态生命周期的可变引用——然后在整个程序的生命周期内对其进行操作。

# 自定义分配器

可以将自定义分配器插入 Rust，就像在 C++ 或类似的系统级编程语言中做的那样。默认情况下，在大多数目标上，Rust 使用 jemalloc，并有一个系统提供的分配器的后备替代方案。在嵌入式应用程序中，可能没有系统，更不用说分配器了，因此建议感兴趣的读者查阅 RFC 1183 ([`github.com/rust-lang/rfcs/blob/master/text/1183-swap-out-jemalloc.md`](https://github.com/rust-lang/rfcs/blob/master/text/1183-swap-out-jemalloc.md))、RFC 1398 ([`github.com/rust-lang/rfcs/pull/1398`](https://github.com/rust-lang/rfcs/pull/1398)) 以及相关的 RFC。截至撰写本文时，将自定义分配器插入稳定 Rust 的接口正在积极讨论中，并且此类功能仅在 nightly Rust 中可用。本书将不会使用自定义分配器。

# 实现

在第二章，“顺序 Rust 性能和测试”中，我们非常简要地探讨了`std::collections::HashMap`的实现。让我们继续以这种方式剖析标准库，特别关注出现的内存问题。

# Option

让我们检查`Option<T>`。我们已经在本章中讨论了`Option<T>`；它由于特别空的`None`变体而受到*空指针优化*的影响。`Option`就像你可能想象的那样简单，它在`src/libcore/option.rs`中定义：

```rs
#[derive(Clone, Copy, PartialEq, PartialOrd, Eq, Ord, Debug, Hash)]
#[stable(feature = "rust1", since = "1.0.0")]
pub enum Option<T> {
    /// No value
    #[stable(feature = "rust1", since = "1.0.0")]
    None,
    /// Some value `T`
    #[stable(feature = "rust1", since = "1.0.0")]
    Some(#[stable(feature = "rust1", since = "1.0.0")] T),
}
```

正如 Rust 内部经常发生的那样，有许多标志用于控制新功能何时何地出现在哪个通道中，以及如何生成文档。`Option<T>`的一个稍微整洁的表达式是：

```rs
#[derive(Clone, Copy, PartialEq, PartialOrd, Eq, Ord, Debug, Hash)]
pub enum Option<T> {
    None,
    Some(T),
}
```

这非常简单，就像你第一次想到它时可能实现选项类型一样。`Option`是其内部`T`的所有者，并且能够根据通常的限制传递对该内部数据的引用。例如：

```rs
    pub fn as_mut(&mut self) -> Option<&mut T> {
        match *self {
            Some(ref mut x) => Some(x),
            None => None,
        }
    }
```

Rust 在每个`trait: Self`内部暴露一个特殊的特质类型。它简单地解糖为引用特质类型，在这种情况下，`Option<T>`。`&mut self`是`self: &mut Self`的缩写，同样`&self`是`self: &Self`的缩写。`as_mut`随后分配一个新的选项，其内部类型是原始内部类型的可变引用。现在，考虑一下谦逊的`map`：

```rs
    pub fn map<U, F: FnOnce(T) -> U>(self, f: F) -> Option<U> {
        match self {
            Some(x) => Some(f(x)),
            None => None,
        }
    }
```

Rust 允许将闭包作为函数的参数。通过这种方式，Rust 与高级语言，尤其是函数式编程语言相似。与这些高级编程语言不同，Rust 对闭包的捕获方式、调用总数和可变性有约束。在这里，我们看到`FnOnce`特质被用来限制传递给 map 函数的闭包`f`为单次使用。函数特质，所有都在`std::ops`中定义，如下所示：

+   `Fn`

+   `FnMut`

+   `FnOnce`

第一个特质`Fn`，Rust 文档描述为是一个*接受不可变接收者的调用操作符*。这可能有点难以理解，直到我们查看`src/libcore/ops/function.rs`中`Fn`的定义：

```rs
pub trait Fn<Args>: FnMut<Args> {
    /// Performs the call operation.
    #[unstable(feature = "fn_traits", issue = "29625")]
    extern "rust-call" fn call(&self, args: Args) -> Self::Output;
}
```

现在它更难以理解了！但是，我们可以回溯。`Args`与`std::env::Args`不同，但扮演着类似的角色，即作为函数参数的占位符，只是在类型级别上。`: FnMut<Args>`意味着`FnMut<Args>`是`Fn<Args>`的*超特质*：所有`FnMut`的方法在用作特质对象时都可用。回想一下，特质对象在之前讨论的动态调度中找到了用途。这也意味着在静态调度的情况下，任何`Fn`的实例都可以用在期望`FnMut`的地方。对理解`Fn`特别感兴趣的是：

```rs
extern "rust-call" fn call(&self, args: Args) -> Self::Output;
```

我们将分部分来处理这个问题。首先，`extern "rust-call"`。在这里，我们定义了一个使用 `"rust-call"` ABI 的内联 extern 块。Rust 支持许多 ABIs，其中三个是跨平台的，并且无论平台如何都保证支持：

+   `extern "Rust" fn`，对于所有未指定其他情况的 Rust 函数是隐式的

+   `extern "C" fn`，常用于 FFI，简写为 `extern fn`

+   `extern "system" fn`，相当于 `extern "C" fn`，但针对一些特殊平台

`"rust-call"` 不出现在该列表中，因为它是一个 Rust 特定的 ABI，它还包括：

+   `extern "rust-intrinsic" fn`，特定于 `rustc` 内置函数

+   `extern "rust-call" fn`，所有 `Fn::call` 函数的 ABI

+   `extern "platform-intrinsic" fn`，文档指出程序员永远不需要处理的东西

在这里，我们向 Rust 编译器发出信号，表示该函数应以特殊的调用 ABI 处理。当启用不稳定的 `fnbox` 功能标志时，这个特定的 `extern` 在编写实现 `Fn` 特性的 traits 时非常重要，因为 `box` 也会这样做：

```rs
impl<'a, A, R> FnOnce<A> for Box<FnBox<A, Output = R> + 'a> {
    type Output = R;

    extern "rust-call" fn call_once(self, args: A) -> R {
        self.call_box(args)
    }
}
```

其次，`fn call(&self, args: Args)`。实现类型以不可变引用的形式传入，除了传入的参数；这是文档中提到的不可变接收者。这里的最后一部分是 `-> Self::Output`，这是在调用操作符使用后的返回类型。这个关联类型默认为 `Self`，但可以被实现者设置。

`FnMut` 与 `Fn` 类似，只是它接受 `&mut self` 而不是 `&self`，而 `FnOnce` 是其超特化：

```rs
pub trait FnMut<Args>: FnOnce<Args> {
    /// Performs the call operation.
    #[unstable(feature = "fn_traits", issue = "29625")]
    extern "rust-call" fn call_mut(&mut self, args: Args) 
        -> Self::Output;
}
```

只有 `FnOnce` 在其定义中有所不同：

```rs
pub trait FnOnce<Args> {
    /// The returned type after the call operator is used.
    #[stable(feature = "fn_once_output", since = "1.12.0")]
    type Output;

    /// Performs the call operation.
    #[unstable(feature = "fn_traits", issue = "29625")]
    extern "rust-call" fn call_once(self, args: Args) -> Self::Output;
}
```

在这里，我们看到 `Self::Output` 作为关联类型出现，并注意 `FnOnce` 的实现是以 `call_once` 而不是 `call` 为基础的。现在我们知道 `FnOnce` 是 `FnMut` 的超特化，而 `FnMut` 是 `Fn` 的超特化，而且这个属性是传递的：如果对 `Fn` 调用 `FnOnce`，则可以使用它。函数特质的许多确切实现都保留在编译器内部，这些实现的细节会随着内部的变化而跳转。实际上，`"rust-call"`、`extern` 的意思是指 `Fn traits` 不能在特殊、编译器特定的上下文中实现；需要启用在夜间构建中确切的功能标志、它们的用途和即将到来的更改并未记录。幸运的是，闭包和函数指针隐式地实现了函数特质。例如：

```rs
fn mlen(input: &str) -> usize {
    input.len()
}

fn main() {
    let v: Option<&str> = Some("hope");

    assert_eq!(v.map(|s| s.len()), v.map(mlen));
}
```

编译器足够智能，可以为我们找出细节。

从概念上讲，`Result<T>` 是一个类似于 `Option<T>` 的类型，但它能够在其 `Err` 变体中传达额外的信息。实际上，`src/libcore/result.rs` 中的实现与我们最初可能写的方式非常相似，尽管与 `Option` 类似：

```rs
pub enum Result<T, E> {
    /// Contains the success value
    #[stable(feature = "rust1", since = "1.0.0")]
    Ok(#[stable(feature = "rust1", since = "1.0.0")] T),

    /// Contains the error value
    #[stable(feature = "rust1", since = "1.0.0")]
    Err(#[stable(feature = "rust1", since = "1.0.0")] E),
}
```

# Cell 和 RefCell

到目前为止，在本章中，我们讨论的可变引用具有 Rust 所称的继承可变性的属性。也就是说，当我们继承其所有权或对值进行独占修改的权利——通过 `&mut`——时，值变得可变。在 Rust 中，继承可变性是首选的，因为它在编译时是静态可强制执行的——我们程序中关于内存变异的任何缺陷都会在编译时被发现。Rust 提供了内部可变性的设施，即对不可变类型的一部分可用的可变性。文档将内部可变性称为一种最后的手段，但尽管不常见，在实践中也并不罕见。Rust 中内部可变性的两种选择是 `Cell<T>` 和 `RefCell<T>`。

让我们考虑 `Cell<T>`。它是如何使用的？正如建议的那样，`Cell<T>` 在某些字段或字段需要可变而您关心的 `T` 是 `T: Copy` 时非常有用。考虑一个执行搜索操作的图结构。这在逻辑上是不可变的——在搜索过程中不需要修改图结构。但是，也考虑一下我们想要记录沿着图边沿的总遍历次数的情况。我们的选择是违反图搜索的逻辑不可变性、要求在图之外存储，或者将一个内部可变的计数器插入到图中。哪种选择最好将取决于具体情况。以下是一个 `Cell<T>` 的例子，与图搜索相比，这个例子要简单得多：

```rs
use std::cell::Cell;

enum Project {
    Apollo,
    Gemini,
    Mercury,
}

struct Mission {
    project: Project,
    number: u8,
    duration_days: Cell<u8>,
}

fn main() {
    let mission = Mission {
        project: Project::Mercury,
        number: 7,
        duration_days: Cell::new(255),
    };

    mission.duration_days.set(0);
    assert_eq!(0, mission.duration_days.get());
}
```

当然，这有点人为。内部可变性通常在非平凡的情况下出现。请注意，`Cell<u8>` 必须通过 `get` 和 `set` 方法来操作，这与直接设置和读取或通过指针读取的正常过程相反。让我们深入探讨 `Cell<T>` 的实现，它在 `src/libcore/cell.rs` 中定义：

```rs
pub struct Cell<T> {
    value: UnsafeCell<T>,
}
```

如同在 Rust 内部常见的那样，一个安全的接口隐藏了一个不安全的内部核心，正如我们在第二章，“顺序 Rust 性能和测试”中看到的那样，使用 `HashMap`。那么 `UnsafeCell` 的定义是什么？

```rs
pub struct UnsafeCell<T: ?Sized> {
    value: T,
}
```

特别值得注意的是紧随其后的一个实现：

```rs
impl<T: ?Sized> !Sync for UnsafeCell<T> {}
```

我们将在下一章更详细地讨论 `Sync`。现在只需说，任何线程安全的类型都必须实现 `Sync`——许多类型会自动实现——通过禁用 `Sync` 特性为 `UnsafeCell<T>`——即 `!Sync` 的含义——类型被特别标记为非线程安全。也就是说，任何线程都可以在任何时候操作 `UnsafeCell` 所拥有的数据。正如我们之前讨论的，Rust 能够利用 `&T` 保证不会在任何地方是可变的以及 `&mut T` 是唯一的这一知识进行大量的优化。`UnsafeCell<T>` 是 Rust 提供的 *唯一* 一种关闭这些编译器优化方法；虽然可以进入一个不安全块并将 `&T` 转换为 `&mut T`，但这被明确指出是未定义的行为。`UnsafeCell<T>` 的关键是，客户端可以检索到多个对内部数据的可变引用，即使 `UnsafeCell<T>` 本身是可变的。确保在任何时候只有一个可变引用的责任在于调用者。`UnsafeCell` 的实现——为了清晰起见，去掉了注释——是简短的：

```rs
impl<T> UnsafeCell<T> {
    pub const fn new(value: T) -> UnsafeCell<T> {
        UnsafeCell { value: value }
    }

    pub unsafe fn into_inner(self) -> T {
        self.value
    }
}

impl<T: ?Sized> UnsafeCell<T> {
    pub fn get(&self) -> *mut T {
        &self.value as *const T as *mut T
    }
}
```

这又带我们回到了 `Cell<T>`。我们知道 `Cell<T>` 是建立在不安全抽象之上的，我们必须要小心不要违反 `&mut T` 的唯一性约束。这是如何做到的？首先，`Cell<T>` 的构建是直接的，正如你所期望的：

```rs
impl<T> Cell<T> {
    pub const fn new(value: T) -> Cell<T> {
        Cell {
            value: UnsafeCell::new(value),
        }
    }
```

`const fn` 可能会让人感到意外，因为截至撰写本书时，它是一个仅在夜间构建版本中存在的特性。在 Rust 中，常量函数是在编译时进行评估的。这种特定定义对程序员的益处在于，`Cell::new` 的结果可以被分配给一个常量变量，这个变量将存在于程序的整个生命周期中。`as_ptr` 和 `get_mut` 是对底层 `T` 的不同视图，一个是原始可变指针，另一个是可变引用：

```rs
    pub fn as_ptr(&self) -> *mut T {
        self.value.get()
    }

    pub fn get_mut(&mut self) -> &mut T {
        unsafe {
            &mut *self.value.get()
        }
    }

    pub fn into_inner(self) -> T {
        unsafe { self.value.into_inner() }
    }
```

注意，虽然 `get_mut` 的内部是不安全的，但借用检查器被用来处理保持 `&mut T` 唯一性的问题，因此 `Cell::get_mut` 本身可以是安全的。`Cell::as_ptr` 没有标记为不安全——在 Rust 中接收原始指针是安全的——但任何调用者都必须在一个不安全块中对那个原始指针进行解引用：可能存在多个原始的可变指针在浮动。将新值设置到单元格中是通过替换来完成的，这将在后面讨论，但需要仔细注意强制丢弃从单元格中拉出的 `T`：

```rs
    pub fn set(&self, val: T) {
        let old = self.replace(val);
        drop(old);
    }
```

`Cell::swap` 和 `Cell::replace` 是基于 `std::ptr` 和 `std::mem` 的底层内存操作工具完成的。`swap` 的目的是用一个单元格的内部替换另一个单元格的内部。其定义如下：

```rs
    pub fn swap(&self, other: &Self) {
        if ptr::eq(self, other) {
            return;
        }
        unsafe {
            ptr::swap(self.value.get(), other.value.get());
        }
    }
```

`swap` 是通过 `std::ptr::swap` 来实现的，这是一个被文档化的函数，它通过传递给它的原始指针作为参数来复制内存。你会注意到 `Cell::swap` 小心避免当传递的 `other` 等同于 `self` 时的 `swap`。当我们查看 `std::ptr::swap` 的定义时，这个原因就变得清晰了，该定义位于 `src/libcore/ptr.rs`：

```rs
pub unsafe fn swap<T>(x: *mut T, y: *mut T) {
    // Give ourselves some scratch space to work with
    let mut tmp: T = mem::uninitialized();

    // Perform the swap
    copy_nonoverlapping(x, &mut tmp, 1);
    copy(y, x, 1); // `x` and `y` may overlap
    copy_nonoverlapping(&tmp, y, 1);

    // y and t now point to the same thing, but we need to completely 
    forget `tmp`
    // because it's no longer relevant.
    mem::forget(tmp);
}
```

在这里，`copy_nonoverlapping` 和复制的确切细节并不重要，除了注意交换确实需要分配未初始化的空间，并从这个空间中来回复制。如果你不需要这样做，避免这项工作是很明智的。`Cell::replace` 的工作方式类似：

```rs
    pub fn replace(&self, val: T) -> T {
        mem::replace(unsafe { &mut *self.value.get() }, val)
    }
}
```

`std::mem::replace` 接受一个 `&mut T` 和一个 `T`，然后将 `&mut T` 中的值替换为传入的 `val`，返回旧值，并且不丢弃。`std::mem::replace` 的定义在 `src/libcore/mem.rs` 中，如下所示：

```rs
pub fn replace<T>(dest: &mut T, mut src: T) -> T {
    swap(dest, &mut src);
    src
}
```

追踪同一模块中 `swap` 的定义，我们发现它是这样的：

```rs
pub fn swap<T>(x: &mut T, y: &mut T) {
    unsafe {
        ptr::swap_nonoverlapping(x, y, 1);
    }
}
```

`std::ptr::swap_nonoverlapping` 的定义如下：

```rs
pub unsafe fn swap_nonoverlapping<T>(x: *mut T, y: *mut T, count: usize) {
    let x = x as *mut u8;
    let y = y as *mut u8;
    let len = mem::size_of::<T>() * count;
    swap_nonoverlapping_bytes(x, y, len)
}
```

最后，私有的 `std::ptr::swap_nonoverlapping_bytes` 定义如下：

```rs
unsafe fn swap_nonoverlapping_bytes(x: *mut u8, y: *mut u8, len: usize) {
    // The approach here is to utilize simd to swap x & y efficiently. 
    // Testing reveals that swapping either 32 bytes or 64 bytes at
    // a time is most efficient for intel Haswell E processors. 
    // LLVM is more able to optimize if we give a struct a 
    // #[repr(simd)], even if we don't actually use this struct
    // directly.
    //
    // FIXME repr(simd) broken on emscripten and redox
    // It's also broken on big-endian powerpc64 and s390x.  #42778
    #[cfg_attr(not(any(target_os = "emscripten", target_os = "redox",
                       target_endian = "big")),
               repr(simd))]
    struct Block(u64, u64, u64, u64);
    struct UnalignedBlock(u64, u64, u64, u64);

    let block_size = mem::size_of::<Block>();

    // Loop through x & y, copying them `Block` at a time
    // The optimizer should unroll the loop fully for most types
    // N.B. We can't use a for loop as the `range` impl calls 
    // `mem::swap` recursively
    let mut i = 0;
    while i + block_size <= len {
        // Create some uninitialized memory as scratch space

        // Declaring `t` here avoids aligning the stack when this loop 
        // is unused
        let mut t: Block = mem::uninitialized();
        let t = &mut t as *mut _ as *mut u8;
        let x = x.offset(i as isize);
        let y = y.offset(i as isize);

        // Swap a block of bytes of x & y, using t as a temporary 
        // buffer. This should be optimized into efficient SIMD 
        // operations where available
        copy_nonoverlapping(x, t, block_size);
        copy_nonoverlapping(y, x, block_size);
        copy_nonoverlapping(t, y, block_size);
        i += block_size;
    }

    if i < len {
        // Swap any remaining bytes
        let mut t: UnalignedBlock = mem::uninitialized();
        let rem = len - i;

        let t = &mut t as *mut _ as *mut u8;
        let x = x.offset(i as isize);
        let y = y.offset(i as isize);

        copy_nonoverlapping(x, t, rem);
        copy_nonoverlapping(y, x, rem);
        copy_nonoverlapping(t, y, rem);
    }
}
```

哇！这真是一个兔子洞。最终，我们在这里发现的是 `std::mem::replace` 是通过在内存中一个非重叠位置到另一个位置的块复制来定义的，这个过程是通过利用 LLVM 在通用处理器上优化位操作的能力，Rust 开发者试图使其尽可能高效。真不错。

那么 `RefCell<T>` 呢？它也是一个在 `UnsafeCell<T>` 之上的安全抽象，除了 `Cell<T>` 的复制限制被取消。这使得实现稍微复杂一些，正如我们将看到的：

```rs
pub struct RefCell<T: ?Sized> {
    borrow: Cell<BorrowFlag>,
    value: UnsafeCell<T>,
}
```

与 `Cell` 一样，`RefCell` 也有一个内部不安全值，这一点是相同的。有趣的是 `borrow: Cell<BorrowFlag>`。`RefCell<T>` 是 `Cell<T>` 的客户端，这很有道理，因为不可变的 `RefCell` 将需要内部可变性来跟踪其内部数据的借用总数。`BorrowFlag` 的定义如下：

```rs
// Values [1, MAX-1] represent the number of `Ref` active
// (will not outgrow its range since `usize` is the size 
// of the address space)
type BorrowFlag = usize;
const UNUSED: BorrowFlag = 0;
const WRITING: BorrowFlag = !0;
```

`RefCell<T>` 的实现类似于 `Cell<T>`。`RefCell::replace` 也是通过 `std::mem::replace` 来实现的，`RefCell::swap` 是通过 `std::mem::swap` 来实现的。有趣的地方在于 `RefCell` 中新出现的函数，这些是与借用相关的函数。我们将首先查看 `try_borrow` 和 `try_borrow_mut`，因为它们被用于其他借用函数的实现。`try_borrow` 的定义如下：

```rs
    pub fn try_borrow(&self) -> Result<Ref<T>, BorrowError> {
        match BorrowRef::new(&self.borrow) {
            Some(b) => Ok(Ref {
                value: unsafe { &*self.value.get() },
                borrow: b,
            }),
            None => Err(BorrowError { _private: () }),
        }
    }
```

`BorrowRef` 的定义如下：

```rs
struct BorrowRef<'b> {
    borrow: &'b Cell<BorrowFlag>,
}

impl<'b> BorrowRef<'b> {
    #[inline]
    fn new(borrow: &'b Cell<BorrowFlag>) -> Option<BorrowRef<'b>> {
        match borrow.get() {
            WRITING => None,
            b => {
                borrow.set(b + 1);
                Some(BorrowRef { borrow: borrow })
            },
        }
    }
}

impl<'b> Drop for BorrowRef<'b> {
    #[inline]
    fn drop(&mut self) {
        let borrow = self.borrow.get();
        debug_assert!(borrow != WRITING && borrow != UNUSED);
        self.borrow.set(borrow - 1);
    }
}
```

`BorrowRef` 是一个结构，它持有 `RefCell<T>` 的借用字段的引用。创建一个新的 `BorrowRef` 依赖于那个借用的值；如果值是 `WRITING`，则不会创建 `BorrowRef`——返回 `None`——否则，总的借用次数会增加。这实现了在需要写入时的互斥性，同时允许有多个读取者——当有写入引用时，`try_borrow` 不可能分配一个引用。现在，让我们考虑 `try_borrow_mut`：

```rs
    pub fn try_borrow_mut(&self) -> Result<RefMut<T>, BorrowMutError> {
        match BorrowRefMut::new(&self.borrow) {
            Some(b) => Ok(RefMut {
                value: unsafe { &mut *self.value.get() },
                borrow: b,
            }),
            None => Err(BorrowMutError { _private: () }),
        }
    }
```

再次，我们发现了一个基于另一种类型 `BorrowRefMut` 的实现：

```rs
struct BorrowRefMut<'b> {
    borrow: &'b Cell<BorrowFlag>,
}

impl<'b> Drop for BorrowRefMut<'b> {
    #[inline]
    fn drop(&mut self) {
        let borrow = self.borrow.get();
        debug_assert!(borrow == WRITING);
        self.borrow.set(UNUSED);
    }
}

impl<'b> BorrowRefMut<'b> {
    #[inline]
    fn new(borrow: &'b Cell<BorrowFlag>) -> Option<BorrowRefMut<'b>> {
        match borrow.get() {
            UNUSED => {
                borrow.set(WRITING);
                Some(BorrowRefMut { borrow: borrow })
            },
            _ => None,
        }
    }
}
```

关键点，就像 `BorrowRef` 一样，在于 `BorrowRefMut::new`。在这里，我们可以看到如果 `RefCell` 的内部借用未被使用，则借用被设置为写入，排除任何潜在的读取引用。同样，如果存在读取引用，创建可变引用将失败。因此，在运行时通过抽象一个允许打破该保证的不安全结构，我们持有排他性可变引用和多个不可变引用。

# Rc

现在，在我们进入多项目容器之前，让我们考虑一个最后的单项目容器。我们将探讨由 Rust 文档描述的 `Rc<T>`，它是一个单线程引用计数的指针。引用计数指针与 Rust 的常规引用不同，因为它们虽然作为 `Box<T>` 在堆上分配，但克隆引用计数指针不会导致新的堆分配，而是进行位拷贝。相反，`Rc<T>` 内部的计数器会增加，这在某种程度上类似于 `RefCell<T>` 的工作方式。`Rc<T>` 的释放会减少内部计数器，当计数器的值等于零时，堆分配就会被释放。让我们看一下 `src/liballoc/rc.s` 的内部结构：

```rs
pub struct Rc<T: ?Sized> {
    ptr: Shared<RcBox<T>>,
    phantom: PhantomData<T>,
}
```

我们在本章中已经遇到了 `PhantomData<T>`，所以我们知道 `Rc<T>` 不会直接持有 `T` 的分配，但编译器会表现得好像它持有一样。我们还知道 `T` 可能是有大小的，也可能不是。我们需要了解的片段是 `RcBox<T>` 和 `Shared<T>`。让我们首先检查 `Shared<T>`。它的全名是 `std::ptr::Shared<T>`，它与 `*mut X` 类似的指针，除了它是非零的。创建新的 `Shared<T>` 有两种变体，`const unsafe fn new_unchecked(ptr: *mut X) -> Self` 和 `fn new(ptr: *mut X) -> Option<Self>`。在第一种变体中，调用者负责确保指针是非空的，而在第二种变体中，会检查指针是否为空，如下所示：

```rs
    pub fn new(ptr: *mut T) -> Option<Self> {
        NonZero::new(ptr as *const T).map(|nz| Shared { pointer: nz })
    }
```

我们在 `src/libcore/nonzero.rs` 中找到了 `NonZero<T>` 的定义和实现，如下所示：

```rs
pub struct NonZero<T: Zeroable>(T);

impl<T: Zeroable> NonZero<T> {
    /// Creates an instance of NonZero with the provided value.
    /// You must indeed ensure that the value is actually "non-zero".
    #[unstable(feature = "nonzero",
               reason = "needs an RFC to flesh out the design",
               issue = "27730")]
    #[inline]
    pub const unsafe fn new_unchecked(inner: T) -> Self {
        NonZero(inner)
    }

    /// Creates an instance of NonZero with the provided value.
    #[inline]
    pub fn new(inner: T) -> Option<Self> {
        if inner.is_zero() {
            None
        } else {
            Some(NonZero(inner))
        }
    }

    /// Gets the inner value.
    pub fn get(self) -> T {
        self.0
    }
}
```

`Zeroable` 是一个不稳定特质，它相当直接：

```rs
pub unsafe trait Zeroable {
    fn is_zero(&self) -> bool;
}
```

每个指针类型都实现了一个`is_null() -> bool`函数，这个特性将调用该函数，而`NonZero::new`则委托给`Zeroable::is_zero`。因此，存在`Shared<T>`时，给程序员提供的自由度与`*mut T`相同，但增加了关于指针可空情况的额外保证。回到`Rc::new`：

```rs
pub fn new(value: T) -> Rc<T> {
    Rc {
        // there is an implicit weak pointer owned by all the strong
        // pointers, which ensures that the weak destructor never frees
        // the allocation while the strong destructor is running, even
        // if the weak pointer is stored inside the strong one.
        ptr: Shared::from(Box::into_unique(box RcBox {
            strong: Cell::new(1),
            weak: Cell::new(1),
            value,
        })),
        phantom: PhantomData,
    }
}
```

`Box::into_unique`将`Box<T>`转换为`Unique<T>`——在本章中已讨论过——然后将其转换为`Shared<T>`。这个链保留了所需的非空保证并确保了唯一性。现在，关于`RcBox`中的强引用和弱引用又是怎样的呢？`Rc<T>`提供了一个方法，`Self::downgrade(&self) -> Weak<T>`，它产生一个非拥有指针，一个不保证引用数据存活性的指针，并且不扩展其生命周期。这被称为*弱引用*。同样，释放`Weak<T>`也不意味着`T`被释放。这里的技巧是，强引用确实扩展了底层`T`的存活性——只有当`Rc<T>`的内部计数器达到零时，才会调用`T`的释放。在大多数情况下，很少需要弱引用，除非存在引用循环。假设要构建一个图结构，其中每个节点都持有对其连接节点的`Rc<T>`引用，并且图中存在循环。在当前节点上调用释放将递归地调用连接节点的释放，这将再次递归到当前节点，依此类推。如果图存储所有节点的向量，并且每个节点存储对连接的弱引用，那么向量的释放将导致所有节点被释放，无论是否存在循环。我们可以通过检查`Rc<T>`的`Drop`实现来了解这是如何在实际中工作的：

```rs
unsafe impl<#[may_dangle] T: ?Sized> Drop for Rc<T> {
    fn drop(&mut self) {
        unsafe {
            let ptr = self.ptr.as_ptr();

            self.dec_strong();
            if self.strong() == 0 {
                // destroy the contained object
                ptr::drop_in_place(self.ptr.as_mut());

                // remove the implicit "strong weak" pointer
                // now that we've destroyed the contents.
                self.dec_weak();

                if self.weak() == 0 {
                    Heap.dealloc(ptr as *mut u8, 
                    Layout::for_value(&*ptr));
                }
            }
        }
    }
}
```

这部分内容为了清晰起见进行了适当的缩减，但概念与我们描述的是一样的——当强引用的总数为零时，将发生完全的释放。被引用的`Heap`和`Layout`是编译器内部实现，这里不再进一步讨论，但有兴趣的读者被鼓励自行探索。回想一下在`Rc<T>::new`中，强引用和弱引用计数器都是从`1`开始的。为了避免使弱指针无效，只有当没有强引用或弱引用可用时，实际的`T`才会被释放。让我们再次看看`Drop for Weak<T>`，为了清晰起见，这里也进行了适当的缩减：

```rs
impl<T: ?Sized> Drop for Weak<T> {
    fn drop(&mut self) {
        unsafe {
            let ptr = self.ptr.as_ptr();

            self.dec_weak();
            // the weak count starts at 1, and will only go to
            // zero if all the strong pointers have disappeared.
            if self.weak() == 0 {
                Heap.dealloc(ptr as *mut u8, Layout::for_value(&*ptr));
            }
        }
    }
}
```

如预期的那样，只有当弱指针总数降至零时，`T`才能被释放，而这只有在没有剩余强指针的情况下才可能。这就是`Rc<T>`——一些重要的特性、一个专门的盒子和一些编译器内部实现。

# Vec

作为本章的最后一个主题，让我们考虑`Vec<T>`。Rust 向量是一个可增长的数组，也就是说，一个连续、同质的存储区域，可以根据需要通过重新分配来增长，或者它可以缩小。与数组相比的优势在于不需要提前知道你需要多少存储空间，以及可切片结构和类型 API 中额外函数的所有好处。`Vec<T>`是一个非常常见的 Rust 结构，如此之多，以至于它的实际名称是`std::vec::Vec<T>`，但 Rust 默认导入此类型。Rust 程序充满了向量，因此理解它与所持内存的交互方式是很有道理的。

`Vec<T>`在`src/liballoc/vec.rs`中定义，如下所示：

```rs
pub struct Vec<T> {
    buf: RawVec<T>,
    len: usize,
}
```

Rust 文档声明，`Vec<T>`将按指针、`usize`容量和`usize`长度的顺序布局。这些字段的顺序完全未指定。正如我们之前讨论的，Rust 将根据需要重新排序字段。此外，指针保证不为空，允许进行空指针优化。之前，我们看到了 Rust 的常用技巧：用较低级（或原始）结构或甚至用完全不安全的结构来定义高级结构。我们已经看到了长度`usize`，称为`len`。这意味着容量和长度之间存在明显的区别，我们将在深入研究`RawVec<T>`时回到这一点。让我们看一下定义在`src/liballoc/raw_vec.rs`中的`RawVec<T>`：

```rs
pub struct RawVec<T, A: Alloc = Heap> {
    ptr: Unique<T>,
    cap: usize,
    a: A,
}
```

我们拥有的是一个指向`Unique<T>`的指针，这是一个添加了额外保证的`*mut T`，即`RawVec<T>`是唯一拥有分配的持有者，且指针不为空。`cap`是原始向量的容量。`a: A`是分配器，默认为`Heap`。Rust 允许交换分配器，只要实现遵守——截至撰写本书时——`std::heap::Alloc`特质。交换分配器是 Rust 的一个不稳定特性，仅在夜间频道中可用，但足够稳定，足以在嵌入式 Rust 社区的库中看到其常见使用。在这本书中，我们不会使用除默认分配器以外的任何东西，但强烈鼓励读者更深入地探索这个主题。抛开分配器不谈，还有 Rust 文档承诺的指针、长度和容量。让我们回到`Vec<T>`，看看`new`：

```rs
    pub fn new() -> Vec<T> {
        Vec {
            buf: RawVec::new(),
            len: 0,
        }
    }
```

现在，让我们看看原始`vec`的`new`：

```rs
    pub fn new() -> Self {
        Self::new_in(Heap)
    }
```

这将创建推迟到`new_in`，这是同一`trait`上的一个函数：

```rs
    pub fn new_in(a: A) -> Self {
        // !0 is usize::MAX. This branch should be stripped 
        // at compile time.
        let cap = if mem::size_of::<T>() == 0 { !0 } else { 0 };

        // Unique::empty() doubles as "unallocated" and "zero-sized
        // allocation"
        RawVec {
            ptr: Unique::empty(),
            cap,
            a,
        }
    }
```

`cap`计算很有趣。在`Vec<T>`——以及扩展的`RawVec<T>`——中可以存储一个`T`，其大小为零。实现者知道这一点，并提出了一种有趣的方法：如果类型的大小为零，则将容量设置为`usize::MAX`，否则为 0。由于在达到容量之前就会耗尽内存，因此不可能将`usize::MAX`个元素塞入一个向量，现在可以区分零大小类型的案例，而无需引入枚举或标志变量。这是一个整洁的技巧。如果我们回到向量并检查`with_capacity`，我们会发现它委托给`RawVec::with_capacity`，它又委托给`RawVec::allocate_in`：

```rs
    fn allocate_in(cap: usize, zeroed: bool, mut a: A) -> Self {
        unsafe {
            let elem_size = mem::size_of::<T>();

            let alloc_size = 
            cap.checked_mul(elem_size).expect("capacity overflow");
            alloc_guard(alloc_size);

            // handles ZSTs and `cap = 0` alike
            let ptr = if alloc_size == 0 {
                mem::align_of::<T>() as *mut u8
            } else {
                let align = mem::align_of::<T>();
                let result = if zeroed {
                    a.alloc_zeroed(Layout::from_size_align(
                        alloc_size, align).unwrap())
                } else {
                    a.alloc(Layout::from_size_align(
                        alloc_size, align).unwrap())
                };
                match result {
                    Ok(ptr) => ptr,
                    Err(err) => a.oom(err),
                }
            };

            RawVec {
                ptr: Unique::new_unchecked(ptr as *mut _),
                cap,
                a,
            }
        }
    }
```

这里有很多事情在进行中，但这是非常重要的，所以让我们将其分解成小块。首先，看看以下内容：

```rs
            let elem_size = mem::size_of::<T>();

            let alloc_size = 
            cap.checked_mul(elem_size).expect("capacity overflow");
            alloc_guard(alloc_size);
```

在函数的开头，计算了元素的大小，并确认调用者请求的总分配量不超过可用的系统内存。注意，`checked_mul`确保我们不会溢出 usize 并意外分配过少的内存。最后，调用了一个名为`alloc_guard`的函数。如下所示：

```rs
fn alloc_guard(alloc_size: usize) {
    if mem::size_of::<usize>() < 8 {
        assert!(
            alloc_size <= ::core::isize::MAX as usize,
            "capacity overflow"
        );
    }
}
```

这是一个保证检查。记住，usize 和 isize 是系统指针的带符号和无符号大小。为了理解这个保护措施，我们必须了解两个问题的答案：

1.  在给定的机器上，我能分配多少内存？

1.  在给定的机器上，我能访问多少内存？

可以从操作系统分配 usize 字节，但在这里，Rust 正在检查分配的大小是否小于 isize 的最大值。Rust 参考（[`doc.rust-lang.org/stable/reference/types.html#machine-dependent-integer-types`](https://doc.rust-lang.org/stable/reference/types.html#machine-dependent-integer-types)）解释了原因：

"isize 类型是一个与平台指针类型位数相同的有符号整数类型。对象和数组大小的理论上限是 isize 的最大值。这确保了 isize 可以用来计算指向对象或数组的指针之间的差异，并且可以访问对象内的每个字节以及末尾之后的一个字节。"

结合`checked_mul`的容量检查，我们知道分配的大小是合适的，并且在整个分配中都是可寻址的：

```rs
            let ptr = if alloc_size == 0 {
                mem::align_of::<T>() as *mut u8
            } else {
```

如果所需的容量为零或`T`的大小为零，则实现将`T`的最小对齐方式强制转换为指向字节的指针，即`*mut u8`。这个指针，嗯，它指向没有用的地方，但实现避免了在没有可以分配的内容时进行分配，无论是由于零大小类型而永远不会分配，还是其他原因。这是好的，并且实现只需要意识到，当容量为零或类型大小为零时，指针不能被解引用。正确：

```rs
            } else {
                let align = mem::align_of::<T>();
                let result = if zeroed {
                    a.alloc_zeroed(Layout::from_size_align(alloc_size, 
                        align).unwrap())
                } else {
                    a.alloc(Layout::from_size_align(alloc_size, 
                        align).unwrap())
                };
                match result {
                    Ok(ptr) => ptr,
                    Err(err) => a.oom(err),
                }
            };
```

当需要分配内存时，会触发此分支。值得注意的是，有两个子可能性，由`zeroed`参数控制：要么内存被初始化为零，要么它保持未初始化状态。`Vec<T>`不向最终用户公开此选项，但我们通过检查知道内存开始时是未初始化的，这是一种对非平凡分配的优化：

```rs
            RawVec {
                ptr: Unique::new_unchecked(ptr as *mut _),
                cap,
                a,
            }
        }
    }
```

在某些情况下，缺乏初始化标志有点棘手。考虑`std::io::Read::read_exact`这个函数。它接受一个`&mut [u8]`，并且通常是从特别创建的 vec 中创建这个切片。以下代码将不会读取 1024 字节：

```rs
use std::io;
use std::io::prelude::*;
use std::fs::File;

let mut f = File::open("foo.txt")?;
let mut buffer = Vec:with_capacity(1024);

f.read_exact(&mut buffer)?;
```

为什么？我们传递的切片实际上长度为零！Rust 的解引用允许通过两个特质：`std::ops::Deref`和`std::ops::DerefMut`，这取决于你对不可变或可变切片的需求。`Deref`特质定义为以下：

```rs
pub trait Deref {
    type Target: ?Sized;
    fn deref(&self) -> &Self::Target;
}
```

以及可变版本的类似操作。当我们像前面的代码块那样切片我们的向量时，我们调用这个`DerefMut::deref`：

```rs
impl<T> ops::DerefMut for Vec<T> {
    fn deref_mut(&mut self) -> &mut [T] {
        unsafe {
            let ptr = self.buf.ptr();
            assume(!ptr.is_null());
            slice::from_raw_parts_mut(ptr, self.len)
        }
    }
}
```

这是非常重要的部分：`slice::from_raw_parts_mut(ptr, self.len)`。向量切片是通过长度构建的，而不是容量。向量的容量用于区分已分配的内存与已初始化的内存，无论是初始化为零还是插入其他值。这是一个重要的区别。我们有可能自行初始化内存：

```rs
let mut buffer: Vec<u8> = Vec::with_capacity(1024);
for _ in 0..1024 {
  buffer.push(0);
}
assert_eq!(1024, buffer.len());
```

或者依赖`Vec` API 从固定大小的数组转换：

```rs
let buffer = [0; 1024];
let buffer = &mut buffer.to_vec();
assert_eq!(1024, buffer.len());
}
```

两种方法都可行。你选择哪种取决于你是否提前知道缓冲区的大小。

对于这样一个重要的类型，`Vec<T>`上有许多 API 函数。在本章的剩余部分，我们将关注两个变异：`push`和`insert`。`push`函数是一个常数操作，除了任何必要的重新分配来应对容量限制达到的情况。以下是`push`函数：

```rs
    pub fn push(&mut self, value: T) {
        // This will panic or abort if we would allocate > isize::MAX bytes
        // or if the length increment would overflow for zero-sized types.
        if self.len == self.buf.cap() {
            self.buf.double();
        }
        unsafe {
            let end = self.as_mut_ptr().offset(self.len as isize);
            ptr::write(end, value);
            self.len += 1;
        }
    }
```

如果你还记得，`self.buf`是底层的`RawVec<T>`。`Vec<T>`的文档指出，当需要重新分配时，底层内存将加倍，这实际上是由`RawVec<T>::double`函数处理的。该函数相当长，并且正如你可能猜到的，它包含一系列计算新加倍大小的算术，并在存在现有分配时执行 realloc，否则在存在新分配时。这是值得列出的：

```rs
                None => {
                    // skip to 4 because tiny Vec's are dumb; 
                    // but not if that would cause overflow
                    let new_cap = if elem_size > (!0) / 8 { 
                                      1 
                                  } else { 
                                      4 
                                  };
                    match self.a.alloc_array::<T>(new_cap) {
                        Ok(ptr) => (new_cap, ptr),
                        Err(e) => self.a.oom(e),
                    }
                }
```

`Alloc::alloc_array`为连续内存分配空间，这适合容纳`new_cap`个大小为`T`的元素，并返回新分配空间第一个地址的指针。那么，这就是`Vec<T>`文档中承诺的连续内存！回到`Vec<T>`，现在容量至少是原来的两倍，值`T`可以插入：

```rs
        unsafe {
            let end = self.as_mut_ptr().offset(self.len as isize);
            ptr::write(end, value);
            self.len += 1;
        }
```

Rust 的指针很方便，因为它们的偏移函数会考虑到`T`的大小；调用者不需要做额外的乘法。实现是确定连续分配中第一个未使用的`T`大小空间——表示为`end`——然后将移动的`T`写入该地址。重要的是这个空间是未使用的，因为`std::ptr::write`不会释放可能原本存在于写入指针处的任何内存。如果你在活动引用上使用`ptr::write`，会发生奇怪的事情。

最后，是`insert`函数。`Vec<T>`的`insert`函数允许在向量的任何有效索引处插入`T`。如果插入到向量的末尾，该方法是功能上等同于推送的，尽管在机械上不同，我们很快就会看到。然而，如果插入发生在向量内部，则插入索引右侧的所有元素都会移动一次，这是一个依赖于分配大小的非平凡操作。以下是`insert`的完整列表：

```rs
    pub fn insert(&mut self, index: usize, element: T) {
        let len = self.len();
        assert!(index <= len);

        // space for the new element
        if len == self.buf.cap() {
            self.buf.double();
        }

        unsafe {
            // infallible
            // The spot to put the new value
            {
                let p = self.as_mut_ptr().offset(index as isize);
                // Shift everything over to make space. (Duplicating the
                // `index`th element into two consecutive places.)
                ptr::copy(p, p.offset(1), len - index);
                // Write it in, overwriting the first copy of the `index`th
                // element.
                ptr::write(p, element);
            }
            self.set_len(len + 1);
        }
    }
```

在本章的这个阶段，索引检查和潜在的缓冲是直接的。在`unsafe`块中，`p`是插入的正确偏移量。两条重要的行是：

```rs
ptr::copy(p, p.offset(1), len - index);
ptr::write(p, element);
```

当一个值被插入到向量中时，无论位置如何，从`p`开始到列表末尾的所有内存都会被复制到以`p+1`开始的内存上。一旦完成这个操作，插入的值就会覆盖`p`。这是一个非平凡的运算，可能会变得非常慢，尤其是如果需要移动的分配相当大时。以性能为导向的 Rust 会尽量少用，甚至不用`Vec::insert`。`Vec<T>`几乎肯定会出现在其中。

# 摘要

在本章中，我们介绍了 Rust 对象在内存中的布局，引用在安全和不安全 Rust 中的工作方式，并解决了 Rust 程序的多种分配策略。我们对 Rust 标准库中的类型进行了深入研究，以使这些概念具体化，并希望读者现在会感到舒适地进一步探索编译器，并充满信心地这样做。

# 进一步阅读

编程语言的内存模型是一个广泛的话题。截至撰写本书时，Rust 的内存模型必须通过检查 Rust 文档、rustc 源代码和对 LLVM 的研究来理解。也就是说，Rust 的内存模型没有正式的文档，尽管社区中有人提出要提供它。独立于这一点，对于工作的程序员来说，理解底层机器也很重要。有大量的材料需要覆盖。

这些笔记是一个小的开始，特别关注与本章最相关的 Rust 文档：

+   *高性能代码 201：混合数据结构*，作者：Chandler Carruth，可在[`www.youtube.com/watch?v=vElZc6zSIXM&index=5&list=PLKW_WLANyJtqQ6IWm3BjzHZvrFxFSooey`](https://www.youtube.com/watch?v=vElZc6zSIXM&index=5&list=PLKW_WLANyJtqQ6IWm3BjzHZvrFxFSooey)找到。实际上，这是 2016 年 CppCon 的一次演讲。Carruth 是一位富有魅力的演讲者，他是专注于编译器性能的 LLVM 团队的一员。从构建与 CPU 缓存良好交互的信息密集型数据结构的角度来看，这次演讲特别有趣。尽管演讲是用 C++进行的，但其中的技术可以直接应用于 Rust。

+   *无缓存感知算法*，作者：Matteo Frigo, Charles Leiserson, Harald Prokop, 和 Sridhar Ramachandran。本文介绍了构建无缓存感知数据结构的概念，或者说是针对具有内存层次结构的机器的本地数据结构，以及如何与它们良好交互，此外还提供了一个机器模型来分析这样的数据结构。

+   *无缓存感知算法与数据结构*，作者：Erik Demaine。这篇论文是无缓存感知领域的经典之作，基于 Frigo 等人之前提出的工作，并对现有工作进行了总结。这是一篇非常值得阅读的论文，特别是与之前的论文结合阅读。浏览其参考文献也是很有价值的。

+   *栈与堆*，可在[`doc.rust-lang.org/book/first-edition/the-stack-and-the-heap.html`](https://doc.rust-lang.org/book/first-edition/the-stack-and-the-heap.html)找到。Rust 书的第一版中的这一章解释了硬件栈和堆之间的区别，以及各自的分配，以及这对 Rust 的影响。在某些方面，这一章已经进行了更深入的探讨，但强烈推荐 Rust 书中的这一章给任何需要复习或更温和地学习的人。

+   *拆分借用*，可在[`doc.rust-lang.org/beta/nomicon/borrow-splitting.html`](https://doc.rust-lang.org/beta/nomicon/borrow-splitting.html)找到。Nomicon 是一本旨在教授低级 Rust 编程的 Rust 书籍，与这本书类似。虽然它还在不断完善中，但其中的信息非常有价值。“拆分借用”解释了新 Rust 开发者常见的一个问题：从向量或数组中执行多个可变借用。这种操作与*结构体*相关的事实常常是造成极大困惑和痛苦的原因。

+   *Rust 参考手册*，可在[`doc.rust-lang.org/reference/`](https://doc.rust-lang.org/reference/)找到。像任何成熟的编程语言一样，Rust 参考手册对于理解语言本身的微妙细节非常有价值，这些细节在多年的邮件列表和聊天中已经讨论过。当前形式的参考手册搜索起来可能有点困难——它曾经是一页长的单页文档——但希望在我们这本书付印时，情况会有所改善。

+   *闭包：能够捕获其环境的匿名函数*，可在[`doc.rust-lang.org/1.23.0/book/second-edition/ch13-01-closures.html`](http://doc.rust-lang.org/1.23.0/book/second-edition/ch13-01-closures.html)找到。Rust 闭包对新手 Rust 开发者来说有一些微妙的影响，可能难以内化。在 Rust Book 的第二版中，这一章节在这方面做得非常出色。

+   *外部块*，可在[`doc.rust-lang.org/reference/items/external-blocks.html`](https://doc.rust-lang.org/reference/items/external-blocks.html)找到。在这本书中，外部块相对较少——也许在大多数你可能会看到的 Rust 代码中也是如此——但确实有一些可用。了解这份文档的存在是非常有价值的。

+   *黑客的乐趣*，亨利·沃伦小儿子著。这本书是经典之作。书中许多技巧现在作为一些芯片（如 x86）上的简单指令可用，但你仍会在`rustc`源代码中偶尔发现一些惊喜，尤其是`swap_nonoverlapping_bytes`技巧。
