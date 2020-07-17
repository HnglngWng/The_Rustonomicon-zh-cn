# Safe与Unsafe如何交互(How Safe and Unsafe Interact)

Safe Rust和Unsafe Rust之间是什么关系?它们如何交互?

Safe Rust和Unsafe Rust之间的分离由`unsafe`关键字控制,该关键字充当从一个到另一个的接口.这就是为什么我们可以说Safe Rust是一种安全的语言:所有不安全的部分都只保留在`unsafe`边界之后.如果你愿意,你甚至可以将`#![forbid(unsafe_code)]`折腾到你的代码库中,以静态保证你只编写Safe Rust.

`unsafe`关键字有两个用途:声明编译器无法检查的契约的存在,和声明程序员已检查这些契约已被维护.

你可以将`unsafe`用在 *函数和traits声明(functions and trait declarations)* 上来指示存在未经检查的契约.在函数上,`unsafe`意味着函数的用户必须检查该函数的文档,以确保它们以维护函数所需的契约的方式使用它.在trait声明上,`unsafe`意味着trait的实现者必须检查trait文档以确保它们的实现维持trait所需的契约.

你可以对块使用`unsafe`来声明在其中执行的所有不安全操作都经过验证,以维护这些操作的契约.例如,传递给`slice::get_unchecked`的索引在界限内(in-bounds).

你可以对trait实现使用`unsafe`来声明实现支持trait的契约.例如,实现`Send`的类型移动到另一个线程是真的安全的.

标准库有许多不安全的函数,包括:

- `slice::get_unchecked`,执行未经检查的索引,允许自由地违反内存安全性.

- `mem::transmute`将某些值重新解释为具有给定类型,以任意方式绕过类型安全(有关详细信息,请参阅[转换](https://github.com/rust-lang-nursery/nomicon/blob/master/src/conversions.html)).

- 每个指向有大小类型的原始指针都有一个`offset`方法,如果传递的偏移量不在界限内"([in bounds](https://github.com/rust-lang-nursery/nomicon/blob/master/std/primitive.pointer.html#method.offset))",则产生未定义行为.

- 所有FFI(外部函数接口)函数都是`unsafe`,因为另一种语言可以执行Rust编译器无法检查的任意操作.

从Rust 1.29.2开始,标准库定义了以下不安全traits(还有其他traits,但它们尚未稳定,其中一些可能永远不会稳定):

- [`Send`](https://github.com/rust-lang-nursery/nomicon/blob/master/std/marker/trait.Send.html)是一个标记trait(没有API的trait),它承诺实现者可以安全地发送(移动)到另一个线程.

- [`Sync`](https://github.com/rust-lang-nursery/nomicon/blob/master/std/marker/trait.Sync.html)是一个标记trait,承诺线程可以通过共享引用安全地共享实现者.

- [`GlobalAlloc`](https://github.com/rust-lang-nursery/nomicon/blob/master/std/alloc/trait.GlobalAlloc.html)允许自定义整个程序的内存分配器.

大多数Rust标准库也在内部使用Unsafe Rust.这些实现通常都经过严格的手动检查,因此可以假设构建在这些实现之上的Safe Rust接口是安全的.

所有这些分离的需要归结为Safe Rust的一个基本属性,*可靠性属性(soundness property)* :

**无论如何,Safe Rust都不会导致未定义行为.**

安全/不安全拆分的设计意味着Safe和Unsafe Rust之间存在不对称的信任关系.Safe Rust本身必须相信它所触及的任何Unsafe Rust都已正确编写.另一方面,Unsafe Rust在信任Safe Rust时必须非常小心.

例如,Rust具有`PartialOrd`和`Ord`trait来区分可以"仅(just)"进行比较的类型,以及提供"总(total)"排序的类型(这基本上意味着比较表现得相当合理).

`BTreeMap`对部分排序的类型没有意义,因此它需要它的键实现`Ord`.但是,`BTreeMap`在其实现中包含Unsafe Rust代码.因为对于导致未定义行为的草率`Ord`实现(写入Safe)是不可接受的,所以`BTreeMap`中的Unsafe代码必须编写得健壮,以对抗实际上不是全部(total)的`Ord`实现--尽管这是需要`Ord`的关键所在.

Unsafe Rust代码无法信任Safe Rust代码是正确编写的.也就是说,如果你输入没有总排序的值,`BTreeMap`仍然会完全不正常.它不会导致未定义行为.

有人可能会想,如果`BTreeMap`不能信任`Ord`,因为它是Safe,为什么它可以信任 *任何(any)* Safe代码?例如,`BTreeMap`依赖于整数和切片的正确实现.那些也很安全,对吧?

区别在于范围之一.当`BTreeMap`依赖于整数和切片时,它依赖于一个非常具体的实现.这是一个可以衡量风险,可以权衡利益.在这种情况下,基本上没有风险;如果整数和切片被破坏, *所有人(everyone)* 都会被破坏.此外,它们由维护`BTreeMap`的人维护,因此很容易密切关注它们.

另一方面,`BTreeMap`的键类型是泛型.信任其`Ord`实现意味着信任过去,现在和未来的每个`Ord`实现.这里的风险很高:某个地方的某个人会犯错误并搞砸他们的`Ord`实现,甚至只是直接说谎提供总排序,因为"它似乎有效".当发生这种情况时,`BTreeMap`需要准备好.

相同的逻辑适用于信任传递给你的闭包正常运行.

无界(unbounded)泛型信任的这个问题是`unsafe`traits存在要解决的问题.从理论上讲,`BTreeMap`类型可以要求键实现一个名为`UnsafeOrd`而不是`Ord`的新trait,它可能如下所示:

```Rust
use std::cmp::Ordering;

unsafe trait UnsafeOrd {
    fn cmp(&self, other: &Self) -> Ordering;
}
```

然后,类型将使用`unsafe`来实现`UnsafeOrd`,表明他们已经确保他们的实现维护了trait所期望的任何契约.在这种情况下,`BTreeMap`内部的Unsafe Rust在信任键类型的`UnsafeOrd`实现是正确的时候是合理的.如果不是,那就是不安全trait实现的错误,这与Rust的安全保证一致.

是否标记trait`unsafe`的决定是API设计选择.Rust传统上避免这样做,因为它使得Unsafe Rust普遍存在,这是不可取的.`Send`和`Sync`被标记为不安全,因为线程安全是一个 *基本属性(fundamental property)* ,不安全的代码不可能希望以它可以防御有bug的`Ord`实现的方式进行防御.类似地,`GlobalAllocator`会保留程序中所有内存的记录,以及在其上构建的`Box`或`Vec`等其他东西. 如果它做了一些奇怪的事情(当它仍然在使用时将相同的内存块分配给另一个请求),那么就没有机会检测到它并对此采取任何措施.

是否标记自己的trait`unsafe`的决定取决于相同的考虑.如果`unsafe`代码不能合理地期望防止trait的破坏实现,那么标记trait`unsafe`是一个合理的选择.

顺便说一句,虽然`Send`和`Sync`是`unsafe`trait,但是当这些派生可以证明是安全的时候,它们 *也(also)* 会自动实现.对于仅由类型也实现`Send`的值组成的所有类型,将自动派生`Send`.对于仅由其类型也实现`Sync`的值组成的所有类型,将自动派生`Sync`.这最大限度地减少了使这两种traits`unsafe`的普遍不安全性.而且没有多少人会 *实现(implement)* 内存分配器(或者直接使用它们).

这是Safe和Unsafe Rust之间的平衡.分离旨在使Safe Rust尽可能符合人体工程学,但在编写Unsafe Rust时需要额外的努力和小心.本书的其余部分主要讨论了必须采取的谨慎措施,以及Unsafe Rust必须遵守的契约.
