# Sync 和 Send – Rust 并发的基石

Rust 旨在成为一个可以无畏地进行并发的编程语言。这意味着什么？它是如何工作的？在第二章，*顺序 Rust 性能测试*中，我们讨论了顺序 Rust 程序的性能，故意省略了对并发程序的讨论。在第三章，*Rust 内存模型 – 所有权、引用和操作*中，我们看到了 Rust 处理内存的概述，特别是关于构建高性能结构的方式。在本章中，我们将扩展我们之前学到的内容，并最终深入探讨 Rust 的并发故事。

到本章结束时，我们将：

+   讨论了`Sync`和`Send`特性

+   使用 Helgrind 检查了环形数据结构中的并行竞态

+   使用互斥锁解决了这个竞态条件

+   调查了标准库 MPSC 的使用

+   构建了一个非平凡的数据复用项目

# 技术要求

本章需要安装一个有效的 Rust 环境。验证安装的详细说明在第一章，*预备知识 – 计算机架构和 Rust 入门*中有所介绍。下面使用 Valgrind 工具套件。许多操作系统捆绑了 valgrind 包，但您可以在[`valgrind.org/`](http://valgrind.org/)找到您系统的进一步安装说明。Linux Perf 被使用，并且被许多 Linux 发行版捆绑。本章所需的其他任何软件都将作为文本的一部分安装。

您可以在 GitHub 上找到本书项目的源代码：[`github.com/PacktPublishing/Hands-On-Concurrency-with-Rust/`](https://github.com/PacktPublishing/Hands-On-Concurrency-with-Rust/). 本章的源代码位于`Chapter04`目录下。

# Sync 和 Send

并行 Rust 程序员必须理解两个关键特性——`Send`和`Sync`。`Send`特性应用于可以在线程边界之间传输的类型。也就是说，任何类型`T: Send`都可以安全地从一条线程移动到另一条线程。`Rc<T>: !Send`表示它被明确标记为在线程之间移动是不安全的。为什么？考虑如果它被标记为`Send`，会发生什么？我们从上一章知道`Rc<T>`是一个带有两个计数器的 boxed `T`，分别用于对`T`的弱引用和强引用。这些计数器是关键。假设我们将一个`Rc<T>`分散在两个线程——称为`A`和`B`——每个线程都创建了引用、丢弃了它们，等等。如果`A`和`B`同时丢弃了`Rc<T>`的最后一个引用，会发生什么？我们有两个线程之间的竞态，看哪个线程可以首先释放`T`，哪个线程会因此遭受双重释放的麻烦。更糟糕的是，假设在`Rc<T>`中的强引用计数器的获取被分散在三个伪指令中：

```rs
LOAD counter            (tmp)
ADD counter 1 INTO tmp  (tmp = tmp+1)
STORE tmp INTO counter  (counter = tmp)
```

同样地，假设一个强引用计数器的释放被分散到三个伪指令中：

```rs
LOAD counter            (tmp)
SUB counter 1 INTO tmp  (tmp = tmp-1)
STORE tmp INTO counter  (counter = tmp)
```

在单线程的上下文中，这工作得很好，但在多线程的上下文中考虑这个结果。在以下思维实验的开始，让所有线程的`counter`都等于 10：

```rs
[A]LOAD counter                        (tmp_a == 10)
[B]LOAD counter                        (tmp_b == 10)
[B]SUB counter 1 INTO tmp              (tmp_b = tmp_b-1)
[A]ADD counter 1 INTO tmp              (tmp_a = tmp_a + 1)
[A]STORE tmp INTO counter              (counter = tmp_a)    == 11
[A]ADD counter 1 INTO tmp              (tmp_a = tmp_a + 1)
[A]STORE tmp INTO counter              (counter = tmp_a)    == 12
[A]ADD counter 1 INTO tmp              (tmp_a = tmp_a + 1)
[A]STORE tmp INTO counter              (counter = tmp_a)    == 13
[B]STORE tmp INTO counter              (counter == tmp_b)   == 9
```

最后，我们失去了对`Rc<T>`的三个引用，这意味着虽然`T`在内存中没有丢失，但在我们删除`T`时，完全有可能在外部仍然有效但不再有效的内存中保留对其的引用，其结果是不确定的，但不太可能是好的。

`Sync`特质是从`Send`派生出来的，与引用有关：如果`&T: Send`，则`T: Sync`。也就是说，只有当共享一个`&T`，它表现得好像那个`T`被发送到线程中时，`T`才是`Sync`。从前面的代码中我们知道`Rc<T>: !Send`，因此我们也知道`Rc<T>: !Sync`。Rust 类型继承其组成部分的`Sync`和`Send`状态。按照惯例，任何`Sync` + `Send`的类型都被称为线程安全的。我们基于`Rc<T>`实现的任何类型都不会是线程安全的。现在，对于大多数情况，`Sync`和`Send`是自动派生的特质。在第三章中讨论的`UnsafeCell`不是线程安全的。同样，原始指针也不是，因为它们缺乏其他安全保证。当你探索 Rust 代码库时，你会发现一些特质在其他情况下会被自动派生为线程安全的，但被标记为不是。

# 竞态线程

你也可以将类型标记为线程安全的，这实际上是在向编译器承诺你已经以使它们安全的方式安排了所有数据竞争。这在实践中相对较少见，但在我们开始构建在安全原语之上之前，了解在 Rust 中使类型线程安全需要什么是有价值的。首先，让我们看看故意写错的代码：

```rs
use std::{mem, thread};
use std::ops::{Deref, DerefMut};

unsafe impl Send for Ring {}
unsafe impl Sync for Ring {}

struct InnerRing {
    capacity: isize,
    size: usize,
    data: *mut Option<u32>,
}

#[derive(Clone)]
struct Ring {
    inner: *mut InnerRing,
}
```

我们这里有一个环形，或者说是一个循环缓冲区，由`u32`组成。`InnerRing`包含一个原始可变指针，因此不是自动线程安全的。但是，我们已经向 Rust 承诺我们知道我们在做什么，并为`Ring`实现了`Send`和`Sync`。为什么不在`InnerRing`上呢？当我们从多个线程中操作内存中的对象时，该对象的位置必须固定。`InnerRing`——以及它包含的数据——必须在内存中占据一个稳定的位置。`Ring`可以并且将会在至少从创建线程到工作线程之间跳动。现在，`InnerRing`中的数据是什么？它是一个指向连续内存块 0 偏移量的指针，这个内存块将作为我们的循环缓冲区的存储。在撰写本书时，Rust 没有稳定的分配器接口，因此，为了获得连续的分配，我们必须以迂回的方式去做——将`Vec<u32>`简化为其指针：

```rs
impl Ring {
    fn with_capacity(capacity: usize) -> Ring {
        let mut data: Vec<Option<u32>> = Vec::with_capacity(capacity);
        for _ in 0..capacity {
            data.push(None);
        }
        let raw_data = (&mut data).as_mut_ptr();
        mem::forget(data);
        let inner_ring = Box::new(InnerRing {
            capacity: capacity as isize,
            size: 0,
            data: raw_data,
        });

        Ring {
            inner: Box::into_raw(inner_ring),
        }
    }
}
```

`Ring::with_capacity`函数与 Rust 生态系统中的其他类型的`with_capacity`函数非常相似：分配足够的空间以容纳容量项。在我们的情况下，我们利用`Vec::with_capacity`，确保为容量`Option<u32>`实例分配足够的房间，并在整个内存块长度上初始化为 None。如果你还记得第三章，*《Rust 内存模型 – 所有权、引用和操作*，这是作为`Vec`在分配上比较懒惰，而我们则需要分配。`Vec::as_mut_ptr`返回一个指向切片的原始指针，但不消耗原始对象，这对`Ring`来说是个问题。当数据超出作用域时，分配的块必须保持存活。标准库的`mem::forget`非常适合这种用途。现在分配是安全的，一个`InnerRing`被装箱以存储它。然后，这个箱子被`Box::into_raw`消耗，并传递给一个`Ring`。哇！

与具有内部原始指针的类型交互可能会很冗长，在周围散布不安全块，效果甚微。为此，`Ring`得到了`Deref`和`DerefMut`的实现，这两个实现都整理了与`Ring`的交互：

```rs
impl Deref for Ring {
    type Target = InnerRing;

    fn deref(&self) -> &InnerRing {
        unsafe { &*self.inner }
    }
}

impl DerefMut for Ring {
    fn deref_mut(&mut self) -> &mut InnerRing {
        unsafe { &mut *self.inner }
    }
}
```

现在我们已经定义了`Ring`，我们可以进入程序的实质部分。我们将定义两个将同时运行的运算——写入者和读取者。这里的想法是，写入者将在环中竞争，只要有机会就会写入，将`u32`值增加到环中。（在类型边界处，`u32`将回绕。）读取者将在写入者后面竞争，读取写入的值，检查每个读取的值是否比前一个读取的值大一个，但要注意回绕。以下是写入者：

```rs
fn writer(mut ring: Ring) -> () {
    let mut offset: isize = 0;
    let mut cur: u32 = 0;
    loop {
        unsafe {
            if (*ring).size != ((*ring).capacity as usize) {
                *(*ring).data.offset(offset) = Some(cur);
                (*ring).size += 1;
                cur = cur.wrapping_add(1);
                offset += 1;
                offset %= (*ring).capacity;
            } else {
                thread::yield_now();
            }
        }
    }
}
```

现在，为了非常清楚，这里有很多地方是错误的。目标是只在环缓冲区的大小未达到其容量时写入——这意味着有可用空间。实际的写入是：

```rs
                *(*ring).data.offset(offset) = Some(cur);
                (*ring).size += 1;
```

也就是说，我们取消引用`ring (*ring)`并得到指向`Option<u32>`大小块的指针，该块位于`(*ring).data.offset(offset)`，然后我们再次取消引用并将`Some(cur)`移动到之前的内容顶部。完全有可能因为`Ring`大小的竞争，我们会覆盖一个未读取的`u32`。写入块的其余部分设置我们的下一个`cur`和下一个偏移量，如果需要则加一并进行模运算：

```rs
            } else {
                thread::yield_now();
            }
```

`thread::yield_now`是新的。写入者是一个快速的旋转循环——它检查一个条件然后再次循环以进行另一次尝试。这非常不高效，对 CPU 和电力来说都是浪费。`thread::yield_now`向操作系统暗示这个线程没有工作要做，应该优先考虑其他线程。效果取决于操作系统和运行环境，但如果你必须旋转循环，那么让步仍然是一个好主意：

```rs
fn reader(mut ring: Ring) -> () {
    let mut offset: isize = 0;
    let mut cur: u32 = 0;
    while cur < 1_000 {
        unsafe {
            if let Some(num) = mem::replace(
                &mut *(*ring).data.offset(offset), 
                None) 
            {
                assert_eq!(num, cur);
                (*ring).size -= 1;
                cur = cur.wrapping_add(1);
                offset += 1;
                offset %= (*ring).capacity;
            } else {
                thread::yield_now();
            }
        }
    }
}
```

读取器与写入器类似，主要区别在于它不是一个无限循环。读取操作使用`mem::replace`完成，将读取偏移量处的块与`None`交换。当我们得到“bingo”并得到一个`Some`时，那个`u32`的内存现在归读取器所有——当它超出作用域时将调用`drop`。这是很重要的。写入器负责在原始指针内部丢失内存，而读取器负责找到它。通过这种方式，我们小心地避免内存泄漏。或者，好吧，我们会这样做，除非在`Ring`的大小上没有竞争。

最后，我们有`main`函数：

```rs
fn main() {
    let capacity = 10;
    let ring = Ring::with_capacity(capacity);

    let reader_ring = ring.clone();
    let reader_jh = thread::spawn(move || {
        reader(reader_ring);
    });
    let _writer_jh = thread::spawn(move || {
        writer(ring);
    });

    reader_jh.join().unwrap();
}
```

这里有两个新的情况。第一个是我们使用`thread::spawn`来启动一个`reader`和一个`writer`。`move || {}`结构被称为*移动闭包*。也就是说，闭包内部对外部作用域中每个变量的引用都被移动到闭包的作用域中。这就是为什么我们需要将环形克隆到`reader_ring`中。否则，就没有`ring`供写入器使用。第二个新情况是`thread::spawn`返回的`JoinHandle`。Rust 线程并没有与常见的 POSIX 或 Windows 线程有太大的不同。Rust 线程有自己的堆栈，并且可以由操作系统独立运行。

每个 Rust 线程都有一个返回值，尽管这里的返回值是`()`。我们通过在线程的`JoinHandler`上*连接*来获取这个返回值，暂停我们的线程的执行，直到线程成功完成或崩溃。我们的主线程假设其子线程将成功返回，因此有`join().unwrap()`。

当我们运行我们的程序时会发生什么？嗯，失败，这正是我们预期的：

```rs
> rustc -C opt-level=3 data_race00.rs && ./data_race00
thread '<unnamed>' panicked at 'assertion failed: `(left == right)`
 left: `31`,
 right: `21`', data_race00.rs:90:17
 note: Run with `RUST_BACKTRACE=1` for a backtrace.
 thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: Any', libcore/result.rs:916:5
```

# 环形结构的缺陷

让我们来看看这里出了什么问题。我们的环形结构在内存中作为一个连续的块布局，并且我们在旁边挂了一些控制变量。以下是任何读取或写入发生之前系统的示意图：

```rs
size: 0
capacity: 5

 rdr
 |
|0: None |1: None |2: None |3: None |4: None |
 |
 wrt
```

以下是写入器写入第一个值后系统的示意图：

```rs
size: 0
capacity: 5

 rdr
 |
|0: Some(0) |1: None |2: None |3: None |4: None |
             |
             wrt
```

要做到这一点，作者进行了一系列关于大小和容量的负载测试，并在它们之间进行了比较，将结果写入块内的偏移量，并增加了大小。这些操作并不保证按照程序中的顺序执行，要么是因为推测性执行，要么是因为编译器重排序。正如我们在之前的运行示例中看到的，作者在其自己的写入上进行了覆盖，并且进展迅速。这是如何做到的？考虑在以下设置中，当读取器和写入器的执行被交错时会发生什么：

```rs
size: 5
capacity: 5

 rdr
 |
|0: Some(5) |1: Some(6) |2: Some(7) |3: Some(8) |4: Some(9) |
 |
 wrt
```

写入线程已经绕过环形结构两次，并处于环形结构的起始位置准备写入`10`。读者已经通过环形结构一次，并期待看到`5`。考虑一下如果读者的减小大小的操作在`mem::replace`之前进入主内存会发生什么。想象一下，如果这时写入线程被唤醒，正在检查其大小和容量。想象一下，除此之外，写入线程在读者再次醒来之前将其新的`cur`写入主内存。你会得到以下情况：

```rs
size: 5
capacity: 5

 rdr
 |
|0: Some(10) |1: Some(6) |2: Some(7) |3: Some(8) |4: Some(9) |
              |
              wrt
```

这是我们实际看到的情况。有时。在机器金属层面进行并发编程的技巧是：你正在处理概率。我们的程序成功运行或，更糟糕的是，在一个 CPU 架构上成功运行但在另一个架构上不成功运行是完全可能的。对这类程序进行随机测试和内省是*至关重要的*。实际上，在第一章，*预备知识 – 机器架构和 Rust 入门*中，我们讨论了一个`valgrind`套件工具`helgrind`，但当时并没有真正用到它。现在我们用到了。`helgrind`对我们故意制造的竞态条件程序有什么看法？

我们将这样运行`helgrind`：

```rs
> valgrind --tool=helgrind --history-level=full --log-file="results.txt" ./data_race00
```

这将打印出程序的完整历史，但将结果放在磁盘上的文件中以便更容易检查。以下是部分输出：

```rs
==19795== ----------------------------------------------------------------
==19795==
==19795== Possible data race during write of size 4 at 0x621A050 by thread #3
==19795== Locks held: none
==19795==    at 0x10F8E2: data_race00::writer::h4524c5c44483b66e (in /home/blt/projects/us/troutwine/concurrency_in_rust/external_projects/data_races/data_race00)
==19795==    by 0x10FE75: _ZN3std10sys_common9backtrace28__rust_begin_short_backtrace17ha5ca0c09855dd05fE.llvm.5C463C64 (in /home/blt/projects/us/troutwine/concurrency_in_rust/external_projects/data_races/data_race00)
==19795==    by 0x12668E: __rust_maybe_catch_panic (lib.rs:102)
==19795==    by 0x110A42: _$LT$F$u20$as$u20$alloc..boxed..FnBox$LT$A$GT$$GT$::call_box::h6b5d5a2a83058684 (in /home/blt/projects/us/troutwine/concurrency_in_rust/external_projects/data_races/data_race00)
==19795==    by 0x1191A7: call_once<(),()> (boxed.rs:798)
==19795==    by 0x1191A7: std::sys_common::thread::start_thread::hdc3a308e21d56a9c (thread.rs:24)
==19795==    by 0x112408: std::sys::unix::thread::Thread::new::thread_start::h555aed63620dece9 (thread.rs:90)
==19795==    by 0x4C32D06: mythread_wrapper (hg_intercepts.c:389)
==19795==    by 0x5251493: start_thread (pthread_create.c:333)
==19795==    by 0x5766AFE: clone (clone.S:97)
==19795==
==19795== This conflicts with a previous write of size 8 by thread #2
==19795== Locks held: none
==19795==    at 0x10F968: data_race00::reader::h87e804792f6b43da (in /home/blt/projects/us/troutwine/concurrency_in_rust/external_projects/data_races/data_race00)
==19795==    by 0x12668E: __rust_maybe_catch_panic (lib.rs:102)
==19795==    by 0x110892: _$LT$F$u20$as$u20$alloc..boxed..FnBox$LT$A$GT$$GT$::call_box::h4b5b7e5f469a419a (in /home/blt/projects/us/troutwine/concurrency_in_rust/external_projects/data_races/data_race00)
==19795==    by 0x1191A7: call_once<(),()> (boxed.rs:798)
==19795==    by 0x1191A7: std::sys_common::thread::start_thread::hdc3a308e21d56a9c (thread.rs:24)
==19795==    by 0x112408: std::sys::unix::thread::Thread::new::thread_start::h555aed63620dece9 (thread.rs:90)
==19795==    by 0x4C32D06: mythread_wrapper (hg_intercepts.c:389)
==19795==    by 0x5251493: start_thread (pthread_create.c:333)
==19795==    by 0x5766AFE: clone (clone.S:97)
==19795==  Address 0x621a050 is in a rw- anonymous segment
==19795==
==19795== ----------------------------------------------------------------
```

输出不如预期清晰，但 helgrind 正在警告我们关于我们已知的数据覆盖问题。鼓励读者亲自运行 helgrind 并检查整个历史。

让我们改善这种情况。显然，我们存在竞态条件读写的问题，但我们在写入者的行为上也有问题。它覆盖了自己的写操作，对此毫无察觉。通过调整写入者，我们可以停止无意中覆盖写操作，如下所示：

```rs
fn writer(mut ring: Ring) -> () {
    let mut offset: isize = 0;
    let mut cur: u32 = 0;
    loop {
        unsafe {
            if (*ring).size != ((*ring).capacity as usize) {
                assert!(mem::replace(&mut *(*ring).data.offset(offset), 
                Some(cur)).is_none());
                (*ring).size += 1;
                cur = cur.wrapping_add(1);
                offset += 1;
                offset %= (*ring).capacity;
            } else {
                thread::yield_now();
            }
        }
    }
}
```

之后，我们运行程序几次，发现以下情况：

```rs
> ./data_race01
thread '<unnamed>thread '' panicked at '<unnamed>assertion failed: mem::replace(&mut *(*ring).data.offset(offset), Some(cur)).is_none()' panicked at '', assertion failed: `(left == right)`
  left: `20`,
   right: `10`data_race01.rs', :data_race01.rs65::8517:
   17note: Run with `RUST_BACKTRACE=1` for a backtrace.

thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: Any', libcore/result.rs:945:5
```

这很有趣！两个线程都因为相同的原因崩溃了。作者不适当地覆盖了一个写操作，而读者读取了它。

# 回到安全性

正如我们之前讨论的，前面的例子故意使用了低级且不安全的方法。我们如何使用 Rust 提供的工具构建类似的东西呢？这里有一个方法：

```rs
use std::{mem, thread};
use std::sync::{Arc, Mutex};

struct Ring {
    size: usize,
    data: Vec<Option<u32>>,
}

impl Ring {
    fn with_capacity(capacity: usize) -> Ring {
        let mut data: Vec<Option<u32>> = Vec::with_capacity(capacity);
        for _ in 0..capacity {
            data.push(None);
        }
        Ring {
            size: 0,
            data: data,
        }
    }

    fn capacity(&self) -> usize {
        self.data.capacity()
    }

    fn is_full(&self) -> bool {
        self.size == self.data.capacity()
    }

    fn emplace(&mut self, offset: usize, val: u32) -> Option<u32> {
        self.size += 1;
        let res = mem::replace(&mut self.data[offset], Some(val));
        res
    }

    fn displace(&mut self, offset: usize) -> Option<u32> {
        let res = mem::replace(&mut self.data[offset], None);
        if res.is_some() {
            self.size -= 1;
        }
        res
    }
}

fn writer(ring_lk: Arc<Mutex<Ring>>) -> () {
    let mut offset: usize = 0;
    let mut cur: u32 = 0;
    loop {
        let mut ring = ring_lk.lock().unwrap();
        if !ring.is_full() {
            assert!(ring.emplace(offset, cur).is_none());
            cur = cur.wrapping_add(1);
            offset += 1;
            offset %= ring.capacity();
        } else {
            thread::yield_now();
        }
    }
}

fn reader(read_limit: usize, ring_lk: Arc<Mutex<Ring>>) -> () {
    let mut offset: usize = 0;
    let mut cur: u32 = 0;
    while (cur as usize) < read_limit {
        let mut ring = ring_lk.lock().unwrap();
        if let Some(num) = ring.displace(offset) {
            assert_eq!(num, cur);
            cur = cur.wrapping_add(1);
            offset += 1;
            offset %= ring.capacity();
        } else {
            drop(ring);
            thread::yield_now();
        }
    }
}

fn main() {
    let capacity = 10;
    let read_limit = 1_000_000;
    let ring = Arc::new(Mutex::new(Ring::with_capacity(capacity)));

    let reader_ring = Arc::clone(&ring);
    let reader_jh = thread::spawn(move || {
        reader(read_limit, reader_ring);
    });
    let _writer_jh = thread::spawn(move || {
        writer(ring);
    });

    reader_jh.join().unwrap();
}
```

这与之前的竞争性程序非常相似。我们有环（Ring），它包含一个大小，并围绕`Vec<Option<u32>>`集操作。这次，`vec`没有被拆分成原始指针，环的实现也更加完善。实际上，在我们的前一个例子中，我们有可能提供更多的抽象实现——就像我们在 Rust 内部探索时所看到的那样——但是间接和不可安全是粗糙的组合。有时，间接的不可安全代码比直接的不可安全代码更难识别为有缺陷。无论如何，你会注意到在这个实现中`Ring: !Send`。相反，写入和读取线程在`Arc<Mutex<Ring>>`上操作。

# 排除法的安全性

让我们谈谈`Mutex`。互斥锁（Mutex）在线程之间提供**互斥**。Rust 的`Mutex`工作方式与来自其他编程语言的预期一致。任何调用`Mutex`锁的线程都会获取锁或者阻塞，直到持有互斥锁的线程将其解锁。锁的类型为`lock(&self) -> LockResult<MutexGuard<T>>`。这里有一个巧妙的技巧。`LockResult<Guard>`是`Result<Guard, PoisonError<Guard>>`。也就是说，获取互斥锁锁的线程实际上得到一个结果，成功时包含`MutexGuard<T>`，或者是一个*中毒*通知。当互斥锁的持有者崩溃时，Rust 会中毒其互斥锁，这种策略有助于防止只有一个线程崩溃的多线程传播继续运行。正因为如此，你会发现许多 Rust 程序不会检查锁调用的返回值，并立即展开它。`MutexGuard<T>`在释放时，会解锁互斥锁。一旦互斥锁被解锁，就不再可能访问其内部封装的数据，因此，一个线程无法与另一个线程进行不良交互。`Mutex`既是`Sync`也是`Send`，这很有道理。那么，为什么我们要用`Arc`包装我们的`Mutex<Ring>`呢？`Arc<T>`究竟是什么呢？

原子引用计数指针`Arc<T>`。如前所述，`Rc<T>`不是线程安全的，因为如果它被标记为线程安全，那么内部强/弱计数器之间会有竞争，就像在我们的故意竞争的程序中一样。`Arc<T>`建立在原子整数之上，我们将在下一章中更详细地介绍。现在只需说，`Arc<T>`能够作为引用计数指针工作，而不会在线程之间引入数据竞争。`Arc<T>`可以在`Rc<T>`可以使用的任何地方使用，除了这些原子整数不是免费的。如果你可以使用`Rc<T>`，那就使用它。无论如何，下一章将有更多关于这个话题的介绍。

为什么是`Arc<Mutex<Ring>>`？因为`Mutex<Ring>`可以移动但不能克隆。让我们看看`Mutex`内部的情况：

```rs
pub struct Mutex<T: ?Sized> {
    // Note that this mutex is in a *box*, not inlined into
    // the struct itself. Once a native mutex has been used 
    // once, its address can never change (it can't be 
    // moved). This mutex type can be safely moved at any time,
    // so to ensure that the native mutex is used correctly we
    // box the inner mutex to give it a constant address.
    inner: Box<sys::Mutex>,
    poison: poison::Flag,
    data: UnsafeCell<T>,
}
```

我们可以看到`T`可能或可能不是有大小，并且`T`被存储在名为`sys::Mutex`的`UnsafeCell<T>`旁边。Rust 运行的每个平台都会提供自己的互斥锁形式，因为这些通常与操作系统环境相关联。如果你查看`rustc`代码，你会发现`sys::Mutex`是系统依赖互斥锁实现的包装器。`Unix`实现位于`src/libstd/sys/unix/mutex.rs`中，它是一个围绕`pthread_mutex_t`的`UnsafeCell`，正如你所期望的那样：

```rs
pub struct Mutex { inner: UnsafeCell<libc::pthread_mutex_t> }
```

并非严格必要在系统依赖的基础上实现`Mutex`，正如我们在原子原语章节中构建自己的锁时将会看到的。一般来说，除非有充分的理由不这样做（比如教学目的），否则使用可用的和经过良好测试的组件是一个好主意。

现在，如果我们克隆`Mutex<T>`会发生什么呢？首先，我们需要为系统互斥锁分配新的内存，如果`T`本身是可克隆的，可能还需要一个新的`UnsafeCell`。新的系统互斥锁是真正的问题——线程必须在内存中的相同结构上同步。将`Mutex`放入`Arc`中可以解决这个问题。像克隆`Rc`一样，克隆`Arc`会创建对`Mutex`的新强引用。这些引用是不可变的。这怎么工作呢？首先，`Arc`内部的互斥锁永远不会改变。在 Rust 提供的抽象模型中，互斥锁本身没有内部状态，并且通过锁定线程并没有真正发生任何修改。当然，这实际上并不完全正确，通过检查我们可以发现。Rust 的`Mutex`之所以表现得这样，是因为它周围包围着系统依赖结构的内部`UnsafeCell`。Rust 的`Mutex`利用了`UnsafeCell`允许的内部可变性。互斥锁本身保持不可变，而内部`T`通过`MutexGuard`可变地引用。这在 Rust 的内存模型中是安全的，因为由于互斥排他，任何给定时间只有一个可变引用到`T`：

```rs
fn main() {
    let capacity = 10;
    let read_limit = 1_000_000;
    let ring = Arc::new(Mutex::new(Ring::with_capacity(capacity)));

    let reader_ring = Arc::clone(&ring);
    let reader_jh = thread::spawn(move || {
        reader(read_limit, reader_ring);
    });
    let _writer_jh = thread::spawn(move || {
        writer(ring);
    });

    reader_jh.join().unwrap();
}
```

我们将`Ring`包裹在一个`Arc`、`Mutex`层中，将其克隆给读者，并将其移动到写者那里。在这里，这是一个风格问题，但重要的是要意识到，如果主线程创建并克隆一个`Arc`到子线程中，那么`Arc`的内容仍然存活，至少在主线程存活期间是这样。例如，如果一个文件处理程序被保存在主线程的`Arc`中，克隆到临时启动的工人，然后没有被丢弃，那么文件处理程序本身将永远不会关闭。当然，这可能是也可能不是你的程序所期望的。读者被鼓励在自己的系统上确认这一点。

此外，程序在多次运行后成功完成。不幸的是，从理论上讲，它并不特别高效。锁定互斥锁不是免费的，尽管我们的线程操作很短——除了一个内存交换外，还有少量算术运算——当一个线程持有互斥锁时，另一个线程会愚蠢地等待。Linux perf 证明了这一点：

```rs
> perf stat --event task-clock,context-switches,page-faults,cycles,instructions,branches,branch-misses,cache-references,cache-misses ./data_race02

 Performance counter stats for './data_race02':

        988.944526      task-clock (msec)         #    1.943 CPUs utilized
             8,005      context-switches          #    0.008 M/sec
               137      page-faults               #    0.139 K/sec
     2,836,237,759      cycles                    #    2.868 GHz 
     1,074,887,696      instructions              #    0.38  insn per cycle
       198,103,111      branches                  #  200.318 M/sec
         1,557,868      branch-misses             #    0.79% of all branches
        63,377,456      cache-references          #   64.086 M/sec
             3,326      cache-misses              #    0.005 % of all cache refs

       0.508990976 seconds time elapsed
```

在执行过程中只使用了 1.3 个 CPU。对于这样一个简单的程序，我们可能更倾向于采用最简单的方法。此外，由于存在内存屏障的缓存写入效应——互斥锁肯定是一种——选择互斥锁而不是细粒度锁定策略可能更便宜。最终，这取决于在开发时间、机器效率的需求以及定义所使用 CPU 的机器效率之间的权衡。我们将在后面的章节中看到一些这方面的内容。

# 使用 MPSC

我们从 Ring 的奇特角度出发，一开始就试图构建一个功能不正常的程序，然后稍后对其进行改进。在一个真正的程序中，目的和功能会逐渐偏离其起源，偶尔问一下软件作为一个抽象概念实际上*做什么*是有意义的。那么，Ring 实际上*做什么*呢？首先，它是有限的，这是众所周知的。它有一个固定的容量，读者/写入器对会小心地保持在那个容量内。其次，它是一个旨在由两个或更多线程操作的架构，这些线程有两个角色：读取和写入。Ring 是线程间传递`u32`的一种方式，按照它们被写入的顺序进行读取。

幸运的是，只要我们愿意只接受一个读取线程，Rust 的标准库就提供了一些东西来满足这个常见需求——`std::sync::mpsc`。多生产者单消费者队列是这样一个队列，其写入端——称为`Sender<T>`——可以被克隆并移动到多个线程中。读取线程——称为`Receiver<T>`——只能被移动。有`Sender<T>`和`SyncSender<T>`两种发送者变体。第一个代表无界 MPSC，这意味着通道将根据需要分配空间以容纳发送到其中的`Ts`。第二个代表有界 MPSC，这意味着通道的内部存储有一个固定的上限容量。虽然 Rust 文档将`Sender<T>`和`SyncSender<T>`描述为异步和同步，但这并不完全正确。`SyncSender<T>::send`如果没有可用空间，将会阻塞，但还有一个`SyncSender<T>::try_send`，它是一个类型为`try_send(&self, t: T) -> Result<()`, `TrySendError<T>>`的函数。可以在有限的内存中使用 Rust 的 MPSC，同时考虑到调用者必须有一个策略来处理无法放入空间中的输入。

我们的`u32`传递程序使用 Rust 的 MPSC 看起来像这样：

```rs
use std::thread;
use std::sync::mpsc;

fn writer(chan: mpsc::SyncSender<u32>) -> () {
    let mut cur: u32 = 0;
    while let Ok(()) = chan.send(cur) {
        cur = cur.wrapping_add(1);
    }
}

fn reader(read_limit: usize, chan: mpsc::Receiver<u32>) -> () {
    let mut cur: u32 = 0;
    while (cur as usize) < read_limit {
        let num = chan.recv().unwrap();
        assert_eq!(num, cur);
        cur = cur.wrapping_add(1);
    }
}

fn main() {
    let capacity = 10;
    let read_limit = 1_000_000;
    let (snd, rcv) = mpsc::sync_channel(capacity);

    let reader_jh = thread::spawn(move || {
        reader(read_limit, rcv);
    });
    let _writer_jh = thread::spawn(move || {
        writer(snd);
    });

    reader_jh.join().unwrap();
}
```

这个程序比我们之前的任何程序都要短，而且根本不会出现算术错误。`SyncSender`的发送类型是`send(&self, t: T) -> Result<()`, `SendError<T>>`，这意味着必须处理`SendError`以避免程序崩溃。`SendError`仅在 MPSC 通道的远程端断开连接时返回，就像在这个程序中当读取器达到其`read_limit`时会发生的那样。这个奇怪程序的性能特性并不像最后一个`Ring`程序那样快：

```rs
> perf stat --event task-clock,context-switches,page-faults,cycles,instructions,branches,branch-misses,cache-references,cache-misses ./data_race03

 Performance counter stats for './data_race03':

        760.250406      task-clock (msec)   #    0.765 CPUs utilized
           200,011      context-switches    #    0.263 M/sec
               135      page-faults         #    0.178 K/sec
     1,740,006,788      cycles              #    2.289 GHz
     1,533,396,017      instructions        #    0.88  insn per cycle
       327,682,344      branches            #  431.019 M/sec
     741,095      branch-misses             #    0.23% of all branches
        71,426,781      cache-references    #   93.952 M/sec
          4,082      cache-misses          #  0.006 % of all cache refs

       0.993142979 seconds time elapsed
```

但它完全在误差范围内，尤其是在考虑到编程的简便性。有一点值得记住的是，MPSC 中的消息只在一个方向上流动：通道不是双向的。因此，MPSC 不适合请求/响应模式。将两个或更多 MPSC 层叠起来以实现这一点并不罕见，理解到单侧的单个消费者有时可能不合适。

# 一个遥测服务器

让我们构建一个描述性统计服务器。这些服务器以某种方式在组织内部频繁出现：需要一个能够消费事件、对这些事件进行某种描述性统计计算，然后将描述输出到其他系统的工具。我的工作项目 postmates/cernan ([`crates.io/crates/cernan`](https://crates.io/crates/cernan))存在的正是为了在资源受限的设备上以规模满足这一需求，同时不将操作人员绑定到任何类型的管道。我们现在要构建的是一个迷你版的 cernan，其流程如下：

```rs
                                _--> high_filter --> cma_egress
                               /
  telemetry -> ingest_point ---
    (udp)                      \_--> low_filter --> ckms_egress
```

理念是从简单的 UDP 协议中提取`telemetry`，尽可能快地接收它以避免操作系统丢弃数据包，然后通过高、低过滤器将这些点传递出去，最后将过滤后的点交给两个不同的统计`egress`点。与高过滤器相关联的`egress`计算连续移动平均，而与低过滤器相关联的`egress`使用 postmates/quantiles ([`crates.io/crates/quantiles`](https://crates.io/crates/quantiles))库中的近似算法计算分位数摘要。

让我们深入探讨。首先，让我们看看项目的`Cargo.toml`文件：

```rs
[package]
name = "telem"
version = "0.1.0"

[dependencies]
quantiles = "0.7"
seahash = "3.0"

[[bin]]
name = "telem"
doc = false
```

简洁明了。我们像之前提到的那样，绘制分位数，以及 seahasher。Seahasher 是一种特别快的哈希器——但不是加密安全的——我们将将其替换到`HashMap`中。关于这一点，稍后会有更多介绍。我们的可执行文件被拆分到`src/bin/telem.rs`，因为这个项目是一个分割的库/二进制设置：

```rs
extern crate telem;

use std::{thread, time};
use std::sync::mpsc;
use telem::IngestPoint;
use telem::egress::{CKMSEgress, CMAEgress, Egress};
use telem::event::Event;
use telem::filter::{Filter, HighFilter, LowFilter};

fn main() {
    let limit = 100;
    let (lp_ic_snd, lp_ic_rcv) = mpsc::channel::<Event>();
    let (hp_ic_snd, hp_ic_rcv) = mpsc::channel::<Event>();
    let (ckms_snd, ckms_rcv) = mpsc::channel::<Event>();
    let (cma_snd, cma_rcv) = mpsc::channel::<Event>();

    let filter_sends = vec![lp_ic_snd, hp_ic_snd];
    let ingest_filter_sends = filter_sends.clone();
    let _ingest_jh = thread::spawn(move || {
        IngestPoint::init("127.0.0.1".to_string(), 1990, 
        ingest_filter_sends).run();
    });
    let _low_jh = thread::spawn(move || {
        let mut low_filter = LowFilter::new(limit);
        low_filter.run(lp_ic_rcv, vec![ckms_snd]);
    });
    let _high_jh = thread::spawn(move || {
        let mut high_filter = HighFilter::new(limit);
        high_filter.run(hp_ic_rcv, vec![cma_snd]);
    });
    let _ckms_egress_jh = thread::spawn(move || {
        CKMSEgress::new(0.01).run(ckms_rcv);
    });
    let _cma_egress_jh = thread::spawn(move || {
        CMAEgress::new().run(cma_rcv);
    });

    let one_second = time::Duration::from_millis(1_000);
    loop {
        for snd in &filter_sends {
            snd.send(Event::Flush).unwrap();
        }
        thread::sleep(one_second);
    }
}
```

这里有很多事情在进行。前八行是我们需要的库片段的导入。`main`的主体主要由设置我们的工作线程并将适当的通道喂给它们组成。注意，一些线程需要通道的多个发送端：

```rs
    let filter_sends = vec![lp_ic_snd, hp_ic_snd];
    let ingest_filter_sends = filter_sends.clone();
    let _ingest_jh = thread::spawn(move || {
        IngestPoint::init("127.0.0.1".to_string(), 1990, 
        ingest_filter_sends).run();
    });
```

这就是我们在 Rust MPSC 中使用扇出方式的方法。让我们看看`IngestPoint`，实际上它在`src/ingest_point.rs`中定义：

```rs
use event;
use std::{net, thread};
use std::net::ToSocketAddrs;
use std::str;
use std::str::FromStr;
use std::sync::mpsc;
use util;

pub struct IngestPoint {
    host: String,
    port: u16,
    chans: Vec<mpsc::Sender<event::Event>>,
}
```

`IngestPoint`是一个主机——根据`ToSocketAddrs`，可以是 IP 地址或 DNS 主机名，一个端口和一个`mpsc::Sender<event::Event>`的向量。内部类型是我们定义的：

```rs
#[derive(Clone)]
pub enum Event {
    Telemetry(Telemetry),
    Flush,
}

#[derive(Debug, Clone)]
pub struct Telemetry {
    pub name: String,
    pub value: u32,
}
```

`telem`有两种*事件*流经它——`Telemetry`来自`IngresPoint`，`Flush`来自主线程。`Flush`像系统的一个时钟滴答，允许项目的各个子系统跟踪时间而不需要参考墙上的时钟。在嵌入式程序中，用一些已知的脉冲来定义时间是很常见的，如果可能的话，我在并行编程中也尽量保持这一点。至少，将时间作为一个系统外部推动的属性有助于测试。无论如何，回到`IngestPoint`：

```rs
impl IngestPoint {
    pub fn init(
        host: String,
        port: u16,
        chans: Vec<mpsc::Sender<event::Event>>,
    ) -> IngestPoint {
        IngestPoint {
            chans: chans,
            host: host,
            port: port,
        }
    }

    pub fn run(&mut self) {
        let mut joins = Vec::new();

        let addrs = (self.host.as_str(), self.port).to_socket_addrs();
        if let Ok(ips) = addrs {
            let ips: Vec<_> = ips.collect();
            for addr in ips {
                let listener =
                    net::UdpSocket::bind(addr)
                        .expect("Unable to bind to UDP socket");
                let chans = self.chans.clone();
                joins.push(thread::spawn(move || handle_udp(chans, 
                                                   &listener)));
            }
        }

        for jh in joins {
            jh.join().expect("Uh oh, child thread panicked!");
        }
    }
}
```

这里的第一部分，`init`，仅仅是设置。运行函数调用`to_socket_addrs`在我们的主机/端口对上，并检索所有相关的 IP 地址。每个这些地址都绑定一个`UdpSocket`和一个操作系统线程来监听从这个套接字来的数据报。这在线程开销方面是浪费的，在本书的后续部分我们将讨论基于事件的 IO 替代方案。之前讨论过的 Cernan，作为一个生产系统，在其*源代码*中使用了 Mio。这里的关键函数是`handle_udp`，这个函数被传递给新的监听线程。它如下所示：

```rs
fn handle_udp(mut chans: Vec<mpsc::Sender<event::Event>>, 
              socket: &net::UdpSocket) {
    let mut buf = vec![0; 16_250];
    loop {
        let (len, _) = match socket.recv_from(&mut buf) {
            Ok(r) => r,
            Err(e) => { 
                panic!(
                    format!("Could not read UDP socket with \
                            error {:?}", e)),
            }
        };
        if let Some(telem) = 
        parse_packet(str::from_utf8(&buf[..len]).unwrap()) {
            util::send(&mut chans, event::Event::Telemetry(telem));
        }
    }
}
```

这个函数是一个简单的无限循环，从套接字中拉取数据报到一个 16 KB 的缓冲区——比大多数数据报都要大得多——然后对结果调用`parse_packet`。如果数据报是我们尚未指定的协议的有效示例，那么我们就调用`util::send`来通过`chans`中的`Sender<event::Event>`发送`Event::Telemetry`。`util::send`不过是一个 for 循环：

```rs
pub fn send(chans: &[mpsc::Sender<event::Event>], event: event::Event) {
    if chans.is_empty() {
        return;
    }

    for chan in chans.iter() {
        chan.send(event.clone()).unwrap();
    }
}
```

消费负载没有什么特别之处：一个非空白字符的名字，后面跟着一个或多个空白字符，然后是一个`u32`，所有字符串都是字符串编码和 utf8 有效的：

```rs
fn parse_packet(buf: &str) -> Option<event::Telemetry> {
    let mut iter = buf.split_whitespace();
    if let Some(name) = iter.next() {
        if let Some(val) = iter.next() {
            match u32::from_str(val) {
                Ok(int) => {
                    return Some(event::Telemetry {
                        name: name.to_string(),
                        value: int,
                    })
                }
                Err(_) => return None,
            };
        }
    }
    None
}
```

转到过滤器，`HighFilter`和`LowFilter`都是基于一个共同的`Filter`特质完成的，这个特质定义在`src/filter/mod.rs`中：

```rs
use event;
use std::sync::mpsc;
use util;

mod high_filter;
mod low_filter;

pub use self::high_filter::*;
pub use self::low_filter::*;

pub trait Filter {
    fn process(
        &mut self,
        event: event::Telemetry,
        res: &mut Vec<event::Telemetry>,
    ) -> ();

    fn run(
        &mut self,
        recv: mpsc::Receiver<event::Event>,
        chans: Vec<mpsc::Sender<event::Event>>,
    ) {
        let mut telems = Vec::with_capacity(64);
        for event in recv.into_iter() {
            match event {
                event::Event::Flush => util::send(&chans, 
                 event::Event::Flush),
                event::Event::Telemetry(telem) => {
                    self.process(telem, &mut telems);
                    for telem in telems.drain(..) {
                        util::send(&chans, 
                        event::Event::Telemetry(telem))
                    }
                }
            }
        }
    }
}
```

任何实现滤波器的类都需要提供自己的处理过程。这是默认运行函数在从`Receiver<Event>`中拉取`Event`并发现它是一个`Telemetry`时调用的函数。尽管高低通滤波器都没有使用它，但处理函数能够通过在传递的`telems`向量上推送更多内容来将新的`Telemetry`注入到流中。这就是 cernan 的*可编程滤波器*能够允许最终用户从 Lua 脚本中创建`telemetry`的原因。此外，为什么传递`telems`而不是让处理函数返回一个`Telemetry`向量呢？这样可以避免持续的小型分配。根据系统不同，线程之间的分配可能不会协调一致——这意味着在高负载情况下可能会出现神秘的暂停——因此，如果代码没有因为这样的关注而扭曲成某种奇怪的版本，那么尽可能避免它们是一种好的编程风格。

无论是低通滤波器还是高通滤波器，基本原理都是相同的。当点小于或等于一个预定义的限制时，低通滤波器会通过自身传递这个点，而高通滤波器则大于或等于这个限制。下面是`LowFilter`的定义，位于`src/filter/low_filter.rs`文件中：

```rs
use event;
use filter::Filter;

pub struct LowFilter {
    limit: u32,
}

impl LowFilter {
    pub fn new(limit: u32) -> Self {
        LowFilter { limit: limit }
    }
}

impl Filter for LowFilter {
    fn process(
        &mut self,
        event: event::Telemetry,
        res: &mut Vec<event::Telemetry>,
    ) -> () {
        if event.value <= self.limit {
            res.push(event);
        }
    }
}
```

`Egress`的出口定义与滤波器的方式类似，分为子模块和公共特质。特质存在于`src/egress/mod.rs`文件中：

```rs
use event;
use std::sync::mpsc;

mod cma_egress;
mod ckms_egress;

pub use self::ckms_egress::*;
pub use self::cma_egress::*;

pub trait Egress {
    fn deliver(&mut self, event: event::Telemetry) -> ();

    fn report(&mut self) -> ();

    fn run(&mut self, recv: mpsc::Receiver<event::Event>) {
        for event in recv.into_iter() {
            match event {
                event::Event::Telemetry(telem) => self.deliver(telem),
                event::Event::Flush => self.report(),
            }
        }
    }
}
```

交付函数的目的是为`egress`提供存储其`Telemetry`的功能。报告函数的目的是迫使`Egress`的实现者向外部世界发布他们的汇总`Telemetry`。我们两个`Egress`实现者——`CKMSEgress`和`CMAEgress`——只是打印它们的信息，但你可以想象一个`Egress`通过某种网络协议将信息发送到远程系统。实际上，这正是 cernan 的`Sinks`所做的事情，跨越许多协议和传输。让我们看看单个出口，因为它们都非常相似。`CKMSEgress`的定义位于`src/egress/ckms_egress.rs`文件中：

```rs
use egress::Egress;
use event;
use quantiles;
use util;

pub struct CKMSEgress {
    error: f64,
    data: util::HashMap<String, quantiles::ckms::CKMS<u32>>,
    new_data_since_last_report: bool,
}

impl Egress for CKMSEgress {
    fn deliver(&mut self, event: event::Telemetry) -> () {
        self.new_data_since_last_report = true;
        let val = event.value;
        let ckms = self.data
            .entry(event.name)
            .or_insert(quantiles::ckms::CKMS::new(self.error));
        ckms.insert(val);
    }

    fn report(&mut self) -> () {
        if self.new_data_since_last_report {
            for (k, v) in &self.data {
                for q in &[0.0, 0.25, 0.5, 0.75, 0.9, 0.99] {
                    println!("[CKMS] {} {}:{}", k, q, 
                             v.query(*q).unwrap().1);
                }
            }
            self.new_data_since_last_report = false;
        }
    }
}

impl CKMSEgress {
    pub fn new(error: f64) -> Self {
        CKMSEgress {
            error: error,
            data: Default::default(),
            new_data_since_last_report: false,
        }
    }
}
```

注意`data: util::HashMap<String, quantiles::ckms::CKMS<u32>>`。这个`util::HashMap`是一个类型别名，代表`std::collections::HashMap<K, V, hash::BuildHasherDefault<SeaHasher>>`，如前所述。在这里，散列的加密安全性不如散列速度重要，这就是我们选择`SeaHasher`的原因。在 crates 中有很多可用的替代散列器，能够根据你的用例进行交换是一个很酷的技巧。"quantiles::ckms::CKMS"是一种近似数据结构，由 Cormode 等人定义在《在数据流上有效计算偏斜分位数》一文中。许多摘要系统在有限空间内运行，但愿意容忍错误。CKMS 数据结构允许在保持分位数近似误差保证界限的同时进行点丢弃。关于数据结构的讨论超出了本书的范围，但实现很有趣，论文也写得非常好。无论如何，这就是错误设置的全部内容。如果你回到主函数，注意我们硬编码错误为 0.01，或者说，任何分位数摘要都保证在 0.01 内偏离真实值。

实话实说，就是这样了。我们刚刚走过了非平凡 Rust 程序的大部分内容，这个程序是围绕标准库中提供的 MPSC 抽象构建的。让我们稍微玩弄一下它。在一个 shell 中，启动`telem`：

```rs
> cargo run --release
   Compiling quantiles v0.7.0
   Compiling seahash v3.0.5
   Compiling telem v0.1.0 (file:///Users/blt/projects/us/troutwine/concurrency_in_rust/external_projects/telem)
    Finished release [optimized] target(s) in 7.16 secs
     Running `target/release/telem`
```

在另一个 shell 中，开始发送 UDP 数据包。在 macOS 上，你可以这样使用`nc`：

```rs
> echo "a 10" | nc -c -u 127.0.0.1 1990
> echo "a 55" | nc -c -u 127.0.0.1 1990
```

在 Linux 上的调用类似；你只需要小心不要等待响应即可。在原始 shell 中，你在一秒后应该看到如下输出：

```rs
[CKMS] a 0:10
[CKMS] a 0.25:10
[CKMS] a 0.5:10
[CKMS] a 0.75:10
[CKMS] a 0.9:10
[CKMS] a 0.99:10
[CKMS] a 0:10
[CKMS] a 0.25:10
[CKMS] a 0.5:10
[CKMS] a 0.75:55
[CKMS] a 0.9:55
[CKMS] a 0.99:55
```

由于这些点低于限制，它们已经通过了低过滤并进入了 CKMS `egress`。回到我们的另一个 shell：

```rs
> echo "b 1000" | nc -c -u 127.0.0.1 1990
> echo "b 2000" | nc -c -u 127.0.0.1 1990
> echo "b 3000" | nc -c -u 127.0.0.1 1990
```

在`telem` shell 中：

```rs
[CMA] b 1000
[CMA] b 1500
[CMA] b 2000
```

嗯，正如预期的那样。由于这些点超过了限制，它们已经通过了高过滤并进入了 CMA `egress`。只要没有点进来，`telem`应该几乎不消耗 CPU，大约每秒唤醒一次线程以进行刷新脉冲。内存消耗也将非常低，主要表现为`IngestPoint`中的大输入缓冲区。

这个程序存在一些问题。在许多地方，我们明确地在潜在的问题点上 panic 或 unwrap，而不是处理这些问题。虽然在生产程序中 unwrap 是合理的，但你应该非常确定你为你的程序爆炸的错误无法以其他方式处理。最令人担忧的是，程序没有*背压*的概念。如果`IngestPoint`能够比过滤器或`egress`更快地产生点，它们可以吸收线程之间的通道，从而保持空间分配。将程序稍作修改以使用`mpsc::SyncSender`将是合适的——一旦通道填满到容量，就会应用背压——但前提是删除点是可以接受的。鉴于摄取协议是 UDP，这几乎肯定是可以的，但读者可以想象一个它不会是这样的场景。Rust 的标准库在需要替代形式的背压的领域存在困难，但诚然，这些是相当晦涩的。如果你对阅读一个类似于`telem`但没有任何这里确定的问题的生产程序感兴趣，我强烈推荐 cernan 代码库。

总的来说，Rust 的 MPSC 在构建并行系统时是一个非常实用的工具。在本章中，我们构建了自己的缓冲通道 Ring，但这并不常见。除非你有相当专业的用例，否则你不会考虑不使用标准库的 MPSC 进行线程间基于通道的通信。在我们介绍更多 Rust 的基本并发原语之后，下一章我们将探讨这样一个用例。

# 摘要

在本章中，我们讨论了 Rust 并发的基石——`Sync`和`Send`。此外，我们开始探讨在 Rust 中使原语线程安全的原因以及如何使用这些原语构建并发结构。我们通过一个不正确同步的程序进行推理，展示了了解 Rust 内存模型以及像`helgrind`这样的工具如何帮助我们确定程序中出了什么问题。这对读者来说可能并不意外，这是一个非常繁琐的过程，而且很可能出错。在第五章，*锁——Mutex, Condvar, Barriers 和 RWLock*，我们将讨论 Rust 向程序员暴露的高级粗粒度同步原语。在第六章，*原子——同步的原始元素*，我们将讨论现代机器暴露的细粒度同步原语。

# 进一步阅读

安全的并发编程是一个非常广泛的话题，这里提供的进一步阅读建议反映了这一点。细心的读者会注意到，这些参考文献跨越了时间和方法，反映了机器和语言随时间发生的广泛变化。

+   *《多处理器编程的艺术》，作者：Maurice Herlihy 和 Nir Shavit。这本书是关于多处理器算法的优秀入门书籍。由于作者假设 Java 环境——垃圾回收在实现回收方面是一个巨大的胜利，因此将应用系统语言变得有些困难。*

+   *《C++并发实战：实用多线程》，作者：Anthony Williams。这本书是 TAoMP 的绝佳伴侣，专注于 C++中类似结构的实现。虽然 C++和 Rust 之间需要翻译步骤，但与 Java 到 Rust 的跳跃相比，这并不是那么大的跳跃。*

+   *《LVars：基于格的数据结构用于确定性并行性》，作者：Lindsey Kuper 和 Ryan Newton。这本书展示了一种特定的并行构造方法，它受到当前机器的影响。然而，我们的当前模型可能过于复杂，将来可能会被视为过时，即使在没有牺牲原始性能的情况下，就像我们在转向基于虚拟机的语言时所做的那样。这篇论文提出了一种受近年来分布式算法研究影响的构造替代方案。它可能是或可能不是未来，但鼓励读者保持关注。*

+   *《ffwd：委托比你想的快得多》，作者：Sepideh Roghanchi，Jakob Eriksson，和 Nilanjana Basu。现代机器是奇怪的生物。从概念上讲，最快的数据结构是那种最小化工作线程等待时间的数据结构，通过快速执行指令来加速工作完成。但这并不完全是这样，正如这篇论文所展示的。作者通过仔细维护缓存局部性，能够超越那些理论上可能更快但线程间更积极共享的复杂结构。*

+   *《平坦组合与同步-并行性权衡》，作者：Danny Hendler，Itai Incze，和 Nir Shavit。与 ffwd 类似，作者们提出了一种构建并发结构的方法，这种方法依赖于线程间的粗粒度独占锁定和操作日志的周期性组合。与缓存的交互使得平坦组合比不与缓存优雅交互的更复杂的无锁/等待自由替代方案*更快*。*

+   *《新 Rustacean，e022：Send 和 Sync》，可在[`www.newrustacean.com/show_notes/e022/struct.Script.html`](http://www.newrustacean.com/show_notes/e022/struct.Script.html)找到。新 Rustacean 是一个针对所有水平 Rust 开发者的优秀播客。这一集与当前章节讨论的内容非常契合。强烈推荐。*

+   *有效计算数据流中的偏斜分位数*，Graham Cormode, Flip Korn, S. Muthukrishnan, 和 Divesh Srivastava。这篇论文奠定了在电信中使用的 CKMS 结构。考察论文中概述的实现与库中发现的实现之间的差异是有教育意义的——基于链表的实现与跳表的一个变体。这种差异完全是由于缓存局部性考虑。

+   *Cernan 项目*，多位开发者，可在[`github.com/postmates/cernan`](https://github.com/postmates/cernan)下以 MIT 许可证获取。Cernan 是一个事件多路复用服务器，是本章讨论的玩具电信的生产版本。截至撰写本书时，它由 14 人贡献了 17,000 行代码。已经仔细注意到了保持低资源消耗和高性能。我是这个项目的首席作者，本书中讨论的技术已应用于 Cernan。
