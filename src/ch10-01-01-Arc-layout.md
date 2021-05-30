# 布局(Layout)

让我们从Arc实现的布局开始。

`Arc<T>`提供了在堆中分配的类型为`T`的值的线程安全共享所有权。 共享意味着Rust中的不变性，所以我们不需要设计任何东西来管理对值的访问，对吗? 虽然像 Mutex 这样的内部可变性类型允许 Arc 的用户创建共享可变性，但 Arc 本身不需要关心这些问题。

然而，*有* 一个地方 Arc 需要关注变化：破坏。 当 Arc 的所有所有者都离开时，我们需要能够`drop`其内容并释放其分配。 因此，我们需要一种让所有者知道它是否是 *最后的* 所有者的方法，而最简单的方法就是对所有者进行计数--引用计数。

不幸的是，这个引用计数本质上是共享的可变状态，所以 Arc *确实* 需要考虑同步。 我们 *可以* 为此使用互斥锁，但这是多余的。 相反，我们将使用原子。 既然每个人都需要一个指向 T 的分配的指针，我们不妨将引用计数放在同一个分配中。

天真的，它看起来像这样：

```Rust
use std::sync::atomic;

pub struct Arc<T> {
    ptr: *mut ArcInner<T>,
}

pub struct ArcInner<T> {
    rc: atomic::AtomicUsize,
    data: T
}
```

这可以编译，但是不正确的。 首先，编译器会给我们带来过于严格的变体。 例如，在需要`Arc<&'a str>`的地方，不能使用`Arc<&'static str>`。 更重要的是，它会向drop检查器提供不正确的所有权信息，因为它将假定我们不拥有类型`T`的任何值。因为这是一个提供值的共享所有权的结构，在某些时候会有完全拥有其数据的这种结构的一个实例。 这种的结构。 有关变体和drop检查的所有详细信息，请参阅关于[所有权和生存期的章节](ch03-00-Ownership.md)。

为了解决第一个问题，我们可以使用`NonNull<T>`。 请注意，`NonNull<T>`是一个原始指针的包装,它声明以下内容：

- 我们是`T`的变体

- 我们的指针永远不会为空

为了解决第二个问题，我们可以包含一个包含`ArcInner<T>`的`PhantomData`标记。 这将告诉drop检查器我们有`ArcInner<T>`值（其本身包含`T`）的所有权概念。

经过这些更改，我们得到了最终的结构：

```Rust
use std::marker::PhantomData;
use std::ptr::NonNull;
use std::sync::atomic::AtomicUsize;

pub struct Arc<T> {
    ptr: NonNull<ArcInner<T>>,
    phantom: PhantomData<ArcInner<T>>
}

pub struct ArcInner<T> {
    rc: AtomicUsize,
    data: T
}
```
