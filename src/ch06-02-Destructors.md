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

这样可以正常工作,因为当Rust删除`ptr`字段时,它只会看到一个没有实际`Drop`实现的Unique.同样地,没有任何东西可以在释放后使用(use-after-free)`ptr`,因为当drop退出时,它变得无法访问.

但是这不起作用:

```Rust
#![feature(allocator_api, ptr_internals)]

use std::alloc::{Alloc, Global, GlobalAlloc, Layout};
use std::ptr::{drop_in_place, Unique, NonNull};
use std::mem;

struct Box<T>{ ptr: Unique<T> }

impl<T> Drop for Box<T> {
    fn drop(&mut self) {
        unsafe {
            drop_in_place(self.ptr.as_ptr());
            let c: NonNull<T> = self.ptr.into();
            Global.dealloc(c.cast(), Layout::new::<T>());
        }
    }
}

struct SuperBox<T> { my_box: Box<T> }

impl<T> Drop for SuperBox<T> {
    fn drop(&mut self) {
        unsafe {
            // Hyper-optimized: deallocate the box's contents for it
            // without `drop`ing the contents
            let c: NonNull<T> = self.my_box.ptr.into();
            Global.dealloc(c.cast::<u8>(), Layout::new::<T>());
        }
    }
}
# fn main() {}
```

在我们在SuperBox的析构函数中释放`box`的ptr之后,Rust会愉快地继续告诉box删除(Drop)自己,所有东西都会出问题,释放后使用(use-after-frees)和双重释放(double-frees).

请注意,递归删除行为适用于所有结构和枚举,无论它们是否实现Drop.因此像

```Rust
struct Boxy<T> {
    data1: Box<T>,
    data2: Box<T>,
    info: u32,
}
```

这样的,只要它"被"删除就会有data1和data2的字段析构函数,即使它本身没有实现Drop.我们说这种类型 *需要Drop(needs Drop)* ,即使它本身不是Drop.

同样的,

```Rust
enum Link {
    Next(Box<Link>),
    None,
}
```

当且仅当实例存储Next变体时,才会删除其内部Box字段.

一般情况下,这非常有效,因为在重构数据布局时无需担心添加/移除删除(drops).仍然有许多有效的用例需要使用析构函数做更棘手的事情.

重写递归删除并允许在`drop`期间移出Self的经典安全解决方案是使用Option:

```Rust
#![feature(allocator_api, ptr_internals)]

use std::alloc::{Alloc, GlobalAlloc, Global, Layout};
use std::ptr::{drop_in_place, Unique, NonNull};
use std::mem;

struct Box<T>{ ptr: Unique<T> }

impl<T> Drop for Box<T> {
    fn drop(&mut self) {
        unsafe {
            drop_in_place(self.ptr.as_ptr());
            let c: NonNull<T> = self.ptr.into();
            Global.dealloc(c.cast(), Layout::new::<T>());
        }
    }
}

struct SuperBox<T> { my_box: Option<Box<T>> }

impl<T> Drop for SuperBox<T> {
    fn drop(&mut self) {
        unsafe {
            // Hyper-optimized: deallocate the box's contents for it
            // without `drop`ing the contents. Need to set the `box`
            // field as `None` to prevent Rust from trying to Drop it.
            let my_box = self.my_box.take().unwrap();
            let c: NonNull<T> = my_box.ptr.into();
            Global.dealloc(c.cast(), Layout::new::<T>());
            mem::forget(my_box);
        }
    }
}
# fn main() {}
```

然而,这有一个相当奇怪的语义:你说一个应该总是Some的字段 *可能是(may be)* None,只是因为它发生在析构函数中.当然,这反过来说很有意义:你可以在析构函数中调用self上的任意方法,这样可以防止你在取消初始化字段之后这样做.并不是说它会阻止你在那里产生任何其他任意无效的状态.

总的来说,这是一个不错的选择.当然,默认情况下你应该达到的目的.但是,在未来我们希望有一种一流的方式来宣布一个字段不应该被自动删除.