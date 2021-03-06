# Rust的表示(repr(Rust))

首先,所有类型都有以字节为单位指定的对齐方式.类型的对齐指定哪些地址有效以存储值.具有对齐`n`的值只能存储在`n`的倍数的地址中.因此,对齐2意味着你必须存储在偶数地址,1意味着你可以存储在任何地方.对齐至少为1,并且总是2的幂.

基元通常与它们的大小一致,尽管这是特定于平台的行为.特别是,在x86上,`u64`和`f64`通常对齐到4个字节(32位).

类型的大小必须始终是其对齐的倍数(0是任何对齐的有效大小).这可以确保始终通过偏移其大小的倍数来索引该类型的数组.请注意,在[动态大小类型](https://github.com/rust-lang-nursery/nomicon/blob/master/src/exotic-sizes.html#dynamically-sized-types-dsts)的情况下,可能不会静态地知道类型的大小和对齐.

Rust为你提供了以下布置复合数据的方法:

- 结构(命名积类型)

- 元组(匿名积类型)

- 数组(同类积类型)

- 枚举(命名和类型--标记联合)

- 联合(无标记联合)

如果没有任何变体具有关联数据,则称枚举为 *无字段(field-less)* .

默认情况下,复合结构的对齐方式等于其字段对齐的最大值.因此,Rust将在必要时插入填充,以确保所有字段都正确对齐,并且整体类型的大小是其对齐的倍数.例如:

```Rust
struct A {
    a: u8,
    b: u32,
    c: u16,
}
```

将在这些基元与各自大小对齐的架构上进行32位对齐.因此整个结构的大小是32位的倍数.它可能会成为:

```Rust
struct A {
    a: u8,
    _pad1: [u8; 3], // to align `b`
    b: u32,
    c: u16,
    _pad2: [u8; 2], // to make overall size multiple of 4
}
```

或可能:

```Rust
struct A {
    b: u32,
    c: u16,
    a: u8,
    _pad: u8,
}
```

这些类型 *没有间接性(no indirection)* ;所有数据都存储在结构中,正如你在C中所期望的那样.但是除了数组(它密集且按顺序排列)之外,默认情况下不指定数据布局.给出以下两个结构定义:

```Rust
struct A {
    a: i32,
    b: u64,
}

struct B {
    a: i32,
    b: u64,
}
```

Rust *确实(does)* 确保两个A的实例的数据布局以完全相同的方式.但是,Rust目前 *不(does not)* 保证A的实例具有与B的实例相同的字段排序或填充.

如A和B所写,这一点似乎很迂腐,但Rust的其他几个特性使得语言可以以复杂的方式使用数据布局.

例如,考虑这个结构:

```Rust
struct Foo<T, U> {
    count: u16,
    data1: T,
    data2: U,
}
```

现在考虑`Foo<u32, u16>`和`Foo<u16, u32>`的单形态.如果Rust以指定的顺序排列字段,我们希望它填充结构中的值以满足它们的对齐要求.因此,如果Rust没有重新排序字段,我们希望它产生以下内容:

```Rust
struct Foo<u16, u32> {
    count: u16,
    data1: u16,
    data2: u32,
}

struct Foo<u32, u16> {
    count: u16,
    _pad1: u16,
    data1: u32,
    data2: u16,
    _pad2: u16,
}
```

后一种情况只会浪费空间.因此,空间的最佳使用需要不同的单形态以具有 *不同的字段排序(different field orderings)* .

枚举使这一考虑变得更加复杂.天真的,如:

```Rust
enum Foo {
    A(u32),
    B(u64),
    C(u8),
}
```

可能被布局为:

```Rust
struct FooRepr {
    data: u64, // this is either a u64, u32, or u8 based on `tag`
    tag: u8,   // 0 = A, 1 = B, 2 = C
}
```

事实上,这大概就是它的布局方式(以`tag`的大小和位置为模).

然而,在某些情况下,这种表示效率低下.典型的例子是Rust的"空指针优化(null pointer optimization)":由单个外部单元变体(例如`None`)和(可能嵌套的)非可空指针(non-nullable pointer)变体(例如`Some(&T)`)组成的枚举使得标签不必要,因为空指针值可以安全地解释为单元(`None`)变体.最终结果是,例如,`size_of::<Option<&T>>() == size_of::<&T>()`.

Rust中有许多类型就是或包含非可空指针,如`Box<T>`,`Vec<T>`,`String`,`&T`和`&mut T`.同样,可以想象嵌套枚举将其标签汇集到单个判别式中,因为根据定义它们具有有限范围的有效值.原则上,枚举可以使用相当精细的算法来存储具有禁止值的嵌套类型中的位.因此,*特别(especially)*希望我们今天不要指定枚举布局.
