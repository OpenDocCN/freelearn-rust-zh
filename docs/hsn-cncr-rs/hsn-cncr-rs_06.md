# 第六章：原子 – 同步的原始元素

在第四章，*Sync 和 Send – Rust 并发的基石*，以及第五章，*锁 – Mutex、Condvar、屏障和 RWLock*中，我们讨论了 Rust 中基于锁的并发基础。然而，在某些情况下，基于锁的编程并不适用，例如当极端性能是关注点或线程可能*永远不会*阻塞时。在这些领域，程序员必须依赖现代 CPU 的原子同步原语。

在本章中，我们将讨论 Rust 程序员可用的原子原语。使用原子编程是一个复杂的话题，也是一个活跃的研究领域。整本书都可以专门讨论原子 Rust 编程。因此，我们将为本章稍微调整策略，更多地采用教程风格，而不是之前章节中对现有软件的深入研究。前几章中介绍的所有内容都将在此处得到应用。到本章结束时，你应该对原子有实际的理解，能够更容易地消化现有文献，并验证你的实现。

到本章结束时，我们将：

+   讨论了可线性化性的概念

+   讨论各种原子内存排序，以及它们的含义和影响

+   从原子原语构建互斥锁

+   从原子原语构建队列

+   从原子原语构建信号量

+   清晰地阐述了在原子上下文中内存回收的困难

# 技术要求

本章需要安装一个可工作的 Rust 环境。验证安装的详细信息请参阅第一章，*预备知识 – 计算机架构和 Rust 入门*。不需要额外的软件工具。

您可以在 GitHub 上找到本书项目的源代码：[`github.com/PacktPublishing/Hands-On-Concurrency-with-Rust`](https://github.com/PacktPublishing/Hands-On-Concurrency-with-Rust)。本章的源代码位于`Chapter06`。

# 可线性化性

到目前为止，本书在指定并发系统时，一直避免使用正式术语，因为虽然它们在讨论思想和推理时非常有用，但如果没有一些背景知识，它们可能很难学习。现在我们已经有了背景知识，是时候了。

我们如何决定一个并发算法是否正确？如果我们对算法感兴趣，我们如何分析实现并推理其正确性？到目前为止，我们使用技术通过随机和重复测试以及模拟（在 helgrind 的情况下）来证明实现的适用性。我们将继续这样做。实际上，如果我们只是这样做，证明实现的适用性，那么我们会处于相当好的状态。工作的程序员会发现他们经常在发明，将算法改编以适应某些新颖的领域——正如前一章关于 hopper 的讨论中所见。这样做很容易，而且往往不会注意到。

当我们坐下来推理我们所构建的系统时，我们正在寻找的是一个统一的概念，这个概念能够将可行的想法与错误区分开来。对我们来说，在这里，在设计并发数据结构时，这个概念就是线性化。我们想要构建的对象，用 Nancy Lynch 的话来说，使得对这些对象的操作似乎是一次一个，按照某种顺序发生，与操作的顺序和响应一致。Lynch 的《分布式算法》关注分布式系统的行为，但我一直认为她对所谓的原子对象的解释非常简洁而精辟。

线性化的概念是在 1986 年 Herlihy 和 Wing 的《并发对象的公理》一文中提出的。他们的定义稍微复杂一些，涉及到历史——*一系列操作调用和响应事件**——以及规范，*简而言之，就是一组关于对象上定义的操作的公理前条件和后条件。

让操作的成果称为 `res(op)`，操作的调用或应用称为 `inv(op)`，并通过一些额外的标识符来区分操作。`op_1` 与 `op_2` 不同，但标识符不影响它们的排序；它只是一个名字。历史 `H` 是操作上的一个偏序，使得 `op_0 < op_1` 如果在 `H` 中 `res(op_0)` 发生在 `inv(op_1)` 之前。在 `<` 没有排序关系的操作在历史中是并发的。如果历史 `H` 中所有操作都按照 `<` 排序，则 `H` 是顺序的。现在，如果历史 `H` 可以通过添加零个或多个操作进行调整，使得另一个历史 `H'` 如下，则 `H` 是可线性化的：

1.  `H'` 只包含调用和响应，并且与某个合法的顺序历史 `S` 等价。

1.  `H'` 的排序操作是 `S` 排序的包含子集。

哇！如果一个对象的操作顺序可以被记录下来，对其进行一些调整，并找到一个等效的同一对象上的操作序列应用，那么这个对象就是可线性化的。我们实际上在之前的章节中已经进行了线性化分析，只是我没有将其称为此类。让我们继续跟随 Herlihy 和 Wing，看看他们关于可线性化与不可线性化历史记录的例子。以下是一个熟悉的队列的历史记录：

```rs
A: Enq(x)       0
B: Enq(y)       1
B: Ok(())       2
A: Ok(())       3
B: Deq()        4
B: Ok(x)        5
A: Deq()        6
A: Ok(y)        7
A: Enq(z)       8
```

我们有两个线程，`A`和`B`，执行操作`enq(val: T) -> Result((), Error)`和`deq() -> Result(T, Error)`，以伪 Rust 类型表示。这个历史记录实际上是可线性化的，并且与以下等效：

```rs
A: Enq(x)       0
A: Ok(())       3
B: Enq(y)       1
B: Ok(())       2
B: Deq()        4
B: Ok(x)        5
A: Deq()        6
A: Ok(y)        7
A: Enq(z)       8
A: Ok(())
```

注意，在原始历史记录中最后一步操作并不存在。正如 Herlihy 和 Wing 所指出的，即使一个操作在其响应之前*生效*，历史记录仍然可能是可线性化的。例如：

```rs
A: Enq(x)       0
B: Deq()        1
C: Ok(x)        2
```

这与以下顺序历史记录等效：

```rs
A: Enq(x)       0
A: Ok(())
B: Deq()        1
B: Ok(x)        2
```

这个历史记录是不可线性化的：

```rs
A: Enq(x)       0
A: Ok(())       1
B: Enq(y)       2
B: Ok(())       3
A: Deq()        4
A: Ok(y)        5
```

在这种情况下，罪魁祸首是队列排序的违反。在顺序历史记录中，出队操作将接收*x*而不是*y*。

就这些了。当我们谈论一个结构是*可线性化的*时，我们真正询问的是，针对该结构的任何有效操作列表是否可以被重新排序或添加，以至于列表与顺序历史记录无法区分？我们在上一章中查看的每个同步工具都被用来通过仔细排序线程间操作的顺序来强制线性化。在另一个层面上，这些工具也操纵了内存加载和存储的顺序。直接控制这种顺序，同时控制结构上操作的顺序，将占据本章的剩余部分。

# 内存排序——happens-before 和 synchronizes-with

每个 CPU 架构对内存排序——加载和存储之间的依赖关系——的处理方式不同。我们已经在第一章中详细讨论了这一点，*预备知识 – 机器架构和 Rust 入门*。在此简要说明，x86 是一个*强排序*架构；某些线程的存储操作将按执行顺序被所有其他线程看到。与此同时，ARM 是一个具有数据依赖性的弱排序架构；加载和存储可以以任何方式重新排序，除了那些会违反单个、孤立线程的行为之外，*并且*，如果一个加载依赖于之前加载的结果，你可以保证之前的加载将发生而不是被缓存。Rust 向程序员暴露了自己的内存排序模型，抽象掉了这些细节。因此，我们的程序必须根据 Rust 的模型*正确*，我们必须信任`rustc`能够正确地为我们目标 CPU 解释这些。如果我们破坏了 Rust 的模型，我们可能会看到`rustc`重新排序指令以破坏线性化，即使我们的目标 CPU 的保证原本可以帮助我们顺利通过。

我们在第三章“Rust 内存模型 – 所有权、引用和操作”中详细讨论了 Rust 的内存模型，指出其底层细节几乎完全继承自 LLVM（以及因此的 C/C++）。然而，我们推迟了对模型中依赖关系的讨论，直到本章。这个话题本身就足够复杂，您将直接看到这一点。由于这样做存在重大困难，我们不会在这里深入细节；请参阅 LLVM 的*并发操作的内存模型*([`llvm.org/docs/LangRef.html#memory-model-for-concurrent-operations`](http://llvm.org/docs/LangRef.html#memory-model-for-concurrent-operations))及其链接树，以获得全面且具有挑战性的阅读材料。相反，我们将指出，除了编写优化器或特别具有挑战性的需要研究 LLVM 内存模型引用的低级代码之外，理解当我们在 Rust 中谈论排序时，我们谈论的是一种特定的因果关系——*发生之前*/*同步于*就足够了。只要我们根据这种因果关系来结构我们的操作，并能证明线性化，我们就会处于良好的状态。

发生之前/同步于因果关系是什么？这些中的每一个都指代一种特定的排序关系。假设我们有三个操作`A`、`B`和`C`。我们说`A` *发生之前* `B`，如果：

+   `A`和`B`在同一线程上执行，并且`A`根据程序顺序先执行，然后`B`执行

+   `A`发生之前`C`，且`C`发生之前`B`

+   `A` *同步于* `B`

注意，第一个点指的是单线程程序顺序。Rust 不保证多线程程序顺序，但确实断言单线程代码将按照其明显的文本顺序运行，即使实际上优化的 Rust 被重新排序，底层硬件也在重新排序。如果我们说`A` *同步于* `B`，如果以下所有条件都成立：

+   `B`是从某个具有`Acquire`或`SeqCst`排序的原子变量`var`中加载

+   `A`是将具有`Release`或`SeqCst`排序的值存储到相同的原子变量`var`中

+   `B`从`var`读取值

最后一点存在是为了确保编译器或硬件不会优化掉第一点中的加载。但是，是否有一些额外的信息卡在了那里？是的！稍后我们将详细介绍。

线性化是如何与我们所描述的因果排序联系起来的？理解这一点非常重要。在 Rust 中，一个结构可以遵循因果排序，但在检查时仍然不能线性化。以之前章节中关于环形结构的讨论为例，尽管因果排序被互斥锁保护，但由于偏移量错误，写入被覆盖。那个实现不是可线性化的，但它是有因果排序的。从这个意义上说，那么，获取你程序的因果性是编写适合用途的并发数据结构的必要条件，但不是充分条件。线性化是针对结构上的操作进行的分析，而 happens-before/synchronizes-with 是一种在操作内部和操作之间发生的排序。

现在，我们提到了`Acquire`、`SeqCst`和`Release`；它们是什么？Rust 的内存排序模型允许对加载和存储应用不同的原子排序。这些排序控制着底层 CPU 和`rustc`优化器的指令调度，以及何时使流水线等待加载的结果。这些排序在名为`std::sync::atomic::Ordering`的枚举中定义。我们将逐一介绍它们。

# Ordering::Relaxed

这种排序并没有出现在“happens-before/synchronizes-with”的定义中。这有一个非常好的原因。`Relaxed`排序意味着没有保证。带有`Relaxed`排序的加载和存储可以自由地围绕一个加载或存储进行重新排序。也就是说，加载和存储可以从`Relaxed`操作的角度在程序顺序中向上或向下迁移。

编译器和硬件在`Relaxed`排序出现时可以随意操作。正如我们将在接下来的详细示例中看到的那样，这有很多实用价值。现在，考虑一下用于自我遥测的并发程序中的计数器，或者设计为最终一致性的数据结构。那里不需要严格的排序。

在我们继续之前，值得非常清楚地说明我们所说的加载和存储的排序是什么意思。加载和存储在*什么*上？嗯，原子操作。在 Rust 中，这些在`std::sync::atomic`中公开，截至撰写本书时，有四种稳定的原子类型可用：

+   `AtomicUsize`

+   `AtomicIsize`

+   `AtomicPtr`

+   `AtomicBool`

目前有一个围绕使用 Rust 问题`#32976`来支持更小的原子类型进行的讨论——你会注意到，在稳定的四种类型中，三种是机器字大小，而`AtomicBool`目前也是其中之一——但那里的进展似乎已经停滞，因为没有找到合适的领导者。所以，现在这些是类型。当然，除了原子操作之外，还有更多的负载和存储——以原始指针的操作为例。这些是数据访问，并且没有比 Rust 一开始内置的更多的排序保证。两个线程可能在同一时刻对相同的原始指针进行负载和存储，而且如果没有事先协调，对此无能为力。

# Ordering::Acquire

`Acquire`是一种处理负载的排序。回想一下，我们之前说过，如果一个线程`B`以`Acquire`排序加载变量，而线程`A`随后以`Release`排序存储，那么`A`与`B`同步，这意味着`A`发生在`B`之前。回想一下，在前一章中，我们在 hopper 的上下文中遇到了`Acquire`：

```rs
    pub unsafe fn push_back(
        &self,
        elem: T,
        guard: &mut MutexGuard<BackGuardInner<S>>,
    ) -> Result<bool, Error<T>> {
        let mut must_wake_dequeuers = false;
        if self.size.load(Ordering::Acquire) == self.capacity {
            return Err(Error::Full(elem));
        } else {
            assert!((*self.data.offset((*guard).offset)).is_none());
            *self.data.offset((*guard).offset) = Some(elem);
            (*guard).offset += 1;
            (*guard).offset %= self.capacity as isize;
            if self.size.fetch_add(1, Ordering::Release) == 0 {
                must_wake_dequeuers = true;
            }
        }
        Ok(must_wake_dequeuers)
    }
```

那里的大小是一个`AtomicUsize`，当前线程正在使用它来确定是否有空间放置新元素。一旦元素被放置，线程将使用`Release`排序来增加大小。这……似乎很荒谬。显然，增加大小的程序顺序在容量检查之前发生是不直观的。而且，这是真的。值得记住的是，游戏中还有另一个线程：

```rs
    pub unsafe fn pop_front(&self) -> T {
        let mut guard = self.front_lock.lock()
                          .expect("front lock poisoned");
        while self.size.load(Ordering::Acquire) == 0 {
            guard = self.not_empty
                .wait(guard)
                .expect("oops could not wait pop_front");
        }
        let elem: Option<T> = mem::replace(&mut 
        *self.data.offset((*guard).offset), None);
        assert!(elem.is_some());
        *self.data.offset((*guard).offset) = None;
        (*guard).offset += 1;
        (*guard).offset %= self.capacity as isize;
        self.size.fetch_sub(1, Ordering::Release);
        elem.unwrap()
    }
```

考虑如果只有两个线程`A`和`B`，`A`只执行`push_back`操作，而`B`只执行`pop_front`操作会发生什么。`mutexes`函数用于为相同类型的线程提供隔离——那些执行推或弹操作的线程——因此在这个假设中是无关紧要的，可以忽略。所有重要的是原子操作。如果大小为零，并且`B`首先被调度，它将在大小为`Acquire`负载的`condvar`循环中清空。当`A`被调度并发现其元素有额外的容量时，该元素将被排队，一旦完成，大小将被`Release`存储，这意味着`B`中某个幸运的循环将在`A`成功执行`push_back`操作之后发生。

Rust 文档中关于`Acquire`的说明是：

"当与负载结合时，所有后续的负载都将看到在具有释放排序的其他线程中相同值的存储之前写入的数据。"

LLVM 文档说明：

"Acquire 提供了一种必要的屏障，以获取锁来使用正常负载和存储访问其他内存。"

我们应该如何理解这一点？`Acquire`负载阻止在其之后的负载和存储迁移到更高的程序顺序。然而，在`Acquire`之前的负载和存储可能会迁移到更低的程序顺序。将`Acquire`视为类似于锁的起点，但顶部是渗透的。

# Ordering::Release

如果 `Acquire` 是锁的起点，那么 `Release` 就是终点。实际上，LLVM 文档中提到：

"Release 与 Acquire 类似，但需要一个释放锁的屏障。"

这可能比 `lock` 可能暗示的要微妙一些。以下是 Rust 文档中的说明：

"当与存储结合时，所有之前的写入对执行 Acquire 排序的同一值的加载的其他线程都变得可见。"

在 `Acquire` 停止其后的加载和存储向上迁移时，`Release` 停止其之前的加载和存储向下迁移，关于程序顺序。就像 `Acquire` 一样，`Release` 不会阻止其之前的加载和存储向上迁移。一个 `Release` 存储类似于锁的终点，但底部是渗透性的。

顺便说一下，这里有一个有趣的复杂情况，锁的直觉在这里并不适用。问题：两个 `Acquire` 加载或两个 `Release` 存储可以同时发生吗？答案是，嗯，这取决于很多因素，但这是可能的。在先前的 hopper 示例中，自旋的 `pop_front` 线程并不会阻止 `push_back` 线程执行其大小检查，即使是通过永久持有大小直到 `Release` 来释放 `pop_front` 线程，即使只是临时地，直到 while 循环可以被检查。记住，语言的因果关系模型对两个 `Acquire` 加载或 `Release` 存储的排序一无所知。发生什么是不确定的，并且可能强烈依赖于你的硬件。我们所知道的是，`pop_front` 线程将不会部分地看到 `push_back` 对内存所做的存储；它将是全部或无的。在 `Acquire` 和 `Release` 之后的加载和存储作为一个单元出现。

# Ordering::AcqRel

这个变体是 `Acquire` 和 `Release` 的组合。当一个加载操作使用 `AcqRel` 执行时，就会使用 `Acquire` 并将其存储到 `Release`。`AcqRel` 不仅是一个便利的组合，它结合了 `Acquire` 和 `Release` 的排序行为——两边的排序都是渗透性的。也就是说，在 `AcqRel` 之后，加载和存储不能向上移动，就像 `Acquire` 一样，而在 `AcqRel` 之前的加载和存储也不能向下移动，就像 `Release` 一样。这是一个巧妙的技巧。

在继续到下一个排序变体之前，值得指出的是，到目前为止，我们只看到了线程以成对的方式执行 `Acquire`/`Release` 的示例，与另一个线程合作。这不必是这样。一个线程始终可以执行 `Acquire`，另一个线程始终可以执行 `Release`。因果关系定义是专门针对执行一个但不是两个的线程，当然，除非 `A` 等于 `B`。让我们分析一个例子：

```rs
A: store X                   1
A: store[Release] Y          2
B: load[Acquire] Y           3
B: load X                    4
B: store[Release] Z          5
C: load[Acquire] Z           6
C: load X                    7
```

在这里，我们有三个线程`A`、`B`和`C`，它们执行了一系列的普通加载和存储操作，以及原子加载和存储操作。`store[Release]`是一个原子存储操作，而存储不是。正如线性可化性部分所讨论的，我们给每个操作都编了号，但请注意，这并不代表时间线，因为数字只是方便的名称。我们能说这个例子中的因果关系是什么吗？我们知道：

+   `1 happens-before 2`

+   `2 synchronizes-with 3`

+   `3 happens-before 4`

+   `3 happens-before 5`

+   `5 synchronizes-with 6`

+   `6 happens-before 7`

我们将在本章后面看到更多此类分析。

# Ordering::SeqCst

最后，在`Relaxed`的对立面，存在`SeqCst`，代表 SEQuentially ConsiSTent（顺序一致）。`SeqCst`类似于`AcqRel`，但额外的好处是，在顺序一致的操作之间，所有线程之间都有一个总顺序，因此得名。`SeqCst`是一个相当严肃的同步原语，你只有在极有必要的情况下才应该使用它，因为强制所有线程之间的总顺序并不便宜；每个原子操作都将需要一个同步周期，无论这对你的 CPU 意味着什么。例如，互斥锁（mutex）是一个获取-释放结构——稍后我们会详细讨论这一点——但确实存在使用类似`SeqCst`这样的超级互斥锁的合理情况。例如，实现旧论文。在原子编程的早期，更宽松——但不是`Relaxed`——的内存顺序的确切后果并没有完全理解。你会看到一些文献会使用顺序一致的内存，但对此并不在意。至少，跟随并按需放松。这种情况可能比你想象的要多，至少对你的作者来说是这样。你的体验可能会有所不同。有些情况下，我们似乎陷入了`SeqCst`的困境。考虑一个多生产者、多消费者设置，如下所示，这是从 CppReference 中改编的（参见最后部分显示的 *进一步阅读* 部分）：

```rs
#[macro_use]
extern crate lazy_static;

use std::thread;
use std::sync::Arc;
use std::sync::atomic::{AtomicBool, AtomicUsize, Ordering};

lazy_static! {
static ref X: Arc<AtomicBool> = Arc::new(AtomicBool::new(false));
static ref Y: Arc<AtomicBool> = Arc::new(AtomicBool::new(false));
static ref Z: Arc<AtomicUsize> = Arc::new(AtomicUsize::new(0));
}

fn write_x() {
    X.store(true, Ordering::SeqCst);
}

fn write_y() {
    Y.store(true, Ordering::SeqCst);
}

fn read_x_then_y() {
    while !X.load(Ordering::SeqCst) {}
    if Y.load(Ordering::SeqCst) {
        Z.fetch_add(1, Ordering::Relaxed);
    }
}

fn read_y_then_x() {
    while !Y.load(Ordering::SeqCst) {}
    if X.load(Ordering::SeqCst) {
        Z.fetch_add(1, Ordering::Relaxed);
    }
}

fn main() {
    let mut jhs = Vec::new();
    jhs.push(thread::spawn(write_x)); // a
    jhs.push(thread::spawn(write_y)); // b
    jhs.push(thread::spawn(read_x_then_y)); // c
    jhs.push(thread::spawn(read_y_then_x)); // d
    for jh in jhs {
        jh.join().unwrap();
    }
    assert!(Z.load(Ordering::Relaxed) != 0);
}
```

假设我们不在我们的示例线程之间强制执行`SeqCst`。那么会怎样？线程`c`会循环，直到线程`a`翻转布尔值`X`，加载`Y`，如果为真，则增加`Z`。同样，对于线程`d`，除了`b`翻转布尔值`Y`，增加操作由`X`保护。为了使程序在所有线程挂起后不会崩溃，线程必须看到`X`和`Y`的翻转以相同的顺序发生。请注意，顺序本身并不重要，重要的是它对每个线程来说都是相同的。否则，存储和加载可能会以某种方式交错，从而避免增加。强烈鼓励读者自己尝试调整顺序。

# 构建同步

现在我们已经对 Rust 标准库中可用的原子原语有了良好的基础，并且还有坚实的理论基础，是时候在此基础上构建了。在过去几章中，我们一直在暗示我们的意图是从原语构建互斥锁和信号量，嗯，现在时机已经成熟。

# 互斥锁

现在我们已经理解了线性化和内存排序，让我们问自己一个问题。互斥锁 *究竟* 是什么？我们知道它作为一个原子对象具有以下属性：

+   互斥锁支持两种操作，*锁定* 和 *解锁*。

+   互斥锁要么是 *锁定* 的，要么是 *未锁定* 的。

+   操作 *锁定* 将将互斥锁移动到 *锁定* 状态，仅当互斥锁是 *未锁定* 的。完成 *锁定* 的线程被称为持有锁。

+   操作 *解锁* 将将互斥锁移动到 *未锁定* 状态，仅当互斥锁之前是 *锁定* 的，并且 *解锁* 的调用者是锁的持有者。

+   在程序顺序中，在 *锁定* 之后和之前发生的所有加载和存储操作不得移动到 *锁定* 操作之前或之后。

+   在程序顺序中，在 *解锁* 之前发生的所有加载和存储操作不得移动到 *解锁* 之后。

第二个到最后一个点很微妙。考虑一个软互斥锁，其锁只保留迁移后的加载和存储。这意味着加载/存储可以迁移到软互斥锁中，这很好，除非你的迁移是另一个软互斥锁的锁。考虑以下情况：

```rs
lock(m0)
unlock(m0)
lock(m1)
unlock(m1)
```

这可以重新排列为以下内容：

```rs
lock(m0)
lock(m1)
unlock(m0)
unlock(m1)
```

这里，`unlock(m0)` 已经在程序顺序中迁移到 `m1` 的互斥排他区域，导致程序死锁。这是一个问题。

你可能不会惊讶地了解到，到目前为止，关于互斥排他问题的研究已有几十年。互斥锁应该如何设计？公平性因素如何影响实现？后者是一个相当广泛的话题，避免线程 *饥饿*，我们在这里或多或少会避开这个问题。大部分历史研究都是针对没有并发控制概念的机器。Lamport 的面包店算法——参见 *进一步阅读* 部分——提供了任何硬件支持中都不存在的互斥排他，但假设读取和写入是顺序一致的，这个假设在现代内存层次结构中并不成立。在很大意义上，我们生活在一个非常美好的并发编程时代：我们的机器直接暴露同步原语，极大地简化了快速和正确同步机制的生产。

# 比较并设置互斥锁

首先，让我们看看一个非常简单的互斥锁实现，一个在循环中旋转直到正确条件出现的原子交换互斥锁。也就是说，让我们构建一个 *自旋锁*。这个互斥锁之所以这样命名，是因为每个在 `lock` 上阻塞的线程都在消耗 CPU，在它获得锁的条件上自旋。无论如何，你很快就会看到。我们首先放下我们的 `Cargo.toml`：

```rs
[package]
name = "synchro"
version = "0.1.0"
authors = ["Brian L. Troutwine <brian@troutwine.us>"]

[[bin]]
name = "swap_mutex"
doc = false

[[bin]]
name = "status_demo"
doc = false

[[bin]]
name = "mutex_status_demo"
doc = false

[[bin]]
name = "spin_mutex_status_demo"
doc = false

[[bin]]
name = "queue_spin"
doc = false

[[bin]]
name = "crossbeam_queue_spin"
doc = false

[dependencies]
crossbeam = { git = "https://github.com/crossbeam-rs/crossbeam.git", rev = "89bd6857cd701bff54f7a8bf47ccaa38d5022bfb" }

[dev-dependencies]
quickcheck = "0.6"
```

现在，我们需要 `src/lib.rs`:

```rs
extern crate crossbeam;

mod queue;
mod swap_mutex;
mod semaphore;

pub use semaphore::*;
pub use swap_mutex::*;
pub use queue::*;
```

这里有相当一部分内容我们暂时不会直接使用，但我们会很快涉及到。现在，让我们深入到`src/swap_mutex.rs`文件。首先，是我们的前言：

```rs
extern crate crossbeam;

mod queue;
mod swap_mutex;
mod semaphore;
```

到这个阶段，这里并没有太多惊喜。我们看到通常的导入，还有一些新的导入——`AtomicBool`和`Ordering`，正如之前讨论的那样。我们自行实现了`Send`和`Sync`——在上一章中讨论过——因为虽然*我们*可以证明我们的`SwapMutex<T>`是线程安全的，但 Rust 不能。`SwapMutex<T>`很小：

```rs
pub struct SwapMutex<T> {
    locked: AtomicBool,
    data: *mut T,
}
```

字段`locked`是一个`AtomicBool`，我们将使用它来在线程之间提供隔离。这个想法很简单——如果一个线程出现并发现`locked`是`false`，那么这个线程可以获取锁，否则它必须自旋。还值得注意的是，我们的实现稍微有些冒险，因为它保留了对`T`的原始指针。截至撰写本书时，Rust 标准库`std::sync::Mutex`将其内部数据存储在`UnsafeCell`中，而我们无法在稳定版本中访问它。在这个实现中，我们只是存储指针，并不通过它进行任何操作。尽管如此，我们仍然必须小心处理`SwapMutex`的析构。创建`SwapMutex`的方式可能正如你所预期的那样：

```rs
impl<T> SwapMutex<T> {
    pub fn new(t: T) -> Self {
        let boxed_data = Box::new(t);
        SwapMutex {
            locked: AtomicBool::new(false),
            data: Box::into_raw(boxed_data),
        }
    }
```

`T`被移动到`SwapMutex`中，然后我们将其装箱以获取其原始指针。这是必要的，以避免栈值被推入`SwapMutex`，然后在栈帧改变时迅速消失，所有这些都已经在第二章中详细讨论过，*顺序 Rust 性能和测试*。互斥锁始终处于未锁定状态，否则将无法获取它。现在，让我们看看如何锁定互斥锁：

```rs
    pub fn lock(&self) -> SwapMutexGuard<T> {
        while self.locked.swap(true, Ordering::AcqRel) {
            thread::yield_now();
        }
        SwapMutexGuard::new(self)
    }
```

与标准库`Mutex`相比，API 略有不同。一方面，`SwapMutex`不跟踪中毒，因此，如果调用锁，则保证会返回一个名为`SwapMutexGuard`的守卫。然而，`SwapMutex`是基于作用域的；就像标准库`MutexGuard`一样，没有必要——也没有能力——永远调用显式的解锁。不过，确实有一个解锁操作，用于`SwapMutexGuard`的析构：

```rs
    fn unlock(&self) -> () {
        assert!(self.locked.load(Ordering::Relaxed) == true);
        self.locked.store(false, Ordering::Release);
    }
```

在这个简单的互斥锁中，要解锁只需要将`locked`字段设置为`false`，但在确认调用线程确实有权解锁互斥锁之前不要这样做。

现在，我们能否说服自己这个互斥锁是正确的？让我们逐一检查我们的标准：

“互斥锁支持两种操作，*锁定*和*解锁*。”

检查。接下来：

“互斥锁要么是*锁定*的，要么是*未锁定*。”

通过布尔值标记来检查，最后：

“操作*锁定*将互斥锁移动到*锁定*状态，当且仅当互斥锁处于*未锁定*状态。完成*锁定*操作的线程被称为持有*锁定*。”

让我们讨论一下 `AtomicBool::swap` 的作用。完整的类型是 `swap(&self, val: bool, order: Ordering) -> bool`，该函数根据其名称执行你可能会期望的操作；它根据传递的排序原子地交换 `self` 中的布尔值与 `val` 中的布尔值，并返回前一个值。在这里，因此，每个线程都在竞争将 true 写入锁定标志，并且由于交换的原子性，一次只有一个线程会看到返回的前一个值为 false。写入 false 的线程现在是锁的持有者，返回 `SwapMutexGuard`。锁定 `SwapMutex` 只能对未锁定的 `SwapMutex` 进行，并且是排他性地对其他线程进行的。

“解锁操作将把互斥锁移动到未锁定状态，如果且仅当互斥锁之前是锁定状态，并且解锁的调用者是锁的持有者。”

让我们考虑解锁。首先回忆一下，只有从 `SwapMutexGuard` 调用才有可能进行操作，所以调用者有权限解锁 `SwapMutex` 的断言是对锁故障的检查：一次只有一个线程可以持有 `SwapMutex`，因此内存中只会存在一个 `SwapMutexGuard`。由于 `Release` 排序的性质，我们保证 `locked` 的 `Relaxed` 加载将在存储之前发生，所以可以保证当存储发生时，`locked` 的值将是 true。只有锁的持有者才能解锁它，并且这个属性也得到了满足。

“在程序顺序中在锁之前和之后发生的所有加载和存储操作不得在锁之前或之后移动。”

这直接源于 `AcqRel` 排序的行为。

“在程序顺序中在解锁之前发生的所有加载和存储操作不得移动到解锁之后。”

这源于 `Release` 排序的行为。

嗯，我们得到了一个互斥锁。它不是一个特别节能的互斥锁，但它确实具有基本属性。为了完整性，以下是其余的内容：

```rs
impl<T> Drop for SwapMutex<T> {
    fn drop(&mut self) {
        let data = unsafe { Box::from_raw(self.data) };
        drop(data);
    }
}

pub struct SwapMutexGuard<'a, T: 'a> {
    __lock: &'a SwapMutex<T>,
}

impl<'a, T> SwapMutexGuard<'a, T> {
    fn new(lock: &'a SwapMutex<T>) -> SwapMutexGuard<'a, T> {
        SwapMutexGuard { __lock: lock }
    }
}

impl<'a, T> Deref for SwapMutexGuard<'a, T> {
    type Target = T;

    fn deref(&self) -> &T {
        unsafe { &*self.__lock.data }
    }
}

impl<'a, T> DerefMut for SwapMutexGuard<'a, T> {
    fn deref_mut(&mut self) -> &mut T {
        unsafe { &mut *self.__lock.data }
    }
}

impl<'a, T> Drop for SwapMutexGuard<'a, T> {
    #[inline]
    fn drop(&mut self) {
        self.__lock.unlock();
    }
}
```

现在，让我们用 `SwapMutex` 来构建一些东西。在 `src/bin/swap_mutex.rs` 中，我们将复制上一章中的桥接问题，但这次是在 `SwapMutex` 上，再加上一些现在我们知道如何使用原子操作的一些花哨的附加功能。下面是前言：

```rs
extern crate synchro;

use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;
use synchro::SwapMutex;
use std::{thread, time};

#[derive(Debug)]
enum Bridge {
    Empty,
    Left(u8),
    Right(u8),
}
```

足够熟悉了。我们引入了 `SwapMutex`，并定义了 `Bridge`。相当不出所料的内容。继续：

```rs
static LHS_TRANSFERS: AtomicUsize = AtomicUsize::new(0);
static RHS_TRANSFERS: AtomicUsize = AtomicUsize::new(0);
```

虽然这是新的。Rust 中的静态是编译时全局变量。我们偶尔看到它们的静态生命周期在周围飘荡，但从未对此发表评论。静态将成为二进制文件的一部分，并且为静态所需的任何初始化代码都必须在编译时评估，并且是线程安全的。我们在程序顶部创建了两个静态`AtomicUsizes`。为什么？前一个绳桥实现的主要问题之一是它的沉默。你当然可以在调试器中观察它，但这很慢，而且不会给你的朋友留下深刻印象。现在，我们有了原子操作，我们将要做的就是让桥的每一侧都统计有多少狒狒通过。`LHS_TRANSFERS`是向左移动的狒狒的数量，`RHS_TRANSFERS`是相反的方向。这次我们将桥的左右两侧分解成函数：

```rs
fn lhs(rope: Arc<SwapMutex<Bridge>>) -> () {
    loop {
        let mut guard = rope.lock();
        match *guard {
            Bridge::Empty => {
                *guard = Bridge::Right(1);
            }
            Bridge::Right(i) => {
                if i < 5 {
                    *guard = Bridge::Right(i + 1);
                }
            }
            Bridge::Left(0) => {
                *guard = Bridge::Empty;
            }
            Bridge::Left(i) => {
                LHS_TRANSFERS.fetch_add(1, Ordering::Relaxed);
                *guard = Bridge::Left(i - 1);
            }
        }
    }
}

fn rhs(rope: Arc<SwapMutex<Bridge>>) -> () {
    loop {
        let mut guard = rope.lock();
        match *guard {
            Bridge::Empty => {
                *guard = Bridge::Left(1);
            }
            Bridge::Left(i) => {
                if i < 5 {
                    *guard = Bridge::Left(i + 1);
                }
            }
            Bridge::Right(0) => {
                *guard = Bridge::Empty;
            }
            Bridge::Right(i) => {
                RHS_TRANSFERS.fetch_add(1, Ordering::Relaxed);
                *guard = Bridge::Right(i - 1);
            }
        }
    }
}
```

这应该足够熟悉了，来自上一章，但请注意，传输计数器现在正在更新。我们使用了`Relaxed`排序——增量保证会到达，但它们严格发生在修改守卫之前或之后并不是特别必要。最后，是`main`函数：

```rs
fn main() {
    let mtx: Arc<SwapMutex<Bridge>> = Arc::new(SwapMutex::new(Bridge::Empty));

    let lhs_mtx = Arc::clone(&mtx);
    let _lhs = thread::spawn(move || lhs(lhs_mtx));
    let _rhs = thread::spawn(move || rhs(mtx));

    let one_second = time::Duration::from_millis(1_000);
    loop {
        thread::sleep(one_second);
        println!(
            "Transfers per second:\n    LHS: {}\n    RHS: {}",
            LHS_TRANSFERS.swap(0, Ordering::Relaxed),
            RHS_TRANSFERS.swap(0, Ordering::Relaxed)
        );
    }
}
```

我们可以看到，这里已经设置了互斥锁，创建了线程，并且主线程花费时间在一个无限循环中打印传输率的信息。这是通过在一秒钟间隔中暂停线程，交换传输计数器与零并打印前一个值来完成的。输出看起来像这样：

```rs
> cargo build
> ./target/debug/swap_mutex
Transfers per second:
    LHS: 787790
    RHS: 719371
Transfers per second:
    LHS: 833537
    RHS: 770782
Transfers per second:
    LHS: 848662
    RHS: 776678
Transfers per second:
    LHS: 783769
    RHS: 726334
Transfers per second:
    LHS: 828969
    RHS: 761439
```

具体的数字将根据您的系统而变化。请注意，在某些情况下，这些数字也不是非常接近。`SwapMutex`不是**公平的**。

# 一个错误的原子队列

在我们构建任何其他东西之前，我们需要一个关键的数据结构——一个无界队列。在第五章中，*锁——互斥锁、条件变量、屏障和读写锁*，我们讨论了由互斥锁保护在两端的有界双端队列。现在，我们正在从事构建同步的工作，不能使用互斥锁的方法。我们的目标是创建一个无界先进先出数据结构，它没有锁，永远不会让入队或出队操作死锁，并且可以线性化为顺序队列。结果证明，有一个相当直接的数据结构可以实现这个目标；迈克尔和斯科特队列，在他们的 1995 年论文*简单、快速和实用的非阻塞和阻塞并发队列算法*中介绍。鼓励读者在继续我们这里的讨论之前快速浏览一下那篇论文，但这不是严格必要的。

有一个警告。我们的实现将是**错误的**。仔细的读者会注意到我们正在紧密遵循论文。我们将展示的实现有两个，可能还有更多，问题，一个是主要且无法解决的，另一个可以通过一些小心处理来解决。这两个问题都相当微妙。让我们深入探讨。

`Queue` 定义在 `src/queue.rs` 中，其前导部分如下：

```rs
use std::ptr::null_mut;
use std::sync::atomic::{AtomicPtr, Ordering};

unsafe impl<T: Send> Send for Queue<T> {}
unsafe impl<T: Send> Sync for Queue<T> {}
```

立刻从 `null_mut` 可以看出，我们将要处理原始指针。`AtomicPtr` 是新的，尽管我们在本章前面提到了它。原子指针将原始指针——`*mut T`——适配为适合原子操作。与 `AtomicPtr` 相关的运行时开销为零；Rust 文档指出，该类型与 `*mut T` 具有相同的内存表示。现代机器暴露出指令，使程序员能够原子地操作内存。正如预期的那样，能力因处理器而异。LLVM 和因此 Rust 通过 `AtomicPtr` 暴露这些原子内存操作能力，允许在顺序语言中原子地执行指针操作。在实践中，这意味着我们可以开始设置指针操作的前置/同步因果关系，这对于构建数据结构至关重要。

这里是下一部分：

```rs
struct Node<T> {
    value: *const T,
    next: AtomicPtr<Node<T>>,
}

impl<T> Default for Node<T> {
    fn default() -> Self {
        Node {
            value: null_mut(),
            next: AtomicPtr::default(),
        }
    }
}

impl<T> Node<T> {
    fn new(val: T) -> Self {
        Node {
            value: Box::into_raw(Box::new(val)),
            next: AtomicPtr::default(),
        }
    }
}
```

上一章中的双端队列的内部是一个单一的、连续的块。我们这里不是采取这种方法。相反，每个插入队列的元素都将获得一个 `Node`，该 `Node` 将指向下一个 `Node`，而这个 `Node` 可能存在也可能不存在。它是一个链表。在原子上下文中实现连续块的方法有点困难——尽管这是完全可能的，并且在 *进一步阅读* 部分的论文中有所讨论——并且将归结为连续块的链表。对于我们这里的用途来说，这比它值得的麻烦要多：

```rs
struct InnerQueue<T> {
    head: AtomicPtr<Node<T>>,
    tail: AtomicPtr<Node<T>>,
}
```

一个需要注意的关键点是 `Node` 持有一个指向堆分配的 `T` 的指针，而不是直接持有 `T`。在先前的代码中，我们有 `Queue<T>` 的 `InnerQueue<T>`，这是这本书和其他地方详细描述的常规内/外结构。为什么要注意 `Node` 不直接持有其 `T` 呢？`Queue<T>` 的头部的值永远不会被检查。`Queue` 的头部是一个哨兵。当创建 `InnerQueue` 时，我们将看到以下内容：

```rs
impl<T> InnerQueue<T> {
    pub fn new() -> Self {
        let node = Box::into_raw(Box::new(Node::default()));
        InnerQueue {
            head: AtomicPtr::new(node),
            tail: AtomicPtr::new(node),
        }
    }
```

如预期的那样，`InnerQueue` 的 `head` 和 `tail` 都指向同一个无意义的 `Node`。实际上，初始值是 null。原子数据结构在内存回收方面存在问题，因为协调释放是困难的，并且必须只执行一次。通过依赖 Rust 的类型系统，可以稍微缓解这个问题，但这仍然是一个非平凡的工程，并且是研究的一个活跃领域。在这里，我们注意到我们非常小心地只分配一次元素的拥有权。作为一个原始指针，它可以一次分配多次，但这条路径会导致双重释放。`InnerQueue` 将 `*const T` 转换为 `T`——这是一个不安全的操作——并且永远不会再次解引用 `*const T`，允许调用者在合适的时间执行释放操作：

```rs
    pub unsafe fn enq(&mut self, val: T) -> () {
        let node = Box::new(Node::new(val));
        let node: *mut Node<T> = Box::into_raw(node);

        loop {
            let tail: *mut Node<T> = self.tail.load(Ordering::Acquire);
            let next: *mut Node<T> = 
            (*tail).next.load(Ordering::Relaxed);
            if tail == self.tail.load(Ordering::Relaxed) {
                if next.is_null() {
                    if (*tail).next.compare_and_swap(next, node, 
                     Ordering::Relaxed) == next {
                        self.tail.compare_and_swap(tail, node, 
                         Ordering::Release);
                        return;
                    }
                }
            } else {
                self.tail.compare_and_swap(tail, next, 
                Ordering::Release);
            }
        }
    }
```

这是`enq`操作，由于涉及原始指针操作，因此标记为不安全。这是一个需要考虑的重要点——`AtomicPtr`必然需要使用原始指针。这里有很多事情在进行，所以让我们将其分解成更小的部分：

```rs
    pub unsafe fn enq(&mut self, val: T) -> () {
        let node = Box::new(Node::new(val));
        let node: *mut Node<T> = Box::into_raw(node);
```

这里，我们正在为`val`构造`Node`。注意我们正在使用与之前章节中经常使用的相同的装箱，即`into_raw`方法。这个节点在队列中还没有位置，调用线程也没有对队列持有独占锁。插入将不得不在其他插入过程中进行：

```rs
        loop {
            let tail: *mut Node<T> = self.tail.load(Ordering::Acquire);
            let next: *mut Node<T> = 
            (*tail).next.load(Ordering::Relaxed);
```

考虑到这一点，插入尝试可能会失败。在队列中入队一个元素发生在`尾`部分，我们加载并调用`last`的指针。`tail`之后的下一个节点称为`next`。在顺序队列中，我们可以保证`tail`的下一个是 null，但在这里并非如此。考虑在加载`tail`指针和加载`next`指针之间，另一个线程可能已经完成了`enq`操作。

入队操作可能需要尝试多次，才能达到成功的条件。这些条件是结构体的`尾`部分，接下来是`null`：

```rs
            if tail == self.tail.load(Ordering::Relaxed) {
                if next.is_null() {
```

注意，第一次加载`tail`是`Acquire`，而可能的存储操作，无论是哪个分支，都是`Release`。这满足了我们的`Acquire`/`Release`需求，关于锁定原语。这里的所有其他存储和加载操作都是明显的`Relaxed`。我们如何确保我们没有意外地覆盖写入，或者由于这是一个链表，在内存中切断它们？这就是`AtomicPtr`发挥作用的地方：

```rs
                    if (*tail).next.compare_and_swap(next, node, 
                     Ordering::Relaxed) == next {
                        self.tail.compare_and_swap(tail, node, 
                         Ordering::Release);
                        return;
                    }
```

完全有可能，当我们检测到入队正确条件的时候，另一个线程已经被调度，已经检测到入队正确条件，并且已经被入队。我们尝试使用`(*last).next.compare_and_swap(next, node, Ordering::Relaxed)`将新节点插入，也就是说，我们比较当前`last`的`next`，如果并且只有如果这成功了——那就是`== next`部分——我们尝试将`tail`设置为节点指针，再次使用比较和交换。如果这两个都成功了，那么新元素已经完全入队。然而，`tail`的交换可能会失败，在这种情况下，链表被正确设置，但`tail`指针偏移了。`enq`和`deq`操作都必须意识到它们可能会遇到需要调整`tail`指针的情况。实际上，这就是`enq`函数结束的方式：

```rs
                }
            } else {
                self.tail.compare_and_swap(tail, next, 
                 Ordering::Release);
            }
        }
    }
```

在 x86 上，所有这些 `Relaxed` 操作都更为严格，但在 ARMv8 上会有各种重排序。在所有修改之间建立因果关系非常重要且困难。例如，如果我们交换了 `tail` 指针然后是 `tail` 指针的 `next`，我们就会使自己面临破坏链表或根据线程对内存的视图创建整个孤立链的风险。`deq` 操作类似：

```rs
    pub unsafe fn deq(&mut self) -> Option<T> {
        let mut head: *mut Node<T>;
        let value: T;
        loop {
            head = self.head.load(Ordering::Acquire);
            let tail: *mut Node<T> = self.tail.load(Ordering::Relaxed);
            let next: *mut Node<T> = 
            (*head).next.load(Ordering::Relaxed);
            if head == self.head.load(Ordering::Relaxed) {
                if head == tail {
                    if next.is_null() {
                        return None;
                    }
                    self.tail.compare_and_swap(tail, next, 
                     Ordering::Relaxed);
                } else {
                    let val: *mut T = (*next).value as *mut T;
                    if self.head.compare_and_swap(head, next, 
                    Ordering::Release) == head {
                        value = *Box::from_raw(val);
                        break;
                    }
                }
            }
        }
        let head: Node<T> = *Box::from_raw(head);
        drop(head);
        Some(value)
    }
}
```

这个函数是一个循环，就像 `enq` 一样，我们在其中寻找正确的条件和情况来出队一个元素。第一个外部 if 子句检查 `head` 是否在我们身上移动，而内部第一个分支是关于一个没有元素的队列，其中第一个和最后一个都是指向相同存储的指针。注意，如果 `next` 不是空，我们会在再次循环回来进行另一次出队尝试之前尝试修复一个部分完成的节点链表。

这是因为，正如之前讨论的那样，`enq` 可能不会完全成功。当 `head` 和 `tail` 不相等时，会触发第二个内部循环，这意味着有一个元素需要被取出。正如内联注释所解释的，当第一个元素没有在我们身上移动时，我们就会给出元素 `T` 的所有权，但我们非常小心，直到我们可以确定我们将是唯一一个将管理该元素的线程，我们才不会解引用指针。我们可以因为只有一个线程能够交换调用线程当前持有的特定第一个和下一个对。

在所有这些之后，实际的 `Queue<T>` 外部函数有点令人失望：

```rs
pub struct Queue<T> {
    inner: *mut InnerQueue<T>,
}

impl<T> Clone for Queue<T> {
    fn clone(&self) -> Queue<T> {
        Queue { inner: self.inner }
    }
}

impl<T> Queue<T> {
    pub fn new() -> Self {
        Queue {
            inner: Box::into_raw(Box::new(InnerQueue::new())),
        }
    }

    pub fn enq(&self, val: T) -> () {
        unsafe { (*self.inner).enq(val) }
    }

    pub fn deq(&self) -> Option<T> {
        unsafe { (*self.inner).deq() }
    }
```

我们已经通过推理完成了实现，并且，希望亲爱的读者，你已经被说服这个想法应该能够工作。真正考验的地方在于测试：

```rs
#[cfg(test)]
mod test {
    extern crate quickcheck;

    use super::*;
    use std::collections::VecDeque;
    use std::sync::atomic::AtomicUsize;
    use std::thread;
    use std::sync::Arc;
    use self::quickcheck::{Arbitrary, Gen, QuickCheck, TestResult};

    #[derive(Clone, Debug)]
    enum Op {
        Enq(u32),
        Deq,
    }

    impl Arbitrary for Op {
        fn arbitrary<G>(g: &mut G) -> Self
        where
            G: Gen,
        {
            let i: usize = g.gen_range(0, 2);
            match i {
                0 => Op::Enq(g.gen()),
                _ => Op::Deq,
            }
        }
    }
```

这是我们在这本书的其他地方看到过的通常的 `test` 前置代码。我们定义一个 `Op` 枚举来驱动解释器风格的 `quickcheck test`，我们在这里称之为 `sequential`：

```rs
    #[test]
    fn sequential() {
        fn inner(ops: Vec<Op>) -> TestResult {
            let mut vd = VecDeque::new();
            let q = Queue::new();

            for op in ops {
                match op {
                    Op::Enq(v) => {
                        vd.push_back(v);
                        q.enq(v);
                    }
                    Op::Deq => {
                        assert_eq!(vd.pop_front(), q.deq());
                    }
                }
            }
            TestResult::passed()
        }
        QuickCheck::new().quickcheck(inner as fn(Vec<Op>) -> 
        TestResult);
    }
```

我们有一个 `VecDeque` 作为模型；我们知道它是一个合适的队列。然后，在不涉及任何真实并发的情况下，我们确认 `Queue` 的行为类似于 `VecDeque`。至少在顺序设置中，`Queue` 将会工作。现在，对于并行 `test`：

```rs
    fn parallel_exp(total: usize, enqs: u8, deqs: u8) -> bool {
        let q = Queue::new();
        let total_expected = total * (enqs as usize);
        let total_retrieved = Arc::new(AtomicUsize::new(0));

        let mut ejhs = Vec::new();
        for _ in 0..enqs {
            let mut q = q.clone();
            ejhs.push(
                thread::Builder::new()
                    .spawn(move || {
                        for i in 0..total {
                            q.enq(i);
                        }
                    })
                    .unwrap(),
            );
        }

        let mut djhs = Vec::new();
        for _ in 0..deqs {
            let mut q = q.clone();
            let total_retrieved = Arc::clone(&total_retrieved);
            djhs.push(
                thread::Builder::new()
                    .spawn(move || {
                        while total_retrieved.load(Ordering::Relaxed) 
                        != total_expected {
                            if q.deq().is_some() {
                                total_retrieved.fetch_add(1, 
                                Ordering::Relaxed);
                            }
                        }
                    })
                    .unwrap(),
            );
        }

        for jh in ejhs {
            jh.join().unwrap();
        }
        for jh in djhs {
            jh.join().unwrap();
        }

        assert_eq!(total_retrieved.load(Ordering::Relaxed), 
        total_expected);
        true
    }
```

我们设置了两组线程，一组负责入队，另一组负责出队。入队线程将`total`数量的项目推入`Queue`，而出队线程则拉取，直到一个共享在每个出队线程之间的计数器达到 bingo。最后，回到主`test`线程，我们确认检索到的总项目数与预期的项目数相同。我们的出队线程可能会因为 while 循环的检查和`q.deq`调用的竞争而读取队列的末尾，这反而对我们有利，因为确认队列允许出队不超过入队元素的数量。此外，当运行`test`时，没有出现双重释放崩溃。这个内部的`test`函数被使用了两次，一次是在重复运行中，然后又在`quickcheck`设置中：

```rs
    #[test]
    fn repeated() {
        for i in 0..10_000 {
            println!("{}", i);
            parallel_exp(73, 2, 2);
        }
    }

    #[test]
    fn parallel() {
        fn inner(total: usize, enqs: u8, deqs: u8) -> TestResult {
            if enqs == 0 || deqs == 0 {
                TestResult::discard()
            } else {
                TestResult::from_bool(parallel_exp(total, enqs, deqs))
            }
        }
        QuickCheck::new().quickcheck(inner as fn(usize, u8, u8) -> 
        TestResult);
    }
}
```

这里有什么问题？如果你运行`test`套件，你可能会或可能不会遇到一个问题。它们相当不可能，尽管我们很快就会看到一种可靠触发最坏情况的方法。我们实现遇到的第一问题是 ABA 问题。在比较和交换操作中，指针`A`将被某个线程与`B`交换。在第一个线程完成检查之前，另一个线程将`A`与`C`交换，然后`C`又回到`A`。然后第一个线程被重新调度，并执行其将`A`比较和交换为`B`的操作，而不知道`A`实际上并不是交换开始时的那个`A`。这将导致队列的链表块指向错误，可能指向队列不应拥有的内存。这已经足够糟糕了。还能更糟吗？

让我们用这个结构引起使用后释放违规。我们的演示程序很短，位于`src/bin/queue_spin.rs`：

```rs
extern crate synchro;

use synchro::Queue;
use std::thread;

fn main() {
    let q = Queue::new();

    let mut jhs = Vec::new();

    for _ in 0..4 {
        let eq = q.clone();
        jhs.push(thread::spawn(move || {
            let mut i = 0;
            loop {
                eq.enq(i);
                i += 1;
                eq.deq();
            }
        }))
    }

    for jh in jhs {
        jh.join().unwrap();
    }
}
```

程序创建了四个线程，每个线程尽可能快地按顺序入队和出队，它们之间没有任何协调。至少需要有两个线程，否则队列是顺序使用的，问题就不会存在：

```rs
> time cargo run --bin queue_spin
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/queue_spin`
Segmentation fault

real    0m0.588s
user    0m0.964s
sys     0m0.016s
```

ouch。这根本没花什么时间。让我们用调试器看看程序。我们将使用`lldb`，但如果你使用`gdb`，结果将是相同的：

```rs
> lldb-3.9 target/debug/queue_spin
(lldb) target create "target/debug/queue_spin"
Current executable set to 'target/debug/queue_spin' (x86_64).
(lldb) run
Process 12917 launched: '/home/blt/projects/us/troutwine/concurrency_in_rust/external_projects/synchro/target/debug/queue_spin' (x86_64)
Process 12917 stopped
* thread #2: tid = 12920, 0x0000555555560585 queue_spin`_$LT$synchro..queue..InnerQueue$LT$T$GT$$GT$::deq::heefaa8c9b1d410ee(self=0x00007ffff6c2a010) + 261 at queue.rs:78, name = 'queue_spin', stop reason = signal SIGSEGV: invalid address (fault address: 0x0)
    frame #0: 0x0000555555560585 queue_spin`_$LT$synchro..queue..InnerQueue$LT$T$GT$$GT$::deq::heefaa8c9b1d410ee(self=0x00007ffff6c2a010) + 261 at queue.rs:78
   75                       }
   76                       self.tail.compare_and_swap(tail, next, 
                            Ordering::Relaxed);
   77                   } else {
-> 78                       let val: *mut T = (*next).value as *mut T;
   79                       if self.head.compare_and_swap(head, next, 
                            Ordering::Release) == head {
   80                           value = *Box::from_raw(val);
   81                           break;
  thread #3: tid = 12921, 0x0000555555560585 queue_spin`_$LT$synchro..queue..InnerQueue$LT$T$GT$$GT$::deq::heefaa8c9b1d410ee(self=0x00007ffff6c2a010) + 261 at queue.rs:78, name = 'queue_spin', stop reason = signal SIGSEGV: invalid address (fault address: 0x0)
    frame #0: 0x0000555555560585 queue_spin`_$LT$synchro..queue..InnerQueue$LT$T$GT$$GT$::deq::heefaa8c9b1d410ee(self=0x00007ffff6c2a010) + 261 at queue.rs:78
   75                       }
   76                       self.tail.compare_and_swap(tail, next, 
                            Ordering::Relaxed);
   77                   } else {
-> 78                       let val: *mut T = (*next).value as *mut T;
   79                       if self.head.compare_and_swap(head, next, 
                            Ordering::Release) == head {
   80                           value = *Box::from_raw(val);
   81                           break;
```

好吧，看看这个！只是为了确认：

```rs
(lldb) p next
(synchro::queue::Node<i32> *) $0 = 0x0000000000000000
```

我们能再增加一个吗？是的：

```rs
(lldb) Process 12893 launched: '/home/blt/projects/us/troutwine/concurrency_in_rust/external_projects/synchro/target/debug/queue_spin' (x86_64)
Process 12893 stopped
* thread #2: tid = 12896, 0x000055555555fb3e queue_spin`core::sync::atomic::atomic_load::hd37078e3d501f11f(dst=0x0000000000000008, order=Relaxed) + 78 at atomic.rs:1502, name = 'queue_spin', stop reason = signal SIGSEGV: invalid address (fault address: 0x8)
    frame #0: 0x000055555555fb3e queue_spin`core::sync::atomic::atomic_load::hd37078e3d501f11f(dst=0x0000000000000008, order=Relaxed) + 78 at atomic.rs:1502
  thread #3: tid = 12897, 0x000055555555fb3e queue_spin`core::sync::atomic::atomic_load::hd37078e3d501f11f(dst=0x0000000000000008, order=Relaxed) + 78 at atomic.rs:1502, name = 'queue_spin', stop reason = signal SIGSEGV: invalid address (fault address: 0x8)
    frame #0: 0x000055555555fb3e queue_spin`core::sync::atomic::atomic_load::hd37078e3d501f11f(dst=0x0000000000000008, order=Relaxed) + 78 at atomic.rs:1502
  thread #4: tid = 12898, 0x000055555555fb3e queue_spin`core::sync::atomic::atomic_load::hd37078e3d501f11f(dst=0x0000000000000008, order=Relaxed) + 78 at atomic.rs:1502, name = 'queue_spin', stop reason = signal SIGSEGV: invalid address (fault address: 0x8)
    frame #0: 0x000055555555fb3e queue_spin`core::sync::atomic::atomic_load::hd37078e3d501f11f(dst=0x0000000000000008, order=Relaxed) + 78 at atomic.rs:1502
  thread #5: tid = 12899, 0x000055555555fb3e queue_spin`core::sync::atomic::atomic_load::hd37078e3d501f11f(dst=0x0000000000000008, order=Relaxed) + 78 at atomic.rs:1502, name = 'queue_spin', stop reason = signal SIGSEGV: invalid address (fault address: 0x8)
    frame #0: 0x000055555555fb3e
queue_spin`core::sync::atomic::atomic_load::hd37078e3d501f11f(dst=0x0000000000000008, order=Relaxed) + 78 at atomic.rs:1502
```

在第一个例子中，头指针指向 null，这将在队列为空时发生，但这个特定的分支只有在队列为空时才会被击中。这里发生了什么？实际上，这里有一个讨厌的竞争，它与释放有关。让我们再次看看`deq`，这次带有行号：

```rs
 64    pub unsafe fn deq(&mut self) -> Option<T> {
 65        let mut head: *mut Node<T>;
 66        let value: T;
 67        loop {
 68            head = self.head.load(Ordering::Acquire);
 69            let tail: *mut Node<T> = 
               self.tail.load(Ordering::Relaxed);
 70            let next: *mut Node<T> = 
               (*head).next.load(Ordering::Relaxed);
 71            if head == self.head.load(Ordering::Relaxed) {
 72                if head == tail {
 73                    if next.is_null() {
 74                        return None;
 75                    }
 76                    self.tail.compare_and_swap(tail, next, 
                       Ordering::Relaxed);
 77                } else {
 78                    let val: *mut T = (*next).value as *mut T;
 79                    if self.head.compare_and_swap(head, next,  
                       Ordering::Release) == head {
 80                        value = *Box::from_raw(val);
 81                        break;
 82                    }
 83                }
 84            }
 85        }
 86        let head: Node<T> = *Box::from_raw(head);
 87        drop(head);
 88        Some(value)
 89    }
```

假设有四个线程，`A` 到 `D`。线程 `A` 到达第 78 行并停止。此时，线程 `A` 拥有一个 `head`、一个 `tail` 和一个 `next`，它们指向内存中的一个合理的链表。现在，线程 `B`、`C` 和 `D` 各自行使多次 `enq` 和 `deq` 操作，使得当 `A` 唤醒时，`A` 的头指针指向的链表已经不复存在。实际上，头节点本身已经被释放，但 `A` 很幸运，操作系统还没有覆盖它的内存。

线程 `A` 唤醒并执行第 78 行，但接下来却指向了无意义的内容，整个程序崩溃。或者，假设我们有两个线程，`A` 和 `B`。线程 `A` 唤醒并通过到第 70 行并停止。线程 `B` 唤醒并且非常幸运，一直前进到第 88 行，释放头节点。操作系统感觉到了自己的力量，覆盖了 `A` 指向的内存。然后线程 `A` 唤醒，触发 `(*head).next.load(Ordering::Relaxed)`，随后崩溃。这里有很多不同的可能性。它们之间的共同点是，在仍有未解决的对一个或多个节点的引用时发生释放。实际上，Michael 和 Scott 的论文确实提到了这个问题，但只是简要地提了一下，很容易被忽略：

“为了获得各种指针的一致值，我们依赖于一系列的读取操作，这些操作重新检查早期值以确保它们没有改变。这些读取序列与 Prakash 等人（我们需要检查一个共享变量而不是两个）的快照类似，但更简单。可以使用类似的技术来防止 Stone 的阻塞算法中的竞争条件。我们使用 Treiber 的简单高效的非阻塞栈算法来实现非阻塞的空闲列表。”

这里的关键思想是*重新检查早期值的读取序列*和*空闲列表*。通过检查我们发现，比较并交换一个已更改的值——ABA 问题——会导致我们的实现指向空间。此外，立即释放节点将使我们即使在没有比较并交换的情况下也容易崩溃。Michael 和 Scott 做的是创建一种最小化的内存管理；而不是删除节点，他们把它们移动到*空闲列表*中，以便稍后重用或删除。空闲列表可以是线程本地的，在这种情况下，你可以避免昂贵的同步，但仍然有点难以确保你的线程本地空闲列表不与另一个线程的指针相同。

# 纠正错误队列的选项

我们应该怎么做？实际上，文献中有很多选项。Hart、McKenney、Brown 和 Walpole 在 2007 年的论文《无锁同步的内存回收性能》中讨论了四种，并附上了它们原始论文的引用：

+   **基于静默状态的回收**（**QSRB**）：应用允许的信号静默'时间段进行回收

+   **基于时代的回收**（**EBR**）：应用程序通过在时代中移动内存来使用结构，以确定何时允许回收

+   **基于危险指针的回收**（**HPBR**）：线程协同并标记指针为回收时的危险状态

+   **无锁引用计数（LFRC**）：指针携带一个原子计数器，用于跟踪其使用情况，线程使用这个计数器来确定安全的回收周期

静止状态回收通过让在共享并发结构上操作的线程在一段时间内发出已进入*静止状态*的信号来实现，这意味着对于已标记的状态，线程不会对共享结构内的引用提出任何要求。一个单独的回收操作者，可能是结构本身或处于静止状态的线程，寻找*宽限期*，在这个期限内回收存储是安全的，要么因为所有线程都已静止，要么因为一些相关子集已静止。这相当棘手，需要在代码中添加进入和退出静止状态的通告。Hart 等人指出，在撰写本文时，这种方法在 Linux 内核中很常见，在撰写*本文*时仍然如此。感兴趣的读者可以参考有关 Linux 内核读-拷贝-更新机制的现有文献。QSRB 作为一项技术的主要困难在于，在用户空间中并不总是清楚是否存在自然静止期；如果你错误地发出信号，你可能会遇到数据竞争。操作系统的上下文切换是一个自然静止期——相关的上下文在它们被调用回来之前不会操作任何东西——但在我看来，在我们的队列中，这样一个时期的存在并不那么清晰。QSRB 的好处是它是一种非常*快速*的技术，比以下任何其他方法都需要更少的线程同步。

基于时代的回收与 QSRB 类似，但它是与应用程序无关的。也就是说，应用程序通过一个中介层与存储交互，而这个层存储了足够的信息来决定何时到来宽限期。特别是，每个线程都会切换一个标志，表示它打算与共享数据交互，交互后，在离开时将标志切换到另一个方向。标志周期数被称为一个“时代”。一旦共享数据通过了足够的时代，线程将尝试回收存储。这种方法的好处是比 QSRB 更自动——应用程序开发者使用专门设计的中间类型——但缺点是在数据访问的两侧都需要一些同步——那些标志切换。（论文实际上介绍了一种第五种新的基于时代的回收方法，它改进了基于时代的复制，但需要应用程序开发者的提示。在某种程度上，它是一种 EBR/QSRB 混合方法。）

危险指针回收方案完全依赖于数据结构。其理念是每个涉及的线程都将保留一个危险指针的集合，每个指针对应于它打算执行的无锁操作。这个线程本地的危险指针集合被映射到某个回收系统中——可能是一个专用垃圾收集器或插件分配器——并且这个系统能够检查危险指针的状态。当一个线程打算对某些共享数据进行无锁操作时，它会将危险指针标记为受保护，不允许其回收。当线程完成操作后，它会标记共享数据不再受保护。根据上下文，共享数据现在可能可以回收，或者可能需要后续回收。就像 EBR 一样，HPBR 需要通过特殊结构访问共享数据，并且在同步方面有开销。此外，与 EBR 相比，实现可能需要更高的负担，但低于 QSRB。更难的是，如果你的数据结构传递对其内部存储的引用，这也需要危险指针。线程越多，你需要更多的危险指针列表。内存开销可能会迅速变得相当大。与宽限期方法相比，由于无需搜索引用不是危险的时间，危险指针在回收内存方面可以更快。HPBR 每个操作的开销更高，总体内存需求也更高，但在回收方面可能更有效。这是一个非同小可的细节。如果 QSBR 或 EBR 在分配操作阻塞 CPU 的系统上运行，这些方法可能*永远*找不到宽限期，最终耗尽系统内存。

最后一个是原子引用计数，类似于标准库中的`Arc`，但用于原子操作。这种方法并不特别快，每次访问引用计数器以及该计数器的分支都会产生开销，但它很简单，回收发生得很快。

除了引用计数之外，这些方法中的每一个都很难实现。但我们很幸运；crate 生态系统提供了一个基于纪元的回收实现和危险指针回收。相关的库是 crossbeam ([`crates.io/crates/crossbeam`](https://crates.io/crates/crossbeam)) 和 conc ([`crates.io/crates/conc`](https://crates.io/crates/conc))。我们将在下一章中详细讨论它们。Crossbeam 旨在成为 Rust 的`libcds` ([`github.com/khizmax/libcds`](https://github.com/khizmax/libcds))，并且已经为我们实现了一个 Michael 和 Scott 队列。让我们试一试。相关文件是`src/bin/crossbeam_queue_spin.rs`：

```rs
extern crate crossbeam;

use crossbeam::sync::MsQueue;
use std::sync::Arc;
use std::thread;

fn main() {
    let q = Arc::new(MsQueue::new());

    let mut jhs = Vec::new();

    for _ in 0..4 {
        let q = Arc::clone(&q);
        jhs.push(thread::spawn(move || {
            let mut i = 0;
            loop {
                q.push(i);
                i += 1;
                q.pop();
            }
        }))
    }

    for jh in jhs {
        jh.join().unwrap();
    }
}
```

这与`queue_spin`相当相似，只是`enq`现在是推送操作，而`deq`是`pop`操作。同样值得注意的是，`MsQueue::pop`是阻塞的。这种机制的实现非常巧妙；我们将在下一章中讨论这一点。

# 信号量

我们在第五章中简要讨论了信号量，*锁 – Mutex, Condvar, Barriers, 和 RWLock*，特别是关于来自*小信号量*的并发难题。现在，正如承诺的那样，是时候实现一个信号量了。信号量究竟是什么？类似于我们将互斥锁分析为*原子对象*，让我们考虑它：

+   信号量支持两种操作，`wait`和`signal`。

+   信号量有一个`isize`类型的`value`，用于跟踪信号量的可用资源容量。这个值只能由`wait`和`signal`操作修改。

+   操作`wait`会减少`value`的值。如果值小于零，则`wait`会阻塞调用线程，直到有值可用。如果值不小于零，则线程不会阻塞。

+   操作`signal`会增加`value`的值。在增加之后，如果值小于或等于零，则有一个或多个等待线程。其中一个线程会被唤醒。如果增加后的值大于零，则没有等待线程。

+   在程序顺序中发生在等待之前和之后的所有加载和存储操作不得移动到等待之前或之后。

+   在程序顺序中发生在信号之前的所有加载和存储操作不得移动到信号之后。

如果这看起来很熟悉，那是有很好的原因的。从某种角度看，互斥锁是一个只有一个可用资源的信号量；锁定互斥锁映射到`wait`，而信号映射到`unwait`。我们已指定了在信号量等待和信号过程中的加载和存储行为，以避免在互斥锁分析中先前确定的相同死锁行为。

前面的分解没有捕捉到的一些细微之处不会影响信号量的分析，但会影响编程模型。我们将构建一个具有固定资源的信号量。也就是说，当创建信号量时，程序员负责设置信号量中可用的最大总资源，并确保信号量以这些资源可用开始。某些信号量实现允许资源容量随时间变化。这些通常被称为*计数信号量*。我们的变体被称为*有界信号量*；这种类型中只有一个资源的子变体称为*二进制信号量*。等待线程的信号行为可能有所不同。我们将根据先来先服务的原则唤醒线程。

让我们深入探讨。我们的信号量实现位于`src/semaphore.rs`中，它非常简短：

```rs
use crossbeam::sync::MsQueue;

unsafe impl Send for Semaphore {}
unsafe impl Sync for Semaphore {}

pub struct Semaphore {
    capacity: MsQueue<()>,
}

impl Semaphore {
    pub fn new(capacity: usize) -> Self {
        let q = MsQueue::new();
        for _ in 0..capacity {
            q.push(());
        }
        Semaphore { capacity: q }
    }

    pub fn wait(&self) -> () {
        self.capacity.pop();
    }

    pub fn signal(&self) -> () {
        self.capacity.push(());
    }
}
```

哇！Crossbeam 的`MsQueue`在`MsQueue::push`和`MsQueue::pop`按该顺序由同一线程完成时具有正确的排序语义，其中`pop`有额外的优势，即阻塞直到队列变为空。因此，我们的信号量是一个容量总计为`()`的`MsQueue`。`wait`操作通过`Acquire`/`Release`排序减少`value`（队列中的`()`总数）。`signal`操作通过将额外的`()`推送到队列中并带有`Release`语义来增加信号量的`value`。编程错误可能导致`wait`/`signal`调用不是一对一的，我们可以通过 Mutex 和`SwapMutex`采取的相同`Guard`方法来解决此问题。底层队列线性化`Guard`——请参阅下一章以了解该讨论——因此我们的信号量也是如此。让我们试试这个。我们有一个项目内的程序来演示`Semaphore`的使用，`src/bin/status_demo.rs`：

```rs
extern crate synchro;

use std::sync::Arc;
use synchro::Semaphore;
use std::{thread, time};

const THRS: usize = 4;
static mut COUNTS: &'static mut [u64] = &mut [0; THRS];
static mut STATUS: &'static mut [bool] = &mut [false; THRS];

fn worker(id: usize, gate: Arc<Semaphore>) -> () {
    unsafe {
        loop {
            gate.wait();
            STATUS[id] = true;
            COUNTS[id] += 1;
            STATUS[id] = false;
            gate.signal();
        }
    }
}

fn main() {
    let semaphore = Arc::new(Semaphore::new(1));

    for i in 0..THRS {
        let semaphore = Arc::clone(&semaphore);
        thread::spawn(move || worker(i, semaphore));
    }

    let mut counts: [u64; THRS] = [0; THRS];
    loop {
        unsafe {
            thread::sleep(time::Duration::from_millis(1_000));
            print!("|");
            for i in 0..THRS {
                print!(" {:>5}; {:010}/sec |", STATUS[i], 
                       COUNTS[i] - counts[i]);
                counts[i] = COUNTS[i];
            }
            println!();
        }
    }
}
```

我们设置`THRS`为总工作线程数，其职责是等待信号量，将它们的`STATUS`设置为 true，将`COUNT`加一，然后在发出信号量之前将`STATUS`设置为 false。可变静态数组对于任何程序来说都有些愚蠢的设置，但这是一个巧妙的技巧，并且在这里不会造成伤害，除了与优化器交互异常。如果你以发布模式编译此程序，你可能会发现优化器确定工作线程是一个无操作。`main`函数创建了一个容量为二的信号量，仔细地偏移了工作线程，然后无限期地打印出`STATUS`和`COUNT`的内容。在我的 x86 测试文章上的运行结果如下：

```rs
> ./target/release/status_demo
| false; 0000170580/sec |  true; 0000170889/sec |  true; 0000169847/sec | false; 0000169220/sec |
| false; 0000170262/sec | false; 0000170560/sec |  true; 0000169077/sec |  true; 0000169699/sec |
| false; 0000169109/sec | false; 0000169333/sec | false; 0000168422/sec | false; 0000168790/sec |
| false; 0000170266/sec |  true; 0000170653/sec | false; 0000168184/sec |  true; 0000169570/sec |
| false; 0000170907/sec | false; 0000171324/sec |  true; 0000170137/sec | false; 0000169227/sec |
...
```

以及从我的 ARMv7 机器上：

```rs
> ./target/release/status_demo
| false; 0000068840/sec |  true; 0000063798/sec | false; 0000063918/sec | false; 0000063652/sec |
| false; 0000074723/sec | false; 0000074253/sec |  true; 0000074392/sec |  true; 0000074485/sec |
|  true; 0000075138/sec | false; 0000074842/sec | false; 0000074791/sec |  true; 0000074698/sec |
| false; 0000075099/sec | false; 0000074117/sec | false; 0000074648/sec | false; 0000075083/sec |
| false; 0000075070/sec |  true; 0000074509/sec | false; 0000076196/sec |  true; 0000074577/sec |
|  true; 0000075257/sec |  true; 0000075682/sec | false; 0000074870/sec | false; 0000075887/sec |
...
```

# 二进制信号量，或者，一个更节省的互斥锁

我们的信号量与标准库的 Mutex 相比如何？我们的自旋锁`Mutex`呢？将此`diff`应用到`status_demo`：

```rs
diff --git a/external_projects/synchro/src/bin/status_demo.rs b/external_projects/synchro/src/bin/status_demo.rs
index cb3e850..fef2955 100644
--- a/external_projects/synchro/src/bin/status_demo.rs
+++ b/external_projects/synchro/src/bin/status_demo.rs
@@ -21,7 +21,7 @@ fn worker(id: usize, gate: Arc<Semaphore>) -> () {
 }

 fn main() {
-    let semaphore = Arc::new(Semaphore::new(2));
+    let semaphore = Arc::new(Semaphore::new(1));

     for i in 0..THRS {
         let semaphore = Arc::clone(&semaphore);
```

你将创建一个信号量状态演示的互斥锁变体。在我的 x86 测试文章上的数字如下：

```rs
| false; 0000090992/sec |  true; 0000090993/sec |  true; 0000091000/sec | false; 0000090993/sec |
| false; 0000090469/sec | false; 0000090468/sec |  true; 0000090467/sec |  true; 0000090469/sec |
|  true; 0000090093/sec | false; 0000090093/sec | false; 0000090095/sec | false; 0000090093/sec |
```

在容量为二的情况下，每秒是 170,000 次，所以有一个相当大的下降。让我们先与标准库互斥锁进行比较。这个适配在`src/bin/mutex_status_demo.rs`中：

```rs
use std::sync::{Arc, Mutex};
use std::{thread, time};

const THRS: usize = 4;
static mut COUNTS: &'static mut [u64] = &mut [0; THRS];
static mut STATUS: &'static mut [bool] = &mut [false; THRS];

fn worker(id: usize, gate: Arc<Mutex<()>>) -> () {
    unsafe {
        loop {
            let guard = gate.lock().unwrap();
            STATUS[id] = true;
            COUNTS[id] += 1;
            STATUS[id] = false;
            drop(guard);
        }
    }
}

fn main() {
    let mutex = Arc::new(Mutex::new(()));

    for i in 0..THRS {
        let mutex = Arc::clone(&mutex);
        thread::spawn(move || worker(i, mutex));
    }

    let mut counts: [u64; THRS] = [0; THRS];
    loop {
        unsafe {
            thread::sleep(time::Duration::from_millis(1_000));
            print!("|");
            for i in 0..THRS {
                print!(" {:>5}; {:010}/sec |", STATUS[i], 
                       COUNTS[i] - counts[i]);
                counts[i] = COUNTS[i];
            }
            println!();
        }
    }
}
```

很直接，对吧？在相同的 x86 测试文章上的数字如下：

```rs
| false; 0001856267/sec | false; 0002109238/sec |  true; 0002036852/sec |  true; 0002172337/sec |
| false; 0001887803/sec |  true; 0002072647/sec | false; 0002065467/sec |  true; 0002143558/sec |
| false; 0001848387/sec | false; 0002044828/sec |  true; 0002098595/sec |  true; 0002178304/sec |
```

让我们稍微慷慨一点，说标准库互斥锁每秒达到 2,000,000 次，而我们的二进制信号量每秒达到 170,000 次。那么自旋锁呢？在这个时候，我将避免列出源代码。只需确保导入`SwapMutex`并将其从标准库的`Mutex`相应地调整即可。这些数字如下：

```rs
|  true; 0012527450/sec | false; 0011959925/sec |  true; 0011863078/sec | false; 0012509126/sec |
|  true; 0012573119/sec | false; 0011949160/sec |  true; 0011894659/sec | false; 0012495174/sec |
| false; 0012481696/sec |  true; 0011952472/sec |  true; 0011856956/sec | false; 0012595455/sec |
```

你知道，这很有道理。自旋锁所做的最少工作量，每个线程都在燃烧 CPU 时间，以便在需要进入它们的临界区时立即到达。以下是我们的结果总结：

+   自旋锁互斥锁：每秒 11,000,000 次

+   标准库互斥锁：每秒 2,000,000 次

+   二进制信号量：每秒 170,000 次

鼓励读者在 Linux perf 中调查这些实现中的每一个。从这些结果中理解的关键不是这些实现中有哪一个比另一个更好或更差。而是它们适用于不同的目的。

例如，我们可以使用下一章中的技术来减少自旋锁互斥锁的 CPU 消耗。这可能会使其速度减慢一些，可能使其接近标准库 Mutex 的范围。在这种情况下，使用标准库 Mutex。

原子编程不是一件可以轻视的事情。正确实现它很困难，一个看似正确的实现可能并不遵守内存模型的因果属性。此外，仅仅因为一个事物是无锁的，并不意味着它在执行时会更*快*。

编写软件是关于识别和适应权衡。原子编程在这方面表现得尤为明显。

# 摘要

在本章中，我们讨论了现代 CPU 架构上可用的原子原语及其在 Rust 中的反映。我们构建了类似于第四章中讨论的*同步和发送 – Rust 并发的基石*、第五章中讨论的*锁 – Mutex、Condvar、屏障和 RWLock*以及我们自己的信号量（在 Rust 中不存在）。根据您的需求，信号量的实现可以改进，我鼓励读者尝试一下。我们还遇到了原子编程的常见问题——内存回收，我们用迈克尔·斯科特队列来讨论这个问题。我们将在下一章深入讨论解决这个问题的方法。

# 进一步阅读

+   *并发对象的公理*，莫里斯·赫尔利希和珍妮特·温。赫尔利希和温的这篇论文介绍了线性化的形式定义和形式分析框架。后续工作在此基础上进行了扩展或简化，但鼓励读者至少消化这篇论文的前半部分。

+   *分布式算法*，南希·林奇。据我所知，林奇的这项工作是首次以教科书形式对分布式算法进行的一般概述。它非常易于理解，林奇的书中的第十三章特别与这次讨论相关。

+   *在 C++中，acquire-release 内存顺序语义是传递的吗？*，可在[`softwareengineering.stackexchange.com/questions/253537/in-c-are-acquire-release-memory-order-semantics-transitive`](https://softwareengineering.stackexchange.com/questions/253537/in-c-are-acquire-release-memory-order-semantics-transitive)找到。内存顺序的影响并不总是清晰的。这个问题在`StackExchange`上，它影响了本章的写作，是关于在多线程中分割`Acquire`和`Release`的后果。鼓励读者在阅读答案之前思考这个问题。

+   *std::memory_order, CppReference*，可在[`en.cppreference.com/w/cpp/atomic/memory_order`](http://en.cppreference.com/w/cpp/atomic/memory_order)找到。Rust 的内存模型是 LLVM 的，而 LLVM 的内存模型又受到 C++内存模型的影响。这篇关于内存排序的讨论特别出色，因为它有示例代码，并且采用了更少语言律师风格的方法来解释排序。

+   *使用 C++0x 原子操作实现 Peterson 锁*，可在[`www.justsoftwaresolutions.co.uk/threading/petersons_lock_with_C++0x_atomics.html`](https://www.justsoftwaresolutions.co.uk/threading/petersons_lock_with_C++0x_atomics.html)找到。这篇文章讨论了 Bartosz Milewski 对 Peterson 互斥锁算法的实现，演示了为什么它是错误的，并描述了一个功能性的替代方案。Milewski 是一位知名的 C++专家。这正好说明了原子编程既困难又容易犯细微的错误。

+   *互斥锁算法*，迈克尔·雷纳尔。雷纳尔的这本书是在 1986 年写的，远在我们讨论的 x86_64 架构之前，并且是在 ARMv1 发布后的第二年。雷纳尔的书仍然很有用，既作为互斥锁算法的历史概述，也适用于同步原语不可用的环境，例如在文件系统中。

+   *多种互斥锁实现综述*，可在[`cbloomrants.blogspot.com/2011/07/07-15-11-review-of-many-mutex.html`](http://cbloomrants.blogspot.com/2011/07/07-15-11-review-of-many-mutex.html)找到。正如标题所示，这篇文章是对许多互斥锁实现的综述，这些实现比 Raynal 的书提供了更现代的背景。一些解释依赖于微软 Windows 的功能，对于这本书主要投资于类 Unix 环境的读者来说可能很受欢迎。

+   *降低 Cernan 度量成本*，可在[`blog.troutwine.us/2017/08/31/encheapening-cernan-internal-metrics/`](http://blog.troutwine.us/2017/08/31/encheapening-cernan-internal-metrics/)找到。在这一章中，我们主要从同步的角度讨论了原子操作。还有许多其他用例。这篇文章讨论了将原子操作应用于为复杂的软件项目提供低成本的自监控。

+   *自己动手实现轻量级互斥锁*，可在[`preshing.com/20120226/roll-your-own-lightweight-mutex/`](http://preshing.com/20120226/roll-your-own-lightweight-mutex/)找到。Preshing On Programming 网站有一系列优秀的原子操作材料，专注于 C++和微软环境。这篇特定的文章是关于实现互斥锁的，这个主题与本章内容紧密相关，并在评论中有很好的后续讨论。文章讨论了信号量的一种变体，称为 benaphores。

+   *简单、快速、实用的非阻塞和阻塞并发队列算法*，作者：Maged Michael 和 Michael Scott。本文介绍了本章讨论的 Michael 和 Scott 队列，这也是它最著名的部分。你也许会从上一章中认出他们的带有两个锁的队列，当然是通过 Erlang 虚拟机进行适配的。

+   *无锁数据结构：栈的演变*，可在[`kukuruku.co/post/lock-free-data-structures-the-evolution-of-a-stack/`](https://kukuruku.co/post/lock-free-data-structures-the-evolution-of-a-stack/)找到。这篇帖子讨论了 `libcds` 对 Trieber 栈的实现，并参考了其他文献领域。读者会注意到之前的 *进一步阅读* 中介绍了一些相同的文献；寻求不同的观点总是好的。特别鼓励读者调查帖子中引用的 Hendler、Shavit 和 Yerushalmi 的论文。
