# 标准库的算法

Rust 的标准库提供了一些基本的数据类型，这些类型覆盖了许多项目的基本需求，通常情况下，如果已经有了适当的数据结构，就无需实现自己的算法。如果由于某种原因，数据类型并不完全适合任务，标准库也为你提供了相应的解决方案。在这个快速总结中，你可以期待了解以下内容：

+   `slice` 原始类型

+   `Iterator` 特质

+   `binary_search()`

+   `sort()`、稳定和不稳定的排序

# 切片和迭代

类似于其他语言的库中接口标准化对功能访问的方式，Rust 的标准库通过类型和特质来提供基本实现。特质 `Iterator<T>` 在本书的多个地方被提及和使用。然而，切片类型并没有被大量显式使用，特别是当 `Vec<T>` 被借用来进行函数调用时，Rust 编译器会自动使用切片。那么，如何利用这种类型呢？我们已经看到了 `Iterator<T>` 的实现，但它提供了更多吗？

# 迭代器

回顾：迭代器是一种遍历集合的模式，在遍历过程中提供对每个元素的指针。这种模式在 1994 年由 Erich Gamma、Richard Helm、Ralph Johnson 和 John Vlissides（四人帮）所著的《设计模式》一书中被提及，并且几乎在每种语言中以某种方式存在。

在 Rust 中，每个元素的指针术语获得了一个新的维度：它是借用的还是拥有的项？它是否也可以被可变借用？

使用标准库的 `Iterator<T>` 特质非常有意义，因为它提供了一系列有用的函数，这些函数都基于单一的 `next()` 实现。

`next()` 返回一个 `Option<Self::Item>`，这是在实现特质时必须声明的关联类型——它可以是你想要的任何类型！

因此，使用 `&MyType`、`&mut MyType` 和 `MyType` 可以分别实现以实现所需的功能。`IntoIter<T>` 是一个特质，它专门设计用来简化这种工作流程，并与 `for` 循环语法完美集成。以下代码来自 Rust 标准库的源代码：

```rs
impl<T> IntoIterator for Vec<T> {
    type Item = T;
    type IntoIter = IntoIter<T>;

    /// Creates a consuming iterator, that is, 
    /// one that moves each value out of
    /// the vector (from start to end). 
    /// The vector cannot be used after calling
    /// this.
    ///
    /// # Examples
    ///
    /// ```

    /// let v = vec!["a".to_string(), "b".to_string()];

    /// for s in v.into_iter() {

    /// // s has type String, not &String

    /// println!("{}", s);

    /// }

    /// ```rs
    #[inline]
    fn into_iter(mut self) -> IntoIter<T> {
        unsafe {
            let begin = self.as_mut_ptr();
            assume(!begin.is_null());
            let end = if mem::size_of::<T>() == 0 {
                arith_offset(begin as *const i8, self.len() 
                             as isize) as *const T
            } else {
                begin.add(self.len()) as *const T
            };
            let cap = self.buf.cap();
            mem::forget(self);
            IntoIter {
                buf: NonNull::new_unchecked(begin),
                phantom: PhantomData,
                cap,
                ptr: begin,
                end,
            }
        }
    }
}
```

Rust 的 `Vec<T>` 正确实现了这种模式，但有一个巧妙的转折。前面的代码消耗了原始数据结构，可能将其转换成更容易迭代的格式，就像树可以被展开成排序后的 `Vec<T>` 或栈。回到原来的主题，`Iterator<T>` 提供了函数（在进一步的结构中实现），这些函数增加了许多在集合中进行搜索和过滤的可能方式。

任何 Rust 用户都会意识到 `Vec<T>` 的 `iter()` 函数，然而，实际上这是由 `Vec` 隐式转换为切片类型提供的？

# 切片

切片是序列的视图，以提供更统一的接口来访问、迭代或以其他方式与这些内存区域交互。因此，它们可以通过 `Vec<T>` 获取，特别是由于它们实现了 `Deref` 特性，可以隐式地将 `Vec<T>` 作为 `[T]`——`T` 的切片来处理。

`Vec<T>` 的实现也暗示了不可变和可变引用的 `IntoIterator` 实现：

```rs
impl<'a, T> IntoIterator for &'a Vec<T> {
    type Item = &'a T;
    type IntoIter = slice::Iter<'a, T>;

    fn into_iter(self) -> slice::Iter<'a, T> {
        self.iter()
    }
}

impl<'a, T> IntoIterator for &'a mut Vec<T> {
    type Item = &'a mut T;
    type IntoIter = slice::IterMut<'a, T>;

    fn into_iter(self) -> slice::IterMut<'a, T> {
        self.iter_mut()
    }
}
```

切片本身只是一个视图，由指向内存部分的指针及其长度表示。由于编译器知道其中包含的数据的性质，它也可以确定单个元素以提供类型安全。

切片及其工作方式的更详细解释可能需要一本书来专门介绍，因此建议至少阅读 `slice` 模块的文档（或源代码）([`doc.rust-lang.org/std/slice/index.html`](https://doc.rust-lang.org/std/slice/index.html))[.](https://doc.rust-lang.org/std/slice/index.html)

# 搜索

在本书中已经讨论了如何在集合中查找事物，Rust 标准库默认提供了一些方法。这些函数附加到 `Iterator<T>` 特性或切片类型上，并且无论实际类型如何，只要提供了比较两个元素的功能，它们就可以工作。

这可以是 `Ord` 特性或自定义比较函数，例如 `Iterator<T>` 上的 `position()` 函数。

# 线性查找

经典的线性查找是通过 `Iterator<T>` 特性上的 `position()`（或 `rposition()`）提供的，它甚至利用了在特性本身上实现的其它迭代器函数：

```rs
fn position<P>(&mut self, mut predicate: P) -> Option<usize> where
    Self: Sized,
    P: FnMut(Self::Item) -> bool,
{
    // The addition might panic on overflow
    self.try_fold(0, move |i, x| {
        if predicate(x) { LoopState::Break(i) }
        else { LoopState::Continue(i + 1) }
    }).break_value()
}
```

`try_fold()` 是 `fold()`（或 `reduce()`，遵循 map/reduce 模式）函数的短路变体，它在返回 `LoopState::Break` 时随时返回。对 `break_value()` 的调用将 `LoopState::Break` 枚举中返回的值转换为 `Option` 和 `None`，如果它运行了整个集合。

这是搜索的暴力方法，如果集合未排序且较短，可能很有用。对于任何更长的情况，排序和使用二分搜索函数可能更有利。

# 二分查找

切片还提供了一个通用的快速搜索函数，称为 `binary_search()`。如第十章[32002bad-c2bb-46e9-918d-12d7dabfe579.xhtml]“查找事物”中讨论的，二分搜索通过反复选择一半来逼近元素的位置，返回元素的索引。

要实现这一点，输入切片必须满足两个先决条件：

+   它是有序的

+   元素类型实现了 `Ord` 特性

`binary_search()`无法检查提供的集合是否已排序，这意味着如果无序集合返回预期的结果，这只能是一种巧合。此外，如果有多个具有相同值的元素，任何这些都可以是结果。

除了使用隐式提供的比较函数（通过实现`Ord`），`binary_search()`还有一个更灵活的兄弟函数—`binary_search_by()`，它需要一个提供的比较函数。

在底层，这个函数与我们在第十章中创建的朴素实现 Chapter 10，*Finding Stuff*相似；有时，它甚至快上几纳秒。然而，代码同样简单：

```rs
pub fn binary_search_by<'a, F>(&'a self, mut f: F) -> Result<usize, usize>
        where F: FnMut(&'a T) -> Ordering
    {
        let s = self;
        let mut size = s.len();
        if size == 0 {
            return Err(0);
        }
        let mut base = 0usize;
        while size > 1 {
            let half = size / 2;
            let mid = base + half;
            // mid is always in [0, size), 
            // that means mid is >= 0 and < size.
            // mid >= 0: by definition
            // mid < size: mid = size / 2 + size / 4 + size / 8 ...
            let cmp = f(unsafe { s.get_unchecked(mid) });
            base = if cmp == Greater { base } else { mid };
            size -= half;
        }
        // base is always in [0, size) because base <= mid.
        let cmp = f(unsafe { s.get_unchecked(base) });
        if cmp == Equal { Ok(base) } else { 
                     Err(base + (cmp == Less) as usize) }

    }
```

该函数的其他变体包括通过键或通过`Ord`特质的比较函数进行搜索（如前所述）。一个主要的注意事项是需要向二分搜索函数提供一个已排序的集合，但幸运的是，Rust 在其标准库中提供了排序功能。

# 排序

排序是用户界面中的一个重要功能，同时也为许多算法提供了必要的可预测性。每当没有适当的数据结构（如树）可以使用时，一个通用的排序算法可以负责创建这种顺序。关于相等值的一个重要问题随之而来：它们是否每次都会出现在同一个确切的位置？当使用稳定排序算法时，答案是*是的*。

# 稳定排序

稳定排序的关键不是重新排列相等元素，所以在`[1, 1, 2, 3, 4, 5]`中，`1`s 相对于彼此的位置永远不会改变。在 Rust 中，当在`Vec<T>`上调用`sort()`时，实际上就是这样使用的。

当前（2018 版）`Vec<T>`的实现使用基于 Timsort 的归并排序变体。以下是源代码：

```rs
pub fn sort(&mut self)
    where T: Ord
{
    merge_sort(self, |a, b| a.lt(b));
}
```

代码相当冗长，但可以分成更小的部分。第一步是对较小的（20 个元素或更少）切片进行排序，通过删除并按顺序重新插入元素来实现（换句话说，插入排序）：

```rs
fn merge_sort<T, F>(v: &mut [T], mut is_less: F)
    where F: FnMut(&T, &T) -> bool
{
    // Slices of up to this length get sorted using insertion sort.
    const MAX_INSERTION: usize = 20;
    // Very short runs are extended using insertion sort 
    // to span at least this many elements.
    const MIN_RUN: usize = 10;

    // Sorting has no meaningful behavior on zero-sized types.
    if size_of::<T>() == 0 {
        return;
    }

    let len = v.len();

    // Short arrays get sorted in-place via insertion 
    // sort to avoid allocations.
    if len <= MAX_INSERTION {
        if len >= 2 {
            for i in (0..len-1).rev() {
                insert_head(&mut v[i..], &mut is_less);
            }
        }
        return;
    }
```

如果集合更长，算法将回溯遍历项目，识别自然运行。常量`MIN_RUN`（前述代码中的`10`）定义了此类运行的最小长度，因此较短的运行（例如`[5, 9, 10, 11, 13, 19, 31, 55, 56]`中的`5, 9, 10, 11, 13, 19, 31, 55, 56`）通过在 1 上执行插入排序以获得 10 个元素来扩展。然后，结果块（对于`[1, 5, 9, 10, 11, 13, 19, 31, 55, 56]`，它将从`0`开始，长度为 10）将被推送到栈上以供后续合并（注意：我们建议阅读代码作者的注释）：

```rs
    // Allocate a buffer to use as scratch memory. 
    // We keep the length 0 so we can keep in it
    // shallow copies of the contents of `v` without risking the dtors     
    // running on copies if `is_less` panics. 
    // When merging two sorted runs, this buffer holds a copy of the 
    // shorter run, which will always have length at most `len / 2`.
    let mut buf = Vec::with_capacity(len / 2);

    // In order to identify natural runs in `v`, we traverse it 
    // backwards. That might seem like a strange decision, but consider 
    // the fact that merges more often go in the opposite direction
    // (forwards). According to benchmarks, merging forwards is 
    // slightly faster than merging backwards. To conclude, identifying 
    // runs by traversing backwards improves performance.
    let mut runs = vec![];
    let mut end = len;
    while end > 0 {
        // Find the next natural run, 
        // and reverse it if it's strictly descending.
        let mut start = end - 1;
        if start > 0 {
            start -= 1;
            unsafe {
                if is_less(v.get_unchecked(start + 1), 
                           v.get_unchecked(start)) {
                    while start > 0 && is_less(v.get_unchecked(start),
                                       v.get_unchecked(start - 1)) {
                        start -= 1;
                    }
                    v[start..end].reverse();
                } else {
                    while start > 0 && !is_less(v.get_unchecked(start),
                                       v.get_unchecked(start - 1)) {
                        start -= 1;
                    }
                }
            }
        }

        // Insert some more elements into the run if it's too short. 
        // Insertion sort is faster than
        // merge sort on short sequences, 
        // so this significantly improves performance.
        while start > 0 && end - start < MIN_RUN {
            start -= 1;
            insert_head(&mut v[start..end], &mut is_less);
        }

        // Push this run onto the stack.
        runs.push(Run {
            start,
            len: end - start,
        });
        end = start;
```

为了结束迭代，栈上的一些对已经合并，在插入排序中折叠它们：

```rs
        while let Some(r) = collapse(&runs) {
            let left = runs[r + 1];
            let right = runs[r];
            unsafe {
                merge(&mut v[left.start .. right.start + right.len], 
                      left.len, buf.as_mut_ptr(), &mut is_less);
            }
            runs[r] = Run {
                start: left.start,
                len: left.len + right.len,
            };
            runs.remove(r + 1);
        }
    }
```

这个`collapse`循环确保栈上只留下一个项目，即已排序的序列。找出要折叠的运行是 Timsort 的关键部分，因为合并只是简单地使用插入排序来完成。折叠函数检查两个基本条件：

+   运行的长度按降序排列（栈顶持有最长的运行）

+   每个生成的运行的长度大于下一个两个运行的长度之和

考虑到这一点，让我们看看折叠函数：

```rs
    // [...]
    fn collapse(runs: &[Run]) -> Option<usize> {
        let n = runs.len();
        if n >= 2 && (runs[n - 1].start == 0 ||
                      runs[n - 2].len <= runs[n - 1].len ||
                      (n >= 3 && runs[n - 3].len <= 
                       runs[n - 2].len + runs[n - 1].len) ||
                      (n >= 4 && runs[n - 4].len <= 
                       runs[n - 3].len + runs[n - 2].len)) {
            if n >= 3 && runs[n - 3].len < runs[n - 1].len {
                Some(n - 3)
            } else {
                Some(n - 2)
            }
        } else {
            None
        }
    }
    // [...]
}
```

它返回要与其后续元素合并的运行索引（`r`和`r + 1`；有关更多信息，请参阅`collapse`循环）。如果最顶部的运行（在最高索引处）不是从开始处开始的，折叠函数会检查前四个运行以满足上述条件。如果是，则几乎到达了终点，需要进行合并，无论违反了哪些条件，从而确保最终要合并的序列是最后一个。

Timsort 结合插入排序和归并排序，使其成为一个真正快速且高效的排序算法，它也是稳定的，并且通过构建这些自然发生的运行来在“块”上操作。另一方面，不稳定的排序使用熟悉的快速排序。

# 不稳定的排序

不稳定的排序不会保留相等值的相对位置，因此由于不需要稳定排序所需额外分配的内存，可以因此实现更快的速度。切片的`sort_unstable()`函数使用了一种由 Orson Peters 称为模式破坏快速排序的快速排序变体，结合堆排序和快速排序，在大多数情况下实现出色的性能。

切片实现简单地将其称为快速排序：

```rs
    pub fn sort_unstable_by<F>(&mut self, mut compare: F)
        where F: FnMut(&T, &T) -> Ordering
    {
        sort::quicksort(self, |a, b| compare(a, b) == Ordering::Less);
    }
```

观察快速排序的实现，它跨越了整个模块——大约 700 行代码。因此，让我们看看最高级函数以了解基础知识；好奇的读者应该深入研究源代码([`doc.rust-lang.org/src/core/slice/sort.rs.html`](https://doc.rust-lang.org/src/core/slice/sort.rs.html))以了解更多信息。

快速排序函数执行一些初步检查以排除无效情况：

```rs
/// Sorts `v` using pattern-defeating quicksort, which is `O(n log n)` worst-case.
pub fn quicksort<T, F>(v: &mut [T], mut is_less: F)
    where F: FnMut(&T, &T) -> bool
{
    // Sorting has no meaningful behavior on zero-sized types.
    if mem::size_of::<T>() == 0 {
        return;
    }
    // Limit the number of imbalanced 
    // partitions to `floor(log2(len)) + 1`.
    let limit = mem::size_of::<usize>() * 8 - v.len()
                .leading_zeros() as usize;

    recurse(v, &mut is_less, None, limit);
}
```

`recurse`函数是这个实现的核心，甚至是一个递归函数：

```rs
/// Sorts `v` recursively.
///
/// If the slice had a predecessor in the original array, 
/// it is specified as `pred`.
///
/// `limit` is the number of allowed imbalanced partitions 
///  before switching to `heapsort`. If zero,
/// this function will immediately switch to heapsort.
fn recurse<'a, T, F>(mut v: &'a mut [T], is_less: &mut F, mut pred: Option<&'a T>, mut limit: usize)
    where F: FnMut(&T, &T) -> bool
{
    // Slices of up to this length get sorted using insertion sort.
    const MAX_INSERTION: usize = 20;

    // True if the last partitioning was reasonably balanced.
    let mut was_balanced = true;
    // True if the last partitioning didn't shuffle elements 
    // (the slice was already partitioned).
    let mut was_partitioned = true;

    loop {
        let len = v.len();
        // Very short slices get sorted using insertion sort.
        if len <= MAX_INSERTION {
            insertion_sort(v, is_less);
            return;
        }
        // If too many bad pivot choices were made, 
        // simply fall back to heapsort in order to
        // guarantee `O(n log n)` worst-case.
        if limit == 0 {
            heapsort(v, is_less);
            return;
        }
        // If the last partitioning was imbalanced, 
        // try breaking patterns in the slice by shuffling
        // some elements around. 
        // Hopefully we'll choose a better pivot this time.
        if !was_balanced {
            break_patterns(v);
            limit -= 1;
        }
        // Choose a pivot and try guessing 
        // whether the slice is already sorted.
        let (pivot, likely_sorted) = choose_pivot(v, is_less);

        // If the last partitioning was decently balanced 
        // and didn't shuffle elements, and if pivot
        // selection predicts the slice is likely already sorted...
        if was_balanced && was_partitioned && likely_sorted {
            // Try identifying several out-of-order elements 
            // and shifting them to correct
            // positions. If the slice ends up being completely sorted, 
            // we're done.
            if partial_insertion_sort(v, is_less) {
                return;
            }
        }
        // If the chosen pivot is equal to the predecessor, 
        // then it's the smallest element in the
        // slice. Partition the slice into elements equal to and 
        // elements greater than the pivot.
        // This case is usually hit when the slice contains many 
        // duplicate elements.
        if let Some(p) = pred {
            if !is_less(p, &v[pivot]) {
                let mid = partition_equal(v, pivot, is_less);

                // Continue sorting elements greater than the pivot.
                v = &mut {v}[mid..];
                continue;
            }
        }
        // Partition the slice.
        let (mid, was_p) = partition(v, pivot, is_less);
        was_balanced = cmp::min(mid, len - mid) >= len / 8;
        was_partitioned = was_p;

        // Split the slice into `left`, `pivot`, and `right`.
        let (left, right) = {v}.split_at_mut(mid);
        let (pivot, right) = right.split_at_mut(1);
        let pivot = &pivot[0];

        // Recurse into the shorter side only in order to 
        // minimize the total number of recursive
        // calls and consume less stack space. 
        // Then just continue with the longer side (this is
        // akin to tail recursion).
        if left.len() < right.len() {
            recurse(left, is_less, pred, limit);
            v = right;
            pred = Some(pivot);
        } else {
            recurse(right, is_less, Some(pivot), limit);
            v = left;
        }
    }
}
```

幸运的是，标准库的源代码中有许多有用的注释。因此，强烈建议阅读前面片段中的所有注释。简而言之，算法做出了很多猜测以避免对枢轴做出糟糕的选择。如果你还记得，当快速排序选择一个糟糕的枢轴元素时，它将分裂成不均匀的分区，从而产生非常糟糕的运行时间行为。因此，选择一个好的枢轴是至关重要的，这就是为什么围绕这个过程有那么多启发式方法被采用，如果所有其他方法都失败了，算法将运行堆排序以确保至少有*O(n log n)*的运行时间复杂度。

# 摘要

Rust 的标准库包括对基本事物如排序或搜索其原始切片类型和`Iterator<T>`特质的好几种实现。特别是切片类型提供了许多非常重要的函数。

`binary_search()`是针对切片类型提供的二分搜索概念的泛型实现。`Vec<T>`可以快速且容易（隐式地）转换为切片，这使得这个函数在所有情况下都可用。然而，它要求切片中存在排序顺序才能工作（如果不存在则不会失败），并且如果使用自定义类型，则需要实现`Ord`特质。

如果切片不能事先排序，`Iterator<T>`变量的`position()`（`find()`的）实现提供了一个基本的线性搜索，返回元素的第一个位置。

排序由一个泛型函数提供，但有两种风味：稳定和不稳定。常规的`sort()`函数使用一个称为 Timsort 的归并排序变体，以实现高效且稳定的排序性能。

`sort_unstable()`利用一种模式破坏的快速排序来巧妙地结合堆排序和快速排序的效率，这通常会导致比`sort()`更好的绝对运行时间。

这是本书的最后一章，如果你已经读到这儿，你终于应该得到一些答案了！你可以在*评估*部分找到所有问题的答案。

# 问题

+   Rust 在集合上的泛型算法实现在哪里？

+   何时线性搜索比二分搜索更好？

+   *可能的面试问题：* 稳定和不稳定排序算法是什么？

+   快速排序表现出的哪种不良行为被模式破坏的快速排序缓解了？

# 进一步阅读

这里有一些额外的参考资料，你可以参考有关本章所涵盖内容的材料：

+   *设计模式*，由 Erich Gamma、Richard Helm、Ralph Johnson 和 John Vlissides 著

+   维基百科上的迭代器模式([`en.wikipedia.org/wiki/Iterator_pattern`](https://en.wikipedia.org/wiki/Iterator_pattern))

+   *OpenJDK 的 java.utils.Collection.sort()存在缺陷：好的、坏的和最坏的情况*，由 de Gow 等人撰写([`envisage-project.eu/wp-content/uploads/2015/02/sorting.pdf`](http://envisage-project.eu/wp-content/uploads/2015/02/sorting.pdf))

+   模式破坏的快速排序([`envisage-project.eu/wp-content/uploads/2015/02/sorting.pdf`](http://envisage-project.eu/wp-content/uploads/2015/02/sorting.pdf))
