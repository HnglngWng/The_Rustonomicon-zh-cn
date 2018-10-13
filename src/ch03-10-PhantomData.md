# PhantomData(PhantomData)

使用不安全的代码时,我们常常会遇到类型或生命周期与结构逻辑上关联但实际上不是字段的一部分的情况.这通常发生在生命周期.例如,`&'a [T]`的`Iter`(大致)定义如下:

```Rust
struct Iter<'a, T: 'a> {
    ptr: *const T,
    end: *const T,
}
```

然而,因为`'a`在结构体内未被使用,所以它是 *无限制的(unbounded)* .由于历史上造成的麻烦,结构定义中 *禁止(forbidden)* 无限制的生命周期和类型.因此,我们必须以某种方式在主体中引用这些类型.正确地执行此操作对于获得正确的可变性和删除检查是必要的.

我们使用`PhantomData`执行此操作,它是一种特殊的标记类型.`PhantomData`不占用空间,而是模拟给定类型的字段,以便进行静态分析,.这比显式地告诉类型系统你想要的可变性种类更不容易出错,同时还提供其他有用的信息,比如删除检查所需的信息.

`Iter`逻辑上包含一系列`&'a T`,所以这正是我们告诉PhantomData模拟的:

```Rust
use std::marker;

struct Iter<'a, T: 'a> {
    ptr: *const T,
    end: *const T,
    _marker: marker::PhantomData<&'a T>,
}
```

就是这样.生命周期将是有限制的,你的迭代器将对`'a`和`T`可变.一切正常.

另一个重要的例子是`Vec`,其(大致)定义如下:

```Rust
struct Vec<T> {
    data: *const T, // *const for variance!
    len: usize,
    cap: usize,
}
```

与前面的示例不同, *似乎(appears)* 一切都完全符合我们的要求.Vec的每个泛型参数都出现在至少一个字段中.准备好了!

不.

删除检查器将慷慨地确定`Vec<T>`不拥有类型`T`的任何值.这将反过来使得它不需要担心Vec在其析构函数中删除任何T以确定删除检查的健全性.这将反过来允许人们使用Vec的析构函数创建不健全.

为了告诉dropck我们 *确实(do)* 拥有类型T的值,因此当 *我们(we)* 删除时可能会删除一些T,我们必须添加一个额外的PhantomData来说明:

```Rust
use std::marker;

struct Vec<T> {
    data: *const T, // *const for variance!
    len: usize,
    cap: usize,
    _marker: marker::PhantomData<T>,
}
```

拥有分配的原始指针是如此普遍的模式,标准库为自己创建了一个名为`Unique<T>`的工具:

- 包装`*const T`的可变性

- 包括`PhantomData<T>`

- 自动派生`Send`/`Sync`,就像包含T一样

- 将指针标记为`NonZero`以进行空指针优化

## PhantomData模式表(Table of PhantomData patterns)

这是可以使用`PhantomData`的所有奇妙方式的表格:

|Phantom type|`'a`| `T`|
|--|--|--|
|`PhantomData<T>`|-|可变(具有删除检查)|
|`PhantomData<&'a T>`|可变|可变|
|`PhantomData<&'a mut T>`|可变|不变|
|`PhantomData<*const T>`|-|可变|
|`PhantomData<*mut T>`|-|不变|
|`PhantomData<fn(T)>`|-|逆变(*)|
|`PhantomData<fn() -> T>`|-|可变|
|`PhantomData<fn(T) -> T>`|-|不变|
|`PhantomData<Cell<&'a ()>>`|不变|-|

(*)如果逆变被废弃,这将是不变的