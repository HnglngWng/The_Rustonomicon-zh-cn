# Splitting Borrows(Splitting Borrows)

当使用复合结构时,可变引用的互斥属性可能非常有限.借用检查器了解一些基本的东西,但很容易摔倒.它确实足够了解结构,以便知道可以同时借用结构的不相交字段.所以现在有效:

```Rust
struct Foo {
    a: i32,
    b: i32,
    c: i32,
}

let mut x = Foo {a: 0, b: 0, c: 0};
let a = &mut x.a;
let b = &mut x.b;
let c = &x.c;
*b += 1;
let c2 = &x.c;
*a += 10;
println!("{} {} {} {}", a, b, c, c2);
```

但是,借用不能以任何方式理解数组或切片,因此这不起作用:

```Rust
let mut x = [1, 2, 3];
let a = &mut x[0];
let b = &mut x[1];
println!("{} {}", a, b);
```

```Rust
error[E0499]: cannot borrow `x[..]` as mutable more than once at a time
 --> src/lib.rs:4:18
  |
3 |     let a = &mut x[0];
  |                  ---- first mutable borrow occurs here
4 |     let b = &mut x[1];
  |                  ^^^^ second mutable borrow occurs here
5 |     println!("{} {}", a, b);
6 | }
  | - first borrow ends here

error: aborting due to previous error
```

虽然借用可以理解这个简单的情况似乎是合理的,但是对于理解一般容器类型(例如树)中的不相交性来说,借助它是非常没有希望的,特别是如果不同的键实际上映射到相同的值.

为了"教(teach)"借用我们正在做的事情是好的,我们需要下降到不安全代码.例如,可变切片暴露`split_at_mut`函数,该函数使用切片并返回两个可变切片.一个用于索引左侧的所有内容,另一个用于右侧的所有内容.直观地,我们知道这是安全的,因为切片不重叠,因此别名.但是,实现需要一些不安全:

```Rust
fn split_at_mut(&mut self, mid: usize) -> (&mut [T], &mut [T]) {
    let len = self.len();
    let ptr = self.as_mut_ptr();
    assert!(mid <= len);
    unsafe {
        (from_raw_parts_mut(ptr, mid),
         from_raw_parts_mut(ptr.offset(mid as isize), len - mid))
    }
}
```

这实际上有点微妙.为了避免将两个`&mut`变为相同的值,我们通过原始指针显式地构建全新的切片.

然而,更微妙的是产生可变引用的迭代器如何工作.迭代器trait定义如下:

```Rust
trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```