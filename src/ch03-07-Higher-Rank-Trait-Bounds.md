# 高阶Trait限制(HRTBs)(Higher-Rank Trait Bounds(HRTBs))

Rust的`Fn`trait有点神奇.例如,我们可以编写以下代码:

```Rust
struct Closure<F> {
    data: (u8, u16),
    func: F,
}

impl<F> Closure<F>
    where F: Fn(&(u8, u16)) -> &u8,
{
    fn call(&self) -> &u8 {
        (self.func)(&self.data)
    }
}

fn do_it(data: &(u8, u16)) -> &u8 { &data.0 }

fn main() {
    let clo = Closure { data: (0, 1), func: do_it };
    println!("{}", clo.call());
}
```

如果我们尝试以与生命周期部分相同的方式天真地脱糖这些代码,我们会遇到一些麻烦:

```Rust
struct Closure<F> {
    data: (u8, u16),
    func: F,
}

impl<F> Closure<F>
    // where F: Fn(&'??? (u8, u16)) -> &'??? u8,
{
    fn call<'a>(&'a self) -> &'a u8 {
        (self.func)(&self.data)
    }
}

fn do_it<'b>(data: &'b (u8, u16)) -> &'b u8 { &'b data.0 }

fn main() {
    'x: {
        let clo = Closure { data: (0, 1), func: do_it };
        println!("{}", clo.call());
    }
}
```

我们究竟应该怎样用`F`的trait限制来表达生命周期?我们需要在那里提供某个生命周期,但是在我们进入`call`主体之前,我们关心的生命周期无法命名!而且,这不是某个固定的生命周期;`call`适用于 *任何(any)* 生命周期,`&self`恰好在那一点上.

这项工作需要高阶Trait限制的(HRTBs)魔力.我们脱糖方式如下:

```Rust
where for<'a> F: Fn(&'a (u8, u16)) -> &'a u8,
```

(其中`Fn(a, b, c) -> d`本身就是不稳定的 *真实(real)* `Fn`trait的糖)

`for<'a>`可以读作"对于`'a`的所有选择(for all choices of `'a`)",并且基本上产生F必须满足的trait 限制的 *无限列表(infinite list)* .Intense.在我们遇到HRTBs的`Fn`traits之外的地方并不多,甚至对于那些常见情况我们都有一个很好的魔法糖的.