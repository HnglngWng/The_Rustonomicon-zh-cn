# 析构函数(Destructors)

该语言 *是(does)* 通过`Drop`trait提供的成熟的自动析构函数,它提供了以下方法:

```Rust
fn drop(&mut self);
```

这种方法给出了类型时间以某种方式完成它正在做的事情.

**在运行`drop`之后,Rust会尝试递归地删除`self`的所有字段.**

这是一个方便的功能,因此你不必编写"析构函数样板"来删除子项.如果一个结构没有特殊逻辑用于删除其它东西而不是删除它的子节点,那么这意味着`Drop`根本不需要实现!

**在Rust 1.0中没有可靠的方法来防止这种行为.**

注意,接受`&mut self`意味着即使你可以抑制递归Drop,Rust也会阻止你,如将字段移出self.对于大多数类型,这是完全正常的.

例如,`Box`的自定义实现可能会像这样写`Drop`:

```Rust
#![feature(ptr_internals, allocator_api)]

use std::alloc::{Alloc, Global, GlobalAlloc, Layout};
use std::mem;
use std::ptr::{drop_in_place, NonNull, Unique};

struct Box<T>{ ptr: Unique<T> }

impl<T> Drop for Box<T> {
    fn drop(&mut self) {
        unsafe {
            drop_in_place(self.ptr.as_ptr());
            let c: NonNull<T> = self.ptr.into();
            Global.dealloc(c.cast(), Layout::new::<T>())
        }
    }
}
# fn main() {}
```
