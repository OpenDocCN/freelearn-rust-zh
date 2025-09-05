# Rust 最佳实践

Rust 是一种强大的语言，但通过实践可以轻松避免的一些事情，在开始时可能会让你的生活变得非常困难。本章旨在向你展示一些良好的实践和技巧。

在本章中，我们将涵盖以下主题：

+   最佳实践

+   API 小贴士和改进

+   使用小贴士

+   代码可读性

现在让我们开始吧！

# Rust 最佳实践

让我们从一些基本（也许很明显）的事情开始。

# 切片

首先，让我们回顾一下；切片是对数组的常量视图，`&[T]` 是 `Vec<T>` 的常量视图，而 `&str` 是 `String` 的常量视图（就像 `Path` 是 `PathBuf` 的常量视图，`OsStr` 是 `OsString` 的常量视图）。现在你有了这个概念，让我们继续吧！

当一个函数期望 `Vec` 或 `String` 类型的常量参数时，总是这样写：

```rs
fn some_func(v: &[u8]) {
    // some code...
}
```

而不是：

```rs
fn some_code(v: &Vec<u8>) {
    // some code
}
```

然后：

```rs
fn some_func(s: &str) {
    // some code...
}
```

而不是：

```rs
fn some_func(s: &String) {
    // some code...
}
```

你可能想知道为什么会这样。所以，让我们想象一下你的函数以 ASCII 字符显示你的 `Vec`：

```rs
fn print_as_ascii(v: &[u8]) {
    for c in v {
        print!("{}", *c as char);
    }
    println!("");
}
```

现在你只想打印出你的 `Vec` 的一部分：

```rs
let v = b"salut!";

print_as_ascii(&v[2..]);
```

现在，如果 `print_as_ascii` 只接受 `Vec` 的引用，你就必须进行一个（无用的）分配，如下所示：

```rs
let v = b"salut!";

print_as_ascii(&v[2..].to_vec());
```

# API 小贴士和改进

当编写公共 API（无论是为你还是其他用户）时，一些小贴士真的可以让每个人的生活变得更轻松。这正是泛型发挥作用的地方。让我们从 `Option` 参数开始：

# 解释 Some 函数

通常，当一个函数期望一个 `Option` 参数时，它看起来是这样的：

```rs
fn some_func(arg: Option<&str>) {
    // some code
}
```

你可以这样调用它：

```rs
some_func(Some("ratatouille"));
some_func(None);
```

现在，如果我说你可以去掉 `Some`，怎么样？不错，对吧？实际上，这实际上非常简单：

```rs
fn some_func<'a, T: Into<Option<&'a str>>>(arg: T) {
    // some code
}
```

你现在可以这样调用它：

```rs
some_func(Some("ratatouille")); // If you *really* like to write "Some"...
some_func("ratatouille");
some_func(None);
```

更好！然而，为了让用户的生活变得更轻松，编写函数的人需要写更多的代码。你不能直接使用 `arg`；你需要添加一个额外的步骤。以前，你只是这样做：

```rs
fn some_func(arg: Option<&str>) {
    if let Some(a) = arg {
        println!("{}", a);
    } else {
        println!("nothing...");
    }
}
```

现在，你需要在能够使用 `arg` 之前添加一个 `.into` 调用：

```rs
fn some_func<'a, T: Into<Option<&'a str>>>(arg: T) {
    let arg = arg.into();
    if let Some(a) = arg {
        println!("{}", a);
    } else {
        println!("nothing...");
    }
}
```

就这样。正如我们之前所说的，这不需要太多，而且让用户的生活变得更轻松，所以为什么不这样做呢？

# 使用 Path 函数

就像上一个部分一样，这将向你展示一些小贴士，使你的 API 通过 *自动转换* 为 `Path` 来使用起来更舒适。

那么，让我们用一个接收 `Path` 作为参数的函数的例子来说明：

```rs
use std::path::Path;

fn some_func(p: &Path) {
    // some code...
}
```

这里没有什么新的。你可以像这样调用这个函数：

```rs
some_func(Path::new("tortuga.txt"));
```

这里令人烦恼的是，你必须自己构建 `Path` 然后才能将其发送到函数。这太烦人了，但我们能做得更好！

```rs
fn some_func<P: AsRef<Path>>(p: P) {
    // some code...
}
```

就这样...你现在可以这样调用函数：

```rs
some_func(Path::new("tortuga.txt")); // If you *really* like to build the "Path" by yourself...
some_func("tortuga.txt");
```

就像对于 `Into` 特质一样，你需要添加一行代码才能使其工作：

```rs
fn some_func<P: AsRef<Path>>(p: P) {
    let p: &Path = p.as_ref();
    // some code...
}
```

就这样！现在，只要给定的类型实现了 `AsRef<Path>`，你就可以这样发送。为了信息，这里是一个（非详尽的）实现了此特质的类型列表：

+   `OsStr` / `OsString`

+   `&str` / `String`

+   `Path`（是的，`Path` 也实现了 `AsRef<Path>`！）/ `PathBuf`

+   `Iter`

这已经很多了，所以你应该能够很容易地做到这一点！

# 使用技巧

现在你已经看到了一些关于如何通过一些小技巧使用户的代码更美观的例子，那么我们来看看可能使 *你的* 代码更好的其他一些事情。

# 构建者模式

构建者模式旨在能够通过多个可以链式调用的调用来 *构建* 一个最终对象。Rust 标准库中的 `OpenOptions` 类型是一个很好的例子。

当你需要玩转 `File` 时，强烈建议你使用 `OpenOptions`！

```rs
use std::fs::OpenOptions;

let file = OpenOptions::new()
                       .read(true)
                       .write(true)
                       .create(true)
                       .open("foo.txt");
```

要创建这样的 API，你有两种方法：

+   玩转可变借用

+   玩转移动语义

让我们从可变借用开始！

# 玩转可变借用

第一个例子工作方式与 `OpenOptions` 相同：

```rs
struct Number(u32);

impl Number {
    fn new(nb: u32) -> Number {
        Number(nb)
    }

    fn add(&mut self, other: u32) -> &mut Number {
        self.0 += other;
        self
    }

    fn sub(&mut self, other: u32) -> &mut Number {
        self.0 -= other;
        self
    }

    fn compute(&self) -> u32 {
        self.0
    }
}
```

如果你想知道 `self.0`，只需记住这是你访问元组字段的方式。

然后，你可以按照以下方式调用它：

```rs
let nb = Number::new(0).add(10).sub(5).add(12).compute();
assert_eq!(nb, 17);
```

这是做这件事的第一种方法。

你会注意到你需要添加一个 *结束* 方法，这样你就可以将你的可变借用转换为对象（否则，你将有一个借用问题）。

让我们现在看看做这件事的第二种方法！

# 玩转移动语义

我们不再每次都取 `&mut`，而是每次直接获取对象的拥有权：

```rs
struct Number(u32);

impl Number {
    fn new(nb: u32) -> Number {
        Number(nb)
    }

    fn add(mut self, other: u32) -> Number {
        self.0 += other;
        self
    }

    fn sub(mut self, other: u32) -> Number {
        self.0 -= other;
        self
    }
}
```

然后，就不再需要 *结束* 方法了：

```rs
let nb = Number::new(0).add(10).sub(5).add(12);
assert_eq!(nb.0, 17);
```

我通常更喜欢这种方式来做构建者模式，但这更多的是个人意见，而不是经过深思熟虑的决定。选择在你所处的情境中看起来最合适的方式！

# 代码可读性

现在，我们将讨论 Rust 的语法本身。一些事情可以提高代码的可读性，并且很重要。让我们从大数字开始。

# 大数字格式化

在代码中看到巨大的常量数字并不罕见，例如：

```rs
let x = 1000000000;
```

然而，这对我们来说（人类大脑在解析这样的数字方面效率不高）相当难以阅读。在 Rust 中，你可以在数字中插入 `_` 字符而不会出现任何问题：

```rs
let x = 1_000_000_000;
```

这已经很好了，对吧？

# 指定类型

Rust 编译器在大多数情况下可以自动检测变量的类型。然而，对于阅读代码的人来说，并不总是很明显代码返回了什么。举个例子？当然可以！

```rs
let x = "a 10 11 coucou 12 14".split(' ')
                              .filter_map(|e| e.parse::<u32>().ok())
                              .filter(|x| x % 2 == 0)
                              .map(|s| format!("{}", s))
                              .collect::<Vec<_>>()
                              .join("::");
```

仔细阅读代码后，你会猜到 `x` 是一个 `String`。然而，你需要阅读所有那些闭包才能得到它，即使如此，你真的确定类型吗？

在这种情况下，强烈建议你只添加类型注解：

```rs
let x: String = "a 10 11 coucou 12 14".split(' ')
                                      .filter_map(|e| e.parse::<u32>().ok())
                                      .filter(|x| x % 2 == 0)
                                      .map(|s| format!("{}", s))
                                      .collect::<Vec<_>>()
                                      .join("::");
```

这并不花费太多，并且允许读者（包括你自己）更快地阅读代码。

# 匹配

在 Rust 中，通常使用 `match` 块通过模式匹配。然而，使用 `if let` 条件通常是一个更好的解决方案。让我们用一个简单的例子来说明：

```rs
enum SomeEnum {
    Ok,
    Err,
    Unknown,
}
```

现在假设你想在得到 `Ok` 时执行一个动作。使用 `match`，你会这样做：

```rs
let x = SomeEnum::Err;

match x {
    SomeEnum::Ok => {
        // Huge code doing a lot of things...
    }
    _ => {}
}
```

这并不是一个问题，对吧？现在让我们用 `if let` 来看看：

```rs
let x = SomeEnum::Err;

if let SomeEnum::Ok = x {
    // Huge code doing a lot of things...
}
```

就这样。它基本上使代码变得更短，同时大大提高了可读性。当你只需要获取一个值时，通常使用 `if let` 而不是 `match` 是更好的解决方案。

# 摘要

通过这一章的最后部分，你应该对 Rust 中的良好实践有一个全面的了解。请记住，良好的代码易于阅读且注释详尽。即使复杂的特性，在制作良好的文档后也会变得容易理解得多。
