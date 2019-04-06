# 子类型和可变性(Subtyping and Variance)

子类型是类型之间的关系,它允许静态类型语言更灵活和更宽松.

Rust中的子类型与其他语言的子类型略有不同.这使得提供简单示例变得更加困难,这是一个问题.因为子类型,尤其是可变性,已经很难正确理解.就像这样,即使编译器的作者也总是把它搞砸.

为了简单起见,本节将考虑对Rust语言的一个小扩展,它增加了一个新的更简单的子类型关系.在这个更简单的系统下建立概念和问题之后,我们将把它与Rust中实际发生的子类型联系起来.

所以这是我们的简单扩展, *Objective Rust* ,有三种新类型:

```Rust
trait Animal {
    fn snuggle(&self);
    fn eat(&mut self);
}

trait Cat: Animal {
    fn meow(&self);
}

trait Dog: Animal {
    fn bark(&self);
}
```

但与普通trait不同,我们可以将它们用作具体和有大小的类型,就像结构一样.

现在,假设我们有一个非常简单的函数,它接受一个Animal,就像这样:

```Rust
fn love(pet: Animal) {
    pet.snuggle();
}
```

默认情况下,静态类型必须与要编译的程序 *完全(exactly)* 匹配. 因此,此代码将无法编译:

```Rust
let mr_snuggles: Cat = ...;
love(mr_snuggles);         // ERROR: expected Animal, found Cat
```

Snuggles先生是猫,猫不 *完全地(exactly)* 是动物,所以我们不能爱他!😿

这很烦人,因为猫 *是(are)* 动物. 它们支持动物支持的每一项操作,所以如果我们传递给它一个`Cat`,直觉上`love`就不应该关心. 我们应该能够 **忘记(forget)** 我们`Cat`的非动物部分,因为它们没有必要去爱它.

这正是 *子类型(subtyping)* 要修复的问题.因为Cat是Animal *和更多东西(and more)* ,我们说Cat是Animal的 *子类型(subtype)* (因为Cat是所有Animal 的 *子集(subset)* ).同样,我们说Animal 是Cat 的 *超类型(supertype)* .使用子类型,我们可以通过一个简单的规则调整我们过于严格的静态类型系统:在预期类型为`T`的值的任何地方,我们也接受`T`的子类型的值.

或者更具体地说:在任何预期Animal 的地方,Cat 或Dog 也会起作用.

正如我们将在本节的其余部分中看到的那样,子类型比这更复杂和微妙,但这个简单的规则是非常好的99%直觉. 除非你编写不安全的代码,编译器将自动为你处理所有角落情况.

但这是Rustonomicon. 我们正在编写不安全的代码,所以我们需要了解这些东西是如何工作的,以及我们如何搞砸它.

核心问题是这个天真适用的规则会导致 *喵喵叫的狗(meowing Dogs)* . 也就是说,我们可以说服某人说狗实际上是猫. 这完全破坏了我们的静态类型系统的结构,使其比无用(并导致Undefined Behaviour)更糟糕.

当我们以完全天真的"查找和替换"方式应用子类型时,这是一个简单的例子.

```Rust
fn evil_feeder(pet: &mut Animal) {
    let spike: Dog = ...;

    // `pet` is an Animal, and Dog is a subtype of Animal,
    // so this should be fine, right..?
    *pet = spike;
}

fn main() {
    let mut mr_snuggles: Cat = ...;
    evil_feeder(&mut mr_snuggles);  // Replaces mr_snuggles with a Dog
    mr_snuggles.meow();             // OH NO, MEOWING DOG!
}
```

显然,我们需要一个比"查找和替换"更强大的系统. 该系统是可变性,这是一组规则如何构成子类型的规则. 最重要的是,可变性定义了应禁用子类型的情况.

但在我们开始可变性之前,让我们快速看看Rust中实际发生的子类型:生命周期!

> 注意:生命周期的类型化(typed-ness)是一种相当随意的结构,有些人不同意. 然而,它简化了我们的分析,以统一处理生命周期和类型.

生命周期只是代码区域,区域可以根据 *包含(contains)* (活得久(outlives))关系对它们进行部分排序. 生命周期的子类型就是根据这种关系:如果`'big: 'small`("大包含小"或"大比小活得久"),那么`'big`是`'small`的子类型.这是一个很大的混乱来源,因为似乎对许多人反向:较大的区域是较小区域的 *子类型(subtype)* .但是,如果你考虑我们的Animal例子,它是有道理的: *Cat* 是一种Animal *和更多东西(and more)* ,就像`'big`是`'small` *和更多东西(and more)* 一样.

换句话说,如果有人想要一个活为`'small`的引用,通常实际上意味着他们想要一个活为 *至少(at least)* `'small`的引用.他们实际上并不关心生命周期是否完全匹配.因此,我们应该 *忘记(forget)* 某物活为`'big`而且只记得它活为`'small`.

生命周期的喵喵狗问题将导致我们能够将一个短命的引用存储在一个期望寿命更长的地方,创造一个悬垂的引用并让我们在释放后使用.

值得注意的是,`'static`,永恒的生命周期,是每个生命周期的子类型,因为根据定义,它比任何东西都活得长.我们将在后面的示例中使用此关系,以使它们尽可能简单.

尽管如此,我们仍然不知道如何实际 *使用(use)* 生命周期的子类型,因为没有什么东西有类型`'a`.生命周期只发生在某些较大类型的一部分,如`&'a u32`或`IterMut<'a, u32>`.要应用生命周期子类型,我们需要知道如何组合子类型.再一次,我们需要 *可变性(variance)* .

# 可变性(variance)

可变性是事情变得有点复杂的地方.

可变性是 *类型构造函数(type constructors)* 关于其参数的属性.Rust中的类型构造函数是具有未限制参数的任何泛型类型.例如,`Vec`是一个类型构造函数,它接受一个`T`并返回一个`Vec<T>`.`&`和`&mut`是接受两个输入的类型构造函数:生命周期和指向的类型.

> 注意:为方便起见,我们经常将`F<T>`称为类型构造函数,以便我们可以轻松地讨论`T`.希望这在上下文中是明确的.

类型构造函数F的 *可变性(variance)* 是其输入的子类型如何影响其输出的子类型.Rust中有三种可变性.给定`Sub`和`Super`两种类型,其中`Sub`是`Super`的子类型:

- `F`是 *协变的(covariant)* ,如果`F<Sub>`是`F<Super>`的子类型(子类型"传递(passes through)")

- `F`是 *逆变的(contravariant)* ,如果`F<Super>`是`F<Sub>`的子类型(子类型是"反转的(inverted)")

- 否则`F`是 *不变的(invariant)* (不存在子类型关系)

如果`F`有多个类型参数,我们可以通过说,例如,`F<T, U>`在`T`上是协变的而在`U`上是不变的,可以讨论各个可变性.

记住协变实际上是"the"可变是非常有用的. 几乎所有对可变性的考虑都取决于某些事物是否应该是协变的或不变的. 实际上,在Rust中见证逆变是相当困难的,尽管它确实存在.

以下是本节其余部分将要解释的重要可变性表:

| ||'a|T|U|
| :--: | :--: | :--: | :--: | :--: |
|*|`&'a T`| 协变|协变||
|*|`&'a mut T`|协变|不变|
|*|`Box<T>`||协变||
| |`Vec<T>`||协变|
|*|`UnsafeCell<T>`||不变|
| |`Cell<T>`||不变||
|*|`fn(T) -> U`||逆变|协变|
| |`*const T`||协变||
| |`*mut T`||不变||

带*的类型是我们将关注的类型,因为它们在某种意义上是"基本的(fundamental)". 所有其他的都可以通过类比来理解:

- Vec和所有其他拥有指针和集合遵循与Box相同的逻辑

- Cell和所有其他内部可变性类型遵循与UnsafeCell相同的逻辑

- `*const`遵循`&T`的逻辑

- `*mut`遵循`&mut T`(或`UnsafeCell<T>`)的逻辑

> 注意:语言中 *唯一的(only)* 逆变来源是函数的参数,这就是为什么它在实践中很少出现的原因. 调用逆变涉及使用函数指针进行高阶编程,这些函数指针具有特定生命周期的引用(与通常的"任何生命周期"相反,后者进入更高级别的生命周期,其独立于子类型工作).

好了,类型理论就讲到这里!让我们尝试将可变性的概念应用于Rust,并查看一些例子.

首先,让我们回顾一下喵喵叫的狗的例子:

```Rust
fn evil_feeder(pet: &mut Animal) {
    let spike: Dog = ...;

    // `pet` is an Animal, and Dog is a subtype of Animal,
    // so this should be fine, right..?
    *pet = spike;
}

fn main() {
    let mut mr_snuggles: Cat = ...;
    evil_feeder(&mut mr_snuggles);  // Replaces mr_snuggles with a Dog
    mr_snuggles.meow();             // OH NO, MEOWING DOG!
}
```

如果我们查看可变性表,我们会发现`&mut T`对`T`不变.事实证明,这完全解决了这个问题! 由于不变性,Cat是Animal的子类型的事实并不重要; `&mut Cat`仍然不会是`&mut Animal`的子类型. 然后,静态类型检查器将正确阻止我们将Cat传递给`evil_feeder`.

子类型的健全性(soundness)基于这样一种思想,即可以忽略不必要的细节. 但是对于引用,总会有人记得这些细节:被引用的值. 该值预期这些细节将保持正确,如果违背了它的期望,那么行为可能不正确.

使`&mut T`在`T`上协变的问题在于, *当我们不记得所有约束时(when we don't remember all of its constraints)* ,它赋予我们修改原始值的能力. 因此,当他们确定他们还有一只猫时,我们可以让某人有一只狗.

有了这个,我们可以很容易地看出为什么`&T`对`T`协变是合理的:它不允许你修改值,只允许查看它. 没有任何改变的方法,我们就没有办法去破坏任何细节. 我们也可以看到为什么`UnsafeCell`和所有其他内部可变性类型必须是不变的:它们使`&T`像`&mut T`一样工作!

那么引用的生命周期呢? 为什么这两种引用可以对其生命周期都是协变的呢? 嗯,这里有一个双管齐下的论点:

首先,基于其生命周期的子类型引用是 *Rust中子类型的全部要点(the entire point of subtyping in Rust.)* . 我们进行子类型化的唯一原因是,这样我们就可以在预期短生命周期的地方传递长生命周期的东西. 所以它更好用!

其次,更严重的是,生命周期只是引用本身的一部分.引用物的类型是共享知识,这就是为什么仅在一个地方(引用)调整该类型可能导致问题.但是,如果你在将引用传递某人时减少了引用的生命周期,那么无论如何都不会共享该生命周期信息.现在有两个具有独立生命周期的独立引用.使用另一个引用不会破坏原始引用的生命周期..

或者更确切地说,搅乱某人生命周期的唯一方法就是构建一只会喵喵叫的狗.但是一旦你试图构建一只喵喵叫的狗,它的生命周期就应该被包装到一个不变的类型,防止它的生命缩短.为了更好地理解这一点,让我们将喵喵叫的狗问题转移到真正的Rust上来.

在喵喵叫的狗问题中,我们接受一个子类型(Cat),将其转换为超类型(Animal),然后使用该事实用一个值覆盖子类型,该值满足超类型的约束,但不满足子类型(Dog).

所以对于生命周期,我们接受一个长命的东西,把它转换成一个短命的东西,然后用它来写一些活得不够长的东西到期望长命的地方.

如下所示:

```Rust
fn evil_feeder<T>(input: &mut T, val: T) {
    *input = val;
}

fn main() {
    let mut mr_snuggles: &'static str = "meow! :3";  // mr. snuggles forever!!
    {
        let spike = String::from("bark! >:V");
        let spike_str: &str = &spike;                // Only lives for the block
        evil_feeder(&mut mr_snuggles, spike_str);    // EVIL!
    }
    println!("{}", mr_snuggles);                     // Use after free?
}
```

当我们运行它时我们会得到什么?

```Rust
error[E0597]: `spike` does not live long enough
  --> src/main.rs:9:32
   |
9  |         let spike_str: &str = &spike;
   |                                ^^^^^ borrowed value does not live long enough
10 |         evil_feeder(&mut mr_snuggles, spike_str);
11 |     }
   |     - borrowed value only lives until here
   |
   = note: borrowed value must be valid for the static lifetime...
```

很好,它不能编译! 我们详细分析一下这里发生了什么.

首先我们看看新的`evil_feeder`函数:

```Rust
fn evil_feeder<T>(input: &mut T, val: T) {
    *input = val;
}
```

它所做的就是接受一个可变引用和一个值,并用它覆盖所指对象. 这个函数的重要之处在于,它创建了一个类型相等约束. 它在其签名中清楚地表明指示物和值必须是完全相同的类型.

同时,在调用者中我们传入`&mut &'static str`和`&'spike_str str`.

因为`&mut T`在`T`上是不变的,所以编译器得出结论,它不能对第一个参数应用任何子类型,因此`T`必须是`&'static str`.

另一个参数只是一个`&'a str`,它对`'a`是协变的. 因此编译器采用了一个约束:`&'spike_str str`必须是`&'static str`(包括)的子类型,这反过来暗示`'spike_str`必须是`'static`(包含)的子类型. 也就是说,`'spike_str`必须包含`'static`. 但只有一个东西包含`'static`--`'static`本身!

这就是为什么当我们试图将`&spike`分配给`spike_str`时,会得到一个错误. 编译器已经向后工作,得出结论`spike_str`必须永远存在,而`&spike`根本不能活得那么久.

因此,即使引用对其生命周期是协变的,但只要它们被置于可能对此做坏事的上下文中,它们就"继承(inherit)"不变性. 本例中,只要我们将引用放在`&mut T`中,我们就会继承不变性.

事实证明,为什么Box(以及Vec,Hashmap等)可以协变的论证非常类似于为什么生命周期可以协变的论证:只要你试图把它们塞进一个类似于可变引用的东西中,它们就继承了不变性并且你被阻止做任何坏事.

然而Box更容易关注我们部分忽略的引用的按值(by-value)方面.
与许多允许值在任何时候自由别名化的语言不同,Rust有一个非常严格的规则:如果允许你修改或移动一个值,那么你一定是惟一能够访问它的人.

请考虑以下代码:

```Rust
let mr_snuggles: Box<Cat> = ..;
let spike: Box<Dog> = ..;

let mut pet: Box<Animal>;
pet = mr_snuggles;
pet = spike;
```

因为我们忘记了`mr_snuggles`是一只猫,或者我们用狗覆盖了它,所以没有任何问题,因为只要我们将`mr_snuggles`移动到一个只知道它是动物的变量, **我们就摧毁了宇宙中唯一记得它是猫的东西(we destroyed the only thing in the universe that remembered he was a Cat)** !

与不可变引用完全协变的论点相反,因为它们不允许你改变任何东西,拥有的值可以是协变的,因为它们让你改变一切. 旧位置和新位置之间没有联系. 按值(by-value)应用子类型是一种不可逆转的知识破坏行为,如果没有对过去情况的任何记忆,那么就不会有任何人被欺骗去对旧信息采取行动!

只剩下一件事要解释:函数指针.

要了解为什么`fn(T) -> U`应该在`U`上是协变的,请考虑以下签名:

```Rust
fn get_animal() -> Animal;
```

该功能声称产生一种动物. 因此,提供具有以下签名的函数是完全有效的:

```Rust
fn get_animal() -> Cat;
```

毕竟,猫是动物,因此生产猫是一种完全有效的生产动物的方法. 或者将它与真正的Rust联系起来:如果我们需要一个能够产生活为`'short`的函数,那么生产活为`'long`的东西完全没问题. 我们不在乎,我们可以就忘记这一事实.

但是,相同的逻辑不适用于 *参数(arguments)* . 考虑尝试满足:

```Rust
fn handle_animal(Animal);
```

用

```Rust
fn handle_animal(Cat);
```

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

第一个函数可以接受Dogs,但第二个函数绝对不能. 协变在这里不起作用. 但是,如果我们翻转它,它确实有效! 如果我们需要一个可以处理Cats的函数,那么一个可以处理 *任何(any)* Animal的函数肯定会正常工作. 或者将它与真正的Rust联系起来:如果我们需要一个能够处理任何至少`'long`寿命的函数,那么它能够处理任何至少`'short`寿命的东西.

这就是为什么函数类型与语言中的其他任何类型不同,它们的参数都是逆变的.

现在,这对于标准库提供的类型来说,这一切都很好,但是如何确定 *你(you)* 定义的类型的可变性? 非正式地说,结构继承其字段的可变性. 如果结构`MyType`有一个泛型参数`A`,它用在字段`a`中,那么MyType在`A`上的可变性就是`a`在`A`的可变性.

但是,如果`A`用于多个字段:

- 如果`A`的所有使用都是协变的,那么MyType在`A`上是协变的

- 如果`A`的所有用法都是逆变的,那么MyType在`A`上是逆变的

- 否则,MyType在`A`上是不变的

```Rust
use std::cell::Cell;

struct MyType<'a, 'b, A: 'a, B: 'b, C, D, E, F, G, H, In, Out, Mixed> {
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