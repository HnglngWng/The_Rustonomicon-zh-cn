# Unsafe Rust可以做什么(What Unsafe Rust Can Do)

在Unsafe Rust中唯一不同的是你可以:

- 解引用原始指针

- 调用`unsafe`函数(包括C函数,编译器内在函数和原始分配器)

- 实现`unsafe`traits

- 改变静态(Mutate statics)

- 访问`union`域

仅此而已.这些操作被降级为Unsafe的原因是滥用任何这些东西都会导致可怕的Undefined Behavior.调用Undefined Behavior使编译器有权对程序执行任意不好的操作.你绝对 *不应该(should not)* 调用Undefined Behavior.

与C不同,Undefined Behavior在Rust中的范围非常有限.所有核心语言都在关注以下事项:

- 解引用(使用`*`运算符)悬空或未对齐的指针(见下文)

- 为`enum`/`struct`/数组/切片/元组字段地址的计算执行越界运算

- 打破[指针别名规则](https://github.com/rust-lang-nursery/nomicon/blob/master/src/references.html)

- 调用具有错误的调用ABI的函数或从具有错误的展开ABI的函数展开.

- 导致[数据竞争](https://github.com/rust-lang-nursery/nomicon/blob/master/src/races.html)

- 执行使用当前执行线程不支持的[目标特性](https://doc.rust-lang.org/reference/attributes/codegen.html#the-target_feature-attribute)编译的代码

- 生成无效的值(单独或作为复合类型(如`enum`/`struct`/数组/元组)的字段):
  - 一个不是0或1的`bool`
  
  - 带有无效判别的`enum`
  
  - 空(null)`fn`指针
  
  - 范围[0x0,0xD7FF]和[0xE000,0x10FFFF]之外的`char`
  
  - `!`(所有值对该类型均无效)

  - 从[未初始化的内存](https://github.com/rust-lang-nursery/nomicon/blob/master/src/uninitialized.html)读取的整数(`i*`/`u*`),浮点值(`f*`)或原始指针,或`str`中的未初始化内存.

  - 悬空,未对齐,或指向无效值的引用/`Box`.
  
  - 有无效元数据的宽引用,`Box`,或原始指针:
    - `dyn Trait`元数据是无效的,如果它不是一个`Trait`的虚函数表的指针，该`Trait`与指针或引用指向的实际动态trait匹配

    - 如果长度不是有效的`usize`,则切片元数据无效(即,不得从未初始化的内存中读取)
  
  - 具有自定义无效值的类型,该值属于这些值之一,例如为null的`NonNull`.(请求自定义无效值是一个不稳定的特性,但是某些稳定的libstd类型,如`NonNull`,会使用它.)

"生成"值发生在赋值,传递给函数/原始操作或从函数/原始操作返回的任何时候.

如果引用/指针是空的或者它所指向的字节不都是同一个分配的一部分(因此特别是它们都必须是 *某一(some)* 分配的一部分),则它是"悬空的".它指向的字节范围由指针值和它指向的对象类型的大小确定.因此,如果范围是空的,"悬空"与"非空"相同.请注意,切片和字符串指向其整个范围,因此,长度元数据永远不要太大是很重要的.(尤其是分配,因此切片和字符串不能大于`isize::MAX`字节).如果由于某种原因这太麻烦了,请考虑使用原始指针.

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
