# Unsafe Rust可以做什么(What Unsafe Rust Can Do)

在Unsafe Rust中唯一不同的是你可以:

- 解引用原始指针

- 调用`unsafe`函数(包括C函数,编译器内在函数和原始分配器)

- 实现`unsafe`traits

- 改变静态(Mutate statics)

- 访问`union`域

仅此而已.这些操作被降级为Unsafe的原因是滥用任何这些东西都会导致可怕的Undefined Behavior.调用Undefined Behavior使编译器有权对程序执行任意不好的操作.你绝对 *不应该(should not)* 调用Undefined Behavior.

与C不同,Undefined Behavior在Rust中的范围非常有限.所有核心语言都在关注以下事项:

- 解引用null,悬空或未对齐的指针

- 读取[未初始化的内存](https://github.com/rust-lang-nursery/nomicon/blob/master/src/uninitialized.html)

- 打破[指针别名规则](https://github.com/rust-lang-nursery/nomicon/blob/master/src/references.html)

- 生成无效的原始值:
  - 悬空/空(null)引用
  
  - 空(null)`fn`指针
  
  - 一个不是0或1的`bool`
  
  - 未定义的`enum`判别式
  
  - 范围[0x0,0xD7FF]和[0xE000,0x10FFFF]之外的`char`
  
  - 非utf8`str`

- 解开(Unwinding)进入另一种语言

- 导致[数据竞争](https://github.com/rust-lang-nursery/nomicon/blob/master/src/races.html)

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
