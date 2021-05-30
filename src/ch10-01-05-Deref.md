# Deref

好的。 我们现在有办法创建、克隆和销毁`Arc`，但是我们如何获取里面的数据呢?

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
        &self.inner().data
    }
}
```

很简单吧? 这只是对指向`ArcInner<T>`的`NonNull`指针进行解引用，然后获取对内部数据的引用。
