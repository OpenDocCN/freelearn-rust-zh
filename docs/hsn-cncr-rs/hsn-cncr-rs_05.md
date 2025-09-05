# 第五章：锁定机制 – Mutex、Condvar、屏障和 RWLock

在本章中，我们将深入研究 hopper，它是第四章，*Sync 和 Send – Rust 并发基础*中 Ring 的成熟版本。hopper 处理背压——我们在 telem*中识别出的弱点——是在填满容量时阻塞，就像`SyncSender`一样。hopper 的特殊技巧是将其页面输出到磁盘。hopper 用户定义了 hopper 允许消耗多少内存空间，就像`SyncSender`一样，只是以字节为单位而不是`T`的总元素数。此外，当 hopper 的内存容量填满并需要页面输出到磁盘时，用户还可以配置消耗的磁盘字节数。MSPC 的其他属性保持不变，如有序交付、存储后保留数据等。

然而，在我们深入挖掘之前，我们需要介绍更多 Rust 的并发原语。我们将通过解决来自*《信号量小书》*的一些谜题来解释它们，由于 Rust 没有提供信号量，因此在某些地方可能会有些棘手。

到本章结束时，我们将：

+   讨论了 Mutex、Condvar、屏障和 RWLock 的目的和使用方法

+   调查了标准库中 MPSC 的磁盘后端特殊化版本 hopper

+   看到了如何在生产环境中应用 QuickCheck、AFL 和全面基准测试

# 技术要求

本章需要安装一个有效的 Rust 环境。验证安装的详细信息在第一章，*预备知识 – 机器架构和 Rust 入门*中有所介绍。不需要额外的软件工具。

您可以在 GitHub 上找到本书项目的源代码：[`github.com/PacktPublishing/Rust-Concurrency/`](https://github.com/PacktPublishing/Rust-Concurrency/)。本章的源代码位于`Chapter05`。

# 读取多个，写入独占锁——RwLock

考虑这样一种情况，你有一个资源，一次只能由一个线程进行操作，但可以被多个线程安全地查询——也就是说，你有多个读者和一个写者。虽然我们可以用`Mutex`来保护这个资源，但问题是互斥锁对它的锁定者没有区分；无论它们的意图如何，每个线程都将被迫等待。`RwLock<T>`是互斥锁概念的替代品，允许两种类型的锁——读锁和写锁。类似于 Rust 的引用，一次只能有一个写锁，但可以有多个读锁，且不包括写锁。让我们看一个例子：

```rs
use std::thread;
use std::sync::{Arc, RwLock};

fn main() {
    let resource: Arc<RwLock<u16>> = Arc::new(RwLock::new(0));

    let total_readers = 5;

    let mut reader_jhs = Vec::with_capacity(total_readers);
    for _ in 0..total_readers {
        let resource = Arc::clone(&resource);
        reader_jhs.push(thread::spawn(move || {
            let mut total_lock_success = 0;
            let mut total_lock_failure = 0;
            let mut total_zeros = 0;
            while total_zeros < 100 {
                match resource.try_read() {
                    Ok(guard) => {
                        total_lock_success += 1;
                        if *guard == 0 {
                            total_zeros += 1;
                        }
                    }
                    Err(_) => {
                        total_lock_failure += 1;
                    }
                }
            }
            (total_lock_failure, total_lock_success)
        }));
    }

    {
        let mut loops = 0;
        while loops < 100 {
            let mut guard = resource.write().unwrap();
            *guard = guard.wrapping_add(1);
            if *guard == 0 {
                loops += 1;
            }
        }
    }

    for jh in reader_jhs {
        println!("{:?}", jh.join().unwrap());
    }
}
```

这里的想法是，我们将有一个写线程在循环中旋转并递增一个共享资源——一个 `u16`。一旦 `u16` 被包装了 100 次，写线程将退出。同时，一定数量的读线程（`total_readers`）将尝试获取共享资源——一个 `u16` 的读锁，直到它达到零 `100` 次。本质上，我们在这里是在赌线程的顺序。程序通常会以以下结果退出：

```rs
(0, 100)
(0, 100)
(0, 100)
(0, 100)
(0, 100)
```

这意味着每个读线程从未失败地获取其读锁——没有写锁存在。也就是说，读线程在写线程之前被调度。我们的主函数只连接到读处理程序，所以写线程在我们退出时仍在写。有时，我们会遇到正确的调度顺序，并得到以下结果：

```rs
(0, 100)
(126143752, 2630308)
(0, 100)
(0, 100)
(126463166, 2736405)
```

在这个特定的例子中，第二个和最后一个读线程在写线程之后被调度，并设法捕捉到守卫不为零的时刻。回想一下，这对中的第一个元素是读线程无法获取读锁并被迫重试的总次数。第二个是获取锁的次数。总的来说，写线程执行了 `(2¹⁸ * 100) ~= 2²⁴` 次写操作，而第二个读线程执行了 `log_2 2630308 ~= 2²¹` 次读操作。这丢失了很多写操作，也许是可以接受的。但更令人担忧的是，这导致了大约 `2²⁶` 次无用的循环。海平面正在上升，而我们像没有人需要为此而死一样燃烧着电力。

我们如何避免所有这些浪费的努力？嗯，像大多数事情一样，这取决于我们想要做什么。如果我们需要每个读者都能获取到每个写操作，那么 MPSC 是一个合理的选择。它看起来会是这样：

```rs
use std::thread;
use std::sync::mpsc;

fn main() {
    let total_readers = 5;
    let mut sends = Vec::with_capacity(total_readers);

    let mut reader_jhs = Vec::with_capacity(total_readers);
    for _ in 0..total_readers {
        let (snd, rcv) = mpsc::sync_channel(64);
        sends.push(snd);
        reader_jhs.push(thread::spawn(move || {
            let mut total_zeros = 0;
            let mut seen_values = 0;
            for v in rcv {
                seen_values += 1;
                if v == 0 {
                    total_zeros += 1;
                }
                if total_zeros >= 100 {
                    break;
                }
            }
            seen_values
        }));
    }

    {
        let mut loops = 0;
        let mut cur: u16 = 0;
        while loops < 100 {
            cur = cur.wrapping_add(1);
            for snd in &sends {
                snd.send(cur).expect("failed to send");
            }
            if cur == 0 {
                loops += 1;
            }
        }
    }

    for jh in reader_jhs {
        println!("{:?}", jh.join().unwrap());
    }
}
```

它将运行——一段时间——并打印出以下内容：

```rs
6553600
6553600
6553600
6553600
6553600
```

但如果每个读者不需要看到每个写操作，也就是说，只要读者没有错过所有写操作，那么错过写操作是可以接受的，我们就有选择了。让我们看看其中一个。

# 阻塞直到条件改变 —— 条件变量

一个选项是条件变量，或称 CONDition VARiable。条件变量是一种巧妙的方式来阻塞一个线程，直到某个布尔条件发生变化。一个困难是条件变量仅与互斥锁相关联，但在这个例子中，我们并不太在意这一点。

条件变量工作的方式是，在获取互斥锁之后，你将 `MutexGuard` 传递给 `Condvar::wait`，这将阻塞线程。其他线程可能通过这个过程，阻塞在相同的条件下。某个其他线程将获取相同的独占锁，并最终在条件变量上调用 `notify_one` 或 `notify_all`。第一个唤醒单个线程，第二个唤醒 *所有* 线程。条件变量可能会发生虚假唤醒，这意味着线程可能在没有收到通知的情况下离开其阻塞状态。因此，条件变量会在循环中检查其条件。但是，一旦条件变量唤醒，你 *确实* 保证持有互斥锁，这防止了虚假唤醒时的死锁。

让我们将我们的例子修改为使用`condvar`。实际上，这个例子中有很多内容，所以我们将它分解成几个部分：

```rs
use std::thread;
use std::sync::{Arc, Condvar, Mutex};

fn main() {
    let total_readers = 5;
    let mutcond: Arc<(Mutex<(bool, u16)>, Condvar)> =
        Arc::new((Mutex::new((false, 0)), Condvar::new()));
```

我们的序言与前面的例子相似，但设置已经相当奇怪。我们正在同步`mutcond`上的线程，它是一个`Arc<(Mutex<(bool, u16)>, Condvar)>`。Rust 的`Condvar`有点尴尬。将`Condvar`与多个互斥量关联是未定义的行为，但`Condvar`的类型中并没有使这成为不变量的东西。我们只需记住保持它们关联。为此，在元组中将`Mutex`和`Condvar`配对是很常见的，就像这里一样。现在，为什么是`Mutex<(bool, u16)>`？元组的第二个元素是我们的*资源*，这在其他例子中是常见的。第一个元素是一个布尔标志，我们用它作为信号表示有可用的写入。以下是我们的读取线程：

```rs
    let mut reader_jhs = Vec::with_capacity(total_readers);
    for _ in 0..total_readers {
        let mutcond = Arc::clone(&mutcond);
        reader_jhs.push(thread::spawn(move || {
            let mut total_zeros = 0;
            let mut total_wakes = 0;
            let &(ref mtx, ref cnd) = &*mutcond;

            while total_zeros < 100 {
                let mut guard = mtx.lock().unwrap();
                while !guard.0 {
                    guard = cnd.wait(guard).unwrap();
                }
                guard.0 = false;

                total_wakes += 1;
                if guard.1 == 0 {
                    total_zeros += 1;
                }
            }
            total_wakes
        }));
    }
```

直到`total_zeros`达到 100，读取线程会锁定互斥量，检查互斥量内的守卫以检查写入可用性，如果没有写入，就会在`condvar`上等待，放弃锁。然后读取线程会阻塞，直到调用`notify_all`——我们很快就会看到。每个读取线程都会争先恐后地重新获取锁。幸运的获胜者会注意到没有更多的写入要读取，然后执行我们在前面的例子中看到的正常流程。需要重复的是，每个从条件等待中唤醒的线程都在争先恐后地第一个重新获取互斥量。我们的读取者不合作，它会立即阻止其他读取线程发现资源可用。然而，它们仍然会意外地醒来并被迫再次等待。也许。读取线程还在与写入线程竞争获取锁。让我们看看写入线程：

```rs
    let _ = thread::spawn(move || {
        let &(ref mtx, ref cnd) = &*mutcond;
        loop {
            let mut guard = mtx.lock().unwrap();
            guard.1 = guard.1.wrapping_add(1);
            guard.0 = true;
            cnd.notify_all();
        }
    });
```

写入线程是一个无限循环，我们在一个未连接的线程中将其孤儿化。现在，完全有可能写入线程会获取锁，增加资源，通知等待的读取线程，放弃锁，然后在任何读取线程被调度之前立即重新获取锁以开始 while 过程。这意味着在读取线程足够幸运地注意到之前，资源为零的情况完全可能发生多次。让我们结束这个程序：

```rs
    for jh in reader_jhs {
        println!("{:?}", jh.join().unwrap());
    }
}
```

理想情况下，我们希望有一种双向性——我们希望写入者发出有读取的信号，而读取者发出有容量的信号。这很可疑地类似于上一章中的 Ring 通过其大小变量工作，当我们小心不要在该变量上竞争时。例如，我们可以在混合中添加另一个条件变量，这个变量是给写入者的，但这里并不是这样，程序因此受到影响。以下是其中一个运行实例：

```rs
7243473
6890156
6018468
6775609
6192116
```

呼吁！这比之前的例子中的循环要多得多。这并不是说条件变量很难使用——它们并不难——只是它们需要与其他原语一起使用。我们将在本章后面看到一个很好的例子。

# 阻塞直到全员到齐——障碍

障碍是一种同步设备，它会阻塞线程，直到预定义数量的线程在同一个障碍上等待。当障碍等待的线程醒来时，会宣布一个领导者——可以通过检查`BarrierWaitResult`来发现——但这并不提供调度优势。当您希望延迟线程在某个资源的不安全初始化之后时，障碍变得有用——比如一个在启动时没有线程安全性的 C 库的内部结构，或者需要强制参与线程在大约相同的时间开始临界区。后者是更广泛的类别，根据作者的实践经验。当使用原子变量编程时，您会遇到障碍有用的场景。此外，考虑一下为低功耗设备编写多线程代码。如今，在电源管理方面有两种可能的策略：将 CPU 扩展以满足要求，实时调整程序的运行时间，或者尽可能快地烧毁程序的一部分，然后关闭芯片。在后一种方法中，障碍正是您需要的原语。

# 更多互斥锁、条件变量和类似功能正在行动

承认，前几节的例子有点复杂。这种疯狂的背后有原因，我保证，但在我们继续之前，我想给你展示一些来自迷人的《信号量小书》的工作示例。如果你跳过了之前的文献注释，这本书是适合自学的一组并发谜题，因为这些谜题很有趣，并且附带很好的提示。正如标题所暗示的，这本书确实使用了信号量原语，而 Rust 没有。尽管如此，正如前一章提到的，我们将在下一章构建一个信号量。

# 火箭准备问题

这个谜题实际上并没有出现在《信号量小书》中，但它基于那里的一则谜题——4.5 节中的吸烟者问题。我个人认为香烟很恶心，所以我们稍微改一下措辞。想法是相同的。

我们总共有四个线程。一个线程是“生产者”，随机发布“燃料”、“氧化剂”或“宇航员”中的一个。剩下的三个线程是消费者，或者说是“火箭”，它们必须按照之前列出的顺序获取资源。如果火箭没有按照这个顺序获取资源，那么准备火箭是不安全的，而且如果没有全部三个资源，火箭就不能起飞。此外，一旦所有火箭都准备好了，我们想要开始 10 秒倒计时，只有在这之后火箭才能起飞。

我们解决方案的序言比平常要长一些，目的是为了将解决方案作为一个独立的单元：

```rs
use std::{thread, time};
use std::sync::{Arc, Barrier, Condvar, Mutex};

// NOTE if this were a crate, rather than a stand-alone
// program, we could and _should_ use the XorShiftRng
// from the 'rand' crate.
pub struct XorShiftRng {
    state: u32,
}

impl XorShiftRng {
    fn with_seed(seed: u32) -> XorShiftRng {
        XorShiftRng { state: seed }
    }

    fn next_u32(&mut self) -> u32 {
        self.state ^= self.state << 13;
        self.state ^= self.state >> 17;
        self.state ^= self.state << 5;
        return self.state;
    }
}
```

对于这个解决方案，我们并不真的需要出色的随机性——操作系统调度器已经注入了足够的随机性——但只需要一点小小的随机性。`XorShift` 就能满足这个要求。现在，对于我们的资源：

```rs
struct Resources {
    lock: Mutex<(bool, bool, bool)>,
    fuel: Condvar,
    oxidizer: Condvar,
    astronauts: Condvar,
}
```

结构体被一个 `Mutex<(bool, bool, bool)>` 保护，布尔值是一个标志，用来指示是否有资源可用。我们持有第一个标志表示 `燃料`，第二个 `氧化剂`，第三个 `宇航员`。结构体的其余部分是匹配每个这些资源关注的条件变量。`生产者` 是一个简单的无限循环：

```rs
fn producer(resources: Arc<Resources>) {
    let mut rng = XorShiftRng::with_seed(2005);
    loop {
        let mut guard = resources.lock.lock().unwrap();
        (*guard).0 = false;
        (*guard).1 = false;
        (*guard).2 = false;
        match rng.next_u32() % 3 {
            0 => {
                (*guard).0 = true;
                resources.fuel.notify_all()
            }
            1 => {
                (*guard).1 = true;
                resources.oxidizer.notify_all()
            }
            2 => {
                (*guard).2 = true;
                resources.astronauts.notify_all()
            }
            _ => unreachable!(),
        }
    }
}
```

在每次迭代中，生产者选择一个新的资源——`rng.next_u32() % 3`——并为该资源设置布尔标志，然后通知所有等待 `燃料` 条件变量的线程。同时，编译器和 CPU 可以自由地重新排序指令，而内存 `notify_all` 则像是一个因果门；代码中的所有内容在因果上都是先于之后的，同样，之后的也是如此。如果资源布尔翻转在通知之后，那么从等待线程的角度来看，唤醒将是虚假的，并且从生产者的角度来看，它将丢失。`火箭` 是简单的：

```rs
fn rocket(name: String, resources: Arc<Resources>, 
          all_go: Arc<Barrier>, lift_off: Arc<Barrier>) {
    {
        let mut guard = resources.lock.lock().unwrap();
        while !(*guard).0 {
            guard = resources.fuel.wait(guard).unwrap();
        }
        (*guard).0 = false;
        println!("{:<6} ACQUIRE FUEL", name);
    }
    {
        let mut guard = resources.lock.lock().unwrap();
        while !(*guard).1 {
            guard = resources.oxidizer.wait(guard).unwrap();
        }
        (*guard).1 = false;
        println!("{:<6} ACQUIRE OXIDIZER", name);
    }
    {
        let mut guard = resources.lock.lock().unwrap();
        while !(*guard).2 {
            guard = resources.astronauts.wait(guard).unwrap();
        }
        (*guard).2 = false;
        println!("{:<6} ACQUIRE ASTRONAUTS", name);
    }

    all_go.wait();
    lift_off.wait();
    println!("{:<6} LIFT OFF", name);
}
```

每个线程，就资源需求而言，都会等待生产者使其可用。正如讨论的那样，发生了一场争夺重新获取互斥锁的竞争，只有一个线程能够获得资源。最后，一旦所有资源都获得，就会遇到 `all_go` 障碍，以延迟任何在倒计时之前的线程。这里我们需要 `main` 函数：

```rs
fn main() {
    let all_go = Arc::new(Barrier::new(4));
    let lift_off = Arc::new(Barrier::new(4));
    let resources = Arc::new(Resources {
        lock: Mutex::new((false, false, false)),
        fuel: Condvar::new(),
        oxidizer: Condvar::new(),
        astronauts: Condvar::new(),
    });

    let mut rockets = Vec::new();
    for name in &["KSC", "VAB", "WSMR"] {
        let all_go = Arc::clone(&all_go);
        let lift_off = Arc::clone(&lift_off);
        let resources = Arc::clone(&resources);
        rockets.push(thread::spawn(move || {
            rocket(name.to_string(), resources, 
                   all_go, lift_off)
        }));
    }

    thread::spawn(move || {
        producer(resources);
    });

    all_go.wait();
    let one_second = time::Duration::from_millis(1_000);
    println!("T-11");
    for i in 0..10 {
        println!("{:>4}", 10 - i);
        thread::sleep(one_second);
    }
    lift_off.wait();

    for jh in rockets {
        jh.join().unwrap();
    }
}
```

注意，大致来说，函数的前半部分由障碍和资源或火箭线程组成。`all_go.wait()` 是有趣的地方。这个主线程已经生成了所有子线程，现在正阻塞在来自火箭线程的 all-go 信号上，这意味着它们已经收集了资源，并且也阻塞在同一个障碍上。完成这些后，发生倒计时，为解决方案增添一点风采；同时，火箭线程已经开始在 `lift_off` 障碍上等待。顺便提一下，值得注意的是，生产者仍在生产，消耗 CPU 和电力。一旦倒计时完成，火箭线程被释放，主线程与它们连接，允许它们打印告别信息，程序结束。输出将会有所不同，但这里有一个代表性的例子：

```rs
KSC    ACQUIRE FUEL
WSMR   ACQUIRE FUEL
WSMR   ACQUIRE OXIDIZER
VAB    ACQUIRE FUEL
WSMR   ACQUIRE ASTRONAUTS
KSC    ACQUIRE OXIDIZER
KSC    ACQUIRE ASTRONAUTS
VAB    ACQUIRE OXIDIZER
VAB    ACQUIRE ASTRONAUTS
T-11
  10
   9
   8
   7
   6
   5
   4
   3
   2
   1
VAB    LIFT OFF
WSMR   LIFT OFF
KSC    LIFT OFF
```

# 绳索桥问题

这个谜题出现在 *《信号量小书》* 中的 *黑猩猩过河问题*，第 6.3 节。Downey 指出，它是从 Tanenbaum 的 *操作系统：设计与实现* 中改编的，所以你在这里得到的是第三手资料。问题描述如下：

在南非克鲁格国家公园的某个地方有一个深峡谷，峡谷上只有一根横跨的绳子。狒狒可以通过手拉手的方式在绳子上荡过峡谷，但如果两只朝相反方向行进的狒狒在中间相遇，它们将会打斗并掉入峡谷死亡。此外，这根绳子只能承受 5 只狒狒的重量。如果有更多的狒狒同时站在绳子上，它将会断裂。假设我们可以教会狒狒使用信号量，我们希望设计一个具有以下特性的同步方案：

+   *一旦狒狒开始过桥，就保证它能到达对岸而不会遇到朝相反方向行进的狒狒。*

+   *绳子上永远不会超过 5 只狒狒。*"

我们的解决方案不使用信号量。相反，我们依赖类型系统来提供保证，依赖它来确保左行狒狒永远不会遇到右行狒狒。让我们深入探讨：

```rs
use std::thread;
use std::sync::{Arc, Mutex};

#[derive(Debug)]
enum Bridge {
    Empty,
    Left(u8),
    Right(u8),
}
```

我们在这里看到了通常的序言，然后是一个类型，`Bridge`。`Bridge`是问题陈述中绳桥的模型，可以是空的，也可以有左行狒狒或右行狒狒在上面；我们不需要通过标志和推断状态来调整，因为我们可以直接将其编码到类型中。事实上，依赖类型系统，我们的同步非常简单：

```rs
fn main() {
    let rope = Arc::new(Mutex::new(Bridge::Empty));
```

只需要一个互斥锁。我们将桥的每一侧表示为一个线程，每一侧都试图让自己的狒狒过桥，但合作地允许狒狒到达其一侧。这里是桥的左侧：

```rs
    let lhs_rope = Arc::clone(&rope);
    let lhs = thread::spawn(move || {
        let rope = lhs_rope;
        loop {
            let mut guard = rope.lock().unwrap();
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
                    *guard = Bridge::Left(i - 1);
                }
            }
        }
    });
```

当桥的左侧发现桥本身是空的，就会派一只右行狒狒上路。进一步地，当左侧发现桥上已经有右行狒狒，且绳子的容量五只尚未达到，就会再派一只狒狒上路。左行狒狒从绳子上被接走，绳子上狒狒的总数减少。这里的特殊情况是`Bridge::Left(0)`的条款。尽管桥上还没有狒狒，但从技术上讲，如果桥的右侧在左侧之前被调度，它就会派一只狒狒上路，正如我们很快就会看到的。当然，我们可以使移除狒狒的操作更加积极，并在没有旅行者时立即将桥设置为`Bridge::Empty`。以下是桥的右侧，与左侧类似：

```rs
    let rhs = thread::spawn(move || loop {
        let mut guard = rope.lock().unwrap();
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
                *guard = Bridge::Right(i - 1);
            }
        }
    });

    rhs.join().unwrap();
    lhs.join().unwrap();
}
```

现在，这是公平的吗？这取决于你的系统互斥锁。一些系统提供了公平的互斥锁，即如果一个线程反复获取锁并饿死尝试锁定相同原语的其他线程，那么贪婪的线程将在其他线程之下被降级。这种公平性在锁定时可能会或可能不会产生额外的开销，具体取决于实现。实际上，你是否想要公平性，很大程度上取决于你试图解决的问题。如果我们只对尽可能快地将狒狒从绳索桥上运过去感兴趣——最大化吞吐量——那么它们来自哪个方向实际上并不重要。原始问题陈述没有提到公平性，它通过允许狒狒流是无限的来某种程度上打乱了这一点。考虑如果桥的两侧的狒狒是有限的，而我们想减少任何单个狒狒过桥到另一侧所需的时间，以最小化延迟会发生什么。我们目前的解决方案，适应有限流，在这方面相当差。通过偶尔让步、分层更积极的绳索打包、有意后退或其他众多策略，可以改善这种情况。一些平台允许你动态地改变线程相对于锁的优先级，但 Rust 的标准库并不提供这一点。

或者，你知道的，我们可以使用两个有界 MPSC 通道。这是一个选择。一切取决于你试图完成的事情。

最终，在安全的 Rust 中使用的工具对于在现有数据结构上执行计算非常有用，而无需涉及任何奇怪的业务。如果你是这样的游戏，你很可能不需要深入到`Mutex`和`Condvar`之外，可能还需要到`RwLock`和`Barrier`。但是，如果你正在构建由指针组成的结构，你将不得不深入研究我们在 Ring 中看到的那些奇怪的业务，以及它带来的所有危险。

# Hopper——一个 MPSC 专业化

如上章末所述，你需要一个相当专业的用例来考虑不使用 stdlib 的 MPSC。在本章的其余部分，我们将讨论这样一个用例以及旨在填补这一空白的库的实现。

# 问题

回想一下上一章，其中 telem 中的角色线程通过 MPSC 通道相互通信。还要记住，telem 是 cernan ([`crates.io/crates/cernan`](https://crates.io/crates/cernan))项目的快速版本，它基本上扮演了相同的角色，但覆盖了更多的入口协议、出口协议，并且边缘更加锋利。cernan 的一个关键设计目标是，如果它接收了你的数据，它至少会将它传递到下游一次。这意味着，为了支持入口协议，cernan 必须在其配置的路由拓扑的全长中知道，有足够的空间来接受一个新事件，无论它是一段遥测数据、原始字节数据缓冲区还是日志行。现在，这是可能的，正如我们所看到的，通过使用`SyncSender`。技巧在于与 cernan 的第二个设计目标相结合——非常低、可配置的资源消耗。您的作者看到 cernan 被部署到高可用性集群中，一个机器专门用于运行 cernan，以及 cernan 与应用程序服务器一起运行或作为 k8s 集群中 daemonset 的一部分。在前一种情况下，cernan 可以被配置为消耗机器的所有资源。在后两种情况下，需要考虑给 cernan 提供足够多的资源来完成其任务，相对于遥测系统的预期输入。

在现代系统中，通常有大量的磁盘空间和有限的——相对而言——RAM。使用`SyncSender`可能需要相对较少的可摄入事件或 cernan 可能消耗大量内存。这些情况通常发生在出口协议失败时，可能是因为远端系统关闭。如果 cernan 因为一个松散耦合系统中的故障而挤占了应用程序服务器，从而加剧了部分系统故障，那么，那将是非常糟糕的。

对于许多用户来说，在这样部分中断期间完全失去视线也是不可接受的。对于`SyncSender`来说，这是一个棘手的约束。实际上足够棘手，以至于我们决定编写我们自己的 MPSC，称为 hopper ([`crates.io/crates/hopper`](https://crates.io/crates/hopper))。hopper 允许最终用户配置队列的总内存消耗，以字节为单位。这并不远，通过一些计算就可以达到`SyncSender`的水平。hopper 的特殊之处在于它可以分页到磁盘。

Hopper 在内存限制非常严格但又不希望丢弃事件的情况下表现突出。或者，你希望尽可能地将丢弃事件推迟。与内存队列类似，hopper 允许最终用户配置磁盘上可消耗的最大字节数。单个 hopper 队列在被迫丢弃事件之前，最多可以持有`in_memory_bytes + on_disk_bytes`字节；我们将在下面直接看到其确切机制。所有这些编程都是为了实现最大吞吐量和避免阻塞没有工作的线程，以节省 CPU 时间。

# Hopper 的使用

在我们深入研究 hopper 的实现之前，看看它在实际中的应用将是有益的。让我们调整我们环形程序系列的最后一迭代。为了参考，这里看起来是这样的：

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

hopper 版本略有不同。这是前言：

```rs
extern crate hopper;
extern crate tempdir;

use std::{mem, thread};
```

没有什么意外的。将作为`writer`线程的功能是：

```rs
fn writer(mut chan: hopper::Sender<u32>) -> () {
    let mut cur: u32 = 0;
    while let Ok(()) = chan.send(cur) {
        cur = cur.wrapping_add(1);
    }
}
```

除了从`mpsc::SyncSender<u32>`切换到`hopper::Sender<u32>`——所有 hopper 发送者都是隐式有界的外——唯一的另一个区别是`chan`必须是可变的。现在，对于`reader`线程：

```rs
fn reader(read_limit: usize, 
          mut chan: hopper::Receiver<u32>) -> () {
    let mut cur: u32 = 0;
    let mut iter = chan.iter();
    while (cur as usize) < read_limit {
        let num = iter.next().unwrap();
        assert_eq!(num, cur);
        cur = cur.wrapping_add(1);
    }
}
```

这里还有更多的事情要做。在 hopper 中，接收者被设计成用作迭代器，这在某种程度上有些尴尬，因为我们想限制接收到的值的总数。迭代器在调用`next`时会阻塞，永远不会返回`None`。然而，hopper 中发送者的`send`与 MPSC 的非常不同——`hopper::Sender<T>::send(&mut self, event: T) -> Result<(), (T, Error)>`。先忽略`Error`，为什么在包含`T`的错误条件下返回一个元组？为了清楚起见，它是指传递进来的同一个`T`。当调用者将`T`发送到 hopper 时，它的所有权也传递给了 hopper，这在出错的情况下是个问题。调用者可能非常希望将`T`拿回来以避免其丢失。Hopper 想要避免对每个通过`T`都进行克隆，因此 hopper 偷偷地将`T`的所有权带回到调用者那里。那么`Error`呢？它是一个简单的枚举：

```rs
pub enum Error {
    NoSuchDirectory,
    IoError(Error),
    NoFlush,
    Full,
}
```

重要的是`Error::Full`。这是在发送时内存和基于磁盘的缓冲区都满了的情况。这个错误是可以恢复的，但只有调用者才能确定。现在，最后，我们的 hopper 示例的`main`函数：

```rs
fn main() {
    let read_limit = 1_000_000;
    let in_memory_capacity = mem::size_of::<u32>() * 10;
    let on_disk_capacity = mem::size_of::<u32>() * 100_000;

    let dir = tempdir::TempDir::new("queue_root").unwrap();
    let (snd, rcv) = hopper::channel_with_explicit_capacity::<u32>(
        "example",
        dir.path(),
        in_memory_capacity,
        on_disk_capacity,
        1,
    ).unwrap();

    let reader_jh = thread::spawn(move || {
        reader(read_limit, rcv);
    });
    let _writer_jh = thread::spawn(move || {
        writer(snd);
    });

    reader_jh.join().unwrap();
}
```

`read_limit`仍然在位，但最大的不同是通道的创建。首先，hopper 必须有一个地方来存放其磁盘存储。在这里，我们正在推迟到某个临时目录——`let dir = tempdir::TempDir::new("queue_root").unwrap();`——以避免在运行示例后进行清理。在实际使用中，磁盘位置会被仔细选择。`hopper::channel_with_explicit_capacity`创建的发送者与`hopper::channel`相同，除了所有配置旋钮都对调用者开放。第一个参数是通道的*名称*。这个值很重要，因为它将被用来在`dir`下创建一个用于磁盘存储的目录。通道名称必须是唯一的。`in_memory_capacity`和`on_disk_capacity`都是以字节为单位，这就是为什么我们使用了我们那位老朋友`mem::size_of`。现在，最后一个配置选项是什么，设置为`1`？那是*队列文件*的最大数量。

Hopper 的磁盘存储被分成多个*队列文件*，每个文件的大小为`on_disk_capacity`。发送者仔细协调他们的写入以避免队列文件过载，接收者负责在确定没有更多写入到来后销毁它们——我们将在本章后面讨论信号机制。使用队列文件允许 hopper 可能回收它可能无法通过使用一个大型文件以循环方式使用的情况下回收的磁盘空间。这确实会增加发送者和接收者代码的复杂性，但为了提供一个资源消耗更少的库，这是值得的。

# hopper 的概念视图

在我们深入研究实现之前，让我们从概念层面了解一下之前的例子是如何工作的。我们有足够的内存空间来存储 10 个 u32。如果写入线程比读取线程快得多——或者获得更多的调度时间——我们可能会遇到这种情况：

```rs
write: 99

in-memory-buffer: [90, 91, 92, 93, 94, 95, 96, 97, 98]
disk-buffer: [87, 88, 89]
```

也就是说，当内存缓冲区完全满并且 diskbuffer 中有三个排队写入时，写入`99`正在进入系统。虽然世界状态可能在写入进入系统排队和排队之间发生变化，但让我们假设在排队期间没有接收者拉取项目。结果将是一个看起来像这样的系统：

```rs
in-memory-buffer: [90, 91, 92, 93, 94, 95, 96, 97, 98]
disk-buffer: [87, 88, 89, 99]
```

接收者总共必须从 diskbuffer 中读取三个元素，然后是所有内存中的元素，最后是从 diskbuffer 中读取一个元素，以保持队列的顺序。考虑到写入可能被接收者的一个或多个读取操作分割，这进一步增加了复杂性。我们在上一章中看到了类似的情况，即通过执行加载和检查来保护写入——满足检查的条件可能在加载、检查和操作之间发生变化。还有一个更复杂的因素；之前显示的统一磁盘缓冲区实际上并不存在。相反，可能存在许多单独的队列文件。

假设 hopper 已经被配置为允许 10 个`u32`的内存中配置，正如之前提到的，以及 10 个在磁盘上，但分布在五个可能的队列文件中。我们的修改后的写入系统如下：

```rs
in-memory-buffer: [90, 91, 92, 93, 94, 95, 96, 97, 98]
disk-buffer: [ [87, 88], [89, 99] ]
```

发送者负责创建队列文件并将它们填满到最大容量。`Receiver`负责从队列文件中读取，并在文件耗尽时删除该文件。确定队列文件耗尽机制的简单方法是：当`Sender`退出一个队列文件时，它将创建一个新的队列文件，并在文件系统中将旧路径标记为只读。当`Receiver`尝试从磁盘读取字节并发现没有字节时，它会检查文件的写入状态。如果文件仍然是读写状态，最终会有更多的字节到来。如果文件是只读的，则文件已耗尽。这里还有一些小技巧，以及`Sender`和`Receiver`之间未解释的进一步协作，但这应该足以让我们有效地深入研究。

# 双端队列（deque）

大致来说，hopper 是一个具有两个协作的有限状态机的并发双端队列。我们将从`src/deque.rs`中定义的双端队列开始。

下面的讨论列出了几乎所有的 hopper 源代码。我们将指出读者需要参考书中源代码仓库中的几个实例。

为了完全清楚，双端队列是一种数据结构，允许在队列的两端进行入队和出队操作。Rust 的`stdlib`有`VecDeque<T>`，这非常有用。然而，hopper 无法使用它，因为其设计目标之一是允许对 hopper 队列进行并行发送和接收，而`VecDeque`不是线程安全的。此外，尽管在 crate 生态系统中存在并发双端队列实现，但 hopper 双端队列与其支持的有限状态机以及 hopper 的内部所有权模型紧密相关。也就是说，你可能无法在不做任何修改的情况下在自己的项目中使用 hopper 的双端队列。无论如何，这里是一些前置说明：

```rs
use std::sync::{Condvar, Mutex, MutexGuard};
use std::sync::atomic::{AtomicUsize, Ordering};
use std::{mem, sync};

unsafe impl<T, S> Send for Queue<T, S> {}
unsafe impl<T, S> Sync for Queue<T, S> {}
```

这里唯一不熟悉的部分是从`std::sync::atomic`导入的内容。我们将在下一章更详细地介绍原子操作，但我们将根据需要以高级别概述它们。请注意，对于某些尚不为人知的类型`Queue<T, S>`，`send`和`sync`的声明是不安全的。我们即将进入非标准领域：

```rs
pub struct Queue<T, S> {
    inner: sync::Arc<InnerQueue<T, S>>,
}
```

`Queue<T, S>`的定义与我们之前章节中看到的是相似的：一个简单的结构封装了一个内部结构，这里称为`InnerQueue<T, S>`。`InnerQueue`被封装在一个`Arc`中，这意味着堆上只有一个分配的`InnerQueue`。正如你所期望的，`Queue`的克隆是将`Arc`复制到一个新的结构体中：

```rs
impl<T, S> Clone for Queue<T, S> {
    fn clone(&self) -> Queue<T, S> {
        Queue {
            inner: sync::Arc::clone(&self.inner),
        }
    }
}
```

重要的是，每个与`Queue`交互的线程都能看到相同的`InnerQueue`。否则，线程正在处理不同的内存区域，彼此之间没有关系。让我们看看`InnerQueue`：

```rs
struct InnerQueue<T, S> {
    capacity: usize,
    data: *mut Option<T>,
    size: AtomicUsize,
    back_lock: Mutex<BackGuardInner<S>>,
    front_lock: Mutex<FrontGuardInner>,
    not_empty: Condvar,
}
```

好吧，这更接近我们在 Rust 本身看到的内部结构，而且有很多事情在进行中。字段 `capacity` 是 `InnerQueue` 将在数据中保留的最大元素数量。就像前一章中的第一个 `Ring` 一样，`InnerQueue` 使用连续的内存块来存储其 `T` 元素，通过将 `Vec` 拆分来获得这个连续块。同样，就像第一个 `Ring` 一样，我们在连续的内存块中存储 `Option<T>` 元素。技术上，我们可以处理原始指针的连续块或复制内存块。但使用 `Option<T>` 简化了代码，无论是插入还是删除，但每个元素需要牺牲一个字节的内存。这种额外的复杂性对于 hopper 尝试达到的性能目标来说并不值得。

`size` 字段是一个原子 `usize`。原子将在下一章中详细介绍，但 `size` 的行为将非常重要。现在，将其视为线程之间非常小的同步机制；一个小的钩子，它将允许我们按顺序进行内存访问，同时也像 `usize` 一样操作。`not_empty` condvar 用于向任何等待新元素弹出的潜在读者发出信号，实际上确实有元素可以弹出。condvar 的使用大大减少了 hopper 的 CPU 负载，而没有牺牲对忙循环的延迟。现在，`back_lock` 和 `front_lock`。这里发生了什么？双端队列的每一侧都由互斥锁保护，这意味着一次只能有一个入队者和一个出队者，但它们可以相互并行运行。以下是互斥锁的两个内部值的定义：

```rs
#[derive(Debug, Clone, Copy)]
pub struct FrontGuardInner {
    offset: isize,
}

#[derive(Debug)]
pub struct BackGuardInner<S> {
    offset: isize,
    pub inner: S,
}
```

`FrontGuardInner` 是两个中较容易解释的。唯一的字段是 `offset`，它定义了从 `InnerGuard` 数据的第一个指针到操作队列前端的线程的偏移量。这种连续存储也以环形缓冲区的方式使用。在 `BackGuardInner` 中，我们看到相同的偏移量，但还有一个额外的 `inner`，`S`。这是什么？正如我们将看到的，操作缓冲区后端的线程需要它们之间的额外协调。这究竟是什么，队列并不关心。因此，我们将其作为类型参数，并允许调用者整理一切，同时小心地按需传递数据。以这种方式，队列通过自身传递状态，但不需要检查它。

让我们从 `InnerQueue` 的实现开始：

```rs
impl<T, S> InnerQueue<T, S>
where
    S: ::std::default::Default,
{
    pub fn with_capacity(capacity: usize) -> InnerQueue<T, S> {
        assert!(capacity > 0);
        let mut data: Vec<Option<T>> = Vec::with_capacity(capacity);
        for _ in 0..capacity {
            data.push(None);
        }
        let raw_data = (&mut data).as_mut_ptr();
        mem::forget(data);
        InnerQueue {
            capacity: capacity,
            data: raw_data,
            size: AtomicUsize::new(0),
            back_lock: Mutex::new(BackGuardInner {
                offset: 0,
                inner: S::default(),
            }),
            front_lock: Mutex::new(FrontGuardInner { offset: 0 }),
            not_empty: Condvar::new(),
        }
    }
```

该类型缺少一个 `new() -> InnerQueue`，因为没有必要调用它。相反，只有 `with_capacity`，这与我们之前看到的 `Ring` 的 `with_capacity` 非常相似——分配了一个向量，并将其拆分为原始指针，然后在指针加载到新创建的结构体之前，原始引用被遗忘。

类型 `S` 必须实现一个默认初始化，该初始化足够充分，因为调用者的走私状态总是相同的值，这可以充分定义为默认值。如果这个去队列打算用于通用用途，我们可能需要提供一个 `with_capacity`，它也直接接受 `S`。现在，我们快速浏览一下实现中的几个其他函数：

```rs
    pub fn capacity(&self) -> usize {
        self.capacity
    }

    pub fn size(&self) -> usize {
        self.size.load(Ordering::Relaxed)
    }

    pub fn lock_back(&self) -> MutexGuard<BackGuardInner<S>> {
        self.back_lock.lock().expect("back lock poisoned")
    }

    pub fn lock_front(&self) -> MutexGuard<FrontGuardInner> {
        self.front_lock.lock().expect("front lock poisoned")
    }
```

下一个函数 `push_back` 非常重要且微妙：

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

`InnerQueue::push_back` 负责将 `T` 放置在环形缓冲区的当前尾部，或者在容量不足时发出信号表示缓冲区已满。当我们讨论环形缓冲区时，我们注意到 `size == capacity` 检查是一个竞争条件。但在 `InnerQueue` 中不是这样，多亏了大小的原子性。`self.size.load(Ordering::Acquire)` 执行对 `self.size` 的内存加载，但确信它是唯一一个可以将 `self.size` 作为可操作值的线程。线程中的后续内存操作将在 `Acquire` 之后排序，至少直到发生 `Ordering::Release` 的存储操作。这种类型的存储操作就在几行之后发生——`self.size.fetch_add(1, Ordering::Release)`。在这两点之间，我们看到元素 `T` 被加载到缓冲区中——在之前检查确保我们不会覆盖一个 `Some` 值——并且对 `BackGuardInner` 的偏移量进行包装增加。就像在上一章中一样。这个实现的不同之处在于返回 `Ok(must_wake_dequeuers)`。因为内部 `S` 被保护，队列无法知道在可以放弃互斥锁之前是否需要做更多的工作。因此，队列本身无法发出有值可读的信号，尽管在函数返回时该值已经被写入内存。调用者必须运行通知。这是一个尖锐的边缘。如果调用者忘记通知在 condvar 上阻塞的线程，那么阻塞的线程将永远保持这种状态。

`InnerQueue::push_front` 比较简单，并且与 `push_back` 没有太大差异：

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
}
```

弹出前端的线程，因为它不需要协调，能够自己获取前端锁，因为没有需要通过线程传递的状态。在获取锁后，线程进入一个条件检查循环，以防止在 `not_empty` 上的虚假唤醒。当线程能够唤醒时，用 `None` 替换偏移量处的项目。通常的偏移量维护也会发生。

与内部结构相比，`Queue<T, S>` 的实现相当简单。以下是 `push_back` 的实现：

```rs
pub fn push_back(
    &self,
    elem: T,
    mut guard: &mut MutexGuard<BackGuardInner<S>>,
) -> Result<bool, Error<T>> {
    unsafe { (*self.inner).push_back(elem, &mut guard) }
}
```

`Queue` 中唯一实质上新的功能是 `notify_not_empty(&self, _guard: &MutexGuard<FrontGuardInner>) -> ()`。调用者负责在 `push_back` 信号指示需要通知去队列时调用此功能，并且尽管守卫未使用，但通过要求传入守卫，该库的一个粗糙边缘得到了平滑处理，证明它被持有。

这就是跳转器核心的双端队列。这个结构非常难以正确实现。任何对原子大小相对于其他内存操作的加载和存储的轻微重新排序都会引入错误，例如在没有协调的情况下对同一区域内存的并行访问。这些错误非常微妙，不会立即显现。在 x86 和 ARM 处理器上广泛使用 helgrind 加 quickcheck 测试——关于这一点稍后还会详细介绍——将有助于提高对实现的信心。测试运行数小时并不罕见，可以发现那些不是确定性的但可以通过足够的时间和重复示例进行推理的错误。从非常原始的组件构建并发数据结构是*困难的*。

# 接收器

既然我们已经涵盖了双端队列，让我们跳转到定义在 `src/receiver.rs` 中的 `Receiver`。如前所述，接收器负责以迭代器的风格从内存或磁盘中拉取值。`Receiver` 声明将自身转换为迭代器的通常机制，我们在这本书中不会涉及这一点，主要是因为它有点繁琐。话虽如此，`Receiver` 中有两个重要的函数需要介绍，而且它们都不在公共接口中。第一个是 `Receiver::next_value`，由接收器的迭代器版本调用。此函数定义如下：

```rs
fn next_value(&mut self) -> Option<T> {
    loop {
        if self.disk_writes_to_read == 0 {
            match self.mem_buffer.pop_front() {
                private::Placement::Memory(ev) => {
                    return Some(ev);
                }
                private::Placement::Disk(sz) => {
                    self.disk_writes_to_read = sz;
                    continue;
                }
            }
        } else {
            match self.read_disk_value() {
                Ok(ev) => return Some(ev),
                Err(_) => return None,
            }
        }
    }
}
```

定义为移动 `T` 值的跳转器并不定义一个在 `T`s 上的双端队列——如前文所述——而是实际上它持有 `Placement<T>`。`Placement` 在 `src/private.rs` 中定义，是一个小的枚举：

```rs
pub enum Placement<T> {
    Memory(T),
    Disk(usize),
}
```

这就是使跳转器工作的小技巧。并发数据结构的主要挑战在于在线程之间提供足够的同步，以便您的结果可以在调度混乱的情况下保持一致性，但又不要求与顺序解决方案相比同步过多。

这种情况确实会发生。

现在，回想一下，保持顺序是任何通道式队列的关键设计需求。Rust 标准库的 MPSC 通过原子标志实现这一点，得益于最终只有一个地方可以存储插入的元素，以及一个地方可以从中移除。但在跳转器中并非如此。但是，这个一站式商店是一个非常有用且低同步的方法。这就是 `Placement` 发挥作用的地方。当遇到 `Memory` 变体时，`T` 已经存在，接收器只需返回它。当返回 `Disk(usize)` 时，这向接收器发送一个信号，使其切换到*磁盘模式*。在磁盘模式下，当 `self.disk_writes_to_read` 不为零时，接收器优先从磁盘读取一个值。只有当没有更多磁盘值可以读取时，接收器才会尝试再次从内存中读取。这种模式切换方法保持了顺序，同时也具有在磁盘模式下不需要同步的额外好处，这有助于在从慢速磁盘读取时节省关键时间。

需要检查的第二个重要函数是`read_disk_value`，它在`next_value`中被引用。这个函数很长，主要是记录，但我确实想在这里指出该函数的第一部分：

```rs
    fn read_disk_value(&mut self) -> Result<T, super::Error> {
        loop {
            match self.fp.read_u32::<BigEndian>() {
                Ok(payload_size_in_bytes) => {
                    let mut payload_buf = vec![0; payload_size_in_bytes 
                     as usize];
                    match self.fp.read_exact(&mut payload_buf[..]) {
                        Ok(()) => {
                            let mut dec = 
                            DeflateDecoder::new(&payload_buf[..]);
                            match deserialize_from(&mut dec) {
                                Ok(event) => {
                                    self.disk_writes_to_read -= 1;
                                    return Ok(event);
                                }
                                Err(e) => panic!("Failed decoding. 
                                Skipping {:?}", e),
                            }
                        }
```

这段小代码使用了两个非常实用的库。Hopper 将其磁盘上的元素以 bincode 格式存储到磁盘上。Bincode（[`crates.io/crates/bincode`](https://crates.io/crates/bincode)）是为 servo 项目发明的，是一个用于 IPC 的序列化库——几乎就是 hopper 用它来做的。bincode 的优点是序列化和反序列化速度快，但缺点是它不是一个标准，并且版本之间没有保证的二进制格式。需要指出的第二个库在这个例子中几乎看不见；byteorder。你可以在这里看到它：`self.fp.read_u32::<BigEndian>`。Byteorder 扩展了`std::io::Read`，以便从字节缓冲区中反序列化原始类型。你可以手动做这件事，但这样做容易出错，而且重复起来很麻烦。使用 byteorder。所以，我们在这里看到的是 hopper 从`self.fp`——一个指向当前磁盘队列文件的`std::io::BufReader`——读取 32 位长度的 big-ending 长度前缀，并使用这个前缀从磁盘上读取恰好这么多字节，然后再将这些字节传递给反序列化器。就是这样。所有 hopper 磁盘上的元素都是 32 位长度的前缀，后面跟着这么多字节。

# 发送者

现在我们已经涵盖了`Receiver`，剩下的就是`Sender`。它在`src/sender.rs`中定义。`Sender`中最重要的函数是`send`。`Sender`遵循与`Receiver`相同的磁盘/内存模式理念，但在其操作上更为复杂。让我们深入探讨：

```rs
        let mut back_guard = self.mem_buffer.lock_back();
        if (*back_guard).inner.total_disk_writes == 0 {
            // in-memory mode
            let placed_event = private::Placement::Memory(event);
            match self.mem_buffer.push_back(placed_event, &mut 
            back_guard) {
                Ok(must_wake_receiver) => {
                    if must_wake_receiver {
                        let front_guard = self.mem_buffer.lock_front();
                        self.mem_buffer.notify_not_empty(&front_guard);
                        drop(front_guard);
                    }
                }
                Err(deque::Error::Full(placed_event)) => {
                    self.write_to_disk(placed_event.extract().unwrap(), 
                    &mut back_guard)?;
                    (*back_guard).inner.total_disk_writes += 1;
                }
            }
```

回想一下，双端队列允许后端守护者通过它走私状态以进行协调。我们在这里看到了它的回报。`Sender`的内部状态被称为`SenderSync`，其定义如下：

```rs
pub struct SenderSync {
    pub sender_fp: Option<BufWriter<fs::File>>,
    pub bytes_written: usize,
    pub sender_seq_num: usize,
    pub total_disk_writes: usize,
    pub path: PathBuf, // active fp filename
}
```

每个发送线程都必须能够写入当前的磁盘队列文件，该文件由 `sender_fp` 指向。同样，`bytes_written` 跟踪已经写入磁盘的字节数。`Sender` 必须跟踪这个值，以便在队列文件变得太大时正确地滚动队列文件。`sender_seq_num` 定义了当前可写入队列文件的名称，因为它们是从零开始的顺序命名。对我们来说，关键字段是 `total_disk_writes`。请注意，内存写入——`self.mem_buffer.push_back(placed_event, &mut back_guard)`——可能会因为 `Full` 错误而失败。在这种情况下，会调用 `self.write_to_disk` 将 `T` 写入磁盘，从而增加磁盘写入的总数。这种写入模式前面有一个检查跨线程 `SenderSync`，以确定是否有挂起的磁盘写入。记住，在这个时候，`Receiver` 没有办法确定是否有额外的写入操作已经写入磁盘；与 `Receiver` 的唯一通信渠道是通过内存中的双端队列。为此，下一个 `Sender` 线程将切换到不同的写入模式：

```rs
        } else {
            // disk mode
            self.write_to_disk(event, &mut back_guard)?;
            (*back_guard).inner.total_disk_writes += 1;
            assert!((*back_guard).inner.sender_fp.is_some());
            if let Some(ref mut fp) = (*back_guard).inner.sender_fp {
                fp.flush().expect("unable to flush");
            } else {
                unreachable!()
            }
            if let Ok(must_wake_receiver) = self.mem_buffer.push_back(
                private::Placement::Disk(
                    (*back_guard).inner.total_disk_writes
                ),
                &mut back_guard,
            ) {
                (*back_guard).inner.total_disk_writes = 0;
                if must_wake_receiver {
                    let front_guard = self.mem_buffer.lock_front();
                    self.mem_buffer.notify_not_empty(&front_guard);
                    drop(front_guard);
                }
            }
        }
```

一旦有单个写入磁盘，`Sender` 就切换到磁盘优先写入模式。在这个分支的开始，`T` 被写入磁盘，并进行刷新。然后，`Sender` 尝试将一个 `Placement::Disk` 推送到内存中的双端队列，如果 `Receiver` 的调度分配缓慢或不走运，可能会失败。然而，如果成功，`total_disk_writes` 就会被设置为零——不再有任何挂起的磁盘写入，并且如果需要，`Receiver` 会被唤醒以读取其新的事件。下一次 `Sender` 线程遍历时，它可能或可能没有足够的空间在内存中的双端队列中执行内存放置，但这将是下一个线程的问题。

这就是 `Sender` 的核心。虽然模块中还有一个大的函数 `write_to_disk`，但我们在这里不会列出它。实现主要是使用互斥锁进行账目管理，这个主题在本章和上一章中已经详细讨论过，还包括文件系统操作。话虽如此，我们热情地鼓励好奇的读者阅读代码。

# 测试并发数据结构

Hopper 是一个微妙的生物，证明要正确地处理它相当棘手，因为要在多个线程之间操作相同的内存很困难。在实现编写过程中，轻微的同步错误很常见，这些错误突出了基本的错误，但很少触发，甚至更难重现。传统的单元测试在这里是不够的。找不到干净的单元，因为计算机的非确定性行为是程序运行最终结果的基本因素。考虑到这一点，hopper 使用了三种关键的测试策略：在随机输入上执行随机测试以寻找逻辑错误（QuickCheck）、在随机输入上执行随机测试以寻找崩溃（模糊测试），以及运行数百万次相同的程序以寻找一致性失败。在接下来的章节中，我们将依次讨论这些策略。

# QuickCheck 和循环

前面的章节详细讨论了 QuickCheck，我们在这里不会重复。相反，让我们深入探讨一个来自`hopper`的测试，该测试在`src/lib.rs`中定义：

```rs
    #[test]
    fn multi_thread_concurrent_snd_and_rcv_round_trip() {
        fn inner(
            total_senders: usize,
            in_memory_bytes: usize,
            disk_bytes: usize,
            max_disk_files: usize,
            vals: Vec<u64>,
        ) -> TestResult {
            let sz = mem::size_of::<u64>();
            if total_senders == 0 || 
               total_senders > 10 || 
               vals.len() == 0 ||
               (vals.len() < total_senders) || 
               (in_memory_bytes / sz) == 0 ||
               (disk_bytes / sz) == 0
            {
                return TestResult::discard();
            }
            TestResult::from_bool(
              multi_thread_concurrent_snd_and_rcv_round_trip_exp(
                total_senders,
                in_memory_bytes,
                disk_bytes,
                max_disk_files,
                vals,
            ))
        }
        QuickCheck::new()
            .quickcheck(inner as fn(usize, usize, usize, usize, 
             Vec<u64>) -> TestResult);
    }
```

这个测试设置了一个多发送者、单接收者的往返环境，并小心地拒绝那些在内存缓冲区中允许小于 64 位空间、没有发送者或类似情况的输入。它将任务委托给另一个函数`multi_thread_concurrent_snd_and_rcv_round_trip_exp`来实际运行测试。

这个设置确实有些尴尬，但它的好处是允许`multi_thread_concurrent_snd_and_rcv_round_trip_exp`在明确的输入上运行。也就是说，当发现错误时，你可以通过创建手动测试或*明确地*（在 hopper 测试术语中）测试来轻松地重新播放那个测试。内部测试函数很复杂，我们将分部分考虑：

```rs
    fn multi_thread_concurrent_snd_and_rcv_round_trip_exp(
        total_senders: usize,
        in_memory_bytes: usize,
        disk_bytes: usize,
        max_disk_files: usize,
        vals: Vec<u64>,
    ) -> bool {
        if let Ok(dir) = tempdir::TempDir::new("hopper") {
            if let Ok((snd, mut rcv)) = 
              channel_with_explicit_capacity(
                "tst",
                dir.path(),
                in_memory_bytes,
                disk_bytes,
                max_disk_files,
            ) {
```

就像我们在本节开头对 hopper 的示例用法一样，内部测试函数使用 tempdir ([`crates.io/crates/tempdir`](https://crates.io/crates/tempdir))创建一个临时路径，以便传递给`channel_with_explicit_capacity`。但是，我们在这里非常小心，不要解包。因为 hopper 使用了文件系统，并且因为 Rust QuickCheck 是积极的多线程的，所以任何单个测试运行都可能遇到文件句柄耗尽的情况。这会导致 QuickCheck 出错，测试失败与这次特定执行的输入完全无关。

下一个部分如下：

```rs
                let chunk_size = vals.len() / total_senders;

                let mut snd_jh = Vec::new();
                let snd_vals = vals.clone();
                for chunk in snd_vals.chunks(chunk_size) {
                    let mut thr_snd = snd.clone();
                    let chunk = chunk.to_vec();
                    snd_jh.push(thread::spawn(move || {
                        let mut queued = Vec::new();
                        for mut ev in chunk {
                            loop {
                                match thr_snd.send(ev) {
                                    Err(res) => {
                                        ev = res.0;
                                    }
                                    Ok(()) => {
                                        break;
                                    }
                                }
                            }
                            queued.push(ev);
                        }
                        let mut attempts = 0;
                        loop {
                            if thr_snd.flush().is_ok() {
                                break;
                            }
                            thread::sleep(
                              ::std::time::Duration::from_millis(100)
                            );
                            attempts += 1;
                            assert!(attempts < 10);
                        }
                        queued
                    }))
                }
```

在这里，我们创建了发送者线程，每个线程从`vals`中抓取一个等大小的块。现在，由于线程调度的不可确定性，我们无法模拟`vals`元素被推入 hopper 的顺序。我们所能做的就是确认在通过 hopper 传输后没有丢失元素。实际上，它们可能在顺序上被破坏：

```rs
                let expected_total_vals = vals.len();
                let rcv_jh = thread::spawn(move || {
                    let mut collected = Vec::new();
                    let mut rcv_iter = rcv.iter();
                    while collected.len() < expected_total_vals {
                        let mut attempts = 0;
                        loop {
                            match rcv_iter.next() {
                                None => {
                                    attempts += 1;
                                    assert!(attempts < 10_000);
                                }
                                Some(res) => {
                                    collected.push(res);
                                    break;
                                }
                            }
                        }
                    }
                    collected
                });

                let mut snd_vals: Vec<u64> = Vec::new();
                for jh in snd_jh {
                    snd_vals.append(&mut jh.join().expect("snd join 
                    failed"));
                }
                let mut rcv_vals = rcv_jh.join().expect("rcv join 
                failed");

                rcv_vals.sort();
                snd_vals.sort();

                assert_eq!(rcv_vals, snd_vals);
```

另一个只有一个发送者的测试`single_sender_single_rcv_round_trip`能够检查正确的顺序以及没有数据丢失：

```rs
    #[test]
    fn single_sender_single_rcv_round_trip() {
        // Similar to the multi sender test except now with a single 
        // sender we can guarantee order.
        fn inner(
            in_memory_bytes: usize,
            disk_bytes: usize,
            max_disk_files: usize,
            total_vals: usize,
        ) -> TestResult {
            let sz = mem::size_of::<u64>();
            if total_vals == 0 || (in_memory_bytes / sz) == 0 || 
            (disk_bytes / sz) == 0 {
                return TestResult::discard();
            }
            TestResult::from_bool(
              single_sender_single_rcv_round_trip_exp(
                in_memory_bytes,
                disk_bytes,
                max_disk_files,
                total_vals,
            ))
        }
        QuickCheck::new().quickcheck(inner as fn(usize, usize, 
                                     usize, usize) -> TestResult);
    }
```

就像它的多堂兄弟一样，这个 QuickCheck 测试使用一个内部函数来执行实际的测试：

```rs
    fn single_sender_single_rcv_round_trip_exp(
        in_memory_bytes: usize,
        disk_bytes: usize,
        max_disk_files: usize,
        total_vals: usize,
    ) -> bool {
        if let Ok(dir) = tempdir::TempDir::new("hopper") {
            if let Ok((mut snd, mut rcv)) = 
            channel_with_explicit_capacity(
                "tst",
                dir.path(),
                in_memory_bytes,
                disk_bytes,
                max_disk_files,
            ) {
                let builder = thread::Builder::new();
                if let Ok(snd_jh) = builder.spawn(move || {
                    for i in 0..total_vals {
                        loop {
                            if snd.send(i).is_ok() {
                                break;
                            }
                        }
                    }
                    let mut attempts = 0;
                    loop {
                        if snd.flush().is_ok() {
                            break;
                        }
                        thread::sleep(
                          ::std::time::Duration::from_millis(100)
                        );
                        attempts += 1;
                        assert!(attempts < 10);
                    }
                }) {
                    let builder = thread::Builder::new();
                    if let Ok(rcv_jh) = builder.spawn(move || {
                        let mut rcv_iter = rcv.iter();
                        let mut cur = 0;
                        while cur != total_vals {
                            let mut attempts = 0;
                            loop {
                                if let Some(rcvd) = rcv_iter.next() {
                                    debug_assert_eq!(
                                        cur, rcvd,
                                    "FAILED TO GET ALL IN ORDER: {:?}",
                                        rcvd,
                                    );
                                    cur += 1;
                                    break;
                                } else {
                                    attempts += 1;
                                    assert!(attempts < 10_000);
                                }
                            }
                        }
                    }) {
                        snd_jh.join().expect("snd join failed");
                        rcv_jh.join().expect("rcv join failed");
                    }
                }
            }
        }
        true
    }
```

这一点也应该很熟悉，只是现在接收线程能够检查顺序。之前是将`Vec<u64>`输入到函数中，我们现在通过跳转队列流式传输`0, 1, 2, .. total_vals`，断言顺序并且另一侧没有间隙。对单个输入的单次运行将无法可靠地触发低概率竞态问题，但这不是目标。我们正在寻找逻辑错误。例如，这个库的早期版本会愉快地允许内存中的最大字节数小于元素`T`的总字节数。另一个版本可以将多个`T`实例放入缓冲区，但如果`total_vals`是奇数并且内存大小足够小以至于需要磁盘分页，那么流中的最后一个元素将永远不会被踢出。实际上，这仍然是一个问题。这是发送者中懒惰翻转至磁盘模式的结果；如果没有其他元素可能触发磁盘放置到内存缓冲区，写入将被刷新到磁盘，但接收者永远不会意识到这一点。为此，发送者确实提供了一个`flush`函数，您可以在测试中看到它的使用。在实践中，在 cernan 中，刷新是不必要的。但是，这是设计者没有预料到的设计角落，如果这个设计被广泛使用，可能很难注意到这个问题。

我们单发送者变体的内部测试也用于跳转测试的重复循环变体：

```rs
    #[test]
    fn explicit_single_sender_single_rcv_round_trip() {
        let mut loops = 0;
        loop {
         assert!(single_sender_single_rcv_round_trip_exp(8, 8, 5, 10));
            loops += 1;
            if loops > 2_500 {
                break;
            }
            thread::sleep(::std::time::Duration::from_millis(1));
        }
    }
```

注意，这里的内部循环仅进行 2,500 次迭代。这是为了尊重 CI 服务器的需求，它们不介意连续几小时运行高 CPU 负载的代码。在开发过程中，这个 2,500 次将会调整增加。但核心思想是明显的；检查一系列有序输入是否能够一次又一次地通过跳转队列按顺序且完整地返回。QuickCheck 搜索状态空间中的暗角，而更传统的手动测试则反复敲打同一位置以挖掘计算机的不确定性。

# 使用 AFL 搜索崩溃

上一节中测试的重复循环变体很有趣。它不像 QuickCheck 那样，完全专注于寻找逻辑错误。我们知道测试将在大多数迭代中工作。在那里寻找的是未能正确控制底层机器的不确定性。在某种程度上，这种测试变体使用逻辑检查来搜索崩溃。它是一种模糊测试。

Hopper 与文件系统交互，创建线程，并从文件中操作字节。这些活动都可能因为资源不足、偏移错误或对介质的基本误解而失败。例如，hopper 的早期版本假设对磁盘的小原子写入不会交错，这在 XFS 中是正确的，但在大多数其他文件系统中则不正确。或者，hopper 的另一个版本总是通过更熟悉的 `thread::spawn` 来创建测试中的线程。然而，如果无法创建线程，这个函数会引发恐慌，这就是为什么测试使用 `thread::Builder` 模式而不是，允许恢复。

崩溃本身很重要。为此，hopper 有一个基于 AFL 的模糊设置。为了避免在用户系统上产生不必要的二进制文件，截至本文撰写时，hopper 项目没有托管自己的模糊代码。相反，它位于 `blt/hopper-fuzz` ([`github.com/blt/hopper-fuzz`](https://github.com/blt/hopper-fuzz))。诚然，这很尴尬，但模糊测试，由于其不常见，通常是这样的。模糊测试可能持续数天，而且不适合现代 CI 系统。AFL 本身也不是一个容易自动化的工具，这也加剧了问题。模糊测试通常在用户私有机器上的小批量中进行，至少对于社区规模较小的开源项目是这样的。在 hopper-fuzz 中，有一个用于模糊测试的单个程序，其前言如下：

```rs
extern crate hopper;
extern crate rand;
#[macro_use]
extern crate serde_derive;
extern crate bincode;
extern crate tempdir;

use bincode::{deserialize_from, Bounded};
use self::hopper::channel_with_explicit_capacity;
use std::{io, process};
use rand::{Rng, SeedableRng, XorShiftRng};
```

基于我们迄今为止所看到的，相当直接。将输入传递给模糊程序有时并不简单，尤其是当输入不仅仅是输入集而不是，比如说，解析器的情况下的程序的实际工作单元。作者的方法是依赖 serde 来反序列化 AFL 将投入程序的字节缓冲区，知道大多数有效载荷将无法解码，但并不太在意。为此，作者模糊程序的顶部通常有一个填充了控制数据的输入结构体，这个程序也不例外：

```rs
#[derive(Debug, PartialEq, Deserialize)]
struct Input { // 22 bytes
    seed_a: u32, // 4
    seed_b: u32, // 4
    seed_c: u32, // 4
    seed_d: u32, // 4
    max_in_memory_bytes: u32, // 4
    max_disk_bytes: u16, // 2
}
```

种子是为 `XorShiftRng`，而 `max_in_memory_bytes` 和 `max_disk_bytes` 是为 hopper。仔细计算输入的字节大小，以避免模糊测试 bincode 反序列化器拒绝异常大输入的能力。虽然 AFL 不是盲目的——毕竟它已经对程序的分支进行了仪器化——但它也不是很聪明。完全有可能，程序员想要模糊测试的内容并不是一开始就模糊测试的内容。在模糊项目中引入任何额外的库中找出错误并不罕见。这是为了将库的使用限制在最小范围内，并使其使用最小化。

模糊程序的 `main` 函数开始于与其他 hopper 测试类似的设置：

```rs
fn main() {
    let stdin = io::stdin();
    let mut stdin = stdin.lock();
    if let Ok(input) = deserialize_from(&mut stdin, Bounded(22)) {
        let mut input: Input = input;
        let mut rng: XorShiftRng =
            SeedableRng::from_seed([input.seed_a, input.seed_b, 
                                    input.seed_c, input.seed_d]);

        input.max_in_memory_bytes = 
          if input.max_in_memory_bytes > 2 << 25 {
            2 << 25
        } else {
            input.max_in_memory_bytes
        };

        // We need to be absolutely sure we don't run into another 
        // running process. Which, this isn't a guarantee but 
        // it's _pretty_ unlikely to hit jackpot.
        let prefix = format!(
            "hopper-{}-{}-{}",
            rng.gen::<u64>(),
            rng.gen::<u64>(),
            rng.gen::<u64>()
        );
```

唯一奇怪的是前缀的产生。与 AFL 将要运行的数以百万计的测试相比，`tempdir`名称中的熵相对较低。我们特别想要确保没有两个料斗运行具有相同的数据目录，因为这将是未定义的行为：

```rs
        if let Ok(dir) = tempdir::TempDir::new(&prefix) {
            if let Ok((mut snd, mut rcv)) = 
            channel_with_explicit_capacity(
                "afl",
                dir.path(),
                input.max_in_memory_bytes as usize,
                input.max_disk_bytes as usize,
            ) {
```

AFL 测试的核心内容令人惊讶：

```rs
                let mut writes = 0;
                let mut rcv_iter = rcv.iter();
                loop {
                    match rng.gen_range(0, 102) {
                        0...50 => {
                            if writes != 0 {
                                let _ = rcv_iter.next();
                            }
                        }
                        50...100 => {
                            snd.send(rng.gen::<u64>());
                            writes += 1;
                        }
                        _ => {
                            process::exit(0);
                        }
                    }
                }
            }
        }
    }
}
```

这个程序引入了一个随机数生成器，在执行读取和写入料斗通道操作之间进行切换。为什么没有使用线程？好吧，结果是，AFL 在模型中很难适应线程。AFL 有一个稳定性概念，即它能够重新运行一个程序，并在指令执行等方面达到相同的结果。在多线程使用料斗的情况下，这行不通。尽管如此，尽管错过了探测发送者和接收者线程之间潜在竞争条件的机会，这个模糊测试发现：

+   文件描述符耗尽导致的崩溃

+   双端队列中的错误偏移计算

+   发送者/接收者级别的错误偏移计算

+   将元素反序列化到未清除的缓冲区中，导致产生幽灵元素

+   算术溢出/下溢导致的崩溃

+   分配足够空间进行序列化失败。

仅在多线程环境中发生的内部双端队列问题将会被忽略，当然，但前面的列表并非微不足道。

# 基准测试

好吧，到目前为止，我们可以合理地确信料斗适合使用，至少在不会崩溃并产生正确结果方面。但是，它也需要*快速*。为此，料斗附带 criterion（[`crates.io/crates/criterion`](https://crates.io/crates/criterion)）基准测试。截至写作时，criterion 是一个快速发展的库，它对基准运行结果进行统计分析，而 Rust 内置的夜间基准测试库则没有。此外，criterion 还可在稳定渠道上使用。要匹配的目标是标准库的 MPSC，这为料斗设定了基线。为此，基准测试套件执行了比较，位于料斗存储库中的`benches/stdlib_comparison.rs`。

前置代码很典型：

```rs
#[macro_use]
extern crate criterion;
extern crate hopper;
extern crate tempdir;

use std::{mem, thread};
use criterion::{Bencher, Criterion};
use hopper::channel_with_explicit_capacity;
use std::sync::mpsc::channel;
```

注意，我们已经引入了 MPSC 和料斗。我们将要基准测试的 MPSC 函数是：

```rs
fn mpsc_tst(input: MpscInput) -> () {
    let (tx, rx) = channel();

    let chunk_size = input.ith / input.total_senders;

    let mut snd_jh = Vec::new();
    for _ in 0..input.total_senders {
        let tx = tx.clone();
        let builder = thread::Builder::new();
        if let Ok(handler) = builder.spawn(move || {
            for i in 0..chunk_size {
                tx.send(i).unwrap();
            }
        }) {
            snd_jh.push(handler);
        }
    }

    let total_senders = snd_jh.len();
    let builder = thread::Builder::new();
    match builder.spawn(move || {
        let mut collected = 0;
        while collected < (chunk_size * total_senders) {
            let _ = rx.recv().unwrap();
            collected += 1;
        }
    }) {
        Ok(rcv_jh) => {
            for jh in snd_jh {
                jh.join().expect("snd join failed");
            }
            rcv_jh.join().expect("rcv join failed");
        }
        _ => {
            return;
        }
    }
}
```

一些发送线程被创建，一个接收者存在，并尽可能快地从 MPSC 中提取值。这根本不是逻辑检查，收集到的材料立即被丢弃。就像模糊测试一样，函数的输入是结构化数据。`MpscInput`定义如下：

```rs
#[derive(Debug, Clone, Copy)]
struct MpscInput {
    total_senders: usize,
    ith: usize,
}
```

这个函数的料斗版本稍微长一些，因为需要处理更多的错误状态，但我们之前都见过：

```rs
fn hopper_tst(input: HopperInput) -> () {
    let sz = mem::size_of::<u64>();
    let in_memory_bytes = sz * input.in_memory_total;
    let max_disk_bytes = sz * input.max_disk_total;
    if let Ok(dir) = tempdir::TempDir::new("hopper") {
        if let Ok((snd, mut rcv)) = channel_with_explicit_capacity(
            "tst",
            dir.path(),
            in_memory_bytes,
            max_disk_bytes,
            usize::max_value(),
        ) {
            let chunk_size = input.ith / input.total_senders;

            let mut snd_jh = Vec::new();
            for _ in 0..input.total_senders {
                let mut thr_snd = snd.clone();
                let builder = thread::Builder::new();
                if let Ok(handler) = builder.spawn(move || {
                    for i in 0..chunk_size {
                        let _ = thr_snd.send(i);
                    }
                }) {
                    snd_jh.push(handler);
                }
            }

            let total_senders = snd_jh.len();
            let builder = thread::Builder::new();
            match builder.spawn(move || {
                let mut collected = 0;
                let mut rcv_iter = rcv.iter();
                while collected < (chunk_size * total_senders) {
                    if rcv_iter.next().is_some() {
                        collected += 1;
                    }
                }
            }) {
                Ok(rcv_jh) => {
                    for jh in snd_jh {
                        jh.join().expect("snd join failed");
                    }
                    rcv_jh.join().expect("rcv join failed");
                }
                _ => {
                    return;
                }
            }
        }
    }
}
```

同样适用于`HopperInput`：

```rs
#[derive(Debug, Clone, Copy)]
struct HopperInput {
    in_memory_total: usize,
    max_disk_total: usize,
    total_senders: usize,
    ith: usize,
}
```

Criterion 提供了许多运行基准测试的选项，但在这里我们选择在输入上运行。以下是 MPSC 的设置：

```rs
fn mpsc_benchmark(c: &mut Criterion) {
    c.bench_function_over_inputs(
        "mpsc_tst",
        |b: &mut Bencher, input: &MpscInput| b.iter(|| 
         mpsc_tst(*input)),
        vec![
            MpscInput {
                total_senders: 2 << 1,
                ith: 2 << 12,
            },
        ],
    );
}
```

为了解释，我们有一个函数，`mpsc_benchmark`，它接受可变的 criterion 结构，这个结构在使用时是透明的，但其中 criterion 会存储运行数据。这个结构暴露了`bench_function_over_inputs`，它消耗一个闭包，我们可以通过`mpsc_test`来传递。唯一的输入列在一个向量中。以下是一个设置，它做的是同样的事情，但针对 hopper：

```rs
fn hopper_benchmark(c: &mut Criterion) {
    c.bench_function_over_inputs(
        "hopper_tst",
        |b: &mut Bencher, input: &HopperInput| b.iter(|| 
         hopper_tst(*input)),
        vec![
            // all in-memory
            HopperInput {
                in_memory_total: 2 << 11,
                max_disk_total: 2 << 14,
                total_senders: 2 << 1,
                ith: 2 << 11,
            },
            // swap to disk
            HopperInput {
                in_memory_total: 2 << 11,
                max_disk_total: 2 << 14,
                total_senders: 2 << 1,
                ith: 2 << 12,
            },
        ],
    );
}
```

注意现在我们有两个输入，一个是保证全部在内存中，另一个保证需要磁盘分页。磁盘分页输入的大小适当，以匹配 MSPC 运行。对于 hopper 和 MPSC 进行内存比较并无害处，但作者更倾向于悲观的基准测试，因为作者是一个乐观主义者。以下是我们将在本书剩余部分看到的基准测试中需要的最终位几乎都是稳定的：

```rs
criterion_group!{
    name = benches;
    config = Criterion::default().without_plots();
    targets = hopper_benchmark, mpsc_benchmark
}
criterion_main!(benches);
```

我鼓励你自己运行基准测试。我们看到 hopper 的时间大约是我们打算为 hopper 设计的系统的三倍快。这已经足够快了。

# 摘要

在本章中，我们介绍了剩余的基本非原子 Rust 同步原语，对 postmates/hopper 库进行了深入研究，以探索其在生产代码库中的应用。在消化了第四章，*同步与发送 – Rust 并发的基石*，以及本章内容后，读者应该能够很好地在 Rust 中构建基于锁的并发数据结构。对于需要更高性能的读者，我们将在第六章，*原子操作 – 同步的原始操作*，和第七章，*原子操作 – 安全地回收内存*中探讨原子编程主题。

如果你认为基于锁的并发很困难，那么等你看到原子操作。

# 进一步阅读

构建并发数据结构是一个广泛关注的领域。这些笔记涵盖了与第四章，*同步与发送 – Rust 并发的基石*中笔记几乎相同的空间。请务必参考那些笔记。

+   *《信号量小书》*，艾伦·道尼，可在[`greenteapress.com/wp/semaphores/`](http://greenteapress.com/wp/semaphores/)找到。这是一本关于并发谜题的迷人书籍，适合本科生阅读，但在 Rust 中由于没有信号量，挑战性也足够大。我们将在下一章构建原子并发原语时再次回顾这本书。

+   *《松散数据结构的可计算性：队列和栈为例》，作者：尼尔·沙维特和加迪·塔本菲尔德。本章讨论了基于《多处理器编程的艺术》的演示和作者对 Erlang 进程队列的了解实现的并发队列。队列是一种常见的并发数据结构，有许多可能的实现方法。本文讨论了一个有趣的概念。也就是说，如果我们放松队列的排序约束，我们能否从现代机器中挤出更多的性能？

+   *《并行编程难吗？如果难，你能做什么？》*，作者：保罗·麦肯尼。这本书涵盖了与第四章，*同步与发送——Rust 并发的基础*和第五章，*锁——互斥锁、条件变量、屏障和读写锁*大致相同的材料，但内容更为详细。我强烈建议读者寻找这本书的副本并阅读它，尤其是关于验证实现的第十一章。
