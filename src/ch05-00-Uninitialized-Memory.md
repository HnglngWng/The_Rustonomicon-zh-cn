# 使用未初始化的内存(Working With Uninitialized Memory)

Rust程序中所有运行时分配的内存都是 *未初始化的(uninitialized)* .在这种状态下,内存中的值是不确定的一堆比特,这些比特可能或甚至不能反映应该驻留在该内存位置的类型的有效状态.尝试将此内存解释为 *任何(any)* 类型的值将导致未定义行为.不要这样做.

Rust提供了以受检查(安全)和未检查(不安全)方式处理未初始化内存的机制.
