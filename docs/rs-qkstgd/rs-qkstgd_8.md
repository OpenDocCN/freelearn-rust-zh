# 第八章：重要标准特质

正如我们之前所看到的，特质是 Rust 生态系统的重要组成部分。Rust 标准库中内置的特质影响了许多事物，包括甚至可以在特定数据值上使用的运算符。在本章中，我们将回顾许多这些特质，并了解如何在我们的数据类型上实现它们。

在本章中，我们将做以下事情：

+   查看由 Rust 标准库定义的特质集合

+   了解这些特质的含义和影响

+   了解哪些特质是自动应用的

+   学习如何使用 `derive` 命令为选定的特质生成特质实现

+   学习如何手动实现剩余的特质

# 可以推导的特质

对于某些特质，编译器本身知道如何为类型实现它们。如果我们想要它们，我们只需要告诉它，然后它会为我们处理其余的事情。

我们仍然可以选择手动实现可推导的特质，但这通常只是浪费时间。

告诉编译器我们希望数据类型具有可推导的特质是很容易的。

在这里，我们告诉它我们希望我们的 `CopyExample` 枚举实现 `Copy` 和 `Clone`：

```rs
#[derive(Copy, Clone)]
pub enum CopyExample {
    Good,
    Bad,
}
```

一个特质只有在创建特质的那些人能够编写一个生成特质实现的程序时才能被推导。当我们编写 `#[derive(Copy, Clone)]` 时，我们告诉编译器在定义特质的包的源代码中查找这些程序，以推导 `Copy` 和 `Clone`，并在继续编译之前运行这些程序来生成特质实现的源代码。如果实现特质所需做出的决策对于没有用户输入的程序来说过于复杂，则特质不能被推导。

# Clone

`Clone` 特质意味着可以显式地复制数据值。编译器永远不会自动这样做，但当我们想要复制一个值时，我们可以通过调用它的 `Clone` 函数来实现。

推导 `Clone` 特质的样式如下：

```rs
#[derive(Clone)]
pub enum CloneExample {
    Good,
    Bad,
}
```

# Copy

`Copy` 特质意味着创建数据值的副本只是复制构成其表示的位。如果数据值包含任何借用或使用堆内存，它就不能有 `Copy` 特质。

当编译器在其他情况下会移动数据值时，它会自动复制具有 `Copy` 特质的数据值。

由于任何具有 `Copy` 特质的对象都可以在请求时肯定被复制，因此 `Copy` 需要 `Clone` 被实现。

推导 `Copy` 的样子如下：

```rs
#[derive(Copy, Clone)]
pub enum CopyExample {
    Good,
    Bad,
}
```

# Debug

`Debug` 特质告诉 Rust 如何格式化数据值以供调试输出。这种用法的一个例子是，如果我们使用 `{:?}` 而不是 `{}` 作为 `println!` 或 `print!` 中数据值的替换标记。

由于数据值的调试表示应该非常接近它在源代码中的表示方式，Rust 能够自动为我们推导它。

推导 Debug 看起来像这样：

```rs
#[derive(Debug)]
pub enum DebugExample {
    Good,
    Bad,
}
```

# PartialEq

PartialEq 特性表示比较数据值以确定它们是否相等的能力。然而，它并不表示一个值被认为是等于它自己。

PartialEq 特性被编译器用来实现`==`比较操作。

浮点数是具有 PartialEq 的经典数据类型示例，因为 NaN（非数字）值的浮点表示被认为不等于它自己。

推导 PartialEq 看起来像这样：

```rs
#[derive(PartialEq)]
pub enum PartialEqSelf {
    Good,
    Bad,
}
```

然而，那个推导只比较了两个 PartialEqSelf 数据值。如果我们想启用与其他类型的数据值的相等比较，我们需要手动实现该特性。

这里，我们有一个手动实现的特性，允许与 u32 数据类型进行比较：

```rs
pub enum PartialEqU32 {
    Good,
    Bad,
}

impl PartialEq<u32> for PartialEqU32 {
    fn eq(&self, other: &u32) -> bool {
        match self {
            PartialEqU32::Good => other % 2 == 0,
            PartialEqU32::Bad => other % 2 == 1,
        }
    }
}
```

在这里，我们已经安排了 PartialEqU32::Good 值与偶数 u32 进行比较视为相等，而 PartialEqU32::Bad 与奇数 u32 进行比较视为相等。

# Eq

Eq 特性意味着与 PartialEq 相同的意思，*除了*数据值总是等于它自己。

实现 Eq 特性需要同时实现 PartialEq 特性，并且它所做的唯一超出 PartialEq 的事情就是向编译器提供提示，即当比较的两边都是相同的数据值时，不需要麻烦地运行 Eq 函数。

推导 Eq 看起来像这样：

```rs
#[derive(Eq, PartialEq)]
pub enum EqExample {
    Good,
    Bad,
}
```

# PartialOrd

PartialOrd 特性表示在两个数据值之间定义某种排序能力，因此我们可以说出一个小于另一个，或大于另一个，或者它们是相同的，*或者*排序关系不适用于这些值。这就是为什么这是一个*部分*排序的原因。

由于“它们相同”是比较的有效结果，实现 PartialOrd 需要实现 PartialEq。

与 PartialEq 一样，我们可以推导出一个实现，用于比较相同类型的数据值，但我们也可以手动实现特性以允许与不同类型的数据比较。

这里，我们有特性的自动推导：

```rs
#[derive(PartialOrd, PartialEq)]
pub enum PartialOrdSelf {
    Good,
    Bad,
}
```

这里，我们手动实现了它以支持与不同数据类型的比较：

```rs
pub enum PartialOrdU32 {
 Good,
 Bad,
}

impl PartialEq<u32> for PartialOrdU32 {
    fn eq(&self, _other: &u32) -> bool {
        false
    }
}

impl PartialOrd<u32> for PartialOrdU32 {
    fn partial_cmp(&self, _other: &u32) -> Option<Ordering> {
        match self {
            PartialOrdU32::Good => Some(Ordering::Greater),
            PartialOrdU32::Bad => None,
        }
    }
}
```

这里，我们告诉 Rust，PartialOrdU32::Good 值总是大于任何 u32 值，但 PartialOrdU32::Bad 值与任何 u32 值都没有任何关系。

# Ord

`Ord`类似于`PartialOrd`，除了它不允许返回“无关系”的选项；对于任何一对值，要么它们相等，要么一个小于另一个。

`Ord`被编译器用来实现`<`（小于）、`>`（大于）、`<=`（小于等于）、`>=`（大于等于）比较运算符。

如果一个数据类型具有`Ord`特质，它也必须具有`PartialOrd`、`Eq`和`PartialEq`特质。像这些特质一样，它们可以手动实现以启用不同数据类型之间的比较，但我们必须非常小心，确保用于实现这些特质的各个函数返回的结果是一致的。当我们推导特质时，我们不必担心这一点。

这里是推导`Ord`的一个示例：

```rs
#[derive(Ord, Eq, PartialOrd, PartialEq)]
pub enum OrdExample {
    Good,
    Bad,
}
```

# 哈希

`Hash`特质使数据值能够用作 Rust 标准库中几个数据结构的键，例如`HashMap`和`HashSet`。

推导`Hash`特质看起来像这样：

```rs
#[derive(Hash)]
pub enum HashExample {
    Good,
    Bad,
}
```

虽然`Eq`和`PartialEq`实际上不是实现`Hash`所必需的，但如果它们被实现，它们需要与之一致，也就是说，如果两个值相等，它们的哈希值也应该相等。自动生成的实现具有这个属性，所以我们只有在进行手动实现时才需要担心它。

# 默认

当为类型实现`Default`特质时，它使我们能够请求该类型数据的默认值。

当我们为一个数据类型推导`Default`时，它将该类型的默认值设置为包含所有包含数据类型的默认值，因此当我们这样做时：

```rs
#[derive(Default)]
pub struct DefaultExample {
    name: String,
    value: i32,
}
```

我们所做的是将`DefaultExample`类型的默认值设置为包含`String`和`i32`的默认值的`DefaultExample`。

我们可以这样请求一个默认值：

```rs
let x: DefaultExample = Default::default();
println!("Default String is {:?}, default i32 is {:?}", x.name, x.value);
```

# 使操作员具备特性的特质

大多数运算符和 Rust 的特殊语法都有特质的支撑，这些特质告诉编译器如何对特定数据类型执行操作。我们已经看到了一些，但其中许多不能被推导，所以如果我们想为我们自己的数据类型启用这种语法，我们需要手动实现它们。

# 添加、乘法、减法和除法

`Add`、`Mul`、`Sub`和`Div`特质代表了对两个值进行加、乘、减或除的能力。这些特质被编译器用来实现`+`、`*`、`-`和`/`运算符。

注意，如果`self`和`other`的值不具有`Copy`特质，它们将被移动到实现函数中并被消耗。

所有这些特质遵循相同的模式，所以这里是一个`Add`的实现示例：

```rs
pub enum AddExample {
    One,
    Two,
    Three,
    Many,
}

impl Add for AddExample {
    type Output = AddExample;

    fn add(self, other: AddExample) -> AddExample {
        match (self, other) {
            (AddExample::One, AddExample::One) => AddExample::Two,
            (AddExample::One, AddExample::Two) => AddExample::Three,
            (AddExample::Two, AddExample::One) => AddExample::Three,
            _ => AddExample::Many,
        }
    }
}
```

`Mul`、`Sub`和`Div`遵循相同的模式。

我们在这里定义的是一种非常原始的计数形式，它将任何大于 3 的数字视为“很多”。

在 `impl` 块内部，我们有 `type Output = AddExample;`。这是一种我们之前没有见过的语法。我们所做的是为这个实现设置 `Output` *关联类型*，它被反馈到特性定义中，用于声明 `add` 函数的签名。毕竟，我们在这里返回一个 `AddExample`，而在特性最初定义时并没有这样的类型。但这不是问题，因为特性说明 `add` 函数返回一个类型为 `Output` 的数据值，而我们刚刚告诉它 `Output` 是 `AddExample` 的别名。

我们还可以通过实现 `Add<OtherType> for OneType` 来实现将两种不同类型的数据相加，这表示在 `+` 的左侧有一个 `OneType` 值，在右侧有一个 `OtherType` 值，这与我们在本章前面能够创建两种不同类型之间比较的方式类似。同样的技巧也适用于 `Mul`、`Sub` 和 `Div`。

# AddAssign, MulAssign, SubAssign, and DivAssign

这些特性为实现了它们的类型启用了 `+=`、`*=`、`-=` 和 `/=` 操作符。

它们类似于 `Add`、`Sub`、`Mul` 和 `Div` 特性，不同之处在于它们的实现函数接受 `&mut self` 而不是普通的 `self`。它们不是消耗它们的左侧输入，而是有改变其包含值的能力。

所有这些特性都遵循相同的模式，所以这里是一个 `AddAssign` 的示例实现：

```rs
pub enum AddExample {
 One,
 Two,
 Three,
 Many,
}

impl AddAssign for AddExample {
    fn add_assign(&mut self, other: AddExample) {
        *self = match (&self, other) {
            (AddExample::One, AddExample::One) => AddExample::Two,
            (AddExample::One, AddExample::Two) => AddExample::Three,
            (AddExample::Two, AddExample::One) => AddExample::Three,
            _ => AddExample::Many,
        };
    }
}
```

除了基于将新值赋给 `&mut self` 的差异之外，这和 `Add` 特性的 `add` 函数实现非常相似，这并不令人惊讶。

特别是，虽然它不消耗它的 `self`，但它仍然消耗操作符右侧的值，假设这个值没有 `Copy` 特性。

# BitAnd

`BitAnd` 特性为实现了它的类型启用了 `&` 操作符。这个操作符用于计算两个整数的 *按位与* 值（因此得名），但对于各种其他数据类型有不同的含义。

实现 `BitAnd` 的样子如下：

```rs
pub enum BitExample {
    Yes,
    No,
}

impl BitAnd for BitExample {
    type Output = BitExample;

 fn bitand(self, other: BitExample) -> BitExample {
 match (self, other) {
 (BitExample::Yes, BitExample::Yes) => BitExample::Yes,
            (BitExample::No, BitExample::Yes) => BitExample::No,
            (BitExample::Yes, BitExample::No) => BitExample::No,
            (BitExample::No, BitExample::No) => BitExample::No,
        }
    }
}
```

# BitAndAssign

`BitAndAssign` 特性为实现了它的数据类型启用了 `&=` 操作符。

实现 `BitAndAssign` 的样子如下：

```rs
pub enum BitExample {
 Yes,
 No,
}

impl BitAndAssign for BitExample {
    fn bitand_assign(&mut self, other: BitExample) {
        *self = match (&self, other) {
            (BitExample::Yes, BitExample::Yes) => BitExample::Yes,
            (BitExample::No, BitExample::Yes) => BitExample::No,
            (BitExample::Yes, BitExample::No) => BitExample::No,
```

```rs
            (BitExample::No, BitExample::No) => BitExample::No,
        };
    }
}
```

# BitOr

`BitOr` 特性为实现了它的类型启用了 `|` 操作符。这个操作符用于计算两个整数的 *按位或* 值，但对于各种其他数据类型有不同的含义。

实现 `BitOr` 的样子如下：

```rs
pub enum BitExample {
    Yes,
    No,
}
impl BitOr for BitExample {
 type Output = BitExample;

 fn bitor(self, other: BitExample) -> BitExample {
        match (self, other) {
            (BitExample::Yes, BitExample::Yes) => BitExample::Yes,
            (BitExample::No, BitExample::Yes) => BitExample::Yes,
            (BitExample::Yes, BitExample::No) => BitExample::Yes,
            (BitExample::No, BitExample::No) => BitExample::No,
        }
    }
}
```

# BitOrAssign

`BitOrAssign` 特性为实现了它的数据类型启用了 `|=` 操作符。

实现 `BitOrAssign` 的样子如下：

```rs
pub enum BitExample {
 Yes,
 No,
}

impl BitOrAssign for BitExample {
    fn bitor_assign(&mut self, other: BitExample) {
        *self = match (&self, other) {
            (BitExample::Yes, BitExample::Yes) => BitExample::Yes,
            (BitExample::No, BitExample::Yes) => BitExample::Yes,
            (BitExample::Yes, BitExample::No) => BitExample::Yes,
            (BitExample::No, BitExample::No) => BitExample::No,
        };
    }
}
```

# BitXor

`BitXor` 特性为实现了它的类型启用了 `^` 操作符。这个操作符用于计算两个整数的 *按位异或* 值，但对于各种其他数据类型有不同的含义。

实现 `BitXor` 的样子如下：

```rs
pub enum BitExample {
    Yes,
    No,
}

impl BitXor for BitExample {
 type Output = BitExample;

    fn bitxor(self, other: BitExample) -> BitExample {
        match (self, other) {
            (BitExample::Yes, BitExample::Yes) => BitExample::No,
            (BitExample::No, BitExample::Yes) => BitExample::Yes,
            (BitExample::Yes, BitExample::No) => BitExample::Yes,
            (BitExample::No, BitExample::No) => BitExample::No,
        }
    }
}
```

# BitXorAssign

`BitXorAssign`特质为实现了它的数据类型启用了`^=`运算符。

实现`BitXorAssign`看起来是这样的：

```rs
pub enum BitExample {
 Yes,
 No,
}

impl BitXorAssign for BitExample {
    fn bitxor_assign(&mut self, other: BitExample) {
        *self = match (&self, other) {
            (BitExample::Yes, BitExample::Yes) => BitExample::No,
            (BitExample::No, BitExample::Yes) => BitExample::Yes,
            (BitExample::Yes, BitExample::No) => BitExample::Yes,
            (BitExample::No, BitExample::No) => BitExample::No,
        };
    }
}
```

# Deref

`Deref`特质赋予了将值解引用为借用一样的功能。智能指针实现了这个特质，这就是为什么它们可以用作包含数据值的借用。`String`也做了同样的事情，这就是为什么我们可以在期望`&str`的地方使用`String`值。

这里是`Deref`特质的实现：

```rs
pub struct DerefExample {
 val: u32,
}

impl Deref for DerefExample {
    type Target = u32;

    fn deref(&self) -> &u32 {
        return &self.val;
    }
}
```

注意到实现函数实际上并没有解引用任何东西。相反，它将一个`&self`借用转换成了对其他东西的借用。

这正是编译器需要的东西，以便正确且高效地处理解引用智能指针等，但编译器也使用这种能力让我们能够像处理普通的`&str`一样与`Rc<Box<String>>`这样的东西交互。`Rc`有一个返回`Box`借用的`deref`函数，`Box`有一个返回`String`借用的`deref`函数，而`String`有一个返回`str`借用的`deref`函数，因此编译器允许我们将整个结构视为`&str`，以便调用其函数或将其用作参数。

# DerefMut

`DerefMut`特质与`Deref`做的是同样的事情，但它用于解引用可变值。编译器决定使用`Deref`还是`DerefMut`，所以通常当我们需要实现一个时，我们通常需要实现两个。

这里是`DerefMut`的实现：

```rs
pub struct DerefExample {
    val: u32,
}

impl DerefMut for DerefExample {
    fn deref_mut(&mut self) -> &mut u32 {
        return &mut self.val;
    }
}
```

`DerefMut`特质要求也实现了`Deref`特质，并且`deref`和`deref_mut`函数有相同的返回类型。

# Drop

当一个数据类型具有`Drop`特质时，程序将在该类型值的生命周期结束之前立即调用`drop`函数。这就是`Rc`、`Mutex`、`RefCell`等能够跟踪它们包含的值有多少借用的方式。

`drop`函数在数据值的生命周期结束之前被调用，所以我们不必担心它是一个无效的引用。此外，我们也不必担心手动清理我们数据类型中包含的值，因为它们将在我们的`drop`函数完成后自动丢弃。我们唯一需要做的就是处理导致我们最初实现`Drop`的特殊情况。

我们不能直接调用`drop`函数，因为这会是一个制造混乱的极好方式。有一个`std::mem::drop`函数我们可以使用，它会消耗一个数据值并将其丢弃，如果我们需要在特定时间触发这个操作。

实现`Drop`看起来是这样的：

```rs
pub enum DropExample {
 Good,
 Bad,
}

impl Drop for DropExample {
    fn drop(&mut self) {
        match self {
            DropExample::Good => println!("Good DropExample dropped"),
            DropExample::Bad => println!("Bad DropExample dropped"),
        };
    }
}
```

# Index

`Index`特质意味着数据类型可以用`x[y]`语法使用，其中根据索引值`y`在`x`内部查找一个值。

当我们实现 `Index` 时，我们需要确定可以用于索引值的什么数据类型，以及操作返回什么数据类型，因此实现看起来是这样的：

```rs
pub struct IndexExample {
 first: u32,
 second: u32,
 third: u32,
}

impl<'a> Index<&'a str> for IndexExample {
    type Output = u32;

    fn index(&self, index: &'a str) -> &u32 {
        match index {
            "first" => &self.first,
            "second" => &self.second,
            "third" => &self.third,
            _ => &0,
        }
    }
}
```

我们使用了 `&str` 作为索引的数据类型，并使用 `u32` 作为值的类型。使用 `&str` 意味着我们需要稍微注意生命周期，但这并不太糟糕。

# IndexMut

`IndexMut` 特性表示使用 `x[y] = z` 语法对包含的值进行赋值的 abilities。像 `Index` 特性一样，它允许我们通过提供一个索引值来查找包含的数据值，但它产生一个可变借用，可以用来更改它。

实现 `IndexMut` 的样子如下：

```rs
pub struct IndexExample {
 first: u32,
 second: u32,
 third: u32,
 junk: u32,
}

impl<'a> IndexMut<&'a str> for IndexExample {
    fn index_mut(&mut self, index: &'a str) -> &mut u32 {
        match index {
            "first" => &mut self.first,
            "second" => &mut self.second,
            "third" => &mut self.third,
            _ => &mut self.junk,
        }
    }
}
```

注意，我们在 `IndexExample` 结构体中添加了一个 `junk` 值。我们这样做是因为没有方法可以表示索引值不映射到一个有效的包含值；如果调用 `index_mut` 函数，它必须返回正确类型的可变借用，并且这个借用必须有足够长的生命周期。将垃圾值添加到数据结构是一种简单的方法，尽管还有其他可以节省内存的方法。

任何实现了 `IndexMut` 的类型都必须实现 `Index`，并且 `index` 和 `index_mut` 函数必须分别返回相同数据类型的借用和可变借用。

# Neg

`Neg` 特性使数据类型能够与 *一元否定* 操作符一起使用，也称为负号。当我们写 `-5` 时，我们正在将一元否定操作符应用于值 5，产生一个结果为负 5。

实现 `Neg` 的样子如下：

```rs
pub enum NegExample {
 Yes,
 No,
}

impl Neg for NegExample {
 type Output = NegExample;

 fn neg(self) -> NegExample {
 match self {
 NegExample::Yes => NegExample::No,
 NegExample::No => NegExample::Yes,
 }
 }
}
```

# Not

`Not` 特性启用了 *逻辑非* 操作符，它被写作一个 `!`。`Not` 在概念上和实际应用上都与 `Neg` 类似，但它的主要用途是布尔逻辑而不是算术。

实现 `Not` 的样子如下：

```rs
pub enum NotExample {
 True,
 False,
}

impl Not for NotExample {
    type Output = NotExample;

    fn not(self) -> NotExample {
        match self {
            NotExample::True => NotExample::False,
            NotExample::False => NotExample::True,
        }
    }
}
```

# Rem 和 RemAssign

`Rem` 特性为实现了它的类型启用了 `%` 操作符。这个操作符用于计算两个整数的 *模数*（也称为除法的余数），但对于各种其他数据类型有不同的含义。

`Rem` 特性既有一个关联的 `Output` 类型，也有通过实现 `Rem<OtherType>` 而不是仅仅 `Rem` 来在多种类型上操作的选择。

`RemAssign` 与 `Rem` 的关系类似于 `AddAssign` 与 `Add` 的关系。

# Shl 和 ShlAssign

`Shl` 特性为实现了它的类型启用了 `<<` 操作符。这个操作符用于将整数左移一定数量的位，但对于各种其他数据类型有不同的含义。

`Shl` 特性既有一个关联的输出类型，也有通过实现 `Shl<OtherType>` 而不是仅仅 `Shl` 来在多种类型上操作的选择。

`ShlAssign` 与 `Shl` 的关系类似于 `AddAssign` 与 `Add` 的关系。

# Shr 和 ShrAssign

`Shr` 特性为实现了它的类型启用了 `>>` 操作符。这个操作符用于将整数右移若干位，但对于其他各种数据类型有不同的含义。

`Shr` 特性既有 `Output` 关联类型，也有通过实现 `Shr<OtherType>` 而不是仅仅 `Shr` 来操作不同类型的选项。

`ShrAssign` 与 `Shr` 的关系类似于 `AddAssign` 与 `Add` 的关系。

# 自动实现的特性

有一些特性在适当的情况下会自动实现，甚至不需要 `#[derive()]` 标签。这些通常代表数据类型极其低级方面的特性。

# Sync

`Sync` 特性会自动应用于任何可以在线程之间安全借用数据的数据类型。

虽然我们的数据类型如果符合条件会自动具有 `Sync` 特性，但有时我们想要确保数据类型**不**具有 `Sync`，即使它看起来对编译器来说应该是这样的。

我们可以通过为我们的数据类型实现 `!Sync` 来实现这一点：

```rs
pub enum NotSyncExample {
 Good,
 Bad,
}

impl !Sync for NotSyncExample {}
```

我们实际上不需要在 `!Sync` 中实现任何函数。我们只是在告诉编译器 `Sync` 特性对于这个类型来说是不合适的。

截至 Rust 1.29 版本，实现 `!Sync` 仍然被视为一个不稳定特性，并且不在编译器的稳定构建中可用。它可以通过在文件顶部放置 `#![feature(optin_builtin_traits)]` 来在夜间构建中启用。

许多数据类型都有 `Sync` 特性，但 `Rc`、`Cell` 和 `RefCell` 是不具该特性的类型中的显著例子。`Arc`、`Mutex` 和 `RwLock` 则具有。

# Send

`Send` 特性会自动应用于任何可以在线程之间安全移动的数据类型。它是 `Sync` 的一个近亲，并且像 `Sync` 一样，我们可以实现 `!Send` 来告诉编译器一个数据类型**不应该**具有该特性。

如果我们没有明确禁止，编译器会根据它包含的类型是否具有该特性来决定一个类型是否具有 `Send` 特性。

# Sized

对于编译器已知大小的任何数据类型，`Sized` 特性都会自动应用。所有特性界限都会自动包含 `Sized` 作为额外的、隐含的要求，除非我们明确告诉它 `?Sized` 是要求。如果我们明确声明特性界限是 `?Sized`，这意味着符合界限的数据类型允许是 `Sized`，但不是必须的。

# Fn

`Fn` 特性会自动应用于任何只使用不可变借用访问其自身作用域之外数据的函数或闭包。

这是一个严格的要求，许多函数和闭包都未能通过这个测试，所以 `Fn` 是函数特性中最不常见的。

# FnMut

`FnMut` 特性会自动应用于任何使用可变或不可变借用访问其自身作用域之外数据的函数或闭包。

这是一个中等要求，但一些函数和闭包未能通过这个测试，所以 `FnMut` 比较常见于 `Fn`。

# FnOnce

`FnOnce`特质会自动应用于任何使用可变借用、不可变借用或移动变量来访问其作用域外数据的函数。

这是一个宽松的要求，任何函数或闭包都会满足，因此`FnOnce`是函数特质的常见类型。

# 摘要

在本章中，我们做了以下工作：

+   研究了许多不同的特质

+   检查了特质的特定含义以及它们如何与 Rust 的语法交互

+   学习了实现特质的细节

+   学习了如何轻松推导支持该特性的特质

我们已经到达了快速入门指南的结尾，但旅程永远不会结束。祝你在下一步好运。
