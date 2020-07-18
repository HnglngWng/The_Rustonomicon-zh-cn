# 不检查未初始化的内存(Unchecked Uninitialized Memory)

此规则的一个有趣例外是使用数组.Safe Rust不允许你部分初始化数组.初始化数组时,可以使用`let x = [val; N]`设置相同的值,或者你可以使用`let x = [val1, val2, val3]`单独指定每个成员.不幸的是,这非常严格,特别是如果你需要以更增量或动态的方式初始化数组.

Unsafe Rust为我们提供了一个强大的工具来处理这个问题:[`mem::uninitialized`](https://doc.rust-lang.org/std/mem/fn.uninitialized.html).该函数假装返回一个值,实际上什么都不做.使用它,我们可以说服Rust已经初始化了一个变量,允许我们通过条件和增量初始化来做一些棘手的事情.

不幸的是,这使我们面临各种各样的问题.根据是否认为变量已初始化,赋值对Rust具有不同的含义.如果它被认为是未初始化的,那么Rust将在语义上只记录未初始化的那些位,而不做任何其他事情.但是,如果Rust认为值已经被初始化,它将尝试`Drop`旧值!由于我们欺骗Rust认为该值已初始化,因此我们无法再安全地使用正常赋值.

如果你正在使用原始系统分配器(allocator),这会返回指向未初始化内存的指针,这也是一个问题.

要处理这个问题,我们必须使用[`ptr`](https://doc.rust-lang.org/std/ptr/index.html)模块.特别是,它提供了三个函数,允许我们将字节分配给内存中的某个位置,而不会删除旧值:[`write`](https://doc.rust-lang.org/std/ptr/fn.write.html),[`copy`](https://doc.rust-lang.org/std/ptr/fn.copy.html)和[`copy_nonoverlapping`](https://doc.rust-lang.org/std/ptr/fn/copy_nonoverlapping.html).

- `ptr::write(ptr, val)`接受一个`val`并将其移动到`ptr`指向的地址.

- `ptr::copy(src, dest, count)`将计数`count`T将占用的位从src复制到dest.(这相当于memmove--注意参数顺序是相反的!)

- `ptr::copy_nonoverlapping(src, dest, count)`执行`copy`操作,但假设两个内存范围不重叠,则会快一点.(这相当于memcpy--注意参数顺序是相反的!)

不言而喻,这些函数如果被滥用,将会造成严重破坏,或者直接导致未定义行为.这些函数 *本身(themselves)* 唯一需要的是分配你想要读写的位置.然而,将任意位写入内存的任意位置的方式可能会破坏事物基本上是不可数的!

综合这些,我们得到以下结果:

```Rust
use std::mem;
use std::ptr;

// size of the array is hard-coded but easy to change. This means we can't
// use [a, b, c] syntax to initialize the array, though!
const SIZE: usize = 10;

let mut x: [Box<u32>; SIZE];

unsafe {
    // convince Rust that x is Totally Initialized
    x = mem::uninitialized();
    for i in 0..SIZE {
        // very carefully overwrite each index without reading it
        // NOTE: exception safety is not a concern; Box can't panic
        ptr::write(&mut x[i], Box::new(i as u32));
    }
}

println!("{:?}", x);
```

值得注意的是,你不需要担心`ptr::write`风格的诡计的类型没有实现`Drop`或包含`Drop`类型,因为Rust知道不会尝试删除它们.类似地,如果这些字段不包含任何`Drop`类型,你应该能够直接赋值给部分初始化的结构的字段.

但是,在使用未初始化的内存时,你需要始终保持警惕,警惕Rust在完全初始化之前尝试删除你制作的值.通过该变量作用域的每个控制路径必须在结束之前初始化该值(如果它有析构函数).这包括代码恐慌.

不小心未初始化的内存通常会导致错误,因此已决定弃用[`mem::uninitialized`](https://doc.rust-lang.org/std/mem/fn.uninitialized.html)函数.[`MaybeUninit`](https://doc.rust-lang.org/std/mem/union.MaybeUninit.html)类型应该替代它,因为其API封装了许多常见的,需要围绕初始化内存执行的操作.现在只是每夜版.

这就是使用未初始化的内存!基本上没有任何地方需要未初始化的内存,所以如果你要传递它,一定要 *非常(really)* 小心.
