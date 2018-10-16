# 构造函数(Constructors)

只有一种方法可以创建用户定义类型的实例:命名它,并立即初始化其所有字段:

```Rust
struct Foo {
    a: u8,
    b: u32,
    c: bool,
}

enum Bar {
    X(u32),
    Y(bool),
}

struct Unit;

let foo = Foo { a: 0, b: 1, c: false };
let bar = Bar::X(0);
let empty = Unit;
```

仅此而已.你创建一个类型的实例的每一种方式都只是调用一个完全普通的函数来完成一些事情并最终触及真正的构造函数(The One True Constructor).

与C++不同，Rust没有大量的内置构造函数.没有复制(Copy),默认(Default),赋值(Assignment),移动(Move)或任何构造函数.造成这种情况的原因是多种多样的,但它在很大程度上归结为Rust的 *显式(being explicit)* 的哲学.

移动构造函数在Rust中没有意义,因为我们不允许类型"关心"它们在内存中的位置.每种类型都必须准备好将它盲目地存储到内存中的其他地方.这意味着纯粹的*在栈上但仍然可移动的(on-the-stack-but- still-movable)*侵入链表根本不会发生在Rust(安全地)中.

类似地,赋值和复制构造函数也不存在,因为移动语义是Rust中唯一的语义.最多`x = y`只是将`y`的位移动到`x`变量中.Rust确实提供了两种用于提供C++面向复制的(copy- oriented)语义的工具:`Copy`和`Clone`.克隆(Clone)是我们复制构造函数的道德等价物,但它永远不会被隐式调用.你必须在要克隆的元素上显式调用`clone`.复制(Copy)是克隆的一个特例,其实现只是"复制位(copy the bits)".Copy类型在移动时 *会(are)* 被隐式克隆,但由于Copy的定义,这意味着不将旧副本视为未初始化--无操作(no-op).

虽然Rust提供了一个`Default`trait来指定默认构造函数的道德等价物,但使用这个trait却极为罕见.这是因为变量未被隐式初始化.默认(Default)基本上只对泛型编程有用.在具体的上下文中,类型将为任何类型提供静态`new`方法用作"默认"构造函数.这与其他语言中的`new`无关,没有特殊含义.这只是一个命名惯例.

TODO: talk about "placement new"?