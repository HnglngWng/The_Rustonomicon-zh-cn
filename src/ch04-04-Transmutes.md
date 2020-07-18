# Transmutes(Transmutes)

滚开吧,类型系统!我们将重新解释这些比特(bits,位)死不罢休!虽然这本书都是关于做不安全事情的,但我真的不能强调,你应该深入思考找到另一种方式,而不是本节所涉及的操作.这真的是,真的,你在Rust中可以做的最可怕的不安全的事情.这里的护栏是牙线.

[`mem::transmute<T, U>`](https://doc.rust-lang.org/std/mem/fn.transmute.html)接受类型为`T`的值,并将其重新解释为类型为`U`.唯一的限制是验证`T`和`U`具有相同的大小.使用此方法导致未定义行为的方法令人难以置信.

- 首先,创建具有无效状态的 *任何(any)* 类型的实例将导致无法真正预测的任意混乱.不要将`3`transmute 为`bool`. 即使从未用`bool`做任何事情. 只是不要.

- Transmute有一个重载的返回类型.如果你没有指定返回类型,它可能会产生令人惊讶的类型以满足推断.

- 将&转换为&mut是UB

  - 将&转换为&mut *始终(always)* 是UB
  
  - 不,你不能做
  
  - 不,你不是特别的

- 在没有显式提供的生命周期的情况下,转换为引用会产生[无限制的生命周期](ch03-06-Unbounded-Lifetimes.md)

在不同的复合类型之间转换时,你必须确保它们以相同的方式排列! 如果布局不同,错误的字段将被错误的数据填充,这将使您不满意,并且可能会是UB(请参见上文).

那么如何知道布局是否相同? 对于`repr(C)`类型和`repr(transparent)`类型,布局是精确定义的. 但是对于普通的`repr（Rust）`,它不是. 即使是同一泛型类型的不同实例,布局也可能完全不同. `Vec<i32>`和`Vec<u32>` *可能* 有相同的字段顺序,也可能不相同. [在UCG WG](https://rust-lang.github.io/unsafe-code-guidelines/layout.html)上仍在研究数据布局的确切细节和不保证细节.

[`mem::transmute_copy<T, U>`](https://doc.rust-lang.org/std/mem/fn.transmute_copy.html)以某种方式设法比这 *甚至更加(even more)* 不安全.它将`size_of<U>`字节从`&T`中复制出来并将它们解释为`U`.`mem::transmute`的大小检查已经消失(因为它可能有效复制前缀),尽管它对于`U`大于`T`是未定义行为 .

当然,你也可以使用指针转换或`union`来获得这些函数的大部分功能.但没有任何lints或其他基本的健康检查. 原始指针强制转换和`union`不能神奇地避免上述规则.
