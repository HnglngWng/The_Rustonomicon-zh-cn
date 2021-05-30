# Dropping

我们现在需要一种方法来减少引用计数,并在计数足够低时删除数据，否则数据将永远存在于堆中。

为此，我们可以实现`Drop`。

基本上，我们需要：

- 获取`Arc`的`ArcInner`值

- 减少引用计数

- 如果数据只剩下一个引用，则：

- 原子地隔离数据以防止数据的使用和删除的重新排序，然后：

- 删除内部数据

现在，我们需要减少引用计数。 如果引用计数不等于 1（因为`fetch_sub`返回前一个值），我们还可以通过返回来引入第 3 步：

```Rust
if self.inner().rc.fetch_sub(1, Ordering::Release) != 1 {
    return;
}
```

然后我们需要创建一个原子原子围栏,来防止重新排序数据的使用和删除数据。 如[标准库的`Arc`实现](https://github.com/rust-lang/rust/blob/e1884a8e3c3e813aada8254edfa120e85bf5ffca/library/alloc/src/sync.rs#L1440-L1467)中所述：

> 需要此围栏以防止重新排序使用数据和删除数据。 因为它被标记为`Release`，所以引用计数的减少与这个`Acquire` 围栏同步。 这意味着数据的使用发生在减少引用计数之前，减少引用计数发生在此围栏之前，发生在删除数据之前。
>
> 如[Boost文档](https://www.boost.org/doc/libs/1_55_0/doc/html/atomic/usage_examples.html)中所述，
>
>> 在另一个线程中删除对象之前，强制执行对一个线程中对象的任何可能访问（通过一个现有的引用），这一点很重要。 这是通过删除引用后的“释放”操作（通过此引用对对象的任何访问显然必须发生在之前）和删除对象之前的“获取”操作来实现的。
>
> 特别是，虽然Arc的内容通常是不可变的，但可以对Mutex之类的内容进行内部写入。 由于Mutex在被删除时不会被获取，我们不能依赖它的同步逻辑来使线程A中的写入对线程B中运行的析构函数可见。
>
> 另请注意，此处的Acquire围栏可能会替换为Acquire负载，这可以提高在竞争激烈的情况下的性能。 见[2](https://github.com/rust-lang/rust/pull/41714)。

为此，我们执行以下操作：

```Rust
atomic::fence(Ordering::Acquire);
```

我们需要导入`std::sync::atomic`本身：

```Rust
use std::sync::atomic;
```

最后，我们可以删除数据本身。 我们使用`Box::from_raw`来删除装箱的`ArcInner<T>`及其数据。 它接受一个`*mut T`而不是`NonNull<T>`，因此我们必须使用`NonNull::as_ptr`进行转换。

```Rust
unsafe { Box::from_raw(self.ptr.as_ptr()); }
```

这是安全的，因为我们知道我们有指向`ArcInner`的最后一个指针,并且它的指针是有效的。

现在，让我们把这些都包含在`Drop`实现中：

```Rust
impl<T> Drop for Arc<T> {
    fn drop(&mut self) {
        if self.inner().rc.fetch_sub(1, Ordering::Release) != 1 {
            return;
        }
        // This fence is needed to prevent reordering of the use and deletion
        // of the data.
        atomic::fence(Ordering::Acquire);
        // This is safe as we know we have the last pointer to the `ArcInner`
        // and that its pointer is valid.
        unsafe { Box::from_raw(self.ptr.as_ptr()); }
    }
}
```
