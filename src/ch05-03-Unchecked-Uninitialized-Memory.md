# 不检查未初始化的内存(Unchecked Uninitialized Memory)

此规则的一个有趣例外是使用数组.Safe Rust不允许你部分初始化数组.初始化数组时,可以使用`let x = [val; N]`设置相同的值,或者你可以使用`let x = [val1, val2, val3]`单独指定每个成员.不幸的是,这非常严格,特别是如果你需要以更增量或动态的方式初始化数组.

Unsafe Rust为我们提供了一个强大的工具来处理这个问题:[`MaybeUninit`](https://doc.rust-lang.org/core/mem/union.MaybeUninit.html). 此类型可用于处理尚未完全初始化的内存.

使用`MaybeUninit`,我们可以如下初始化一个数组的元素:

```Rust
use std::mem::{self, MaybeUninit};

// Size of the array is hard-coded but easy to change. This means we can't
// use [a, b, c] syntax to initialize the array, though!
const SIZE: usize = 10;

let x = {
    // Create an uninitialized array of `MaybeUninit`. The `assume_init` is
    // safe because the type we are claiming to have initialized here is a
    // bunch of `MaybeUninit`s, which do not require initialization.
    let mut x: [MaybeUninit<Box<u32>>; SIZE] = unsafe {
        MaybeUninit::uninit().assume_init()
    };

    // Dropping a `MaybeUninit` does nothing. Thus using raw pointer
    // assignment instead of `ptr::write` does not cause the old
    // uninitialized value to be dropped.
    // Exception safety is not a concern because Box can't panic
    for i in 0..SIZE {
        x[i] = MaybeUninit::new(Box::new(i as u32));
    }

    // Everything is initialized. Transmute the array to the
    // initialized type.
    unsafe { mem::transmute::<_, [Box<u32>; SIZE]>(x) }
};

dbg!(x);
```

此代码分三步进行:

1. 创建一个`MaybeUninit<T>`数组. 对于当前稳定的Rust,我们必须为此使用不安全的代码:我们取一些未初始化的内存(`MaybeUninit::uninit()`),并声称我们已经完全初始化了它([assume_init()](https://doc.rust-lang.org/core/mem/union.MaybeUninit.html#method.assume_init)) . 这似乎很荒谬,因为我们没有! 这是正确的原因是该数组本身完全由`MaybeUninit`组成,而实际上并不需要初始化. 对于大多数其他类型,执行`MaybeUninit::uninit().assume_init()`会产生该类型的无效实例,因此您会得到一些未定义的行为.

2. 初始化数组.这样做的微妙之处在于,通常,当我们使用`=`来赋值给一个Rust类型检查器认为已经初始化的值(例如`x[i]`)时,存储在左手边的旧值会被删除. 这将是一场灾难. 但是,在本例中,左侧的类型是`MaybeUninit<Box<u32>>`,删除不会做任何事情! 有关此`drop`问题的更多讨论,请参见下文.

3. 最后,我们必须更改数组的类型以删除`MaybeUninit`. 对于当前稳定的Rust,这需要一个`transmute`. 这种转换是合法的,因为在内存中,`MaybeUninit<T>`看起来与`T`相同.
但是,请注意,通常,`Container<MaybeUninit<T>>>`确实与`Container<T>`看起来 *不(not)* 一样! 想象一下,如果`Container`是`Option`,而`T`是`bool`,那么`Option<bool>`会利用`bool`只有两个有效值,但是`Option<MaybeUninit<bool>>`不能做到这一点. 因为`bool`不必初始化.
因此,它取决于`Container`是否允许转换`MaybeUninit`. 对于数组,它是(最终标准库将通过提供适当的方法来确认这一点).

在中间的循环上花费更多的时间是值得的,尤其是赋值运算符及其与`drop`的交互.如果我们这样写

```Rust
*x[i].as_mut_ptr() = Box::new(i as u32); // WRONG!
```

我们实际上会覆盖`Box<u32>`,从而导致未初始化数据的`drop`,这会引起很多悲伤和痛苦.

如果由于某种原因我们不能使用`MaybeUninit::new`,则正确的替代选择是使用[`ptr`](https://doc.rust-lang.org/core/ptr/index.html)模块. 特别是,它提供了三个函数,这些函数使我们能够在不删除旧值的情况下将字节分配到内存中的某个位置:[`write`](https://doc.rust-lang.org/core/ptr/fn.write.html),[`copy`](https://doc.rust-lang.org/core/ptr/fn.copy.html)和[`copy_nonoverlapping`](https://doc.rust-lang.org/core/ptr/fn.copy_nonoverlapping.html).

- `ptr::write(ptr, val)`接受一个`val`并将其移动到`ptr`指向的地址.

- `ptr::copy(src, dest, count)`将计数`count`T将占用的位从src复制到dest.(这相当于memmove--注意参数顺序是相反的!)

- `ptr::copy_nonoverlapping(src, dest, count)`执行`copy`操作,但假设两个内存范围不重叠,则会快一点.(这相当于memcpy--注意参数顺序是相反的!)

不言而喻,这些函数如果被滥用,将会造成严重破坏,或者直接导致未定义行为.这些函数 *本身(themselves)* 唯一需要的是分配你想要读写的位置,并正确对齐它们.然而,将任意位写入内存的任意位置的方式可能会破坏事物基本上是不可数的!

值得注意的是,你不需要担心`ptr::write`风格的诡计的类型没有实现`Drop`或包含`Drop`类型,因为Rust知道不会尝试删除它们.这就是我们在上面的示例中所依赖的.类似地,如果这些字段不包含任何`Drop`类型,你应该能够直接赋值给部分初始化的结构的字段.

但是,在使用未初始化的内存时,你需要始终保持警惕,警惕Rust在完全初始化之前尝试删除你制作的值.通过该变量作用域的每个控制路径必须在结束之前初始化该值(如果它有析构函数).这包括代码恐慌.`MaybeUninit`在这里有所帮助,因为它不会隐式删除其内容--但所有这些真正的意思是在panic的情况下,您将导致已经初始化的部分的内存泄漏,而不是未初始化部分的双重释放.

这就是使用未初始化的内存!基本上没有任何地方需要未初始化的内存,所以如果你要传递它,一定要 *非常(really)* 小心.
