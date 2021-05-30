# 布局(Layout)

首先,我们需要提出结构布局.Vec有三个部分:指向分配的指针,分配的大小以及已初始化的元素数.

天真,这意味着我们只想要这个设计:

```Rust
pub struct Vec<T> {
    ptr: *mut T,
    cap: usize,
    len: usize,
}
```

确实这会编译.不幸的是,这是不正确的.首先,编译器会给我们太严格的可变性.因此,如果预期`&Vec<&'a str>`,则无法使用`&Vec<&'static str>`.更重要的是,它会向删除检查器提供不正确的所有权信息,因为它会保守地假设我们不拥有类型`T`的任何值.有关可变性和删除检查的所有详细信息,请参阅有关所有权和生命周期的章节.

正如我们在所有权章节中看到的那样,当有一个指向它拥有的分配的原始指针时,标准库使用`Unique<T>`代替`*mut T`.Unique是不稳定的,所以我们希望尽可能不使用它.

回顾一下,Unique是一个原始指针的包装器,它声明:

- 我们对`T`协变

- 我们可能拥有类型`T`的值(用于删除检查)

- 如果`T`是Send/Sync,我们是Send/Sync

- 我们的指针永远不为null(因此`Option<Vec<T>>`是空指针优化的)

我们可以在稳定Rust中实现上述所有要求. 为此，我们将使用 [`NonNull<T>`](../std/ptr/struct.NonNull.html)，而不是使用 `Unique<T>`，这是原始指针的另一个包装，它为我们提供了上面的两个属性，即它对 `T` 是协变的,并被声明为永远不会为空。 通过添加`PhantomData<T>`（用于丢弃检查）并在`T` Send/Sync时实现Send/Sync，我们得到与使用`Unique<T>` 相同的结果:

```Rust
use std::ptr::NonNull;
use std::marker::PhantomData;

pub struct Vec<T> {
    ptr: NonNull<T>,
    cap: usize,
    len: usize,
    _marker: PhantomData<T>,
}

unsafe impl<T: Send> Send for Vec<T> {}
unsafe impl<T: Sync> Sync for Vec<T> {}
# fn main() {}
```
