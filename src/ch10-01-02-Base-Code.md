# 基本代码(Base Code)

现在我们已经确定了Arc实现的布局，让我们创建一些基本代码。

## 构建Arc(Constructing the Arc)

我们首先需要一种构造`Arc<T>`的方法。

这非常简单，因为我们只需要将`ArcInner<T>`框起来，并获得一个指向它的`NonNull<T>`指针。

```Rust
impl<T> Arc<T> {
    pub fn new(data: T) -> Arc<T> {
        // We start the reference count at 1, as that first reference is the
        // current pointer.
        let boxed = Box::new(ArcInner {
            rc: AtomicUsize::new(1),
            data,
        });
        Arc {
            // It is okay to call `.unwrap()` here as we get a pointer from
            // `Box::into_raw` which is guaranteed to not be null.
            ptr: NonNull::new(Box::into_raw(boxed)).unwrap(),
            phantom: PhantomData,
        }
    }
}
```

## Send和Sync

因为我们正在构建一个并发原语，所以我们需要能够跨线程发送它。因此，我们可以实现`Send`和`Sync`标记trait。有关这些的更多信息，请参阅[`Send`和`Sync`一节](ch08-02-Send-and-Sync.md)。

这很好，因为:

- 只有当且仅当它是引用该数据的唯一`Arc`时，您才能获得对`Arc`内值的可变引用（这只发生在`Drop`中）

- 我们使用原子进行共享的可变引用计数

```Rust
unsafe impl<T: Sync + Send> Send for Arc<T> {}
unsafe impl<T: Sync + Send> Sync for Arc<T> {}
```

我们需要限定`T: Sync + Send`，因为如果我们不提供这些边界，就有可能通过`Arc`跨线程边界共享线程不安全的值，这可能会导致数据竞争或不健全。

例如，如果这些限定不存在，`Arc<Rc<u32>>` 将是 `Sync` 或 `Send`，这意味着你可以从 `Arc` 中克隆出 `Rc` 以通过线程发送它 （不创建一个全新的 `Rc`），这将创建数据竞争，因为 `Rc` 不是线程安全的。

## 获取`ArcInner`

要将`NonNull<T>`指针解引用为`&T`，我们可以调用`NonNull::as_ref`。 这是不安全的，不像典型的`as_ref`函数，所以我们必须像这样调用它：

```Rust
unsafe { self.ptr.as_ref() }
```

我们将在此代码中多次使用此代码段（通常使用一个关联的 `let` 绑定）。

这种不安全是没有问题的，因为当这个`Arc`存在时，我们保证内部指针是有效的。

## Deref

好的。 现在我们可以制作`Arc`s（很快就能正确地克隆和销毁它们），但是我们如何获取里面的数据呢?

我们现在需要的是`Deref`的一个实现。

我们需要导入trait：

```rust
use std::ops::Deref;
```

这是实现：

```rust
impl<T> Deref for Arc<T> {
    type Target = T;
    fn deref(&self) -> &T {
        let inner = unsafe { self.ptr.as_ref() };
        &inner.data
    }
}
```

很简单吧? 这只是对指向`ArcInner<T>`的`NonNull`指针进行解引用，然后获取对内部数据的引用。

## 代码

这是本节的所有代码：

```rust
 use std::ops::Deref;

impl<T> Arc<T> {
    pub fn new(data: T) -> Arc<T> {
        // We start the reference count at 1, as that first reference is the
        // current pointer.
        let boxed = Box::new(ArcInner {
            rc: AtomicUsize::new(1),
            data,
        });
        Arc {
            // It is okay to call `.unwrap()` here as we get a pointer from
            // `Box::into_raw` which is guaranteed to not be null.
            ptr: NonNull::new(Box::into_raw(boxed)).unwrap(),
            phantom: PhantomData,
        }
    }
}

unsafe impl<T: Sync + Send> Send for Arc<T> {}
unsafe impl<T: Sync + Send> Sync for Arc<T> {}


impl<T> Deref for Arc<T> {
    type Target = T;
    fn deref(&self) -> &T {
        let inner = unsafe { self.ptr.as_ref() };
        &inner.data
    }
}
```
