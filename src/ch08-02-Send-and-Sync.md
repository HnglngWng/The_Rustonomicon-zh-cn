# Send和Sync(Send and Sync)

但是,并非一切都遵循继承的可变性.某些类型允许你在改变它时,拥有多个内存中位置的别名.除非这些类型使用同步(synchronization)来管理此访问,否则它们绝对不是线程安全的.Rust通过Send和Sync trait捕获这一点.

- 如果将其发送到另一个线程是安全的,则类型为Send.

- 如果在线程之间共享是安全的,则类型为Sync(当且仅当`&T`为Send时,T为Sync).

Send和Sync是Rust的并发故事的基础.因此,存在大量特殊工具以使它们正常工作.首先,它们是不安全trait.这意味着它们实现起来不安全,而其他不安全的代码可以假设它们已正确实现.由于它们是 *标记trait(marker traits)* (它们没有像方法那样的关联项),所以正确实现只是意味着它们具有实现者应具有的内在属性.错误地实现Send或Sync可能会导致未定义行为.

Send和Sync也是自动派生的traits.这意味着,与其他所有trait不同,如果类型完全由Send或Sync类型组成,则它是Send或Sync.几乎所有基元都是Send和Sync,因此几乎所有与之交互的类型都是Send和Sync.

主要例外包括:

- 原始指针既不是Send也不是Sync(因为它们没有安全防护).

- `UnsafeCell`不是Sync(因此`Cell`和`RefCell`不是).

- `Rc`不是Send或Sync(因为引用计数(refcount)是共享和不同步的).

`Rc`和`UnsafeCell`从根本上说不是线程安全的:它们启用了不同步的共享可变状态.然而,严格地说,原始指针是,标记为线程不安全更多的是 *棉绒(lint)* .使用原始指针执行任何有用的操作都需要解引用它,这已经是不安全的.从这个意义上说,人们可以争辩说,将它们标记为线程安全是"好的(fine)".

但是,重要的是它们不是线程安全的,以防止包含它们的类型被自动标记为线程安全.这些类型具有非平凡的未经跟踪的所有权,并且他们的作者不太可能一直在思考线程安全性.在`Rc`的情况下,我们有一个很好的例子,它包含一个绝对不是线程安全的`*mut`.

如果需要,非自动派生的类型可以简单地实现它们:

```Rust
struct MyBox(*mut u8);

unsafe impl Send for MyBox {}
unsafe impl Sync for MyBox {}
```

在 *非常罕见的(incredibly rare)* 情况下,类型被不适当地自动派生为Send或Sync,那么人们也可以不实现Send和Sync:

```Rust
#![feature(negative_impls)]

// I have some magic semantics for some synchronization primitive!
struct SpecialThreadToken(u8);

impl !Send for SpecialThreadToken {}
impl !Sync for SpecialThreadToken {}
```

请注意, *它本身(n and of itself)* 不可能错误地派生Send和Sync.只有被其他不安全代码赋予特殊含义的类型才可能因错误Send或Sync而导致问题.

原始指针的大多数用法应该封装在足够的抽象之后,可以派生Send和Sync.例如,Rust的所有标准集合都是Send和Sync(当它们包含Send和Sync类型时),尽管它们普遍使用原始指针来管理分配和复杂的所有权.类似地,这些集合的大多数迭代器都是Send和Sync,因为它们在很大程度上表现为集合的`&`或`&mut`.

TODO:更好地解释什么可以或不可以是Send或Sync.是否足以吸引数据竞争?
