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

正如我们在所有权章节中看到的那样,当我们有一个指向我们拥有的分配的原始指针时,我们应该使用`Unique<T>`代替`*mut T`.Unique是不稳定的,所以我们希望尽可能不使用它.

回顾一下,Unique是一个原始指针的包装器,它声明:

- 我们是`T`的变体

- 我们可能拥有类型`T`的值(用于删除检查)

- 如果`T`是Send/Sync,我们是Send/Sync

- 我们的指针永远不为null(因此`Option<Vec<T>>`是空指针优化的)

除了最后一个,我们可以在稳定Rust中实现上述所有要求:

```Rust
use std::marker::PhantomData;
use std::ops::Deref;
use std::mem;

struct Unique<T> {
    ptr: *const T,              // *const for variance
    _marker: PhantomData<T>,    // For the drop checker
}

// Deriving Send and Sync is safe because we are the Unique owners
// of this data. It's like Unique<T> is "just" T.
unsafe impl<T: Send> Send for Unique<T> {}
unsafe impl<T: Sync> Sync for Unique<T> {}

impl<T> Unique<T> {
    pub fn new(ptr: *mut T) -> Self {
        Unique { ptr: ptr, _marker: PhantomData }
    }

    pub fn as_ptr(&self) -> *mut T {
        self.ptr as *mut T
    }
}
```

不幸的是,说明你的价非零(non-zero)的机制是不稳定的,不太可能很快稳定下来.因此,我们只接受打击,使用std的Unique:

```Rust
pub struct Vec<T> {
    ptr: Unique<T>,
    cap: usize,
    len: usize,
}
```

如果你不关心空指针优化,那么你可以使用稳定代码.但是,我们将围绕启用此优化来设计其余代码.应该注意的是,调用`Unique::new`是不安全的,因为在其中放置null是未定义行为.我们的稳定Unique不需要`new`是不安全的,因为它没有对其内容做出任何有趣的保证.