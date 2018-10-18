# Send和Sync(Send and Sync)

但是,并非一切都遵循继承的可变性.某些类型允许你在改变它时,拥有多个内存中位置的别名.除非这些类型使用同步(synchronization)来管理此访问,否则它们绝对不是线程安全的.Rust通过Send和Sync trait捕获这一点.

- 如果将其发送到另一个线程是安全的,则类型为Send.

- 如果在线程之间共享是安全的,则类型为Sync(`&T`为Send).

Send和Sync是Rust的并发故事的基础.因此,存在大量特殊工具以使它们正常工作.首先,它们是不安全trait.这意味着它们实现起来不安全,而其他不安全的代码可以假设它们已正确实现.由于它们是 *标记trait(marker traits)* (它们没有像方法那样的关联项),所以正确实现只是意味着它们具有实现者应具有的内在属性.错误地实现Send或Sync可能会导致未定义行为.

Send和Sync也是自动派生的traits.这意味着,与其他所有trait不同,如果类型完全由Send或Sync类型组成,则它是Send或Sync.几乎所有基元都是Send和Sync,因此几乎所有与之交互的类型都是Send和Sync.

主要例外包括:

- 原始指针既不是Send也不是Sync(因为它们没有安全防护).

- `UnsafeCell`不是Sync(因此`Cell`和`RefCell`不是).

- `Rc`不是Send或Sync(因为引用计数(refcount)是共享和不同步的).