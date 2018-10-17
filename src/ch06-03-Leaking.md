# 泄漏(Leaking)

基于所有权的资源管理旨在简化组合.你在创建对象时获取资源,并在销毁资源时释放资源.因为析构为你处理,这意味着你不会忘记释放资源,它会尽快发生!当然这是完美的,我们所有的问题都解决了.

一切都很糟糕,我们有新的和外来的问题试图解决.

许多人都喜欢相信Rust可以消除资源泄漏.在实践中,这基本上是正确的.你会惊讶地发现Safe Rust程序以不受控制的方式泄漏资源.

然而,从理论的角度来看,无论你如何看待它,情况绝对不是这样.从最严格的意义上说,"泄漏(leaking)"是如此抽象,以至于是不可避免的.在程序开始时初始化一个集合,用大量带有析构函数的对象填充它,然后输入一个永远不会引用它的无限事件循环,这是非常简单的.该集合将毫无用处地存在,保留其宝贵的资源,直到程序终止(此时所有这些资源都将由操作系统回收).

我们可能会考虑更严格的泄漏形式:未能删除无法访问的值.Rust也不会阻止这种情况.实际上Rust *有这样做的函数(has a function for doing this)* :`mem::forget`.此函数消费传递给它的值, *然后不运行其析构函数(and then doesn't run its destructor)* .

在过去,`mem::forget`被标记为不安全作为一种使用它的提示(lint),因为无法调用析构函数通常不是一件好事(尽管对某些特殊的不安全代码很有用).然而,这通常被认为是一种站不住脚的立场:有许多方法无法在安全代码中调用析构函数.最着名的例子是使用内部可变性创建一个引用计数指针的循环.

安全代码假设析构函数泄漏不会发生是合理的,因为泄漏析构函数的任何程序都可能是错误的.但是, *不安全(unsafe)* 代码不能依赖析构函数来运行以确保安全.对于大多数类型而言,这并不重要:如果泄漏析构函数,那么类型根据定义是不可访问的,所以无所谓,对吧?例如,如果你泄漏一个`Box<u8>`,那么你会浪费一些内存,但这几乎不会违反内存安全.

但是,我们必须小心使用析构函数泄漏是 *代理(proxy)* 类型.这些是管理对不同对象的访问的类型,但实际上并不拥有它.代理对象非常罕见.你需要关注的代理对象甚至更少见.但是,我们将关注标准库中的三个有趣示例:

- `vec::Drain`

- `Rc`

- `thread::scoped::JoinGuard`

## Drain(Drain)

`drain`是一个集合API,可以在不消费容器的情况下将数据移出容器.这使我们能够在声明对其所有内容的所有权后重用`Vec`的分配.它生成一个迭代器(Drain),通过值返回Vec的内容.

现在,在迭代过程中考虑Drain:一些值已被移出,而其他值则没有.这意味着Vec的一部分现在充满了逻辑上未初始化的数据!每次我们移除一个值时,我们都可以对Vec中的所有元素进行反向移动,但这会产生非常灾难性的性能后果.

相反,我们希望在Vec的备份存储被删除时,将其删除以修复它.它应该自己运行完成,反向移动任何未被删除的元素(drain支持子范围),然后修复Vec的`len`.它甚至是展开安全的(unwinding-safe)!简单!

现在考虑以下内容:

```Rust
let mut vec = vec![Box::new(0); 4];

{
    // start draining, vec can no longer be accessed
    let mut drainer = vec.drain(..);

    // pull out two elements and immediately drop them
    drainer.next();
    drainer.next();

    // get rid of drainer, but don't call its destructor
    mem::forget(drainer);
}

// Oops, vec[0] was dropped, we're reading a pointer into free'd memory!
println!("{}", vec[0]);
```

这显然不是很好.不幸的是,我们有点陷入左右为难的境地:在每一步保持一致的状态会产生巨大的成本(并且会否定API的任何好处).未能保持一致状态会在安全代码中给出未定义行为(使API不健全).

所以,我们能做些什么?好吧,我们可以选择一个简单的一致状态:当我们开始迭代时将Vec的len设置为0,并在必要时在析构函数中修复它.这样,如果一切都像正常一样执行,我们会以最小的开销获得所需的行为.但是,如果某人 *大胆的(audacity)* 在迭代过程中mem::forget我们,那么所有这一切都会 *泄漏更多(leak even more)* (并且可能使Vec处于非预期的但其他方面一致的状态).因为我们已经接受了mem::forget是安全的,所以这绝对是安全的.我们称泄漏导致更多泄漏为 *泄漏放大(leak amplification)* .

## Rc(Rc)

Rc是一个有趣的案例,因为乍一看它似乎根本不是代理值.毕竟,它管理它指向的数据,并且删除值的所有Rcs(引用计数)将删除该值.泄漏Rc似乎并不特别危险.它将使引用计数(refcount)永久递增并防止数据被释放或删除,但这看起来就像Box,对吧?

不.

让我们考虑一个Rc的简化实现:

```Rust
struct Rc<T> {
    ptr: *mut RcBox<T>,
}

struct RcBox<T> {
    data: T,
    ref_count: usize,
}

impl<T> Rc<T> {
    fn new(data: T) -> Self {
        unsafe {
            // Wouldn't it be nice if heap::allocate worked like this?
            let ptr = heap::allocate::<RcBox<T>>();
            ptr::write(ptr, RcBox {
                data: data,
                ref_count: 1,
            });
            Rc { ptr: ptr }
        }
    }

    fn clone(&self) -> Self {
        unsafe {
            (*self.ptr).ref_count += 1;
        }
        Rc { ptr: self.ptr }
    }
}

impl<T> Drop for Rc<T> {
    fn drop(&mut self) {
        unsafe {
            (*self.ptr).ref_count -= 1;
            if (*self.ptr).ref_count == 0 {
                // drop the data and then free it
                ptr::read(self.ptr);
                heap::deallocate(self.ptr);
            }
        }
    }
}
```

这段代码包含一个隐式的,微妙的假设:`ref_count`可以适用于`usize`,因为在内存中不能超过`usize::MAX`Rcs.然而,这本身假设`ref_count`准确地反映了内存中的Rcs数量,我们知道使用`mem::forget`是错误的.使用`mem::forget`我们可以溢出`ref_count`,然后使用优秀的Rcs将其降低到0.然后我们可以愉快地在释放后使用(use-after-free)内部数据.Bad Bad Not Good.

这可以通过检查`ref_count`并执行 *某些操作(something)* 来解决.标准库的立场就是中止,因为你的程序变得非常堕落. *哦,我的天啊(oh my gosh)* ,这是一个荒谬的角落案件.

## thread::scoped::JoinGuard(thread::scoped::JoinGuard)

thread::scoped API打算允许生成线程,这些线程通过确保父级在任何共享数据超出作用域之前加入线程而引用在其父级堆栈上的数据而不对该数据进行任何同步.

```Rust
pub fn scoped<'a, F>(f: F) -> JoinGuard<'a>
    where F: FnOnce() + Send + 'a
```

这里`f`是其他线程执行的一些闭包.就是说`F: Send +'a`表示它关闭了活为`'a`的数据,它拥有该数据或者数据是Sync(暗示`&data`是Send).

因为JoinGuard具有生命周期,所以它保留了在父线程中借用的所有数据.这意味着JoinGuard不能活得超过其他线程正在处理的数据.当JoinGuard被删除时,它会阻塞父线程,确保子(线程)在父(线程)上的任何已关闭的数据超出作用域之前终止.

用法看起来像:

```Rust
let mut data = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
{
    let guards = vec![];
    for x in &mut data {
        // Move the mutable reference into the closure, and execute
        // it on a different thread. The closure has a lifetime bound
        // by the lifetime of the mutable reference `x` we store in it.
        // The guard that is returned is in turn assigned the lifetime
        // of the closure, so it also mutably borrows `data` as `x` did.
        // This means we cannot access `data` until the guard goes away.
        let guard = thread::scoped(move || {
            *x *= 2;
        });
        // store the thread's guard for later
        guards.push(guard);
    }
    // All guards are dropped here, forcing the threads to join
    // (this thread blocks here until the others terminate).
    // Once the threads join, the borrow expires and the data becomes
    // accessible again in this thread.
}
// data is definitely mutated here.
```

原则上,这完全有效!Rust的所有权系统完美地确保了它!...只不过它依赖于析构函数被安全调用.

```Rust
let mut data = Box::new(0);
{
    let guard = thread::scoped(|| {
        // This is at best a data race. At worst, it's also a use-after-free.
        *data += 1;
    });
    // Because the guard is forgotten, expiring the loan without blocking this
    // thread.
    mem::forget(guard);
}
// So the Box is dropped here while the scoped thread may or may not be trying
// to access it.
```

该死.在这里,析构函数的运行对于API来说是非常基本的,必须废弃它以支持完全不同的设计.