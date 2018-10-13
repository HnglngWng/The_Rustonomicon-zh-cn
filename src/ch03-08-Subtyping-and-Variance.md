# 子类型和可变性(Subtyping and Variance)

子类型是类型之间的关系,它允许静态类型语言更灵活和更宽松.

最常见且易于理解的示例可以在具有继承的语言中找到.考虑一个具有`eat()`方法的Animal类型,以及一个扩展Animal的Cat类型,添加一个`meow()`方法.如果没有子类型,如果有人要编写一个`Feed(Animal)`函数,他们将不能将`Cat`传递给此函数,因为Cat并不 *完全地(exactly)* 是Animal.但是在预期Animal的地方能够传递Cat似乎是相当合理的.毕竟,Cat只是一种Animal *和更多东西(and more)* .某物具有可以忽略的额外功能,不应该是使用它的任何障碍!

这正是子类型允许我们做的事情.因为Cat是Animal *和更多东西(and more)* ,我们说Cat是Animal的 *子类型(subtype)* .然后我们说,在需要某个类型的值的任何地方,也可以提供子类型的值.好吧,实际上它比那要复杂得多,微妙得多,但这是在99%的情况下,你的基本直觉.我们将在后面的部分中讨论为什么只有99%.

尽管Rust没有任何结构继承的概念,但它确实包含子类型.在Rust中,子类型完全来自于生命周期.由于生命周期是代码的区域,我们可以根据包含(活得久(outlives))关系对它们进行部分排序.

生命周期的子类型就是根据这种关系:如果`'big: 'small`("大包含小"或"大比小活得久"),那么`'big`是`'small`的子类型.这是一个很大的混乱来源,因为似乎对许多人反向:较大的区域是较小区域的 *子类型(subtype)* .但是,如果你考虑我们的Animal例子,它是有道理的: *Cat* 是一种Animal *和更多东西(and more)* ,就像`'big`是`'small` *和更多东西(and more)* 一样.

换句话说,如果有人想要一个活为`'small`的引用,通常实际上意味着他们想要一个活为 *至少(at least)* `'small`的引用.他们实际上并不关心生命周期是否完全匹配.因为这个原因,`'static`(永久的生命周期)是每一个生命周期的子类型.

更高阶的生命周期也是每个具体生命周期的子类型.这是因为采取任意生命周期比采取特定生命周期更为普遍.

(生命周期的类型化(typed-ness)是一个相当随意的构造,有些人不同意.但它简化了我们的分析,统一地处理生命周期和类型.)

但是你不能编写一个接受类型`'a`值的函数!生命周期总是另一种类型的一部分,所以我们需要一种处理方式.要处理它,我们需要讨论 *可变性(variance)* .

# 可变性(variance)

可变性是事情变得有点复杂的地方.

可变性是 *类型构造函数(type constructors)* 关于其参数的属性.Rust中的类型构造函数是具有未限制参数的泛型类型.例如,`Vec`是一个类型构造函数,它接受一个`T`并返回一个`Vec<T>`.`&`和`&mut`是接受两个输入的类型构造函数:生命周期和指向的类型.

类型构造函数F的 *可变性(variance)* 是其输入的子类型如何影响其输出的子类型.Rust中有三种可变性:

- F在`T`上是 *协变的(covariant)* ,如果`T`是`U`的子类型,意味着`F<T>`是`F<U>`的子类型(子类型"传递(passes through)")

- F在`T`上是 *逆变的(contravariant)* ,如果`T`是`U`的子类型,意味着`F<U>`是`F<T>`的子类型(子类型是"反转的(inverted)")

- 否则F在`T`上是 *不变的(invariant)* (不能导出子类型关系)

应该指出的是,在Rust中协变比逆变更常见和重要 *得多(far)* .Rust中逆变的存在几乎可以忽略不计.

一些重要的可变性(我们将在下面详细解释):

- `&'a T`在`'a`和`T`上是协变的(隐喻为`*const T`)

- `&'a mut T`在`'a`上是协变的,但在`T`上是不变的.

- `fn(T() -> U`在`T`上是逆变的,但在`U`上是协变的.

- `Box`,`Vec`和所有其他集合对其内容的类型是协变的

- `UnsafeCell<T>`,`Cell<T>`,`RefCell<T>`,`Mutex<T>`和所有其他内部可变性类型在`T`上是不变的(隐喻为`*mut T`)

要理解为什么这些可变性是正确和可取的,我们将考虑几个例子.

在引入子类型时,我们已经介绍了为什么`&'a T`应该对`'a`进行协变:希望能够在需要较短寿命的事物中传递寿命更长的事物.

类似的推理适用于为什么它应该在`T`上是协变的:在需要`&&'a str`时,能够传递`&&'static str`是合理的.额外的间接水平并没有改变当需要寿命较短的事物时能够传递寿命更长的事物的愿望.

然而,这种逻辑不适用于`&mut`.要了解为什么`&mut`应该对`T`不变,请考虑以下代码:

```Rust
fn overwrite<T: Copy>(input: &mut T, new: &mut T) {
    *input = *new;
}

fn main() {
    let mut forever_str: &'static str = "hello";
    {
        let string = String::from("world");
        overwrite(&mut forever_str, &mut &*string);
    }
    // Oops, printing free'd memory
    println!("{}", forever_str);
}
```

`overwrite`的签名显然是有效的:它接受对相同类型的两个值的可变引用,并用另一个覆盖一个.

但是,如果`&mut T`在`T`上是协变的,那么`&mut &'static str`将是`&mut &'a str`的子类型,因为`&'static str`是`&'a str`的子类型.因此，`forever_str`的生命周期将成功"缩小(shrunk)"到`string`的较短生命周期,并且`overwrite`将成功调用.随后将删除`string`,`forever_str`将指向我们打印时释放的内存!因此,`&mut`应该是不变的.

这是可变性与不变性的一般主题:如果可变性,会允许你在较长寿命的时隙中存储短寿命的值,否则必须使用不变性.

更一般地说,子类型和可变性的合理性是基于它可以忘记细节的想法,但是通过可变引用,总会有人(被引用的原始值)记住被遗忘的细节,并假设这些细节没有改变.如果我们做了什么使这些细节无效,原始位置可能会表现不佳.

然而,`&'a mut T`对`'a`协变 *是(is)* 合理的.`'a`和T之间的关键区别在于`'a`是引用本身的属性,而T是引用所借用的东西.如果更改T的类型,则源仍会记住原始类型.但是,如果你改变了生命周期的类型,除了引用之外没有人知道这个信息,所以没关系.换句话说:`&'a mut T`拥有`'a`,但只 *借用(borrows)* T.

`Box`和`Vec`是很有趣的情况,因为它们是协变的,但你绝对可以在它们中存储值!这就是Rust的类型系统允许它比其他更聪明的地方.要理解为什么拥有容器对其内容协变是合理的,我们必须考虑变化(mutation)可能发生的两种方式:通过值(by-value)或通过引用(by-reference).

如果变化是通过值,那么记住额外细节的旧位置将被移出,这意味着它不能再使用该值.所以我们根本不需要担心任何人记住危险的细节.换句话说,在通过值传递时应用子类型会 *永远破坏细节(destroys details forever)* .例如,这可以编译,没问题:

```Rust
fn get_box<'a>(str: &'a str) -> Box<&'a str> {
    // String literals are `&'static str`s, but it's fine for us to
    // "forget" this and let the caller think the string won't live that long.
    Box::new("hello")
}
```

如果变化是通过引用,那么我们的容器将作为`&mut Vec<T>`传递.但是`&mut`对它的值是不变的,所以`&mut Vec<T>`实际上对`T`不变.因此`Vec<T>`对`T`协变的这一事实在通过引用变化时根本不重要.

但是,协变仍然允许`Box`和`Vec`在不可变地共享时被削弱.所以你可以在需要`&Vec<&'a str>`时,传递一个`&Vec<&'static str>`.

cell类型的不变性可以看作如下:`&`就像是cell的`&mut`,因为你仍然可以通过`&`在它们中存储值.因此,cell必须是不变的,以避免生命周期走私.

`fn`是最微妙的情况,因为它们具有混合可变性,实际上是逆变的唯一来源.要了解为什么`fn(T) -> U`应该在`T`上逆变,请考虑以下函数签名:

```Rust
// 'a is derived from some parent scope
fn foo(&'a str) -> usize;
```

这个签名声称它可以处理至少活得和`'a`一样长的任何`&str`.现在,如果这个签名对于`&'a str`是协变的,那就意味着

```Rust
fn foo(&'static str) -> usize;
```

可以在它的位置提供,因为它将是一个子类型.然而,这个函数有一个更强的要求:它说它只能处理`&'static str`,而不是其他任何东西.给予它`&'a str`是不合理的,因为它可以自由地假设它给予的东西永远存活.因此,函数绝对不应该对它们的参数进行协变.

但是,如果我们翻转它并使用逆变,它 *确实(does)* 有效!如果某些东西需要一个可以处理永久存活的字符串的函数,那么提供一个能够处理生命周期活 *不到(less)* 永久的字符串的函数是完全合理的.所以

```Rust
fn foo(&'a str) -> usize;
```

可以传递在

```Rust
fn foo(&'static str) -> usize;
```

被需要的地方.

要了解为什么`fn(T) -> U`应该对U进行协变,请考虑以下函数签名:

```Rust
// 'a is derived from some parent scope
fn foo(usize) -> &'a str;
```

这个签名声称它会返回活得比`'a`更长的东西.因此提供是完全合理的

```Rust
fn foo(usize) -> &'static str;
```

在它的位置,因为它确实返回活得比`'a`更长的东西.因此,函数在返回类型上是协变的.

`*const`与`&`具有完全相同的语义,因此可变性一样.另一方面,`*mut`可以解引用`&mut`,无论它是否共享,因此它像cell一样被标记为不变的.

这对于标准库提供的类型来说都很好,但是如何为 *你(you)* 定义的类型确定可变性?非正式地说,结构继承其字段的可变性,如果结构`Foo`有一个泛型参数`A`,它用在字段`a`中,那么`Foo`在`A`上的可变性就是`a`的可变性.但是,如果在多个字段中使用`A`:

- 如果A的所有使用都是协变的,那么Foo在`A`上是协变的

- 如果A的所有用法都是逆变的,那么Foo在`A`上是逆变的

- 否则,Foo在`A`上是不变的

```Rust
use std::cell::Cell;

struct Foo<'a, 'b, A: 'a, B: 'b, C, D, E, F, G, H, In, Out, Mixed> {
    a: &'a A,     // covariant over 'a and A
    b: &'b mut B, // covariant over 'b and invariant over B

    c: *const C,  // covariant over C
    d: *mut D,    // invariant over D

    e: E,         // covariant over E
    f: Vec<F>,    // covariant over F
    g: Cell<G>,   // invariant over G

    h1: H,        // would also be variant over H except...
    h2: Cell<H>,  // invariant over H, because invariance wins all conflicts

    i: fn(In) -> Out,       // contravariant over In, covariant over Out

    k1: fn(Mixed) -> usize, // would be contravariant over Mixed except..
    k2: Mixed,              // invariant over Mixed, because invariance wins all conflicts
}
```