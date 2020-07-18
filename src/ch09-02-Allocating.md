# 分配内存(Allocating Memory)

使用Unique会破坏Vec(以及所有std集合)的一个重要特性:空的Vec实际上根本没有分配.所以如果我们不能分配,也不能在`ptr`中放一个空指针,我们在`Vec::new`中做什么?好吧,我们只是放了一些其它的垃圾在里面!

这非常好,因为我们已经将`cap == 0`作为我们无分配的哨兵.我们甚至不需要在几乎任何代码中专门处理它,因为我们通常需要检查`cap > len`或`len > 0`.这里推荐的Rust值是`mem::align_of::<T>()`.Unique为此提供便利:`Unique::dangling()`.有很多地方我们想要使用`dangling`,因为没有真正的分配,但是`null`会使编译器做坏事.

所以:

```Rust
#![feature(alloc, heap_api)]

use std::mem;

impl<T> Vec<T> {
    fn new() -> Self {
        assert!(mem::size_of::<T>() != 0, "We're not ready to handle ZSTs");
        Vec { ptr: Unique::dangling(), len: 0, cap: 0 }
    }
}
```

我在那里断言,因为零大小的类型需要在我们的整个代码中进行一些特殊的处理,我想暂时推迟这个问题.如果没有这个断言,我们的一些早期草案将会做一些非常糟糕的事情.

接下来,我们需要弄清楚当我们想要空间时实际要做什么.为此,我们需要使用其余的堆API.这些基本上允许我们直接与Rust的分配器(默认为jemalloc)对话.

我们还需要一种处理内存不足(OOM)情况的方法.标准库调用`std::alloc::oom()`,后者又调用`oom`langitem,它以特定于平台的方式中止程序.我们中止和不要恐慌的原因是因为展开可能导致分配发生,当你的分配器返回时,"嘿我没有更多的内存",这似乎是一件坏事.

当然,这有点傻,因为大多数平台实际上并没有以传统方式耗尽内存.如果你合法地开始耗尽所有内存,你的操作系统可能会通过另一种方式终止应用程序.我们触发OOM的最可能方式是一次请求大量的内存(例如理论地址空间的一半).因此,恐慌可能没什么大不了的,也不会有什么坏事发生.尽管如此,我们仍试图尽可能地像标准库一样,所以我们就杀死整个程序.

好的,现在我们可以写增长.粗略地说,我们希望有这样的逻辑:

```Rust
if cap == 0:
    allocate()
    cap = 1
else:
    reallocate()
    cap *= 2
```

但是Rust唯一支持的分配器API非常低级,以至于我们需要做一些额外的工作.我们还需要防止在真正大量分配或空分配时可能出现的某些特殊情况.

特别是,`ptr::offset`会给我们带来很多麻烦,因为它具有LLVM的GEP inbounds指令的语义.如果你有幸没有处理过这个指令,这里是GEP的基本情况:别名分析(alias analysis),别名分析,别名分析.优化编译器能够推断数据依赖和别名是非常重要的.

举个简单的例子,考虑以下代码片段:

```Rust
# let x = &mut 0;
# let y = &mut 0;
*x *= 7;
*y *= 3;
```

如果编译器能够证明`x`和`y`指向内存中的不同位置,那么这两个操作在理论上可以并行执行(例如,将它们加载到不同的寄存器中并独立地处理它们).但是编译器通常不能这样做,因为如果x和y指向内存中的相同位置,则需要对相同的值执行操作,并且之后不能将它们合并.

当你使用GEP inbounds时,你是在明确地告诉LLVM你要执行的偏移是在单个"已分配"实体的范围内.最终的回报是,LLVM可以假设,如果已知两个指针指向两个不相交的对象,那么这些指针的所有偏移量也都知道不是别名(因为你不会只是在内存中的某个随机位置结束).LLVM对处理GEP偏移量进行了大量优化,而内界偏移量是最好的,因此尽可能多地使用它们非常重要.

这就是GEP的意义所在,它怎么会给我们带来麻烦?

第一个问题是我们使用无符号整数索引数组,但GEP(作为结果`ptr::offset`)采用有符号整数.这意味着数组中看似有效的索引的一半会溢出GEP,并且实际上走向错误的方向!因此,我们必须将所有分配限制为`isize::MAX`元素.这实际上意味着我们只需要担心字节大小的对象,因为例如`> isize::MAX` `u16`将真正耗尽系统的所有内存.但是为了避免细微的角落情况,有人将某些`< isize::MAX`对象的数组重新解释为字节,std将所有分配限制为`isize::MAX`字节.

在Rust目前支持的所有64位目标上,我们人为地限制为远远少于所有64位地址空间(现代x64平台仅暴露48位寻址),因此我们可以依赖于先耗尽内存.但是在32位目标上,特别是那些扩展使用更多地址空间(PAE x86或x32)的目标,理论上可以成功分配超过`isize::MAX`字节的内存.

但是,由于这是一个教程,我们不会在这里特别优化,只是无条件地检查,而不是使用聪明的平台特定的`cfg`s.

我们需要担心的另一个角落情况是空的(empty)分配.我们需要担心两种空的分配:对于所有T,`cap = 0`,对于零大小的类型,`cap > 0`.

这些情况很棘手,因为它们归结为LLVM"已分配(allocated)"的含义.LLVM的分配概念比我们通常使用的要抽象得多.因为LLVM需要处理不同语言的语义和自定义分配器,所以它不能真正理解分配.相反,分配背后的主要思想是"不与其他东西重叠(doesn't overlap with other stuff)".也就是说,堆分配,栈分配和全局变量不会随机重叠.是的,这是关于别名分析.因此,在技术上,Rust可以在分配的概念上稍微快速,松散一点,只要它是 *一致的(consistent)* .

回到空的分配情况,有几个地方我们想要偏移0作为泛型代码的结果.问题是:这样做是否一致?对于零大小的类型,我们得出结论,使用任意数量的元素进行GEP内界偏移确实是一致的.这是一个运行时无操作(no-op),因为每个元素都不占用空间,可以假装在`0x01`处分配了无限的零大小类型.没有分配器将分配该地址,因为它们不会分配`0x00`,它们通常分配到高于一个字节的某个最小对齐.另外,通常整个内存的第一页也被保护以免被分配(在许多平台上,总共4k).

然而,对于正大小的类型呢?那个有点棘手.原则上,你可以说偏移0不会给LLVM信息:要么在地址之前要么在之后有一个元素,但它不知道哪个.然而,我们选择保守地假设它可能做坏事.因此,我们将明确防范情况的发生.

*唷(Phew).*

好了,这些废话就讲到这里吧,我们来实际分配一些内存:

```Rust
use std::alloc::oom;

fn grow(&mut self) {
    // this is all pretty delicate, so let's say it's all unsafe
    unsafe {
        // current API requires us to specify size and alignment manually.
        let align = mem::align_of::<T>();
        let elem_size = mem::size_of::<T>();

        let (new_cap, ptr) = if self.cap == 0 {
            let ptr = heap::allocate(elem_size, align);
            (1, ptr)
        } else {
            // as an invariant, we can assume that `self.cap < isize::MAX`,
            // so this doesn't need to be checked.
            let new_cap = self.cap * 2;
            // Similarly this can't overflow due to previously allocating this
            let old_num_bytes = self.cap * elem_size;

            // check that the new allocation doesn't exceed `isize::MAX` at all
            // regardless of the actual size of the capacity. This combines the
            // `new_cap <= isize::MAX` and `new_num_bytes <= usize::MAX` checks
            // we need to make. We lose the ability to allocate e.g. 2/3rds of
            // the address space with a single Vec of i16's on 32-bit though.
            // Alas, poor Yorick -- I knew him, Horatio.
            assert!(old_num_bytes <= (isize::MAX as usize) / 2,
                    "capacity overflow");

            let new_num_bytes = old_num_bytes * 2;
            let ptr = heap::reallocate(self.ptr.as_ptr() as *mut _,
                                        old_num_bytes,
                                        new_num_bytes,
                                        align);
            (new_cap, ptr)
        };

        // If allocate or reallocate fail, we'll get `null` back
        if ptr.is_null() { oom(); }

        self.ptr = Unique::new(ptr as *mut _);
        self.cap = new_cap;
    }
}
```

这里没什么特别棘手的.只需计算大小和对齐,并进行一些仔细的乘法检查.