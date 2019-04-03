# 替代表示(Alternative representations)

Rust允许你从默认情况下指定替代的数据布局策略.见[reference](https://github.com/rust-rfcs/unsafe-code-guidelines/tree/master/reference/src/representation.md).

# repr(C)

这是最重要的`repr`.它意图相当简单:做C做的事情.字段的顺序,大小和对齐方式正是你对C或C++的期望的那样.你希望通过FFI边界传递的任何类型都应该具有`repr(C)`,因为C是编程世界的通用语言.这对于使用数据布局合理地做更精细的技巧也是必要的,例如将值重新解释为不同的类型.

我们强烈建议你使用[rust-bindgen](https://rust-lang.github.io/rust-bindgen/)和/或[cbindgen](https://github.com/eqrion/cbindgen)来管理你的FFI边界. Rust团队与这些项目密切合作,以确保它们运行稳健,并与当前和未来有关类型布局和表示的保证兼容.

必须牢记与Rust的更奇特的数据布局功能的相互作用.由于其"用于FFI"和"用于布局控制"的双重目的,`repr(C)`可以应用于通过FFI边界时将是无意义或有问题的类型.

- ZST仍然是零大小的,即使这不是C中的标准行为,并且显然与C++中的空类型的行为相反,C++仍然消耗一个字节的空间.

- DST指针(宽指针)和元组不是C中的概念,因此不是FFI安全的.

- 带字段的枚举也不是C或C++中的概念,但[定义了](https://github.com/rust-lang/rfcs/blob/master/text/2195-really-tagged-unions.md)有效的类型桥接.

- 如果`T`是[FFI安全的非可空指针类型](https://github.com/rust-lang-nursery/nomicon/blob/master/src/ffi.html#the-nullable-pointer-optimization),则`Option<T>`保证具有与`T`相同的布局和ABI,因此也是FFI安全的.在撰写本文时,这涵盖了`&`,`&mut`和函数指针,所有这些都永远不会为空(null).

- 元组结构类似于`repr(C)`的结构,因为与结构的唯一区别是字段没有命名.

- 对于无字段枚举,`repr(C)`相当于`repr(u*)`(参见下一节)中的一个.所选大小是目标平台的C应用程序二进制接口(ABI)的默认枚举大小.请注意,C中的枚举表示是实现定义的,因此这实际上是"最佳猜测".特别是,当使用某些标志编译感兴趣的C代码时,这可能是不正确的.

- 具有`repr(C)`或`repr(u*)`的无字段枚举仍然可能不会设置为没有相应变体的整数值,即使这是C或C++中允许的行为.(不安全地)构造与其变体之一不匹配的枚举实例是未定义行为.(这样可以继续正常编写和编译详尽的匹配.)

# repr(transparent)

这只能用于具有单个非零大小字段的结构(可能有其他零大小的字段). 结果是整个结构的布局和ABI保证与一个字段相同.

目标是使单个字段和结构之间的转换成为可能. 一个例子是[`UnsafeCell`](https://github.com/rust-lang-nursery/nomicon/blob/master/std/cell/struct.UnsafeCell.html),它可以转换成它包装的类型.

此外,通过FFI将结构传递到另一侧预期内部字段类型可以保证工作. 特别是,这对于`struct Foo(f32)`始终具有与`f32`相同的ABI是必要的.

更多细节在[RFC](https://github.com/rust-lang/rfcs/blob/master/text/1758-repr-transparent.md)中.

# repr(u*), repr(i*)

这些指定了无字段枚举的大小.如果判别式溢出它必须适合的整数,则会产生编译时错误.你可以通过将溢出元素设置显式为0来手动要求Rust允许此操作.但Rust不允许你创建枚举,其中两个变体具有相同的判别式.

术语"无字段枚举(fieldless enum)"仅表示枚举任何变体中都没有数据.没有`repr(u*)`或`repr(C)`的无字段枚举仍然是Rust native类型,并且没有稳定的ABI表示.添加`repr`会使其被处理与ABI目的的指定整数大小完全相同.

如果枚举具有字段,则效果类似于`repr(C)`的效果,因为存在已定义的类型布局.这使得可以将枚举传递给C代码,或访问类型的原始表示并直接操作其标记和字段.有关详细信息,请参阅[RFC](https://github.com/rust-lang/rfcs/blob/master/text/2195-really-tagged-unions.md).

将明确的`repr`添加到枚举会抑制空指针优化.

这些`reprs`对结构没有影响.

# repr(packed)

`repr(packed)`强制Rust去除任何填充,只将类型与一个字节对齐.这可能会改善内存占用,但可能会产生其他负面影响.

特别是,大多数架构都 *强烈(strongly)* 希望将值对齐.这可能意味着未对齐的加载受到惩罚(x86),甚至是故障(某些ARM芯片).对于直接加载或存储压缩(packed)字段的简单情况,编译器可能能够通过移位和掩码来解决对齐问题.但是,如果你引用压缩字段,则编译器不太可能产生代码以避免未对齐的加载.

**[从Rust 2018开始,这仍然会导致未定义行为.](https://github.com/rust-lang/rust/issues/27060)**

`repr(packed)`不能轻易使用.除非你有极端要求,否则不应使用此选项.

这个repr是`repr(C)`和`repr(rust)`的修饰符.

# repr(align(n))

`repr(align(n))`(其中`n`是2的幂)强制类型具有 *至少(at least)* 为n的对齐.

这可以实现一些技巧,例如确保数组的相邻元素永远不会彼此共享相同的高速缓存行(这可能会加速某些类型的并发代码).

这是`repr(C)`和`repr(rust)`的修饰符. 它与`repr(packed)`不兼容.
