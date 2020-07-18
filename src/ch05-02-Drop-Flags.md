# Drop Flags(Drop Flags)

上一节中的示例为Rust引入了一个有趣的问题.我们已经看到,可以完全安全地有条件地初始化,取消初始化和重新初始化内存位置.对于Copy类型,这并不是特别值得注意,因为它们只是一堆随机的比特.但是,带有析构函数的类型是一个不同的故事:Rust需要知道每当赋值变量时是否调用析构函数,或者变量超出作用域.如何通过条件初始化来做到这一点?

请注意,这不是所有赋值都需要担心的问题.特别是,无条件地通过解引用进行赋值,并且无条件地在`let`中赋值不会删除:

```Rust
let mut x = Box::new(0); // let makes a fresh variable, so never need to drop
let y = &mut x;
*y = Box::new(1); // Deref assumes the referent is initialized, so always drops
```

当覆盖先前初始化的变量或其子字段之一时,这只是一个问题.

事实证明,Rust实际上跟踪是否应该删除类型或者 *在运行时(at runtime)* .当变量初始化和未初始化时,将切换该变量的 *删除标志(drop flag)* .当可能需要删除变量时,将评估此标志以确定是否应删除该变量.

当然,通常情况下,可以在程序的每个点静态地知道值的初始化状态.如果是这种情况，那么编译器理论上可以生成更高效的代码!例如,直线(straight-line)代码具有这样的 *静态删除语义(static drop semantics)* :

```Rust
let mut x = Box::new(0); // x was uninit; just overwrite.
let mut y = x;           // y was uninit; just overwrite and make x uninit.
x = Box::new(0);         // x was uninit; just overwrite.
y = x;                   // y was init; Drop y, overwrite it, and make x uninit!
                         // y goes out of scope; y was init; Drop y!
                         // x goes out of scope; x was uninit; do nothing.
```

类似地,分支代码在所有分支在初始化方面具有相同行为时,具有静态删除语义:

```Rust
# let condition = true;
let mut x = Box::new(0);    // x was uninit; just overwrite.
if condition {
    drop(x)                 // x gets moved out; make x uninit.
} else {
    println!("{}", x);
    drop(x)                 // x gets moved out; make x uninit.
}
x = Box::new(0);            // x was uninit; just overwrite.
                            // x goes out of scope; x was init; Drop x!
```

但是,这样的代码 *需要(requires)* 运行时信息才能正确删除:

```Rust
# let condition = true;
let x;
if condition {
    x = Box::new(0);        // x was uninit; just overwrite.
    println!("{}", x);
}
                            // x goes out of scope; x might be uninit;
                            // check the flag!
```

当然,在这种情况下,检索静态删除语义是微不足道的:

```Rust
# let condition = true;
if condition {
    let x = Box::new(0);
    println!("{}", x);
}
```

删除标记在栈上被跟踪,不再存储在实现drop的类型中.
