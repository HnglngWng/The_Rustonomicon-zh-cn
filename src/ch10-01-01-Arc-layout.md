# 布局(Layout)

让我们从Arc实现的布局开始。

我们至少需要两样东西：一个指向我们的数据的指针和一个共享的原子引用计数。 因为我们正在构建Arc，所以不能将引用计数存储在另一个Arc中。 相反，我们可以两全其美，将数据和引用计数存储在一个结构中，并获得指向该结构的指针。 这还避免了在代码中对两个独立的指针进行解引用，也可以提高性能。（耶！）

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

经过这些更改，我们的新结构将如下所示：

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
