# Unsafe Rust可以做什么(What Unsafe Rust Can Do)

在Unsafe Rust中唯一不同的是你可以:

- 解引用原始指针

- 调用`unsafe`函数(包括C函数,编译器内在函数和原始分配器)

- 实现`unsafe`traits

- 改变静态(Mutate statics)

- 访问`union`域

仅此而已.这些操作被降级为Unsafe的原因是滥用任何这些东西都会导致可怕的Undefined Behavior.调用Undefined Behavior使编译器有权对程序执行任意不好的操作.你绝对 *不应该(should not)* 调用Undefined Behavior.

与C不同,Undefined Behavior在Rust中的范围非常有限.所有核心语言都在关注以下事项:

- 解引用(使用`*`运算符)null,悬空或未对齐的指针,或带有无效元数据的胖指针(请参见下文)

- 为`enum`/`struct`/数组/切片/元组字段地址的计算执行越界运算

- 读取[未初始化的内存](https://github.com/rust-lang-nursery/nomicon/blob/master/src/uninitialized.html)

- 打破[指针别名规则](https://github.com/rust-lang-nursery/nomicon/blob/master/src/references.html)

- 生成无效的原始值(单独或作为复合类型(如`enum`/`struct`/数组/元组)的字段):
  - 一个不是0或1的`bool`
  
  - 未定义的`enum`判别式
  
  - 空(null)`fn`指针
  
  - 范围[0x0,0xD7FF]和[0xE000,0x10FFFF]之外的`char`
  
  - `!`(所有值对该类型均无效)

  - 悬空/空(null)/未对齐的引用,本身指向无效值的引用,或带有无效元数据的胖引用(指向动态大小类型)
    - 如果切片的总大小在内存中大于`isize::MAX`字节,那么切片元数据无效
    - `dyn Trait`元数据是无效的,如果它不是一个`Trait`的虚函数表的指针，该`Trait`与引用指向的实际动态trait匹配
  
  - 非utf8`str`

  - 未初始化的整数(`i*`/`u*`)或浮点值(`f*`)

  - 具有自定义无效值的无效库类型,如`NonNull`或`NonZero*`,即0

- 解开(Unwinding)进入另一种语言

- 导致[数据竞争](https://github.com/rust-lang-nursery/nomicon/blob/master/src/races.html)

- 执行使用当前执行线程不支持的目标特性编译的代码(请参阅[`target_feature`](https://doc.rust-lang.org/reference/attributes/codegen.html#the-target_feature-attribute)

"生成"值发生在赋值,传递给函数/原始操作或从函数/原始操作返回的任何时候.

如果引用/指针所指向的字节不都是同一个分配的一部分，则它是"悬空的". 它指向的字节范围由指针值和它指向的对象类型的大小确定.

仅此而已.这就是Rust中Undefined Behavior的所有原因.当然,不安全的函数和traits可以自由地声明程序必须维护的任意其他约束,以避免Undefined Behavior.例如,分配器API声明解除分配未分配的内存是Undefined Behavior.

但是,违反这些限制通常只会导致上述问题之一.一些额外的约束也可能来自编译器内在函数,这些内在函数对如何优化代码做出特殊假设.例如,`Vec`和`Box`使用了要求它们的指针始终为非空(null)的内在函数,

Rust对于其他可疑的操作是相当宽容的.Rust认为以下行为是"安全的":

- 死锁

- 有[竞争条件](https://github.com/rust-lang-nursery/nomicon/blob/master/src/races.html)

- 内存泄漏

- 调用析构函数失败

- 整数溢出

- 中止程序

- 删除生产数据库

然而,任何实际上设法做这种事情的程序都 *可能(probably)* 是不正确的.Rust提供了许多工具来使这些事情变得罕见,但是绝对阻止这些问题是不切实际的.
