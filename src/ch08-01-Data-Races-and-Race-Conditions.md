# 数据竞争和竞争条件(Data Races and Race Conditions)

Safe Rust保证不存在数据竞争,其定义为:

- 两个或多个线程同时访问内存位置

- 其中一个或多个是写

- 其中一个或多个是不同步的

数据竞争具有未定义行为,因此无法在Safe Rust中执行.数据竞争 *主要(mostly)* 通过Rust的所有权系统来防止:不可能为可变引用添加别名,因此无法执行数据竞争.内部可变性使这更加复杂,这很大程度上是我们拥有Send和Sync trait的原因(见下文).

**但Rust并不能阻止一般的竞争条件.**

这基本上是不可能的,而且可能真的不合需要.你的硬件很活泼,你的操作系统(OS)很活泼,你电脑上的其他程序都很活泼,而这一切都是充满活力的世界.任何可以真正声称可以防止 *所有(all)* 竞争条件的系统都会非常糟糕,如果不是不正确的话.

因此,对于Safe Rust程序来说,使用死锁或使用不正确的同步做一些荒谬的事情是完全"好(fine)"的.显然这样的程序不是很好,但Rust到目前为止只能抓住你的手.但是,竞争条件本身不能违反Rust程序中的内存安全性.只有与其他一些不安全代码一起使用,竞争条件才会真正违反内存安全.例如:

```Rust
use std::thread;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;

let data = vec![1, 2, 3, 4];
// Arc so that the memory the AtomicUsize is stored in still exists for
// the other thread to increment, even if we completely finish executing
// before it. Rust won't compile the program without it, because of the
// lifetime requirements of thread::spawn!
let idx = Arc::new(AtomicUsize::new(0));
let other_idx = idx.clone();

// `move` captures other_idx by-value, moving it into this thread
thread::spawn(move || {
    // It's ok to mutate idx because this value
    // is an atomic, so it can't cause a Data Race.
    other_idx.fetch_add(10, Ordering::SeqCst);
});

// Index with the value loaded from the atomic. This is safe because we
// read the atomic memory only once, and then pass a copy of that value
// to the Vec's indexing implementation. This indexing will be correctly
// bounds checked, and there's no chance of the value getting changed
// in the middle. However our program may panic if the thread we spawned
// managed to increment before this ran. A race condition because correct
// program execution (panicking is rarely correct) depends on order of
// thread execution.
println!("{}", data[idx.load(Ordering::SeqCst)]);

use std::thread;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;

let data = vec![1, 2, 3, 4];

let idx = Arc::new(AtomicUsize::new(0));
let other_idx = idx.clone();

// `move` captures other_idx by-value, moving it into this thread
thread::spawn(move || {
    // It's ok to mutate idx because this value
    // is an atomic, so it can't cause a Data Race.
    other_idx.fetch_add(10, Ordering::SeqCst);
});

if idx.load(Ordering::SeqCst) < data.len() {
    unsafe {
        // Incorrectly loading the idx after we did the bounds check.
        // It could have changed. This is a race condition, *and dangerous*
        // because we decided to do `get_unchecked`, which is `unsafe`.
        println!("{}", data.get_unchecked(idx.load(Ordering::SeqCst)));
    }
}
```
