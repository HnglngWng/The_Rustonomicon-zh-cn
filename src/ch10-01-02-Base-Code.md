# 基本代码(Base Code)

现在我们已经确定了Arc实现的布局，让我们创建一些基本代码。

## 构建Arc(Constructing the Arc)

我们首先需要一种构造`Arc<T>`的方法。

这非常简单，因为我们只需要将`ArcInner<T>`框起来，并获得一个指向它的`NonNull<T>`指针。

我们从1开始引用计数器，因为第一个引用是当前引用指针。当`Arc`被克隆或删除时，它会被更新。可以对`NonNull`返回的`Option`调用`unwrap()`,因为`Box::into_raw`保证返回的指针不为空。

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
            _marker: PhantomData,
        }
    }
}
```

## Send和Sync

因为我们正在构建一个并发原语，所以我们需要能够跨线程发送它。因此，我们可以实现`Send`和`Sync`标记trait。有关这些的更多信息，请参阅[`Send`和`Sync`一节](ch08-02-Send-and-Sync.md)。

这很好，因为:

- 只有当且仅当它是引用该数据的唯一`Arc`时，您才能获得对`Arc`内值的可变引用

- 我们使用原子计数器进行引用计数

```Rust
unsafe impl<T: Sync + Send> Send for Arc<T> {}
unsafe impl<T: Sync + Send> Sync for Arc<T> {}
```

我们需要限定`T: Sync + Send`，因为如果我们不提供这些边界，就有可能通过`Arc`跨线程边界共享线程不安全的值，这可能会导致数据竞争或不健全。

## 获取`ArcInner`

现在，我们要创建一个私有帮助函数`inner()`，它只是返回解引用的`NonNull`指针。

要将`NonNull<T>`指针解引用为`&T`，我们可以调用`NonNull::as_ref`。 这是不安全的，不像典型的`as_ref`函数，所以我们必须像这样调用它：

```Rust
// inside the impl<T> Arc<T> block from before:
fn inner(&self) -> &ArcInner<T> {
    unsafe { self.ptr.as_ref() }
}
```

这种不安全是没有问题的，因为当这个`Arc`存在时，我们保证内部指针是有效的。

这是本节的所有代码：

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
            _marker: PhantomData,
        }
    }

    fn inner(&self) -> &ArcInner<T> {
        // This unsafety is okay because while this Arc is alive, we're
        // guaranteed that the inner pointer is valid.
        unsafe { self.ptr.as_ref() }
    }
}

unsafe impl<T: Sync + Send> Send for Arc<T> {}
unsafe impl<T: Sync + Send> Sync for Arc<T> {}
```
