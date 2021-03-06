# 生命周期的限制(Limits of Lifetimes)

给出以下代码:

```Rust
#[derive(Debug)]
struct Foo;

impl Foo {
    fn mutate_and_share(&mut self) -> &Self { &*self }
    fn share(&self) {}
    println!("{:?}", loan);
}

fn main() {
    let mut foo = Foo;
    let loan = foo.mutate_and_share();
    foo.share();
}
```

人们可能期望它能够编译.我们调用`mutate_and_share`,它暂时地可变借用`foo`,但后来只返回一个共享引用.因此我们希望`foo.share()`成功,因为`foo`不应该被可变地借用.

但是当我们尝试编译它时:

```Rust
error[E0502]: cannot borrow `foo` as immutable because it is also borrowed as mutable
  --> src/main.rs:12:5
   |
11 |     let loan = foo.mutate_and_share();
   |                --- mutable borrow occurs here
12 |     foo.share();
   |     ^^^ immutable borrow occurs here
13 |     println!("{:?}", loan);
```

发生了什么?好吧,我们得到了与上一节中的示例2完全相同的推理.我们脱糖该程序,得到以下结果:

```Rust
struct Foo;

impl Foo {
    fn mutate_and_share<'a>(&'a mut self) -> &'a Self { &'a *self }
    fn share<'a>(&'a self) {}
}

fn main() {
    'b: {
        let mut foo: Foo = Foo;
        'c: {
            let loan: &'c Foo = Foo::mutate_and_share::<'c>(&'c mut foo);
            'd: {
                Foo::share::<'d>(&'d foo);
            }
            println!("{:?}", loan);
        }
    }
}
```

由于`loan`的生命周期和`mutate_and_share`的签名,生命周期系统被迫将`&mut foo`扩展为以具有生命周期`'c`.然后,当我们试图调用`share`时,它看到我们正试图别名那个`&'c mut foo`,我们就失败了.

根据我们实际关注的引用语义,这个程序显然是正确的,但是生命周期系统太粗粒度,无法处理.

# 不当减少借用(Improperly reduced borrows)

下面的代码无法编译,因为Rust不知道不再需要借用,因此保守地退回到使用整个作用域. 这最终将得到解决.

```Rust
# use std::collections::HashMap;
# use std::hash::Hash;
fn get_default<'m, K, V>(map: &'m mut HashMap<K, V>, key: K) -> &'m mut V
where
    K: Clone + Eq + Hash,
    V: Default,
{
    match map.get_mut(&key) {
        Some(value) => value,
        None => {
            map.insert(key.clone(), V::default());
            map.get_mut(&key).unwrap()
        }
    }
}
```

由于强加的生命周期限制，`&mut map` 的生命周期与其他可变借用重叠，导致编译错误：

```rust
error[E0499]: cannot borrow `*map` as mutable more than once at a time
  --> src/main.rs:12:13
   |
4  |   fn get_default<'m, K, V>(map: &'m mut HashMap<K, V>, key: K) -> &'m mut V
   |                  -- lifetime `'m` defined here
...
9  |       match map.get_mut(&key) {
   |       -     --- first mutable borrow occurs here
   |  _____|
   | |
10 | |         Some(value) => value,
11 | |         None => {
12 | |             map.insert(key.clone(), V::default());
   | |             ^^^ second mutable borrow occurs here
13 | |             map.get_mut(&key).unwrap()
14 | |         }
15 | |     }
   | |_____- returning this value requires that `*map` is borrowed for `'m`

```