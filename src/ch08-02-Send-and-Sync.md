# Send和Sync(Send and Sync)

但是,并非一切都遵循继承的可变性.某些类型允许你在改变它时,拥有多个内存中位置的别名.除非这些类型使用同步(synchronization)来管理此访问,否则它们绝对不是线程安全的.Rust通过Send和Sync trait捕获这一点.

- 如果将其发送到另一个线程是安全的,则类型为Send.

- 如果在线程之间共享是安全的,则类型为Sync(当且仅当`&T`为Send时,T为Sync).

Send和Sync是Rust的并发故事的基础.因此,存在大量特殊工具以使它们正常工作.首先,它们是不安全trait.这意味着它们实现起来不安全,而其他不安全的代码可以假设它们已正确实现.由于它们是 *标记trait(marker traits)* (它们没有像方法那样的关联项),所以正确实现只是意味着它们具有实现者应具有的内在属性.错误地实现Send或Sync可能会导致未定义行为.

Send和Sync也是自动派生的traits.这意味着,与其他所有trait不同,如果类型完全由Send或Sync类型组成,则它是Send或Sync.几乎所有基元都是Send和Sync,因此几乎所有与之交互的类型都是Send和Sync.

主要例外包括:

- 原始指针既不是Send也不是Sync(因为它们没有安全防护).

- `UnsafeCell`不是Sync(因此`Cell`和`RefCell`不是).

- `Rc`不是Send或Sync(因为引用计数(refcount)是共享和不同步的).

`Rc`和`UnsafeCell`从根本上说不是线程安全的:它们启用了不同步的共享可变状态.然而,严格地说,原始指针是,标记为线程不安全更多的是 *棉绒(lint)* .使用原始指针执行任何有用的操作都需要解引用它,这已经是不安全的.从这个意义上说,人们可以争辩说,将它们标记为线程安全是"好的(fine)".

但是,重要的是它们不是线程安全的,以防止包含它们的类型被自动标记为线程安全.这些类型具有非平凡的未经跟踪的所有权,并且他们的作者不太可能一直在思考线程安全性.在`Rc`的情况下,我们有一个很好的例子,它包含一个绝对不是线程安全的`*mut`.

如果需要,非自动派生的类型可以简单地实现它们:

```Rust
struct MyBox(*mut u8);

unsafe impl Send for MyBox {}
unsafe impl Sync for MyBox {}
```

在 *非常罕见的(incredibly rare)* 情况下,类型被不适当地自动派生为Send或Sync,那么人们也可以不实现Send和Sync:

```Rust
#![feature(negative_impls)]

// I have some magic semantics for some synchronization primitive!
struct SpecialThreadToken(u8);

impl !Send for SpecialThreadToken {}
impl !Sync for SpecialThreadToken {}
```

请注意, *它本身(n and of itself)* 不可能错误地派生Send和Sync.只有被其他不安全代码赋予特殊含义的类型才可能因错误Send或Sync而导致问题.

原始指针的大多数用法应该封装在足够的抽象之后,可以派生Send和Sync.例如,Rust的所有标准集合都是Send和Sync(当它们包含Send和Sync类型时),尽管它们普遍使用原始指针来管理分配和复杂的所有权.类似地,这些集合的大多数迭代器都是Send和Sync,因为它们在很大程度上表现为集合的`&`或`&mut`.

## 示例

出于[各种原因](https://manishearth.github.io/blog/2017/01/10/rust-tidbits-box-is-special/)，编译器将 [`Box`](https://doc.rust-lang.org/std/boxed/struct.Box.html) 实现为它自己的特殊内在类型，但是我们可以自己实现一些具有类似行为的东西，以查看何时实现Send 和Sync 是可靠的示例。 我们称它为`Carton`。

我们首先编写代码获取分配在栈上的值,并将其转移到堆。

```rust
# pub mod libc {
#    pub use ::std::os::raw::{c_int, c_void};
#    #[allow(non_camel_case_types)]
#    pub type size_t = usize;
#    extern "C" { pub fn posix_memalign(memptr: *mut *mut c_void, align: size_t, size: size_t) -> c_int; }
# }
use std::{
    mem::{align_of, size_of},
    ptr,
};

struct Carton<T>(ptr::NonNull<T>);

impl<T> Carton<T> {
    pub fn new(value: T) -> Self {
        // Allocate enough memory on the heap to store one T.
        assert_ne!(size_of::<T>(), 0, "Zero-sized types are out of the scope of this example");
        let mut memptr = ptr::null_mut() as *mut T;
        unsafe {
            let ret = libc::posix_memalign(
                (&mut memptr).cast(),
                align_of::<T>(),
                size_of::<T>()
            );
            assert_eq!(ret, 0, "Failed to allocate or invalid alignment");
        };

        // NonNull is just a wrapper that enforces that the pointer isn't null.
        let mut ptr = unsafe {
            // Safety: memptr is dereferenceable because we created it from a
            // reference and have exclusive access.
            ptr::NonNull::new(memptr.cast::<T>())
                .expect("Guaranteed non-null if posix_memalign returns 0")
        };

        // Move value from the stack to the location we allocated on the heap.
        unsafe {
            // Safety: If non-null, posix_memalign gives us a ptr that is valid
            // for writes and properly aligned.
            ptr.as_ptr().write(value);
        }

        Self(ptr)
    }
}
```

这不是很有用，因为一旦我们的用户给我们一个值，他们就无法访问它。 [`Box`](https://doc.rust-lang.org/std/boxed/struct.Box.html) 实现了 [`Deref`](https://doc.rust-lang.org/core/ops/trait.Deref.html) 和 [`DerefMut`](https://doc.rust-lang.org/core/ops/trait.DerefMut.html) ，这样你就可以访问内部值。 让我们这样做。

```rust
use std::ops::{Deref, DerefMut};

# struct Carton<T>(std::ptr::NonNull<T>);
#
impl<T> Deref for Carton<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        unsafe {
            // Safety: The pointer is aligned, initialized, and dereferenceable
            //   by the logic in [`Self::new`]. We require writers to borrow the
            //   Carton, and the lifetime of the return value is elided to the
            //   lifetime of the input. This means the borrow checker will
            //   enforce that no one can mutate the contents of the Carton until
            //   the reference returned is dropped.
            self.0.as_ref()
        }
    }
}

impl<T> DerefMut for Carton<T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        unsafe {
            // Safety: The pointer is aligned, initialized, and dereferenceable
            //   by the logic in [`Self::new`]. We require writers to mutably
            //   borrow the Carton, and the lifetime of the return value is
            //   elided to the lifetime of the input. This means the borrow
            //   checker will enforce that no one else can access the contents
            //   of the Carton until the mutable reference returned is dropped.
            self.0.as_mut()
        }
    }
}
```

最后，让我们考虑一下我们的`Carton`是否为Send 和Sync。 某些东西可以安全地Send，除非它与其他东西共享可变状态,而不强制对其进行独占访问。 每个 `Carton` 都有一个唯一的指针，所以没问题。

```rust
# struct Carton<T>(std::ptr::NonNull<T>);
// Safety: No one besides us has the raw pointer, so we can safely transfer the
// Carton to another thread if T can be safely transferred.
unsafe impl<T> Send for Carton<T> where T: Send {}
```

Sync呢? 为了使 `Carton` Sync，我们必须强制您不能写入存储在 `&Carton` 中的内容，而可以从另一个 `&Carton` 读取或写入相同的内容。 由于您需要一个 `&mut Carton` 来写入指针，并且借用检查器强制可变引用必须是独占的，因此也不存在使 Carton 同步的健全性问题。

```rust
# struct Carton<T>(std::ptr::NonNull<T>);
// Safety: Since there exists a public way to go from a `&Carton<T>` to a `&T`
// in an unsynchronized fashion (such as `Deref`), then `Carton<T>` can't be
// `Sync` if `T` isn't.
// Conversely, `Carton` itself does not use any interior mutability whatsoever:
// all the mutations are performed through an exclusive reference (`&mut`). This
// means it suffices that `T` be `Sync` for `Carton<T>` to be `Sync`:
unsafe impl<T> Sync for Carton<T> where T: Sync  {}
```

当我们断言我们的类型是 `Send`和`Sync` 时，我们通常需要强制每个包含的类型都是 `Send`和`Sync`。 在编写行为类似于标准库类型的自定义类型时，我们可以断言我们有相同的要求。 例如，以下代码断言 `Carton` 是 Send，如果同一种 Box 是 Send，在这种情况下，这就相当于说T是Send。

```rust
# struct Carton<T>(std::ptr::NonNull<T>);
unsafe impl<T> Send for Carton<T> where Box<T>: Send {}
```

现在 `Carton<T>` 有内存泄漏，因为它永远不会释放它分配的内存。 一旦我们解决了这个问题,我们有一个新的要求，我们必须确保我们满足Send：我们需要知道可以在一个指针上调用 `free`，该指针是由在另一个线程上完成的分配产生的。 我们可以在 [`libc::free`](https://linux.die.net/man/3/free) 的文档中检查这是真的。

```rust
# struct Carton<T>(std::ptr::NonNull<T>);
# mod libc {
#     pub use ::std::os::raw::c_void;
#     extern "C" { pub fn free(p: *mut c_void); }
# }
impl<T> Drop for Carton<T> {
    fn drop(&mut self) {
        unsafe {
            libc::free(self.0.as_ptr().cast());
        }
    }
}
```

一个不会发生这种情况的好例子是 MutexGuard：注意[它不是 Send](https://doc.rust-lang.org/std/sync/struct.MutexGuard.html#impl-Send)。 MutexGuard 的实现[使用的库](https://github.com/rust-lang/rust/issues/23465#issuecomment-82730326)要求您确保不会尝试释放在不同线程中获得的锁。 如果您能够将 MutexGuard 发送到另一个线程，析构函数将在您发送到的线程中运行，从而违反了要求。 MutexGuard仍可以是Sync，因为您可以发送到另一个线程的只是`&MutexGuard`，而删除引用没做什么事。

TODO:更好地解释什么可以或不可以是Send或Sync.是否足以吸引数据竞争?
