# 检查未初始化的内存 (Checked Uninitialized Memory)

与C一样,Rust中的所有栈变量都是未初始化的,直到为它们显式赋值.与C不同,Rust会在你执行以下操作之前静态地阻止你读取它们:

```Rust
fn main() {
    let x: i32;
    println!("{}", x);
}
```

```Rust
  |
3 |     println!("{}", x);
  |                    ^ use of possibly uninitialized `x`
```

这是基于一个基本的分支分析:每个分支必须在第一次使用之前为`x`赋值.有趣的是,如果每个分支只赋值一次,Rust不要求变量是可变的来执行延迟初始化.然而,分析没有利用常数分析或类似的东西.所以这个可以编译:

```Rust
fn main() {
    let x: i32;

    if true {
        x = 1;
    } else {
        x = 2;
    }

    println!("{}", x);
}
```

但是这个不可以:

```Rust
fn main() {
    let x: i32;
    if true {
        x = 1;
    }
    println!("{}", x);
}
```

```Rust
  |
6 |     println!("{}", x);
  |                    ^ use of possibly uninitialized `x`
```

而这样可以:

```Rust
fn main() {
    let x: i32;
    if true {
        x = 1;
        println!("{}", x);
    }
    // Don't care that there are branches where it's not initialized
    // since we don't use the value in those branches
}
```

当然,虽然分析没有考虑实际值,但它确实对依赖关系和控制流有相对复杂的理解.例如,这有效:

```Rust
let x: i32;

loop {
    // Rust doesn't understand that this branch will be taken unconditionally,
    // because it relies on actual values.
    if true {
        // But it does understand that it will only be taken once because
        // we unconditionally break out of it. Therefore `x` doesn't
        // need to be marked as mutable.
        x = 0;
        break;
    }
}
// It also knows that it's impossible to get here without reaching the break.
// And therefore that `x` must be initialized here!
println!("{}", x);
```

如果某个值被移出变量,那么如果值的类型不是Copy,则该变量在逻辑上将变为未初始化.那是:

```Rust
fn main() {
    let x = 0;
    let y = Box::new(0);
    let z1 = x; // x is still valid because i32 is Copy
    let z2 = y; // y is now logically uninitialized because Box isn't Copy
}
```

但是,在此示例中重新赋值`y`将需要 *将(would)* `y`标记为可变,因为Safe Rust程序可以观察到`y`的值已更改:

```Rust
fn main() {
    let mut y = Box::new(0);
    let z = y; // y is now logically uninitialized because Box isn't Copy
    y = Box::new(1); // reinitialize y
}
```

否则就好像`y`是一个全新的变量.