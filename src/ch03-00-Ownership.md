# 所有权和生命周期(Ownership and Lifetimes)

所有权是Rust的突出特性.它允许Rust完全内存安全且高效,同时避免垃圾回收.在详细介绍所有权系统之前,我们将考虑这个设计的动机.

我们假设你接受垃圾收集(GC)并不总是最佳解决方案,并且希望在某些上下文中手动管理内存.如果你不接受,我能用让你对另一种语言感兴趣吗?

无论你对GC的感觉如何,使代码安全显然是一个巨大的福音.你永远不必担心过早消失的事情(尽管你是否仍然想要指出那件事是另一个问题......).这是C和C++程序需要处理的普遍问题.考虑一下这个简单的错误,即我们所有使用非GC语言的人都曾犯过:

```Rust
fn as_str(data: &u32) -> &str {
    // compute the string
    let s = format!("{}", data);

    // OH NO! We returned a reference to something that
    // exists only in this function!
    // Dangling pointer! Use after free! Alas!
    // (this does not compile in Rust)
    &s
}
```

这正是Rust的所有权系统要解决的问题.Rust知道`&s`存活的作用域,因此可以防止它逃逸.然而,这是一个简单的案例,即使是C编译器也可以合理地捕获.随着代码越来越大,指针通过各种各样的函数,各种事情变得越来越复杂.最终,C编译器将崩溃,并且无法执行足够的逃逸分析来证明你的代码不健全.因此,它将被迫接受你的程序,假设它是正确的.

这将永远不会发生在Rust上.程序员可以向编译器证明一切都是正确的.

当然,Rust关于所有权的故事要比仅仅验证引用不会超出其引用对象的作用域复杂得多.这是因为确保指针总是有效的要复杂得多.例如,在此代码中,

```Rust
let mut data = vec![1, 2, 3];
// get an internal reference
let x = &data[0];

// OH NO! `push` causes the backing storage of `data` to be reallocated.
// Dangling pointer! Use after free! Alas!
// (this does not compile in Rust)
data.push(4);

println!("{}", x);
```

天真的作用域分析不足以防止这个bug,因为`data`实际上可以像我们需要的那样长时间存在.然而,当我们引用它时它被 *改变了(change)* .这就是为什么Rust要求任何引用冻结引用对象及其所有者.
