# 第十三章：标准库

与所有编程语言一样，Rust 提供了一个丰富的库，通过提供常用功能来简化开发者的生活，而无需反复重写相同的内容。我们在这本书中已经遇到了标准库的一部分，我毫不怀疑你已经在其他代码示例中看到了它的许多实例。

在接下来的两章中，我们将探讨库提供的内容以及如何使用它。

在本章中，我们将处理标准包（`std::`）。

# 章节格式

与本书中的其他章节不同，由于库的规模庞大，本章将略有不同。它看起来像这样：

*特型名称*

*它做什么/提供什么*

*笔记*

*提供的特性和结构体/枚举*

*下载示例*

由于 Rust 也有两个主要变体（稳定版和不稳定版），我不会涵盖库中目前被分类为不稳定的内容；没有保证它将保留在库中或保持不变。

# 标准库是什么？

标准库包含 Rust 的核心功能。它分为四个部分：

+   标准模块

+   原始类型

+   宏

+   预设

# 标准模块（概述）

标准模块实现了字符串处理、IO、网络和操作系统调用等功能。总共有大约 60 个这样的模块。有些是自包含的，而有些为特性和结构体提供了实现。

模块名称可能与原始类型（如 `i32`）同名，这可能会引起一些混淆。

# 原始类型（概述）

原始类型是我们提供的类型。在其他语言中，它们可能是 `int`、`float` 和 `char`。在 Rust 中，我们有 `i32`、`d32` 和 `i8`（分别）。Rust 为开发者提供了 19 个原始类型，其中一些将提供额外的实现。

# 宏（概述）

宏在 Rust 应用程序开发中起着重要作用；它们被设计用来提供许多非常方便的快捷方式，以避免实现常见功能（如 `println!(...)` 和 `format!(...)`）的痛苦。Rust 提供了 30 个宏。

# 预设

预设（Prelude）非常有用。你可能想知道为什么这本书中的许多示例都使用了标准模块，但你很少在源文件的顶部看到 `use std::`。原因是 Rust 会自动将预设模块注入到每个源文件中，为源文件提供许多核心模块。它按无特定顺序插入以下内容：

+   `std::marker::{Copy, Send, Sized, Sync}`

+   `std::ops::{Drop, Fn, FnMut, FnOnce}`

+   `std::mem::drop`

+   `std::boxed::Box`

+   `std::borrow::ToOwned`

+   `std::clone::Clone`

+   `std::cmp::{PartialEq, PartialOrd, Eq, Ord }`

+   `std::convert::{AsRef, AsMut, Into, From}`

+   `std::default::Default`

+   `std::iter::{Iterator, Extend, IntoIterator, DoubleEndedIterator, ExactSizeIterator}`

+   `std::option::Option::{self, Some, None}`

+   `std::result::Result::{self, Ok, Err}`

+   std::slice::SliceConcatExt

+   std::string::{String, ToString}

+   `std::vec::Vec`

它将 `extern crate std;` 插入到每个包中，并将 `use std::prelude::v1::*;` 插入到每个模块中。这就是预览所需的所有内容——就这么简单！然而，每个模块都将依次处理。

# 标准模块

完成概述后，让我们看看标准模块。

# std::Any

此模块通过运行时反射启用 `'static` 的动态转换。

它可以用来获取一个 `TypeId`。当用作借用特质引用 (`&Any`) 时，它可以用来确定值是否是给定类型（使用 `Is`），也可以用来获取内部值的引用作为类型（使用 `downcast_ref`）。`&mut Any` 将允许访问 `downcast_mut`，它获取内部值的可变引用。`&Any` 只能用于测试特定类型，不能用来测试类型是否实现了特质。

**结构体**

+   `TypeId`: `TypeId` 是一个无法检查的不可见对象，但允许进行克隆、比较、打印和显示。仅适用于使用 `'static` 的类型。

**实现**

+   `of<T>() -> TypeId where T:’static + Reflect + ?Sized`: 返回函数实例化的类型 `T` 的 `TypeId`

**特质**

+   `pub trait Any: 'static + Reflect {fn get_type_id(&self) -> TypeId;}`: 模拟动态类型。

**特质方法**

+   `impl Any + 'static`

    +   `is<T>(&self) -> bool where T:Any`: 如果装箱类型与 `T` 相同，则返回 `true`

    +   `downcast_ref<T>(&self) -> Option<&T> where T:Any`: 返回 `ref` 到装箱值，无论它是否为类型 `T` 或 `None`

    +   `downcast_mut<T>(&mut self) -> Option<&mut T> where T:Any`: 对于 `downcast_ref` 但返回一个可变引用或 `None`

+   `impl Any + 'static + Send`

    +   `is<T>(&self) -> bool where T:Any`: 将发送到在类型 `Any` 上定义的方法

    +   `downcast_ref<T>(&self) -> Option<&T> where T:Any`: 将发送到在类型 `Any` 上定义的方法

    +   `downcast_mut<T>(&mut self) -> Option<&mut T> where T:Any`: 将发送到在类型 `Any` 上定义的方法

**特质实现**

+   `impl Debug for Any + ‘static`

    +   `fmt(&self, f: &mut Formatter) -> Result<(), Error>`: 使用格式化器格式化值

+   `impl Debug for Any + ‘static + Send`

    +   `fmt(&self, f: &mut Formatter) -> Result<(), Error>`: 发送到 `Debug` 方法上定义的方法

# std::ascii

此模块对 ASCII 字符串执行操作。

`AsciiExt` 特质包含许多有用的字符串切片实用工具，用于测试，以及转换为大写和小写。

**结构体**

+   `pub struct EscapeDefault`: 迭代字节的逃逸版本

+   `impl` 迭代器为 `EscapeDefault`

    +   `type Item = u8`: 迭代器遍历的元素类型

+   `impl` 迭代器为 `EscapeDefault` 函数

    +   `next(&mut self) -> Option<u8>`: 前进迭代器并返回下一个值

    +   `size_hint(&self) -> (usize, Option<usize>)`: 返回迭代器剩余长度的界限

    +   `count(self) -> usize`: 返回迭代次数

    +   `last(self) -> Option<Self::Item>`: 返回最后一个元素

    +   `nth(&mut self, n:usize) -> Option<Self::Item>`: 返回第 n 个位置之后的下一个元素

    +   `chain<U>(self, other:U) -> Chain<Self, U::IntoIterator> where U: IntoIterator<Item=Self::Item>`: 取两个迭代器并按顺序创建一个新的迭代器

    +   `zip<U>(self, other: U) -> Zip<Self, U:IntoIterator> where U:IntoIterator`: 取两个迭代器并将它们合并成一个单一的对迭代器

    +   `map<T,U>(self, u: U) -> Map<Self, U> where U:FnMut(Self::Item) -> T`: 从闭包创建迭代器，该闭包对每个元素调用该闭包

    +   `filter<F>(self, predicate: F) -> Filter<Self, F> where F: FnMut(&Self::Item) -> bool`: 创建一个使用闭包确定是否返回元素的迭代器

    +   `enumerate(self) -> Enumerate<Self>`: 提供当前迭代计数和下一个值

    +   `peekable(self) -> Peekable<Self>`: 查看下一个值而不消耗迭代器

    +   `skip_while<P>(self, predicate:P) -> SkipWhile<Self, P> where P:FnMut(&Self::Item) -> bool`: 创建一个基于谓词跳过 *n* 个元素的迭代器

    +   `take_while<P>(self, predicate:P) -> TakeWhile<Self, P> where P:FnMut(&Self::Item) -> bool`: 创建一个基于谓词的迭代器，它产生元素。

    +   `skip(self, n: usize) -> Skip<Self>`: 跳过前 *n* 个元素

    +   `take(self, n: usize) -> Take<Self>`: 产生前 *n* 个元素的迭代器

    +   `scan<S, T, U>(self, interal_state: S, u: U) -> Scan<Self, S, U> where U:FnMut(&mut S, Self::Item)-> Option<T>`: 持有内部状态并产生新迭代器的迭代器适配器

    +   `flat_map<T, U>(self, u:U) -> Flat_Map<Self, T, U> where U:FnMut(Self::Item) -> T, T:IntoIterator`: 创建一个像 map 一样工作的迭代器，但产生一个扁平、嵌套的结构

    +   `fuse(self)->Fuse(Self)`: 在遇到第一个 `None` 实例后终止的迭代器

    +   `inspect<T>(self, t: T)->Insepect<Self, T> where T: FnMut(&self::Item)->()`: 对每个迭代的元素执行一些操作并将值传递下去

    +   `by_ref(&mut self) -> &mut Self`: 从迭代器中借用而不是消耗它

    +   `collect<T>(self) -> T where T:FromIterator(Self::Item)`: 从迭代器创建集合

    +   `partition<T, U>(self, u:U) -> (T,T) where T:Default + Extend<Self::Item>, U:FnMut(&Self::Item> -> bool`: 从迭代器中创建两个集合

    +   `fold<T, U>(self, init:T, u:U)->T where U:FnMut(T, Self::Item) -> T`: 应用函数以产生单个最终结果的迭代器适配器

    +   `all<T>(&mut self, t:T) -> bool where T:FnMut(Self::Item) -> bool`: 测试迭代器中的所有元素是否匹配谓词 `T`

    +   `any<T>(&mut self, t:T) -> bool where T:FnMut(Self::Item) -> bool`: 测试迭代器中的任何元素是否匹配谓词 `T`

    +   `find<T>(&mut self, predicate:T) -> Option<Self::Item> where T: FnMut(&Self::Item) -> bool`: 在迭代器中搜索与谓词匹配的元素

    +   `position<T>(&mut self, predicate:T) -> Option<usize> where T:FnMut(Self::Item) -> bool`: 在迭代器中搜索与谓词匹配的元素并返回索引

    +   `rposition<T>(&mut self, predicate:T) -> Option<usize> where T:FnMut(Self::Item) -> bool, Self:ExtractSizeIterator + doubleEndedIterator`: 与`position`类似，但从右侧搜索

    +   `max(self_ => Option<Self::Item>`: 返回迭代器的最大元素

    +   `min(self_ => Option<Self::Item>`: 返回迭代器的最小元素

    +   `rev(self) -> Rev<Self> where Self:DoubleEndedIterator`: 反转迭代器的方向

    +   `unzip<T, U, FromT, FromU>(self) -> (FromT, FromU) -> Where FromT: Default + Extend<T>, FromU: Default + Extend<U>, Self::Iterator<Item=(T,U)>`: 执行 ZIP 的逆操作（从单个迭代器获取两个集合）

    +   `cloned<'a, Y>(self) -> Cloned<Self> where Self:Iterator<Item = &'a T>, T: 'a + Clone`: 创建一个迭代器，克隆其所有元素

    +   `cycle(self) -> Cycle<Self> where Self:Clone`: 无限重复迭代器

    +   `sum<T>(self) -> T where Y:Add<Self::Item, Output=T> + Zero`: 返回迭代器元素的累加和

    +   `Product<T>(self) -> T where T: Mul<Self::Item, Output = T> + One`: 将迭代器的元素相乘并返回结果

+   `impl DoubleEndedIterator for EscapeDefault`

    +   `next_back(&mut self) -> Option<u8>`: 能够从两端产生结果的迭代器

+   `impl ExactSizeIterator for EscapeDefault`

    +   `Len(&self) -> usize`: 返回迭代器迭代的次数

**特质**

```rs
pub trait AsciiExt {
  type Owned;
  fn is_ascii(&self) -> bool ;
  fn to_ascii_uppercase(&self) -> Self:: Owned ;
  fn to_ascii_lowercase(&self) -> Self:: Owned ;
  fn eq_ignore_ascii_case(&self, other: &Self) -> bool ;
  fn make_ascii_uppercase(&mut self);
  fn make_ascii_lowercase(&mut self);
}
```

以下是对字符串切片进行 ASCII 子集操作的扩展方法：

+   关联类型

    +   `Owned:` 复制 ASCII 字符的容器。

+   必要方法

    +   `is_ascii(&self) -> bool`: 值是否为 ASCII 值

    +   `to_ascii_uppercase(&self) -> Self::Owned`: 将字符串复制为 ASCII 大写形式

    +   `to_ascii_lowercase(&self) -> Self::Owned`: 与大写类似，但为小写

    +   `eq_ignore_ascii_case(&self, other: &Self) -> bool`: 两个字符串在不考虑大小写的情况下是否相同

# `std::borrow`

此模块用于处理借用数据。

`enum `Cow`` (写时复制智能指针)

`Cow`允许对借用数据进行不可变访问（并且可以包含此数据），并在需要修改或拥有时允许懒惰地克隆。它设计为使用`Borrow`特质。它还实现了`Deref`，这将允许访问`Cow`包含的数据上的非修改方法。`to_mut`将提供对拥有值的可变引用。

+   `Trait std::borrow::Borrow`: 数据可以通过多种方式借用：共享借用（`T`和`&T`）、可变借用（`&mut T`）以及从`Vec<T>`（`&[T]`和`&mut[T]`）等类似类型借用的切片。

    `Borrow`特质提供了一个方便的方法来抽象给定类型。例如：`T: Borrow<U>`意味着从`&T`借用了`&U`

    +   `fn borrow(&self) -> &Borrowed`: 从拥有值不可变借用

+   `Trait std::borrow::BorrowMut`: 用于可变借用数据

    +   `fn borrow_mut(&mut self) -> &mut Borrowed`: 从拥有值可变借用

+   `Trait std::borrow:ToOwned`: `Clone` 的借用数据泛化。`Clone` 仅在从 `&T` 到 `T` 转换时工作。`ToOwned` 将 `Clone` 泛化以从任何给定类型的任何借用中构建拥有数据。

    +   `fn to_owned(&self) -> Self::Owned`: 从借用数据创建拥有数据

# std::boxed

此模块用于堆分配。

一种非常简单的方法来在堆上分配内存、提供所有权并在作用域之外释放。

+   `impl<T> Box<T>`

    +   `fn new(x:T) -> Box<T>`: 在堆上分配内存并将 `x` 放入其中

+   `impl <T> Box<T> where T: ?Sized`

    +   `unsafe fn from_raw(raw: *mut T) -> Box<T>`: 从原始指针构建 box。创建后，指针由新的 `Box` 拥有。这非常不安全；`Box` 析构函数将调用 `T` 的析构函数并释放分配的内存。这可能导致双重释放，从而引发崩溃。

    +   `fn into_raw(b: Box<T> -> *mut T`: 消耗 box 并返回包装的原始指针。

+   `impl Box<Any + 'static>`

    +   `fn downcast<T>(self) -> Result<Box<T>, Box<Any + 'static>> where T:Any`: 尝试将 box 降级为具体类型。

+   `impl Box<Any + 'static + Send>`

    +   `fn downcast<T>(self) -> Result<Box<T>, Box<Any + ‘static + Send>> where T:Any`: 尝试将 box 降级为具体类型

方法

特征实现

+   `Impl <T> Default for Box<T> where T:Default`

    +   `fn default() -> Box<T>`: 返回类型的默认值

+   `impl<T> Default for Box<[T]>`

    +   `fn default() -> Box<T>`: 返回类型的默认值

+   `impl<T> Clone for Box<T> where T:Clone`

    +   `fn clone(&self) -> Box<T>`: 返回一个包含 box 内容副本的新 box

    +   `fn clone_from(&mut self, source: &Box<T>)`: 将 *sources* 的内容复制到 self 中而不创建新的分配

+   `impl Clone for Box<str>`

    +   `fn clone(&self) -> Box<str>`: 返回值的副本

    +   `fn clone_from(&mut self, source: &Self)`: 从 *source* 执行复制赋值

+   `impl<T> PartialEq<Box<T>> for Box<T> where T:PartialEq<T> + ?Sized`

    +   `fn eq(&self, other: &Box<T>) -> bool`: 测试 self 和 other 是否相等。由 `==` 使用

    +   `fn ne(&self, other: &Box<T>) ->`: 测试不等式。由 `!=` 使用

+   `impl<T> PartialOrd<Box<T>> for Box<T> where T:PartialOrd<T> + ?Sized`

    +   `fn partial_cmp(&self, other: &Box<T>) -> Option<Ordering>`: 如果存在，返回 self 和 other 之间的排序

    +   `fn lt(&self, other: &Box<T>) -> bool`: 测试 self 是否小于 other。由 `<` 使用

    +   `fn le(&self, other: &Box<T>) -> bool`: 测试 self 是否小于或等于 other。由 `<=` 使用

    +   `fn ge(&self, other: &Box<T>) -> bool`: 测试 self 是否大于或等于 other。由 `>=` 使用

    +   `Fn gt(&self, other: &Box<T>) -> bool`: 测试 self 是否大于 other。由 `>` 使用

+   `impl <T> Ord for Box<T> where T:Ord + ?Sized`

    +   `fn cmp(&self, other: &Box<T>) -> Ordering`: 返回 self 和 other 之间的排序

+   `impl <T> Hash for Box<T> where T: Hash + ?Sized`

    +   `fn hash<H>(&self, state: &mut H) where H: Hasher`: 将值输入到状态中，并在需要时更新哈希器

    +   `fn hash_slice<H>(data: &[Self], state &mut H) where H: Hasher`: 将此类型的切片输入到状态中

+   `impl<T> From<T> for Box<T>`

    +   `fn from(t: T) -> Box<T>`: 执行转换

+   `impl<T> Display for Box<T> where T: Display + ?Sized`

    +   `fn fmt(&self, f: &mut Formatter) -> Result<(), Error>`: 使用给定的格式化器格式化值

+   `impl<T> Debug for Box<T> where T:Debug + ?Sized`

    +   `fn fmt(&self, f: &mut Formatter) -> Result<(), Error>`: 使用给定的格式化器格式化值

+   `impl<T> Pointer for Box<T> where T: ?Sized`

    +   `fn fmt(&self, f: &mut Formatter) -> Result<(), Error>`: 使用给定的格式化器格式化值

+   `impl<T> Deref for Box<T> where T: ?Sized`

    +   `fn deref(&self) -> &T`: 解引用一个值

+   `impl<T> DerefMut for Box<T> where T: ?Sized`

    +   `fn deref_mut(&mut self) -> &mut T`: 可变解引用一个值

+   `impl<I> Iterator for Box<I> where I: Iterator + ?Sized`

    +   `fn next(&mut self) -> Option<I::Item>`: 前进迭代器并返回下一个值

    +   `fn size_hint(&self) -> (usize, Option<usize>)`: 返回迭代器剩余长度的界限

    +   `fn count(self) -> usize`: 返回迭代次数

    +   `fn last(self) -> Option<Self::Item>`: 返回最后一个元素

    +   `fn nth(&mut self, n: usize) -> Option<Self::Item>`: 消耗迭代器的*n*个元素，并返回之后的下一个元素

    +   `fn chain<U>(self, other: U) -> Chain<Self, U::Iterator> where U: IntoIterator <Item=Self::Item>`: 取两个迭代器，并按顺序创建一个新的迭代器

    +   `fn zip<U>(self, other: U) -> Zip<Self, U::IntoIter> where U: IntoIter`: 将两个迭代器合并成一个单一的对

    +   `fn map<B, F>(self, f: F) -> Map<Self, F> where F: FnMut(Self::Item) -> B`: 使用闭包创建一个迭代器，对每个元素调用该闭包

    +   `fn filter<P>(self, predicate: P) -> Filter<Self, P> where P: FnMut(&Self::Item) -> bool`: 使用闭包创建一个迭代器，以判断元素是否应该被产生

    +   `Fn filter_map<B, F>(self, f: F) -> FilterMap<Self, F> where F: FnMut(Self::Item) -> Option<B<`: 创建一个过滤并映射的迭代器

    +   `fn enumerate(self) -> Enumerate<Self>`: 创建一个迭代器，提供当前迭代计数和下一个值

    +   `fn peekable(self) -> Peekable<Self>`: 创建一个迭代器，可以查看迭代器的下一个元素而不消耗它

    +   `fn skip_while<P>(self, predicate: P)-> SkipWhile<Self, P> where P: FnMut(&Self::Item) -> bool`: 创建一个迭代器，根据谓词跳过元素

    +   `fn skip(self, n: usize) -> Skip<Self>`: 创建一个迭代器，跳过前*n*个元素

    +   `fn take(self, n:usize) -> Take<Self>`: 创建一个迭代器，产生前*n*个元素

    +   `fn take_while<P>(self, predicate: P) -> TakeWhile<Self, P> where P: FnMut(&Self::Item) -> bool`: 创建一个基于谓词产生元素的迭代器

    +   `fn scan<St, B, F>(self, init_state: St, f: F) -> Scan<Self, St, F> where F: FnMut(&mut St, Self::Item) -> Option<B>`: 类似于`fold()`的迭代器适配器，持有内部状态并产生一个新的迭代器

    +   `fn flat_map<U, F>(self f: F) -> FlatMap<Self, U, F> where F: FnMut(&Self::Item) -> U, U: IntoIterator`: 创建一个扁平化的嵌套结构。类似于 map。

    +   `fn fuse(self) -> Fuse<Self>`: 创建一个在第一个`None`实例后结束的迭代器

    +   `fn inspect<F>(self, f: F) -> Inspect<Self, F> where F: FnMut(&Self::Item) -> ()`: 对每个元素执行一些操作并将值传递下去

    +   `fn by_ref(&mut self) -> &mut Self`: 借用迭代器。不会消耗它。

    +   `fn collect<B>(self) -> B where B: FromIterator<Self::Item>`: 将迭代器转换为集合

    +   `fn partition<B, F>(self, f: F) -> (B, B) where B: Default + Extend<Self::Item>, F: FnMut(&Self::Item) -> bool`: 消耗迭代器，从中创建两个集合

    +   `fn fold<B, F>(self, init: B, f: F) -> B where F: FnMut(B, Self::Item) -> B`: 应用函数的迭代器适配器，产生一个单一、最终值

    +   `fn all<F>(&mut self, f: F) -> bool where F: FnMut(&Self::Item) -> bool`: 测试迭代器中的每个元素是否匹配谓词

    +   `fn any<F>(&mut self, f: F) -> bool where F: FnMut(&Self::Item) -> bool`: 测试迭代器中的任何元素是否匹配谓词

    +   `fn find<P>(&mut self, predicate: P) -> Option<Self::Item> where P: FnMut(&Self::Item) -> bool`: 搜索满足谓词的迭代器元素

    +   `fn position<P>(&mut self, predicate: P) -> Option<usize> where P: FnMut(&Self::Item) -> bool`: 在迭代器中搜索元素，返回其索引

    +   `fn rposition<P>(&mut self, predicate: P) -> Option<usize> where P: FnMut(&Self::Item) -> bool, Self: ExactSizeIterator + DoubleEndedIterator`: 从右向迭代器中搜索元素，返回其索引

    +   `fn max(self) -> Option<Self::Item> where Self::Item: Ord`: 返回迭代器的最大元素

    +   `fn min(self) -> Option<Self::Item> where Self::Item: Ord`: 返回迭代器的最小元素

    +   `fn max_by_key<B, F>(self, f: F) -> Option<Self::Item> where B: Ord, F: FnMut(&Self::Item) -> B`: 返回指定函数的最大值元素

    +   `fn min_by_key<B, F>(self, f: F) -> Option<Self::Item> where B: Ord, F: FnMut(&Self::Item) -> B`: 返回指定函数的最小值元素

    +   `fn rev(self) -> Rev<Self> where Self: DoubleEndedIterator`: 反转迭代器的方向

    +   `fn unzip <A, B, FromA, FromB> (self) -> (FromA, FromB) where FromA: Default + Extend<A>, FromB: Default + Extend<B>, Self: Iterator<Item=(A, B)>`: 将对偶迭代器转换为两个容器

    +   `fn cloned<'a, T>(self) -> Cloned<Self> where Self: Iterator<Item=&'a T>, T: 'a + Clone`: 创建一个迭代器，它会克隆其所有元素

    +   `fn cycle(self) -> Cycle<Self> where Self: Clone`: 无限重复迭代器

    +   `fn sum<S>(self) -> S where S: Sum<Self::Item>`: 迭代迭代器的元素并求和

    +   `fn product<P>(self) -> P where P: Product<Self::Item>`: 迭代整个迭代器，将所有元素相乘

    +   `fn cmp<I>(self, other: I) -> Ordering where I: IntoIterator <Item=Self::Item>, Self::Item: Ord`: 比较这个迭代器的元素与另一个迭代器的元素

    +   `fn partial_cmp<I>(self, other: I) -> Option<Ordering> where I: IntoIterator, Self::Item: PartialOrd<I::Item>`: 比较这个迭代器的元素与另一个迭代器的元素

    +   `fn eq<I>(self, other: I) -> bool where I: IntoIterator, Self::Item: PartialEq<I::Item>`: 判断这个迭代器的元素是否等于另一个迭代器的元素

    +   `fn ne<I>(self, other: I) -> bool where I: IntoIterator, Self::Item: PartialEq<I::Item>`: 判断这个 `Iterator` 的元素是否不等于另一个迭代器的元素

    +   `fn lt<I>(self, other: I) -> bool where I: IntoIterator, Self::Item: PartialOrd<I::Item>`: 判断这个迭代器的元素是否小于另一个迭代器的元素

    +   `fn le<I>(self, other: I) -> bool where I: IntoIterator, Self::Item: PartialOrd<I::Item>`: 判断这个迭代器的元素是否小于或等于另一个迭代器的元素

    +   `fn gt<I>(self, other: I) -> bool where I: IntoIterator, Self::Item: PartialOrd<I::Item>`: 判断这个迭代器的元素是否大于另一个迭代器的元素

    +   `fn ge<I>(self, other: I) -> bool where I: IntoIterator, Self::Item: PartialOrd<I::Item>`: 判断这个迭代器的元素是否大于或等于另一个迭代器的元素

+   `impl<I> DoubleEndedIterator for Box<I> where I: DoubleEndedIterator + ?Sized`: 实现 `DoubleEndedIterator` trait，使得 `Box<I>` 可以双向迭代

    +   `fn next_back(&mut self) -> Option<I::Item>`: 从迭代器的末尾移除并返回一个元素

+   `impl <T> ExactSizeIterator for Box<I> where I: ExactSizeIterator + ?Sized`: 实现 `ExactSizeIterator` trait，使得 `Box<I>` 可以获取其确切大小

    +   `fn len(&self) -> usize`: 返回迭代器迭代的精确次数

+   `impl<T> Clone for Box<[T]> where T:Clone`: 实现 `Clone` trait，使得 `Box<[T]>` 可以被克隆

    +   `fn clone(&self) -> Box<[T]>`: 返回值的副本

    +   `fn clone_from(&mut self, source: &Self)`: 从源执行复制赋值

+   `impl<T> Borrow<T> for Box<T> where T:?Sized`: 实现 `Borrow` trait，使得 `Box<T>` 可以被借用

    +   `fn borrow(&self) -> &T`: 从拥有值中不可变借用

+   `impl<T> BorrowMut<T> for Box<T> where T:?Sized`: 实现 `BorrowMut` trait，使得 `Box<T>` 可以被可变借用

    +   `fn borrow_mut(&mut self) -> &mut T`: 从拥有值中可变借用

+   `impl<T> AsRef<T> for Box<T> where T:?Sized`: 实现 `AsRef` trait，使得 `Box<T>` 可以被转换为引用

    +   `fn as_ref(&self) -> &T`: 执行转换

+   `impl<T> AsMut for Box<T> where T:?Sized`: 实现 `AsMut` trait，使得 `Box<T>` 可以被转换为可变引用

    +   `fn as_mut(&mut self) -> &mut T`: 执行转换

+   `impl<’a, E: Error + ‘a> From<E> from Box<Error + ‘a>`: 实现 `From<E>` trait，将 `Box<Error + ‘a>` 从 `E` 类型转换而来

    +   `fn from(err: E) -> Box<Error + 'a>`: 执行转换

+   `impl From<String> for Box<Error + Send + Sync>`: 将 `String` 类型转换为 `Box<Error + Send + Sync>`

    +   `fn from(err: String) -> Box<Error + Send + Sync>`: 执行转换

+   `impl From<’a, ‘b> From<&’b str> for Box<Error + Send + Sync + ‘a>`: 将`Box<Error + Send + Sync + ‘a>`实现为从`&’b str`转换的类型

    +   `fn from(err: &'b str) -> Box<Error + Send + Sync + 'a>`: 执行转换

+   `impl<T: Error> Error for Box<T>`: 将`Box<T>`实现为`Error`类型

    +   `fn description(&self) -> &str`: 错误的简短描述

    +   `fn cause(&self) -> Option<&Error>`: 此错误的低级原因，如果有

+   `impl<R: Read + ?Sized> Read for Box<R>`: 将`Box<R>`实现为`Read`类型

    +   `fn read(&mut self, buf: &mut [u8]) -> Result<usize>`: 从指定缓冲区中拉取一些字节到这个源，并返回读取的字节数

    +   `fn read_to_end(&mut self, buf: &mut Vec<u8>) -> Result<usize>`: 读取此源中的所有字节直到 EOF，并将它们放入`buf`

    +   `fn read_to_string(&mut self, buf: &mut String) -> Result<usize>`: 读取此源中的所有字节直到 EOF，并将它们放入`buf`

    +   `fn read_exact(&mut self, buf: &mut [u8]) -> Result<()>`: 读取恰好足够的字节以填充`buf`

    +   `fn by_ref(&mut self) -> &mut Self where Self: Sized`: 为`Read`实例创建一个*引用*适配器

    +   `fn bytes(self) -> Bytes<Self> where Self: Sized`: 将此`Read`实例转换为对其字节的迭代器

    +   `fn chain<R: Read>(self, next: R) -> Chain<Self, R> where Self: Sized`: 创建一个适配器，将此流与另一个流连接起来

    +   `fn take(self, limit: u64) -> Take<Self> where Self: Sized`: 创建一个适配器，最多从它读取 limit 字节

+   `impl <W: Write + ?Sized> Write for Box<W>`: 将`Box<W>`实现为`Write`类型

    +   `fn write(&mut self, buf: &[u8]) -> Result<usize>`: 将缓冲区写入此对象，并返回写入的字节数

    +   `fn flush(&mut self) -> Result<()>`: 清空此输出流，确保所有中间缓冲的内容都达到目的地

    +   `fn write_all(&mut self, buf: &[u8]) -> Result<()>`: 尝试将整个缓冲区写入此写入器

    +   `fn write_fmt(&mut self, fmt: Arguments) -> Result<()>`: 将格式化字符串写入此写入器，并返回遇到的任何错误

    +   `fn by_ref(&mut self) -> &mut Self where Self: Sized`: 为`Write`实例创建一个*引用*适配器

+   `impl<S: Seek + ?Sized> Seek for Box<S>`: 将`Box<S>`实现为`Seek`类型

    +   `fn seek(&mut self, pos: SeekFrom) -> Result<u64>`: 在流中定位到指定的偏移量（以字节为单位）

+   `impl<B: BufRead + ?Sized> BufRead for Box<B>`: 将`Box<B>`实现为`BufRead`类型

    +   `fn fill_buf(&mut self) -> Result<&[u8]>`: 填充此对象的内部缓冲区，并返回缓冲区内容

    +   `fn consume(&mut self, amt: usize)`: 告诉此缓冲区已从缓冲区中消耗了 amt 字节，因此它们不应在调用`be_read`时返回

    +   `fn read_until(&mut self, byte: u8, buf: &mut Vec<u8>) -> Result<usize>`: 将所有字节读取到`buf`中，直到遇到分隔字节

    +   `fn read_line(&mut self, buf: &mut String) -> Result<usize>`: 读取所有字节直到遇到换行符（0 x A 字节），并将它们追加到提供的缓冲区

    +   `fn split(self, byte: u8) -> Split<Self> where Self: Sized`: 返回一个迭代器，该迭代器遍历此读取器的内容，按字节分割

    +   `fn lines(self) -> Lines<Self> where Self: Sized`: 返回此读取器的行迭代器

# `std::cell`

与共享可变容器一起使用：

关于使用`Cells`、`RefCell`以及内部和外部引用的详细信息，请参阅第十一章，*Rust 中的并发*。第十一章

+   `Std::cell::BorrowError`: 由`RefCell::try_borrow`返回

+   `impl Display for BorrowError`

    +   `fn fmt(&self, f: &mut Formatter) -> Result<(), Error>`: 使用给定的格式化器格式化值。

+   `impl Debug for BorrowError`

    +   `fn fmt(&self, f: &mut Formatter) -> Result<(), Error>`: 使用给定的格式化器格式化值。

+   `impl Error for BorrowError`

    +   `fn description(&self) -> &str`: 错误的简短描述

    +   `fn cause(&self) -> Option<&Error>`: 如果有的话，这是此错误的低级原因

+   `std::cell::BorrowMutError`: 由`RefCell::try_borrow_mut`返回

+   `impl Display for BorrowMutError`

    +   `fn fmt(&self, f: &mut Formatter) -> Result<(), Error>`: 使用给定的格式化器格式化值

+   `impl Debug for BorrowMutError`

    +   `fn fmt(&self, f: &mut Formatter) -> Result<(), Error>`: 使用给定的格式化器格式化值

+   `impl Error for BorrowMutError`

    +   `fn description(&self) -> &str`: 错误的简短描述

    +   `fn cause(&self) -> Option<&Error>`: 如果有的话，这是此错误的低级原因

+   `std::cell::Cell`: 只允许`Copy`数据的可变内存位置

**方法**

+   `impl<T> Cell<T> where T: Copy`

    +   `fn new(value: T) -> Cell<T>`: 创建一个包含给定值的新的 Cell

    +   `fn get(&self) -> T`: 返回包含的值的副本

    +   `fn set(&self, value: T)`: 设置包含的值

    +   `fn as_ptr(&self) -> *mut T`: 返回此单元格中底层数据的原始指针

    +   `fn get_mut(&mut self) -> &mut T`: 返回对底层数据的可变引用

**特质**

+   `impl<T> PartialEq<Cell<T>> for Cell<T> where T: Copy + PartialEq<T>`

    +   `fn eq(&self, other: &Cell<T>) -> bool`: 测试 self 和 other 值是否相等，并由`==`使用

    +   `fn ne(&self, other: &Rhs) -> bool`: 测试`!=`

+   `impl<T> Default for Cell<T> where T: Copy + Default`

    +   `fn default() -> Cell<T>`: 创建一个`Cell<T>`，其中`T`具有`Default`值

+   `impl<T> Clone for Cell<T> where T: Copy`

    +   `fn clone(&self) -> Cell<T>`: 返回值的副本

    +   `fn clone_from(&mut self, source: &Self)`: 从源执行复制赋值

+   `impl<T> From<T> for Cell<T> where T: Copy`

    +   `fn from(t: T) -> Cell<T>`: 执行转换

+   `impl<T> Ord for Cell<T> where T: Copy + Ord`

    +   `fn cmp(&self, other: &Cell<T>) -> Ordering`: 此方法返回 self 和 other 之间的`Ordering`

+   `impl<T> Debug for Cell<T> where T: Copy + Debug`

    +   `fn fmt(&self, f: &mut Formatter) -> Result<(), Error>`: 使用给定的格式化器格式化值

+   `impl<T> PartialOrd<Cell<T>> for Cell<T> where T: Copy + PartialOrd<T>`

    +   `fn partial_cmp(&self, other: &Cell<T>) -> Option<Ordering>`: 如果存在，此方法返回 self 和 other 值之间的排序

    +   `fn lt(&self, other: &Cell<T>) -> bool`: 此方法测试（对于自身和其他）小于，并由 `<` 操作符使用

    +   `fn le(&self, other: &Cell<T>) -> bool`: 此方法测试小于或等于（对于自身和其他），并由 `<=` 操作符使用

    +   `fn gt(&self, other: &Cell<T>) -> bool`: 此方法测试大于（对于自身和其他），并由 `>` 操作符使用

    +   `fn ge(&self, other: &Cell<T>) -> bool`: 此方法测试大于或等于（对于自身和其他），并由 `>=` 操作符使用

+   `Std::cell::Ref`: 在 `RefCell` 框中封装对值的借用引用

**方法**

+   `impl<'b, T> Ref<'b, T> where T: ?Sized`

    +   `fn clone(orig: &Ref<'b, T>) -> Ref<'b, T>`: 复制一个 `Ref`。`RefCell` 已经不可变借用，所以这不会失败。

    +   `fn map<U, F>(orig: Ref<'b, T>, f: F) -> Ref<'b, U>`

        `where F: FnOnce(&T) -> &U, U: ?Sized`: 为借用数据的组件创建一个新的 `Ref`。`RefCell` 已经不可变借用，所以这不会失败。

**特质实现**

+   `impl<'b, T> Debug for Ref<'b, T> where T: Debug + ?Sized`

    +   `fn fmt(&self, f: &mut Formatter) -> Result<(), Error>`: 使用给定的格式化器格式化值

+   `impl<'b, T> Deref for Ref<'b, T> where T: ?Sized`

    +   `fn deref(&self) -> &T`: 调用此方法来取消引用一个值

+   `Std::cell::RefCell`: 具有动态检查借用规则的可变内存位置

**方法**

+   `impl<T> RefCell<T>`

    +   `fn new(value: T) -> RefCell<T>`: 创建一个包含值的 `RefCell`

    +   `fn into_inner(self) -> T`: 消耗 `RefCell`，返回封装的值。

+   `impl<T> RefCell<T> where T: ?Sized`

    +   `fn borrow(&self) -> Ref<T>`: 不可变借用封装的值。

        借用持续到返回的 `Ref` 离开作用域。可以同时取出多个不可变借用。如果值当前正在可变借用，则引发恐慌。

    +   `fn try_borrow(&self) -> Result<Ref<T>, BorrowError>`: 不可变借用封装的值，如果值当前正在可变借用，则返回错误。借用持续到返回的 `Ref` 离开作用域。可以同时取出多个不可变借用。

    +   `fn borrow_mut(&self) -> RefMut<T>`: 可变借用封装的值。这个借用在返回的 `RefMut` 离开作用域时结束。在这次借用活动期间，不能再次借用该值（会引发恐慌）。

    +   `fn try_borrow_mut(&self) -> Result<RefMut<T>, BorrowMutError>`: 可变借用封装的值，如果值当前正在借用，则返回错误。借用持续到返回的 `RefMut` 离开作用域。在这次借用活动期间，值不能被借用。

    +   `fn as_ptr(&self) -> *mut T`: 返回此单元格中底层数据的原始指针。

    +   `fn get_mut(&mut self) -> &mut T`: 返回对底层数据的可变引用。

**特质实现**

+   `impl<T> PartialEq<RefCell<T>> for RefCell<T> where T: PartialEq<T> + ?Sized`

    +   `fn eq(&self, other: &RefCell<T>) -> bool`: 测试自身和其他值是否相等，并由 `==` 操作符使用

    +   `fn ne(&self, other: &Rhs) -> bool`: 测试 `!=`

+   `impl<T> Default for RefCell<T> where T: Default`

    +   `fn default() -> RefCell<T>`: 创建一个 `RefCell<T>`，具有 `T` 的 `Default` 值

+   `impl<T> Clone for RefCell<T> where T: Clone`

    +   `fn clone(&self) -> RefCell<T>`: 返回值的副本

    +   `fn clone_from(&mut self, source: &Self)`: 从源执行复制赋值

+   `impl<T> From<T> for RefCell<T>`

    +   `fn from(t: T) -> RefCell<T>`: 执行转换

+   `impl<T> Ord for RefCell<T> where T: Ord + ?Sized`

    +   `fn cmp(&self, other: &RefCell<T>) -> Ordering`: 返回 self 和 other 之间的 `Ordering`

+   `impl<T> Debug for RefCell<T> where T: Debug + ?Sized`

    +   `fn fmt(&self, f: &mut Formatter) -> Result<(), Error>`: 使用给定的格式化程序格式化值

+   `impl<T> PartialOrd<RefCell<T>> for RefCell<T> where T: PartialOrd<T> + ?Sized`

    +   `fn partial_cmp(&self, other: &RefCell<T>) -> Option<Ordering>`: 如果存在，返回 self 和其他值之间的排序

    +   `fn lt(&self, other: &RefCell<T>) -> bool`: 测试小于（对于 self 和 other）并用于 `<` 操作符

    +   `fn le(&self, other: &RefCell<T>) -> bool`: 测试小于等于（对于 self 和 other）并用于 `<=` 操作符

    +   `fn gt(&self, other: &RefCell<T>) -> bool`: 测试大于（对于 self 和 other）并用于 `>` 操作符

    +   `fn ge(&self, other: &RefCell<T>) -> bool`: 测试大于等于（对于 self 和 other）并用于 `>=` 操作符

代码示例在 第十一章，*Rust 的并发*。

# `std::char`

此模块用于结构体、特性和枚举字符类型。

+   **结构体**：`DecodeUtf16`、`DecodeUtf16Error`、`EscapeDefault`、`EscapeUnicode`、`ToLowercase` 和 `ToUpperCase`。

+   **常量**：`Max` 和 `Replacement_Character`。

+   **函数**：`decode_utf16`、`from_digit`、`from_u32` 和 `from_u32_unchecked`。

# `std::clone`

这是为了与不能隐式复制的类型一起使用。

更复杂的类型（如字符串）不能隐式复制。这些类型必须使用 `Clone` 特性和 `clone` 方法显式地使其可复制。

**结构体、特性和枚举**：特性 `Clone`。

# `std::cmp`

此模块提供了对数据进行排序和比较的能力。

此模块定义了 `PartialOrd`（重载 `<`、`<=`、`>` 和 `>=`）和 `PartialEq` 特性（重载 `==` 和 `!=`）。

**结构体、特性和枚举**：枚举 `Ordering`、特性 `Eq`（相等比较）、`Ord`（全序）、`PartialEq`（部分相等关系）、`PartialOrd`（可以比较排序顺序的值）。

# `std::collections`

这涵盖了向量、映射、集合和二叉堆。

有四种主要的集合类别，但在大多数情况下应使用 `Vec` 和 `HashMap`。

集合类型包括

+   序列（`Vec`、`VecDeque`、`LinkedList` - 如果你习惯于 C#，这些提供了 `List<T>` 的功能）

+   映射（`HashMap`、`BTreeMap`。对于 C#用户，这些大致等同于`Dictionary<T, U>`和`Map`）

+   集合（`HashSet`、`BTreeSet`）

+   二叉堆

应该使用哪个集合取决于你想做什么。每个都会根据你所做的事情产生不同的性能影响，尽管通常只有`HashMap`会产生负面影响。

使用示例：

+   `Vec`：创建一个可以调整大小的类型为`T`的集合；可以在末尾添加元素

+   `VecDeque`：创建一个类型为`T`的集合，但可以在两端插入元素；需要一个队列或双端队列（deque）

+   `LinkedList`：当需要`Vec`或`VecDeque`，以及分割和连接列表时使用

+   `HashMap`：创建键与值的缓存关联

+   BTreeMap：用于需要最大和最小键值对的键值对

+   二叉堆：存储元素，但在需要时只处理最大或最重要的元素

这些集合各自处理自己的内存管理。这对于集合能够根据需要分配更多空间（并且在其运行的机器的硬件容量限制内）非常重要。

考虑以下情况：我创建了一个没有设置容量的`Vec<T>`。让`T`为一个结构体。这种情况并不少见。我向`Vec`中添加了一些对象，然后每个对象都在堆上分配。堆扩展了，这是可以的。然后我删除了这些对象中的几个。Rust 随后将属于`Vec`的堆的其他成员*重新定位*。

如果我使用`with_capacity`分配空间，那么我们就有了一个最大分配可用，这有助于内存管理。我们可以通过使用`shrink_to_fit`进一步帮助内存分配，这会减小我们的`Vec`的大小以适应所需的大小。

**迭代器**

迭代器非常实用，并且在库中广泛使用。主要来说，迭代器用于 for 循环中。几乎所有的集合都提供了三个迭代器：`iter`、`iter_mut`和`into_iter`。每种迭代器类型执行不同的功能：

+   `iter`：这提供了一个不可变引用的迭代器，按最适合集合类型的顺序遍历集合的所有内容。

+   `iter_mut`：这提供了一个与`iter`相同的顺序的可变引用迭代器。

+   `into_iter`：将集合转换为迭代器。当集合本身不需要，但其内容需要时非常有用。`into_iter`迭代器还包括扩展向量的能力。

**结构体**：`BTreeMap`、`BTreeSet`、`BinaryHeap`、`HashMap`、`HashSet`、`LinkedList`和`VecDeque`。

# std::convert

此模块用于类型之间的转换。

当编写库时，实现`From<T>`和`TryFrom<T>`而不是`Into<T>`和`TryInto<T>`，因为`From`提供了更大的灵活性。

+   **实现**: `As*`（引用到引用的转换），`Into`（在转换中消耗值），`From`（用于值和引用转换），`TryFrom`，和`TryInto`（类似于`From`和`Into`，允许失败）

+   **结构体、特性和枚举**: 特质`AsMut`，`AsRef`，`From`，和`Into`

# std::default

此特质为类型提供有意义的值。

默认为各种原始类型提供默认值。如果使用复杂类型，则需要实现`Default`。

**结构体、特性和枚举**: 特质`Default`。

# std:env

此模块用于处理进程环境。

提供了从当前操作系统获取值的一组函数。

**结构体、特性和枚举**

+   **枚举**: `VarError`（`env::var`方法的可能错误）

+   **结构体**: `Args`（为每个参数生成一个`String`），`ArgOs`（为每个参数生成一个`OsString`），`JoinPathsError`（当路径无法连接时返回错误），`SplitPaths`（遍历`PathBuf`以解析环境变量到平台特定的约定），`Vars`，以及`VarsOS`（遍历进程的环境变量快照）

# std:error

此模块用于处理错误。

**结构体、特性和枚举**: 特质`Error`（所有错误的基功能）

# std::f32

此模块用于处理 32 位浮点类型。

此模块提供基本的数学常数：`Digits`，`Epsilon`，`Infinity`，`Mantissa_Digits`，`Max`（最大的有限`f32`值），`Max_10_Exp`，`Max_Exp`，`Min`（最小的有限`f32`值），`Min_10_Exp`，`Min_Exp`，`Min_Positive`（最小的可能归一化`f32`值），`NAN`，`Neg_Infinity`，和`Radix`。

# std::f64

此模块用于处理 64 位浮点类型。

此模块提供基本的数学常数：`Digits`，`Epsilon`，`Infinity`，`Mantissa_Digits`，`Max`（最大的有限`f64`值），`Max_10_Exp`，`Max_Exp`，`Min`（最小的有限`f64`值），`Min_10_Exp`，`Min_Exp`，`Min_Positive`（最小的可能归一化`f64`值），`NAN`，`Neg_Infinity`，和`Radix`。

# std:ffi

FFI 是 Rust 与非 Rust 库交互的方法。此特质为此目的提供了一些实用工具。

**结构体、特性和枚举**: 结构体`CStr`，`CString`（分别表示借用的 C 字符串和所有权的 C 兼容字符串），`FromBytesWithNullError`（从`CStr::from_bytes_with_nul`返回的错误），`IntoStringError`（从`CString::into_string`返回的错误，表示转换期间的 UTF8 错误），`NulError`（从`CString::new`返回的错误，指示在提供的向量中找到了空字节），`OsStr`，和`OsString`（切片到操作系统字符串）。

# std::fmt

此模块用于格式化和输出字符串。

此模块提供了`format!`宏来处理输出。该宏非常强大且非常灵活，提供了大量的功能。

**结构体、特性和枚举**

+   **结构体**: `Arguments`（表示格式字符串和参数的安全预编译版本），`DebugList`，`DebugMap`，`DebugSet`，`DebugStruct`，`DebugTuple`（帮助实现 `fmt::Debug`），`Error`（将消息格式化为流时返回的错误类型），以及 `Formatter`（表示输出格式化字符串的位置以及如何格式化它们）。

+   **特性**: `Binary`，`Debug`，`Display`，`LowerExp`，`LowerHex`，`Octal`，`Pointer`，`UpperExp`，`UpperHex` 和 `Write`（提供将消息格式化为流所需的方法集合）。

+   **函数**: `format`（接受一个预编译的格式字符串和参数，并返回一个格式化后的字符串）和 `write`（接受一个输出流、一个预编译的格式字符串和参数列表。参数将根据指定的格式字符串进行格式化）。

# std::fs

此模块在使用文件系统和操作文件时使用。

此模块提供了一组跨平台方法来操作应用程序所在的文件系统。如果可能的话，请避免使用 `remove_dir_all` 函数。

结构体、特性和枚举

+   **结构体**: `DirBuilder`（用于创建目录），`DirEntry`（由 `ReadDir` 迭代器返回），`File`（在文件系统上打开文件），`FileType`（表示文件类型，具有访问每个文件类型的访问器），`Metadata`（关于文件的信息），`OpenOptions`（用于配置如何打开文件的选项和标志），`Permissions`（文件上的文件权限），以及 `ReadDir`（目录内条目的迭代器）。

+   **函数**: `canonicalize`（返回路径的规范形式），`copy`（复制文件），`create_dir`，`create_dir_all`（递归创建目录及其所有父组件，如果缺失），`hard_link`（在文件系统上创建硬链接），`metadata`（获取给定路径和文件的元数据），`read_dir`（返回目录内条目的迭代器），`read_link`（读取符号链接，返回它指向的文件），`remove_dir`（删除空目录），`remove_dir_all`（递归删除路径上的目录——在某些操作系统中这可能会完全删除您的硬盘，所以请小心！），`remove_file`（删除文件），`rename`（重命名给定的文件或目录），`set_permissions`（设置给定文件或目录的权限），以及 `symlink_metadata`（查询不跟随任何符号链接的文件的元数据）。

# std::hash

此模块用于提供哈希支持。

此模块确保为给定类型创建哈希的最简单方法是使用 `#[derive(Hash)]`。

**结构体、特性和枚举**

+   **结构体**: `BuildHasherDefault`（为所有实现 `Default` 的 Hasher 类型实现 `BuildHasher`）和 `SipHasher`（`SipHash` 的实现）

+   **特性**: `BuildHasher`，`Hash` 和 `Hasher`

# std::i8

此模块定义了 8 位整数类型。

此模块定义了 `MAX` 和 `MIN` 常量。

# std::i16

此模块定义了 16 位整数类型。

本模块定义了`MAX`和`MIN`常量。

# std::i32

本模块定义了 32 位整数类型。

本模块定义了`MAX`和`MIN`常量。

# std::i64

本模块用于处理 64 位整数类型。

本模块定义了`MAX`和`MIN`常量。

# std::io

本模块提供了核心输入/输出的一系列功能。

本模块不仅为正常控制提供代码`Read`和`Write`功能，还为各种流类型（如 TCP 和文件）提供功能。访问可以是顺序的或随机的。IO 行为还取决于应用程序所在的平台，因此强烈建议进行测试。

**结构体、特性和枚举**

+   **结构体**: `BufReader`（为任何读取器添加缓冲），`BufWriter`（缓冲写入器的输出），`Bytes`（读取器值的迭代器），`Chain`（连接两个读取器），`Cursor`（包装另一个类型并提供 Seek 实现），`Empty`（总是位于 EOF 的读取器），`Error`（IO 操作的错误类型），`IntoInnerError`（`into_inner`返回的错误，它结合了错误和缓冲写入器对象，可能可以恢复），`LineWriter`（包装写入器并将缓冲写入其中），`Lines`（遍历`BufRead`的行），`Repeat`（不断返回字节的读取器），`Sink`（将数据移动到 null 的写入器），`Split`（在`BufRead`内容中分割点的迭代器），`Stderr`（进程标准错误流的句柄），`StdErrLock`（`Stderr`的锁定引用），`Stdin`（标准输入流），`StdinLock`（`Stdin`的锁定引用），`Stdout`（全局输出流），`StdoutLock`（`Stdout`的锁定引用），以及`Take`（限制从读取器读取的字节数）。

+   **枚举**: `ErrorKind`和`SeekFrom`。

+   **特性**: `BufRead`（缓冲输入读取），`Read`（从源读取字节），`Seek`（提供可以在流中移动的光标），和`Write`。

+   **函数**: `copy`（将读取器的内容复制到写入器），`empty`（创建一个空读取器的新句柄），`repeat`（创建一个不断重复 1 字节的读取器实例），`sink`（消耗所有数据的写入器实例），`stderr`（`stderr`的新句柄），`stdin`（`stdin`的新句柄），和`stdout`（`stdout`的新句柄）。

# std::isize

本模块用于与指针大小的整数类型一起使用。

本模块定义了`MAX`和`MIN`常量。

# std::iter

本模块用于迭代。

**结构体、特性和枚举**

+   **结构体**：`Chain`（将两个迭代器连接起来）、`Cloned`（克隆底层迭代器）、`Cycle`（永不结束的迭代器）、`Empty`（不产生任何内容）、`Enumerate`（在迭代时产生当前计数和元素）、`Filter`（使用谓词过滤`iter`的元素）、`FilterMap`（使用`iter`的类型的迭代器，用于过滤和映射）、`FlatMap`（将每个元素映射到迭代器，产生元素）、`Fuse`（一旦底层迭代器首次迭代`None`，则持续产生`None`）、`Inspect`（在产生元素之前调用一个带有元素引用的函数）、`Map`（使用类型映射`iter`的值）、`Once`（只产生一个元素）、`Peekable`（允许使用`peek()`）、`Repeat`（无限重复一个元素）、`Rev`（具有反向读取方向的双向迭代器）、`Scan`（在迭代另一个迭代器时保持状态）、`Skip`（跳过`iter`的*n*个元素）、`SkipWhile`（在谓词为真时拒绝元素）、`Take`（只迭代`iter`的前*n*个元素）、`TakeWhile`（在谓词为真时只接受迭代元素），以及`Zip`（同时迭代两个迭代器）。

+   **特性**：`DoubleEndedIterator`（从两端产生元素）、`ExactSizeIterator`（已知确切长度）、`Extend`（使用迭代器的内容扩展集合）、`FromIterator`（从`Iterator`转换）、`ToIterator`（转换为`Iterator`）、`Iterator`（处理迭代器的接口）。

+   **函数**：`empty`（产生不产生任何内容的新的迭代器）、`once`（只产生一个元素的新的迭代器）和`repeat`（持续重复单个元素的新的迭代器）。

# std::marker

此模块提供了原始特性和标记来表示基本类型种类。

**结构体、特性和枚举**

+   **结构体**：`PhantomData`（允许描述类型`T`）。

+   **特性**：`Copy`（可复制的类型）、`Send`（可在线程间传输的类型）、`Sized`（具有固定大小的类型）和`Sync`（可在线程间安全共享的类型）。

# std::mem

此模块执行内存处理函数。

此模块用于查询大小和对齐类型、初始化以及内存操作。

**结构体、特性和枚举**

+   **函数**: `align_of`（返回类型的内存对齐方式），`align_of_val`（指向值`val`的类型的最小对齐方式），`drop`（处置），`forget`（将值留为 void，获取所有权但不运行析构函数），`replace`（用新值替换`mut`位置的值，返回旧值但不初始化或复制任何一个），`size_of`（返回类型的大小，以字节为单位），`size_of_val`（返回值的大小，以字节为单位），`swap`（交换两个 mut 位置的值；必须为同一类型），`transmute`（不安全地将一个类型的值转换为另一个类型），`transmute_copy`（将`src`解释为`&T`，然后读取`src`而不移动包含的值），`uninitialized`（绕过 Rust 的内存初始化要求），以及`zeroed`（创建一个初始化为零的值）。

# std:net

此模块提供基本的 TCP/UDP 通信原语。

**结构体、特性和枚举**

+   **结构体**: `AddrParseError`（解析 IP 或套接字地址时返回的错误），`Incoming`（`TcpListener`连接的无穷迭代器），`Ipv4Addr`（表示 IPv4 地址），`Ipv6Addr`（表示 IPv6 地址），`SocketAddrV4`（IPv4 套接字地址），`SocketAddrV6`（IPv6 套接字地址），`TcpListener`（表示套接字服务器），`TcpStream`（表示本地和远程套接字之间的 TCP 流），以及`UdpSocket`（UDP 套接字）。

+   **枚举**: `IpAddr`（IPv4 或 IPv6 地址），`Shutdown`（传递给`TcpStream`的 shutdown 方法的值），以及`SocketAddr`（网络应用程序的套接字地址）。

+   **特性**: `ToSocketAddrs`（可以转换为或解析一个或多个`SocketAddr`值的对象）。

# std::num

此模块用于处理数字。

此模块提供处理数字的有用类型。

**结构体、特性和枚举**

+   **结构体**: `ParseFloatError`（解析`float`时返回的错误），`ParseIntError`（解析`int`时返回的错误），以及`Wrapping`（对`T`有意包裹的算术运算）。

+   **枚举**: `FpCategory`（浮点数的分类）。

# std::os

此模块包含提供对应用程序正在运行的 OS 抽象访问的函数。

此模块包含三个模块：`linux`（特定于 Linux），`raw`（当前平台特定的原始 OS 类型），以及`unix`（实验性扩展）。

# std::panic

此模块为标准库中的 panic 提供支持。

**结构体、特性和枚举**

+   **结构体**: `AssertUnwindSafe`（检查类型是否为 panic 安全），`Location`（关于 panic 位置的信息），以及`PanicInfo`（关于 panic 的信息）。

+   **特性**: `RefUnwindSafe`（表示共享 ref 被认为是`recovery`安全的特性）和`UnwindSafe`（表示 Rust 中 panic 安全的类型的特性）。

+   **函数**: `catch_unwind`（调用闭包，捕获恢复的原因）、`resume_unwind`（触发恐慌而不调用恐慌）、`set_hook`（注册自定义恐慌钩子并替换以前的钩子）和`take_hook`（注销当前恐慌钩子）。

# std::path

此模块以跨平台方式提供对路径的抽象访问，以便进行操作。

提供了两种类型，`PathBuf`和`Path`。这些是`OsString`和`OsStr`的包装器，允许根据本地平台路径直接在字符串上执行操作。

**结构体、特性和枚举**

+   **结构体**: `Components`（核心迭代器，提供路径的部分）、`Display`（用于安全地使用`format!()`和`{}`打印路径）、`Iter`（路径部分的迭代器）、`Path`（路径切片）、`PathBuf`（所有者可变路径）、`PrefixComponent`（Windows 特定的路径前缀）和`StripPrefixError`（从`Path::strip_prefix`方法返回的错误，指示前缀在自身中未找到）。

+   **枚举**: `Component`（路径的单个组件）和`Prefix`（路径前缀 [仅限 Windows]）。

+   **函数**: `is_separator`（确定字符是否是允许的路径分隔符之一）。

# std::process

此模块用于处理进程。

**结构体、特性和枚举**

+   **结构体**: `Child`（表示正在运行或已退出的子进程）、`ChildStderr`（子进程标准错误句柄）、`ChildStdin`（子进程标准输入句柄）、`ChildStdout`（子进程标准输出句柄）、`Command`（作为进程构建器）、`ExitStatus`（描述进程终止后的结果）、`Output`（已完成进程的输出）和`Stdio`（描述对子进程标准 IO 流的处理）。

+   **函数**: `exit`（使用退出代码终止当前进程）。

# std::ptr

此模块提供处理原始、不安全指针的访问。

请参阅第五章，*记住，记住*，以获取更多详细信息。

**结构体、特性和枚举**

**函数**: `copy`（从`src`复制`count * size_of<T>`到`dest`；可以重叠）、`copy_nonoverlapping`（与`copy`相同，但不能重叠）、`drop_in_place`（执行指向的值的析构函数）、`null`（创建新的空原始指针）、`null_mut`（创建新的空可变原始指针）、`read`（从`src`读取值而不移动它）、`read_volatile`（从`src`以非移动方式执行可变读取）、`replace`（将`dest`中的值替换为`src`，返回旧值）、`swap`（交换两个相同类型的可变位置的值）、`write`（覆盖内存位置，不读取或丢弃旧值）、`write_bytes`（在指定的指针上调用`memset`）和`write_volatile`（执行具有给定值的内存位置的可变写入）。

# std::slice

此模块提供动态大小的放置到连续的 `[T]` 中。

切片是内存的表示为指针的可变切片（`&mut [T]`）或共享切片（`&[T]`）。它们实现了`IntoIter`，该类型正在执行`IntoIter`。

**结构体、特性和枚举**

+   **结构体**: `Chunks`（每次迭代非重叠切片的`size_of<T>`元素大小的块），`ChunksMut`（与`Chunks`类似，但可变），`Iter`（不可变迭代器），`IterMut`（可变迭代器），`RSplitN`和`RSplitNMut`（迭代匹配谓词的子切片，限制在给定的分割次数内，并从切片的末尾开始）。`Split`和`SplitMut`（分别通过匹配谓词函数或谓词分隔的子切片迭代器），以及`SplitN`和`SplitNMut`（迭代匹配谓词函数的子切片），还有`Windows`（迭代长度为`size_of<T>`的重叠子切片）。

+   **函数**: `from_raw_parts`（从指针和长度形成切片）和`from_raw_parts_mut`（与`from_raw_parts`类似，但返回的切片是可变的）。

# std::str

此模块用于 Unicode 字符串切片。

**结构体、特性和枚举**

+   **结构体**: `Bytes`（字符串字节的迭代器），`CharIndices`（字符串字符和字节偏移量的迭代器），`Chars`（字符串字符的迭代器），`EncodeUtf16`（字符串 UTF16 代码的外部迭代器），`Lines`（使用`lines()`创建），`MatchIndices`（使用`match_indices()`创建），`Matches`（使用`matches()`创建），`ParseBoolError`（当从字符串传递`bool`失败时返回的错误），`RMatchIndicies`（使用`rmatch_indicies()`创建），`RMatches`（使用`rmatches()`创建），`RSplit`（使用`rsplit()`创建），`RSplitN`（使用`rsplitn()`创建），`RSplitTerminator`（使用`rsplit_terminator()`创建），`Split`（使用`split()`创建），`SplitN`（使用`splitn()`创建），`SplitTerminator`（使用`split_terminator()`创建），`SplitWhitespace`（迭代字符串的非空白子字符串），以及`Utf8Error`（尝试将`u8`序列解释为字符串时可能发生的错误）。

+   **特性**: `FromStr`（抽象了从字符串创建类型新实例的想法）。

+   **函数**: `from_utf8`（将字节数组切片转换为字符串切片）和`from_utf8_unchecked`（与`from_utf8`类似，但不检查字符串是否包含有效的 UTF8）。

# std::string

此模块提供使用 UTF-8 编码的可增长字符串进行字符串处理。

包含 String 类型以及将其转换为 String 的特性和错误类型。

**结构体、特性和枚举**

+   **结构体**: `Drain`（排空迭代器），`FromUtf16Error`（从 UTF16 切片转换时可能出现的错误值），以及`FromUtf8Error`（与`FromUtf16Error`类似，但针对 UTF8），还有`String`（UTF8 编码的可增长字符串）。

+   **枚举**: `ParseError`。

+   **特性**: `ToString`（将值转换为字符串）。

# std::sync

此模块提供线程同步函数。

这在第十一章第十一章，*Rust 中的并发*中有介绍。

**结构体、特性和枚举**

+   **结构体**：`Arc`（原子引用计数包装器），`Barrier`（使多个线程能够同步某些计算的开始），`BarrierWaitResult`（线程等待的结果），`Condvar`（条件变量），`Mutex`（互斥原语），`MutexGuard`（作用域锁互斥量；当结构退出作用域时解锁），`Once`（用于运行一次性全局初始化的同步原语），`PoisonError`（当需要锁时可能返回的错误），`RwLock`（读写锁），`RWLockReadGuard`（用于在丢弃时释放对锁的共享读访问），`RWWriteGuard`（用于在丢弃时释放对锁的共享写访问），`WaitTimeoutResult`（用于确定条件变量是否超时的类型），以及`Weak`（指向`Arc`的弱指针）。

+   **枚举**：`TryLockError`（在调用`try_lock`时可能发生的错误）。

请参阅第十一章中的代码示例第十一章，*Rust 中的并发*。

# std::thread

这是主要的线程模块，为 Rust 应用程序提供原生线程。

线程在第十一章第十一章，*Rust 中的并发*中有介绍。

**结构体、特性和枚举**

+   **结构体**：`Builder`（提供对新线程的详细控制），`JoinHandle`（拥有在某个线程上加入的权限），`LocalKey`（拥有内容的本地存储键），以及`Thread`（线程句柄）。

+   **函数**：`current`（获取线程调用的句柄），`panicking`（如果线程由于 panic 而正在回溯），`park`（除非或直到令牌可用否则阻塞），`park_timeout`（阻塞一段时间），以及`sleep`（使当前线程休眠一段时间），`spawn`（创建新线程，返回`JoinHandle`），和`yield_now`（向操作系统调度器放弃时间片）。

请参阅第十一章中的代码示例第十一章，*Rust 中的并发*。

# std::time

这是一个处理时间的模块。

**结构体、特性和枚举**

**结构体**：`Duration`（表示时间跨度），`Instant`（单调递增时钟的测量），`SystemTime`（测量系统时钟），以及`SystemTimeError`（从`SystemTime.duration_since()`返回的错误）。

# std::u8

此模块定义了无符号 8 位整数类型。

此模块定义了`MAX`和`MIN`常量。

# std::u16

此模块定义了无符号 16 位整数类型。

此模块定义了`MAX`和`MIN`常量。

# std::u32

此模块定义了无符号 32 位整数类型。

此模块定义了`MAX`和`MIN`常量。

# std::u64

此模块定义了无符号 64 位整数类型。

此模块定义了`MAX`和`MIN`常量。

# std::usize

这是指针大小的无符号整数类型。

此模块定义了`MAX`和`MIN`常量。

# std::vec

此模块定义了具有堆分配内容的可增长数组类型。

这被写成 `Vec<T>`，值可以通过 `push` 和 `pull` 分别添加到（或从）`vec` 的末尾。

**结构体、特性和枚举**

**结构体**：`Drain`（`Vec<T>` 的排空迭代器）、`IntoIter`（从向量中移出的迭代器）和 `Vec`（连续可增长数组类型）。

# 摘要

我们已经覆盖了 Rust 标准库的大部分内容。请始终检查在线官方文档 [`doc.rust-lang.org/std/`](https://doc.rust-lang.org/std/)——它质量极高且始终是最新的！

在下一章和最后一章中，我们将探讨如何通过 Rust 的 **外部函数接口**（**FFI**）使用外部库。
