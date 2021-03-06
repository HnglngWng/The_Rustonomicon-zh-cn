# 无限制生命周期(Unbounded Lifetimes)

不安全的代码通常最终会凭空产生引用或生命周期.这样的生命周期进入 *无限制的(unbounded)* 世界.最常见的来源是解引用原始指针,该指针生成具有无限制生命周期的引用.这样的生命周期变得像上下文要求一样大.这实际上比简单地变成`'static`更强大,因为例如`&'static &'a T`将无法进行类型检查,但是无限制的生命周期将根据需要完美地塑造成`&'a &'a T`.然而,对于大多数意图和目的,这种无限制生命周期可以被视为`'static`.

几乎没有引用是`'static`,所以这可能是错误的.`transmute`和`transmute_copy`是另外两个主要罪犯.人们应尽可能快地限制无限制的生命周期,特别是跨越函数边界.

给定一个函数,任何不是从输入派生的输出生命周期都是无限制的.例如:

```Rust
fn get_str<'a>() -> &'a str;
```

将产生一个无限制的生命周期的`&str`.避免无限制的生命期的最简单方法是在函数边界使用生命周期省略.如果省略输出生命周期,则 *必须(must)* 以输入生命周期为限制.当然它可能受到 *错误的(wrong)* 生命周期的限制,但这通常只会导致编译器错误,而不是简单地违反内存安全性.

在函数内,限制生命周期更容易出错.限制生命周期的最安全和最简单的方式是从具有限制生命周期的函数返回它.但是,如果它是不可接受的,则可以将引用放置在具有特定生命周期的位置.不幸的是,不可能命名函数中涉及的所有生命周期.
