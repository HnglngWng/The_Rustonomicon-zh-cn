# Cloning

现在我们已经设置了一些基本的代码，我们需要一种方法来克隆`Arc`。

基本上，我们需要：

1. 增加原子引用计数

2. 从内部指针构造一个`Arc`的新实例

首先，我们需要访问 `ArcInner`：

```Rust
let inner = unsafe { self.ptr.as_ref() };
```

我们可以按如下方式更新原子引用计数：

```rust
let old_rc = inner.rc.fetch_add(1, Ordering::???);
```

但是我们应该在这里使用什么顺序? 我们实际上没有任何在克隆时需要原子同步的代码，因为我们在克隆时不会修改内部值。 因此，我们可以在这里使用Relaxed排序，这意味着没有happens-before关系,而是原子的。 然而，当`Drop` Arc时，我们需要在减少引用计数时进行原子同步。 这在[`Arc`的`Drop`实现一节](ch10-01-04-Dropping.md)中有更多描述。有关原子关系和Relaxed 排序的更多信息，请参阅[原子一节](ch08-03-Atomics.md)。

于是，代码变成了这样：

```rust
 let old_rc = inner.rc.fetch_add(1, Ordering::Relaxed);
```

我们需要添加另一个导入以使用`Ordering`：

```Rust
use std::sync::atomic::Ordering;
```

然而，我们现在在这个实现上有一个问题。 如果有人决定 `mem::forget` 一堆 Arcs 怎么办？ 到目前为止我们编写的代码（以及将要编写的）假设引用计数准确地描绘了内存中有多少 Arcs，但是使用 `mem::forget` 这是错误的。 因此，当越来越多的 Arcs 被从这个克隆出来时，它们没有被 `Drop`, 并且引用计数被递减，我们可能会溢出！ 这将导致释放后使用，这是 **非常糟糕的！**

为了处理这个问题，我们需要检查引用计数没有超过某个任意值（低于 `usize::MAX`，因为我们将引用计数存储为 `AtomicUsize`），然后做 *一些事情* 。

标准库的实现决定,如果引用计数在任何线程上达到`isize::MAX`（大约是`usize ::MAX`的一半) ，就中止程序(因为在正常代码中这是一个非常不可能的情况，如果发生，程序可能会非常退化),假设可能没有大约 20 亿个线程（或在某些 64 位机器上大约 **9 quintillion**）一次增加引用计数。 这就是我们要做的。

实现这种行为非常简单：

```rust
if old_rc >= isize::MAX as usize {
    std::process::abort();
}
```

然后，我们需要返回`Arc`的一个新实例：

```Rust
Self {
    ptr: self.ptr,
    phantom: PhantomData
}
```

现在，让我们把这一切都包含在`Clone`实现中：

```Rust
use std::sync::atomic::Ordering;

impl<T> Clone for Arc<T> {
    fn clone(&self) -> Arc<T> {
        let inner = unsafe { self.ptr.as_ref() };
        // Using a relaxed ordering is alright here as we don't need any atomic
        // synchronization here as we're not modifying or accessing the inner
        // data.
        let old_rc = inner.rc.fetch_add(1, Ordering::Relaxed);
        if old_rc >= isize::MAX as usize {
            std::process::abort();
        }

        Self {
            ptr: self.ptr,
            phantom: PhantomData,
        }
    }
}
```
