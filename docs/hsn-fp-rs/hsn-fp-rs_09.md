# 第九章：性能、调试和元编程

编写快速高效的代码可以是一件值得骄傲的事情。这也可能浪费你雇主资源。在性能部分，我们将探讨如何区分这两者，并给出最佳实践、流程和指南，以保持你的应用程序精简。

在调试部分，我们提供了一些技巧，帮助您更快地找到和解决错误。我们还介绍了防御性编码的概念，它描述了防止或隔离潜在问题的技术和习惯。

在元编程部分，我们解释了宏和其他类似宏的功能。Rust 有一个相当复杂的元编程系统，允许用户或库通过自动代码生成或自定义语法形式扩展语言。

在本章中，我们将学习以下内容：

+   识别和应用良好的性能代码实践

+   诊断和改进性能瓶颈

+   识别和应用良好的防御性编码实践

+   诊断和解决软件错误

+   识别和应用元编程技术

# 技术要求

运行提供的示例需要 Rust 的最近版本：

[`www.rust-lang.org/en-US/install.html`](https://www.rust-lang.org/en-US/install.html)

本章的代码可在 GitHub 上找到：

[`github.com/PacktPublishing/Hands-On-Functional-Programming-in-RUST`](https://github.com/PacktPublishing/Hands-On-Functional-Programming-in-RUST)

每个章节的`README.md`文件中都包含了具体的安装和构建说明。

# 编写更快的代码

过早的优化是万恶之源

– 唐纳德·克努特

良好的软件设计往往能创建更快的程序，而糟糕的软件设计往往能创建更慢的程序。如果你发现自己正在问，“我的程序为什么这么慢？”，那么首先问问自己，“我的程序是否混乱？”

在本节中，我们描述了一些性能技巧。这些通常是 Rust 编程中的良好习惯，无意中会导致性能提升。如果你的程序运行缓慢，那么首先检查你是否违反了这些原则之一。

# 以发布模式编译

这是一个你应该知道的非常简单的建议，如果你对性能有任何关注的话。

+   Rust 通常以调试模式编译，这比较慢：

```rs
cargo build
```

+   Rust 可以选择以发布模式编译，这比较快：

```rs
cargo build --release
```

+   这里是一个使用调试模式为玩具程序进行比较的示例：

```rs
$ time performance_release_mode
real 0m13.424s
user 0m13.406s
sys 0m0.010s
```

+   以下为发布模式：

```rs
$ time ./performance_release_mode
real 0m0.316s
user 0m0.309s
sys 0m0.005s
```

发布模式在此示例中相对于 CPU 使用效率提高了 98%。

# 做更少的工作

更快的程序做更少的工作。所有优化都是一个寻找不需要完成的工作的过程，然后不去做它。

同样，最小的程序使用更少的资源。所有空间优化都是一个寻找不需要使用的资源的过程，然后不使用它们。

例如，当你不需要结果时，不要收集迭代器，考虑以下示例：

```rs
extern crate flame;
use std::fs::File;

fn main() {
   let v: Vec<u64> = vec![2; 1000000];

   flame::start("Iterator .collect");
   let mut _z = vec![];
   for _ in 0..1000 {
      _z = v.iter().map(|x| x*x).collect::<Vec<u64>>();
   }
   flame::end("Iterator .collect");

   flame::start("Iterator iterate");
   for _ in 0..1000 {
      v.iter().map(|x| x * x).for_each(drop);
   }
   flame::end("Iterator iterate");

   flame::dump_html(&mut File::create("flame-graph.html").unwrap()).unwrap();
}
```

无需收集迭代器的结果会使代码比仅丢弃结果的代码慢 27%。

内存分配类似。设计良好的代码倾向于使用纯函数并避免副作用，从而最小化内存使用。相反，混乱的代码可能导致旧数据滞留。Rust 的内存安全性并不包括防止内存泄漏。泄漏被视为安全代码：

```rs
use std::mem::forget;

fn main() {
   for _ in 0..10000 {
      let mut a = vec![2; 10000000];
      a[2] = 2;
      forget(a);
   }
}

```

`forget` 函数很少使用。同样，内存泄漏是被允许的，但被充分劝阻，以至于它们相对不常见。Rust 的内存管理往往是这样，当你造成内存泄漏时，你可能已经陷入了其他糟糕的设计决策中。

然而，未使用的内存并不少见。如果你不跟踪你正在积极使用的变量，那么旧变量很可能会保留在作用域内。这并不是内存泄漏的典型定义；然而，未使用的数据是类似资源的浪费。

# 优化需要优化的代码——分析

不要优化那些不需要优化的代码。这是浪费时间，也可能是糟糕的软件工程。省去麻烦，在尝试优化程序之前，准确识别性能问题。

# 对于很少执行的代码，性能不受影响

初始化一些资源并多次使用它是非常常见的。优化资源的 `initialization` 可能是错误的。你应该考虑专注于提高 `work` 的效率。这可以通过以下方式完成：

```rs
use std::{thread,time};

fn initialization() {
   let t = time::Duration::from_millis(15000);
   thread::sleep(t);
}

fn work() {
   let t = time::Duration::from_millis(15000);
   loop {
      thread::sleep(t);
      println!("Work.");
   }
}

fn main() {
   initialization();
   println!("Done initializing, start work.");
   work();
}
```

# 小数的倍数也是小数

反过来也可能成立。有时 `work` 的低频率会被频繁且昂贵的 `initialization` 所淹没。了解你遇到的问题将帮助你确定从哪里开始寻找以改进：

```rs
use std::{thread,time};

fn initialization() -> Vec<i32> {
   let t = time::Duration::from_millis(15000);
   thread::sleep(t);
   println!("Initialize data.");
   vec![1, 2, 3];
}

fn work(x: i32) -> i32 {
   let t = time::Duration::from_millis(150);
   thread::sleep(t);
   println!("Work.");
   x * x
}

fn main() {
   for _ in 0..10 {
      let data = initialization();
      data.iter().map(|x| work(*x)).for_each(drop);
   }
}
```

# 先测量，再优化

分析有很多选项。以下是我们推荐的一些。

`flame` crate 是手动分析应用程序的一个选项。在这里，我们创建了嵌套过程 `a`、`b` 和 `c`。每个函数创建一个与该方法对应的分析上下文。在运行分析器后，我们将看到每个函数的每次调用所花费的时间比例。

从函数 `a` 开始，此过程创建一个新的分析上下文，休眠一秒钟，然后调用 `b` 三次：

```rs
extern crate flame;
use std::fs::File;
use std::{thread,time};

fn a() {
   flame::start("fn a");
   let t = time::Duration::from_millis(1000);
   thread::sleep(t);
   b();
   b();
   b();
   flame::end("fn a");
}
```

函数 `b` 几乎与 `a` 相同，并进一步调用函数 `c`：

```rs
fn b() {
   flame::start("fn b");
   let t = time::Duration::from_millis(1000);
   thread::sleep(t);
   c();
   c();
   c();
   flame::end("fn b");
}
```

函数 `c` 会自我分析并休眠，但不会调用任何更深层的嵌套函数：

```rs
fn c() {
   flame::start("fn c");
   let t = time::Duration::from_millis(1000);
   thread::sleep(t);
   flame::end("fn c");
}
```

`main` 入口设置火焰图库并调用三次，然后保存火焰图到文件：

```rs
fn main() {
   flame::start("fn main");
   let t = time::Duration::from_millis(1000);
   thread::sleep(t);
   a();
   a();
   a();
   flame::end("fn main");
   flame::dump_html(&mut File::create("flame-graph.html").unwrap()).unwrap();
}
```

运行此程序后，`flame-graph.html` 文件将包含程序各部分占资源百分比的可视化。`flame` crate 容易安装，需要一些手动代码操作，但会产生一个看起来很酷的图表。

`cargo profiler` 是一个工具，它扩展了 `cargo` 以进行性能分析，而无需任何代码更改。以下是一个我们将要分析的随机程序：

```rs
fn a(n: u64) -> u64 {
   if n>0 {
      b(n);
      b(n);
   }
   n * n
}

fn b(n: u64) -> u64 {
   c(n);
   c(n);
   n + 2 / 3
}

fn c(n: u64) -> u64 {
   a(n-1);
   a(n-1);
   vec![1, 2, 3].into_iter().map(|x| x+2).sum()
}

fn main() {
   a(6);
}
```

要分析应用程序，我们运行以下命令：

```rs
$ cargo profiler callgrind --bin ./target/debug/performance_profiling4 -n 10

```

这将运行程序并收集有关哪些函数被最频繁使用的相关信息。这个分析器还有一个选项来分析内存使用情况。输出将如下所示：

```rs
Profiling performance_profiling4 with callgrind...

Total Instructions...344,529,557

27,262,872 (7.9%) ???:core::iter::iterator::Iterator
----------------------------------------------------------
22,319,604 (6.5%) ???:<alloc::vec
----------------------------------------------------------
16,627,356 (4.8%) ???:<core::iter
----------------------------------------------------------
13,182,048 (3.8%) ???:<alloc::vec
----------------------------------------------------------
10,785,312 (3.1%) ???:core::iter::iterator::Iterator::fold
----------------------------------------------------------
10,485,720 (3.0%) ???:core::mem
----------------------------------------------------------
8,088,984 (2.3%) ???:alloc::slice::hack
----------------------------------------------------------
7,639,596 (2.2%) ???:core::ptr
----------------------------------------------------------
7,190,208 (2.1%) ???:core::ptr
----------------------------------------------------------
7,190,016 (2.1%) ???:performance_profiling4
```

这清楚地表明，大部分时间都花在迭代器和向量的创建上。运行这个命令可能会使程序执行速度比正常情况下慢得多，但它也节省了在分析之前编写任何代码的时间。

# 将冰箱放在电脑旁边

如果你编程时想休息一下，那么在电脑旁边有一个冰箱和微波炉会非常方便。如果你去厨房吃零食，那么满足你的胃口需要更长的时间。如果你的厨房空了，你需要去购物，那么休息时间会更长。如果你的杂货店也空了，你需要开车去农场采摘蔬菜，那么你的工作环境显然不是为吃零食而设计的。

这个奇怪的类比说明了时间和空间之间必要的权衡。这种关系对于我们来说几乎不是一条物理定律，但几乎是。规则是，在更长的距离上旅行或通信，与花费的时间成正比。在一个方向上更多的距离（d）也意味着可用空间以二次（d²）或三次（d³）的比例增加。换句话说，将冰箱建得更远，可以为更大的冰箱提供更多的空间。

将这个故事带回到技术环境中，以下是一些程序员应该知道的延迟数字（~2012：[`gist.github.com/jboner/2841832`](https://gist.github.com/jboner/2841832)）：

| **请求** | **时间** |
| --- | --- |
| L1 缓存引用 | 0.5 ns |
| 分支预测错误 | 5 ns |
| L2 缓存引用 | 7 ns |
| 锁定/解锁互斥锁 | 25 ns |
| 主内存引用 | 100 ns |
| 使用 Zippy 压缩 1 Kb | 3000 ns |
| 在 1 Gbps 网络上发送 1 Kb | 10000 ns |
| 从 SSD 随机读取 4 Kb | 150000 ns |
| 从内存中顺序读取 1 Mb | 250000 ns |
| 同一数据中心内的往返 | 500000 ns |
| 发送数据包 CA &#124; 荷兰 &#124; CA | 150000000 ns |

在这里，我们可以看到具体的数字，如果你想吃甜甜圈和一些咖啡，那么在你从丹麦咬第一口之前，你可以在电脑旁边的冰箱里吃掉 3 亿个甜甜圈。

# 限制大 O

大 O 记号是计算机科学中的一个术语，用于根据输入值增大时函数增长的速度来分组函数。这个术语最常用于算法的运行时间或空间需求。

当在软件工程中使用这个术语时，我们通常关注以下四种情况之一：

+   常数

+   对数增长

+   多项式增长

+   指数增长

当我们关注应用程序性能时，考虑你使用的逻辑的大 O 效率是好的。根据你处理的前四个案例中的哪一个，优化策略的适当反应可能会改变。

# 持续无增长

常数时间操作是运行性能的不可分割的单位。在前一节中，我们提供了一个常见操作及其所需时间的表格。对我们程序员来说，这些都是基本物理常数。你不能优化光速使其更快。

然而，并非所有常数时间操作都是不可减少的。如果你有一个对固定大小数据进行固定数量操作的程序，那么它将是常数时间。这并不意味着该程序自动是高效的。在尝试优化常数时间程序时，问问自己这两个问题：

+   是否有任何工作可以避免？

+   冰箱离电脑太远了吗？

这里有一个强调常数时间操作的程序：

```rs
fn allocate() -> [u64; 1000] {
   [22; 1000]
}

fn flop(x: f64, y: f64) -> f64 {
   x * y
}

fn lookup(x: &[u64; 1000]) -> u64 {
   x[234] * x[345]
}

fn main() {
   let mut data = allocate();
   for _ in 0..1000 {
      //constant size memory allocation
      data = allocate();
   }

   for _ in 0..1000000 {
      //reference data
      lookup(&data);
   }

   for _ in 0..1000000 {
      //floating point operation
      flop(2.0, 3.0);
   }
}
```

然后，让我们分析这个程序：

```rs
Profiling performance_constant with callgrind...

Total Instructions...896,049,080

217,133,740 (24.2%) ???:_platform_memmove$VARIANT$Haswell
-----------------------------------------------------------
108,054,000 (12.1%) ???:core::ptr
-----------------------------------------------------------
102,051,069 (11.4%) ???:core::iter::range
-----------------------------------------------------------
76,038,000 (8.5%) ???:<i32
-----------------------------------------------------------
56,028,000 (6.3%) ???:core::ptr
-----------------------------------------------------------
46,023,000 (5.1%) ???:core::iter::range::ptr_try_from_impls
-----------------------------------------------------------
45,027,072 (5.0%) ???:performance_constant
-----------------------------------------------------------
44,022,000 (4.9%) ???:core::ptr
-----------------------------------------------------------
40,020,000 (4.5%) ???:core::mem
-----------------------------------------------------------
30,015,045 (3.3%) ???:core::cmp::impls
```

我们看到，大量的内存分配相当昂贵。至于内存访问和浮点运算，它们似乎被多次执行的循环的开销所压倒。除非在常数时间过程中有明显的性能不佳的原因，否则优化此代码可能并不简单。

# 对数增长

对数算法是计算机科学的骄傲。如果你的代码对于 n=5 的 O(n)复杂度可以用 O(log n)算法编写，那么至少会有一个人指出这一点。

二分搜索是 O(log n)。排序通常是 O(n log n)。任何包含对数的都是更好的。这种喜爱并非没有道理。对数增长有一个惊人的特性——随着输入值的增加，增长速度会减慢。

这里有一个强调对数增长的程序。我们初始化一个大小为 1000 或 10000 的随机数向量。然后我们使用内置库进行排序并执行 100 次二分搜索操作。首先让我们捕捉 1000 个案例的排序和搜索时间：

```rs
extern crate rand;
extern crate flame;
use std::fs::File;

fn main() {
   let mut data = vec![0; 1000];
   for di in 0..data.len() {
      data[di] = rand::random::<u64>();
   }

   flame::start("sort n=1000");
   data.sort();
   flame::end("sort n=1000");

   flame::start("binary search n=1000 100 times");
   for _ in 0..100 {
      let c = rand::random::<u64>();
      data.binary_search(&c).ok();
   }
   flame::end("binary search n=1000 100 times");
```

现在我们分析 10000 个案例：

```rs
   let mut data = vec![0; 10000];
   for di in 0..data.len() {
      data[di] = rand::random::<u64>();
   }

   flame::start("sort n=10000");
   data.sort();
   flame::end("sort n=10000");

   flame::start("binary search n=10000 100 times");
   for _ in 0..100 {
      let c = rand::random::<u64>();
      data.binary_search(&c).ok();
   }
   flame::end("binary search n=10000 100 times");

   flame::dump_html(&mut File::create("flame-graph.html").unwrap()).unwrap();
}
```

运行此程序并检查火焰图后，我们可以看到，对于 10 倍更大的向量进行排序所需的时间几乎只增加了 10 倍——`O(n log n)`。搜索性能几乎不受影响——`O(log n)`。因此，对于实际应用，对数增长几乎可以忽略不计。

在尝试优化对数代码时，遵循与常数时间优化相同的方法。对数复杂度通常不是优化的好目标，尤其是考虑到对数复杂度是良好算法设计的强烈指标。

# 多项式增长

大多数算法都是多项式时间复杂度。

如果你有一个`for`循环，那么你的复杂度是*O(n*)。这在上面的代码中显示：

```rs
fn main() {
   for _ in 0..1000 {
      //O(n)
      //n = 1000
   }
}
```

如果你有两个`for`循环，那么你的复杂度是*O(n²*)：

```rs
fn main() {
   for _ in 0..1000 {
      for _ in 0..1000 {
         //O(n²)
         //n = 1000
      }
   }
}
```

高阶多项式相对较少见。有时代码意外地变成了高阶多项式，你应该小心对待；否则，让我们只考虑前两种情况。

线性复杂度非常常见。每次你处理集合中的所有数据时，复杂度将是线性的。线性算法的运行时间将大约是处理的项目数量（*n*）乘以处理单个项目的时间（*c*）。如果你想使线性算法更快，你需要：

+   减少处理的项目数量（n）

+   减少处理一个项目（*c*）相关的常数时间

如果处理一个项目的时间不是常数或近似常数，那么你的整体时间复杂度现在将递归地依赖于那个处理时间。以下代码展示了这一点：

```rs
fn a(n: u64) {
   //Is this O(n)?
   for _ in 0..n {
      b(n)
   }
}

fn b(n: u64) {
   //Is this O(n)?
   for _ in 0..n {
      c(n)
   }
}

fn c(n: u64) {
   //This is O(n)
   for _ in 0..n {
      let _ = 1 + 1;
   }
}

fn main() {
   //What time complexity is this?
   a(1000)
}
```

高阶多项式复杂度也很常见，但可能表明你的算法设计得不好。在前面的描述中，我们提到线性处理时间可能依赖于处理单个项目的时间。如果你的程序设计得草率，那么很容易将三个或四个线性算法串联起来，无意中创建一个*O*(*n*⁴)的怪物。

高阶多项式按比例更慢。对于需要高阶多项式计算的算法，通常可以通过剪枝来移除冗余或完全不必要的数据计算。考虑以下代码：

```rs
extern crate rusty_machine;
use rusty_machine::linalg::{Matrix,Vector};
use rusty_machine::learning::gp::{GaussianProcess,ConstMean};
use rusty_machine::learning::toolkit::kernel;
use rusty_machine::learning::SupModel;

fn main() {
   let inputs = Matrix::new(3,3,vec![1.1,1.2,1.3,2.1,2.2,2.3,3.1,3.2,3.3]);
   let targets = Vector::new(vec![0.1,0.8,0.3]);
   let test_inputs = Matrix::new(2,3, vec![1.2,1.3,1.4,2.2,2.3,2.4]);
   let ker = kernel::SquaredExp::new(2., 1.);
   let zero_mean = ConstMean::default();
   let mut gp = GaussianProcess::new(ker, zero_mean, 0.5);

   gp.train(&inputs, &targets).unwrap();
   let _ = gp.predict(&test_inputs).unwrap();
}
```

当你需要使用高阶多项式算法时，使用库！这些内容很快就会变得复杂，改进这些算法是学术计算机科学家的主要工作。如果你正在对常见算法进行性能调优，并且不打算发表你的结果，那么你很可能会重复工作。

# 指数增长

在工程中，指数性能几乎总是错误或死胡同。这是我们将使用的算法与我们希望使用但无法因为性能原因使用的算法之间的墙。

程序中的指数增长经常伴随着术语“炸弹”：

```rs
fn bomb(n: u64) -> u64 {
   if n > 0 {
      bomb(n-1);
      bomb(n-1);
   }
   n
}

fn main() {
   bomb(1000);
}
```

这个程序只有*O*(2^n)，因此几乎连指数增长都算不上！

# 引用数据更快

有一个经验法则，即引用数据比复制数据更快。同样，复制数据比克隆数据更快。这并不总是正确的，但当你试图提高程序性能时，这是一个值得考虑的好规则。

这里有一个函数，它交替使用通过引用、复制、内建克隆或自定义克隆的数据：

```rs
extern crate flame;
use std::fs::File;

fn byref(n: u64, data: &[u64; 1024]) {
   if n>0 {
      byref(n-1, data);
      byref(n-1, data);
   }
}

fn bycopy(n: u64, data: [u64; 1024]) {
   if n>0 {
      bycopy(n-1, data);
      bycopy(n-1, data);
   }
}

struct DataClonable([u64; 1024]);
impl Clone for DataClonable {
   fn clone(&self) -> Self {
      let mut newdata = [0; 1024];
      for i in 0..1024 {
         newdata[i] = self.0[i];
      }
      DataClonable(newdata)
   }
}

fn byclone<T: Clone>(n: u64, data: T) {
   if n>0 {
      byclone(n-1, data.clone());
      byclone(n-1, data.clone());
   }
}
```

这里我们声明了一个包含`1024`个元素的数组。然后使用火焰图分析库应用上述函数来测量引用、复制和克隆性能之间的差异：

```rs
fn main() {
   let data = [0; 1024];
   flame::start("by reference");
   byref(15, &data);
   flame::end("by reference");

   let data = [0; 1024];
   flame::start("by copy");
   bycopy(15, data);
   flame::end("by copy");

   let data = [0; 1024];
   flame::start("by clone");
   byclone(15, data);
   flame::end("by clone");

   let data = DataClonable([0; 1024]);
   flame::start("by clone (with extras)");
   //2⁴ instead of 2¹⁵!!!!
   byclone(4, data);
   flame::end("by clone (with extras)");

   flame::dump_html(&mut File::create("flame-graph.html").unwrap()).unwrap();
}
```

观察这个应用程序的运行时间，我们看到与复制或克隆此数据相比，引用的数据只使用了很小的一部分资源。默认的克隆和复制特性不出所料给出了相似的性能。自定义克隆实际上非常慢。它在语义上与所有其他操作相同，但在底层优化方面并不一样。

# 通过防御性编程防止错误

你不需要修复永远不会发生的错误。预防性医学是好的软件工程，从长远来看会为你节省时间。

# 使用 Option 和 Result 而不是 panic!

在许多其他语言中，异常处理是通过 `try…catch` 块来执行的。Rust 并不自动提供这种功能，相反，它鼓励程序员显式地局部化所有的错误处理。

在许多 Rust 上下文中，如果你不想处理错误处理，你总是可以选择使用 `panic!`。这将立即结束程序并提供一个简短的错误消息。不要这样做。恐慌通常只是避免处理错误责任的一种方式。

相反，使用 `Option` 或 `Result` 类型来传达错误或异常情况。`Option` 表示没有值可用。`Option` 的 `None` 值应表示没有值，但一切正常且符合预期。

`Result` 类型用于传达处理过程中是否出现错误。`Result` 类型可以与 `?` 语法结合使用，以传播错误同时避免引入过多的额外语法。`?` 操作将返回函数中的错误（如果有），因此该函数必须有一个 `Result` 返回类型。

在这里，我们创建了两个函数，它们返回 `Option` 或 `Result` 来处理异常情况。注意处理 `Result` 返回值时使用 try `?` 语法。这种语法将传递 `Ok` 值或立即返回该函数中的任何 `Err`。因此，任何使用 `?` 的函数也必须返回兼容的 `Result` 类型：

```rs
//This function returns an Option if the value is not expected
fn expect_1or2or_other(n: u64) -> Option<u64> {
   match n {
      1|2 => Some(n),
      _ => None
   }
}

//This function returns an Err if the value is not expected
fn expect_1or2or_error(n: u64) -> Result<u64,()> {
   match n {
      1|2 => Ok(n),
      _ => Err(())
   }
}

//This function uses functions that return Option and Return types
fn mixed_1or2() -> Result<(),()> {
   expect_1or2or_other(1);
   expect_1or2or_other(2);
   expect_1or2or_other(3);

   expect_1or2or_error(1)?;
   expect_1or2or_error(2)?;
   expect_1or2or_error(3).unwrap_or(222);
   Ok(())
}

fn main() {
   mixed_1or2().expect("mixed 1 or 2 is OK.");
}
```

`Result` 类型在与外部资源（如文件）交互时非常常见：

```rs
use std::fs::File;
use std::io::prelude::*;
use std::io;

fn lots_of_io() -> io::Result<()> {
   {
      let mut file = File::create("data.txt")?;
      file.write_all(b"data\ndata\ndata")?;
   }

   {
      let mut file = File::open("data.txt")?;
      let mut data = String::new();
      file.read_to_string(&mut data)?;
      println!("{}", data);
   }
   Ok(())
}

fn main() {
   lots_of_io().expect("lots of io is OK.");
}
```

# 使用类型安全的接口而不是字符串类型接口

Rust 中的枚举比使用数字或字符串更不容易出错。只要可能，编写以下代码：

```rs
const MyEnum_A: u32 = 1;
const MyEnum_B: u32 = 2;
const MyEnum_C: u32 = 3;
```

类似地，你可以编写一个字符串枚举：

```rs
"a"
"b"
"c"
```

最好使用以下枚举类型：

```rs
enum MyEnum {
   A,
   B,
   C,
}
```

这样，接受枚举类型的函数将会是类型安全的：

```rs
fn foo(n: u64) {} //not all u64 are valid inputs
fn bar(n: &str) {} //not all &str are valid inputs
fn baz(n: MyEnum) {} //all MyEnum are valid
```

枚举也自然地与模式匹配相结合，原因相同。对枚举进行模式匹配不需要像整数或字符串类型那样有一个最终的错误情况：

```rs
match a {
   1 => println!(“1 is ok”),
   2 => println!(“2 is ok”),
   3 => println!(“3 is ok”),
   n => println!(“{} was unexpected”, n)
}
```

# 使用心跳模式处理长时间运行的过程

当你想创建一个长时间运行的过程时，能够从程序错误中恢复，这些错误会导致进程崩溃或终止，这会很好。也许进程耗尽了栈空间，或者遇到了某些代码路径的`panic!`。由于任何数量的原因，一个进程可能会被终止，并需要重新启动。

为了满足这种需求，有许多工具会为你监视程序，并在它死亡或停止对健康检查做出响应时重新启动它。在这里，我们推荐一个基于 Rust 并发的完全自包含的这种模式版本。

目标是创建一个充当监控器并监督一个或多个工作进程的父进程。进程树应该看起来像这样：

```rs
parent
 —- child 1
 —- child 2
 —- child 3
```

当一个子进程死亡或停止对健康检查做出响应时，父进程应该终止或以其他方式清理进程资源，然后启动一个新的进程来替换它。以下是一个这种行为示例，从一个有时会死亡的子进程开始：

```rs
use std::{thread,time,process};

fn main() {
   let life_expectancy = process::id() % 8;
   let t = time::Duration::from_millis(1000);
   for _ in 0..life_expectancy {
      thread::sleep(t);
   }
   println!("process {} dies unexpectedly.", process::id());
}
```

这个工作进程非常不可靠，其寿命不超过八秒。然而，如果我们用心跳监控器将其包装起来，那么我们可以使其更加可靠：

```rs
use std::process::Command;
use std::env::current_exe;
use std::{thread,time};

fn main() {
   //There is an executable called debugging_buggy_worker
   //it crashes a lot but we still want to run it
   let path = current_exe()
             .expect("could not find current executable");
   let path = path.with_file_name("debugging_buggy_worker");
   let mut children = Vec::new();

   //we start 3 workers
   for _ in 0..3 {
      children.push(
         Command::new(path.as_os_str())
                 .spawn()
                 .expect("failed to spawn child")
      );
   }

   //those workers will randomly die because they are buggy
   //so after they die, we restart a new process to replace them
   let t = time::Duration::from_millis(1000);
   loop {
      thread::sleep(t);
      for ci in 0..children.len() {
         let is_dead = children[ci].try_wait().expect("failed to try_wait");
         if let Some(_exit_code) = is_dead {
            children[ci] = Command::new(path.as_os_str())
                                   .spawn()
                                   .expect("failed to spawn child");
            println!("starting a new process from parent.");
         }
      }
   }
}
```

现在，如果运行中的进程意外终止，它们将会被重新启动。可选地，父进程可以检查每个子进程的健康状态，并重新启动无响应的工作进程。

# 验证输入和输出

预设条件和后置条件是锁定程序行为并找到在失控之前可能出现的错误或无效状态的好方法。

如果你使用宏来做这件事，那么预设条件和后置条件可以可选地仅在调试模式下运行，并从生产代码中删除。内置的`debug_assert!`宏就是这样做的。然而，使用断言作为返回值并不特别优雅，如果你忘记检查带有返回语句的分支，那么你的后置条件将不会被检查。

`debug_assert!`不是验证任何依赖于外部数据或非确定性行为的任何内容的良好选择。当你想在生产代码中检查预设条件或后置条件时，你应该使用`Result`或`Option`值来处理异常行为。

这里有一些 Rust 中预设条件和后置条件的示例：

```rs
use std::io;

//This function checks the precondition that [n < 100]
fn debug_precondition(n: u64) -> u64 {
   debug_assert!(n < 100);
   n * n
}

//This function checks the postcondition that [return > 10]
fn debug_postcondition(n: u64) -> u64 {
   let r = n * n;
   debug_assert!(r > 10);
   r
}

//this function dynamically checks the precondition [n < 100]
fn runtime_precondition(n: u64) -> Result<u64,()> {
   if !(n<100) { return Err(()) };
   Ok(n * n)
}

//this function dynamically checks the postcondition [return > 10]
fn runtime_postcondition(n: u64) -> Result<u64,()> {
   let r = n * n;
   if !(r>10) { return Err(()) };
   Ok(r)
}

//This main function uses all of the functions
//The dynamically validated functions are subjected to user input
fn main() {
   //inward facing code should assert expectations
   debug_precondition(5);
   debug_postcondition(5);

   //outward facing code should handle errors
   let mut s = String::new();
   println!("Please input a positive integer greater or equal to 4:");
   io::stdin().read_line(&mut s).expect("error reading input");
   let i = s.trim().parse::<u64>().expect("error parsing input as integer");
   runtime_precondition(i).expect("runtime precondition violated");
   runtime_postcondition(i).expect("runtime postcondition violated");
}
```

注意，用户输入超出了我们的控制。验证用户输入的最佳选项是在输入无效时返回一个`Error`条件。

# 寻找和修复错误

调试工具很大程度上依赖于平台。在这里，我们将解释`lldb`，它可用，并且适用于 macOS 和其他类 Unix 系统。

要开始调试，你需要编译程序并开启调试符号。正常的`cargo debug build`通常足够：

```rs
cargo build
```

程序编译完成后，启动调试器：

```rs
$ sudo rust-lldb target/debug/deps/performance_polynomial3-8048e39c94dd7157
```

在这里，我们引用了`debugs/deps/program_name-GITHASH`程序的副本。这现在是因为 lldb 的工作方式。

运行`lldb`后，你将在启动时看到一些信息滚动过去。然后，你应该进入 LLDB 命令提示符：

```rs
(lldb) command source -s 0 '/tmp/rust-lldb-commands.YnRBkV'
Executing commands in '/tmp/rust-lldb-commands.YnRBkV'.
(lldb) command script import "/Users/andrewjohnson/.rustup/toolchains/nightly-x86_64-apple-darwin/lib/rustlib/etc/lldb_rust_formatters.py"
(lldb) type summary add --no-value --python-function lldb_rust_formatters.print_val -x ".*" --category Rust
(lldb) type category enable Rust
(lldb) target create "target/debug/deps/performance_polynomial3-8048e39c94dd7157"
Current executable set to 'target/debug/deps/performance_polynomial3-8048e39c94dd7157' (x86_64).
(lldb)

```

现在，设置一个断点。我们将设置一个断点在函数 `a` 处停止：

```rs
(lldb) b a
Breakpoint 1: where = performance_polynomial3-8048e39c94dd7157`performance_polynomial3::a::h0b267f360bbf8caa + 12 at performance_polynomial3.rs:3, address = 0x000000010000191c
```

现在我们已经设置了断点，运行 `r` 命令：

```rs
(lldb) r
Process 99468 launched: '/Users/andrewjohnson/subarctic.org/subarctic.org/Hands-On-Functional-Programming-in-RUST/Chapter09/target/debug/deps/performance_polynomial3-8048e39c94dd7157' (x86_64)
Process 99468 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
  frame #0: 0x000000010000191c performance_polynomial3-8048e39c94dd7157`performance_polynomial3::a::h0b267f360bbf8caa(n=1000) at performance_polynomial3.rs:3
   1    fn a(n: u64) {
   2       //Is this O(n);
-> 3       for _ in 0..n {
   4          b(n);
   5       }
   6    }
   7    
Target 0: (performance_polynomial3-8048e39c94dd7157) stopped.
```

在代码停止点停止后，LLDB 将打印出代码停止处的上下文。现在我们可以检查程序了。让我们打印出在这个函数中定义了哪些变量：

```rs
(lldb) frame variable
(unsigned long) n = 1000
```

我们可以类似地打印作用域内的任何变量：

```rs
(lldb) p n
(unsigned long) $0 = 1000
```

当我们想要继续程序时，输入 `c` 以继续：

```rs
(lldb) c
Process 99468 resuming
Process 99468 exited with status = 0 (0x00000000)
```

程序在这里退出，因为我们没有设置更多的断点。这种调试方法很棒，因为它允许你在不不断添加 `println!` 语句和重新编译的情况下检查运行中的程序。如果其他方法都不奏效，这仍然是一个可行的选择。

# 元编程

Rust 中的元编程有两种形式——宏和过程宏。这两种实用工具都接受抽象语法树作为新的输入和输出符号进行编译。过程宏与正常宏非常相似，但它们在如何工作以及如何定义方面有更少的限制。

使用 `macro_rules!` 语法定义的宏通过匹配输入语法来递归地定义输出。理解这一点至关重要：宏匹配发生在解析之后。这意味着以下内容：

+   宏在创建新的语法形式时必须遵循某些规则

+   AST 被装饰了有关每个节点语法类别的信息

宏可以匹配单个标记，或者宏可以匹配（并捕获）整个语法类别。Rust 的语法类别如下：

+   `tt`：这是一个标记树（这是在解析之前从词法分析器输出的标记）

+   `ident`：这是一个标识符

+   `expr`：这是一个表达式

+   `ty`：这是一个类型

+   `stmt`：这是一个语句

+   `block`：这些是包含语句块的括号

+   `item`：这是一个顶层定义，例如函数或结构体

+   `pat`：这是模式匹配表达式的匹配部分，也称为**左侧**

+   `path`：这是一个路径，例如 `std::fs::File`

+   `meta`：这是一个元项，可以放在 `#[...]` 或 `#![...]` 语法形式内部

使用这些模式，我们可以创建宏来匹配各种语法表达式组：

```rs
//This macro rule matches one token tree "tt"
macro_rules! match_tt {
   ($e: tt) => { println!("match_tt: {}", stringify!($e)) }
}

//This macro rule matches one identifier "ident"
macro_rules! match_ident {
   ($e: ident) => { println!("match_ident: {}", stringify!($e)) }
}

//This macro rule matches one expression "expr"
macro_rules! match_expr {
   ($e: expr) => { println!("match_expr: {}", stringify!($e)) }
}

//This macro rule matches one type "ty" 
macro_rules! match_ty {
   ($e: ty) => { println!("match_ty: {}", stringify!($e)) }
}

//This macro rule matches one statement "stmt"
macro_rules! match_stmt {
   ($e: stmt) => { println!("match_stmt: {}", stringify!($e)) }
}

//This macro rule matches one block "block"
macro_rules! match_block {
   ($e: block) => { println!("match_block: {}", stringify!($e)) }
}

//This macro rule matches one item "item"
//items are things like function definitions, struct definitions, ...
macro_rules! match_item {
   ($e: item) => { println!("match_item: {}", stringify!($e)) }
}

//This macro rule matches one pattern "pat"
macro_rules! match_pat {
   ($e: pat) => { println!("match_pat: {}", stringify!($e)) }
}

//This macro rule matches one path "path"
//A path is a canonical named path like std::fs::File
macro_rules! match_path {
   ($e: path) => { println!("match_path: {}", stringify!($e)) }
}

//This macro rule matches one meta "meta"
//A meta is anything inside of the #[...] or #![...] syntax
macro_rules! match_meta {
   ($e: meta) => { println!("match_meta: {}", stringify!($e)) }
}
```

然后，让我们将这些宏应用到不同的输入上：

```rs
fn main() {
   match_tt!(a);
   match_tt!(let);
   match_tt!(+);

   match_ident!(a);
   match_ident!(bcd);
   match_ident!(_def);

   match_expr!(1.2);
   match_expr!(bcd);
   match_expr!(1.2 + bcd / "b" - [1, 3, 4] .. vec![1, 2, 3]);

   match_ty!(A);
   match_ty!(B + 'static);
   match_ty!(A<&(B + 'b),&mut (C + 'c)> + 'static);

   match_stmt!(let x = y);
   match_stmt!(());
   match_stmt!(fn f(){});

   match_block!({});
   match_block!({1; 2});
   match_block!({1; 2 + 3});

   match_item!(struct A(u64););
   match_item!(enum B { C, D });
   match_item!(fn C(n: NotAType) -> F<F<F<F<F>>>> { a + b });

   match_pat!(_);
   match_pat!(1);
   match_pat!(A {b, c:D( d@3 )} );

   match_path!(A);
   match_path!(::A);
   match_path!(std::A);
   match_path!(a::<A,_>);

   match_meta!(A);
   match_meta!(Property(B,C));
}
```

从示例中我们可以看出，标记树在大多数情况下并不受正常 Rust 语法限制，只受 Rust 词法分析器的限制。词法分析器知道开闭括号 `() [] {}` 的形式。这就是为什么标记是以标记树而不是标记列表的形式组织的。这也意味着宏调用内的所有标记都将作为标记树存储，并且不会进一步处理，直到宏被调用；只要我们创建与 Rust 标记树兼容的语法，那么通常应该允许其他语法创新。这条规则也适用于其他语法类别：语法类别只是匹配某些标记模式的一种简写，这些模式恰好对应 Rust 语法形式。

仅匹配单个标记或语法类别可能对宏来说不太有用。为了在实用场景中使用宏，我们需要利用宏语法序列和语法替代项。语法序列是在同一规则中请求匹配多个标记或语法类别。语法替代项是在同一宏中的单独规则，它匹配不同的语法。语法序列和替代项也可以在同一宏中组合。此外，还有一个特殊的语法形式来匹配许多标记或语法类别。

下面是一些相应的示例来展示这些模式：

```rs
//this is a grammar sequence
macro_rules! abc {
   (a b c) => { println!("'a b c' is the only correct syntax.") };
}

//this is a grammar alternative
macro_rules! a_or_b {
   (a) => { println!("'a' is one correct syntax.") };
   (b) => { println!("'b' is also correct syntax.") };
}

//this is a grammar of alternative sequences
macro_rules! abc_or_aaa {
   (a b c) => { println!("'a b c' is one correct syntax.") };
   (a a a) => { println!("'a a a' is also correct syntax.") };
}

//this is a grammar sequence matching many of one token
macro_rules! many_a {
   ( $($a:ident)* ) => {{ $( print!("one {} ", stringify!($a)); )* println!(""); }};
   ( $($a:ident),* ) => {{ $( print!("one {} comma ", stringify!($a)); )* println!(""); }};
}

fn main() {
   abc!(a b c);

   a_or_b!(a);
   a_or_b!(b);

   abc_or_aaa!(a b c);
   abc_or_aaa!(a a a);

   many_a!(a a a);
   many_a!(a, a, a);
}
```

如果你注意到了所有这些宏生成的代码，你可能会注意到所有生产规则都创建了表达式。宏输入可以是标记，但输出必须是上下文中良好形成的 Rust 语法。因此，你不能像下面这样编写 `macro_rules!`：

```rs
macro_rules! f {
   () => { f!(1) f!(2) f!(3) };
   (1) => { 1 };
   (2) => { + };
   (3) => { 2 };
}

fn main() {
   f!()
}
```

编译器产生的具体错误如下：

```rs
error: macro expansion ignores token `f` and any following
--> t.rs:2:19
  |
2 |    () => { f!(1); f!(2); f!(3) };
  |                   ^
  |
note: caused by the macro expansion here; the usage of `f!` is likely invalid in expression context
--> t.rs:9:4
  |
9 |    f!()
  |    ^^^^

error: aborting due to previous error
```

这里的关键短语是 `f!`，在表达式上下文中可能是不合法的。`macro_rules!` 的每个输出模式都必须是一个良好形成的表达式。前面的例子最终将创建良好的 Rust 语法，但它的中间结果是碎片化的表达式。这种尴尬是使用过程宏的几个原因之一，过程宏与 `macro_rules!` 类似，但直接在 Rust 中编程，而不是通过特殊的 `macro_rules!` 语法。

过程宏是用 Rust 编程的，但也被用来编译 Rust 程序。这是怎么做到的？过程宏必须被隔离到它们自己的模块中并单独编译；它们基本上是一个编译器插件。

为了开始我们的过程宏，让我们创建一个新的子项目：

1.  在项目根目录下创建一个 `procmacro` 目录

1.  在 `procmacro` 目录中，创建一个包含以下内容的 `Cargo.toml` 文件：

```rs
[package]
name = "procmacro"
version = "1.0.0"

[dependencies]
syn = "0.12"
quote = "0.4"

[lib]
proc-macro = true
```

1.  在 `procmacro` 目录中，创建一个包含以下内容的 `src/lib.rs` 文件：

```rs
#![feature(proc_macro)]
#![crate_type = "proc-macro"]
extern crate proc_macro;
extern crate syn;
#[macro_use] extern crate quote;
use proc_macro::TokenStream;
#[proc_macro]

pub fn f(input: TokenStream) -> TokenStream {
   assert!(input.is_empty());

   (quote! {
      1 + 2
   }).into()
}
```

这个 `f!` 宏现在实现了前面的语义，没有任何抱怨。使用这个宏的示例如下：

```rs
#![feature(proc_macro_non_items)]
#![feature(use_extern_macros)]
extern crate procmacro;

fn main() {
   let _ = procmacro::f!();
}
```

过程宏的接口非常简单。有一个 `TokenStream` 作为输入，还有一个 `TokenStream` 作为输出。`proc_macro` 和 `syn` 包还提供了解析标记或使用 `quote!` 宏轻松创建标记流的实用工具。要使用过程宏，有一些额外的设置和样板代码，但过了这些障碍后，接口现在相当直接了。

此外，通过 `syn` crate，过程宏还有许多更详细的语法类别可供使用。目前有 163 个类别（[`dtolnay.github.io/syn/syn/#macros`](https://dtolnay.github.io/syn/syn/#macros)）！这些类别包括递归宏中的相同模糊的语法树，但也包括非常具体的语法形式。这些类别对应于完整的 Rust 语法，因此允许在不创建自己的解析器的情况下，使用非常表达性的宏语法。

让我们创建一个使用这些语法类别的过程宏。首先，我们创建一个新的过程宏文件夹，就像之前的 `procmacro` 一样；这个我们将命名为 `procmacro2`。现在我们定义将持有所有程序信息的 AST（抽象语法树），如果用户输入有效的话：

```rs
#![feature(proc_macro)]
#![crate_type = "proc-macro"]
extern crate proc_macro;
#[macro_use] extern crate syn;
#[macro_use] extern crate quote;
use proc_macro::TokenStream;
use syn::{Ident, Type, Expr, WhereClause, TypeSlice, Path};
use syn::synom::Synom;

struct MiscSyntax {
   id: Ident,
   ty: Type,
   expr: Expr,
   where_clause: WhereClause,
   type_slice: TypeSlice,
   path: Path
}
```

`MiscSyntax` 结构将包含从我们的宏中收集的所有信息。我们现在应该定义这个宏及其语法：

```rs
impl Synom for MiscSyntax {
   named!(parse -> Self, do_parse!(
      keyword!(where) >>
      keyword!(while) >>
      id: syn!(Ident) >>
      punct!(:) >>
      ty: syn!(Type) >>
      punct!(>>) >>
      expr: syn!(Expr) >>
      punct!(;) >>
      where_clause: syn!(WhereClause) >>
      punct!(;) >>
      type_slice: syn!(TypeSlice) >>
      punct!(;) >>
      path: syn!(Path) >>
      (MiscSyntax { id, ty, expr, where_clause, type_slice, path })
   ));
}
```

`do_parse!` 宏有助于简化 `syn` crate 中解析组合器的使用。`id: expr >>` 语法对应于单调绑定操作，而 `expr >>` 语法也是一种单调绑定。

现在我们利用这些定义来解析输入，生成输出，并暴露宏：

```rs
#[proc_macro]
pub fn misc_syntax(input: TokenStream) -> TokenStream {
   let m: MiscSyntax = syn::parse(input).expect("expected Miscellaneous Syntax");
   let MiscSyntax { id, ty, expr, where_clause, type_slice, path } = m;

   (quote! {
      let #id: #ty = #expr;
      println!("variable = {}", #id);
    }).into()
}
```

当使用这个宏时，它实际上是一堆随机的语法。这强调了宏并不局限于有效的 Rust 语法，它看起来如下：

```rs
#![feature(proc_macro_non_items)]
#![feature(use_extern_macros)]
extern crate procmacro2;

fn main() {
   procmacro2::misc_syntax!(
      where while abcd : u64 >> 1 + 2 * 3;
      where T: 'x + A<B='y+C+D>;
      [M];A::f
   );
}
```

如果 Rust 语法对您来说很烦人，过程宏非常强大且有用。在特定上下文中，可以使用宏创建非常语义密集的代码，否则这将需要大量的样板代码和复制粘贴编程。

# 摘要

在这一章中，我们介绍了 Rust 编程的许多应用和实践考虑因素。性能和调试当然不是仅限于函数式编程的问题。在这里，我们试图介绍一些普遍适用但高度兼容于函数式编程的技巧。

在 Rust 中，元编程可能被视为一种功能特性本身。逻辑编程及其由此派生的功能与函数式编程原则密切相关。宏的递归、上下文无关特性也使其适合于函数式视角。

这也是本书的最后一章。我们希望您喜欢这本书，并欢迎任何反馈。如果您正在寻找进一步阅读，您可能想研究一下书中最后三章中提出的某些主题。关于这些主题有大量的材料可用，并且任何选择的路径都将无疑进一步加深您对 Rust 和函数式编程的理解。

# 问题

1.  发布模式与调试模式有何不同？

1.  一个空的循环将运行多长时间？

1.  在 *Big O* 表示法中，线性时间是什么意思？

1.  请举一个比指数增长更快的函数的例子。

1.  磁盘读取和网络读取哪个更快？

1.  你会如何返回一个包含多个错误条件的 `Result`？

1.  什么是标记树？

1.  抽象语法树是什么？

1.  为什么过程宏需要单独编译？
