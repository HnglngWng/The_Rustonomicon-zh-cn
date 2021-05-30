# Cloning

现在我们已经设置了一些基本的代码，我们需要一种方法来克隆`Arc`。

基本上，我们需要：

- 获取`Arc`的`ArcInner`值

- 增加原子引用计数

- 从内部指针构造一个`Arc`的新实例

接下来，我们可以按如下方式更新原子引用计数：

```Rust
self.inner().rc.fetch_add(1, Ordering::Relaxed);
```

如[标准库的`Arc`克隆实现](https://github.com/rust-lang/rust/blob/e1884a8e3c3e813aada8254edfa120e85bf5ffca/library/alloc/src/sync.rs#L1171-L1181)中所述：

> 在这里使用宽松的排序是没问题的，因为原始引用的知识可以防止其他线程错误地删除对象。
>
> 如[Boost文档](https://www.boost.org/doc/libs/1_55_0/doc/html/atomic/usage_examples.html)中所述：
>
>> 始终可以使用 memory_order_relaxed 来增加引用计数器：对对象的新引用只能从现有引用形成，并且将现有引用从一个线程传递到另一个线程必须已经提供了任何所需的同步。

我们需要添加另一个导入以使用`Ordering`：

```Rust
use std::sync::atomic::Ordering;
```

在一些人为设计的程序中（例如使用 `mem::forget`），引用计数可能会溢出，但在任何合理的程序中都会发生这种情况是不合理的。

然后，我们需要返回`Arc`的一个新实例：

```Rust
Self {
    ptr: self.ptr,
    _marker: PhantomData
}
```

现在，让我们把这一切都包含在`Clone`实现中：

```Rust
use std::sync::atomic::Ordering;

impl<T> Clone for Arc<T> {
    fn clone(&self) -> Arc<T> {
        // Using a relaxed ordering is alright here as knowledge of the original
        // reference prevents other threads from wrongly deleting the object.
        self.inner().rc.fetch_add(1, Ordering::Relaxed);
        Self {
            ptr: self.ptr,
            _marker: PhantomData,
        }
    }
}
```
