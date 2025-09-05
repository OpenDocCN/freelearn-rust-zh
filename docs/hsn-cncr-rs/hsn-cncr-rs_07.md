# 第七章：原子操作 – 安全地回收内存

在上一章中，我们讨论了 Rust 程序员可用的原子原语，实现了高级同步原语和一些完全由原子组成的数据结构。与使用高级同步原语相比，原子编程的一个关键挑战是内存回收。内存只能安全地释放一次。当我们仅从原子原语构建并发算法时，要只做一次并保持性能是非常具有挑战性的。也就是说，安全地回收内存需要某种形式的同步。但是，随着并发角色的总数增加，同步的成本会超过原子编程的延迟或吞吐量收益。

在本章中，我们将讨论三种解决原子编程内存回收问题的技术——引用计数、危害指针和基于时代的回收。这些方法你从前几章中应该已经熟悉，但在这里我们将深入探讨它们，研究它们的权衡。我们将介绍两个新的库，`conc`和 crossbeam，分别实现了危害指针和基于时代的回收。到本章结束时，你应该对这三种方法有很好的掌握，并准备好开始编写生产级别的代码。

到本章结束时，我们将：

+   讨论引用计数及其相关的权衡

+   讨论危害指针的内存回收方法

+   使用`conc`库通过危害指针方法研究了 Treiber 栈

+   讨论基于时代的回收策略

+   对`crossbeam_epoch`库进行了深入研究

+   使用基于时代的回收方法研究了 Treiber 栈

# 技术要求

本章需要安装一个可工作的 Rust 环境。验证安装的详细信息请参阅第一章，*预备知识 – 计算机架构和 Rust 入门*。不需要额外的软件工具。

你可以在 GitHub 上找到本书项目的源代码：[`github.com/PacktPublishing/Hands-On-Concurrency-with-Rust`](https://github.com/PacktPublishing/Hands-On-Concurrency-with-Rust)。本章的源代码位于`Chapter07`。

# 内存回收方法

在接下来的讨论中，三种技术将按其速度大致排序，从慢到快，以及它们的难度，从易到难。你不应该气馁——你现在正处于软件工程世界的前沿。欢迎。大胆前行。

# 引用计数

引用计数内存回收将一块受保护的数据与一个原子计数器关联。每个读取或写入该受保护数据的线程在获取时增加计数器，在释放时减少计数器。减少计数器并发现它为零的线程是最后一个持有数据的线程，它可以标记数据为可回收，或者立即释放它。这应该听起来很熟悉。Rust 标准库附带`std::sync::Arc`——在第四章 4，*同步和发送——Rust 并发的基石*，以及第五章 5，*锁——Mutex, Condvar, Barriers 和 RWLock*中特别提到，以满足这一确切需求。虽然我们讨论了`Arc`的使用，但我们之前没有讨论其内部结构，因为我们需要先达到第六章 6，*原子操作——同步的原始操作*，以获得必要的背景知识。现在让我们深入探讨`Arc`。

Rust 编译器的`Arc`代码在`src/liballoc/arc.rs`中。结构定义足够紧凑：

```rs
pub struct Arc<T: ?Sized> {
    ptr: Shared<ArcInner<T>>,
    phantom: PhantomData<T>,
}
```

我们在第三章 3，*Rust 内存模型——所有权、引用和操作*中遇到了`Shared<T>`，它是一个具有与`*mut X`相同特性的指针，除了它不是空指针。创建新的`Shared<T>`有两个函数，分别是`const unsafe fn new_unchecked(ptr: *mut X) -> Self`和`fn new(ptr: *mut X) -> Option<Self>`。在第一个变体中，调用者负责确保指针不是空指针，在第二个变体中，检查指针是否为空。请参阅第三章 3，*Rust 内存模型——所有权、引用和操作*中的*Rc*部分以深入了解。我们知道，在`Arc<T>`中的`ptr`不是空指针，由于幻数数据字段，`T`的存储不在`Arc<T>`中。那会在`ArcInner<T>`中：

```rs
struct ArcInner<T: ?Sized> {
    strong: atomic::AtomicUsize,

    // the value usize::MAX acts as a sentinel for temporarily 
    // "locking" the ability to upgrade weak pointers or 
    // downgrade strong ones; this is used to avoid races
    // in `make_mut` and `get_mut`.
    weak: atomic::AtomicUsize,

    data: T,
}
```

我们看到两个内部`AtomicUsize`字段：`strong`和`weak`。像`Rc`一样，`Arc<T>`允许对内部数据有两种类型的强度引用。弱引用不拥有内部数据，允许`Arc<T>`在循环中排列而不造成不可能的内存回收情况。强引用拥有内部数据。在这里，我们看到通过`Arc<T>::clone() -> Arc<T>`创建一个新的强引用：

```rs
    fn clone(&self) -> Arc<T> {
        let old_size = self.inner().strong.fetch_add(1, Relaxed);

        if old_size > MAX_REFCOUNT {
            unsafe {
                abort();
            }
        }

        Arc { ptr: self.ptr, phantom: PhantomData }
    }
```

在这里，我们可以看到，在每次克隆时，使用 relaxed 排序时，`ArcInner<T>`的`strong`会增加一个。对此代码块的注释——由于文本简洁性在此省略——断言由于`Arc`的使用情况，relaxed 排序是足够的。在这里使用 relaxed 排序是可以的，因为对原始引用的了解可以防止其他线程错误地删除对象。这是一个重要的考虑因素。为什么克隆不需要更严格的排序？考虑一个线程`A`，它克隆了一些`Arc<T>`并将其传递给另一个线程`B`，然后，在传递给`B`之后，立即释放自己的`Arc`副本。我们可以从检查`Arc<T>::new() -> Arc<T>`中确定，在创建时，`Arc`始终有一个强引用和一个弱引用：

```rs
    pub fn new(data: T) -> Arc<T> {
        // Start the weak pointer count as 1 which is the weak 
        // pointer that's held by all the strong pointers 
        // (kinda), see std/rc.rs for more info
        let x: Box<_> = box ArcInner {
            strong: atomic::AtomicUsize::new(1),
            weak: atomic::AtomicUsize::new(1),
            data,
        };
        Arc { ptr: Shared::from(Box::into_unique(x)), 
              phantom: PhantomData }
    }
```

从线程`A`的角度来看，`Arc`的克隆由以下操作组成：

```rs
clone Arc
    INCR strong, Relaxed
    guard against strong wrap-around
    allocate New Arc
move New Arc into B
drop Arc
```

从前一章的讨论中我们知道，`Release`排序不提供因果关系。完全有可能一个敌对的 CPU 会重新排序`strong`的增加，在`New Arc`移动到`B`之后。如果`B`随后立即释放`Arc`，我们会遇到麻烦吗？我们需要检查`Arc<T>::drop()`：

```rs
    fn drop(&mut self) {
        if self.inner().strong.fetch_sub(1, Release) != 1 {
            return;
        }

        atomic::fence(Acquire);

        unsafe {
            let ptr = self.ptr.as_ptr();

            ptr::drop_in_place(&mut self.ptr.as_mut().data);

            if self.inner().weak.fetch_sub(1, Release) == 1 {
                atomic::fence(Acquire);
                Heap.dealloc(ptr as *mut u8, Layout::for_value(&*ptr))
            }
        }
    }
```

这里开始了。`atomic::fence(Acquire)`对我们来说是新的。`std::sync::atomic::fence`根据提供的内存排序建立的因果关系，防止内存迁移。栅栏适用于同一线程和其他线程的内存操作。回想一下，`Release`排序禁止在源代码顺序中向下迁移加载和存储。在这里，我们看到，强引用的加载和存储将禁止向下迁移，但不会在`Acquire`栅栏之后重新排序。因此，`T`内部的分配给`Arc`将不会发生，直到`A`和`B`两个线程都已同步并移除了所有强引用。由于建立了因果关系，一个额外的线程`C`不能在此时通过并增加内部`T`的强引用，因为`A`和`B`都不能在不增加强计数器的情况下给`C`一个强引用，这会导致`A`或`B`的释放提前退出。对于弱引用也有类似的结论。

对`Arc`进行不可变解引用不会增加强或弱计数：

```rs
impl<T: ?Sized> Deref for Arc<T> {
    type Target = T;

    #[inline]
    fn deref(&self) -> &T {
        &self.inner().data
    }
}
```

因为`drop`需要一个可变的`self`，所以在存在有效的`&T`时，无法释放`Arc<T>`。获取`&mut T`更为复杂，这是通过`Arc<T>::get_mut(&mut Self) -> Option<&mut T>`来实现的。请注意，返回值是一个`Option`。如果内部`T`有其他强或弱引用，那么消耗`Arc`是不安全的。`get_mut`的实现如下：

```rs
    pub fn get_mut(this: &mut Self) -> Option<&mut T> {
        if this.is_unique() {
            // This unsafety is ok because we're guaranteed that the
            // pointer returned is the *only* pointer that will ever
            // be returned to T. Our reference count is guaranteed
            // to be 1 at this point, and we required the Arc itself
            // to be `mut`, so we're returning the only possible 
            // reference to the inner data.
            unsafe {
                Some(&mut this.ptr.as_mut().data)
            }
        } else {
            None
        }
    }
```

其中`is_unique`是：

```rs
    fn is_unique(&mut self) -> bool {
        if self.inner().weak.compare_exchange(1, usize::MAX, 
                                              Acquire, Relaxed).is_ok() {
            let unique = self.inner().strong.load(Relaxed) == 1;
            self.inner().weak.store(1, Release);
            unique
        } else {
            false
        }
    }
```

在弱引用上进行的比较和交换操作确保只有一个弱引用存在——意味着唯一性——并在成功时使用 `Acquire` 排序。这种排序将迫使后续对强引用的检查在代码顺序中发生在弱引用检查之后，并且在释放存储之前。这种确切的技术在我们上一章关于互斥锁的讨论中已经熟悉。

# 权衡

引用计数作为原子内存回收的方法有很多优点。引用计数的编程模型易于理解，并且一旦可以安全回收，就会立即回收内存。在 Rust 中，由于缺乏双字比较和交换，不使用 `Arc<T>` 编程引用计数的实现仍然很困难。考虑一个 Treiber 栈，其中引用计数器是栈节点内部的。栈的头部可能被快照，然后由其他线程释放，后续对引用计数器的读取将被迫通过一个无效指针。

以下引用计数的 Treiber 栈存在缺陷：

```rs
use std::sync::atomic::{fence, AtomicPtr, AtomicUsize, Ordering};
use std::{mem, ptr};

unsafe impl<T: Send> Send for Stack<T> {}
unsafe impl<T: Send> Sync for Stack<T> {}

struct Node<T> {
    references: AtomicUsize,
    next: *mut Node<T>,
    data: Option<T>,
}

impl<T> Node<T> {
    fn new(t: T) -> Self {
        Node {
            references: AtomicUsize::new(1),
            next: ptr::null_mut(),
            data: Some(t),
        }
    }
}

pub struct Stack<T> {
    head: AtomicPtr<Node<T>>,
}

impl<T> Stack<T> {
    pub fn new() -> Self {
        Stack {
            head: AtomicPtr::default(),
        }
    }

    pub fn pop(&self) -> Option<T> {
        loop {
            let head: *mut Node<T> = self.head.load(Ordering::Relaxed);

            if head.is_null() {
                return None;
            }
            let next: *mut Node<T> = unsafe { (*head).next };

            if self.head.compare_and_swap(head, next, 
            Ordering::Relaxed) == head {
                let mut head: Box<Node<T>> = unsafe {
                    Box::from_raw(head) 
                };
                let data: Option<T> = mem::replace(&mut (*head).data, 
                                                   None);
                unsafe {
                    assert_eq!(
                        (*(*head).next).references.fetch_sub(1, 
                             Ordering::Release),
                        2
                    );
                }
                drop(head);
                return data;
            }
        }
    }

    pub fn push(&self, t: T) -> () {
        let node: *mut Node<T> = Box::into_raw(Box::new(Node::new(t)));
        loop {
            let head = self.head.load(Ordering::Relaxed);
            unsafe {
                (*node).next = head;
            }

            fence(Ordering::Acquire);
            if self.head.compare_and_swap(head, node, 
            Ordering::Release) == head {
                // node is now self.head
                // head is now self.head.next
                if !head.is_null() {
                    unsafe {
                        assert_eq!(1, 
                                   (*head).references.fetch_add(1, 
                                       Ordering::Release)
                        );
                    }
                }
                break;
            }
        }
    }
}
```

这本书中故意存在的一些（故意）有缺陷的例子之一。鼓励你将本书前面讨论的技术应用到识别和纠正这个实现中的缺陷。这应该很有趣。J.D. Valois 1995 年的论文，《使用比较和交换的无锁链表》，将会有所帮助。此外，强烈鼓励你尝试仅使用 `Arc` 实现一个 Treiber 栈。

最终，当操作同一引用计数字据的线程数量增加时，引用计数的竞争问题就会出现，那些获取/释放对并不便宜。当预期线程数量相对较低或绝对性能不是关键考虑因素时，引用计数是一个很好的模型。否则，你需要调查以下方法，这些方法也有自己的权衡。

# 危险指针

我们在上一章中提到了危险指针作为安全内存回收的方法。现在，我们将深入探讨这个概念，并通过对 Redox 项目（[`crates.io/crates/conc`](https://crates.io/crates/conc)）的更多或更少的可用实现进行考察。我们将检查 `conc` 作为其父项目 tfs（[`github.com/redox-os/tfs`](https://github.com/redox-os/tfs)）的一部分，在 SHA `3e7dcdb0c586d0d8bb3f25bfd948d2f418a4ab10`。顺便说一句，如果你不熟悉这个，Redox 是一个基于 Rust 微内核的操作系统。分配器、coreutils 和 netutils 都是推荐阅读的内容。

Conc crate 并没有列出全部内容。你可以在本书的源代码仓库中找到完整的列表。

危险指针是由 Maged Michael——以 Michael 和 Scott 队列闻名——在他的 2004 年著作《Hazard Pointers: Safe Memory Reclamation for Lock-Free Objects》中引入的。在这个上下文中，“危险”是指任何在多个线程参与某些共享数据结构的 rendezvous 中可能发生竞态或 ABA 问题的指针。在上一节的 Treiber 栈中，危险是指 Stack 结构体内部的头指针，而在 Michael 和 Scott 队列中，它是头和尾指针。危险指针将这些危险与由单个写入线程拥有并由任何其他参与数据结构的线程读取的 *单写入多读取共享指针* 关联起来。

每个参与线程都维护对其自己的危险指针和所有其他参与线程的引用，以及用于协调分配的私有、线程局部列表。Michael 论文第三部分中的算法描述是在高层次上进行的，并且很难通过具体实现来跟踪。而不是在这里重复，我们将检查一个具体的实现，`conc`。鼓励读者在讨论 `conc` 之前先阅读 Michael 的论文，但这不是强制性的。

# 一个危险指针 Treiber 栈

在上一节关于引用计数的部分中引入 Treiber 栈并非无的放矢。我们将通过一个有效回收的 Treiber 栈的视角来检查危险指针和基于纪元的回收。碰巧的是，`conc` 包含了一个 Treiber 栈，位于 `tfs/conc/src/sync/treiber.rs`。前言部分大部分是熟悉的：

```rs
use std::sync::atomic::{self, AtomicPtr};
use std::marker::PhantomData;
use std::ptr;
use {Guard, add_garbage_box};
```

`Guard` 和 `add_garbage_box` 都是新的，但我们将直接进入正题。`Treiber` 结构体正如你所想象的那样：

```rs
pub struct Treiber<T> {
    /// The head node.
    head: AtomicPtr<Node<T>>,
    /// Make the `Sync` and `Send` (and other OIBITs) transitive.
    _marker: PhantomData<T>,
}
```

节点也是一样：

```rs
struct Node<T> {
    /// The data this node holds.
    item: T,
    /// The next node.
    next: *mut Node<T>,
}
```

这是我们需要理解 `conc::Guard` 的地方。与标准库中的 `MutexGuard` 类似，`Guard` 在这里存在是为了保护包含的数据不被多次修改。`Guard` 是 `conc` 的危险指针接口。让我们详细检查 `Guard` 并了解危险指针算法。`Guard` 定义在 `tfs/conc/src/guard.rs`：

```rs
pub struct Guard<T: 'static + ?Sized> {
    /// The inner hazard.
    hazard: hazard::Writer,
    /// The pointer to the protected object.
    pointer: &'static T,
}
```

在撰写本书时，所有由 `Guard` 保护的 `T` 都必须是静态的，但我们将忽略这一点，因为代码库中引用了放宽该限制的愿望。如果你想在项目中立即使用 `conc`，请注意这一点。就像 `MutexGuard` 一样，`Guard` 并不拥有底层数据，而只是保护对它的引用，我们可以看到我们的 `Guard` 是危险指针的写入端。那么什么是 `hazard::Writer`？它在 `tfs/conc/src/hazard.rs` 中定义：

```rs
pub struct Writer {
    /// The pointer to the heap-allocated hazard.
    ptr: &'static AtomicPtr<u8>,
}
```

指向堆分配的危险指针？好吧，让我们退一步。我们知道从算法描述中，必须有与堆存储协调的线程局部存储。我们还知道 `Guard` 是我们访问危险指针的主要接口。有三个函数用于创建 `Guard` 的新实例：

+   `Guard<T>::new<F: FnOnce() -> &'static T>(ptr: F) -> Guard<T>`

+   `Guard<T>::maybe_new<F: FnOnce() -> Option<&'static T>>(ptr: F) -> Option<Guard<T>>`

+   `Guard<T>::try_new<F: FnOnce() -> Result<&'static T, E>>(ptr: F) -> Result<Guard<T>, E>`

前两个是通过`try_new`定义的。让我们剖析一下`try_new`：

```rs
    pub fn try_new<F, E>(ptr: F) -> Result<Guard<T>, E>
    where F: FnOnce() -> Result<&'static T, E> {
        // Increment the number of guards currently being created.
        #[cfg(debug_assertions)]
        CURRENT_CREATING.with(|x| x.set(x.get() + 1));
```

函数接受`FnOnce`，其责任是提供一个`T`的引用或失败。`try_new`中的第一个调用是`CURRENT_CREATING`的增加，它是一个`thread-local Cell<usize>`：

```rs
thread_local! {
    /// Number of guards the current thread is creating.
    static CURRENT_CREATING: Cell<usize> = Cell::new(0);
}
```

我们在第三章中看到了`Cell`，*《Rust 内存模型——所有权、引用和操作*，但`thread_local!`是新的。这个宏将一个或多个静态值包装到`std::thread::LocalKey`中，这是 Rust 对线程局部存储的实现。具体的实现因平台而异，但基本思想是一致的：值将作为静态全局变量，但将限制在单个线程中。这样，我们可以像使用全局变量一样编程，而无需管理线程之间的协调。一旦`CURRENT_CREATING`增加：

```rs
        // Get a hazard in blocked state.
        let hazard = local::get_hazard();
```

`local::get_hazard`函数定义在`tfs/conc/src/local.rs`中：

```rs
pub fn get_hazard() -> hazard::Writer {
    if STATE.state() == thread::LocalKeyState::Destroyed {
        // The state was deinitialized, so we must rely on the
        // global state for creating new hazards.
        global::create_hazard()
    } else {
        STATE.with(|s| s.borrow_mut().get_hazard())
    }
}
```

此函数中引用的`STATE`是：

```rs
thread_local! {
    /// The state of this thread.
    static STATE: RefCell<State> = RefCell::new(State::default());
}
```

`State`的定义如下：

```rs
#[derive(Default)]
struct State {
    garbage: Vec<Garbage>,
    available_hazards: Vec<hazard::Writer>,
    available_hazards_free_before: usize,
}
```

字段垃圾维护了一个尚未移动到全局状态的`Garbage`列表——关于这一点，我们稍后再谈——用于回收。`Garbage`是一个指向字节的指针，以及一个指向字节的函数指针，称为`dtor`，用于析构函数。内存回收方案必须能够释放，无论底层类型如何。常见的做法，也是`Garbage`采取的做法，是在类型信息可用时构建一个单态化析构函数，否则在字节缓冲区上工作。我们鼓励你自己浏览`Garbage`的实现，但主要的技巧是`Box::from_raw(ptr as *mut u8 as *mut T)`，这是我们在这本书中反复看到的。

`available_hazards`字段存储了之前分配但尚未使用的分配器危害写入器。实现将其作为缓存以避免分配器抖动。我们可以在`local::State::get_hazard`中看到这一点：

```rs
    fn get_hazard(&mut self) -> hazard::Writer {
        // Check if there is hazards in the cache.
        if let Some(hazard) = self.available_hazards.pop() {
            // There is; we don't need to create a new hazard.
            //
            // Since the hazard popped from the cache is not 
            // blocked, we must block the hazard to satisfy 
            // the requirements of this function.
            hazard.block();
            hazard
        } else {
            // There is not; we must create a new hazard.
            global::create_hazard()
        }
    }
```

最后一个字段`available_hazards_free_before`存储了在释放底层类型之前的释放状态中的危害。我们稍后会进一步讨论这个问题。危害处于四种状态之一：空闲、已死亡、阻塞或保护。一个已死亡的危害可以安全地释放，包括受保护的内存。一个已死亡的危害不应该被读取。一个空闲的危害没有保护任何东西，可以重用。一个阻塞的危害正被其他线程使用，并且会导致危害读取停滞。一个保护危害正在保护一些内存。现在，回到`local::get_hazard`的这个分支：

```rs
    if STATE.state() == thread::LocalKeyState::Destroyed {
        // The state was deinitialized, so we must rely on the 
        // global state for creating new hazards.
        global::create_hazard()
    } else {
```

`global::create_hazard`是什么？此模块是`tfs/conc/src/global.rs`，函数如下：

```rs
pub fn create_hazard() -> hazard::Writer {
    STATE.create_hazard()
}
```

变量名令人困惑。这个 `STATE` 不是线程本地的 `STATE`，而是一个全局作用域的 `STATE`：

```rs
lazy_static! {
    /// The global state.
    ///
    /// This state is shared between all the threads.
    static ref STATE: State = State::new();
}
```

让我们深入探讨：

```rs
struct State {
    /// The message-passing channel.
    chan: mpsc::Sender<Message>,
    /// The garbo part of the state.
    garbo: Mutex<Garbo>,
}
```

全局 `STATE` 是 `Message` 的 mpsc `Sender` 和一个互斥锁保护的 `Garbo`。`Message` 是一个简单的枚举：

```rs
enum Message {
    /// Add new garbage.
    Garbage(Vec<Garbage>),
    /// Add a new hazard.
    NewHazard(hazard::Reader),
}
```

`Garbo` 是我们将直接探讨的内容。现在只需说，`Garbo` 作为此实现的全球垃圾回收器。全局状态设置了一个通道，维护全局状态中的发送方，并将接收方喂入 `Garbo`：

```rs
impl State {
    /// Initialize a new state.
    fn new() -> State {
        // Create the message-passing channel.
        let (send, recv) = mpsc::channel();

        // Construct the state from the two halfs of the channel.
        State {
            chan: send,
            garbo: Mutex::new(Garbo {
                chan: recv,
                garbage: Vec::new(),
                hazards: Vec::new(),
            })
        }
    }
```

创建一个新的全局风险并不需要太多：

```rs
    fn create_hazard(&self) -> hazard::Writer {
        // Create the hazard.
        let (writer, reader) = hazard::create();
        // Communicate the new hazard to the global state 
        // through the channel.
        self.chan.send(Message::NewHazard(reader));
        // Return the other half of the hazard.
        writer
    }
```

这通过 `hazard::create()` 建立了一个新的风险，并将读取器侧向下传递到 `Garbo`，将写入器侧返回到 `local::get_hazard()`。虽然 writer 和 reader 的名字暗示风险本身是一个 MPSC，但这并不正确。风险模块是 `tfs/conc/src/hazard.rs`，创建如下：

```rs
pub fn create() -> (Writer, Reader) {
    // Allocate the hazard on the heap.
    let ptr = unsafe {
        &*Box::into_raw(Box::new(AtomicPtr::new(
                        &BLOCKED as *const u8 as *mut u8)))
    };

    // Construct the values.
    (Writer {
        ptr: ptr,
    }, Reader {
        ptr: ptr,
    })
}
```

好吧，看看这个。我们这里有两个结构体，`Writer` 和 `Reader`，它们各自存储指向堆分配的原子指针的可变字节指针的相同原始指针。呼！我们之前已经看到过这个技巧，但这里特殊的是利用类型系统为同一块原始内存提供不同的读写接口。

那么 `Garbo` 呢？它在全局模块中定义，定义为：

```rs
struct Garbo {
    /// The channel of messages.
    chan: mpsc::Receiver<Message>,
    /// The to-be-destroyed garbage.
    garbage: Vec<Garbage>,
    /// The current hazards.
    hazards: Vec<hazard::Reader>,
}
```

`Garbo` 定义了一个 `gc` 函数，该函数从其通道读取所有 `Messages`，将垃圾存储到垃圾字段，将自由风险存储到风险字段。已死亡的风险被销毁，释放其存储空间，因为其他持有者已经保证已经挂起。受保护的风险也进入风险字段，这些字段将在下一次调用 gc 时被扫描。当线程调用 `global::tick()` 或调用 `global::try_gc()` 时，有时会执行垃圾回收。每当调用 `local::add_garbage` 时，就会执行一个 tick，这是 `whatconc::add_garbage_box` 调用的。

我们在本节的开头首次遇到了 `add_barbage_box`。每当一个线程将一个节点标记为垃圾时，它掷骰子，并可能成为负责在所有线程的风险指针内存上执行全局垃圾回收的责任人。

现在我们已经了解了内存回收的工作原理，剩下要理解的是风险指针如何保护内存免受读写。让我们一次性完成 `guard::try_new`：

```rs
        // This fence is necessary for ensuring that `hazard` does not
        // get reordered to after `ptr` has run.
        // TODO: Is this fence even necessary?
        atomic::fence(atomic::Ordering::SeqCst);

        // Right here, any garbage collection is blocked, due to the
        // hazard above. This ensures that between the potential 
        // read in `ptr` and it being protected by the hazard, there
        // will be no premature free.

        // Evaluate the pointer through the closure.
        let res = ptr();

        // Decrement the number of guards currently being created.
        #[cfg(debug_assertions)]
        CURRENT_CREATING.with(|x| x.set(x.get() - 1));

        match res {
            Ok(ptr) => {
                // Now that we have the pointer, we can protect it by 
                // the hazard, unblocking a pending garbage collection
                // if it exists.
                hazard.protect(ptr as *const T as *const u8);

                Ok(Guard {
                    hazard: hazard,
                    pointer: ptr,
                })
            },
            Err(err) => {
                // Set the hazard to free to ensure that the hazard 
                // doesn't remain blocking.
                hazard.free();

                Err(err)
            }
        }
    }
```

我们可以看到，conc 的作者插入了一个他们质疑的顺序一致性栅栏。Michael 提出的模型不需要顺序一致性，我相信这个栅栏是不必要的，因为它会显著降低性能。这里要注意的关键点是 `hazard::protect` 调用和 `'hazard::free'`。这两个调用都是 `hazard::Writer` 的部分，前者将内部指针设置为提供给它的字节指针，后者将危害标记为自由。这两种状态都与垃圾回收器交互，正如我们所见。剩下的部分与 `hardard::Reader::get` 有关，这是用来检索危害状态的函数。这里就是它：

```rs
impl Reader {
    /// Get the state of the hazard.
    ///
    /// It will spin until the hazard is no longer in a blocked state, 
    /// unless it is in debug mode, where it will panic given enough
    /// spins.
    pub fn get(&self) -> State {
        // In debug mode, we count the number of spins. In release 
        // mode, this should be trivially optimized out.
        let mut spins = 0;

        // Spin until not blocked.
        loop {
            let ptr = self.ptr.load(atomic::Ordering::Acquire) 
                          as *const u8;

            // Blocked means that the hazard is blocked by another 
            // thread, and we must loop until it assumes another 
            // state.
            if ptr == &BLOCKED {
                // Increment the number of spins.
                spins += 1;
                debug_assert!(spins < 100_000_000, "\
                    Hazard blocked for 100 millions rounds. Panicking 
                    as chances are that it will \
                    never get unblocked.\
                ");

                continue;
            } else if ptr == &FREE {
                return State::Free;
            } else if ptr == &DEAD {
                return State::Dead;
            } else {
                return State::Protect(ptr);
            }
```

只有当危害被阻塞时，状态的获取才会旋转，直到它死亡、自由或仅仅被保护。什么阻塞了危害？回想一下，它们是在阻塞状态下创建的。通过在阻塞状态下创建危害，我们无法在未完全初始化的指针上执行垃圾回收——这是我们参考计数实现中遇到的问题——也无法从部分初始化的危害指针中读取。只有当指针移动到保护状态后，读取才能继续。

就这样——垃圾回收和原子隔离。

让我们一路追溯到栈，看看其推送实现：

```rs
    pub fn push(&self, item: T)
    where T: 'static {
        let mut snapshot = Guard::maybe_new(|| unsafe {
            self.head.load(atomic::Ordering::Relaxed).as_ref()
        });

        let mut node = Box::into_raw(Box::new(Node {
            item: item,
            next: ptr::null_mut(),
        }));

        loop {
            let next = snapshot.map_or(ptr::null_mut(), 
                                       |x| x.as_ptr() as *mut _);
            unsafe { (*node).next = next; }

            match Guard::maybe_new(|| unsafe {
                self.head.compare_and_swap(next, node, 
                    atomic::Ordering::Release).as_ref()
            }) {
                Some(ref new) if new.as_ptr() == next => break,
                None if next.is_null() => break,
                // If it fails, we will retry the CAS with updated 
                // values.
                new => snapshot = new,
            }
        }
    }
```

在函数顶部，实现将头节点的危害指针加载到快照中。`Guard::as_ptr(&self) -> *const T` 在每次调用时检索危害的当前指针，随着底层数据的向前移动而适应。节点是包含 `item: T` 的已分配和原始指针 `Node`。循环的其余部分与我们所见到的其他此类数据结构的比较和交换相同，只是用危害 `Guard` 而不是原始 `AtomicPtrs` 或类似的东西。编程模型非常直接，就像 `pop` 一样：

```rs
    pub fn pop(&self) -> Option<Guard<T>> {
        let mut snapshot = Guard::maybe_new(|| unsafe {
            self.head.load(atomic::Ordering::Acquire).as_ref()
        });

        while let Some(old) = snapshot {
            snapshot = Guard::maybe_new(|| unsafe {
                self.head.compare_and_swap(
                    old.as_ptr() as *mut _,
                    old.next as *mut Node<T>,
                    atomic::Ordering::Release,
                ).as_ref()
            });

            if let Some(ref new) = snapshot {
                if new.as_ptr() == old.as_ptr() {
                    unsafe { add_garbage_box(old.as_ptr()); }
                    return Some(old.map(|x| &x.item));
                }
            } else {
                break;
            }
        }

        None
    }
```

注意，当旧节点从栈中移除时，会对其调用 `add_garbage_box`，将节点添加到稍后进行垃圾回收。此外，通过检查我们知道，这个“稍后”可能正好是调用的那一刻，这取决于运气。

# 夜间危害

不幸的是，conc 依赖于 Rust 编译器的一个不稳定特性，并且无法与最近的 `rustc` 编译。`conc` 以及其父项目 TFS 的最后一次更新是在 2017 年 8 月。我们将不得不回到那个时期的夜间 `rustc`。如果你遵循了 第一章 中概述的说明，*预备知识 – 计算机架构和 Rust 入门*，并且使用 `rustup`，那么只需简单地使用 `rustup` 默认的 nightly-2017-08-17。当你玩完 conc 后，别忘了切换回更现代的 Rust。

# 练习危害指针 Treiber 栈

让我们用 conc、Treiber 栈来测试这个方法，以了解其性能。对我们来说，关键的兴趣领域包括：

+   每秒的推入/弹出周期

+   内存行为，最高水位线，等等

+   缓存行为

我们将在 x86 和 ARM 上运行我们的程序，就像前几章所做的那样。在此进行之前需要注意的一点是，至少在版本 0.5 中，conc 需要使用夜间频道进行编译。如果您在书中跳来跳去，请阅读本节之前的部分以获取完整详情。

因为我们必须运行在旧版本的夜间版本上，所以我们不得不将静态 `AtomicUsize` 放入 `lazy_static!` 中，这是一种你会在旧 Rust 代码中看到但通常不会在这本书中看到的技巧。考虑到这一点，以下是我们练习程序：

```rs
extern crate conc;
#[macro_use]
extern crate lazy_static;
extern crate num_cpus;
extern crate quantiles;

use conc::sync::Treiber;
use quantiles::ckms::CKMS;
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::{thread, time};

lazy_static! {
    static ref WORKERS: AtomicUsize = AtomicUsize::new(0);
    static ref COUNT: AtomicUsize = AtomicUsize::new(0);
}
static MAX_I: u32 = 67_108_864; // 2 ** 26

fn main() {
    let stk: Arc<Treiber<(u64, u64, u64)>> = Arc::new(Treiber::new());

    let mut jhs = Vec::new();

    let cpus = num_cpus::get();
    WORKERS.store(cpus, Ordering::Release);

    for _ in 0..cpus {
        let stk = Arc::clone(&stk);
        jhs.push(thread::spawn(move || {
            for i in 0..MAX_I {
                stk.push((i as u64, i as u64, i as u64));
                stk.pop();
                COUNT.fetch_add(1, Ordering::Relaxed);
            }
            WORKERS.fetch_sub(1, Ordering::Relaxed)
        }))
    }

    let one_second = time::Duration::from_millis(1_000);
    let mut iter = 0;
    let mut cycles: CKMS<u32> = CKMS::new(0.001);
    while WORKERS.load(Ordering::Relaxed) != 0 {
        let count = COUNT.swap(0, Ordering::Relaxed);
        cycles.insert((count / cpus) as u32);
        println!(
            "CYCLES PER SECOND({}):\n  25th: \
            {}\n  50th: {}\n  75th: \
            {}\n  90th: {}\n  max:  {}\n",
            iter,
            cycles.query(0.25).unwrap().1,
            cycles.query(0.50).unwrap().1,
            cycles.query(0.75).unwrap().1,
            cycles.query(0.90).unwrap().1,
            cycles.query(1.0).unwrap().1
        );
        thread::sleep(one_second);
        iter += 1;
    }

    for jh in jhs {
        jh.join().unwrap();
    }
}
```

我们有与目标机器中 CPU 总数相等的多个工作线程，每个线程执行一次推入然后立即弹出栈的操作。一个 `COUNT` 被保持，主线程大约每秒交换那个值到 `0`，并将该值投入到一个分位数估计结构中——在 第四章 中更详细地讨论了这一点，*同步和发送——Rust 并发的基础*——并打印出每秒记录的周期摘要，按 CPU 数量进行缩放。工作线程将周期推进到 `MAX_I`，这个值任意设置为 `2**26` 的小值。当工作线程完成周期后，它减少 `WORKERS` 并退出。一旦 `WORKERS` 达到零，主循环也退出。

在我的 x86 机器上，这个程序退出大约需要 58 秒，输出如下：

```rs
CYCLES PER SECOND(0):
  25th: 0
  50th: 0
  75th: 0
  90th: 0
  max:  0

CYCLES PER SECOND(1):
  25th: 0
  50th: 0
  75th: 1124493
  90th: 1124493
  max:  1124493

...

CYCLES PER SECOND(56):
  25th: 1139055
  50th: 1141656
  75th: 1143781
  90th: 1144324
  max:  1145284

CYCLES PER SECOND(57):
  25th: 1139097
  50th: 1141656
  75th: 1143792
  90th: 1144324
  max:  1145284
CYCLES PER SECOND(58):
  25th: 1139097
  50th: 1141809
  75th: 1143792
  90th: 1144398
  max:  1152384
```

x86 的性能运行如下：

```rs
> perf stat --event task-clock,context-switches,page-faults,cycles,instructions,branches,branch-misses,cache-references,cache-misses target/release/conc_stack > /dev/null

 Performance counter stats for 'target/release/conc_stack':

     230503.932161      task-clock (msec)         #    3.906 CPUs utilized
               943      context-switches          #    0.004 K/sec
             9,140      page-faults               #    0.040 K/sec
   665,734,124,408      cycles                    #    2.888 GHz 
   529,230,473,047      instructions              #    0.79  insn per cycle 
    99,146,219,517      branches                  #  430.128 M/sec 
     1,140,398,326      branch-misses             #    1.15% of all branches
    12,759,284,944      cache-references          #   55.354 M/sec
       124,487,844      cache-misses              #    0.976 % of all cache refs

      59.006842741 seconds time elapsed
```

这个程序的内存分配行为也非常有利。

在我的 ARM 机器上，程序运行到完成大约需要 460 秒，输出如下：

```rs
CYCLES PER SECOND(0):
  25th: 0
  50th: 0
  75th: 0
  90th: 0
  max:  0
CYCLES PER SECOND(1):
  25th: 0
  50th: 0
  75th: 150477
  90th: 150477
  max:  150477

...

CYCLES PER SECOND(462):
  25th: 137736
  50th: 150371
  75th: 150928
  90th: 151129
  max:  151381

CYCLES PER SECOND(463):
  25th: 137721
  50th: 150370
  75th: 150928
  90th: 151129
  max:  151381
```

这比 x86 机器慢大约 7.5 倍，尽管两台机器的核心数量相同。x86 机器的时钟和内存总线比我的 ARM 快：

```rs
> perf stat --event task-clock,context-switches,page-faults,cycles,instructions,branches,branch-misses,cache
-references,cache-misses target/release/conc_stack > /dev/null

 Performance counter stats for 'target/release/conc_stack':

    1882745.172714      task-clock (msec)         #    3.955 CPUs utilized
                 0      context-switches          #    0.000 K/sec
             3,212      page-faults               #    0.002 K/sec
 2,109,536,434,019      cycles                    #    1.120 GHz
 1,457,344,087,067      instructions              #    0.69  insn per cycle
   264,210,403,390      branches                  #  140.333 M/sec
    36,052,283,984      branch-misses             #   13.65% of all branches
   642,854,259,575      cache-references          #  341.445 M/sec
     2,401,725,425      cache-misses              #    0.374 % of all cache refs

     476.095973090 seconds time elapsed
```

ARM 上的分支缺失率比 x86 高 12%，这是一个有趣的结果。

在继续之前，请记住执行 rustup default stable。

# 折衷

假设 conc 已经与现代化的 Rust 同步，您希望在哪里应用它？好吧，让我们将其与引用计数进行比较。在引用计数的数据结构中，每次读取和修改都需要操作原子值，这意味着线程之间需要一定程度的同步，这会影响性能。在危险指针的情况下也存在类似的情况，尽管程度较小。正如我们所见，危险指针需要同步，但通过它读取除危险指针之外的内容，将会以机器速度进行。这比引用计数有显著的改进。此外，我们知道危险指针方法确实会积累垃圾——而 conc 的实现是通过一个特别巧妙的方法完成的——但无论更新危险结构的频率如何，这种垃圾的大小都是有限的。垃圾回收的成本被放在一个线程上，这可能导致某些操作的高延迟峰值，但同时也从批量操作调用分配器中受益。这与引用计数方法不同，在引用计数方法中，每个线程都负责在内存死亡时立即释放死亡内存，保持操作成本开销相似但高于基线，并且在每个操作上产生，而没有批量积累释放操作的好处。随着线程数量的增加，危险指针方法会面临挑战：回想一下，所需的危险数量会随着线程数量的线性增长而增长，但也要考虑到线程数量的增加会增加任何结构的遍历成本。此外，虽然许多结构都有容易识别的危险点，但这并不普遍适用。对于 b 树，您将把危险指针放在哪里？

总结来说——当您需要处理的危险内存引用相对较少，并且需要有限制的垃圾回收，无论结构上的负载如何时，危险指针是一个非常好的选择。基于危险的内存回收并不具备基于时代的回收的原始速度，但在所有情况下有限的内存使用都是难以放弃的。

# 基于时代的回收

在上一章中，我们提到了基于纪元的内存回收。现在，我们将深入探讨这一概念，并通过对 crossbeam 项目（[`github.com/crossbeam-rs`](https://github.com/crossbeam-rs)）的现成实现进行考察。一些 conc 的开发者与 crossbeam 有交集，但为了追求 Rust 中的低级原子编程的通用框架，crossbeam 上做了更多的工作。开发者们指出，他们计划支持其他内存管理方案，例如危害指针（HP）和静默状态基于回收（QSBR）。该包被重新导出为纪元模块。本书的未来版本可能只会涵盖 crossbeam，并简单地讨论其中嵌入的各种内存回收技术。本节我们将讨论的特定包是 crossbeam-epoch，其 SHA 值为 `3179e799f384816a0a9f2a3da82a147850e14d38`。这与项目的 0.4.1 版本相一致。

这里讨论的 crossbeam 包并不完整。您可以在本书的源代码库中找到完整的列表。

基于纪元的内存回收是由 Keir Fraser 在他的 2004 年博士论文《实用锁自由》中引入的。Fraser 的工作建立在之前的 limbo 列表方法之上，其中垃圾被放置在全局列表中，并通过周期性扫描回收，当可以证明垃圾不再被引用时。确切的机制因作者而异，但 limbo 列表有一些严重的正确性和性能缺点。Fraser 的改进是将 limbo 列表分为三个纪元：当前纪元、中间纪元和可以回收垃圾的纪元。每个参与线程在启动时负责观察当前纪元，将其垃圾添加到当前纪元，并在其操作完成后尝试增加纪元计数。最后一个完成操作的线程将成功增加纪元，并负责在最终纪元上执行垃圾回收。当线程独立移动到纪元时，需要三个总纪元：一个位于纪元 n 的线程可能仍然持有来自 n-1 的引用，但不能持有来自 n-2 的引用，因此只有 n-2 是安全的可以回收。Fraser 的论文相当长——毕竟，这是获得博士学位的足够工作量——但 Hart 等人 2007 年的论文《无锁同步的内存回收性能》强烈推荐给时间紧迫的读者。

# 基于纪元的 Treiber 栈

我们在本章早期检查了基于引用计数和危害指针的 Treiber 栈，本节也将采用这种方法。幸运的是，crossbeam 还提供了一个 Treiber 栈实现，位于 crossbeam-rs/crossbeam 项目中，我们将在 SHA 值为 `89bd6857cd701bff54f7a8bf47ccaa38d5022bfb` 的位置进行考察，这是 `crossbeam::sync::TreiberStack` 的来源，位于 `src/sync/treiber_stack.rs`。前言部分几乎立即就很有趣：

```rs
use std::sync::atomic::Ordering::{Acquire, Relaxed, Release};
use std::ptr;

use epoch::{self, Atomic, Owned};
```

比如，`epoch::Atomic`和`epoch::Owned`是什么意思？嗯，我们在这里会探讨这些。与探索 conc 实现的深度优先方法不同，我们将采取广度优先的方法，或多或少。原因在于 crossbeam-epoch 很复杂——很容易迷路。抛开对`epoch::Atomic`的某种未知性质，`TreiberStack`和`Node`结构体与我们看到的其他实现类似：

```rs
#[derive(Debug)]
pub struct TreiberStack<T> {
    head: Atomic<Node<T>>,
}

#[derive(Debug)]
struct Node<T> {
    data: T,
    next: Atomic<Node<T>>,
}
```

以及创建一个新的栈：

```rs
impl<T> TreiberStack<T> {
    /// Create a new, empty stack.
    pub fn new() -> TreiberStack<T> {
        TreiberStack {
            head: Atomic::null(),
        }
    }
```

推入一个新元素看起来也很熟悉：

```rs
    pub fn push(&self, t: T) {
        let mut n = Owned::new(Node {
            data: t,
            next: Atomic::null(),
        });
        let guard = epoch::pin();
        loop {
            let head = self.head.load(Relaxed, &guard);
            n.next.store(head, Relaxed);
            match self.head.compare_and_set(head, n, Release, &guard) {
                Ok(_) => break,
                Err(e) => n = e.new,
            }
        }
    }
```

除了，`epoch::pin()`是什么意思？这个函数`pins`当前线程，返回一个`crossbeam_epoch::Guard`。就像我们在书中看到的之前的守卫一样，crossbeam_epoch 的`Guard`保护一个资源。在这种情况下，守卫保护线程的固定性质。一旦守卫释放，线程就不再固定。好吧，太好了，那么线程固定意味着什么呢？回想一下，基于 epoch 的回收是通过线程进入一个 epoch，对内存进行一些操作，然后在完成与内存的交互后可能增加 epoch 来工作的。这个过程在 crossbeam 中通过固定来实现。持有`Guard`意味着已经进入了一个安全的 epoch——线程能够进行堆分配并获取其栈引用。然而，任何旧的堆分配和栈引用都是不安全的。这里没有魔法。这就是`Atomic`和`Owned`出现的地方。它们是标准库中的`AtomicPtr`和`Box`的类似物，只不过对它们的操作需要`crossbeam_epoch::Guard`的引用。我们在第四章中看到了这种技术，当时在讨论 hopper 时，使用类型系统确保通过要求调用者传递某种类型的守卫来安全地执行可能不安全的线程操作。程序员错误可能会通过使用`AtomicPtr`、`Box`或任何非 epoch 未受保护的内存访问而悄悄出现，但这些通常会突出显示。

让我们看看从栈中弹出元素的情况，我们知道标记内存为可删除的安全操作将必须发生：

```rs
    pub fn try_pop(&self) -> Option<T> {
        let guard = epoch::pin();
        loop {
            let head_shared = self.head.load(Acquire, &guard);
            match unsafe { head_shared.as_ref() } {
                Some(head) => {
                    let next = head.next.load(Relaxed, &guard);
                    if self.head
                        .compare_and_set(head_shared, next, Release, 
                         &guard)
                        .is_ok()
                    {
                        unsafe {
                            guard.defer(move || 
                            head_shared.into_owned());
                            return Some(ptr::read(&(*head).data));
                        }
                    }
                }
                None => return None,
            }
        }
    }
```

需要注意的关键点是通过对 Atomic 进行加载创建`head_shared`，然后是常规的比较和设置操作，以及以下内容：

```rs
    guard.defer(move || head_shared.into_owned());
    return Some(ptr::read(&(*head).data));
```

`head_shared`被移动到一个闭包中，转换，然后这个闭包被传递到`Guard`尚未检查的 defer 函数中。但是，头被解引用，并从中读取数据并返回。如果没有一些特殊的技巧，这是危险的。我们需要了解更多信息。

# crossbeam_epoch::Atomic

让我们深入探讨我们的数据持有者`Atomic`。这个结构来自 crossbeam_epoch 库，其实现位于`src/atomic.rs`：

```rs
pub struct Atomic<T> {
    data: AtomicUsize,
    _marker: PhantomData<*mut T>,
}

unsafe impl<T: Send + Sync> Send for Atomic<T> {}
unsafe impl<T: Send + Sync> Sync for Atomic<T> {}
```

表示有点奇怪：为什么不直接是 `data: *mut T`？Crossbeam 的开发者在这里做了一些巧妙的事情。在现代机器上，指针内部有相当多的空间来存储信息，在最低有效位。考虑一下，如果我们只指向对齐的数据，内存地址在 32 位系统上将是 4 的倍数，在 64 位系统上是 8 的倍数。这会在 32 位系统上留下指针末尾的两个零位，在 64 位系统上留下三个零位。这些位可以存储信息，称为标记。`Atomic` 可以被标记的事实将发挥作用。但是，Rust 不允许我们像其他系统编程语言那样操作指针。为此，并且因为 `usize` 是机器字的大小，crossbeam 将其 `*mut T` 存储为 `usize`，允许对指针进行标记。空 `Atomics` 等价于指向零的标记指针：

```rs
    pub fn null() -> Atomic<T> {
        Self {
            data: ATOMIC_USIZE_INIT,
            _marker: PhantomData,
        }
    }
```

新的 `Atomic` 通过将 `T` 传递给 `Owned`—这是我们之前看到的，然后从那个 `Owned` 转换而来来创建：

```rs
    pub fn new(value: T) -> Atomic<T> {
        Self::from(Owned::new(value))
    }
```

`Owned` 盒子 `T`，执行堆分配：

```rs
impl<T> Owned<T> {
    pub fn new(value: T) -> Owned<T> {
        Self::from(Box::new(value))
    }
```

然后将那个盒转换为 `*mut T`：

```rs
impl<T> From<Box<T>> for Owned<T> {
    fn from(b: Box<T>) -> Self {
        unsafe { Self::from_raw(Box::into_raw(b)) }
    }
}
```

然后将它转换为一个标记指针：

```rs
pub unsafe fn from_raw(raw: *mut T) -> Owned<T> {
    ensure_aligned(raw);
    Self::from_data(raw as usize)
}
```

这有点远，但那告诉我们 `Atomic` 在数据表示方面与 `AtomicPtr` 没有区别，只是指针本身被标记了。

现在，关于原子指针的常规操作，如加载等，又是如何不同的？在 `Atomic` 中，这些有什么区别？好吧，首先，我们知道每个调用都必须传递一个 epoch `Guard`，但还有其他重要的区别。这是 `Atomic::load`：

```rs
    pub fn load<'g>(&self, ord: Ordering, _: &'g Guard) 
        -> Shared<'g, T> 
    {
        unsafe { Shared::from_data(self.data.load(ord)) }
    }
```

我们可以看到 `self.data.load(ord)`，所以底层原子加载是按预期进行的。但是，什么是 `Shared`？

```rs
pub struct Shared<'g, T: 'g> {
    data: usize,
    _marker: PhantomData<(&'g (), *const T)>,
}
```

它是对 `Atomic` 的引用，其中嵌入了 `Guard` 的引用。只要 `Shared` 存在，对它进行内存操作的安全 `Guard` 也会存在，并且，重要的是，不能在 `Shared` 被丢弃之前停止存在。`Atomic::compare_and_set` 引入了一些额外的特质：

```rs
    pub fn compare_and_set<'g, O, P>(
        &self,
        current: Shared<T>,
        new: P,
        ord: O,
        _: &'g Guard,
    ) -> Result<Shared<'g, T>, CompareAndSetError<'g, T, P>>
    where
        O: CompareAndSetOrdering,
        P: Pointer<T>,
    {
        let new = new.into_data();
        self.data
            .compare_exchange(current.into_data(), new,
                              ord.success(), ord.failure())
            .map(|_| unsafe { Shared::from_data(new) })
            .map_err(|current| unsafe {
                CompareAndSetError {
                    current: Shared::from_data(current),
                    new: P::from_data(new),
                }
            })
    }
```

注意到 `compare_and_set` 是基于 `compare_exchange` 定义的。这个 CAS 原语相当于比较交换，但那个交换允许失败具有更宽松的语义，在某些平台上提供性能提升。比较和设置的实现需要理解哪种成功 `Ordering` 与哪种失败 `Ordering` 匹配，从而产生 `CompareAndSetOrdering`：

```rs
pub trait CompareAndSetOrdering {
    /// The ordering of the operation when it succeeds.
    fn success(&self) -> Ordering;

    /// The ordering of the operation when it fails.
    ///
    /// The failure ordering can't be `Release` or `AcqRel` and must be 
    /// equivalent or weaker than the success ordering.
    fn failure(&self) -> Ordering;
}

impl CompareAndSetOrdering for Ordering {
    #[inline]
    fn success(&self) -> Ordering {
        *self
    }

    #[inline]
    fn failure(&self) -> Ordering {
        strongest_failure_ordering(*self)
    }
}
```

在 `strongest_failure_ordering` 是：

```rs
fn strongest_failure_ordering(ord: Ordering) -> Ordering {
    use self::Ordering::*;
    match ord {
        Relaxed | Release => Relaxed,
        Acquire | AcqRel => Acquire,
        _ => SeqCst,
    }
}
```

最后一个新特质是 `Pointer`，这是一个提供对 `Owned` 和 `Shared` 上的函数的小工具特质：

```rs
pub trait Pointer<T> {
    /// Returns the machine representation of the pointer.
    fn into_data(self) -> usize;

    /// Returns a new pointer pointing to the tagged pointer `data`.
    unsafe fn from_data(data: usize) -> Self;
}
```

# crossbeam_epoch::Guard::defer

现在，再次，以下在 `TreiberStack::try_pop` 中的操作是如何安全进行的？

```rs
guard.defer(move || head_shared.into_owned());
return Some(ptr::read(&(*head).data));
```

我们现在需要探索的是 `defer`，它在 `crossbeam-epoch` 的 `src/guard.rs` 中：

```rs
    pub unsafe fn defer<F, R>(&self, f: F)
    where
        F: FnOnce() -> R,
    {
        let garbage = Garbage::new(|| drop(f()));

        if let Some(local) = self.local.as_ref() {
            local.defer(garbage, self);
        }
    }
```

我们需要理解 `Garbage`。它在 `src/garbage.rs` 中定义为：

```rs
pub struct Garbage {
    func: Deferred,
}

unsafe impl Sync for Garbage {}
unsafe impl Send for Garbage {}
```

`Garbage` 的实现非常简短：

```rs
impl Garbage {
    /// Make a closure that will later be called.
    pub fn new<F: FnOnce()>(f: F) -> Self {
        Garbage { func: Deferred::new(move || f()) }
    }
}

impl Drop for Garbage {
    fn drop(&mut self) {
        self.func.call();
    }
}
```

`Deferred` 是一个小的结构，它包装了 `FnOnce()`，根据大小允许将其存储在行内或堆上。鼓励你自己检查实现，但基本思想是在堆分配的结构中保持 `FnOnce` 的相同属性，该结构在 `Garbage` 丢弃实现中找到用途。什么丢弃 `Garbage`？这正是以下内容发挥作用的地方：

```rs
        if let Some(local) = self.local.as_ref() {
            local.defer(garbage, self);
        }
    }
```

`Guard` 的 `self.local` 是 `*const Local`，这是一个在 crossbeam 文档中被称为垃圾回收参与者的结构。我们需要了解这个 `Local` 是在哪里以及如何被创建的。首先让我们来理解一下 `Guard` 的内部结构：

```rs
pub struct Guard {
    pub(crate) local: *const Local,
}
```

`Guard` 是一个指向 `Local` 的常量指针的包装器。我们知道 `Guard` 的实例是由 `crossbeam_epoch::pin` 创建的，该函数定义在 `src/default.rs` 中：

```rs
pub fn pin() -> Guard {
    HANDLE.with(|handle| handle.pin())
}
```

`HANDLE` 是一个线程局部静态变量：

```rs
thread_local! {
    /// The per-thread participant for the default garbage collector.
    static HANDLE: Handle = COLLECTOR.register();
}
```

`COLLECTOR` 是一个静态的全局变量：

```rs
lazy_static! {
    /// The global data for the default garbage collector.
    static ref COLLECTOR: Collector = Collector::new();
}
```

这一次同时出现了很多新事物。固定是一个非平凡的活动！正如其名称所暗示的以及其文档所述，`COLLECTOR` 是 crossbeam-epoch 的全局垃圾收集器的入口点，称为 `Global`。`Collector` 定义在 `src/collector.rs` 中，并且有一个非常简短的实现：

```rs
pub struct Collector {
    pub(crate) global: Arc<Global>,
}

unsafe impl Send for Collector {}
unsafe impl Sync for Collector {}

impl Collector {
    /// Creates a new collector.
    pub fn new() -> Self {
        Collector { global: Arc::new(Global::new()) }
    }

    /// Registers a new handle for the collector.
    pub fn register(&self) -> Handle {
        Local::register(self)
    }
}
```

我们知道 `Collector` 是一个全局静态变量，这意味着 `Global::new()` 只会调用一次。`Global` 定义在 `src/internal.rs` 中，是全局垃圾收集器的数据存储库：

```rs
pub struct Global {
    /// The intrusive linked list of `Local`s.
    locals: List<Local>,

    /// The global queue of bags of deferred functions.
    queue: Queue<(Epoch, Bag)>,

    /// The global epoch.
    pub(crate) epoch: CachePadded<AtomicEpoch>,
}
```

列表是一个侵入式列表，我们在 `Local` 的定义中看到了对其的引用。侵入式列表是一个链表，其中列表中下一个节点的指针存储在数据本身中，即 `Local` 的 `entry: Entry` 字段。侵入式列表并不常见，但当你关心小内存分配或需要在多个集合中存储元素时，它们非常有用，这两者都适用于 crossbeam。`Queue` 是 Michael 和 Scott 队列。时期是一个缓存填充的 `AtomicEpoch`，缓存填充是一种禁用缓存行写冲突的技术，称为伪共享。`AtomicEpoch` 是 `AtomicUsize` 的包装器。那么，`Global` 是一个 `Local` 实例的链表——这些实例本身与线程固定的 `Guards` 关联——一个 `Bag` 的队列，我们尚未研究，与某些时期号（`usize`）和一个原子的全局 `Epoch` 关联。这种布局与算法描述所建议的非常相似。一旦唯一的 `Global` 被初始化，每个线程局部 `Handle` 都是通过调用 `Collector::register` 创建的，该调用在内部是 `Local::register`：

```rs
    pub fn register(collector: &Collector) -> Handle {
        unsafe {
            // Since we dereference no pointers in this block, it is 
            // safe to use `unprotected`.

            let local = Owned::new(Local {
                entry: Entry::default(),
                epoch: AtomicEpoch::new(Epoch::starting()),
                collector: UnsafeCell::new(
                    ManuallyDrop::new(collector.clone())
                ),
                bag: UnsafeCell::new(Bag::new()),
                guard_count: Cell::new(0),
                handle_count: Cell::new(1),
                pin_count: Cell::new(Wrapping(0)),
            }).into_shared(&unprotected());
            collector.global.locals.insert(local, &unprotected());
            Handle { local: local.as_raw() }
        }
    }
```

注意，特别是 `collector.global.locals.insert(local, &unprotected())` 这个调用是将新创建的 `Local` 插入到 `Global` 局部列表中。（`unprotected` 是一个指向空指针的 `Guard`，而不是指向某个有效 `Local`）。每个固定线程都会在全局垃圾收集器的数据中注册一个 `Local`。实际上，在我们完成 `defer` 之前，让我们看看当 `Guard` 被丢弃时会发生什么：

```rs
impl Drop for Guard {
    #[inline]
    fn drop(&mut self) {
        if let Some(local) = unsafe { self.local.as_ref() } {
            local.unpin();
        }
    }
}
```

调用 `Local` 的 `unpin` 方法：

```rs
    pub fn unpin(&self) {
        let guard_count = self.guard_count.get();
        self.guard_count.set(guard_count - 1);

        if guard_count == 1 {
            self.epoch.store(Epoch::starting(), Ordering::Release);

            if self.handle_count.get() == 0 {
                self.finalize();
            }
        }
    }
```

回想一下，`guard_count`字段是参与线程的总数或为保持线程锁定而安排的守护者的总数。`handle_count`字段是一个类似的机制，但由`Collector`和`Local`使用。`Local::finalize`是动作发生的地方：

```rs
    fn finalize(&self) {
        debug_assert_eq!(self.guard_count.get(), 0);
        debug_assert_eq!(self.handle_count.get(), 0);

        // Temporarily increment handle count. This is required so that  
        // the following call to `pin` doesn't call `finalize` again.
        self.handle_count.set(1);
        unsafe {
            // Pin and move the local bag into the global queue. It's 
            // important that `push_bag` doesn't defer destruction 
            // on any new garbage.
            let guard = &self.pin();
            self.global().push_bag(&mut *self.bag.get(), guard);
        }
        // Revert the handle count back to zero.
        self.handle_count.set(0);
```

`Local`包含一个`self.bag`字段，其类型为`UnsafeCell<Bag>`。以下是`Bag`，在`src/garbage.rs`中定义：

```rs
pub struct Bag {
    /// Stashed objects.
    objects: ArrayVec<[Garbage; MAX_OBJECTS]>,
}
```

`ArrayVec`是这本书的新内容。它在`ArrayVec`crate 中定义，是一个`Vec`，但具有最大容量限制。它是我们熟悉和喜爱的可增长向量，但不能无限分配。然后，`Bag`是一个可增长向量，包含`Garbage`，总大小限制为`MAX_OBJECTS`：

```rs
#[cfg(not(feature = "strict_gc"))]
const MAX_OBJECTS: usize = 64;
#[cfg(feature = "strict_gc")]
const MAX_OBJECTS: usize = 4;
```

尝试将垃圾推入超过`MAX_OBJECTS`的袋子将失败，向调用者发出信号，表明是时候收集一些垃圾了。`Local::finalize`具体做了什么，特别是与`self.global().push_bag(&mut *self.bag.get(), guard)`相关，是将`Local`的垃圾袋推入`Global`的垃圾袋，作为关闭`Local`的一部分。`Global::push_bag`是：

```rs
    pub fn push_bag(&self, bag: &mut Bag, guard: &Guard) {
        let bag = mem::replace(bag, Bag::new());

        atomic::fence(Ordering::SeqCst);

        let epoch = self.epoch.load(Ordering::Relaxed);
        self.queue.push((epoch, bag), guard);
    }
```

解除`Guard`的锁定可能会关闭一个`Local`，将其垃圾推入`Global`中标记为纪元的垃圾队列。现在我们明白了，让我们通过检查`Local::defer`来完成`Guard::defer`：

```rs
    pub fn defer(&self, mut garbage: Garbage, guard: &Guard) {
        let bag = unsafe { &mut *self.bag.get() };

        while let Err(g) = bag.try_push(garbage) {
            self.global().push_bag(bag, guard);
            garbage = g;
        }
    }
```

锁定的调用者通过调用`defer`将垃圾信号传递给其`Guard`。然后，`Guard`将此垃圾延迟到其`Local`，后者进入一个 while 循环，尝试将垃圾推入本地垃圾袋，但只要本地垃圾袋满了，就会将垃圾移入`Global`。

垃圾何时从`Global`中回收？答案是存在于我们尚未检查的函数中，即`Local::pin`。

# crossbeam_epoch::Local::pin

回想一下，`crossbeam_epoch::pin`调用`Handle::pin`，后者又调用其`Local`上的`pin`。在`Local::pin`执行期间会发生什么？很多：

```rs
    pub fn pin(&self) -> Guard {
        let guard = Guard { local: self };

        let guard_count = self.guard_count.get();
        self.guard_count.set(guard_count.checked_add(1).unwrap());
```

`Local`的守护者计数增加，这是通过创建一个带有`self`的`Guard`来完成的。如果守护者计数之前为零：

```rs
        if guard_count == 0 {
            let global_epoch = 
         self.global().epoch.load(Ordering::Relaxed);
            let new_epoch = global_epoch.pinned();
```

这意味着在`Local`中没有其他活跃的参与者，并且`Local`需要查询全局纪元。全局纪元被加载为`Local`纪元：

```rs
              self.epoch.store(new_epoch, Ordering::Relaxed);
              atomic::fence(Ordering::SeqCst);
```

这里使用顺序一致性来确保未来和过去原子操作的一致性。最后，`Local`的`pin_count`增加，这可能会启动`Global::collect`：

```rs
            let count = self.pin_count.get();
            self.pin_count.set(count + Wrapping(1));

            // After every `PINNINGS_BETWEEN_COLLECT` try advancing the 
            // epoch and collecting some garbage.
            if count.0 % Self::PINNINGS_BETWEEN_COLLECT == 0 {
                self.global().collect(&guard);
            }
        }

        guard
    }
```

`Local`可以反复锁定和解锁，这时`pin_count`就派上用场。这种机制在我们的讨论中并未使用，但读者可以参考`Guard::repin`和`Guard::repin_after`。后者函数在混合网络调用和原子数据结构时特别有用，因为正如你回忆的那样，垃圾回收只能在纪元推进后进行，而纪元的推进通常是通过解除锁定来实现的。`Global::collect`非常简洁：

```rs
    pub fn collect(&self, guard: &Guard) {
        let global_epoch = self.try_advance(guard);
```

全球时代可能通过`Global::try_advance`向前推进，这种推进只有在全局`Local`列表中的每个`Local`都不在另一个时代，或者列表在检查期间没有被修改时才会发生。并发修改的检测是一个特别巧妙的技巧，它是 crossbeam-epoch 的私有`List`迭代的一部分。标记指针在这个迭代中扮演了一定的角色，强烈建议读者阅读并理解 crossbeam-epoch 使用的`List`实现：

```rs
        let condition = |item: &(Epoch, Bag)| {
            // A pinned participant can witness at most one epoch 
            advancement. Therefore, any bag
            // that is within one epoch of the current one cannot be 
            destroyed yet.
            global_epoch.wrapping_sub(item.0) >= 2
        };

        let steps = if cfg!(feature = "sanitize") {
            usize::max_value()
        } else {
            Self::COLLECT_STEPS
        };

        for _ in 0..steps {
            match self.queue.try_pop_if(&condition, guard) {
                None => break,
                Some(bag) => drop(bag),
            }
        }
    }
```

条件将被传递到 Global 的`Queue::try_pop_if`，它只会从队列中弹出与条件匹配的元素。我们在这里再次看到了算法的作用。回想一下`self.queue :: Queue<(Epoch, Bag)>`？垃圾袋只有在距离当前时代超过两个时代时才会从队列中取出，否则可能仍然存在对它们的活跃和危险的引用。这些步骤控制着最终将被收集的垃圾量，在时间与内存使用之间进行权衡。回想一下，每个新固定的线程都在参与这个过程，释放一些全局垃圾。

总结一下，当程序员调用线程固定时，这会为存储线程局部垃圾创建一个`Local`，并将其链接到`Global`上下文中所有`Local`的列表中。线程将尝试收集`Global`垃圾，在这个过程中可能推动时代向前。程序员在执行期间延迟的任何垃圾都会优先推入`Local`垃圾袋，如果这个袋满了，就会导致一些垃圾转移到`Global`袋中。当`Local`被取消固定时，`Local`袋中的任何垃圾都会转移到`Global`袋中。由于每个原子操作都需要对`Guard`的引用，因此每个原子操作隐式地与某个`Local`、某个时代相关联，并且可以安全地通过前面算法中概述的方法回收。

哎呀！

# 练习基于时代的 Treiber 栈

让我们将 crossbeam-epoch Treiber 栈进行彻底测试，以了解这种方法的表现。对我们来说，关键的兴趣领域将是：

+   每秒的推/弹周期

+   内存行为、水位线以及诸如此类的事情

+   缓存行为

我们将在 x86 和 ARM 上运行我们的程序，就像我们在前面的章节中所做的那样。我们的练习程序与上一节中的危险指针程序类似：

```rs
extern crate crossbeam;
#[macro_use]
extern crate lazy_static;
extern crate num_cpus;
extern crate quantiles;

use crossbeam::sync::TreiberStack;
use quantiles::ckms::CKMS;
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::{thread, time};

lazy_static! {
    static ref WORKERS: AtomicUsize = AtomicUsize::new(0);
    static ref COUNT: AtomicUsize = AtomicUsize::new(0);
}
static MAX_I: u32 = 67_108_864; // 2 ** 26

fn main() {
    let stk: Arc<TreiberStack<(u64, u64, u64)>> = Arc::new(TreiberStack::new());

    let mut jhs = Vec::new();

    let cpus = num_cpus::get();
    WORKERS.store(cpus, Ordering::Release);

    for _ in 0..cpus {
        let stk = Arc::clone(&stk);
        jhs.push(thread::spawn(move || {
            for i in 0..MAX_I {
                stk.push((i as u64, i as u64, i as u64));
                stk.pop();
                COUNT.fetch_add(1, Ordering::Relaxed);
            }
            WORKERS.fetch_sub(1, Ordering::Relaxed)
        }))
    }

    let one_second = time::Duration::from_millis(1_000);
    let mut iter = 0;
    let mut cycles: CKMS<u32> = CKMS::new(0.001);
    while WORKERS.load(Ordering::Relaxed) != 0 {
        let count = COUNT.swap(0, Ordering::Relaxed);
        cycles.insert((count / cpus) as u32);
        println!(
            "CYCLES PER SECOND({}):\n  25th: \
             {}\n  50th: {}\n  75th: \
             {}\n  90th: {}\n  max:  {}\n",
            iter,
            cycles.query(0.25).unwrap().1,
            cycles.query(0.50).unwrap().1,
            cycles.query(0.75).unwrap().1,
            cycles.query(0.90).unwrap().1,
            cycles.query(1.0).unwrap().1
        );
        thread::sleep(one_second);
        iter += 1;
    }

    for jh in jhs {
        jh.join().unwrap();
    }
}
```

我们有与目标机器中 CPU 总数相等的多个工作线程，每个线程执行一次推入然后立即弹出栈的操作。每秒左右，主线程将`COUNT`的值交换为 0，并将该值投入到一个分位数估计结构中——在第四章中更详细地讨论了这一点，*同步与发送——Rust 并发的基石*——并打印出每秒记录的周期摘要，按 CPU 数量缩放。工作线程循环到`MAX_I`，这个值任意设置为`2**26`的小值。当工作线程完成循环后，它会减少`WORKERS`的值并退出。一旦`WORKERS`达到零，主循环也会退出。

在我的 x86 机器上，这个程序退出大约需要 38 秒，输出如下：

```rs
CYCLES PER SECOND(0):
  25th: 0
  50th: 0
  75th: 0
  90th: 0
  max:  0

CYCLES PER SECOND(1):
  25th: 0
  50th: 0
  75th: 1739270
  90th: 1739270
  max:  1739270

...

CYCLES PER SECOND(37):
  25th: 1738976
  50th: 1739528
  75th: 1740474
  90th: 1757650
  max:  1759459

CYCLES PER SECOND(38):
  25th: 1738868
  50th: 1739528
  75th: 1740452
  90th: 1757650
  max:  1759459
```

与 x86 危险实现相比，它总共需要 58 秒。x86 perf 运行结果如下：

```rs
> perf stat --event task-clock,context-switches,page-faults,cycles,instructions,branches,branch-misses,cache-references,cache-misses target/release/epoch_stack > /dev/null

 Performance counter stats for 'target/release/epoch_stack':

     148830.337380      task-clock (msec)         #    3.916 CPUs utilized
             1,043      context-switches          #    0.007 K/sec
             1,039      page-faults               #    0.007 K/sec
   429,969,505,981      cycles                    #    2.889 GHz
   161,901,052,886      instructions              #    0.38  insn per cycle
    27,531,697,676      branches                  #  184.987 M/sec
       627,050,474      branch-misses             #    2.28% branches
    11,885,394,803      cache-references          #   79.859 M/sec
         1,772,308      cache-misses              #    0.015 % cache refs

      38.004548310 seconds time elapsed
```

在基于纪元的处理方法中，执行指令的总数要低得多，这与本节开头讨论中的分析相吻合。即使只有一个危险指针，基于纪元的处理方法所做的工也要少得多。在我的 ARM 机器上，程序运行到完成大约需要 72 秒，输出如下：

```rs
CYCLES PER SECOND(0):
  25th: 0
  50th: 0
  75th: 0
  90th: 0
  max:  0

CYCLES PER SECOND(1):
  25th: 0
  50th: 0
  75th: 921743
  90th: 921743
  max:  921743

...

CYCLES PER SECOND(71):
  25th: 921908
  50th: 922326
  75th: 922737
  90th: 923235
  max:  924084

CYCLES PER SECOND(72):
  25th: 921908
  50th: 922333
  75th: 922751
  90th: 923235
  max:  924084
```

与 ARM 危险实现相比，它总共需要 463 秒！ARM perf 运行结果如下：

```rs
> perf stat --event task-clock,context-switches,page-faults,cycles,instructions,branches,branch-misses,cache-references,cache-misses target/release/epoch_stack > /dev/null

 Performance counter stats for 'target/release/epoch_stack':

     304880.898057      task-clock (msec)         #    3.959 CPUs utilized
                 0      context-switches          #    0.000 K/sec
               248      page-faults               #    0.001 K/sec
   364,139,169,537      cycles                    #    1.194 GHz
   215,754,991,724      instructions              #    0.59  insn per cycle
    28,019,208,815      branches                  #   91.902 M/sec
     3,525,977,068      branch-misses             #   12.58% branches
   105,165,886,677      cache-references          #  344.941 M/sec
     1,450,538,471      cache-misses              #    1.379 % cache refs

      77.014340901 seconds time elapsed
```

与同一处理器架构上的危险指针实现相比，执行指令的数量大大减少。顺便提一下，值得注意的一点是，x86 和 ARM 版本的`epoch_stack`内存使用量都低于危险指针栈实现，尽管它们都不是内存消耗大户，每个只消耗了几千字节。如果我们的纪元线程在离开其纪元之前长时间休眠——比如说一秒钟左右——那么在执行过程中内存使用量会增加。鼓励读者在自己的系统上造成混乱。

# 权衡

在本章概述的三种方法中，crossbeam-epoch 在测量和文献讨论中都是最快的。更快的技巧，基于静默状态的内存回收，在本章中没有讨论，因为它不适合用户空间程序，而且在 Rust 中没有现成的实现。此外，库的作者已经投入了多年的工作来实现它：它有很好的文档记录，并在多个 CPU 架构上的现代 Rust 中运行。

该方法的传统问题——即线程引入到固定关键部分的长时间延迟，导致长期存在的时代和垃圾积累——在 API 中通过`Guard`能够任意重新固定的能力得到解决。将标准库的`AtomicPtr`数据结构过渡到 crossbeam 是一个项目，但是一个可接近的项目。正如我们所见，crossbeam-epoch 确实引入了一些开销，包括缓存填充和线程同步。这种开销是最小的，并且预计会随着时间的推移而改善。

# 摘要

在本章中，我们讨论了三种内存回收技术：引用计数、危险指针和基于时代的回收。每种方法都比前一种更快，尽管每种方法都有其权衡。引用计数产生的开销最大，必须谨慎地将其集成到你的数据结构中，除非 Arc 符合你的需求，这完全有可能。危险指针需要识别危险，即导致无法回收内存的内存访问，这种访问需要某种类型的协调。这种方法在每次访问危险指针时都会产生开销，如果必须遍历危险结构，这将是昂贵的。最后，基于时代的回收在线程固定时产生开销，这表示时代的开始，可能需要新固定的线程参与垃圾回收。在固定之后，内存访问不会产生额外的开销，如果你正在执行遍历或以其他方式在固定部分包含许多内存操作，这将是一个巨大的胜利。

crossbeam 库做得非常好。除非你有专门设计为不需要分配或在没有垃圾回收的情况下处理在宽松内存系统上的分配的原子数据结构，否则强烈建议你在 Rust 中进行原子编程时将 crossbeam 视为不可或缺的一部分，至少在撰写本文时是这样的。

在下一章中，我们将离开原子编程的领域，讨论并行编程的高级方法，使用线程池和数据并行迭代器。

# 进一步阅读

+   *危险指针：无锁对象的内存回收安全方法*，Maged Michael。本文介绍了本章讨论的危险指针回收技术。该论文特别讨论了与引用计数相比的新发明技术，并展示了使用危险指针技术构建安全的 Michael 和 Scott 队列。

+   *实用的无锁编程*，Keir Fraser。这是 Keir Fraser 的博士论文，相当长，涉及引入抽象以简化无锁结构的编写——其中之一是基于时代的回收——以及引入无锁搜索结构、跳表、二叉搜索树和红黑树。强烈推荐。

+   *Performance of Memory Reclamation for Lockless Synchronization*，托马斯·哈特等人。这篇论文概述了四种技术，这些技术都在本书的第六章 Atomics – the Primitives of Synchronization 中有所提及，其中三种在本章中进行了深入讨论。此外，论文介绍了一种针对基于纪元的回收的优化，如果作者没有弄错的话，这已经影响了 crossbeam。这篇论文是算法的绝佳概述，并以定义良好的方式提供了算法性能的比较测量。

+   *RFCs for changes to Crossbeam*，可在[`github.com/crossbeam-rs/rfcs`](https://github.com/crossbeam-rs/rfcs)找到。这个 GitHub 仓库是关于对 crossbeam 库进行大规模更改的主要讨论点，其讨论内容特别值得阅读。例如，RFC 2017-07-23-relaxed-memory 概述了 crossbeam 在松散内存系统上运行所需的变化，这是一个在文献中很少讨论的话题。

+   *CDSCHECKER: Checking Concurrent Data Structures Written with C/C++ Atomics*，布莱恩·诺里斯和布莱恩·德姆斯基。这篇论文介绍了一个工具，用于根据 C++11/LLVM 内存模型检查并发数据结构的行为。鉴于松散内存排序可能有多么令人困惑，这个工具在进行 C++工作时非常有用。我不知道有 Rust 的类似工具，但希望这会被视为一个机会，让一些有才华的人阅读。祝你好运，你。

+   *A Promising Semantics for Relaxed-Memory Concurrency*，金浩恩·康等人。对 C++/LLVM 中提出的内存模型进行推理非常困难。尽管多年来许多人已经深入思考过这个问题，但仍然有活跃的研究致力于将此模型形式化以验证优化器和算法的正确功能。请与诺里斯和德姆斯基的《CDSCHECKER》一起阅读这篇论文。
