# 删除检查Drop Check)

我们已经看到生命周期如何为我们提供一些相当简单的规则,以确保我们永远不会读取悬空引用.然而,到目前为止,我们只是以包容的方式对待 *活得久(outlives)*的关系.也就是说,当我们谈到`'a: '`b时,`'a`与`'b`活得 *完全(exactly)* 一样长是可以的.乍一看,这似乎是一个毫无意义的区别.没有什么与另一个同时被删除,对吧?这就是为什么我们使用下面的`let`语句的脱糖:

```Rust
let x;
let y;
```

```Rust
{
    let x;
    {
        let y;
    }
}
```

每个都创造自己的作用域,明确地确定一个先于另一个删除.但是,如果我们以下这样做怎么样?

```Rust
let (x, y) = (vec![], vec![]);
```

这个值是否严格活得超过另一个?答案实际上是 *否定的(no)* ,两个值都没有严格活得超过另一个.当然,x或y中的一个将在另一个之前被删除,但是没有指定实际的顺序.元组在这方面并不特别;复合结构只是不保证它们的析构顺序,如Rust 1.0.

我们 *可以(could)* 为元组和结构等内置组合的字段指定它.然而,像Vec这样的东西呢?Vec必须通过纯库代码手动删除其元素.一般来说,任何实现Drop的东西都有机会在最后的丧钟期间摆弄它的内部结构.因此,编译器无法充分推理实现Drop的任何类型的内容的实际析构顺序.

那我们为什么要关心呢?我们关心,因为如果类型系统不小心,它可能会意外地制造悬空指针.考虑以下简单程序:

```Rust
struct Inspector<'a>(&'a u8);

fn main() {
    let (inspector, days);
    days = Box::new(1);
    inspector = Inspector(&days);
}
```

这个程序现在完全合理,正常编译.事实上`days`没有 *严格(strictly)* 活得超过`inspector`并不重要.只要`inspector`还活着,days也是如此.

但是如果我们添加一个析构函数,程序将不再编译!

```Rust
struct Inspector<'a>(&'a u8);

impl<'a> Drop for Inspector<'a> {
    fn drop(&mut self) {
        println!("I was only {} days from retirement!", self.0);
    }
}

fn main() {
    let (inspector, days);
    days = Box::new(1);
    inspector = Inspector(&days);
    // Let's say `days` happens to get dropped first.
    // Then when Inspector is dropped, it will try to read free'd memory!
}
```

```Rust
error[E0597]: `days` does not live long enough
  --> src/main.rs:12:28
   |
12 |     inspector = Inspector(&days);
   |                            ^^^^ borrowed value does not live long enough
...
15 | }
   | - `days` dropped here while still borrowed
   |
   = note: values in a scope are dropped in the opposite order they are created

error: aborting due to previous error
```

实现`Drop`可让`Inspector`在其死亡期间执行一些任意代码.这意味着它可以潜在地观察那些应该活着的类型,只要它实际上首先被销毁.

有趣的是,只有泛型类型需要担心这一点.如果它们不是泛型的,那么它们可以庇护的唯一生命周期是`'static`,这将 *永远(forever)* 存活.这就是为什么这个问题被称为 *合理的泛型删除(sound generic drop)* . *删除检查器(drop checker)* 强制执行合理的泛型删除.在撰写本文时，关于删除检查器如何验证类型的一些更精细的细节完全在空中.然而,大规则(Big Rule)是我们整节关注的微妙之处:

**对于一个完全实现drop的泛型类型,它的泛型参数必须严格活得超过它(For a generic type to soundly implement drop, its generics arguments must strictly outlive it).**

遵守这一规则(通常)是满足借用检查器所必需的;遵守它是充分,但非必要的对于健全来说.也就是说,如果你的类型服从这个规则那么它肯定是删除合理的.

满足上述规则并不总是必要的,原因是某些Drop实现不会访问借用的数据,即使它们的类型赋予它们这种访问的能力.

例如,上述`Inspector`示例的此变体将永远不会访问借用的数据:

```Rust
struct Inspector<'a>(&'a u8, &'static str);

impl<'a> Drop for Inspector<'a> {
    fn drop(&mut self) {
        println!("Inspector(_, {}) knows when *not* to inspect.", self.1);
    }
}

fn main() {
    let (inspector, days);
    days = Box::new(1);
    inspector = Inspector(&days, "gadget");
    // Let's say `days` happens to get dropped first.
    // Even when Inspector is dropped, its destructor will not access the
    // borrowed `days`.
}
```

同样,此变体也永远不会访问借来的数据:

```Rust
use std::fmt;

struct Inspector<T: fmt::Display>(T, &'static str);

impl<T: fmt::Display> Drop for Inspector<T> {
    fn drop(&mut self) {
        println!("Inspector(_, {}) knows when *not* to inspect.", self.1);
    }
}

fn main() {
    let (inspector, days): (Inspector<&u8>, Box<u8>);
    days = Box::new(1);
    inspector = Inspector(&days, "gadget");
    // Let's say `days` happens to get dropped first.
    // Even when Inspector is dropped, its destructor will not access the
    // borrowed `days`.
}
```

但是,在分析`fn main`期间,借用检查器拒绝了上述 *两种(both)* 变体,并表示`days`活得不够长.

原因是`main`的借用检查分析不知道每个`Inspector`的`Drop`实现的内部结构.借用检查器在分析`main`时知道,inspector的析构函数的主体可能会访问所借用的数据.

因此,删除检查其强制所有借来的数据值严格活得超过该值.

# An Escape Hatch(An Escape Hatch)

管理删除检查的精确规则将来可能不那么严格.

目前的分析是故意保守和琐碎的;它强制值中所有借来的数据都比这个值活得更长,这肯定是合理的.

语言的未来版本可以使分析更加精确,以减少合理代码被拒绝为不安全的情况.这将有助于解决上述两名`Inspector`在析构期间不知道检查的情况.

与此同时,有一个不稳定的属性可以用来断言(不安全地)泛型类型的析构函数 *保证(guaranteed)* 不访问任何过期数据,即使它的类型赋予它这样做的能力.

该属性称为`may_dangle`,在[RFC 1327](https://github.com/rust-lang/rfcs/blob/master/text/1327-dropck-param-eyepatch.md)中引入.要将其部署在上面的`Inspector`示例中,我们将编写:

```Rust
struct Inspector<'a>(&'a u8, &'static str);

unsafe impl<#[may_dangle] 'a> Drop for Inspector<'a> {
    fn drop(&mut self) {
        println!("Inspector(_, {}) knows when *not* to inspect.", self.1);
    }
}
```

使用此属性要求将`Drop`impl标记为`unsafe`,因为编译器不检查隐式断言,即没有可能访问任何过期的数据(例如,上面的`self.0`).

该属性可以应用于任意数量的生命周期和类型参数.在下面的示例中,我们断言我们不会访问生命周期`'b`的引用后面的数据,并且`T`的唯一用途是移动或删除,但在`'a`和`U`上省略属性,因为我们确实访问具有该生命周期和那个类型的数据:

```Rust
use std::fmt::Display;

struct Inspector<'a, 'b, T, U: Display>(&'a u8, &'b u8, T, U);

unsafe impl<'a, #[may_dangle] 'b, #[may_dangle] T, U: Display> Drop for Inspector<'a, 'b, T, U> {
    fn drop(&mut self) {
        println!("Inspector({}, _, _, {})", self.0, self.3);
    }
}
```

有时很明显,不会发生这样的访问,如上所述.但是,在处理泛型类型参数时,这种访问可以间接发生.此类间接访问的示例如下:

- 调用回调,

- 通过trait方法调用.

(对语言的未来更改,例如impl专业化,可能会为此类间接访问添加其他途径.)

以下是调用回调的示例:

```Rust
struct Inspector<T>(T, &'static str, Box<for <'r> fn(&'r T) -> String>);

impl<T> Drop for Inspector<T> {
    fn drop(&mut self) {
        // The `self.2` call could access a borrow e.g. if `T` is `&'a _`.
        println!("Inspector({}, {}) unwittingly inspects expired data.",
                 (self.2)(&self.0), self.1);
    }
}
```

以下是trait方法调用的示例:

```Rust
use std::fmt;

struct Inspector<T: fmt::Display>(T, &'static str);

impl<T: fmt::Display> Drop for Inspector<T> {
    fn drop(&mut self) {
        // There is a hidden call to `<T as Display>::fmt` below, which
        // could access a borrow e.g. if `T` is `&'a _`
        println!("Inspector({}, {}) unwittingly inspects expired data.",
                 self.0, self.1);
    }
}
```

当然,所有这些访问都可以在析构函数调用的其他方法中进一步隐藏,而不是直接在其中编写.

在所有上述情况下,在析构函数中访问`&'a u8`,添加`#[may_dangle]`属性会使该类型容易被滥用,借用检查程序将无法捕获,从而引发破坏.最好避免添加属性.

# 这是关于删除检查器的全部吗(Is that all about drop checker)?

事实证明,在编写不安全代码时,我们通常不需要担心为删除检查器做正确的事情.但是,你需要担心一个特殊情况,我们将在下一节中介绍.