# 奇异的Sized Type(Exotically Sized Types)

大多数时候,我们认为类型具有静态已知,正数大小.Rust中的情况并非总是如此.

# 动态大小类型(Dynamically Sized Types(DSTs))

Rust支持动态大小类型(DSTs):没有静态已知大小或对齐的类型.从表面上看,这有点荒谬:Rust *必须(must)* 知道某些东西的大小和对齐方式才能正确使用它!在这方面,DSTs不是正常类型.因为它们缺少静态已知的大小,所以这些类型只能存在于指针后面.任何指向DST的指针都会变成一个 *宽(wide)* 指针,由指针和"完成(completes)"它们的信息组成(下面将详细介绍).

该语言暴露了两种主要的DSTs:

- trait objects: `dyn MyTrait`

- 切片:`[T]`, `str`, 和其它

trait对象表示实现其指定的trait的某种类型.使用包含使用该类型所需的所有信息的虚表(vtable)来 *擦除(erased)* 确切的原始类型以支持运行时反射.完成trait对象指针的信息是虚表指针.可以从虚表动态请求指针对象的运行时大小.

切片只是一些连续存储的视图--通常是数组或`Vec`.完成切片指针的信息就是它指向的元素数量.指针对象的运行时大小就是一个元素的静态已知大小乘以元素的数量.

结构实际上可以直接存储单个DST作为它们的最后一个字段,但这也使它们成为DST:

```Rust
// Can't be stored on the stack directly
struct Foo {
    info: u32,
    data: [u8],
}
```

尽管如果没有构造它的方法,这样的类型在很大程度上是无用的. 目前,唯一正确支持的创建自定义DST的方法是使你的类型泛型,并执行 *unsizing coercion* :

```Rust
struct MySuperSliceable<T: ?Sized> {
    info: u32,
    data: T,
}

fn main() {
    let sized: MySuperSliceable<[u8; 8]> = MySuperSliceable {
        info: 17,
        data: [0; 8],
    };

    let dynamic: &MySuperSliceable<[u8]> = &sized;

    // prints: "17 [0, 0, 0, 0, 0, 0, 0, 0]"
    println!("{} {:?}", dynamic.info, &dynamic.data);
}
```

(是的,目前,自定义DSTs很大程度上是一个半生不熟的功能.)

# 零大小类型(Zero Sized Types (ZSTs))

Rust也允许指定不占用空间的类型:

```Rust
struct Foo; // No fields = no size

// All fields have no size = no size
struct Baz {
    foo: Foo,
    qux: (),      // empty tuple has no size
    baz: [u8; 0], // empty array has no size
}
```

就其自身而言,零大小类型(ZSTs)由于显而易见的原因而非常无用.然而,正如Rust中许多奇特的布局选择一样,它们的潜力是在泛型上下文中实现的:Rust很大程度上理解生成或存储ZST的任何操作都可以简化为无操作(no-op).首先,存储它甚至没有意义--它不占用任何空间.此外,这种类型只有一个值,所以加载它的任何东西都可以从以太产生它--这也是一种无操作,因为它不占用任何空间.

其中一个最极端的例子是集(Sets)和映射(Map).给定`Map<Key, Value>`,通常将`Set<Key>`实现为`Map<Key, UselessJunk>`的一个薄包装器.在许多语言中,这将需要为UselessJunk分配空间并且存储和加载UselessJunk仅用来丢弃它.证明这是不必要的对编译器来说是一个困难的分析.

但是在Rust中,我们可以说`Set<Key> = Map<Key, ()>`.现在Rust静态地知道每个加载和存储都是无用的,并且没有任何大小的分配.结果是此单态化代码基本上是HashSet的自定义实现,没有HashMap必须支持值的开销.

安全代码不必担心ZSTs,但 *不安全的(unsafe)* 代码必须小心没有大小的类型的后果.特别是,指针偏移是无操作,并且当请求零大小的分配时,标准分配器可能返回`null`,这与内存不足结果无法区分.

# 空类型(Empty Types)

Rust还允许声明甚至 *无法实例化的类型(cannot even be instantiated)* .这些类型只能在类型级别进行讨论,而不能在值级别进行讨论.可以通过指定不带变体的枚举来声明空类型:

```Rust
enum Void {} // No variants = EMPTY
```

空类型比ZSTs更加边缘化.Void类型的主要激励示例是类型级不可达性.例如,假设API通常需要返回Result,但实际上特定情况是绝对可靠的.实际上,可以通过返回`Result<T, Void>`在类型级别进行通信.API的消费者可以自信地打开这样的Result,因为知道这个值在 *静态上不可能(statically impossible)* 是`Err`,因为这需要提供一个`Void`类型的值.

原则上,Rust可以基于这个事实做一些有趣的分析和优化.例如,`Result<T, Void>`就表示为`T`,因为`Err`情况实际上并不存在(严格来说,这只是一个无法保证的优化,因此,例如将一个转换为另一个仍然是UB).

以下也 *可以(could)* 编译:

```Rust
enum Void {}

let res: Result<u32, Void> = Ok(0);

// Err doesn't exist anymore, so Ok is actually irrefutable.
let Ok(num) = res;
```

但是这个技巧还不行.

关于空类型的最后一个细微的细节是,指向它们的原始指针实际上是可以构造的,但是解引用它们是Undefined Behavior,因为这没有意义.

我们建议不要使用`* const Void`对C的`void *`类型进行建模.很多人开始这样做,但很快就遇到了麻烦,因为Rust没有任何安全措施来防止尝试用不安全的代码实例化空类型,如果你这样做,那就是Undefined Behaviour.这尤其成问题,因为开发人员习惯将原始指针转换为引用,而构造`&Void` *也(also)* 是Undefined Behaviour.

`*const ()`(或等效的)适用于`void*`,并且可以在没有任何安全问题的情况下成为引用. 它仍然不会阻止你尝试读取或写入值,但至少它会编译为无操作而不是UB.

# 外部类型(Extern Types)

有一个[被接受的RFC](https://github.com/rust-lang/rfcs/blob/master/text/1861-extern-types.md)来添加具有未知大小的正确类型,称为 *extern types* ,这将使Rust开发人员更准确地对C的`void*`和其他"声明但未定义"类型进行建模.但是,从Rust 2018开始,该功能在`size_of::<MyExternType>()`行为应该如何上陷入了困境.
